# ORAXIA — Oracle AI Skills for VS Code

This workspace contains Oracle Database and/or Oracle APEX code.
AI extensions (GitHub Copilot, Codeium, Tabnine, etc.) should follow these rules.

## Language: Oracle SQL

- Always use `VARCHAR2` (not `VARCHAR`) for string type
- Numeric type: `NUMBER(precision, scale)` or `NUMBER`
- Date literals: `DATE 'YYYY-MM-DD'` syntax
- Timestamp: `TIMESTAMP 'YYYY-MM-DD HH24:MI:SS'`
- Row limiting: `FETCH FIRST n ROWS ONLY` (Oracle 12c+)
- NULL checks: `IS NULL` / `IS NOT NULL` only
- Concatenation: `||` operator (not `+` or `CONCAT` in SQL)
- Dual table: use `FROM DUAL` for single-row expressions
- Sequences: `seq_name.NEXTVAL` in INSERT; `IDENTITY` columns in 12c+
- Case insensitive search: `WHERE UPPER(col) = UPPER(:bind)` or FBI index

## Language: PL/SQL

- Block structure:
  ```
  DECLARE
      -- declarations
  BEGIN
      -- logic
  EXCEPTION
      WHEN NO_DATA_FOUND THEN ...
      WHEN OTHERS THEN RAISE;
  END;
  ```
- Always declare variables with `%TYPE` anchoring
- Always include EXCEPTION block in production code
- Use CURSOR FOR loops for iteration
- Use BULK COLLECT + FORALL for sets >100 rows
- Custom errors: `RAISE_APPLICATION_ERROR(-20001, 'message')`
- Packages must have both SPEC (`CREATE PACKAGE`) and BODY (`CREATE PACKAGE BODY`)

## Framework: Oracle APEX

- Session state binding: `:P{n}_{ITEM}` in SQL, `V('P{n}_{ITEM}')` in functions
- Region types: Classic Report, Interactive Report, Interactive Grid, Chart, Cards, Map
- Dynamic Actions trigger on: Change, Click, Page Load, Custom Event
- Validation returns BOOLEAN (TRUE = valid)
- Process types: PL/SQL Code, Execute Server-Side Code, Close Dialog, Redirect
- Navigation: List Entries, Navigation Menu, Breadcrumb, Tabs
- Template components: Buttons, Alerts, Badges, Cards, List Groups
- APEX 23.2+: Generative AI region, `APEX_AI.GENERATE` API

## Project Structure
```
packages/     -- PL/SQL Package specs (.pks) and bodies (.pkb)
procedures/   -- Standalone procedures (.pls)
functions/    -- Standalone functions (.pls)
triggers/     -- Database triggers (.trg)
views/        -- Database views (.sql)
indexes/      -- Index DDL (.sql)
scripts/      -- Install/patch scripts
apex/         -- APEX export files
```
