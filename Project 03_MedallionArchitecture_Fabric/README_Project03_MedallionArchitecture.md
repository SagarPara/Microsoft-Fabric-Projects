# Project 03 — Medallion Architecture in Microsoft Fabric Lakehouse

**Workspace:** `Project 03_MedallionArchitecture_Fabric`
**Lakehouse:** `Sales`
**Notebooks:** `Transform data for Silver` · `Transform data for Gold`
**Semantic Model:** `Sales_Gold`
**Dataset:** Sales Orders CSV (2019, 2020, 2021 — Adventure Works)
**Reference:** [MS Learn Lab — Create a medallion architecture in a Microsoft Fabric lakehouse](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/03b-medallion-lakehouse.html)

---

## Overview

This project builds a **Medallion Architecture** (Bronze → Silver → Gold) inside a Microsoft Fabric Lakehouse using PySpark Notebooks. The idea behind this pattern is to progressively clean, enrich, and structure data as it moves through the layers — making it analysis-ready by the time it reaches the Gold layer.

| Layer | Location in Lakehouse | What it holds |
|-------|-----------------------|---------------|
| **Bronze** | `Files/bronze/` | Raw CSV files as-is — no transformation |
| **Silver** | `Tables/sales_silver` | Cleaned, validated Delta table |
| **Gold** | `Tables/dim*` and `Tables/factsales_gold` | Star schema tables ready for reporting |

By the end of this exercise you should understand:
- How to upload raw files to the bronze layer of a Lakehouse
- How to use PySpark to clean, validate, and load data into a Silver Delta table
- How to build a star schema (dimensions + fact table) in the Gold layer
- How to use Delta Lake's **upsert/merge** operation to keep tables current
- How to create a Semantic Model on top of the Gold layer

---

## Pre-requisites

- Active Microsoft Fabric trial
- Access to workspace: `Project 03_MedallionArchitecture_Fabric`
- Lakehouse `Sales` already created under this workspace
- Download the source files from:
  `https://github.com/MicrosoftLearning/dp-data/blob/main/orders.zip`
  Extract the zip — you should get **3 CSV files**: `2019.csv`, `2020.csv`, `2021.csv`

---

## Workspace Items Overview

| Name | Type | Purpose |
|------|------|---------|
| `Sales` | Lakehouse | Main data store — contains all 3 layers |
| `Sales` | SQL analytics endpoint | Auto-created; used for SQL exploration |
| `Transform data for Silver` | Notebook | Bronze → Silver transformation |
| `Transform data for Gold` | Notebook | Silver → Gold star schema transformation |
| `Sales_Gold` | Semantic model | Connects Gold tables for reporting |

---

## Architecture Flow

```
[Raw CSVs on local machine]
        ↓  Upload manually
[Files/bronze/]           ← BRONZE LAYER (raw files)
        ↓  Transform data for Silver notebook
[Tables/sales_silver]     ← SILVER LAYER (cleaned Delta table)
        ↓  Transform data for Gold notebook
[Tables/dimdate_gold]
[Tables/dimcustomer_gold] ← GOLD LAYER (star schema)
[Tables/dimproduct_gold]
[Tables/factsales_gold]
        ↓
[Sales_Gold Semantic Model] ← Reporting layer
```

---

## Step-by-Step Setup

---

### Step 1 — Create the Lakehouse and Upload Bronze Files

1. Inside the workspace, click **New item** → Under *Data Engineering*, select **Lakehouse**.
2. Name it `Sales`.
   > Make sure **"Lakehouse schemas (Public Preview)"** is **disabled** during creation.
3. In the **Explorer** pane, click **…** next to the **Files** folder → **New subfolder** → Name it `bronze`.
4. Click **…** next to the `bronze` folder → **Upload** → **Upload files**.
5. Upload all three files: `2019.csv`, `2020.csv`, `2021.csv` (use Shift to select all at once).
6. Click the `bronze` folder to confirm all 3 files are listed.

> These raw files form the **Bronze layer** — untouched source data, kept as-is for traceability.

---

### Step 2 — Transform Bronze to Silver (`Transform data for Silver` Notebook)

1. On the Lakehouse Home page, click **Open notebook** → **New notebook**.
2. Rename the notebook to `Transform data for Silver` (click the notebook title at top-left to rename).

---

#### Cell 1 — Read CSVs from Bronze with Schema

```python
from pyspark.sql.types import *

# Define the schema explicitly (no header in source files)
orderSchema = StructType([
    StructField("SalesOrderNumber", StringType()),
    StructField("SalesOrderLineNumber", IntegerType()),
    StructField("OrderDate", DateType()),
    StructField("CustomerName", StringType()),
    StructField("Email", StringType()),
    StructField("Item", StringType()),
    StructField("Quantity", IntegerType()),
    StructField("UnitPrice", FloatType()),
    StructField("Tax", FloatType())
])

# Load all CSV files from the bronze folder
df = spark.read.format("csv").option("header", "false").schema(orderSchema).load("Files/bronze/*.csv")

display(df.head(10))
```

> **Note:** First run may take 1–2 minutes for Spark pool to start.

---

#### Cell 2 — Add Audit Columns and Handle Nulls

```python
from pyspark.sql.functions import when, lit, col, current_timestamp, input_file_name

# Add source file name, a flag for orders before Aug 2019, and timestamps
df = df.withColumn("FileName", input_file_name()) \
    .withColumn("IsFlagged", when(col("OrderDate") < '2019-08-01', True).otherwise(False)) \
    .withColumn("CreatedTS", current_timestamp()) \
    .withColumn("ModifiedTS", current_timestamp())

# Replace null or empty CustomerName with "Unknown"
df = df.withColumn("CustomerName", when(
    (col("CustomerName").isNull() | (col("CustomerName") == "")), lit("Unknown")
).otherwise(col("CustomerName")))
```

---

#### Cell 3 — Create the `sales_silver` Delta Table Schema

```python
from pyspark.sql.types import *
from delta.tables import *

DeltaTable.createIfNotExists(spark) \
    .tableName("sales.sales_silver") \
    .addColumn("SalesOrderNumber", StringType()) \
    .addColumn("SalesOrderLineNumber", IntegerType()) \
    .addColumn("OrderDate", DateType()) \
    .addColumn("CustomerName", StringType()) \
    .addColumn("Email", StringType()) \
    .addColumn("Item", StringType()) \
    .addColumn("Quantity", IntegerType()) \
    .addColumn("UnitPrice", FloatType()) \
    .addColumn("Tax", FloatType()) \
    .addColumn("FileName", StringType()) \
    .addColumn("IsFlagged", BooleanType()) \
    .addColumn("CreatedTS", DateType()) \
    .addColumn("ModifiedTS", DateType()) \
    .execute()
```

After running, go to **Explorer → Tables → Refresh**. You should see `sales_silver` listed with a ▲ Delta icon.

---

#### Cell 4 — Upsert Data into `sales_silver`

```python
from delta.tables import *

deltaTable = DeltaTable.forPath(spark, 'Tables/sales_silver')
dfUpdates = df

deltaTable.alias('silver') \
  .merge(
    dfUpdates.alias('updates'),
    'silver.SalesOrderNumber = updates.SalesOrderNumber AND \
     silver.OrderDate = updates.OrderDate AND \
     silver.CustomerName = updates.CustomerName AND \
     silver.Item = updates.Item'
  ) \
  .whenMatchedUpdate(set={}) \
  .whenNotMatchedInsert(values={
      "SalesOrderNumber": "updates.SalesOrderNumber",
      "SalesOrderLineNumber": "updates.SalesOrderLineNumber",
      "OrderDate": "updates.OrderDate",
      "CustomerName": "updates.CustomerName",
      "Email": "updates.Email",
      "Item": "updates.Item",
      "Quantity": "updates.Quantity",
      "UnitPrice": "updates.UnitPrice",
      "Tax": "updates.Tax",
      "FileName": "updates.FileName",
      "IsFlagged": "updates.IsFlagged",
      "CreatedTS": "updates.CreatedTS",
      "ModifiedTS": "updates.ModifiedTS"
  }) \
  .execute()
```

> The **merge/upsert** ensures no duplicate rows are inserted if the notebook is re-run.

Once done, click **Stop session** from the notebook Run menu to free up compute.

---

### Step 3 — Explore Silver Layer via SQL Endpoint (Optional)

1. From the workspace, open the **Sales SQL analytics endpoint**.
2. Click **New SQL query** and run the following to check total sales by year:

```sql
SELECT YEAR(OrderDate) AS Year,
       CAST(SUM(Quantity * (UnitPrice + Tax)) AS DECIMAL(12, 2)) AS TotalSales
FROM sales_silver
GROUP BY YEAR(OrderDate)
ORDER BY YEAR(OrderDate)
```

3. Run this to see the top 10 customers by quantity:

```sql
SELECT TOP 10 CustomerName, SUM(Quantity) AS TotalQuantity
FROM sales_silver
GROUP BY CustomerName
ORDER BY TotalQuantity DESC
```

> This is a quick sanity check on the Silver data before building the Gold layer.

---

### Step 4 — Transform Silver to Gold (`Transform data for Gold` Notebook)

1. Return to the workspace home → Click **New item** → **Notebook**.
2. Rename it to `Transform data for Gold`.
3. In the **Explorer** pane, click **Add data items** → Select the `Sales` lakehouse.
   You should see `sales_silver` listed under Tables.

---

#### Cell 1 — Load Silver Data

```python
df = spark.read.table("Sales.sales_silver")
```

> If you get a `[TooManyRequestsForCapacity]` error, make sure the Silver notebook session is stopped first.

---

#### Cell 2 — Create `dimdate_gold` Schema

```python
from pyspark.sql.types import *
from delta.tables import *

DeltaTable.createIfNotExists(spark) \
    .tableName("sales.dimdate_gold") \
    .addColumn("OrderDate", DateType()) \
    .addColumn("Day", IntegerType()) \
    .addColumn("Month", IntegerType()) \
    .addColumn("Year", IntegerType()) \
    .addColumn("mmmyyyy", StringType()) \
    .addColumn("yyyymm", StringType()) \
    .execute()
```

---

#### Cell 3 — Build and Upsert `dimdate_gold`

```python
from pyspark.sql.functions import col, dayofmonth, month, year, date_format

dfdimDate_gold = df.dropDuplicates(["OrderDate"]).select(
    col("OrderDate"),
    dayofmonth("OrderDate").alias("Day"),
    month("OrderDate").alias("Month"),
    year("OrderDate").alias("Year"),
    date_format(col("OrderDate"), "MMM-yyyy").alias("mmmyyyy"),
    date_format(col("OrderDate"), "yyyyMM").alias("yyyymm")
).orderBy("OrderDate")

display(dfdimDate_gold.head(10))
```

```python
from delta.tables import *

deltaTable = DeltaTable.forPath(spark, 'Tables/dimdate_gold')

deltaTable.alias('gold') \
  .merge(dfdimDate_gold.alias('updates'), 'gold.OrderDate = updates.OrderDate') \
  .whenMatchedUpdate(set={}) \
  .whenNotMatchedInsert(values={
      "OrderDate": "updates.OrderDate",
      "Day": "updates.Day",
      "Month": "updates.Month",
      "Year": "updates.Year",
      "mmmyyyy": "updates.mmmyyyy",
      "yyyymm": "updates.yyyymm"
  }) \
  .execute()
```

---

#### Cell 4 — Build `dimcustomer_gold`

**Create schema:**
```python
from pyspark.sql.types import *
from delta.tables import *

DeltaTable.createIfNotExists(spark) \
    .tableName("sales.dimcustomer_gold") \
    .addColumn("CustomerName", StringType()) \
    .addColumn("Email", StringType()) \
    .addColumn("First", StringType()) \
    .addColumn("Last", StringType()) \
    .addColumn("CustomerID", LongType()) \
    .execute()
```

**Prepare customer data (split name, deduplicate):**
```python
from pyspark.sql.functions import col, split

dfdimCustomer_silver = df.dropDuplicates(["CustomerName", "Email"]).select(
    col("CustomerName"), col("Email")
).withColumn("First", split(col("CustomerName"), " ").getItem(0)) \
 .withColumn("Last", split(col("CustomerName"), " ").getItem(1))

display(dfdimCustomer_silver.head(10))
```

**Generate new CustomerIDs (avoid collision with existing):**
```python
from pyspark.sql.functions import monotonically_increasing_id, col, coalesce, max, lit

dfdimCustomer_temp = spark.read.table("Sales.dimCustomer_gold")
MAXCustomerID = dfdimCustomer_temp.select(
    coalesce(max(col("CustomerID")), lit(0)).alias("MAXCustomerID")
).first()[0]

# Only get customers not already in the Gold table
dfdimCustomer_gold = dfdimCustomer_silver.join(
    dfdimCustomer_temp,
    (dfdimCustomer_silver.CustomerName == dfdimCustomer_temp.CustomerName) &
    (dfdimCustomer_silver.Email == dfdimCustomer_temp.Email),
    "left_anti"
)

dfdimCustomer_gold = dfdimCustomer_gold.withColumn(
    "CustomerID", monotonically_increasing_id() + MAXCustomerID + 1
)

display(dfdimCustomer_gold.head(10))
```

**Upsert into `dimcustomer_gold`:**
```python
from delta.tables import *

deltaTable = DeltaTable.forPath(spark, 'Tables/dimcustomer_gold')

deltaTable.alias('gold') \
  .merge(
    dfdimCustomer_gold.alias('updates'),
    'gold.CustomerName = updates.CustomerName AND gold.Email = updates.Email'
  ) \
  .whenMatchedUpdate(set={}) \
  .whenNotMatchedInsert(values={
      "CustomerName": "updates.CustomerName",
      "Email": "updates.Email",
      "First": "updates.First",
      "Last": "updates.Last",
      "CustomerID": "updates.CustomerID"
  }) \
  .execute()
```

---

#### Cell 5 — Build `dimproduct_gold`

**Create schema:**
```python
from pyspark.sql.types import *
from delta.tables import *

DeltaTable.createIfNotExists(spark) \
    .tableName("sales.dimproduct_gold") \
    .addColumn("ItemName", StringType()) \
    .addColumn("ItemID", LongType()) \
    .addColumn("ItemInfo", StringType()) \
    .execute()
```

**Prepare product data:**
```python
from pyspark.sql.functions import col, split, lit, when

dfdimProduct_silver = df.dropDuplicates(["Item"]).select(col("Item")) \
    .withColumn("ItemName", split(col("Item"), ", ").getItem(0)) \
    .withColumn("ItemInfo", when(
        (split(col("Item"), ", ").getItem(1).isNull() | (split(col("Item"), ", ").getItem(1) == "")),
        lit("")
    ).otherwise(split(col("Item"), ", ").getItem(1)))

display(dfdimProduct_silver.head(10))
```

**Generate ItemIDs and upsert** (same pattern as customer — read existing max ID, left anti join, assign new IDs, then merge into Gold table).

---

#### Cell 6 — Build `factsales_gold`

**Create schema:**
```python
from pyspark.sql.types import *
from delta.tables import *

DeltaTable.createIfNotExists(spark) \
    .tableName("sales.factsales_gold") \
    .addColumn("CustomerID", LongType()) \
    .addColumn("ItemID", LongType()) \
    .addColumn("OrderDate", DateType()) \
    .addColumn("Quantity", IntegerType()) \
    .addColumn("UnitPrice", FloatType()) \
    .addColumn("Tax", FloatType()) \
    .execute()
```

**Join Silver with Dim tables to create the fact:**
```python
from pyspark.sql.functions import col, split, when, lit

dfdimCustomer_temp = spark.read.table("Sales.dimCustomer_gold")
dfdimProduct_temp = spark.read.table("Sales.dimProduct_gold")

df = df.withColumn("ItemName", split(col("Item"), ", ").getItem(0)) \
       .withColumn("ItemInfo", when(
           (split(col("Item"), ", ").getItem(1).isNull() | (split(col("Item"), ", ").getItem(1) == "")),
           lit("")
       ).otherwise(split(col("Item"), ", ").getItem(1)))

dffactSales_gold = df.alias("df1") \
    .join(dfdimCustomer_temp.alias("df2"),
          (df.CustomerName == dfdimCustomer_temp.CustomerName) &
          (df.Email == dfdimCustomer_temp.Email), "left") \
    .join(dfdimProduct_temp.alias("df3"),
          (df.ItemName == dfdimProduct_temp.ItemName) &
          (df.ItemInfo == dfdimProduct_temp.ItemInfo), "left") \
    .select(
        col("df2.CustomerID"),
        col("df3.ItemID"),
        col("df1.OrderDate"),
        col("df1.Quantity"),
        col("df1.UnitPrice"),
        col("df1.Tax")
    ).orderBy("df1.OrderDate", "df2.CustomerID", "df3.ItemID")

display(dffactSales_gold.head(10))
```

**Upsert into `factsales_gold`:**
```python
from delta.tables import *

deltaTable = DeltaTable.forPath(spark, 'Tables/factsales_gold')

deltaTable.alias('gold') \
  .merge(
    dffactSales_gold.alias('updates'),
    'gold.OrderDate = updates.OrderDate AND gold.CustomerID = updates.CustomerID AND gold.ItemID = updates.ItemID'
  ) \
  .whenMatchedUpdate(set={}) \
  .whenNotMatchedInsert(values={
      "CustomerID": "updates.CustomerID",
      "ItemID": "updates.ItemID",
      "OrderDate": "updates.OrderDate",
      "Quantity": "updates.Quantity",
      "UnitPrice": "updates.UnitPrice",
      "Tax": "updates.Tax"
  }) \
  .execute()
```

---

### Step 5 — Create the Semantic Model (Optional)

> **Note:** Requires a Power BI license in addition to Fabric trial.

1. Navigate to the `Sales` Lakehouse in your workspace.
2. Click **New semantic model** from the ribbon.
3. Name it `Sales_Gold`.
4. Select the following Gold tables and click **Confirm**:
   - `dimdate_gold`
   - `dimcustomer_gold`
   - `dimproduct_gold`
   - `factsales_gold`

The semantic model will open in Fabric where you can define relationships between tables and create measures for Power BI reporting.

---

## Gold Layer Tables — Final Structure

| Table | Type | Key Columns |
|-------|------|-------------|
| `dimcustomer_gold` | Dimension | CustomerID, CustomerName, Email, First, Last |
| `dimdate_gold` | Dimension | OrderDate, Day, Month, Year, mmmyyyy, yyyymm |
| `dimproduct_gold` | Dimension | ItemID, ItemName, ItemInfo |
| `factsales_gold` | Fact | CustomerID, ItemID, OrderDate, Quantity, UnitPrice, Tax |
| `sales_silver` | Silver (intermediate) | All raw order columns + audit columns |

---

## Summary

| Step | Action | Layer | Tool |
|------|--------|-------|------|
| 1 | Upload 2019–2021 CSV files | Bronze | Manual upload to Lakehouse Files |
| 2 | Read CSVs, define schema, add audit columns | Bronze → Silver | PySpark Notebook |
| 3 | Create & upsert into `sales_silver` Delta table | Silver | Delta Lake merge |
| 4 | Explore Silver data with SQL queries | Silver | SQL analytics endpoint |
| 5 | Build date, customer, product dimension tables | Silver → Gold | PySpark Notebook |
| 6 | Build `factsales_gold` fact table | Gold | PySpark Notebook (joins) |
| 7 | Create Semantic Model on Gold tables | Gold | Fabric Semantic Model |

---

## Notes

- All Gold tables use **Delta Lake merge (upsert)** — safe to re-run notebooks without duplicating data.
- The **left anti join** pattern is used when building dimension tables to only insert customers/products that don't already exist in the Gold table.
- Two separate notebooks are used intentionally — `Transform data for Silver` and `Transform data for Gold` — to keep transformations modular and easier to debug.
- Always **Stop the Spark session** after each notebook run to avoid `TooManyRequestsForCapacity` errors in the next notebook.
- The `bronze` folder holds the original files untouched — this is by design so you always have a recovery point.

---

*Documented by: Sagar Parab | Workspace: Project 03_MedallionArchitecture_Fabric | Last updated: March 2026*
