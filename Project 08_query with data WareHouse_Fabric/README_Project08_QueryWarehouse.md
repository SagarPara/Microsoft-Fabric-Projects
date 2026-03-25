# Project 08 — Query a Data Warehouse in Microsoft Fabric

**Workspace:** `Project 08_query with data WareHouse_Fabric`
**Warehouse:** `sample-dw`
**Schema:** `dbo`
**Views created:** `vw_JanTrip` · `vw_Pay...`
**Saved queries:** `AnalysisData_query1` · `Verify_data_Consist...`
**Dataset:** Taxi ride analysis sample (NYC-style taxi trips)
**Reference:** [MS Learn Lab — Query a data warehouse in Microsoft Fabric](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/06b-data-warehouse-query.html)

---

## Overview

This project covers how to work with a **Fabric Data Warehouse** — a relational database built for large-scale analytics using full T-SQL support. Unlike a Lakehouse (which stores files and Delta tables), a Fabric Warehouse gives you a proper SQL engine with schemas, views, DDL/DML support, and native integration with Power BI in DirectLake mode.

The exercise uses a **pre-built sample warehouse** (`sample-dw`) that comes loaded with NYC taxi ride data. You'll write SQL queries to analyse trips, check data quality, clean bad records, and save a filtered query as a reusable view.

**Three areas covered:**
1. **Analyse data** — aggregate queries across multiple tables (JOINs, GROUP BY)
2. **Verify data consistency** — find and remove bad records
3. **Save as view** — create `vw_JanTrip` for filtered, reusable reporting

---

## Pre-requisites

- Active Microsoft Fabric trial
- Access to workspace: `Project 08_query with data WareHouse_Fabric`

---

## Workspace Items Overview

| Name | Type | Purpose |
|------|------|---------|
| `sample-dw` | Warehouse | Pre-populated taxi rides data warehouse |

---

## Warehouse Structure (`sample-dw`)

The warehouse uses the `dbo` schema and contains the following tables:

| Table | Description |
|-------|-------------|
| `Trip` | Core fact table — trip records with duration, distance, revenue |
| `Date` | Date dimension — DayName, MonthName, Month number, etc. |
| `Geography` | Location dimension — City, pickup/dropoff areas |
| `HackneyLicense` | Taxi licence dimension |
| `Medallion` | Taxi medallion dimension |
| `Time` | Time dimension |
| `Weather` | Weather data dimension |

**Views in `dbo`:**

| View | Columns | Description |
|------|---------|-------------|
| `vw_JanTrip` | DayName, AvgDuration, AvgDistance | Average trip stats per day of week — filtered to January |
| `vw_Pay...` | *(payment-related columns)* | Payment amount summary view |

**Saved queries (My queries):**

| Query name | Purpose |
|------------|---------|
| `AnalysisData_query1` | Analysis queries (revenue, duration, top locations) |
| `Verify_data_Consist...` | Data consistency checks and cleanup |

---

## Step 1 — Create the Sample Data Warehouse

1. From the left menu, click **Create** → Under *Data Warehouse*, select **Sample warehouse**.
2. Name it `sample-dw` → Click **Create**.

After ~1 minute, Fabric creates and populates the warehouse with taxi ride sample data. The **Explorer** pane on the left will show `sample-dw` with the full schema tree: **Schemas → dbo → Tables / Views / Functions / Stored Procedures**.

---

## Step 2 — Analyse Data with SQL Queries

Open **New SQL query** from the toolbar. The SQL editor supports IntelliSense, syntax highlighting, and both DDL and DML statements.

---

### Query 1 — Total Trips and Revenue by Month

```sql
SELECT
    D.MonthName,
    COUNT(*) AS TotalTrips,
    SUM(T.TotalAmount) AS TotalRevenue
FROM dbo.Trip AS T
JOIN dbo.[Date] AS D
    ON T.[DateID] = D.[DateID]
GROUP BY D.MonthName
```

> Results show total trip count and total revenue grouped by month name. Useful for spotting seasonal patterns.

---

### Query 2 — Average Trip Duration and Distance by Day of Week

```sql
SELECT
    D.DayName,
    AVG(T.TripDurationSeconds) AS AvgDuration,
    AVG(T.TripDistanceMiles) AS AvgDistance
FROM dbo.Trip AS T
JOIN dbo.[Date] AS D
    ON T.[DateID] = D.[DateID]
GROUP BY D.DayName
```

> Results show average trip duration (seconds) and average distance (miles) for each day of the week — all 7 days appear in the output.

---

### Query 3 — Top 10 Cities by Trip Count (Dropoff Location)

```sql
SELECT TOP 10
    G.City,
    COUNT(*) AS TotalTrips
FROM dbo.Trip AS T
JOIN dbo.Geography AS G
    ON T.DropoffGeographyID = G.GeographyID
GROUP BY G.City
ORDER BY TotalTrips DESC
```

> Returns the 10 most popular dropoff cities. Saved as part of `AnalysisData_query1`.

Close all query tabs after running these.

---

## Step 3 — Verify Data Consistency

Data quality checks are essential before using data for reporting. Open a new SQL query tab for each check below.

---

### Check 1 — Trips with Unusually Long Duration (> 24 hours)

```sql
-- Check for trips with unusually long duration
SELECT COUNT(*) FROM dbo.Trip WHERE TripDurationSeconds > 86400; -- 86400 seconds = 24 hours
```

> Returns the count of trips longer than 24 hours. These are likely data entry errors or system anomalies.

---

### Check 2 — Trips with Negative Duration

```sql
-- Check for trips with negative trip duration
SELECT COUNT(*) FROM dbo.Trip WHERE TripDurationSeconds < 0;
```

> Negative duration is impossible — these records are clearly invalid.

---

### Fix — Delete Negative Duration Records

```sql
-- Remove trips with negative trip duration
DELETE FROM dbo.Trip WHERE TripDurationSeconds < 0;
```

> Removes the invalid records. These queries are saved as `Verify_data_Consist...` in **My queries**.

> **Note:** Deleting is one approach. An alternative is to replace invalid values with the mean or median — depends on your reporting requirements.

Close all query tabs after running these.

---

## Step 4 — Save Query as a View (`vw_JanTrip`)

Views are saved, reusable SQL queries that behave like virtual tables. They are useful for sharing filtered or aggregated datasets with report consumers without exposing the raw tables.

---

### Step 4a — Write the Filtered Query

Open a new SQL query tab and run the base query first (same as Query 2 above):

```sql
SELECT
    D.DayName,
    AVG(T.TripDurationSeconds) AS AvgDuration,
    AVG(T.TripDistanceMiles) AS AvgDistance
FROM dbo.Trip AS T
JOIN dbo.[Date] AS D
    ON T.[DateID] = D.[DateID]
GROUP BY D.DayName
```

---

### Step 4b — Add a January Filter

Modify the query to add `WHERE D.Month = 1` to restrict results to January only:

```sql
SELECT
    D.DayName,
    AVG(T.TripDurationSeconds) AS AvgDuration,
    AVG(T.TripDistanceMiles) AS AvgDistance
FROM dbo.Trip AS T
JOIN dbo.[Date] AS D
    ON T.[DateID] = D.[DateID]
WHERE D.Month = 1
GROUP BY D.DayName
```

---

### Step 4c — Save as View

1. **Select all** the text of the SELECT statement above.
2. Next to the **▷ Run** button, click **Save as view**.
3. Name the view `vw_JanTrip` → Click **OK**.

The view is now saved under **Explorer → Schemas → dbo → Views → vw_JanTrip**.

Click on `vw_JanTrip` in the Explorer to see the **Data preview** — it shows 7 rows (one per day of the week) with columns:

| DayName | AvgDuration | AvgDistance |
|---------|-------------|-------------|
| Tuesday | 2079 | 7.9756... |
| Thursday | 2089 | 7.9617... |
| Friday | 2098 | 7.9532... |
| Sunday | 2082 | 8.0696... |
| Saturday | 2074 | 7.9444... |
| Monday | 2116 | 8.0374... |
| Wednesday | 2043 | 8.0207... |

> Monday has the longest average trip duration (~35 min), while Wednesday is the shortest. Average distances are fairly consistent across all days (~8 miles).

Close all query tabs after saving.

---

## What Makes a Fabric Warehouse Different from a Lakehouse?

| Feature | Fabric Warehouse | Fabric Lakehouse |
|---------|-----------------|-----------------|
| Storage format | Parquet in OneLake (Delta) | Delta tables in OneLake |
| Query language | **Full T-SQL** (DDL + DML) | PySpark / SQL via notebook or endpoint |
| Schema support | Full schema (dbo, views, stored procs) | Limited (Tables + Files) |
| Views | ✅ Native SQL views | ✅ Via SQL analytics endpoint |
| Delete/Update | ✅ Supported | Limited (Delta only) |
| Best for | Structured analytics, reporting teams familiar with SQL | Data engineering, ML, mixed workloads |
| Power BI integration | DirectLake mode | DirectLake mode |

---

## Summary

| Step | Action | Tool |
|------|--------|------|
| 1 | Create `sample-dw` using Sample Warehouse template | Fabric Data Warehouse |
| 2a | Query total trips and revenue by month | SQL query editor |
| 2b | Query average duration and distance by day | SQL query editor |
| 2c | Query top 10 dropoff cities | SQL query editor |
| 3a | Check for trips > 24 hours | SQL (data quality check) |
| 3b | Check for negative trip durations | SQL (data quality check) |
| 3c | Delete negative duration records | SQL DELETE statement |
| 4 | Save January trips query as `vw_JanTrip` view | Save as view |

---

## Notes

- The **`dbo.[Date]`** table name requires square brackets because `Date` is a reserved word in T-SQL.
- `Save as view` only works when the **full SELECT statement text is selected** before clicking the button — if nothing is selected, the option may not work as expected.
- The `vw_JanTrip` view will appear under **Schemas → dbo → Views** in the Explorer immediately after saving. If it doesn't show, refresh the Explorer pane.
- The Fabric Warehouse SQL editor supports **DDL** (CREATE, ALTER, DROP), **DML** (SELECT, INSERT, UPDATE, DELETE), and **DCL** (GRANT, REVOKE) — much more capable than the T-SQL endpoint available on a KQL database (covered in Project 07).
- **`TripDurationSeconds > 86400`** — 86,400 seconds = 60 × 60 × 24 = exactly 24 hours. Any trip longer than this is flagged as anomalous.
- Rather than deleting bad records, a safer production approach is to move them to a separate quarantine table for review before permanent removal.

---

*Documented by: Sagar Parab | Workspace: Project 08_query with data WareHouse_Fabric | Last updated: March 2026*
