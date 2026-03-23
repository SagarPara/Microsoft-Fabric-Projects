# Project 04 — Real-Time Intelligence with Eventstream in Microsoft Fabric

**Workspace:** `Project 04_Real-Time Intelligence_Eventstream_Fabric`
**Eventhouse:** `my_first_Eventhouse`
**KQL Database:** `my_first_Eventhouse`
**KQL Queryset:** `KustoQueryWorkbench_1`
**Eventstream:** `stock-data`
**Dashboard:** `Stock Dashboard`
**Dataset:** Stock Market sample data (real-time — BMZM, NSFT, HOOJ)
**Reference:** [MS Learn Lab — Get started with Real-Time Intelligence in Microsoft Fabric](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/07-real-time-Intelligence.html)

---

## Overview

This project covers how to work with **Real-Time Intelligence** in Microsoft Fabric. Unlike the previous projects where data was batch-loaded, here we deal with a **live, continuously flowing data stream** — specifically stock market data.

The end-to-end flow looks like this:

```
[Stock Market Sample Source]
        ↓  Real-Time Hub → Add data
[Eventstream: stock-data]          ← captures & routes live stream
        ↓  Destination: Eventhouse table
[KQL Database Table: stock]        ← stores streaming data
        ↓  KQL Query
[KustoQueryWorkbench_1]            ← query & explore in real-time
        ↓  Save to Dashboard
[Stock Dashboard]                  ← live visual (column chart)
        ↓  Set Alert
[Activator Alert]                  ← email alert on price change
```

By the end of this exercise you should be able to:
- Create an **Eventstream** from a sample real-time data source
- Create an **Eventhouse** and route stream data into a KQL table
- Write basic **KQL (Kusto Query Language)** queries to explore live data
- Build a **Real-Time Dashboard** with a live chart tile
- Set up an **Activator alert** to trigger on a data condition

---

## Pre-requisites

- Active Microsoft Fabric trial
- Access to workspace: `Project 04_Real-Time Intelligence_Eventstream_Fabric`

---

## Workspace Items Overview

| Name | Type | Purpose |
|------|------|---------|
| `stock-data` | Eventstream | Captures live stock market stream |
| `my_first_Eventhouse` | Eventhouse | Container for the KQL database |
| `my_first_Eventhouse` | KQL Database | Stores `stock` table with streaming data |
| `KustoQueryWorkbench_1` | KQL Queryset | Saved KQL queries for analysis |
| `stock - example dashboard` | Real-Time Dashboard | Auto-generated sample from the stream |
| `Stock Dashboard` | Real-Time Dashboard | Custom dashboard built during this exercise |

---

## Step-by-Step Setup

---

### Step 1 — Create the Eventstream

An **Eventstream** connects a data source to downstream destinations. Here we use Fabric's built-in **Stock market** sample to simulate a live stream.

1. In the left menu bar, select **Real-Time** hub (pin it via **…** if not visible).
2. In the Real-Time hub, click **Add data**.
3. From the list of sample sources, select **Stock market**.
4. Configure as follows:

   | Setting | Value |
   |---------|-------|
   | Source name | `stock` |
   | Workspace | `Project 04_Real-Time Intelligence_Eventstream_Fabric` |
   | Eventstream name | `stock-data` |

   > The default stream will be auto-named `stock-data-stream`.

5. Click **Next** → **Connect** → then **Open eventstream**.

You should now see the Eventstream canvas with:
- **Source:** `stock` (Active ✅)
- **Stream node:** `stock-data-stream`
- A placeholder on the right saying *"Switch to edit mode to Transform event or add destination"*

The **Data preview** tab at the bottom will show live ticking rows with columns like `time`, `symbol`, `bidPrice`, `askPrice`, etc., for stock tickers **BMZM**, **NSFT**, and **HOOJ**.

---

### Step 2 — Create the Eventhouse

An **Eventhouse** is the Real-Time Intelligence storage layer in Fabric — it contains one or more **KQL databases** where streaming data can be stored and queried.

1. From the left menu, click **Create** → Under *Real-Time Intelligence*, select **Eventhouse**.
2. Name it `my_first_Eventhouse`.
3. Close any tips or prompts. You should see an empty Eventhouse with a KQL database of the same name.

---

### Step 3 — Route Eventstream Data into the Eventhouse Table

Now we connect the `stock-data` eventstream to a new table inside the KQL database.

1. Inside the `my_first_Eventhouse` KQL database page, click **Get data**.
2. Select **Eventstream** → **Existing eventstream**.
3. In the **Select or create a destination table** pane:
   - Create a new table named `stock`
4. In the **Configure the data source** pane:
   - Workspace: your workspace
   - Eventstream: `stock-data`
   - Connection name: `stock-table`
5. Click **Next** through the steps to inspect data, then **Finish**.
6. Close the configuration window.

You should now see the `stock` table listed under **Tables** inside `my_first_Eventhouse`. The database overview will show live ingestion stats — the **Data Activity Tracker** chart confirms rows flowing in continuously.

To verify the connection is working end-to-end:
- Go back to **Real-Time hub** → find `stock-data-stream` in the **…** menu → click **Open eventstream**.
- The eventstream canvas should now show the `stock` table as a **destination** connected to the stream.

---

### Step 4 — Query the Data Using KQL

1. Open the `my_first_Eventhouse` KQL database.
2. Click on the queryset — this opens `KustoQueryWorkbench_1` in the editor.

**Basic query — see the first 100 rows:**

```kql
stock
| take 100
```

Run it to verify data is flowing into the table.

---

**Analytical query — average bid price per stock symbol in the last 5 minutes:**

```kql
stock
| where ["time"] > ago(5m)
| summarize avgPrice = avg(todecimal(bidPrice)) by symbol
| project symbol, avgPrice
```

Run this query. You should see results for **BMZM**, **NSFT**, and **HOOJ** with their respective average bid prices.

> **Tip:** Wait a few seconds and run again — the `avgPrice` values will change as new data streams in. This is the real-time nature of the data.

---

### Step 5 — Create the Real-Time Dashboard

1. In `KustoQueryWorkbench_1`, make sure the average price query (from Step 4) is selected/highlighted.
2. On the toolbar, click **Save to Dashboard** → **Pin to a new dashboard**.
3. Configure as follows:

   | Setting | Value |
   |---------|-------|
   | Dashboard name | `Stock Dashboard` |
   | Tile name | `Average Prices` |

4. Click **Create** and then **Open dashboard**.

The dashboard will open in **Viewing** mode showing a table tile with `symbol` and `avgPrice` columns.

---

**Change the tile to a Column Chart:**

1. Switch to **Editing** mode (top-right toggle).
2. Click the **pencil (Edit)** icon on the **Average Prices** tile.
3. In the **Visual formatting** pane, change **Visual** from *Table* to **Column chart**.
4. Click **Apply changes** at the top of the dashboard.

You should now see a live bar/column chart showing average prices for **NSFT**, **BMZM**, and **HOOJ** — updating automatically as new data comes in. The dashboard time range defaults to **Last 1 hour**.

---

### Step 6 — Set Up an Activator Alert (Optional)

**Activator** is the alerting engine in Fabric Real-Time Intelligence. It monitors data conditions and triggers actions automatically.

1. On the `Stock Dashboard` toolbar, click **Set alert**.
2. Configure the alert as follows:

   | Setting | Value |
   |---------|-------|
   | Run query every | 5 minutes |
   | Check | On each event grouped by |
   | Grouping field | symbol |
   | When | avgPrice |
   | Condition | Increases by |
   | Value | 100 |
   | Action | Send me an email |
   | Workspace | `Project 04_Real-Time Intelligence_Eventstream_Fabric` |
   | Item | Create a new item |
   | New item name | *(any unique name)* |

3. Click **Create** and wait for it to save. Close the confirmation pane.

To check alert history:
- Go to the workspace → Open the **Activator** item that was created.
- Under the `avgPrice` node, select your alert's unique identifier.
- Click the **History** tab to see if it has been triggered.

> The alert fires only if the average price of any stock symbol increases by more than 100 in a 5-minute window. If the market simulation doesn't hit that threshold, history will be empty — that's expected.

---

## KQL Reference — Queries Used in This Exercise

| Query | Purpose |
|-------|---------|
| `stock \| take 100` | Preview first 100 rows from the table |
| `stock \| where ["time"] > ago(5m) \| summarize avgPrice = avg(todecimal(bidPrice)) by symbol \| project symbol, avgPrice` | Average bid price per stock in last 5 min |

---

## Key Concepts Covered

| Concept | What it is |
|---------|-----------|
| **Eventstream** | A Fabric item that captures, routes, and transforms real-time event data |
| **Eventhouse** | Storage container optimised for time-series and high-frequency streaming data |
| **KQL Database** | The database inside an Eventhouse; queried using Kusto Query Language |
| **KQL Queryset** | A saved file of KQL queries, similar to a SQL query editor |
| **Real-Time Dashboard** | A live dashboard that auto-refreshes from a KQL query |
| **Activator** | An alert engine that monitors conditions on streaming data and triggers actions |

---

## Summary

| Step | Action | Tool Used |
|------|--------|-----------|
| 1 | Create Eventstream from Stock Market sample | Real-Time Hub |
| 2 | Create Eventhouse with KQL database | Fabric Real-Time Intelligence |
| 3 | Route stream into `stock` table in KQL DB | Eventstream destination config |
| 4 | Query live data using KQL | KQL Queryset (`KustoQueryWorkbench_1`) |
| 5 | Save query to Real-Time Dashboard as column chart | Stock Dashboard |
| 6 | Set Activator alert on avgPrice increase | Activator |

---

## Notes

- The **stock** source in the eventstream is a Fabric-provided **sample** — it simulates a real stock exchange feed with tickers BMZM, NSFT, and HOOJ.
- The `["time"]` syntax with square brackets is needed in KQL because `time` is a reserved keyword in Kusto.
- The **Eventhouse** compresses data efficiently — as seen in the screenshot, 795.9 KB of original data compresses to 75.4 KB.
- **OneLake availability** on the KQL database is disabled by default — enabling it makes the tables accessible from other Fabric workloads like notebooks or pipelines.
- Unlike Lakehouse tables, data in the Eventhouse is optimised for **time-series queries** and very high ingestion rates — not for batch transforms.

---

*Documented by: Sagar Parab | Workspace: Project 04_Real-Time Intelligence_Eventstream_Fabric | Last updated: March 2026*
