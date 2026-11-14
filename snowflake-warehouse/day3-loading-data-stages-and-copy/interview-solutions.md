# Day 3 — Interview Solutions: Loading Data, Stages, and COPY INTO

> Complete answers to all 30 questions. Read these after attempting the questions yourself.

---

## Concept 1: Stages and File Formats

**Q1 — What is a Snowflake Stage? What problem does it solve?**

A **stage** is a named pointer to a file storage location — either inside Snowflake's own managed storage (internal) or in your cloud account (external S3, Azure Blob, GCS).

**Problem it solves:** Snowflake tables store data in Snowflake's proprietary compressed columnar format. You cannot INSERT a CSV file directly. The stage is the **intermediary landing zone** — files land in the stage first, then `COPY INTO` reads them from the stage and loads them into tables. This separation means you can:
- Validate files before committing them to tables
- Load the same file into multiple tables
- Keep files for reprocessing if the load fails
- Apply transformations during load

---

**Q2 — Three types of internal stages. How do they differ?**

| Type | Reference | Created By | Visible To |
|---|---|---|---|
| **User Stage** | `@~` | Automatically — one per user | Only the owning user |
| **Table Stage** | `@%TABLE_NAME` | Automatically — one per table | Anyone with table privileges |
| **Named Internal Stage** | `@STAGE_NAME` | You — via `CREATE STAGE` | Anyone granted USAGE on the stage |

User stage: personal scratch area — `PUT file://local.csv @~;`
Table stage: convenient when files only go to one table — `PUT file://local.csv @%ORDERS_RAW;`
Named stage: production use — shared, visible, manageable, can have file format and encryption attached.

---

**Q3 — Internal vs external stage. Real-world scenario for each.**

**Internal stage:**
- Storage is managed by Snowflake inside its own cloud infrastructure
- You upload files using the `PUT` command from SnowSQL
- Good when: files originate on a local server, a CI/CD job, or a process that can't write directly to cloud storage

**Real-world scenario:** A nightly extract from an on-premise Oracle database writes CSV files to a local server. A script uses SnowSQL to `PUT` those CSVs into an internal stage, then triggers COPY INTO.

**External stage:**
- Storage is in your cloud account (S3 bucket, Azure Blob container, GCS bucket)
- Files stay in your storage — Snowflake reads them from there
- Good when: files are already landing in cloud storage from other systems (Kafka connectors, Fivetran, ADF pipelines, etc.)

**Real-world scenario:** Azure Data Factory copies order files from an SFTP server into an Azure Blob container. A Snowflake external stage points at that container. COPY INTO reads directly from Azure Blob — no movement needed.

---

**Q4 — Command to upload to internal stage. Can you run from Snowsight?**

The command is **`PUT`**:

```bash
PUT file:///C:/data/customers.csv @ECOMM_DB.RAW_DATA.ECOMM_INTERNAL_STAGE;
```

**No — `PUT` cannot run from a Snowsight worksheet.** It is a **SnowSQL CLI command** only. It requires direct network access from your machine to Snowflake's storage endpoint.

Alternatives if you cannot use SnowSQL:
- Use the Snowsight UI → Data → Add Data → Load Files (browser-based uploader)
- Use the Python Snowflake Connector: `cursor.execute("PUT file://local.csv @stage")`
- Use dbt's `dbt seed` for small reference data CSVs

---

**Q5 — What is a File Format? Named vs inline advantage.**

A **file format** is a named Snowflake object that stores parsing instructions for a file type — delimiter, quote character, header skip, null handling, compression, date format, etc.

**Named file format advantage:**

```sql
-- Create once
CREATE FILE FORMAT CSV_FORMAT TYPE='CSV' SKIP_HEADER=1 FIELD_DELIMITER=',';

-- Reuse everywhere — consistent, single source of truth
COPY INTO TABLE_A FROM @STAGE FILE_FORMAT = (FORMAT_NAME = 'CSV_FORMAT');
COPY INTO TABLE_B FROM @STAGE FILE_FORMAT = (FORMAT_NAME = 'CSV_FORMAT');
COPY INTO TABLE_C FROM @STAGE FILE_FORMAT = (FORMAT_NAME = 'CSV_FORMAT');

-- If the format changes (e.g., delimiter changes to |), update ONE place
ALTER FILE FORMAT CSV_FORMAT SET FIELD_DELIMITER = '|';
-- All three COPY INTOs automatically use the new format
```

Inline definitions are copy-pasted everywhere and get out of sync. Named formats enforce consistency across the pipeline.

---

**Q6 — CSV with pipe delimiter, no quotes, no header, empty = NULL.**

```sql
CREATE FILE FORMAT IF NOT EXISTS PIPE_DELIMITED_FORMAT
  TYPE                     = 'CSV'
  FIELD_DELIMITER          = '|'          -- pipe separator
  RECORD_DELIMITER         = '\n'
  SKIP_HEADER              = 0            -- no header row
  FIELD_OPTIONALLY_ENCLOSED_BY = 'NONE'  -- values not quoted
  NULL_IF                  = ('NULL', 'null', 'N/A', '\\N')
  EMPTY_FIELD_AS_NULL      = TRUE         -- empty string → NULL
  TRIM_SPACE               = TRUE
  COMPRESSION              = 'AUTO'
  COMMENT                  = 'Pipe-delimited, no header, empty=NULL';
```

---

**Q7 — External stage created. 50 new files added to container. Update stage? How to verify?**

**No — you do not need to update the stage object.** An external stage is a pointer to a storage location (a URL + credentials). It does not store a list of files — it reads whatever is in the location at query time.

The 50 new files are automatically visible the next time you:

```sql
-- List all files currently in the external stage
LIST @ECOMM_DB.RAW_DATA.ECOMM_AZURE_STAGE;

-- Or list files matching a pattern
LIST @ECOMM_DB.RAW_DATA.ECOMM_AZURE_STAGE PATTERN='.*\.csv\.gz';
```

This queries the actual storage location live and returns all files present.

---

## Concept 2: COPY INTO — All Approaches

**Q8 — What does COPY INTO do? Two required things?**

`COPY INTO` reads files from a stage and bulk-loads them into a Snowflake table. It is the primary mechanism for getting data into Snowflake in production.

**Two things always required:**
1. **Target table** — `COPY INTO ECOMM_DB.RAW_DATA.CUSTOMERS_RAW`
2. **Source stage (or stage path)** — `FROM @ECOMM_DB.RAW_DATA.ECOMM_INTERNAL_STAGE`

Everything else (FILE_FORMAT, ON_ERROR, PATTERN) is optional — Snowflake applies defaults if not specified.

---

**Q9 — What does "idempotent" mean for COPY INTO? How long is load history kept?**

**Idempotent** means: running the same COPY INTO command multiple times produces the same result as running it once. The second (and third, fourth...) run does not duplicate data.

Snowflake achieves this by **tracking every loaded file** in its internal load history. When COPY INTO runs, it checks the history — if a file was already loaded successfully, it skips it silently.

**Load history is kept for 64 days.**

This means:
- If you accidentally run COPY INTO twice, no duplicate rows are created
- If you delete rows and re-run COPY INTO, those rows are NOT reloaded (Snowflake still sees the file as "loaded")
- To force a reload: use `FORCE = TRUE`

```sql
COPY INTO my_table FROM @my_stage FORCE = TRUE;  -- bypasses history
```

---

**Q10 — `ABORT_STATEMENT` vs `CONTINUE`. Which for financial data load?**

| Setting | Behaviour |
|---|---|
| `ABORT_STATEMENT` | The entire load stops at the first bad row. No rows from any file are committed. All-or-nothing. |
| `CONTINUE` | Bad rows are skipped and logged. All valid rows from all files are committed. Load completes with partial data. |

**For production financial data: `ABORT_STATEMENT`**

Reason: Financial data requires 100% accuracy. If a row fails (e.g., an amount field has corrupted data), you want the load to stop completely so the issue is immediately visible and investigated. Silently committing partial financial data with `CONTINUE` could cause:
- Incorrect account balances
- Missing transactions that balance sheets rely on
- Compliance violations (SOX, PCI-DSS require complete, auditable data)

Use `CONTINUE` only when missing a small percentage of rows is acceptable — e.g., log data, analytics events where losing a few rows does not affect business decisions.

---

**Q11 — What does `VALIDATION_MODE = 'RETURN_ERRORS'` do? Does it load data?**

**No data is loaded.** `VALIDATION_MODE` puts COPY INTO into a **dry-run / preview mode**. It scans the files and parses every row but commits nothing to the table.

`RETURN_ERRORS` specifically: returns a result set showing every row that would fail — including the error message, the file name, the row number, and the rejected value.

```sql
-- Dry run — see errors without loading
COPY INTO my_table FROM @my_stage
FILE_FORMAT      = (FORMAT_NAME = 'CSV_FORMAT')
VALIDATION_MODE  = 'RETURN_ERRORS';
-- Returns a table of errors. 0 rows loaded into my_table.
```

**Other VALIDATION_MODE options:**
- `RETURN_N_ROWS` (e.g., `RETURN_10_ROWS`) — preview the first N rows that would be loaded
- `RETURN_ALL_ERRORS` — same as RETURN_ERRORS but includes errors from files that would be skipped

Use this before every first production load to validate the file format assumptions.

---

**Q12 — 200 files, load only January 2024 (names like `orders_2024_01_*.csv.gz`).**

```sql
COPY INTO ECOMM_DB.RAW_DATA.ORDERS_RAW
FROM @ECOMM_DB.RAW_DATA.ECOMM_INTERNAL_STAGE
PATTERN     = '.*orders_2024_01_.*\.csv\.gz'
FILE_FORMAT = (FORMAT_NAME = 'ECOMM_DB.RAW_DATA.CSV_FORMAT')
ON_ERROR    = 'ABORT_STATEMENT';
```

The `PATTERN` parameter takes a **Java regular expression**. Key regex elements:
- `.*` — match any characters (any prefix/suffix)
- `\.` — literal dot (escaped because `.` means "any char" in regex)
- `_2024_01_` — literal string match for January 2024

Alternatively, use `FILES` to list specific files explicitly:
```sql
FILES = ('orders_2024_01_01.csv.gz', 'orders_2024_01_02.csv.gz', ...)
```
But PATTERN is better when files are numerous.

---

**Q13 — What are `$1`, `$2`, `$3` in COPY INTO? When to use column references?**

`$1`, `$2`, `$3`... are **positional file column references** — they refer to the 1st, 2nd, 3rd column in the source file, in order.

They are used when you wrap the FROM clause in a SELECT subquery:

```sql
COPY INTO my_table (col_a, col_b)
FROM (
  SELECT $3, UPPER($1)   -- take column 3 from file into col_a, uppercase column 1 into col_b
  FROM @my_stage
)
FILE_FORMAT = (...);
```

**Use column references when:**
- File column order differs from table column order
- You want to skip certain file columns
- You need to transform values during load (cast types, clean strings, generate computed values)
- You want to capture `METADATA$FILENAME` as an audit column

Without a SELECT subquery, COPY INTO maps file columns to table columns positionally (1st file column → 1st table column) — no flexibility.

---

**Q14 — Run same COPY INTO twice. How many rows loaded second time? Force reload?**

**Second run loads 0 rows.**

Snowflake recorded the file as successfully loaded in its load history after the first run. The second COPY INTO checks history, sees the file is already loaded, and skips it — completing immediately with status "Copy executed with 0 files processed."

**To force a reload:**

```sql
COPY INTO my_table FROM @my_stage
FILE_FORMAT = (FORMAT_NAME = 'CSV_FORMAT')
FORCE = TRUE;   -- bypass load history
```

`FORCE = TRUE` does not clear the history — it overrides it for this run only. The rows are inserted again, potentially creating duplicates if you don't truncate first.

---

**Q15 — What is `METADATA$FILENAME`? Other metadata columns?**

`METADATA$FILENAME` is a **virtual column** available during COPY INTO that contains the full path and name of the source file being loaded for each row.

```sql
COPY INTO RAW_DATA.CUSTOMERS_RAW (_source_file, customer_id, ...)
FROM (
  SELECT METADATA$FILENAME, $1, ...
  FROM @my_stage
);
```

**Why capture it in every raw table:** When a data quality issue is found (a bad value, a wrong calculation), you need to trace which file the row came from. `METADATA$FILENAME` lets you answer "which file contained this bad row?" and reload or investigate just that file.

**All available metadata columns:**

| Column | Contains |
|---|---|
| `METADATA$FILENAME` | Full stage path and filename of the source file |
| `METADATA$FILE_ROW_NUMBER` | Row number of this record within its source file |
| `METADATA$FILE_CONTENT_KEY` | MD5 hash of the file — uniquely identifies the file version |
| `METADATA$FILE_LAST_MODIFIED` | Last modified timestamp of the source file |
| `METADATA$START_SCAN_TIME` | Timestamp when this COPY INTO started scanning |

---

## Concept 3: Semi-Structured Data and VARIANT

**Q16 — What is VARIANT? What can it store?**

`VARIANT` is Snowflake's native semi-structured data type. A VARIANT column can store any of:
- **JSON objects** — `{"key": "value"}`
- **JSON arrays** — `[1, 2, 3]`
- **Nested structures** — objects inside arrays inside objects
- **Primitive values** — a single string, number, boolean, or null
- **XML** — after parsing with `PARSE_XML()`
- **Avro and Parquet data** — loaded natively, stored as VARIANT

A single VARIANT column can hold values of different shapes in different rows — one row might be a flat object, another a deeply nested one. No schema enforced at the column level.

---

**Q17 — How to extract from VARIANT? What is `::`?**

Use the **colon `:` operator** to navigate into JSON keys. Use **double colon `::` to cast** the extracted value to a SQL type.

```sql
-- Extract and cast
SELECT
    raw_event:order_id::NUMBER       AS order_id,     -- extract key, cast to NUMBER
    raw_event:customer:name::VARCHAR AS customer_name, -- nested key
    raw_event:items[0]:sku::VARCHAR  AS first_sku      -- array index + key
FROM ORDER_EVENTS_RAW;
```

Without `::`, the extracted value is still VARIANT type. With `::`, it becomes a standard SQL type (NUMBER, VARCHAR, DATE, BOOLEAN, etc.) that SQL functions can operate on.

---

**Q18 — What does `STRIP_OUTER_ARRAY = TRUE` do in a JSON file format?**

When a JSON file contains a **top-level array** of objects, like this:

```json
[
  {"order_id": 1001, "total": 250.00},
  {"order_id": 1002, "total": 180.50},
  {"order_id": 1003, "total": 340.00}
]
```

Without `STRIP_OUTER_ARRAY = TRUE`, Snowflake treats the entire file as one VARIANT value — the whole array is one row. That is almost never what you want.

With `STRIP_OUTER_ARRAY = TRUE`, Snowflake strips the outer `[...]` and loads **each object as a separate row** — 3 rows in this example.

```sql
CREATE FILE FORMAT JSON_FORMAT
  TYPE              = 'JSON'
  STRIP_OUTER_ARRAY = TRUE;   -- each array element = one row
```

**When NOT to use it:** If the file contains one JSON object per line (newline-delimited JSON / NDJSON), there is no outer array — `STRIP_OUTER_ARRAY = FALSE` (default) is correct.

---

**Q19 — Explain `LATERAL FLATTEN`. What problem does it solve?**

**Problem:** SQL is row-oriented — one row in, one row out. But JSON arrays contain multiple values in a single field. If an order has 5 items in a JSON array, you have 1 row but need 5 rows to work with the items individually.

**LATERAL FLATTEN** explodes a JSON array into multiple rows — one row per array element. It is Snowflake's equivalent of SQL `UNNEST` or Spark's `EXPLODE`.

```sql
-- Without FLATTEN: 1 row per order, items array is opaque
SELECT raw_event:order_id, raw_event:items FROM ORDER_EVENTS_RAW;
-- order 1001 | [{"sku":"WH-001","qty":1},{"sku":"RS-002","qty":2}]

-- With FLATTEN: 1 row per item
SELECT
    o.raw_event:order_id::NUMBER   AS order_id,
    f.value:sku::VARCHAR           AS sku,
    f.value:qty::NUMBER            AS qty
FROM ORDER_EVENTS_RAW o,
LATERAL FLATTEN(INPUT => o.raw_event:items) f;
-- order 1001 | WH-001 | 1
-- order 1001 | RS-002 | 2
```

The `f.value` reference accesses each element of the array in turn. `LATERAL` means the FLATTEN can reference the row from the outer table (`o`) — it's correlated.

---

**Q20 — Write a query: one row per event, columns: user_id, event_type, event_timestamp.**

```sql
-- Assuming the JSON is in a table called USER_EVENTS_RAW with column raw_event (VARIANT)

SELECT
    e.raw_event:user_id::NUMBER         AS user_id,
    f.value:type::VARCHAR               AS event_type,
    f.value:ts::TIMESTAMP_NTZ           AS event_timestamp
FROM USER_EVENTS_RAW e,
LATERAL FLATTEN(INPUT => e.raw_event:events) f
ORDER BY user_id, event_timestamp;
```

Output:
```
user_id | event_type | event_timestamp
--------|------------|-------------------
101     | click      | 2024-01-01 10:00:00
101     | purchase   | 2024-01-01 10:05:00
```

---

**Q21 — What does `TYPEOF` return? Three possible values.**

`TYPEOF` returns a VARCHAR string describing the data type of the value stored inside a VARIANT at runtime:

| Return Value | Meaning |
|---|---|
| `'INTEGER'` | Whole number (no decimal) |
| `'REAL'` | Floating-point number |
| `'TEXT'` | String value |
| `'BOOLEAN'` | true or false |
| `'ARRAY'` | JSON array `[...]` |
| `'OBJECT'` | JSON object `{...}` |
| `'NULL_VALUE'` | Explicit JSON null |

Three example return values: `'TEXT'`, `'INTEGER'`, `'ARRAY'`.

---

**Q22 — `raw_event:customer:email::VARCHAR` returns NULL. Three possible reasons.**

1. **The key `email` does not exist** in that row's JSON object. When a key is missing, Snowflake returns NULL rather than an error. The row's JSON might be `{"customer": {"id": 42, "name": "Alice"}}` — no email key.

2. **The JSON value is explicitly null** — `{"customer": {"email": null}}`. JSON null is stored as VARIANT null, which casts to SQL NULL.

3. **The key path is wrong** — a typo or case mismatch. VARIANT key access is **case-sensitive**. If the JSON has `"Email"` (capital E) but you query `raw_event:customer:email`, you get NULL. The correct query would be `raw_event:customer:Email::VARCHAR`.

---

**Q23 — What is `OBJECT_KEYS`? What is `ARRAY_SIZE`? Use case for each.**

**`OBJECT_KEYS(variant_object)`**
Returns an ARRAY of all the top-level key names in a JSON object.

```sql
SELECT OBJECT_KEYS(raw_event) FROM ORDER_EVENTS_RAW LIMIT 1;
-- Returns: ["customer","items","order_date","order_id","status","total"]
```

**Use case:** Schema discovery on unknown JSON — when a new data source arrives and you don't know what keys the JSON contains. Run OBJECT_KEYS on a sample to understand the structure before building the extraction query.

---

**`ARRAY_SIZE(variant_array)`**
Returns the number of elements in a JSON array.

```sql
SELECT raw_event:order_id::NUMBER, ARRAY_SIZE(raw_event:items) AS item_count
FROM ORDER_EVENTS_RAW;
```

**Use case:** Data quality validation — checking that multi-item orders have the expected number of line items. If `ARRAY_SIZE(raw_event:items) = 0` for any order, that is a data quality issue to flag.

---

**Q24 — Load Parquet files from a stage.**

```sql
-- Step 1: Create the file format
CREATE FILE FORMAT IF NOT EXISTS PARQUET_FORMAT
  TYPE              = 'PARQUET'
  SNAPPY_COMPRESSION = TRUE;

-- Step 2: Create the stage (pointing to where Parquet files are)
CREATE STAGE IF NOT EXISTS PARQUET_STAGE
  URL         = 'azure://mystorageaccount.blob.core.windows.net/parquet-data/'
  CREDENTIALS = (AZURE_SAS_TOKEN = '<token>');

-- Step 3: Create a target table with VARIANT column (Parquet schema is embedded)
CREATE TABLE IF NOT EXISTS PRODUCTS_PARQUET_RAW (
    raw_data   VARIANT,
    _loaded_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    _source_file VARCHAR(500)
);

-- Step 4: COPY INTO — Parquet loads into VARIANT automatically
COPY INTO PRODUCTS_PARQUET_RAW (raw_data, _source_file)
FROM (
  SELECT $1, METADATA$FILENAME
  FROM @PARQUET_STAGE
)
FILE_FORMAT = (FORMAT_NAME = 'PARQUET_FORMAT')
ON_ERROR    = 'ABORT_STATEMENT';
```

After loading, query Parquet data the same way as JSON using colon notation: `raw_data:product_name::VARCHAR`.

---

## Mixed / Senior-Level Questions

**Q25 — 500 files nightly, 2–3 have issues. Load good files, capture bad ones, alert team.**

**Strategy:**

```sql
-- Step 1: Use SKIP_FILE to skip bad files, load good ones
COPY INTO ECOMM_DB.RAW_DATA.ORDERS_RAW
FROM @ECOMM_DB.RAW_DATA.ECOMM_AZURE_STAGE
FILE_FORMAT = (FORMAT_NAME = 'ECOMM_DB.RAW_DATA.CSV_FORMAT')
ON_ERROR    = 'SKIP_FILE_5%'   -- skip file if more than 5% of rows fail
PURGE       = FALSE;           -- keep files in Azure (don't delete after load)

-- Step 2: Immediately after, query what failed
SELECT
    file_name,
    first_error_message,
    rows_parsed,
    rows_loaded,
    rows_parsed - rows_loaded AS rows_failed,
    status
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'ORDERS_RAW',
    START_TIME => DATEADD('minutes', -30, CURRENT_TIMESTAMP())
))
WHERE status = 'LOAD_FAILED' OR rows_parsed != rows_loaded
ORDER BY last_load_time DESC;
```

**Step 3: Capture failed file names into a table for reprocessing:**

```sql
CREATE TRANSIENT TABLE IF NOT EXISTS ETL_FAILED_FILES (
    file_name         VARCHAR,
    error_message     VARCHAR,
    detected_at       TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

INSERT INTO ETL_FAILED_FILES (file_name, error_message)
SELECT file_name, first_error_message
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(TABLE_NAME => 'ORDERS_RAW', START_TIME => DATEADD('minutes', -30, CURRENT_TIMESTAMP())))
WHERE status = 'LOAD_FAILED';
```

**Step 4: Alert** — trigger from Airflow (email/Slack operator) or Snowflake Alert object if `ETL_FAILED_FILES` has rows after the load step.

**Step 5: Retry** — After fixing the bad files, reload using `FORCE = TRUE` on only those files:
```sql
COPY INTO ORDERS_RAW FROM @ECOMM_AZURE_STAGE
FILES       = ('bad_file_1.csv', 'bad_file_2.csv')   -- specific files
FILE_FORMAT = (FORMAT_NAME = 'CSV_FORMAT')
FORCE       = TRUE;   -- override "already loaded" history
```

---

**Q26 — User Stage vs Table Stage vs Named Internal Stage for production ETL?**

**Choose Named Internal Stage for production ETL.** Here is why:

| Stage | Why not for production |
|---|---|
| **User Stage (`@~`)** | Tied to a specific user — if that user is deleted or their session expires, the stage is inaccessible. Cannot be shared between pipeline steps or teams. |
| **Table Stage (`@%TABLE`)** | Tied to one specific table — if the table is renamed or dropped, the stage reference breaks. Cannot easily load the same files into multiple tables. |
| **Named Internal Stage** | Fully independent object. Can be granted to any role. Can be used in multiple COPY INTO statements for different tables. Has its own file format and encryption settings. Can be monitored with `LIST`. Survives user/table changes. |

```sql
-- Production pattern: named stage + named file format
CREATE STAGE PROD_ORDERS_STAGE ENCRYPTION = (TYPE='SNOWFLAKE_SSE');
-- Grant to the ETL service account role
GRANT READ ON STAGE PROD_ORDERS_STAGE TO ROLE DATA_ENGINEER;
```

---

**Q27 — JSON where `order_total` is sometimes number, sometimes string, sometimes missing.**

**How Snowflake stores it in VARIANT:**

VARIANT accepts any JSON value regardless of type inconsistency between rows. Each row stores whatever the JSON contains:
- Row 1: `order_total` = `250.00` → stored as REAL inside VARIANT
- Row 2: `order_total` = `"250.00"` → stored as TEXT inside VARIANT
- Row 3: `order_total` missing → returns NULL when queried

**How to safely extract as NUMBER:**

```sql
SELECT
    raw_event:order_id::NUMBER                AS order_id,
    -- TRY_CAST handles both numeric and string values without erroring
    TRY_CAST(raw_event:order_total::VARCHAR AS NUMBER(12,2)) AS order_total
FROM ORDER_EVENTS_RAW;
```

`TRY_CAST` (or `TRY_TO_NUMBER`) returns NULL instead of erroring when the cast fails. This handles:
- Numeric `250.00` → VARCHAR `"250.00"` → NUMBER `250.00` ✓
- String `"250.00"` → VARCHAR `"250.00"` → NUMBER `250.00` ✓
- Missing key → NULL → TRY_CAST(NULL) = NULL ✓
- Invalid string `"N/A"` → TRY_CAST fails gracefully → NULL (not an error) ✓

---

**Q28 — Loaded with `ON_ERROR = 'CONTINUE'`. How to see which rows failed?**

```sql
-- Method 1: COPY_HISTORY — file-level errors
SELECT file_name, first_error_message, rows_parsed, rows_loaded,
       rows_parsed - rows_loaded AS failed_rows
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'ORDERS_RAW',
    START_TIME => DATEADD('hours', -1, CURRENT_TIMESTAMP())
));

-- Method 2: Use VALIDATE() function — returns rejected rows
-- (Must be run referencing the specific COPY INTO job ID)
SELECT * FROM TABLE(VALIDATE(ORDERS_RAW, JOB_ID => '<copy_job_id>'));
```

To get the COPY job ID: `SHOW COPY HISTORY IN TABLE ORDERS_RAW` — capture the `JOB_ID` column from the most recent entry and pass it to `VALIDATE()`.

The `VALIDATE()` function returns: rejected row content, error message, file name, row number — everything needed to understand and fix the bad rows.

---

**Q29 — `COPY INTO` vs `INSERT INTO ... SELECT` from a stage query. When to use each?**

| Approach | Characteristics | Use When |
|---|---|---|
| **COPY INTO** | Parallel bulk load. Load history tracking (idempotent). `ON_ERROR` modes. File-level operation. Very fast for large files. | Production ETL loading raw files. 1000s of rows to billions of rows. Need idempotency and error tracking. |
| **INSERT INTO ... SELECT** | Runs like a SQL query. No load history. No file-level tracking. Slower for large volumes. | Small row counts. Loading from one Snowflake table to another. Applying complex multi-step SQL transformations. When you need full SQL flexibility (window functions, CTEs, complex JOINs). |

**Example where INSERT wins:**
```sql
-- Transforming and loading in one step between Snowflake tables — use INSERT
INSERT INTO MARTS.SALES_MART
SELECT o.order_id, c.full_name, o.total_amount * fx.rate AS amount_usd
FROM STAGING.STG_ORDERS o
JOIN STAGING.STG_CUSTOMERS c ON o.customer_id = c.customer_id
JOIN REFERENCE.FX_RATES fx ON o.currency = fx.currency AND o.order_date = fx.rate_date;
```

This can't use COPY INTO — it's reading from other Snowflake tables, not files in a stage.

---

**Q30 (System design) — E-commerce ingestion pipeline: CSV daily, JSON every 5 min, Parquet weekly.**

**Source A — Customer CSV (daily from Azure Blob):**

```sql
-- Stage
CREATE STAGE CUSTOMER_CSV_STAGE
  URL         = 'azure://account.blob.core.windows.net/customers/'
  CREDENTIALS = (AZURE_SAS_TOKEN = '<token>');

-- File format
CREATE FILE FORMAT CUSTOMER_CSV_FMT
  TYPE='CSV' SKIP_HEADER=1 FIELD_DELIMITER=',' FIELD_OPTIONALLY_ENCLOSED_BY='"'
  NULL_IF=('NULL','') EMPTY_FIELD_AS_NULL=TRUE DATE_FORMAT='YYYY-MM-DD';

-- Target: PERMANENT table (customer data is precious, full Time Travel)
CREATE TABLE RAW_DATA.CUSTOMERS_RAW (...) DATA_RETENTION_TIME_IN_DAYS = 7;

-- COPY INTO: run once daily via Airflow, ABORT on error (all-or-nothing)
COPY INTO RAW_DATA.CUSTOMERS_RAW
FROM (SELECT $1,$2,$3,$4,$5,$6,$7,$8, METADATA$FILENAME FROM @CUSTOMER_CSV_STAGE)
FILE_FORMAT = (FORMAT_NAME='CUSTOMER_CSV_FMT')
ON_ERROR    = 'ABORT_STATEMENT'
PATTERN     = '.*customers_[0-9]{8}\.csv\.gz';
```

**Source B — Order Events JSON (every 5 min from Azure Blob):**

```sql
-- Stage
CREATE STAGE ORDER_EVENTS_STAGE
  URL         = 'azure://account.blob.core.windows.net/order-events/'
  CREDENTIALS = (AZURE_SAS_TOKEN = '<token>');

-- File format: JSON array of events per file
CREATE FILE FORMAT ORDER_EVENTS_FMT
  TYPE='JSON' STRIP_OUTER_ARRAY=TRUE COMPRESSION='AUTO';

-- Target: PERMANENT table with VARIANT column (raw JSON preserved)
CREATE TABLE RAW_DATA.ORDER_EVENTS_RAW (
    raw_event    VARIANT,
    _loaded_at   TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    _source_file VARCHAR(500)
);

-- COPY INTO: run every 5 min via Airflow, CONTINUE (losing a few events acceptable)
COPY INTO RAW_DATA.ORDER_EVENTS_RAW (raw_event, _source_file)
FROM (SELECT $1, METADATA$FILENAME FROM @ORDER_EVENTS_STAGE)
FILE_FORMAT = (FORMAT_NAME='ORDER_EVENTS_FMT')
ON_ERROR    = 'SKIP_FILE_10%';   -- skip file only if >10% of records fail
-- Files automatically deduplicated by Snowflake load history (idempotent)
```

**Source C — Product Catalog Parquet (weekly):**

```sql
-- Stage
CREATE STAGE PRODUCT_PARQUET_STAGE
  URL         = 'azure://account.blob.core.windows.net/product-catalog/'
  CREDENTIALS = (AZURE_SAS_TOKEN = '<token>');

-- File format
CREATE FILE FORMAT PRODUCT_PARQUET_FMT
  TYPE='PARQUET' SNAPPY_COMPRESSION=TRUE;

-- Target: PERMANENT table (product master data — full recovery needed)
CREATE TABLE RAW_DATA.PRODUCTS_PARQUET_RAW (
    raw_data     VARIANT,
    _loaded_at   TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    _source_file VARCHAR(500)
);

-- COPY INTO: run once weekly, ABORT (product catalog must be complete)
COPY INTO RAW_DATA.PRODUCTS_PARQUET_RAW (raw_data, _source_file)
FROM (SELECT $1, METADATA$FILENAME FROM @PRODUCT_PARQUET_STAGE)
FILE_FORMAT = (FORMAT_NAME='PRODUCT_PARQUET_FMT')
ON_ERROR    = 'ABORT_STATEMENT';
```

**Error handling strategy across all three:**
- All use `METADATA$FILENAME` for lineage
- Failed loads write to `ETL_FAILED_FILES` table (monitored by Airflow)
- Airflow retries failed files after team investigation
- Load history prevents re-loading on retry (unless FORCE=TRUE is set after fix)
