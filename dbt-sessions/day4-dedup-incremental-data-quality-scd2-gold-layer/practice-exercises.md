# Day 4 — Practice Exercises: End-to-End Project

> Build the full pipeline from scratch: seeds → staging (dedup + incremental) → tests → snapshot → gold (dim + fact).
> Run all dbt commands from inside the `ecomm_dbt/` project directory.
> Estimated total time: 90–120 minutes.

---

## Pre-Check

Confirm the project is working from Day 3:

```bash
dbt run
dbt test
```

If anything is broken, fix it before starting Day 4.

---

## Exercise 1 — Create the Raw Data Seeds

### Goal
Load three CSV files as raw tables in Snowflake — intentionally dirty with duplicates and bad values.

### Step 1: Update `dbt_project.yml`

Replace the entire file content with:

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

### Step 2: Create the seed files

**`seeds/customers_raw.csv`** — includes duplicate `customer_id=1` and `customer_id=2`, and a row with missing email:

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

**`seeds/products_raw.csv`** — includes a duplicate `product_id=1` and a negative price:

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

**`seeds/orders_raw.csv`** — includes a duplicate `order_id=5`, an orphan `customer_id=99`, and a wrong `total_amount`:

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

### Step 3: Load the seeds

```bash
dbt seed
```

### Step 4: Verify raw data in Snowflake

```sql
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.CUSTOMERS_RAW;  -- expect 9 (with duplicates)
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.PRODUCTS_RAW;   -- expect 8 (with duplicate)
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.ORDERS_RAW;     -- expect 12 (with duplicate + bad row)
```

**Checkpoint:** 9, 8, 12 raw rows respectively. Duplicates are present — that is intentional.

---

## Exercise 2 — Staging: Deduplication

### Goal
Build staging models that deduplicate raw data using `ROW_NUMBER()`.

### Step 1: Create `models/staging/stg_customers.sql`

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

### Step 2: Create `models/staging/stg_products.sql`

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

### Step 3: Run and verify deduplication

```bash
dbt run --select stg_customers stg_products
```

```sql
SELECT COUNT(*) FROM ECOMM_DB.STAGING.STG_CUSTOMERS;  -- expect 7 (not 9)
SELECT COUNT(*) FROM ECOMM_DB.STAGING.STG_PRODUCTS;   -- expect 7 (not 8)

-- Confirm no duplicate customer_ids remain
SELECT customer_id, COUNT(*) AS cnt
FROM ECOMM_DB.STAGING.STG_CUSTOMERS
GROUP BY customer_id
HAVING cnt > 1;
-- Expected: 0 rows (no duplicates)
```

**Checkpoint:** 7 customers, 7 products. `HAVING cnt > 1` returns no rows.

---

## Exercise 3 — Staging: Incremental Model with Dedup

### Goal
Build `stg_orders` as an incremental model that also deduplicates — combining both patterns.

### Step 1: Create `models/staging/stg_orders.sql`

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

### Step 2: First run — full load

```bash
dbt run --select stg_orders
```

```sql
SELECT COUNT(*) FROM ECOMM_DB.STAGING.STG_ORDERS;  -- expect 11 (12 minus 1 duplicate)
```

### Step 3: Simulate incremental — add new orders to the seed

Add these two rows to the bottom of `seeds/orders_raw.csv`:

```csv
12,1,2,2,29.99,59.98,CONFIRMED,2024-04-01
13,3,4,1,49.99,49.99,SHIPPED,2024-04-02
```

Re-seed and re-run:

```bash
dbt seed
dbt run --select stg_orders
```

```sql
SELECT COUNT(*) FROM ECOMM_DB.STAGING.STG_ORDERS;  -- expect 13 (11 + 2 new)
SELECT * FROM ECOMM_DB.STAGING.STG_ORDERS WHERE order_date >= '2024-04-01';
-- expect 2 rows (the newly added orders)
```

**Checkpoint:** Count goes from 11 to 13. Only the 2 new rows were processed on the second run.

### Step 4: Test `--full-refresh`

```bash
dbt run --select stg_orders --full-refresh
```

```sql
SELECT COUNT(*) FROM ECOMM_DB.STAGING.STG_ORDERS;  -- still 13 — full rebuild same result
```

**Checkpoint:** Same row count after full refresh confirms the model is correct.

---

## Exercise 4 — Data Quality Tests

### Goal
Add generic tests via `schema.yml` and three business-level singular tests.

### Step 1: Create the custom generic test macro

Create `macros/test_not_negative.sql`:

```sql
{% test not_negative(model, column_name) %}

SELECT {{ column_name }}
FROM {{ model }}
WHERE {{ column_name }} < 0

{% endtest %}
```

### Step 2: Create `models/staging/schema.yml`

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
    description: "Deduplicated incremental orders. One row per order_id."
    columns:
      - name: order_id
        tests:
          - not_null
          - unique
      - name: customer_id
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

### Step 3: Create three singular tests

**`tests/assert_order_amount_matches_quantity_x_price.sql`**

```sql
SELECT
    order_id,
    quantity,
    unit_price,
    total_amount,
    ROUND(quantity * unit_price, 2) AS expected_amount
FROM {{ ref('stg_orders') }}
WHERE ABS(total_amount - ROUND(quantity * unit_price, 2)) > 0.01
```

**`tests/assert_no_future_order_dates.sql`**

```sql
SELECT
    order_id,
    order_date,
    CURRENT_DATE() AS today
FROM {{ ref('stg_orders') }}
WHERE order_date > CURRENT_DATE()
```

**`tests/assert_customer_exists_for_order.sql`**

```sql
SELECT
    o.order_id,
    o.customer_id
FROM {{ ref('stg_orders') }} o
WHERE o.customer_id NOT IN (
    SELECT customer_id FROM {{ ref('stg_customers') }}
)
```

### Step 4: Run all staging tests

```bash
dbt test --select staging
```

**Expected output — 3 intentional failures:**

```
FAIL  not_negative_stg_products_unit_price
FAIL  assert_order_amount_matches_quantity_x_price
FAIL  assert_customer_exists_for_order
```

These failures are expected. They prove the tests work. All other tests pass.

**Checkpoint:** Exactly 3 failures. All related to the intentional dirty data in the seeds.

---

## Exercise 5 — SCD Type 2 Snapshot

### Goal
Capture customer address history. Simulate an address change and verify two versions exist in the snapshot.

### Step 1: Create `snapshots/customers_snapshot.sql`

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

### Step 2: Run snapshot for the first time

```bash
dbt snapshot
```

```sql
-- All 7 customers, all current (dbt_valid_to = NULL)
SELECT customer_id, customer_name, city, dbt_valid_from, dbt_valid_to
FROM ECOMM_DB.SNAPSHOTS.CUSTOMERS_SNAPSHOT
ORDER BY customer_id;
```

**Checkpoint:** 7 rows, all with `dbt_valid_to = NULL`.

### Step 3: Simulate an address change

In `seeds/customers_raw.csv`, change Alice (customer_id=1) from `New York` to `Los Angeles`:

```csv
1,Alice Johnson,alice@email.com,Los Angeles,US,2024-01-10
```

Re-seed, re-run staging, re-snapshot:

```bash
dbt seed
dbt run --select stg_customers
dbt snapshot
```

### Step 4: Verify SCD Type 2 history

```sql
-- Alice should have 2 rows now
SELECT customer_id, customer_name, city, dbt_valid_from, dbt_valid_to
FROM ECOMM_DB.SNAPSHOTS.CUSTOMERS_SNAPSHOT
WHERE customer_id = 1
ORDER BY dbt_valid_from;
```

Expected:
```
customer_id  city          dbt_valid_from   dbt_valid_to
1            New York      <first run time> <second run time>  ← closed
1            Los Angeles   <second run time> NULL              ← current
```

```sql
-- All other customers: still 1 row each
SELECT customer_id, COUNT(*) AS versions
FROM ECOMM_DB.SNAPSHOTS.CUSTOMERS_SNAPSHOT
GROUP BY customer_id
ORDER BY customer_id;
-- Expected: customer_id=1 has 2, all others have 1
```

**Checkpoint:** `customer_id=1` has 2 versions. Total snapshot rows = 8 (7 original + 1 new version for Alice).

---

## Exercise 6 — Gold Layer: Dimension and Fact Tables

### Goal
Build `dim_customers`, `dim_products`, and `fct_orders` using surrogate keys and explicit WHERE clauses to filter bad data.

### Step 1: Create the `gold/` folder

Create `models/gold/` directory. Add these three models:

**`models/gold/dim_customers.sql`**

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

**`models/gold/dim_products.sql`**

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

**`models/gold/fct_orders.sql`**

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
    o.customer_id IN (SELECT customer_id FROM {{ ref('stg_customers') }})
    AND ABS(o.total_amount - ROUND(o.quantity * o.unit_price, 2)) <= 0.01
```

### Step 2: Add `models/gold/schema.yml`

```yaml
version: 2

models:
  - name: dim_customers
    description: "Current state of every customer. One row per customer_id."
    columns:
      - name: customer_key
        tests: [not_null, unique]
      - name: customer_id
        tests: [not_null, unique]
      - name: customer_name
        tests: [not_null]

  - name: dim_products
    description: "Active products with valid price. Excludes inactive and negative-price rows."
    columns:
      - name: product_key
        tests: [not_null, unique]
      - name: product_id
        tests: [not_null, unique]
      - name: price_tier
        tests:
          - not_null
          - accepted_values:
              values: ['Budget', 'Mid-range', 'Premium', 'Enterprise']

  - name: fct_orders
    description: "One row per valid order. Orphan FK and amount-mismatch rows excluded."
    columns:
      - name: order_id
        tests: [not_null, unique]
      - name: customer_key
        tests: [not_null]
      - name: product_key
        tests: [not_null]
      - name: total_amount
        tests: [not_null, not_negative]
      - name: status
        tests:
          - not_null
          - accepted_values:
              values: ['CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED']
```

### Step 3: Run the gold layer

```bash
dbt run --select gold
dbt test --select gold
```

**Expected: all gold tests PASS** — the bad rows were filtered by the WHERE clauses.

### Step 4: Verify gold layer counts

```sql
-- 7 customers (all current)
SELECT COUNT(*) FROM ECOMM_DB.GOLD.DIM_CUSTOMERS;

-- 6 products (product_id=7 excluded: negative price + inactive)
SELECT COUNT(*) FROM ECOMM_DB.GOLD.DIM_PRODUCTS;

-- 12 valid orders (order_id=11 excluded: orphan FK + wrong amount)
SELECT COUNT(*) FROM ECOMM_DB.GOLD.FCT_ORDERS;
```

**Checkpoint:** 7 / 6 / 12. The one bad order (`order_id=11`) is excluded from the fact table.

---

## Exercise 7 — Full `dbt build`

### Goal
Run the entire pipeline in one command and observe topological ordering.

```bash
dbt build
```

### What to observe

Watch the output — things run in this order:
1. Seeds load
2. Staging models build (views)
3. Staging tests run — 3 intentional failures appear
4. Snapshot runs
5. Gold models build (tables)
6. Gold tests run — all pass

### Expected final summary

```
Done. PASS=N WARN=0 ERROR=0 FAIL=3 SKIP=0 TOTAL=N
```

The 3 failures are the intentional staging tests. All gold tests pass because the bad rows were filtered out.

---

## Exercise 8 — The Star Schema Query

### Goal
Write the final BI-layer query that joins all three gold tables — the query that Tableau, Power BI, or any analytics tool would run.

```sql
SELECT
    f.order_id,
    c.customer_name,
    c.city,
    c.country,
    p.product_name,
    p.category,
    p.price_tier,
    f.quantity,
    f.total_amount,
    f.status,
    f.order_date,
    f.delivered_revenue,
    f.cancelled_revenue
FROM ECOMM_DB.GOLD.FCT_ORDERS       f
JOIN ECOMM_DB.GOLD.DIM_CUSTOMERS    c ON f.customer_key = c.customer_key
JOIN ECOMM_DB.GOLD.DIM_PRODUCTS     p ON f.product_key  = p.product_key
ORDER BY f.order_date;
```

Run this in Snowsight.

**Checkpoint:** Every order row is enriched with the customer name, city, product name, and category. No `customer_id=99` row appears (filtered out). No bad amount row appears (filtered out).

---

## Stretch Goals

1. **Point-in-time query** — Write a query against `CUSTOMERS_SNAPSHOT` that shows Alice's city at the time each of her orders was placed. Join `FCT_ORDERS` to `CUSTOMERS_SNAPSHOT` using `dbt_valid_from` and `dbt_valid_to` date range logic.

2. **Revenue summary by category** — Write a query on the gold layer that shows total delivered revenue and total cancelled revenue grouped by `p.category`.

3. **Fix the bad seed data** — Remove `order_id=11` from `orders_raw.csv`, remove the negative price from `products_raw.csv`. Re-seed and re-run. Confirm all staging tests now pass and `fct_orders` has 14 rows (12 original valid + 2 added in Exercise 3).

4. **Add a `dbt_valid_from` column to `fct_orders`** — Join `fct_orders` to `customers_snapshot` (not `dim_customers`) using the order date to bring in the city the customer was in at the time of the order. This is the proper historical join — not the current city from the dimension.
