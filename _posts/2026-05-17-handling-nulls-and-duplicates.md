
---
layout: post
title: "Part 2: Handling Nulls & Duplicates in Python and SQL"
date: 2026-05-17
categories: [data-cleaning, beginner]
tags: [python, sql, pandas, nulls, duplicates]
---

NULL values and duplicate rows are the two most common data problems you'll face. Here's how to find them, understand them, and fix them — with working code for both pandas and SQL.

---

## Part A: Handling NULL / NaN Values

A **null value** means the data is missing. In Python/pandas it shows up as `NaN`. In SQL it's `NULL`. They can silently destroy your analysis — `AVG()` ignores nulls in SQL, `df.mean()` skips `NaN` in pandas, and joins can quietly drop rows that have nulls in key columns.

### Step 1: Find the nulls

**Python:**

```python
import pandas as pd

df = pd.read_csv("customers.csv")

# Count nulls per column
print(df.isnull().sum())

# See the percentage of nulls per column
null_pct = (df.isnull().sum() / len(df) * 100).round(2)
print(null_pct)

# View rows where a specific column is null
nulls = df[df["customer_email"].isnull()]
print(nulls.head())
```

**SQL:**

```sql
-- Count nulls in the email column
SELECT
  COUNT(*) AS total_rows,
  COUNT(customer_email) AS non_null_emails,
  COUNT(*) - COUNT(customer_email) AS null_emails
FROM customers;

-- See the actual null rows
SELECT *
FROM customers
WHERE customer_email IS NULL
LIMIT 10;
```

> 💡 In SQL, `COUNT(column_name)` only counts non-null values. `COUNT(*)` counts everything. Subtracting them gives you the null count.

---

### Step 2: Decide what to do with nulls

There's no one-size-fits-all answer. Use this as your guide:

| Situation | Best action |
|---|---|
| Column is optional (e.g. phone number) | Leave it as null — it's fine |
| Numeric column, null means 0 (e.g. no discount) | Fill with `0` |
| Numeric column, null means unknown | Fill with median |
| Text column, small % null | Fill with `"Unknown"` or most common value |
| Key column (ID, required field) is null | Drop the row |
| More than 50% of column is null | Consider dropping the column |

---

### Step 3: Fix the nulls

**Python:**

```python
# Drop rows where a critical column is null
df_clean = df.dropna(subset=["customer_id", "order_id"])

# Fill nulls with a fixed value
df["country"] = df["country"].fillna("Unknown")
df["discount"] = df["discount"].fillna(0)

# Fill numeric nulls with the column median
df["age"] = df["age"].fillna(df["age"].median())

# Forward fill — useful for time series
# Fills each null with the previous row's value
df["price"] = df["price"].ffill()

# Confirm result
print(df.isnull().sum())
```

**SQL:**

```sql
-- COALESCE: replace nulls in query output (non-destructive)
SELECT
  customer_id,
  COALESCE(customer_email, 'no_email@unknown.com') AS email,
  COALESCE(discount, 0) AS discount
FROM customers;

-- UPDATE: permanently fix nulls in the table
UPDATE customers
SET country = 'Unknown'
WHERE country IS NULL;

-- DELETE: remove rows with null in a critical column
DELETE FROM customers
WHERE customer_id IS NULL;
```

> ⚠️ **Before running UPDATE or DELETE:** always run a `SELECT` first to see which rows will be affected. Work on a copy of the table in production until you're confident.

---

## Part B: Removing Duplicate Rows

Duplicates happen when the same record appears in your dataset more than once. They can silently double-count revenue, users, or events — and you often won't notice until something looks too good to be true.

### Step 1: Find duplicates

**Python:**

```python
# Count total duplicate rows
print(df.duplicated().sum())

# See the actual duplicate rows
dupes = df[df.duplicated(keep=False)]
print(dupes.sort_values("order_id").head(10))

# Check duplicates on specific columns only
dupes_by_id = df[df.duplicated(subset=["order_id"], keep=False)]
print(dupes_by_id)
```

**SQL:**

```sql
-- Find duplicate order IDs
SELECT order_id, COUNT(*) AS occurrences
FROM sales
GROUP BY order_id
HAVING COUNT(*) > 1
ORDER BY occurrences DESC;
```

---

### Step 2: Remove duplicates

**Python:**

```python
# Remove all fully duplicate rows (keep first occurrence)
df_clean = df.drop_duplicates()

# Remove duplicates based on a specific column
df_clean = df.drop_duplicates(subset=["order_id"], keep="first")

# Confirm the result
print(f"Before: {len(df)} rows")
print(f"After:  {len(df_clean)} rows")
print(f"Removed: {len(df) - len(df_clean)} duplicates")
```

**SQL:**

```sql
-- Delete duplicates, keeping only the row with the lowest id
DELETE FROM sales
WHERE id NOT IN (
  SELECT MIN(id)
  FROM sales
  GROUP BY order_id
);

-- Alternative: use ROW_NUMBER() — works in most modern databases
WITH ranked AS (
  SELECT *,
    ROW_NUMBER() OVER (
      PARTITION BY order_id
      ORDER BY created_at ASC  -- keep the earliest
    ) AS row_num
  FROM sales
)
SELECT * FROM ranked
WHERE row_num = 1;
```

> ✅ The `ROW_NUMBER()` approach is the most flexible. Change `ORDER BY created_at ASC` to `DESC` to keep the most recent row instead of the oldest.

---

## Putting It Together: Full Python Script

```python
import pandas as pd

df = pd.read_csv("raw_sales.csv")
print(f"Original shape: {df.shape}")

# Remove duplicates
df = df.drop_duplicates(subset=["order_id"], keep="first")
print(f"After dedup: {df.shape}")

# Drop rows where required fields are null
df = df.dropna(subset=["order_id", "customer_id"])

# Fill optional fields
df["discount"] = df["discount"].fillna(0)
df["country"]  = df["country"].fillna("Unknown")
df["age"]      = df["age"].fillna(df["age"].median())

print(f"Final shape: {df.shape}")
print(f"Nulls remaining:\n{df.isnull().sum()}")

df.to_csv("clean_sales.csv", index=False)
print("✓ Saved to clean_sales.csv")
```
