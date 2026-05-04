---
layout: post
title: "My First Post"
date: 2026-05-04
---

## Introduction
In this post, I explore how to clean messy CSV data using SQL.

## The Problem
The dataset had:
- Missing values
- Duplicate records
- Inconsistent formatting

## The Solution

```sql
SELECT DISTINCT *
FROM raw_data
WHERE column IS NOT NULL;
