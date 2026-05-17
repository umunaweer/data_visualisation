---
layout: post
title: "Part 5: Scheduling & Automating Your Dashboard"
date: 2026-05-18
categories: [data-visualisation, beginner]
tags: [python, automation, scheduling, cron, github-actions, plotly]
---

In Part 4 we built a sales dashboard using Plotly. But running a script manually every time you want fresh data gets old fast. In this post we'll make the dashboard **refresh itself automatically** — on a schedule, without you touching anything.

---

## The Problem with Manual Dashboards

Every time your data updates, you have to:

1. Pull the latest data
2. Run the Python script
3. Re-export the HTML file
4. Share it again

This is fine once. It's painful daily. Automation fixes this.

---

## Your Options at a Glance

| Method | Best for | Requires |
|---|---|---|
| **cron** (Mac/Linux) | Local machine, scheduled runs | Terminal access |
| **Task Scheduler** | Windows local machine | Windows |
| **GitHub Actions** | Cloud, free, no server needed | GitHub repo |
| **Airflow** | Complex pipelines, teams | Server / Docker |

We'll cover the two most practical ones for beginners: **cron** for local use and **GitHub Actions** for cloud automation.

---

## Option 1: Automate Locally with cron (Mac / Linux)

cron is a built-in scheduler on Mac and Linux. You tell it "run this script every day at 8am" and it does exactly that.

### Step 1: Make your script self-contained

Save your dashboard script as `run_dashboard.py`. Make sure it reads data, builds the chart, and saves the output all in one go:

```python
import pandas as pd
import plotly.express as px
from datetime import datetime

# Load latest data
df = pd.read_csv("clean_sales.csv")
df["sale_date"] = pd.to_datetime(df["sale_date"])
df["sale_month"] = df["sale_date"].dt.to_period("M").astype(str)

# Build chart
monthly = df.groupby("sale_month")["sale_amount"].sum().reset_index()
fig = px.line(monthly, x="sale_month", y="sale_amount", title="Monthly Revenue")

# Save with timestamp so you keep a history
timestamp = datetime.now().strftime("%Y-%m-%d")
fig.write_html(f"dashboard_{timestamp}.html")
fig.write_html("dashboard_latest.html")  # always overwrites the "latest" copy

print(f"✓ Dashboard updated: {timestamp}")
```

### Step 2: Open your cron editor

```bash
crontab -e
```

### Step 3: Add a schedule

```bash
# Run every day at 8:00am
0 8 * * * /usr/bin/python3 /Users/yourname/projects/run_dashboard.py

# Run every Monday at 9:00am
0 9 * * 1 /usr/bin/python3 /Users/yourname/projects/run_dashboard.py

# Run every hour
0 * * * * /usr/bin/python3 /Users/yourname/projects/run_dashboard.py
```

**cron format explained:**

```
┌─ minute  (0–59)
│ ┌─ hour    (0–23)
│ │ ┌─ day of month (1–31)
│ │ │ ┌─ month (1–12)
│ │ │ │ ┌─ day of week (0=Sun, 1=Mon ... 6=Sat)
│ │ │ │ │
0 8 * * *   →  every day at 8:00am
0 9 * * 1   →  every Monday at 9:00am
*/30 * * * * →  every 30 minutes
```

> 💡 Use [crontab.guru](https://crontab.guru) to check your cron expression in plain English before saving it.

### Step 4: Verify it's saved

```bash
crontab -l
```

That's it — your script will now run on schedule as long as your machine is on.

---

## Option 2: Automate on Windows with Task Scheduler

### Step 1: Open Task Scheduler
Press `Win + S`, search for **Task Scheduler**, open it.

### Step 2: Create a new task
Click **"Create Basic Task"** → give it a name like `Dashboard Refresh`.

### Step 3: Set the trigger
Choose **Daily** (or Weekly, etc.) and set your preferred time.

### Step 4: Set the action
- Action: **Start a program**
- Program: path to your Python executable, e.g. `C:\Python311\python.exe`
- Arguments: full path to your script, e.g. `C:\projects\run_dashboard.py`

### Step 5: Finish and test
Click **Finish**, then right-click the task and choose **Run** to test it immediately.

---

## Option 3: Automate in the Cloud with GitHub Actions (Recommended)

This is the most powerful option for beginners — it runs in GitHub's cloud for free, no server required, and works even when your laptop is off.

### Step 1: Add your script to your repo

Make sure `run_dashboard.py` and `clean_sales.csv` are committed to your GitHub repo.

### Step 2: Create the workflow file

In your repo, create this file at exactly this path:

```
.github/workflows/update-dashboard.yml
```

Paste this content:

```yaml
name: Update Dashboard

on:
  schedule:
    - cron: "0 8 * * *"   # every day at 8am UTC
  workflow_dispatch:        # also allow manual trigger

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repo
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install pandas plotly

      - name: Run dashboard script
        run: python run_dashboard.py

      - name: Commit updated dashboard
        run: |
          git config --global user.name "github-actions"
          git config --global user.email "actions@github.com"
          git add dashboard_latest.html
          git commit -m "Auto-update dashboard $(date +'%Y-%m-%d')" || echo "No changes"
          git push
```

### Step 3: Commit the workflow file

Once committed, GitHub Actions will:
- Run your script every day at 8am UTC automatically
- Commit the refreshed `dashboard_latest.html` back to your repo
- You can also trigger it manually from the **Actions** tab any time

### Step 4: Watch it run

Go to the **Actions** tab in your repo. You'll see the workflow listed. Click it to watch each step run live.

---

## Pulling Fresh Data Automatically

Automation is only useful if the **data itself is also fresh**. A few common approaches:

**From a CSV that gets updated regularly:**
```python
# If your CSV lives in a shared folder or is downloaded from a URL
import pandas as pd
df = pd.read_csv("https://example.com/data/sales.csv")
```

**From a database:**
```python
import pandas as pd
import sqlalchemy

engine = sqlalchemy.create_engine("postgresql://user:password@host/dbname")
df = pd.read_sql("SELECT * FROM sales WHERE sale_date >= CURRENT_DATE - 30", engine)
```

**From a Google Sheet (public):**
```python
sheet_id = "your_google_sheet_id"
url = f"https://docs.google.com/spreadsheets/d/{sheet_id}/export?format=csv"
df = pd.read_csv(url)
```

---

## Logging: Know When Something Goes Wrong

Add basic logging to your script so you can see what happened and when:

```python
import logging
from datetime import datetime

logging.basicConfig(
    filename="dashboard.log",
    level=logging.INFO,
    format="%(asctime)s — %(message)s"
)

logging.info("Dashboard run started")

try:
    # your dashboard code here
    logging.info(f"Dashboard updated successfully: {datetime.now().date()}")
except Exception as e:
    logging.error(f"Dashboard failed: {e}")
    raise
```

Check `dashboard.log` any time to see a history of every run.

---

## Quick Reference Cheat Sheet

| Task | Tool | Key command / file |
|---|---|---|
| Schedule on Mac/Linux | cron | `crontab -e` |
| Schedule on Windows | Task Scheduler | GUI — set program + time |
| Schedule in cloud | GitHub Actions | `.github/workflows/update-dashboard.yml` |
| Read data from URL | pandas | `pd.read_csv("https://...")` |
| Read from database | sqlalchemy | `pd.read_sql(query, engine)` |
| Save dashboard | plotly | `fig.write_html("dashboard.html")` |
| Add logging | logging | `logging.basicConfig(filename="app.log")` |
