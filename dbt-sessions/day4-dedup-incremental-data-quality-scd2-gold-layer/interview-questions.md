# Day 4 — Interview Questions: Dedup, Incremental, Data Quality, SCD Type 2, and Gold Layer

> 18 questions. Difficulty: (Warm-up) → (Conceptual) → (Scenario) → (Tricky)
> All scenarios use the ecomm_dbt project built in Day 4.

---

**Q1 (Warm-up)** — What is deduplication in the context of a dbt staging model? Why do you deduplicate in the staging layer rather than the gold layer?

**Q2 (Warm-up)** — Write the SQL pattern used to deduplicate rows in dbt staging models. Explain what `PARTITION BY`, `ORDER BY`, and `WHERE row_num = 1` each do in this pattern.

**Q3 (Conceptual)** — What is an incremental model in dbt? How is the first run different from subsequent runs? What does `unique_key` do in an incremental model?

**Q4 (Scenario)** — Your `stg_orders` is an incremental model with `unique_key = 'order_id'`. An existing order changes its `status` from `CONFIRMED` to `SHIPPED` in the source. On the next `dbt run`, what happens? What SQL does dbt generate behind the scenes?

**Q5 (Warm-up)** — What is the difference between a generic test and a singular test in dbt? Give one example of each from the Day 4 project.

**Q6 (Conceptual)** — Your `test_not_negative` custom generic test is defined in `macros/`. How does dbt know it is a test macro and not a regular macro? How do you attach it to a column?

**Q7 (Scenario)** — You run `dbt test --select staging` and see three failures:
- `not_negative` on `stg_products.unit_price`
- `assert_order_amount_matches_quantity_x_price`
- `assert_customer_exists_for_order`

The staging tables still contain the bad rows. What do you do in the gold layer to prevent these bad rows from reaching `fct_orders` and `dim_products`?

**Q8 (Conceptual)** — What is SCD Type 2? What four columns does dbt add automatically to a snapshot table? How do you query only the **current** version of each row?

**Q9 (Scenario)** — A customer moves from New York to Los Angeles. Walk through exactly what happens in `customers_snapshot` when you re-run `dbt snapshot` after updating the seed data. How many rows exist for that customer after the snapshot runs, and what are their `dbt_valid_to` values?

**Q10 (Conceptual)** — Why does `dim_customers` read from `customers_snapshot` rather than directly from `stg_customers`? What would you lose if you read from staging instead?

**Q11 (Warm-up)** — What is a surrogate key? Why do dimension tables use `MD5(CAST(id AS VARCHAR))` instead of using the natural `customer_id` directly as the join key?

**Q12 (Conceptual)** — Explain the difference between a dimension table and a fact table. In the Day 4 project, which tables are dimensions and which is the fact? What columns link them?

**Q13 (Scenario)** — You run `dbt build` and `stg_orders` passes all tests, but `assert_customer_exists_for_order` fails. `fct_orders` depends on `stg_orders`. Does `fct_orders` get built? What would happen if you ran `dbt run` and `dbt test` as separate commands instead?

**Q14 (Tricky)** — Your `stg_orders` is incremental. A data engineer accidentally deletes 3 rows from `orders_raw` (the seed) and re-runs `dbt seed` and then `dbt run`. Will those 3 rows disappear from `stg_orders`? Why or why not? How do you fix it?

**Q15 (Tricky)** — In `stg_orders`, the deduplication CTE runs `ROW_NUMBER()` and then the incremental filter applies `WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})`. If a duplicate order arrives with the same `order_id` but a newer `order_date`, which row wins the dedup and does it make it past the incremental filter? Walk through step by step.

**Q16 (Scenario)** — The business asks: "What city was customer Alice in when she placed her order on 2024-02-01?" Write the SQL query against the snapshot table that answers this question.

**Q17 (Conceptual)** — What is a star schema? Draw the relationship between `fct_orders`, `dim_customers`, and `dim_products`. Why is the fact table joined to dimensions using surrogate keys rather than natural keys?

**Q18 (Tricky)** — You have a singular test `assert_customer_exists_for_order` that catches orphan foreign keys in staging. You also have a WHERE clause in `fct_orders` that filters them out. Isn't this redundant — why do both? What is the role of each?
