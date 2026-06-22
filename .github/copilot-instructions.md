# ORAXIA — Oracle AI Skills for GitHub Copilot & Codex

This project uses Oracle Database and/or Oracle APEX.
Always generate code that follows Oracle-specific syntax and best practices.

## Oracle SQL Rules

- Use `VARCHAR2` not `VARCHAR` for Oracle string columns
- Use `NUMBER(p,s)` for numerics; `NUMBER` alone for flexible precision
- Use DATE literals: `DATE '2024-01-15'` not string casts
- `SYSDATE` for current date/time; `SYSTIMESTAMP` for timestamp with timezone
- `ROWNUM` for pre-12c row limiting; `FETCH FIRST n ROWS ONLY` for 12c+
- `NVL(expr, default)` for NULL substitution; `COALESCE` also works
- Use `DECODE` or `CASE WHEN` for conditional logic in SQL
- Sequences: `sequence_name.NEXTVAL`; or `GENERATED ALWAYS AS IDENTITY` (12c+)
- NULL comparison: use `IS NULL` / `IS NOT NULL`, never `= NULL`
- `DUAL` is the dummy single-row table for expression evaluation

## PL/SQL Rules

- Always wrap code in `BEGIN ... EXCEPTION ... END;` blocks
- Declare variables with `%TYPE`: `v_id employees.employee_id%TYPE;`
- Use cursor FOR loops: `FOR rec IN (SELECT ...) LOOP ... END LOOP;`
- Use `BULK COLLECT` + `FORALL` for high-volume DML (>1000 rows)
- Error codes: `SQLCODE` (negative integer), `SQLERRM` (text message)
- `RAISE_APPLICATION_ERROR(-20001 to -20999, 'message')` for custom errors
- Avoid DDL inside PL/SQL unless using `EXECUTE IMMEDIATE`
- Use `DBMS_OUTPUT.PUT_LINE` only in development; remove for production
- Commit/Rollback explicitly; never rely on implicit commit

## Oracle APEX Rules

- Page items: `P{n}_{NAME}` e.g. `P3_ORDER_ID`, referenced as `:P3_ORDER_ID` in SQL
- Dynamic Actions: use 'Execute PL/SQL Code' with 'Items to Submit'
- APEX APIs: `APEX_JSON`, `APEX_MAIL`, `APEX_UTIL`, `APEX_WEB_SERVICE`, `APEX_ESCAPE`
- REST/ORDS: use `ORDS.DEFINE_MODULE`, `ORDS.DEFINE_HANDLER` for REST APIs
- Validations return BOOLEAN (TRUE = valid, FALSE = invalid)
- Use `APEX_ERROR.ADD_ERROR` to add multiple validation errors
- Escape output: `APEX_ESCAPE.HTML(value)` to prevent XSS
- Session state: `V('ITEM_NAME')` in SQL, `:ITEM_NAME` in PL/SQL bind contexts

## Security Rules

- NEVER concatenate user input into dynamic SQL strings
- Always use bind variables for parameterized queries
- Use VPD (`DBMS_RLS`) for row-level security
- Apply TDE for sensitive columns (credit cards, SSN, etc.)
- Use `AUTHID CURRENT_USER` for invoker-rights procedures
- Validate and sanitize all user inputs before DML operations

## Performance Rules

- Add indexes on foreign key columns and frequently filtered columns
- Use `DBMS_STATS.GATHER_TABLE_STATS` after bulk data loads
- Use `/*+ RESULT_CACHE */` hint for reference data queries
- Use `/*+ APPEND */` hint for large bulk inserts
- Avoid functions on indexed columns in WHERE clauses (defeats the index)
- Use partitioning for tables exceeding 10 million rows
