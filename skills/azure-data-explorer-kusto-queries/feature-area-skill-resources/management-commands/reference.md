# Azure Data Explorer Management Commands Reference

## Overview

Management commands (control commands) in Azure Data Explorer use the `.command` syntax to manage cluster, database, and table configuration. There are 297+ documented management commands covering schema management, policies, materialized views, and security roles.

---

## 1. Schema Management

### 1.1 Table Commands

#### Create Table

Creates a new empty table with a defined schema.

```kusto
.create table TableName (Column1:Type1, Column2:Type2, ...)
```

**With properties:**
```kusto
.create table TableName (Column1:Type1, Column2:Type2)
with (docstring = "Description", folder = "FolderName")
```

**Permission required:** Database User

#### Create Multiple Tables

```kusto
.create tables
    Table1 (Col1:string, Col2:int),
    Table2 (Col1:datetime, Col2:long)
```

#### Create-Merge Table

Creates a table if it doesn't exist, or merges schema if it does:

```kusto
.create-merge table TableName (Column1:Type1, Column2:Type2)
```

#### Alter Table

Modifies table schema (add/remove columns):

```kusto
.alter table TableName (Column1:Type1, Column2:Type2, NewColumn:Type3)
```

#### Alter-Merge Table

Adds columns without affecting existing ones:

```kusto
.alter-merge table TableName (NewColumn:Type)
```

#### Drop Table

```kusto
.drop table TableName [ifexists]
```

#### Rename Table

```kusto
.rename table OldName to NewName
```

#### Show Tables

```kusto
.show tables
.show table TableName
.show table TableName schema as json
```

---

### 1.2 Column Commands

#### Alter Column Type

```kusto
.alter column TableName.ColumnName type = newType
```

#### Drop Column

```kusto
.drop column TableName.ColumnName
.drop table TableName columns (Col1, Col2)
```

#### Rename Column

```kusto
.rename column TableName.OldName to NewName
.rename columns Col1 = Table1.OldCol1, Col2 = Table2.OldCol2
```

#### Alter Column Docstring

```kusto
.alter column TableName.ColumnName docstring "Description"
```

---

### 1.3 Function Commands

#### Create Function

```kusto
.create function FunctionName() { Query }
```

**With parameters:**
```kusto
.create function
with (docstring = "Description", folder = "Functions")
MyFunction(param1:string, param2:datetime)
{
    TableName
    | where Column1 == param1 and Timestamp > param2
}
```

**Supported properties:**
- `docstring`: Description for UI
- `folder`: Folder for categorization
- `view`: If true, function participates in search and union * operations
- `skipvalidation`: Skip validation (useful for cross-cluster functions)

#### Create-or-Alter Function

```kusto
.create-or-alter function FunctionName() { Query }
```

#### Alter Function

```kusto
.alter function FunctionName() { NewQuery }
```

#### Drop Function

```kusto
.drop function FunctionName
```

#### Show Functions

```kusto
.show functions
.show function FunctionName
```

---

### 1.4 Ingestion Mappings

#### Create Ingestion Mapping

```kusto
.create table TableName ingestion json mapping "MappingName"
'[{"column":"Col1","path":"$.field1"},{"column":"Col2","path":"$.field2"}]'
```

**Mapping types:** json, csv, avro, parquet, orc, w3clogfile

#### Show Ingestion Mappings

```kusto
.show table TableName ingestion json mappings
.show table TableName ingestion json mapping "MappingName"
```

#### Drop Ingestion Mapping

```kusto
.drop table TableName ingestion json mapping "MappingName"
```

---

## 2. Policy Management

### 2.1 Retention Policy

Controls automatic data removal based on age.

**Policy object properties:**
- `SoftDeletePeriod`: Time data is kept (default: 1000 years)
- `Recoverability`: Enabled/Disabled (default: Enabled)

#### Show Retention Policy

```kusto
.show table TableName policy retention
.show database DatabaseName policy retention
```

#### Alter Retention Policy

```kusto
// Set 30-day retention
.alter table TableName policy retention
```
'{"SoftDeletePeriod": "30.00:00:00", "Recoverability": "Enabled"}'
```

// Using shorthand syntax
.alter-merge table TableName policy retention softdelete = 30d recoverability = enabled
```

#### Delete Retention Policy

```kusto
.delete table TableName policy retention
```

---

### 2.2 Caching Policy (Hot/Cold Cache)

Controls which data is kept in SSD (hot) vs blob storage (cold).

#### Show Caching Policy

```kusto
.show table TableName policy caching
.show database DatabaseName policy caching
```

#### Alter Caching Policy

```kusto
// Keep last 7 days in hot cache
.alter table TableName policy caching hot = 7d

// Database level
.alter database DatabaseName policy caching hot = 14d
```

#### Query Hot Cache Only

```kusto
set query_datascope = "hotcache";
TableName | where ...

// Or inline
TableName datascope=hotcache | where ...
```

---

### 2.3 Partitioning Policy

Optimizes query performance for specific access patterns.

**Partition key types:**
- **Hash partition:** For high-cardinality string/guid columns
- **Uniform range datetime:** For datetime columns with out-of-order data

#### Show Partitioning Policy

```kusto
.show table TableName policy partitioning
```

#### Alter Partitioning Policy

**Hash partition example:**
```kusto
.alter table TableName policy partitioning
```
{
  "PartitionKeys": [
    {
      "ColumnName": "tenant_id",
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

**Datetime partition example:**
```kusto
.alter table TableName policy partitioning
```
{
  "PartitionKeys": [
    {
      "ColumnName": "timestamp",
      "Kind": "UniformRange",
      "Properties": {
        "Reference": "2021-01-01T00:00:00",
        "RangeSize": "1.00:00:00",
        "OverrideCreationTime": false
      }
    }
  ]
}
```

---

### 2.4 Merge Policy

Controls how extents (data shards) are merged.

**Properties:**
- `RowCountUpperBoundForMerge`: Max rows in merged extent (default: 16M)
- `MaxExtentsToMerge`: Max extents per operation (default: 100)
- `MaxRangeInHours`: Max time difference for merge (default: 24)
- `AllowRebuild`: Enable rebuild operations (default: true)
- `AllowMerge`: Enable merge operations (default: true)

#### Show/Alter Merge Policy

```kusto
.show table TableName policy merge

.alter table TableName policy merge
```
{
  "RowCountUpperBoundForMerge": 16000000,
  "MaxRangeInHours": 24,
  "AllowRebuild": true,
  "AllowMerge": true
}
```

---

### 2.5 Sharding Policy

Controls extent creation during ingestion.

**Properties:**
- `ShardEngineMaxRowCount`: Max rows per extent (default: 1,048,576)
- `ShardEngineMaxExtentSizeInMb`: Max compressed size (default: 8GB)

```kusto
.show table TableName policy sharding
.alter table TableName policy sharding
```
{"ShardEngineMaxRowCount": 1000000}
```

---

### 2.6 Ingestion Batching Policy

Controls how ingestion batches are formed.

**Properties:**
- `MaximumBatchingTimeSpan`: Max time to wait (default: 5 min)
- `MaximumNumberOfItems`: Max items per batch (default: 500)
- `MaximumRawDataSizeMB`: Max size per batch (default: 1024 MB)

```kusto
.show table TableName policy ingestionbatching

.alter table TableName policy ingestionbatching
```
{
  "MaximumBatchingTimeSpan": "00:01:00",
  "MaximumNumberOfItems": 100,
  "MaximumRawDataSizeMB": 256
}
```

---

### 2.7 Streaming Ingestion Policy

Enables low-latency streaming ingestion.

```kusto
.show table TableName policy streamingingestion

// Enable streaming ingestion
.alter table TableName policy streamingingestion enable

// Disable streaming ingestion
.alter table TableName policy streamingingestion disable
```

---

### 2.8 Update Policy

Automatically transforms data during ingestion.

**Policy properties:**
- `IsEnabled`: Enable/disable the policy
- `Source`: Source table name
- `Query`: Transformation query
- `IsTransactional`: Atomic ingestion (default: false)
- `PropagateIngestionProperties`: Pass extent tags to target

#### Create Update Policy

```kusto
// Create transformation function
.create function ExtractLogs()
{
    SourceTable
    | parse Message with "[" Timestamp:datetime "]" Content:string
}

// Apply update policy
.alter table TargetTable policy update
@'[{"IsEnabled": true, "Source": "SourceTable", "Query": "ExtractLogs()", "IsTransactional": true}]'
```

#### Show/Delete Update Policy

```kusto
.show table TargetTable policy update
.delete table TargetTable policy update
```

---

### 2.9 Row Level Security Policy

Controls row-level access based on user identity.

```kusto
// Enable RLS
.alter table TableName policy row_level_security enable "TableName | where Region == 'US'"

// Using function
.create function RLSFunction()
{
    TableName
    | where current_principal_is_member_of('aadgroup=team@domain.com')
}

.alter table TableName policy row_level_security enable "RLSFunction()"

// Disable RLS
.alter table TableName policy row_level_security disable
```

---

### 2.10 Restricted View Access Policy

Limits table access to specific principals.

```kusto
.alter table TableName policy restricted_view_access true
.alter table TableName policy restricted_view_access false
```

---

### 2.11 Auto Delete Policy

Automatically deletes data based on datetime column.

```kusto
.alter table TableName policy auto_delete
@'{"ExpiryDate": "2025-01-01", "DeleteIfNotEmpty": false}'
```

---

## 3. Materialized Views

### 3.1 Create Materialized View

```kusto
.create materialized-view ViewName on table SourceTable
{
    SourceTable
    | summarize count(), dcount(User) by bin(Timestamp, 1d), Customer
}
```

**With backfill:**
```kusto
.create async materialized-view with (backfill=true) ViewName on table SourceTable
{
    SourceTable
    | summarize arg_max(Timestamp, *) by UserId
}
```

**With lookback (for deduplication):**
```kusto
.create materialized-view with (lookback=6h) DedupeView on table SourceTable
{
    SourceTable
    | summarize take_any(*) by EventId
}
```

### 3.2 Supported Aggregation Functions

- `count`, `countif`
- `dcount`, `dcountif`
- `min`, `max`
- `avg`, `avgif`
- `sum`, `sumif`
- `arg_max`, `arg_min`
- `take_any`, `take_anyif`
- `hll`
- `make_set`, `make_list`, `make_bag`
- `percentile`, `percentiles`

### 3.3 Alter Materialized View

```kusto
.alter materialized-view ViewName on table SourceTable
{
    NewQuery
}
```

### 3.4 Enable/Disable Materialized View

```kusto
.disable materialized-view ViewName
.enable materialized-view ViewName
```

### 3.5 Show Materialized Views

```kusto
.show materialized-views
.show materialized-view ViewName
.show materialized-view ViewName extents
.show materialized-view ViewName failures
```

### 3.6 Drop Materialized View

```kusto
.drop materialized-view ViewName
```

### 3.7 Query Materialized Views

```kusto
// Query entire view (includes delta)
ViewName | where ...

// Query materialized part only (better performance)
materialized_view("ViewName") | where ...
```

---

## 4. Security Roles

### 4.1 Role Types

| Role | Permissions |
|------|-------------|
| `admins` | View, modify, remove object and subobjects |
| `users` | View object and create new subobjects |
| `viewers` | View object (unless RestrictedViewAccess is on) |
| `unrestrictedviewers` | View even with RestrictedViewAccess |
| `ingestors` | Ingest data without query access |
| `monitors` | View metadata (schemas, operations, permissions) |

### 4.2 Show Principals

```kusto
.show database DatabaseName principals
.show table TableName principals
.show cluster principal roles
```

### 4.3 Add Principals

```kusto
// Add database admin
.add database DatabaseName admins ('aaduser=user@domain.com')

// Add table ingestor
.add table TableName ingestors ('aadapp=app-id;tenant-id')

// Add multiple principals
.add database DatabaseName viewers ('aadgroup=group@domain.com', 'aaduser=user2@domain.com')
```

### 4.4 Drop Principals

```kusto
.drop database DatabaseName admins ('aaduser=user@domain.com')
.drop table TableName ingestors ('aadapp=app-id')
```

### 4.5 Set Principals (Replace All)

```kusto
.set database DatabaseName viewers ('aaduser=user1@domain.com', 'aaduser=user2@domain.com')
```

---

## 5. Data Management Commands

### 5.1 Show Extents

```kusto
.show table TableName extents
.show table TableName extents hot
.show database DatabaseName extents
```

### 5.2 Move/Merge Extents

```kusto
.move extents from table SourceTable to table TargetTable (extent-id-1, extent-id-2)
.merge table TableName extents (extent-id-1, extent-id-2)
```

### 5.3 Drop Extents

```kusto
.drop extents from table TableName
.drop extents (extent-id-1, extent-id-2) from table TableName
```

### 5.4 Clear Table Data

```kusto
.clear table TableName data
```

---

## 6. Continuous Export

### 6.1 Create Continuous Export

```kusto
.create-or-alter continuous-export ExportName
over (TableName)
to table ExternalTableName
with (intervalBetweenRuns=1h)
<| TableName | where ...
```

### 6.2 Show/Disable/Enable

```kusto
.show continuous-exports
.show continuous-export ExportName
.disable continuous-export ExportName
.enable continuous-export ExportName
```

---

## 7. External Tables

### 7.1 Create External Table (Azure Storage)

```kusto
.create external table ExternalTableName (Col1:string, Col2:datetime)
kind=storage
dataformat=parquet
(
    h@'https://storageaccount.blob.core.windows.net/container;secretkey'
)
```

### 7.2 Show External Tables

```kusto
.show external tables
.show external table ExternalTableName
.show external table ExternalTableName artifacts
```

---

## 8. Operations and Diagnostics

### 8.1 Show Operations

```kusto
.show operations
.show operations operation-id
```

### 8.2 Show Running Queries

```kusto
.show running queries
.cancel query "query-id"
```

### 8.3 Show Commands

```kusto
.show commands
.show commands-and-queries
```

### 8.4 Show Ingestion Failures

```kusto
.show ingestion failures
| where FailedOn > ago(1d)
```

---

## Quick Reference Table

| Category | Show | Alter | Delete |
|----------|------|-------|--------|
| Retention | `.show table T policy retention` | `.alter table T policy retention {...}` | `.delete table T policy retention` |
| Caching | `.show table T policy caching` | `.alter table T policy caching hot = 7d` | `.delete table T policy caching` |
| Partitioning | `.show table T policy partitioning` | `.alter table T policy partitioning {...}` | `.delete table T policy partitioning` |
| Merge | `.show table T policy merge` | `.alter table T policy merge {...}` | `.delete table T policy merge` |
| Update | `.show table T policy update` | `.alter table T policy update [...]` | `.delete table T policy update` |
| RLS | `.show table T policy row_level_security` | `.alter table T policy row_level_security enable "..."` | N/A (use disable) |

---

*Generated: 2026-02-13 | Source: Azure Data Explorer Documentation*
