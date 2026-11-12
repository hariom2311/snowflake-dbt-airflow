# Day 2 — Objects, Roles, and Views

## Overview

Day 1 gave you the architecture foundation and a basic account. Today you go deeper into how Snowflake organizes data objects, how access is controlled through roles, and how views work — including types you won't find in most other databases.

**Three core concepts today:**

1. **Databases, Tables, and Object Types** — Permanent vs transient vs temporary tables, table DDL, and the full object model
2. **Roles and Access Control** — Snowflake's role hierarchy, creating roles for real teams (Data Engineer, Analyst, QA, Manager), and granting the right privileges to each
3. **Views — All Types** — Standard views, secure views, materialized views, and transient tables used as view alternatives; when to use each

---

## Concept 1: Databases, Tables, and Object Types

### Picking Up from Day 1

In Day 1 you created a database, schemas, and a simple table. Today you will understand all the table types available and build a proper layered structure that mirrors real data warehouses.

### The Three-Layer Data Architecture (Industry Standard)

Before creating any objects, establish the layer structure. This is the pattern used in production data teams worldwide:

```
ECOMM_DB (database)
├── RAW_DATA (schema)        ← data as it arrives, never modified
│   ├── ORDERS_RAW
│   ├── CUSTOMERS_RAW
│   └── PRODUCTS_RAW
├── STAGING (schema)         ← cleaned, typed, deduplicated
│   ├── STG_ORDERS
│   ├── STG_CUSTOMERS
│   └── STG_PRODUCTS
└── MARTS (schema)           ← business-ready, aggregated, joined
    ├── SALES_MART
    ├── CUSTOMER_MART
    └── PRODUCT_PERFORMANCE
```

**Create this full structure:**

```sql
-- Switch to admin role for object creation
USE ROLE SYSADMIN;
USE WAREHOUSE LEARNING_WH;

-- Create the main database
CREATE DATABASE IF NOT EXISTS ECOMM_DB
  COMMENT = 'E-commerce data warehouse — course project';

-- Create the three layers
CREATE SCHEMA IF NOT EXISTS ECOMM_DB.RAW_DATA
  COMMENT = 'Raw ingested data — never modified after load';

CREATE SCHEMA IF NOT EXISTS ECOMM_DB.STAGING
  COMMENT = 'Cleaned and validated intermediate data';

CREATE SCHEMA IF NOT EXISTS ECOMM_DB.MARTS
  COMMENT = 'Business-ready aggregated tables and views';
```

### Table Types in Snowflake

Snowflake has four distinct table types. This is different from most databases — understanding which to use directly affects storage costs and data recovery options.

```
Table Types:
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  PERMANENT    →  Default. Full Time Travel + Fail-safe.         │
│                  Your production tables. Data never lost.       │
│                                                                 │
│  TRANSIENT    →  No Fail-safe. Reduced storage cost.            │
│                  Good for staging/intermediate data.            │
│                                                                 │
│  TEMPORARY    →  Exists only for the current session.           │
│                  Dropped automatically when session ends.       │
│                  Good for intermediate query results.           │
│                                                                 │
│  EXTERNAL     →  Points to files in cloud storage (S3/Blob).    │
│                  Data is NOT inside Snowflake. Read-only.       │
│                  Covered in Day 3 (Stages and Loading).         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

| Feature | Permanent | Transient | Temporary |
|---|---|---|---|
| **Time Travel** | Up to 90 days (Enterprise) | Up to 1 day | Up to 1 day |
| **Fail-safe** | 7 days (after Time Travel) | None | None |
| **Storage cost** | Full | ~50% less | Session only |
| **Visible to others** | Yes | Yes | No (session only) |
| **Survives session end** | Yes | Yes | No |
| **Use case** | Production facts/dims | Staging, temp ETL | In-query scratch work |

### Creating Permanent Tables (Production Data)

```sql
USE DATABASE ECOMM_DB;
USE SCHEMA RAW_DATA;

-- Customers raw table (permanent — keep full history)
CREATE TABLE IF NOT EXISTS CUSTOMERS_RAW (
    customer_id       NUMBER(10, 0)     NOT NULL,
    first_name        VARCHAR(100),
    last_name         VARCHAR(100),
    email             VARCHAR(255),
    phone             VARCHAR(20),
    country           VARCHAR(100),
    city              VARCHAR(100),
    signup_date       DATE,
    loyalty_tier      VARCHAR(20),        -- BRONZE, SILVER, GOLD, PLATINUM
    is_active         BOOLEAN            DEFAULT TRUE,
    _loaded_at        TIMESTAMP_NTZ      DEFAULT CURRENT_TIMESTAMP(),
    _source_file      VARCHAR(500)
);

-- Orders raw table
CREATE TABLE IF NOT EXISTS ORDERS_RAW (
    order_id          NUMBER(15, 0)     NOT NULL,
    customer_id       NUMBER(10, 0),
    order_date        DATE,
    ship_date         DATE,
    status            VARCHAR(50),       -- PENDING, SHIPPED, DELIVERED, CANCELLED
    total_amount      NUMBER(12, 2),
    discount_amount   NUMBER(12, 2)      DEFAULT 0,
    payment_method    VARCHAR(50),
    region            VARCHAR(100),
    _loaded_at        TIMESTAMP_NTZ      DEFAULT CURRENT_TIMESTAMP(),
    _source_file      VARCHAR(500)
);

-- Products raw table
CREATE TABLE IF NOT EXISTS PRODUCTS_RAW (
    product_id        NUMBER(10, 0)     NOT NULL,
    product_name      VARCHAR(255),
    category          VARCHAR(100),
    sub_category      VARCHAR(100),
    unit_price        NUMBER(10, 2),
    cost_price        NUMBER(10, 2),
    supplier_id       NUMBER(10, 0),
    is_active         BOOLEAN            DEFAULT TRUE,
    _loaded_at        TIMESTAMP_NTZ      DEFAULT CURRENT_TIMESTAMP()
);
```

### Creating Transient Tables (Staging Layer)

Staging tables hold intermediate data — cleaned but not yet aggregated. They are recreated on every ETL run, so paying for Fail-safe storage would be wasteful.

```sql
USE SCHEMA STAGING;

-- Staging customers — transient, no fail-safe needed
CREATE TRANSIENT TABLE IF NOT EXISTS STG_CUSTOMERS (
    customer_id       NUMBER(10, 0)     NOT NULL,
    full_name         VARCHAR(200),
    email             VARCHAR(255),
    country           VARCHAR(100),
    city              VARCHAR(100),
    signup_date       DATE,
    loyalty_tier      VARCHAR(20),
    is_active         BOOLEAN,
    _loaded_at        TIMESTAMP_NTZ
);

-- Staging orders — transient
CREATE TRANSIENT TABLE IF NOT EXISTS STG_ORDERS (
    order_id          NUMBER(15, 0)     NOT NULL,
    customer_id       NUMBER(10, 0),
    order_date        DATE,
    ship_date         DATE,
    status            VARCHAR(50),
    gross_amount      NUMBER(12, 2),
    net_amount        NUMBER(12, 2),
    payment_method    VARCHAR(50),
    region            VARCHAR(100),
    _loaded_at        TIMESTAMP_NTZ
);
```

### Creating Temporary Tables (Session Scratch Work)

```sql
-- Temporary table — only you can see it, gone when session ends
CREATE TEMPORARY TABLE TEMP_HIGH_VALUE_CUSTOMERS AS
SELECT
    customer_id,
    full_name,
    country
FROM ECOMM_DB.STAGING.STG_CUSTOMERS
WHERE loyalty_tier IN ('GOLD', 'PLATINUM');

-- Use it in the same session
SELECT COUNT(*) FROM TEMP_HIGH_VALUE_CUSTOMERS;

-- No need to DROP it — gone automatically when you close the session
```

### Insert Sample Data for the Rest of Day 2

```sql
USE DATABASE ECOMM_DB;
USE SCHEMA RAW_DATA;

INSERT INTO CUSTOMERS_RAW
    (customer_id, first_name, last_name, email, phone, country, city, signup_date, loyalty_tier, is_active)
VALUES
    (1,  'Alice',   'Johnson', 'alice@example.com',   '+1-555-0101', 'USA',    'New York',    '2023-01-15', 'GOLD',     TRUE),
    (2,  'Bob',     'Smith',   'bob@example.com',     '+1-555-0102', 'USA',    'Chicago',     '2023-02-20', 'SILVER',   TRUE),
    (3,  'Priya',   'Sharma',  'priya@example.com',   '+91-9876543', 'India',  'Mumbai',      '2023-03-05', 'PLATINUM', TRUE),
    (4,  'Carlos',  'Mendez',  'carlos@example.com',  '+52-5551234', 'Mexico', 'Mexico City', '2023-01-30', 'BRONZE',   TRUE),
    (5,  'Yuki',    'Tanaka',  'yuki@example.com',    '+81-9012345', 'Japan',  'Tokyo',       '2023-04-01', 'GOLD',     TRUE),
    (6,  'Emma',    'Wilson',  'emma@example.com',    '+44-7912345', 'UK',     'London',      '2023-05-10', 'SILVER',   FALSE),
    (7,  'Ahmed',   'Hassan',  'ahmed@example.com',   '+20-1012345', 'Egypt',  'Cairo',       '2023-06-15', 'BRONZE',   TRUE),
    (8,  'Sophie',  'Dubois',  'sophie@example.com',  '+33-6123456', 'France', 'Paris',       '2023-07-20', 'PLATINUM', TRUE),
    (9,  'Raj',     'Patel',   'raj@example.com',     '+91-8765432', 'India',  'Delhi',       '2023-08-25', 'GOLD',     TRUE),
    (10, 'Maria',   'Garcia',  'maria@example.com',   '+34-6123456', 'Spain',  'Madrid',      '2023-09-01', 'SILVER',   TRUE);

INSERT INTO ORDERS_RAW
    (order_id, customer_id, order_date, ship_date, status, total_amount, discount_amount, payment_method, region)
VALUES
    (1001, 1,  '2024-01-10', '2024-01-13', 'DELIVERED',  250.00,  10.00, 'CREDIT_CARD', 'North America'),
    (1002, 2,  '2024-01-15', '2024-01-18', 'DELIVERED',  120.50,   0.00, 'PAYPAL',      'North America'),
    (1003, 3,  '2024-01-20', '2024-01-25', 'DELIVERED',  980.00,  50.00, 'CREDIT_CARD', 'Asia'),
    (1004, 4,  '2024-02-01', NULL,         'CANCELLED',   75.00,   0.00, 'DEBIT_CARD',  'Latin America'),
    (1005, 5,  '2024-02-10', '2024-02-14', 'DELIVERED',  450.75,  25.00, 'CREDIT_CARD', 'Asia'),
    (1006, 1,  '2024-02-20', '2024-02-23', 'DELIVERED',  310.00,  15.00, 'CREDIT_CARD', 'North America'),
    (1007, 8,  '2024-03-01', '2024-03-05', 'SHIPPED',    720.00,   0.00, 'WIRE',        'Europe'),
    (1008, 9,  '2024-03-10', '2024-03-15', 'DELIVERED',  180.25,   5.00, 'PAYPAL',      'Asia'),
    (1009, 3,  '2024-03-20', '2024-03-24', 'DELIVERED', 1200.00, 100.00, 'CREDIT_CARD', 'Asia'),
    (1010, 10, '2024-04-01', NULL,         'PENDING',    340.00,   0.00, 'CREDIT_CARD', 'Europe');

INSERT INTO PRODUCTS_RAW
    (product_id, product_name, category, sub_category, unit_price, cost_price, supplier_id, is_active)
VALUES
    (101, 'Wireless Headphones',  'Electronics', 'Audio',       149.99,  60.00, 501, TRUE),
    (102, 'Running Shoes',        'Footwear',    'Sports',       89.99,  35.00, 502, TRUE),
    (103, 'Coffee Maker',         'Appliances',  'Kitchen',     129.99,  55.00, 503, TRUE),
    (104, 'Python Programming',   'Books',       'Technology',   49.99,  12.00, 504, TRUE),
    (105, 'Yoga Mat',             'Sports',      'Fitness',      39.99,  15.00, 505, TRUE);
```

---

## Concept 2: Roles and Access Control

### Why Roles Matter

In a real data team, different people need different access:
- A **Data Engineer** loads and transforms data — needs read/write on all schemas
- A **Data Analyst** queries transformed data — needs read-only on MARTS only
- A **QA Engineer** validates data quality — needs read on STAGING and MARTS
- A **Manager** reviews dashboards — needs read-only on MARTS, cannot see raw PII

Giving everyone ACCOUNTADMIN access is the equivalent of giving every employee a master key to the building — a major security risk. Snowflake uses **Role-Based Access Control (RBAC)** to prevent this.

### Snowflake's Built-in Role Hierarchy

Snowflake ships with these system roles — never use ACCOUNTADMIN for daily work:

```
ACCOUNTADMIN  (highest — billing, account settings, security)
     │
     ▼
SECURITYADMIN (manages users, roles, grants)
     │
     ▼
SYSADMIN      (creates databases, schemas, warehouses, tables)
     │
     ▼
PUBLIC        (every user gets this by default — minimal access)
```

**Rule:** Always grant privileges using the **minimum necessary** for the job. This is called the **Principle of Least Privilege**.

### Creating the Team Role Hierarchy

The pattern is: create a role → grant it to a parent role → grant it to a user. Roles can inherit from other roles.

```
ACCOUNTADMIN
     │
SECURITYADMIN
     │
  SYSADMIN
     │
     ├── DATA_ENGINEER    (full access — loads, transforms, creates objects)
     │        │
     │        └── QA_ENGINEER  (inherits from DE read access + staging write)
     │
     └── DATA_ANALYST     (read-only on MARTS)
              │
              └── MANAGER  (read-only on MARTS, no PII columns)
```

**Step 1 — Create all custom roles:**

```sql
-- Switch to SECURITYADMIN for role and user management
USE ROLE SECURITYADMIN;

-- Create custom roles
CREATE ROLE IF NOT EXISTS DATA_ENGINEER
  COMMENT = 'Full access to all schemas — loads and transforms data';

CREATE ROLE IF NOT EXISTS DATA_ANALYST
  COMMENT = 'Read-only access to MARTS schema — business queries';

CREATE ROLE IF NOT EXISTS QA_ENGINEER
  COMMENT = 'Read access to STAGING and MARTS for data quality validation';

CREATE ROLE IF NOT EXISTS MANAGER
  COMMENT = 'Read-only access to MARTS — dashboard and reporting only, no PII';

-- Wire roles into the SYSADMIN hierarchy so SYSADMIN can manage them
GRANT ROLE DATA_ENGINEER TO ROLE SYSADMIN;
GRANT ROLE DATA_ANALYST  TO ROLE SYSADMIN;
GRANT ROLE QA_ENGINEER   TO ROLE SYSADMIN;
GRANT ROLE MANAGER       TO ROLE SYSADMIN;
```

**Step 2 — Grant warehouse usage to each role:**

```sql
USE ROLE SYSADMIN;

-- Create dedicated warehouses per team (from Day 1 concepts)
CREATE WAREHOUSE IF NOT EXISTS DE_WH
  WAREHOUSE_SIZE = 'SMALL'   AUTO_SUSPEND = 120 AUTO_RESUME = TRUE INITIALLY_SUSPENDED = TRUE
  COMMENT = 'Data Engineer warehouse — ETL and transformations';

CREATE WAREHOUSE IF NOT EXISTS ANALYST_WH
  WAREHOUSE_SIZE = 'X-SMALL' AUTO_SUSPEND = 60  AUTO_RESUME = TRUE INITIALLY_SUSPENDED = TRUE
  COMMENT = 'Analyst and Manager warehouse — reporting queries';

CREATE WAREHOUSE IF NOT EXISTS QA_WH
  WAREHOUSE_SIZE = 'X-SMALL' AUTO_SUSPEND = 60  AUTO_RESUME = TRUE INITIALLY_SUSPENDED = TRUE
  COMMENT = 'QA Engineer warehouse — validation queries';

-- Grant USAGE on warehouse to each role
GRANT USAGE ON WAREHOUSE DE_WH      TO ROLE DATA_ENGINEER;
GRANT USAGE ON WAREHOUSE ANALYST_WH TO ROLE DATA_ANALYST;
GRANT USAGE ON WAREHOUSE ANALYST_WH TO ROLE MANAGER;      -- share analyst WH with manager
GRANT USAGE ON WAREHOUSE QA_WH      TO ROLE QA_ENGINEER;
```

**Step 3 — Grant database and schema privileges:**

```sql
-- DATA ENGINEER — full access to everything
GRANT USAGE ON DATABASE ECOMM_DB TO ROLE DATA_ENGINEER;
GRANT USAGE ON ALL SCHEMAS IN DATABASE ECOMM_DB TO ROLE DATA_ENGINEER;
GRANT ALL PRIVILEGES ON ALL TABLES IN DATABASE ECOMM_DB TO ROLE DATA_ENGINEER;
GRANT ALL PRIVILEGES ON FUTURE TABLES IN SCHEMA ECOMM_DB.RAW_DATA  TO ROLE DATA_ENGINEER;
GRANT ALL PRIVILEGES ON FUTURE TABLES IN SCHEMA ECOMM_DB.STAGING   TO ROLE DATA_ENGINEER;
GRANT ALL PRIVILEGES ON FUTURE TABLES IN SCHEMA ECOMM_DB.MARTS     TO ROLE DATA_ENGINEER;
-- Also allow DE to create objects
GRANT CREATE TABLE  ON SCHEMA ECOMM_DB.RAW_DATA TO ROLE DATA_ENGINEER;
GRANT CREATE TABLE  ON SCHEMA ECOMM_DB.STAGING  TO ROLE DATA_ENGINEER;
GRANT CREATE TABLE  ON SCHEMA ECOMM_DB.MARTS    TO ROLE DATA_ENGINEER;
GRANT CREATE VIEW   ON SCHEMA ECOMM_DB.MARTS    TO ROLE DATA_ENGINEER;
GRANT CREATE STAGE  ON SCHEMA ECOMM_DB.RAW_DATA TO ROLE DATA_ENGINEER;

-- DATA ANALYST — read-only on MARTS only
GRANT USAGE  ON DATABASE ECOMM_DB              TO ROLE DATA_ANALYST;
GRANT USAGE  ON SCHEMA   ECOMM_DB.MARTS        TO ROLE DATA_ANALYST;
GRANT SELECT ON ALL TABLES IN SCHEMA ECOMM_DB.MARTS TO ROLE DATA_ANALYST;
GRANT SELECT ON ALL VIEWS  IN SCHEMA ECOMM_DB.MARTS TO ROLE DATA_ANALYST;
GRANT SELECT ON FUTURE TABLES IN SCHEMA ECOMM_DB.MARTS TO ROLE DATA_ANALYST;
GRANT SELECT ON FUTURE VIEWS  IN SCHEMA ECOMM_DB.MARTS TO ROLE DATA_ANALYST;

-- QA ENGINEER — read on STAGING + MARTS (validate data at both layers)
GRANT USAGE  ON DATABASE ECOMM_DB                 TO ROLE QA_ENGINEER;
GRANT USAGE  ON SCHEMA   ECOMM_DB.STAGING         TO ROLE QA_ENGINEER;
GRANT USAGE  ON SCHEMA   ECOMM_DB.MARTS           TO ROLE QA_ENGINEER;
GRANT SELECT ON ALL TABLES IN SCHEMA ECOMM_DB.STAGING TO ROLE QA_ENGINEER;
GRANT SELECT ON ALL TABLES IN SCHEMA ECOMM_DB.MARTS   TO ROLE QA_ENGINEER;
GRANT SELECT ON FUTURE TABLES IN SCHEMA ECOMM_DB.STAGING TO ROLE QA_ENGINEER;
GRANT SELECT ON FUTURE TABLES IN SCHEMA ECOMM_DB.MARTS   TO ROLE QA_ENGINEER;

-- MANAGER — read-only on MARTS only (same as analyst, restricted to safe views)
GRANT USAGE  ON DATABASE ECOMM_DB              TO ROLE MANAGER;
GRANT USAGE  ON SCHEMA   ECOMM_DB.MARTS        TO ROLE MANAGER;
GRANT SELECT ON ALL VIEWS  IN SCHEMA ECOMM_DB.MARTS TO ROLE MANAGER;
GRANT SELECT ON FUTURE VIEWS  IN SCHEMA ECOMM_DB.MARTS TO ROLE MANAGER;
-- Note: Manager gets SELECT on VIEWS only, not raw tables — PII protection
```

**Step 4 — Create users and assign roles:**

```sql
USE ROLE SECURITYADMIN;

-- Data Engineer user
CREATE USER IF NOT EXISTS DE_USER
  PASSWORD           = 'TempPass123!'
  DEFAULT_ROLE       = DATA_ENGINEER
  DEFAULT_WAREHOUSE  = DE_WH
  DEFAULT_NAMESPACE  = ECOMM_DB.RAW_DATA
  MUST_CHANGE_PASSWORD = TRUE
  COMMENT = 'Data Engineer team member';

-- Analyst user
CREATE USER IF NOT EXISTS ANALYST_USER
  PASSWORD           = 'TempPass123!'
  DEFAULT_ROLE       = DATA_ANALYST
  DEFAULT_WAREHOUSE  = ANALYST_WH
  DEFAULT_NAMESPACE  = ECOMM_DB.MARTS
  MUST_CHANGE_PASSWORD = TRUE
  COMMENT = 'Data Analyst team member';

-- QA user
CREATE USER IF NOT EXISTS QA_USER
  PASSWORD           = 'TempPass123!'
  DEFAULT_ROLE       = QA_ENGINEER
  DEFAULT_WAREHOUSE  = QA_WH
  DEFAULT_NAMESPACE  = ECOMM_DB.STAGING
  MUST_CHANGE_PASSWORD = TRUE
  COMMENT = 'QA Engineer team member';

-- Manager user
CREATE USER IF NOT EXISTS MANAGER_USER
  PASSWORD           = 'TempPass123!'
  DEFAULT_ROLE       = MANAGER
  DEFAULT_WAREHOUSE  = ANALYST_WH
  DEFAULT_NAMESPACE  = ECOMM_DB.MARTS
  MUST_CHANGE_PASSWORD = TRUE
  COMMENT = 'Manager — dashboard access only';

-- Assign roles to users
GRANT ROLE DATA_ENGINEER TO USER DE_USER;
GRANT ROLE DATA_ANALYST  TO USER ANALYST_USER;
GRANT ROLE QA_ENGINEER   TO USER QA_USER;
GRANT ROLE MANAGER       TO USER MANAGER_USER;
```

### Access Summary by Role

| Privilege | DATA_ENGINEER | DATA_ANALYST | QA_ENGINEER | MANAGER |
|---|---|---|---|---|
| RAW_DATA — SELECT | Yes | No | No | No |
| RAW_DATA — INSERT/UPDATE | Yes | No | No | No |
| STAGING — SELECT | Yes | No | Yes | No |
| STAGING — INSERT/UPDATE | Yes | No | No | No |
| MARTS — SELECT tables | Yes | Yes | Yes | No |
| MARTS — SELECT views | Yes | Yes | Yes | Yes |
| Create tables/views | Yes | No | No | No |
| Warehouse | DE_WH | ANALYST_WH | QA_WH | ANALYST_WH |

> Manager sees only **views** in MARTS — not raw tables. This means you can build views that mask PII columns (email, phone) while giving managers aggregated reports. This is covered next in Concept 3.

### Verify Role Grants

```sql
-- Check what a role can do
SHOW GRANTS TO ROLE DATA_ANALYST;
SHOW GRANTS TO ROLE MANAGER;

-- Check which roles a user has
SHOW GRANTS TO USER ANALYST_USER;

-- Check who has access to a specific table
SHOW GRANTS ON TABLE ECOMM_DB.MARTS.SALES_MART;
```

---

## Concept 3: Views — All Types

### What is a View?

A view is a **saved SQL query** that behaves like a table. When you query a view, Snowflake runs the underlying SQL and returns the result. Views do not store data themselves (with one exception — materialized views).

```
Without a view:                      With a view:
─────────────                        ────────────
Analyst writes:                      Analyst writes:
  SELECT                               SELECT *
    customer_id,                       FROM ECOMM_DB.MARTS.SAFE_CUSTOMERS
    full_name,                         WHERE country = 'USA';
    country
  FROM ECOMM_DB.RAW_DATA.CUSTOMERS_RAW
  WHERE is_active = TRUE
    AND country = 'USA'
  -- must remember to exclude PII columns
  -- must know which schema to query
  -- must know the filter logic
```

Views provide **abstraction, security, and reuse** in one object.

### Type 1: Standard View

The default view type. A named query — no data stored, runs the SQL fresh every time.

```sql
USE ROLE SYSADMIN;
USE DATABASE ECOMM_DB;
USE SCHEMA MARTS;

-- Standard view — customer summary without PII
CREATE OR REPLACE VIEW CUSTOMER_SUMMARY AS
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name   AS full_name,
    c.country,
    c.city,
    c.loyalty_tier,
    c.signup_date,
    COUNT(o.order_id)                     AS total_orders,
    SUM(o.total_amount)                   AS lifetime_value,
    MAX(o.order_date)                     AS last_order_date
FROM ECOMM_DB.RAW_DATA.CUSTOMERS_RAW c
LEFT JOIN ECOMM_DB.RAW_DATA.ORDERS_RAW o
       ON c.customer_id = o.customer_id
      AND o.status != 'CANCELLED'
WHERE c.is_active = TRUE
GROUP BY
    c.customer_id, c.first_name, c.last_name,
    c.country, c.city, c.loyalty_tier, c.signup_date;

-- Query the view — simple, clean, no PII exposed
SELECT * FROM CUSTOMER_SUMMARY WHERE country = 'USA';
```

**Characteristics of standard views:**

| Property | Value |
|---|---|
| Data stored | No — query runs at SELECT time |
| Performance | Same as underlying query |
| Always fresh | Yes |
| PII masking | Yes (by omitting columns) |
| Visible to others | Yes |
| Can be queried | Yes |
| Can be inserted into | No |

### Type 2: Secure View

A **secure view** hides the view definition from users who have SELECT access. Without the SECURE keyword, anyone with SELECT on the view can run `GET DDL` and see the underlying SQL — including which tables it joins, what filters it applies, and what columns it masks.

```sql
-- Without SECURE — the query definition is visible to analysts:
-- SELECT GET_DDL('VIEW', 'CUSTOMER_SUMMARY');
-- → returns the full SQL including filter logic, source tables, etc.

-- WITH SECURE — definition is hidden from non-owners
CREATE OR REPLACE SECURE VIEW CUSTOMER_SUMMARY_SECURE AS
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name   AS full_name,
    c.country,
    c.city,
    c.loyalty_tier,
    c.signup_date,
    COUNT(o.order_id)                     AS total_orders,
    SUM(o.total_amount - o.discount_amount) AS net_lifetime_value
FROM ECOMM_DB.RAW_DATA.CUSTOMERS_RAW c
LEFT JOIN ECOMM_DB.RAW_DATA.ORDERS_RAW o
       ON c.customer_id = o.customer_id
      AND o.status = 'DELIVERED'
WHERE c.is_active = TRUE
GROUP BY
    c.customer_id, c.first_name, c.last_name,
    c.country, c.city, c.loyalty_tier, c.signup_date;
```

**When to use SECURE views:**
- When the view filters sensitive rows (e.g., `WHERE department = CURRENT_USER()` — a user shouldn't know other departments exist)
- When the join logic or business rules in the SQL are proprietary
- When sharing views via Snowflake Data Sharing with external organizations — SECURE is required for shared views

```sql
-- As ANALYST_USER, trying to see the definition fails:
SELECT GET_DDL('VIEW', 'ECOMM_DB.MARTS.CUSTOMER_SUMMARY_SECURE');
-- Returns: "insufficient privileges" or empty definition
```

### Type 3: Materialized View

A **materialized view** pre-computes and **stores the query result**. Unlike a standard view which runs the SQL every time, a materialized view has its result cached in storage and updated automatically by Snowflake in the background when base table data changes.

```sql
-- Materialized view — pre-computed sales by region and month
-- (expensive aggregation on large ORDERS table, run once, cached)
CREATE OR REPLACE MATERIALIZED VIEW MV_SALES_BY_REGION_MONTH AS
SELECT
    region,
    DATE_TRUNC('MONTH', order_date)       AS order_month,
    COUNT(order_id)                        AS total_orders,
    SUM(total_amount)                      AS gross_revenue,
    SUM(total_amount - discount_amount)    AS net_revenue,
    AVG(total_amount)                      AS avg_order_value
FROM ECOMM_DB.RAW_DATA.ORDERS_RAW
WHERE status != 'CANCELLED'
GROUP BY region, DATE_TRUNC('MONTH', order_date);

-- Querying it is instant — no aggregation needed at query time
SELECT * FROM MV_SALES_BY_REGION_MONTH
WHERE order_month >= '2024-01-01'
ORDER BY order_month, gross_revenue DESC;
```

**How materialized views work:**

```
Base Table (ORDERS_RAW) changes
         │
         ▼
Snowflake background service detects change
         │
         ▼
Incrementally updates the MV result cache
         │
         ▼
Next query hits the cache — not the base table
         → Fast response, low compute at query time
```

**When to use materialized views:**

| Good fit | Bad fit |
|---|---|
| Expensive aggregations queried repeatedly | Simple SELECT with no aggregation |
| Large base tables (millions+ rows) | Small tables where standard view is fast enough |
| Dashboard queries that run many times per hour | One-time or rare queries |
| Read-heavy workloads | Frequently updated base tables (MV refresh has overhead) |

**Important limitations:**
- Requires Enterprise edition (you have this on trial)
- Cannot use JOINs in materialized views (Snowflake limitation — single table or simple subquery only)
- Costs storage for the cached result
- Has a maintenance cost when the base table is updated frequently

```sql
-- Check MV refresh status
SHOW MATERIALIZED VIEWS IN SCHEMA ECOMM_DB.MARTS;
```

### Type 4: Transient Table Used as a View Alternative

For cases where a materialized view's JOIN restriction is a problem, data engineers often create a **transient table** that is refreshed on a schedule (by an Airflow DAG or Snowflake Task). This gives you the performance of a materialized view with the flexibility of full SQL including JOINs.

```sql
-- Transient table as pre-computed result with full JOIN support
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
    c.full_name                           AS customer_name,
    c.country                             AS customer_country,
    c.loyalty_tier,
    p.product_name,
    p.category
FROM ECOMM_DB.STAGING.STG_ORDERS o
JOIN ECOMM_DB.STAGING.STG_CUSTOMERS c ON o.customer_id = c.customer_id
LEFT JOIN ECOMM_DB.RAW_DATA.PRODUCTS_RAW p ON TRUE  -- simplified for demo
ORDER BY o.order_date;

SELECT * FROM ECOMM_DB.MARTS.SALES_MART LIMIT 10;
```

### View Type Comparison — Full Summary

| Property | Standard View | Secure View | Materialized View | Transient Table |
|---|---|---|---|---|
| **Stores data** | No | No | Yes (cached) | Yes |
| **Query speed** | Runs SQL each time | Runs SQL each time | Instant (cached) | Instant |
| **Definition visible** | Yes | No (hidden) | Yes | N/A |
| **JOINs supported** | Yes | Yes | No | Yes |
| **Auto-refresh** | Always fresh | Always fresh | Yes (auto) | No (manual/scheduled) |
| **Storage cost** | None | None | Yes | Yes (no fail-safe) |
| **Edition required** | Standard | Standard | Enterprise | Standard |
| **Best for** | Abstraction, reuse | Security, sharing | Repeated heavy aggregations | Pre-computed with JOINs |

### Practical Manager-Safe View (Combining Roles + Views)

This shows how roles and views work together — the Manager role can only see views, and this view masks PII:

```sql
USE ROLE SYSADMIN;

-- Manager-safe view: no email, no phone, no customer_id — aggregated only
CREATE OR REPLACE SECURE VIEW ECOMM_DB.MARTS.MANAGER_SALES_DASHBOARD AS
SELECT
    DATE_TRUNC('MONTH', o.order_date)      AS month,
    o.region,
    c.country,
    c.loyalty_tier,
    COUNT(DISTINCT o.customer_id)          AS unique_customers,
    COUNT(o.order_id)                      AS total_orders,
    SUM(o.total_amount)                    AS gross_revenue,
    SUM(o.total_amount - o.discount_amount) AS net_revenue,
    ROUND(AVG(o.total_amount), 2)          AS avg_order_value
FROM ECOMM_DB.RAW_DATA.ORDERS_RAW o
JOIN ECOMM_DB.RAW_DATA.CUSTOMERS_RAW c ON o.customer_id = c.customer_id
WHERE o.status != 'CANCELLED'
GROUP BY
    DATE_TRUNC('MONTH', o.order_date),
    o.region, c.country, c.loyalty_tier
ORDER BY month DESC, gross_revenue DESC;

-- Grant to manager role
GRANT SELECT ON VIEW ECOMM_DB.MARTS.MANAGER_SALES_DASHBOARD TO ROLE MANAGER;

-- Manager queries this and sees no PII, no individual customer data
SELECT * FROM ECOMM_DB.MARTS.MANAGER_SALES_DASHBOARD;
```

---

## Summary

| Concept | Key Takeaway |
|---|---|
| **Table types** | Permanent (full recovery), Transient (no fail-safe, cheaper), Temporary (session only) — choose based on data criticality |
| **Three-layer architecture** | RAW_DATA → STAGING → MARTS — always separate layers, never query raw from dashboards |
| **RBAC** | Create roles → grant privileges → grant roles to users. Never use ACCOUNTADMIN for daily work |
| **Role design** | Data Engineer (full), Analyst (MARTS read), QA (STAGING + MARTS read), Manager (MARTS views only) |
| **Standard view** | Saved query, no storage, always fresh — use for abstraction and reuse |
| **Secure view** | Hides definition — use when sharing externally or masking filter logic |
| **Materialized view** | Pre-computed, cached, auto-refreshed — use for heavy repeated aggregations on large tables |
| **Transient table as mart** | Pre-computed with full JOIN support — use when MV's JOIN restriction is a problem |

**Next:** Day 3 covers loading data into Snowflake — Stages, File Formats, COPY INTO, and the VARIANT type for semi-structured (JSON/CSV) data.
