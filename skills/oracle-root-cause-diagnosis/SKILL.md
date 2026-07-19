
# Oracle Root-Cause Diagnosis: Design Problem or Database Problem? — AI Skill Definition

## Purpose

This skill teaches AI assistants how to correctly diagnose the root cause of an Oracle performance issue before recommending a fix — including recognizing when a proposed index won't help because the real problem was never on the database side at all. Use it whenever asked to explain a slow batch job or business process, review a "let's add an index" suggestion, investigate why something that ran fine in testing has slowed down in production, or generally reason about where performance work should be aimed.

## Goal

The goal of this skill is not "make the query faster." It is: **before recommending any technical fix, determine whether the bottleneck is a database problem or a design problem, because only one of those is fixable by touching the database.**

Hold two hypotheses open until evidence decides between them:

1. **Database problem** — a single unit of work (one SQL statement, one PL/SQL call) is intrinsically expensive because of a bad execution plan, a missing index, stale statistics, or contention.
2. **Design problem** — the unit of work itself is fine, but the surrounding process invokes it far more times than the business logic actually requires.

An agent that defaults straight to hypothesis 1 — "let's add an index" — will sometimes be right and sometimes waste a cycle solving the wrong problem while the real cost driver, an inflated execution count, goes untouched.

## The Model: Total Cost = Cost Per Execution × Number of Executions

Every measured slowdown decomposes into two independent factors: how expensive one execution of the logic is, and how many times that logic executes. Database-side tuning — indexes, hints, statistics, plan changes — can only ever move the first factor. It has no leverage at all over the second. If a piece of logic is fast per call but is being called far more times than the business process actually needs, the total cost can dominate the whole batch, and no amount of database tuning on that call will fix it, because each individual call was never the problem.

This is the trap: a slow batch process *looks* like a database problem, because the symptom shows up as "this batch is slow" or "this query runs a lot." But "runs a lot" and "runs slowly" are different diagnoses, and only one of them responds to an index.

## Reading the Signal: "Each Execution Was Fast — the Design Wasn't"

The clearest tell that you're looking at a design problem rather than a database problem is when individual operations profile as fast in isolation, yet the aggregate process is slow. If you pull one execution of the suspect logic and run it standalone, it comes back quickly — sub-second, no red flags in the plan. That's a strong signal the database is doing its job correctly on each call. The problem, if there is one, is upstream: something in the process design is invoking that fast operation far more often than the business rule requires — inside a loop, once per row, once per line item, once per iteration — when it could have been computed once, cached, or converted into a single set-based operation.

## The Diagnostic Loop

**Step 1 — Define the goal explicitly.** Before touching any tooling, state the actual question: "is this a database problem or a design problem?" — not "how do I make this faster." That framing keeps an index from becoming the reflexive first answer.

**Step 2 — Measure, don't guess.** Gather two numbers for the suspect logic: the cost of a single execution, and the number of times it actually executes during one run of the process. Guessing either number from the code alone is unreliable — instrument and measure.

For SQL executed directly, `v$sql` gives both numbers together:

```sql
SELECT sql_id,
       executions,
       ROUND(elapsed_time / 1e6, 3)                          AS total_elapsed_sec,
       ROUND(elapsed_time / NULLIF(executions, 0) / 1e6, 4)   AS avg_elapsed_sec
FROM   v$sql
WHERE  sql_text LIKE '%<distinctive fragment>%'
ORDER  BY executions DESC;
```

An `executions` count that scales with the batch's row count — rather than staying constant regardless of batch size — is the signature of a design problem: the same logic is being invoked once per row instead of once per batch.

For PL/SQL function or procedure calls, which don't always show up in `v$sql`, use the hierarchical profiler to get exact call counts:

```sql
BEGIN
  DBMS_HPROF.START_PROFILING(location => 'PROFILER_OUTPUT', filename => 'batch_run.trc');
END;
/
-- run the batch process under test here

BEGIN
  DBMS_HPROF.STOP_PROFILING;
END;
/

-- load the trace, then inspect call counts per subprogram
SELECT owner, module, function, calls,
       ROUND(function_elapsed_time / 1e6, 3) AS elapsed_sec
FROM   dbmshp_function_info
ORDER  BY calls DESC
FETCH FIRST 15 ROWS ONLY;
```

A function with a modest per-call elapsed time but a `calls` count in the tens of thousands for a batch of a few thousand business records is exactly the pattern this skill is built around: cheap per call, expensive because of how often it's called.

**Step 3 — Ask the root question, out loud, before proposing a fix.** *"Is this really a database problem, or is it a design problem?"* Say it as a checkpoint every time a performance issue reaches the point of proposing a technical change. It costs nothing and it's one of the highest-leverage questions in performance work.

**Step 4 — Branch on the evidence.**
- If per-call cost is high and call count is reasonable → database problem. Go to plan analysis, indexing, and statistics.
- If per-call cost is low and call count is inflated relative to the business unit of work (rows, customers, line items) → design problem. The fix is in the application or PL/SQL logic: hoist loop-invariant computations out of the loop, cache lookups instead of re-deriving them, or replace row-by-row (RBAR) calls with one set-based SQL statement.
- If both are true, fix the design first — collapsing the execution count usually removes most of the total cost, and whatever database-level tuning is still worthwhile becomes much cheaper to test once the batch runs the suspect logic once instead of thousands of times.

**Step 5 — Verify the fix targets the number that was actually broken.** After changing the design, re-measure the call count, not just the wall-clock time. Wall-clock time can improve for the wrong reason. Confirming the execution count actually dropped is what proves the diagnosis was correct.

## Example Pattern: Collapsing a Loop-Invariant Call

Illustrative before/after — the specifics vary, but the shape recurs constantly in batch processes: a row-by-row loop makes an isolated call per record for something that is either identical across many rows or could be computed as a single set-based operation.

```sql
-- BEFORE: row-by-row (RBAR). One call to get_discount_rate() per order row —
-- N executions for a batch of N orders, even though many rows share a customer.
FOR rec IN (SELECT order_id, customer_id, amount
            FROM   orders
            WHERE  batch_id = p_batch_id) LOOP
    v_rate := get_discount_rate(rec.customer_id);      -- N calls
    UPDATE orders
    SET    discounted_amount = rec.amount * (1 - v_rate)
    WHERE  order_id = rec.order_id;                    -- N updates
END LOOP;
```

```sql
-- AFTER: one set-based statement. The lookup becomes a join, and the whole
-- batch is updated in a single execution instead of N.
UPDATE orders o
SET    discounted_amount = o.amount * (1 - (
           SELECT c.discount_rate
           FROM   customer_discount_rates c
           WHERE  c.customer_id = o.customer_id
       ))
WHERE  o.batch_id = p_batch_id;
```

Where a pure join isn't feasible because the logic is genuinely procedural, the intermediate fix is to cache the loop-invariant value once per distinct key instead of re-deriving it every row:

```sql
DECLARE
  TYPE rate_tab IS TABLE OF NUMBER INDEX BY PLS_INTEGER;
  v_rates rate_tab;
BEGIN
  FOR r IN (SELECT customer_id, discount_rate FROM customer_discount_rates) LOOP
    v_rates(r.customer_id) := r.discount_rate;         -- one pass, not one per order
  END LOOP;

  FOR rec IN (SELECT order_id, customer_id, amount
              FROM   orders WHERE batch_id = p_batch_id) LOOP
    UPDATE orders
    SET    discounted_amount = rec.amount * (1 - v_rates(rec.customer_id))
    WHERE  order_id = rec.order_id;
  END LOOP;
END;
/
```

Both rewrites keep the exact same business rule. Neither touches an index. The entire performance gain comes from cutting the execution count of the logic from N down to one — the kind of improvement no index could ever produce, because the original per-call cost was never the bottleneck.

## Anti-Patterns to Avoid When Generating Oracle Performance Guidance

| Anti-pattern | Why it's wrong | Do this instead |
|---|---|---|
| Recommending an index as the first response to "this batch process is slow" | An index only reduces per-call cost; it does nothing for an inflated execution count | Measure execution count for the suspect logic before proposing any database-level fix |
| Assuming a fast individual query means the whole process must be fine | A cheap operation called thousands of times can still dominate total runtime | Multiply per-call cost by call count before concluding a component is "not the problem" |
| Profiling only at the SQL level when the suspect logic is PL/SQL | `v$sql` won't show call counts for logic that doesn't issue its own SQL each time | Use `DBMS_HPROF` (hierarchical profiler) to get exact per-subprogram call counts |
| Declaring a fix successful because wall-clock time improved | Wall-clock time can drop for reasons unrelated to the actual diagnosis | Re-measure the execution count specifically to confirm it actually dropped |
| Treating "design problem" and "database problem" as mutually exclusive | Both can coexist; an inflated call count can mask a genuinely bad per-call plan too | Fix the design first, then re-run the database-level diagnostics on the reduced workload |

## Quick Mental Model

An index makes one expensive thing cheaper. It cannot make one cheap thing stop happening ten thousand times. Before reaching for a database-level fix, ask whether the thing that's slow is a single operation that's genuinely expensive, or a cheap operation that's been multiplied by a design decision made somewhere upstream. The biggest performance wins usually come from questioning that design decision, not from tuning the database around it.

## Related Oraxia Skills

- `oracle-performance/` — execution plans, indexing, hints, AWR/ASH, and optimizer statistics for when the evidence points to a genuine database problem.
- `oracle-plsql/` — BULK COLLECT / FORALL and other set-based rewrites for collapsing row-by-row PL/SQL into set-based operations.
