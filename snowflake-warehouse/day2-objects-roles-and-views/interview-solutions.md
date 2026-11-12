# Day 2 — Interview Solutions: Objects, Roles, and Views

> Complete answers to all 30 questions. Read these after attempting the questions yourself.

---

## Concept 1: Databases, Tables, and Object Types

**Q1 — Four table types in Snowflake?**

| Type | One-line description |
|---|---|
| **Permanent** | Default table type. Full Time Travel (up to 90 days) + 7-day Fail-safe. For production data. |
| **Transient** | No Fail-safe, Time Travel up to 1 day. Cheaper storage. For staging/intermediate data. |
| **Temporary** | Lives only for the current session. Dropped automatically at session end. For scratch work. |
| **External** | A pointer to files in cloud storage (S3/Blob/GCS). Data is NOT inside Snowflake. Read-only. |

---

**Q2 — Difference between Time Travel and Fail-safe?**

**Time Travel:** A user-controlled feature that lets you query, restore, or clone data as it existed at a past point in time. You control the retention period (0–90 days on Enterprise). You trigger it yourself using `AT (TIMESTAMP => ...)` or `UNDROP TABLE`.

**Fail-safe:** A Snowflake-managed safety net that kicks in AFTER Time Travel expires. It provides an additional 7 days of data recovery, but only Snowflake Support can recover data from Fail-safe — you cannot access it yourself. It is a last-resort disaster recovery mechanism, not a self-service tool.

| Feature | Time Travel | Fail-safe |
|---|---|---|
| Duration | 0–90 days (configurable) | 7 days (fixed) |
| Who triggers recovery | You (SQL) | Snowflake Support only |
| Table types | Permanent + Transient | Permanent only |
| Storage cost | Yes | Yes |

---

**Q3 — Why use TRANSIENT for staging instead of permanent?**

Staging tables are recreated on every ETL run — typically truncated and reloaded nightly. They are throwaway intermediate state.

**What you gain with TRANSIENT:**
- **No Fail-safe storage cost** — Fail-safe keeps 7 extra days of storage for permanent tables. For staging data you recreate every night, this is pure waste.
- Lower total storage bill — sometimes 30–50% less for the staging layer

**What you lose:**
- **No Fail-safe recovery** — if something goes wrong, you can only go back 1 day (Time Travel still works for 1 day)
- Slightly less protection against catastrophic errors

**Rule of thumb:** Raw and Marts tables = Permanent (data is precious). Staging tables = Transient (data can be reloaded from raw).

---

**Q4 — What is a TEMPORARY table? When is it better than a CTE?**

A TEMPORARY table exists only for the duration of the current session. It is invisible to other users and is automatically dropped when the session closes or the user logs out.

**CTE (WITH clause):** Re-executed every time the query runs. If your main query references the same CTE 3 times, it runs 3 times.

**TEMPORARY table:** Computed once and stored in memory/session storage. If you reference it 3 times, it's read 3 times from cache (not recomputed).

**Use TEMPORARY table when:**
- The intermediate result is expensive to compute (large aggregation, complex JOIN)
- You need to reference the same intermediate result multiple times in different queries within the same session
- Debugging a multi-step transformation — inspect intermediate state at each step

```sql
-- CTE re-runs the aggregation 3 times
WITH expensive_cte AS (SELECT ... FROM large_table GROUP BY ...)
SELECT * FROM expensive_cte WHERE x = 1
UNION ALL
SELECT * FROM expensive_cte WHERE x = 2;  -- runs aggregation again

-- TEMP TABLE computes once
CREATE TEMPORARY TABLE t AS SELECT ... FROM large_table GROUP BY ...;
SELECT * FROM t WHERE x = 1;
SELECT * FROM t WHERE x = 2;  -- reads from cached result
```

---

**Q5 — 20 intermediate tables held for 4–6 hours, high storage bill. Fix?**

Change them from **PERMANENT to TRANSIENT**.

Permanent tables pay for Fail-safe (7 days of additional storage). If these tables hold data for only 4–6 hours before being replaced, you are paying for 7 extra days of Fail-safe storage for data you don't need to recover.

Additionally, consider using **TEMPORARY tables** for tables that are only needed within a single ETL session — they are dropped automatically and incur no long-term storage cost at all.

```sql
-- Change existing table type (must recreate)
CREATE OR REPLACE TRANSIENT TABLE staging.intermediate_table AS
SELECT * FROM staging.intermediate_table_old;

-- Or create new ones as transient from the start
CREATE TRANSIENT TABLE staging.step1_result AS SELECT ...;
```

---

**Q6 — What do `_loaded_at` and `_source_file` represent? Why include them?**

These are **metadata audit columns** — they don't come from the source system, they are added by the loading process.

| Column | What it stores | Why it matters |
|---|---|---|
| `_loaded_at` | Timestamp when the row was inserted into Snowflake | Lets you answer "when did we receive this data?" — essential for debugging data freshness issues and incremental load logic |
| `_source_file` | The name of the file or source the row came from | Lets you trace exactly which file a bad row came from — critical for reloading specific files or debugging data quality issues |

In production, you might also add `_source_system`, `_batch_id`, or `_pipeline_run_id` for the same reason — full data lineage traceability.

---

**Q7 — TEMPORARY table and permanent table with same name — which is queried?**

**The TEMPORARY table takes priority.**

Snowflake uses name resolution in this order:
1. TEMPORARY tables in the current session (highest priority)
2. Permanent/transient tables in the current schema

So if a permanent table `ORDERS` exists and you create `CREATE TEMPORARY TABLE ORDERS AS SELECT ...`, your `SELECT * FROM ORDERS` in that session will always hit the temporary version. The permanent table is hidden (shadowed) for that session.

**This is a common gotcha** — a developer may think they're querying production data but are actually querying a temporary scratch table. Always use fully-qualified names (`DB.SCHEMA.TABLE`) when the distinction matters.

---

## Concept 2: Roles and Access Control

**Q8 — Four built-in system roles, highest to lowest?**

```
ACCOUNTADMIN   → Full account control: billing, security, all objects, account parameters
SECURITYADMIN  → Manage users, roles, and grants across the entire account
SYSADMIN       → Create and manage databases, schemas, warehouses, tables
PUBLIC         → Minimum default role every user gets; almost no privileges
```

There is also `USERADMIN` (create users and roles only, subset of SECURITYADMIN) which is sometimes used in organizations that want to split user management from full security admin.

---

**Q9 — Principle of Least Privilege. Concrete example.**

The Principle of Least Privilege means: **grant only the minimum access a person needs to do their job — nothing more.**

**Example:**

A Data Analyst needs to query the MARTS schema to build dashboards. Applying least privilege:

- Grant `USAGE` on the database (needed to see it exists)
- Grant `USAGE` on MARTS schema (needed to access objects inside)
- Grant `SELECT` on tables and views in MARTS (needed to query data)

**Do NOT grant:**
- `SELECT` on RAW_DATA or STAGING (not needed for their job)
- `INSERT`, `UPDATE`, `DELETE` on anything (they only read)
- `CREATE TABLE` (they don't create objects)
- `ACCOUNTADMIN` (never for daily work)

If this analyst's account is compromised, the attacker can only read MARTS data — they cannot modify data, see raw PII, or access billing information.

---

**Q10 — `GRANT USAGE ON SCHEMA` vs `GRANT SELECT ON ALL TABLES IN SCHEMA`. Need both?**

Yes, **both are required** to query a table. They serve different purposes:

| Grant | What it allows |
|---|---|
| `GRANT USAGE ON SCHEMA` | The role can "see" the schema exists and navigate into it. Without this, objects inside are invisible. |
| `GRANT SELECT ON ALL TABLES IN SCHEMA` | The role can read rows from tables. Without this, the schema is visible but tables can't be queried. |

Also need `GRANT USAGE ON DATABASE` — the full chain is:
```
USAGE on DATABASE → USAGE on SCHEMA → SELECT on TABLE
```
Missing any one of these in the chain causes "Object does not exist" errors.

---

**Q11 — What does `GRANT SELECT ON FUTURE TABLES IN SCHEMA` do?**

It is a **forward-looking grant** that automatically gives SELECT privilege on any table created in that schema in the future, without requiring a manual grant each time.

Without `FUTURE TABLES`:
```sql
-- Today you grant on existing tables
GRANT SELECT ON ALL TABLES IN SCHEMA ECOMM_DB.MARTS TO ROLE DATA_ANALYST;

-- Next week, a data engineer creates SALES_MART_V2
-- DATA_ANALYST gets "Object does not exist" on the new table
-- You have to manually grant again: GRANT SELECT ON TABLE SALES_MART_V2 TO ROLE DATA_ANALYST
```

With `FUTURE TABLES`:
```sql
GRANT SELECT ON FUTURE TABLES IN SCHEMA ECOMM_DB.MARTS TO ROLE DATA_ANALYST;
-- New tables created from this point forward are automatically accessible
```

**Always pair both grants:**
```sql
GRANT SELECT ON ALL TABLES IN SCHEMA ...    -- covers existing tables
GRANT SELECT ON FUTURE TABLES IN SCHEMA ... -- covers new tables
```

---

**Q12 — Analyst gets "Object does not exist" after being granted SELECT. Three causes?**

1. **Missing USAGE on the database or schema** — SELECT on a table is not enough; you also need `GRANT USAGE ON DATABASE` and `GRANT USAGE ON SCHEMA`. The chain must be complete.

2. **Session context not set correctly** — The analyst may be in the wrong database/schema. `SELECT * FROM SALES_MART` will fail if the session is pointing at a different database. Fix: `USE DATABASE ECOMM_DB; USE SCHEMA MARTS;` or use fully-qualified names.

3. **Wrong role active** — The analyst may be logged in with their personal role (e.g., PUBLIC) instead of DATA_ANALYST. Run `SELECT CURRENT_ROLE()` to check. Fix: `USE ROLE DATA_ANALYST;` or set the default role on their user profile.

---

**Q13 — SECURITYADMIN vs SYSADMIN — which creates what?**

| Task | Correct Role |
|---|---|
| Create a **database**, schema, table, warehouse | `SYSADMIN` |
| Create a **user** | `SECURITYADMIN` (or `USERADMIN`) |
| Create a **role** | `SECURITYADMIN` |
| Grant a **role to a user** | `SECURITYADMIN` |
| Grant **privileges on objects** | `SYSADMIN` (owns the objects) or `ACCOUNTADMIN` |
| Billing, account limits | `ACCOUNTADMIN` only |

The separation exists so that the person managing user accounts (HR/IT) doesn't need access to create databases or run queries, and the person creating databases doesn't need access to billing.

---

**Q14 — Manager role has SELECT on VIEWS only. Can they query the table directly?**

No. If the MANAGER role only has `GRANT SELECT ON FUTURE VIEWS IN SCHEMA ECOMM_DB.MARTS TO ROLE MANAGER`, then:

```sql
-- As MANAGER_USER:
SELECT * FROM ECOMM_DB.MARTS.SALES_MART_TABLE;
-- Error: "Insufficient privileges to operate on table SALES_MART_TABLE"
```

**How to give them data access without table-level SELECT:**

Create a secure view that selects from the table, omitting PII columns and aggregating sensitive data:

```sql
CREATE OR REPLACE SECURE VIEW ECOMM_DB.MARTS.MANAGER_SALES_VIEW AS
SELECT
    order_month, region, country,
    total_orders, gross_revenue, net_revenue
FROM ECOMM_DB.MARTS.SALES_MART_TABLE;

GRANT SELECT ON VIEW ECOMM_DB.MARTS.MANAGER_SALES_VIEW TO ROLE MANAGER;
```

Now the manager can query the view and see aggregated data, but cannot access the underlying table or any columns not included in the view.

---

**Q15 — What does `MUST_CHANGE_PASSWORD = TRUE` do?**

When set, Snowflake forces the user to change their password on their very first login. The temporary password you set during `CREATE USER` is invalidated after first use.

**Why it matters in production:**
- The admin who creates the user knows the initial password — MUST_CHANGE_PASSWORD ensures the user immediately sets a private password that only they know
- Prevents admins from using new users' credentials to impersonate them
- Satisfies compliance requirements (SOC 2, ISO 27001) that mandate no shared credentials
- If the welcome email is intercepted, the attacker can't use the credential after the user has logged in and changed it

---

## Concept 3: Views — All Types

**Q16 — What is a view? Does it store data?**

A view is a **named, saved SQL query** stored in Snowflake as a schema object. It behaves like a table — you can SELECT from it, grant access on it, and reference it in other queries.

**Standard, secure views: No data stored.** The underlying SQL runs fresh every time you query the view. The view is just the query definition.

**Materialized views: Yes, data is stored.** The query result is pre-computed and cached in Snowflake's storage. It is refreshed automatically in the background when the base table changes.

---

**Q17 — Four view-related types. What makes each different?**

| Type | One distinguishing characteristic |
|---|---|
| **Standard view** | Saved query, no storage, definition visible to anyone with SELECT |
| **Secure view** | Same as standard but definition is hidden from non-owners; required for Data Sharing |
| **Materialized view** | Pre-computes and caches the result; query hits cache not the base table; Enterprise only |
| **Transient table (as mart)** | Fully pre-computed with full JOIN support; must be manually refreshed; cheaper storage than permanent |

---

**Q18 — Standard view vs secure view. When must you use secure?**

**Standard view:** The SQL definition (the `AS SELECT ...` part) is visible to anyone who has SELECT on the view. They can run `SELECT GET_DDL('VIEW', 'my_view')` and see your full query.

**Secure view:** The definition is hidden. Only the view owner (the role that created it) can see the definition. Users with SELECT see results but not the query that produced them.

**Must use SECURE when:**
1. **Data Sharing** — Sharing a view with an external Snowflake account requires SECURE. Non-secure views cannot be shared.
2. **Row-level security** — If your view filter contains logic like `WHERE department = CURRENT_USER()`, a standard view leaks the filter — the user can infer what other departments exist. SECURE hides this.
3. **Proprietary business logic** — When the aggregation formula or join logic is a trade secret you don't want customers or partners to see.

---

**Q19 — How does a materialized view work at query time?**

**Standard view at query time:**
```
SELECT * FROM my_view
  → Snowflake reads the view definition
  → Runs the full underlying SQL against base tables
  → Returns results
  → (same cost as running the underlying query directly)
```

**Materialized view at query time:**
```
SELECT * FROM my_mv
  → Snowflake checks the MV's cached result store
  → Returns the pre-computed result directly
  → (no base table scan, very fast)
```

**Background refresh (when base table changes):**
```
INSERT/UPDATE/DELETE on ORDERS_RAW
  → Snowflake's background service detects the change
  → Incrementally updates only the affected rows in the MV cache
  → Next query sees fresh data without doing the aggregation itself
```

The refresh is incremental (not a full recompute) for most operations — only the changed rows are recalculated.

---

**Q20 — 500M row orders table, same heavy aggregation every 10 minutes, 45 seconds each time. Fix?**

Use a **Materialized View**.

The aggregation runs once when the MV is created (or when base data changes). Every subsequent query hits the pre-computed cache — sub-second response instead of 45 seconds.

```sql
CREATE OR REPLACE MATERIALIZED VIEW MV_ORDERS_SUMMARY AS
SELECT
    region,
    DATE_TRUNC('DAY', order_date) AS order_day,
    COUNT(*)                       AS total_orders,
    SUM(total_amount)              AS gross_revenue
FROM ORDERS_RAW
WHERE status != 'CANCELLED'
GROUP BY region, DATE_TRUNC('DAY', order_date);
```

Dashboard queries hit `MV_ORDERS_SUMMARY` instead of `ORDERS_RAW`. Response goes from 45 seconds to milliseconds.

**Cost trade-off:** You pay storage for the MV's cached result + a small background refresh cost when new orders arrive. But you save massive compute by not re-scanning 500M rows every 10 minutes.

---

**Q21 — Materialized view with JOIN throws an error. Why? Workaround?**

**Why:** Snowflake's materialized views have a restriction — they can only be defined on a **single base table** (or simple subquery). JOINs between multiple tables are not supported in materialized view definitions.

```sql
-- This FAILS:
CREATE MATERIALIZED VIEW MV_ORDERS_WITH_CUSTOMERS AS
SELECT o.order_id, c.full_name, o.total_amount
FROM ORDERS_RAW o
JOIN CUSTOMERS_RAW c ON o.customer_id = c.customer_id;
-- Error: "Materialized views do not support joins"
```

**Workaround — Transient Table as a pre-computed mart:**

```sql
-- Create a transient table (supports full JOINs)
CREATE OR REPLACE TRANSIENT TABLE MARTS.ORDERS_ENRICHED AS
SELECT o.order_id, c.full_name, c.country, o.total_amount, o.order_date
FROM ORDERS_RAW o
JOIN CUSTOMERS_RAW c ON o.customer_id = c.customer_id;
```

Then schedule this `CREATE OR REPLACE` to run on a Snowflake Task or Airflow DAG on your refresh interval. Same result as a materialized view, full JOIN support.

---

**Q22 — `CREATE OR REPLACE VIEW` vs `CREATE VIEW IF NOT EXISTS`?**

| Statement | Behavior | Use when |
|---|---|---|
| `CREATE OR REPLACE VIEW` | If the view already exists, drop it and recreate it. If not, create it. Always succeeds. Loses grants on the old view. | Production pipelines where you deploy updated SQL regularly (dbt models, CI/CD) |
| `CREATE VIEW IF NOT EXISTS` | Only creates the view if it doesn't already exist. If it exists, do nothing. No error. | One-time setup scripts where you don't want to overwrite a view that may have been customized |

**Important:** `CREATE OR REPLACE` removes all existing GRANTs on the view. After replacing a view, you must re-grant privileges. In production, pair it with explicit GRANT statements in your deployment script.

---

**Q23 — Partner can see your proprietary view logic via GET_DDL. How to prevent?**

Convert the view to a **SECURE view**:

```sql
-- Change standard view to secure view
CREATE OR REPLACE SECURE VIEW your_view AS
SELECT ...;  -- same query as before
```

After this:
- The business partner runs `GET_DDL('VIEW', 'your_view')` → returns empty or "insufficient privileges"
- The partner can still SELECT from the view and get results
- Only the role that owns the view (created it) can see the definition

**Note:** For Data Sharing specifically, Snowflake **requires** SECURE views — you cannot share a non-secure view with an external account. Snowflake enforces this at the sharing layer.

---

**Q24 — MV on a table that gets 10,000 new rows per minute. Downside?**

**High maintenance cost.** Every time new rows arrive in the base table, Snowflake's background service must update the materialized view's cached result. At 10,000 rows/minute, the MV refresh service runs almost continuously.

This causes:
1. **Background compute cost** — the refresh service consumes Snowflake credits
2. **Potential refresh lag** — if data arrives faster than the MV can refresh, queries may see slightly stale data
3. **Storage amplification** — you're storing both the raw data and the materialized result

**Better alternative for high-frequency updates:**

Use a **standard view** for real-time accuracy, or a **transient table refreshed on a schedule** (e.g., every 5 minutes via Snowflake Task) where you control the refresh rate and can batch the updates.

```sql
-- Snowflake Task to refresh every 5 minutes
CREATE TASK REFRESH_SALES_MART
  WAREHOUSE = ANALYST_WH
  SCHEDULE = '5 MINUTE'
AS
  CREATE OR REPLACE TRANSIENT TABLE MARTS.SALES_MART AS
  SELECT ... FROM ORDERS_RAW JOIN CUSTOMERS_RAW ...;
```

---

## Mixed / Senior-Level Questions

**Q25 — Healthcare data team: Data Engineer, Clinical Analyst, Compliance Officer, Executive.**

```sql
-- Roles
CREATE ROLE DATA_ENGINEER;        -- loads HL7 data, full ETL access
CREATE ROLE CLINICAL_ANALYST;     -- queries patient summaries, no raw PII
CREATE ROLE COMPLIANCE_OFFICER;   -- reads query history and audit logs, no data modification
CREATE ROLE EXECUTIVE;            -- aggregated KPIs only, no patient data

-- Data Engineer: full access to all schemas
GRANT ALL PRIVILEGES ON ALL TABLES IN DATABASE HEALTH_DB TO ROLE DATA_ENGINEER;

-- Clinical Analyst: read-only on MARTS only via secure views that mask PII
CREATE SECURE VIEW MARTS.PATIENT_SUMMARY AS
SELECT diagnosis_code, age_group, region, COUNT(*) as patient_count
FROM RAW.PATIENT_RECORDS GROUP BY 1,2,3;  -- no names, DOB, SSN
GRANT SELECT ON VIEW MARTS.PATIENT_SUMMARY TO ROLE CLINICAL_ANALYST;

-- Compliance Officer: read Snowflake audit logs, no table data access
GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE COMPLIANCE_OFFICER;
-- (SNOWFLAKE database contains LOGIN_HISTORY, QUERY_HISTORY, ACCESS_HISTORY)

-- Executive: read-only on aggregated KPI views only
CREATE SECURE VIEW MARTS.EXECUTIVE_KPI AS
SELECT month, total_patients, avg_los, readmission_rate FROM MARTS.KPI_TABLE;
GRANT SELECT ON VIEW MARTS.EXECUTIVE_KPI TO ROLE EXECUTIVE;
```

---

**Q26 — Does `GRANT SELECT ON FUTURE TABLES` cover a table created 3 months ago? Tomorrow?**

**3 months ago (before the grant):** No. FUTURE grants only apply to objects created **after** the grant statement was executed. It is not retroactive. Use `GRANT SELECT ON ALL TABLES IN SCHEMA` to cover existing tables.

**Tomorrow (after the grant):** Yes. Any table created from the moment of the FUTURE grant onward is automatically covered.

**Production pattern — always pair both:**
```sql
-- Cover existing tables
GRANT SELECT ON ALL TABLES IN SCHEMA ECOMM_DB.MARTS TO ROLE DATA_ANALYST;
-- Cover future tables
GRANT SELECT ON FUTURE TABLES IN SCHEMA ECOMM_DB.MARTS TO ROLE DATA_ANALYST;
```

---

**Q27 — `DROP TABLE ECOMM_DB.MARTS.SALES_MART` accidentally. Recovery?**

**If PERMANENT table (Time Travel available):**

```sql
-- Step 1: Undrop the table (must do this before Time Travel period expires)
UNDROP TABLE ECOMM_DB.MARTS.SALES_MART;
-- Table is instantly restored as it was at the moment of DROP

-- If the name was reused by a new table with the same name:
-- Use Time Travel to query the old table's data
SELECT * FROM ECOMM_DB.MARTS.SALES_MART
BEFORE (STATEMENT => '<drop_query_id>');
-- Then INSERT INTO a new table from the above query
```

**If TRANSIENT table:**

No `UNDROP` available after the Time Travel window (1 day max) expires. Recovery options:
1. If it's within 1 day, `UNDROP TABLE` still works
2. After 1 day, the data is gone — you must rerun the ETL pipeline that created the table
3. This is why transient tables should only be used for data that can be rebuilt from upstream sources

**Lesson:** Always keep raw data in PERMANENT tables. Only intermediate reproducible data should be transient.

---

**Q28 — `SHOW GRANTS TO ROLE` vs `SHOW GRANTS ON TABLE`. Which for which audit question?**

| Command | Shows | Use to answer |
|---|---|---|
| `SHOW GRANTS TO ROLE role_name` | All privileges the role has been granted (databases, schemas, tables, warehouses) | "What can this role access?" — role-centric view |
| `SHOW GRANTS ON TABLE db.schema.table` | All roles and users that have been granted any privilege on this specific table | "Who can read/modify this table?" — object-centric view |

**Security audit use:**
```sql
-- "Who can access the CUSTOMERS_RAW table?" → object-centric
SHOW GRANTS ON TABLE ECOMM_DB.RAW_DATA.CUSTOMERS_RAW;

-- "What can the DATA_ANALYST role do?" → role-centric
SHOW GRANTS TO ROLE DATA_ANALYST;
```

---

**Q29 — Why does Manager get SELECT on VIEWS but not TABLES? What does it protect against?**

**Security benefit:** By giving the Manager only SELECT on VIEWS (not tables), all data access is mediated through view definitions that you control. The view author decides:
- Which columns are exposed (can exclude email, phone, SSN)
- Which rows are visible (can filter to aggregated data only)
- What calculations are pre-applied (can show net_revenue instead of raw transaction amounts)

**Scenario it protects against:**

If the Manager had `SELECT ON TABLES`:
```sql
-- Manager could do this:
SELECT email, phone, credit_card_last4 FROM ECOMM_DB.MARTS.SALES_MART_TABLE;
-- Sees PII they should not have access to
```

With `SELECT ON VIEWS only`:
```sql
-- Manager can only do this (the view excludes PII columns):
SELECT month, region, total_orders, net_revenue FROM MANAGER_SALES_DASHBOARD;
-- PII columns don't exist in the view — impossible to access them
```

Even if the manager knows PII column names exist in the underlying table, they cannot query them because they have no path to the table itself.

---

**Q30 (System design) — Fintech: PII masking, 2B row reports, real-time fraud, exec P&L.**

**Roles:**
```sql
CREATE ROLE FINTECH_DE;         -- Data Engineers: full ETL access
CREATE ROLE FINANCE_ANALYST;    -- Month-end reports: MARTS read
CREATE ROLE RISK_ANALYST;       -- Real-time fraud: STAGING + MARTS read
CREATE ROLE EXECUTIVE;          -- P&L summaries: secure aggregated views only
CREATE ROLE COMPLIANCE;         -- Audit logs only: SNOWFLAKE db read
```

**Warehouses:**
```sql
CREATE WAREHOUSE ETL_WH         WAREHOUSE_SIZE = 'LARGE'   AUTO_SUSPEND = 300;
CREATE WAREHOUSE FINANCE_WH     WAREHOUSE_SIZE = '2X-LARGE' AUTO_SUSPEND = 120;  -- month-end 2B rows
CREATE WAREHOUSE RISK_WH        WAREHOUSE_SIZE = 'MEDIUM'  AUTO_SUSPEND = 60;   -- real-time, fast
CREATE WAREHOUSE EXEC_WH        WAREHOUSE_SIZE = 'X-SMALL' AUTO_SUSPEND = 60;   -- light aggregated views
```

**PII masking (a — never visible to analysts):**
```sql
-- Raw table has SSN, account_number
-- Analysts only see MARTS secure views that exclude these columns
CREATE SECURE VIEW MARTS.SAFE_TRANSACTIONS AS
SELECT transaction_id, amount, merchant_category, risk_score, transaction_date
FROM RAW.TRANSACTIONS;  -- SSN and account_number deliberately excluded
GRANT SELECT ON VIEW MARTS.SAFE_TRANSACTIONS TO ROLE FINANCE_ANALYST;
```

**Finance 2B row reports (b — month-end):**
```sql
-- Materialized view for month-end aggregations (pre-computed, no 2B row scan at query time)
CREATE MATERIALIZED VIEW MARTS.MV_MONTHLY_PNL AS
SELECT DATE_TRUNC('MONTH', txn_date) AS month, product_line,
       SUM(revenue) AS total_revenue, SUM(cost) AS total_cost
FROM RAW.TRANSACTIONS GROUP BY 1, 2;
-- Finance analyst queries MV — instant, no warehouse thrashing
```

**Real-time fraud scores (c):**
```sql
-- Standard view on STAGING (always fresh, no MV lag)
CREATE VIEW MARTS.FRAUD_REALTIME AS
SELECT transaction_id, customer_id, risk_score, fraud_flag, flagged_at
FROM STAGING.FRAUD_SCORES
WHERE flagged_at >= DATEADD('hour', -1, CURRENT_TIMESTAMP());
GRANT SELECT ON VIEW MARTS.FRAUD_REALTIME TO ROLE RISK_ANALYST;
-- Risk analyst uses RISK_WH (fast resume) for sub-second access
```

**Executive P&L (d):**
```sql
-- Secure view — aggregated only, no customer-level data
CREATE SECURE VIEW MARTS.EXEC_PNL AS
SELECT quarter, business_unit, total_revenue, total_cost,
       total_revenue - total_cost AS net_profit,
       ROUND((total_revenue - total_cost) / total_revenue * 100, 2) AS margin_pct
FROM MARTS.MV_MONTHLY_PNL GROUP BY quarter, business_unit;
GRANT SELECT ON VIEW MARTS.EXEC_PNL TO ROLE EXECUTIVE;
```
