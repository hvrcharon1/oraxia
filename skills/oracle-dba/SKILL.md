# Oracle DBA — AI Skill Definition

## Purpose
This skill teaches AI assistants to help with Oracle Database administration tasks including storage, users, backup/recovery, and multitenant architecture.

---

## User and Privilege Management

```sql
-- Create a user
CREATE USER app_user
    IDENTIFIED BY "SecureP@ssw0rd!"
    DEFAULT TABLESPACE users
    TEMPORARY TABLESPACE temp
    QUOTA UNLIMITED ON users
    QUOTA 100M ON data_ts;

-- Basic connect privileges
GRANT CREATE SESSION TO app_user;

-- Application privileges
GRANT CREATE TABLE, CREATE VIEW, CREATE PROCEDURE,
      CREATE SEQUENCE, CREATE TRIGGER, CREATE TYPE TO app_user;

-- Role management
CREATE ROLE app_read_role;
GRANT SELECT ON schema_owner.customers TO app_read_role;
GRANT SELECT ON schema_owner.orders    TO app_read_role;
GRANT app_read_role TO app_user;

-- Object privileges
GRANT SELECT, INSERT, UPDATE ON customers TO app_user;
GRANT EXECUTE ON pkg_customer_mgmt TO app_user;

-- View current user's privileges
SELECT * FROM USER_SYS_PRIVS;
SELECT * FROM USER_ROLE_PRIVS;
SELECT * FROM USER_TAB_PRIVS;
```

---

## Tablespace Management

```sql
-- Create bigfile tablespace
CREATE BIGFILE TABLESPACE data_ts
    DATAFILE '/u01/oradata/ORCL/data_ts01.dbf'
    SIZE 10G
    AUTOEXTEND ON NEXT 512M MAXSIZE UNLIMITED
    EXTENT MANAGEMENT LOCAL AUTOALLOCATE
    SEGMENT SPACE MANAGEMENT AUTO;

-- Add datafile to existing tablespace
ALTER TABLESPACE data_ts
    ADD DATAFILE '/u01/oradata/ORCL/data_ts02.dbf'
    SIZE 5G AUTOEXTEND ON;

-- Monitor tablespace usage
SELECT
    df.tablespace_name,
    ROUND(df.total_mb, 2)        AS total_mb,
    ROUND(fs.free_mb, 2)         AS free_mb,
    ROUND(df.total_mb - fs.free_mb, 2) AS used_mb,
    ROUND((1 - fs.free_mb / df.total_mb) * 100, 1) AS pct_used
FROM
    (SELECT tablespace_name, SUM(bytes)/1024/1024 AS total_mb FROM dba_data_files GROUP BY tablespace_name) df
    JOIN
    (SELECT tablespace_name, SUM(bytes)/1024/1024 AS free_mb  FROM dba_free_space  GROUP BY tablespace_name) fs
    ON df.tablespace_name = fs.tablespace_name
ORDER BY pct_used DESC;

-- Resize datafile
ALTER DATABASE DATAFILE '/u01/oradata/ORCL/data_ts01.dbf' RESIZE 20G;
```

---

## RMAN Backup & Recovery

```bash
# Full database backup
rman target /
RMAN> BACKUP DATABASE PLUS ARCHIVELOG;

# Incremental backup (Level 0 = full baseline, Level 1 = changed blocks)
RMAN> BACKUP INCREMENTAL LEVEL 0 DATABASE;
RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE;

# Configure retention policy
RMAN> CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;
RMAN> CONFIGURE BACKUP OPTIMIZATION ON;

# Delete obsolete backups
RMAN> DELETE OBSOLETE;

# Restore and recover after complete loss
RMAN> STARTUP NOMOUNT;
RMAN> RESTORE CONTROLFILE FROM AUTOBACKUP;
RMAN> ALTER DATABASE MOUNT;
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE;
RMAN> ALTER DATABASE OPEN RESETLOGS;
```

---

## Data Pump (Export / Import)

```bash
# Full schema export
expdp system/password@ORCL \
    SCHEMAS=MY_SCHEMA \
    DIRECTORY=DATA_PUMP_DIR \
    DUMPFILE=my_schema_%U.dmp \
    LOGFILE=my_schema_export.log \
    PARALLEL=4 \
    COMPRESSION=ALL

# Import to different schema
impdp system/password@ORCL \
    DIRECTORY=DATA_PUMP_DIR \
    DUMPFILE=my_schema_%U.dmp \
    REMAP_SCHEMA=MY_SCHEMA:NEW_SCHEMA \
    REMAP_TABLESPACE=DATA_TS:NEW_DATA_TS \
    LOGFILE=my_schema_import.log

# Table-level export
expdp my_schema/password@ORCL \
    TABLES=CUSTOMERS,ORDERS \
    DIRECTORY=DATA_PUMP_DIR \
    DUMPFILE=tables_export.dmp \
    QUERY="CUSTOMERS:\"WHERE status='ACTIVE'\"" \
    LOGFILE=tables_export.log
```

---

## Oracle Multitenant (CDB / PDB)

```sql
-- Connect to root container
CONNECT / AS SYSDBA
-- or: CONNECT sys/password@ORCL AS SYSDBA

-- List all containers
SELECT con_id, name, open_mode FROM v$pdbs;
SELECT con_id, name, open_mode FROM v$containers;

-- Switch to a PDB
ALTER SESSION SET CONTAINER = MYPDB;

-- Open a PDB
ALTER PLUGGABLE DATABASE MYPDB OPEN;
ALTER PLUGGABLE DATABASE ALL OPEN;  -- Open all PDBs

-- Create a new PDB from seed
CREATE PLUGGABLE DATABASE newpdb
    ADMIN USER pdb_admin IDENTIFIED BY Password123
    ROLES = (DBA)
    DEFAULT TABLESPACE users
        DATAFILE '/u01/oradata/CDB01/newpdb/users01.dbf' SIZE 250M AUTOEXTEND ON
    FILE_NAME_CONVERT = ('/u01/oradata/CDB01/pdbseed/', '/u01/oradata/CDB01/newpdb/');

-- Common users (start with C##)
CREATE USER c##common_admin IDENTIFIED BY Password123 CONTAINER=ALL;
GRANT DBA TO c##common_admin CONTAINER=ALL;

-- Local user (inside PDB only)
ALTER SESSION SET CONTAINER = MYPDB;
CREATE USER local_app_user IDENTIFIED BY Password123;
```

---

## Monitoring Queries

```sql
-- Active sessions
SELECT
    s.sid,
    s.serial#,
    s.username,
    s.status,
    s.machine,
    s.program,
    s.wait_class,
    s.event,
    s.seconds_in_wait,
    SUBSTR(sq.sql_text, 1, 100) AS sql_preview
FROM v$session s
LEFT JOIN v$sql sq ON s.sql_id = sq.sql_id
WHERE s.type = 'USER'
  AND s.status = 'ACTIVE'
ORDER BY s.seconds_in_wait DESC;

-- Top wait events
SELECT event, total_waits, time_waited_micro/1000000 AS time_waited_sec
FROM v$system_event
WHERE wait_class != 'Idle'
ORDER BY time_waited_micro DESC
FETCH FIRST 20 ROWS ONLY;

-- Kill a session
ALTER SYSTEM KILL SESSION '123,456' IMMEDIATE;

-- Check redo log switches (high frequency = undersized logs)
SELECT
    TRUNC(first_time, 'HH24') AS log_hour,
    COUNT(*) AS switches_per_hour
FROM v$log_history
WHERE first_time > SYSDATE - 1
GROUP BY TRUNC(first_time, 'HH24')
ORDER BY 1;
```
