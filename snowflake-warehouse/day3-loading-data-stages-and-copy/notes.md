# Day 3 — Loading Data: Stages, File Formats, and COPY INTO

## Overview

Day 2 gave you the full object model — warehouses, databases, schemas, tables, roles, and views. Today you learn how to actually get data into Snowflake. This is the most critical skill for a data engineer: without loading, there is no data to transform or analyse.

**Everything in Day 3 follows this order:**

```
Step 1 → Understand Stages         (where files live before loading)
Step 2 → Create File Formats       (tell Snowflake how to parse the files)
Step 3 → Load with COPY INTO       (move data from stage into a table)
Step 4 → Understand VARIANT        (loading JSON and semi-structured data)
Step 5 → Monitor and troubleshoot  (query load history, handle errors)
```

**Three core concepts today:**

1. **Stages and File Formats** — Internal vs external stages, all stage types, creating file formats for CSV, JSON, Parquet
2. **COPY INTO — All Approaches** — Loading from internal stage, external stage, S3/Azure Blob, with all options and error handling modes
3. **Semi-Structured Data and VARIANT** — Loading JSON into VARIANT columns, querying nested data with colon syntax and FLATTEN

---

## Concept 1: Stages and File Formats

### What is a Stage?

A **stage** is a named pointer to a file storage location. It tells Snowflake where to find files before loading them into tables — or where to write files when unloading data.

Think of a stage as a **landing zone** between your source files and your Snowflake tables:

```
Your Files                   Stage                    Snowflake Table
(CSV, JSON,     ──PUT──►  (named pointer      ──COPY INTO──►  CUSTOMERS_RAW
 Parquet,                  to file location)
 Avro, ORC)
```

Stages do NOT store data themselves — they are just pointers (or in the case of internal stages, a managed storage area inside Snowflake's own cloud storage).

### Stage Types

```
┌───────────────────────────────────────────────────────────────────────┐
│                         STAGE TYPES                                   │
│                                                                       │
│  INTERNAL STAGES  (Snowflake manages the storage)                     │
│  ├── User Stage       — one per user, automatic, no CREATE needed     │
│  ├── Table Stage      — one per table, automatic, no CREATE needed    │
│  └── Named Stage      — you create it, shared, most flexible ✓        │
│                                                                       │
│  EXTERNAL STAGES  (you manage the storage in your cloud account)      │
│  ├── Amazon S3        — s3://your-bucket/your-prefix/                 │
│  ├── Azure Blob       — azure://your-account.blob.core.windows.net/   │
│  └── Google GCS       — gcs://your-bucket/your-prefix/               │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

| Stage Type | Storage Managed By | Requires CREATE? | Use Case |
|---|---|---|---|
| **User Stage** (`@~`) | Snowflake | No — exists automatically | Personal ad hoc file uploads |
| **Table Stage** (`@%TABLE_NAME`) | Snowflake | No — one per table automatically | Loading files specifically for one table |
| **Named Internal Stage** | Snowflake | Yes — `CREATE STAGE` | Shared pipelines, production ETL |
| **Named External Stage (S3)** | AWS | Yes — `CREATE STAGE` | Files already in S3, no need to move |
| **Named External Stage (Azure Blob)** | Azure | Yes — `CREATE STAGE` | Files in Azure Blob Storage (matches our course setup) |
| **Named External Stage (GCS)** | Google | Yes — `CREATE STAGE` | Files in GCS |

### Creating an Internal Named Stage

An **internal named stage** stores files inside Snowflake's own managed cloud storage. Use it when you have files on your local machine or a server that you want to push into Snowflake using the `PUT` command.

```sql
USE ROLE SYSADMIN;
USE WAREHOUSE LEARNING_WH;
USE DATABASE ECOMM_DB;
USE SCHEMA RAW_DATA;

-- Minimal syntax
CREATE STAGE IF NOT EXISTS my_internal_stage;

-- Full syntax with encryption and directory table enabled
CREATE STAGE IF NOT EXISTS ECOMM_INTERNAL_STAGE
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')   -- server-side encryption (default)
  DIRECTORY  = (ENABLE = TRUE)            -- enables listing files in the stage
  COMMENT    = 'Internal stage for loading e-commerce CSV files';
```

**After creating, verify it exists:**

```sql
SHOW STAGES IN SCHEMA ECOMM_DB.RAW_DATA;

-- List files currently in the stage (empty until you PUT something)
LIST @ECOMM_DB.RAW_DATA.ECOMM_INTERNAL_STAGE;
```

### Uploading Files to an Internal Stage (PUT command)

`PUT` is a **SnowSQL CLI** command — it runs from your local terminal, not from a Snowsight worksheet. It uploads local files into an internal stage.

```bash
# Install SnowSQL first (Snowflake's CLI tool):
# https://docs.snowflake.com/en/user-guide/snowsql-install-config

# Connect via SnowSQL
snowsql -a <your-account-identifier> -u <your-username>

# Upload a single file
PUT file:///C:/data/customers.csv @ECOMM_DB.RAW_DATA.ECOMM_INTERNAL_STAGE;

# Upload all CSV files from a folder
PUT file:///C:/data/*.csv @ECOMM_DB.RAW_DATA.ECOMM_INTERNAL_STAGE;

# Upload with auto-compression disabled (file already compressed)
PUT file:///C:/data/orders.csv.gz @ECOMM_DB.RAW_DATA.ECOMM_INTERNAL_STAGE
  AUTO_COMPRESS = FALSE;
```

**After PUT, list what arrived:**

```sql
LIST @ECOMM_DB.RAW_DATA.ECOMM_INTERNAL_STAGE;
-- Shows: name, size, md5, last_modified
```

### Creating an External Stage (Azure Blob — matches our course)

An **external stage** points to a cloud storage location you own. Files stay in your Azure Blob container — Snowflake reads them directly from there during COPY INTO.

```sql
-- External stage pointing to Azure Blob Storage
-- (replace with your actual storage account details)
CREATE STAGE IF NOT EXISTS ECOMM_AZURE_STAGE
  URL                = 'azure://mystorageaccount.blob.core.windows.net/ecomm-data/'
  CREDENTIALS        = (AZURE_SAS_TOKEN = '<your-sas-token>')
  FILE_FORMAT        = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"')
  COMMENT            = 'External stage pointing to Azure Blob ecomm-data container';
```

**For learning purposes — using Snowflake sample data as a stage:**

Since setting up Azure Blob requires a real storage account, we will use Snowflake's built-in sample data files for exercises today:

```sql
-- Snowflake provides sample files at this internal stage (read-only, pre-loaded)
LIST @SNOWFLAKE_SAMPLE_DATA.INFORMATION_SCHEMA.LOAD_HISTORY;

-- Or use the tutorial stage Snowflake provides
-- @SNOWFLAKE_TUTORIAL_DATA (available in some accounts)
```

### File Formats

A **file format** tells Snowflake how to parse a file — delimiter, quote character, header row, null handling, compression, date formats, etc.

You can define a file format:
1. **Inline in COPY INTO** — one-time use, not reusable
2. **As a named object** — reusable across multiple COPY INTO statements (recommended for production)

#### Creating Named File Formats

```sql
USE SCHEMA ECOMM_DB.RAW_DATA;

-- ── CSV File Format ────────────────────────────────────────────────────
CREATE FILE FORMAT IF NOT EXISTS CSV_FORMAT
  TYPE                      = 'CSV'
  FIELD_DELIMITER           = ','          -- column separator
  RECORD_DELIMITER          = '\n'         -- row separator
  SKIP_HEADER               = 1            -- skip the first row (column names)
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'       -- values may be quoted with "
  NULL_IF                   = ('NULL', 'null', 'N/A', '', '\\N')  -- treat these as NULL
  EMPTY_FIELD_AS_NULL       = TRUE         -- empty string → NULL
  TRIM_SPACE                = TRUE         -- strip leading/trailing spaces
  DATE_FORMAT               = 'YYYY-MM-DD'
  TIMESTAMP_FORMAT          = 'YYYY-MM-DD HH24:MI:SS'
  COMPRESSION               = 'AUTO'       -- auto-detect gzip, bz2, etc.
  COMMENT                   = 'Standard CSV format for e-commerce data files';

-- ── JSON File Format ───────────────────────────────────────────────────
CREATE FILE FORMAT IF NOT EXISTS JSON_FORMAT
  TYPE                = 'JSON'
  STRIP_OUTER_ARRAY   = TRUE     -- if file is [{...},{...}], strip the outer []
  STRIP_NULL_VALUES   = FALSE    -- keep null values (don't silently drop them)
  IGNORE_UTF8_ERRORS  = FALSE    -- fail loudly on bad encoding
  COMPRESSION         = 'AUTO'
  COMMENT             = 'JSON format for event and API data';

-- ── Parquet File Format ────────────────────────────────────────────────
CREATE FILE FORMAT IF NOT EXISTS PARQUET_FORMAT
  TYPE              = 'PARQUET'
  SNAPPY_COMPRESSION = TRUE      -- Parquet files are often Snappy-compressed
  COMMENT           = 'Parquet format for columnar data files';

-- View all file formats
SHOW FILE FORMATS IN SCHEMA ECOMM_DB.RAW_DATA;
```

**All supported file types in Snowflake:**

| Format | TYPE value | Notes |
|---|---|---|
| CSV / TSV / PSV | `CSV` | Most common — configure delimiter accordingly |
| JSON | `JSON` | Load into VARIANT columns |
| Avro | `AVRO` | Schema embedded in file |
| ORC | `ORC` | Hadoop columnar format |
| Parquet | `PARQUET` | Columnar, widely used in data lakes |
| XML | `XML` | Loads into VARIANT |

---

## Concept 2: COPY INTO — All Approaches

### What is COPY INTO?

`COPY INTO` is Snowflake's bulk loading command. It reads files from a stage and loads them into a table. It is:
- **Batch-oriented** — designed for loading thousands to billions of rows at once
- **Idempotent** — by default, Snowflake tracks which files have already been loaded and skips them (load history is kept for 64 days)
- **Parallel** — Snowflake splits large files and loads them in parallel across warehouse nodes

### Approach 1: COPY INTO from Internal Stage (Most Common for Learning)

First, let's create sample CSV data to work with. Since we cannot PUT from a worksheet, we'll use a VALUES approach to stage data via a temporary table, then demonstrate COPY INTO against Snowflake's own sample files.

**Using Snowflake's tutorial sample files (no file upload needed):**

```sql
USE ROLE SYSADMIN;
USE WAREHOUSE LEARNING_WH;
USE DATABASE ECOMM_DB;
USE SCHEMA RAW_DATA;

-- Snowflake provides sample CSV data at this path in every account
-- Let's use the TPCH data files directly

-- Step 1: Create a stage pointing to Snowflake's sample data
CREATE STAGE IF NOT EXISTS SNOWFLAKE_SAMPLE_STAGE
  URL = 's3://snowflake-workshop-lab/citibike-trips-csv/'
  FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 0);

LIST @SNOWFLAKE_SAMPLE_STAGE;
```

**Standard COPY INTO from a named stage:**

```sql
-- Full COPY INTO syntax
COPY INTO ECOMM_DB.RAW_DATA.CUSTOMERS_RAW
FROM @ECOMM_DB.RAW_DATA.ECOMM_INTERNAL_STAGE/customers.csv
FILE_FORMAT = (FORMAT_NAME = 'ECOMM_DB.RAW_DATA.CSV_FORMAT')
ON_ERROR    = 'ABORT_STATEMENT';   -- stop entire load if any error
```

### Approach 2: COPY INTO with Inline File Format (No Named Format)

When you do not have a named file format, define the format inline:

```sql
COPY INTO ECOMM_DB.RAW_DATA.ORDERS_RAW
FROM @ECOMM_DB.RAW_DATA.ECOMM_INTERNAL_STAGE
FILES = ('orders_2024_01.csv', 'orders_2024_02.csv')  -- specific files only
FILE_FORMAT = (
  TYPE                     = 'CSV'
  SKIP_HEADER              = 1
  FIELD_DELIMITER          = ','
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  NULL_IF                  = ('NULL', '')
  EMPTY_FIELD_AS_NULL      = TRUE
)
ON_ERROR = 'CONTINUE';   -- skip bad rows, keep going
```

### Approach 3: COPY INTO with PATTERN (Load Multiple Files by Name Pattern)

Load all files matching a regex pattern — useful when files arrive with timestamps in their names:

```sql
-- Load all CSV files whose names start with "customers_"
COPY INTO ECOMM_DB.RAW_DATA.CUSTOMERS_RAW
FROM @ECOMM_DB.RAW_DATA.ECOMM_INTERNAL_STAGE
PATTERN     = '.*customers_.*\.csv'   -- regex pattern to match filenames
FILE_FORMAT = (FORMAT_NAME = 'ECOMM_DB.RAW_DATA.CSV_FORMAT')
ON_ERROR    = 'SKIP_FILE';
```

### Approach 4: COPY INTO from External Stage (Azure Blob)

```sql
-- Load directly from Azure Blob — no PUT required
COPY INTO ECOMM_DB.RAW_DATA.CUSTOMERS_RAW
FROM @ECOMM_DB.RAW_DATA.ECOMM_AZURE_STAGE/raw/customers/
PATTERN     = '.*\.csv\.gz'
FILE_FORMAT = (FORMAT_NAME = 'ECOMM_DB.RAW_DATA.CSV_FORMAT')
ON_ERROR    = 'ABORT_STATEMENT'
PURGE       = FALSE;   -- keep source files in Azure after loading (default)
```

### Approach 5: COPY INTO with Column Mapping (File columns ≠ Table columns)

When the CSV column order does not match the table column order, or the file has extra columns you don't want:

```sql
-- Map specific CSV columns to specific table columns
COPY INTO ECOMM_DB.RAW_DATA.CUSTOMERS_RAW
  (customer_id, first_name, last_name, email, country, signup_date)
FROM (
  SELECT
    $1,   -- column 1 in the file → customer_id
    $2,   -- column 2 → first_name
    $3,   -- column 3 → last_name
    $5,   -- column 5 → email  (skipping column 4)
    $7,   -- column 7 → country
    TRY_TO_DATE($8, 'MM/DD/YYYY')   -- column 8 with date conversion → signup_date
  FROM @ECOMM_DB.RAW_DATA.ECOMM_INTERNAL_STAGE/customers.csv
)
FILE_FORMAT = (FORMAT_NAME = 'ECOMM_DB.RAW_DATA.CSV_FORMAT');
```

`$1`, `$2`, `$3`... are **positional column references** in Snowflake — they refer to the Nth column in the source file.

### Approach 6: COPY INTO with Transformation (Clean During Load)

You can apply SQL transformations inline during the load — no staging table needed:

```sql
COPY INTO ECOMM_DB.RAW_DATA.CUSTOMERS_RAW
  (customer_id, first_name, last_name, email, country, city, signup_date, loyalty_tier, is_active, _source_file)
FROM (
  SELECT
    $1::NUMBER,
    TRIM(UPPER($2)),               -- clean: trim spaces, uppercase first name
    TRIM(UPPER($3)),               -- clean: trim spaces, uppercase last name
    LOWER($4),                     -- clean: lowercase email
    INITCAP($5),                   -- clean: proper case country
    INITCAP($6),                   -- clean: proper case city
    TRY_TO_DATE($7, 'YYYY-MM-DD'),
    UPPER($8),                     -- loyalty tier always uppercase
    ($9 = 'Y' OR $9 = 'TRUE' OR $9 = '1')::BOOLEAN,  -- normalize boolean
    METADATA$FILENAME              -- built-in metadata column — name of source file
  FROM @ECOMM_DB.RAW_DATA.ECOMM_INTERNAL_STAGE
)
FILE_FORMAT = (FORMAT_NAME = 'ECOMM_DB.RAW_DATA.CSV_FORMAT');
```

**Useful metadata columns available during COPY INTO:**

| Metadata Column | What it Contains |
|---|---|
| `METADATA$FILENAME` | Full path and name of the source file |
| `METADATA$FILE_ROW_NUMBER` | Row number within the file |
| `METADATA$FILE_LAST_MODIFIED` | Last modified timestamp of the file |
| `METADATA$START_SCAN_TIME` | When this COPY INTO started scanning |

### ON_ERROR Options — Handling Bad Rows

| Option | Behaviour | Use When |
|---|---|---|
| `ABORT_STATEMENT` | Stop entire load on first error. No rows committed. | Production — you want all-or-nothing loading |
| `CONTINUE` | Skip bad rows, load everything else. Report errors. | Acceptable to lose a few rows, want maximum throughput |
| `SKIP_FILE` | Skip the entire file if it has any error, load other files | File-level granularity control |
| `SKIP_FILE_<N>` | Skip file only if it has more than N errors | e.g., `SKIP_FILE_10` — tolerate up to 10 errors per file |
| `SKIP_FILE_<N>%` | Skip file if more than N% of rows error | e.g., `SKIP_FILE_5%` — tolerate up to 5% error rate |

```sql
-- Example: tolerate up to 5% bad rows per file, abort if more
COPY INTO ECOMM_DB.RAW_DATA.ORDERS_RAW
FROM @ECOMM_DB.RAW_DATA.ECOMM_INTERNAL_STAGE
FILE_FORMAT = (FORMAT_NAME = 'ECOMM_DB.RAW_DATA.CSV_FORMAT')
ON_ERROR    = 'SKIP_FILE_5%';
```

### Checking Load History

Snowflake remembers every file it has loaded (for 64 days) and **skips already-loaded files** automatically on subsequent COPY INTO runs. This makes COPY INTO idempotent — safe to run repeatedly.

```sql
-- View load history for a table (last 14 days by default)
SELECT *
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
  TABLE_NAME    => 'CUSTOMERS_RAW',
  START_TIME    => DATEADD('hours', -24, CURRENT_TIMESTAMP())
));

-- View load history via account usage (longer history, slight delay)
SELECT
    table_name,
    file_name,
    row_count,
    row_parsed,
    first_error_message,
    status,
    last_load_time
FROM SNOWFLAKE.ACCOUNT_USAGE.LOAD_HISTORY
WHERE table_name = 'CUSTOMERS_RAW'
  AND last_load_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY last_load_time DESC;
```

**Force re-loading a file that was already loaded** (override the load history):

```sql
COPY INTO ECOMM_DB.RAW_DATA.CUSTOMERS_RAW
FROM @ECOMM_DB.RAW_DATA.ECOMM_INTERNAL_STAGE/customers.csv
FILE_FORMAT = (FORMAT_NAME = 'ECOMM_DB.RAW_DATA.CSV_FORMAT')
FORCE = TRUE;   -- bypass load history, reload even if file was loaded before
```

### Validating Before Loading (VALIDATION_MODE)

Before committing data to a table, check for errors without actually loading:

```sql
-- Scan files and return all rows that would fail — no data loaded
COPY INTO ECOMM_DB.RAW_DATA.CUSTOMERS_RAW
FROM @ECOMM_DB.RAW_DATA.ECOMM_INTERNAL_STAGE
FILE_FORMAT      = (FORMAT_NAME = 'ECOMM_DB.RAW_DATA.CSV_FORMAT')
VALIDATION_MODE  = 'RETURN_ERRORS';    -- show only the rows that would fail

-- Return first N rows as a preview — no data loaded
COPY INTO ECOMM_DB.RAW_DATA.CUSTOMERS_RAW
FROM @ECOMM_DB.RAW_DATA.ECOMM_INTERNAL_STAGE
FILE_FORMAT      = (FORMAT_NAME = 'ECOMM_DB.RAW_DATA.CSV_FORMAT')
VALIDATION_MODE  = 'RETURN_10_ROWS';   -- preview first 10 rows
```

---

## Concept 3: Semi-Structured Data and VARIANT

### What is Semi-Structured Data?

Most real-world data sources don't send you clean flat CSVs. APIs return JSON, Kafka emits nested events, mobile apps log JSON arrays, and data lakes store Parquet files with nested structs. Snowflake handles all of these natively through the **VARIANT** data type.

**VARIANT** is a column that can hold any JSON, XML, Avro, or Parquet value — including nested objects, arrays, and mixed types. No schema definition needed upfront.

```
Flat CSV:                           Nested JSON:
─────────                           ────────────
order_id, customer_id, amount       {
1001,     42,          250.00         "order_id": 1001,
1002,     43,          180.50         "customer": {
                                        "id": 42,
                                        "name": "Alice",
                                        "tier": "GOLD"
                                      },
                                      "items": [
                                        {"sku": "A1", "qty": 2, "price": 49.99},
                                        {"sku": "B3", "qty": 1, "price": 150.00}
                                      ],
                                      "total": 250.00
                                    }
```

### Creating a Table with VARIANT Column

```sql
USE DATABASE ECOMM_DB;
USE SCHEMA RAW_DATA;

-- Table to hold raw JSON events from an e-commerce event stream
CREATE TABLE IF NOT EXISTS ORDER_EVENTS_RAW (
    event_id      NUMBER         AUTOINCREMENT PRIMARY KEY,
    raw_event     VARIANT,                     -- the entire JSON object
    _loaded_at    TIMESTAMP_NTZ  DEFAULT CURRENT_TIMESTAMP(),
    _source_file  VARCHAR(500)
);
```

### Loading JSON into VARIANT

```sql
-- Using PARSE_JSON to insert JSON directly in SQL (for testing)
INSERT INTO ORDER_EVENTS_RAW (raw_event)
SELECT PARSE_JSON(column1)
FROM VALUES
  ('{"order_id": 1001, "customer": {"id": 1, "name": "Alice Johnson", "tier": "GOLD"}, "items": [{"sku": "WH-001", "product": "Wireless Headphones", "qty": 1, "price": 149.99}], "total": 149.99, "status": "DELIVERED", "order_date": "2024-01-10"}'),
  ('{"order_id": 1002, "customer": {"id": 2, "name": "Bob Smith", "tier": "SILVER"}, "items": [{"sku": "RS-002", "product": "Running Shoes", "qty": 2, "price": 89.99}], "total": 179.98, "status": "DELIVERED", "order_date": "2024-01-15"}'),
  ('{"order_id": 1003, "customer": {"id": 3, "name": "Priya Sharma", "tier": "PLATINUM"}, "items": [{"sku": "CM-003", "product": "Coffee Maker", "qty": 1, "price": 129.99}, {"sku": "YM-005", "product": "Yoga Mat", "qty": 2, "price": 39.99}], "total": 209.97, "status": "SHIPPED", "order_date": "2024-01-20"}');
```

**Loading JSON files via COPY INTO:**

```sql
-- Load JSON files from stage into VARIANT column
COPY INTO ECOMM_DB.RAW_DATA.ORDER_EVENTS_RAW (raw_event, _source_file)
FROM (
  SELECT $1, METADATA$FILENAME
  FROM @ECOMM_DB.RAW_DATA.ECOMM_INTERNAL_STAGE/events/
)
FILE_FORMAT = (FORMAT_NAME = 'ECOMM_DB.RAW_DATA.JSON_FORMAT');
```

### Querying VARIANT Data — Colon Syntax

Snowflake uses `:` (colon) to navigate into JSON keys and `::` to cast the extracted value to a SQL type.

```sql
-- Basic extraction using colon notation
SELECT
    raw_event:order_id::NUMBER         AS order_id,
    raw_event:status::VARCHAR          AS status,
    raw_event:order_date::DATE         AS order_date,
    raw_event:total::NUMBER(10,2)      AS total_amount,

    -- Nested object — drill down with more colons
    raw_event:customer:id::NUMBER      AS customer_id,
    raw_event:customer:name::VARCHAR   AS customer_name,
    raw_event:customer:tier::VARCHAR   AS loyalty_tier

FROM ECOMM_DB.RAW_DATA.ORDER_EVENTS_RAW;
```

**Accessing array elements by index:**

```sql
-- Access first item in the items array (index 0)
SELECT
    raw_event:order_id::NUMBER               AS order_id,
    raw_event:items[0]:sku::VARCHAR          AS first_item_sku,
    raw_event:items[0]:product::VARCHAR      AS first_item_name,
    raw_event:items[0]:qty::NUMBER           AS first_item_qty,
    raw_event:items[0]:price::NUMBER(10,2)   AS first_item_price
FROM ECOMM_DB.RAW_DATA.ORDER_EVENTS_RAW;
```

### Flattening Arrays with LATERAL FLATTEN

When a JSON object has an array and you need **one row per array element**, use `LATERAL FLATTEN`. This is the equivalent of a SQL `UNNEST` or `EXPLODE` in Spark.

```sql
-- One row per item in the order's items array
SELECT
    o.raw_event:order_id::NUMBER         AS order_id,
    o.raw_event:customer:name::VARCHAR   AS customer_name,
    o.raw_event:order_date::DATE         AS order_date,
    item.value:sku::VARCHAR              AS sku,
    item.value:product::VARCHAR          AS product_name,
    item.value:qty::NUMBER               AS quantity,
    item.value:price::NUMBER(10,2)       AS unit_price,
    item.value:qty::NUMBER
      * item.value:price::NUMBER(10,2)   AS line_total
FROM ECOMM_DB.RAW_DATA.ORDER_EVENTS_RAW o,
LATERAL FLATTEN(INPUT => o.raw_event:items) item;
```

**Output — each order becomes multiple rows, one per item:**

```
order_id | customer_name  | sku    | product_name        | qty | unit_price | line_total
---------|----------------|--------|---------------------|-----|------------|----------
1001     | Alice Johnson  | WH-001 | Wireless Headphones | 1   | 149.99     | 149.99
1002     | Bob Smith      | RS-002 | Running Shoes       | 2   |  89.99     | 179.98
1003     | Priya Sharma   | CM-003 | Coffee Maker        | 1   | 129.99     | 129.99
1003     | Priya Sharma   | YM-005 | Yoga Mat            | 2   |  39.99     |  79.98
```

### Useful VARIANT Functions

```sql
-- Check if a key exists in a VARIANT object
SELECT
    raw_event:order_id,
    raw_event:discount IS NULL AS has_no_discount  -- NULL if key missing
FROM ECOMM_DB.RAW_DATA.ORDER_EVENTS_RAW;

-- GET_PATH — alternative dot-path notation
SELECT GET_PATH(raw_event, 'customer.name') FROM ORDER_EVENTS_RAW;

-- TYPEOF — what type is a value inside VARIANT?
SELECT TYPEOF(raw_event:order_id) FROM ORDER_EVENTS_RAW;
-- Returns: 'INTEGER', 'REAL', 'TEXT', 'BOOLEAN', 'ARRAY', 'OBJECT', 'NULL'

-- ARRAY_SIZE — how many items in an array?
SELECT
    raw_event:order_id::NUMBER         AS order_id,
    ARRAY_SIZE(raw_event:items)        AS item_count
FROM ECOMM_DB.RAW_DATA.ORDER_EVENTS_RAW;

-- OBJECT_KEYS — list all keys in a JSON object
SELECT OBJECT_KEYS(raw_event) FROM ORDER_EVENTS_RAW LIMIT 1;
-- Returns: ["customer", "items", "order_date", "order_id", "status", "total"]
```

### Flattening VARIANT into a Structured Table (Staging Step)

After loading raw JSON into VARIANT, the staging step extracts it into a proper structured table:

```sql
-- Flatten JSON events into a clean structured staging table
CREATE OR REPLACE TRANSIENT TABLE ECOMM_DB.STAGING.STG_ORDER_EVENTS AS
SELECT
    o.raw_event:order_id::NUMBER(15,0)       AS order_id,
    o.raw_event:customer:id::NUMBER(10,0)    AS customer_id,
    o.raw_event:customer:name::VARCHAR       AS customer_name,
    o.raw_event:customer:tier::VARCHAR       AS loyalty_tier,
    o.raw_event:order_date::DATE             AS order_date,
    o.raw_event:status::VARCHAR              AS status,
    o.raw_event:total::NUMBER(12,2)          AS total_amount,
    ARRAY_SIZE(o.raw_event:items)            AS item_count,
    o._loaded_at
FROM ECOMM_DB.RAW_DATA.ORDER_EVENTS_RAW o;

SELECT * FROM ECOMM_DB.STAGING.STG_ORDER_EVENTS;
```

---

## Summary

| Concept | Key Takeaway |
|---|---|
| **Stages** | Named pointers to file storage. Internal = Snowflake manages the files. External = you manage the files in Azure/S3/GCS. |
| **Internal stage** | Use `PUT` from SnowSQL CLI to upload files. Use `LIST @stage` to see what's there. |
| **External stage** | Points to your cloud storage. Files stay where they are — Snowflake reads them directly. |
| **File formats** | Named objects that define how to parse a file. Create once, reuse across many COPY INTO statements. |
| **COPY INTO** | Bulk loader. Idempotent — tracks loaded files for 64 days. Use `VALIDATION_MODE` to preview before loading. |
| **ON_ERROR** | `ABORT_STATEMENT` for production, `CONTINUE` or `SKIP_FILE_N%` when some bad rows are tolerable. |
| **VARIANT** | Snowflake's native type for JSON/XML/Parquet. No schema needed upfront — query with `:` colon notation. |
| **LATERAL FLATTEN** | Explode JSON arrays into rows — one row per array element. Essential for event data. |
| **Metadata columns** | `METADATA$FILENAME` during COPY INTO captures the source file name — critical for data lineage. |

**Next:** Day 4 covers Time Travel, Fail-safe, Cloning, and data recovery — Snowflake's built-in protection features.
