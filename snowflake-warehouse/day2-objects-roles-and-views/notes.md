# Day 2 — Warehouses, Objects, Roles, and Views

## Overview

Day 1 covered Snowflake's architecture and account setup. Today you build everything you need to actually start working — compute clusters (warehouses), a proper data structure (databases, schemas, tables), access control (roles), and the full range of view types.

**Everything in Day 2 follows this order — each step depends on the previous:**

```
Step 1 → Create a Virtual Warehouse     (need compute before anything else)
Step 2 → Create Databases and Schemas   (need a place to put objects)
Step 3 → Create Tables                  (all types — permanent, transient, temporary)
Step 4 → Create Roles and Users         (control who can access what)
Step 5 → Create Views                   (all types — standard, secure, materialized)
```

**Three core concepts today:**

1. **Virtual Warehouses — Deep Dive** — All ways to create a warehouse, all parameters, sizing, auto-suspend, multi-cluster, managing from UI and SQL
2. **Databases, Schemas, and Tables** — Object hierarchy, naming conventions, all table types, sample data setup
3. **Roles, Access Control, and Views** — Role hierarchy, team roles (DE/Analyst/QA/Manager), privilege grants, all view types

---

## Concept 1: Virtual Warehouses — Deep Dive

### What is a Virtual Warehouse?

A **virtual warehouse** is a named cluster of compute resources (CPUs + RAM) that executes SQL queries. It is **completely separate from storage** — your data always lives in cloud storage (Azure Blob, S3, or GCS) regardless of whether any warehouse is running or suspended.

You cannot run a single SQL query in Snowflake without a warehouse. This is the first thing you create.

```
                    ┌─────────────────────────────────┐
                    │      Azure Blob Storage          │
                    │   (your data lives here 24/7)    │
                    └────────────┬────────────────────┘
                                 │ read/write
              ┌──────────────────┼──────────────────┐
              │                  │                  │
     ┌────────▼──────┐  ┌────────▼──────┐  ┌────────▼──────┐
     │   DE_WH       │  │  ANALYST_WH   │  │   QA_WH       │
     │  (compute)    │  │  (compute)    │  │  (compute)    │
     │  Running ETL  │  │  Running      │  │  Running      │
     │  jobs         │  │  dashboards   │  │  validation   │
     └───────────────┘  └───────────────┘  └───────────────┘
         ↑                   ↑                   ↑
   Data Engineers        Analysts             QA Engineers
   pay for DE_WH         pay for             pay for QA_WH
   credits only          ANALYST_WH          credits only
                         credits only
```

### Virtual Warehouse States and Billing

```
Warehouse lifecycle:
                                              ┌──────────────────┐
                                              │    SUSPENDED     │◄──── Billing STOPS
                                              └────────┬─────────┘
                                                       │ query arrives (AUTO_RESUME)
                                                       │ or ALTER WAREHOUSE RESUME
                                                       ▼
                                              ┌──────────────────┐
                                              │    STARTING      │  ~5–10 seconds
                                              └────────┬─────────┘
                                                       │
                                                       ▼
  Billing STARTS ────►                        ┌──────────────────┐
  (minimum 60 sec)                            │     RUNNING      │
                                              └────────┬─────────┘
                                                       │ no queries for N seconds
                                                       │ (AUTO_SUSPEND)
                                                       ▼
                                              ┌──────────────────┐
                                              │    SUSPENDED     │◄──── Billing STOPS
                                              └──────────────────┘

Key rule: You are billed per second, minimum 60 seconds per startup.
          If a query takes 10 seconds, you are billed 60 seconds.
          If a query takes 90 seconds, you are billed 90 seconds.
```

### Warehouse Sizes and Credit Consumption

| Size | Servers | Credits/Hour | Credits/Second | Typical Use |
|---|---|---|---|---|
| **X-Small** | 1 | 1 | 0.000278 | Learning, small dev queries |
| **Small** | 2 | 2 | 0.000556 | Development, light ETL |
| **Medium** | 4 | 4 | 0.001111 | Standard ETL, moderate queries |
| **Large** | 8 | 8 | 0.002222 | Heavy transformation jobs |
| **X-Large** | 16 | 16 | 0.004444 | Large data loads |
| **2X-Large** | 32 | 32 | 0.008889 | Very large models |
| **3X-Large** | 64 | 64 | 0.017778 | Massive parallel processing |
| **4X-Large** | 128 | 128 | 0.035556 | Extreme workloads |

**Credit cost:** ~$2–$4 per credit (varies by cloud region and edition). For this course, **X-Small** is always sufficient.

**Doubling the size halves the runtime** — a query taking 60 seconds on X-Small typically takes ~30 seconds on Small. Total credit cost is roughly the same, but users get results faster.

---

### Approach 1: Create a Warehouse via Snowsight UI

The easiest way for beginners. No SQL required.

**Steps:**
1. Log in to Snowsight → click **Admin** in the left nav
2. Click **Warehouses** → **+ Warehouse** button (top right)
3. Fill in the form:
   - **Name:** `LEARNING_WH`
   - **Size:** `X-Small`
   - **Suspend after inactivity:** `1 minute`
   - **Resume warehouse automatically:** toggle ON
4. Click **Create Warehouse**

The warehouse is created in SUSPENDED state — it will auto-start when you run your first query.

---

### Approach 2: Create a Warehouse via SQL — Minimal Syntax

The bare minimum to create a working warehouse. Snowflake fills in defaults for everything not specified.

```sql
-- Minimal — Snowflake uses defaults for all unspecified parameters
USE ROLE SYSADMIN;

CREATE WAREHOUSE LEARNING_WH;
```

**What defaults Snowflake applies when you don't specify:**

| Parameter | Default Value |
|---|---|
| `WAREHOUSE_SIZE` | `X-SMALL` |
| `AUTO_SUSPEND` | `600` seconds (10 minutes) |
| `AUTO_RESUME` | `TRUE` |
| `INITIALLY_SUSPENDED` | `FALSE` (starts immediately — costs credits!) |
| `MAX_CLUSTER_COUNT` | `1` |
| `MIN_CLUSTER_COUNT` | `1` |

**Warning:** The default `INITIALLY_SUSPENDED = FALSE` means the warehouse starts running immediately after creation and billing begins. Always set it explicitly.

---

### Approach 3: Create a Warehouse via SQL — Full Syntax (Recommended)

Always use explicit parameters in production so the behavior is predictable and documented.

```sql
USE ROLE SYSADMIN;

CREATE WAREHOUSE LEARNING_WH
  WAREHOUSE_SIZE      = 'X-SMALL'       -- compute size
  AUTO_SUSPEND        = 60              -- suspend after 60s of no queries
  AUTO_RESUME         = TRUE            -- auto-start when a query arrives
  INITIALLY_SUSPENDED = TRUE            -- don't start billing immediately
  MAX_CLUSTER_COUNT   = 1               -- single cluster (Standard edition)
  MIN_CLUSTER_COUNT   = 1               -- minimum clusters to maintain
  COMMENT             = 'Learning warehouse — X-Small, auto-suspends in 60s';
```

**What each parameter does:**

| Parameter | What it Controls | Recommendation |
|---|---|---|
| `WAREHOUSE_SIZE` | Node count and compute power | X-Small for learning; size up for production |
| `AUTO_SUSPEND` | Seconds of idle before shutdown | 60 for dev, 120–300 for production |
| `AUTO_RESUME` | Auto-start on incoming query | Always TRUE — no reason to disable |
| `INITIALLY_SUSPENDED` | Whether to start billing immediately at creation | Always TRUE — avoid surprise credits |
| `MAX_CLUSTER_COUNT` | Upper bound for multi-cluster scaling | 1 for Standard; 2–10 for Enterprise dashboards |
| `MIN_CLUSTER_COUNT` | Minimum clusters always running | 1 for most cases; 0 not allowed |
| `COMMENT` | Description visible in SHOW WAREHOUSES | Always fill in — important for ops teams |

---

### Approach 4: Create Multiple Warehouses for Different Teams

In production, each team gets its own warehouse. This prevents one team's heavy queries from slowing down another team's work.

```sql
USE ROLE SYSADMIN;

-- Data Engineers — larger size for ETL jobs, longer suspend (jobs run continuously)
CREATE WAREHOUSE IF NOT EXISTS DE_WH
  WAREHOUSE_SIZE      = 'SMALL'
  AUTO_SUSPEND        = 300
  AUTO_RESUME         = TRUE
  INITIALLY_SUSPENDED = TRUE
  COMMENT             = 'Data Engineering — ETL loads and dbt transformations';

-- Analysts — small size, short suspend (queries are quick, idle time is long)
CREATE WAREHOUSE IF NOT EXISTS ANALYST_WH
  WAREHOUSE_SIZE      = 'X-SMALL'
  AUTO_SUSPEND        = 60
  AUTO_RESUME         = TRUE
  INITIALLY_SUSPENDED = TRUE
  COMMENT             = 'Data Analysts and Managers — reporting and dashboards';

-- QA Engineers — small size, short suspend
CREATE WAREHOUSE IF NOT EXISTS QA_WH
  WAREHOUSE_SIZE      = 'X-SMALL'
  AUTO_SUSPEND        = 60
  AUTO_RESUME         = TRUE
  INITIALLY_SUSPENDED = TRUE
  COMMENT             = 'QA Engineers — data validation queries';
```

---

### Approach 5: Create a Multi-Cluster Warehouse (Enterprise)

A multi-cluster warehouse automatically adds more clusters when many users query simultaneously, then removes them when demand drops. This prevents the queue buildup that happens when 50 analysts all hit a dashboard at the same time.

```sql
-- Multi-cluster warehouse — Enterprise edition only
CREATE WAREHOUSE IF NOT EXISTS DASHBOARD_WH
  WAREHOUSE_SIZE      = 'SMALL'
  AUTO_SUSPEND        = 120
  AUTO_RESUME         = TRUE
  INITIALLY_SUSPENDED = TRUE
  MIN_CLUSTER_COUNT   = 1              -- always at least 1 cluster ready
  MAX_CLUSTER_COUNT   = 3              -- scale up to 3 clusters at peak
  SCALING_POLICY      = 'STANDARD'     -- STANDARD = scale out quickly; ECONOMY = wait longer
  COMMENT             = 'Dashboard warehouse — auto-scales for concurrent users';
```

**SCALING_POLICY options:**

| Policy | Behaviour | Use When |
|---|---|---|
| `STANDARD` | Adds a new cluster as soon as queries start queuing | User-facing dashboards where latency matters |
| `ECONOMY` | Waits until the queue is fully utilized before scaling | Batch jobs where slight delay is acceptable |

---

### Managing Warehouses After Creation

```sql
-- View all warehouses and their current state
SHOW WAREHOUSES;

-- Check one specific warehouse
SHOW WAREHOUSES LIKE 'LEARNING_WH';

-- Manually suspend (stops billing immediately)
ALTER WAREHOUSE LEARNING_WH SUSPEND;

-- Manually resume (starts billing)
ALTER WAREHOUSE LEARNING_WH RESUME;

-- Resize (can be done while running — in-flight queries finish at old size)
ALTER WAREHOUSE LEARNING_WH SET WAREHOUSE_SIZE = 'SMALL';

-- Change auto-suspend interval
ALTER WAREHOUSE LEARNING_WH SET AUTO_SUSPEND = 120;

-- Rename
ALTER WAREHOUSE LEARNING_WH RENAME TO LEARNING_WH_V2;

-- Drop (irreversible)
DROP WAREHOUSE LEARNING_WH_V2;
```

### Setting Your Warehouse for the Session

Before running any SQL query, tell Snowflake which warehouse to use:

```sql
-- Set the warehouse for this session
USE WAREHOUSE LEARNING_WH;

-- Verify
SELECT CURRENT_WAREHOUSE();
```

If you forget this step and run a query, you will get:
```
Error: No active warehouse selected for the session.
```

**For the rest of Day 2, use LEARNING_WH:**

```sql
USE ROLE SYSADMIN;
USE WAREHOUSE LEARNING_WH;
```

---

## Concept 2: Databases, Schemas, and Tables

### Snowflake Object Hierarchy

Everything in Snowflake sits inside a strict hierarchy. You must create objects at each level before creating things inside them:

```
Snowflake Account
└── Organization
    └── Account
        ├── Virtual Warehouses    ← created in Concept 1
        ├── Roles / Users         ← created in Concept 3
        └── Databases
              └── Schemas
                    ├── Tables
                    ├── Views
                    ├── Stages
                    ├── File Formats
                    ├── Stored Procedures
                    └── UDFs (User-Defined Functions)
```

You cannot create a table without a schema. You cannot create a schema without a database. This hierarchy is enforced.

### Databases

A **database** is a top-level logical namespace. It has no compute or storage of its own — it is purely an organisational container.

**All approaches to create a database:**

```sql
USE ROLE SYSADMIN;
USE WAREHOUSE LEARNING_WH;

-- Approach 1: Minimal
CREATE DATABASE ECOMM_DB;

-- Approach 2: With IF NOT EXISTS (safe to run multiple times)
CREATE DATABASE IF NOT EXISTS ECOMM_DB;

-- Approach 3: Full syntax with metadata
CREATE DATABASE IF NOT EXISTS ECOMM_DB
  DATA_RETENTION_TIME_IN_DAYS = 7        -- Time Travel window (default is 1 for Standard, up to 90 for Enterprise)
  COMMENT                     = 'E-commerce data warehouse — course project';

-- View all databases
SHOW DATABASES;

-- Drop a database (drops everything inside it — irreversible without Time Travel)
-- DROP DATABASE ECOMM_DB;

-- Rename
-- ALTER DATABASE ECOMM_DB RENAME TO ECOMM_PROD_DB;
```

**Two system databases already in your account:**

| Database | Purpose | Editable? |
|---|---|---|
| `SNOWFLAKE` | Account-level metadata — query history, login history, usage | Read-only |
| `SNOWFLAKE_SAMPLE_DATA` | TPC-H and TPC-DS benchmark datasets for learning | Read-only |

### Schemas

A **schema** is a namespace inside a database. It holds tables, views, stages, and other objects.

Every new database comes with two built-in schemas automatically:

| Built-in Schema | Purpose |
|---|---|
| `PUBLIC` | Default schema — Snowflake puts objects here if you don't specify a schema |
| `INFORMATION_SCHEMA` | Read-only system views describing all objects in this database |

**Creating schemas — the three-layer industry pattern:**

```sql
USE DATABASE ECOMM_DB;

-- RAW_DATA — data exactly as it arrived from source systems, never modified
CREATE SCHEMA IF NOT EXISTS RAW_DATA
  DATA_RETENTION_TIME_IN_DAYS = 7
  COMMENT = 'Raw ingested data — never modified after load';

-- STAGING — cleaned, typed, deduplicated intermediate data
CREATE SCHEMA IF NOT EXISTS STAGING
  DATA_RETENTION_TIME_IN_DAYS = 1       -- transient data, 1 day is enough
  COMMENT = 'Cleaned and validated intermediate data';

-- MARTS — business-ready, aggregated, joined data for analysts and dashboards
CREATE SCHEMA IF NOT EXISTS MARTS
  DATA_RETENTION_TIME_IN_DAYS = 7
  COMMENT = 'Business-ready aggregated tables and views';
```

**What this structure looks like:**

```
ECOMM_DB
├── RAW_DATA      ← source data lands here (from files, streams, APIs)
│   ├── CUSTOMERS_RAW
│   ├── ORDERS_RAW
│   └── PRODUCTS_RAW
├── STAGING       ← data engineer cleans and validates here
│   ├── STG_CUSTOMERS
│   └── STG_ORDERS
└── MARTS         ← analysts and dashboards read from here only
    ├── SALES_MART
    ├── CUSTOMER_SUMMARY (view)
    └── MANAGER_DASHBOARD (secure view)
```

**Naming conventions (industry standard):**

```
Databases:   UPPERCASE_WITH_UNDERSCORES  → ECOMM_DB, PROD_DW, ANALYTICS_DB
Schemas:     UPPERCASE_WITH_UNDERSCORES  → RAW_DATA, STAGING, MARTS
Tables:      UPPERCASE_WITH_UNDERSCORES  → ORDERS_RAW, STG_CUSTOMERS
Warehouses:  UPPERCASE_WITH_UNDERSCORES  → DE_WH, ANALYST_WH
Roles:       UPPERCASE_WITH_UNDERSCORES  → DATA_ENGINEER, DATA_ANALYST
```

### Setting Session Context

Before creating any tables, set your session context so you don't have to prefix every object name:

```sql
USE ROLE SYSADMIN;
USE WAREHOUSE LEARNING_WH;
USE DATABASE ECOMM_DB;
USE SCHEMA RAW_DATA;

-- Verify everything is set
SELECT
    CURRENT_ROLE()      AS my_role,
    CURRENT_WAREHOUSE() AS my_warehouse,
    CURRENT_DATABASE()  AS my_database,
    CURRENT_SCHEMA()    AS my_schema;
```

After this, `SELECT * FROM CUSTOMERS_RAW` works the same as `SELECT * FROM ECOMM_DB.RAW_DATA.CUSTOMERS_RAW`.

### Table Types in Snowflake

Snowflake has four distinct table types. This is different from most databases — the type you choose directly affects storage cost and data recovery capability.

```
┌─────────────────────────────────────────────────────────────────┐
│  PERMANENT    Default. Full Time Travel (up to 90 days) +       │
│               7-day Fail-safe after Time Travel expires.        │
│               Use for: production facts, dimensions, raw data   │
│                                                                 │
│  TRANSIENT    No Fail-safe. Time Travel up to 1 day only.       │
│               ~50% cheaper storage than permanent.              │
│               Use for: staging tables, intermediate ETL data    │
│                                                                 │
│  TEMPORARY    Session-only. Dropped automatically at logout.    │
│               Invisible to other users and sessions.            │
│               Use for: in-session scratch work, debug steps     │
│                                                                 │
│  EXTERNAL     Points to files in cloud storage (S3/Blob/GCS).   │
│               Data NOT inside Snowflake. Read-only.             │
│               Use for: querying files without loading them      │
│               (covered in Day 3 — Stages and Loading)           │
└─────────────────────────────────────────────────────────────────┘
```

| Feature | Permanent | Transient | Temporary |
|---|---|---|---|
| **Time Travel** | Up to 90 days | Up to 1 day | Up to 1 day |
| **Fail-safe** | 7 days (after Time Travel) | None | None |
| **Storage cost** | Full | ~50% less | Session only — no long-term cost |
| **Visible to others** | Yes | Yes | No — current session only |
| **Survives session end** | Yes | Yes | No — dropped automatically |
| **Best for** | Production data | Staging / ETL intermediates | Scratch / debug work |

### Creating Permanent Tables (RAW_DATA layer)

```sql
USE DATABASE ECOMM_DB;
USE SCHEMA RAW_DATA;

-- Customers raw table
CREATE TABLE IF NOT EXISTS CUSTOMERS_RAW (
    customer_id       NUMBER(10, 0)   NOT NULL,
    first_name        VARCHAR(100),
    last_name         VARCHAR(100),
    email             VARCHAR(255),
    phone             VARCHAR(20),
    country           VARCHAR(100),
    city              VARCHAR(100),
    signup_date       DATE,
    loyalty_tier      VARCHAR(20),          -- BRONZE, SILVER, GOLD, PLATINUM
    is_active         BOOLEAN         DEFAULT TRUE,
    _loaded_at        TIMESTAMP_NTZ   DEFAULT CURRENT_TIMESTAMP(),
    _source_file      VARCHAR(500)
);

-- Orders raw table
CREATE TABLE IF NOT EXISTS ORDERS_RAW (
    order_id          NUMBER(15, 0)   NOT NULL,
    customer_id       NUMBER(10, 0),
    order_date        DATE,
    ship_date         DATE,
    status            VARCHAR(50),          -- PENDING, SHIPPED, DELIVERED, CANCELLED
    total_amount      NUMBER(12, 2),
    discount_amount   NUMBER(12, 2)   DEFAULT 0,
    payment_method    VARCHAR(50),
    region            VARCHAR(100),
    _loaded_at        TIMESTAMP_NTZ   DEFAULT CURRENT_TIMESTAMP(),
    _source_file      VARCHAR(500)
);

-- Products raw table
CREATE TABLE IF NOT EXISTS PRODUCTS_RAW (
    product_id        NUMBER(10, 0)   NOT NULL,
    product_name      VARCHAR(255),
    category          VARCHAR(100),
    sub_category      VARCHAR(100),
    unit_price        NUMBER(10, 2),
    cost_price        NUMBER(10, 2),
    supplier_id       NUMBER(10, 0),
    is_active         BOOLEAN         DEFAULT TRUE,
    _loaded_at        TIMESTAMP_NTZ   DEFAULT CURRENT_TIMESTAMP()
);
```

> `_loaded_at` and `_source_file` are **audit columns** — they don't come from the source, they are added by the pipeline. `_loaded_at` tells you when each row arrived. `_source_file` tells you which file it came from. These are essential for debugging data quality issues.

### Creating Transient Tables (STAGING layer)

Staging tables are recreated every ETL run — they are throwaway intermediates. No point paying for Fail-safe on data you rebuild nightly.

```sql
USE SCHEMA STAGING;

CREATE TRANSIENT TABLE IF NOT EXISTS STG_CUSTOMERS (
    customer_id     NUMBER(10, 0)   NOT NULL,
    full_name       VARCHAR(200),
    email           VARCHAR(255),
    country         VARCHAR(100),
    city            VARCHAR(100),
    signup_date     DATE,
    loyalty_tier    VARCHAR(20),
    is_active       BOOLEAN,
    _loaded_at      TIMESTAMP_NTZ
);

CREATE TRANSIENT TABLE IF NOT EXISTS STG_ORDERS (
    order_id        NUMBER(15, 0)   NOT NULL,
    customer_id     NUMBER(10, 0),
    order_date      DATE,
    ship_date       DATE,
    status          VARCHAR(50),
    gross_amount    NUMBER(12, 2),
    net_amount      NUMBER(12, 2),
    payment_method  VARCHAR(50),
    region          VARCHAR(100),
    _loaded_at      TIMESTAMP_NTZ
);
```

### Creating a Temporary Table (Session scratch work)

```sql
-- Visible only in your current session — gone when you log out
CREATE TEMPORARY TABLE TEMP_HIGH_VALUE_CUSTOMERS AS
SELECT customer_id, full_name, country
FROM ECOMM_DB.STAGING.STG_CUSTOMERS
WHERE loyalty_tier IN ('GOLD', 'PLATINUM');

SELECT COUNT(*) FROM TEMP_HIGH_VALUE_CUSTOMERS;
-- No need to DROP — Snowflake drops it automatically at session end
```

### Insert Sample Data

```sql
USE DATABASE ECOMM_DB;
USE SCHEMA RAW_DATA;

INSERT INTO CUSTOMERS_RAW
    (customer_id, first_name, last_name, email, phone, country, city, signup_date, loyalty_tier, is_active)
VALUES
    (1,  'Alice',  'Johnson', 'alice@example.com',  '+1-555-0101', 'USA',    'New York',    '2023-01-15', 'GOLD',     TRUE),
    (2,  'Bob',    'Smith',   'bob@example.com',    '+1-555-0102', 'USA',    'Chicago',     '2023-02-20', 'SILVER',   TRUE),
    (3,  'Priya',  'Sharma',  'priya@example.com',  '+91-9876543', 'India',  'Mumbai',      '2023-03-05', 'PLATINUM', TRUE),
    (4,  'Carlos', 'Mendez',  'carlos@example.com', '+52-5551234', 'Mexico', 'Mexico City', '2023-01-30', 'BRONZE',   TRUE),
    (5,  'Yuki',   'Tanaka',  'yuki@example.com',   '+81-9012345', 'Japan',  'Tokyo',       '2023-04-01', 'GOLD',     TRUE),
    (6,  'Emma',   'Wilson',  'emma@example.com',   '+44-7912345', 'UK',     'London',      '2023-05-10', 'SILVER',   FALSE),
    (7,  'Ahmed',  'Hassan',  'ahmed@example.com',  '+20-1012345', 'Egypt',  'Cairo',       '2023-06-15', 'BRONZE',   TRUE),
    (8,  'Sophie', 'Dubois',  'sophie@example.com', '+33-6123456', 'France', 'Paris',       '2023-07-20', 'PLATINUM', TRUE),
    (9,  'Raj',    'Patel',   'raj@example.com',    '+91-8765432', 'India',  'Delhi',       '2023-08-25', 'GOLD',     TRUE),
    (10, 'Maria',  'Garcia',  'maria@example.com',  '+34-6123456', 'Spain',  'Madrid',      '2023-09-01', 'SILVER',   TRUE);

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
    (101, 'Wireless Headphones', 'Electronics', 'Audio',      149.99, 60.00, 501, TRUE),
    (102, 'Running Shoes',       'Footwear',    'Sports',      89.99, 35.00, 502, TRUE),
    (103, 'Coffee Maker',        'Appliances',  'Kitchen',    129.99, 55.00, 503, TRUE),
    (104, 'Python Programming',  'Books',       'Technology',  49.99, 12.00, 504, TRUE),
    (105, 'Yoga Mat',            'Sports',      'Fitness',     39.99, 15.00, 505, TRUE);
```

### Verify Everything

```sql
-- See all tables, their type, and row counts
SELECT table_schema, table_name, table_type, row_count
FROM ECOMM_DB.INFORMATION_SCHEMA.TABLES
WHERE table_schema != 'INFORMATION_SCHEMA'
ORDER BY table_schema, table_name;
```

---

## Concept 3: Roles, Access Control, and Views

### Why Roles Matter

In a real data team, different people need different access:

| Team | What They Do | What They Need |
|---|---|---|
| **Data Engineer** | Loads raw data, runs transformations | Full read/write on all schemas, create objects |
| **Data Analyst** | Builds dashboards and reports | Read-only on MARTS only |
| **QA Engineer** | Validates data quality at each layer | Read on STAGING + MARTS |
| **Manager** | Reviews business dashboards | Read-only on MARTS views — no raw tables, no PII |

Giving everyone ACCOUNTADMIN is like giving every employee a master key. Snowflake uses **Role-Based Access Control (RBAC)** to prevent this.

### Snowflake's Built-in Role Hierarchy

```
ACCOUNTADMIN     ← billing, account settings, all security — never use for daily work
     │
SECURITYADMIN    ← create/manage users, roles, and grants
     │
SYSADMIN         ← create/manage databases, schemas, warehouses, tables
     │
PUBLIC           ← absolute minimum — every user gets this automatically
```

**Rule:** Always use the **minimum role** needed for each task.

### Step 1 — Create Custom Roles

```sql
USE ROLE SECURITYADMIN;

CREATE ROLE IF NOT EXISTS DATA_ENGINEER
  COMMENT = 'Full access to all schemas — loads and transforms data';

CREATE ROLE IF NOT EXISTS DATA_ANALYST
  COMMENT = 'Read-only on MARTS schema — business queries and dashboards';

CREATE ROLE IF NOT EXISTS QA_ENGINEER
  COMMENT = 'Read access to STAGING and MARTS for data quality validation';

CREATE ROLE IF NOT EXISTS MANAGER
  COMMENT = 'Read-only on MARTS views only — dashboards, no raw tables, no PII';

-- Wire into SYSADMIN so SYSADMIN can manage these roles
GRANT ROLE DATA_ENGINEER TO ROLE SYSADMIN;
GRANT ROLE DATA_ANALYST  TO ROLE SYSADMIN;
GRANT ROLE QA_ENGINEER   TO ROLE SYSADMIN;
GRANT ROLE MANAGER       TO ROLE SYSADMIN;
```

### Step 2 — Grant Warehouse Access to Each Role

```sql
USE ROLE SYSADMIN;

-- Each role gets USAGE on the warehouse appropriate for their workload
GRANT USAGE ON WAREHOUSE DE_WH      TO ROLE DATA_ENGINEER;
GRANT USAGE ON WAREHOUSE ANALYST_WH TO ROLE DATA_ANALYST;
GRANT USAGE ON WAREHOUSE ANALYST_WH TO ROLE MANAGER;      -- share with manager
GRANT USAGE ON WAREHOUSE QA_WH      TO ROLE QA_ENGINEER;
```

### Step 3 — Grant Database and Schema Privileges

The privilege chain must be complete: `USAGE on DATABASE` → `USAGE on SCHEMA` → `SELECT on TABLE`. Missing any link causes "Object does not exist" errors.

```sql
-- ── DATA ENGINEER: full access to everything ───────────────────────────
GRANT USAGE ON DATABASE ECOMM_DB                          TO ROLE DATA_ENGINEER;
GRANT USAGE ON ALL SCHEMAS IN DATABASE ECOMM_DB           TO ROLE DATA_ENGINEER;
GRANT ALL PRIVILEGES ON ALL TABLES IN DATABASE ECOMM_DB   TO ROLE DATA_ENGINEER;
GRANT ALL PRIVILEGES ON FUTURE TABLES IN SCHEMA ECOMM_DB.RAW_DATA TO ROLE DATA_ENGINEER;
GRANT ALL PRIVILEGES ON FUTURE TABLES IN SCHEMA ECOMM_DB.STAGING  TO ROLE DATA_ENGINEER;
GRANT ALL PRIVILEGES ON FUTURE TABLES IN SCHEMA ECOMM_DB.MARTS    TO ROLE DATA_ENGINEER;
GRANT CREATE TABLE ON SCHEMA ECOMM_DB.RAW_DATA            TO ROLE DATA_ENGINEER;
GRANT CREATE TABLE ON SCHEMA ECOMM_DB.STAGING             TO ROLE DATA_ENGINEER;
GRANT CREATE TABLE ON SCHEMA ECOMM_DB.MARTS               TO ROLE DATA_ENGINEER;
GRANT CREATE VIEW  ON SCHEMA ECOMM_DB.MARTS               TO ROLE DATA_ENGINEER;
GRANT CREATE STAGE ON SCHEMA ECOMM_DB.RAW_DATA            TO ROLE DATA_ENGINEER;

-- ── DATA ANALYST: read-only on MARTS only ──────────────────────────────
GRANT USAGE  ON DATABASE ECOMM_DB                         TO ROLE DATA_ANALYST;
GRANT USAGE  ON SCHEMA   ECOMM_DB.MARTS                   TO ROLE DATA_ANALYST;
GRANT SELECT ON ALL TABLES IN SCHEMA ECOMM_DB.MARTS       TO ROLE DATA_ANALYST;
GRANT SELECT ON ALL VIEWS  IN SCHEMA ECOMM_DB.MARTS       TO ROLE DATA_ANALYST;
GRANT SELECT ON FUTURE TABLES IN SCHEMA ECOMM_DB.MARTS    TO ROLE DATA_ANALYST;
GRANT SELECT ON FUTURE VIEWS  IN SCHEMA ECOMM_DB.MARTS    TO ROLE DATA_ANALYST;

-- ── QA ENGINEER: read on STAGING + MARTS ───────────────────────────────
GRANT USAGE  ON DATABASE ECOMM_DB                              TO ROLE QA_ENGINEER;
GRANT USAGE  ON SCHEMA   ECOMM_DB.STAGING                      TO ROLE QA_ENGINEER;
GRANT USAGE  ON SCHEMA   ECOMM_DB.MARTS                        TO ROLE QA_ENGINEER;
GRANT SELECT ON ALL TABLES IN SCHEMA ECOMM_DB.STAGING          TO ROLE QA_ENGINEER;
GRANT SELECT ON ALL TABLES IN SCHEMA ECOMM_DB.MARTS            TO ROLE QA_ENGINEER;
GRANT SELECT ON FUTURE TABLES IN SCHEMA ECOMM_DB.STAGING       TO ROLE QA_ENGINEER;
GRANT SELECT ON FUTURE TABLES IN SCHEMA ECOMM_DB.MARTS         TO ROLE QA_ENGINEER;

-- ── MANAGER: read-only on MARTS views only (no raw tables, no PII) ─────
GRANT USAGE  ON DATABASE ECOMM_DB                              TO ROLE MANAGER;
GRANT USAGE  ON SCHEMA   ECOMM_DB.MARTS                        TO ROLE MANAGER;
GRANT SELECT ON ALL VIEWS  IN SCHEMA ECOMM_DB.MARTS            TO ROLE MANAGER;
GRANT SELECT ON FUTURE VIEWS IN SCHEMA ECOMM_DB.MARTS          TO ROLE MANAGER;
-- Manager gets VIEWS only — not tables. Views can mask PII columns.
```

### Step 4 — Create Users and Assign Roles

```sql
USE ROLE SECURITYADMIN;

CREATE USER IF NOT EXISTS DE_USER
  PASSWORD            = 'TempPass123!'
  DEFAULT_ROLE        = DATA_ENGINEER
  DEFAULT_WAREHOUSE   = DE_WH
  DEFAULT_NAMESPACE   = ECOMM_DB.RAW_DATA
  MUST_CHANGE_PASSWORD = TRUE
  COMMENT             = 'Data Engineer';

CREATE USER IF NOT EXISTS ANALYST_USER
  PASSWORD            = 'TempPass123!'
  DEFAULT_ROLE        = DATA_ANALYST
  DEFAULT_WAREHOUSE   = ANALYST_WH
  DEFAULT_NAMESPACE   = ECOMM_DB.MARTS
  MUST_CHANGE_PASSWORD = TRUE
  COMMENT             = 'Data Analyst';

CREATE USER IF NOT EXISTS QA_USER
  PASSWORD            = 'TempPass123!'
  DEFAULT_ROLE        = QA_ENGINEER
  DEFAULT_WAREHOUSE   = QA_WH
  DEFAULT_NAMESPACE   = ECOMM_DB.STAGING
  MUST_CHANGE_PASSWORD = TRUE
  COMMENT             = 'QA Engineer';

CREATE USER IF NOT EXISTS MANAGER_USER
  PASSWORD            = 'TempPass123!'
  DEFAULT_ROLE        = MANAGER
  DEFAULT_WAREHOUSE   = ANALYST_WH
  DEFAULT_NAMESPACE   = ECOMM_DB.MARTS
  MUST_CHANGE_PASSWORD = TRUE
  COMMENT             = 'Manager — dashboard access only';

GRANT ROLE DATA_ENGINEER TO USER DE_USER;
GRANT ROLE DATA_ANALYST  TO USER ANALYST_USER;
GRANT ROLE QA_ENGINEER   TO USER QA_USER;
GRANT ROLE MANAGER       TO USER MANAGER_USER;
```

**Access summary:**

| Privilege | DATA_ENGINEER | DATA_ANALYST | QA_ENGINEER | MANAGER |
|---|---|---|---|---|
| RAW_DATA — SELECT | Yes | No | No | No |
| RAW_DATA — INSERT/UPDATE | Yes | No | No | No |
| STAGING — SELECT | Yes | No | Yes | No |
| MARTS — SELECT tables | Yes | Yes | Yes | No |
| MARTS — SELECT views | Yes | Yes | Yes | Yes |
| Create objects | Yes | No | No | No |
| Warehouse | DE_WH | ANALYST_WH | QA_WH | ANALYST_WH |

### Views — All Types

A **view** is a saved SQL query that behaves like a table. It provides abstraction (hide complexity), security (hide columns/rows), and reuse (write the logic once).

#### Type 1: Standard View

A named query. No data stored. Runs the underlying SQL fresh on every SELECT.

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
    SUM(o.total_amount)                   AS lifetime_value,
    MAX(o.order_date)                     AS last_order_date
FROM ECOMM_DB.RAW_DATA.CUSTOMERS_RAW c
LEFT JOIN ECOMM_DB.RAW_DATA.ORDERS_RAW o
       ON c.customer_id = o.customer_id
      AND o.status != 'CANCELLED'
WHERE c.is_active = TRUE
GROUP BY c.customer_id, c.first_name, c.last_name, c.country, c.city, c.loyalty_tier, c.signup_date;

SELECT * FROM CUSTOMER_SUMMARY WHERE country = 'USA';

-- Anyone with SELECT can see the full SQL definition
SELECT GET_DDL('VIEW', 'ECOMM_DB.MARTS.CUSTOMER_SUMMARY');
```

#### Type 2: Secure View

Same as a standard view but the SQL definition is **hidden** from users who have SELECT on it. Only the owner role can see the definition.

Required when:
- Sharing views via Snowflake Data Sharing with external organizations
- The view contains row-level filtering logic you don't want users to reverse-engineer
- The join/aggregation logic is proprietary

```sql
CREATE OR REPLACE SECURE VIEW CUSTOMER_SUMMARY_SECURE AS
SELECT
    c.customer_id,
    c.first_name || ' ' || c.last_name     AS full_name,
    c.country,
    c.city,
    c.loyalty_tier,
    COUNT(o.order_id)                       AS total_orders,
    SUM(o.total_amount - o.discount_amount) AS net_lifetime_value
FROM ECOMM_DB.RAW_DATA.CUSTOMERS_RAW c
LEFT JOIN ECOMM_DB.RAW_DATA.ORDERS_RAW o
       ON c.customer_id = o.customer_id
      AND o.status = 'DELIVERED'
WHERE c.is_active = TRUE
GROUP BY c.customer_id, c.first_name, c.last_name, c.country, c.city, c.loyalty_tier;

-- A non-owner trying to see the definition gets nothing
SELECT GET_DDL('VIEW', 'ECOMM_DB.MARTS.CUSTOMER_SUMMARY_SECURE');
-- Returns: empty or "insufficient privileges"
```

#### Type 3: Materialized View

Pre-computes and **stores the query result** in Snowflake's storage. Snowflake automatically refreshes it in the background when the base table data changes. Queries hit the cache — not the base table.

Best for: expensive aggregations on large tables, queried repeatedly by dashboards.

Limitation: **JOINs are not supported** in materialized view definitions (single table only).

```sql
-- Pre-computed aggregation — cached, auto-refreshed, Enterprise edition required
CREATE OR REPLACE MATERIALIZED VIEW MV_SALES_BY_REGION AS
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

-- This query is instant — hits cache, not the raw table
SELECT * FROM MV_SALES_BY_REGION ORDER BY order_month, gross_revenue DESC;

-- Check refresh status
SHOW MATERIALIZED VIEWS IN SCHEMA ECOMM_DB.MARTS;
```

#### Type 4: Transient Table as Pre-Computed Mart

When you need materialized-view performance but also need JOINs (which MVs don't support), use a transient table refreshed on a schedule via Airflow or Snowflake Task.

```sql
-- Full JOIN support, pre-computed, refreshed on schedule
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
    c.country,
    c.loyalty_tier
FROM ECOMM_DB.RAW_DATA.ORDERS_RAW o
JOIN ECOMM_DB.RAW_DATA.CUSTOMERS_RAW c ON o.customer_id = c.customer_id;

SELECT * FROM ECOMM_DB.MARTS.SALES_MART ORDER BY order_date;
```

#### Manager-Safe View: Combining Roles + Views + PII Masking

This is how roles and views work together in production — the MANAGER role can only access views, and the view excludes PII:

```sql
CREATE OR REPLACE SECURE VIEW ECOMM_DB.MARTS.MANAGER_SALES_DASHBOARD AS
SELECT
    DATE_TRUNC('MONTH', o.order_date)        AS month,
    o.region,
    c.country,
    c.loyalty_tier,
    COUNT(DISTINCT o.customer_id)            AS unique_customers,
    COUNT(o.order_id)                        AS total_orders,
    SUM(o.total_amount)                      AS gross_revenue,
    SUM(o.total_amount - o.discount_amount)  AS net_revenue,
    ROUND(AVG(o.total_amount), 2)            AS avg_order_value
FROM ECOMM_DB.RAW_DATA.ORDERS_RAW o
JOIN ECOMM_DB.RAW_DATA.CUSTOMERS_RAW c ON o.customer_id = c.customer_id
WHERE o.status != 'CANCELLED'
GROUP BY DATE_TRUNC('MONTH', o.order_date), o.region, c.country, c.loyalty_tier
ORDER BY month DESC, gross_revenue DESC;

GRANT SELECT ON VIEW ECOMM_DB.MARTS.MANAGER_SALES_DASHBOARD TO ROLE MANAGER;

-- Manager sees aggregated data only — no email, phone, customer_id
SELECT * FROM ECOMM_DB.MARTS.MANAGER_SALES_DASHBOARD;
```

### View Type Comparison

| Property | Standard View | Secure View | Materialized View | Transient Table |
|---|---|---|---|---|
| **Stores data** | No | No | Yes (cached) | Yes |
| **Query speed** | Reruns SQL each time | Reruns SQL each time | Instant (from cache) | Instant |
| **Definition visible** | Yes | No — hidden | Yes | N/A |
| **JOINs supported** | Yes | Yes | No | Yes |
| **Auto-refreshed** | Always fresh | Always fresh | Yes (background) | No — manual/scheduled |
| **Storage cost** | None | None | Yes | Yes (no fail-safe) |
| **Edition required** | Standard | Standard | Enterprise | Standard |
| **Best for** | Abstraction, reuse | Sharing, security | Heavy repeated aggregations | Pre-computed with JOINs |

---

## Summary

| Step | What You Built | Key Takeaway |
|---|---|---|
| **Warehouse** | LEARNING_WH, DE_WH, ANALYST_WH, QA_WH | Create warehouse before anything else; always set INITIALLY_SUSPENDED = TRUE |
| **Database** | ECOMM_DB | Top-level namespace — no compute, no storage |
| **Schemas** | RAW_DATA, STAGING, MARTS | Three-layer pattern: raw → staging → marts |
| **Tables** | Permanent (raw), Transient (staging), Temporary (scratch) | Choose based on data criticality and storage cost |
| **Roles** | DATA_ENGINEER, DATA_ANALYST, QA_ENGINEER, MANAGER | Least privilege — each role sees only what it needs |
| **Views** | Standard, Secure, Materialized, Transient mart | Standard = reuse, Secure = hide logic, Materialized = cache, Transient = JOIN-capable cache |

**Next:** Day 3 covers loading data into Snowflake — Stages, File Formats, and COPY INTO.
