# KQL Query Examples - Common Scenarios

## Overview

This document provides practical KQL query examples for common real-world scenarios including log analysis, metrics monitoring, security investigations, and data exploration.

---

## Table of Contents

1. [Basic Data Exploration](#basic-data-exploration)
2. [Filtering and Searching](#filtering-and-searching)
3. [Aggregation and Statistics](#aggregation-and-statistics)
4. [Time-Based Analysis](#time-based-analysis)
5. [String Parsing and Extraction](#string-parsing-and-extraction)
6. [Joins and Lookups](#joins-and-lookups)
7. [Time Series Analysis](#time-series-analysis)
8. [Log Analytics Scenarios](#log-analytics-scenarios)
9. [Security Investigation Queries](#security-investigation-queries)
10. [Performance Monitoring](#performance-monitoring)

---

## Basic Data Exploration

### Preview Table Data

```kusto
// Quick sample of data
MyTable
| take 10

// Count total rows
MyTable
| count

// Get table schema
MyTable
| getschema
```

### Explore Column Values

```kusto
// Distinct values in a column
StormEvents
| distinct State
| sort by State asc

// Count by category
StormEvents
| summarize Count = count() by EventType
| sort by Count desc

// Top 10 most common values
StormEvents
| summarize Count = count() by State
| top 10 by Count
```

### Data Distribution

```kusto
// Numeric distribution
StormEvents
| summarize
    Min = min(DamageProperty),
    Max = max(DamageProperty),
    Avg = avg(DamageProperty),
    Median = percentile(DamageProperty, 50),
    P95 = percentile(DamageProperty, 95),
    P99 = percentile(DamageProperty, 99)
```

---

## Filtering and Searching

### Basic Filters

```kusto
// Equality filter
StormEvents
| where State == "TEXAS"

// Multiple conditions (AND)
StormEvents
| where State == "TEXAS" and EventType == "Tornado"

// Multiple conditions (OR)
StormEvents
| where State == "TEXAS" or State == "FLORIDA"

// IN operator for multiple values
StormEvents
| where State in ("TEXAS", "FLORIDA", "CALIFORNIA")

// NOT IN
StormEvents
| where State !in ("TEXAS", "FLORIDA")
```

### String Searches

```kusto
// Contains token (fast, uses index)
StormEvents
| where EventNarrative has "damage"

// Contains substring (slower)
StormEvents
| where EventNarrative contains "roof damage"

// Starts with
StormEvents
| where State startswith "NEW"

// Regex matching
StormEvents
| where EventNarrative matches regex @"wind.*damage"
```

### Range Filters

```kusto
// Between range (inclusive)
StormEvents
| where DamageProperty between (1000 .. 10000)

// Date range
StormEvents
| where StartTime between (datetime(2007-01-01) .. datetime(2007-06-30))

// Greater than / less than
StormEvents
| where DamageProperty > 10000
| where StartTime >= ago(30d)
```

### Null Handling

```kusto
// Filter out nulls
StormEvents
| where isnotnull(DamageProperty)

// Filter for nulls
StormEvents
| where isnull(EndLocation)

// Handle empty strings
StormEvents
| where isnotempty(BeginLocation)
```

---

## Aggregation and Statistics

### Basic Aggregations

```kusto
// Count, sum, average by group
StormEvents
| summarize
    EventCount = count(),
    TotalDamage = sum(DamageProperty),
    AvgDamage = avg(DamageProperty)
    by State

// Multiple grouping columns
StormEvents
| summarize count() by State, EventType
| sort by State, count_ desc
```

### Advanced Statistics

```kusto
// Percentiles and statistics
StormEvents
| summarize
    Min = min(DamageProperty),
    P25 = percentile(DamageProperty, 25),
    Median = percentile(DamageProperty, 50),
    P75 = percentile(DamageProperty, 75),
    P90 = percentile(DamageProperty, 90),
    P99 = percentile(DamageProperty, 99),
    Max = max(DamageProperty),
    Avg = avg(DamageProperty),
    StdDev = stdev(DamageProperty)
    by State

// Distinct count
StormEvents
| summarize
    TotalEvents = count(),
    UniqueEventTypes = dcount(EventType),
    UniqueLocations = dcount(BeginLocation)
    by State
```

### Conditional Aggregations

```kusto
// Count with conditions
StormEvents
| summarize
    TotalEvents = count(),
    HighDamageEvents = countif(DamageProperty > 100000),
    TornadoEvents = countif(EventType == "Tornado")
    by State

// Sum with conditions
StormEvents
| summarize
    TotalDamage = sum(DamageProperty),
    TornadoDamage = sumif(DamageProperty, EventType == "Tornado"),
    FloodDamage = sumif(DamageProperty, EventType == "Flood")
    by State
```

### Collecting Values

```kusto
// Collect into list
StormEvents
| summarize EventTypes = make_list(EventType) by State
| where array_length(EventTypes) > 5

// Collect unique values
StormEvents
| summarize UniqueEventTypes = make_set(EventType) by State

// Get row with max value
StormEvents
| summarize arg_max(DamageProperty, *) by State
```

---

## Time-Based Analysis

### Time Filtering

```kusto
// Last N days
StormEvents
| where StartTime > ago(30d)

// Specific date range
StormEvents
| where StartTime between (datetime(2007-01-01) .. datetime(2007-12-31))

// Today only
StormEvents
| where StartTime > startofday(now())

// This week
StormEvents
| where StartTime > startofweek(now())

// This month
StormEvents
| where StartTime > startofmonth(now())
```

### Time Aggregation (Binning)

```kusto
// Hourly counts
StormEvents
| where StartTime > ago(7d)
| summarize EventCount = count() by bin(StartTime, 1h)
| sort by StartTime asc

// Daily counts
StormEvents
| summarize EventCount = count() by bin(StartTime, 1d)
| sort by StartTime asc

// Weekly summary
StormEvents
| summarize
    EventCount = count(),
    TotalDamage = sum(DamageProperty)
    by Week = startofweek(StartTime)
| sort by Week asc
```

### Time Calculations

```kusto
// Duration calculations
StormEvents
| extend Duration = EndTime - StartTime
| where Duration > 1h
| project State, EventType, StartTime, EndTime, Duration

// Day of week analysis
StormEvents
| extend DayOfWeek = dayofweek(StartTime)
| extend DayName = case(
    DayOfWeek == 0d, "Sunday",
    DayOfWeek == 1d, "Monday",
    DayOfWeek == 2d, "Tuesday",
    DayOfWeek == 3d, "Wednesday",
    DayOfWeek == 4d, "Thursday",
    DayOfWeek == 5d, "Friday",
    "Saturday")
| summarize count() by DayName
```

### Hour-of-Day Analysis

```kusto
// Events by hour of day
StormEvents
| extend HourOfDay = datetime_part("hour", StartTime)
| summarize EventCount = count() by HourOfDay
| sort by HourOfDay asc
| render columnchart
```

---

## String Parsing and Extraction

### Parse Operator

```kusto
// Parse structured log messages
Logs
| parse Message with "User " User " performed " Action " on " Resource
| project Timestamp, User, Action, Resource

// Parse with wildcards
Logs
| parse Message with * "error code=" ErrorCode:int * "message=" ErrorMessage
| where ErrorCode > 500
```

### Extract with Regex

```kusto
// Extract email domain
Users
| extend Domain = extract(@"@(.+)$", 1, Email)
| summarize count() by Domain

// Extract IP address
Logs
| extend IPAddress = extract(@"(\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3})", 1, Message)
| where isnotempty(IPAddress)

// Extract multiple groups
Logs
| extend
    Method = extract(@"(GET|POST|PUT|DELETE)", 1, Message),
    URL = extract(@"\"[A-Z]+ ([^ ]+)", 1, Message),
    StatusCode = extract(@"\" (\d{3})", 1, Message)
```

### Split and Array Operations

```kusto
// Split string into array
Logs
| extend Parts = split(Path, "/")
| extend FirstPart = Parts[0], SecondPart = Parts[1]

// Parse CSV values
Data
| extend Values = split(CSVColumn, ",")
| mv-expand Value = Values
| extend Value = tostring(Value)
```

### JSON Parsing

```kusto
// Parse JSON string
Logs
| extend ParsedJson = parse_json(JsonColumn)
| extend
    Name = ParsedJson.name,
    Value = ParsedJson.value,
    Tags = ParsedJson.tags

// Access nested JSON
Logs
| extend ParsedJson = parse_json(JsonColumn)
| extend NestedValue = ParsedJson.level1.level2.value

// Expand JSON arrays
Logs
| extend ParsedJson = parse_json(JsonColumn)
| mv-expand Item = ParsedJson.items
| extend ItemName = Item.name, ItemValue = Item.value
```

---

## Joins and Lookups

### Inner Join

```kusto
// Basic inner join
Orders
| join kind=inner (Customers) on CustomerId
| project OrderId, CustomerName, OrderDate, Amount
```

### Left Outer Join

```kusto
// Keep all orders, add customer info if available
Orders
| join kind=leftouter (Customers) on CustomerId
| project OrderId, CustomerName = coalesce(CustomerName, "Unknown"), Amount
```

### Lookup for Dimension Tables

```kusto
// Efficient lookup for small dimension tables
FactSales
| lookup (
    DimProduct | project ProductId, ProductName, Category
) on ProductId
| summarize TotalSales = sum(Amount) by Category
```

### Anti-Join (Find Missing)

```kusto
// Find orders without customers
Orders
| join kind=leftanti (Customers) on CustomerId
| summarize OrphanOrders = count()

// Find customers without orders
Customers
| join kind=leftanti (Orders) on CustomerId
| project CustomerId, CustomerName
```

### Semi-Join (Filter by Existence)

```kusto
// Get customers who have made orders
Customers
| join kind=leftsemi (Orders) on CustomerId

// Filter events by user list
let TargetUsers = Users | where Department == "Engineering" | project UserId;
Events
| where UserId in (TargetUsers)
```

### Self-Join

```kusto
// Find events that happened close together
Events as e1
| join kind=inner (Events) on $left.UserId == $right.UserId
| where e1.Timestamp < Timestamp
| where Timestamp - e1.Timestamp < 1h
| project UserId, FirstEvent = e1.EventType, SecondEvent = EventType, TimeDiff = Timestamp - e1.Timestamp
```

---

## Time Series Analysis

### Create Time Series

```kusto
// Create hourly time series
StormEvents
| make-series EventCount = count() default=0
    on StartTime from datetime(2007-01-01) to datetime(2007-12-31) step 1d
    by State

// With multiple aggregations
Metrics
| make-series
    AvgCPU = avg(CPU),
    MaxCPU = max(CPU),
    MinCPU = min(CPU)
    on Timestamp from ago(7d) to now() step 1h
    by Server
```

### Fill Gaps in Time Series

```kusto
// Forward fill
StormEvents
| make-series EventCount = count() default=long(null)
    on StartTime from datetime(2007-01-01) to datetime(2007-12-31) step 1d
| extend EventCount = series_fill_forward(EventCount)

// Linear interpolation
| extend EventCount = series_fill_linear(EventCount)
```

### Moving Averages

```kusto
// Calculate moving average
Metrics
| make-series AvgCPU = avg(CPU) on Timestamp from ago(7d) to now() step 1h
| extend MovingAvg = series_fir(AvgCPU, repeat(1, 24), true, true)
| mv-expand Timestamp, AvgCPU, MovingAvg
| render timechart
```

---

## Log Analytics Scenarios

### Error Analysis

```kusto
// Error count by type
Logs
| where Level == "Error"
| summarize ErrorCount = count() by ErrorType
| top 10 by ErrorCount
| render piechart

// Error rate over time
Logs
| summarize
    TotalRequests = count(),
    Errors = countif(Level == "Error")
    by bin(Timestamp, 1h)
| extend ErrorRate = 100.0 * Errors / TotalRequests
| render timechart
```

### HTTP Request Analysis

```kusto
// Request latency percentiles
Requests
| summarize
    P50 = percentile(Duration, 50),
    P90 = percentile(Duration, 90),
    P95 = percentile(Duration, 95),
    P99 = percentile(Duration, 99)
    by bin(Timestamp, 5m)
| render timechart

// Slow requests
Requests
| where Duration > 5s
| summarize count() by URL, bin(Timestamp, 1h)
| where count_ > 10

// Status code distribution
Requests
| summarize count() by StatusCode = tostring(ResponseCode)
| render piechart
```

### User Activity Analysis

```kusto
// Daily active users
Events
| summarize DAU = dcount(UserId) by bin(Timestamp, 1d)
| render timechart

// User session analysis
Events
| summarize
    SessionStart = min(Timestamp),
    SessionEnd = max(Timestamp),
    EventCount = count()
    by UserId, SessionId
| extend SessionDuration = SessionEnd - SessionStart
| summarize
    AvgSessionDuration = avg(SessionDuration),
    AvgEventsPerSession = avg(EventCount)
```

---

## Security Investigation Queries

### Failed Login Analysis

```kusto
// Failed logins by user
SecurityLogs
| where EventType == "LoginFailed"
| summarize FailedAttempts = count() by User, bin(Timestamp, 1h)
| where FailedAttempts > 5

// Brute force detection
SecurityLogs
| where EventType == "LoginFailed"
| summarize
    FailedAttempts = count(),
    UniqueUsers = dcount(User)
    by SourceIP, bin(Timestamp, 10m)
| where FailedAttempts > 10 and UniqueUsers > 3
```

### Suspicious Activity Detection

```kusto
// Unusual login locations
SecurityLogs
| where EventType == "LoginSuccess"
| summarize Locations = make_set(Location) by User
| where array_length(Locations) > 3

// After-hours activity
SecurityLogs
| extend Hour = datetime_part("hour", Timestamp)
| where Hour < 6 or Hour > 22
| summarize count() by User, EventType
| where count_ > 10
```

### Data Exfiltration Detection

```kusto
// Large data transfers
NetworkLogs
| where BytesSent > 100000000  // 100MB
| summarize
    TotalBytes = sum(BytesSent),
    TransferCount = count()
    by SourceUser, DestinationIP
| where TotalBytes > 1000000000  // 1GB
| sort by TotalBytes desc
```

---

## Performance Monitoring

### Resource Utilization

```kusto
// CPU utilization by server
PerformanceMetrics
| where CounterName == "% Processor Time"
| summarize AvgCPU = avg(CounterValue), MaxCPU = max(CounterValue)
    by Computer, bin(Timestamp, 5m)
| where AvgCPU > 80
| render timechart

// Memory pressure
PerformanceMetrics
| where CounterName == "Available MBytes"
| summarize AvgAvailableMB = avg(CounterValue) by Computer, bin(Timestamp, 5m)
| where AvgAvailableMB < 1000
```

### Application Performance

```kusto
// Response time by endpoint
AppRequests
| summarize
    AvgDuration = avg(Duration),
    P95Duration = percentile(Duration, 95),
    RequestCount = count()
    by Name
| where RequestCount > 100
| top 20 by P95Duration desc

// Dependency call failures
AppDependencies
| where Success == false
| summarize FailureCount = count() by Target, DependencyType
| top 10 by FailureCount
```

### Anomaly Detection

```kusto
// Simple threshold-based anomaly
Metrics
| summarize AvgValue = avg(Value), StdDev = stdev(Value) by MetricName
| join kind=inner (Metrics) on MetricName
| where Value > AvgValue + (3 * StdDev) or Value < AvgValue - (3 * StdDev)

// Using series decomposition
Metrics
| make-series Value = avg(Value) on Timestamp from ago(7d) to now() step 1h
| extend (anomalies, score, baseline) = series_decompose_anomalies(Value)
| mv-expand Timestamp, Value, anomalies, score, baseline
| where anomalies != 0
```

---

## Visualization Examples

### Time Chart

```kusto
StormEvents
| summarize EventCount = count() by bin(StartTime, 1d)
| render timechart with (title="Daily Storm Events")
```

### Bar Chart

```kusto
StormEvents
| summarize count() by State
| top 10 by count_
| render barchart with (title="Top 10 States by Storm Events")
```

### Pie Chart

```kusto
StormEvents
| summarize count() by EventType
| top 5 by count_
| render piechart with (title="Storm Events by Type")
```

### Scatter Plot

```kusto
StormEvents
| where DamageProperty > 0 and DamageCrops > 0
| project DamageProperty, DamageCrops
| render scatterchart
```

### Multi-Series Line Chart

```kusto
StormEvents
| where State in ("TEXAS", "FLORIDA", "CALIFORNIA")
| summarize EventCount = count() by bin(StartTime, 1d), State
| render timechart
```

---

## Document Information

- **Generated**: 2026-02-13
- **Source**: Azure Data Explorer documentation and common patterns
- **Phase**: 3 - In-depth Investigation
