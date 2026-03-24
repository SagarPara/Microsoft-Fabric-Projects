# Project 06 — Data Wrangler and MLflow in Microsoft Fabric

**Workspace:** `Project 06_DataWrangler and MLFlow in MicrosoftFabric`
**Lakehouse:** `LH_Notebook_for_DataWrangler_and_MLFlow`
**Notebooks:** `Pre-process_DataWrangler` · `Train and Track with ML-Workflow_in_Notebook`
**Experiment:** `experiment_diabetes`
**ML Model:** `model-diabetes` (Version 1)
**Datasets:** OJ Sales (Data Wrangler) · Diabetes (MLflow training)
**References:**
- [MS Learn Lab — Preprocess data with Data Wrangler in Microsoft Fabric](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/08b-data-science-preprocess-data-wrangler.html)
- [MS Learn Lab — Train and track machine learning models with MLflow in Microsoft Fabric](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/08c-data-science-train.html)

---

## Overview

This project combines **two labs** into a single workspace, covering the full pre-modelling and modelling workflow in Microsoft Fabric Data Science:

**Part A — Data Wrangler (`Pre-process_DataWrangler` notebook)**
Covers how to use Fabric's built-in **Data Wrangler** GUI tool to explore, clean, and transform the **OJ Sales dataset**. Data Wrangler auto-generates Python code for every operation you perform, which can be copied back into the notebook — great for reproducibility.

**Part B — MLflow Model Training (`Train and Track with ML-Workflow_in_Notebook` notebook)**
Covers how to train regression models on the **Diabetes dataset** and track every run (parameters, metrics, artifacts) using **MLflow**. Multiple model runs are compared, the best model is saved as `model-diabetes`, and the experiment is explored visually inside Fabric.

```
Part A:                               Part B:
OJ Sales dataset                      Diabetes dataset
        ↓                                     ↓
Load → Data Wrangler (GUI)            Load → Train/Test Split
        ↓                                     ↓
  - Clean text (Brand)                 LinearRegression (MLflow run)
  - One-hot encode                     DecisionTreeRegressor (MLflow run)
  - Filter by Store                              ↓
  - Sort by Price                       experiment_diabetes
  - Group by Brand (avg Revenue)                 ↓
        ↓                               Compare runs → Save best
  Generate & export clean_data()        model-diabetes (Version 1)
  function back to notebook
```

---

## Pre-requisites

- Active Microsoft Fabric trial
- Access to workspace: `Project 06_DataWrangler and MLFlow in MicrosoftFabric`
- Lakehouse `LH_Notebook_for_DataWrangler_and_MLFlow` already set up

---

## Workspace Items Overview

| Name | Type | Purpose |
|------|------|---------|
| `LH_Notebook_for_DataWrangler_and_MLFlow` | Lakehouse | Storage layer attached to both notebooks |
| `LH_Notebook_for_DataWrangler_and_MLFlow` | SQL analytics endpoint | Auto-created with the Lakehouse |
| `Pre-process_DataWrangler` | Notebook | Part A — OJ Sales data preprocessing using Data Wrangler |
| `Train and Track with ML-Workflow_in_Notebook` | Notebook | Part B — Diabetes model training with MLflow |
| `experiment_diabetes` | Experiment | Tracks all MLflow runs from Part B |
| `model-diabetes` | ML model | Saved best model from Part B (Version 1) |

---

---

# Part A — Preprocess Data with Data Wrangler

**Notebook:** `Pre-process_DataWrangler`
**Dataset:** OJ Sales (Azure Open Datasets — `ojsales-simulatedcontainer`)

---

## What is Data Wrangler?

**Data Wrangler** is a Fabric built-in GUI tool that opens directly inside a notebook. You interact with your dataframe visually — apply transformations, see previews instantly — and it auto-generates the equivalent Python/Pandas code for you. This is especially useful for:
- Quick exploratory cleaning without writing boilerplate code
- Learning what Pandas functions correspond to common operations
- Generating reusable, clean preprocessing functions

---

## Step A1 — Load OJ Sales Data

Create a new notebook named `Pre-process_DataWrangler`. Convert the first cell to Markdown and add a heading. Then in a new code cell:

```python
# Azure storage access info for OJ Sales dataset
blob_account_name = "azureopendatastorage"
blob_container_name = "ojsales-simulatedcontainer"
blob_relative_path = "oj_sales_data"
blob_sas_token = r""  # Anonymous access

wasbs_path = f"wasbs://%s@%s.blob.core.windows.net/%s" % (blob_container_name, blob_account_name, blob_relative_path)
spark.conf.set("fs.azure.sas.%s.%s.blob.core.windows.net" % (blob_container_name, blob_account_name), blob_sas_token)
print("Remote blob path: " + wasbs_path)

df = spark.read.csv(wasbs_path, header=True)
```

Convert to Pandas, sample 500 rows, and fix data types:

```python
import pandas as pd

df = df.toPandas()
df = df.sample(n=500, random_state=1)

df['WeekStarting'] = pd.to_datetime(df['WeekStarting'])
df['Quantity'] = df['Quantity'].astype('int')
df['Advert'] = df['Advert'].astype('int')
df['Price'] = df['Price'].astype('float')
df['Revenue'] = df['Revenue'].astype('float')

df = df.reset_index(drop=True)
df.head(4)
```

The dataset has columns: `WeekStarting`, `Store`, `Brand`, `Quantity`, `Advert`, `Price`, `Revenue`.

---

## Step A2 — Launch Data Wrangler

1. In the notebook ribbon, click **Data** → **Launch Data Wrangler** → select the `df` dataset.
2. Data Wrangler opens in a new tab showing:
   - A **data grid** (your dataframe)
   - A **Summary panel** on the right (stats per column)
   - An **Operations panel** on the left (all transformation options)
   - A **Cleaning steps** panel showing your applied steps

Click the **Revenue** column in the grid — the Summary panel shows mean ≈ **$33,459.54**, std ≈ **$8,032.23**.

---

## Step A3 — Clean the Brand Column (Find & Replace + Capitalize)

The `Brand` column has dots in values like `minute.maid`. We clean this in two steps:

1. In **Operations** → expand **Find and replace** → select **Find and replace**
   - Old value: `.` (dot)
   - New value: ` ` (space)
   - Click **Apply**

2. In **Operations** → expand **Format** → select **Capitalize first character**
   - Toggle **Capitalize all words** ON
   - Click **Apply**

3. Click **Add code to notebook** — Data Wrangler generates and pastes the code.

4. In the notebook cell, update lines 10-11 to call the function on `df`:

```python
def clean_data(df):
    # Replace all instances of "." with " " in column: 'Brand'
    df['Brand'] = df['Brand'].str.replace(".", " ", case=False, regex=False)
    # Capitalize the first character in column: 'Brand'
    df['Brand'] = df['Brand'].str.title()
    return df

df = clean_data(df)
```

Verify results:

```python
df['Brand'].unique()
# Expected: ['Minute Maid', 'Dominicks', 'Tropicana']
```

---

## Step A4 — One-Hot Encoding (No code export — for reference)

1. Re-launch Data Wrangler on `df`.
2. Select the `Brand` column.
3. **Operations** → **Formulas** → **One-hot encode** → **Apply**.

Data Wrangler adds three new columns: `Brand_Dominicks`, `Brand_Minute Maid`, `Brand_Tropicana`, and removes the original `Brand` column.

> Exit Data Wrangler **without** generating code for this step — it's shown here just to demonstrate the feature.

---

## Step A5 — Filter and Sort

1. Re-launch Data Wrangler on `df`.
2. **Operations** → **Sort and filter** → **Filter**:
   - Target column: `Store` | Operation: `Equal to` | Value: `1227` | Action: `Keep matching rows`
   - Click **Apply**
3. Click the **Revenue** column — the Summary shows skewness ≈ **-0.751** (slight left skew).
4. **Operations** → **Sort and filter** → **Sort values**:
   - Column: `Price` | Sort order: `Descending`
   - Click **Apply**
   - Highest price for Store 1227 = **$2.68**
5. To undo the sort: go to **Cleaning steps** panel → click the **delete icon** on the **Sort values** step.
6. Exit without generating code.

---

## Step A6 — Group By Aggregation (Average Revenue by Brand)

1. Re-launch Data Wrangler on `df`.
2. **Operations** → **Group by and aggregate**:
   - Group by: `Brand`
   - Add aggregation: `Revenue` → Aggregation type: `Mean`
   - Click **Apply** → **Copy code to clipboard**
3. Exit Data Wrangler.
4. Combine the aggregation code into the `clean_data` function:

```python
def clean_data(df):
    # Replace all instances of "." with " " in column: 'Brand'
    df['Brand'] = df['Brand'].str.replace(".", " ", case=False, regex=False)
    # Capitalize the first character in column: 'Brand'
    df['Brand'] = df['Brand'].str.title()

    # Group by Brand and calculate mean Revenue
    df = df.groupby(['Brand']).agg(Revenue_mean=('Revenue', 'mean')).reset_index()

    return df

df = clean_data(df)
print(df)
```

**Expected output:**

| Brand | Revenue_mean |
|-------|-------------|
| Dominicks | 33206.33 |
| Minute Maid | 33532.99 |
| Tropicana | 33637.86 |

---

## Step A7 — Save and Stop Session

1. Click ⚙️ **Settings** → rename notebook to `Pre-process_DataWrangler`.
2. Click **Stop session** to end Spark.

---

---

# Part B — Train and Track ML Models with MLflow

**Notebook:** `Train and Track with ML-Workflow_in_Notebook`
**Dataset:** Diabetes (Azure Open Datasets — `mlsamples/diabetes`)
**Experiment:** `experiment_diabetes`
**Saved Model:** `model-diabetes`

---

## What is MLflow?

**MLflow** is an open-source ML lifecycle management tool, integrated natively into Fabric. Every time you train a model inside an `mlflow.start_run()` block, it automatically logs:
- **Parameters** — model hyperparameters
- **Metrics** — training scores (R², MAE, RMSE, etc.)
- **Artifacts** — the saved model file

All runs are grouped under an **experiment**, making it easy to compare different models side by side.

---

## Step B1 — Load Diabetes Data

Create a new notebook named `Train and Track with ML-Workflow_in_Notebook`. Add a Markdown heading, then load the data:

```python
# Azure storage access info for Diabetes dataset
blob_account_name = "azureopendatastorage"
blob_container_name = "mlsamples"
blob_relative_path = "diabetes"
blob_sas_token = r""  # Anonymous access

wasbs_path = f"wasbs://%s@%s.blob.core.windows.net/%s" % (blob_container_name, blob_account_name, blob_relative_path)
spark.conf.set("fs.azure.sas.%s.%s.blob.core.windows.net" % (blob_container_name, blob_account_name), blob_sas_token)
print("Remote blob path: " + wasbs_path)

df = spark.read.parquet(wasbs_path)
```

Display and convert to Pandas:

```python
display(df)

import pandas as pd
df = df.toPandas()
df.head()
```

The dataset has 11 columns: `AGE`, `SEX`, `BMI`, `BP`, `S1`–`S6`, and `Y` (target — disease progression score).

---

## Step B2 — Prepare Train/Test Split

```python
from sklearn.model_selection import train_test_split

X, y = df[['AGE','SEX','BMI','BP','S1','S2','S3','S4','S5','S6']].values, df['Y'].values

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.30, random_state=0)
```

---

## Step B3 — Create the MLflow Experiment

```python
import mlflow
experiment_name = "experiment_diabetes"
mlflow.set_experiment(experiment_name)
```

This creates (or connects to) an experiment named `experiment_diabetes` in the workspace. All subsequent runs will be tracked here.

---

## Step B4 — Train Model 1: Linear Regression

```python
from sklearn.linear_model import LinearRegression

with mlflow.start_run():
    mlflow.autolog()

    model = LinearRegression()
    model.fit(X_train, y_train)

    mlflow.log_param("estimator", "LinearRegression")
```

`mlflow.autolog()` automatically logs all scikit-learn metrics and parameters. The extra `log_param` adds a custom label for easier comparison.

---

## Step B5 — Train Model 2: Decision Tree Regressor

```python
from sklearn.tree import DecisionTreeRegressor

with mlflow.start_run():
    mlflow.autolog()

    model = DecisionTreeRegressor(max_depth=5)
    model.fit(X_train, y_train)

    mlflow.log_param("estimator", "DecisionTreeRegressor")
```

---

## Step B6 — Query Experiment Runs Programmatically

List all experiments:

```python
experiments = mlflow.search_experiments()
for exp in experiments:
    print(exp.name)
```

Get a specific experiment and its runs:

```python
experiment_name = "experiment_diabetes"
exp = mlflow.get_experiment_by_name(experiment_name)

# All runs
mlflow.search_runs(exp.experiment_id)

# Top 2 most recent runs
mlflow.search_runs(exp.experiment_id, order_by=["start_time DESC"], max_results=2)
```

---

## Step B7 — Compare Models with a Bar Chart

```python
import matplotlib.pyplot as plt

df_results = mlflow.search_runs(
    exp.experiment_id,
    order_by=["start_time DESC"],
    max_results=2
)[["metrics.training_r2_score", "params.estimator"]]

fig, ax = plt.subplots()
ax.bar(df_results["params.estimator"], df_results["metrics.training_r2_score"])
ax.set_xlabel("Estimator")
ax.set_ylabel("R2 score")
ax.set_title("R2 score by Estimator")
for i, v in enumerate(df_results["metrics.training_r2_score"]):
    ax.text(i, v, str(round(v, 2)), ha='center', va='bottom', fontweight='bold')
plt.show()
```

This bar chart gives a visual comparison of R² scores between LinearRegression and DecisionTreeRegressor.

---

## Step B8 — Explore Experiment in Fabric UI

1. From the workspace, open `experiment_diabetes`.
2. Click the **View** tab → **Run list**.
3. Select the **two most recent runs** (checkbox).
4. In the **Metric comparison** pane, click the **🖉 Edit** button on the MAE graph.
5. Change: **Visualization type** → Bar | **X-axis** → estimator → click **Replace**.

You can repeat this for other metric graphs (RMSE, R², etc.) to visually compare the models.

> Your `experiment_diabetes` experiment shows **8 runs** in total (including re-runs), all tracked automatically by MLflow.

---

## Step B9 — Save the Best Model

1. In the experiment, go to **View** → **Run details**.
2. Select the run with the **highest R² score** — in this case `mango_horse_3cp...` with **R² = 0.7321**.
3. Click **Save** in the **Save run as model** box.
4. Select **Create a new model** → Navigate to the `model` folder.
5. Name the model `model-diabetes` → Click **Save**.
6. Click **View ML model** in the notification.

The saved model (`model-diabetes`, Version 1) shows the following metrics:

| Metric | Value |
|--------|-------|
| training_mean_absolute_error | 31.48 |
| training_mean_squared_error | 1683.83 |
| training_r2_score | **0.7321** |
| training_root_mean_squared_error | 41.03 |
| training_score | 0.7321 |

The model is **linked to the experiment run** — you can always trace back which run produced this version and what parameters were used.

---

## Step B10 — Save and Stop Session

1. Click ⚙️ **Settings** → rename the notebook to `Train and Track with ML-Workflow_in_Notebook`.
2. Click **Stop session** to end the Spark session.

---

## Summary

| Step | Part | Action | Tool Used |
|------|------|--------|-----------|
| A1 | Data Wrangler | Load OJ Sales CSV from Azure Blob | PySpark + Pandas |
| A2 | Data Wrangler | Explore data stats | Data Wrangler Summary panel |
| A3 | Data Wrangler | Clean Brand column (replace + capitalize) | Data Wrangler → exported to notebook |
| A4 | Data Wrangler | One-hot encode Brand | Data Wrangler (demo only) |
| A5 | Data Wrangler | Filter by Store, Sort by Price, Undo sort | Data Wrangler |
| A6 | Data Wrangler | Group by Brand, avg Revenue | Data Wrangler → exported to notebook |
| B1 | MLflow | Load Diabetes dataset | PySpark + Pandas |
| B2 | MLflow | Train/test split (70/30) | scikit-learn |
| B3 | MLflow | Create `experiment_diabetes` | MLflow |
| B4 | MLflow | Train LinearRegression + autolog | MLflow + scikit-learn |
| B5 | MLflow | Train DecisionTreeRegressor + autolog | MLflow + scikit-learn |
| B6 | MLflow | Query and compare runs programmatically | MLflow Python API |
| B7 | MLflow | Plot R² comparison bar chart | matplotlib |
| B8 | MLflow | Explore experiment runs in Fabric UI | Fabric Experiment view |
| B9 | MLflow | Save best run as `model-diabetes` v1 | Fabric ML model registry |

---

## Notes

- **Data Wrangler** only works with **Pandas DataFrames** — the Spark DataFrame must be converted with `.toPandas()` before launching it.
- Every time you re-open Data Wrangler from the notebook, it starts fresh — previous cleaning steps are not remembered unless you've already run the generated code.
- **`mlflow.autolog()`** must be called **inside** the `with mlflow.start_run():` block to capture metrics for that specific run.
- The experiment name in the notebook (`"experiment_diabetes"`) maps directly to the `experiment_diabetes` item visible in the workspace.
- Each MLflow run gets a randomly generated name (e.g., `mango_horse_3cp`, `gifted_eye_rhh`) — these are auto-assigned by MLflow and visible in the experiment run list.
- The **model-diabetes Version 1** is linked to the `mango_horse_3cp` run — this traceability is one of the core benefits of using MLflow for model tracking.

---

*Documented by: Sagar Parab | Workspace: Project 06_DataWrangler and MLFlow in MicrosoftFabric | Last updated: March 2026*
