# Project 02 — Ingest Data with a Pipeline in Microsoft Fabric

**Workspace:** `Project 02_Ingest data_pipeline_Microsoft Fabric`
**Lakehouse:** `pipeline_LH_notebook`
**Pipeline:** `Ingest Sales Data`
**Notebook:** `Load Sales`
**Dataset:** Sales CSV (hosted on GitHub via HTTP)
**Reference:** [MS Learn Lab — Ingest data with a pipeline in Microsoft Fabric](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/04-ingest-pipeline.html)

---

## Overview

This project covers how to build a **data ingestion pipeline** in Microsoft Fabric that:
1. Copies raw sales data from an external HTTP source into the Lakehouse Files area
2. Uses a **Spark Notebook** to transform and load the data into a Delta table
3. Combines both steps into a single reusable **pipeline** with proper sequencing (Delete → Copy → Transform)

By the end of this exercise, you should understand how to:
- Create and configure a **Copy Data** pipeline activity
- Write a PySpark notebook to transform CSV data and save it as a Delta table
- Chain pipeline activities (Delete Data → Copy Data → Notebook) into an end-to-end ETL flow
- Pass parameters from a pipeline into a notebook

---

## Pre-requisites

- Active Microsoft Fabric trial
- Access to workspace: `Project 02_Ingest data_pipeline_Microsoft Fabric`
- Lakehouse `pipeline_LH_notebook` already created under this workspace

---

## Workspace Items Overview

Once everything is set up, your workspace should contain the following items:

| Name | Type | Purpose |
|------|------|---------|
| `pipeline_LH_notebook` | Lakehouse | Stores raw files and Delta tables |
| `pipeline_LH_notebook` | SQL analytics endpoint | Auto-created with the Lakehouse |
| `Ingest Sales Data` | Pipeline | End-to-end ETL pipeline |
| `Load Sales` | Notebook | Transforms CSV and loads to Delta table |
| `CopyJob_1` | Copy job | Created as part of the copy activity setup |

---

## Step-by-Step Setup

---

### Step 1 — Create the Lakehouse and Subfolder

1. Inside the workspace, click **New item** → Under *Data Engineering*, select **Lakehouse**.
2. Name it `pipeline_LH_notebook`.
   > Make sure the **"Lakehouse schemas (Public Preview)"** option is **disabled** during creation.
3. Once the Lakehouse opens, in the **Explorer** pane on the left, find the **Files** node.
4. Click the **…** menu next to **Files** → Select **New subfolder** → Name it `new_data`.

This `new_data` folder will be the landing zone for the raw CSV file copied by the pipeline.

---

### Step 2 — Create the Pipeline (Ingest Sales Data)

1. On the Lakehouse **Home** page, click **Get data** → **New data pipeline**.
2. Name the pipeline `Ingest Sales Data`.
3. If the **Copy Data** wizard does not open automatically, click **Copy Data → Use copy assistant** from the pipeline editor.

---

#### 2a — Configure the Data Source (HTTP)

In the **Copy Data** wizard:

- **Choose data source:** Search for `HTTP` and select it under *New sources*
- Fill in the connection details:

| Setting | Value |
|---------|-------|
| URL | `https://raw.githubusercontent.com/MicrosoftLearning/dp-data/main/sales.csv` |
| Connection | Create new connection |
| Connection name | Any unique name |
| Data gateway | (none) |
| Authentication kind | Anonymous |

- Click **Next**, then verify these settings:

| Setting | Value |
|---------|-------|
| Relative URL | *(leave blank)* |
| Request method | GET |
| Binary copy | Unselected |

- Click **Next** again and confirm file format settings:

| Setting | Value |
|---------|-------|
| File format | DelimitedText |
| Column delimiter | Comma (,) |
| Row delimiter | Line feed (\n) |
| First row as header | Selected |
| Compression type | None |

- Click **Preview data** to check a sample, then click **Next**.

---

#### 2b — Configure the Data Destination (Lakehouse Files)

| Setting | Value |
|---------|-------|
| Root folder | Files |
| Folder path | `new_data` |
| File name | `sales.csv` |
| Copy behavior | None |
| File format | DelimitedText |
| Column delimiter | Comma (,) |
| Row delimiter | Line feed (\n) |
| Add header to file | Selected |

- Click **Next** → Review the **Copy summary** → Click **Save + Run**.

The pipeline will run and copy `sales.csv` into `Files/new_data/` inside the Lakehouse.

> **Tip:** Monitor the run status in the **Output** pane. Use the **↻ Refresh** icon to check progress. Wait for a **Succeeded** status before moving on.

- Navigate back to the Lakehouse → **Files → new_data** to confirm `sales.csv` is there.

---

### Step 3 — Create the Load Sales Notebook

1. On the Lakehouse **Home** page, click **Open notebook** → **New notebook**.
2. In the first cell (default code cell), **replace** the existing code with:

```python
table_name = "sales"
```

3. Click the **…** menu on the top-right of this cell → Select **Toggle parameter cell**.
   > This marks the cell as a *parameters cell*, so its variables can be overridden when the notebook is triggered from a pipeline.

4. Click **+ Code** below the parameters cell to add a new code cell. Paste the following:

```python
from pyspark.sql.functions import *

# Read the new sales data from CSV
df = spark.read.format("csv").option("header", "true").load("Files/new_data/*.csv")

# Add Year and Month columns derived from OrderDate
df = df.withColumn("Year", year(col("OrderDate"))).withColumn("Month", month(col("OrderDate")))

# Split CustomerName into FirstName and LastName
df = df.withColumn("FirstName", split(col("CustomerName"), " ").getItem(0)) \
       .withColumn("LastName", split(col("CustomerName"), " ").getItem(1))

# Select and reorder the required columns
df = df["SalesOrderNumber", "SalesOrderLineNumber", "OrderDate", "Year", "Month",
        "FirstName", "LastName", "EmailAddress", "Item", "Quantity", "UnitPrice", "TaxAmount"]

# Write to Delta table (append if table already exists)
df.write.format("delta").mode("append").saveAsTable(table_name)
```

5. Click **▷ Run all** to execute the notebook.
   > First run may take 1–2 minutes for the Spark pool to start. This is expected.

6. After the run completes, go to the **Explorer** pane → Click **…** next to **Tables** → Select **Refresh**.
   You should now see a `sales` table listed under Tables.

7. Click the ⚙️ **Settings** icon in the notebook toolbar → Rename the notebook to `Load Sales` → Close the settings pane.

---

### Step 4 — Update the Pipeline to Include Notebook Activity

Now we enhance the `Ingest Sales Data` pipeline to include a **Delete Data** step before copying, and a **Notebook** step after copying. This makes the pipeline fully automated and reusable.

#### 4a — Add Delete Data Activity

1. Open the `Ingest Sales Data` pipeline.
2. On the **Activities** tab → Select **Delete data** from the *All activities* list.
3. Position the **Delete data** activity to the **left** of the existing **Copy data** activity.
4. Connect the **On Completion** output of **Delete data** → to the **Copy data** activity.
5. Configure the **Delete data** activity properties:

| Section | Setting | Value |
|---------|---------|-------|
| General | Name | `Delete old files` |
| Source | Connection | Your Lakehouse (`pipeline_LH_notebook`) |
| Source | File path type | Wildcard file path |
| Source | Folder path | `Files / new_data` |
| Source | Wildcard file name | `*.csv` |
| Source | Recursively | Selected |
| Logging settings | Enable logging | Unselected |

> This step clears out any old CSV files before the new one is copied in — preventing duplicate data.

---

#### 4b — Add Notebook Activity

1. On the **Activities** tab → Select **Notebook**.
2. Connect the **On Completion** output of **Copy data** → to the **Notebook** activity.
3. Configure the **Notebook** activity properties:

| Section | Setting | Value |
|---------|---------|-------|
| General | Name | `Load Sales notebook` |
| Settings | Notebook | `Load Sales` |
| Settings | Base parameters | Add parameter: `table_name` / String / `new_sales` |

> The `table_name` parameter overrides the default `"sales"` value in the notebook. This means the pipeline will write the data into a table called `new_sales` instead.

---

#### 4c — Save and Run the Full Pipeline

1. Click the 💾 **Save** icon on the **Home** tab.
2. Click **▷ Run** to run the full pipeline.
3. Wait for all three activities to show **Succeeded**:
   - Delete old files ✓
   - Copy data ✓
   - Load Sales notebook ✓

> **Note:** If you get an error like *"Spark SQL queries are only possible in the context of a lakehouse"*, open the `Load Sales` notebook → Remove attached lakehouses → Re-attach `pipeline_LH_notebook` → Go back to the pipeline and re-run.

4. Once done, navigate to the Lakehouse → **Explorer → Tables**.
5. You should now see a `new_sales` table with 12 columns and 1000 rows (as visible in the Lakehouse view).

---

## Final Lakehouse Table — `sales` / `new_sales`

Both tables contain the same structure (12 columns):

| Column | Description |
|--------|-------------|
| SalesOrderNumber | Order ID (e.g., SO43701) |
| SalesOrderLineNumber | Line item number |
| OrderDate | Date of order |
| Year | Derived from OrderDate |
| Month | Derived from OrderDate |
| FirstName | Derived from CustomerName |
| LastName | Derived from CustomerName |
| EmailAddress | Customer email |
| Item | Product name |
| Quantity | Units ordered |
| UnitPrice | Price per unit |
| TaxAmount | Tax applied |

---

## Pipeline Flow Summary

```
[Delete old files]  →  [Copy data (HTTP → Files/new_data/sales.csv)]  →  [Load Sales notebook (CSV → Delta table)]
```

---

## Summary

| Step | Action | Tool Used |
|------|--------|-----------|
| 1 | Create Lakehouse and `new_data` subfolder | Fabric Lakehouse |
| 2 | Copy sales CSV from HTTP source to Lakehouse | Copy Data activity (Pipeline) |
| 3 | Transform data and save as Delta table | PySpark Notebook |
| 4 | Add Delete step to clean old files | Delete Data activity (Pipeline) |
| 5 | Chain all steps into a single automated pipeline | Fabric Pipeline |

---

## Notes

- The `sales` table is created when you run the notebook manually. The `new_sales` table is created when the pipeline runs the notebook with the overridden `table_name` parameter.
- The **Delete Data** activity ensures the pipeline is **idempotent** — safe to run multiple times without duplicating raw files.
- The notebook uses `mode("append")` when writing to the Delta table — if you run it multiple times, rows will accumulate. Consider using `mode("overwrite")` if you want to replace data each time.
- All items are stored under the `Project 02_Ingest data_pipeline_Microsoft Fabric` workspace.

---

*Documented by: Sagar Parab | Workspace: Project 02_Ingest data_pipeline_Microsoft Fabric | Last updated: March 2026*
