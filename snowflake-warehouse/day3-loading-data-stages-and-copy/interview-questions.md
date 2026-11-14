# Day 3 — Interview Questions: Loading Data, Stages, and COPY INTO

> 30 questions across all 3 concepts. Attempt your own answer first, then check the solutions file.
> Difficulty tags: (Warm-up) → (Conceptual) → (Scenario) → (Tricky) → (Deep dive) → (System design)

---

## Concept 1: Stages and File Formats

**Q1 (Warm-up)** — What is a Snowflake Stage? What problem does it solve in the data loading process?

**Q2 (Warm-up)** — Name all three types of internal stages in Snowflake. How does each one differ in terms of creation and visibility?

**Q3 (Conceptual)** — What is the difference between an internal stage and an external stage? Give a real-world scenario where you would choose each.

**Q4 (Conceptual)** — What command do you use to upload a file from your local machine into an internal stage? Can you run this command from a Snowsight worksheet?

**Q5 (Conceptual)** — What is a File Format object in Snowflake? What is the advantage of creating a named file format vs defining the format inline in every COPY INTO statement?

**Q6 (Scenario)** — Your source CSV files have a pipe `|` delimiter instead of a comma, values are not quoted, there is no header row, and empty strings should be treated as NULL. Write the CREATE FILE FORMAT statement for this.

**Q7 (Tricky)** — You create an external stage pointing to an Azure Blob container. After creating the stage, you add 50 new files to the container. Do you need to update the stage object? What command do you use to verify the files are visible to Snowflake?

---

## Concept 2: COPY INTO — All Approaches

**Q8 (Warm-up)** — What does `COPY INTO` do? What are the two things you must always specify in a COPY INTO statement?

**Q9 (Warm-up)** — Snowflake says COPY INTO is "idempotent." What does this mean in practice? How long does Snowflake remember which files have already been loaded?

**Q10 (Conceptual)** — Explain the difference between `ON_ERROR = 'ABORT_STATEMENT'` and `ON_ERROR = 'CONTINUE'`. Which would you use for a production financial data load and why?

**Q11 (Conceptual)** — What does `VALIDATION_MODE = 'RETURN_ERRORS'` do? Does it load any data into the table?

**Q12 (Scenario)** — You have 200 files in a stage named `orders_2024_01_15.csv.gz`, `orders_2024_01_16.csv.gz`, etc. You only want to load files from January 2024. How do you load only those files using COPY INTO?

**Q13 (Conceptual)** — What are `$1`, `$2`, `$3` in the context of COPY INTO with a SELECT subquery? When would you use column references instead of a direct COPY?

**Q14 (Tricky)** — You run COPY INTO and it completes successfully, loading 10,000 rows. You immediately run the exact same COPY INTO command again. How many rows are loaded the second time, and why? How do you force it to reload?

**Q15 (Deep dive)** — What is `METADATA$FILENAME` and why should you capture it in every load into a raw table? What other metadata columns are available during COPY INTO?

---

## Concept 3: Semi-Structured Data and VARIANT

**Q16 (Warm-up)** — What is the VARIANT data type in Snowflake? What kinds of data can it store?

**Q17 (Warm-up)** — How do you extract a value from a VARIANT column in Snowflake? What is the `::` operator used for?

**Q18 (Conceptual)** — What does `STRIP_OUTER_ARRAY = TRUE` do in a JSON file format? When would you need it?

**Q19 (Conceptual)** — Explain `LATERAL FLATTEN`. What problem does it solve? Give an example of data where you would need it.

**Q20 (Scenario)** — You receive a JSON file where each record looks like this:
```json
{"user_id": 101, "events": [{"type": "click", "ts": "2024-01-01T10:00:00"}, {"type": "purchase", "ts": "2024-01-01T10:05:00"}]}
```
Write a query that returns one row per event, with columns: `user_id`, `event_type`, `event_timestamp`.

**Q21 (Conceptual)** — What does the `TYPEOF` function return when used on a VARIANT value? Give three possible return values.

**Q22 (Tricky)** — A developer queries `raw_event:customer:email::VARCHAR` from a VARIANT column. For some rows the value returned is `NULL`. What are three possible reasons the value could be NULL?

**Q23 (Deep dive)** — What is `OBJECT_KEYS`? What is `ARRAY_SIZE`? Give a use case for each in a real data pipeline.

**Q24 (Scenario)** — Your team receives Parquet files from a data lake. You need to load them into Snowflake. What file format do you create, and what COPY INTO statement would you write?

---

## Mixed / Senior-Level Questions

**Q25 (Scenario)** — Your nightly ETL loads 500 CSV files from Azure Blob using COPY INTO. Some nights, 2–3 files have data quality issues and cause the load to fail. You want to: (a) load all good files, (b) capture the bad files for reprocessing, (c) alert the team. Design the full approach including ON_ERROR setting, how to identify failed files, and how you would retry them.

**Q26 (Deep dive)** — Explain the difference between using a **User Stage** (`@~`), a **Table Stage** (`@%TABLE_NAME`), and a **Named Internal Stage** for a production ETL pipeline. Which would you choose and why?

**Q27 (Scenario)** — A new data source sends JSON where sometimes `order_total` is a number (`250.00`), sometimes a string (`"250.00"`), and sometimes missing entirely. How would Snowflake store this in VARIANT? How would you safely extract it as a NUMBER in SQL?

**Q28 (Tricky)** — You load a CSV file with `ON_ERROR = 'CONTINUE'`. After the load, you want to see which rows failed. What query do you run?

**Q29 (Deep dive)** — What is the difference between loading data with `COPY INTO` vs using `INSERT INTO ... SELECT` from a stage query? When is each approach appropriate?

**Q30 (System design)** — Design a complete data ingestion pipeline for an e-commerce company that receives: (a) customer data as daily CSV files dropped in Azure Blob, (b) order events as a real-time JSON stream written to Azure Blob every 5 minutes, (c) product catalog as weekly Parquet files from a data lake. Specify stages, file formats, table types, COPY INTO approaches, and error handling strategy for each source.
