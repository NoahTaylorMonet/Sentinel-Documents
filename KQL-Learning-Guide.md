# KQL Learning Guide for SOC Analysts

A complete, progressive guide to Kusto Query Language (KQL) for analysts working in **Microsoft Sentinel** and **Microsoft Defender XDR (Advanced Hunting)**.

**How to use this document**
1. Work through the **Skilling Script** section-by-section — each block runs standalone and builds on the last. Every section has an instructor **BLURB** (what to say) and a **NEW IN THIS SECTION** callout explaining every new operator/function.
2. Keep the **Top 10 Operators** section as a quick reference.
3. Use the **`join` Kinds Deep Dive** when correlation gets confusing — it's the section that separates new analysts from fluent ones.

---

## Table of Contents
1. [KQL Skilling Script (Sections 0–18)](#kql-skilling-script)
2. [Top 10 KQL Operators Every SOC Analyst Should Know](#top-10-kql-operators-every-soc-analyst-should-know)
3. [KQL `join` Kinds — Complete Guide with Examples](#kql-join-kinds--complete-guide-with-examples)

---

# KQL Skilling Script

A progressive, copy-paste-runnable tutorial. Each section builds on the last. Run section-by-section in **Sentinel → Logs** or **Defender XDR → Advanced hunting**.

```kusto
// =====================================================================
//  KQL SKILLING SCRIPT FOR SOC ANALYSTS
//  BLURB  = what to say before running the block
//  NEW    = every new operator/function introduced in that block
// =====================================================================


// ---------------------------------------------------------------------
// 0. Orient yourself — what tables exist?
// ---------------------------------------------------------------------
// BLURB:
// Before writing queries, you need to know what data you have. Every log
// source lands in a "table" — SigninLogs, DeviceProcessEvents, etc.
// This section shows how to list tables that have data in the last day.
// Skip `search *` in production — it scans everything and is slow.
//
// NEW IN THIS SECTION:
//   search *        Scans every column of every table for a match. Powerful but
//                   expensive — treat as read-only emergency tool.
//   take N          Returns N arbitrary rows (NOT sorted). Great for "peek" queries.
//   union *         Merges multiple tables into one result stream. `withsource=`
//                   adds a column telling you which table each row came from.
//   ago(1d)         Returns "now minus 1 day." Used with TimeGenerated for filters.
//   where           Row filter — keeps rows that match the condition.
//   summarize       Aggregates rows into groups (like SQL GROUP BY).
//   count()         Counts rows per group.
//   by              Specifies the grouping column(s) for summarize.
//   order by ... desc   Sorts results; desc = largest first.

search *
| take 1

union withsource=TableName *
| where TimeGenerated > ago(1d)
| summarize Count=count() by TableName
| order by Count desc


// ---------------------------------------------------------------------
// 1. The shape of a KQL query: Tabular pipeline
// ---------------------------------------------------------------------
// BLURB:
// KQL reads left-to-right like a conveyor belt. You start with a table,
// then pipe ( | ) the results through operators that filter, transform,
// or summarize. Each operator hands its output to the next.
//
// NEW IN THIS SECTION:
//   |               The pipe. Sends the result of the left side into the right side.
//   <TableName>     Starting a query with a bare table name means "scan this table."

SigninLogs
| take 10


// ---------------------------------------------------------------------
// 2. Always filter time FIRST
// ---------------------------------------------------------------------
// BLURB:
// `TimeGenerated` is the one column that's indexed in every Sentinel
// table. Filtering on it first makes queries fast; skipping it makes
// them slow and expensive.
//
// NEW IN THIS SECTION:
//   TimeGenerated   Universal timestamp column in Sentinel tables.
//   > ago(1h)       Comparison against a relative time.
//   datetime(...)   Literal timestamp in ISO format.
//   between (a..b)  Range test; inclusive of both ends. `..` is the range operator.

SigninLogs
| where TimeGenerated > ago(1h)
| take 10

SigninLogs
| where TimeGenerated between (datetime(2026-04-19 00:00) .. datetime(2026-04-20 00:00))
| take 10


// ---------------------------------------------------------------------
// 3. where — filtering rows
// ---------------------------------------------------------------------
// BLURB:
// `where` narrows down to what matters. Favor fast operators: `has`,
// `startswith`, `in()`. Only use `contains` or regex when you must.
//
// NEW IN THIS SECTION:
//   !=              Not equal.
//   has             Token match — finds a whole word in the field. FAST (indexed).
//   !has            Negated token match.
//   contains        Substring match anywhere in the field. Slower, not indexed.
//   startswith / endswith   Prefix / suffix match.
//   matches regex   Full regular-expression match. Slow; last resort.
//   in (...) / !in (...)    Membership test against a list of values.

SigninLogs
| where TimeGenerated > ago(24h)
| where ResultType != 0
| where UserPrincipalName has "@contoso.com"
| take 50


// ---------------------------------------------------------------------
// 4. project / project-away / extend — choose & create columns
// ---------------------------------------------------------------------
// BLURB:
// `project` picks the columns you want (like SQL SELECT). `extend` adds
// computed columns. Projecting early makes results readable and queries
// faster.
//
// NEW IN THIS SECTION:
//   project         Keeps only the listed columns. You can rename with `NewName=OldName`.
//   project-away    Removes the listed columns; keeps everything else.
//   extend          Adds a new computed column without removing existing ones.

SigninLogs
| where TimeGenerated > ago(1h)
| project TimeGenerated, UserPrincipalName, IPAddress, ResultType, AppDisplayName
| extend IsFailure = ResultType != 0
| take 20


// ---------------------------------------------------------------------
// 5. summarize — the single most important operator
// ---------------------------------------------------------------------
// BLURB:
// If you learn one KQL operator well, make it this one. `summarize` is
// GROUP BY + aggregation. 90% of detections and hunts boil down to
// "count X by Y and flag the outliers." Get comfortable with `count()`,
// `countif()`, `dcount()` (distinct count), `min()`, `max()` — they'll
// cover most of what you do.
//
// NEW IN THIS SECTION:
//   countif(cond)   Counts only rows where the condition is true.
//   dcount(col)     Distinct-count of values in a column (approximate, fast).
//   Named aggregates   `FailureCount = countif(...)` names the output column.
//   Multiple aggregates in one summarize, separated by commas.

SigninLogs
| where TimeGenerated > ago(24h)
| summarize FailureCount = countif(ResultType != 0),
            Total        = count(),
            DistinctIPs  = dcount(IPAddress)
          by UserPrincipalName
| where FailureCount > 10
| order by FailureCount desc


// ---------------------------------------------------------------------
// 6. bin() — time-bucketing for trends
// ---------------------------------------------------------------------
// BLURB:
// `bin(TimeGenerated, 1h)` rounds timestamps down to the nearest hour
// so you can count per hour. Combine with `render timechart` for an
// instant visualization.
//
// NEW IN THIS SECTION:
//   bin(col, size)  Rounds a numeric or datetime value down to a bucket size.
//                   Works with timespans (1h, 5m, 1d) and numbers.
//   render          Chooses a visualization for the result set.
//   timechart       A time-series line chart; first column is time, rest are values.

SigninLogs
| where TimeGenerated > ago(7d)
| summarize Signins = count() by bin(TimeGenerated, 1h)
| render timechart


// ---------------------------------------------------------------------
// 7. Multi-dimension summarize + render
// ---------------------------------------------------------------------
// BLURB:
// Same as section 6, but grouping by two things: time AND country.
// Perfect for "which country spiked at 2am on Saturday?"
//
// NEW IN THIS SECTION:
//   Multiple `by` columns   Summarize can group by several columns at once.
//   tostring(expr)          Converts a dynamic/JSON value to a string so you can
//                           group/sort/compare it like normal text.
//   Dot-access (.field)     Reads a property out of a dynamic (JSON) column.

SigninLogs
| where TimeGenerated > ago(7d)
| summarize Failures = countif(ResultType != 0)
          by bin(TimeGenerated, 1h),
             Country = tostring(LocationDetails.countryOrRegion)
| render timechart


// ---------------------------------------------------------------------
// 8. let — variables / reusable snippets
// ---------------------------------------------------------------------
// BLURB:
// `let` names values and sub-queries at the top of your query for
// readability, reuse, and sometimes performance. Think of it like
// constants + functions for KQL.
//
// NEW IN THIS SECTION:
//   let             Declares a named scalar, list, table, or function.
//   dynamic([...])  Creates an inline JSON-style array or object literal.
//   in (suspectCountries)   Membership against a dynamic array.

let lookback   = 7d;
let failThresh = 20;
let suspectCountries = dynamic(["RU","KP","IR","CN"]);
SigninLogs
| where TimeGenerated > ago(lookback)
| where ResultType != 0
| where tostring(LocationDetails.countryOrRegion) in (suspectCountries)
| summarize Failures=count()
          by UserPrincipalName,
             Country=tostring(LocationDetails.countryOrRegion)
| where Failures >= failThresh
| order by Failures desc


// ---------------------------------------------------------------------
// 9. Parsing nested fields: dynamic columns, tostring
// ---------------------------------------------------------------------
// BLURB:
// Many tables store rich data inside JSON objects. KQL exposes those as
// dynamic columns you dot into, then wrap in `tostring()` for normal
// string behavior. If a column renders as `{…}`, this is how you crack
// it open.
//
// NEW IN THIS SECTION:
//   Dynamic columns   Columns whose values are JSON objects or arrays.
//   parse_json(str)   Turns a string of JSON into a dynamic value (not shown here
//                     but worth mentioning — used when JSON arrives as a string).
//   Multiple extends in one statement, comma-separated.

SigninLogs
| where TimeGenerated > ago(1h)
| extend City    = tostring(LocationDetails.city),
         Country = tostring(LocationDetails.countryOrRegion),
         DeviceOS= tostring(DeviceDetail.operatingSystem)
| project TimeGenerated, UserPrincipalName, IPAddress, City, Country, DeviceOS
| take 50


// ---------------------------------------------------------------------
// 10. join — correlating across tables
// ---------------------------------------------------------------------
// BLURB:
// `join` turns "something suspicious happened" into a full story. The
// example is password-spray: many failures then a success. Mention
// `leftanti` — "left rows NOT in the right table" — which is powerful
// for finding missing events.
//
// NEW IN THIS SECTION:
//   join kind=inner     Keeps only rows that match on both sides (default).
//        kind=leftouter Keeps all left rows, right columns null if no match.
//        kind=leftanti  Keeps left rows with NO match on the right.
//        kind=leftsemi  Keeps left rows that DO have a match, but no right columns.
//   on col1, col2       Columns to match across sides. If names differ use
//                       `on $left.A == $right.B`.
//   min() / max()       Aggregates — earliest / latest value in a group.
//   any(col)            Returns an arbitrary sample value from the group.
//   Sub-queries via let Named mini-tables used on both sides of the join.

let window = 1h;
let fails =
    SigninLogs
    | where TimeGenerated > ago(1d) and ResultType != 0
    | project FailTime=TimeGenerated, UserPrincipalName, IPAddress;
let wins =
    SigninLogs
    | where TimeGenerated > ago(1d) and ResultType == 0
    | project WinTime=TimeGenerated, UserPrincipalName, IPAddress;
fails
| join kind=inner wins on UserPrincipalName, IPAddress
| where WinTime between (FailTime .. FailTime + window)
| summarize FailsBefore=count(), FirstFail=min(FailTime), SuccessAt=any(WinTime)
          by UserPrincipalName, IPAddress
| where FailsBefore >= 10
| order by FailsBefore desc


// ---------------------------------------------------------------------
// 11. union — combining similar tables
// ---------------------------------------------------------------------
// BLURB:
// Defender XDR splits device telemetry across many tables. `union`
// stitches them back together so you can see a device's full timeline.
//
// NEW IN THIS SECTION:
//   union T1, T2, T3    Stacks rows from multiple tables. Columns are merged;
//                       missing columns become null on the rows that lack them.
//   coalesce(a, b, c)   Returns the first non-null argument. Useful to pick a
//                       single "best" column from tables with different schemas.
//   order by ... asc    Sort ascending (useful for timelines).

union DeviceProcessEvents, DeviceNetworkEvents, DeviceFileEvents
| where Timestamp > ago(1h)
| where DeviceName == "victim-01"
| project Timestamp, DeviceName, ActionType,
          FileName=coalesce(FileName, InitiatingProcessFileName),
          RemoteUrl, RemoteIP
| order by Timestamp asc


// ---------------------------------------------------------------------
// 12. Windowing with prev() / next() — detect sequences
// ---------------------------------------------------------------------
// BLURB:
// Sometimes a single event isn't suspicious — the sequence is. `prev()`
// lets you compare the current row to the one before it after sorting.
//
// NEW IN THIS SECTION:
//   serialize              Freezes row order so window functions produce stable
//                          results. Required before prev()/next().
//   prev(col [, N])        Value from the row N positions earlier (default 1).
//   next(col [, N])        Same, but forward.
//   datetime_diff(unit,a,b)  Difference between two timestamps in the given unit
//                            ('minute','second','hour', etc.).

SigninLogs
| where TimeGenerated > ago(1d) and ResultType == 0
| extend City = tostring(LocationDetails.city)
| order by UserPrincipalName asc, TimeGenerated asc
| serialize
| extend PrevCity = prev(City),
         PrevUser = prev(UserPrincipalName),
         PrevTime = prev(TimeGenerated)
| where UserPrincipalName == PrevUser and City != PrevCity
| extend MinutesBetween = datetime_diff('minute', TimeGenerated, PrevTime)
| where MinutesBetween < 60
| project TimeGenerated, UserPrincipalName, PrevCity, City, MinutesBetween


// ---------------------------------------------------------------------
// 13. make_set / make_list / arg_max — summarize patterns per entity
// ---------------------------------------------------------------------
// BLURB:
// These aggregations remember, not just count. Perfect for building a
// behavior profile per user or device.
//
// NEW IN THIS SECTION:
//   make_set(col, N)       Collects up to N distinct values into a dynamic array.
//   make_list(col, N)      Like make_set but keeps duplicates and order.
//   arg_max(col, *)        Returns the full row with the maximum value of `col`.
//                          Use `*` to bring back every column of that row.
//   arg_min(col, *)        Mirror of arg_max — earliest / smallest.

SigninLogs
| where TimeGenerated > ago(7d)
| summarize IPs       = make_set(IPAddress, 50),
            Apps      = make_set(AppDisplayName, 50),
            FirstSeen = min(TimeGenerated),
            LastSeen  = max(TimeGenerated),
            LastEvent = arg_max(TimeGenerated, *)
          by UserPrincipalName


// ---------------------------------------------------------------------
// 14. externaldata / watchlists — bring your own context
// ---------------------------------------------------------------------
// BLURB:
// Real detections need context not in the logs: VIP lists, known-good
// subnets, vendor IPs, exception lists. Watchlists (Sentinel) and inline
// `datatable()` let you bring that context in. The `ipv4_lookup` example
// filters OUT your own IP space in a clean, subnet-aware way.
//
// NEW IN THIS SECTION:
//   _GetWatchlist('name')          Sentinel function that returns a watchlist as a table.
//   datatable(col:type) [ ... ]    Inline literal table. Great for small allow/blocklists.
//   evaluate <plugin>              Runs a special plugin. Required for ipv4_lookup.
//   ipv4_lookup(table, ip, col)    Enrichment — joins an IP against a table of CIDRs.
//   return_unmatched=true          Tells ipv4_lookup to keep rows that didn't match,
//                                  with the lookup column null — perfect for "NOT in".
//   isempty(col)                   True if the column is null/empty string.
//   externaldata (alluded to)      Reads a CSV/JSON from a URL on the fly. Powerful
//                                  for pulling IOC feeds directly into a query.

// Watchlist (Sentinel):
// let vips = _GetWatchlist('VIP_Users') | project UserPrincipalName;
// SigninLogs | where UserPrincipalName in (vips)

// Inline allowlist:
let corpSubnets = datatable(network:string) [
    "10.0.0.0/8",
    "172.16.0.0/12",
    "192.168.0.0/16"
];
SigninLogs
| where TimeGenerated > ago(1h)
| evaluate ipv4_lookup(corpSubnets, IPAddress, network, return_unmatched=true)
| where isempty(network)                    // keep only NON-corporate IPs


// ---------------------------------------------------------------------
// 15. Performance rules of thumb
// ---------------------------------------------------------------------
// BLURB:
// Reinforce good habits early. Fast queries = cheaper Sentinel bills and
// scheduled rules that actually finish on time.
//
// NEW IN THIS SECTION:
//   (No new operators — this is a checklist section.)

// 1. Filter TimeGenerated first.
// 2. Prefer `has` over `contains`.
// 3. Push `where` as early as possible; project only needed columns.
// 4. Avoid `search *` in prod.
// 5. Use `summarize` before `join` when possible (shrink the left side).
// 6. `join kind=inner` with small left table; big table on the right.
// 7. Use `take`/`limit` while developing; remove before saving.


// ---------------------------------------------------------------------
// 16. Putting it together — a mini "threat hunt" pattern
// ---------------------------------------------------------------------
// BLURB:
// Real hunts follow a shape: baseline normal, find deviations, pivot on
// outcome. This block combines almost every operator so far.
//
// NEW IN THIS SECTION:
//   Chained `let` tables          Build a baseline as a named table, then use it
//                                 downstream. Keeps queries modular and readable.
//   `in (subquery)`               The right side of `in` can be a projected column
//                                 from another let-table.

let lookback = 7d;
let rareCountryThreshold = 5;
let baseline =
    SigninLogs
    | where TimeGenerated between (ago(30d) .. ago(lookback))
    | summarize HistSignins = count()
              by Country = tostring(LocationDetails.countryOrRegion);
let rareCountries =
    baseline | where HistSignins < rareCountryThreshold | project Country;
SigninLogs
| where TimeGenerated > ago(lookback)
| extend Country = tostring(LocationDetails.countryOrRegion)
| where Country in (rareCountries)
| summarize Fails   = countif(ResultType != 0),
            Success = countif(ResultType == 0),
            IPs     = make_set(IPAddress, 20),
            FirstSeen = min(TimeGenerated),
            LastSeen  = max(TimeGenerated)
          by UserPrincipalName, Country
| where Fails >= 10 and Success >= 1
| order by Fails desc


// ---------------------------------------------------------------------
// 17. Hardening your queries for analytics rules
// ---------------------------------------------------------------------
// BLURB:
// A query that works in Logs isn't automatically a good scheduled rule.
// Rules need a stable TimeGenerated, entity columns (AccountUpn,
// IPAddress, DeviceName, FileHashes), and must finish within the
// lookback window.
//
// NEW IN THIS SECTION:
//   Column renaming inside summarize   `AccountUpn = UserPrincipalName` aligns
//                                      with Sentinel's entity-mapping expectations.
//   `extend TimeGenerated = LastFail`  Sentinel scheduled rules require a
//                                      TimeGenerated column on the final result.
//   Entity columns (concept)           AccountUpn, IPAddress, DeviceName, FileHashes,
//                                      Url, etc. — map these so alerts link to
//                                      real entities in the incident.

SigninLogs
| where TimeGenerated > ago(1h)
| where ResultType != 0
| summarize Fails=count(), FirstFail=min(TimeGenerated), LastFail=max(TimeGenerated)
          by AccountUpn=UserPrincipalName, IPAddress
| where Fails >= 25
| extend TimeGenerated = LastFail
| project TimeGenerated, AccountUpn, IPAddress, Fails, FirstFail, LastFail


// ---------------------------------------------------------------------
// 18. Self-study challenges
// ---------------------------------------------------------------------
// BLURB:
// No answer keys on purpose. Pair up, try, then compare. Treat as weekly
// homework; review together in a short standup.
//
// NEW IN THIS SECTION (introduced via the challenges):
//   has_any (dynamic)     Fast token match against any value in a list.
//   has_all (dynamic)     Match ALL tokens in a list (less common, but useful).
//   countif + total       Pattern for computing a rate:
//                           Rate = 1.0 * countif(cond) / count()
//   Time-bound joins      Using `between` on joined timestamps to bound the
//                         correlation window.

// A. List the top 10 apps by sign-in failure rate over the last 7 days.
// B. Find users who signed in from 3+ distinct countries in 24 hours.
// C. In Defender, find processes that spawned `powershell.exe` with
//    encoded command arguments in the last 24 hours.
// D. Correlate DeviceLogonEvents failures with DeviceProcessEvents
//    launches of `net.exe` within 10 minutes on the same device.
// E. Rewrite query C using `has_any` and `matches regex`; compare runtimes.


// ---------------------------------------------------------------------
//  CLOSING THOUGHT
// ---------------------------------------------------------------------
// KQL isn't a separate skill from "being a SOC analyst" — it *is* the
// skill. You don't need to memorize every operator; you need to
// recognize the *shape* of what you want and pick the operator that fits.
// That only comes from running queries every day.
```

---

# Top 10 KQL Operators Every SOC Analyst Should Know

Ranked by how often you'll actually use them on the job.

---

## 1. `where` — filter rows
**What it does:** Keeps only the rows that match a condition. Everything else is discarded before the next stage of the pipeline.
**Why it matters:** It's the cheapest way to shrink a dataset. Always filter on `TimeGenerated` first — that column is indexed, so it cuts the data volume before any other work happens.

```kusto
SigninLogs
| where TimeGenerated > ago(1h) and ResultType != 0
```

---

## 2. `summarize` — group and aggregate
**What it does:** Collapses many rows into fewer rows by grouping on one or more columns and running aggregate functions (`count()`, `dcount()`, `countif()`, `min()`, `max()`, etc.). Think SQL `GROUP BY` on steroids.
**Why it matters:** 90% of detections and hunts boil down to *"count X by Y and flag the outliers."* Once you're fluent with `summarize`, you can answer most analyst questions in one query.

```kusto
SigninLogs
| where TimeGenerated > ago(24h) and ResultType != 0
| summarize Fails=count(), IPs=dcount(IPAddress) by UserPrincipalName
| where Fails > 20
```

---

## 3. `project` / `extend` — shape your output
**What it does:**
- `project` **picks** the columns you want (and can rename them), dropping the rest.
- `project-away` does the opposite — drops specific columns, keeps the rest.
- `extend` **adds** a new computed column without removing any existing ones.

**Why it matters:** Keeps your results readable and your queries fast. Projecting narrow, early in the pipeline, means less data moves through downstream operators.

```kusto
SigninLogs
| project TimeGenerated, UserPrincipalName, IPAddress, ResultType
| extend IsFailure = ResultType != 0
```

---

## 4. `join` — correlate tables
**What it does:** Matches rows between two tables on one or more shared columns, similar to a SQL JOIN. The `kind=` argument controls behavior:
- `inner` — only rows that match on both sides (default).
- `leftouter` — all left rows; right columns are null if there's no match.
- `leftanti` — left rows that have **no** match on the right (great for "what's missing?").
- `leftsemi` — left rows that **do** have a match, but without pulling right-side columns.

**Why it matters:** A single signal is rarely enough. Correlating two tables is how "failed logon" becomes "password spray followed by successful logon" — a real incident.

```kusto
DeviceLogonEvents
| where ActionType == "LogonFailed"
| join kind=inner (DeviceProcessEvents | where FileName =~ "net.exe")
  on DeviceId
```

---

## 5. `union` — stack tables together
**What it does:** Vertically combines rows from multiple tables into one result stream. Columns are merged; missing columns on either side become null. `withsource=` adds a column telling you which table each row came from.
**Why it matters:** Defender XDR splits device telemetry across many tables (`DeviceProcessEvents`, `DeviceNetworkEvents`, `DeviceFileEvents`, etc.). `union` stitches them back together into a single device timeline during investigations.

```kusto
union DeviceProcessEvents, DeviceNetworkEvents, DeviceFileEvents
| where DeviceName == "victim-01" and Timestamp > ago(1h)
```

---

## 6. `bin()` + `render timechart` — see trends over time
**What it does:**
- `bin(col, size)` rounds a number or timestamp down to a bucket (e.g. `bin(TimeGenerated, 1h)` → the start of the hour).
- `render timechart` turns your result into a time-series line chart; the first column is the x-axis (time), the rest become lines.

**Why it matters:** Humans spot anomalies visually — spikes, off-hours activity, gaps — that are invisible in a table of raw rows. This is the one-line path from data to picture.

```kusto
SigninLogs
| where TimeGenerated > ago(7d)
| summarize Signins=count() by bin(TimeGenerated, 1h)
| render timechart
```

---

## 7. `let` — variables and sub-queries
**What it does:** Declares a named value, list, or sub-query at the top of your query. You can `let` a scalar (a threshold), a dynamic array (suspect countries), or a whole table (a baseline) and reuse it downstream.
**Why it matters:** Makes queries readable ("the threshold lives here, edit it once"), reusable, and often faster because named sub-queries can be optimized by the engine.

```kusto
let lookback = 7d;
let threshold = 25;
SigninLogs
| where TimeGenerated > ago(lookback) and ResultType != 0
| summarize Fails=count() by UserPrincipalName
| where Fails >= threshold
```

---

## 8. String operators: `has`, `contains`, `startswith`, `in()`, `matches regex`
**What they do:**
- `has` / `!has` — whole-word (token) match. **Fast**, uses the text index.
- `contains` / `!contains` — substring match anywhere in the field. **Slow**, not indexed.
- `startswith` / `endswith` — prefix / suffix match.
- `in (…)` / `!in (…)` — membership test against a list of values.
- `matches regex` — full regular-expression match. Powerful but slow; last resort.

**Why it matters:** Choosing the right string operator is the difference between a query that finishes in seconds and one that times out. Default to `has` and `in()`; reach for `contains` or regex only when you must.

```kusto
DeviceProcessEvents
| where ProcessCommandLine has "encodedcommand"
| where FileName in~ ("powershell.exe","pwsh.exe")
```

---

## 9. `make_set` / `make_list` / `arg_max` — remember, don't just count
**What they do (used inside `summarize`):**
- `make_set(col, N)` — collects up to N **distinct** values into a dynamic array.
- `make_list(col, N)` — like `make_set` but keeps duplicates and order.
- `arg_max(col, *)` — returns the **entire row** where `col` is at its maximum (use `*` to bring all columns back). `arg_min` is the mirror.

**Why it matters:** These aggregations don't just count — they *remember*. They're how you build per-user or per-device behavior profiles in a single query ("all distinct IPs this user logged in from," "the most recent process this host ran").

```kusto
SigninLogs
| where TimeGenerated > ago(7d)
| summarize IPs=make_set(IPAddress, 50),
            LastEvent=arg_max(TimeGenerated, *)
          by UserPrincipalName
```

---

## 10. Dynamic/JSON parsing: `tostring()`, `parse_json()`, dot-access
**What they do:**
- Dot-access (`col.field.subfield`) reads a property out of a dynamic/JSON column.
- `tostring(expr)` forces a dynamic value to a plain string so you can compare, group, or sort it.
- `parse_json(str)` converts a JSON **string** into a dynamic value you can then dot into.

**Why it matters:** Many of the richest columns in Sentinel and Defender (`LocationDetails`, `DeviceDetail`, `AdditionalFields`, `RawEventData`) are stored as JSON. If you can't open them, you can't hunt in them.

```kusto
SigninLogs
| extend City    = tostring(LocationDetails.city),
         Country = tostring(LocationDetails.countryOrRegion),
         OS      = tostring(DeviceDetail.operatingSystem)
```

---

## Honorable mentions (round out the skillset)
- **`ago()` / `datetime()` / `between`** — time math and explicit time ranges.
- **`count()`, `countif()`, `dcount()`, `min()`, `max()`** — the five aggregates you'll use daily.
- **`evaluate ipv4_lookup`** — subnet-aware IP filtering against a CIDR table.
- **`serialize` + `prev()` / `next()`** — sequence detection (impossible travel, behavior chains).
- **`take` / `limit`** — grab a few rows while developing; remove before saving as a rule.
- **`_GetWatchlist()` / `externaldata`** — pull in outside context (VIP lists, IOC feeds).

> If you can fluently combine `where → project/extend → summarize → join/union → render`, you can write ~95% of the queries a SOC job demands.

---

# KQL `join` Kinds — Complete Guide with Examples

`join` matches rows between two tables using one or more shared columns. The `kind=` argument controls **which rows come back** and **how the columns merge**. Getting this right is the difference between a correct correlation and a quietly wrong one.

> **Syntax reminder**
> ```kusto
> LeftTable
> | join kind=<kind> [hint.strategy=shuffle] (RightTable) on Col1, $left.A == $right.B
> ```
> - The left table is always the one *before* `| join`.
> - The right table is the one in parentheses.
> - If the join columns have the same name on both sides, just list them. Otherwise use `$left.X == $right.Y`.

---

## 1. `innerunique` — **the default in KQL**
**What it does:** For each unique key on the **left**, picks **one** matching row from the right. Duplicates on the left are preserved; duplicates on the right are deduplicated *before* the join.
**When to use it:** Almost never on purpose. It's the default only because it's the fastest. It silently drops right-side duplicates, which can make row counts look wrong in investigations.
**Gotcha:** If you don't specify `kind=`, you get this — and new analysts are often surprised when their counts don't match SQL intuition.

```kusto
// Which failed-logon users ever had ANY process event? (dedup right side)
DeviceLogonEvents
| where ActionType == "LogonFailed"
| join (DeviceProcessEvents | project DeviceId, Timestamp) on DeviceId
```

---

## 2. `inner` — classic inner join
**What it does:** Returns **every** combination of matching left/right rows. If the left has 3 matches and the right has 2, you get 6 rows.
**When to use it:** When you want full correlation — e.g., every failed logon paired with every suspicious process on the same device. This is the most SQL-like behavior.
**Gotcha:** Row counts can explode on high-cardinality keys. Shrink both sides with `where`/`summarize` before joining.

```kusto
// Failed logons correlated with net.exe launches on the same device
DeviceLogonEvents
| where ActionType == "LogonFailed"
| project LogonTime=Timestamp, DeviceId, AccountName
| join kind=inner (
    DeviceProcessEvents
    | where FileName =~ "net.exe"
    | project ProcTime=Timestamp, DeviceId, ProcessCommandLine
  ) on DeviceId
| where ProcTime between (LogonTime .. LogonTime + 10m)
```

---

## 3. `leftouter` — keep all left rows
**What it does:** Returns every row from the left. If there's a match on the right, its columns are populated; if not, they're `null`.
**When to use it:** Enrichment. You want to keep your main dataset intact and tack on extra context where available (e.g., adding user risk level to every sign-in, even for users with no risk record).
**Gotcha:** `null` handling — downstream `where` clauses on enriched columns need `isnotempty()` or `isnull()`.

```kusto
// All sign-ins, enriched with UEBA risk score where we have one
SigninLogs
| where TimeGenerated > ago(1d)
| project TimeGenerated, UserPrincipalName, IPAddress
| join kind=leftouter (
    BehaviorAnalytics
    | project UserPrincipalName, InvestigationPriority
  ) on UserPrincipalName
```

---

## 4. `rightouter` — keep all right rows
**What it does:** Mirror of `leftouter`. Returns every row from the right; left columns are null when no match.
**When to use it:** Honestly, rarely. If you find yourself reaching for this, swap the sides and use `leftouter` instead — easier to read, and the engine optimizes left-biased joins better.

```kusto
// Rarely used — prefer swapping sides and using leftouter
```

---

## 5. `fullouter` — keep everything from both sides
**What it does:** Returns all matches plus all unmatched rows from *both* sides. Unmatched columns become null on whichever side has no partner row.
**When to use it:** Reconciliation — "what's in one data source that isn't in the other, and what's shared?" Useful when comparing two telemetry sources for coverage gaps.
**Gotcha:** Can produce very large result sets. Use after narrowing both sides.

```kusto
// Compare devices reporting to Defender vs. devices in an inventory table
DeviceInfo
| summarize by DeviceName
| join kind=fullouter (
    InventoryTable_CL
    | summarize by DeviceName=Hostname_s
  ) on DeviceName
| extend InDefender = isnotempty(DeviceName), InInventory = isnotempty(DeviceName1)
```

---

## 6. `leftsemi` — "does it exist on the right?" (keep left columns only)
**What it does:** Returns left rows that **have at least one match** on the right, but **only the left table's columns**.
**When to use it:** "Filter the left table by existence in the right." Cleaner and often faster than `| where X in (RightTable | project X)` because the engine can optimize it.
**Gotcha:** You don't get any right-side columns, by design. If you need them, use `inner` instead.

```kusto
// Sign-ins from users who ALSO have Defender device alerts in the last day
SigninLogs
| where TimeGenerated > ago(1d)
| join kind=leftsemi (
    AlertEvidence
    | where TimeGenerated > ago(1d)
    | project UserPrincipalName=AccountUpn
  ) on UserPrincipalName
```

---

## 7. `leftanti` — "what's on the left that's NOT on the right?"
**What it does:** Returns left rows that have **no** match on the right. Only left-side columns.
**When to use it:** Arguably the most **underrated** join for hunting. "Hosts that connected to the VPN but *didn't* phone home to EDR." "Users who signed in but *aren't* in the employee watchlist." Absence is often more suspicious than presence.
**Gotcha:** Make sure the right side is complete for your lookback window — partial data produces false "missing" alerts.

```kusto
// Devices that generated process events but did NOT send heartbeat in the last hour
DeviceProcessEvents
| where Timestamp > ago(1h)
| summarize by DeviceId
| join kind=leftanti (
    DeviceInfo
    | where TimeGenerated > ago(1h)
    | summarize by DeviceId
  ) on DeviceId
```

---

## 8. `rightsemi` / `rightanti` — mirror versions
**What they do:** Same as `leftsemi` / `leftanti` but keep right-side rows instead of left.
**When to use them:** Again, rarely — readability almost always favors flipping the tables and using the `left*` forms.

---

## Special: cross-cluster / cross-workspace joins
Not a `kind=` but worth knowing. You can join against another workspace or cluster using `workspace("name").Table` or `cluster("uri").database("db").Table`.

```kusto
SigninLogs
| join kind=inner (
    workspace("prod-sec").SigninLogs | where ResultType == 0
  ) on UserPrincipalName
```

---

## Performance tips that apply to every join kind
1. **Filter and summarize both sides first.** Smaller tables = faster joins.
2. **Put the smaller table on the left.** KQL optimizes left-biased joins.
3. **Project only the columns you need** on both sides before joining.
4. **Use `hint.strategy=shuffle`** on very large joins with high-cardinality keys:
   ```kusto
   | join hint.strategy=shuffle kind=inner (Right) on DeviceId
   ```
5. **Prefer `leftsemi` / `leftanti` over `| where X in (subquery)`** when you just need existence/absence.
6. **Watch for accidental `innerunique`** (the default) silently de-duplicating.

---

## Cheat sheet

| Kind            | Left rows kept         | Right rows kept         | Right columns returned | Common use                                  |
|-----------------|------------------------|-------------------------|------------------------|---------------------------------------------|
| `innerunique`   | Matches (left dups ok) | 1 per left key          | Yes                    | Default — rarely what you actually want     |
| `inner`         | All matches            | All matches             | Yes                    | Full correlation, classic SQL-style join    |
| `leftouter`     | All                    | Matches                 | Yes (null if no match) | Enrichment                                  |
| `rightouter`    | Matches                | All                     | Yes (null if no match) | Rare — swap sides and use `leftouter`       |
| `fullouter`     | All                    | All                     | Yes (null on either)   | Reconciliation / coverage gap analysis      |
| `leftsemi`      | Matches                | —                       | No                     | "Exists in right?" filter                   |
| `leftanti`      | Non-matches            | —                       | No                     | "NOT in right" — missing/absent events      |
| `rightsemi`     | —                      | Matches                 | Right only             | Rare — swap sides                           |
| `rightanti`     | —                      | Non-matches             | Right only             | Rare — swap sides                           |

> **Rule of thumb:** Default to `inner` for correlation, `leftouter` for enrichment, `leftanti` for "what's missing?" hunts. Those three cover 90% of SOC work.
