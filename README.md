# Retail Sales Data: Exploratory Data Analysis (EDA) Project

A step-by-step exploratory data analysis of 1,000 retail sales orders from 2025, done in Python with pandas, NumPy and Matplotlib. The project covers the full EDA workflow: loading and inspecting the data, correcting column names, converting data types, treating missing values, checking duplicates and outliers, and running univariate, bivariate and multivariate analysis.

A 12-slide presentation (`Sales_EDA_Presentation.pptx`) walks through every step with the code and results.

---

## Table of contents

- [Dataset]
- [Tech stack]
- [Project structure]
- [Getting started]
- [EDA workflow]
- [Key results]
- [Data quality issues and limitations]
- [Next steps]

---

## Dataset

`sales_data.csv` has **1,000 rows and 9 columns**. Each row is one sales order placed between **1 January and 31 December 2025**.

| Column | Description | Type after cleaning |
|---|---|---|
| `order_id` | Order identifier (e.g. ORD10001) | string |
| `order_date` | Date of the order | datetime |
| `customer_name` | Customer who placed the order (10 distinct customers) | string |
| `city` | City of the order | string |
| `category` | Product category (5 categories) | string |
| `product` | Product name (20 products) | string |
| `quantity` | Units ordered | int |
| `unit_price` | Price per unit (₹) | float |
| `discount` | Discount given (%), ranging 0 to about 25 | float |

> The dataset file is not included in this repository. Place your own copy of `sales_data.csv` in the `data/` folder.

## Tech stack

- Python 3
- pandas
- NumPy
- Matplotlib
- Jupyter Notebook

## Project structure

```
.
├── data/
│   └── sales_data.csv              # input data (add your own copy)
├── notebooks/
│   └── sales_eda.ipynb             # the analysis notebook
├── presentation/
│   └── Sales_EDA_Presentation.pptx # 12-slide walkthrough of the project
└── README.md
```

Adjust the folder names to match your repository.

## Getting started

```bash
# 1. clone the repository
git clone <your-repo-url>
cd <your-repo-folder>

# 2. install dependencies
pip install pandas numpy matplotlib jupyter

# 3. start the notebook
jupyter notebook notebooks/sales_eda.ipynb
```

Before running, change the file path in the load step to a relative path:

```python
data = pd.read_csv("../data/sales_data.csv")
```

## EDA workflow

### Step 1: Import libraries and load the data

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

data = pd.read_csv("sales_data.csv")
df = data.copy()   # work on a copy and keep the raw data safe
```

### Step 2: Understand the data

```python
df.head()
df.tail()
df.shape                         # (1000, 9)
df.info()
df.describe()
df.describe(include="object")
df.dtypes
df.columns

for e in df.columns:
    print("\n---", e, "---")
    print("unique:", df[e].nunique(dropna=True))
    print(df[e].value_counts(dropna=False).head(5))
```

What this showed:
- `unit_price` was loaded as text and needed conversion.
- Missing values in `customer_name`, `city` and `discount`.
- Only 988 unique `order_id` values across 1,000 rows.
- `quantity` ranges from 0 to 75, with a mean of about 4.2.
- `city` has 40 distinct labels.

### Step 3: Correct column names and text

```python
# standardise column names
df.columns = (df.columns.str.strip().str.lower()
              .str.replace(r"[^a-z0-9]+", "_", regex=True))

# trim spaces in every text column
for c in df.select_dtypes(include=["object", "string"]).columns:
    df[c] = df[c].astype("string").str.strip()
```

Recommended addition (not yet in the notebook), to unify city casing such as "PUNE" and "jaipur":

```python
df["city"] = df["city"].str.title()
```

### Step 4: Convert data types

```python
df["order_date"] = pd.to_datetime(df["order_date"], errors="coerce")

for c in ["quantity", "unit_price", "discount"]:
    df[c] = pd.to_numeric(df[c], errors="coerce")
```

`errors="coerce"` turns invalid values into `NaN`, so they are caught by the missing-value step.

### Step 5: Detect and treat missing values

```python
missing = pd.DataFrame({
    "count": df.isna().sum(),
    "percent": (df.isna().mean() * 100).round(2)})
display(missing[missing["count"] > 0])

df["discount"] = df["discount"].fillna(0)
df["unit_price"] = df["unit_price"].fillna(df["unit_price"].median())
df = df.dropna(subset=["order_date"])
df["customer_name"] = df["customer_name"].fillna("Unknown")
df["city"] = df["city"].fillna("Unknown")
```

| Column | Missing | Share | Treatment |
|---|---|---|---|
| `order_date` | 20 | 2.0% | rows dropped |
| `city` | 18 | 1.8% | filled with "Unknown" |
| `discount` | 15 | 1.5% | filled with 0 |
| `unit_price` | 12 | 1.2% | filled with the median |
| `customer_name` | 10 | 1.0% | filled with "Unknown" |

After treatment, **980 rows** remain with no missing values.

### Step 6: Check duplicates

```python
print("duplicate rows:", df.duplicated().sum())
df = df.drop_duplicates().reset_index(drop=True)
```

No fully duplicate rows were found.

### Step 7: Detect outliers with the IQR rule

```python
rows = []
for c in df.select_dtypes(np.number):
    q1, q3 = df[c].quantile([.25, .75])
    iqr = q3 - q1
    lo, hi = q1 - 1.5 * iqr, q3 + 1.5 * iqr
    n = ((df[c] < lo) | (df[c] > hi)).sum()
    rows.append([c, q1, q3, lo, hi, int(n), round(n / len(df) * 100, 2)])

display(pd.DataFrame(rows, columns=["column", "q1", "q3", "lower",
                                    "upper", "outlier_count", "outlier_percent"]))
```

| Column | Outliers | Share |
|---|---|---|
| `quantity` | 21 | 2.16% |
| `unit_price` | 79 | 8.14% |
| `discount` | 0 | 0% |

Outliers were flagged for review and not removed.

### Step 8: Univariate analysis

```python
for c in df.select_dtypes(np.number):
    plt.figure(figsize=(8, 4))
    plt.hist(df[c].dropna(), bins=30)
    plt.title(c); plt.xlabel(c); plt.ylabel("Frequency")
    plt.show()
```

- **Quantity:** most orders are 1 to 6 units, with a few extreme orders of 50 and 75 units.
- **Unit price:** right-skewed, with most orders low-priced and a long tail up to about ₹90,000.
- **Discount:** roughly even from 0% to 25%, with no outliers.

### Step 9: Bivariate analysis

```python
pairs = [("unit_price", "quantity"),
         ("unit_price", "discount"),
         ("quantity", "discount")]

for x, y in pairs:
    if x in df.columns and y in df.columns:
        plt.figure(figsize=(7, 5))
        plt.scatter(df[x], df[y], alpha=0.65)
        plt.xlabel(x); plt.ylabel(y); plt.title(f"{y} vs {x}")
        plt.show()
```

- Quantity stays under about 10 units per order at every price level.
- There is no visible relationship between discount and unit price.
- There is no visible relationship between discount and quantity.

### Step 10: Group-by and multivariate analysis

```python
# units sold per product
df.groupby("product")["quantity"].agg(["sum", "mean", "count"]) \
  .sort_values("sum", ascending=False)

# discount per product
df.groupby("product")["discount"].agg(["sum", "mean", "count"]) \
  .sort_values("sum", ascending=False)

# discount per order size
df.groupby("quantity")["discount"].agg(["sum", "mean", "count"]) \
  .sort_values("sum", ascending=False)

# city x category
dims = ["city", "category"]
df.groupby(dims)[["quantity", "discount"]].sum() \
  .sort_values("quantity").head(20)
```

## Key results

- **Units sold:** Rice (306), Jeans (294), Office Chair (268), Tea (267) and Washing Machine (255) lead. Tablet (130), Microwave (132), Laptop (135) and Jacket (129) sell the fewest.
- **Discounts:** the average is about 12%. Microwave (14.3%), Laptop (14.1%) and Jacket (13.4%) get the highest average discounts. Cooking Oil (9.4%) and Headphones (10.1%) get the lowest.
- **Relationships:** discount is not visibly related to price or order size.
- **Cleaned data:** 980 of the 1,000 orders were kept, which is 98%.

These results are based on **units sold, not revenue**. The notebook does not yet calculate revenue.

## Data quality issues and limitations

- **Inconsistent city labels.** There are 40 city values because of case and spacing variants such as "PUNE", "jaipur" and " Delhi". City-level analysis is not reliable until these are standardised.
- **Repeated order IDs.** 12 `order_id` values appear more than once, although no row is fully duplicated.
- **Suspicious quantities.** Orders with quantity 0, 50 and 75 need to be checked against the source system.
- **Discount fill.** Missing discounts were filled with 0, which pulls the mean discount down slightly (12.17 before, 11.99 after). Filling with the median is an alternative worth testing.
- **No revenue column.** Revenue and profit were not analysed.

## Next steps

1. Standardise city text, for example with `df["city"] = df["city"].str.title()`.
2. Investigate the repeated `order_id` values and the quantity 0, 50 and 75 rows.
3. Add a revenue column and analyse it by category, month and city:

   ```python
   df["revenue"] = df["quantity"] * df["unit_price"] * (1 - df["discount"] / 100)
   ```

4. Add a monthly time-series analysis using `order_date`.
5. Re-test the discount treatment using the median instead of 0.

## Presentation

`Sales_EDA_Presentation.pptx` has 12 slides: a title slide, the 10-step workflow overview, one slide per step with the code and results, and a closing slide with key findings and next steps.
