# Project 05 — Ingest Real-Time Data with Eventstream in Microsoft Fabric

**Workspace:** `Project 05_Real-Time_with_Eventstream-Fabric`
**Eventhouse:** `real-time_data_eventhouse`
**KQL Database:** `real-time_data_eventhouse`
**KQL Queryset:** `bike_per_Street_within_each_5_seconds`
**Eventstream:** `Bicycle-data`
**Dashboard:** `Dashboard_Bike_on_street_5_seconds`
**Dataset:** Bicycle sample data (real-time — bike-share docking points across city streets)
**Reference:** [MS Learn Lab — Ingest real-time data with Eventstream in Microsoft Fabric](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/09-real-time-analytics-eventstream.html)

---

## Overview

This project goes a step further than Project 04. Instead of just routing a stream into a table, here we also **transform the stream mid-flow** using a **Group By** aggregation before storing it. This introduces the concept of **event processing before ingestion** — a pattern commonly used when you want to aggregate high-frequency data into summarised windows rather than storing every raw event.

The scenario simulates a city **bike-share system** where sensors at docking stations report bike availability on each street every few seconds.

**End-to-end flow:**

```
[Bicycles sample source]
        ↓
[Eventstream: Bicycle-data]
  ├── Destination 1 (direct, no transform)
  │         ↓
  │   [KQL Table: bikes]          ← raw individual bike-point records
  │
  └── Transform: GroupByStreet
       (Tumbling window, 5 sec, SUM of No_Bikes per Street)
                ↓
        [KQL Table: bikes-by-street]  ← aggregated bikes per street, per 5s window
```

By the end of this exercise you should understand:
- How to create an **Eventhouse** and **Eventstream** from scratch in the right order
- How to add a **sample data source** to an Eventstream
- How to send raw stream data directly to a KQL table
- How to add a **Group By transformation** (tumbling window) to the stream
- How to route the transformed output to a second KQL table
- How to write KQL queries on both raw and aggregated tables
- How to build a **Real-Time Dashboard** from a KQL query

---

## Pre-requisites

- Active Microsoft Fabric trial
- Access to workspace: `Project 05_Real-Time_with_Eventstream-Fabric`

---

## Workspace Items Overview

| Name | Type | Purpose |
|------|------|---------|
| `Bicycle-data` | Eventstream | Captures and transforms live bicycle data stream |
| `real-time_data_eventhouse` | Eventhouse | Container for the KQL database |
| `real-time_data_eventhouse` | KQL Database | Stores `bikes` and `bikes-by-street` tables |
| `bike_per_Street_within_each_5_seconds` | KQL Queryset | Saved KQL queries for the aggregated table |
| `Dashboard_Bike_on_street_5_seconds` | Real-Time Dashboard | Live view of bikes per street per 5-second window |

---

## Step-by-Step Setup

---

### Step 1 — Create the Eventhouse

Unlike Project 04 where we started from the Real-Time Hub, here we **start by creating the Eventhouse first**, then build the Eventstream from within the KQL database.

1. In your workspace, click **+ New item** → Under *Real-Time Intelligence*, select **Eventhouse**.
2. Name it `real-time_data_eventhouse`.
3. Close any prompts. You should see the empty Eventhouse with a KQL database of the same name listed on the left.
4. Click the **KQL database** (`real-time_data_eventhouse`) to open it.

> There are no tables yet — we'll create them by connecting an Eventstream.

---

### Step 2 — Create the Eventstream

We create the Eventstream directly from within the KQL database page.

1. Inside the KQL database, click **Get data**.
2. Select **Eventstream** → **New eventstream**.
3. Name it `Bicycle-data` → Click **Create**.

You'll be redirected to the Eventstream editor canvas — empty, ready for sources and destinations.

---

### Step 3 — Add the Bicycles Sample Source

1. On the Eventstream canvas, click **Use sample data**.
2. Configure as follows:

   | Setting | Value |
   |---------|-------|
   | Source name | `Bicycles` |
   | Sample data | Bicycles |

3. Confirm. The canvas will now show:
   - **Bicycles** source node (Active ✅) on the left
   - **Bicycle-data** stream node in the middle
   - **Bicycle-data-stream** as the default stream output

The **Data preview** tab at the bottom shows live rows with fields like: `BikepointID`, `Street`, `Neighbourhood`, `Latitude`, `Longitude`, `No_Bikes`, `No_Empty_Docks`.

---

### Step 4 — Add First Destination: `bikes` Table (Raw Data)

1. Select the **Transform events or add destination** tile on the canvas.
2. Search for and select **Eventhouse**.
3. Configure the destination:

   | Setting | Value |
   |---------|-------|
   | Data ingestion mode | Event processing before ingestion |
   | Destination name | `bikes-table` |
   | Workspace | `Project 05_Real-Time_with_Eventstream-Fabric` |
   | Eventhouse | `real-time_data_eventhouse` |
   | KQL database | `real-time_data_eventhouse` |
   | Destination table | Create new table → `bikes` |
   | Input data format | JSON |

4. Click **Save** in the Eventhouse pane.
5. Click **Publish** on the toolbar.
6. Wait ~1 minute for the destination to become **Active**.
7. Select the `bikes-table` node → check the **Data preview** pane to confirm rows are flowing in.

---

### Step 5 — Query the Raw `bikes` Table

1. From the left menu, open the `real-time_data_eventhouse` KQL database.
2. Click **Refresh** until you see the `bikes` table listed under Tables.
3. Click **…** on the `bikes` table → **Query table** → **Records ingested in the last 24 hours**.

The following query is auto-generated:

```kql
// See the most recent data - records ingested in the last 24 hours.
bikes
| where ingestion_time() between (now(-1d) .. now())
```

Run it to confirm data is landing in the table.

---

### Step 6 — Add a Group By Transformation

Now we go back to the Eventstream and add a **transformation layer** that aggregates bicycle counts per street in 5-second windows before writing to a second table.

1. From the left menu, open the `Bicycle-data` Eventstream.
2. Click **Edit** on the toolbar to enter edit mode.
3. In the **Transform events** menu, select **Group by** — this adds a new `Group by` node to the canvas.
4. **Drag a connection** from the output of the **Bicycle-data** stream node → to the input of the new **Group by** node.
5. Click the **pencil icon** on the **Group by** node to configure it:

   | Setting | Value |
   |---------|-------|
   | Operation name | `GroupByStreet` |
   | Aggregate type | Sum |
   | Field | `No_Bikes` → click **Add** to create `SUM of No_Bikes` |
   | Group aggregations by | `Street` |
   | Time window | Tumbling |
   | Duration | 5 seconds |
   | Offset | 0 seconds |

   > This tells Fabric: *"Every 5 seconds, sum up all No_Bikes values grouped by Street and emit one row per street."*

6. Save the configuration. You'll notice an **error indicator** on the canvas — this is expected because the GroupByStreet output has no destination yet.

---

### Step 7 — Add Second Destination: `bikes-by-street` Table (Transformed Data)

1. Click the **+** icon to the right of the **GroupByStreet** node.
2. Select **Eventhouse** and configure:

   | Setting | Value |
   |---------|-------|
   | Data ingestion mode | Event processing before ingestion |
   | Destination name | `bikes-by-street-table` |
   | Workspace | `Project 05_Real-Time_with_Eventstream-Fabric` |
   | Eventhouse | `real-time_data_eventhouse` |
   | KQL database | `real-time_data_eventhouse` |
   | Destination table | Create new table → `bikes-by-street` |
   | Input data format | JSON |

3. Click **Save** → then **Publish** on the toolbar.
4. Wait ~1 minute for the `bikes-by-street-table` destination to become **Active** (green ✅).
5. Select the `bikes-by-street-table` node → check **Data preview** to confirm grouped data is flowing in.

The transformed data will have these columns:
- `Street` — the street name
- `SUM_No_Bikes` — total bikes on that street in the window
- `Window_End_Time` — timestamp marking the end of each 5-second tumbling window

---

### Step 8 — Query the Transformed `bikes-by-street` Table

1. Go back to the `real-time_data_eventhouse` KQL database.
2. Click **Refresh** → you should now see both `bikes` and `bikes-by-street` under Tables.
3. Click **…** on `bikes-by-street` → **Query data** → **Show any 100 records**.

Auto-generated query:

```kql
['bikes-by-street']
| take 100
```

Now modify it to see the total bikes per street per 5-second window, sorted by most recent:

```kql
['bikes-by-street']
| summarize TotalBikes = sum(tolong(SUM_No_Bikes)) by Window_End_Time, Street
| sort by Window_End_Time desc, Street asc
```

Run the query. The results show each street with the total bike count observed within each 5-second window — this is the `bike_per_Street_within_each_5_seconds` queryset saved in the workspace.

---

### Step 9 — Real-Time Dashboard

The **Dashboard_Bike_on_street_5_seconds** dashboard visualises the query results from the `bikes-by-street` table. It shows a live table tile (`BikeOnStreetPer5Seconds`) with columns:

| Column | Description |
|--------|-------------|
| `Window_End_Time` | End of the 5-second window |
| `Street` | Street name (e.g., Fawcett Close, Flood Street) |
| `TotalBikes` | Total bikes on that street in the window |

The dashboard time range defaults to **Last 1 hour** and refreshes live as new 5-second windows come in.

To create or update this dashboard:
1. Run the `bikes-by-street` KQL query above in the KQL Queryset.
2. Click **Save to Dashboard** on the toolbar → pin to a new or existing dashboard.
3. Name the dashboard `Dashboard_Bike_on_street_5_seconds` and the tile `BikeOnStreetPer5Seconds`.

---

## KQL Reference — Queries Used in This Exercise

| Query | Table | Purpose |
|-------|-------|---------|
| `bikes \| where ingestion_time() between (now(-1d) .. now())` | `bikes` | View raw records from the last 24 hours |
| `['bikes-by-street'] \| take 100` | `bikes-by-street` | Quick preview of grouped data |
| `['bikes-by-street'] \| summarize TotalBikes = sum(tolong(SUM_No_Bikes)) by Window_End_Time, Street \| sort by Window_End_Time desc, Street asc` | `bikes-by-street` | Total bikes per street per 5-second window |

---

## Key Concepts Covered

| Concept | What it is |
|---------|-----------|
| **Eventstream** | Captures, transforms, and routes live event data to one or more destinations |
| **Sample data source** | Fabric built-in simulated sources (Bicycles, Stock Market, etc.) useful for practice |
| **Event processing before ingestion** | Transform or filter data in the stream *before* it lands in the destination table |
| **Group By (Tumbling Window)** | Aggregates events into fixed-size, non-overlapping time buckets (e.g., every 5 seconds) |
| **Multiple destinations** | A single stream can fan out to multiple tables with different transformations |
| **KQL Queryset** | Saved, reusable KQL queries; can be pinned to dashboards |
| **Real-Time Dashboard** | Auto-refreshing dashboard connected directly to a KQL query |

---

## Difference from Project 04

| | Project 04 | Project 05 |
|--|-----------|-----------|
| Data source | Stock market (symbols) | Bicycles (city bike-share) |
| Transformation | None — raw stream to table | GroupByStreet — 5-second tumbling window aggregation |
| Tables created | 1 (`stock`) | 2 (`bikes` + `bikes-by-street`) |
| Stream fan-out | Single destination | Two destinations (raw + transformed) |
| New concept | Activator alert | Event processing / Tumbling window |

---

## Summary

| Step | Action | Tool Used |
|------|--------|-----------|
| 1 | Create Eventhouse and KQL database | Fabric Real-Time Intelligence |
| 2 | Create Eventstream (`Bicycle-data`) from KQL database | Eventstream |
| 3 | Add Bicycles sample source | Eventstream source config |
| 4 | Route raw stream → `bikes` table | Eventhouse destination |
| 5 | Query raw data in KQL | KQL Queryset |
| 6 | Add GroupByStreet transform (5-sec tumbling window) | Eventstream Group By node |
| 7 | Route transformed stream → `bikes-by-street` table | Eventhouse destination |
| 8 | Query aggregated data in KQL | KQL Queryset (`bike_per_Street_within_each_5_seconds`) |
| 9 | Build Real-Time Dashboard | `Dashboard_Bike_on_street_5_seconds` |

---

## Notes

- The **`['bikes-by-street']`** syntax with square brackets is required in KQL because the table name contains a hyphen, which is not a valid identifier character without quoting.
- The **Tumbling window** emits exactly one output row per group per time period — no overlapping. This is different from a Sliding or Hopping window which can produce overlapping results.
- Both destinations (`bikes-table` and `bikes-by-street-table`) use **"Event processing before ingestion"** mode — this is required when you have a transformation node in the path.
- The Eventhouse stats (53K rows ingested, 284.9KB → 34.1KB compressed) confirm active continuous ingestion during the session.
- Once you stop the Fabric session or delete the Eventstream, data will stop flowing. The tables will retain whatever was ingested up to that point.

---

*Documented by: Sagar Parab | Workspace: Project 05_Real-Time_with_Eventstream-Fabric | Last updated: March 2026*
