# Day 4 — Interview Questions: Packages, Exposures, Analyses, and Advanced Mart Modeling

> 18 questions. Difficulty: (Warm-up) → (Conceptual) → (Scenario) → (Tricky)
> All scenarios use the ecomm_dbt project from Days 1–3.

---

**Q1 (Warm-up)** — What is a dbt package? How do you install packages, and which file do you declare them in?

**Q2 (Warm-up)** — What is dbt-utils and who maintains it? Name three macros from dbt-utils and describe what each does in one sentence.

**Q3 (Conceptual)** — What is the difference between `dbt_utils.generate_surrogate_key` and the custom `generate_surrogate_key` macro you wrote in Day 3? When would you prefer one over the other?

**Q4 (Warm-up)** — What is an **analysis** in dbt? How is it different from a model?

**Q5 (Conceptual)** — You write `{{ ref('stg_orders') }}` inside `analyses/revenue_by_category.sql`. What command do you run to see the final compiled SQL? Where is the compiled file written to?

**Q6 (Warm-up)** — What is an **exposure** in dbt? What does it represent, and where in the dbt docs UI does it appear?

**Q7 (Conceptual)** — What are the valid `type` values for a dbt exposure? Give a real-world example for each type.

**Q8 (Warm-up)** — What does `dbt build` do that running `dbt seed`, `dbt run`, and `dbt test` separately does not?

**Q9 (Scenario)** — You run `dbt build` and `stg_orders` passes all tests, but `int_orders_enriched` (which depends on `stg_orders`) fails a test. What happens to `customer_summary_mart` (which depends on `int_orders_enriched`)? What would happen if you ran `dbt run` + `dbt test` as separate commands instead?

**Q10 (Conceptual)** — Explain the difference between the `+model` selector, the `model+` selector, and the `+model+` selector. When would you use each?

**Q11 (Scenario)** — You need to rebuild the entire downstream pipeline starting from `int_orders_enriched` — that model and everything that depends on it. Write the exact dbt command you would use.

**Q12 (Conceptual)** — What is the purpose of an **intermediate layer** in a dbt project? Why do you name models with an `int_` prefix? What materialization should intermediate models typically use and why?

**Q13 (Scenario)** — Your `employees_mart` model uses a self-join — it references `{{ ref('stg_employees') }}` twice (once for the employee, once for the manager). Will dbt throw a circular dependency error? Why or why not?

**Q14 (Tricky)** — You use `dbt_utils.get_column_values(table=ref('stg_orders'), column='product')` inside a model. When exactly does this query run — at `dbt run` time, or at `dbt compile` time? What are the implications if `stg_orders` has not been built yet?

**Q15 (Tricky)** — You add `packages.yml` to your project and run `dbt deps`. A team member checks out your repository fresh and runs `dbt run` without running `dbt deps` first. What error do they get and how do you prevent this in a CI pipeline?

**Q16 (Conceptual)** — What is `state:modified+` in dbt node selection? When would you use it and what does it require to work?

**Q17 (Scenario)** — You declare an exposure for a Tableau dashboard that reads from `customer_summary_mart`. A new analyst changes the `lifetime_value` column name to `total_lifetime_value` in the mart model. How does the exposure in `exposures.yml` help the team understand the impact of this change before merging?

**Q18 (Tricky)** — You have a `date_spine_mart` built with `dbt_utils.date_spine`. The model runs today and covers `2024-01-01` to `2024-12-31`. Six months from now, the spine is out of date. How would you make the `end_date` dynamic so it always covers up to the current date, without hardcoding the year?
