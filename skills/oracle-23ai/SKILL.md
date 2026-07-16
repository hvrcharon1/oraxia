# Oracle 23ai — AI Skill Definition

## Purpose
This skill teaches AI assistants to use Oracle Database 23ai's new features correctly and idiomatically. Always apply these patterns when targeting Oracle 23ai (also released as 23c Free). Oracle 23ai adds developer-friendly syntax, native AI/vector capabilities, and richer JSON and graph support.

---

## Boolean Data Type

Oracle 23ai introduces a native `BOOLEAN` column type for SQL, eliminating the historical `NUMBER(1)` or `CHAR(1)` workarounds. It accepts the literals `TRUE`, `FALSE`, and `NULL`. PL/SQL's longstanding `BOOLEAN` type now shares the same underlying representation, so you can pass boolean values between SQL and PL/SQL without conversion.

```sql
CREATE TABLE feature_flags (
    flag_id     NUMBER        GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    flag_name   VARCHAR2(100) NOT NULL,
    is_enabled  BOOLEAN       DEFAULT FALSE NOT NULL,
    is_public   BOOLEAN       DEFAULT TRUE  NOT NULL,
    created_at  TIMESTAMP     DEFAULT SYSTIMESTAMP NOT NULL
);

-- Insert with boolean literals
INSERT INTO feature_flags (flag_name, is_enabled, is_public)
VALUES ('dark_mode', TRUE, TRUE);

-- Query with boolean predicates
SELECT flag_name FROM feature_flags WHERE is_enabled = TRUE;
SELECT flag_name FROM feature_flags WHERE NOT is_enabled;
SELECT flag_name FROM feature_flags WHERE is_enabled IS NOT NULL;

-- PL/SQL: boolean SQL column interop
DECLARE
    v_flag_on  BOOLEAN;
BEGIN
    SELECT is_enabled INTO v_flag_on FROM feature_flags WHERE flag_name = 'dark_mode';
    IF v_flag_on THEN
        DBMS_OUTPUT.PUT_LINE('Feature is on');
    END IF;
END;
/
```

---

## IF [NOT] EXISTS for DDL

Oracle 23ai adds `IF NOT EXISTS` to `CREATE` statements and `IF EXISTS` to `DROP` statements. This makes schema migration scripts idempotent: no more exception-handler wrappers for "ORA-00955: name is already used by an existing object".

```sql
-- Create only if the table does not already exist
CREATE TABLE IF NOT EXISTS audit_log (
    log_id     NUMBER         GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    log_msg    VARCHAR2(4000) NOT NULL,
    created_at TIMESTAMP      DEFAULT SYSTIMESTAMP NOT NULL
);

-- Drop only if the object exists
DROP SEQUENCE  IF EXISTS seq_customer_id;
DROP INDEX     IF EXISTS idx_orders_customer;
DROP TABLE     IF EXISTS temp_migration;
DROP VIEW      IF EXISTS v_active_customers;
DROP PROCEDURE IF EXISTS proc_old_cleanup;

-- Safe schema evolution
ALTER TABLE orders ADD    IF NOT EXISTS archived_at TIMESTAMP;
ALTER TABLE orders MODIFY IF EXISTS     legacy_flag  VARCHAR2(1) NULL;
```

---

## SQL Domains

SQL Domains are reusable schema-level objects that encapsulate data type, constraints, default values, and optional display expressions. Define a domain once and apply it to any column across multiple tables for consistent validation and semantics.

```sql
-- Create domains
CREATE DOMAIN email_address AS VARCHAR2(320)
    CONSTRAINT dom_email_fmt
        CHECK (REGEXP_LIKE(VALUE, '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z]{2,}$', 'i'))
    DISPLAY UPPER(VALUE);

CREATE DOMAIN us_phone AS VARCHAR2(20)
    CONSTRAINT dom_phone_fmt
        CHECK (REGEXP_LIKE(VALUE, '^\+?1?\s*\(?\d{3}\)?[\s.-]?\d{3}[\s.-]?\d{4}$'));

CREATE DOMAIN positive_amount AS NUMBER(18,2)
    CONSTRAINT dom_positive CHECK (VALUE > 0)
    DEFAULT 0.00;

CREATE DOMAIN order_status AS VARCHAR2(20)
    CONSTRAINT dom_order_status CHECK (VALUE IN ('PENDING','PROCESSING','SHIPPED','DELIVERED','CANCELLED'))
    DEFAULT 'PENDING';

-- Use domains in table definitions
CREATE TABLE IF NOT EXISTS contacts (
    contact_id NUMBER        GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email      email_address NOT NULL,
    phone      us_phone,
    balance    positive_amount NOT NULL
);

CREATE TABLE IF NOT EXISTS orders_v2 (
    order_id   NUMBER        GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    status     order_status  NOT NULL,
    total      positive_amount NOT NULL
);

-- Inspect domain metadata
SELECT domain_name, data_type, data_length, nullable
FROM user_domains
ORDER BY domain_name;
```

---

## Table Value Constructors

Oracle 23ai supports the SQL standard `FROM (VALUES ...)` syntax for building inline tables on-the-fly, without a staging table, `DUAL` chains, or `UNION ALL` stacks. This is particularly useful in migration scripts and unit tests.

```sql
-- Inline value table
SELECT *
FROM (
    VALUES
        (1, 'Alice',  'Engineering'),
        (2, 'Bob',    'Marketing'),
        (3, 'Carol',  'Finance')
) AS team_members (emp_id, full_name, department);

-- Join a value constructor against a real table
SELECT e.employee_id, e.last_name, r.region
FROM employees e
JOIN (
    VALUES
        ('IT',      'APAC'),
        ('HR',      'EMEA'),
        ('Finance', 'AMER')
) AS dept_regions (department, region)
  ON e.department_name = dept_regions.department;

-- Use in a WITH clause for seed data in tests
WITH test_prices (product_id, price) AS (
    VALUES (100, 29.99), (101, 49.99), (102, 9.99)
)
SELECT p.product_name, tp.price
FROM products p
JOIN test_prices tp ON p.product_id = tp.product_id;
```

---

## Direct Joins in UPDATE and DELETE

Oracle 23ai adds a `FROM` join clause to `UPDATE` and `DELETE`, eliminating the need for correlated subqueries or `MERGE` statements when modifying rows based on another table's values.

```sql
-- UPDATE with a direct join: give Engineering a 10% raise
UPDATE employees e
SET e.salary = e.salary * 1.10
FROM departments d
WHERE e.department_id = d.department_id
  AND d.department_name = 'Engineering';

-- DELETE with a direct join: remove pending orders from closed accounts
DELETE orders o
FROM customers c
WHERE o.customer_id = c.customer_id
  AND c.account_status = 'CLOSED'
  AND o.status         = 'PENDING';

-- Multi-table join in UPDATE
UPDATE order_items oi
SET oi.discounted_price = oi.unit_price * (1 - pc.discount_pct)
FROM orders o
JOIN promo_codes pc ON o.promo_code = pc.code
WHERE oi.order_id   = o.order_id
  AND pc.expiry_date > SYSDATE;
```

---

## JSON Relational Duality Views

JSON Relational Duality Views let applications read and write JSON documents that map transparently over normalized relational tables. The database maintains full ACID semantics on the underlying rows while exposing a clean JSON API. Applications never need to hand-write SQL; they operate entirely on JSON documents.

```sql
-- Underlying relational tables
CREATE TABLE IF NOT EXISTS depts (
    dept_id   NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name      VARCHAR2(100) NOT NULL
);

CREATE TABLE IF NOT EXISTS emps (
    emp_id     NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    dept_id    NUMBER        REFERENCES depts(dept_id),
    first_name VARCHAR2(100) NOT NULL,
    last_name  VARCHAR2(100) NOT NULL,
    salary     NUMBER(10,2)  NOT NULL
);

-- JSON Relational Duality View over both tables
CREATE OR REPLACE JSON RELATIONAL DUALITY VIEW dept_emp_dv AS
    SELECT JSON {
        '_id'  : d.dept_id,
        'name' : d.name,
        'employees' : [
            SELECT JSON {
                'empId'     : e.emp_id,
                'firstName' : e.first_name,
                'lastName'  : e.last_name,
                'salary'    : e.salary
            }
            FROM emps e WITH INSERT UPDATE DELETE
            WHERE e.dept_id = d.dept_id
        ]
    }
    FROM depts d WITH INSERT UPDATE DELETE;

-- Read: returns each department as a JSON document
SELECT * FROM dept_emp_dv WHERE JSON_VALUE(data, '$._id') = 1;

-- Insert: one JSON document writes to both depts and emps tables
INSERT INTO dept_emp_dv VALUES (JSON {
    'name' : 'Data Science',
    'employees' : [
        { 'firstName': 'Ana',  'lastName': 'Ruiz',  'salary': 95000 },
        { 'firstName': 'Liam', 'lastName': 'Patel', 'salary': 88000 }
    ]
});
COMMIT;

-- Update: modify the department name via JSON transform
UPDATE dept_emp_dv dv
SET dv.data = JSON_TRANSFORM(dv.data, SET '$.name' = 'Advanced Data Science')
WHERE JSON_VALUE(dv.data, '$._id' RETURNING NUMBER) = 1;
COMMIT;

-- Delete: removes relational rows in depts (and cascades to emps if FK ON DELETE CASCADE)
DELETE FROM dept_emp_dv
WHERE JSON_VALUE(data, '$._id' RETURNING NUMBER) = 1;
```

---

## SQL/PGQ — Property Graph Queries

Oracle 23ai introduces the SQL/PGQ standard for graph queries directly inside SQL. Define a property graph over existing tables and query vertices and edges using `GRAPH_TABLE` — no separate graph database required.

```sql
-- Create the property graph
CREATE PROPERTY GRAPH IF NOT EXISTS hr_graph
    VERTEX TABLES (
        employees
            KEY (employee_id)
            LABEL Employee
            PROPERTIES (first_name, last_name, salary, department_id),
        departments
            KEY (department_id)
            LABEL Department
            PROPERTIES (department_name)
    )
    EDGE TABLES (
        employees AS manages
            KEY (employee_id)
            SOURCE KEY      (manager_id)   REFERENCES employees (employee_id)
            DESTINATION KEY (employee_id)  REFERENCES employees (employee_id)
            LABEL REPORTS_TO,
        employees AS belongs_to
            KEY (employee_id)
            SOURCE KEY      (employee_id)  REFERENCES employees (employee_id)
            DESTINATION KEY (department_id) REFERENCES departments (department_id)
            LABEL WORKS_IN
    );

-- Find direct reports of employee 100
SELECT g.first_name, g.last_name
FROM GRAPH_TABLE (
    hr_graph
    MATCH (mgr IS Employee) -[IS REPORTS_TO]-> (emp IS Employee)
    WHERE mgr.employee_id = 100
    COLUMNS (emp.first_name, emp.last_name)
) g;

-- Find all managers in a chain up to 3 hops deep
SELECT g.src_name, g.depth, g.dst_name
FROM GRAPH_TABLE (
    hr_graph
    MATCH (a IS Employee) -[IS REPORTS_TO]->{1,3} (b IS Employee)
    COLUMNS (
        a.last_name                AS src_name,
        GRAPH_TABLE_ROW_DEPTH      AS depth,
        b.last_name                AS dst_name
    )
) g;
```

---

## Schema-Level Privilege Grants

Oracle 23ai adds schema-level privilege grants so you can grant a privilege on all current and future objects in a schema with a single statement, instead of per-object grants.

```sql
-- Grant SELECT on every table and view in the HR schema (current and future)
GRANT SELECT ANY TABLE ON SCHEMA hr TO reporting_role;

-- Grant EXECUTE on every procedure and function in app_owner
GRANT EXECUTE ANY PROCEDURE ON SCHEMA app_owner TO api_service_role;

-- Grant INSERT, UPDATE, DELETE on all tables in the schema
GRANT INSERT ANY TABLE, UPDATE ANY TABLE, DELETE ANY TABLE ON SCHEMA hr TO data_entry_role;

-- Revoke schema-level privilege
REVOKE SELECT ANY TABLE ON SCHEMA hr FROM reporting_role;
```

---

## DB_DEVELOPER_ROLE

Oracle 23ai ships a built-in `DB_DEVELOPER_ROLE` that bundles development privileges (CREATE TABLE, PROCEDURE, VIEW, TYPE, SEQUENCE, TRIGGER, etc.) without granting DBA. Use it for application developers instead of the overly permissive CONNECT + RESOURCE combination.

```sql
-- Create a new developer user and grant the role
CREATE USER dev_user IDENTIFIED BY "StrongPass#1"
    DEFAULT TABLESPACE users
    QUOTA UNLIMITED ON users;

GRANT DB_DEVELOPER_ROLE TO dev_user;

-- Inspect what the role includes
SELECT privilege FROM role_sys_privs WHERE role = 'DB_DEVELOPER_ROLE';
-- Includes: CREATE TABLE, CREATE VIEW, CREATE PROCEDURE, CREATE TRIGGER,
--           CREATE SEQUENCE, CREATE TYPE, CREATE SYNONYM, CREATE JOB, etc.
```

---

## Version Compatibility Quick Reference

When writing Oracle 23ai features, always confirm the minimum version required:

| Feature | Minimum Version |
|---|---|
| BOOLEAN SQL column, IF [NOT] EXISTS DDL | 23ai / 23c Free |
| SQL Domains, Table Value Constructors | 23ai / 23c Free |
| Direct-join UPDATE / DELETE | 23ai / 23c Free |
| JSON Relational Duality Views, SQL/PGQ | 23ai / 23c Free |
| VECTOR type, DBMS_VECTOR | 23ai / 23c Free |
| DB_DEVELOPER_ROLE, schema-level grants | 23ai / 23c Free |
| SQL Assertions (cross-table CHECK) | 23ai 23.26.1+ |
| JSON dot notation, JSON_TABLE | 12c Release 2+ |
| FETCH FIRST n ROWS ONLY | 12c+ |
| Analytic functions, WITH clause | 9i+ |

```sql
-- Check current database version
SELECT banner, banner_full FROM v$version;

-- Confirm BOOLEAN column support
SELECT data_type
FROM user_tab_columns
WHERE table_name = 'FEATURE_FLAGS'
  AND column_name = 'IS_ENABLED';
-- Returns 'BOOLEAN' on 23ai, error on earlier versions
```

---

## Assertions — Guaranteeing At-Least-One Relationships

A "one-to-at-least-one" relationship requires every parent row to have **one or more** child rows, not zero (every account needs a payment method, every team needs a player, every order needs a line item). A plain foreign key only constrains child → parent; it never guarantees the parent side has any children. Oracle 23ai (23.26.1+) closes this gap with **assertions** — cross-table `CHECK`-style constraints. On earlier releases, use the nominated-child foreign key technique instead.

Always make an assertion (or any constraint used for this pattern) `DEFERRABLE INITIALLY DEFERRED`. Without it, the parent row is checked the instant it's inserted — before its first child can exist — so the insert always fails. Deferring the check to `COMMIT` lets you insert the parent and its first child in the same transaction.

```sql
-- Guarantee every team has at least one player (23.26.1+)
CREATE ASSERTION team_requires_player
CHECK (
    ALL ( SELECT team_id FROM teams ) t
    SATISFY (
        EXISTS (
            SELECT 1 FROM players p WHERE p.team_id = t.team_id
        )
    )
)
DEFERRABLE INITIALLY DEFERRED;

-- Fails immediately if checked eagerly, or at COMMIT if deferred and no player was added:
INSERT INTO teams (team_id, team_name) VALUES (1, 'No players');
COMMIT;
-- ORA-02091: transaction rolled back
-- ORA-08601: SQL assertion (SCHEMA.TEAM_REQUIRES_PLAYER) violated

-- Correct pattern: insert parent and first child in the same transaction
INSERT INTO teams   (team_id, team_name) VALUES (1, 'One player');
INSERT INTO players (player_id, player_name, team_id) VALUES (1, 'First player', 1);
COMMIT;  -- succeeds

-- Conditional enforcement: only require a player for active teams
CREATE ASSERTION active_team_requires_player
CHECK (
    ALL ( SELECT team_id FROM teams WHERE is_active ) t
    SATISFY (
        EXISTS ( SELECT 1 FROM players p WHERE p.team_id = t.team_id )
    )
)
DEFERRABLE INITIALLY DEFERRED;
```

### Fallback for pre-23.26.1: nominated-child foreign key

Add a "primary child" column on the parent (e.g. `captain_player_id`) with a `NOT NULL` foreign key back to the child table. This also documents a genuinely meaningful "primary child" concept when one exists, regardless of version.

**Critical gotcha:** a single-column FK only proves the value references *some* child row — not one that belongs to *this* parent. Make the FK composite, including the parent's key, backed by a unique constraint on the child that also includes the parent's key:

```sql
-- Unique constraint on the child scoped to its parent
ALTER TABLE players
  ADD CONSTRAINT players_team_player_uk UNIQUE (team_id, player_id);

-- Composite FK: captain must be a player on THIS team
ALTER TABLE teams
  ADD CONSTRAINT team_captain_fk
    FOREIGN KEY (team_id, captain_player_id)
    REFERENCES players (team_id, player_id)
    DEFERRABLE INITIALLY DEFERRED
    NOVALIDATE;

-- Wrong-team captain now correctly fails at commit:
-- ORA-02291: integrity constraint (SCHEMA.TEAM_CAPTAIN_FK) violated - parent key not found
```

`NOVALIDATE` enforces the constraint for new/changed rows without immediately validating pre-existing data — useful when retrofitting onto a populated table; run a separate validation pass once the data is clean.

**Maintenance cost:** because the parent's column points at a specific child row, you can't delete that child directly — reassign the parent to a different/new child first, then delete the old one.

| Situation | Use |
|---|---|
| 23.26.1+, no natural primary child | Assertion |
| 23.26.1+, natural primary child exists (captain, default payment method) | Either — FK documents the concept explicitly; assertion is lower-maintenance |
| 23.26.1+, rule applies only to a subset of parents | Assertion with filtered `ALL` clause |
| Pre-23.26.1 | Composite foreign key (see gotcha above) |

**Error quick reference**

| Error | Cause | Fix |
|---|---|---|
| `ORA-08601` | Assertion's `SATISFY` clause evaluated false | Insert parent + required child in the same transaction; confirm `DEFERRABLE INITIALLY DEFERRED` |
| `ORA-02091` | Deferred constraint failed at commit | Add the missing child row before committing |
| `ORA-02291` | Primary-child FK value doesn't belong to this specific parent | Make the FK composite against a unique constraint that includes the parent's key |

> Adapted from Chris Saxon (Oracle Developer Advocate), *"Guarantee at least one in one-to-many relationships in Oracle AI Database,"* Oracle "All Things SQL" blog, June 25, 2026: https://blogs.oracle.com/sql/guarantee-at-least-one-in-one-to-many-relationships-in-oracle-ai-database
