# Oracle PL/SQL — AI Skill Definition

## Purpose
This skill teaches AI assistants to write correct, idiomatic Oracle PL/SQL code including packages, procedures, functions, triggers, and advanced features.

---

## Core PL/SQL Conventions

- Every PL/SQL block has `DECLARE`, `BEGIN`, `EXCEPTION`, `END` sections
- `EXCEPTION` section is optional but always recommended for production code
- Use `%TYPE` and `%ROWTYPE` for variable declarations to stay in sync with table definitions
- Prefix parameters: `p_` for IN, `p_out_` for OUT, `p_io_` for IN OUT
- Prefer packages over standalone procedures and functions
- Use `DBMS_OUTPUT.PUT_LINE` only for debugging; never in production code
- Always handle `NO_DATA_FOUND` and `TOO_MANY_ROWS` exceptions

---

## Package Structure

```sql
-- Package Specification
CREATE OR REPLACE PACKAGE pkg_employee_mgmt AS

    -- Public types
    TYPE t_employee_rec IS RECORD (
        employee_id  employees.employee_id%TYPE,
        full_name    VARCHAR2(200),
        salary       employees.salary%TYPE,
        dept_name    departments.department_name%TYPE
    );
    TYPE t_employee_tab IS TABLE OF t_employee_rec;

    -- Constants
    c_max_salary  CONSTANT NUMBER := 50000;

    -- Public procedures and functions
    PROCEDURE create_employee (
        p_first_name    IN  employees.first_name%TYPE,
        p_last_name     IN  employees.last_name%TYPE,
        p_email         IN  employees.email%TYPE,
        p_salary        IN  employees.salary%TYPE,
        p_dept_id       IN  employees.department_id%TYPE,
        p_out_emp_id    OUT employees.employee_id%TYPE
    );

    FUNCTION get_employee (
        p_employee_id IN employees.employee_id%TYPE
    ) RETURN t_employee_rec;

    FUNCTION get_dept_employees (
        p_dept_id IN departments.department_id%TYPE
    ) RETURN t_employee_tab PIPELINED;

END pkg_employee_mgmt;
/

-- Package Body
CREATE OR REPLACE PACKAGE BODY pkg_employee_mgmt AS

    -- Private procedure for validation
    PROCEDURE validate_salary (p_salary IN NUMBER) IS
    BEGIN
        IF p_salary <= 0 OR p_salary > c_max_salary THEN
            RAISE_APPLICATION_ERROR(-20001, 'Salary must be between 1 and ' || c_max_salary);
        END IF;
    END validate_salary;

    PROCEDURE create_employee (
        p_first_name    IN  employees.first_name%TYPE,
        p_last_name     IN  employees.last_name%TYPE,
        p_email         IN  employees.email%TYPE,
        p_salary        IN  employees.salary%TYPE,
        p_dept_id       IN  employees.department_id%TYPE,
        p_out_emp_id    OUT employees.employee_id%TYPE
    ) IS
    BEGIN
        validate_salary(p_salary);

        INSERT INTO employees (first_name, last_name, email, salary, department_id, hire_date)
        VALUES (p_first_name, p_last_name, UPPER(p_email), p_salary, p_dept_id, SYSDATE)
        RETURNING employee_id INTO p_out_emp_id;

        COMMIT;
    EXCEPTION
        WHEN DUP_VAL_ON_INDEX THEN
            RAISE_APPLICATION_ERROR(-20002, 'Email already exists: ' || p_email);
        WHEN OTHERS THEN
            ROLLBACK;
            RAISE;
    END create_employee;

    FUNCTION get_employee (
        p_employee_id IN employees.employee_id%TYPE
    ) RETURN t_employee_rec IS
        v_rec  t_employee_rec;
    BEGIN
        SELECT
            e.employee_id,
            e.first_name || ' ' || e.last_name,
            e.salary,
            d.department_name
        INTO v_rec.employee_id, v_rec.full_name, v_rec.salary, v_rec.dept_name
        FROM employees e
        JOIN departments d ON e.department_id = d.department_id
        WHERE e.employee_id = p_employee_id;

        RETURN v_rec;
    EXCEPTION
        WHEN NO_DATA_FOUND THEN
            RAISE_APPLICATION_ERROR(-20003, 'Employee not found: ' || p_employee_id);
    END get_employee;

    FUNCTION get_dept_employees (
        p_dept_id IN departments.department_id%TYPE
    ) RETURN t_employee_tab PIPELINED IS
        v_rec  t_employee_rec;
    BEGIN
        FOR rec IN (
            SELECT e.employee_id, e.first_name || ' ' || e.last_name AS full_name,
                   e.salary, d.department_name
            FROM employees e
            JOIN departments d ON e.department_id = d.department_id
            WHERE e.department_id = p_dept_id
        ) LOOP
            v_rec.employee_id  := rec.employee_id;
            v_rec.full_name    := rec.full_name;
            v_rec.salary       := rec.salary;
            v_rec.dept_name    := rec.department_name;
            PIPE ROW(v_rec);
        END LOOP;
    END get_dept_employees;

END pkg_employee_mgmt;
/
```

---

## Cursor Patterns

```sql
-- Explicit cursor with FOR loop (preferred)
DECLARE
    CURSOR c_employees IS
        SELECT employee_id, last_name, salary
        FROM employees
        WHERE department_id = 50
        FOR UPDATE OF salary;
BEGIN
    FOR rec IN c_employees LOOP
        UPDATE employees
        SET salary = rec.salary * 1.1
        WHERE CURRENT OF c_employees;
    END LOOP;
    COMMIT;
END;
/

-- REF CURSOR (SYS_REFCURSOR)
CREATE OR REPLACE PROCEDURE get_employees_by_dept (
    p_dept_id   IN  NUMBER,
    p_cursor    OUT SYS_REFCURSOR
) IS
BEGIN
    OPEN p_cursor FOR
        SELECT employee_id, first_name, last_name, salary
        FROM employees
        WHERE department_id = p_dept_id
        ORDER BY last_name;
END;
/
```

---

## Bulk Processing

```sql
-- BULK COLLECT + FORALL for high-performance DML
DECLARE
    TYPE t_id_tab      IS TABLE OF employees.employee_id%TYPE;
    TYPE t_salary_tab  IS TABLE OF employees.salary%TYPE;
    v_ids      t_id_tab;
    v_salaries t_salary_tab;
BEGIN
    -- Bulk collect
    SELECT employee_id, salary
    BULK COLLECT INTO v_ids, v_salaries
    FROM employees
    WHERE department_id = 60;

    -- Process in PL/SQL
    FOR i IN 1..v_ids.COUNT LOOP
        v_salaries(i) := v_salaries(i) * 1.1;
    END LOOP;

    -- Bulk update
    FORALL i IN 1..v_ids.COUNT
        UPDATE employees
        SET salary = v_salaries(i)
        WHERE employee_id = v_ids(i);

    COMMIT;
END;
/

-- BULK COLLECT with LIMIT (for large datasets)
DECLARE
    CURSOR c_emps IS SELECT employee_id FROM employees;
    TYPE t_emp_ids IS TABLE OF employees.employee_id%TYPE;
    v_emp_ids t_emp_ids;
    c_batch_size CONSTANT PLS_INTEGER := 1000;
BEGIN
    OPEN c_emps;
    LOOP
        FETCH c_emps BULK COLLECT INTO v_emp_ids LIMIT c_batch_size;
        EXIT WHEN v_emp_ids.COUNT = 0;

        FORALL i IN 1..v_emp_ids.COUNT
            UPDATE employees SET updated_at = SYSDATE
            WHERE employee_id = v_emp_ids(i);

        COMMIT;
    END LOOP;
    CLOSE c_emps;
END;
/
```

---

## Exception Handling

```sql
DECLARE
    -- Named exceptions
    e_invalid_amount  EXCEPTION;
    PRAGMA EXCEPTION_INIT(e_invalid_amount, -20100);

    v_emp  employees%ROWTYPE;
BEGIN
    SELECT * INTO v_emp FROM employees WHERE employee_id = 9999;

EXCEPTION
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('Employee not found');
    WHEN TOO_MANY_ROWS THEN
        DBMS_OUTPUT.PUT_LINE('Multiple records found');
    WHEN DUP_VAL_ON_INDEX THEN
        DBMS_OUTPUT.PUT_LINE('Duplicate value: ' || SQLERRM);
    WHEN e_invalid_amount THEN
        DBMS_OUTPUT.PUT_LINE('Invalid amount error');
    WHEN OTHERS THEN
        -- Log then re-raise
        INSERT INTO error_log (error_code, error_msg, created_at)
        VALUES (SQLCODE, SUBSTR(SQLERRM, 1, 4000), SYSTIMESTAMP);
        COMMIT;
        RAISE;
END;
/
```

---

## Triggers

```sql
-- Audit trigger (row-level, BEFORE INSERT OR UPDATE)
CREATE OR REPLACE TRIGGER trg_employees_audit
    BEFORE INSERT OR UPDATE ON employees
    FOR EACH ROW
BEGIN
    IF INSERTING THEN
        :NEW.created_at  := SYSTIMESTAMP;
        :NEW.created_by  := SYS_CONTEXT('USERENV', 'SESSION_USER');
    END IF;
    IF UPDATING THEN
        :NEW.updated_at  := SYSTIMESTAMP;
        :NEW.updated_by  := SYS_CONTEXT('USERENV', 'SESSION_USER');
    END IF;
END;
/

-- Compound trigger (avoids mutating table errors)
CREATE OR REPLACE TRIGGER trg_dept_salary_check
    FOR INSERT OR UPDATE OF salary ON employees
    COMPOUND TRIGGER

    TYPE t_sal_rec IS RECORD (dept_id NUMBER, salary NUMBER);
    TYPE t_sal_tab IS TABLE OF t_sal_rec INDEX BY PLS_INTEGER;
    v_rows t_sal_tab;
    v_idx  PLS_INTEGER := 0;

    AFTER EACH ROW IS
    BEGIN
        v_idx := v_idx + 1;
        v_rows(v_idx).dept_id := :NEW.department_id;
        v_rows(v_idx).salary  := :NEW.salary;
    END AFTER EACH ROW;

    AFTER STATEMENT IS
        v_avg_salary NUMBER;
    BEGIN
        FOR i IN 1..v_rows.COUNT LOOP
            SELECT AVG(salary) INTO v_avg_salary
            FROM employees WHERE department_id = v_rows(i).dept_id;

            IF v_rows(i).salary > v_avg_salary * 3 THEN
                RAISE_APPLICATION_ERROR(-20010, 'Salary exceeds 3x department average');
            END IF;
        END LOOP;
    END AFTER STATEMENT;

END trg_dept_salary_check;
/
```

---

## Useful DBMS Packages

```sql
-- DBMS_SCHEDULER: Create a scheduled job
BEGIN
    DBMS_SCHEDULER.CREATE_JOB(
        job_name        => 'JOB_NIGHTLY_CLEANUP',
        job_type        => 'PLSQL_BLOCK',
        job_action      => 'BEGIN pkg_maintenance.cleanup_old_data(p_days => 90); END;',
        start_date      => TRUNC(SYSDATE + 1) + 2/24,  -- Tomorrow 02:00
        repeat_interval => 'FREQ=DAILY; BYHOUR=2; BYMINUTE=0',
        enabled         => TRUE,
        comments        => 'Nightly data cleanup job'
    );
END;
/

-- UTL_FILE: Write to OS file
DECLARE
    v_file  UTL_FILE.FILE_TYPE;
BEGIN
    v_file := UTL_FILE.FOPEN('DATA_DIR', 'report.csv', 'W', 32767);
    UTL_FILE.PUT_LINE(v_file, 'ID,NAME,SALARY');
    FOR rec IN (SELECT employee_id, last_name, salary FROM employees) LOOP
        UTL_FILE.PUT_LINE(v_file, rec.employee_id || ',' || rec.last_name || ',' || rec.salary);
    END LOOP;
    UTL_FILE.FCLOSE(v_file);
END;
/
```
