# Oracle SQL — AI Skill Definition

## Purpose
This skill teaches AI assistants to write correct, idiomatic, and performant Oracle SQL. Always apply these patterns when working in Oracle Database environments.

---

## Core Syntax Rules

- Oracle uses `VARCHAR2` (not `VARCHAR`) for variable-length strings
- Use `NUMBER(p,s)` for numeric data; prefer `NUMBER` without precision for flexible numerics
- Strings use single quotes only: `'value'` (never double quotes for literals)
- Double quotes are for **identifiers** only: `"My Column"`
- Oracle NULL handling: `NVL(expr, default)`, `NVL2(expr, not_null_val, null_val)`, `NULLIF(a, b)`
- `SYSDATE` returns current date+time; `CURRENT_DATE` returns session timezone date
- `SYSTIMESTAMP` for timestamp with time zone
- Sequences: `seq_name.NEXTVAL`, `seq_name.CURRVAL`
- Use `ROWNUM` for row limiting in pre-12c; use `FETCH FIRST n ROWS ONLY` in 12c+

---

## SELECT Patterns

### Basic SELECT with Oracle conventions
```sql
SELECT
    e.employee_id,
    e.first_name || ' ' || e.last_name AS full_name,
    d.department_name,
    NVL(e.commission_pct, 0) AS commission
FROM
    employees e
    JOIN departments d ON e.department_id = d.department_id
WHERE
    e.hire_date >= DATE '2020-01-01'
ORDER BY
    e.hire_date DESC;
```

### Date literals — always use DATE keyword
```sql
-- Correct Oracle date literal
WHERE order_date = DATE '2024-01-15'

-- Date arithmetic
WHERE order_date BETWEEN DATE '2024-01-01' AND DATE '2024-12-31'

-- Add days to date
SELECT SYSDATE + 30 AS thirty_days_later FROM DUAL;

-- Truncate to day
SELECT TRUNC(SYSDATE) AS today FROM DUAL;

-- Date formatting
SELECT TO_CHAR(SYSDATE, 'YYYY-MM-DD HH24:MI:SS') AS formatted_date FROM DUAL;

-- String to date
SELECT TO_DATE('2024-06-15', 'YYYY-MM-DD') FROM DUAL;
```

### Common Table Expressions (WITH clause)
```sql
WITH
    dept_salary AS (
        SELECT
            department_id,
            AVG(salary) AS avg_salary,
            MAX(salary) AS max_salary,
            COUNT(*) AS emp_count
        FROM employees
        GROUP BY department_id
    ),
    high_performers AS (
        SELECT e.employee_id, e.last_name, e.salary, e.department_id
        FROM employees e
        JOIN dept_salary ds ON e.department_id = ds.department_id
        WHERE e.salary > ds.avg_salary
    )
SELECT
    hp.last_name,
    hp.salary,
    ds.avg_salary,
    ds.dept_name
FROM high_performers hp
JOIN dept_salary ds ON hp.department_id = ds.department_id;
```

---

## Analytic (Window) Functions

```sql
-- ROW_NUMBER, RANK, DENSE_RANK
SELECT
    employee_id,
    last_name,
    department_id,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC) AS rn,
    RANK()       OVER (PARTITION BY department_id ORDER BY salary DESC) AS rnk,
    DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dense_rnk
FROM employees;

-- LAG and LEAD
SELECT
    order_id,
    order_date,
    total_amount,
    LAG(total_amount, 1, 0)  OVER (PARTITION BY customer_id ORDER BY order_date) AS prev_order_amount,
    LEAD(total_amount, 1, 0) OVER (PARTITION BY customer_id ORDER BY order_date) AS next_order_amount
FROM orders;

-- Running totals and moving averages
SELECT
    sale_date,
    daily_sales,
    SUM(daily_sales) OVER (ORDER BY sale_date ROWS UNBOUNDED PRECEDING) AS running_total,
    AVG(daily_sales) OVER (ORDER BY sale_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS moving_avg_7d
FROM daily_sales;

-- NTILE for bucketing
SELECT
    employee_id,
    salary,
    NTILE(4) OVER (ORDER BY salary) AS salary_quartile
FROM employees;
```

---

## Hierarchical Queries (CONNECT BY)

```sql
-- Organisation hierarchy
SELECT
    LEVEL AS depth,
    LPAD(' ', (LEVEL - 1) * 4) || last_name AS org_chart,
    employee_id,
    manager_id
FROM employees
START WITH manager_id IS NULL
CONNECT BY PRIOR employee_id = manager_id
ORDER SIBLINGS BY last_name;

-- SYS_CONNECT_BY_PATH for full path
SELECT
    employee_id,
    SYS_CONNECT_BY_PATH(last_name, ' > ') AS path
FROM employees
START WITH manager_id IS NULL
CONNECT BY PRIOR employee_id = manager_id;
```

---

## PIVOT and UNPIVOT

```sql
-- PIVOT: rows to columns
SELECT *
FROM (
    SELECT department_id, job_id, salary
    FROM employees
)
PIVOT (
    SUM(salary)
    FOR job_id IN ('IT_PROG' AS it_prog, 'SA_REP' AS sa_rep, 'AD_VP' AS ad_vp)
);

-- UNPIVOT: columns to rows
SELECT product_id, quarter, sales
FROM quarterly_sales
UNPIVOT (
    sales
    FOR quarter IN (q1_sales AS 'Q1', q2_sales AS 'Q2', q3_sales AS 'Q3', q4_sales AS 'Q4')
);
```

---

## MERGE (Upsert)

```sql
MERGE INTO target_table t
USING source_table s
ON (t.id = s.id)
WHEN MATCHED THEN
    UPDATE SET
        t.name    = s.name,
        t.updated = SYSDATE
    WHERE t.status != 'LOCKED'
WHEN NOT MATCHED THEN
    INSERT (t.id, t.name, t.created)
    VALUES (s.id, s.name, SYSDATE);
```

---

## JSON in Oracle (12c+)

```sql
-- Store JSON
CREATE TABLE orders_json (
    id     NUMBER PRIMARY KEY,
    data   CLOB CHECK (data IS JSON)
);

-- Query JSON with dot notation (21c+)
SELECT o.data.customer.name, o.data.total
FROM orders_json o;

-- JSON_VALUE and JSON_QUERY
SELECT
    JSON_VALUE(data, '$.customer.name')           AS customer_name,
    JSON_QUERY(data, '$.items' WITH WRAPPER)      AS items_array,
    JSON_VALUE(data, '$.total' RETURNING NUMBER)  AS total
FROM orders_json;

-- JSON_TABLE
SELECT jt.*
FROM orders_json o,
    JSON_TABLE(o.data, '$'
        COLUMNS (
            order_id   NUMBER         PATH '$.id',
            cust_name  VARCHAR2(100)  PATH '$.customer.name',
            total      NUMBER         PATH '$.total'
        )
    ) jt;
```

---

## Flashback Queries

```sql
-- Query table as it was 1 hour ago
SELECT * FROM employees AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '1' HOUR);

-- Query using SCN
SELECT * FROM employees AS OF SCN 1234567;

-- See versions of a row over time
SELECT
    VERSIONS_STARTTIME,
    VERSIONS_ENDTIME,
    VERSIONS_OPERATION,
    employee_id,
    salary
FROM employees
    VERSIONS BETWEEN TIMESTAMP SYSTIMESTAMP - INTERVAL '1' HOUR AND SYSTIMESTAMP
WHERE employee_id = 100;
```

---

## Performance Best Practices

- Always use bind variables for repeated queries: `:param_name` in SQL, `p_param` in PL/SQL
- Avoid `SELECT *` in production code — name columns explicitly
- Use EXISTS instead of IN with subqueries when the subquery returns many rows
- Prefer JOINs over correlated subqueries
- Use `TRUNC(date_col)` carefully — it prevents index use; consider function-based indexes
- When using `LIKE`, only leading wildcards prevent index use: `'%value'` is slow, `'value%'` can use index
- Add `/*+ PARALLEL(4) */` hint for large analytical queries on partitioned tables

---

## DDL Conventions

```sql
-- Table with constraints
CREATE TABLE customers (
    customer_id   NUMBER          GENERATED ALWAYS AS IDENTITY,
    email         VARCHAR2(255)   NOT NULL,
    first_name    VARCHAR2(100)   NOT NULL,
    last_name     VARCHAR2(100)   NOT NULL,
    status        VARCHAR2(20)    DEFAULT 'ACTIVE' NOT NULL,
    created_at    TIMESTAMP       DEFAULT SYSTIMESTAMP NOT NULL,
    updated_at    TIMESTAMP,
    CONSTRAINT pk_customers      PRIMARY KEY (customer_id),
    CONSTRAINT uk_customers_email UNIQUE (email),
    CONSTRAINT ck_customers_status CHECK (status IN ('ACTIVE','INACTIVE','SUSPENDED'))
);

-- Index creation
CREATE INDEX idx_customers_email    ON customers (email);
CREATE INDEX idx_customers_status   ON customers (status) WHERE status = 'ACTIVE';  -- Partial
CREATE INDEX idx_customers_name_fn  ON customers (UPPER(last_name));                -- Function-based

-- Sequence (pre-12c identity column alternative)
CREATE SEQUENCE seq_customers START WITH 1 INCREMENT BY 1 NOCACHE NOCYCLE;
```
