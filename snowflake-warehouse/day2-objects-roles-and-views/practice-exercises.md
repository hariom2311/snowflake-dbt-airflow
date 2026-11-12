# Day 2 — Practice Exercises: Objects, Roles, and Views

> All exercises are done in Snowsight using SQL worksheets.
> Complete them in order — each builds on the previous one.
> Estimated total time: 75–100 minutes.

---

## Exercise 1 — Build the Three-Layer Database Structure

**Concept:** Object hierarchy, naming conventions, permanent vs transient tables

**Tasks:**

**1a. Create the database and three-layer schemas**
```sql
USE ROLE SYSADMIN;
USE WAREHOUSE LEARNING_WH;

CREATE DATABASE IF NOT EXISTS ECOMM_DB
  COMMENT = 'E-commerce data warehouse — Day 2 exercises';

CREATE SCHEMA IF NOT EXISTS ECOMM_DB.RAW_DATA  COMMENT = 'Raw ingested data';
CREATE SCHEMA IF NOT EXISTS ECOMM_DB.STAGING   COMMENT = 'Cleaned intermediate data';
CREATE SCHEMA IF NOT EXISTS ECOMM_DB.MARTS     COMMENT = 'Business-ready data';
```

**1b. Create permanent tables in RAW_DATA**

Copy and run the `CREATE TABLE` statements for `CUSTOMERS_RAW`, `ORDERS_RAW`, and `PRODUCTS_RAW` from the notes.

**1c. Create transient tables in STAGING**

Copy and run the `CREATE TRANSIENT TABLE` statements for `STG_CUSTOMERS` and `STG_ORDERS`.

**1d. Load the sample data**

Copy and run all three `INSERT INTO` blocks from the notes (Customers, Orders, Products).

**1e. Verify what you built**
```sql
-- See all tables and their types
SELECT table_name, table_schema, table_type, row_count
FROM ECOMM_DB.INFORMATION_SCHEMA.TABLES
WHERE table_schema != 'INFORMATION_SCHEMA'
ORDER BY table_schema, table_name;
```

**Expected outcome:** 5 tables total — 3 permanent in RAW_DATA (each with data), 2 transient in STAGING (empty for now).

**Discussion:** The STAGING tables show `TABLE_TYPE = 'TRANSIENT'` and `TABLE_TYPE = 'BASE TABLE'` for permanent ones. Why would you NOT create the `CUSTOMERS_RAW` table as transient, even though it also gets replaced during each ETL run?

---

## Exercise 2 — Explore Table Types Hands-On

**Concept:** Permanent vs transient vs temporary tables, Time Travel

**Tasks:**

**2a. Create a temporary table and verify session behavior**
```sql
USE DATABASE ECOMM_DB;
USE SCHEMA STAGING;

-- Create a temp table
CREATE TEMPORARY TABLE TEMP_HIGH_VALUE AS
SELECT customer_id, first_name, last_name, loyalty_tier
FROM ECOMM_DB.RAW_DATA.CUSTOMERS_RAW
WHERE loyalty_tier IN ('GOLD', 'PLATINUM');

SELECT * FROM TEMP_HIGH_VALUE;
```

**2b. Verify it's invisible to others**
- Open a **second worksheet tab** in Snowsight
- In the new tab, run:
  ```sql
  USE DATABASE ECOMM_DB;
  USE SCHEMA STAGING;
  SELECT * FROM TEMP_HIGH_VALUE;
  -- This should fail: "Object does not exist"
  ```
- Why does this fail even though both worksheets are in the same schema?

**2c. Test Time Travel on a permanent table**
```sql
-- Record the current time
SELECT CURRENT_TIMESTAMP();  -- note this value

-- Delete a row
DELETE FROM ECOMM_DB.RAW_DATA.CUSTOMERS_RAW
WHERE customer_id = 10;

-- Verify the row is gone
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.CUSTOMERS_RAW;  -- should be 9

-- Travel back to before the delete using the timestamp you noted
SELECT COUNT(*) FROM ECOMM_DB.RAW_DATA.CUSTOMERS_RAW
AT (TIMESTAMP => '<paste your timestamp here>'::TIMESTAMP_NTZ);
-- Should return 10

-- Restore the deleted row
INSERT INTO ECOMM_DB.RAW_DATA.CUSTOMERS_RAW
SELECT * FROM ECOMM_DB.RAW_DATA.CUSTOMERS_RAW
AT (TIMESTAMP => '<paste your timestamp here>'::TIMESTAMP_NTZ)
WHERE customer_id = 10;
```

**Expected outcome:** You successfully travel back in time and restore a deleted row.

**Discussion:** What would have happened if `CUSTOMERS_RAW` were a TRANSIENT table and the Time Travel period (1 day) had already expired? What is the production implication?

---

## Exercise 3 — Create Roles and Grant Privileges

**Concept:** RBAC, role hierarchy, least privilege

**Tasks:**

**3a. Create the four custom roles**
```sql
USE ROLE SECURITYADMIN;

CREATE ROLE IF NOT EXISTS DATA_ENGINEER   COMMENT = 'Full ETL access';
CREATE ROLE IF NOT EXISTS DATA_ANALYST    COMMENT = 'Read-only on MARTS';
CREATE ROLE IF NOT EXISTS QA_ENGINEER     COMMENT = 'Read STAGING + MARTS';
CREATE ROLE IF NOT EXISTS MANAGER         COMMENT = 'Read MARTS views only';

-- Wire into SYSADMIN hierarchy
GRANT ROLE DATA_ENGINEER TO ROLE SYSADMIN;
GRANT ROLE DATA_ANALYST  TO ROLE SYSADMIN;
GRANT ROLE QA_ENGINEER   TO ROLE SYSADMIN;
GRANT ROLE MANAGER       TO ROLE SYSADMIN;
```

**3b. Create warehouses and grant to roles**

Copy and run the warehouse creation and grant blocks from the notes (DE_WH, ANALYST_WH, QA_WH).

**3c. Grant database and schema privileges**

Copy and run all four privilege grant blocks from the notes (DATA_ENGINEER, DATA_ANALYST, QA_ENGINEER, MANAGER).

**3d. Create users and assign roles**

Copy and run all four `CREATE USER` blocks and their `GRANT ROLE` statements from the notes.

**3e. Verify the grants**
```sql
-- Check what DATA_ANALYST can access
SHOW GRANTS TO ROLE DATA_ANALYST;

-- Check what MANAGER can access
SHOW GRANTS TO ROLE MANAGER;

-- Check users' role assignments
SHOW GRANTS TO USER ANALYST_USER;
SHOW GRANTS TO USER MANAGER_USER;
```

**Expected outcome:** Each role shows only the grants defined in the notes. MANAGER shows grants only on views, not tables.

**Discussion:** Run `SHOW GRANTS TO ROLE QA_ENGINEER`. Notice QA_ENGINEER does not have USAGE on the RAW_DATA schema. Why was this excluded? What would a QA engineer do if they needed to investigate a data quality issue all the way back to the raw layer?

---

## Exercise 4 — Test Access Control by Role Switching

**Concept:** Role-based access enforcement, least privilege in action

**Tasks:**

**4a. Switch to the DATA_ANALYST role and test access**
```sql
-- Switch role
USE ROLE DATA_ANALYST;
USE WAREHOUSE ANALYST_WH;

-- This should WORK — analyst has SELECT on MARTS
USE DATABASE ECOMM_DB;
USE SCHEMA MARTS;
SHOW TABLES;  -- see which tables/views are accessible

-- Try to access RAW_DATA — should FAIL
SELECT * FROM ECOMM_DB.RAW_DATA.CUSTOMERS_RAW;
-- Error: "Insufficient privileges" or "Object does not exist"
```

**4b. Switch to MANAGER and test view-only access**
```sql
USE ROLE MANAGER;
USE WAREHOUSE ANALYST_WH;

-- Try to query a table directly — should FAIL
SELECT * FROM ECOMM_DB.MARTS.SALES_MART;  -- if this table exists
-- Error: "Insufficient privileges"

-- (Views will work after Exercise 5 creates them)
```

**4c. Switch back to SYSADMIN**
```sql
USE ROLE SYSADMIN;
-- Always return to SYSADMIN after testing to avoid confusion
```

**4d. Check the current role at any point**
```sql
SELECT CURRENT_ROLE(), CURRENT_WAREHOUSE(), CURRENT_DATABASE(), CURRENT_SCHEMA();
```

**Expected outcome:** Role switching works. DATA_ANALYST cannot see RAW_DATA. MANAGER cannot directly query tables.

**Discussion:** You switched roles by running `USE ROLE`. In a real company, the data analyst would log in with their own user account (ANALYST_USER) which defaults to DATA_ANALYST. They would never have the ability to `USE ROLE SYSADMIN` because SYSADMIN has not been granted to them. Try it: `USE ROLE SYSADMIN;` while logged in as ANALYST_USER — what happens?

---

## Exercise 5 — Create All View Types

**Concept:** Standard, secure, materialized views and transient mart tables

**Tasks:**

**5a. Create a standard view**
```sql
USE ROLE SYSADMIN;
USE DATABASE ECOMM_DB;
USE SCHEMA MARTS;

CREATE OR REPLACE VIEW CUSTOMER_SUMMARY AS
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name   AS full_name,
    c.country,
    c.city,
    c.loyalty_tier,
    c.signup_date,
    COUNT(o.order_id)                     AS total_orders,
    SUM(o.total_amount)                   AS lifetime_value
FROM ECOMM_DB.RAW_DATA.CUSTOMERS_RAW c
LEFT JOIN ECOMM_DB.RAW_DATA.ORDERS_RAW o
       ON c.customer_id = o.customer_id
      AND o.status != 'CANCELLED'
WHERE c.is_active = TRUE
GROUP BY c.customer_id, c.first_name, c.last_name, c.country, c.city, c.loyalty_tier, c.signup_date;

-- Query it
SELECT * FROM CUSTOMER_SUMMARY ORDER BY lifetime_value DESC;
```

**5b. Inspect the view definition (possible on standard views)**
```sql
SELECT GET_DDL('VIEW', 'ECOMM_DB.MARTS.CUSTOMER_SUMMARY');
-- You can see the full SQL — note this for comparison with secure view
```

**5c. Create a secure view**
```sql
CREATE OR REPLACE SECURE VIEW MANAGER_SALES_DASHBOARD AS
SELECT
    DATE_TRUNC('MONTH', o.order_date)        AS month,
    o.region,
    c.country,
    c.loyalty_tier,
    COUNT(DISTINCT o.customer_id)            AS unique_customers,
    COUNT(o.order_id)                        AS total_orders,
    SUM(o.total_amount)                      AS gross_revenue,
    SUM(o.total_amount - o.discount_amount)  AS net_revenue
FROM ECOMM_DB.RAW_DATA.ORDERS_RAW o
JOIN ECOMM_DB.RAW_DATA.CUSTOMERS_RAW c ON o.customer_id = c.customer_id
WHERE o.status != 'CANCELLED'
GROUP BY DATE_TRUNC('MONTH', o.order_date), o.region, c.country, c.loyalty_tier
ORDER BY month DESC, gross_revenue DESC;

-- Grant to MANAGER role
GRANT SELECT ON VIEW ECOMM_DB.MARTS.MANAGER_SALES_DASHBOARD TO ROLE MANAGER;

-- Try to inspect the definition — should fail or return empty
SELECT GET_DDL('VIEW', 'ECOMM_DB.MARTS.MANAGER_SALES_DASHBOARD');
```

**5d. Create a materialized view**
```sql
CREATE OR REPLACE MATERIALIZED VIEW MV_SALES_BY_REGION AS
SELECT
    region,
    DATE_TRUNC('MONTH', order_date)       AS order_month,
    COUNT(order_id)                        AS total_orders,
    SUM(total_amount)                      AS gross_revenue,
    SUM(total_amount - discount_amount)    AS net_revenue
FROM ECOMM_DB.RAW_DATA.ORDERS_RAW
WHERE status != 'CANCELLED'
GROUP BY region, DATE_TRUNC('MONTH', order_date);

-- Query it
SELECT * FROM MV_SALES_BY_REGION ORDER BY order_month, gross_revenue DESC;

-- Check its status
SHOW MATERIALIZED VIEWS IN SCHEMA ECOMM_DB.MARTS;
```

**5e. Create a transient table as a pre-computed mart (supports JOINs)**
```sql
CREATE OR REPLACE TRANSIENT TABLE ECOMM_DB.MARTS.SALES_MART AS
SELECT
    o.order_id,
    o.order_date,
    o.region,
    o.status,
    o.total_amount,
    o.discount_amount,
    o.total_amount - o.discount_amount   AS net_amount,
    o.payment_method,
    c.first_name || ' ' || c.last_name  AS customer_name,
    c.country                            AS customer_country,
    c.loyalty_tier
FROM ECOMM_DB.RAW_DATA.ORDERS_RAW o
JOIN ECOMM_DB.RAW_DATA.CUSTOMERS_RAW c ON o.customer_id = c.customer_id;

SELECT * FROM ECOMM_DB.MARTS.SALES_MART ORDER BY order_date;
```

**Expected outcome:** Four different view/table objects in MARTS, each accessible and returning correct data.

**Discussion:** You cannot query `MANAGER_SALES_DASHBOARD`'s definition. Switch to MANAGER role and query the view — confirm you can see aggregated data but cannot access email, phone, or customer_id directly. What specific threat does this protect against?

---

## Exercise 6 — Grant Views to Roles and Test End-to-End

**Concept:** Combining roles + views for real access control

**Tasks:**

**6a. Grant the remaining views to appropriate roles**
```sql
USE ROLE SYSADMIN;

GRANT SELECT ON VIEW  ECOMM_DB.MARTS.CUSTOMER_SUMMARY       TO ROLE DATA_ANALYST;
GRANT SELECT ON VIEW  ECOMM_DB.MARTS.CUSTOMER_SUMMARY       TO ROLE QA_ENGINEER;
GRANT SELECT ON TABLE ECOMM_DB.MARTS.SALES_MART             TO ROLE DATA_ANALYST;
GRANT SELECT ON TABLE ECOMM_DB.MARTS.SALES_MART             TO ROLE QA_ENGINEER;
GRANT SELECT ON VIEW  ECOMM_DB.MARTS.MV_SALES_BY_REGION     TO ROLE DATA_ANALYST;
-- Manager already granted MANAGER_SALES_DASHBOARD in Exercise 5
```

**6b. Test as DATA_ANALYST**
```sql
USE ROLE DATA_ANALYST;
USE WAREHOUSE ANALYST_WH;

-- Should work
SELECT * FROM ECOMM_DB.MARTS.CUSTOMER_SUMMARY;
SELECT * FROM ECOMM_DB.MARTS.SALES_MART;
SELECT * FROM ECOMM_DB.MARTS.MV_SALES_BY_REGION;

-- Should fail
SELECT * FROM ECOMM_DB.RAW_DATA.CUSTOMERS_RAW;  -- no access to RAW
SELECT * FROM ECOMM_DB.STAGING.STG_CUSTOMERS;   -- no access to STAGING
```

**6c. Test as MANAGER**
```sql
USE ROLE MANAGER;
USE WAREHOUSE ANALYST_WH;

-- Should work — view only
SELECT * FROM ECOMM_DB.MARTS.MANAGER_SALES_DASHBOARD;

-- Should fail — tables are off-limits for manager
SELECT * FROM ECOMM_DB.MARTS.SALES_MART;
SELECT * FROM ECOMM_DB.MARTS.CUSTOMER_SUMMARY;
```

**6d. Full inventory of your Day 2 work**
```sql
USE ROLE SYSADMIN;

-- All objects created today
SELECT table_schema, table_name, table_type
FROM ECOMM_DB.INFORMATION_SCHEMA.TABLES
WHERE table_schema != 'INFORMATION_SCHEMA'
ORDER BY table_schema, table_name;

-- All views
SELECT table_schema, table_name, view_definition IS NOT NULL AS has_definition
FROM ECOMM_DB.INFORMATION_SCHEMA.VIEWS
ORDER BY table_schema, table_name;

-- All roles and their grants
SHOW ROLES LIKE 'DATA_%';
SHOW ROLES LIKE 'QA_%';
SHOW ROLES LIKE 'MANAGER';
```

**Expected outcome:** Each role sees exactly the data it should — no more, no less. The access matrix from the notes is fully enforced.

---

## Bonus Challenge — Build a Row-Level Security Pattern

**Concept:** Advanced — using secure views to restrict which ROWS a role sees (not just columns)

In a multi-region company, the APAC analyst should only see APAC orders, not North America or Europe data.

```sql
USE ROLE SYSADMIN;

-- Create a region-scoped secure view
-- CURRENT_ROLE() returns the active role name at query time
CREATE OR REPLACE SECURE VIEW ECOMM_DB.MARTS.REGIONAL_ORDERS AS
SELECT *
FROM ECOMM_DB.MARTS.SALES_MART
WHERE
    -- DATA_ANALYST and MANAGER see all regions
    CURRENT_ROLE() IN ('DATA_ANALYST', 'MANAGER', 'SYSADMIN')
    OR
    -- Future: APAC_ANALYST role would only see Asia region
    (CURRENT_ROLE() = 'APAC_ANALYST' AND customer_country IN ('India', 'Japan', 'Singapore'))
    OR
    -- Future: NA_ANALYST role would only see North America
    (CURRENT_ROLE() = 'NA_ANALYST' AND customer_country IN ('USA', 'Canada', 'Mexico'));

GRANT SELECT ON VIEW ECOMM_DB.MARTS.REGIONAL_ORDERS TO ROLE DATA_ANALYST;

-- Test as DATA_ANALYST — sees all rows
USE ROLE DATA_ANALYST;
SELECT customer_country, COUNT(*) FROM ECOMM_DB.MARTS.REGIONAL_ORDERS GROUP BY 1;
```

**Design question:** Why must this view be SECURE? What would happen if it were a standard view and an APAC_ANALYST ran `GET_DDL('VIEW', 'REGIONAL_ORDERS')`? What information would they learn about other regions from the view definition alone?
