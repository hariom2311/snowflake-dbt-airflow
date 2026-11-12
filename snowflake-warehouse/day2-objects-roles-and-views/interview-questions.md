# Day 2 — Interview Questions: Objects, Roles, and Views

> 30 questions across all 3 concepts. Attempt your own answer first, then check the solutions file.
> Difficulty tags: (Warm-up) → (Conceptual) → (Scenario) → (Tricky) → (Deep dive) → (System design)

---

## Concept 1: Databases, Tables, and Object Types

**Q1 (Warm-up)** — What are the four table types in Snowflake? Name them and give one sentence describing each.

**Q2 (Warm-up)** — What is the difference between Time Travel and Fail-safe in Snowflake? Which table types support each?

**Q3 (Conceptual)** — Why would you create a TRANSIENT table for a staging layer instead of a permanent table? What do you gain and what do you lose?

**Q4 (Conceptual)** — What is a TEMPORARY table in Snowflake? In what scenario would it be more appropriate than a CTE (Common Table Expression)?

**Q5 (Scenario)** — Your ETL pipeline creates 20 intermediate tables during each nightly run. These tables hold data for 4–6 hours before being replaced. You notice your Snowflake storage bill is high. What table type change would reduce costs, and why?

**Q6 (Conceptual)** — What do the `_loaded_at` and `_source_file` columns in a raw table represent? Why is it considered good practice to include them?

**Q7 (Tricky)** — A developer creates a TEMPORARY table called `ORDERS` in their session. The database already has a permanent table also called `ORDERS` in the same schema. When the developer runs `SELECT * FROM ORDERS`, which table is queried?

---

## Concept 2: Roles and Access Control

**Q8 (Warm-up)** — What are the four built-in system roles in Snowflake? List them from highest to lowest privilege.

**Q9 (Warm-up)** — What is the Principle of Least Privilege? Give a concrete example of applying it in Snowflake.

**Q10 (Conceptual)** — What is the difference between `GRANT USAGE ON SCHEMA` and `GRANT SELECT ON ALL TABLES IN SCHEMA`? Do you need both to query a table?

**Q11 (Conceptual)** — What does `GRANT SELECT ON FUTURE TABLES IN SCHEMA` do, and why is it important to include alongside `GRANT SELECT ON ALL TABLES IN SCHEMA`?

**Q12 (Scenario)** — A new data analyst joins the team. They need to query the MARTS schema but keep getting "Object does not exist" errors even after being granted SELECT on the tables. What are three possible causes?

**Q13 (Conceptual)** — Explain the difference between using SECURITYADMIN vs SYSADMIN for creating objects. Which role should you use to create a new database, and which to create a new user?

**Q14 (Tricky)** — A manager wants to query a table but their role only has SELECT on VIEWS in MARTS. Can they run `SELECT * FROM SALES_MART_TABLE` directly? What would happen, and how would you give them data access without granting table-level SELECT?

**Q15 (Deep dive)** — What does `MUST_CHANGE_PASSWORD = TRUE` do when creating a user? Why is this important from a security perspective in production?

---

## Concept 3: Views — All Types

**Q16 (Warm-up)** — What is a view in Snowflake? Does it store data?

**Q17 (Warm-up)** — What are the four view-related object types covered today? Name them and state in one sentence what makes each different.

**Q18 (Conceptual)** — What is the difference between a standard view and a secure view? When must you use a secure view?

**Q19 (Conceptual)** — Explain how a materialized view works differently from a standard view at query time. What happens in the background when the base table data changes?

**Q20 (Scenario)** — A BI dashboard runs the same complex aggregation query on a 500-million-row orders table every 10 minutes. The query takes 45 seconds each time. How would you fix this with a view type, and which type would you choose?

**Q21 (Tricky)** — You try to create a materialized view that JOINs two tables. Snowflake throws an error. Why? What is the workaround?

**Q22 (Conceptual)** — What is `CREATE OR REPLACE VIEW` vs `CREATE VIEW IF NOT EXISTS`? When would you use each in a production pipeline?

**Q23 (Scenario)** — Your company shares a Snowflake view with a business partner via Data Sharing. The partner's developers run `GET_DDL('VIEW', 'your_view')` and can see your proprietary business logic inside the view. How do you prevent this?

**Q24 (Tricky)** — A materialized view was created on a table that receives 10,000 new rows every minute. What is the downside of using a materialized view in this scenario, and what would you use instead?

---

## Mixed / Senior-Level Questions

**Q25 (Scenario)** — Design the complete access control setup for a healthcare data team with these roles: Data Engineer (loads HL7 data), Clinical Analyst (queries patient summaries), Compliance Officer (audits all access, no data modification), Executive (sees only aggregated KPIs, no patient identifiers). What roles, grants, and view types would you use?

**Q26 (Deep dive)** — Explain how `GRANT SELECT ON FUTURE TABLES` works. If you granted it today and a new table was created 3 months ago before the grant, does that table also get covered? What about a table created tomorrow?

**Q27 (Scenario)** — A data engineer accidentally ran `DROP TABLE ECOMM_DB.MARTS.SALES_MART`. The table was permanent. What is the recovery process? What if it had been transient?

**Q28 (Conceptual)** — What is the difference between `SHOW GRANTS TO ROLE` and `SHOW GRANTS ON TABLE`? In a security audit, which would you use to answer "who can read this table?" and which to answer "what can this role access?"

**Q29 (Deep dive)** — Why does the Manager role in our setup get SELECT on VIEWS but not on TABLES in MARTS? What specific security benefit does this give? Give an example of a scenario it protects against.

**Q30 (System design)** — Design the complete role, warehouse, and view architecture for a fintech company with these requirements: (a) PII data (SSN, account numbers) in RAW layer must never be visible to analysts, (b) Finance team runs month-end reports on 2 billion rows, (c) Risk team needs real-time access to fraud scores, (d) Executives see only aggregated P&L. Name all roles, warehouses, view types, and grant strategy.
