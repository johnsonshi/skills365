# Azure Data Explorer Management Commands Examples

## Overview

This document provides practical, copy-paste ready examples for common management scenarios in Azure Data Explorer.

---

## 1. Table Management Examples

### 1.1 Create Tables with Various Data Types

```kusto
// Telemetry table
.create table Telemetry (
    Timestamp: datetime,
    DeviceId: string,
    Temperature: real,
    Humidity: real,
    Status: string,
    Metadata: dynamic
) with (folder = "IoT", docstring = "Device telemetry data")

// User events table
.create table UserEvents (
    EventId: guid,
    UserId: long,
    EventType: string,
    EventTime: datetime,
    Properties: dynamic,
    SessionId: string
) with (folder = "Analytics")

// Audit log table
.create table AuditLog (
    Id: long,
    Timestamp: datetime,
    Principal: string,
    Action: string,
    Resource: string,
    Result: string,
    Details: dynamic
) with (folder = "Security")
```

### 1.2 Create Multiple Tables at Once

```kusto
.create tables
    RawEvents (Timestamp:datetime, Data:dynamic),
    ProcessedEvents (Timestamp:datetime, EventType:string, Value:real),
    ErrorEvents (Timestamp:datetime, ErrorCode:string, Message:string)
```

### 1.3 Modify Table Schema

```kusto
// Add columns safely (preserves existing data)
.alter-merge table Telemetry (
    NewColumn1: string,
    NewColumn2: int
)

// Replace entire schema (use with caution)
.alter table Telemetry (
    Timestamp: datetime,
    DeviceId: string,
    Temperature: real,
    Humidity: real,
    Status: string,
    Metadata: dynamic,
    Region: string
)
```

### 1.4 Column Operations

```kusto
// Rename a column
.rename column Telemetry.Status to DeviceStatus

// Rename multiple columns
.rename columns
    NewName1 = Table1.OldName1,
    NewName2 = Table2.OldName2

// Drop columns
.drop column Telemetry.Metadata
.drop table Telemetry columns (OldCol1, OldCol2)
```

---

## 2. Function Examples

### 2.1 Simple Query Function

```kusto
.create function
with (docstring = "Get events from the last N days", folder = "Utilities")
GetRecentEvents(days: int)
{
    Events
    | where Timestamp > ago(1d * days)
    | order by Timestamp desc
}
```

### 2.2 Parameterized Analytics Function

```kusto
.create function
with (docstring = "Calculate metrics for a specific customer", folder = "Analytics")
CustomerMetrics(customerId: string, startTime: datetime, endTime: datetime)
{
    Transactions
    | where CustomerId == customerId
    | where Timestamp between (startTime .. endTime)
    | summarize
        TotalTransactions = count(),
        TotalAmount = sum(Amount),
        AvgAmount = avg(Amount),
        UniqueProducts = dcount(ProductId)
}
```

### 2.3 Table-Valued Function

```kusto
.create function
with (folder = "ETL")
ParseJsonLogs()
{
    RawLogs
    | extend
        Timestamp = todatetime(Data["timestamp"]),
        Level = tostring(Data["level"]),
        Message = tostring(Data["message"]),
        Source = tostring(Data["source"]),
        CorrelationId = tostring(Data["correlationId"])
    | project-away Data
}
```

### 2.4 Function as View

```kusto
.create function
with (view = true, folder = "Views")
ActiveDevices
{
    DeviceStatus
    | where LastSeen > ago(1h)
    | where Status == "Online"
}
```

---

## 3. Ingestion Mapping Examples

### 3.1 JSON Mapping

```kusto
.create table Telemetry ingestion json mapping "TelemetryMapping"
```
[
    {"column": "Timestamp", "path": "$.timestamp", "datatype": "datetime"},
    {"column": "DeviceId", "path": "$.device_id"},
    {"column": "Temperature", "path": "$.readings.temperature", "datatype": "real"},
    {"column": "Humidity", "path": "$.readings.humidity", "datatype": "real"},
    {"column": "Status", "path": "$.status"},
    {"column": "Metadata", "path": "$", "datatype": "dynamic"}
]
```

### 3.2 CSV Mapping

```kusto
.create table LogData ingestion csv mapping "LogMapping"
```
[
    {"column": "Timestamp", "ordinal": 0, "datatype": "datetime"},
    {"column": "Level", "ordinal": 1},
    {"column": "Message", "ordinal": 2},
    {"column": "Source", "ordinal": 3}
]
```

### 3.3 Parquet Mapping

```kusto
.create table Analytics ingestion parquet mapping "ParquetMapping"
```
[
    {"column": "Id", "path": "$.id", "datatype": "long"},
    {"column": "EventTime", "path": "$.event_time", "datatype": "datetime"},
    {"column": "Category", "path": "$.category"},
    {"column": "Value", "path": "$.value", "datatype": "real"}
]
```

---

## 4. Policy Configuration Examples

### 4.1 Retention Policy Setup

```kusto
// 30-day retention for operational data
.alter-merge table OperationalLogs policy retention
    softdelete = 30d recoverability = enabled

// 7-day retention for temp data (no recovery needed)
.alter-merge table TempProcessing policy retention
    softdelete = 7d recoverability = disabled

// 2-year retention for compliance
.alter-merge table AuditLog policy retention
    softdelete = 730d recoverability = enabled

// Database-level default
.alter-merge database MyDatabase policy retention
    softdelete = 90d recoverability = enabled
```

### 4.2 Caching Policy Setup

```kusto
// Hot cache for recent data (dashboard queries)
.alter table DashboardData policy caching hot = 14d

// Minimal hot cache for archival data
.alter table HistoricalData policy caching hot = 3d

// Extended cache for analytics
.alter table AnalyticsData policy caching hot = 60d

// Database default
.alter database MyDatabase policy caching hot = 30d
```

### 4.3 Partitioning Policy Examples

```kusto
// Hash partition for multi-tenant data
.alter table MultiTenantData policy partitioning
```
{
    "PartitionKeys": [
        {
            "ColumnName": "TenantId",
            "Kind": "Hash",
            "Properties": {
                "Function": "XxHash64",
                "MaxPartitionCount": 128,
                "Seed": 1,
                "PartitionAssignmentMode": "Uniform"
            }
        }
    ]
}
```

// Datetime partition for out-of-order ingestion
.alter table TimeSeriesData policy partitioning
```
{
    "PartitionKeys": [
        {
            "ColumnName": "EventTime",
            "Kind": "UniformRange",
            "Properties": {
                "Reference": "2020-01-01T00:00:00",
                "RangeSize": "1.00:00:00",
                "OverrideCreationTime": true
            }
        }
    ]
}
```

// Combined hash + datetime partitioning
.alter table IoTData policy partitioning
```
{
    "PartitionKeys": [
        {
            "ColumnName": "DeviceId",
            "Kind": "Hash",
            "Properties": {
                "Function": "XxHash64",
                "MaxPartitionCount": 64,
                "PartitionAssignmentMode": "Uniform"
            }
        },
        {
            "ColumnName": "Timestamp",
            "Kind": "UniformRange",
            "Properties": {
                "Reference": "2020-01-01T00:00:00",
                "RangeSize": "7.00:00:00"
            }
        }
    ]
}
```

### 4.4 Ingestion Batching Configuration

```kusto
// Low latency - streaming-like behavior
.alter table RealtimeEvents policy ingestionbatching
```
{
    "MaximumBatchingTimeSpan": "00:00:30",
    "MaximumNumberOfItems": 100,
    "MaximumRawDataSizeMB": 100
}
```

// Balanced configuration
.alter table StandardEvents policy ingestionbatching
```
{
    "MaximumBatchingTimeSpan": "00:02:00",
    "MaximumNumberOfItems": 500,
    "MaximumRawDataSizeMB": 512
}
```

// High throughput - bulk ingestion
.alter table BulkData policy ingestionbatching
```
{
    "MaximumBatchingTimeSpan": "00:10:00",
    "MaximumNumberOfItems": 2000,
    "MaximumRawDataSizeMB": 2048
}
```

### 4.5 Streaming Ingestion

```kusto
// Enable streaming ingestion on table
.alter table RealtimeEvents policy streamingingestion enable

// Enable for database
.alter database RealtimeDB policy streamingingestion enable

// Disable streaming (fall back to batching)
.alter table RealtimeEvents policy streamingingestion disable
```

---

## 5. Update Policy Examples

### 5.1 Basic ETL Pipeline

```kusto
// Step 1: Create source table
.create table RawJson (Data: dynamic)

// Step 2: Create target table
.create table ParsedData (
    Timestamp: datetime,
    EventType: string,
    UserId: string,
    Value: real
)

// Step 3: Create transformation function
.create function ParseJsonData()
{
    RawJson
    | extend
        Timestamp = todatetime(Data.timestamp),
        EventType = tostring(Data.event_type),
        UserId = tostring(Data.user_id),
        Value = todouble(Data.value)
    | project Timestamp, EventType, UserId, Value
}

// Step 4: Apply update policy
.alter table ParsedData policy update
@'[{
    "IsEnabled": true,
    "Source": "RawJson",
    "Query": "ParseJsonData()",
    "IsTransactional": true,
    "PropagateIngestionProperties": true
}]'
```

### 5.2 Data Routing to Multiple Tables

```kusto
// Create target tables
.create table ErrorLogs (Timestamp:datetime, Message:string, StackTrace:string)
.create table InfoLogs (Timestamp:datetime, Message:string)
.create table DebugLogs (Timestamp:datetime, Message:string)

// Create routing functions
.create function RouteErrors()
{
    AllLogs
    | where Level == "Error"
    | project Timestamp, Message, StackTrace = tostring(Data.stackTrace)
}

.create function RouteInfo()
{
    AllLogs
    | where Level == "Info"
    | project Timestamp, Message
}

.create function RouteDebug()
{
    AllLogs
    | where Level == "Debug"
    | project Timestamp, Message
}

// Apply update policies
.alter table ErrorLogs policy update
@'[{"IsEnabled": true, "Source": "AllLogs", "Query": "RouteErrors()", "IsTransactional": true}]'

.alter table InfoLogs policy update
@'[{"IsEnabled": true, "Source": "AllLogs", "Query": "RouteInfo()", "IsTransactional": true}]'

.alter table DebugLogs policy update
@'[{"IsEnabled": true, "Source": "AllLogs", "Query": "RouteDebug()", "IsTransactional": true}]'
```

### 5.3 Enrichment with Lookup Table

```kusto
// Dimension table
.create table DeviceInfo (DeviceId:string, DeviceName:string, Location:string, Owner:string)

// Create enrichment function
.create function EnrichTelemetry()
{
    RawTelemetry
    | lookup DeviceInfo on DeviceId
    | project
        Timestamp,
        DeviceId,
        DeviceName = coalesce(DeviceName, "Unknown"),
        Location = coalesce(Location, "Unknown"),
        Temperature,
        Humidity
}

// Apply update policy
.alter table EnrichedTelemetry policy update
@'[{"IsEnabled": true, "Source": "RawTelemetry", "Query": "EnrichTelemetry()", "IsTransactional": true}]'
```

### 5.4 Zero Retention Source Pattern

```kusto
// Source table gets emptied after processing
.alter-merge table RawIngestion policy retention softdelete = 0s

// Must be transactional when source has zero retention
.alter table ProcessedData policy update
@'[{"IsEnabled": true, "Source": "RawIngestion", "Query": "TransformData()", "IsTransactional": true}]'
```

---

## 6. Materialized View Examples

### 6.1 Deduplication View

```kusto
// Deduplicate events by EventId (keep latest)
.create materialized-view with (lookback=6h)
UniqueEvents on table RawEvents
{
    RawEvents
    | summarize arg_max(Timestamp, *) by EventId
}

// Deduplicate keeping any record
.create materialized-view with (lookback=24h)
DeduplicatedSessions on table Sessions
{
    Sessions
    | summarize take_any(*) by SessionId
}
```

### 6.2 Pre-Aggregation Views

```kusto
// Daily metrics aggregation
.create materialized-view DailyMetrics on table Telemetry
{
    Telemetry
    | summarize
        AvgTemperature = avg(Temperature),
        MaxTemperature = max(Temperature),
        MinTemperature = min(Temperature),
        ReadingCount = count()
      by DeviceId, bin(Timestamp, 1d)
}

// Hourly user activity
.create materialized-view HourlyUserActivity on table UserEvents
{
    UserEvents
    | summarize
        EventCount = count(),
        UniqueUsers = dcount(UserId),
        EventTypes = make_set(EventType)
      by bin(EventTime, 1h)
}
```

### 6.3 Latest State View

```kusto
// Track latest device status
.create materialized-view with (lookback=1d, lookback_column="LastUpdate")
DeviceLatestStatus on table DeviceHeartbeats
{
    DeviceHeartbeats
    | summarize arg_max(LastUpdate, *) by DeviceId
}
```

### 6.4 Backfill Existing Data

```kusto
// Create view with historical backfill
.create async materialized-view
with (
    backfill=true,
    effectiveDateTime=datetime(2024-01-01),
    docstring="Monthly customer summary"
)
MonthlyCustomerSummary on table Transactions
{
    Transactions
    | summarize
        TotalRevenue = sum(Amount),
        TransactionCount = count(),
        UniqueProducts = dcount(ProductId)
      by CustomerId, startofmonth(TransactionTime)
}
```

### 6.5 View Over View (Dedup + Aggregate)

```kusto
// First: deduplication view
.create materialized-view with (lookback=6h)
DeduplicatedOrders on table RawOrders
{
    RawOrders
    | summarize take_any(*) by OrderId
}

// Second: aggregation view over the dedup view
.create materialized-view
DailyOrderSummary on materialized-view DeduplicatedOrders
{
    DeduplicatedOrders
    | summarize
        OrderCount = count(),
        TotalValue = sum(OrderValue),
        UniqueCustomers = dcount(CustomerId)
      by bin(OrderTime, 1d), Region
}
```

---

## 7. Security Examples

### 7.1 Role Management

```kusto
// Add database admin
.add database MyDatabase admins ('aaduser=admin@contoso.com')

// Add group as viewers
.add database MyDatabase viewers ('aadgroup=analysts@contoso.com')

// Add service principal for ingestion
.add table Events ingestors ('aadapp=12345678-1234-1234-1234-123456789012;contoso.com')

// Remove access
.drop database MyDatabase viewers ('aaduser=former-employee@contoso.com')

// Replace all principals of a role
.set database MyDatabase admins ('aaduser=admin1@contoso.com', 'aaduser=admin2@contoso.com')
```

### 7.2 Row Level Security Examples

```kusto
// Simple department filter
.create function DepartmentRLS()
{
    SalesData
    | where Department == current_principal_details()["Department"]
}
.alter table SalesData policy row_level_security enable "DepartmentRLS()"

// Manager can see all, others see own data
.create function ManagerOrOwnerRLS()
{
    let IsManager = current_principal_is_member_of('aadgroup=managers@contoso.com');
    let AllData = SensitiveData | where IsManager;
    let OwnData = SensitiveData | where not(IsManager) and Owner == current_principal();
    union AllData, OwnData
}
.alter table SensitiveData policy row_level_security enable "ManagerOrOwnerRLS()"

// Multi-tenant isolation
.create function TenantRLS()
{
    MultiTenantData
    | where TenantId in (
        TenantAccess
        | where Principal == current_principal()
        | project TenantId
    )
}
.alter table MultiTenantData policy row_level_security enable "TenantRLS()"

// Mask sensitive data for non-privileged users
.create function MaskingRLS()
{
    let CanSeeAll = current_principal_is_member_of('aadgroup=privileged@contoso.com');
    CustomerData
    | extend Email = iff(CanSeeAll, Email, strcat(substring(Email, 0, 3), "***"))
    | extend Phone = iff(CanSeeAll, Phone, "***-***-****")
}
.alter table CustomerData policy row_level_security enable "MaskingRLS()"
```

### 7.3 Restricted View Access

```kusto
// Restrict table access
.alter table ConfidentialData policy restricted_view_access true

// Grant unrestricted viewer access to specific group
.add database MyDatabase unrestrictedviewers ('aadgroup=security-team@contoso.com')
```

---

## 8. External Tables

### 8.1 Azure Blob Storage External Table

```kusto
// Create external table pointing to Azure Blob Storage
.create external table ExternalLogs (
    Timestamp: datetime,
    Level: string,
    Message: string
)
kind=storage
dataformat=parquet
(
    h@'https://mystorageaccount.blob.core.windows.net/logs;managed_identity=system'
)
with (folder = "External")
```

### 8.2 Azure Data Lake External Table

```kusto
.create external table DataLakeEvents (
    EventId: guid,
    EventTime: datetime,
    EventType: string,
    Payload: dynamic
)
kind=storage
dataformat=json
(
    h@'https://mydatalake.dfs.core.windows.net/events/;managed_identity=system'
)
with (
    folder = "External",
    docstring = "Events from Data Lake"
)
```

---

## 9. Continuous Export

### 9.1 Export to External Table

```kusto
// Create external table for export
.create external table ExportedMetrics (
    Day: datetime,
    MetricName: string,
    Value: real
)
kind=storage
dataformat=parquet
(
    h@'https://mystorageaccount.blob.core.windows.net/exports;managed_identity=system'
)

// Create continuous export
.create-or-alter continuous-export MetricsExport
over (DailyMetrics)
to table ExportedMetrics
with (
    intervalBetweenRuns=1h,
    forcedLatency=10m,
    sizeLimit=1073741824
)
<|
    DailyMetrics
    | project Day = bin(Timestamp, 1d), MetricName, Value
```

### 9.2 Manage Continuous Export

```kusto
// Show all continuous exports
.show continuous-exports

// Show specific export
.show continuous-export MetricsExport

// Show export failures
.show continuous-export MetricsExport failures

// Disable/enable
.disable continuous-export MetricsExport
.enable continuous-export MetricsExport

// Drop continuous export
.drop continuous-export MetricsExport
```

---

## 10. Diagnostics and Monitoring

### 10.1 Show Operations

```kusto
// Show recent operations
.show operations
| where StartedOn > ago(1d)
| order by StartedOn desc

// Show specific operation
.show operations 12345678-1234-1234-1234-123456789012
```

### 10.2 Query Diagnostics

```kusto
// Show running queries
.show running queries

// Cancel a query
.cancel query "query-client-activity-id"

// Show recent commands
.show commands
| where StartedOn > ago(1h)

// Show commands and queries together
.show commands-and-queries
| where StartedOn > ago(1h)
| summarize count() by CommandType
```

### 10.3 Ingestion Diagnostics

```kusto
// Show recent ingestion failures
.show ingestion failures
| where FailedOn > ago(1d)
| summarize FailureCount=count() by Table, FailureKind
| order by FailureCount desc

// Show ingestion failures with details
.show ingestion failures
| where FailedOn > ago(1h)
| project FailedOn, Table, FailureKind, Details
```

### 10.4 Extent Information

```kusto
// Show table extents
.show table MyTable extents
| summarize
    ExtentCount = count(),
    TotalRows = sum(RowCount),
    TotalSize = sum(OriginalSize)

// Show hot extents only
.show table MyTable extents hot
| summarize count()

// Database extent summary
.show database MyDatabase extents
| summarize
    ExtentCount = count(),
    TotalRows = sum(RowCount),
    TotalSizeGB = sum(OriginalSize) / 1024 / 1024 / 1024
  by TableName
| order by TotalSizeGB desc
```

### 10.5 Materialized View Health

```kusto
// Show all materialized views
.show materialized-views
| project Name, SourceTable, IsHealthy, IsEnabled, MaterializedTo

// Show materialized view failures
.show materialized-view MyView failures
| where Timestamp > ago(1d)
| order by Timestamp desc
```

---

## Quick Copy-Paste Templates

### New Table Setup Template

```kusto
// 1. Create table
.create table MyNewTable (
    Id: long,
    Timestamp: datetime,
    Category: string,
    Value: real,
    Metadata: dynamic
) with (folder = "MyFolder", docstring = "Description")

// 2. Set retention (30 days)
.alter-merge table MyNewTable policy retention softdelete = 30d

// 3. Set caching (14 days hot)
.alter table MyNewTable policy caching hot = 14d

// 4. Create JSON mapping
.create table MyNewTable ingestion json mapping "DefaultMapping"
'[{"column":"Id","path":"$.id"},{"column":"Timestamp","path":"$.timestamp"},{"column":"Category","path":"$.category"},{"column":"Value","path":"$.value"},{"column":"Metadata","path":"$"}]'
```

### Update Policy Template

```kusto
// 1. Source table
.create table RawData (Data: dynamic)

// 2. Target table
.create table ProcessedData (Timestamp:datetime, Category:string, Value:real)

// 3. Transform function
.create function TransformData() { RawData | extend Timestamp=todatetime(Data.ts), Category=tostring(Data.cat), Value=todouble(Data.val) | project Timestamp, Category, Value }

// 4. Update policy
.alter table ProcessedData policy update @'[{"IsEnabled":true,"Source":"RawData","Query":"TransformData()","IsTransactional":true}]'
```

### Materialized View Template

```kusto
.create materialized-view DailyAggregates on table SourceTable
{
    SourceTable
    | summarize
        Count = count(),
        Sum = sum(Value),
        Avg = avg(Value)
      by Category, bin(Timestamp, 1d)
}
```

---

*Generated: 2026-02-13 | Source: Azure Data Explorer Documentation*
