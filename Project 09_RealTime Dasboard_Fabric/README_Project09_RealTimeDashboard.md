# Project 09 — Get Started with Real-Time Dashboards in Microsoft Fabric

**Workspace:** `Project 09_RealTime Dasboard_Fabric`
**Eventhouse:** `RTDashboard`
**KQL Database:** `RTDashboard`
**KQL Queryset:** `RTDashboard_queryset`
**Eventstream:** `Bicycle-data`
**Dashboard:** `bikes-dashboard` (2 pages)
**Dataset:** Bicycle sample data (city bike-share docking points — London)
**Reference:** [MS Learn Lab — Get started with Real-Time Dashboards in Microsoft Fabric](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/13-real-time-dashboards.html)

---

## Overview

This project builds a **multi-tile, multi-page Real-Time Dashboard** in Microsoft Fabric that visualises live city bike-share data. Unlike Project 04/05 where dashboards were a side output, here the dashboard is the primary deliverable — and the exercise goes deeper into dashboard features: base queries, parameters, auto-refresh, and multi-page layouts.

**End-to-end flow:**

```
[Bicycles sample source]
        ↓
[Eventstream: Bicycle-data]
  (Bicycle-data-stream → Eventhouse destination, no transform)
        ↓
[KQL Table: bikes]        ← raw live data, 32.6K rows
        ↓
[bikes-dashboard]
  Page 1:
    ├── Tile 1: "Bikes and Docks" — stacked bar chart (No_Bikes + No_Empty_Docks by Neighbourhood)
    └── Tile 2: "Bike Locations" — map (Latitude/Longitude, bubble size = No_Bikes)
  Page 2:
    └── Tile 3: Latest observation per Neighbourhood (table)
  + base_bike_data base query (shared across tiles)
  + Neighbourhood parameter (multi-select filter)
  + Auto refresh: every 30 minutes
```

By the end of this exercise you should understand how to:
- Create a Real-Time Dashboard and connect it to a KQL data source
- Add tiles with different visual types (table → bar chart → map)
- Create a **base query** to centralise shared KQL logic across tiles
- Add a **parameter** (multi-select dropdown) to filter dashboard data
- Add a **second page** to the dashboard
- Configure **auto-refresh** on the dashboard

---

## Pre-requisites

- Active Microsoft Fabric trial
- Access to workspace: `Project 09_RealTime Dasboard_Fabric`

---

## Workspace Items Overview

| Name | Type | Purpose |
|------|------|---------|
| `Bicycle-data` | Eventstream | Captures live bicycle sample data |
| `RTDashboard` | Eventhouse | Container for the KQL database |
| `RTDashboard` | KQL Database | Stores `bikes` table — 32.6K rows |
| `RTDashboard_queryset` | KQL Queryset | Auto-created queryset for the database |
| `bikes-dashboard` | Real-Time Dashboard | Main dashboard with 2 pages |

---

## Step 1 — Create the Eventhouse

1. From the left menu, click **Create** → Under *Real-Time Intelligence*, select **Eventhouse**.
2. Name it `RTDashboard`.
3. Close any tips. The Eventhouse opens with a KQL database of the same name.
4. Click the **KQL database** (`RTDashboard`) to open it — no tables yet.

---

## Step 2 — Create the Eventstream and Connect to Eventhouse

1. Inside the KQL database, click **Get data** → **Eventstream** → **New eventstream**.
2. Name it `Bicycle-data` → Click **Create**.
3. On the Eventstream canvas, click **Use sample data** → Source name: `Bicycles` → Sample: **Bicycles** → Confirm.
4. In the **Add destination** dropdown, select **Eventhouse**.
5. Configure the destination:

   | Setting | Value |
   |---------|-------|
   | Data ingestion mode | Event processing before ingestion |
   | Destination name | `bikes-table` |
   | Workspace | `Project 09_RealTime Dasboard_Fabric` |
   | Eventhouse | `RTDashboard` |
   | KQL database | `RTDashboard` |
   | Destination table | Create new → `bikes` |
   | Input data format | JSON |

6. Click **Save** → Connect the **Bicycle-data** node output to the **bikes-table** node → Click **Publish**.
7. Wait ~1 minute for the destination to become **Active** (green ✅).
8. Select the `bikes-table` node → click **Refresh** in the Data preview pane to confirm data is flowing.

> **Note:** This eventstream is simpler than Project 05 — it routes data **directly** to the Eventhouse with no Group By transformation. The raw bike-point records land straight into the `bikes` table.

---

## Step 3 — Create the Real-Time Dashboard

1. From the left menu, click **Create** → Under *Real-Time Intelligence*, select **Real-Time Dashboard**.
2. Name it `bikes-dashboard` → Click **Create**.

A new empty dashboard opens.

---

### Step 3a — Add the Data Source

1. In the dashboard toolbar, click **New data source** → Select **Eventhouse / KQL Database**.
2. Configure:

   | Setting | Value |
   |---------|-------|
   | Display name | `Bike Rental Data` |
   | Database | `RTDashboard` (default database in your eventhouse) |
   | Passthrough identity | Selected |

3. Click **Add**.

---

### Step 3b — Add Tile 1: Bikes and Docks (Stacked Bar Chart)

1. On the dashboard canvas, click **Add tile**.
2. In the query editor, make sure **Bike Rental Data** is selected, then enter:

```kql
bikes
| where ingestion_time() between (ago(30min) .. now())
| summarize latest_observation = arg_max(ingestion_time(), *) by Neighbourhood
| project Neighbourhood, latest_observation, No_Bikes, No_Empty_Docks
| order by Neighbourhood asc
```

3. Click **Run** → verify results show one row per neighbourhood with bike and dock counts.
4. Click **Apply changes** — tile shows as a table first.
5. Click the **Edit (pencil) icon** on the tile → In **Visual Formatting**, set:

   | Property | Value |
   |----------|-------|
   | Tile name | `Bikes and Docks` |
   | Visual type | Bar chart |
   | Visual format | Stacked bar chart |
   | Y columns | `No_Bikes`, `No_Empty_Docks` |
   | X column | `Neighbourhood` |
   | Series columns | Infer |
   | Legend location | Bottom |

6. Click **Apply changes** → Resize the tile to take the **full left half** of the dashboard.

> The stacked bar chart shows neighbourhoods like Bankside, Battersea, Belgravia, Chelsea, Fitzrovia, Knightsbridge, London Bridge, Mile End, etc. — each bar split into orange (bikes available) and blue (empty docks).

---

### Step 3c — Add Tile 2: Bike Locations (Map)

1. In the toolbar, click **New tile**.
2. Enter the following KQL query:

```kql
bikes
| where ingestion_time() between (ago(30min) .. now())
| summarize latest_observation = arg_max(ingestion_time(), *) by Neighbourhood
| project Neighbourhood, latest_observation, Latitude, Longitude, No_Bikes
| order by Neighbourhood asc
```

3. Click **Run** → verify results include Latitude/Longitude per neighbourhood.
4. Click **Apply changes** → tile shows as table first.
5. Click the **Edit icon** → In **Visual Formatting**, set:

   | Property | Value |
   |----------|-------|
   | Tile name | `Bike Locations` |
   | Visual type | Map |
   | Define location by | Latitude and longitude |
   | Latitude column | `Latitude` |
   | Longitude column | `Longitude` |
   | Label column | `Neighbourhood` |
   | Size | Show |
   | Size column | `No_Bikes` |

6. Click **Apply changes** → Resize the map tile to fill the **right half** of the dashboard.

> The map shows London with coloured bubbles at each neighbourhood — bubble size reflects the number of bikes available. Neighbourhoods visible include Chelsea, Battersea, Camden, Southwark, Strand, Mile End, etc.

---

## Step 4 — Create a Base Query

Both tiles use similar KQL logic (same `where` and `summarize` clauses). A **base query** centralises this shared logic so you don't duplicate it — and future changes only need to be made in one place.

1. In the dashboard toolbar, click **Base queries** → click **+ Add**.
2. Set **Variable name** to `base_bike_data`, ensure **Bike Rental Data** source is selected.
3. Enter:

```kql
bikes
| where ingestion_time() between (ago(30min) .. now())
| summarize latest_observation = arg_max(ingestion_time(), *) by Neighbourhood
```

4. Click **Run** to verify it returns all required columns → Click **Done** → Close the Base queries pane.

---

### Update Tile 1 to Use Base Query

Edit the **Bikes and Docks** tile and replace the full query with:

```kql
base_bike_data
| project Neighbourhood, latest_observation, No_Bikes, No_Empty_Docks
| order by Neighbourhood asc
```

Click **Apply changes** → verify bar chart still displays correctly.

---

### Update Tile 2 to Use Base Query

Edit the **Bike Locations** tile and replace the full query with:

```kql
base_bike_data
| project Neighbourhood, latest_observation, No_Bikes, Latitude, Longitude
| order by Neighbourhood asc
```

Click **Apply changes** → verify map still shows all neighbourhoods.

---

## Step 5 — Add a Neighbourhood Parameter

A **parameter** adds an interactive filter to the dashboard — users can select one or more neighbourhoods to filter all tiles at once.

1. On the dashboard toolbar → **Manage** tab → click **Parameters**.
2. Delete any auto-created parameters (e.g., Time range).
3. Click **+ Add** and configure:

   | Setting | Value |
   |---------|-------|
   | Label | `Neighbourhood` |
   | Parameter type | Multiple selection |
   | Description | `Choose neighbourhoods` |
   | Variable name | `selected_neighbourhoods` |
   | Data type | string |
   | Show on pages | Select all |
   | Source | Query |
   | Data source | Bike Rental Data |
   | Edit query | `bikes \| distinct Neighbourhood \| order by Neighbourhood asc` |
   | Value column | Neighbourhood |
   | Label column | Match value selection |
   | Add "Select all" value | Selected |
   | "Select all" sends empty string | Selected |
   | Auto-reset to default value | Selected |
   | Default value | Select all |

4. Click **Done**.

---

### Update Base Query to Respect the Parameter

1. Open **Base queries** → Edit `base_bike_data`.
2. Update the `where` clause to include the neighbourhood filter:

```kql
bikes
| where ingestion_time() between (ago(30min) .. now())
  and (isempty(['selected_neighbourhoods']) or Neighbourhood in (['selected_neighbourhoods']))
| summarize latest_observation = arg_max(ingestion_time(), *) by Neighbourhood
```

3. Click **Done** to save.

> The `isempty()` check ensures that when **"Select all"** is chosen (which sends an empty string), no filter is applied — all neighbourhoods show. When specific neighbourhoods are selected, only those appear.

Test: Use the **Neighbourhood** dropdown on the dashboard to filter — then click **Reset** to clear filters.

---

## Step 6 — Add Page 2

1. In the **Pages** pane on the left, click **+ Add page**.
2. Name the new page `Page 2` → Select it.
3. Click **+ Add tile** and enter:

```kql
base_bike_data
| project Neighbourhood, latest_observation
| order by latest_observation desc
```

4. Click **Apply changes** → Resize the tile to fill the full dashboard height.

> Page 2 shows each neighbourhood and the timestamp of its most recent data observation, sorted with newest first.

---

## Step 7 — Configure Auto Refresh

1. On the dashboard toolbar → **Manage** tab → click **Auto refresh**.
2. Configure:

   | Setting | Value |
   |---------|-------|
   | Enabled | Selected ✅ |
   | Minimum time interval | Allow all refresh intervals |
   | Default refresh rate | 30 minutes |

3. Click **Apply**.

---

## Step 8 — Save and Share

1. Click **Save** in the toolbar.
2. Click **Share** → **Copy link** → paste in a new browser tab to test the shared view.

---

## KQL Reference — All Queries Used

| Query | Used in | Purpose |
|-------|---------|---------|
| `bikes \| where ingestion_time() between (ago(30min) .. now()) \| summarize latest_observation = arg_max(...) by Neighbourhood \| project ..., No_Bikes, No_Empty_Docks \| order by Neighbourhood asc` | Tile 1 (original) | Bikes and docks per neighbourhood |
| `bikes \| where ... \| summarize ... \| project ..., Latitude, Longitude, No_Bikes \| order by Neighbourhood asc` | Tile 2 (original) | Map locations |
| `bikes \| where ingestion_time() between (ago(30min) .. now()) \| summarize latest_observation = arg_max(...) by Neighbourhood` | `base_bike_data` base query | Shared base — used by both tiles |
| `base_bike_data \| project Neighbourhood, latest_observation, No_Bikes, No_Empty_Docks \| order by Neighbourhood asc` | Tile 1 (updated) | Bar chart using base query |
| `base_bike_data \| project Neighbourhood, latest_observation, No_Bikes, Latitude, Longitude \| order by Neighbourhood asc` | Tile 2 (updated) | Map using base query |
| `bikes \| distinct Neighbourhood \| order by Neighbourhood asc` | Parameter | Populates Neighbourhood dropdown |
| `base_bike_data \| project Neighbourhood, latest_observation \| order by latest_observation desc` | Page 2 tile | Latest observations per neighbourhood |

---

## Difference from Previous Dashboard Projects

| Feature | Project 04 | Project 05 | Project 09 |
|---------|-----------|-----------|-----------|
| Dashboard tiles | 1 (column chart) | 1 (table, no chart) | 2 tiles (bar chart + map) |
| Visual types | Column chart | Table | Stacked bar + Map |
| Base query | ✗ | ✗ | ✅ `base_bike_data` |
| Parameters | ✗ | ✗ | ✅ Neighbourhood multi-select |
| Multiple pages | ✗ | ✗ | ✅ Page 1 + Page 2 |
| Auto refresh | ✗ | ✗ | ✅ 30 minutes |
| Stream transform | None | GroupByStreet (5s tumbling) | None (raw to table) |

---

## Summary

| Step | Action | Tool |
|------|--------|------|
| 1 | Create `RTDashboard` Eventhouse and KQL database | Fabric Real-Time Intelligence |
| 2 | Create `Bicycle-data` Eventstream → `bikes` table | Eventstream |
| 3a | Connect `Bike Rental Data` source to dashboard | Real-Time Dashboard |
| 3b | Add Tile 1 — stacked bar chart (Bikes and Docks) | KQL + Visual formatting |
| 3c | Add Tile 2 — map (Bike Locations) | KQL + Visual formatting |
| 4 | Create `base_bike_data` base query; update both tiles | Base queries |
| 5 | Add Neighbourhood multi-select parameter; update base query | Dashboard parameters |
| 6 | Add Page 2 with latest observations tile | Dashboard pages |
| 7 | Enable auto-refresh every 30 minutes | Auto refresh settings |
| 8 | Save and share dashboard | Dashboard Share |

---

## Notes

- **`arg_max(ingestion_time(), *)`** returns the entire row corresponding to the most recent ingestion time per group — this is the standard KQL pattern for getting the latest snapshot per category.
- **`isempty(['selected_neighbourhoods'])`** is the key pattern when a "Select all" parameter sends an empty string — without this check, the filter would return zero results when nothing is explicitly selected.
- The **base query** variable name (`base_bike_data`) acts like a virtual table name — you reference it just like a table name in subsequent tile queries.
- **Map tile** uses Latitude/Longitude columns directly from the `bikes` table — no geocoding step is needed since the sample data already includes coordinates.
- The `bikes-dashboard` has a **Neighbourhood: All** filter visible at the top in the screenshot — this is the parameter created in Step 5 in its default (Select all) state.
- Auto-refresh only activates when the dashboard is in **Viewing** mode — it does not refresh while in Editing mode.

---

*Documented by: Sagar Parab | Workspace: Project 09_RealTime Dasboard_Fabric | Last updated: March 2026*
