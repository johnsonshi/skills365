# Azure Data Explorer Management Commands Best Practices

## Overview

This document provides best practices for configuring policies, designing schemas, and managing access control in Azure Data Explorer. Following these patterns will help optimize performance, control costs, and maintain security.

---

## 1. Schema Design Best Practices

### 1.1 Table Design

**Use appropriate data types:**
- Use `datetime` instead of `string` for timestamps
- Use `long` for IDs instead of `string` when possible
- Use `dynamic` sparingly - it has higher storage and query overhead
- Consider `guid` for unique identifiers

**Column naming:**
- Use descriptive, PascalCase names
- Avoid reserved keywords
- Include units in names where appropriate (e.g., `DurationMs`, `SizeBytes`)

**Table organization:**
```kusto
// Good: Organize tables in folders
.create table Logs (Timestamp:datetime, Message:string)
with (folder = "Telemetry")

.create table Errors (Timestamp:datetime, Error:string)
with (folder = "Telemetry")
```

### 1.2 Function Design

**Use functions for reusable logic:**
```kusto
// Create parameterized functions for common queries
.create function with (folder = "Analytics")
GetActiveUsers(lookback:timespan)
{
    Sessions
    | where Timestamp > ago(lookback)
    | summarize dcount(UserId)
}
```

**Mark functions as views when appropriate:**
```kusto
// Views participate in search and union * operations
.create function with (view = true)
ActiveSessions
{
    Sessions | where IsActive == true
}
```

### 1.3 Schema Evolution

**Use create-merge for safe schema updates:**
```kusto
// Adds new columns without affecting existing ones
.create-merge table Events (
    Timestamp:datetime,
    EventType:string,
    NewColumn:string  // Safe addition
)
```

**Plan for column type changes:**
- Changing column types requires careful migration
- Consider creating new tables and using update policies for transformations

---

## 2. Policy Configuration Patterns

### 2.1 Retention Policy Strategy

**Tiered retention approach:**
```kusto
// Hot data: Recent, frequently accessed
.alter-merge table HotTable policy retention softdelete = 30d

// Warm data: Moderate access
.alter-merge table WarmTable policy retention softdelete = 90d

// Cold data: Archival, rare access
.alter-merge table ColdTable policy retention softdelete = 365d
```

**Database defaults with table overrides:**
```kusto
// Set database default
.alter-merge database MyDatabase policy retention softdelete = 90d

// Override for specific high-value table
.alter-merge table CriticalAuditLog policy retention softdelete = 730d
```

**Disable recoverability for cost savings:**
```kusto
// When data recovery isn't needed
.alter-merge table TempData policy retention softdelete = 7d recoverability = disabled
```

### 2.2 Caching Policy Strategy

**Align caching with access patterns:**
```kusto
// Frequently queried recent data
.alter table RecentEvents policy caching hot = 30d

// Rarely accessed historical data
.alter table HistoricalEvents policy caching hot = 7d
```

**Calculate optimal cache period:**
- Analyze query patterns to determine access frequency
- Cache period should cover 90%+ of queries
- Balance cost vs. performance

**Caching policy hierarchy:**
```
Cluster default > Database policy > Table policy
```

**Use query hints for mixed workloads:**
```kusto
// Force hot cache for dashboard queries
set query_datascope = "hotcache";
Events | summarize count() by bin(Timestamp, 1h)

// Allow cold data for ad-hoc analysis
set query_datascope = "all";
Events | where Timestamp > ago(365d)
```

### 2.3 Partitioning Policy Guidelines

**When to use partitioning:**
- Frequent filters on medium/high cardinality string/guid columns
- Frequent aggregations/joins on high cardinality columns
- Out-of-order datetime data ingestion

**When NOT to use partitioning:**
- Tables with less than 1GB compressed data per partition
- Low cardinality filter columns
- Randomly distributed queries

**Hash partition for multi-tenant scenarios:**
```kusto
.alter table TenantData policy partitioning
```
{
  "PartitionKeys": [{
    "ColumnName": "TenantId",
    "Kind": "Hash",
    "Properties": {
      "Function": "XxHash64",
      "MaxPartitionCount": 128,
      "PartitionAssignmentMode": "Uniform"
    }
  }]
}
```

**Datetime partition for out-of-order data:**
```kusto
.alter table TimeSeriesData policy partitioning
```
{
  "PartitionKeys": [{
    "ColumnName": "EventTime",
    "Kind": "UniformRange",
    "Properties": {
      "Reference": "2020-01-01T00:00:00",
      "RangeSize": "1.00:00:00",
      "OverrideCreationTime": true
    }
  }]
}
```

### 2.4 Ingestion Batching Optimization

**Low latency configuration:**
```kusto
.alter table RealTimeEvents policy ingestionbatching
```
{
  "MaximumBatchingTimeSpan": "00:00:30",
  "MaximumNumberOfItems": 100,
  "MaximumRawDataSizeMB": 100
}
```

**High throughput configuration:**
```kusto
.alter table BulkImport policy ingestionbatching
```
{
  "MaximumBatchingTimeSpan": "00:05:00",
  "MaximumNumberOfItems": 1000,
  "MaximumRawDataSizeMB": 1024
}
```

**Trade-offs:**
| Setting | Low Latency | High Throughput |
|---------|-------------|-----------------|
| BatchingTimeSpan | 30s - 1m | 5m - 10m |
| NumberOfItems | 50 - 100 | 500 - 1000 |
| Impact | More small extents | Fewer large extents |

### 2.5 Merge Policy Tuning

**Align MaxRangeInHours with retention:**
```kusto
// For 30-day retention, use ~2-3% = ~18 hours
.alter table MyTable policy merge
```
{"MaxRangeInHours": 18}
```
```

**MaxRangeInHours guidelines:**
| Retention Period | Recommended MaxRangeInHours |
|------------------|----------------------------|
| 7 days | 4 |
| 14 days | 8 |
| 30 days | 18 |
| 90 days | 60 |
| 365 days | 250 |

---

## 3. Materialized View Patterns

### 3.1 Deduplication Pattern

```kusto
// Deduplicate events by EventId
.create materialized-view with (lookback=6h) DeduplicatedEvents on table RawEvents
{
    RawEvents
    | summarize take_any(*) by EventId
}
```

### 3.2 Pre-aggregation Pattern

```kusto
// Daily aggregates for dashboard
.create materialized-view DailyMetrics on table Telemetry
{
    Telemetry
    | summarize
        TotalRequests = count(),
        UniqueUsers = dcount(UserId),
        AvgLatency = avg(LatencyMs)
      by bin(Timestamp, 1d), Service
}
```

### 3.3 Latest State Pattern

```kusto
// Track latest state per device
.create materialized-view with (lookback=1d) DeviceLatestState on table DeviceEvents
{
    DeviceEvents
    | summarize arg_max(Timestamp, *) by DeviceId
}
```

### 3.4 Materialized View Best Practices

**Add datetime group-by keys:**
```kusto
// Better performance with datetime key
.create materialized-view BetterView on table T
{
    T | summarize count() by UserId, bin(Timestamp, 1h)
}
```

**Use update policies for transformations:**
```kusto
// Transform in update policy, aggregate in materialized view
// Step 1: Update policy for transformation
.alter table TransformedData policy update
@'[{"Source": "RawData", "Query": "TransformFunction()"}]'

// Step 2: Materialized view for aggregation
.create materialized-view Aggregates on table TransformedData
{
    TransformedData | summarize count() by Category
}
```

**Query materialized part for dashboards:**
```kusto
// Use materialized_view() for better performance
materialized_view("DailyMetrics")
| where Timestamp > ago(30d)
```

---

## 4. Access Control Best Practices

### 4.1 Principle of Least Privilege

**Role hierarchy:**
```
admins > users > viewers > ingestors > monitors
```

**Grant minimal required access:**
```kusto
// Data engineers: ingest only
.add table RawData ingestors ('aadgroup=data-engineers@domain.com')

// Analysts: read only
.add database Analytics viewers ('aadgroup=analysts@domain.com')

// Power users: create objects
.add database Analytics users ('aadgroup=power-users@domain.com')
```

### 4.2 Row Level Security Patterns

**Department-based access:**
```kusto
.create function DepartmentRLS()
{
    let UserDept = toscalar(
        DepartmentMapping
        | where User == current_principal_details()["UserPrincipalName"]
        | project Department
    );
    SalesData | where Department == UserDept
}

.alter table SalesData policy row_level_security enable "DepartmentRLS()"
```

**Manager escalation pattern:**
```kusto
.create function ManagerEscalationRLS()
{
    let IsManager = current_principal_is_member_of('aadgroup=managers@domain.com');
    let AllData = ConfidentialData | where IsManager;
    let FilteredData = ConfidentialData | where not(IsManager)
        | where Owner == current_principal();
    union AllData, FilteredData
}
```

**Multi-tenant isolation:**
```kusto
.create function TenantIsolationRLS()
{
    let TenantId = toscalar(
        TenantMapping
        | where Principal == current_principal()
        | project TenantId
    );
    MultiTenantData | where TenantId == TenantId
}
```

### 4.3 Service Principal Access

```kusto
// Grant ingestion access to service principal
.add table Events ingestors ('aadapp=<app-id>;<tenant-id>')

// Grant read access for reporting service
.add database Reporting viewers ('aadapp=<app-id>;<tenant-id>')
```

---

## 5. Update Policy Patterns

### 5.1 ETL Pattern

```kusto
// Source table for raw JSON
.create table RawLogs (Data:dynamic)

// Target table with parsed schema
.create table ParsedLogs (Timestamp:datetime, Level:string, Message:string)

// Transformation function
.create function ParseLogs()
{
    RawLogs
    | extend
        Timestamp = todatetime(Data.timestamp),
        Level = tostring(Data.level),
        Message = tostring(Data.message)
    | project-away Data
}

// Update policy (transactional for data integrity)
.alter table ParsedLogs policy update
@'[{"IsEnabled": true, "Source": "RawLogs", "Query": "ParseLogs()", "IsTransactional": true}]'
```

### 5.2 Data Routing Pattern

```kusto
// Route data to different tables based on type
.create function RouteErrors()
{
    AllEvents
    | where Level == "Error"
}

.create function RouteWarnings()
{
    AllEvents
    | where Level == "Warning"
}

.alter table ErrorEvents policy update
@'[{"IsEnabled": true, "Source": "AllEvents", "Query": "RouteErrors()", "IsTransactional": true}]'

.alter table WarningEvents policy update
@'[{"IsEnabled": true, "Source": "AllEvents", "Query": "RouteWarnings()", "IsTransactional": true}]'
```

### 5.3 Zero Retention Source Pattern

```kusto
// Remove data from source after processing
.alter-merge table RawLogs policy retention softdelete = 0s

// Ensure update policy is transactional
.alter table ProcessedLogs policy update
@'[{"IsEnabled": true, "Source": "RawLogs", "Query": "TransformLogs()", "IsTransactional": true}]'
```

---

## 6. Monitoring and Maintenance

### 6.1 Regular Health Checks

```kusto
// Check materialized view health
.show materialized-views
| project Name, IsHealthy, MaterializedTo, IsEnabled

// Check for failed ingestions
.show ingestion failures
| where FailedOn > ago(1d)
| summarize FailureCount = count() by Table, FailureKind

// Check extent fragmentation
.show database MyDatabase extents
| summarize ExtentCount = count(), TotalRows = sum(RowCount) by TableName
| extend AvgRowsPerExtent = TotalRows / ExtentCount
```

### 6.2 Policy Review Checklist

- [ ] Retention policies align with compliance requirements
- [ ] Caching policies match query patterns
- [ ] Partitioning only on appropriate tables
- [ ] Materialized views are healthy and up-to-date
- [ ] Security roles follow least privilege principle
- [ ] Update policies are transactional where data integrity matters

---

## 7. Common Mistakes to Avoid

### 7.1 Policy Anti-patterns

**Over-partitioning:**
```kusto
// BAD: Partitioning small tables
.alter table SmallTable policy partitioning {...}  // Table < 1GB

// GOOD: Only partition tables with sufficient data
// Check table size first
.show table LargeTable details
```

**Aggressive batching:**
```kusto
// BAD: Too aggressive batching causes many small extents
{"MaximumBatchingTimeSpan": "00:00:10"}

// GOOD: Balance latency with extent efficiency
{"MaximumBatchingTimeSpan": "00:01:00"}
```

### 7.2 Security Anti-patterns

**Over-privileged access:**
```kusto
// BAD: Giving admin to everyone
.add database MyDB admins ('aadgroup=all-users@domain.com')

// GOOD: Use appropriate roles
.add database MyDB viewers ('aadgroup=all-users@domain.com')
.add database MyDB admins ('aadgroup=db-admins@domain.com')
```

**Unvalidated RLS queries:**
```kusto
// BAD: RLS that might fail or return all data
.alter table T policy row_level_security enable
    "T | where something_that_might_fail"

// GOOD: Defensive RLS with fallback
.alter table T policy row_level_security enable
    "T | where coalesce(current_principal_is_member_of('aadgroup=...'), false)"
```

---

## Summary Checklist

| Area | Best Practice |
|------|--------------|
| **Schema** | Use appropriate types, organize in folders, use create-merge |
| **Retention** | Tier by access pattern, disable recoverability for temp data |
| **Caching** | Match query patterns, use 90% coverage rule |
| **Partitioning** | Only for large tables with specific access patterns |
| **Materialized Views** | Add datetime keys, use lookback, query materialized part |
| **Security** | Least privilege, use RLS for row-level access |
| **Update Policies** | Use transactional mode, combine with zero retention |

---

*Generated: 2026-02-13 | Source: Azure Data Explorer Documentation*
