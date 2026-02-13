# Integration Services Reference

## Overview

Azure Data Explorer integrates with Azure Monitor, Synapse Analytics, Logic Apps, Power Automate, Azure Functions, Data Factory, and external data sources through query plugins and external tables.

---

## Azure Monitor Integration

### Cluster Monitoring

ADX Clusters Insights provides unified monitoring through Azure Monitor:

- **At-scale view**: Cluster metrics across subscriptions
- **Key metrics**: Query performance, ingestion latency, streaming rates, export operations, resource utilization (CPU, cache)
- **Diagnostic logs**: Command, Query, SucceededIngestion, FailedIngestion, IngestionBatching, TableDetails, TableUsageStatistics

### Cross-Service Queries

```kusto
// Query Log Analytics workspace from ADX
cluster('https://ade.loganalytics.io/subscriptions/<sub>/resourcegroups/<rg>/providers/microsoft.operationalinsights/workspaces/<workspace>').database('<workspace>')
.<TableName>
| <query>

// Query Application Insights from ADX
cluster('https://ade.applicationinsights.io/subscriptions/<sub>/resourcegroups/<rg>/providers/microsoft.insights/components/<app>').database('<app>')
.<TableName>
| <query>
```

---

## Azure Synapse Analytics Integration

### Spark Connector

Bidirectional data flow between ADX and Spark clusters (including Synapse Spark).

| Capability | Description |
|------------|-------------|
| Write modes | Queued ingestion, streaming ingestion |
| Read optimizations | Column pruning, predicate pushdown |
| Operations | `write`, `read`, `writeStream` |
| Spark versions | 2.4+ (Scala 2.11), 3+ (Scala 2.12) |

**Maven Dependency:**
```xml
<dependency>
    <groupId>com.microsoft.azure.kusto</groupId>
    <artifactId>spark-kusto-connector</artifactId>
</dependency>
```

### Data Lake Integration

Query ADLS Gen1/Gen2 data without ingestion via external tables:

- Use columnar formats (Parquet preferred)
- Optimal file size: 100MB - 1GB
- Colocate data in same Azure region
- Use compression (Snappy, Gzip)
- Implement folder-based partitioning

---

## Logic Apps Integration

### Kusto Connector Capabilities

| Feature | Description |
|---------|-------------|
| Query execution | Run KQL queries in workflows |
| Management commands | Execute control commands |
| Workflow integration | Connect with 400+ other connectors |

### Authentication Methods

- User credentials (OAuth)
- Service Principal (Azure AD App)
- Managed Identity

### Use Cases

- Automated data processing pipelines
- Alert-driven workflows
- Scheduled reporting
- Cross-service data orchestration

---

## Power Automate Integration

### Available Actions

| Action | Description |
|--------|-------------|
| Run KQL query | Execute query, iterate over results |
| Run KQL query and render chart | Visualize query results |
| Run async management command | Execute long-running commands |
| Run management command and render chart | Visualize command output |
| Run show management command | Execute `.show` commands |

### Limits

| Limit | Value |
|-------|-------|
| Records per request | 50,000 max |
| Data size per request | 32 MB max |
| Sync timeout | 8 minutes |
| Async timeout | 60 minutes |

---

## Azure Functions Integration

### Integration Patterns

| Trigger Type | Use Case |
|--------------|----------|
| Event Hub | Process events, ingest to ADX |
| Timer | Scheduled queries, data processing |
| HTTP | API endpoints for ADX operations |
| Blob | Process new files, ingest to ADX |

### SDK Support

- .NET: `Kusto.Data`, `Kusto.Ingest`
- Python: `azure-kusto-python`
- Node.js: `azure-kusto-node`
- Java: `azure-kusto-java`

### Managed Identity Authentication

```csharp
var builder = new KustoConnectionStringBuilder(kustoUri)
    .WithAadManagedIdentity("system"); // or user-assigned identity client ID
```

---

## Azure Data Factory Integration

### Available Activities

| Activity | Description |
|----------|-------------|
| Copy Activity | Transfer data to/from ADX |
| Lookup Activity | Execute queries, use results in pipelines |
| Command Activity | Run management commands |

### Copy vs Command

**FROM ADX:**
- Copy Activity: Wide data store variety, centralized performance
- `.export` Command: Distributed (faster), limited to ADLSv2/Blob/SQL

**TO ADX:**
- Copy Activity: High volume optimized, wide sources
- Ingest Commands: Small datasets, quick operations

---

## Cross-Product Query Plugins

### sql_request Plugin

Query Azure SQL databases inline:

```kusto
evaluate sql_request(
    'Server=tcp:server.database.windows.net,1433;'
    'Authentication="Active Directory Integrated";'
    'Initial Catalog=DB;',
    'SELECT * FROM Table WHERE Active = 1'
) : (Id:long, Name:string)
```

**Authentication**: Azure AD Integrated, Managed Identity, Username/Password, Azure AD Token

### cosmosdb_sql_request Plugin

Query Cosmos DB inline:

```kusto
evaluate cosmosdb_sql_request(
    'AccountEndpoint=https://account.documents.azure.com/;'
    'Database=DB;Collection=Col;',
    'SELECT c.Id, c.Name FROM c'
) : (Id:long, Name:string)
```

**Authentication**: Managed Identity (recommended), ARM resource ID, Account Key, Azure AD Token

### http_request Plugin

Call REST APIs inline:

```kusto
evaluate http_request('https://api.example.com/data')
| project ResponseBody
| mv-expand data = ResponseBody.items
```

**Requirements**: Enable plugin, configure callout policy for allowed URIs

---

## External Tables

### Overview

Reference data stored outside ADX without ingestion.

**Supported Storage:**
- Azure Blob Storage
- Azure Data Lake Storage Gen1/Gen2
- Azure SQL (via external SQL tables)

### Creating External Tables

```kusto
.create external table MyExternalTable (
    Timestamp: datetime,
    ProductId: string,
    Value: real
)
kind=blob
partition by (Date: datetime = bin(Timestamp, 1d))
dataformat=parquet
(
    h@'https://storage.blob.core.windows.net/container;StorageKey'
)
```

### Querying

```kusto
external_table("MyExternalTable")
| where Timestamp > ago(30d)
| summarize count() by bin(Timestamp, 1h)
```

---

## Integration Summary

| Integration | Primary Method | Key Benefits |
|-------------|----------------|--------------|
| Azure Monitor | Native integration | Unified monitoring, diagnostics |
| Synapse/Spark | Kusto Spark Connector | Big data analytics, bidirectional flow |
| Logic Apps | Logic App Connector | Enterprise workflow automation |
| Power Automate | ADX Connector | Low-code automation, alerts |
| Functions | SDKs + Managed Identity | Event-driven processing, flexibility |
| Data Factory | Copy/Command Activities | Large-scale ETL, batch processing |
| Cross-service | `cluster()` function | Unified querying across services |
| External data | External tables | Query without ingestion cost |
| SQL/Cosmos/HTTP | Query plugins | Real-time data enrichment |
