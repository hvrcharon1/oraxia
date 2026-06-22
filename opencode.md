# ORAXIA — Oracle AI Skills for OpenCode

## Project Context
This is an Oracle Database / Oracle APEX project.
All AI suggestions must follow Oracle-specific conventions.

## Oracle SQL Conventions
- String type: VARCHAR2 (not VARCHAR)
- Numeric type: NUMBER or NUMBER(precision, scale)
- Date literal: DATE 'YYYY-MM-DD'
- Current datetime: SYSDATE (date), SYSTIMESTAMP (timestamp)
- Row limit: FETCH FIRST n ROWS ONLY
- NULL test: IS NULL / IS NOT NULL
- Concatenation: || operator
- Dummy table: DUAL
- Sequences: seq.NEXTVAL

## PL/SQL Conventions
- Structure: DECLARE / BEGIN / EXCEPTION / END
- Anchored types: v_name table.column%TYPE
- Cursor: FOR rec IN (SELECT ...) LOOP ... END LOOP
- Bulk: BULK COLLECT INTO arr LIMIT 1000; FORALL i IN arr.FIRST..arr.LAST
- Exception: NO_DATA_FOUND, TOO_MANY_ROWS, DUP_VAL_ON_INDEX, OTHERS
- Custom error: RAISE_APPLICATION_ERROR(-20001, 'Custom message')
- Package pattern: CREATE PACKAGE spec + CREATE PACKAGE BODY

## Oracle APEX Conventions
- Item naming: P{page}_{NAME} e.g. P1_CUSTOMER_ID
- SQL binding: :P1_CUSTOMER_ID (colon prefix)
- Process type: PL/SQL Code or Execute Server-Side Code
- APIs: APEX_JSON, APEX_MAIL, APEX_UTIL, APEX_WEB_SERVICE
- Escape: APEX_ESCAPE.HTML() for all user-supplied output values
- REST: ORDS module + handler definitions
- Dynamic Actions: JavaScript (apex.item, apex.region) + PL/SQL server-side
