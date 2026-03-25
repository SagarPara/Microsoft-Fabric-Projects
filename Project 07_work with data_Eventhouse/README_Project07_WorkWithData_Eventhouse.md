# Project 07 — Work with Data in a Microsoft Fabric Eventhouse

**Workspace:** `Project 07_work with data_Eventhouse`
**Eventhouse:** `RTISample`
**KQL Database:** `RTISample`
**KQL Queryset:** `KustoQueryWorkbench_1` · `RTISample` (queryset)
**Eventstream:** `RTISample`
**Real-Time Dashboard:** `RTISample`
**Table:** `Bikestream`
**Dataset:** Real-Time Intelligence Sample (Bike-share docking points)
**Reference:** [MS Learn Lab — Work with data in a Microsoft Fabric eventhouse](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/12-query-data-in-kql-database.html)

---

## Overview

This project focuses on **querying data** inside a KQL database using two different query languages:
- **KQL (Kusto Query Language)** — native to Eventhouse, optimised for time-series and high-frequency data
- **T-SQL (Transact-SQL)** — supported via a T-SQL endpoint that emulates SQL Server, for teams who prefer SQL

Unlike Projects 04 and 05, this exercise uses Fabric's built-in **"Explore Real-Time Intelligence Sample"** to auto-create a pre-populated Eventhouse named `RTISample` with a live `Bikestream` table — so there is no manual pipeline or eventstream setup required here.

The exercise covers the four most common data operations in both languages: **Retrieve → Summarize → Sort → Filter**.

---

## Pre-requisites

- Active Microsoft Fabric trial
- Access to workspace: `Project 07_work with data_Eventhouse`

---

## Workspace Items Overview

| Name | Type | Purpose |
|------|------|---------|
| `RTISample` | Eventhouse | Container with pre-populated KQL database |
| `RTISample` | KQL Database | Contains `Bikestream` table — 104.4K rows |
| `RTISample` | Eventstream | Auto-created sample stream feeding `Bikestream` |
| `RTISample` | Real-Time Dashboard | Auto-created dashboard from the sample |
| `RTISample_queryset` | KQL Queryset | Default queryset bundled with the sample |
| `KustoQueryWorkbench_1` | KQL Queryset | Custom queryset used during this exercise |

> **Note:** OneLake availability is **Enabled** on the `RTISample` KQL database (unlike previous projects where it was disabled). This means the `Bikestream` table is accessible from other Fabric workloads.

---

## Step 1 — Create the Eventhouse Using the RTI Sample

Instead of building from scratch, this exercise uses Fabric's sample:

1. From the left menu, select **Workloads** → click **Real-Time Intelligence**.
2. On the Real-Time Intelligence home page, click **Explore Real-Time Intelligence Sample**.
3. Fabric will automatically create:
   - An Eventhouse named `RTISample`
   - A KQL Database named `RTISample`
   - A `Bikestream` table inside the database
   - An Eventstream, Real-Time Dashboard, and KQL Queryset — all named `RTISample`

4. Open the `RTISample` KQL database and confirm the **Bikestream** table is listed under **Tables**.
5. The **Data Activity Tracker** in the database overview will show ongoing ingestion (104.4K rows at the time of this exercise).

---

## Step 2 — Open the KQL Queryset

1. In the left pane of the Eventhouse, select the **queryset** file (`RTISample_queryset` or open `KustoQueryWorkbench_1`).
2. This is where all KQL and T-SQL queries below will be run.

---

---

# Part A — Query Data Using KQL

KQL is the native, recommended language for querying a KQL database. It uses a **pipe (`|`) operator** to chain operations — each step passes its output to the next.

---

## A1 — Retrieve Data

**Preview 100 rows:**

```kql
Bikestream
| take 100
```

**Select specific columns with `project`:**

```kql
// Use 'project' and 'take' to view a sample number of records
Bikestream
| project Street, No_Bikes
| take 10
```

**Rename a column using `project` with alias:**

```kql
Bikestream
| project Street, ["Number of Empty Docks"] = No_Empty_Docks
| take 10
```

> The `["column name"]` bracket syntax lets you use spaces or special characters in column names/aliases.

---

## A2 — Summarize Data

**Total bikes across all docking points:**

```kql
Bikestream
| summarize ["Total Number of Bikes"] = sum(No_Bikes)
```

**Total bikes grouped by neighbourhood:**

```kql
Bikestream
| summarize ["Total Number of Bikes"] = sum(No_Bikes) by Neighbourhood
| project Neighbourhood, ["Total Number of Bikes"]
```

**Handle null/empty Neighbourhood — label as "Unidentified":**

```kql
Bikestream
| summarize ["Total Number of Bikes"] = sum(No_Bikes) by Neighbourhood
| project Neighbourhood = case(isempty(Neighbourhood) or isnull(Neighbourhood), "Unidentified", Neighbourhood), ["Total Number of Bikes"]
```

> The `case()` function works like an IF-ELSE. `isempty()` catches empty strings, `isnull()` catches null values.

---

## A3 — Sort Data

**Sort by Neighbourhood (ascending) using `sort by`:**

```kql
Bikestream
| summarize ["Total Number of Bikes"] = sum(No_Bikes) by Neighbourhood
| project Neighbourhood = case(isempty(Neighbourhood) or isnull(Neighbourhood), "Unidentified", Neighbourhood), ["Total Number of Bikes"]
| sort by Neighbourhood asc
```

> In KQL, `sort by` and `order by` are interchangeable — both work the same way.

---

## A4 — Filter Data

**Filter to show only Chelsea neighbourhood:**

```kql
Bikestream
| where Neighbourhood == "Chelsea"
| summarize ["Total No of Bikes"] = sum(No_Bikes) by Neighbourhood
| project Neighbourhood = case(isempty(Neighbourhood) or isnull(Neighbourhood), "Unidentified", Neighbourhood), ["Total No of Bikes"]
| order by Neighbourhood asc
```

> `where` is the KQL equivalent of SQL's `WHERE`. Use `==` for equality (not `=`).

---

---

# Part B — Query Data Using T-SQL

The KQL database provides a **T-SQL endpoint** that emulates SQL Server — useful if you or your team are more familiar with SQL. You can run T-SQL queries directly in the same queryset by switching the query mode.

> **Important limitations of the T-SQL endpoint:**
> - Cannot create, alter, or drop tables
> - Cannot insert, update, or delete data
> - Some T-SQL functions and syntax not compatible with KQL are unsupported
> - Use KQL as the primary language where possible — it's more powerful and performant for this type of data

---

## B1 — Retrieve Data

**Top 100 rows:**

```sql
SELECT TOP 100 * FROM Bikestream
```

**Specific columns:**

```sql
SELECT TOP 10 Street, No_Bikes
FROM Bikestream
```

**Rename a column using alias:**

```sql
SELECT TOP 10 Street, No_Empty_Docks AS [Number of Empty Docks]
FROM Bikestream
```

---

## B2 — Summarize Data

**Total bikes:**

```sql
SELECT SUM(No_Bikes) AS [Total Number of Bikes]
FROM Bikestream
```

**Group by Neighbourhood:**

```sql
SELECT Neighbourhood, SUM(No_Bikes) AS [Total Number of Bikes]
FROM Bikestream
GROUP BY Neighbourhood
```

**Handle null/empty Neighbourhood — label as "Unidentified":**

```sql
SELECT CASE
         WHEN Neighbourhood IS NULL OR Neighbourhood = '' THEN 'Unidentified'
         ELSE Neighbourhood
       END AS Neighbourhood,
       SUM(No_Bikes) AS [Total Number of Bikes]
FROM Bikestream
GROUP BY CASE
           WHEN Neighbourhood IS NULL OR Neighbourhood = '' THEN 'Unidentified'
           ELSE Neighbourhood
         END
```

---

## B3 — Sort Data

**Order results by Neighbourhood (ascending):**

```sql
SELECT CASE
         WHEN Neighbourhood IS NULL OR Neighbourhood = '' THEN 'Unidentified'
         ELSE Neighbourhood
       END AS Neighbourhood,
       SUM(No_Bikes) AS [Total Number of Bikes]
FROM Bikestream
GROUP BY CASE
           WHEN Neighbourhood IS NULL OR Neighbourhood = '' THEN 'Unidentified'
           ELSE Neighbourhood
         END
ORDER BY Neighbourhood ASC
```

---

## B4 — Filter Data

**Filter to Chelsea neighbourhood only (using `HAVING`):**

```sql
SELECT CASE
         WHEN Neighbourhood IS NULL OR Neighbourhood = '' THEN 'Unidentified'
         ELSE Neighbourhood
       END AS Neighbourhood,
       SUM(No_Bikes) AS [Total Number of Bikes]
FROM Bikestream
GROUP BY CASE
           WHEN Neighbourhood IS NULL OR Neighbourhood = '' THEN 'Unidentified'
           ELSE Neighbourhood
         END
HAVING Neighbourhood = 'Chelsea'
ORDER BY Neighbourhood ASC
```

> In T-SQL, `HAVING` filters **after** aggregation, while `WHERE` filters before. Since we're filtering on a grouped result, `HAVING` is the correct clause here.

---

---

## KQL vs T-SQL — Side-by-Side Comparison

The Chelsea filter query from `KustoQueryWorkbench_1` shows both versions together — here's a clean comparison:

| Operation | KQL | T-SQL |
|-----------|-----|-------|
| Retrieve rows | `\| take 100` | `SELECT TOP 100 *` |
| Select columns | `\| project Col1, Col2` | `SELECT Col1, Col2` |
| Rename column | `\| project ["New Name"] = Col` | `Col AS [New Name]` |
| Aggregate | `\| summarize sum(Col)` | `SELECT SUM(Col)` |
| Group by | `\| summarize ... by Col` | `GROUP BY Col` |
| Handle nulls | `case(isnull(...), "X", Col)` | `CASE WHEN Col IS NULL THEN 'X' ELSE Col END` |
| Filter rows | `\| where Col == "value"` | `WHERE Col = 'value'` |
| Filter groups | *(use where before summarize)* | `HAVING Col = 'value'` |
| Sort | `\| sort by Col asc` | `ORDER BY Col ASC` |

---

## KQL Reference — All Queries Used

| # | Query | Purpose |
|---|-------|---------|
| 1 | `Bikestream \| take 100` | Preview 100 rows |
| 2 | `Bikestream \| project Street, No_Bikes \| take 10` | Select specific columns |
| 3 | `Bikestream \| project Street, ["Number of Empty Docks"] = No_Empty_Docks \| take 10` | Rename column |
| 4 | `Bikestream \| summarize ["Total Number of Bikes"] = sum(No_Bikes)` | Total bikes |
| 5 | `Bikestream \| summarize ... by Neighbourhood \| project ...` | Group by neighbourhood |
| 6 | Full `case(isempty \| isnull)` version | Handle empty Neighbourhood |
| 7 | Add `\| sort by Neighbourhood asc` | Sort results |
| 8 | Add `\| where Neighbourhood == "Chelsea"` | Filter to Chelsea |

---

## Summary

| Step | Part | Action | Tool |
|------|------|--------|------|
| 1 | Setup | Auto-create RTISample Eventhouse | Real-Time Intelligence Sample |
| A1 | KQL | Retrieve rows with `take` and `project` | KQL Queryset |
| A2 | KQL | Summarize with `sum`, group by, handle nulls with `case` | KQL Queryset |
| A3 | KQL | Sort with `sort by` / `order by` | KQL Queryset |
| A4 | KQL | Filter with `where` | KQL Queryset |
| B1 | T-SQL | Retrieve with `SELECT TOP`, column alias | KQL Queryset (T-SQL mode) |
| B2 | T-SQL | `SUM`, `GROUP BY`, `CASE WHEN` for nulls | KQL Queryset (T-SQL mode) |
| B3 | T-SQL | `ORDER BY ASC` | KQL Queryset (T-SQL mode) |
| B4 | T-SQL | `HAVING` for post-aggregation filter | KQL Queryset (T-SQL mode) |

---

## Notes

- The **`RTISample`** Eventhouse is created entirely automatically by clicking the sample tile — no Eventstream configuration or table setup is needed manually.
- **OneLake availability is Enabled** on this database (`RTISample`) — this is different from the previous Real-Time projects where it was disabled. It means `Bikestream` data is accessible from Lakehouse notebooks or other Fabric tools via OneLake path.
- `sort by` and `order by` are **identical** in KQL — both are valid, and this lab demonstrates they produce the same results.
- The T-SQL `HAVING` clause is needed (not `WHERE`) when filtering on grouped/aggregated results. `WHERE` filters individual rows before grouping; `HAVING` filters the grouped output.
- The **pipe `|`** in KQL serves two purposes: as a query operator separator (chaining steps), and as a logical OR inside brackets. They are context-dependent.
- Bracket notation `["column name"]` in KQL is required when a column name or alias contains spaces — same concept as `[Column Name]` in T-SQL.

---

*Documented by: Sagar Parab | Workspace: Project 07_work with data_Eventhouse | Last updated: March 2026*
