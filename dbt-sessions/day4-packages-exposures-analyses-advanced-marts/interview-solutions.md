# Day 4 — Interview Solutions: Packages, Exposures, Analyses, and Advanced Mart Modeling

---

**A1** — A dbt package is a collection of macros, models, and tests published to [hub.getdbt.com](https://hub.getdbt.com). You declare packages in `packages.yml` at the root of your dbt project, then run `dbt deps` to download them into `dbt_packages/`. Never commit `dbt_packages/` to git.

```yaml
# packages.yml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.3.0
```

```bash
dbt deps
```

---

**A2** — dbt-utils is a utility package maintained by dbt Labs (the company that created dbt). Three commonly-used macros:

| Macro | What it does |
|---|---|
| `dbt_utils.generate_surrogate_key(['col1', 'col2'])` | Hashes a list of columns into a stable surrogate key, adapter-aware |
| `dbt_utils.date_spine(datepart, start_date, end_date)` | Generates a table of consecutive dates between two boundaries |
| `dbt_utils.pivot(column, values, agg, then_value)` | Converts distinct row values into columns using conditional aggregation |

---

**A3** — Both produce a surrogate key by hashing columns. The difference:

| | Custom macro (Day 3) | `dbt_utils.generate_surrogate_key` |
|---|---|---|
| Hash function | Always `MD5` (Snowflake-specific) | Adapter-aware — `MD5` on Snowflake, `FARM_FINGERPRINT` on BigQuery |
| Maintained by | You | dbt Labs |
| Versioned | Only via git | Pinned in `packages.yml` |

Prefer `dbt_utils` for the surrogate key because it is adapter-portable. Keep custom macros (`classify_price`, `cents_to_dollars`) for business-specific logic that no community package covers.

---

**A4** — An analysis is a `.sql` file inside the `analyses/` folder. It is compiled by dbt (Jinja is evaluated, `ref()` works) but is **never materialized** — dbt does not create any Snowflake object for it. It is for ad-hoc, investigative, or one-off queries you want to version-control alongside your models. A model in `models/` creates a table, view, or incremental table; an analysis creates nothing.

---

**A5** — You run:

```bash
dbt compile --select revenue_by_category
```

The compiled SQL is written to:
```
target/compiled/ecomm_dbt/analyses/revenue_by_category.sql
```

All `ref()` calls are replaced with their fully-qualified Snowflake paths. You copy this file and paste it into Snowsight to run it. `dbt compile` does not execute any SQL against Snowflake.

---

**A6** — An exposure declares a **downstream consumer** of your dbt models — a BI dashboard, an ML pipeline, an API, or a notebook. It creates no Snowflake objects; it is metadata only. In `dbt docs serve`, exposures appear as **orange nodes** in the lineage graph, downstream of the mart models they depend on. Clicking an exposure shows its description, owner, URL, and which models it reads from.

---

**A7** — Valid exposure types and real-world examples:

| Type | Example |
|---|---|
| `dashboard` | A Tableau / Metabase revenue dashboard reading from `orders_mart` |
| `notebook` | A Jupyter notebook for exploratory data analysis on `customer_summary_mart` |
| `analysis` | A monthly headcount report run from `analyses/employee_headcount_by_department.sql` |
| `ml` | A churn prediction pipeline that reads `cancellation_rate_pct` from `customer_summary_mart` |
| `application` | A product feature that calls `products_mart` directly for inventory display |

---

**A8** — `dbt build` combines seed + run + test + snapshot into one **topologically-ordered** execution. The critical difference: with `dbt build`, tests run **immediately after their model is built** — if `stg_orders` tests fail, `int_orders_enriched` is SKIPPED before it even starts. With separate `dbt run` then `dbt test`, ALL models run first (including `int_orders_enriched`), and THEN tests run. Failed data can propagate downstream before tests catch it.

---

**A9** — With `dbt build`: `customer_summary_mart` is **SKIPPED**. dbt sees that `int_orders_enriched` failed its tests, marks all downstream nodes as errored/skipped, and does not run them. The final output shows `SKIP=1` for `customer_summary_mart`.

With separate `dbt run` + `dbt test`: `customer_summary_mart` **runs and builds successfully** even though `int_orders_enriched` has bad data — because `dbt run` does not check test results. Only when `dbt test` runs afterward do you see the `int_orders_enriched` failure, but by then `customer_summary_mart` already contains corrupted data.

This is why `dbt build` is the correct production pattern.

---

**A10** —

| Selector | Meaning | Use when |
|---|---|---|
| `+model` | All **upstream** ancestors of this model | You want to trace the root cause — rebuild everything that feeds into a broken model |
| `model+` | This model AND all its **downstream** dependents | You changed a staging model and want to cascade the rebuild through all its consumers |
| `+model+` | All upstream AND all downstream | Full end-to-end rebuild of an entire pipeline branch |

---

**A11** —

```bash
dbt build --select int_orders_enriched+
```

The `+` after the model name means "this model and all downstream nodes". dbt builds `int_orders_enriched`, then all models that depend on it (`customer_summary_mart`, etc.), in topological order, running tests after each.

---

**A12** — The intermediate layer is where **joins, enrichments, and row-level transformations** happen — logic that is too complex for staging (which should be clean, one-to-one with the source) but too granular for marts (which should be clean aggregations). The `int_` prefix is a naming convention that signals "this is not for end consumers — only mart models should ref this".

Intermediate models should use `materialized='view'` (or `ephemeral`). They are stepping stones, not end-products — materializing them as tables wastes storage and adds rebuild time for data no BI tool reads directly.

---

**A13** — No, dbt does not throw a circular dependency error. A self-join references the same model twice as two different **aliases** (`e` and `m`). dbt's DAG has only one node for `stg_employees`, and `employees_mart` depends on it once. The two `ref()` calls both resolve to the same node — there is no cycle.

A circular dependency would require model A to `ref()` model B while model B `ref()`s model A — that is impossible here since both references point to the same model, and that model does not reference `employees_mart`.

---

**A14** — `dbt_utils.get_column_values` runs at **compile time** (`dbt compile` or the compile phase of `dbt run`), not at execution time. It issues a `SELECT DISTINCT` query against Snowflake to fetch the column values before any models are run.

The implication: if `stg_orders` has not been built yet (first run on a fresh Snowflake environment), `get_column_values` fails because the table does not exist. The fix is to run `dbt run --select stg_orders` first, or use a hardcoded list for the first run, then switch to `get_column_values` afterward.

---

**A15** — They get a macro not found error like:
```
Compilation Error in model products_mart (models/marts/products_mart.sql)
  'dbt_utils' is undefined
```

To prevent this in CI: add `dbt deps` as a step **before** `dbt build` in your pipeline:

```yaml
# GitHub Actions example
- name: Install dbt packages
  run: dbt deps

- name: Build dbt project
  run: dbt build
```

---

**A16** — `state:modified+` selects only models whose SQL or config has changed since the last successful run, plus all their downstream dependents. It requires a `manifest.json` from the last run to exist (usually downloaded as a CI artifact from the previous successful pipeline run).

Use it in pull request CI pipelines to avoid rebuilding the entire project on every PR — only the changed models and their downstream impact are tested:

```bash
dbt build --select state:modified+
```

This makes CI dramatically faster on large projects. Without it, every PR rebuilds everything even if only one staging model changed.

---

**A17** — The exposure in `exposures.yml` does not automatically detect column renames. What it does is make the **blast radius visible in the lineage graph**: anyone who opens `dbt docs serve` can see that `customer_summary_mart` has a downstream consumer (`revenue_dashboard`) with `maturity: high`. This signals that the rename needs to be coordinated with the BI team before merging.

Best practice: before renaming a column in a mart that has a `high` maturity exposure, check the exposure's `owner.email` and notify them. Some teams add a CI check that fails if a mart column referenced in an exposure YAML is renamed without updating the exposure description.

---

**A18** — Use Jinja's `run_query` or simply use Snowflake's `CURRENT_DATE()` inside the macro call with string formatting:

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
        end_date   = "cast('" ~ modules.datetime.date.today().strftime('%Y-%m-%d') ~ "' as date)"
    ) }}
)

SELECT date_day AS calendar_date
FROM date_spine
```

`modules.datetime` is available in dbt's Jinja context and gives you the current date at compile time. Alternatively, for Snowflake specifically, `end_date = "current_date()"` works directly in the date_spine macro because it passes the expression as raw SQL:

```sql
{{ dbt_utils.date_spine(
    datepart   = "day",
    start_date = "cast('2024-01-01' as date)",
    end_date   = "current_date()"
) }}
```

This way the spine always extends to today without any hardcoded year.
