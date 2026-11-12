# Day 1 — Snowflake Basics and Setup

## Overview

Today you will build the foundational understanding of Snowflake's architecture, create your first Snowflake account, and learn how Snowflake organizes compute and storage. These fundamentals underpin everything else in the platform.

**Three core concepts today:**

1. **What is Snowflake?** — Cloud-native data warehouse, its architecture, and why it differs from traditional databases
2. **Snowflake Account Setup** — Signing up, navigating the UI (Snowsight), and understanding editions and cloud providers
3. **Databases, Schemas, and Warehouses** — Snowflake's object hierarchy and how compute is separated from storage

---

## Concept 1: What is Snowflake?

### What is Snowflake?

Snowflake is a **cloud-native data warehouse** built entirely for the cloud — it was not ported from an on-premise system. It runs on AWS, Azure, or GCP and provides SQL-based analytics, data sharing, and data engineering capabilities without infrastructure management.

Think of Snowflake as a managed SQL database that scales infinitely, charges you only for what you use, and separates compute from storage so you can scale them independently.

### Why Snowflake is Different from Traditional Databases

Traditional databases (Oracle, SQL Server, PostgreSQL) couple compute and storage tightly — to store more data, you buy a bigger server; to run faster queries, you upgrade the same machine. This creates a bottleneck.

Snowflake's **multi-cluster shared data architecture** solves this:

```
Traditional RDBMS:
┌─────────────────────────┐
│  Single Server          │
│  ┌─────────┬─────────┐  │
│  │ Compute │ Storage │  │
│  └─────────┴─────────┘  │
└─────────────────────────┘
  → Scale either = scale both = buy bigger machine

Snowflake:
┌──────────────────────────────────────────┐
│              Cloud Storage               │
│         (S3 / Azure Blob / GCS)          │
│    Your data lives here — infinitely     │
└──────────────┬───────────────────────────┘
               │
   ┌───────────┴───────────┐
   │   Query Processing    │  ← Virtual Warehouses (compute)
   │  ┌──────┐  ┌──────┐   │    Scale independently
   │  │  WH1 │  │  WH2 │   │    Pay per second of use
   │  └──────┘  └──────┘   │
   └───────────────────────┘
               │
   ┌───────────┴───────────┐
   │   Cloud Services      │  ← Metadata, optimizer, auth
   │  (optimizer, auth,    │    Always running, no charge
   │   metadata, security) │    separate from compute
   └───────────────────────┘
```

### Three-Layer Architecture (The Core Concept)

| Layer | What it Does | Who Manages It |
|---|---|---|
| **Storage** | Stores all data in Snowflake's compressed, columnar format on cloud blob storage | Snowflake (you never touch S3/Blob directly) |
| **Compute (Virtual Warehouses)** | Executes SQL queries. You can have many warehouses running simultaneously on the same data | You control size, number, and auto-suspend |
| **Cloud Services** | Authentication, query optimization, metadata, transactions, security | Snowflake manages entirely |

### Key Differentiators vs Redshift / BigQuery / Databricks

| Feature | Snowflake | Amazon Redshift | Google BigQuery | Databricks |
|---|---|---|---|---|
| **Architecture** | Shared-disk multi-cluster | Shared-nothing MPP | Serverless columnar | Lakehouse (Delta Lake) |
| **Compute/storage separation** | Yes | Partial (RA3) | Yes | Yes |
| **SQL first** | Yes | Yes | Yes | No (Spark primary) |
| **Multi-cloud** | AWS + Azure + GCP | AWS only | GCP only | AWS + Azure + GCP |
| **Data sharing** | Native, zero-copy | Limited | Limited | Via Delta Sharing |
| **Pricing model** | Credits per second | Per hour | Per TB scanned | DBU per hour |
| **Setup complexity** | Very low | Medium | Low | Medium–High |

### When to Choose Snowflake

Use Snowflake when:
- You need a SQL data warehouse without infrastructure headaches
- Multiple teams need concurrent access to the same data with separate compute
- You need to share data across organizations without copying it
- Your workloads are mixed (ETL + analytics + ad hoc queries)
- You need to run on more than one cloud provider

Avoid Snowflake when:
- You need real-time streaming with sub-second latency (use Kafka + Flink)
- Your team primarily works with Spark/Python (consider Databricks)
- Budget is very tight and your data volume is small (consider DuckDB or Postgres)

---

## Concept 2: Snowflake Account Setup

### Trial Account Creation (Step-by-Step)

Snowflake offers a 30-day free trial with $400 in credits — enough to learn everything in this course.

**Step 1: Go to trial signup**
- Navigate to `app.snowflake.com` → "Start for free"
- Enter your name, email, company name

**Step 2: Choose your cloud and region**

```
Cloud Provider Options:
├── AWS
│   ├── US East (N. Virginia)
│   ├── US West (Oregon)
│   ├── EU (Frankfurt)
│   └── AP (Singapore, Sydney, Tokyo...)
├── Azure
│   ├── East US 2                  ← recommended for this course
│   ├── West Europe
│   └── Southeast Asia
└── GCP
    ├── US Central (Iowa)
    └── Europe West (Netherlands)
```

**Recommendation:** Choose **Azure East US 2** for the trial — this course uses Azure-based tooling (dbt, Airflow on Azure), so keeping Snowflake on the same cloud avoids cross-cloud data transfer and aligns with the rest of the stack.

**Step 3: Choose your Snowflake edition**

| Edition | Key Features | Use Case |
|---|---|---|
| **Standard** | Core SQL, basic security | Learning, small teams |
| **Enterprise** | Multi-cluster warehouses, column-level security, time travel up to 90 days | Most production use cases |
| **Business Critical** | HIPAA, PCI, encryption everywhere | Healthcare, finance |
| **Virtual Private Snowflake** | Dedicated infrastructure | Government, extreme compliance |

For the trial, you get **Enterprise** features — excellent for learning.

**Step 4: Activate your account**
- Check your email for the activation link
- Set a username and password
- Note your **account identifier** — it looks like: `abc12345.us-east-1.aws`

### Navigating Snowsight (The UI)

Snowsight is Snowflake's modern web UI. Key areas:

```
Snowsight Layout:
┌─────────────────────────────────────────────────────┐
│  [≡] Snowflake Logo    Search...    [?] [User icon] │  ← Top bar
├────────┬────────────────────────────────────────────┤
│        │                                            │
│  Nav   │          Main Content Area                 │
│  Panel │                                            │
│        │                                            │
│  ◉ Home│                                            │
│  ◉ Work│                                            │
│    sheets                                           │
│  ◉ Data│                                            │
│  ◉ Mon │                                            │
│    itor│                                            │
│  ◉ Adm │                                            │
│    in  │                                            │
└────────┴────────────────────────────────────────────┘
```

| Nav Section | What You Do Here |
|---|---|
| **Home** | Dashboard, activity feed, recent worksheets |
| **Worksheets** | Write and run SQL queries — your main workspace |
| **Data** | Browse databases, schemas, tables, views |
| **Marketplace** | Access free and paid external datasets |
| **Monitor** | Query history, warehouse usage, billing |
| **Admin** | Users, roles, warehouses, resource monitors |

### Your First Query in Snowsight

After logging in:

1. Click **Worksheets** → **+** to create a new worksheet
2. Type:

```sql
SELECT CURRENT_VERSION();
```

3. Click the **Run** button (or press Ctrl+Enter)
4. You should see Snowflake's version number returned

Checkpoint: If you see a version like `8.x.x`, your account is active and working.

### Understanding Account Identifiers

Your account identifier is used to connect from external tools (Python, dbt, Airflow). It has this format:

```
Format:  <organization_name>-<account_name>
Example: myorg-abc12345

Or legacy format:
<locator>.<region>.<cloud>
Example: abc12345.us-east-1.aws
```

You can find yours at: **Admin → Accounts** or by running:

```sql
SELECT CURRENT_ACCOUNT(), CURRENT_REGION(), CURRENT_ORGANIZATION_NAME();
```

---

## Concept 3: Databases, Schemas, and Virtual Warehouses

### Snowflake Object Hierarchy

Everything in Snowflake is organized in a strict hierarchy. Understanding this is essential before writing any SQL:

```
Snowflake Account
└── Organization (top-level billing/governance)
    └── Account (your trial/production environment)
        ├── Databases
        │   └── Schemas
        │       ├── Tables
        │       ├── Views
        │       ├── Stages (file storage pointers)
        │       ├── File Formats
        │       ├── Stored Procedures
        │       └── UDFs
        ├── Virtual Warehouses (compute clusters)
        ├── Users
        ├── Roles
        └── Resource Monitors
```

### Databases and Schemas

A **database** in Snowflake is a logical container — like a namespace. It holds schemas. It does NOT correspond to a physical database server.

A **schema** is a container inside a database. It holds tables, views, and other objects. Every database comes with two built-in schemas:

| Built-in Schema | Purpose |
|---|---|
| `INFORMATION_SCHEMA` | Read-only system views about all objects in the database |
| `PUBLIC` | Default schema — created for convenience, not required |

**Creating your first database and schema:**

```sql
-- Create a database
CREATE DATABASE SNOWFLAKE_LEARNING;

-- Create a schema inside it
CREATE SCHEMA SNOWFLAKE_LEARNING.RAW_DATA;

-- Another schema for transformed data
CREATE SCHEMA SNOWFLAKE_LEARNING.ANALYTICS;

-- Set context so you don't have to prefix everything
USE DATABASE SNOWFLAKE_LEARNING;
USE SCHEMA RAW_DATA;
```

**Naming conventions (industry standard):**

```
Databases:   UPPERCASE_WITH_UNDERSCORES  (e.g., PROD_DW, ANALYTICS_DB)
Schemas:     UPPERCASE_WITH_UNDERSCORES  (e.g., RAW_DATA, STAGING, MARTS)
Tables:      UPPERCASE_WITH_UNDERSCORES  (e.g., ORDERS, CUSTOMER_EVENTS)
Warehouses:  UPPERCASE_WITH_UNDERSCORES  (e.g., TRANSFORM_WH, REPORTING_WH)
Roles:       UPPERCASE_WITH_UNDERSCORES  (e.g., DATA_ENGINEER, ANALYST)
```

### Virtual Warehouses (The Compute Layer)

A **virtual warehouse** is a cluster of compute resources that executes SQL queries. It is completely separate from storage — your data always lives in cloud storage regardless of whether any warehouse is running.

**Key warehouse concepts:**

```
Virtual Warehouse States:
┌───────────────────────────────────────────────────────┐
│                                                       │
│  SUSPENDED  ──(query arrives)──▶  STARTING            │
│     ↑                                 │               │
│     │                                 ▼               │
│  (auto-suspend                    RUNNING             │
│   after N min)                        │               │
│     ↑                            (query done)         │
│     └─────────────────────────────────┘               │
│                                                       │
└───────────────────────────────────────────────────────┘

Billing: You are charged only when RUNNING
         Charged per second (minimum 60 seconds)
```

**Warehouse sizes and credit consumption:**

| Size | Servers | Credits/Hour | Use Case |
|---|---|---|---|
| X-Small (XS) | 1 | 1 | Learning, small queries |
| Small (S) | 2 | 2 | Development |
| Medium (M) | 4 | 4 | Standard ETL |
| Large (L) | 8 | 8 | Heavy transformation |
| X-Large (XL) | 16 | 16 | Large data loads |
| 2X-Large | 32 | 32 | Very large models |
| 3X-Large | 64 | 64 | Massive parallel processing |
| 4X-Large | 128 | 128 | Extreme workloads |

**Credit cost:** ~$2–$4 per credit depending on your Snowflake edition and cloud region. For learning, XS or S is always sufficient.

**Creating and managing a warehouse:**

```sql
-- Create a warehouse
CREATE WAREHOUSE LEARNING_WH
  WAREHOUSE_SIZE = 'X-SMALL'
  AUTO_SUSPEND = 60          -- suspend after 60 seconds of inactivity
  AUTO_RESUME = TRUE         -- automatically start when a query hits it
  INITIALLY_SUSPENDED = TRUE -- start in suspended state (save credits)
  COMMENT = 'Warehouse for Day 1 learning exercises';

-- Use the warehouse
USE WAREHOUSE LEARNING_WH;

-- Check warehouse status
SHOW WAREHOUSES;

-- Suspend manually
ALTER WAREHOUSE LEARNING_WH SUSPEND;

-- Resume manually
ALTER WAREHOUSE LEARNING_WH RESUME;

-- Resize a warehouse (can be done without stopping it)
ALTER WAREHOUSE LEARNING_WH SET WAREHOUSE_SIZE = 'SMALL';
```

### Understanding Auto-Suspend and Auto-Resume

**Auto-suspend** is one of Snowflake's most important cost-saving features. When a warehouse has no queries running for N seconds, it automatically shuts down — and billing stops.

**Auto-resume** means when a new query arrives, the warehouse automatically starts up (adds ~5–10 seconds to the first query).

**Recommended settings:**
- Development: `AUTO_SUSPEND = 60` (1 minute — saves money during idle time)
- Production ETL: `AUTO_SUSPEND = 300` (5 minutes — avoids thrashing for batch jobs)
- Reporting: `AUTO_SUSPEND = 120` (2 minutes — balance between cost and restart delay)

### Putting It All Together: Session Context

Before running any query, Snowflake needs to know three things: which warehouse to use for compute, which database to look in, and which schema to default to. Set these as your session context:

```sql
-- Set session context
USE ROLE SYSADMIN;
USE WAREHOUSE LEARNING_WH;
USE DATABASE SNOWFLAKE_LEARNING;
USE SCHEMA RAW_DATA;

-- Now you can query without fully-qualifying names
SELECT * FROM MY_TABLE;  -- instead of SNOWFLAKE_LEARNING.RAW_DATA.MY_TABLE

-- Check current context
SELECT
    CURRENT_ROLE()      AS my_role,
    CURRENT_WAREHOUSE() AS my_warehouse,
    CURRENT_DATABASE()  AS my_database,
    CURRENT_SCHEMA()    AS my_schema;
```

### Built-in Sample Data

Snowflake includes a **SNOWFLAKE_SAMPLE_DATA** database with real datasets for learning. No setup needed — it's already in your account:

```sql
-- Explore the sample database
SHOW SCHEMAS IN DATABASE SNOWFLAKE_SAMPLE_DATA;

-- TPC-H is a standard benchmark dataset — orders, customers, etc.
USE DATABASE SNOWFLAKE_SAMPLE_DATA;
USE SCHEMA TPCH_SF1;   -- SF1 = scale factor 1 (~1GB of data)

SHOW TABLES;

-- Count rows in the ORDERS table
SELECT COUNT(*) FROM ORDERS;   -- Should return 1,500,000

-- Preview the CUSTOMER table
SELECT * FROM CUSTOMER LIMIT 10;

-- Simple join — customers and their orders
SELECT
    C.C_NAME        AS customer_name,
    COUNT(O.O_ORDERKEY) AS total_orders
FROM CUSTOMER C
JOIN ORDERS O ON C.C_CUSTKEY = O.O_CUSTKEY
GROUP BY C.C_NAME
ORDER BY total_orders DESC
LIMIT 10;
```

### System Databases You Should Know

| Database | Purpose | Can You Modify? |
|---|---|---|
| `SNOWFLAKE` | Account-level metadata, usage views, login history | Read-only |
| `SNOWFLAKE_SAMPLE_DATA` | TPC-H and TPC-DS benchmark datasets | Read-only |
| Your databases | Your data | Full control |

---

## Summary

| Concept | Key Takeaway |
|---|---|
| **What is Snowflake** | Cloud-native data warehouse with 3-layer architecture: storage, compute, cloud services |
| **Storage vs Compute** | Fully separated — you scale them independently and pay for each separately |
| **Virtual Warehouses** | Compute clusters — you pay per second only when running; auto-suspend saves money |
| **Object Hierarchy** | Account → Database → Schema → Tables/Views/Stages |
| **Session Context** | Always set ROLE, WAREHOUSE, DATABASE, SCHEMA before querying |
| **Trial** | 30-day free trial, $400 credits, Enterprise edition — sufficient for all learning |

**Next:** Day 2 will cover loading data into Snowflake using Stages, File Formats, and the COPY INTO command.
