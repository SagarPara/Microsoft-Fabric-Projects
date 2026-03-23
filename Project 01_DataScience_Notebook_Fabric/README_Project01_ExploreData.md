# Project 01 — Explore Data for Data Science using MS Fabric Notebooks

**Workspace:** `Project 01_DataScience_Notebook_Fabric`
**Lakehouse:** `DataScience_files`
**Notebook:** `01_ExploreData_DataScience`
**Dataset:** Diabetes Dataset (Azure Open Datasets)
**Reference:** [MS Learn Lab — Explore data for data science with notebooks in Microsoft Fabric](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/08a-data-science-explore-data.html)

---

## Overview

This project covers how to use a **Fabric Notebook** to explore a dataset for data science purposes. We use the **Diabetes dataset** from Azure Open Datasets. The main goal is to understand the data's structure, check for missing values, generate statistics, and build basic visualizations — all inside a Fabric Notebook using PySpark and Pandas.

By the end of this exercise, you should be comfortable with:
- Loading data from Azure Blob Storage into a Spark DataFrame
- Converting it to a Pandas DataFrame
- Running EDA (Exploratory Data Analysis) steps
- Plotting charts using matplotlib and seaborn

---

## Pre-requisites

- Access to the workspace: `Project 01_DataScience_Notebook_Fabric`
- Lakehouse `DataScience_files` already created under this workspace

---

## Workspace & Lakehouse Setup

1. Go to [https://app.fabric.microsoft.com](https://app.fabric.microsoft.com) and sign in.
2. From the left sidebar, open **Workspaces** and navigate to `Project 01_DataScience_Notebook_Fabric`.
3. The workspace should already have the following items available:
   - `DataScience_files` — Lakehouse
   - `01_ExploreData_DataScience` — Notebook
   - `02_read_txt_from_lakehouse_files...` — Notebook (for ingestion, used in a separate step)
   - `diabetes_powerbi_report` — Report
   - `Diabetes_report_test_01` — Semantic Model

> **Note:** The Lakehouse `DataScience_files` contains a table called `diabetes_table` with 442 rows and 11 columns. This is the table we are exploring in this notebook.

---

## Step-by-Step: Running the Notebook

### Step 1 — Open the Notebook

1. Inside the workspace, click on `01_ExploreData_DataScience` to open it.
2. The notebook will open with a Spark session attached.
3. The first cell should be a **Markdown** cell with a heading. If it is a code cell, convert it using the **M↓** button on the top-right of the cell.

---

### Step 2 — Load Data into a DataFrame

Add a new code cell and paste the following code. This loads the Diabetes dataset from Azure Open Datasets into a Spark DataFrame.

```python
# Azure storage access info for open dataset diabetes
blob_account_name = "azureopendatastorage"
blob_container_name = "mlsamples"
blob_relative_path = "diabetes"
blob_sas_token = r""  # Container is anonymously accessible

# Set Spark config to access blob storage
wasbs_path = f"wasbs://%s@%s.blob.core.windows.net/%s" % (blob_container_name, blob_account_name, blob_relative_path)
spark.conf.set("fs.azure.sas.%s.%s.blob.core.windows.net" % (blob_container_name, blob_account_name), blob_sas_token)
print("Remote blob path: " + wasbs_path)

# Read as Spark DataFrame (parquet format)
df = spark.read.parquet(wasbs_path)
```

> **Note:** First run may take 1–2 minutes as the Spark pool needs to start. Subsequent runs are faster.

Run the next cell to preview the data:

```python
display(df)
```

The dataset has **11 columns**: AGE, SEX, BMI, BP, S1, S2, S3, S4, S5, S6, and Y (target variable — disease progression score).

---

### Step 3 — Convert Spark DataFrame to Pandas

Scikit-learn works with Pandas DataFrames, so we convert here:

```python
df = df.toPandas()
df.head()
```

---

### Step 4 — Check Shape and Data Types

```python
print("Number of rows:", df.shape[0])
print("Number of columns:", df.shape[1])

print("\nData types of columns:")
print(df.dtypes)
```

**Expected output:** 442 rows, 11 columns. The `SEX` column contains numeric values (1 = Male, 2 = Female).

---

### Step 5 — Check for Missing Values

```python
missing_values = df.isnull().sum()
print("\nMissing values per column:")
print(missing_values)
```

**Result:** No missing values found in this dataset.

---

### Step 6 — Descriptive Statistics

```python
df.describe()
```

Key observations:
- Average age is ~48.5 years (range: 19–79)
- Average BMI is ~26.4, which falls in the **overweight** category (WHO standard)
- BMI range is 18 to 42.2

---

### Step 7 — Plot BMI Distribution

```python
import matplotlib.pyplot as plt
import seaborn as sns
import numpy as np

mean = df['BMI'].mean()
median = df['BMI'].median()

plt.figure(figsize=(8, 6))
plt.hist(df['BMI'], bins=20, color='skyblue', edgecolor='black')
plt.title('BMI Distribution')
plt.xlabel('BMI')
plt.ylabel('Frequency')
plt.axvline(mean, color='red', linestyle='dashed', linewidth=2, label='Mean')
plt.axvline(median, color='green', linestyle='dashed', linewidth=2, label='Median')
plt.legend()
plt.show()
```

**Observation:** Most BMI values fall between 23.2 and 29.2. The distribution is slightly right-skewed.

---

### Step 8 — Multivariate Analysis

**BMI vs Target Variable (Scatter Plot)**

```python
plt.figure(figsize=(8, 6))
sns.scatterplot(x='BMI', y='Y', data=df)
plt.title('BMI vs. Target variable')
plt.xlabel('BMI')
plt.ylabel('Target')
plt.show()
```

**Observation:** Positive linear relationship — as BMI increases, the disease progression score (Y) also tends to increase.

---

**Blood Pressure by Gender (Box Plot)**

```python
fig, ax = plt.subplots(figsize=(7, 5))
df['SEX'] = df['SEX'].replace({1: 'Male', 2: 'Female'})
sns.boxplot(x='SEX', y='BP', data=df, ax=ax)
ax.set_title('Blood pressure across Gender')
plt.tight_layout()
plt.show()
```

**Observation:** Female patients show a slightly higher average blood pressure than male patients.

---

**Average BP and BMI by Gender (Bar Chart)**

```python
avg_values = df.groupby('SEX')[['BP', 'BMI']].mean()
ax = avg_values.plot(kind='bar', figsize=(15, 6), edgecolor='black')
plt.title('Avg. Blood Pressure and BMI by Gender')
plt.xlabel('Gender')
plt.ylabel('Average')

for p in ax.patches:
    ax.annotate(format(p.get_height(), '.2f'),
                (p.get_x() + p.get_width() / 2., p.get_height()),
                ha='center', va='center',
                xytext=(0, 10),
                textcoords='offset points')
plt.show()
```

---

**BMI over Age (Line Chart)**

```python
plt.figure(figsize=(10, 6))
sns.lineplot(x='AGE', y='BMI', data=df, errorbar=None)
plt.title('BMI over Age')
plt.xlabel('Age')
plt.ylabel('BMI')
plt.show()
```

**Observation:** BMI is lowest in the 19–30 age group and highest in the 65–79 group.

---

### Step 9 — Correlation Analysis

**Correlation Matrix**

```python
df.corr(numeric_only=True)
```

**Heatmap**

```python
plt.figure(figsize=(15, 7))
sns.heatmap(df.corr(numeric_only=True), annot=True, vmin=-1, vmax=1, cmap="Blues")
```

Key findings:
- **S1 and S2** have a strong positive correlation of **0.89** — they move together
- **S3 and S4** have a strong negative correlation of **-0.73** — they move in opposite directions

---

### Step 10 — Save the Notebook and Stop Spark Session

1. Click the ⚙️ **Settings** icon in the notebook toolbar.
2. Rename the notebook to `01_ExploreData_DataScience` (or keep it as is).
3. Close the settings pane.
4. Click **Stop session** from the notebook menu to end the Spark session and free up resources.

---

## Summary

| Step | Action | Tool Used |
|------|--------|-----------|
| 1 | Load dataset from Azure Blob Storage | PySpark |
| 2 | Convert to Pandas DataFrame | Pandas |
| 3 | Check shape, data types, missing values | Pandas |
| 4 | Descriptive statistics | Pandas `.describe()` |
| 5 | BMI distribution plot | matplotlib |
| 6 | Multivariate analysis (scatter, box, bar, line) | seaborn + matplotlib |
| 7 | Correlation heatmap | seaborn |

---

## Notes

- This is an **exploratory** exercise — no model is trained here. Model training is covered in a follow-up project.
- The `diabetes_table` in `DataScience_files` Lakehouse is the persisted version of this dataset, loaded via the `02_read_txt_from_lakehouse_files` notebook.
- All notebooks are stored under the `Project 01_DataScience_Notebook_Fabric` workspace in the **Development** environment.

---

*Documented by: Sagar Parab | Workspace: Project 01_DataScience_Notebook_Fabric | Last updated: March 2026*
