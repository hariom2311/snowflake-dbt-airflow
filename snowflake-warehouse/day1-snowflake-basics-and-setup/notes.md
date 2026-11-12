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

## Summary

| Concept | Key Takeaway |
|---|---|
| **What is Snowflake** | Cloud-native data warehouse with 3-layer architecture: storage, compute, cloud services |
| **Storage vs Compute** | Fully separated — you scale them independently and pay for each separately |
| **Trial** | 30-day free trial, $400 credits, Enterprise edition — sufficient for all learning |
| **Snowsight** | Web UI — Worksheets for SQL, Data browser, Monitor for query history and billing |
| **Account identifier** | Used to connect external tools (dbt, Python, Airflow) — find via Admin → Accounts |

**Next:** Day 2 covers Virtual Warehouses (all creation approaches), Databases, Schemas, Tables, Roles, and Views — the full object model you need before loading any real data.
