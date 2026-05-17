---
layout: post
title: "Part 4: Building Your First Dashboard — Turning Clean Data into Insights"
date: 2026-05-18
categories: [data-visualisation, beginner]
tags: [python, sql, pandas, matplotlib, plotly, dashboard]
---

You've cleaned your data, fixed your types, and removed duplicates. Now comes the rewarding part — turning that clean data into a dashboard that tells a story at a glance.

---

## What Makes a Good Dashboard?

A dashboard is only useful if it answers specific questions quickly. Before writing a single line of code, ask yourself:

- **Who is reading this?** (a manager, a client, yourself?)
- **What decisions does it need to support?**
- **What are the 3–5 most important numbers to show?**

A common mistake is cramming too much into one dashboard. Less is more — focus on the metrics that matter.

| Good dashboard | Bad dashboard |
|---|---|
| 3–5 key metrics, clearly labelled | 20+ charts with no clear focus |
| Consistent colours with meaning | Random colours everywhere |
| Filters to drill down | Static, one-size-fits-all view |
| Answers a specific question | Shows everything, answers nothing |

---

## 1. Choose Your Tool

Different tools suit different situations:

| Tool | Best for | Skill level |
|---|---|---|
| **Matplotlib** | Quick static charts in Python | Beginner |
| **Plotly** | Interactive charts in Python | Beginner–Intermediate |
| **Plotly Dash** | Full web dashboards in Python | Intermediate |
| **Power BI** | Business dashboards, no code | Beginner |
| **Tableau** | Visual, drag-and-drop dashboards | Beginner–Intermediate |
| **Metabase** | SQL-based dashboards, open source | Intermediate |

For this post we'll use **Plotly** — it's free, runs in Python, and produces interactive charts with very little code.

---

## 2. Start with Clean Data

We'll use a simple sales dataset. Make sure it's already cleaned (see Parts 1–3 of this series):

```python
import pandas as pd

df = pd.read_csv("clean_sales.csv")
df["sale_date"] = pd.to_datetime(df["sale_date"])
df["sale_month"] = df["sale_date"].dt.to_period("M").astype(str)

print(df.head())
print(df.shape)
```

Expected output:
```
   order_id  customer_id  sale_amount sale_month  country
0      1001         C001       120.00    2026-01      USA
1      1002         C002        85.50    2026-01      GBR
...
(500, 5)
```

---

## 3. Build the Key Charts

### Chart 1 — Monthly Revenue (Line Chart)

```python
import plotly.express as px

monthly = df.groupby("sale_month")["sale_amount"].sum().reset_index()
monthly.columns = ["Month", "Total Revenue"]

fig = px.line(
    monthly,
    x="Month",
    y="Total Revenue",
    title="Monthly Revenue",
    markers=True
)
fig.update_layout(xaxis_tickangle=-45)
fig.show()
```

> 💡 Use a line chart for trends over time — it makes rises and dips immediately visible.

---

### Chart 2 — Revenue by Country (Bar Chart)

```python
by_country = df.groupby("country")["sale_amount"].sum().reset_index()
by_country = by_country.sort_values("sale_amount", ascending=False)

fig = px.bar(
    by_country,
    x="country",
    y="sale_amount",
    title="Revenue by Country",
    color="sale_amount",
    color_continuous_scale="Blues"
)
fig.show()
```

---

### Chart 3 — Top 10 Customers (Horizontal Bar)

```python
top_customers = (
    df.groupby("customer_id")["sale_amount"]
    .sum()
    .sort_values(ascending=False)
    .head(10)
    .reset_index()
)

fig = px.bar(
    top_customers,
    x="sale_amount",
    y="customer_id",
    orientation="h",
    title="Top 10 Customers by Revenue",
    color="sale_amount",
    color_continuous_scale="Greens"
)
fig.update_layout(yaxis={"categoryorder": "total ascending"})
fig.show()
```

---

### Chart 4 — Sales Distribution (Histogram)

```python
fig = px.histogram(
    df,
    x="sale_amount",
    nbins=30,
    title="Distribution of Sale Amounts",
    color_discrete_sequence=["steelblue"]
)
fig.show()
```

> 💡 A histogram tells you if most sales are small with a few large ones (right-skewed), or evenly spread. This shapes how you set targets and segment customers.

---

## 4. Combine into a Simple Dashboard

Using Plotly's `make_subplots`, you can combine all charts into one view:

```python
from plotly.subplots import make_subplots
import plotly.graph_objects as go

fig = make_subplots(
    rows=2, cols=2,
    subplot_titles=(
        "Monthly Revenue",
        "Revenue by Country",
        "Top 10 Customers",
        "Sale Amount Distribution"
    )
)

# Monthly revenue line
monthly_trace = go.Scatter(
    x=monthly["Month"],
    y=monthly["Total Revenue"],
    mode="lines+markers",
    name="Monthly Revenue"
)

# Revenue by country bar
country_trace = go.Bar(
    x=by_country["country"],
    y=by_country["sale_amount"],
    name="By Country"
)

# Top customers horizontal bar
customer_trace = go.Bar(
    x=top_customers["sale_amount"],
    y=top_customers["customer_id"],
    orientation="h",
    name="Top Customers"
)

# Sale amount histogram
hist_trace = go.Histogram(
    x=df["sale_amount"],
    nbinsx=30,
    name="Distribution"
)

fig.add_trace(monthly_trace,   row=1, col=1)
fig.add_trace(country_trace,   row=1, col=2)
fig.add_trace(customer_trace,  row=2, col=1)
fig.add_trace(hist_trace,      row=2, col=2)

fig.update_layout(
    height=800,
    title_text="Sales Dashboard",
    showlegend=False
)

fig.show()

# Save as a standalone HTML file
fig.write_html("sales_dashboard.html")
print("✓ Dashboard saved as sales_dashboard.html")
```

---

## 5. Add KPI Summary Cards

Every good dashboard starts with the big numbers — total revenue, number of orders, average order value:

```python
total_revenue   = df["sale_amount"].sum()
total_orders    = df["order_id"].nunique()
avg_order_value = df["sale_amount"].mean()
top_country     = df.groupby("country")["sale_amount"].sum().idxmax()

print("=" * 40)
print(f"  Total Revenue:     ${total_revenue:,.2f}")
print(f"  Total Orders:      {total_orders:,}")
print(f"  Avg Order Value:   ${avg_order_value:,.2f}")
print(f"  Top Country:       {top_country}")
print("=" * 40)
```

---

## 6. Dashboard Design Tips

**Colour** — use one colour palette consistently. Don't use red unless something is actually bad.

**Labels** — always label axes and add a chart title. Never make the reader guess what they're looking at.

**Order** — sort bar charts by value (highest to lowest) unless there's a reason not to.

**Interactivity** — Plotly charts are interactive by default (hover, zoom, filter). Use this to your advantage instead of building 10 static charts.

**Export** — use `fig.write_html("dashboard.html")` to share a fully interactive dashboard as a single file anyone can open in a browser.

---

## Quick Reference Cheat Sheet

| Chart type | Use when | Plotly function |
|---|---|---|
| Line chart | Trend over time | `px.line()` |
| Bar chart | Compare categories | `px.bar()` |
| Histogram | Show distribution | `px.histogram()` |
| Scatter plot | Show relationship between two numbers | `px.scatter()` |
| Pie / Donut | Show proportion (use sparingly) | `px.pie()` |
| Box plot | Show spread and outliers | `px.box()` |

---

## What's Next?

In the next part of this series we'll look at **scheduling and automating your dashboard** — so it refreshes with new data automatically, without you having to run the script manually every time.
