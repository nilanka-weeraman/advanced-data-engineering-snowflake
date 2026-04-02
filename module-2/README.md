# Module 2: Snowflake Observability

This module is about making Snowflake workloads observable and actionable:

- capture telemetry in an event table
- emit logs from Python stored procedures
- emit trace events and span attributes
- query telemetry to debug pipeline behavior
- create alerts for data quality issues
- schedule alerts and trigger notifications

The examples in this folder are based on:

- [sproc.sql](C:\claudecode\coursera-snowflake\advanced-data-engineering-snowflake\module-2\sproc.sql)
- [solution_sproc_logs.sql](C:\claudecode\coursera-snowflake\advanced-data-engineering-snowflake\module-2\solutions\solution_sproc_logs.sql)
- [solution_sproc_traces.sql](C:\claudecode\coursera-snowflake\advanced-data-engineering-snowflake\module-2\solutions\solution_sproc_traces.sql)
- [notification.sql](C:\claudecode\coursera-snowflake\advanced-data-engineering-snowflake\module-2\notification.sql)
- [solution_notification.sql](C:\claudecode\coursera-snowflake\advanced-data-engineering-snowflake\module-2\solutions\solution_notification.sql)
- [alert.sql](C:\claudecode\coursera-snowflake\advanced-data-engineering-snowflake\module-2\alert.sql)
- [solution_alert.sql](C:\claudecode\coursera-snowflake\advanced-data-engineering-snowflake\module-2\solutions\solution_alert.sql)

## 1. Mental Model

The flow in this module is:

1. Create or choose an event table.
2. Set telemetry levels so Snowflake actually records logs and traces.
3. Instrument a stored procedure with Python `logging` and Snowflake telemetry APIs.
4. Query the event table to inspect what happened.
5. Create an alert that checks for a condition.
6. Schedule the alert.
7. Optionally call a notification procedure to send an email.

## 2. Event Tables

An event table is where Snowflake stores telemetry such as logs and trace events.

Snowflake already provides a default event table:

```sql
SNOWFLAKE.TELEMETRY.EVENTS
```

In this module, telemetry is queried from:

```sql
USE DATABASE staging_tasty_bytes;
USE SCHEMA telemetry;

SELECT * FROM pipeline_events;
```

That means your environment already has an event table or a view wired into the `TELEMETRY` schema for this tutorial.

### Create and configure a custom event table

If you want your own event table for a database or account:

```sql
USE ROLE ACCOUNTADMIN;

CREATE EVENT TABLE staging_tasty_bytes.telemetry.pipeline_events;

ALTER DATABASE staging_tasty_bytes
  SET EVENT_TABLE = staging_tasty_bytes.telemetry.pipeline_events;

SHOW PARAMETERS LIKE 'event_table' IN DATABASE staging_tasty_bytes;
```

If you want account-wide collection instead:

```sql
ALTER ACCOUNT
  SET EVENT_TABLE = staging_tasty_bytes.telemetry.pipeline_events;

SHOW PARAMETERS LIKE 'event_table' IN ACCOUNT;
```

### Query useful telemetry from the event table

Logs:

```sql
SELECT *
FROM staging_tasty_bytes.telemetry.pipeline_events
WHERE record_type = 'LOG'
ORDER BY timestamp DESC;
```

Trace rows and span-related records:

```sql
SELECT *
FROM staging_tasty_bytes.telemetry.pipeline_events
WHERE record_type ILIKE '%SPAN%'
ORDER BY timestamp DESC;
```

Compact query for recall:

```sql
SELECT
  timestamp,
  record_type,
  scope['name']::string AS logger_or_scope,
  record,
  record_attributes
FROM staging_tasty_bytes.telemetry.pipeline_events
ORDER BY timestamp DESC;
```

## 3. Configure Logging and Tracing

Telemetry is only captured if the levels are enabled.

### Logging level

Module 2 sets account-level logging like this:

```sql
ALTER ACCOUNT SET LOG_LEVEL = 'INFO';
```

Meaning:

- `INFO`, `WARN`, `ERROR`, and `FATAL` logs are stored
- lower-severity messages are ignored

### Trace level

To collect trace events for the current session:

```sql
ALTER SESSION SET TRACE_LEVEL = ALWAYS;
```

This is the key switch that allows `telemetry.add_event(...)` and span attributes to show up in the event table.

## 4. Stored Procedure with Logs

The module’s stored procedure reads from `order_header_stream`, filters Hamburg orders, and writes or prepares daily sales output.

Basic pattern:

```sql
CREATE OR REPLACE PROCEDURE staging_tasty_bytes.raw_pos.process_order_headers_stream()
  RETURNS STRING
  LANGUAGE PYTHON
  RUNTIME_VERSION = '3.10'
  HANDLER = 'process_order_headers_stream'
  PACKAGES = ('snowflake-snowpark-python')
AS
$$
import snowflake.snowpark.functions as F
from snowflake.snowpark import Session
import logging

def process_order_headers_stream(session: Session) -> str:
    logger = logging.getLogger('order_headers_stream_sproc')
    logger.info("Starting process_order_headers_stream procedure")

    try:
        logger.info("Querying order_header_stream for new records")
        recent_orders = session.table("order_header_stream").filter(
            F.col("METADATA$ACTION") == "INSERT"
        )

        logger.info("Filtering orders for Hamburg, Germany")
        locations = session.table("location")
        hamburg_orders = recent_orders.join(
            locations,
            recent_orders["LOCATION_ID"] == locations["LOCATION_ID"]
        ).filter(
            (locations["CITY"] == "Hamburg") &
            (locations["COUNTRY"] == "Germany")
        )

        hamburg_count = hamburg_orders.count()
        logger.info(f"Found {hamburg_count} orders from Hamburg")

        logger.info("Procedure completed successfully")
        return "Success"
    except Exception as e:
        logger.error(f"Error processing orders: {str(e)}")
        raise
$$;
```

### What to remember

- Use Python `logging`.
- Create a named logger with `logging.getLogger(...)`.
- Add `logger.info(...)` around key business steps.
- Add `logger.error(...)` in the exception path.
- Query the event table with `record_type = 'LOG'`.

## 5. Trace Events, Spans, and Attributes

Logs tell you what happened. Traces tell you where and in what execution context it happened.

In this module, tracing is added with the Snowflake telemetry package:

```sql
PACKAGES = ('snowflake-snowpark-python', 'snowflake-telemetry-python')
```

Python pattern used in the solution:

```python
from snowflake import telemetry
import uuid

trace_id = str(uuid.uuid4())

telemetry.set_span_attribute("procedure", "process_order_headers_stream")
telemetry.set_span_attribute("trace_id", trace_id)

telemetry.set_span_attribute("process_step", "query_stream")
telemetry.add_event("query_begin", {"description": "Starting to query order_header_stream"})

telemetry.add_event("query_complete", {"description": "Completed query of order_header_stream"})

telemetry.set_span_attribute("process_step", "filter_locations")
telemetry.add_event("filter_begin", {"description": "Filtering for Hamburg, Germany"})

telemetry.set_span_attribute("hamburg_order_count", hamburg_count)
telemetry.add_event("filter_complete", {
    "description": "Completed Hamburg filtering",
    "order_count": hamburg_count
})
```

### What each concept means

- `telemetry.add_event(...)`
  Adds a trace event inside the current span.
- `telemetry.set_span_attribute(...)`
  Adds metadata to the current span so you can filter and understand execution later.
- span
  The execution unit for a procedure call. Snowflake creates a span for the handler execution, and your events/attributes get attached to it.

### Practical trace design

Use span attributes for stable facts about the run:

- procedure name
- trace id
- step name
- counts
- error flag

Use trace events for step boundaries:

- query begin / complete
- filter begin / complete
- aggregation begin / complete
- write begin / complete
- error event

### Error tracing pattern

```python
except Exception as e:
    error_msg = str(e)
    telemetry.set_span_attribute("error", True)
    telemetry.set_span_attribute("error_message", error_msg)
    telemetry.add_event("procedure_error", {
        "description": "Error occurred during procedure execution",
        "error_message": error_msg
    })
    logger.error(f"Error processing orders: {error_msg}")
    raise
```

### Query traces

```sql
SELECT *
FROM staging_tasty_bytes.telemetry.pipeline_events
WHERE record_type ILIKE '%SPAN%'
ORDER BY timestamp DESC;
```

If you want to inspect both the event name and its attributes:

```sql
SELECT
  timestamp,
  record_type,
  record,
  record_attributes
FROM staging_tasty_bytes.telemetry.pipeline_events
WHERE record_type ILIKE '%TRACE%'
   OR record_type ILIKE '%SPAN%'
ORDER BY timestamp DESC;
```

## 6. Running the Procedure End-to-End

The tutorial inserts sample `ORDER_HEADER` rows, then calls the procedure:

```sql
CALL staging_tasty_bytes.raw_pos.process_order_headers_stream();
```

Then inspect telemetry:

```sql
USE DATABASE staging_tasty_bytes;
USE SCHEMA telemetry;

SELECT * FROM pipeline_events;
SELECT * FROM pipeline_events WHERE record_type = 'LOG';
SELECT * FROM pipeline_events WHERE record_type ILIKE '%SPAN%';
```

This is the core observability loop:

1. run the workload
2. inspect logs
3. inspect trace events
4. confirm counts, steps, and failures

## 7. Notifications by Email

The module also sets up an email notification integration and a stored procedure that sends a rich HTML message.

### Create the email integration

```sql
CREATE OR REPLACE NOTIFICATION INTEGRATION email_notification_int
  TYPE = EMAIL
  ENABLED = TRUE
  ALLOWED_RECIPIENTS = ('your-email@example.com');
```

### Notification procedure pattern

The procedure:

- queries problematic `ORDER_HEADER` records
- counts them
- converts the result to pandas
- renders HTML
- sends the email with `SYSTEM$SEND_EMAIL`

Core send step:

```python
session.call(
    "system$send_email",
    "email_notification_int",
    "your-email@example.com",
    f"ALERT: {record_count} orders with NULL values detected",
    email_content,
    "text/html"
)
```

### What to remember

- `NOTIFICATION INTEGRATION` is required before email sending.
- `SYSTEM$SEND_EMAIL` is called from SQL or from the Python session.
- The recipient must be allowed by the integration.
- The procedure is a clean way to separate notification formatting from alert logic.

## 8. Alerts

Alerts are how you automate checks and actions inside Snowflake.

In this module, the alert checks whether `ORDER_AMOUNT` or `ORDER_TOTAL` is `NULL` in recent records.

### Supporting audit table

The tutorial first creates a table to store alert outcomes:

```sql
CREATE TABLE staging_tasty_bytes.telemetry.data_quality_alerts (
  alert_time TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
  alert_name VARCHAR,
  severity VARCHAR,
  message VARCHAR,
  record_count INTEGER
);
```

### Alert pattern from the module

```sql
CREATE OR REPLACE ALERT order_data_quality_alert
  SCHEDULE = '30 MINUTE'
  IF (EXISTS (
    SELECT *
    FROM staging_tasty_bytes.raw_pos.order_header
    WHERE (order_amount IS NULL OR order_total IS NULL)
      AND order_ts > DATEADD(hour, -6, CURRENT_TIMESTAMP())
  ))
  THEN
    BEGIN
      INSERT INTO staging_tasty_bytes.telemetry.data_quality_alerts
        (alert_name, severity, message, record_count)
      SELECT
        'ORDER_HEADER_NULL_VALUES',
        'ERROR',
        'Data quality issue detected: missing amount or total values',
        COUNT(*)
      FROM staging_tasty_bytes.raw_pos.order_header
      WHERE (order_amount IS NULL OR order_total IS NULL)
        AND order_ts > DATEADD(hour, -6, CURRENT_TIMESTAMP());

      -- Optional:
      -- CALL staging_tasty_bytes.raw_pos.notify_data_quality_team();
    END;
```

### Alert setup steps

1. Define the condition with `IF (EXISTS (...))`.
2. Define the action in `THEN`.
3. Add a `SCHEDULE`.
4. Resume the alert.
5. Optionally execute it manually for testing.
6. Inspect the audit table and email results.

### Start, test, pause

```sql
SHOW ALERTS LIKE 'order_data_quality_alert';

ALTER ALERT order_data_quality_alert RESUME;

EXECUTE ALERT order_data_quality_alert;

ALTER ALERT order_data_quality_alert SUSPEND;
```

### Schedule options to remember

Simple interval:

```sql
SCHEDULE = '30 MINUTE'
```

Cron schedule:

```sql
SCHEDULE = 'USING CRON 0 * * * * UTC'
```

Useful recall:

- alerts are created suspended
- `ALTER ALERT ... RESUME` starts scheduling
- `EXECUTE ALERT ...` is for manual testing
- if no warehouse is specified, it is a serverless alert

## 9. End-to-End Alert Workflow in This Module

This module’s alert flow is:

1. detect bad records in `RAW_POS.ORDER_HEADER`
2. insert a row into `TELEMETRY.DATA_QUALITY_ALERTS`
3. optionally call `NOTIFY_DATA_QUALITY_TEAM()`
4. send an HTML email to the configured recipient

That gives you both:

- an internal audit trail in Snowflake
- an external notification by email

## 10. Quick Recall Cheatsheet

### Event table

```sql
CREATE EVENT TABLE my_db.my_schema.my_events;
ALTER DATABASE my_db SET EVENT_TABLE = my_db.my_schema.my_events;
```

### Enable telemetry

```sql
ALTER ACCOUNT SET LOG_LEVEL = 'INFO';
ALTER SESSION SET TRACE_LEVEL = ALWAYS;
```

### Python logging

```python
import logging
logger = logging.getLogger("my_logger")
logger.info("step started")
logger.error("something failed")
```

### Python tracing

```python
from snowflake import telemetry

telemetry.set_span_attribute("step", "filter")
telemetry.add_event("filter_begin", {"city": "Hamburg"})
```

### Query logs

```sql
SELECT *
FROM staging_tasty_bytes.telemetry.pipeline_events
WHERE record_type = 'LOG';
```

### Query traces

```sql
SELECT *
FROM staging_tasty_bytes.telemetry.pipeline_events
WHERE record_type ILIKE '%SPAN%';
```

### Email integration

```sql
CREATE OR REPLACE NOTIFICATION INTEGRATION email_notification_int
  TYPE = EMAIL
  ENABLED = TRUE
  ALLOWED_RECIPIENTS = ('your-email@example.com');
```

### Send email

```sql
CALL SYSTEM$SEND_EMAIL(
  'email_notification_int',
  'your-email@example.com',
  'Alert subject',
  'Alert body',
  'text/plain'
);
```

### Scheduled alert

```sql
CREATE OR REPLACE ALERT my_alert
  SCHEDULE = '30 MINUTE'
  IF (EXISTS (SELECT 1 FROM my_table WHERE status = 'ERROR'))
  THEN
    INSERT INTO my_alert_log SELECT CURRENT_TIMESTAMP(), 'ERROR FOUND';
```

### Control alert execution

```sql
ALTER ALERT my_alert RESUME;
EXECUTE ALERT my_alert;
ALTER ALERT my_alert SUSPEND;
```

## 11. What Module 2 Really Taught

By the end of this module, you practiced how to:

- instrument Snowflake stored procedures for observability
- store logs and traces in event tables
- inspect pipeline execution after a run
- convert telemetry into action with alerts
- schedule and manually test those alerts
- notify stakeholders when data quality checks fail

## 12. References

Official Snowflake docs used to verify syntax:

- Event tables: https://docs.snowflake.com/en/developer-guide/logging-tracing/event-table-setting-up
- Logging in Python: https://docs.snowflake.com/en/developer-guide/logging-tracing/logging-python
- Tracing in Python: https://docs.snowflake.com/en/developer-guide/logging-tracing/tracing-python
- Custom spans: https://docs.snowflake.com/en/developer-guide/logging-tracing/tracing-custom-spans
- Create alert: https://docs.snowflake.com/en/sql-reference/sql/create-alert
- Execute alert: https://docs.snowflake.com/en/sql-reference/sql/execute-alert
- Send email: https://docs.snowflake.com/en/user-guide/notifications/email-stored-procedures
