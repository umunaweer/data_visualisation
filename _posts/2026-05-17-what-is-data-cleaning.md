---
layout: post
title: "Part 1: What Is Data Cleaning & Why Does It Matter?"
date: 2026-05-17
categories: [data-cleaning, beginner]
tags: [python, sql, pandas]
---

Before you run a single query or train a model, your data is probably lying to you. Here's what dirty data actually looks like — and how to think about fixing it.

---

## The Problem: Real Data Is Messy

Imagine you're handed a CSV file from a client. You open it and see 50,000 rows of customer sales data. You're excited — this is going to be a great analysis. Then you look closer:

- Some rows have a blank `customer_email` column.
- The `sale_date` column has dates formatted as `"2024/03/15"`, `"March 15, 2024"`, and `"15-03-2024"` — all mixed together.
- The `sale_amount` column has values like `"$1,200.00"` stored as text.
- One customer's name is `"JOHN SMITH"` in some rows and `"John Smith"` in others.
- There are 3,000 duplicate rows because someone exported the file twice.

This is **dirty data** — and it's extremely common. Studies estimate that data scientists spend **60–80% of their time** cleaning and preparing data before they can do anything useful with it.

> ⚠️ **Why this matters:** If you run a query or build a report on dirty data, your results will be wrong — even if your code is perfect. Garbage in, garbage out.

---

## What Exactly Is "Dirty Data"?

Dirty data is any data that is inaccurate, incomplete, inconsistent, or duplicated. Here are the main types you'll encounter:

| Problem Type | Example | Impact |
|---|---|---|
| Missing values | `NULL`, empty string, `NaN` | Breaks calculations, skews averages |
| Duplicates | Same row appearing twice | Inflates counts and totals |
| Wrong data type | Date stored as text | Can't sort, filter, or calculate |
| Inconsistent format | "USA", "us", "United States" | Groups fragment, counts split |
| Outliers / errors | Age = 999, salary = -500 | Skews statistics drastically |

---

## Where Does Dirty Data Come From?

Dirty data isn't always someone's fault — it's often just the reality of how data gets collected:

- **Manual data entry** — humans make typos, use different formats, leave things blank.
- **System migrations** — moving data between systems often loses or corrupts formatting.
- **Multiple data sources** — joining data from different teams or tools creates inconsistencies.
- **Sensor / API errors** — devices fail, APIs timeout, and you get nulls or zeroes you didn't expect.
- **Evolving schemas** — a column that used to store phone numbers now stores codes. The old data is still there.

---

## A Simple First Look: Python with pandas

The first thing you should always do with a new dataset is **explore it**. Don't clean it yet — just look.

```python
import pandas as pd

# Load your dataset
df = pd.read_csv("sales_data.csv")

# 1. How big is it?
print(df.shape)           # (rows, columns)

# 2. What does it look like?
print(df.head())          # first 5 rows

# 3. What data types do we have?
print(df.dtypes)          # column types

# 4. How many nulls per column?
print(df.isnull().sum())  # null counts

# 5. Any duplicates?
print(df.duplicated().sum())

# 6. Quick statistics
print(df.describe())      # min, max, mean, etc.
```

When you run `df.describe()`, check if **min values** make sense (can age really be 0 or negative?), and if **max values** are realistic. These outliers often reveal data entry errors.

---

## A Simple First Look: SQL

If your data lives in a database, run these on any new table:

```sql
-- 1. How many rows?
SELECT COUNT(*) AS total_rows
FROM sales_data;

-- 2. How many nulls in the email column?
SELECT COUNT(*) AS null_emails
FROM sales_data
WHERE customer_email IS NULL;

-- 3. How many duplicate rows?
SELECT COUNT(*) - COUNT(DISTINCT order_id) AS duplicates
FROM sales_data;

-- 4. Check for impossible values
SELECT MIN(sale_amount), MAX(sale_amount), AVG(sale_amount)
FROM sales_data;

-- 5. See all unique values in a category column
SELECT DISTINCT country, COUNT(*) AS count
FROM sales_data
GROUP BY country
ORDER BY count DESC;
```

That last query is especially useful. If you see `"US"`, `"USA"`, `"United States"`, and `"united states"` all as separate countries — you've found an inconsistency problem to fix.

---

## The Data Cleaning Mindset

Before you start fixing things, ask yourself these questions for each problem you find:

1. **Is this actually a problem?** — A null in an optional phone number field might be fine. A null in a required order ID is not.
2. **Can I fix it, or should I drop it?** — Sometimes you can fill in missing values. Other times the row is just unusable.
3. **What does this affect downstream?** — Will this null break a JOIN? Will this duplicate inflate a revenue total?
4. **Document every change.** — Keep a log of what you changed and why.

> ✅ **Key takeaway:** Data cleaning is not just a technical task — it's a decision-making process. You need to understand the data's context and business rules before you start changing anything.

---

## What's Coming in This Series

- **Part 2:** Handling nulls and removing duplicates — in Python and SQL.
- **Part 3:** Fixing data types, standardizing dates, cleaning text columns, and dealing with inconsistent categories.
