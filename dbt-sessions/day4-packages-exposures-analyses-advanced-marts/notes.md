# Day 4 — End-to-End Project: Dedup, Incremental, Data Quality, SCD Type 2, and Gold Layer

## What You Will Build

A single end-to-end project simulating a **real e-commerce data pipeline** — from raw seed data all the way to production-ready fact and dimension tables. Every concept builds on the previous one inside the same project.

| Concept | What you implement |
|---|---|
| Seeds as raw source | `customers_raw`, `orders_raw`, `products_raw` CSVs with intentional dirty data |
| Deduplication | Remove duplicate rows in the staging layer before anything else runs |
| Incremental models | Process only new orders on each run — do not rebuild the full table |
| Generic data quality checks | Column-level tests: `not_null`, `unique`, `accepted_values`, custom `not_negative` |
| Business-level quality checks | Singular tests: `total_amount = quantity * unit_price`, no order before customer created |
| SCD Type 2 snapshot | Track customer address changes over time |
| Dimension tables (Gold) | `dim_customers`, `dim_products` — one clean row per entity |
| Fact table (Gold) | `fct_orders` — one row per order, joined to dims via surrogate keys |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                        ECOMM_DB (Snowflake)                          │
│                                                                      │
│  ┌─────────────────┐                                                 │
│  │   RAW_DATA      │  Seeds (CSV files loaded via dbt seed)          │
│  │  customers_raw  │  Intentionally dirty: duplicates, bad values    │
│  │  orders_raw     │                                                 │
│  │  products_raw   │                                                 │
│  └────────┬────────┘                                                 │
│           │  dbt run (staging layer)                                 │
│           ▼                                                          │
│  ┌─────────────────┐                                                 │
│  │    STAGING      │  Clean, deduplicated, cast, renamed             │
│  │  stg_customers  │  Dedup via ROW_NUMBER()                        │
│  │  stg_orders     │  Incremental — only new rows each run          │
│  │  stg_products   │  Dedup + basic cleaning                        │
│  └────────┬────────┘                                                 │
│           │  dbt test (data quality gate)                            │
│           │  Generic: not_null, unique, accepted_values              │
│           │  Singular: amount = qty × price, no future orders        │
│           ▼                                                          │
│  ┌─────────────────┐                                                 │
│  │   SNAPSHOTS     │  SCD Type 2 history                             │
│  │ customers_snap  │  Tracks address changes over time               │
│  └────────┬────────┘                                                 │
│           │  dbt run (gold layer)                                    │
│           ▼                                                          │
│  ┌─────────────────┐                                                 │
│  │      GOLD       │  Production-ready for BI / reporting            │
│  │  dim_customers  │  One row per customer (current state)           │
│  │  dim_products   │  One row per product                            │
│  │  fct_orders     │  One row per order, joined to dims              │
│  └─────────────────┘                                                 │
└──────────────────────────────────────────────────────────────────────┘
```

**Data flow rule:** Raw data is never modified. Each layer only reads from the layer below it. BI tools only read from the Gold layer.

---

## Project Setup

### Update `dbt_project.yml`

```yaml
name: 'ecomm_dbt'
version: '1.0.0'
config-version: 2

profile: 'ecomm_dbt'

model-paths:    ["models"]
seed-paths:     ["seeds"]
test-paths:     ["tests"]
macro-paths:    ["macros"]
snapshot-paths: ["snapshots"]
analysis-paths: ["analyses"]

models:
  ecomm_dbt:
    staging:
      +materialized: view
      +schema: STAGING
    gold:
      +materialized: table
      +schema: GOLD

seeds:
  ecomm_dbt:
    +schema: RAW_DATA
```

> **Note:** We use `staging` and `gold` as the two layers today. No intermediate layer — keep it simple and focused.

---

## Step 1 — Seeds: Raw Data With Intentional Problems

Seeds are CSV files in the `seeds/` folder. `dbt seed` loads them as tables into Snowflake. Today we use seeds to simulate raw data arriving from an upstream system — with real-world problems baked in: **duplicates, bad values, and missing fields**.

### `seeds/customers_raw.csv`

```csv
customer_id,customer_name,email,city,country,created_at
1,Alice Johnson,alice@email.com,New York,US,2024-01-10
2,Bob Smith,bob@email.com,London,UK,2024-01-15
3,Carol White,carol@email.com,Sydney,AU,2024-01-20
4,Dave Brown,dave@email.com,Toronto,CA,2024-02-01
5,Eve Davis,eve@email.com,New York,US,2024-02-10
1,Alice Johnson,alice@email.com,New York,US,2024-01-10
2,Bob Smith,bob_new@email.com,London,UK,2024-01-15
6,Frank Miller,,Chicago,US,2024-03-01
7,Grace Wilson,grace@email.com,Berlin,DE,2024-03-15
```

**Problems intentionally baked in:**
- Rows 6 and 7 (`customer_id=1` and `customer_id=2`) are **exact and near duplicates** — same `customer_id`, slightly different `email` on the second Bob
- Row 8 (`customer_id=6`) has a **NULL email** — missing value

---

### `seeds/products_raw.csv`

```csv
product_id,product_name,category,unit_price,is_active
1,Laptop Pro,Electronics,1299.99,true
2,Wireless Mouse,Electronics,29.99,true
3,Standing Desk,Furniture,599.99,true
4,USB-C Hub,Electronics,49.99,true
5,Office Chair,Furniture,399.99,true
6,Notebook Set,Stationery,12.99,true
7,Discontinued Item,Gadgets,-5.00,false
1,Laptop Pro,Electronics,1299.99,true
```

**Problems intentionally baked in:**
- Row 8 is a **duplicate** of row 1 (`product_id=1`)
- Row 7 (`product_id=7`) has a **negative price** — invalid business value

---

### `seeds/orders_raw.csv`

```csv
order_id,customer_id,product_id,quantity,unit_price,total_amount,status,order_date
1,1,1,1,1299.99,1299.99,DELIVERED,2024-02-01
2,2,2,3,29.99,89.97,SHIPPED,2024-02-05
3,3,3,1,599.99,599.99,CONFIRMED,2024-02-10
4,4,4,2,49.99,99.98,DELIVERED,2024-02-15
5,5,2,1,29.99,29.99,CANCELLED,2024-02-20
6,1,5,1,399.99,399.99,SHIPPED,2024-03-01
7,2,1,1,1299.99,1299.99,CONFIRMED,2024-03-05
8,3,6,4,12.99,51.96,DELIVERED,2024-03-10
9,4,2,2,29.99,59.98,SHIPPED,2024-03-15
10,5,4,1,49.99,49.99,DELIVERED,2024-03-20
5,5,2,1,29.99,29.99,CANCELLED,2024-02-20
11,99,1,1,1299.99,1399.99,CONFIRMED,2024-03-25
```

**Problems intentionally baked in:**
- Row 11 is a **duplicate** of row 5 (same `order_id=5`)
- Row 12 (`order_id=11`) has `customer_id=99` who does not exist — orphan foreign key
- Row 12 also has `total_amount=1399.99` but `quantity * unit_price = 1299.99` — **amount mismatch**

### Load all seeds

```bash
dbt seed
```

Verify in Snowflake:
```sql
SELECT * FROM ECOMM_DB.RAW_DATA.CUSTOMERS_RAW;   -- 9 rows (with duplicates)
SELECT * FROM ECOMM_DB.RAW_DATA.PRODUCTS_RAW;    -- 8 rows (with duplicate)
SELECT * FROM ECOMM_DB.RAW_DATA.ORDERS_RAW;      -- 12 rows (with duplicate + bad row)
```

---

## Step 2 — Staging: Deduplication

The first job of the staging layer is to **remove duplicates before they propagate downstream**. We use `ROW_NUMBER()` window function — standard SQL, works on any warehouse.

### The dedup pattern

```sql
ROW_NUMBER() OVER (
    PARTITION BY <unique_business_key>
    ORDER BY <tiebreaker — usually updated_at or a reliable column>
) AS row_num
```

Keep only `WHERE row_num = 1` — that is the "best" row per business key.

---

### `models/staging/stg_customers.sql`

```sql
WITH source AS (
    SELECT * FROM {{ ref('customers_raw') }}
),

deduplicated AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY created_at DESC
        ) AS row_num
    FROM source
),

final AS (
    SELECT
        customer_id,
        customer_name,
        email,
        city,
        country,
        created_at::DATE AS created_date
    FROM deduplicated
    WHERE row_num = 1
)

SELECT * FROM final
```

**What this does:**
- `PARTITION BY customer_id` — groups all rows with the same `customer_id` together
- `ORDER BY created_at DESC` — the most recent row wins the tie (gets `row_num = 1`)
- `WHERE row_num = 1` — only keeps the winning row per `customer_id`
- Result: 7 clean rows from 9 raw rows (duplicates for `customer_id=1` and `customer_id=2` collapsed)

---

### `models/staging/stg_products.sql`

```sql
WITH source AS (
    SELECT * FROM {{ ref('products_raw') }}
),

deduplicated AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY product_id
            ORDER BY product_id
        ) AS row_num
    FROM source
),

final AS (
    SELECT
        product_id,
        product_name,
        category,
        unit_price,
        is_active::BOOLEAN AS is_active
    FROM deduplicated
    WHERE row_num = 1
)

SELECT * FROM final
```

Result: 7 rows from 8 (duplicate `product_id=1` removed). The row with `unit_price=-5.00` is still present — we do NOT filter it here. Filtering on business rules belongs in **tests**, not in the model itself. The model's job is to deduplicate and clean; tests catch violations.

---

### `models/staging/stg_orders.sql` — Incremental Model

Orders grow every day. We do not want to rebuild the entire orders table on every run. This model is **incremental** — on the first run it loads everything; on every subsequent run it only loads rows newer than what is already in the table.

```sql
{{
    config(
        materialized = 'incremental',
        unique_key   = 'order_id'
    )
}}

WITH source AS (
    SELECT * FROM {{ ref('orders_raw') }}
),

deduplicated AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY order_id
            ORDER BY order_id
        ) AS row_num
    FROM source
),

final AS (
    SELECT
        order_id,
        customer_id,
        product_id,
        quantity,
        unit_price,
        total_amount,
        status,
        order_date::DATE AS order_date
    FROM deduplicated
    WHERE row_num = 1
)

SELECT * FROM final

{% if is_incremental() %}
    WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})
{% endif %}
```

**What this does:**

1. **Dedup first** — `ROW_NUMBER()` removes the duplicate `order_id=5` before the incremental filter
2. **Incremental filter** — `{% if is_incremental() %}` block only runs on the 2nd+ run
3. `{{ this }}` — refers to the existing `stg_orders` table to find the latest loaded date
4. `unique_key = 'order_id'` — if an order already exists and comes in again, UPDATE it (MERGE) instead of inserting a duplicate

**First run:** All 11 unique rows loaded (12 raw minus 1 duplicate = 11)
**Subsequent runs:** Only orders with `order_date` newer than the max already loaded

```bash
dbt run --select stg_customers stg_products stg_orders
```

Verify dedup worked:
```sql
-- Should be 7 (not 9)
SELECT COUNT(*) FROM ECOMM_DB.STAGING.STG_CUSTOMERS;

-- Should be 7 (not 8)
SELECT COUNT(*) FROM ECOMM_DB.STAGING.STG_PRODUCTS;

-- Should be 11 (not 12)
SELECT COUNT(*) FROM ECOMM_DB.STAGING.STG_ORDERS;
```

---

## Step 3 — Data Quality Tests

Tests are the **quality gate** between staging and gold. If any test fails, `dbt build` stops — bad data never reaches the gold layer.

We write two types:
1. **Generic tests** (in `schema.yml`) — column-level rules: `not_null`, `unique`, `accepted_values`, custom `not_negative`
2. **Singular tests** (in `tests/`) — business-level rules that require SQL logic

### Custom generic test macro — `macros/test_not_negative.sql`

```sql
{% test not_negative(model, column_name) %}

SELECT {{ column_name }}
FROM {{ model }}
WHERE {{ column_name }} < 0

{% endtest %}
```

---

### `models/staging/schema.yml`

```yaml
version: 2

models:
  - name: stg_customers
    description: "Deduplicated customers. One row per customer_id."
    columns:
      - name: customer_id
        tests:
          - not_null
          - unique
      - name: customer_name
        tests:
          - not_null
      - name: country
        tests:
          - not_null
          - accepted_values:
              values: ['US', 'UK', 'AU', 'CA', 'DE']

  - name: stg_products
    description: "Deduplicated products. One row per product_id."
    columns:
      - name: product_id
        tests:
          - not_null
          - unique
      - name: product_name
        tests:
          - not_null
      - name: unit_price
        tests:
          - not_null
          - not_negative
      - name: category
        tests:
          - not_null
          - accepted_values:
              values: ['Electronics', 'Furniture', 'Stationery', 'Gadgets']

  - name: stg_orders
    description: "Deduplicated orders. Incremental — one row per order_id."
    columns:
      - name: order_id
        tests:
          - not_null
          - unique
      - name: customer_id
        tests:
          - not_null
      - name: product_id
        tests:
          - not_null
      - name: quantity
        tests:
          - not_null
          - not_negative
      - name: total_amount
        tests:
          - not_null
          - not_negative
      - name: status
        tests:
          - not_null
          - accepted_values:
              values: ['CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED']
```

---

### Business-level singular tests

These catch problems that column-level tests cannot — they require comparing columns across the same row or joining two models.

**`tests/assert_order_amount_matches_quantity_x_price.sql`**

Business rule: `total_amount` must equal `quantity * unit_price`. The seed has one row (`order_id=11`) where this is violated.

```sql
-- Returns rows where total_amount does not match quantity * unit_price
-- A small tolerance (0.01) handles floating point rounding
SELECT
    order_id,
    quantity,
    unit_price,
    total_amount,
    ROUND(quantity * unit_price, 2)        AS expected_amount,
    total_amount - ROUND(quantity * unit_price, 2) AS discrepancy
FROM {{ ref('stg_orders') }}
WHERE ABS(total_amount - ROUND(quantity * unit_price, 2)) > 0.01
```

**`tests/assert_no_future_order_dates.sql`**

Business rule: no order can have an `order_date` in the future. This catches system clock issues or ETL bugs.

```sql
SELECT
    order_id,
    order_date,
    CURRENT_DATE() AS today
FROM {{ ref('stg_orders') }}
WHERE order_date > CURRENT_DATE()
```

**`tests/assert_customer_exists_for_order.sql`**

Business rule: every `customer_id` in `stg_orders` must exist in `stg_customers`. The seed has `customer_id=99` which does not exist.

```sql
SELECT
    o.order_id,
    o.customer_id
FROM {{ ref('stg_orders') }} o
WHERE o.customer_id NOT IN (
    SELECT customer_id FROM {{ ref('stg_customers') }}
)
```

### Run all tests

```bash
dbt test --select staging
```

Expected failures (from the intentional bad data in seeds):

| Test | Model | Why it fails |
|---|---|---|
| `not_negative` on `unit_price` | `stg_products` | `product_id=7` has `unit_price=-5.00` |
| `assert_order_amount_matches_quantity_x_price` | `stg_orders` | `order_id=11` has wrong `total_amount` |
| `assert_customer_exists_for_order` | `stg_orders` | `order_id=11` has `customer_id=99` |

These failures are **expected and intentional** — they prove the tests work. In a real project you would fix the source data or the ETL pipeline. For the rest of Day 4, we move on to building the gold layer with the clean data that did pass.

> **Important:** The rows that fail tests are still present in the staging tables. Tests report failures — they do not delete rows. Filtering bad rows out of the gold layer is done with explicit WHERE clauses in the gold models (shown in Step 5).

---

## Step 4 — SCD Type 2 Snapshot

Imagine a customer changes their `city` or `country`. Without a snapshot, you only ever know their **current** address. With an SCD Type 2 snapshot you keep the full history — you know what their address was when each order was placed.

**`snapshots/customers_snapshot.sql`**

```sql
{% snapshot customers_snapshot %}

{{
    config(
        target_schema = 'SNAPSHOTS',
        unique_key    = 'customer_id',
        strategy      = 'check',
        check_cols    = ['city', 'country', 'email']
    )
}}

SELECT
    customer_id,
    customer_name,
    email,
    city,
    country,
    created_date
FROM {{ ref('stg_customers') }}

{% endsnapshot %}
```

**Config explained:**

| Config | Value | Meaning |
|---|---|---|
| `target_schema` | `SNAPSHOTS` | Snapshot tables land in a dedicated schema |
| `unique_key` | `customer_id` | The business key identifying one customer |
| `strategy` | `check` | Detect changes by comparing the listed columns |
| `check_cols` | `['city', 'country', 'email']` | Only these column changes trigger a new version |

**Columns dbt adds automatically:**

| Column | Meaning |
|---|---|
| `dbt_scd_id` | Unique hash per version |
| `dbt_valid_from` | When this version became current |
| `dbt_valid_to` | When superseded — `NULL` means it is the current version |

### Run the snapshot (first time)

```bash
dbt snapshot
```

Verify — all 7 customers, all with `dbt_valid_to = NULL` (all current):
```sql
SELECT customer_id, customer_name, city, dbt_valid_from, dbt_valid_to
FROM ECOMM_DB.SNAPSHOTS.CUSTOMERS_SNAPSHOT
ORDER BY customer_id;
```

### Simulate a customer address change

Update `seeds/customers_raw.csv` — change Alice (customer_id=1) from `New York` to `Los Angeles`:

```csv
1,Alice Johnson,alice@email.com,Los Angeles,US,2024-01-10
```

Re-seed and re-run staging:
```bash
dbt seed
dbt run --select stg_customers
dbt snapshot
```

Verify history:
```sql
SELECT customer_id, customer_name, city, dbt_valid_from, dbt_valid_to
FROM ECOMM_DB.SNAPSHOTS.CUSTOMERS_SNAPSHOT
WHERE customer_id = 1
ORDER BY dbt_valid_from;
```

Expected result:
```
customer_id  city          dbt_valid_from        dbt_valid_to
1            New York      2024-11-28 10:00:00   2024-11-28 10:30:00  ← old version, closed
1            Los Angeles   2024-11-28 10:30:00   NULL                 ← current version
```

**Query: what city was Alice in when she placed order_id=1 on 2024-02-01?**

```sql
SELECT s.customer_name, s.city, s.country
FROM ECOMM_DB.SNAPSHOTS.CUSTOMERS_SNAPSHOT s
WHERE s.customer_id = 1
  AND s.dbt_valid_from <= '2024-02-01'
  AND (s.dbt_valid_to > '2024-02-01' OR s.dbt_valid_to IS NULL);
```

This is the power of SCD Type 2 — you can answer "what did the data look like at any point in time."

---

## Step 5 — Gold Layer: Dimension and Fact Tables

The gold layer is what BI tools and analysts query. The design pattern here is a **star schema**:

```
        dim_customers
              │
              │ customer_key
              │
fct_orders ──┤
              │ product_key
              │
        dim_products
```

- **Dimension tables** (`dim_`) — descriptive data about entities: who is the customer, what is the product
- **Fact table** (`fct_`) — measurable events: each order with its metrics (quantity, revenue) and foreign keys to dimensions

### `models/gold/dim_customers.sql`

The dimension reads from the **snapshot** — not from staging. This way the dimension always shows the **current state** of each customer (the most recent version).

```sql
{{
    config(
        materialized = 'table',
        schema       = 'GOLD'
    )
}}

SELECT
    MD5(CAST(customer_id AS VARCHAR))   AS customer_key,
    customer_id,
    customer_name,
    email,
    city,
    country,
    created_date
FROM {{ ref('customers_snapshot') }}
WHERE dbt_valid_to IS NULL
```

`WHERE dbt_valid_to IS NULL` — only the **current version** of each customer. This turns the SCD Type 2 history table into a clean "current state" dimension.

`MD5(CAST(customer_id AS VARCHAR))` — a surrogate key. In a real warehouse dimensions from multiple source systems may have overlapping natural keys. A surrogate key (hash of the natural key) is the standard dimensional modeling approach.

---

### `models/gold/dim_products.sql`

```sql
{{
    config(
        materialized = 'table',
        schema       = 'GOLD'
    )
}}

SELECT
    MD5(CAST(product_id AS VARCHAR))    AS product_key,
    product_id,
    product_name,
    category,
    unit_price,
    is_active,
    CASE
        WHEN unit_price < 50    THEN 'Budget'
        WHEN unit_price < 300   THEN 'Mid-range'
        WHEN unit_price < 1000  THEN 'Premium'
        ELSE 'Enterprise'
    END                                 AS price_tier
FROM {{ ref('stg_products') }}
WHERE unit_price > 0
  AND is_active = TRUE
```

`WHERE unit_price > 0` — explicitly excludes the bad row (`product_id=7` with `-5.00`) from the gold layer. The staging layer kept it so the test could flag it; the gold layer filters it out.

`WHERE is_active = TRUE` — only active products. Discontinued products should not be in the dimension that BI tools query for current inventory.

---

### `models/gold/fct_orders.sql`

The fact table joins to dimensions using **surrogate keys** and excludes the bad rows identified by our tests.

```sql
{{
    config(
        materialized = 'table',
        schema       = 'GOLD'
    )
}}

SELECT
    o.order_id,
    MD5(CAST(o.customer_id AS VARCHAR)) AS customer_key,
    MD5(CAST(o.product_id  AS VARCHAR)) AS product_key,

    o.quantity,
    o.unit_price,
    o.total_amount,
    o.status,
    o.order_date,

    CASE WHEN o.status = 'DELIVERED' THEN o.total_amount ELSE 0 END AS delivered_revenue,
    CASE WHEN o.status = 'CANCELLED' THEN o.total_amount ELSE 0 END AS cancelled_revenue

FROM {{ ref('stg_orders') }} o

WHERE
    -- Exclude orders with no matching customer (orphan FK — caught by singular test)
    o.customer_id IN (SELECT customer_id FROM {{ ref('stg_customers') }})
    -- Exclude orders where total_amount does not match quantity * unit_price
    AND ABS(o.total_amount - ROUND(o.quantity * o.unit_price, 2)) <= 0.01
```

The WHERE clause mirrors exactly what the singular tests check — bad rows that fail tests are also excluded from gold. This is the gold standard pattern:
- **Tests** tell you something is wrong
- **Gold model WHERE clause** ensures the problem does not reach BI

---

### Add a `schema.yml` for the gold layer

**`models/gold/schema.yml`**

```yaml
version: 2

models:
  - name: dim_customers
    description: "Current state of every customer. One row per customer_id."
    columns:
      - name: customer_key
        tests:
          - not_null
          - unique
      - name: customer_id
        tests:
          - not_null
          - unique
      - name: customer_name
        tests:
          - not_null

  - name: dim_products
    description: "Active products with price tier. Excludes inactive and negative-price rows."
    columns:
      - name: product_key
        tests:
          - not_null
          - unique
      - name: product_id
        tests:
          - not_null
          - unique
      - name: price_tier
        tests:
          - not_null
          - accepted_values:
              values: ['Budget', 'Mid-range', 'Premium', 'Enterprise']

  - name: fct_orders
    description: "One row per valid order. Orphan FK and amount-mismatch rows excluded."
    columns:
      - name: order_id
        tests:
          - not_null
          - unique
      - name: customer_key
        tests:
          - not_null
      - name: product_key
        tests:
          - not_null
      - name: total_amount
        tests:
          - not_null
          - not_negative
      - name: status
        tests:
          - not_null
          - accepted_values:
              values: ['CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED']
```

---

## Step 6 — Run Everything with `dbt build`

`dbt build` runs seeds + models + tests + snapshots in one topologically-ordered command. If any model's tests fail, its downstream models are automatically skipped.

```bash
dbt build
```

### What runs in order

```
1. seeds            → customers_raw, products_raw, orders_raw
2. stg_customers    → deduplicated view
3. stg_products     → deduplicated view
4. stg_orders       → deduplicated incremental table
5. tests on staging → generic + singular (3 expected failures flagged)
6. customers_snapshot
7. dim_customers    → reads from snapshot WHERE dbt_valid_to IS NULL
8. dim_products     → reads from stg_products WHERE unit_price > 0
9. fct_orders       → reads from stg_orders with exclusion WHERE clause
10. tests on gold   → all pass (bad rows excluded by WHERE)
```

### Expected test results

| Test | Layer | Result | Why |
|---|---|---|---|
| `unique` on `stg_customers.customer_id` | Staging | PASS | Dedup removed duplicates |
| `unique` on `stg_orders.order_id` | Staging | PASS | Dedup removed duplicate order |
| `not_negative` on `stg_products.unit_price` | Staging | **FAIL** | `product_id=7` has `-5.00` |
| `assert_order_amount_matches_quantity_x_price` | Staging | **FAIL** | `order_id=11` has wrong amount |
| `assert_customer_exists_for_order` | Staging | **FAIL** | `order_id=11` has `customer_id=99` |
| `unique` on `dim_customers.customer_key` | Gold | PASS | Clean dimension |
| `unique` on `fct_orders.order_id` | Gold | PASS | Bad rows excluded |
| All gold tests | Gold | PASS | Gold WHERE clauses filtered bad data |

The 3 staging failures are **intentional** — they prove the quality checks work. In production you would fix the upstream data source; in this exercise they demonstrate that tests catch real problems.

---

## Final File Structure After Day 4

```
ecomm_dbt/
├── dbt_project.yml                          ← updated: staging + gold layers
├── seeds/
│   ├── customers_raw.csv                    ← intentional duplicates + NULL
│   ├── products_raw.csv                     ← intentional duplicate + negative price
│   └── orders_raw.csv                       ← intentional duplicate + bad amount + orphan FK
├── models/
│   ├── staging/
│   │   ├── sources.yml
│   │   ├── schema.yml                       ← generic tests: not_null, unique, accepted_values, not_negative
│   │   ├── stg_customers.sql                ← dedup via ROW_NUMBER()
│   │   ├── stg_products.sql                 ← dedup via ROW_NUMBER()
│   │   └── stg_orders.sql                   ← dedup + incremental
│   └── gold/
│       ├── schema.yml                       ← tests on dim + fact tables
│       ├── dim_customers.sql                ← reads from snapshot, WHERE dbt_valid_to IS NULL
│       ├── dim_products.sql                 ← filters inactive + negative-price products
│       └── fct_orders.sql                   ← excludes orphan FK + amount-mismatch rows
├── tests/
│   ├── assert_order_amount_matches_quantity_x_price.sql  ← business rule test
│   ├── assert_no_future_order_dates.sql                  ← business rule test
│   └── assert_customer_exists_for_order.sql              ← referential integrity test
├── macros/
│   ├── generate_schema_name.sql
│   └── test_not_negative.sql                ← custom generic test macro
└── snapshots/
    └── customers_snapshot.sql               ← SCD Type 2 on city/country/email
```

---

## Verify the Gold Layer in Snowflake

```sql
-- Dimension: current customers (7 rows — no duplicates)
SELECT customer_key, customer_id, customer_name, city, country
FROM ECOMM_DB.GOLD.DIM_CUSTOMERS
ORDER BY customer_id;

-- Dimension: active products with valid price (6 rows — product_id=7 excluded)
SELECT product_key, product_id, product_name, unit_price, price_tier
FROM ECOMM_DB.GOLD.DIM_PRODUCTS
ORDER BY product_id;

-- Fact: valid orders only (10 rows — order_id=11 excluded)
SELECT order_id, customer_key, product_key, total_amount, status, order_date
FROM ECOMM_DB.GOLD.FCT_ORDERS
ORDER BY order_date;

-- Star schema join: orders enriched with customer and product names
SELECT
    f.order_id,
    c.customer_name,
    c.city,
    p.product_name,
    p.category,
    f.quantity,
    f.total_amount,
    f.status,
    f.order_date
FROM ECOMM_DB.GOLD.FCT_ORDERS       f
JOIN ECOMM_DB.GOLD.DIM_CUSTOMERS    c ON f.customer_key = c.customer_key
JOIN ECOMM_DB.GOLD.DIM_PRODUCTS     p ON f.product_key  = p.product_key
ORDER BY f.order_date;
```

---

## Summary — Concepts and Where They Appeared

| Concept | Where | Key takeaway |
|---|---|---|
| Seeds as raw source | `seeds/` CSVs | `dbt seed` loads CSVs as tables — good for simulating raw data |
| Intentional dirty data | All 3 seeds | Duplicates, negative values, orphan FKs — real problems in real pipelines |
| Deduplication | `stg_customers`, `stg_products`, `stg_orders` | `ROW_NUMBER() OVER (PARTITION BY key ORDER BY tiebreaker) = 1` |
| Incremental model | `stg_orders` | First run = full load; subsequent runs = only new rows via `is_incremental()` |
| `unique_key` in incremental | `stg_orders` config | UPDATE existing rows instead of inserting duplicates |
| Generic tests | `schema.yml` | `not_null`, `unique`, `accepted_values` — reusable, declared in YAML |
| Custom generic test | `macros/test_not_negative.sql` | `{% test not_negative %}` — write once, attach anywhere |
| Singular tests | `tests/` folder | SQL that returns failing rows — for business logic that YAML tests cannot express |
| Tests don't delete rows | Staging layer | Tests flag bad data; your WHERE clause in gold removes it |
| SCD Type 2 | `customers_snapshot` | `dbt snapshot` tracks row history with `dbt_valid_from` / `dbt_valid_to` |
| `WHERE dbt_valid_to IS NULL` | `dim_customers` | Converts SCD Type 2 history into a "current state" dimension |
| Surrogate key | `dim_customers`, `dim_products`, `fct_orders` | `MD5(CAST(id AS VARCHAR))` — joins dimensions to fact without natural key conflicts |
| Dimension table | `dim_customers`, `dim_products` | Descriptive entity data — one clean row per entity |
| Fact table | `fct_orders` | Measurable events — one row per order, FK joins to dims |
| Star schema join | Verification SQL | `fct_orders JOIN dim_customers JOIN dim_products` — the query BI tools run |
| `dbt build` | Final step | Seeds + models + tests + snapshots in one topological run |
