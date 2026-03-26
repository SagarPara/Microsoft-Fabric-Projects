# Project 11 — Monitor Fabric Activity in the Monitoring Hub

**Workspace:** `Project 11_Activity in the Monitoring Hub`
**Lakehouse:** `LH_MonitoringHub`
**Dataflow Gen2:** `Get Product Data_3`
**Notebook:** `Query Products`
**Table created:** `products` (295 rows — bicycle/mountain bike products)
**Reference:** [MS Learn Lab — Monitor Fabric activity in the monitoring hub](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/18-monitor-hub.html)

---

## Overview

The **Monitoring Hub** in Microsoft Fabric is a central place to track and audit all activity across your workspaces — Dataflow runs, Notebook executions, Pipeline runs, Experiment runs, and more. It shows real-time status, historical run logs, and allows filtering and custom column configuration.

This project is deliberately lightweight in terms of data work — the goal is not complex analytics, but to **generate activity** using a Dataflow and a Notebook, then learn how to **observe, filter, and audit** that activity in the Monitoring Hub.

**What you build:**

```
[products.csv on GitHub]
        ↓
[Dataflow Gen2: Get Product Data]
        ↓
[LH_MonitoringHub → Table: products]   ← 295 rows, 4 columns
        ↓
[Notebook: Query Products]             ← loads products table via Spark
        ↓
[Monitor Hub]
  ├── View in-progress activities
  ├── View completed activities
  ├── Check historical runs for an item
  ├── Filter by status + item type
  └── Customise visible columns
```

---

## Pre-requisites

- Active Microsoft Fabric trial
- Access to workspace: `Project 11_Activity in the Monitoring Hub`

---

## Workspace Items Overview

| Name | Type | Purpose |
|------|------|---------|
| `LH_MonitoringHub` | Lakehouse | Stores the `products` table loaded by the dataflow |
| `LH_MonitoringHub` | SQL analytics endpoint | Auto-created with the Lakehouse |
| `Get Product Data_3` | Dataflow Gen2 | Ingests products.csv → loads to `products` table |
| `Query Products` | Notebook | Queries the `products` table using Spark |

> The `_3` suffix on `Get Product Data_3` indicates this dataflow was run (or re-created) multiple times during the exercise — each run creates a new entry in the Monitoring Hub history.

---

## Step 1 — Create the Lakehouse

1. From the left menu, click **Create** → Under *Data Engineering*, select **Lakehouse**.
2. Name it `LH_MonitoringHub`.
   > Ensure **"Lakehouse schemas (Public Preview)"** is disabled.
3. The lakehouse opens empty — no tables or files yet.

---

## Step 2 — Create and Run the Dataflow Gen2

A **Dataflow Gen2** uses a Power Query-based visual designer to connect to a data source and load data into a destination. Here we pull a products CSV from GitHub and load it into the Lakehouse as a Delta table.

1. On the Lakehouse Home page, click **Get data** → **New Dataflow Gen2**.
2. Name it `Get Product Data` → Click **Create**.
3. In the dataflow designer, click **Import from a Text/CSV file**.
4. In the Get Data wizard, enter the following URL and use **Anonymous** authentication:

   ```
   https://raw.githubusercontent.com/MicrosoftLearning/dp-data/main/products.csv
   ```

5. Complete the wizard — a preview of the data will appear in the designer.
6. Click **Publish** to save and run the dataflow.

---

### Watch it in the Monitor Hub

1. In the left navigation bar, click **Monitor**.
2. The monitoring hub opens — you should see your dataflow listed as **In Progress**.
3. Refresh the page every few seconds until the status changes to **Succeeded** ✅.

---

### Verify the Table Was Created

1. Go back to the `LH_MonitoringHub` Lakehouse.
2. Expand **Tables** in the Explorer pane (refresh if needed).
3. The `products` table should now be listed.
4. Click the `products` table to preview the data:

| Column | Type | Sample |
|--------|------|--------|
| ProductID | Number | 771, 772, 773... |
| ProductName | Text | Mountain-100 Silver, 38 |
| Category | Text | Mountain Bikes |
| ListPrice | Number | 3399.99, 3374.99... |

> The table has **295 rows** and **4 columns** — all Mountain Bikes and Road Bikes products from the Adventure Works dataset.

---

## Step 3 — Create and Run the Notebook

1. From the left menu, click **Create** → Under *Data Engineering*, select **Notebook**.
2. A new notebook opens as **Notebook 1**.
3. Click the notebook title at the top-left → Rename it to `Query Products`.
4. In the **Explorer** pane, click **Add data items** → **Existing data sources**.
5. Select and add the `LH_MonitoringHub` lakehouse.
6. Expand the lakehouse in Explorer until you reach the `products` table.
7. Click **…** next to `products` → **Load data** → **Spark**.

   A new code cell is added automatically:

   ```python
   df = spark.read.format("delta").load("Tables/products")
   display(df)
   ```

8. Click **▷ Run all** to execute all cells.
   > The Spark session will take ~1 minute to start on first run.
9. Review the output — the `products` table data is displayed below the cell.
10. Click **◻ Stop session** on the toolbar to end the Spark session.

---

### Watch it in the Monitor Hub

1. Click **Monitor** in the left navigation bar.
2. The notebook activity (`Query Products`) should appear at the top of the list with status **Succeeded**.

---

## Step 4 — View Historical Runs for the Dataflow

Items can be run multiple times. The Monitor Hub keeps a full history for each item.

1. Return to your workspace.
2. Find `Get Product Data_3` in the workspace list — click the **↻ Refresh now** button to re-run it.
3. Navigate back to **Monitor** — the dataflow should show as **In Progress** again.
4. Once complete, click the **…** menu next to the `Get Product Data` entry in the Monitor Hub.
5. Select **Historical runs** — you'll see all past runs listed with timestamps, status, and duration.
6. Click **…** on any run → **View detail** to see the full run details.
7. Close the Details pane → Click **Back to main view**.

---

## Step 5 — Filter and Customise the Monitor Hub View

In a real environment with many workspaces and hundreds of activities, you need filters and column customisation to find what you're looking for quickly.

### Apply a Filter

1. Click **Filter** in the Monitor Hub toolbar.
2. Set the following:

   | Filter | Value |
   |--------|-------|
   | Status | Succeeded |
   | Item type | Dataflow Gen2 |

3. Click **Apply** — only successful Dataflow Gen2 runs are now listed.

---

### Customise Columns

1. Click **Column Options** in the toolbar.
2. Enable the following columns and click **Apply**:

   - Activity name
   - Status
   - Item type
   - Start time
   - Submitted by
   - Location
   - End time
   - Duration
   - Refresh type

> You may need to scroll horizontally to see all columns once added.

---

## What the Monitor Hub Shows Across Projects

Your Monitor Hub screenshot shows cross-workspace visibility — all workspaces you have access to appear here. As of this exercise, the Monitor Hub log includes:

| Activity | Type | Status | Time | Workspace |
|----------|------|--------|------|-----------|
| Query Products | Notebook | ✅ Succeeded | 03/26 2:40 PM | Project 11 |
| Get Product Data_3 | Dataflow Gen2 | ✅ Succeeded | 03/26 2:39 PM | Project 11 |
| Get Product Data | Dataflow Gen2 | ✅ Succeeded | 03/26 2:27 PM | Project 11 |
| experiment_diabetes_mango_horse_3cptn8t2 | Experiment | ✅ Succeeded | 03/24 2:22 PM | Project 06 |
| experiment_diabetes_gifted_eye_rhhcj28d | Experiment | ✅ Succeeded | 03/24 2:22 PM | Project 06 |
| experiment_diabetes_tidy_ticket_xtql3nbq | Experiment | ✅ Succeeded | 03/24 2:02 PM | Project 06 |
| experiment_diabetes_loyal_book_hjvll56m | Experiment | ✅ Succeeded | 03/24 2:01 PM | Project 06 |
| experiment_diabetes_gentle_town_cm0y8wqp | Experiment | ❌ Failed | 03/24 2:01 PM | Project 06 |
| experiment_diabetes_placid_tail_66vcr7gt | Experiment | ✅ Succeeded | 03/24 2:01 PM | Project 06 |
| experiment_diabetes_nice_cow_tg7zn0c6 | Experiment | ❌ Failed | 03/24 2:00 PM | Project 06 |
| experiment_diabetes_neat_cheese_5jltjs3q | Experiment | ✅ Succeeded | 03/24 1:58 PM | Project 06 |
| Train and Track with ML-Workflow_in_Notebook | Notebook | ✅ Succeeded | 03/24 1:44 PM | Project 06 |

> The two **Failed** experiment runs (`gentle_town` and `nice_cow`) from Project 06 are also visible here — the Monitor Hub shows failures across all workspaces, not just the current one. This is useful for troubleshooting past runs without having to navigate back to each project.

---

## Summary

| Step | Action | Tool |
|------|--------|------|
| 1 | Create `LH_MonitoringHub` Lakehouse | Fabric Data Engineering |
| 2 | Create `Get Product Data` Dataflow Gen2 — import products.csv | Dataflow Gen2 designer |
| 2b | Publish dataflow and watch in Monitor Hub (In Progress → Succeeded) | Monitor Hub |
| 2c | Verify `products` table created in Lakehouse (295 rows) | Lakehouse Explorer |
| 3 | Create `Query Products` notebook — load and display products table via Spark | Fabric Notebook |
| 3b | Run notebook and observe in Monitor Hub | Monitor Hub |
| 4 | Re-run dataflow → view Historical runs → View detail | Monitor Hub — Historical runs |
| 5a | Apply filter: Status = Succeeded, Item type = Dataflow Gen2 | Monitor Hub — Filter |
| 5b | Customise columns: Duration, End time, Submitted by, Refresh type, etc. | Monitor Hub — Column Options |

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Monitoring Hub** | Central view of all activity across all workspaces you have access to — not just the current one |
| **Historical runs** | Full log of past runs for a specific item — useful for tracking re-runs and failures |
| **Filter** | Narrow the Monitor Hub view by Status, Item type, Workspace, Date range, etc. |
| **Column Options** | Add/remove columns like Duration, End time, Submitted by to customise the activity view |
| **Dataflow Gen2** | A Power Query-based ETL tool for ingesting and transforming data into Lakehouse tables |
| **Cross-workspace visibility** | The Monitor Hub shows activities from all workspaces you have permissions on — not just the one you're currently in |

---

## Notes

- The Monitor Hub shows activities from **all workspaces you have permission to view** — this is why Project 06 MLflow experiment runs appear even though you're working in Project 11's workspace.
- **Failed runs are permanently recorded** — the `gentle_town` and `nice_cow` experiment failures from Project 06 are still visible here, providing a useful audit trail.
- **Dataflow Gen2 vs Pipeline:** Both can ingest data, but Dataflow Gen2 uses a Power Query visual designer (no code), while Pipelines (used in Project 02) use activity-based orchestration with more control over scheduling and error handling.
- The `products.csv` source file is the Adventure Works product catalogue — it contains 295 bicycle products across categories like Mountain Bikes, Road Bikes, etc. The `products` table in the Lakehouse is the Delta table version of this CSV.
- **Refresh type** column (added via Column Options) shows whether a run was **Scheduled**, **On-demand**, or triggered by another item — useful for understanding what caused a particular run.

---

*Documented by: Sagar Parab | Workspace: Project 11_Activity in the Monitoring Hub | Last updated: March 2026*
