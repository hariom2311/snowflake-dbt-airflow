# Troubleshooting — "Cortex Code is not enabled or the usage limit has been reached"

## What is this error?

When you open a new Snowflake trial account and try to use **Snowflake Cortex** (the AI/ML features inside Snowsight), you may see:

```
Cortex Code is not enabled or the usage limit has been reached,
please upgrade your account to continue using Cortex.
```

This is **not a problem with your account** and is **not related to this course**. It is a regional availability issue with Snowflake's AI features.

---

## Why does this happen?

Snowflake Cortex (AI features like Cortex Analyst, Cortex Search, Document AI) is **not available in all regions**. When you create a trial account in a region where Cortex is not supported, you will see this error the moment you try to access any Cortex feature.

The error message says "upgrade your account" but **upgrading does not fix it** — the root cause is the region, not your edition.

| Region | Cortex Availability |
|---|---|
| AWS US East (N. Virginia) | Available |
| AWS US West (Oregon) | Available |
| **Azure East US 2** | **Available — use this for the course** |
| Azure West Europe | Limited / Not available |
| GCP US Central | Available |
| Other regions | Often not available |

---

## Can You Change the Region on an Existing Account?

**No — the region cannot be changed after account creation.** It is permanent.

The region is set at signup because your data physically lives in that cloud region's storage. Moving it would require migrating all data to a different data center, which Snowflake does not support as a self-service operation for trial accounts.

| Option | Works? | Notes |
|---|---|---|
| Change region in account settings | No | Not possible — region is locked at creation |
| Contact Snowflake support to migrate | No | Only for paid enterprise accounts, not trials |
| Create a new trial account | Yes | Easiest fix — no data to lose at this stage |
| Stay on current region and ignore Cortex | Yes | Everything else in this course still works |

---

## The Fix — Use Azure East US 2

The simplest fix is to **create a new trial account** with the correct region. Trial accounts are free and you can have one per email address.

### Step-by-step fix:

**Step 1: Go to the trial signup page**
- Navigate to `app.snowflake.com` → "Start for free"

**Step 2: Fill in your details**
- Use the same email or a different one (each email gets one free trial)

**Step 3: Select the correct cloud and region**

```
Cloud Provider:  Azure          ← select this
Region:          East US 2      ← select this specifically
```

> Do NOT select AWS or GCP if you want Cortex to work reliably for this course.

**Step 4: Complete signup and activate**
- Check your email, activate the account
- Log in and open a worksheet

**Step 5: Verify Cortex is working**
```sql
-- Run this query to test Cortex
SELECT SNOWFLAKE.CORTEX.SENTIMENT('Snowflake is an amazing data platform!');
```

If this returns a number between -1 and 1, Cortex is enabled and working.

---

## Why your instructor's account worked fine

The instructor created the account on **Azure East US 2** where Cortex is fully enabled. Students who chose a different region (especially non-US or non-Azure regions) during signup hit this error because Cortex is not rolled out uniformly across all Snowflake regions.

---

## This course does NOT require Cortex

Everything in this course — loading data, writing SQL, dbt models, Airflow pipelines — works on **any Snowflake region**. Cortex is Snowflake's AI layer and is not part of the core curriculum.

If you do not want to create a new account, simply **ignore the Cortex error** and continue with the exercises. You will not be blocked on any Day 1–onwards content because of it.

---

## Quick Reference — Account Creation Settings for This Course

| Setting | Recommended Value |
|---|---|
| Cloud Provider | **Azure** |
| Region | **East US 2** |
| Edition | Enterprise (default for trial) |
| Cortex available | Yes |
| Compatible with course stack | Yes (dbt + Airflow on Azure) |

---

## Still seeing the error after recreating the account?

1. Make sure you selected **Azure** (not AWS or GCP) and **East US 2** specifically
2. Clear your browser cache and log in fresh
3. Run the Cortex test query above — the UI sometimes shows the error even when Cortex is working
4. If the SQL test query also fails, raise it in the course discussion with your account region details
