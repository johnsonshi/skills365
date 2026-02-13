# KQL Query Language - Best Practices Guide

## Overview

This guide covers query optimization techniques, common patterns, anti-patterns, and performance tips for writing efficient KQL queries in Azure Data Explorer.

---

## Query Optimization Principles

### 1. Reduce Data Volume Early

The single most important optimization is reducing the amount of data processed as early as possible in the query pipeline.

**Order of predicate importance:**
1. **DateTime predicates first** - Use efficient index on datetime columns
2. **String term predicates** - Use `has` operator for indexed term search
3. **Numeric predicates** - Apply selective numeric filters
4. **Column comparisons last** - These force full scans

```kusto
// GOOD: DateTime filter first, then selective predicates
StormEvents
| where StartTime > ago(7d)           // DateTime first
| where State has "TEXAS"             // Indexed term search
| where DamageProperty > 10000        // Numeric filter
| where BeginLocation != EndLocation  // Column comparison last
```

### 2. Filter Selectivity

Place more selective (narrow) filters before less selective (broad) filters:

```kusto
// GOOD: More selective predicate first
StormEvents
| where EventId == 12345              // Very selective (single row)
| where State == "TEXAS"              // Less selective

// BAD: Less selective first
StormEvents
| where State == "TEXAS"              // Processes many rows first
| where EventId == 12345
```

---

## String Operator Best Practices

### Use `has` Instead of `contains`

The `has` operator uses the term index and is significantly faster than `contains`:

| Operator | Uses Index | Performance | Use Case |
|----------|------------|-------------|----------|
| `has` | Yes | Fast | Full token matching |
| `contains` | No | Slow | Substring matching |
| `startswith` | Partial | Medium | Prefix matching |
| `endswith` | No | Slow | Suffix matching |

```kusto
// GOOD: Uses term index
StormEvents
| where EventNarrative has "tornado"

// BAD: Full scan required
StormEvents
| where EventNarrative contains "torn"
```

### Use Case-Sensitive Operators When Possible

Case-sensitive operators are faster:

```kusto
// GOOD: Case-sensitive, faster
| where State == "TEXAS"
| where Column has_cs "Value"
| where Column contains_cs "value"

// SLOWER: Case-insensitive
| where State =~ "texas"
| where Column has "value"
| where Column contains "value"
```

### Use `in` Instead of Multiple `or` Conditions

```kusto
// GOOD: Single in operator
| where State in ("TEXAS", "FLORIDA", "CALIFORNIA")

// BAD: Multiple or conditions
| where State == "TEXAS" or State == "FLORIDA" or State == "CALIFORNIA"
```

---

## Join Optimization

### Place Smaller Table on Left

The smaller table should always be on the left side of the join:

```kusto
// GOOD: Small dimension table on left
DimensionTable  // 1,000 rows
| join FactTable on Key  // 1,000,000 rows

// BAD: Large table on left
FactTable  // 1,000,000 rows
| join DimensionTable on Key  // 1,000 rows
```

### Use `lookup` for Small Dimension Tables

When the right side is small (< tens of MB), use `lookup` instead of `join`:

```kusto
// GOOD: Efficient for small dimensions
FactTable
| lookup DimensionTable on Key

// LESS EFFICIENT for small dimensions
FactTable
| join kind=leftouter DimensionTable on Key
```

### Use `in` Instead of `leftsemi` Join

For filtering by a single column:

```kusto
// GOOD: Use in operator
let FilteredKeys = DimensionTable | where Condition | project Key;
FactTable
| where Key in (FilteredKeys)

// LESS EFFICIENT
FactTable
| join kind=leftsemi (DimensionTable | where Condition) on Key
```

### Use Broadcast Hint for Small Right Tables

When the right table is small (< 100 MB):

```kusto
FactTable
| join hint.strategy=broadcast (SmallDimensionTable) on Key
```

### Use Shuffle for High Cardinality Joins

When both sides are large with high-cardinality keys:

```kusto
LargeTable1
| join hint.shufflekey=Key (LargeTable2) on Key
```

---

## Summarize Optimization

### Use Shuffle for High Cardinality Grouping

When `group by` keys have high cardinality (> 1 million distinct values):

```kusto
StormEvents
| summarize hint.shufflekey=EventId count() by EventId
```

### Avoid Unnecessary Columns in Aggregation

Only reference columns needed for aggregation:

```kusto
// GOOD: Project first to reduce data
StormEvents
| project State, DamageProperty
| summarize TotalDamage = sum(DamageProperty) by State

// BAD: Carries all columns through pipeline
StormEvents
| summarize TotalDamage = sum(DamageProperty) by State
```

---

## Let Statement Best Practices

### Use `materialize()` for Reused Expressions

When a `let` expression is used multiple times, materialize it:

```kusto
// GOOD: Computed once, reused
let RecentEvents = materialize(
    StormEvents
    | where StartTime > ago(7d)
);
RecentEvents
| summarize count() by State
| join (RecentEvents | summarize avg(DamageProperty) by State) on State
```

### Avoid Large `let` Expressions

Break complex queries into smaller `let` statements:

```kusto
// GOOD: Clear, maintainable
let RecentEvents = StormEvents | where StartTime > ago(7d);
let TexasEvents = RecentEvents | where State == "TEXAS";
let HighDamage = TexasEvents | where DamageProperty > 10000;
HighDamage | summarize count()

// BAD: Hard to read and debug
StormEvents
| where StartTime > ago(7d)
| where State == "TEXAS"
| where DamageProperty > 10000
| summarize count()
```

---

## DateTime Best Practices

### Use `datetime` Type, Not `long`

Store dates as `datetime`, not Unix timestamps:

```kusto
// GOOD: Native datetime
| where Timestamp > ago(1d)

// BAD: Requires conversion
| where unixtime_milliseconds_todatetime(TimestampMs) > ago(1d)
```

### Use `bin()` for Time Aggregation

```kusto
// GOOD: Efficient binning
StormEvents
| summarize count() by bin(StartTime, 1h)

// BAD: Manual calculation
StormEvents
| extend Hour = datetime_part("hour", StartTime)
| summarize count() by Hour
```

### Use `ago()` and `now()` for Relative Time

```kusto
// GOOD: Relative time
| where StartTime > ago(7d)
| where StartTime between (ago(30d) .. now())

// BAD: Hard-coded dates (stale quickly)
| where StartTime > datetime(2024-01-01)
```

---

## Dynamic Column Best Practices

### Pre-filter Before JSON Parsing

Use `has` to filter before accessing dynamic properties:

```kusto
// GOOD: Filter first, then parse
| where DynamicColumn has "targetValue"
| where DynamicColumn.property == "targetValue"

// BAD: Parses JSON for every row
| where DynamicColumn.property == "targetValue"
```

### Materialize Dynamic Properties at Ingestion

Use update policies to extract frequently-queried dynamic properties:

```kusto
// Better: Query pre-extracted column
| where ExtractedProperty == "value"

// Slower: Parse at query time
| where DynamicColumn.property == "value"
```

---

## Common Anti-Patterns

### Anti-Pattern 1: Using `contains` for Token Search

```kusto
// BAD
| where Message contains "error"

// GOOD
| where Message has "error"
```

### Anti-Pattern 2: Using `*` for Column Search

```kusto
// BAD: Searches all columns
| where * has "value"

// GOOD: Search specific column
| where TargetColumn has "value"
```

### Anti-Pattern 3: Using `tolower()` for Case-Insensitive Comparison

```kusto
// BAD: Transforms every value
| where tolower(State) == "texas"

// GOOD: Use case-insensitive operator
| where State =~ "texas"
```

### Anti-Pattern 4: Filtering on Calculated Columns

```kusto
// BAD: Filter on calculated column
| extend Duration = EndTime - StartTime
| where Duration > 1h

// GOOD: Filter on original columns
| where EndTime - StartTime > 1h
```

### Anti-Pattern 5: Unbounded Queries

```kusto
// BAD: No time filter, no limit
StormEvents
| project State, EventType

// GOOD: Always limit initial exploration
StormEvents
| where StartTime > ago(1d)  // or
| take 100                    // or
| count
```

### Anti-Pattern 6: Multiple `extract()` Statements

```kusto
// BAD: Multiple regex extractions
| extend Field1 = extract(@"field1=(\w+)", 1, Message)
| extend Field2 = extract(@"field2=(\w+)", 1, Message)
| extend Field3 = extract(@"field3=(\w+)", 1, Message)

// GOOD: Single parse operation
| parse Message with * "field1=" Field1 " field2=" Field2 " field3=" Field3
```

### Anti-Pattern 7: Using Qualified Names When Not Needed

```kusto
// BAD: Unnecessary qualification
cluster("mycluster.kusto.windows.net").database("mydb").MyTable

// GOOD: Unqualified when in same context
MyTable
```

---

## Performance Monitoring Tips

### Use `count` to Test Query Scope

Before running full queries, check the row count:

```kusto
StormEvents
| where StartTime > ago(7d)
| count
```

### Use `take` for Query Development

```kusto
StormEvents
| where StartTime > ago(7d)
| take 10  // Test with small sample
```

### Check Query Statistics

Examine query execution statistics in the results pane to identify bottlenecks.

---

## Materialized Views

Use materialized views for frequently-run aggregation queries:

```kusto
// Query materialized view directly
materialized_view('DailyStormCounts')

// Use materialized_view() function to ensure only materialized data
StormEvents
| summarize count() by bin(StartTime, 1d)
// vs.
materialized_view('DailyStormCounts')  // Pre-computed, faster
```

---

## Query Patterns Summary

| Pattern | When to Use | Example |
|---------|-------------|---------|
| Filter early | Always | `where` before `extend` |
| Use `has` | Token matching | `has "error"` not `contains "error"` |
| Case-sensitive | When data is consistent | `==` not `=~` |
| `in` operator | Multiple value matching | `in ("a","b","c")` |
| Small table left | All joins | Small `join` Large |
| `lookup` | Small dimension tables | `lookup DimTable` |
| `materialize()` | Reused `let` expressions | `materialize(...)` |
| `bin()` | Time aggregation | `bin(Time, 1h)` |
| `take`/`limit` | Development/exploration | `take 100` |

---

## Performance Checklist

Before running production queries:

- [ ] DateTime filter applied first
- [ ] Using `has` instead of `contains` where possible
- [ ] Using case-sensitive operators where possible
- [ ] Smaller table on left side of join
- [ ] `materialize()` for reused expressions
- [ ] No unbounded queries
- [ ] Using `bin()` for time aggregation
- [ ] Filtering before extending
- [ ] Using `lookup` for small dimensions
- [ ] Using appropriate join hints for large tables

---

## Document Information

- **Generated**: 2026-02-13
- **Source**: Azure Data Explorer documentation
- **Phase**: 3 - In-depth Investigation
