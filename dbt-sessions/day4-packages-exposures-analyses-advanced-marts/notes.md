# Day 4 — dbt Packages, Exposures, Analyses, and Advanced Mart Modeling

## What You Will Learn

1. **dbt Packages** — Pull in community-built macros and models with `packages.yml` and `dbt deps`
2. **dbt-utils deep dive** — `generate_surrogate_key`, `date_spine`, `pivot`, `get_column_values`, and `union_relations`
3. **Advanced mart models** — Build multi-model marts that join staging tables and add business logic
4. **Analyses** — Ad-hoc SQL files tracked in version control but NOT materialized as tables/views
5. **Exposures** — Declare downstream consumers (BI dashboards, ML pipelines) in YAML so they appear in the lineage graph
6. **`dbt build`** — Run seeds + models + tests + snapshots in one topologically-ordered command
7. **DAG concepts** — Understand node selection, tags, and graph operators (`+`, `@`)

> **Starting point:** The `ecomm_dbt` project from Day 3 — seeds loaded (`products`, `employees`), staging models running, macros and snapshots in place.
> Today we wire everything together into a production-grade project structure.

---

## The Plan for Today

By the end of Day 4 the project will have:

| Layer | Models |
|---|---|
| Seeds | `products`, `employees` (Day 3) |
| Staging | `stg_orders`, `stg_products`, `stg_employees` |
| Intermediate | `int_orders_enriched` ← new today |
| Marts | `orders_mart` (Day 2), `products_mart` (Day 3), `employees_mart` ← new today, `customer_summary_mart` ← new today |
| Analyses | `revenue_by_category.sql` ← new today |
| Exposures | BI dashboard, revenue report ← new today |

---

## Concept 1 — dbt Packages

### What is a dbt package?

A **dbt package** is a collection of macros, models, and tests published to [hub.getdbt.com](https://hub.getdbt.com). You declare the packages you want in a `packages.yml` file, run `dbt deps`, and they are downloaded into the `dbt_packages/` folder.

The most widely-used package is **dbt-utils** — it ships with over 30 macros that fill in gaps in dbt's built-in functionality.

### Create `packages.yml`

Create this file at the root of your dbt project (same level as `dbt_project.yml`):

**`packages.yml`**

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.3.0
```

### Install packages

```bash
dbt deps
```

Expected output:
```
Installing dbt-labs/dbt_utils@1.3.0
  Installed from version 1.3.0
  Up to date!
```

Packages are downloaded to `dbt_packages/dbt_utils/`. Never commit the `dbt_packages/` folder to git — treat it like `node_modules`. Add it to `.gitignore`:

```
dbt_packages/
target/
logs/
```

### Why use dbt-utils instead of writing your own macros?

| Your custom macro | dbt-utils equivalent |
|---|---|
| `generate_surrogate_key` (Day 3) | `{{ dbt_utils.generate_surrogate_key([...]) }}` |
| Manual date loops | `{{ dbt_utils.date_spine(...) }}` |
| Manual CASE-based pivot | `{{ dbt_utils.pivot(...) }}` |

dbt-utils macros are tested across all adapters (Snowflake, BigQuery, Redshift, Postgres) and are maintained by dbt Labs. Using them reduces the amount of SQL you have to write and maintain.

---

## Concept 2 — dbt-utils Macros in Practice

### Macro 1 — `dbt_utils.generate_surrogate_key`

Replaces the custom `generate_surrogate_key` macro you wrote in Day 3. Hashes a list of columns into a stable surrogate key.

**Before (custom macro):**
```sql
{{ generate_surrogate_key(['product_id', 'category']) }} AS product_key
```

**After (dbt-utils):**
```sql
{{ dbt_utils.generate_surrogate_key(['product_id', 'category']) }} AS product_key
```

The dbt-utils version is adapter-aware — it uses `MD5` on Snowflake, `FARM_FINGERPRINT` on BigQuery, etc.

---

### Macro 2 — `dbt_utils.date_spine`

Generates a table of consecutive dates between two boundaries — useful for filling gaps in time-series data (days with no orders should still appear in a revenue report as 0).

**`models/marts/date_spine_mart.sql`**

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
    date_day                                              AS calendar_date,
    YEAR(date_day)                                        AS year,
    MONTH(date_day)                                       AS month,
    DAY(date_day)                                         AS day_of_month,
    DAYOFWEEK(date_day)                                   AS day_of_week,
    CASE WHEN DAYOFWEEK(date_day) IN (0, 6)
         THEN FALSE ELSE TRUE END                         AS is_weekday,
    MONTHNAME(date_day)                                   AS month_name,
    DAYNAME(date_day)                                     AS day_name
FROM date_spine
```

```bash
dbt run --select date_spine_mart
```

Verify in Snowflake:
```sql
SELECT * FROM ECOMM_DB.MARTS.DATE_SPINE_MART LIMIT 10;
```

---

### Macro 3 — `dbt_utils.pivot`

Converts row values into columns. Classic use case: turn a list of product categories into one column per category.

**`models/marts/orders_by_category_pivot.sql`**

```sql
{{
    config(
        materialized = 'table',
        schema       = 'MARTS'
    )
}}

SELECT
    order_date,
    {{ dbt_utils.pivot(
        column  = 'product',
        values  = ['Laptop Pro 15', 'Wireless Mouse', 'Standing Desk', 'USB-C Hub'],
        agg     = 'SUM',
        then_value = 'total_amount',
        else_value = '0',
        quote_identifiers = false
    ) }}
FROM {{ ref('stg_orders') }}
GROUP BY order_date
ORDER BY order_date
```

This generates SQL like:
```sql
SELECT
    order_date,
    SUM(CASE WHEN product = 'Laptop Pro 15'  THEN total_amount ELSE 0 END) AS laptop_pro_15,
    SUM(CASE WHEN product = 'Wireless Mouse' THEN total_amount ELSE 0 END) AS wireless_mouse,
    ...
FROM ECOMM_DB.STAGING.STG_ORDERS
GROUP BY order_date
```

---

### Macro 4 — `dbt_utils.get_column_values`

Dynamically fetches all distinct values from a column at **compile time** — eliminates the need to hardcode the list in `pivot`.

```sql
{% set product_names = dbt_utils.get_column_values(
    table  = ref('stg_orders'),
    column = 'product'
) %}

SELECT
    order_date,
    {{ dbt_utils.pivot(
        column  = 'product',
        values  = product_names,
        agg     = 'SUM',
        then_value = 'total_amount'
    ) }}
FROM {{ ref('stg_orders') }}
GROUP BY order_date
```

`get_column_values` runs a `SELECT DISTINCT` query against Snowflake during `dbt compile`, so the pivot columns are always up to date — no manual list to maintain.

---

## Concept 3 — Intermediate Layer Models

### Why add an intermediate layer?

Day 2 had two layers: staging → marts. For simple projects that works. As complexity grows, mart models start pulling logic from multiple staging models and duplicating transformations. An **intermediate layer** solves this:

```
staging/        stg_orders, stg_products, stg_employees
                   ↓
intermediate/   int_orders_enriched    ← join + enrich, single place
                   ↓
marts/          customer_summary_mart  ← aggregate, no raw logic here
```

Intermediate models are:
- Materialized as **views** (or ephemeral, never tables) — they are stepping stones, not end-products
- Never consumed by BI tools directly
- Named with the prefix `int_`

### Add `intermediate` to `dbt_project.yml`

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

### Create the intermediate model

**`models/intermediate/int_orders_enriched.sql`**

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

This model:
- Joins orders with products to bring in category and stock info
- Adds an `order_value_tier` label
- Pre-computes `cancelled_revenue` and `delivered_revenue` once here — downstream marts just SUM it

```bash
dbt run --select int_orders_enriched
```

---

## Concept 4 — Advanced Mart Models

### Mart 1 — `customer_summary_mart`

**`models/marts/customer_summary_mart.sql`**

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
    MAX(order_value_tier)                                   AS highest_order_tier,
    {{ dbt_utils.generate_surrogate_key(['customer_id']) }} AS customer_key
FROM {{ ref('int_orders_enriched') }}
GROUP BY customer_id, customer_name
```

```bash
dbt run --select customer_summary_mart
```

Verify:
```sql
SELECT customer_name, total_orders, lifetime_value, cancellation_rate_pct
FROM ECOMM_DB.MARTS.CUSTOMER_SUMMARY_MART
ORDER BY lifetime_value DESC
LIMIT 10;
```

---

### Mart 2 — `employees_mart`

**`models/marts/employees_mart.sql`**

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

    m.employee_name                                         AS manager_name,
    m.department                                            AS manager_department,

    CASE
        WHEN e.salary >= 90000 THEN 'Senior Band'
        WHEN e.salary >= 70000 THEN 'Mid Band'
        ELSE 'Junior Band'
    END                                                     AS salary_band,

    e.salary - AVG(e.salary) OVER (
        PARTITION BY e.department
    )                                                       AS salary_vs_dept_avg,

    RANK() OVER (
        PARTITION BY e.department
        ORDER BY e.salary DESC
    )                                                       AS salary_rank_in_dept,

    {{ dbt_utils.generate_surrogate_key(['e.employee_id']) }} AS employee_key

FROM {{ ref('stg_employees') }} e
LEFT JOIN {{ ref('stg_employees') }} m
    ON e.manager_id = m.employee_id
```

This model is a **self-join** — the `stg_employees` table is joined to itself to look up each employee's manager. `ref()` is used twice for the same model; dbt handles this correctly.

```bash
dbt run --select employees_mart
```

Verify:
```sql
SELECT employee_name, department, salary_band, manager_name, salary_rank_in_dept
FROM ECOMM_DB.MARTS.EMPLOYEES_MART
ORDER BY department, salary_rank_in_dept;
```

---

### Mart 3 — `products_mart` (upgraded from Day 3)

Replace the Day 3 version with an enriched version that uses `dbt_utils`:

**`models/marts/products_mart.sql`**

```sql
{{
    config(
        materialized = 'table',
        schema       = 'MARTS'
    )
}}

SELECT
    product_id,
    product_name,
    category,
    unit_price,
    stock_quantity,
    is_active,

    {{ classify_price('unit_price') }}                                  AS price_tier,
    {{ dbt_utils.generate_surrogate_key(['product_id', 'category']) }} AS product_key,

    CASE
        WHEN stock_quantity = 0  THEN 'Out of Stock'
        WHEN stock_quantity < 20 THEN 'Low Stock'
        ELSE 'In Stock'
    END                                                                 AS stock_status,

    unit_price * stock_quantity                                         AS inventory_value,

    RANK() OVER (
        PARTITION BY category
        ORDER BY unit_price DESC
    )                                                                   AS price_rank_in_category,

    SUM(unit_price * stock_quantity) OVER (
        PARTITION BY category
    )                                                                   AS category_inventory_value

FROM {{ ref('stg_products') }}
```

```bash
dbt run --select products_mart
```

---

### Add a `schema.yml` for the marts layer

**`models/marts/schema.yml`**

```yaml
version: 2

models:
  - name: orders_mart
    description: "Daily order summary aggregated from stg_orders. One row per order_date."
    columns:
      - name: order_date
        tests:
          - not_null
          - unique
      - name: total_orders
        tests:
          - not_null
      - name: total_revenue
        tests:
          - not_null

  - name: customer_summary_mart
    description: "One row per customer with lifetime order metrics and value tier."
    columns:
      - name: customer_id
        tests:
          - not_null
          - unique
      - name: customer_key
        tests:
          - not_null
          - unique
      - name: lifetime_value
        tests:
          - not_null
      - name: cancellation_rate_pct
        tests:
          - not_null

  - name: products_mart
    description: "Product catalogue enriched with price tiers, inventory value, and category rankings."
    columns:
      - name: product_id
        tests:
          - not_null
          - unique
      - name: product_key
        tests:
          - not_null
          - unique
      - name: price_tier
        tests:
          - not_null
          - accepted_values:
              values: ['Budget', 'Mid-range', 'Premium', 'Enterprise']

  - name: employees_mart
    description: "Employee directory with manager lookup, salary band, and department ranking."
    columns:
      - name: employee_id
        tests:
          - not_null
          - unique
      - name: employee_key
        tests:
          - not_null
          - unique
      - name: salary_band
        tests:
          - not_null
          - accepted_values:
              values: ['Junior Band', 'Mid Band', 'Senior Band']
```

---

## Concept 5 — Analyses

### What is an analysis?

An **analysis** is a `.sql` file inside the `analyses/` folder. It is compiled by dbt (Jinja is evaluated, `ref()` works) but it is **never materialized** — dbt does not create a table or view for it.

Analyses are for:
- Ad-hoc investigative queries you want to version-control alongside your models
- One-off reports that stakeholders request
- SQL you used to debug an issue or validate data quality

They are compiled to `target/compiled/` so you can copy the final SQL and run it in Snowsight.

### Create two analyses

**`analyses/revenue_by_category.sql`**

```sql
-- Revenue breakdown by product category for the full dataset.
-- Run with: dbt compile --select revenue_by_category
-- Then find compiled SQL in: target/compiled/ecomm_dbt/analyses/

SELECT
    p.category,
    COUNT(DISTINCT o.order_id)                              AS total_orders,
    SUM(o.total_amount)                                     AS total_revenue,
    AVG(o.total_amount)                                     AS avg_order_value,
    MIN(o.order_date)                                       AS first_order_date,
    MAX(o.order_date)                                       AS latest_order_date,
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

**`analyses/employee_headcount_by_department.sql`**

```sql
-- Headcount and salary summary per department.
-- Use this to answer: which departments are over/under the company average?

SELECT
    department,
    COUNT(*)                                                AS headcount,
    MIN(salary)                                             AS min_salary,
    MAX(salary)                                             AS max_salary,
    ROUND(AVG(salary), 2)                                   AS avg_salary,
    SUM(salary)                                             AS total_salary_cost,
    ROUND(AVG(salary) - AVG(AVG(salary)) OVER (), 2)        AS salary_vs_company_avg
FROM {{ ref('stg_employees') }}
WHERE status = 'active'
GROUP BY department
ORDER BY headcount DESC
```

### Compile analyses (does NOT run SQL)

```bash
dbt compile --select revenue_by_category
```

Then open the compiled SQL from:
```
target/compiled/ecomm_dbt/analyses/revenue_by_category.sql
```

Copy and paste it into Snowsight to run it. The `ref()` calls have been replaced with fully-qualified Snowflake paths.

---

## Concept 6 — Exposures

### What is an exposure?

An **exposure** declares a downstream consumer of your dbt models — a BI dashboard, a machine learning pipeline, an API, a spreadsheet. Exposures:

- Appear in the lineage graph as an **orange node** downstream of your mart models
- Help the team understand what breaks if a mart model changes
- Are declared in YAML — they create no Snowflake objects

### Where do exposures live?

Exposures are declared in any YAML file inside `models/`. By convention, create a dedicated `exposures.yml` file.

### Create `models/exposures.yml`

**`models/exposures.yml`**

```yaml
version: 2

exposures:

  - name: revenue_dashboard
    type: dashboard
    maturity: high
    url: https://your-bi-tool.company.com/dashboards/revenue
    description: >
      Executive revenue dashboard updated daily.
      Shows total revenue, orders by status, and top customers.
      Owned by the Data Analytics team.
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
      Product inventory and stock health report.
      Used by the Supply Chain team to flag low-stock items.
    owner:
      name: Supply Chain Team
      email: supply@company.com
    depends_on:
      - ref('products_mart')

  - name: employee_headcount_report
    type: analysis
    maturity: low
    description: >
      Ad-hoc headcount and salary analysis requested by HR.
      Runs monthly — compiled from analyses/employee_headcount_by_department.sql.
    owner:
      name: HR Team
      email: hr@company.com
    depends_on:
      - ref('employees_mart')

  - name: customer_churn_ml_pipeline
    type: ml
    maturity: low
    description: >
      Machine learning pipeline that scores customer churn probability.
      Reads from customer_summary_mart — lifetime_value and cancellation_rate_pct
      are the primary features.
    owner:
      name: ML Engineering Team
      email: ml@company.com
    depends_on:
      - ref('customer_summary_mart')
```

**Exposure `type` options:**

| Type | Use when |
|---|---|
| `dashboard` | A BI tool report (Tableau, Looker, Metabase) |
| `notebook` | A Jupyter or Hex notebook |
| `analysis` | An ad-hoc SQL report |
| `ml` | A machine learning pipeline |
| `application` | A product feature that reads the mart directly |

**Exposure `maturity` options:** `low`, `medium`, `high` — signals how production-critical this consumer is. A `high` maturity exposure means changes to its upstream models need careful review.

### View exposures in the lineage graph

```bash
dbt docs generate
dbt docs serve
```

In the lineage graph, click any mart model — orange nodes downstream of it are its exposures. Click an exposure node to see its `description`, `owner`, and `url`.

---

## Concept 7 — `dbt build` — The Production Command

### The problem with running commands separately

In development you typically run:
```bash
dbt seed
dbt run
dbt test
dbt snapshot
```

In production you want one command that:
1. Loads seeds
2. Runs all models in dependency order
3. Runs tests after each model's direct dependencies are tested
4. Skips downstream models if an upstream test fails

### `dbt build`

```bash
dbt build
```

`dbt build` combines `dbt seed` + `dbt run` + `dbt test` + `dbt snapshot` into one topologically-ordered execution. The key difference from running them separately:

> With separate commands: ALL models run first, THEN ALL tests run.
> With `dbt build`: tests run **immediately after their model is built** — if `stg_orders` tests fail, `int_orders_enriched` is SKIPPED before it even runs.

This is the correct production pattern. Failed data never propagates downstream.

### Build a specific subset

```bash
dbt build --select stg_orders+
```

The `+` after a model name means "this model AND all its downstream dependencies". This runs:
- `stg_orders` → its tests
- `int_orders_enriched` (depends on `stg_orders`) → its tests
- `customer_summary_mart` (depends on `int_orders_enriched`) → its tests
- `orders_mart` (depends on `stg_orders`) → its tests

### Build only the models that changed

```bash
dbt build --select state:modified+
```

Requires a previous run artifact in `target/`. dbt compares current SQL against the last compiled version and only rebuilds what changed. Used in CI pipelines (GitHub Actions, etc.) to avoid rebuilding everything on every pull request.

---

## Concept 8 — DAG, Node Selection, and Graph Operators

### The DAG (Directed Acyclic Graph)

Every model, test, seed, snapshot, and exposure is a **node** in dbt's DAG. The edges between nodes are the `ref()` and `source()` calls. dbt uses this graph to determine execution order — a model only runs after all its dependencies are built.

```
[source: orders_raw]   [seed: products]   [seed: employees]
         ↓                    ↓                   ↓
    stg_orders          stg_products        stg_employees
         ↓    ↘                                   ↓
         ↓   int_orders_enriched             employees_mart
         ↓          ↓
    orders_mart  customer_summary_mart
                        ↓
              [exposure: revenue_dashboard]
```

### Graph operators

| Operator | Meaning | Example |
|---|---|---|
| `model` | Only this model | `--select stg_orders` |
| `model+` | This model and all its **downstream** nodes | `--select stg_orders+` |
| `+model` | All **upstream** nodes of this model | `--select +customer_summary_mart` |
| `+model+` | All upstream AND all downstream | `--select +orders_mart+` |
| `@model` | The model, its ancestors, and all tests on those ancestors | `--select @stg_orders` |

### Tags

You can tag models in `dbt_project.yml` or in a model's `config()` block to run named subsets:

**In `dbt_project.yml`:**
```yaml
models:
  ecomm_dbt:
    staging:
      +tags: ["staging", "daily"]
    marts:
      +tags: ["marts", "daily"]
```

**In a model's config block:**
```sql
{{ config(tags=["high_priority", "finance"]) }}
```

**Run only tagged models:**
```bash
dbt build --select tag:daily
dbt build --select tag:high_priority
dbt build --exclude tag:marts
```

### Practical selection patterns

```bash
# Rebuild only the intermediate + mart layer (skip staging)
dbt build --select intermediate+ marts

# Rebuild everything upstream of the customer_summary_mart (find the root cause of an issue)
dbt build --select +customer_summary_mart

# Run only the marts layer tests without rebuilding
dbt test --select marts

# Full production run
dbt build

# Full production run, skipping seeds (seeds rarely change in prod)
dbt build --exclude seeds
```

---

## Final File Structure After Day 4

```
ecomm_dbt/
├── packages.yml                         ← NEW: declares dbt-utils
├── dbt_project.yml                      ← UPDATED: adds intermediate layer
├── seeds/
│   ├── products.csv
│   └── employees.csv
├── models/
│   ├── staging/
│   │   ├── sources.yml
│   │   ├── schema.yml
│   │   ├── stg_orders.sql
│   │   ├── stg_orders_incremental.sql
│   │   ├── stg_products.sql
│   │   └── stg_employees.sql
│   ├── intermediate/                    ← NEW layer
│   │   └── int_orders_enriched.sql     ← NEW
│   ├── marts/
│   │   ├── schema.yml                  ← UPDATED: adds new mart tests
│   │   ├── orders_mart.sql
│   │   ├── products_mart.sql           ← UPDATED: uses dbt_utils
│   │   ├── customer_summary_mart.sql   ← NEW
│   │   ├── employees_mart.sql          ← NEW
│   │   ├── date_spine_mart.sql         ← NEW
│   │   └── orders_by_category_pivot.sql ← NEW
│   └── exposures.yml                   ← NEW
├── analyses/
│   ├── revenue_by_category.sql         ← NEW
│   └── employee_headcount_by_department.sql ← NEW
├── tests/
│   ├── assert_product_price_is_positive.sql
│   ├── assert_stock_not_negative.sql
│   ├── assert_employee_salary_in_range.sql
│   └── assert_manager_exists.sql
├── macros/
│   ├── generate_schema_name.sql
│   ├── cents_to_dollars.sql
│   ├── classify_price.sql
│   ├── is_working_day.sql
│   ├── generate_surrogate_key.sql
│   ├── test_not_negative.sql
│   ├── test_accepted_range.sql
│   └── test_not_empty_string.sql
└── snapshots/
    └── products_snapshot.sql
```

---

## Run Everything Together (Day 4 End-to-End)

```bash
# 1. Install packages (one-time, or after packages.yml changes)
dbt deps

# 2. Load seeds
dbt seed

# 3. Build all models + run all tests in dependency order
dbt build

# 4. Generate and serve docs (shows exposures in lineage graph)
dbt docs generate
dbt docs serve
```

If `dbt build` passes clean:
```
Done. PASS=N WARN=0 ERROR=0 FAIL=0 SKIP=0 TOTAL=N
```

Verify new marts in Snowflake:
```sql
-- Customer summary
SELECT customer_name, total_orders, lifetime_value, cancellation_rate_pct
FROM ECOMM_DB.MARTS.CUSTOMER_SUMMARY_MART
ORDER BY lifetime_value DESC;

-- Employees with manager and salary band
SELECT employee_name, department, salary_band, manager_name, salary_rank_in_dept
FROM ECOMM_DB.MARTS.EMPLOYEES_MART
ORDER BY department, salary_rank_in_dept;

-- Products with inventory value and category rank
SELECT product_name, category, price_tier, inventory_value, price_rank_in_category
FROM ECOMM_DB.MARTS.PRODUCTS_MART
ORDER BY category, price_rank_in_category;
```

---

## Key Differences — Analyses vs Models

| | Model (`.sql` in `models/`) | Analysis (`.sql` in `analyses/`) |
|---|---|---|
| **Materialized?** | Yes — creates table/view/incremental in Snowflake | No — compiled but never executed by dbt |
| **`ref()` works?** | Yes | Yes |
| **Appears in lineage?** | Yes | No |
| **Tested?** | Yes via `schema.yml` | No |
| **Best for** | Persistent datasets consumed by BI or other models | Ad-hoc queries, investigative SQL, one-off reports |

---

## Key Differences — Packages vs Custom Macros

| | Custom macro (`macros/`) | dbt package (`dbt_packages/`) |
|---|---|---|
| **Who maintains it?** | You | dbt Labs or community |
| **Adapter-aware?** | Only if you write adapter-specific logic | Yes — dbt-utils handles Snowflake vs BigQuery differences |
| **Versioned?** | Only by your git history | Yes — pinned in `packages.yml` |
| **When to use** | Business-specific logic (e.g. `classify_price`, `cents_to_dollars`) | Generic utilities (surrogate keys, date spines, pivots) |

---

## Summary

| Concept | Key takeaway |
|---|---|
| `packages.yml` + `dbt deps` | Declare and install community packages — never commit `dbt_packages/` to git |
| `dbt_utils.generate_surrogate_key` | Adapter-aware surrogate key — prefer over custom MD5 macro |
| `dbt_utils.date_spine` | Generates a continuous date sequence — use to fill gaps in time-series marts |
| `dbt_utils.pivot` | Turns row values into columns — combine with `get_column_values` to avoid hardcoding |
| Intermediate layer | Joins and enrichment go here — marts stay clean aggregations, no raw logic |
| Self-join with `ref()` | Use `ref()` twice for the same model — dbt handles it without circular dependency |
| Window functions in marts | `RANK()`, `AVG() OVER()` belong in mart SQL, not staging |
| Analyses | Version-controlled SQL that is compiled but never materialized — `dbt compile` to get runnable SQL |
| Exposures | Declare BI dashboards, ML pipelines in YAML — appear as orange nodes in lineage graph |
| `dbt build` | The production command — seeds + models + tests + snapshots in one topologically-ordered run |
| `model+` selector | Run a model and all its downstream dependents |
| `+model` selector | Run all upstream ancestors of a model |
| Tags | Group models with `+tags:` in `dbt_project.yml` — select with `--select tag:name` |
| `state:modified+` | Only build what changed — the CI/CD pattern for fast feedback |
