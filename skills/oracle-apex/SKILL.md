# Oracle APEX — AI Skill Definition

## Purpose
This skill teaches AI assistants to build correct, idiomatic Oracle APEX applications. Always apply these patterns when working with Oracle Application Express (APEX).

---

## APEX Architecture Fundamentals

- APEX applications run inside Oracle Database — all computation happens in DB context
- Pages render via HTML/CSS/JS but all data logic is PL/SQL
- Session state stores page item values — always use `V()` or `:ITEM_NAME` to reference them
- Page items names: `P{page_number}_{ITEM_NAME}` e.g. `P1_CUSTOMER_ID`
- Global page = Page 0 — components here appear on every page
- Always reference items with colon syntax in SQL/PL/SQL: `:P1_CUSTOMER_ID`
- Use `APEX_APPLICATION.G_USER` for current user; `V('APP_USER')` also works

---

## Page Item Patterns

```sql
-- Reference page items in queries (bind variable syntax)
SELECT * FROM customers WHERE customer_id = :P5_CUSTOMER_ID;

-- Set item value in PL/SQL process
APEX_UTIL.SET_SESSION_STATE('P5_STATUS', 'ACTIVE');

-- Get item value in PL/SQL
DECLARE
    v_id NUMBER := V('P5_CUSTOMER_ID');
BEGIN
    NULL;
END;

-- Clear item value
APEX_UTIL.SET_SESSION_STATE('P5_TEMP_FLAG', NULL);
```

---

## Interactive Report (IR) Tips

```sql
-- IR Source Query — always alias computed columns
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    c.email,
    c.status,
    COUNT(o.order_id)   AS order_count,
    SUM(o.total_amount) AS lifetime_value,
    MAX(o.order_date)   AS last_order_date
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE (:P3_STATUS IS NULL OR c.status = :P3_STATUS)
GROUP BY c.customer_id, c.first_name, c.last_name, c.email, c.status;
```

```javascript
// Refresh IR from Dynamic Action (Execute JavaScript)
apex.region("my-ir-static-id").refresh();

// Set IR filter programmatically
apex.region("my-ir-static-id").widget().interactiveReport("addFilter", {
    filterType: "column",
    columnName: "STATUS",
    operator: "=",
    value: "ACTIVE"
});
```

---

## Interactive Grid (IG) Tips

```javascript
// Refresh IG
apex.region("my-ig-static-id").refresh();

// Get IG model and selected records
var model = apex.region("my-ig-static-id").call("getViews").grid.model;
var selectedRecords = apex.region("my-ig-static-id").call("getSelectedRecords");

// Add a row
apex.region("my-ig-static-id").call("addRow");

// Save IG
apex.region("my-ig-static-id").call("save");
```

---

## Dynamic Actions (DA)

### PL/SQL process in DA
```sql
-- DA > Execute PL/SQL Code > PL/SQL Code:
BEGIN
    UPDATE orders
    SET status = :P10_STATUS,
        updated_at = SYSTIMESTAMP
    WHERE order_id = :P10_ORDER_ID;
    COMMIT;
EXCEPTION
    WHEN OTHERS THEN
        APEX_ERROR.ADD_ERROR(
            p_message          => 'Update failed: ' || SQLERRM,
            p_display_location => apex_error.c_inline_in_notification
        );
END;
-- Items to Submit: P10_STATUS, P10_ORDER_ID
-- Items to Return: (none required unless you need to refresh)
```

### JavaScript DA patterns
```javascript
// Show/hide items
apex.item("P5_SECONDARY_FIELD").show();
apex.item("P5_SECONDARY_FIELD").hide();

// Set item value
apex.item("P5_STATUS").setValue("ACTIVE");

// Get item value
var val = apex.item("P5_CUSTOMER_ID").getValue();

// Trigger event on item change
$("#P5_CATEGORY").trigger("change");

// APEX notification
apex.message.showPageSuccess("Record saved successfully!");
apex.message.alert("Please select at least one record.");
```

---

## APEX_ITEMS and Session State APIs

```sql
-- apex_util: user management
BEGIN
    APEX_UTIL.EDIT_USER(
        p_user_id  => APEX_UTIL.GET_USER_ID('JOHN.DOE'),
        p_web_password => 'NewPassword123!'
    );
END;

-- apex_json: build JSON responses
BEGIN
    APEX_JSON.OPEN_OBJECT;
    APEX_JSON.WRITE('status',  'success');
    APEX_JSON.WRITE('user_id', :APP_USER);
    APEX_JSON.WRITE('count',   42);
    APEX_JSON.CLOSE_OBJECT;
END;

-- apex_mail: send email
BEGIN
    APEX_MAIL.SEND(
        p_to        => 'recipient@example.com',
        p_from      => 'no-reply@myapp.com',
        p_subj      => 'Your Order Confirmation',
        p_body      => 'Thank you for your order.',
        p_body_html => '<h2>Thank you for your order!</h2>'
    );
    APEX_MAIL.PUSH_QUEUE;
END;

-- apex_web_service: REST calls
DECLARE
    v_response  CLOB;
BEGIN
    v_response := APEX_WEB_SERVICE.MAKE_REST_REQUEST(
        p_url         => 'https://api.example.com/users',
        p_http_method => 'GET',
        p_parm_name   => APEX_T_VARCHAR2('Authorization'),
        p_parm_value  => APEX_T_VARCHAR2('Bearer ' || :P1_API_TOKEN)
    );
    APEX_JSON.PARSE(v_response);
    -- read values: APEX_JSON.GET_VARCHAR2('$.name')
END;
```

---

## ORDS — RESTful Services

```sql
-- Enable schema for ORDS
BEGIN
    ORDS.ENABLE_SCHEMA(
        p_enabled             => TRUE,
        p_schema              => 'MY_SCHEMA',
        p_url_mapping_type    => 'BASE_PATH',
        p_url_mapping_pattern => 'api'
    );
    COMMIT;
END;

-- Define a REST module
BEGIN
    ORDS.DEFINE_MODULE(
        p_module_name    => 'customers.v1',
        p_base_path      => '/customers/v1/',
        p_items_per_page => 25
    );

    -- GET /customers/v1/
    ORDS.DEFINE_TEMPLATE(
        p_module_name    => 'customers.v1',
        p_pattern        => '/'
    );
    ORDS.DEFINE_HANDLER(
        p_module_name    => 'customers.v1',
        p_pattern        => '/',
        p_method         => 'GET',
        p_source_type    => ORDS.SOURCE_TYPE_COLLECTION_FEED,
        p_source         => 'SELECT customer_id, first_name, last_name, email FROM customers ORDER BY customer_id'
    );

    -- GET /customers/v1/:id
    ORDS.DEFINE_TEMPLATE(
        p_module_name    => 'customers.v1',
        p_pattern        => '/:id'
    );
    ORDS.DEFINE_HANDLER(
        p_module_name    => 'customers.v1',
        p_pattern        => '/:id',
        p_method         => 'GET',
        p_source_type    => ORDS.SOURCE_TYPE_COLLECTION_ITEM,
        p_source         => 'SELECT * FROM customers WHERE customer_id = :id'
    );

    COMMIT;
END;
```

---

## Authorization Schemes

```sql
-- Custom authorization scheme (PL/SQL function returning Boolean)
DECLARE
    v_count NUMBER;
BEGIN
    SELECT COUNT(*)
    INTO v_count
    FROM user_roles
    WHERE username = :APP_USER
      AND role_name = :APP_PAGE_ALIAS;  -- or hardcoded role

    RETURN v_count > 0;
END;

-- APEX Access Control
CREATE OR REPLACE FUNCTION is_admin_user RETURN BOOLEAN IS
    v_count NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_count
    FROM apex_workspace_apex_users
    WHERE user_name = UPPER(V('APP_USER'))
    AND is_admin = 'Y';
    RETURN v_count > 0;
END;
```

---

## APEX 23.2+ AI / Generative AI Integration

```sql
-- Configure AI service in APEX Admin, then use:
DECLARE
    v_prompt    VARCHAR2(4000);
    v_response  VARCHAR2(32767);
BEGIN
    v_prompt := 'Summarize this customer feedback in 2 sentences: ' || :P5_FEEDBACK_TEXT;

    v_response := APEX_AI.GENERATE(
        p_prompt           => v_prompt,
        p_service_alias    => 'MY_AI_SERVICE'
    );

    :P5_AI_SUMMARY := v_response;
END;
```

---

## Common Page Validations (PL/SQL)

```sql
-- Validation: check required field
RETURN :P5_EMAIL IS NOT NULL;

-- Validation: check date range
RETURN :P5_START_DATE <= :P5_END_DATE;

-- Validation: check uniqueness
DECLARE
    v_count NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_count
    FROM customers
    WHERE LOWER(email) = LOWER(:P5_EMAIL)
    AND customer_id != NVL(:P5_CUSTOMER_ID, -1);
    RETURN v_count = 0;
END;
```

---

## Performance Tips for APEX

- Use bind variables in all SQL regions: `:P1_ID` not `'` || :P1_ID || `'`
- Enable Result Cache on slow report queries: `/*+ RESULT_CACHE */`
- Use IR/IG with lazy loading for large datasets
- Avoid PL/SQL functions in SELECT clause of report queries — use SQL expressions instead
- Compress and cache static JS/CSS files using APEX File Manager
- Use Partial Page Refresh (PPR) instead of full page submit where possible
