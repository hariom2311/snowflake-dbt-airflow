# Day 4 — Interview Solutions: Dedup, Incremental, Data Quality, SCD Type 2, and Gold Layer

---

**A1** — Deduplication is the process of removing duplicate rows so that each business entity (customer, product, order) appears exactly once. It belongs in the staging layer — not the gold layer — because:

1. **Catch it early.** Every downstream model that reads staging inherits clean data automatically. If you deduplicate in gold, you have to remember to do it in every gold model.
2. **Single responsibility.** Staging's job is to clean and normalize raw data. Gold's job is to apply business logic and aggregate. Mixing concerns makes models harder to debug.
3. **Tests run on staging.** `unique` tests on `stg_customers.customer_id` will fail if deduplication is missing, alerting you immediately.

---

**A2** — The standard deduplication pattern:

```sql
WITH deduplicated AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id    -- group all rows with the same business key
            ORDER BY created_at DESC    -- most recent row wins (row_num = 1)
        ) AS row_num
    FROM source
)
SELECT * FROM deduplicated WHERE row_num = 1
```

- `PARTITION BY customer_id` — resets the row counter for each unique `customer_id`
- `ORDER BY created_at DESC` — the tiebreaker determines which duplicate is "best". Descending puts the most recent row first (row_num = 1). For exact duplicates with no reliable tiebreaker, any `ORDER BY` column works since all rows are identical anyway
- `WHERE row_num = 1` — keeps exactly one row per `customer_id` — the one ranked first

---

**A3** — An incremental model builds a table that grows over time by processing only new rows, rather than rebuilding the entire table from scratch on every run.

- **First run:** dbt creates the table with ALL rows (no incremental filter applied — `is_incremental()` returns FALSE)
- **Subsequent runs:** dbt only fetches rows newer than what is already in the table, then INSERTs or MERGEs them in (`is_incremental()` returns TRUE, the WHERE block is applied)

`unique_key = 'order_id'` controls what happens when a row already exists in the table:
- **Without `unique_key`:** dbt always INSERTs — duplicates accumulate
- **With `unique_key`:** dbt generates a MERGE — if `order_id` already exists, UPDATE the row; if it is new, INSERT it

---

**A4** — The existing order row (`order_id=X, status=CONFIRMED`) is already in `stg_orders`. On the next run:

1. The source returns the row again with `status=SHIPPED` and a newer `updated_at`
2. The incremental WHERE filter passes it through (it is "new" relative to the max date)
3. Because `unique_key = 'order_id'` is set, dbt generates a MERGE statement:

```sql
MERGE INTO stg_orders AS target
USING (SELECT ... WHERE order_date > ...) AS source
ON target.order_id = source.order_id
WHEN MATCHED THEN UPDATE SET status = 'SHIPPED', ...
WHEN NOT MATCHED THEN INSERT (order_id, status, ...) VALUES (...)
```

Result: the existing row is UPDATED from `CONFIRMED` to `SHIPPED`. No duplicate is created.

---

**A5** —

| | Generic test | Singular test |
|---|---|---|
| **Location** | Defined in `macros/` as `{% test name %}`, attached in `schema.yml` | `.sql` file in `tests/` |
| **Reusable?** | Yes — attach to any column on any model | No — one test, one specific check |
| **Example (Day 4)** | `not_negative` on `stg_products.unit_price` | `assert_order_amount_matches_quantity_x_price.sql` |

---

**A6** — dbt identifies a test macro by its name prefix: the macro must be named `test_<name>` (e.g. `test_not_negative`). When dbt encounters `- not_negative` in `schema.yml`, it looks for a macro named `test_not_negative` in the `macros/` folder and calls it, passing `model` and `column_name` automatically.

You attach it in `schema.yml` exactly like a built-in test:

```yaml
- name: unit_price
  tests:
    - not_negative
```

No other configuration needed. dbt generates and runs the SQL from the macro body, substituting the actual model and column name at runtime.

---

**A7** — Tests report failures — they do NOT delete rows from staging tables. The staging table still contains the bad rows after a failed test. The gold layer is responsible for explicitly excluding them via WHERE clauses:

**In `dim_products`:**
```sql
WHERE unit_price > 0    -- excludes product_id=7 with unit_price=-5.00
```

**In `fct_orders`:**
```sql
WHERE customer_id IN (SELECT customer_id FROM {{ ref('stg_customers') }})
  AND ABS(total_amount - ROUND(quantity * unit_price, 2)) <= 0.01
```

The pattern is: **tests flag problems at the staging gate; gold model WHERE clauses filter them out**. Both layers have a role — tests alert the engineer; the WHERE clause protects the BI layer.

---

**A8** — SCD Type 2 (Slowly Changing Dimension Type 2) is a technique for tracking the full history of changes to a row. Instead of overwriting the old value, a new row is inserted with the new value, and the old row is "closed" with an end timestamp.

Four columns dbt adds automatically:

| Column | Meaning |
|---|---|
| `dbt_scd_id` | Unique hash per version of a row |
| `dbt_updated_at` | When dbt detected this version |
| `dbt_valid_from` | When this version became current |
| `dbt_valid_to` | When superseded — `NULL` means it is the current row |

To query only the current version:
```sql
WHERE dbt_valid_to IS NULL
```

---

**A9** — Before the change: 1 row for Alice with `dbt_valid_to = NULL`.

After updating `city` from `New York` to `Los Angeles`, re-seeding, re-running staging, and running `dbt snapshot`:

| Row | city | dbt_valid_from | dbt_valid_to |
|---|---|---|---|
| 1 (old) | New York | 2024-11-28 10:00:00 | 2024-11-28 10:30:00 ← closed |
| 2 (new) | Los Angeles | 2024-11-28 10:30:00 | NULL ← current |

There are now **2 rows** for Alice. The old row is closed (`dbt_valid_to` is set). The new row is open (`dbt_valid_to = NULL`). This is the SCD Type 2 history preserved — you can query what Alice's city was at any point in time.

---

**A10** — `dim_customers` reads from `customers_snapshot` because the snapshot is the **history store** — it tracks every version of every customer. `dim_customers` applies `WHERE dbt_valid_to IS NULL` to get only the current version, turning the history into a clean current-state dimension.

If `dim_customers` read directly from `stg_customers`, it would still show current data — but you would **lose the ability to do point-in-time analysis**. For example: "What city was Alice in when she placed her order in January?" cannot be answered without the snapshot. The snapshot also acts as an audit trail of every change.

---

**A11** — A surrogate key is a system-generated identifier (usually a hash or sequence) that uniquely identifies a row in a dimension table. It is used in place of the natural key (like `customer_id`) as the join key between fact and dimension tables.

Reasons to use `MD5(CAST(customer_id AS VARCHAR))` over raw `customer_id`:

1. **Source system conflicts.** If you ever merge data from two source systems, both may have `customer_id = 1`. The surrogate key hashes the source + the id together, keeping them distinct
2. **Data type consistency.** Fact and dimension tables from different sources may have `customer_id` as INT in one and VARCHAR in another. The MD5 hash is always a VARCHAR of fixed length
3. **Dimensional modeling convention.** Industry standard — fact tables reference dimension rows by surrogate key, never by natural key

---

**A12** — 

**Dimension table** — describes *what* an entity is. Contains attributes about the entity that do not change frequently. One row per entity. Used for filtering and grouping in reports.

**Fact table** — records *what happened*. Contains measurable events (orders, transactions, clicks) with numeric metrics (quantity, revenue) and foreign keys to dimensions.

In the Day 4 project:
- Dimensions: `dim_customers` (who is the customer), `dim_products` (what is the product)
- Fact: `fct_orders` (what order happened)

Linking columns (the join keys):
```
fct_orders.customer_key → dim_customers.customer_key
fct_orders.product_key  → dim_products.product_key
```

---

**A13** — With `dbt build`: **`fct_orders` is SKIPPED**. dbt sees that `assert_customer_exists_for_order` failed on `stg_orders`. Even though `stg_orders` itself built successfully, its test failure marks all downstream nodes (including `fct_orders`) as blocked. `fct_orders` never runs.

With `dbt run` + `dbt test` as separate commands: `fct_orders` **builds and runs** even though the test fails — because `dbt run` does not check test results. Only when `dbt test` runs afterward do you see the failure, but `fct_orders` already contains the bad row (`order_id=11` with `customer_id=99`).

This is why `dbt build` is the correct production command — bad data never propagates downstream.

---

**A14** — Those 3 rows **will NOT disappear** from `stg_orders`. Incremental models only ADD rows — they never delete. The incremental filter `WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})` only looks for rows newer than what is in the table. It never checks whether existing rows have been removed from the source.

To fix it:

```bash
dbt run --select stg_orders --full-refresh
```

`--full-refresh` drops and rebuilds the entire incremental table from scratch. Use this whenever:
- Rows are deleted from the source
- You change the model's SQL logic that affects historical rows
- The table schema changes (new columns)

---

**A15** — Step by step:

1. The deduplication CTE runs first, over ALL rows from `orders_raw`
2. Both duplicates (`order_id=5`, `order_date=2024-02-20`) and the new row (`order_id=5`, `order_date=2024-03-01`) are in the CTE
3. `ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY order_id)` — with two rows having the same `order_id`, the tiebreaker `ORDER BY order_id` is not meaningful (same value). The database picks one arbitrarily as `row_num=1`

The issue here is a weak tiebreaker. In the Day 4 model the tiebreaker is just `ORDER BY order_id`, which does not distinguish between the two duplicates. In a real project you would use a more meaningful tiebreaker like `ORDER BY updated_at DESC` so the most recent version wins.

4. After dedup: 1 row for `order_id=5` with `order_date=2024-02-20` (original)
5. Incremental filter: `WHERE order_date > MAX(order_date already in table)` — if `2024-02-20` is already in the table, this row does NOT make it past the incremental filter
6. Because `unique_key='order_id'` is set, even if it did make it through, it would MERGE/UPDATE rather than insert a duplicate

---

**A16** —

```sql
SELECT
    s.customer_name,
    s.city,
    s.country
FROM ECOMM_DB.SNAPSHOTS.CUSTOMERS_SNAPSHOT s
WHERE s.customer_id = 1
  AND s.dbt_valid_from <= '2024-02-01'
  AND (s.dbt_valid_to > '2024-02-01' OR s.dbt_valid_to IS NULL);
```

This finds the row version that was **active on 2024-02-01** — meaning it had already started (`dbt_valid_from <= target_date`) and had not yet been closed (`dbt_valid_to > target_date`, or still open). Result: `New York`, which was Alice's city before she moved to Los Angeles.

---

**A17** — A star schema is a database design pattern where a central fact table is surrounded by dimension tables, connected via foreign keys (surrogate keys). The shape resembles a star when drawn — fact in the middle, dimensions radiating outward.

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

Why surrogate keys instead of natural keys:
- Natural keys (`customer_id=1`) can collide when merging data from multiple source systems
- Surrogate keys are always unique across all sources and data types
- Dimension tables can be updated (new surrogate key for a new SCD version) without changing the fact table's join logic

---

**A18** — They serve different purposes and neither is redundant:

**The singular test (`assert_customer_exists_for_order`):**
- Runs at the **staging layer**
- **Alerts the data engineer** that there is a data quality problem in the source
- Answers: "Is our raw data clean?" and "Do we have an upstream ETL bug?"
- Action: someone needs to fix the source system or the EL pipeline

**The WHERE clause in `fct_orders`:**
- Runs at the **gold layer**
- **Protects BI tools and analysts** from ever seeing bad data
- Answers: "Even if the source has a problem, is our gold layer clean?"
- Action: automatic — no intervention needed, bad rows are silently excluded

Together: the test raises an alarm (fix the source), the WHERE clause is a safety net (BI never sees bad data in the meantime). Removing the test means silent failures — bad data arrives, gold excludes it, but no one is alerted that the source is broken. Removing the WHERE clause means bad data reaches BI tools the moment a test is missed or bypassed.
