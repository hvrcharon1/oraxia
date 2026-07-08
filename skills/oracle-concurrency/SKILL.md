# Oracle Locking & Concurrency — AI Skill Definition

## Purpose
This skill teaches AI assistants how Oracle Database actually resolves concurrent writes at the **transaction** level — including several counter-intuitive but well-documented behaviors around `SAVEPOINT`, `ROLLBACK TO SAVEPOINT`, row-level locking (ITL/Uba), transaction-level wait queues, and 23ai `CREATE ASSERTION` enqueues. Use it when asked to design retry logic, insert-or-update patterns, contention-heavy schemas, or to explain "why did my blocked session get a duplicate-key error instead of succeeding?"

---

## The Core Model: Locks Live On the Data, Waits Live On the Transaction

Oracle does not keep a central lock table the way some other engines do. Instead:

- Every locked row carries a pointer, in its block's **Interested Transaction List (ITL)**, to the transaction that locked it and to the **undo record (Uba)** needed to undo that change.
- A session that wants to modify a row already locked by another transaction does not keep re-checking the row's contents. It parks itself in a wait queue **keyed on the blocking transaction ID**, not on the row or the key value.
- That queued session is only woken up when the blocking transaction **commits or fully rolls back** — not when an intermediate `ROLLBACK TO SAVEPOINT` changes what the row logically contains.

This distinction — "waiting on a transaction" vs. "waiting on a row's current visible value" — is the source of most of the surprising behavior below, and it differs from RDBMSs that re-evaluate visibility per-tuple rather than per-transaction.

---

## Savepoints Really Do Rewrite History (Within the Transaction)

```sql
CREATE TABLE seat_reservations (
    seat_id NUMBER PRIMARY KEY
);

INSERT INTO seat_reservations VALUES (100);

SAVEPOINT before_seat_101;
INSERT INTO seat_reservations VALUES (101);
-- session now sees seat_id 100 and 101

ROLLBACK TO SAVEPOINT before_seat_101;
-- seat_id 101 is gone -- not just uncommitted, but as if it never
-- existed. The index entry, the ITL lock slot, everything reverts.

INSERT INTO seat_reservations VALUES (102);
COMMIT;   -- or ROLLBACK -- the seat_id=100 row is unaffected either way
```

After `ROLLBACK TO SAVEPOINT`, the block that held seat_id 101 loses its row entry entirely — free space increases, the ITL lock count for that transaction drops, and the undo pointer walks backward by exactly the undone records. As far as the block is concerned, that insert never happened. The transaction is still open; only the work since the savepoint is undone.

---

## The Counter-Intuitive Part: A Blocked Waiter Doesn't Get Freed by a Savepoint Rollback

Walk through three sessions against the table above:

1. **Session A** — `INSERT ... VALUES (101)`, then `SAVEPOINT sp1`.
2. **Session B** — also tries `INSERT ... VALUES (101)`. It hangs, having joined the wait queue for **Session A's transaction**.
3. **Session A** — `ROLLBACK TO SAVEPOINT sp1`. Seat 101 is now, logically, never inserted. **Session B is still waiting** — it queued on A's transaction ID, not on the seat_id=101 key, so nothing about the savepoint rollback wakes it up or re-checks the row.
4. **Session C** (fresh session, never previously queued) — `INSERT ... VALUES (101)` **succeeds immediately**. Nothing in the index says seat 101 was ever touched, so C never joins any queue at all.
5. **Session A** finally issues a full `ROLLBACK` (or `COMMIT`). Session B wakes up, retries its insert, discovers Session C's now-uncommitted row with the same key, and **re-queues itself — this time behind Session C** — still with no error, just a continued wait.

Net effect: a session already parked in line keeps waiting even after the row it was waiting on stops existing, while a session that shows up "late" but at the right moment can walk straight through. Wait order is not arrival order.

**Practical implication for generated code:** don't build retry/backoff logic that assumes "the conflicting row is logically gone now, so my blocked insert should succeed." Rely on catching `ORA-00001` and retrying a fresh statement rather than trusting that an in-flight wait resolves itself favorably.

---

## Reading the Modern (23ai) Duplicate-Key Error

Recent Oracle patch trains return a more detailed, multi-line error instead of the classic one-liner:

```
insert into seat_reservations values (101)
*
ERROR at line 1:
ORA-00001: unique constraint (APP.SYS_C0012345) violated on table
APP.SEAT_RESERVATIONS columns (SEAT_ID)
ORA-03301: (ORA-00001 details) row with column values (SEAT_ID:101) already exists
Help: https://docs.oracle.com/error-help/db/ora-00001/
```

When generating error-handling code, still match on `ORA-00001` for backward compatibility, but surface the `ORA-03301` detail line when present — it names the actual conflicting value, a real debugging upgrade over versions that report only the constraint name.

---

## Diagnosing "Who Is Really Blocking Whom"

Because a waiter re-queues onto whichever transaction currently holds the conflicting key, the session reported as "blocking" can be stale by the time you look at it. Join on the lock resource, not just on session id:

```sql
-- Blocking chain: who is waiting on whom, right now
SELECT
    w.sid          AS waiting_sid,
    w.serial#      AS waiting_serial,
    b.sid          AS blocking_sid,
    b.serial#      AS blocking_serial,
    o.object_name,
    l.type,
    l.lmode,
    l.request
FROM v$lock l
JOIN v$session w ON w.sid = l.sid AND l.request > 0
JOIN v$lock bl   ON bl.id1 = l.id1 AND bl.id2 = l.id2 AND bl.lmode > 0
JOIN v$session b ON b.sid = bl.sid
LEFT JOIN dba_objects o ON o.object_id = l.id1
ORDER BY waiting_sid;

-- Savepoints currently defined for active transactions (in-memory only --
-- there is no file-based record of savepoints)
SELECT * FROM x$ktcsp;
```

All savepoints created during a transaction persist in `x$ktcsp` until the transaction fully commits or rolls back — even ones already superseded by a later `ROLLBACK TO SAVEPOINT`. Don't assume rolling back to an earlier savepoint clears its own bookkeeping entry; it doesn't, until the whole transaction ends.

---

## 23ai `CREATE ASSERTION`: A Different, Statement-Level Enqueue

23ai's declarative `CREATE ASSERTION` cross-row constraints use a distinct **AN enqueue** (keyed on a hash of the changed values) rather than the ITL-based row lock described above:

```sql
CREATE ASSERTION seat_unique_check
    CHECK (NOT EXISTS (
        SELECT 1 FROM seat_reservations a, seat_reservations b
        WHERE a.seat_id = b.seat_id AND a.rowid != b.rowid
    ));
```

- The AN enqueue is taken **at the statement level**, whether the assertion is declared deferred or immediate.
- The first session to insert a given value holds the enqueue until its **whole transaction** commits or rolls back — a `ROLLBACK TO SAVEPOINT` inside that transaction does **not** release it.
- Waiters queue up behind whoever currently holds the enqueue — the same "line-cutting" risk described above applies: queue order can shift to a different transaction entirely once the original holder finishes.
- Availability varies by release/edition. Confirm `CREATE ASSERTION` is actually supported in the target Oracle version/edition before generating code that depends on it — some Free/XE builds do not support it yet.

---

## Anti-Patterns to Avoid When Generating Oracle Concurrency Code

| Anti-pattern | Why it's wrong | Do this instead |
|---|---|---|
| Treating `ROLLBACK TO SAVEPOINT` as releasing locks/enqueues taken earlier in the transaction | Only a full `COMMIT`/`ROLLBACK` releases them | Use savepoints for logical undo of business steps, never for concurrency control |
| Assuming blocked sessions succeed in arrival order | A session queued later against a different, still-open transaction can complete first | Never rely on wait order for correctness; use `SELECT ... FOR UPDATE` plus explicit application-level sequencing when order matters |
| Polling a row's visible state to decide whether a blocked insert "should" now succeed | The queued session isn't watching the row at all — it's watching a transaction id | Catch `ORA-00001`, roll back the failed statement, and retry cleanly |
| Assuming a rolled-back-to-savepoint row is completely gone from disk | Undo records for it still physically exist until the transaction ends (visible via block dumps / LogMiner) | Don't rely on savepoint rollback for scrubbing sensitive data pre-commit |
| Using `CREATE ASSERTION` as a drop-in replacement for a `UNIQUE` index without checking enqueue cost | Assertions add statement-level AN-enqueue contention on top of the uniqueness check itself | Prefer `UNIQUE` constraints/indexes for simple single-table uniqueness; reserve assertions for genuine cross-row/cross-table invariants |

---

## Quick Reference

| Need | Use |
|---|---|
| See lock holders / waiters | `v$lock`, `v$session` |
| See a transaction's savepoints | `x$ktcsp` |
| See the undo pointer behind a row's lock | Block dump (`ALTER SYSTEM DUMP DATAFILE ... BLOCK ...`) — Itl / Uba columns |
| Detailed duplicate-key diagnosis | `ORA-03301` line under `ORA-00001` (23ai+) |
| Cross-row declarative constraint (23ai+) | `CREATE ASSERTION` — statement-level AN enqueue |

---

*Background: this reflects a deliberate trade-off from Oracle's original (v6) transaction-based locking design, which avoided maintaining a separate central lock table by storing lock state directly on the row/ITL and clearing it lazily. Other MVCC engines that re-evaluate tuple visibility per-waiter (rather than per-transaction) don't reproduce this exact behavior — useful context when comparing cross-database concurrency semantics, but not a reason to change Oracle's locking model in generated code.*
