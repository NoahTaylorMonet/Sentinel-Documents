# 30 Minute KQL Learning Session

> **Purpose:** This document provides a practical 30-minute session outline for teaching Kusto Query Language (KQL) in a Microsoft Sentinel or Log Analytics context. It is designed for live walkthroughs, short workshops, or team enablement sessions.

---

## Session Goal

By the end of this session, the audience should be able to:

- Understand the core KQL pipeline model.
- Read and explain a basic KQL query.
- Write simple investigation and hunting queries.
- Use the most common operators with confidence.
- Avoid the most common beginner mistakes.

---

## Recommended Audience

This session is best for:

- SOC analysts
- Security engineers
- Cloud engineers supporting Sentinel
- Technical stakeholders who need a practical KQL foundation

---

## Core Message

The most important concept to teach is this:

`Get Data | Filter | Shape | Summarize | Sort | Visualize`

KQL is easiest to learn when it is treated like a pipeline. Each line takes the output of the previous line, transforms it, and passes it forward.


## Session Flow

## 1. What KQL Is

Kusto Query Language, or KQL, is the language used across Microsoft Sentinel, Azure Monitor, Log Analytics, Azure Data Explorer, and Defender. It is the primary way to investigate data, hunt for activity, summarize patterns, and build detection logic.

### Key point to emphasize

You do not need to learn all of KQL to be productive. A small set of operators handles most day-to-day investigation work.

---

## 2. The KQL Mental Model


Every KQL query starts with a table and then moves through a pipeline. The most common pattern is:

1. Start with data.
2. Filter to what matters.
3. Keep or create the columns you care about.
4. Summarize or group if needed.
5. Sort, rank, or visualize the result.

### Example pipeline

```kusto
SigninLogs
| where TimeGenerated >= ago(7d)
| where ResultType != 0
| summarize FailedSignIns = count() by UserPrincipalName
| top 10 by FailedSignIns desc
```

### Teaching point

Read the query top to bottom. Each line narrows or reshapes the data.

---

## 3. Live Demo Structure

For the live demo, use a single table throughout if possible. Recommended order:

- `SigninLogs`
- `SecurityEvent`
- `OfficeActivity`

`SigninLogs` is usually the cleanest table for a beginner session because it is easy to explain and supports good examples for filtering, summarization, and trend analysis.

---

## 4. Demo Walkthrough

## Step 1: Start by inspecting the table

```kusto
SigninLogs
| take 10
```

### What to say

- Every query starts with a table.
- `take` is useful when you want to understand the schema.
- Before writing a complex query, inspect the available columns.

---

## Step 2: Add a time filter

```kusto
SigninLogs
| where TimeGenerated >= ago(24h)
| sort by TimeGenerated desc
| take 20
```

### What to say

- Most useful queries start with a time filter.
- Filtering early improves performance and keeps results relevant.
- `ago(24h)` is one of the most useful KQL patterns to remember.

---

## Step 3: Reduce noise with `project`

```kusto
SigninLogs
| where TimeGenerated >= ago(24h)
| project TimeGenerated, UserPrincipalName, AppDisplayName, IPAddress, ResultType
| sort by TimeGenerated desc
| take 20
```

### What to say

- `project` keeps only the columns you need.
- It makes query output easier to read and easier to present.
- This is one of the best habits for beginners.

---

## Step 4: Ask a more specific question with `where`

```kusto
SigninLogs
| where TimeGenerated >= ago(24h)
| where ResultType != 0
| project TimeGenerated, UserPrincipalName, AppDisplayName, IPAddress, ResultType
| sort by TimeGenerated desc
```

### What to say

- This is the point where the query becomes investigative.
- You are no longer browsing data. You are asking a question.
- Stack multiple `where` statements when that improves readability.

---

## Step 5: Summarize a pattern

```kusto
SigninLogs
| where TimeGenerated >= ago(7d)
| summarize SignInCount = count() by AppDisplayName
| top 10 by SignInCount desc
```

### What to say

- `summarize` is one of the most important KQL operators.
- It groups rows and calculates aggregates.
- After `summarize`, only grouped and aggregated columns remain.

### Important warning

This is a common beginner mistake: trying to use a column after `summarize` that no longer exists in the result set.

---

## Step 6: Trend over time with `bin()`

```kusto
SigninLogs
| where TimeGenerated >= ago(7d)
| where ResultType != 0
| summarize FailedSignIns = count() by bin(TimeGenerated, 1d)
| render timechart
```

### What to say

- `bin()` groups time into buckets.
- If you want trends, think `summarize` plus `bin()`.
- `render timechart` turns the result into a chart when supported by the query experience.

---

## Step 7: Create calculated values with `extend`

```kusto
SigninLogs
| where TimeGenerated >= ago(24h)
| extend City = tostring(LocationDetails.city),
         Country = tostring(LocationDetails.countryOrRegion),
         Location = strcat(City, ", ", Country)
| project TimeGenerated, UserPrincipalName, IPAddress, Location, ResultType
| take 20
```

### What to say

- `extend` adds calculated columns.
- Use it when the raw schema is technically correct but not friendly for analysis.
- This is how you turn raw data into readable context.

---

## Step 8: Make the query reusable with `let`

```kusto
let Lookback = 7d;
SigninLogs
| where TimeGenerated >= ago(Lookback)
| where ResultType != 0
| summarize FailedSignIns = count() by UserPrincipalName
| top 10 by FailedSignIns desc
```

### What to say

- `let` is useful for parameters and reusable subqueries.
- It makes your queries easier to maintain.
- If a value is likely to change, avoid hardcoding it multiple times.

---

## Step 9: Show one useful `join`

```kusto
let Failed =
SigninLogs
| where TimeGenerated >= ago(24h)
| where ResultType != 0
| summarize FailedCount = count() by UserPrincipalName;

let Success =
SigninLogs
| where TimeGenerated >= ago(24h)
| where ResultType == 0
| summarize SuccessCount = count() by UserPrincipalName;

Failed
| join kind=leftouter Success on UserPrincipalName
| extend SuccessCount = coalesce(SuccessCount, 0)
| top 10 by FailedCount desc
```

### What to say

- `join` lets you correlate data sets.
- Keep the first `join` example simple.
- Most beginners should learn `where`, `project`, `extend`, and `summarize` before relying on joins.

---

## 5. Key Operators to Teach

| Operator | Purpose | Why it matters |
|---|---|---|
| `where` | Filter rows | Most common operator in investigations |
| `project` | Keep or rename columns | Reduces noise and improves readability |
| `extend` | Add calculated columns | Creates analysis-friendly fields |
| `summarize` | Aggregate and group data | Identifies trends and counts |
| `sort by` | Order results | Makes outputs readable |
| `top` | Return the top N rows | Fast ranking by a metric |
| `distinct` | Return unique values | Good for exploration |
| `let` | Define variables or subqueries | Makes queries reusable |
| `join` | Correlate multiple data sets | Useful when one table is not enough |
| `render` | Visualize results | Good for reporting and storytelling |

---

## 6. Tips and Tricks

These are useful to call out explicitly:

- Start with `take 10` when you do not know the table well.
- Put `where TimeGenerated >= ago(...)` near the top of the query.
- Use `project` before presenting results to others.
- Use `top` for rankings and `sort by` for full ordering.
- Use `let` for lookback windows, usernames, IP addresses, and reusable logic.
- Build queries incrementally instead of writing the full query in one shot.
- Comment queries with `//` when the logic becomes complex.
- Use `distinct` when you only need to know what unique values exist.
- Use `countif()` inside `summarize` when you need conditional totals.
- Use time bucketing with `bin()` when looking for patterns over time.

---

## 7. Common Mistakes

These mistakes are worth highlighting because nearly every beginner makes them:

- Forgetting the time filter.
- Trying to write a complex query before exploring the schema.
- Keeping too many columns in the result set.
- Forgetting that `summarize` creates a new table shape.
- Jumping to `join` before a single-table query is understood.
- Not validating the query in smaller steps.
- Using broad text matching when a more precise filter would work better.

---

## 8. Presenter Notes

### Good pacing strategy

Use this pattern during the demo:

1. Show the simplest version of the query.
2. Explain what changed on the next line.
3. Run the query again.
4. Ask what the audience now knows that they did not know before.

### Good teaching habit

Narrate the query as a pipeline rather than explaining syntax in isolation.

Example:

- Start with sign-in logs.
- Filter to the last 24 hours.
- Keep the fields we care about.
- Limit to failures.
- Summarize by user.
- Rank by volume.

That pattern helps people understand why the query is written the way it is.

---

## 9. Closing



KQL gets easier once you stop treating it like a language to memorize and start treating it like a pipeline to reason through. Most real-world queries start with a table, filter aggressively, shape the output, summarize the pattern, and only then add correlation or visualization if needed.

---

## 10. Optional One-Slide Cheat Sheet

If you want a quick reference slide, use this:

- `where` filters rows.
- `project` keeps the fields you care about.
- `extend` creates new fields.
- `summarize` groups and aggregates data.
- `sort by` orders results.
- `top` ranks results.
- `distinct` shows unique values.
- `let` makes queries reusable.
- `join` correlates data sets.
- `render` visualizes data.

---

## 11. Prep Checklist Before Delivering the Session

- Confirm which demo table has recent data in your environment.
- Test every query before the session.
- Keep a backup table ready in case your primary table is sparse.
- Pre-save the demo queries so you can recover quickly if the session is interrupted.
- Decide whether the session is investigation-focused, hunting-focused, or analytics-focused.

---

## 12. Suggested Follow-On Sessions

If this session lands well, natural follow-up topics include:

- Multi-table KQL with `join`, `union`, and subqueries
- Parsing data with `parse`, `extract`, and dynamic fields
- Building Sentinel detections from KQL
- KQL performance tuning and query optimization
- Visualization, workbooks, and dashboards
