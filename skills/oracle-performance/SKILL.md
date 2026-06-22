# Oracle Performance Tuning — AI Skill Definition

## Purpose
This skill teaches AI assistants to identify and resolve Oracle Database performance issues using explain plans, indexes, hints, AWR, and optimizer statistics.

---

## Explain Plan Analysis

```sql
-- Generate explain plan
EXPLAIN PLAN FOR
    SELECT e.last_name, d.department_name
    FROM employees e JOIN departments d ON e.department_id = d.department_id
    WHERE e.salary > 10000;

-- View with DBMS_XPLAN (most informative)
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(FORMAT => 'ALL'));

-- View last executed plan from cursor cache
SELECT * FROM TABLE(
    DBMS_XPLAN.DISPLAY_CURSOR(
        sql_id     => 'abc123xyz',
        child_no   => 0,
        format     => 'ALLSTATS LAST'
    )
);

-- Find SQL ID for a query
SELECT sql_id, executions, elapsed_time/1000000 AS elapsed_sec, sql_text
FROM v$sql
WHERE sql_text LIKE '%employees%'
  AND parsing_schema_name = 'MY_SCHEMA'
ORDER BY elapsed_time DESC
FETCH FIRST 10 ROWS ONLY;
```

### Reading the Plan — Key Operations
| Operation | Meaning | Action |
|---|---|---|
| TABLE ACCESS FULL | Full table scan | Add index or add WHERE clause |
| INDEX RANGE SCAN | Using index range | Good |
| INDEX UNIQUE SCAN | Single row via unique index | Best |
| NESTED LOOPS | Good for small driving sets | OK |
| HASH JOIN | Good for large sets | OK |
| SORT MERGE JOIN | After sorting both sides | Investigate if slow |
| CARTESIAN JOIN | Missing join condition! | Fix immediately |
| MERGE JOIN CARTESIAN | Unintentional cross join | Fix join conditions |

---

## Index Strategies

```sql
-- B-Tree index (default, best for high cardinality)
CREATE INDEX idx_emp_email ON employees(email);

-- Composite index (put most selective column first; matches WHERE clause order)
CREATE INDEX idx_orders_cust_date ON orders(customer_id, order_date);

-- Function-based index (for UPPER(), LOWER(), or expressions in WHERE)
CREATE INDEX idx_emp_last_name_upper ON employees(UPPER(last_name));
-- Now this query uses the index:
SELECT * FROM employees WHERE UPPER(last_name) = 'SMITH';

-- Partial index (filtered, smaller and faster for common queries)
CREATE INDEX idx_orders_active ON orders(customer_id)
    WHERE status = 'ACTIVE';  -- Oracle: use invisible/partial with FBI

-- Invisible index (test without affecting optimizer)
CREATE INDEX idx_emp_dept_invisible ON employees(department_id) INVISIBLE;
-- Force it for testing:
ALTER SESSION SET OPTIMIZER_USE_INVISIBLE_INDEXES = TRUE;

-- Bitmap index (only for low cardinality, data warehouse, no OLTP)
CREATE BITMAP INDEX idx_emp_status_bmp ON employees(status);

-- Rebuild fragmented index
ALTER INDEX idx_emp_email REBUILD ONLINE;

-- Check index usage
SELECT index_name, num_rows, leaf_blocks, distinct_keys, clustering_factor
FROM user_indexes
WHERE table_name = 'EMPLOYEES'
ORDER BY clustering_factor;
```

---

## SQL Hints

```sql
-- Force full table scan
SELECT /*+ FULL(e) */ * FROM employees e WHERE salary > 5000;

-- Force index use
SELECT /*+ INDEX(e idx_emp_email) */ * FROM employees e WHERE email = 'test@example.com';

-- Force hash join
SELECT /*+ USE_HASH(e d) */ e.last_name, d.department_name
FROM employees e JOIN departments d ON e.department_id = d.department_id;

-- Parallel execution
SELECT /*+ PARALLEL(e, 8) */ COUNT(*) FROM large_table e;

-- Result cache (caches result across sessions)
SELECT /*+ RESULT_CACHE */ country_code, country_name FROM countries;

-- Append hint for fast INSERT (bypasses buffer cache)
INSERT /*+ APPEND */ INTO archive_orders SELECT * FROM orders WHERE order_date < DATE '2023-01-01';
COMMIT;  -- Must commit immediately after APPEND

-- Leading hint (controls join order)
SELECT /*+ LEADING(o c) USE_HASH(c) */
    o.order_id, c.customer_name
FROM orders o JOIN customers c ON o.customer_id = c.customer_id;
```

---

## Optimizer Statistics

```sql
-- Gather stats on table
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS(
        ownname          => 'MY_SCHEMA',
        tabname          => 'ORDERS',
        estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
        method_opt       => 'FOR ALL COLUMNS SIZE AUTO',
        cascade          => TRUE,  -- includes indexes
        degree           => 4
    );
END;

-- Gather schema stats
BEGIN
    DBMS_STATS.GATHER_SCHEMA_STATS(
        ownname          => 'MY_SCHEMA',
        estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
        cascade          => TRUE,
        degree           => 4
    );
END;

-- Lock stats to prevent auto update
BEGIN
    DBMS_STATS.LOCK_TABLE_STATS('MY_SCHEMA', 'REFERENCE_TABLE');
END;

-- Check stale stats
SELECT table_name, last_analyzed, num_rows, stale_stats
FROM dba_tab_statistics
WHERE owner = 'MY_SCHEMA'
ORDER BY last_analyzed NULLS FIRST;
```

---

## AWR & ASH Analysis

```sql
-- List recent AWR snapshots
SELECT snap_id, begin_interval_time, end_interval_time
FROM dba_hist_snapshot
WHERE begin_interval_time > SYSDATE - 1
ORDER BY snap_id;

-- Generate AWR report (HTML)
SELECT * FROM TABLE(
    DBMS_WORKLOAD_REPOSITORY.AWR_REPORT_HTML(
        l_dbid       => (SELECT dbid FROM v$database),
        l_inst_num   => 1,
        l_bid        => 100,   -- Begin snap ID
        l_eid        => 110    -- End snap ID
    )
);

-- Top SQL by elapsed time from AWR
SELECT
    sql_id,
    ROUND(elapsed_time_delta / 1e6 / executions_delta, 3) AS avg_elapsed_sec,
    executions_delta AS executions,
    ROUND(buffer_gets_delta / executions_delta) AS avg_buffer_gets,
    SUBSTR(sql_text, 1, 80) AS sql_preview
FROM dba_hist_sqlstat
JOIN dba_hist_sqltext USING (sql_id)
WHERE snap_id BETWEEN 100 AND 110
  AND executions_delta > 0
ORDER BY elapsed_time_delta DESC
FETCH FIRST 20 ROWS ONLY;

-- ASH: what was DB doing in last 30 minutes?
SELECT
    event,
    wait_class,
    COUNT(*) AS sample_count,
    ROUND(COUNT(*) * 100 / SUM(COUNT(*)) OVER (), 1) AS pct
FROM v$active_session_history
WHERE sample_time > SYSDATE - 30/1440
  AND session_type = 'FOREGROUND'
GROUP BY event, wait_class
ORDER BY sample_count DESC;
```

---

## Partitioning

```sql
-- Range partitioning by date (most common for time-series)
CREATE TABLE orders_partitioned (
    order_id     NUMBER,
    order_date   DATE NOT NULL,
    customer_id  NUMBER,
    total        NUMBER
)
PARTITION BY RANGE (order_date) INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'))
(
    PARTITION p_before_2024 VALUES LESS THAN (DATE '2024-01-01')
);

-- List partitioning
CREATE TABLE customers_by_region (
    customer_id  NUMBER,
    region       VARCHAR2(20),
    name         VARCHAR2(100)
)
PARTITION BY LIST (region)
(
    PARTITION p_north VALUES ('NORTH', 'NE'),
    PARTITION p_south VALUES ('SOUTH', 'SE'),
    PARTITION p_west  VALUES ('WEST'),
    PARTITION p_other VALUES (DEFAULT)
);

-- Partition pruning verification (look for PARTITION RANGE SINGLE in plan)
EXPLAIN PLAN FOR
    SELECT * FROM orders_partitioned WHERE order_date = DATE '2024-06-01';
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```
