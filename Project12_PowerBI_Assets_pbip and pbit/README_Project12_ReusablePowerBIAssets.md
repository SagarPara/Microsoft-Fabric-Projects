# Project 12 — Create Reusable Power BI Assets

**Tool:** Power BI Desktop (local exercise — no Fabric workspace required)
**Source files folder:** `C:\Users\Student\Downloads\16-reusable-assets\`
**Power BI Project file:** `16-Starter-Sales Analysis.pbip`
**Power BI Template file:** `regional-sales.pbit`
**Datasets used:** Sales Analysis (.pbix), US Population (HTML), region-north.csv, region-south.csv
**Reference:** [MS Learn Lab — Create reusable Power BI assets](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/16-create-reusable-power-bi-assets.html)

---

## Overview

This project is different from all previous ones — it is a **local Power BI Desktop exercise** and does not require a Fabric workspace or trial. It focuses on two types of **reusable Power BI assets**:

**Part A — Power BI Project file (`.pbip`)**
A `.pbip` file stores a report and its semantic model as **flat text files** (including TMDL format for the semantic model) rather than a single binary `.pbix` file. This makes it compatible with **source control systems like Git** — you can track every change to measures, relationships, and tables as readable text diffs.

**Part B — Power BI Template file (`.pbit`)**
A `.pbit` file is a reusable report template that contains the **report layout and data model structure**, but no data. It uses a **Power Query parameter** to let users select which dataset to load when opening the template. This is useful for creating standardised reports that run across multiple regions, departments, or time periods.

```
Part A: Power BI Project (.pbip)                Part B: Power BI Template (.pbit)
─────────────────────────────                   ──────────────────────────────────
16-Starter-Sales Analysis.pbix                  region-north.csv + region-south.csv
        ↓ Save as .pbip                                   ↓ Load via Get Data
16-Starter-Sales Analysis.pbip                  sales query (Power Query)
        ↓ Enables flat file storage                       ↓ Create Region parameter
16-Starter-Sales Analysis.Report/               Region = "north" | "south"
16-Starter-Sales Analysis.SemanticModel/                  ↓ Parameterize file path
  └── definition/                               regional-sales report created
       ├── tables/Sales.tmdl                             ↓ Save as .pbit
       ├── relationships.tmdl                   regional-sales.pbit (template, 12KB)
       └── ...                                           ↓ Open template
        ↓ Add US Population table                Region prompt → select "south"
        ↓ Add relationship                                ↓
        ↓ Add Sales per Capita measure           Report loads South region data only
Changes visible in .tmdl files                  (Revenue 680, Widget A/B/C)
```

---

## Pre-requisites

- Power BI Desktop installed (no Fabric licence needed for this exercise)
- Download and extract the lab files:

  ```
  https://github.com/MicrosoftLearning/mslearn-fabric/raw/refs/heads/main/Allfiles/Labs/16b/16-reusable-assets.zip
  ```

  Extract to: `C:\Users\Student\Downloads\16-reusable-assets\`

### Folder structure after extraction:

```
16-reusable-assets\
├── data\
│   ├── region-north.csv
│   └── region-south.csv
├── PowerBI_Project\           ← used in Part A
├── Solution\
├── 16-Starter-Sales Analysis.pbix   (1,089 KB)
├── regional-sales.pbit              (12 KB — template output from Part B)
└── us-resident-population-estimates-2020.html
```

---

---

# Part A — Power BI Project File (.pbip)

---

## Step A1 — Enable TMDL Preview Feature

Before saving as `.pbip`, you need to enable the TMDL (Tabular Model Definition Language) preview feature.

1. Open `16-Starter-Sales Analysis.pbix` from the lab folder.
2. Go to **File → Options and settings → Options → Preview features**.
3. Check **Store semantic model using TMDL format** → Click **OK**.
4. Restart Power BI Desktop if prompted.

> **TMDL** stores the semantic model as human-readable text files — measures, relationships, table definitions — instead of a single binary blob. This is what makes the `.pbip` format compatible with Git.

---

## Step A2 — Save as Power BI Project (.pbip)

1. Go to **File → Save as**.
2. Click the dropdown arrow on the file type selector.
3. Choose **.pbip** as the file extension.
4. Name the file (e.g., `16-Starter-Sales Analysis`) and save it in a folder you'll remember (e.g., inside `PowerBI_Project\`).
5. The title bar will now show the report name followed by **(Power BI Project)**.
6. Close Power BI Desktop.

---

## Step A3 — Review the Project File Structure

1. Open File Explorer and navigate to where you saved the `.pbip`.
2. You should see:

   ```
   PowerBI_Project\
   ├── 16-Starter-Sales Analysis.pbip     ← the project entry point (1 KB)
   ├── 16-Starter-Sales Analysis.Report\  ← report pages, visuals, layout
   ├── 16-Starter-Sales Analysis.SemanticModel\  ← tables, measures, relationships
   └── .gitignore                         ← auto-generated for source control
   ```

3. Re-open the `.pbip` file in Power BI Desktop.
4. If prompted to **Upgrade** to TMDL format → Click **Upgrade**.
5. Save.

---

## Step A4 — Add US Population Table

The existing model doesn't have population data. We add it from a local HTML file.

1. In Power BI Desktop, go to **Get data → Web**.
2. Select **Basic** and enter the file path:

   ```
   C:\Users\Student\Downloads\16-reusable-assets\us-resident-population-estimates-2020.html
   ```

3. In the Navigator, check **HTML Tables → Table 2** → Click **Transform Data**.
4. In Power Query Editor:
   - Rename **Table 2** → `US Population`
   - Rename column **STATE** → `State`
   - Rename column **NUMBER** → `Population`
   - **Remove** the `RANK` column
5. Click **Close & Apply**.
6. Click **OK** if a security risk dialog appears.
7. Save.
   > If prompted to upgrade to enhanced report format → Click **Don't upgrade**.

---

## Step A5 — Create a Relationship

1. Open File Explorer → Navigate to the `.SemanticModel\definition\` folder.
2. Open `relationships.tmdl` in Notepad — note it lists **9 relationships**. Close the file.
3. Back in Power BI Desktop, go to **Modeling → Manage relationships** — also 9 relationships.
4. Click **New** to create a relationship:

   | Setting | Value |
   |---------|-------|
   | From table | Reseller |
   | From key column | State-Province |
   | To table | US Population |
   | To key column | State |
   | Cardinality | Many-to-one (\*:1) |
   | Cross-filter direction | Both |

5. Click **OK** → Save.
6. Check `relationships.tmdl` again — the new relationship is now added as readable text.

> This is the key advantage of `.pbip` — every structural change shows up in the flat files and can be tracked in Git as a diff.

---

## Step A6 — Add a Measure and Visual

### Create the Measure

1. In the **Data** pane, select the **Sales** table.
2. Click **New measure** on the Table tools ribbon.
3. Enter the following DAX:

   ```dax
   Sales per Capita =
   DIVIDE(
       SUM(Sales[Sales]),
       SUM('US Population'[Population])
   )
   ```

4. Drag the `Sales per Capita` measure onto the canvas.
5. Also drag: `Sales | Sales`, `US Population | State`, `US Population | Population` to the same visual.
6. Change the visual type to **Table**.

### Format the Fields

| Field | Format | Decimal places |
|-------|--------|----------------|
| Sales per Capita | Currency | 4 |
| Population | Whole number, comma separated | 0 |

7. Save.

### Verify in TMDL

1. Open Notepad.
2. Open: `C:\Users\Student\Downloads\16-Starter-Sales Analysis.SemanticModel\definition\tables\Sales.tmdl`
3. Around line 163 you will see the measure has been added:

   ```
   measure 'Sales per Capita' =
       DIVIDE(
           SUM(Sales[Sales]),
           SUM('US Population'[Population])
       )
   formatString: $#,0.0000;($#,0.0000);$#,0.0000
   ```

> This confirms that every DAX measure you write is stored as plain text inside the TMDL files — version-controllable and human-readable.

8. Close Notepad and close Power BI Desktop (no need to save).

---

---

# Part B — Power BI Template File (.pbit)

---

## What is a `.pbit` Template?

A **Power BI Template** (`.pbit`) packages a report's layout, data model, and Power Query queries — but **without the actual data**. When a user opens a `.pbit`, they are prompted to fill in any defined parameters, and Power BI loads the corresponding data fresh.

This is ideal for:
- Reports that run against different regions, departments, or environments
- Sharing a report structure without sharing the underlying data
- Standardising report design across a team

---

## Step B1 — Explore the Regional Data Files

Navigate to `C:\Users\Student\Downloads\16-reusable-assets\data\` and open both CSV files:

- `region-north.csv` — sales data for North region
- `region-south.csv` — sales data for South region

Both have the same structure: `Date`, `Region`, `Product`, `Units`, `Revenue`

---

## Step B2 — Create a New Report and Load Data

1. Open Power BI Desktop → Create a **blank report**.
2. **Home → Get Data → Text/CSV**.
3. Load: `C:\Users\Student\Downloads\16-reusable-assets\data\region-north.csv`
4. Click **Transform Data**.
5. In Power Query Editor, rename the query to `sales`.
6. Do not close Power Query yet.

---

## Step B3 — Create a Power Query Parameter

Still in Power Query Editor:

1. **Home → Manage Parameters → New Parameter**.
2. Configure:

   | Property | Value |
   |----------|-------|
   | Name | `Region` |
   | Type | Text |
   | Suggested Values | List of values |
   | Values | `north`, `south` |
   | Default Value | `north` |
   | Current Value | `north` |

3. Click **OK**.

---

## Step B4 — Parameterize the File Path in the Query

1. Select the `sales` query.
2. **View → Advanced Editor**.
3. The current query has a hard-coded file path:

   ```m
   let
       Source = Csv.Document(File.Contents("C:\Users\Student\Downloads\16-reusable-assets\data\region-north.csv"),
           [Delimiter=",", Columns=5, Encoding=1252, QuoteStyle=QuoteStyle.None]),
       #"Promoted Headers" = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
       #"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers",
           {{"Date", type date}, {"Region", type text}, {"Product", type text},
            {"Units", Int64.Type}, {"Revenue", Int64.Type}})
   in
       #"Changed Type"
   ```

4. Replace `"region-north.csv"` with `"region-" & Region & ".csv"`:

   ```m
   let
       Source = Csv.Document(File.Contents("C:\Users\Student\Downloads\16-reusable-assets\data\region-" & Region & ".csv"),
           [Delimiter=",", Columns=5, Encoding=1252, QuoteStyle=QuoteStyle.None]),
       #"Promoted Headers" = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
       #"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers",
           {{"Date", type date}, {"Region", type text}, {"Product", type text},
            {"Units", Int64.Type}, {"Revenue", Int64.Type}})
   in
       #"Changed Type"
   ```

   > In Power Query M language, the `&` operator concatenates text strings. So `"region-" & Region & ".csv"` builds the filename dynamically based on whatever the `Region` parameter is set to.

5. Click **Close & Apply**.

---

## Step B5 — Build the Report Visuals

1. In Report view, create:
   - A **Card** visual showing total Revenue
   - A **Column chart** showing Revenue by Product
   - A **Table** showing all fields (Date, Region, Product, Units, Revenue)
2. Add a title text box: `Regional Sales Report Template`

---

## Step B6 — Save as Template (.pbit)

1. **File → Save as**.
2. Select the save location (e.g., the `16-reusable-assets` root folder).
3. File name: `regional-sales`
4. File type: **PBIT** (Power BI Template)
5. When prompted for a template description, enter:

   ```
   Select your region.
   ```

6. Click **OK** — the `.pbit` file is saved (12 KB).

---

## Step B7 — Test the Template

1. Close Power BI Desktop. Click **Don't save** when prompted.
2. Open `regional-sales.pbit` from File Explorer.
3. The **parameter prompt** appears:

   ```
   regional-sales
   Select your region.
   Region: [south ▾]
   ```

4. Select **south** from the dropdown → Click **Load**.
5. The report opens with South region data:

   | Year | Quarter | Month | Day | Region | Product | Sum of Revenue | Sum of Units |
   |------|---------|-------|-----|--------|---------|----------------|--------------|
   | 2025 | Qtr 1 | January | 1 | South | Widget A | 200 | 10 |
   | 2025 | Qtr 1 | January | 2 | South | Widget C | 210 | 7 |
   | 2025 | Qtr 1 | January | 3 | South | Widget B | 270 | 9 |
   | **Total** | | | | | | **680** | **26** |

   The bar chart shows: Widget B (270) > Widget C (210) > Widget A (200)

6. Close the report — no need to save.

---

## Summary

| Step | Part | Action | Output |
|------|------|--------|--------|
| A1 | PBIP | Enable TMDL preview feature | TMDL format available in Save as |
| A2 | PBIP | Save `.pbix` as `.pbip` | Flat file project structure created |
| A3 | PBIP | Review project folder | Report/, SemanticModel/, .gitignore visible |
| A4 | PBIP | Add US Population table from HTML | New table in semantic model |
| A5 | PBIP | Create Reseller ↔ US Population relationship | relationships.tmdl updated |
| A6 | PBIP | Add `Sales per Capita` DAX measure | Sales.tmdl updated — measure visible as text |
| B1 | PBIT | Explore north/south CSV source files | Understand data structure |
| B2 | PBIT | Create blank report, load region-north.csv | `sales` query created |
| B3 | PBIT | Create `Region` parameter (north/south) | Parameter available in M query |
| B4 | PBIT | Parameterize file path using `&` concatenation | File loads dynamically by region |
| B5 | PBIT | Build Card + Column chart + Table visuals | Report layout complete |
| B6 | PBIT | Save as `regional-sales.pbit` | 12 KB reusable template |
| B7 | PBIT | Open template, select "south" → Load | South region report loads correctly |

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| `.pbip` | Power BI Project format — stores report and semantic model as flat files; Git-compatible |
| `.pbit` | Power BI Template — report layout + model structure, no data; parameterised for reuse |
| **TMDL** | Tabular Model Definition Language — text-based format for semantic model definitions |
| **Power Query parameter** | A named variable in Power Query that can be changed at open-time to control what data loads |
| **`&` operator (M language)** | Text concatenation in Power Query — used to build dynamic file paths |
| **`relationships.tmdl`** | Flat file listing all model relationships — changes here are trackable in Git |
| **`Sales.tmdl`** | Flat file containing table definition including any DAX measures added to that table |

---

## Notes

- This exercise is entirely **local to Power BI Desktop** — no Fabric workspace, no cloud upload required. A Power BI licence is not needed either.
- The `.pbip` format is most valuable when **combined with Git** (e.g., GitHub, Azure DevOps) — each commit shows exactly which measures, relationships, or visuals changed, unlike `.pbix` which is binary and not diffable.
- The `.pbit` template stores the **Region parameter definition and query logic** but not the actual CSV data. Every time someone opens the `.pbit`, they pick a region and fresh data is loaded from the local CSV files.
- The `"region-" & Region & ".csv"` path pattern in Power Query means the `data` folder must exist at the same relative path on any machine that opens the template — or the file path must be updated accordingly.
- **Sales per Capita** is a ratio measure — it divides total sales revenue by total US population. In a real scenario this would be useful for regional market penetration analysis.
- The `16-Starter-Sales Analysis.SemanticModel\definition\tables\Sales.tmdl` file referenced in Step A6 is at the **downloaded/extracted** path (`C:\Users\Student\Downloads\`) — not inside the `PowerBI_Project` folder. Keep this in mind when locating the file.

---

*Documented by: Sagar Parab | Local Power BI Desktop Exercise (no Fabric workspace) | Last updated: March 2026*
