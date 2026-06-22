# Oracle ORDS — AI Skill Definition

## Purpose
This skill teaches AI assistants to build, configure, and consume Oracle REST Data Services (ORDS) APIs. ORDS turns Oracle database objects directly into RESTful HTTP endpoints without a middleware layer. Use this skill whenever you need to expose tables, views, or stored procedures as REST APIs, protect those APIs with OAuth 2.0, or call external REST services from PL/SQL.

---

## AutoREST — Instantly REST-Enable a Table or View

AutoREST is the fastest route from table to REST API. Enabling AutoREST on an object generates GET (collection + single item), POST, PUT, PATCH, and DELETE handlers automatically, with built-in pagination and JSON serialization.

```sql
-- Enable ORDS for a schema (run once as DBA / ORDS admin)
BEGIN
    ORDS.ENABLE_SCHEMA(
        p_enabled             => TRUE,
        p_schema              => 'HR',
        p_url_mapping_type    => 'BASE_PATH',
        p_url_mapping_pattern => 'hr'             -- becomes /ords/hr/...
    );
    COMMIT;
END;
/

-- Enable AutoREST on a table
BEGIN
    ORDS.ENABLE_OBJECT(
        p_enabled      => TRUE,
        p_schema       => 'HR',
        p_object       => 'EMPLOYEES',
        p_object_type  => 'TABLE',
        p_object_alias => 'employees'
    );
    COMMIT;
END;
/

-- Enable AutoREST on a view
BEGIN
    ORDS.ENABLE_OBJECT(
        p_enabled      => TRUE,
        p_schema       => 'HR',
        p_object       => 'V_ACTIVE_EMPLOYEES',
        p_object_type  => 'VIEW',
        p_object_alias => 'active-employees'
    );
    COMMIT;
END;
/
```

After enabling, these endpoints exist automatically (assuming base URL `/ords/hr`):

```
GET    /ords/hr/employees/          — paginated JSON collection
GET    /ords/hr/employees/:id       — single item by primary key
POST   /ords/hr/employees/          — insert a new row
PUT    /ords/hr/employees/:id       — replace a row
PATCH  /ords/hr/employees/:id       — partial update
DELETE /ords/hr/employees/:id       — delete a row
```

---

## Custom REST Modules

For full control over SQL, response shape, parameter handling, and HTTP method behaviour, define custom REST modules with handlers.

```sql
-- Create the module (a named group of REST endpoints sharing a base path)
BEGIN
    ORDS.DEFINE_MODULE(
        p_module_name    => 'api.employees',
        p_base_path      => '/employees/',
        p_items_per_page => 25,
        p_status         => 'PUBLISHED',
        p_comments       => 'Employee management REST module'
    );
    COMMIT;
END;
/

-- Define a URL template and a GET handler for employees by department
BEGIN
    ORDS.DEFINE_TEMPLATE(
        p_module_name => 'api.employees',
        p_pattern     => 'by-dept/:dept_id'
    );

    ORDS.DEFINE_HANDLER(
        p_module_name => 'api.employees',
        p_pattern     => 'by-dept/:dept_id',
        p_method      => 'GET',
        p_source_type => ORDS.source_type_collection_feed,
        p_source      => q'[
            SELECT
                e.employee_id,
                e.first_name || ' ' || e.last_name AS full_name,
                e.email,
                e.salary,
                e.hire_date
            FROM employees e
            WHERE e.department_id = :dept_id
            ORDER BY e.last_name
        ]'
    );
    COMMIT;
END;
/

-- POST handler: create a new employee from a JSON body
BEGIN
    ORDS.DEFINE_TEMPLATE(
        p_module_name => 'api.employees',
        p_pattern     => '.'
    );

    ORDS.DEFINE_HANDLER(
        p_module_name => 'api.employees',
        p_pattern     => '.',
        p_method      => 'POST',
        p_source_type => ORDS.source_type_plsql,
        p_source      => q'[
            DECLARE
                v_emp_id NUMBER;
            BEGIN
                INSERT INTO employees (
                    first_name, last_name, email,
                    salary, department_id, hire_date
                )
                VALUES (
                    :first_name, :last_name, :email,
                    :salary,     :department_id, SYSDATE
                )
                RETURNING employee_id INTO v_emp_id;
                COMMIT;
                :status_code := 201;
                :location    := '/ords/hr/employees/' || v_emp_id;
            END;
        ]'
    );
    COMMIT;
END;
/
```

---

## Handler Source Types

Choose the correct `p_source_type` for each handler — it determines how ORDS executes the SQL or PL/SQL and serialises the result.

| Source Type Constant | Alias | Use When |
|---|---|---|
| `ORDS.source_type_collection_feed` | `"feed"` | GET collection: SELECT → paginated JSON array with `hasMore`, `links` |
| `ORDS.source_type_collection_item` | `"item"` | GET single item: SELECT → one JSON object |
| `ORDS.source_type_media` | `"media"` | Binary download: BLOB/CLOB with custom Content-Type header |
| `ORDS.source_type_plsql` | `"plsql"` | POST/PUT/DELETE: PL/SQL block with bind variable output |
| `ORDS.source_type_query` | `"query"` | SELECT with complex CTEs or multi-result-set logic |
| `ORDS.source_type_query_one_row` | `"query/one-row"` | Complex SELECT that returns exactly one JSON object |

---

## Automatic Pagination

When using `source_type_collection_feed`, ORDS wraps the result in a standard envelope and adds pagination links. Clients control page size with `?limit=n&offset=n` query parameters. The module-level `p_items_per_page` sets the default page size.

```json
{
  "items": [
    { "employee_id": 1, "full_name": "John Smith", "salary": 75000 },
    ...
  ],
  "hasMore": true,
  "limit": 25,
  "offset": 0,
  "count": 25,
  "links": [
    { "rel": "self",  "href": "/ords/hr/employees/by-dept/10?offset=0&limit=25" },
    { "rel": "next",  "href": "/ords/hr/employees/by-dept/10?offset=25&limit=25" },
    { "rel": "first", "href": "/ords/hr/employees/by-dept/10?offset=0&limit=25" }
  ]
}
```

---

## Bind Variables and Special Binds

ORDS automatically binds URI template parameters, query-string parameters, and JSON body fields as SQL or PL/SQL bind variables. Never concatenate user input into SQL inside handlers.

```sql
-- URI template   /employees/:id     → :id is bound
-- Query string   /employees/?status=ACTIVE → :status is bound
-- JSON body field "salary"          → :salary is bound in PL/SQL handlers

-- Reserved bind variables always available in every handler:
-- :status_code       — set HTTP response status (default 200)
-- :location          — set Location response header (for 201 Created)
-- :forward_location  — redirect processing to another template path
-- :body              — raw request body as CLOB
-- :body_text         — raw request body as VARCHAR2(32767)
-- :content_type      — response Content-Type header (for media handlers)
-- :fetch_offset      — current page offset (read-only, for feed handlers)
-- :fetch_size        — current page limit  (read-only, for feed handlers)

-- Example: full control PL/SQL handler using special binds
p_source => q'[
    BEGIN
        UPDATE employees SET status = :new_status
        WHERE employee_id = :id;
        IF SQL%ROWCOUNT = 0 THEN
            :status_code := 404;
        ELSE
            COMMIT;
            :status_code := 200;
        END IF;
    END;
]'
```

---

## Documenting Parameters (OpenAPI)

ORDS auto-generates an OpenAPI 3.0 specification. Use `ORDS.DEFINE_PARAMETER` to document each parameter so the spec is accurate.

```sql
BEGIN
    ORDS.DEFINE_PARAMETER(
        p_module_name        => 'api.employees',
        p_pattern            => 'by-dept/:dept_id',
        p_method             => 'GET',
        p_name               => 'dept_id',
        p_bind_variable_name => 'dept_id',
        p_source_type        => 'URI',
        p_param_type         => 'INTEGER',
        p_access_method      => 'IN',
        p_comments           => 'Department ID to filter employees'
    );
    COMMIT;
END;
/

-- View the auto-generated OpenAPI spec
-- GET /ords/hr/employees/metadata-catalog/
```

---

## OAuth 2.0 — Protecting Endpoints

ORDS includes a built-in OAuth 2.0 authorization server. Use the Client Credentials flow for server-to-server integrations and Authorization Code for user-facing apps.

```sql
-- 1. Create a privilege: maps a role name to a set of protected URL patterns
BEGIN
    ORDS.CREATE_PRIVILEGE(
        p_name        => 'hr.employees.read',
        p_role_name   => 'HR_READ_ROLE',
        p_label       => 'HR Read',
        p_description => 'Read access to the employee REST endpoints'
    );
    COMMIT;
END;
/

-- 2. Map the privilege to a URL pattern
BEGIN
    ORDS.CREATE_PRIVILEGE_MAPPING(
        p_privilege_name => 'hr.employees.read',
        p_pattern        => '/employees/*'
    );
    COMMIT;
END;
/

-- 3. Register an OAuth client (Client Credentials)
BEGIN
    OAUTH.CREATE_CLIENT(
        p_name            => 'payroll_service',
        p_grant_type      => 'client_credentials',
        p_description     => 'Payroll service integration client',
        p_support_email   => 'devops@example.com',
        p_privilege_names => 'hr.employees.read'
    );
    COMMIT;
END;
/

-- 4. Retrieve the generated client_id and client_secret
SELECT name, client_id, client_secret
FROM user_ords_clients
WHERE name = 'payroll_service';

-- 5. Token request from the consuming application:
-- POST /ords/hr/oauth/token
-- Content-Type: application/x-www-form-urlencoded
-- Body: grant_type=client_credentials&client_id=...&client_secret=...
-- Response: { "access_token": "...", "token_type": "bearer", "expires_in": 3600 }

-- 6. Use the token in subsequent API calls:
-- Authorization: Bearer <access_token>
```

---

## Calling External REST Services from PL/SQL

From Oracle's PL/SQL engine you can call outbound REST APIs using `APEX_WEB_SERVICE` (requires APEX) or `UTL_HTTP` (requires wallet configuration for HTTPS).

```sql
-- Using APEX_WEB_SERVICE (available when APEX is installed)
DECLARE
    v_response  CLOB;
BEGIN
    APEX_WEB_SERVICE.SET_REQUEST_HEADERS(
        p_name_01  => 'Authorization',
        p_value_01 => 'Bearer ' || :token
    );
    v_response := APEX_WEB_SERVICE.MAKE_REST_REQUEST(
        p_url         => 'https://api.exchangerate.host/latest?base=USD&symbols=EUR,GBP',
        p_http_method => 'GET'
    );
    DBMS_OUTPUT.PUT_LINE('Status: ' || APEX_WEB_SERVICE.G_STATUS_CODE);
    DBMS_OUTPUT.PUT_LINE(SUBSTR(v_response, 1, 500));
END;
/

-- Using UTL_HTTP (no APEX dependency; HTTPS requires Oracle Wallet)
DECLARE
    l_req   UTL_HTTP.REQ;
    l_resp  UTL_HTTP.RESP;
    l_body  VARCHAR2(32767);
    l_line  VARCHAR2(32767);
BEGIN
    UTL_HTTP.SET_WALLET('file:/oracle/wallet', 'wallet_password');

    l_req := UTL_HTTP.BEGIN_REQUEST(
        url    => 'https://api.example.com/orders',
        method => 'POST'
    );
    UTL_HTTP.SET_HEADER(l_req, 'Content-Type',  'application/json');
    UTL_HTTP.SET_HEADER(l_req, 'Authorization', 'Bearer ' || :api_key);
    UTL_HTTP.WRITE_TEXT(l_req, '{"status": "SHIPPED", "order_id": 1001}');

    l_resp := UTL_HTTP.GET_RESPONSE(l_req);
    DBMS_OUTPUT.PUT_LINE('HTTP ' || l_resp.status_code);

    LOOP
        BEGIN
            UTL_HTTP.READ_LINE(l_resp, l_line, TRUE);
            l_body := l_body || l_line;
        EXCEPTION WHEN UTL_HTTP.END_OF_BODY THEN EXIT;
        END;
    END LOOP;
    UTL_HTTP.END_RESPONSE(l_resp);
    DBMS_OUTPUT.PUT_LINE(SUBSTR(l_body, 1, 500));
END;
/
```

---

## Inspecting ORDS Metadata

```sql
-- List all modules in the current schema
SELECT module_name, base_path, status, items_per_page
FROM user_ords_modules
ORDER BY module_name;

-- List all URL templates for a module
SELECT uri_template, pattern
FROM user_ords_templates
WHERE module_name = 'api.employees';

-- List all handlers (method + source type) for a module
SELECT t.uri_template, h.method, h.source_type, h.items_per_page
FROM user_ords_templates t
JOIN user_ords_handlers  h ON t.id = h.template_id
WHERE t.module_name = 'api.employees';

-- List AutoREST-enabled objects
SELECT object_name, object_type, object_alias, enabled
FROM user_ords_enabled_objects
ORDER BY object_type, object_name;

-- List OAuth clients and privileges
SELECT name, client_id, grant_type
FROM user_ords_clients;

SELECT privilege_name, pattern
FROM user_ords_privilege_mappings;
```

---

## Best Practices

- Use `source_type_collection_feed` for all GET collection endpoints — it gives automatic pagination, metadata, and `hasMore` for free; only fall back to `source_type_plsql` for DML (POST, PUT, DELETE) or complex transactional logic
- Bind variables are mandatory for parameterisation — never concatenate user input into query strings inside handlers, as ORDS handlers are not immune to SQL injection
- Always `COMMIT` ORDS metadata changes (`ORDS.DEFINE_MODULE`, `ORDS.DEFINE_HANDLER`, etc.) — these are regular DML statements against ORDS metadata tables, not auto-committed DDL
- Set `p_status => 'NOT_PUBLISHED'` during development and flip to `PUBLISHED` only when the endpoint is ready for use
- Use `ORDS.DEFINE_PARAMETER` for all URI and query parameters so the auto-generated OpenAPI spec is complete and accurate
- Test ORDS endpoints from SQL Developer Web → REST Workshop before any front-end integration
