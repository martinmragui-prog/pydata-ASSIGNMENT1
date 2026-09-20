import numpy as np
import pandas as pd

pd.set_option("display.width", 160)
pd.set_option("display.max_columns", 20)
89

# Raw Markdown Content
MARKDOWN_TEXT = """# Reading the Fine Print: A CRISP-DM Walkthrough of a Kenyan Retail Chain’s September Budget

**Cleaning, understanding, and questioning a 72-branch sales budget workbook in Python and VS Code.**

---

When I opened `article_001.xlsx`, I expected a straightforward budget review: branch names, monthly targets, actual sales, and a clean rollup. 

Instead, I got a practical reminder of the gap between *"the data looks fine"* and *"the data **is** fine."* 

Inside were five loosely structured sheets, a merged header that broke initial imports, an entire block of duplicated columns, and actual sales figures that didn't align with the month claimed on the label.

Here is how I audited the dataset using the CRISP-DM (Cross-Industry Standard Process for Data Mining) framework in a VS Code Jupyter Notebook.

---

## Table of Contents
1. **Business & Domain Understanding**
2. **Data Understanding**
3. **Data Preparation**
4. **Filtering Anomalies**
5. **Sorting & Revenue Distribution**
6. **Final Takeaways & Actionable Next Steps**

---

## 1. Business & Domain Understanding

Before running code, I needed to understand the mechanics of the business. The workbook covers **72 supermarket branches** in Kenya, each operating up to four key fresh-food departments: **Bakery, Butchery, Deli, and Veges**.

The `WORKINGS` sheet revealed the core logic behind the budget. Targets aren't built ground-up by store managers; they're set at the store level and split top-down using fixed chain-wide departmental ratios:

* **Veges:** 29.35%
* **Bakery:** 27.70%
* **Deli:** 22.00%
* **Butchery:** 20.94%

```python
# Inspecting departmental weights from the WORKINGS sheet
print(
    "Domain summary:\\n"
    "- Supermarket chain: 72 branches, each running up to 4 departments (Bakery, Butchery, Deli, Veges).\\n"
    "- Monthly TARGET is set per branch-department, compared day-by-day against ACTUAL sales.\\n"
    "- Targets are built top-down: a store-level number is split across departments using fixed weights."
)
```

For newly opened branches, the store target relies on an assumed annual turnover formula (12.19%). Knowing that the budget relies on rigid, top-down allocation made it much easier to spot anomalies down the line.

---

## 2. Data Understanding

When loading the Excel sheets into pandas, several structural issues surfaced right away.

### Issue A: Merged Excel Headers
The `SEPTEMBER` sheet uses a two-row merged header. Parsing it naively causes pandas to pull weekday labels into the column names. Skipping the offset row with `header=1` brings the dimensions into focus:

```python
actual = pd.read_excel(xl, "SEPTEMBER", header=1)
budget = pd.read_excel(xl, "SEPTEMBER BUDGET", header=1)

print("actual:", actual.shape, "| budget:", budget.shape)
# Output: actual: (290, 66) | budget: (297, 33)
```

The `actual` table yielded 66 columns—far too wide for a single month's daily tracking—signaling duplicate or mislabelled date blocks.

### Issue B: Non-Existent Branch Departments
Checking for missing values revealed **7 branch-department combinations with zero actual sales**:

```python
no_actuals = actual[actual["BRANCH"].notna() & actual["Total Actual Sales"].isna()]
print(f"Branch-departments with no actual sales: {len(no_actuals)}")
```

* **Pipeline & Outering:** No Butchery department.
* **Shabaab:** Operates Bakery only (missing Butchery, Deli, Veges).
* **Kericho:** No Veges department.

These aren't missing data points or operational failures—these locations simply do not stock those product lines. Imputing zeros blindly would skew performance averages across the chain.

---

## 3. Data Preparation

Cleaning involved dropping trailing grand-total rows (where `BRANCH` was NaN), trimming redundant columns, and computing simple metrics like `VARIANCE` and `ACHIEVEMENT_PCT`.

```python
# Drop totals and unify summary metrics
actual_clean = actual.dropna(subset=["BRANCH"]).copy()
budget_clean = budget.dropna(subset=["BRANCH"]).copy()

summary = actual_clean[["BRANCH", "Department", "Total Actual Sales", "SEPT SALES TARGET"]].copy()
summary.columns = ["BRANCH", "DEPARTMENT", "ACTUAL_SALES", "TARGET_SALES"]

summary["VARIANCE"] = summary["ACTUAL_SALES"] - summary["TARGET_SALES"]
summary["ACHIEVEMENT_PCT"] = (summary["ACTUAL_SALES"] / summary["TARGET_SALES"]) * 100
```

After dropping summary rows, both datasets normalized to **288 rows** (72 branches × 4 departments).

---

## 4. Filtering Anomalies

Filtering for low target values exposed severe data entry bugs upstream.

```python
# Isolate branch departments with implausibly low targets
suspect = summary[summary["TARGET_SALES"] < 100_000]
print(f"Targets under KES 100,000/month (suspected errors): {len(suspect)}")
```

| BRANCH | DEPARTMENT | ACTUAL_SALES (KES) | TARGET_SALES (KES) | Implied Achievement |
| :--- | :--- | :--- | :--- | :--- |
| KONDELE | BUTCHERY | 298,783.10 | 10,622.05 | 2,812% |
| MAYFAIR | BUTCHERY | 55,296.68 | 45,768.58 | 120% |
| TOM MBOYA | BUTCHERY | 126,380.41 | 37,249.41 | 339% |
| KERICHO | BUTCHERY | 64,155.21 | 31,056.56 | 206% |
| PIPELINE | DELI | 69,229.29 | 34,202.12 | 202% |
| OTC | VEGES | 160,751.59 | 28,847.26 | 557% |

Kondele Butchery's actual sales are 28 times its budget target. That isn't high performance; it's a misplaced decimal point in the target allocation. Six branches suffered from this error and need manual target fixes before generating executive reports.

---

## 5. Sorting & Revenue Distribution

Sorting active departments by actual sales highlights revenue concentration across the footprint:

```python
biggest_first = summary.sort_values("ACTUAL_SALES", ascending=False).reset_index(drop=True)
biggest_first.head(7)
```

1. **Lavington (Veges):** KES 12.84M actual vs. KES 10.44M target
2. **Kileleshwa (Veges):** KES 12.20M actual vs. KES 12.00M target
3. **Kilimani (Veges):** KES 10.12M actual vs. KES 8.85M target
4. **Kileleshwa (Butchery):** KES 9.95M actual vs. KES 8.34M target
5. **Lavington (Butchery):** KES 9.50M actual vs. KES 7.81M target

Just three fresh-produce departments (Lavington, Kileleshwa, and Kilimani Veges) drive over **KES 35 Million** in revenue. A tiny tracking error in these flagship stores impacts overall chain performance more than dozens of smaller branches combined.

---

## 6. Takeaways & Next Steps

This project didn't end with a complex machine learning model—and it didn't need one. What it needed was basic data auditing: checking headers, verifying formulas, and flagging impossible targets.

### Key Action Items:
1. **Fix Broken Targets:** Correct the six sub-KES 100,000 targets (such as Kondele Butchery) before distributing monthly KPI reports.
2. **Revise New-Store Logic:** Base targets for newer stores (e.g., Nakuru branches) on early trading data rather than fixed top-down projections.
3. **Correct Column Headers:** Relabel the actuals data block with explicit date ranges instead of vague labels.
4. **Clean Excel Assets:** Remove redundant working sheets and duplicate column blocks to prevent sync issues.

In data analysis, **Data Understanding** isn't just a setup phase—it's often where the most critical business value is found.
