# Oracle Security — AI Skill Definition

## Purpose
This skill teaches AI assistants to implement Oracle Database security features including VPD/RLS, encryption, auditing, and secure coding practices.

---

## Virtual Private Database (VPD / Row Level Security)

```sql
-- Create a VPD policy function
CREATE OR REPLACE FUNCTION fn_vpd_customer_policy (
    p_schema  IN VARCHAR2,
    p_table   IN VARCHAR2
) RETURN VARCHAR2 IS
    v_user     VARCHAR2(100);
    v_pred     VARCHAR2(4000);
BEGIN
    v_user := SYS_CONTEXT('USERENV', 'SESSION_USER');

    -- Admins see everything
    IF v_user = 'APP_ADMIN' THEN
        RETURN NULL;  -- No filter
    END IF;

    -- Salesreps see only their own customers
    v_pred := 'ASSIGNED_REP = SYS_CONTEXT(''USERENV'', ''SESSION_USER'')';
    RETURN v_pred;
END;
/

-- Apply VPD policy to table
BEGIN
    DBMS_RLS.ADD_POLICY(
        object_schema   => 'MY_SCHEMA',
        object_name     => 'CUSTOMERS',
        policy_name     => 'CUST_ROW_POLICY',
        function_schema => 'MY_SCHEMA',
        policy_function => 'FN_VPD_CUSTOMER_POLICY',
        statement_types => 'SELECT, INSERT, UPDATE, DELETE',
        update_check    => TRUE,
        enable          => TRUE
    );
END;

-- Using Application Context for VPD
CREATE OR REPLACE PACKAGE pkg_app_ctx AS
    PROCEDURE set_user_context (p_user_id IN NUMBER, p_org_id IN NUMBER);
END;
/
CREATE OR REPLACE PACKAGE BODY pkg_app_ctx AS
    PROCEDURE set_user_context (p_user_id IN NUMBER, p_org_id IN NUMBER) IS
    BEGIN
        DBMS_SESSION.SET_CONTEXT('MY_APP_CTX', 'USER_ID', p_user_id);
        DBMS_SESSION.SET_CONTEXT('MY_APP_CTX', 'ORG_ID',  p_org_id);
    END;
END;
/

-- Reference context in VPD function
RETURN 'ORG_ID = SYS_CONTEXT(''MY_APP_CTX'', ''ORG_ID'')';
```

---

## Transparent Data Encryption (TDE)

```sql
-- Set master key (run once per DB)
ADMINISTER KEY MANAGEMENT SET KEYSTORE OPEN IDENTIFIED BY "WalletPassword123";
ADMINISTER KEY MANAGEMENT SET KEY IDENTIFIED BY "WalletPassword123" WITH BACKUP;

-- Encrypt a tablespace (19c+)
ALTER TABLESPACE sensitive_data ENCRYPTION ONLINE USING 'AES256' ENCRYPT;

-- Encrypt a column
ALTER TABLE customers
    MODIFY (credit_card_number ENCRYPT USING 'AES256' NO SALT);
-- Note: SALT prevents exact match queries; NO SALT allows equality search

-- Encrypt at column level with default algorithm
CREATE TABLE payment_info (
    payment_id    NUMBER PRIMARY KEY,
    customer_id   NUMBER,
    card_number   VARCHAR2(20) ENCRYPT,
    cvv           VARCHAR2(4)  ENCRYPT NO SALT,
    card_expiry   DATE         ENCRYPT
);

-- Check TDE status
SELECT tablespace_name, encrypted FROM dba_tablespaces;
SELECT table_name, column_name, encryption_alg FROM dba_encrypted_columns;
```

---

## Unified Auditing (Oracle 12c+)

```sql
-- Create audit policy
CREATE AUDIT POLICY pol_sensitive_data_access
    ACTIONS SELECT ON customers.credit_cards,
             INSERT ON customers.credit_cards,
             UPDATE ON customers.credit_cards,
             DELETE ON customers.credit_cards
    WHEN 'SYS_CONTEXT(''USERENV'',''SESSION_USER'') != ''APP_SERVICE_USER'''
    EVALUATE PER SESSION;

-- Enable policy
AUDIT POLICY pol_sensitive_data_access;

-- Audit failed logins
CREATE AUDIT POLICY pol_failed_logins
    ACTIONS LOGON
    WHEN 'action_name = ''LOGON''';
AUDIT POLICY pol_failed_logins WHENEVER NOT SUCCESSFUL;

-- View audit trail
SELECT
    event_timestamp,
    db_username,
    action_name,
    object_schema,
    object_name,
    return_code,
    client_identifier
FROM unified_audit_trail
WHERE event_timestamp > SYSTIMESTAMP - INTERVAL '1' HOUR
  AND db_username NOT IN ('SYS', 'SYSTEM')
ORDER BY event_timestamp DESC;

-- Purge audit trail
BEGIN
    DBMS_AUDIT_MGMT.CLEAN_AUDIT_TRAIL(
        audit_trail_type        => DBMS_AUDIT_MGMT.AUDIT_TRAIL_UNIFIED,
        use_last_arch_timestamp => FALSE,
        delete_percent          => 100
    );
END;
```

---

## Oracle Wallet Configuration

```bash
# Create wallet directory
mkdir -p /etc/oracle/wallet

# Create wallet
orapki wallet create -wallet /etc/oracle/wallet -pwd WalletPwd123 -auto_login

# Add credential (for external connections)
mkstore -wrl /etc/oracle/wallet -createCredential mydb_alias app_user app_password

# sqlnet.ora entry
WALLET_LOCATION = (SOURCE = (METHOD = FILE) (METHOD_DATA = (DIRECTORY = /etc/oracle/wallet)))
SQLNET.WALLET_OVERRIDE = TRUE
```

---

## Secure PL/SQL Coding

```sql
-- NEVER use dynamic SQL with string concatenation for user input:
-- BAD:
EXECUTE IMMEDIATE 'SELECT * FROM ' || p_table_name;  -- SQL injection risk!

-- GOOD: Validate input against allowed values
DECLARE
    v_allowed_tables CONSTANT SYS.ODCIVARCHAR2LIST :=
        SYS.ODCIVARCHAR2LIST('EMPLOYEES', 'DEPARTMENTS', 'ORDERS');
    v_is_valid NUMBER := 0;
BEGIN
    SELECT COUNT(*) INTO v_is_valid
    FROM TABLE(v_allowed_tables)
    WHERE COLUMN_VALUE = UPPER(p_table_name);

    IF v_is_valid = 0 THEN
        RAISE_APPLICATION_ERROR(-20099, 'Invalid table name');
    END IF;
    EXECUTE IMMEDIATE 'SELECT COUNT(*) FROM ' || UPPER(p_table_name);
END;

-- GOOD: Use bind variables in dynamic SQL
EXECUTE IMMEDIATE
    'SELECT COUNT(*) FROM employees WHERE department_id = :1'
    INTO v_count USING p_dept_id;

-- Invoker rights for privilege security
CREATE OR REPLACE PROCEDURE run_as_caller
AUTHID CURRENT_USER AS
BEGIN
    -- This runs with the CALLER'S privileges, not the owner's
    -- Prevents privilege escalation through stored procedures
    SELECT COUNT(*) FROM sensitive_table INTO v_count;
END;
```

---

## APEX Security Checklist

```sql
-- 1. Always use bind variables in APEX SQL
-- WRONG: 'WHERE id = ' || :P1_ID
-- RIGHT:  WHERE id = :P1_ID

-- 2. Enable APEX Application Security attributes:
--    Session Management > Maximum Session Duration
--    Authentication > Expired Password
--    Browser Security > Embed in Frames: Deny

-- 3. Content Security Policy headers (APEX 20.1+)
--    Application > Security > Browser Security > Content Security Policy

-- 4. APEX_ESCAPE for output escaping in PL/SQL
v_safe_output := APEX_ESCAPE.HTML(p_user_input);
v_safe_js     := APEX_ESCAPE.JS_LITERAL(p_value);

-- 5. Validate file uploads
IF APEX_APPLICATION.G_F01(1) NOT IN ('image/jpeg','image/png','application/pdf') THEN
    APEX_ERROR.ADD_ERROR(
        p_message => 'Invalid file type',
        p_display_location => apex_error.c_inline_in_notification
    );
END IF;
```
