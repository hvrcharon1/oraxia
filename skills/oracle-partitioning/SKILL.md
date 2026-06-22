# Oracle Partitioning — AI Skill Definition

## Purpose
This skill teaches AI assistants to design, implement, and manage Oracle table partitioning correctly. Partitioning splits a large table into independently managed segments (partitions) that share the same logical schema. The key benefits are faster queries through partition pruning (the optimizer skips entire partitions), instant data lifecycle management (DROP PARTITION instead of mass DELETE), and improved DML parallelism. Always consider partitioning for tables that will exceed tens of millions of rows or that require time-based archival.

---

## When to Partition

Partitioning is a good fit when the table:
- Has a clear filter column in most queries (date, region, status) that maps to a partition key
- Will hold tens of millions of rows or more
- Requires periodic purging of old data — DROP PARTITION is instant and generates no undo
- Benefits from parallel DML, bulk loads via partition exchange, or partition-wise joins
- Has mixed hot/cold access patterns (recent data queried constantly, old data rarely accessed)

---

## Range Partitioning

Range partitioning assigns rows to partitions based on value ranges of the partition key. The most common use case is time-based partitioning on a date or timestamp column.

```sql
-- Orders partitioned by year of order_date
CREATE TABLE orders (
    order_id      NUMBER        GENERATED ALWAYS AS IDENTITY,
    customer_id   NUMBER        NOT NULL,
    order_date    DATE          NOT NULL,
    total_amount  NUMBER(12,2)  NOT NULL,
    status        VARCHAR2(20)  DEFAULT 'PENDING' NOT NULL,
    CONSTRAINT pk_orders PRIMARY KEY (order_id, order_date)  -- PK must include partition key
)
PARTITION BY RANGE (order_date) (
    PARTITION p_2022   VALUES LESS THAN (DATE '2023-01-01'),
    PARTITION p_2023   VALUES LESS THAN (DATE '2024-01-01'),
    PARTITION p_2024   VALUES LESS THAN (DATE '2025-01-01'),
    PARTITION p_future VALUES LESS THAN (MAXVALUE)   -- catch-all for any future date
);
```

---

## Interval Partitioning

Interval partitioning extends range partitioning by automatically creating new partitions as data arrives. It eliminates the ORA-14400 error (no matching partition) and the manual task of pre-creating future partitions.

```sql
-- Monthly interval partitioning on a timestamp column
CREATE TABLE sales_events (
    event_id    NUMBER       GENERATED ALWAYS AS IDENTITY,
    event_ts    TIMESTAMP    NOT NULL,
    product_id  NUMBER       NOT NULL,
    amount      NUMBER(10,2) NOT NULL,
    CONSTRAINT pk_sales_events PRIMARY KEY (event_id, event_ts)
)
PARTITION BY RANGE (event_ts)
INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'))   -- new partition every calendar month
(
    PARTITION p_seed VALUES LESS THAN (TIMESTAMP '2024-01-01 00:00:00')
);

-- Weekly interval example
CREATE TABLE access_log (
    log_id    NUMBER    GENERATED ALWAYS AS IDENTITY,
    log_ts    DATE      NOT NULL,
    user_id   NUMBER    NOT NULL,
    action    VARCHAR2(100)
)
PARTITION BY RANGE (log_ts)
INTERVAL (NUMTODSINTERVAL(7, 'DAY'))     -- new partition every 7 days
(
    PARTITION p_initial VALUES LESS THAN (DATE '2024-01-01')
);

-- System-generated partition names (SYS_Pnnn) can be renamed
ALTER TABLE sales_events RENAME PARTITION SYS_P12345 TO p_2024_07;

-- Inspect existing partitions
SELECT partition_name, high_value, num_rows
FROM user_tab_partitions
WHERE table_name = 'SALES_EVENTS'
ORDER BY partition_position;
```

---

## List Partitioning

List partitioning assigns rows to partitions based on discrete values (e.g., country, region, status). Use it when the partition key is categorical rather than ordinal.

```sql
CREATE TABLE global_customers (
    customer_id  NUMBER        GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    region       VARCHAR2(10)  NOT NULL,
    company_name VARCHAR2(200) NOT NULL,
    country_code CHAR(2)       NOT NULL
)
PARTITION BY LIST (region) (
    PARTITION p_amer    VALUES ('AMER', 'LATAM'),
    PARTITION p_europe  VALUES ('EMEA'),
    PARTITION p_apac    VALUES ('APAC'),
    PARTITION p_default VALUES (DEFAULT)   -- catch-all for any unmatched value
);

-- Status-based list partitioning for a work queue
CREATE TABLE task_queue (
    task_id    NUMBER       GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    status     VARCHAR2(20) NOT NULL,
    payload    CLOB,
    created_at TIMESTAMP    DEFAULT SYSTIMESTAMP
)
PARTITION BY LIST (status) (
    PARTITION p_pending    VALUES ('PENDING'),
    PARTITION p_processing VALUES ('PROCESSING'),
    PARTITION p_done       VALUES ('COMPLETED', 'CANCELLED', 'FAILED')
);
```

---

## Hash Partitioning

Hash partitioning evenly distributes rows across a fixed number of partitions by hashing the partition key. Use it when no natural range or list exists and you want even I/O distribution across disks or tablespaces.

```sql
-- Hash-partition a user event table by user_id to distribute write I/O evenly
CREATE TABLE user_events (
    event_id   NUMBER         GENERATED ALWAYS AS IDENTITY,
    user_id    NUMBER         NOT NULL,
    event_type VARCHAR2(50)   NOT NULL,
    payload    CLOB,
    created_at TIMESTAMP      DEFAULT SYSTIMESTAMP NOT NULL
)
PARTITION BY HASH (user_id)
PARTITIONS 16             -- should be a power of 2 for even distribution
STORE IN (ts_data_01, ts_data_02, ts_data_03, ts_data_04);  -- spread across tablespaces
```

---

## Composite Partitioning

Composite partitioning uses two levels: the top level partitions by range or list, and the second level sub-partitions by hash or list. This is ideal for tables that need both time-based archival and I/O load balancing within each time period.

```sql
-- Range-Hash: partition by quarter, sub-partition by hash of customer_id
CREATE TABLE transactions (
    txn_id      NUMBER       GENERATED ALWAYS AS IDENTITY,
    txn_date    DATE         NOT NULL,
    customer_id NUMBER       NOT NULL,
    amount      NUMBER(14,2) NOT NULL
)
PARTITION BY RANGE (txn_date)
SUBPARTITION BY HASH (customer_id)
SUBPARTITIONS 8
(
    PARTITION p_q1_2024 VALUES LESS THAN (DATE '2024-04-01'),
    PARTITION p_q2_2024 VALUES LESS THAN (DATE '2024-07-01'),
    PARTITION p_q3_2024 VALUES LESS THAN (DATE '2024-10-01'),
    PARTITION p_q4_2024 VALUES LESS THAN (DATE '2025-01-01'),
    PARTITION p_future  VALUES LESS THAN (MAXVALUE)
);

-- Range-List with a subpartition template (each year-partition gets the same region sub-partitions)
CREATE TABLE global_orders (
    order_id    NUMBER       GENERATED ALWAYS AS IDENTITY,
    order_year  NUMBER(4)    NOT NULL,
    region      VARCHAR2(10) NOT NULL,
    amount      NUMBER(12,2)
)
PARTITION BY RANGE (order_year)
SUBPARTITION BY LIST (region)
SUBPARTITION TEMPLATE (
    SUBPARTITION sp_amer    VALUES ('AMER'),
    SUBPARTITION sp_emea    VALUES ('EMEA'),
    SUBPARTITION sp_apac    VALUES ('APAC'),
    SUBPARTITION sp_default VALUES (DEFAULT)
)
(
    PARTITION p_2023 VALUES LESS THAN (2024),
    PARTITION p_2024 VALUES LESS THAN (2025),
    PARTITION p_2025 VALUES LESS THAN (2026)
);
```

---

## Reference Partitioning

Reference partitioning (available from Oracle 11g) lets a child table inherit its parent's partitioning scheme through a foreign key, without duplicating the partition key column in the child. Oracle ensures that child rows are always co-located with their parent row in the same partition.

```sql
-- Parent table: range-partitioned on order_date
CREATE TABLE ord_hdr (
    order_id   NUMBER PRIMARY KEY,
    order_date DATE   NOT NULL,
    customer_id NUMBER NOT NULL
)
PARTITION BY RANGE (order_date) (
    PARTITION p_2023   VALUES LESS THAN (DATE '2024-01-01'),
    PARTITION p_2024   VALUES LESS THAN (DATE '2025-01-01'),
    PARTITION p_future VALUES LESS THAN (MAXVALUE)
);

-- Child table: inherits partitioning via FK — no need for order_date in ord_lines
CREATE TABLE ord_lines (
    line_id    NUMBER       PRIMARY KEY,
    order_id   NUMBER       NOT NULL,
    product_id NUMBER       NOT NULL,
    quantity   NUMBER       NOT NULL,
    unit_price NUMBER(10,2) NOT NULL,
    CONSTRAINT fk_ord_lines_order
        FOREIGN KEY (order_id) REFERENCES ord_hdr (order_id)
)
PARTITION BY REFERENCE (fk_ord_lines_order);
-- ord_lines is automatically co-partitioned with ord_hdr
-- Partition-wise joins between ord_hdr and ord_lines are now possible
```

---

## Local vs Global Indexes on Partitioned Tables

A **local index** is partitioned identically to the table — one index segment per partition, maintained automatically during partition DDL. A **global index** spans all partitions and is never automatically maintained.

```sql
-- Local index — best for queries that filter on the partition key
CREATE INDEX idx_orders_customer_local
    ON orders (customer_id) LOCAL;

-- Local prefixed index (index starts with partition key — optimizer can prune index partitions too)
CREATE INDEX idx_orders_date_status
    ON orders (order_date, status) LOCAL;

-- Global non-partitioned index — best for queries NOT filtering on partition key
-- but must be rebuilt after DROP/SPLIT partition unless UPDATE GLOBAL INDEXES is used
CREATE INDEX idx_orders_customer_global
    ON orders (customer_id);   -- GLOBAL is the default for non-partitioned index

-- Global partitioned index — range-partitioned on a different column
CREATE INDEX idx_orders_status_global_part
    ON orders (status)
    GLOBAL PARTITION BY RANGE (status) (
        PARTITION pi_a VALUES LESS THAN ('M'),
        PARTITION pi_z VALUES LESS THAN (MAXVALUE)
    );
```

---

## Partition Management DDL

Oracle provides rich DDL for partition lifecycle management. These operations are typically instantaneous or fast because they manipulate metadata, not rows.

```sql
-- Add a new partition (range tables without INTERVAL)
ALTER TABLE orders
    ADD PARTITION p_2025 VALUES LESS THAN (DATE '2026-01-01');

-- Drop a partition — instant, no undo, data is permanently removed
ALTER TABLE orders DROP PARTITION p_2022 UPDATE GLOBAL INDEXES;
--                                        ^^^^^^^^^^^^^^^^^^^^
-- UPDATE GLOBAL INDEXES keeps global indexes valid without a full rebuild.
-- Omitting this marks global indexes UNUSABLE.

-- Truncate a partition — removes data, keeps the partition shell
ALTER TABLE orders TRUNCATE PARTITION p_2022;

-- Split one partition into two (e.g., split the catch-all future partition)
ALTER TABLE orders
    SPLIT PARTITION p_future AT (DATE '2026-01-01')
    INTO (
        PARTITION p_2025,
        PARTITION p_future
    )
    UPDATE GLOBAL INDEXES;

-- Merge two adjacent partitions into one
ALTER TABLE orders
    MERGE PARTITIONS p_2022, p_2023
    INTO PARTITION p_2022_2023
    UPDATE GLOBAL INDEXES;

-- Move a partition to a different tablespace (online in 12c+)
ALTER TABLE orders
    MOVE PARTITION p_2022
    TABLESPACE ts_archive
    ONLINE;
```

---

## Partition Exchange — Fast Bulk Load

Partition exchange swaps a non-partitioned staging table atomically into a table partition in O(1) time (metadata only). This is the recommended pattern for high-speed bulk data loads.

```sql
-- Step 1: Create a staging table with the same structure (no partitioning)
CREATE TABLE orders_stage AS SELECT * FROM orders WHERE 1 = 0;

-- Step 2: Bulk load into staging (fast — no index maintenance during insert)
INSERT /*+ APPEND */ INTO orders_stage
    SELECT * FROM external_source_table;
COMMIT;

-- Step 3: Build indexes on the staging table to match the partitioned table
CREATE INDEX idx_orders_stage_customer ON orders_stage (customer_id);

-- Step 4: Exchange the staging table into the target partition
ALTER TABLE orders
    EXCHANGE PARTITION p_2024
    WITH TABLE orders_stage
    INCLUDING INDEXES
    WITHOUT VALIDATION;    -- skip row-by-row check; add WITH VALIDATION if data integrity is uncertain

-- After exchange: orders_stage now contains the old partition data (if any)
-- and the partition p_2024 now holds the staged data
```

---

## Partition Pruning

Partition pruning is the optimizer skipping partitions that cannot satisfy the WHERE clause. Always filter on the partition key to enable pruning; queries without a partition key filter scan all partitions.

```sql
-- This query causes partition pruning: optimizer reads only p_2024
SELECT order_id, total_amount
FROM orders
WHERE order_date BETWEEN DATE '2024-01-01' AND DATE '2024-12-31';

-- Verify pruning by inspecting the execution plan
EXPLAIN PLAN FOR
    SELECT * FROM orders WHERE order_date >= DATE '2024-06-01';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(FORMAT => 'BASIC +PARTITION'));
-- Look for Pstart and Pstop columns in the plan output
-- Pstart=3, Pstop=3 means only partition #3 is accessed

-- Dynamic pruning works with bind variables too
SELECT * FROM orders
WHERE order_date >= :start_dt AND order_date < :end_dt;
-- Oracle prunes partitions at execution time based on bind values
```

---

## Useful Metadata Queries

```sql
-- List all partitions for a table with row counts and high-water marks
SELECT
    partition_name,
    partition_position,
    num_rows,
    blocks,
    high_value
FROM user_tab_partitions
WHERE table_name = 'ORDERS'
ORDER BY partition_position;

-- List subpartitions
SELECT table_name, partition_name, subpartition_name,
       subpartition_position, num_rows
FROM user_tab_subpartitions
WHERE table_name = 'TRANSACTIONS'
ORDER BY partition_name, subpartition_position;

-- Check whether indexes are local or global
SELECT index_name, partitioned, locality, alignment
FROM user_part_indexes
WHERE table_name = 'ORDERS';

-- Check index partition status (UNUSABLE after DROP/SPLIT without UPDATE GLOBAL INDEXES)
SELECT index_name, partition_name, status
FROM user_ind_partitions
WHERE status != 'USABLE'
ORDER BY index_name, partition_name;

-- Identify the largest partitions by block count
SELECT table_name, partition_name, blocks, num_rows
FROM user_tab_partitions
ORDER BY blocks DESC NULLS LAST
FETCH FIRST 10 ROWS ONLY;
```

---

## Best Practices

- Always include the partition key in the primary key or a unique index on a range or list partitioned table — Oracle enforces this because partition-local uniqueness cannot be guaranteed otherwise
- Prefer `INTERVAL` partitioning over adding future `RANGE` partitions manually — eliminates ORA-14400 and removes the maintenance overhead of partition pre-creation scripts
- Always specify `UPDATE GLOBAL INDEXES` on `DROP PARTITION`, `SPLIT PARTITION`, and `MERGE PARTITIONS` to keep global indexes valid; omitting it marks them `UNUSABLE` and causes query errors until rebuilt
- Use `DROP PARTITION` and `TRUNCATE PARTITION` instead of `DELETE FROM ... WHERE` for data archival and purging — DDL is instantaneous and generates no undo or redo at the row level
- Create `LOCAL` indexes for all queries that filter on the partition key; use `GLOBAL` indexes only for queries that frequently access data across many partitions without a partition key filter
- Rebuild optimizer statistics after bulk partition exchanges: `DBMS_STATS.GATHER_TABLE_STATS('HR', 'ORDERS', granularity => 'PARTITION')`
- Use partition-wise joins by ensuring both sides of a join are partitioned on the join key — this allows Oracle to execute each partition pair independently in parallel, dramatically reducing join memory and CPU
