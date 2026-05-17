---
layout: post
title: "Part 3: Fixing Data Types, Formats & Inconsistent Values"
date: 2026-05-17
categories: [data-cleaning, beginner]
tags: [python, sql, pandas, data-types, formatting]
---

Dates stored as text. Prices with dollar signs. "Yes", "y", "TRUE", and "1" all meaning the same thing. Data type chaos is everywhere — here's how to tame it using Python and SQL.

---

## Why Data Types Matter

Imagine you import a CSV and `sale_date` is loaded as a string. Everything looks fine — until you try to sort by date, filter for "last 30 days", or calculate days between orders. It all breaks, because text doesn't know what a date means.

The same goes for numbers stored as text (`"$1,200"`), booleans stored inconsistently (`"Yes"`, `"TRUE"`, `"1"`), and categories with mixed capitalisation or spelling.

| Column | Raw value | What you need |
|---|---|---|
| `sale_date` | `"2024-03-15"` (string) | `datetime` |
| `sale_amount` | `"$1,200.00"` (string) | `float` |
| `is_active` | `"Yes"` / `"y"` / `"1"` | `True` / `False` |
| `country` | `"USA"`, `"usa"`, `"U.S.A"` | `"USA"` (consistent) |
| `customer_name` | `"JOHN SMITH"`, `"john smith"` | `"John Smith"` |

---

## 1. Fixing Dates

Dates are the most common data type problem. Convert them to a proper datetime type so you can sort, filter, and do date arithmetic.

**Python:**

```python
import pandas as pd

# pandas is usually smart enough to detect the format
df["sale_date"] = pd.to_datetime(df["sale_date"])

# If the format is ambiguous, specify it explicitly
# %Y = 4-digit year, %m = month, %d = day
df["sale_date"] = pd.to_datetime(df["sale_date"], format="%d/%m/%Y")

# If some values can't be parsed, use errors='coerce'
# Unparseable values become NaT instead of crashing the script
df["sale_date"] = pd.to_datetime(df["sale_date"], errors="coerce")

# Now you can do date arithmetic
df["days_since_sale"] = (pd.Timestamp("today") - df["sale_date"]).dt.days
df["sale_year"]  = df["sale_date"].dt.year
df["sale_month"] = df["sale_date"].dt.month
```

**SQL:**

```sql
-- Cast a text column to a date in a query
SELECT
  order_id,
  CAST(sale_date_text AS DATE) AS sale_date
FROM sales;

-- PostgreSQL: use TO_DATE for custom formats
SELECT TO_DATE(sale_date_text, 'DD/MM/YYYY') AS sale_date
FROM sales;

-- Permanently change the column type (PostgreSQL)
ALTER TABLE sales
ALTER COLUMN sale_date TYPE DATE
  USING TO_DATE(sale_date, 'YYYY-MM-DD');

-- Calculate date differences
SELECT
  order_id,
  CURRENT_DATE - sale_date AS days_since_sale
FROM sales;
```

---

## 2. Cleaning Numeric Columns Stored as Text

This happens constantly with money — values like `"$1,200.00"` are strings and need to become actual numbers.

**Python:**

```python
# Remove $ and commas, then convert to float
df["sale_amount"] = (
    df["sale_amount"]
    .str.replace("$", "", regex=False)
    .str.replace(",", "", regex=False)
    .str.strip()
    .astype(float)
)

# Safer version: bad values become NaN instead of crashing
df["sale_amount"] = pd.to_numeric(
    df["sale_amount"].str.replace(r"[$,]", "", regex=True),
    errors="coerce"
)
```

**SQL:**

```sql
-- Remove $ and commas, then cast to numeric
SELECT
  order_id,
  CAST(
    REPLACE(REPLACE(sale_amount_text, '$', ''), ',', '')
    AS DECIMAL(10, 2)
  ) AS sale_amount
FROM sales;
```

---

## 3. Standardising Text Columns

Text columns attract inconsistency — different capitalisations, extra spaces, abbreviations.

**Python:**

```python
# Standardise capitalisation
df["customer_name"] = df["customer_name"].str.title()  # "john smith" → "John Smith"
df["country"]       = df["country"].str.upper()        # "usa" → "USA"
df["email"]         = df["email"].str.lower()          # "JOHN@..." → "john@..."

# Remove leading/trailing whitespace (very common)
df["customer_name"] = df["customer_name"].str.strip()

# Map multiple variants to one standard value
country_map = {
    "usa": "USA", "us": "USA", "united states": "USA",
    "uk": "GBR",  "england": "GBR", "great britain": "GBR",
}
df["country"] = df["country"].str.lower().map(country_map).fillna(df["country"])
```

> 💡 Apply `.lower()` first before mapping so you only need lowercase keys in your dictionary — no need to list every capitalisation variant.

**SQL:**

```sql
-- Standardise capitalisation in a query
SELECT
  UPPER(country)         AS country,
  LOWER(email)           AS email,
  INITCAP(customer_name) AS customer_name,  -- PostgreSQL
  TRIM(customer_name)    AS name_trimmed
FROM customers;

-- Fix specific inconsistent values with CASE WHEN
SELECT
  order_id,
  CASE
    WHEN LOWER(country) IN ('usa', 'us', 'united states') THEN 'USA'
    WHEN LOWER(country) IN ('uk', 'england', 'great britain') THEN 'GBR'
    ELSE UPPER(country)
  END AS country_clean
FROM sales;

-- Permanently update the table
UPDATE customers
SET country =
  CASE
    WHEN LOWER(country) IN ('usa', 'us', 'united states') THEN 'USA'
    WHEN LOWER(country) IN ('uk', 'england')              THEN 'GBR'
    ELSE UPPER(country)
  END;
```

---

## 4. Fixing Boolean / Flag Columns

A boolean column should have exactly two values: true or false. In practice you'll find `"Yes"`, `"No"`, `"Y"`, `"N"`, `"1"`, `"0"`, `"TRUE"`, `"FALSE"`, and sometimes `None` all in the same column.

**Python:**

```python
true_values  = {"yes", "y", "true", "t", "1", "1.0"}
false_values = {"no",  "n", "false", "f", "0", "0.0"}

df["is_active"] = df["is_active"].apply(
    lambda x:
        True  if str(x).strip().lower() in true_values  else
        False if str(x).strip().lower() in false_values else
        None  # unknown values become null
)
```

**SQL:**

```sql
UPDATE customers
SET is_active =
  CASE
    WHEN LOWER(is_active) IN ('yes', 'y', 'true', '1') THEN 1
    WHEN LOWER(is_active) IN ('no',  'n', 'false', '0') THEN 0
    ELSE NULL
  END;
```

---

## 5. The Full Pipeline

A complete script that covers everything from this series — nulls, duplicates, types, and formatting:

```python
import pandas as pd

# Load
df = pd.read_csv("raw_sales.csv")
print(f"Loaded: {df.shape[0]} rows, {df.shape[1]} columns")

# 1. Remove duplicates
before = len(df)
df = df.drop_duplicates(subset=["order_id"], keep="first")
print(f"Removed {before - len(df)} duplicate rows")

# 2. Fix nulls
df = df.dropna(subset=["order_id", "customer_id"])
df["discount"] = df["discount"].fillna(0)
df["country"]  = df["country"].fillna("Unknown")

# 3. Fix data types
df["sale_date"] = pd.to_datetime(df["sale_date"], errors="coerce")
df["sale_amount"] = pd.to_numeric(
    df["sale_amount"].str.replace(r"[$,]", "", regex=True),
    errors="coerce"
)

# 4. Standardise text
df["customer_name"] = df["customer_name"].str.strip().str.title()
df["email"]         = df["email"].str.strip().str.lower()

country_map = {"usa": "USA", "us": "USA", "uk": "GBR", "england": "GBR"}
df["country"] = df["country"].str.lower().map(country_map).fillna(df["country"])

# 5. Validate and save
print(f"\nFinal shape: {df.shape}")
print(f"Remaining nulls:\n{df.isnull().sum()}")
df.to_csv("clean_sales.csv", index=False)
print("\n✓ Saved clean_sales.csv")
```

---

## Quick Reference Cheat Sheet

| Task | Python (pandas) | SQL |
|---|---|---|
| Find nulls | `df.isnull().sum()` | `WHERE col IS NULL` |
| Drop null rows | `df.dropna(subset=[...])` | `DELETE WHERE col IS NULL` |
| Fill nulls | `df[col].fillna(val)` | `COALESCE(col, val)` |
| Find duplicates | `df.duplicated().sum()` | `GROUP BY ... HAVING COUNT(*) > 1` |
| Remove duplicates | `df.drop_duplicates()` | `ROW_NUMBER() OVER (PARTITION BY ...)` |
| Fix dates | `pd.to_datetime(col)` | `CAST(col AS DATE)` |
| Fix numbers | `pd.to_numeric(col)` | `CAST(col AS DECIMAL)` |
| Uppercase text | `col.str.upper()` | `UPPER(col)` |
| Strip spaces | `col.str.strip()` | `TRIM(col)` |
| Replace values | `col.map({dict})` | `CASE WHEN ... THEN ... END` |
