# Day 4 — Practice Exercises: Packages, Exposures, Analyses, and Advanced Mart Modeling

> All exercises build on the `ecomm_dbt` project from Day 3.
> Seeds (`products.csv`, `employees.csv`) must be loaded and staging models (`stg_orders`, `stg_products`, `stg_employees`) must be running before you start.
> Run all dbt commands from inside the `ecomm_dbt/` project directory.
> Estimated total time: 75–90 minutes.

---

## Pre-Check

```bash
dbt seed
dbt run
dbt test
```

Expected: all models pass, no test failures. If any model fails, fix it before proceeding.

---

## Exercise 1 — Install dbt-utils

### Goal
Add dbt-utils to the project and verify the package is accessible.

### Step 1: Create `packages.yml`

At the root of `ecomm_dbt/` (same level as `dbt_project.yml`), create:

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.3.0
```

### Step 2: Install packages

```bash
dbt deps
```

Expected output:
```
Installing dbt-labs/dbt_utils@1.3.0
  Installed from version 1.3.0
  Up to date!
```

### Step 3: Add `dbt_packages/` to `.gitignore`

Open or create `.gitignore` at the project root and add:

```
dbt_packages/
target/
logs/
```

### Step 4: Verify the package loaded

```bash
dbt compile
```

No errors = package is installed correctly.

**Checkpoint:** `dbt_packages/dbt_utils/` folder exists in your project.

---

## Exercise 2 — Add an Intermediate Layer

### Goal
Add an `intermediate/` folder to `dbt_project.yml` and create `int_orders_enriched.sql` — a model that joins `stg_orders` with `stg_products` and adds enrichment columns.

### Step 1: Update `dbt_project.yml`

Add the `intermediate` block:

```yaml
models:
  ecomm_dbt:
    staging:
      +materialized: view
      +schema: STAGING
    intermediate:
      +materialized: view
      +schema: INTERMEDIATE
    marts:
      +materialized: table
      +schema: MARTS
```

### Step 2: Create the intermediate model

Create `models/intermediate/int_orders_enriched.sql`:

```sql
SELECT
    o.order_id,
    o.customer_id,
    o.customer_name,
    o.product,
    o.quantity,
    o.unit_price,
    o.total_amount,
    o.status,
    o.order_date,
    o.updated_at,

    p.category                                              AS product_category,
    p.stock_quantity                                        AS current_stock,
    p.is_active                                             AS product_is_active,

    CASE
        WHEN o.total_amount >= 1000 THEN 'High Value'
        WHEN o.total_amount >= 300  THEN 'Mid Value'
        ELSE 'Low Value'
    END                                                     AS order_value_tier,

    DATEDIFF('day', o.order_date, CURRENT_DATE())           AS days_since_order,

    CASE WHEN o.status = 'CANCELLED' THEN o.total_amount ELSE 0 END  AS cancelled_revenue,
    CASE WHEN o.status = 'DELIVERED' THEN o.total_amount ELSE 0 END  AS delivered_revenue

FROM {{ ref('stg_orders') }}     o
LEFT JOIN {{ ref('stg_products') }} p
    ON LOWER(o.product) = LOWER(p.product_name)
```

### Step 3: Run and verify

```bash
dbt run --select int_orders_enriched
```

```sql
SELECT product_category, order_value_tier, days_since_order
FROM ECOMM_DB.INTERMEDIATE.INT_ORDERS_ENRICHED
LIMIT 10;
```

**Checkpoint:** Rows show `product_category` populated from the join and `order_value_tier` as `'High Value'`, `'Mid Value'`, or `'Low Value'`.

---

## Exercise 3 — Build the `customer_summary_mart`

### Goal
Create a mart that aggregates one row per customer using `int_orders_enriched`, and uses `dbt_utils.generate_surrogate_key`.

### Step 1: Create the mart model

Create `models/marts/customer_summary_mart.sql`:

```sql
{{
    config(
        materialized = 'table',
        schema       = 'MARTS'
    )
}}

SELECT
    customer_id,
    customer_name,
    COUNT(DISTINCT order_id)                                AS total_orders,
    MIN(order_date)                                         AS first_order_date,
    MAX(order_date)                                         AS latest_order_date,
    SUM(total_amount)                                       AS lifetime_value,
    AVG(total_amount)                                       AS avg_order_value,
    SUM(delivered_revenue)                                  AS delivered_revenue,
    SUM(cancelled_revenue)                                  AS cancelled_revenue,
    COUNT(DISTINCT CASE WHEN status = 'CANCELLED'
                   THEN order_id END)                       AS cancelled_orders,
    ROUND(
        COUNT(DISTINCT CASE WHEN status = 'CANCELLED'
                       THEN order_id END) * 100.0
        / NULLIF(COUNT(DISTINCT order_id), 0),
    2)                                                      AS cancellation_rate_pct,
    {{ dbt_utils.generate_surrogate_key(['customer_id']) }} AS customer_key
FROM {{ ref('int_orders_enriched') }}
GROUP BY customer_id, customer_name
```

### Step 2: Run and verify

```bash
dbt run --select customer_summary_mart
```

```sql
SELECT customer_name, total_orders, lifetime_value, cancellation_rate_pct, customer_key
FROM ECOMM_DB.MARTS.CUSTOMER_SUMMARY_MART
ORDER BY lifetime_value DESC;
```

**Checkpoint:** `customer_key` is a 32-character hex string (MD5 hash). `cancellation_rate_pct` is a number between 0 and 100.

---

## Exercise 4 — Build the `employees_mart` (Self-Join)

### Goal
Create a mart that joins `stg_employees` to itself to look up each employee's manager, and uses a window function for ranking.

### Step 1: Create the mart model

Create `models/marts/employees_mart.sql`:

```sql
{{
    config(
        materialized = 'table',
        schema       = 'MARTS'
    )
}}

SELECT
    e.employee_id,
    e.employee_name,
    e.department,
    e.salary,
    e.status,

    m.employee_name                                             AS manager_name,
    m.department                                                AS manager_department,

    CASE
        WHEN e.salary >= 90000 THEN 'Senior Band'
        WHEN e.salary >= 70000 THEN 'Mid Band'
        ELSE 'Junior Band'
    END                                                         AS salary_band,

    e.salary - AVG(e.salary) OVER (
        PARTITION BY e.department
    )                                                           AS salary_vs_dept_avg,

    RANK() OVER (
        PARTITION BY e.department
        ORDER BY e.salary DESC
    )                                                           AS salary_rank_in_dept,

    {{ dbt_utils.generate_surrogate_key(['e.employee_id']) }}  AS employee_key

FROM {{ ref('stg_employees') }} e
LEFT JOIN {{ ref('stg_employees') }} m
    ON e.manager_id = m.employee_id
```

### Step 2: Run and verify

```bash
dbt run --select employees_mart
```

```sql
SELECT employee_name, department, salary_band, manager_name, salary_rank_in_dept,
       ROUND(salary_vs_dept_avg, 2) AS vs_dept_avg
FROM ECOMM_DB.MARTS.EMPLOYEES_MART
ORDER BY department, salary_rank_in_dept;
```

**Checkpoint:** `manager_name` is NULL for department heads (employees with no `manager_id`). `salary_rank_in_dept = 1` is the highest earner per department.

---

## Exercise 5 — Create a Date Spine

### Goal
Build a date dimension table using `dbt_utils.date_spine` that covers all of 2024.

### Step 1: Create the model

Create `models/marts/date_spine_mart.sql`:

```sql
{{
    config(
        materialized = 'table',
        schema       = 'MARTS'
    )
}}

WITH date_spine AS (
    {{ dbt_utils.date_spine(
        datepart   = "day",
        start_date = "cast('2024-01-01' as date)",
        end_date   = "cast('2024-12-31' as date)"
    ) }}
)

SELECT
    date_day                                                  AS calendar_date,
    YEAR(date_day)                                            AS year,
    MONTH(date_day)                                           AS month,
    DAY(date_day)                                             AS day_of_month,
    DAYOFWEEK(date_day)                                       AS day_of_week,
    CASE WHEN DAYOFWEEK(date_day) IN (0, 6)
         THEN FALSE ELSE TRUE END                             AS is_weekday,
    MONTHNAME(date_day)                                       AS month_name,
    DAYNAME(date_day)                                         AS day_name
FROM date_spine
```

### Step 2: Run and verify

```bash
dbt run --select date_spine_mart
```

```sql
SELECT COUNT(*) FROM ECOMM_DB.MARTS.DATE_SPINE_MART;
-- Expected: 365 (2024 is a leap year — verify it's 366)

SELECT * FROM ECOMM_DB.MARTS.DATE_SPINE_MART
WHERE is_weekday = FALSE
LIMIT 5;
-- Expected: weekend dates
```

**Checkpoint:** 365 or 366 rows (2024 is a leap year — 366 is correct). Weekend rows have `is_weekday = FALSE`.

---

## Exercise 6 — Write Two Analyses

### Goal
Create analysis files for ad-hoc queries that use `ref()` but should not be materialized.

### Analysis 1 — Revenue by product category

Create `analyses/revenue_by_category.sql`:

```sql
SELECT
    p.category,
    COUNT(DISTINCT o.order_id)                              AS total_orders,
    SUM(o.total_amount)                                     AS total_revenue,
    AVG(o.total_amount)                                     AS avg_order_value,
    ROUND(
        SUM(o.total_amount) * 100.0
        / SUM(SUM(o.total_amount)) OVER (),
    2)                                                      AS revenue_share_pct
FROM {{ ref('int_orders_enriched') }} o
JOIN {{ ref('stg_products') }}        p
    ON LOWER(o.product) = LOWER(p.product_name)
GROUP BY p.category
ORDER BY total_revenue DESC
```

### Analysis 2 — Headcount by department

Create `analyses/employee_headcount_by_department.sql`:

```sql
SELECT
    department,
    COUNT(*)                                                AS headcount,
    ROUND(AVG(salary), 2)                                   AS avg_salary,
    SUM(salary)                                             AS total_salary_cost,
    ROUND(AVG(salary) - AVG(AVG(salary)) OVER (), 2)        AS salary_vs_company_avg
FROM {{ ref('stg_employees') }}
WHERE status = 'active'
GROUP BY department
ORDER BY headcount DESC
```

### Step 3: Compile and run manually

```bash
dbt compile --select revenue_by_category
```

Open the compiled file from:
```
target/compiled/ecomm_dbt/analyses/revenue_by_category.sql
```

Copy the SQL, paste it into Snowsight, and run it.

**Checkpoint:** Compiled SQL has no Jinja — only raw Snowflake SQL with fully-qualified table paths. Running it in Snowsight returns revenue grouped by Electronics, Furniture, etc.

---

## Exercise 7 — Declare Exposures

### Goal
Add two exposures to represent downstream consumers of the mart models.

### Create `models/exposures.yml`

```yaml
version: 2

exposures:

  - name: revenue_dashboard
    type: dashboard
    maturity: high
    description: >
      Executive revenue dashboard. Shows total revenue, orders by status,
      and top customers by lifetime value. Updated daily.
    owner:
      name: Data Analytics Team
      email: analytics@company.com
    depends_on:
      - ref('orders_mart')
      - ref('customer_summary_mart')

  - name: inventory_report
    type: dashboard
    maturity: medium
    description: >
      Product inventory and stock health report used by the Supply Chain team.
    owner:
      name: Supply Chain Team
      email: supply@company.com
    depends_on:
      - ref('products_mart')

  - name: customer_churn_ml_pipeline
    type: ml
    maturity: low
    description: >
      ML pipeline that scores customer churn probability.
      Uses lifetime_value and cancellation_rate_pct as features.
    owner:
      name: ML Engineering Team
      email: ml@company.com
    depends_on:
      - ref('customer_summary_mart')
```

### Verify exposures appear in the docs

```bash
dbt docs generate
dbt docs serve
```

In the lineage graph, click `customer_summary_mart` — you should see orange exposure nodes downstream of it.

**Checkpoint:** `revenue_dashboard` and `customer_churn_ml_pipeline` appear as orange nodes in the lineage graph. `inventory_report` appears downstream of `products_mart`.

---

## Exercise 8 — Run `dbt build` End-to-End

### Goal
Run the complete project — seeds, models, tests — in one command and observe topological ordering.

### Step 1: Full build

```bash
dbt build
```

Watch the output: seeds run first, then staging models + their tests, then intermediate + its tests, then marts + their tests.

**Checkpoint:** `FAIL=0 ERROR=0 SKIP=0` in the final output.

### Step 2: Understand node selection

Run only the downstream pipeline from `int_orders_enriched`:

```bash
dbt build --select int_orders_enriched+
```

Verify only `int_orders_enriched`, `customer_summary_mart`, and their tests ran (not `stg_orders` or `stg_products` — those are upstream).

### Step 3: Break a test and observe the cascade

Add a row to `seeds/products.csv` with a negative price:

```csv
13,Broken Product,Gadgets,-5.00,10,true
```

Run:
```bash
dbt seed
dbt build --select stg_products+
```

Expected: `stg_products` builds, the `assert_product_price_is_positive` singular test FAILS, and `products_mart` is SKIPPED.

Restore: remove row 13 from `products.csv` and re-seed:

```bash
dbt seed
dbt build --select stg_products+
```

**Checkpoint:** After restoring, `FAIL=0 SKIP=0`.

---

## Exercise 9 — Add Tests to Mart `schema.yml`

### Goal
Add tests to the new mart models so `dbt build` validates them automatically.

### Update `models/marts/schema.yml`

Add entries for `customer_summary_mart` and `employees_mart`:

```yaml
version: 2

models:
  - name: orders_mart
    description: "Daily order summary. One row per order_date."
    columns:
      - name: order_date
        tests: [not_null, unique]
      - name: total_orders
        tests: [not_null]
      - name: total_revenue
        tests: [not_null]

  - name: customer_summary_mart
    description: "One row per customer with lifetime order metrics."
    columns:
      - name: customer_id
        tests: [not_null, unique]
      - name: customer_key
        tests: [not_null, unique]
      - name: lifetime_value
        tests: [not_null]

  - name: employees_mart
    description: "Employee directory with manager lookup and salary band."
    columns:
      - name: employee_id
        tests: [not_null, unique]
      - name: employee_key
        tests: [not_null, unique]
      - name: salary_band
        tests:
          - not_null
          - accepted_values:
              values: ['Junior Band', 'Mid Band', 'Senior Band']

  - name: products_mart
    description: "Product catalogue enriched with price tiers and rankings."
    columns:
      - name: product_id
        tests: [not_null, unique]
      - name: product_key
        tests: [not_null, unique]
      - name: price_tier
        tests:
          - not_null
          - accepted_values:
              values: ['Budget', 'Mid-range', 'Premium', 'Enterprise']
```

### Run tests only on marts

```bash
dbt test --select marts
```

**Checkpoint:** All mart tests pass. If `accepted_values` on `price_tier` fails, check the values your `classify_price` macro produces against the list you declared.

---

## Final Verification

```bash
dbt build
```

Expected final output:
```
Done. PASS=N WARN=0 ERROR=0 FAIL=0 SKIP=0 TOTAL=N
```

Verify all new objects exist in Snowflake:

```sql
SHOW VIEWS  IN SCHEMA ECOMM_DB.INTERMEDIATE;
SHOW TABLES IN SCHEMA ECOMM_DB.MARTS;
```

Expected marts: `ORDERS_MART`, `PRODUCTS_MART`, `CUSTOMER_SUMMARY_MART`, `EMPLOYEES_MART`, `DATE_SPINE_MART`, `ORDERS_BY_CATEGORY_PIVOT`.

---

## Stretch Goals

1. **Dynamic pivot** — Replace the hardcoded product list in `orders_by_category_pivot.sql` with `dbt_utils.get_column_values` so it always covers all products without manual updates.

2. **Tag-based selection** — Add `+tags: ["daily"]` to the staging and marts layers in `dbt_project.yml`. Run `dbt build --select tag:daily` and confirm only tagged models run.

3. **Dynamic date spine** — Update `date_spine_mart.sql` so `end_date` always equals today using `modules.datetime.date.today()` in Jinja, eliminating the hardcoded `2024-12-31`.

4. **Exposure-driven audit** — Add a fourth exposure for `employees_mart` owned by HR. Run `dbt docs generate` and confirm all four exposures appear in the lineage graph with their correct upstream dependencies.
