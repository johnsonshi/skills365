# KQL Query Language Fundamentals - Reference Guide

## Overview

Kusto Query Language (KQL) is a powerful, read-only query language used in Azure Data Explorer, Azure Monitor, Microsoft Sentinel, and Microsoft Fabric. KQL is designed for exploring large volumes of data using a pipe-based syntax where each operator transforms data and passes it to the next.

---

## Query Structure

KQL queries follow a pipe (`|`) syntax where data flows from left to right:

```kusto
TableName
| operator1
| operator2
| operator3
```

Key structural elements:
- **Table reference**: Starting point for data
- **Pipe operator (`|`)**: Chains operations together
- **Operators**: Transform, filter, or aggregate data
- **Expressions**: Scalar or tabular expressions within operators

---

## Data Types

KQL supports the following scalar data types:

| Type | Aliases | Description | Example |
|------|---------|-------------|---------|
| `bool` | `boolean` | True or false | `true`, `false` |
| `datetime` | `date` | Date and time instant | `datetime(2024-01-15 10:30:00)` |
| `decimal` | - | 128-bit decimal number | `decimal(3.14159265358979)` |
| `dynamic` | - | JSON-like array or property bag | `dynamic({"key": "value"})` |
| `guid` | `uuid`, `uniqueid` | 128-bit unique identifier | `guid(12345678-1234-1234-1234-123456789012)` |
| `int` | - | 32-bit signed integer | `42` |
| `long` | - | 64-bit signed integer | `9223372036854775807` |
| `real` | `double` | 64-bit floating-point | `3.14159` |
| `string` | - | Unicode text | `"Hello World"` |
| `timespan` | `time` | Time duration | `1h`, `30m`, `5d` |

### Timespan Literals

| Unit | Suffix | Example |
|------|--------|---------|
| Days | `d` | `7d` |
| Hours | `h` | `24h` |
| Minutes | `m` | `30m` |
| Seconds | `s` | `60s` |
| Milliseconds | `ms` | `500ms` |
| Microseconds | `microsecond` | `100microsecond` |
| Ticks | `tick` | `1000tick` |

---

## Top 20 Tabular Operators

### 1. `where` - Filter Rows

Filters rows based on a predicate expression.

```kusto
StormEvents
| where State == "TEXAS"
| where DamageProperty > 10000
| where StartTime > ago(30d)
```

**Key Points:**
- `where` and `filter` are equivalent
- Place datetime predicates first for best performance
- Use `has` instead of `contains` for full-token matching

### 2. `project` - Select/Rename Columns

Selects columns to include, rename, or create computed columns.

```kusto
StormEvents
| project EventId, State, EventType, TotalDamage = DamageProperty + DamageCrops
```

**Variants:**
- `project-away`: Remove specific columns
- `project-keep`: Keep matching columns
- `project-rename`: Rename without reordering
- `project-reorder`: Change column order

### 3. `extend` - Add Calculated Columns

Adds new calculated columns while keeping existing ones.

```kusto
StormEvents
| extend Duration = EndTime - StartTime
| extend DayOfWeek = dayofweek(StartTime)
```

### 4. `summarize` - Aggregate Data

Groups rows and calculates aggregations.

```kusto
StormEvents
| summarize
    TotalEvents = count(),
    TotalDamage = sum(DamageProperty),
    AvgDamage = avg(DamageProperty)
    by State, EventType
```

**Hints for high cardinality:**
- `hint.shufflekey=<column>`: Distribute processing
- `hint.strategy=shuffle`: Parallel execution

### 5. `join` - Combine Tables

Merges rows from two tables based on matching keys.

```kusto
Table1
| join kind=inner (Table2) on CommonColumn
```

**Join Flavors:**

| Kind | Description |
|------|-------------|
| `innerunique` | Default; deduplicated inner join |
| `inner` | Standard inner join |
| `leftouter` | All left rows, matching right |
| `rightouter` | All right rows, matching left |
| `fullouter` | All rows from both tables |
| `leftsemi` | Left rows that have matches |
| `leftanti` | Left rows without matches |
| `rightsemi` | Right rows that have matches |
| `rightanti` | Right rows without matches |

### 6. `union` - Concatenate Tables

Combines rows from multiple tables.

```kusto
Table1
| union Table2, Table3
| union withsource=SourceTable Table*
```

**Options:**
- `kind=inner`: Only common columns
- `kind=outer`: All columns (default)
- `withsource=ColumnName`: Add source identifier

### 7. `sort` / `order by` - Sort Rows

Orders rows by specified columns.

```kusto
StormEvents
| sort by StartTime desc, State asc
```

**Options:**
- `asc` / `desc`: Sort direction
- `nulls first` / `nulls last`: Null handling

### 8. `top` - Return Top N Rows

Returns first N rows sorted by expression.

```kusto
StormEvents
| top 10 by DamageProperty desc
```

Equivalent to `sort by | take`.

### 9. `take` / `limit` - Sample Rows

Returns up to N rows (no guaranteed order).

```kusto
StormEvents
| take 100
```

Use for quick data exploration; not for production queries.

### 10. `distinct` - Unique Values

Returns distinct combinations of columns.

```kusto
StormEvents
| distinct State, EventType
```

### 11. `count` - Count Rows

Returns the total number of rows.

```kusto
StormEvents
| count

StormEvents
| where State == "TEXAS"
| count
```

### 12. `parse` - Extract Values from Strings

Parses string values into typed columns.

```kusto
Traces
| parse EventText with * "user=" User ", action=" Action
```

**Modes:**
- `kind=simple`: Strict matching (default)
- `kind=regex`: Regular expression patterns
- `kind=relaxed`: Partial matching allowed

### 13. `mv-expand` - Expand Arrays

Expands multi-value (array) columns into separate rows.

```kusto
T
| mv-expand ArrayColumn
```

### 14. `mv-apply` - Apply Operations to Arrays

Applies subquery to each element of an array.

```kusto
T
| mv-apply item = ArrayColumn to typeof(long) on (
    summarize sum(item)
)
```

### 15. `make-series` - Create Time Series

Creates series of aggregated values for time-series analysis.

```kusto
StormEvents
| make-series EventCount = count() default=0
    on StartTime from datetime(2007-01-01) to datetime(2007-12-31) step 1d
    by State
```

### 16. `render` - Visualize Results

Generates chart visualizations.

```kusto
StormEvents
| summarize count() by State
| top 10 by count_
| render piechart
```

**Visualization Types:**
- `timechart`, `linechart`, `areachart`
- `barchart`, `columnchart`
- `piechart`, `scatterchart`
- `table`, `card`

### 17. `lookup` - Dimension Lookup

Extends with columns from a dimension table.

```kusto
FactTable
| lookup DimensionTable on Key
```

More efficient than `join` when right side is small.

### 18. `let` - Define Variables/Functions

Creates named expressions or functions.

```kusto
let threshold = 100;
let startDate = ago(7d);
let myFunction = (x: long) { x * 2 };

StormEvents
| where DamageProperty > threshold
| where StartTime > startDate
```

### 19. `invoke` - Call Tabular Functions

Invokes a tabular function with the pipe input.

```kusto
StormEvents
| invoke MyCustomFunction()
```

### 20. `serialize` - Mark Row Order

Marks that row order is preserved (required for window functions).

```kusto
StormEvents
| serialize
| extend RowNum = row_number()
```

---

## Key Aggregation Functions

### Counting Functions

| Function | Description |
|----------|-------------|
| `count()` | Count all rows |
| `countif(predicate)` | Count rows matching condition |
| `dcount(column)` | Approximate distinct count |
| `dcountif(column, predicate)` | Conditional distinct count |
| `count_distinct(column)` | Exact distinct count |

### Statistical Functions

| Function | Description |
|----------|-------------|
| `sum(column)` | Sum of values |
| `avg(column)` | Average value |
| `min(column)` | Minimum value |
| `max(column)` | Maximum value |
| `stdev(column)` | Standard deviation |
| `variance(column)` | Variance |
| `percentile(column, n)` | Nth percentile |
| `percentiles(column, n1, n2, ...)` | Multiple percentiles |

### Collection Functions

| Function | Description |
|----------|-------------|
| `make_list(column)` | Collect values into array |
| `make_set(column)` | Collect unique values into array |
| `make_bag(column)` | Collect into property bag |
| `arg_max(col1, col2, ...)` | Row with max value of col1 |
| `arg_min(col1, col2, ...)` | Row with min value of col1 |
| `take_any(column)` | Any non-empty value |

---

## Key Scalar Functions

### String Functions

| Function | Description | Example |
|----------|-------------|---------|
| `strlen(s)` | String length | `strlen("hello")` = 5 |
| `substring(s, start, len)` | Extract substring | `substring("hello", 0, 2)` = "he" |
| `strcat(s1, s2, ...)` | Concatenate strings | `strcat("a", "b")` = "ab" |
| `split(s, delimiter)` | Split into array | `split("a,b,c", ",")` |
| `replace_string(s, old, new)` | Replace substring | `replace_string("abc", "b", "x")` |
| `tolower(s)` | Lowercase | `tolower("ABC")` = "abc" |
| `toupper(s)` | Uppercase | `toupper("abc")` = "ABC" |
| `trim(s)` | Remove whitespace | `trim("  a  ")` = "a" |
| `extract(regex, group, s)` | Regex extract | `extract(@"(\d+)", 1, "abc123")` |

### String Comparison Operators

| Operator | Description | Case-sensitive |
|----------|-------------|----------------|
| `==` | Equals | Yes |
| `=~` | Equals | No |
| `!=` | Not equals | Yes |
| `!~` | Not equals | No |
| `has` | Contains token | No |
| `has_cs` | Contains token | Yes |
| `contains` | Contains substring | No |
| `contains_cs` | Contains substring | Yes |
| `startswith` | Starts with | No |
| `startswith_cs` | Starts with | Yes |
| `endswith` | Ends with | No |
| `endswith_cs` | Ends with | Yes |
| `matches regex` | Regex match | Yes |

### DateTime Functions

| Function | Description | Example |
|----------|-------------|---------|
| `now()` | Current UTC time | `now()` |
| `ago(timespan)` | Time before now | `ago(1d)` |
| `datetime_add(part, n, dt)` | Add to datetime | `datetime_add('day', 7, now())` |
| `datetime_diff(part, dt1, dt2)` | Difference | `datetime_diff('hour', dt1, dt2)` |
| `startofday(dt)` | Start of day | `startofday(now())` |
| `startofweek(dt)` | Start of week | `startofweek(now())` |
| `startofmonth(dt)` | Start of month | `startofmonth(now())` |
| `startofyear(dt)` | Start of year | `startofyear(now())` |
| `dayofweek(dt)` | Day of week (0-6) | `dayofweek(now())` |
| `dayofmonth(dt)` | Day of month | `dayofmonth(now())` |
| `format_datetime(dt, fmt)` | Format as string | `format_datetime(now(), "yyyy-MM-dd")` |
| `bin(value, roundTo)` | Round to bin | `bin(StartTime, 1h)` |

### Conditional Functions

| Function | Description | Example |
|----------|-------------|---------|
| `iff(cond, true_val, false_val)` | If-then-else | `iff(x > 0, "positive", "non-positive")` |
| `iif(cond, true_val, false_val)` | Alias for iff | Same as iff |
| `case(cond1, val1, cond2, val2, ..., default)` | Multiple conditions | `case(x<0,"neg",x>0,"pos","zero")` |
| `coalesce(v1, v2, ...)` | First non-null | `coalesce(col1, col2, "default")` |
| `max_of(v1, v2, ...)` | Maximum of values | `max_of(a, b, c)` |
| `min_of(v1, v2, ...)` | Minimum of values | `min_of(a, b, c)` |

### Null-Handling Functions

| Function | Description |
|----------|-------------|
| `isnull(expr)` | True if null |
| `isnotnull(expr)` | True if not null |
| `isempty(expr)` | True if null or empty string |
| `isnotempty(expr)` | True if not null and not empty |

### Type Conversion Functions

| Function | Description |
|----------|-------------|
| `tostring(expr)` | Convert to string |
| `toint(expr)` | Convert to int |
| `tolong(expr)` | Convert to long |
| `todouble(expr)` | Convert to real/double |
| `tobool(expr)` | Convert to boolean |
| `todatetime(expr)` | Convert to datetime |
| `totimespan(expr)` | Convert to timespan |
| `todynamic(expr)` | Convert to dynamic |

### Dynamic/JSON Functions

| Function | Description |
|----------|-------------|
| `parse_json(s)` | Parse JSON string |
| `bag_keys(bag)` | Get property bag keys |
| `bag_merge(bag1, bag2)` | Merge property bags |
| `array_length(arr)` | Array length |
| `array_concat(arr1, arr2)` | Concatenate arrays |
| `array_slice(arr, start, end)` | Slice array |
| `pack(k1, v1, k2, v2, ...)` | Create property bag |

---

## Window Functions

Window functions operate on ordered rows using `serialize`:

| Function | Description |
|----------|-------------|
| `row_number()` | Sequential row number |
| `row_rank()` | Rank (with gaps for ties) |
| `row_dense_rank()` | Dense rank (no gaps) |
| `row_cumsum(col)` | Cumulative sum |
| `prev(col, offset)` | Previous row value |
| `next(col, offset)` | Next row value |

```kusto
StormEvents
| serialize
| extend RowNum = row_number()
| extend PrevState = prev(State, 1)
```

---

## Comments

```kusto
// Single-line comment

/*
   Multi-line
   comment
*/
```

---

## Document Information

- **Generated**: 2026-02-13
- **Source**: Azure Data Explorer documentation
- **Phase**: 3 - In-depth Investigation
