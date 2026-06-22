# ORAXIA — Oracle AI Skills for Claude Code

You are working on an Oracle Database and/or Oracle APEX project.
Always apply the patterns and conventions from the skill files in this repository.

## Active Skills

@skills/oracle-sql/SKILL.md
@skills/oracle-plsql/SKILL.md
@skills/oracle-apex/SKILL.md
@skills/oracle-dba/SKILL.md
@skills/oracle-performance/SKILL.md
@skills/oracle-security/SKILL.md

## Core Oracle Conventions

- Use `VARCHAR2` (never `VARCHAR`) for string columns in Oracle
- Use `NUMBER` for numeric types; be explicit with precision where needed
- Use `DATE '2024-01-01'` literal syntax (not string concatenation for dates)
- Always use bind variables: `:P1_ITEM` in APEX, `:param` in PL/SQL
- Prefer packages over standalone procedures and functions
- Prefix package parameters: `p_` for IN, `p_out_` for OUT parameters
- Always handle `NO_DATA_FOUND` and `OTHERS` in PL/SQL exception blocks
- Use `%TYPE` and `%ROWTYPE` for variable declarations
- Use `SYSDATE` for date, `SYSTIMESTAMP` for timestamp with timezone
- Sequences: `seq_name.NEXTVAL` or use `GENERATED ALWAYS AS IDENTITY` (12c+)

## Oracle APEX Conventions

- Page items are named `P{page_num}_{ITEM_NAME}` e.g. `P5_CUSTOMER_ID`
- Reference session state in SQL/PL/SQL with colon: `:P5_CUSTOMER_ID`
- Use `V('APP_USER')` or `:APP_USER` for current APEX user
- Use `APEX_ERROR.ADD_ERROR` for user-friendly error messages
- Use `APEX_JSON` for building JSON responses in PL/SQL
- Use `APEX_MAIL.SEND` for email; `APEX_UTIL` for user management
- Use `APEX_ESCAPE.HTML` to sanitize output and prevent XSS
- Always use bind variables in SQL regions — never string concatenation

## SQL Style Guide

- Use uppercase for SQL keywords: `SELECT`, `FROM`, `WHERE`, `JOIN`
- Alias all tables and qualify all columns with alias
- Add comments for complex queries
- Use CTEs (`WITH` clause) for readability over nested subqueries
- Prefer `FETCH FIRST n ROWS ONLY` over `ROWNUM` (12c+)

## File Naming Conventions

```
packages/    pkg_{name}.pks (spec), pkg_{name}.pkb (body)
procedures/  prc_{name}.pls
functions/   fnc_{name}.pls
triggers/    trg_{table}_{timing}.pls
views/       vw_{name}.sql
indexes/     idx_{table}_{columns}.sql
```
