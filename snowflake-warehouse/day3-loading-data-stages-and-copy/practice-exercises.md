# Day 3 — Practice Exercises: Loading Data, Stages, and COPY INTO

> All exercises are done in Snowsight SQL worksheets unless noted otherwise.
> Complete them in order — each builds on the previous one.
> Estimated total time: 75–100 minutes.

---

## Exercise 1 — Create Stages and File Formats

**Concept:** Stage types, file format creation, LIST command

**Tasks:**

**1a. Set up your session context**
```sql
USE ROLE SYSADMIN;
USE WAREHOUSE LEARNING_WH;
USE DATABASE ECOMM_DB;
USE SCHEMA RAW_DATA;
```

**1b. Create a named internal stage**
```sql
CREATE STAGE IF NOT EXISTS ECOMM_INTERNAL_STAGE
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')
  DIRECTORY  = (ENABLE = TRUE)
  COMMENT    = 'Internal stage for Day 3 exercises';
```

**1c. Verify the stage was created**
```sql
SHOW STAGES IN SCHEMA ECOMM_DB.RAW_DATA;
-- You should see ECOMM_INTERNAL_STAGE in the results

LIST @ECOMM_DB.RAW_DATA.ECOMM_INTERNAL_STAGE;
-- Empty for now — no files uploaded yet
```

**1d. Create named file formats for all three file types**

```sql
-- CSV format
CREATE FILE FORMAT IF NOT EXISTS CSV_FORMAT
  TYPE                        = 'CSV'
  FIELD_DELIMITER             = ','
  RECORD_DELIMITER            = '\n'
  SKIP_HEADER                 = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  NULL_IF                     = ('NULL', 'null', 'N/A', '')
  EMPTY_FIELD_AS_NULL         = TRUE
  TRIM_SPACE                  = TRUE
  DATE_FORMAT                 = 'YYYY-MM-DD'
  COMPRESSION                 = 'AUTO'
  COMMENT                     = 'Standard CSV format';

-- JSON format
CREATE FILE FORMAT IF NOT EXISTS JSON_FORMAT
  TYPE              = 'JSON'
  STRIP_OUTER_ARRAY = TRUE
  STRIP_NULL_VALUES = FALSE
  COMPRESSION       = 'AUTO'
  COMMENT           = 'JSON format for event data';

-- Parquet format
CREATE FILE FORMAT IF NOT EXISTS PARQUET_FORMAT
  TYPE              = 'PARQUET'
  SNAPPY_COMPRESSION = TRUE
  COMMENT           = 'Parquet format for columnar files';
```

**1e. Verify all three file formats**
```sql
SHOW FILE FORMATS IN SCHEMA ECOMM_DB.RAW_DATA;
-- You should see CSV_FORMAT, JSON_FORMAT, PARQUET_FORMAT
```

**Expected outcome:** One internal stage and three file format objects created.

**Discussion:** Why create a JSON format with `STRIP_OUTER_ARRAY = TRUE`? What would happen if your JSON file had one object per line (NDJSON format) and you used that setting?

---

## Exercise 2 — Explore a Public Stage (No File Upload Required)

**Concept:** External stages, LIST, COPY INTO from public data

Snowflake provides a public S3 bucket with sample files. This lets us practice loading without needing to set up Azure credentials.

**Tasks:**

**2a. Create a stage pointing to Snowflake's public sample data bucket**
```sql
USE DATABASE ECOMM_DB;
USE SCHEMA RAW_DATA;

-- Public sample data hosted by Snowflake (no credentials needed — public bucket)
CREATE STAGE IF NOT EXISTS CITIBIKE_STAGE
  URL         = 's3://snowflake-workshop-lab/citibike-trips-csv/'
  FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 0 FIELD_OPTIONALLY_ENCLOSED_BY = '"');
```

**2b. List files in the public stage**
```sql
LIST @CITIBIKE_STAGE;
-- You should see many .csv.gz files
-- Note the file names and sizes
```

**2c. Preview data before loading using VALIDATION_MODE**
```sql
-- Create a simple target table for the citibike data
CREATE TABLE IF NOT EXISTS CITIBIKE_TRIPS_RAW (
    tripduration        INTEGER,
    starttime           TIMESTAMP_NTZ,
    stoptime            TIMESTAMP_NTZ,
    start_station_id    INTEGER,
    start_station_name  VARCHAR(255),
    start_lat           FLOAT,
    start_lon           FLOAT,
    end_station_id      INTEGER,
    end_station_name    VARCHAR(255),
    end_lat             FLOAT,
    end_lon             FLOAT,
    bikeid              INTEGER,
    membership_type     VARCHAR(50),
    birth_year          INTEGER,
    gender              INTEGER,
    _source_file        VARCHAR(500)
);

-- Preview first 10 rows WITHOUT loading any data
COPY INTO ECOMM_DB.RAW_DATA.CITIBIKE_TRIPS_RAW
FROM @CITIBIKE_STAGE
VALIDATION_MODE = 'RETURN_10_ROWS';
```

**Expected outcome:** 10 rows returned as a preview — no data inserted into the table yet.

**Discussion:** `SELECT COUNT(*) FROM CITIBIKE_TRIPS_RAW` should return 0 after validation mode. Why is this useful before a production load? What would you check in those 10 preview rows?

---

## Exercise 3 — Load Data with COPY INTO

**Concept:** COPY INTO, load history, idempotency, ON_ERROR

**Tasks:**

**3a. Load data from the public stage**
```sql
-- Load from the public Citibike stage — column mapping with METADATA$FILENAME
COPY INTO ECOMM_DB.RAW_DATA.CITIBIKE_TRIPS_RAW
  (tripduration, starttime, stoptime, start_station_id, start_station_name,
   start_lat, start_lon, end_station_id, end_station_name, end_lat, end_lon,
   bikeid, membership_type, birth_year, gender, _source_file)
FROM (
  SELECT
    $1::INTEGER, $2::TIMESTAMP_NTZ, $3::TIMESTAMP_NTZ,
    $4::INTEGER, $5::VARCHAR,
    $6::FLOAT,   $7::FLOAT,
    $8::INTEGER, $9::VARCHAR,
    $10::FLOAT,  $11::FLOAT,
    $12::INTEGER,$13::VARCHAR,
    $14::INTEGER,$15::INTEGER,
    METADATA$FILENAME
  FROM @CITIBIKE_STAGE
)
FILE_FORMAT = (TYPE='CSV' SKIP_HEADER=0 FIELD_OPTIONALLY_ENCLOSED_BY='"')
ON_ERROR    = 'CONTINUE'
PATTERN     = '.*trips_2013.*\.csv\.gz';   -- load just 2013 files to keep it small
```

**3b. Check how many rows loaded**
```sql
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.CITIBIKE_TRIPS_RAW;
```

**3c. Check which files were loaded and their row counts**
```sql
SELECT *
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'CITIBIKE_TRIPS_RAW',
    START_TIME => DATEADD('hours', -1, CURRENT_TIMESTAMP())
))
ORDER BY last_load_time DESC;
```
- How many files were loaded?
- Were any rows skipped (`rows_parsed` ≠ `rows_loaded`)?

**3d. Test idempotency — run the same COPY INTO again**
```sql
-- Run the exact same COPY INTO from 3a again
-- How many rows are loaded this time?
```

**3e. Check source file distribution**
```sql
-- Which file did each row come from?
SELECT _source_file, COUNT(*) AS row_count
FROM ECOMM_DB.RAW_DATA.CITIBIKE_TRIPS_RAW
GROUP BY _source_file
ORDER BY row_count DESC;
```

**Expected outcome:** Data loaded from 2013 files. Second COPY INTO loads 0 rows. Source file column shows file names.

**Discussion:** You used `ON_ERROR = 'CONTINUE'` for the Citibike data. Would this be appropriate for a financial transactions table? What setting would you use instead and why?

---

## Exercise 4 — Working with VARIANT and JSON Data

**Concept:** VARIANT column, PARSE_JSON, colon notation, LATERAL FLATTEN

**Tasks:**

**4a. Create a table for JSON event data**
```sql
USE DATABASE ECOMM_DB;
USE SCHEMA RAW_DATA;

CREATE TABLE IF NOT EXISTS ORDER_EVENTS_RAW (
    event_id      NUMBER         AUTOINCREMENT PRIMARY KEY,
    raw_event     VARIANT,
    _loaded_at    TIMESTAMP_NTZ  DEFAULT CURRENT_TIMESTAMP(),
    _source_file  VARCHAR(500)
);
```

**4b. Insert sample JSON events using PARSE_JSON**
```sql
INSERT INTO ORDER_EVENTS_RAW (raw_event)
SELECT PARSE_JSON(column1)
FROM VALUES
  ('{"order_id": 1001, "customer": {"id": 1, "name": "Alice Johnson", "tier": "GOLD"}, "items": [{"sku": "WH-001", "product": "Wireless Headphones", "qty": 1, "price": 149.99}], "total": 149.99, "status": "DELIVERED", "order_date": "2024-01-10"}'),
  ('{"order_id": 1002, "customer": {"id": 2, "name": "Bob Smith", "tier": "SILVER"}, "items": [{"sku": "RS-002", "product": "Running Shoes", "qty": 2, "price": 89.99}], "total": 179.98, "status": "DELIVERED", "order_date": "2024-01-15"}'),
  ('{"order_id": 1003, "customer": {"id": 3, "name": "Priya Sharma", "tier": "PLATINUM"}, "items": [{"sku": "CM-003", "product": "Coffee Maker", "qty": 1, "price": 129.99}, {"sku": "YM-005", "product": "Yoga Mat", "qty": 2, "price": 39.99}], "total": 209.97, "status": "SHIPPED", "order_date": "2024-01-20"}'),
  ('{"order_id": 1004, "customer": {"id": 4, "name": "Carlos Mendez", "tier": "BRONZE"}, "items": [{"sku": "PB-004", "product": "Python Book", "qty": 3, "price": 49.99}], "total": 149.97, "status": "PENDING", "order_date": "2024-02-01"}'),
  ('{"order_id": 1005, "customer": {"id": 5, "name": "Yuki Tanaka", "tier": "GOLD"}, "items": [{"sku": "WH-001", "product": "Wireless Headphones", "qty": 1, "price": 149.99}, {"sku": "YM-005", "product": "Yoga Mat", "qty": 1, "price": 39.99}], "total": 189.98, "status": "DELIVERED", "order_date": "2024-02-10"}');
```

**4c. See what the raw VARIANT looks like**
```sql
SELECT raw_event FROM ORDER_EVENTS_RAW LIMIT 2;
-- Notice the entire JSON object appears in one column
```

**4d. Extract flat fields using colon notation**
```sql
SELECT
    raw_event:order_id::NUMBER         AS order_id,
    raw_event:status::VARCHAR          AS status,
    raw_event:order_date::DATE         AS order_date,
    raw_event:total::NUMBER(10,2)      AS total_amount,
    raw_event:customer:id::NUMBER      AS customer_id,
    raw_event:customer:name::VARCHAR   AS customer_name,
    raw_event:customer:tier::VARCHAR   AS loyalty_tier
FROM ORDER_EVENTS_RAW
ORDER BY order_id;
```

**4e. Access array elements by index**
```sql
SELECT
    raw_event:order_id::NUMBER               AS order_id,
    ARRAY_SIZE(raw_event:items)              AS total_items,
    raw_event:items[0]:sku::VARCHAR          AS first_item_sku,
    raw_event:items[0]:product::VARCHAR      AS first_item_name,
    raw_event:items[0]:price::NUMBER(10,2)   AS first_item_price
FROM ORDER_EVENTS_RAW
ORDER BY order_id;
```

**4f. Use LATERAL FLATTEN to get one row per item**
```sql
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
FROM ORDER_EVENTS_RAW o,
LATERAL FLATTEN(INPUT => o.raw_event:items) item
ORDER BY order_id, sku;
```

How many rows does this return? How many orders exist? Why are they different?

**Expected outcome:** 7 rows from 5 orders — orders 1003 and 1005 each have 2 items.

**Discussion:** Order 1003 has items with SKUs `CM-003` and `YM-005`. Without `LATERAL FLATTEN`, how would you access both items in one query? What limitation would that have as the number of items grows?

---

## Exercise 5 — VARIANT Utility Functions

**Concept:** TYPEOF, OBJECT_KEYS, ARRAY_SIZE, TRY_CAST, GET_PATH

**Tasks:**

**5a. Inspect the structure of your VARIANT data**
```sql
-- What keys does the top-level JSON object have?
SELECT OBJECT_KEYS(raw_event) AS top_level_keys
FROM ORDER_EVENTS_RAW
LIMIT 1;

-- What type is each value?
SELECT
    TYPEOF(raw_event:order_id)   AS order_id_type,
    TYPEOF(raw_event:total)      AS total_type,
    TYPEOF(raw_event:status)     AS status_type,
    TYPEOF(raw_event:items)      AS items_type,
    TYPEOF(raw_event:customer)   AS customer_type
FROM ORDER_EVENTS_RAW
LIMIT 1;
```

**5b. Count items per order**
```sql
SELECT
    raw_event:order_id::NUMBER     AS order_id,
    raw_event:customer:name::VARCHAR AS customer,
    ARRAY_SIZE(raw_event:items)    AS item_count,
    raw_event:total::NUMBER(10,2)  AS order_total
FROM ORDER_EVENTS_RAW
ORDER BY item_count DESC;
```

**5c. Handle missing fields safely with TRY_CAST**
```sql
-- Insert a record with a missing field and a string total (bad data simulation)
INSERT INTO ORDER_EVENTS_RAW (raw_event)
SELECT PARSE_JSON('{"order_id": 9999, "customer": {"id": 99, "name": "Test User"}, "total": "UNKNOWN", "status": "ERROR"}');

-- Now try extracting total — the UNKNOWN string would fail a direct ::NUMBER cast
-- Use TRY_TO_NUMBER to handle gracefully
SELECT
    raw_event:order_id::NUMBER                        AS order_id,
    raw_event:customer:name::VARCHAR                  AS customer_name,
    raw_event:status::VARCHAR                         AS status,
    TRY_TO_NUMBER(raw_event:total::VARCHAR)           AS total_safe,   -- NULL for bad values
    raw_event:items IS NULL                           AS has_no_items  -- missing key = NULL
FROM ORDER_EVENTS_RAW
ORDER BY order_id;
-- order 9999 should show total_safe=NULL and has_no_items=TRUE
```

**5d. Alternative path navigation with GET_PATH**
```sql
-- GET_PATH is equivalent to colon notation — both work
SELECT
    GET_PATH(raw_event, 'order_id')::NUMBER          AS order_id,
    GET_PATH(raw_event, 'customer.name')::VARCHAR    AS customer_name,
    GET_PATH(raw_event, 'items[0].sku')::VARCHAR     AS first_sku
FROM ORDER_EVENTS_RAW
WHERE raw_event:order_id::NUMBER != 9999;
```

**Expected outcome:** You can inspect, navigate, and safely extract VARIANT data even when fields are missing or have unexpected types.

**Discussion:** Row 9999 has `"total": "UNKNOWN"`. In a real pipeline, what would you do with rows like this — drop them, load them as-is, or flag them for manual review? What column would you add to the table to track this?

---

## Exercise 6 — Flatten JSON into a Structured Staging Table

**Concept:** Combining VARIANT querying with staging table creation

**Tasks:**

**6a. Create a structured staging table from the JSON events**
```sql
USE SCHEMA ECOMM_DB.STAGING;

-- Flat order-level staging table
CREATE OR REPLACE TRANSIENT TABLE STG_ORDER_EVENTS AS
SELECT
    o.raw_event:order_id::NUMBER(15,0)       AS order_id,
    o.raw_event:customer:id::NUMBER(10,0)    AS customer_id,
    o.raw_event:customer:name::VARCHAR       AS customer_name,
    o.raw_event:customer:tier::VARCHAR       AS loyalty_tier,
    o.raw_event:order_date::DATE             AS order_date,
    o.raw_event:status::VARCHAR              AS status,
    TRY_TO_NUMBER(o.raw_event:total::VARCHAR) AS total_amount,
    ARRAY_SIZE(o.raw_event:items)            AS item_count,
    o._loaded_at
FROM ECOMM_DB.RAW_DATA.ORDER_EVENTS_RAW o
WHERE o.raw_event:order_id::NUMBER != 9999;  -- exclude bad test record

SELECT * FROM STG_ORDER_EVENTS ORDER BY order_id;
```

**6b. Create a line-item level staging table using FLATTEN**
```sql
-- One row per order item
CREATE OR REPLACE TRANSIENT TABLE STG_ORDER_LINE_ITEMS AS
SELECT
    o.raw_event:order_id::NUMBER(15,0)       AS order_id,
    o.raw_event:customer:id::NUMBER(10,0)    AS customer_id,
    o.raw_event:order_date::DATE             AS order_date,
    item.index + 1                           AS line_number,       -- 1-based line number
    item.value:sku::VARCHAR                  AS sku,
    item.value:product::VARCHAR              AS product_name,
    item.value:qty::NUMBER                   AS quantity,
    item.value:price::NUMBER(10,2)           AS unit_price,
    item.value:qty::NUMBER
      * item.value:price::NUMBER(10,2)       AS line_total,
    o._loaded_at
FROM ECOMM_DB.RAW_DATA.ORDER_EVENTS_RAW o,
LATERAL FLATTEN(INPUT => o.raw_event:items) item
WHERE o.raw_event:order_id::NUMBER != 9999;

SELECT * FROM STG_ORDER_LINE_ITEMS ORDER BY order_id, line_number;
```

**6c. Validate the data makes sense**
```sql
-- Total revenue per order from line items should match order total
SELECT
    e.order_id,
    e.total_amount                          AS order_total_from_header,
    SUM(li.line_total)                      AS order_total_from_lines,
    e.total_amount - SUM(li.line_total)     AS discrepancy
FROM STG_ORDER_EVENTS e
JOIN STG_ORDER_LINE_ITEMS li ON e.order_id = li.order_id
GROUP BY e.order_id, e.total_amount
ORDER BY order_id;
-- Discrepancy should be 0.00 for all orders
```

**6d. Full inventory of Day 3 objects**
```sql
SELECT table_schema, table_name, table_type, row_count
FROM ECOMM_DB.INFORMATION_SCHEMA.TABLES
WHERE table_schema != 'INFORMATION_SCHEMA'
ORDER BY table_schema, table_name;
```

**Expected outcome:** `STG_ORDER_EVENTS` has 5 rows (excluding test record). `STG_ORDER_LINE_ITEMS` has 7 rows. Totals match across both tables.

---

## Bonus Challenge — Build a Complete Mini Pipeline

**Concept:** End-to-end: ingest → stage → flatten → mart

Using everything from Days 2 and 3, build a mini pipeline entirely in SQL:

```sql
-- Step 1: Simulate new raw JSON arriving (new orders)
INSERT INTO ECOMM_DB.RAW_DATA.ORDER_EVENTS_RAW (raw_event)
SELECT PARSE_JSON(column1) FROM VALUES
  ('{"order_id": 2001, "customer": {"id": 8, "name": "Sophie Dubois", "tier": "PLATINUM"}, "items": [{"sku": "WH-001", "product": "Wireless Headphones", "qty": 2, "price": 149.99}, {"sku": "CM-003", "product": "Coffee Maker", "qty": 1, "price": 129.99}], "total": 429.97, "status": "DELIVERED", "order_date": "2024-03-01"}'),
  ('{"order_id": 2002, "customer": {"id": 9, "name": "Raj Patel", "tier": "GOLD"}, "items": [{"sku": "RS-002", "product": "Running Shoes", "qty": 1, "price": 89.99}], "total": 89.99, "status": "SHIPPED", "order_date": "2024-03-05"}');

-- Step 2: Refresh the staging tables (simulate the ETL run)
CREATE OR REPLACE TRANSIENT TABLE ECOMM_DB.STAGING.STG_ORDER_EVENTS AS
SELECT
    o.raw_event:order_id::NUMBER(15,0)       AS order_id,
    o.raw_event:customer:id::NUMBER(10,0)    AS customer_id,
    o.raw_event:customer:name::VARCHAR       AS customer_name,
    o.raw_event:customer:tier::VARCHAR       AS loyalty_tier,
    o.raw_event:order_date::DATE             AS order_date,
    o.raw_event:status::VARCHAR              AS status,
    TRY_TO_NUMBER(o.raw_event:total::VARCHAR) AS total_amount,
    ARRAY_SIZE(o.raw_event:items)            AS item_count,
    o._loaded_at
FROM ECOMM_DB.RAW_DATA.ORDER_EVENTS_RAW o
WHERE o.raw_event:order_id::NUMBER < 9000;

-- Step 3: Refresh the mart
CREATE OR REPLACE TRANSIENT TABLE ECOMM_DB.MARTS.ORDER_SUMMARY_MART AS
SELECT
    e.order_id,
    e.order_date,
    e.customer_name,
    e.loyalty_tier,
    e.status,
    e.total_amount,
    e.item_count,
    -- Enrich from customer table if data matches
    c.country,
    c.city
FROM ECOMM_DB.STAGING.STG_ORDER_EVENTS e
LEFT JOIN ECOMM_DB.RAW_DATA.CUSTOMERS_RAW c ON e.customer_id = c.customer_id;

-- Step 4: Query the mart as a DATA_ANALYST would
USE ROLE DATA_ANALYST;
USE WAREHOUSE ANALYST_WH;
SELECT * FROM ECOMM_DB.MARTS.ORDER_SUMMARY_MART ORDER BY order_date;
```

**Design question:** This pipeline runs `CREATE OR REPLACE TRANSIENT TABLE` to refresh staging and marts. In production with millions of rows, what are two problems with this approach? What Snowflake features (covered in later days) solve each problem?
