# Implementing a Logging Wrapper Package in PostgreSQL

Because **PostgreSQL does not natively support an explicit `CREATE PACKAGE` syntax** like Oracle or PL/SQL, database engineers use **Database Schemas** to build logical "packages." This design pattern groups related configurations, custom session states, and logging routines into a single namespace. 

This document outlines how to build a unified, lightweight logging wrapper package designed to structure `RAISE` messages cleanly for cloud log parsing tools like **GCP Cloud Logging**.

---

## 🏛️ Architecture Breakdown

The framework mimics an enterprise package structure by mapping features directly to native PostgreSQL constructs:

| Package Component | PostgreSQL Implementation Strategy |
| :--- | :--- |
| **Namespace Container** | **Database Schema** (e.g., `CREATE SCHEMA log;`) |
| **Package Procedures** | **Schema-scoped Stored Procedures** (e.g., `log.write()`) |
| **Global Package Variables** | **Custom Session Settings** via GUC (`current_setting()`) |

---

## 💾 Core Code Implementation

Execute the following SQL blocks in your PostgreSQL instance (such as GCP Cloud SQL) using an account with `cloudsqlsuperuser` or appropriate structural deployment privileges.

### 1. Initialize the Package Container
Isolate the framework objects away from your `public` schema to keep your database environment organized.

```sql
-- Create the dedicated package schema
CREATE SCHEMA IF NOT EXISTS log;
```

### 2. Create the Centralized Logger Routine
This stored procedure encapsulates the `RAISE` engine behavior. It dynamically extracts operational session metadata, maps text severities to engine log states, and enforces a standardized JSON-like payload structure.

```sql
CREATE OR REPLACE PROCEDURE log.write(
    p_level VARCHAR,     -- Allowed values: 'DEBUG', 'INFO', 'NOTICE', 'WARNING', 'ERROR'
    p_message TEXT,
    p_module VARCHAR DEFAULT NULL
)
LANGUAGE plpgsql AS $$
DECLARE
    v_corr_id TEXT;
    v_payload TEXT;
BEGIN
    -- Pull the transaction's unique correlation ID from custom session parameters
    BEGIN
        v_corr_id := current_setting('custom.current_correlation_id', true);
    EXCEPTION WHEN OTHERS THEN
        v_corr_id := 'UNKNOWN';
    END;

    -- Format the message text layout cleanly for cloud-native parsing tools
    v_payload := format('[MODULE: %s] [CORR_ID: %s] %s', 
                        coalesce(upper(p_module), 'GLOBAL'), 
                        coalesce(v_corr_id, 'N/A'), 
                        p_message);

    -- Map text evaluation strings dynamically to internal structural RAISE levels
    CASE upper(p_level)
        WHEN 'DEBUG' THEN 
            RAISE DEBUG '%', v_payload;
        WHEN 'INFO' THEN 
            RAISE INFO '%', v_payload;
        WHEN 'NOTICE' THEN 
            RAISE NOTICE '%', v_payload;
        WHEN 'WARNING' THEN 
            RAISE WARNING '%', v_payload;
        WHEN 'ERROR', 'EXCEPTION' THEN 
            RAISE EXCEPTION '%', v_payload;
        ELSE 
            RAISE NOTICE '[LEVEL: %] %', upper(p_level), v_payload;
    END CASE;
END;
$$;
```

---

## 🚀 Consumption & Usage Examples

### Step A: Initialize Session State
At the very beginning of your application request pipeline, API routing middleware, or master job script, inject your operational state tracing IDs.

```sql
-- Inject a state identifier for tracking execution across nested queries
SET custom.current_correlation_id = 'api-req-abc123xyz';
```

### Step B: Consuming the Framework within Business Stored Procedures
Instead of typing unformatted `RAISE` commands directly into routines, developers call the schema package helper.

```sql
CREATE OR REPLACE PROCEDURE public.process_billing_queue()
LANGUAGE plpgsql AS $$
BEGIN
    -- Write a standard operational status trace
    CALL log.write('INFO', 'Starting billing processor execution loop.', 'BILLING_ENGINE');

    -- Simulating data evaluations / exception hooks
    IF 1=1 THEN
        CALL log.write('WARNING', 'Skipping processing for Account #4059 due to null token.', 'BILLING_ENGINE');
    END IF;

    -- Finish execution cycle
    CALL log.write('DEBUG', 'Billing engine cycle finished clean.', 'BILLING_ENGINE');

EXCEPTION WHEN OTHERS THEN
    -- Capture unhandled crashes cleanly while forwarding the error payload
    CALL log.write('ERROR', format('Fatal execution crash: %s', SQLERRM), 'BILLING_ENGINE');
END;
$$;
```

---

## 🎨 Maximizing for Google Cloud Logging

When running inside **GCP Cloud SQL**, the native logging subsystem automatically forwards structural outputs to your cloud workspace. 

1. **Configure Your Instance Database Flags:** 
   * Ensure `log_min_messages` is configured to `NOTICE` or `INFO` within your GCP Console. If left at the default `WARNING` constraint level, your custom `INFO`, `DEBUG`, and `NOTICE` entries will be suppressed before they hit the stream.
2. **Execute Filtering inside Logs Explorer:**
   * Because the wrapper standardizes the string schema (`[MODULE: X] [CORR_ID: Y]`), you can easily isolate issues inside the **GCP Logs Explorer** using precise text matches:
     ```text
     resource.type="cloudsql_database"
     textPayload:"[MODULE: BILLING_ENGINE]"
     textPayload:"[CORR_ID: api-req-abc123xyz]"
     ```

---
*This is for informational purposes only. AI responses may include mistakes.*
