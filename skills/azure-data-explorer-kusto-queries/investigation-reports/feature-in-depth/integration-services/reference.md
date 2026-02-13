# Integration Services Reference

## Overview

Azure Data Explorer (ADX) provides extensive integration capabilities with the Azure ecosystem and external services. This reference covers Azure Monitor integration, Synapse Analytics integration, Azure Machine Learning, Logic Apps, Azure Functions, Power Automate, and cross-product queries.

---

## Azure Monitor Integration

### Overview

Azure Data Explorer integrates deeply with Azure Monitor for comprehensive monitoring of ADX clusters and cross-service data analysis.

### Azure Data Explorer Clusters Insights

ADX Clusters Insights provides a unified monitoring experience accessible through Azure Monitor:

**Features:**
- **At-scale perspective**: Snapshot view of cluster metrics across subscriptions
- **Drill-down analysis**: Detailed cluster-specific analysis
- **Customization**: Custom workbooks and dashboards

**Key Metrics Monitored:**
- Query performance (duration, concurrent queries, throttled queries)
- Ingestion performance (latency, success/failure rates, volume)
- Streaming ingestion (data rate, duration, request rate)
- Export operations (records exported, lateness, pending count)
- Cluster boundaries (CPU, ingestion, cache utilization)

**Documentation Path:** `data-explorer-insights.md`

### Diagnostic Logs

Enable diagnostic logs for comprehensive monitoring:

| Log Type | Description |
|----------|-------------|
| Command | Control command execution logs |
| Query | Query execution logs |
| SucceededIngestion | Successful ingestion operations |
| FailedIngestion | Failed ingestion operations |
| IngestionBatching | Batching policy behavior |
| TableDetails | Table-level statistics |
| TableUsageStatistics | Table usage patterns |

### Azure Monitor Metrics

Raw metrics data is stored in Azure Monitor and can be:
- Queried directly via Metrics and Insights pages
- Migrated to Log Analytics workspace via diagnostic settings

> **Note**: Data migrated to Log Analytics may have slight precision loss due to rounding (error margin < 1%).

---

## Azure Synapse Analytics Integration

### Apache Spark Connector

The Kusto Spark Connector enables bidirectional data flow between ADX and Spark clusters (including Synapse Spark).

**Key Capabilities:**
- Write to ADX via queued or streaming ingestion
- Read from ADX with column pruning and predicate pushdown
- Standard Spark source/sink operations (`write`, `read`, `writeStream`)

**Supported Spark Versions:**
- Spark 2.4+ with Scala 2.11
- Spark 3+ with Scala 2.12

**Installation:**
```xml
<!-- Maven Dependency -->
<dependency>
    <groupId>com.microsoft.azure.kusto</groupId>
    <artifactId>spark-kusto-connector</artifactId>
</dependency>
```

**GitHub Repository:** [azure-kusto-spark](https://github.com/Azure/azure-kusto-spark)

**Documentation Path:** `spark-connector.md`

### Data Lake Integration

ADX integrates with Azure Data Lake Storage (ADLS Gen1 and Gen2) for querying external data:

**Features:**
- Query data without ingestion via external tables
- Fast, cached, indexed access to external storage
- Query both ingested and external data simultaneously

**Best Practices:**
- Use columnar formats (Parquet recommended over ORC)
- Optimize file sizes (100MB - 1GB per file)
- Colocate external data in same Azure region
- Use compression (Snappy, Gzip)
- Implement folder-based partitioning

**Documentation Path:** `data-lake-query-data.md`

---

## Azure Machine Learning Integration

### Python Plugin

ADX supports inline Python execution for ML workloads:

```kusto
range x from 1 to 360 step 1
| evaluate python(
    typeof(x:long, y:double),
    'import numpy as np\n'
    'result = df\n'
    'result["y"] = np.sin(df["x"]/180*np.pi)\n'
)
```

**Requirements:**
- Enable Python plugin on cluster
- Configure sandbox policy for ML packages

### ONNX Model Inference

Use `predict_onnx_fl()` function for ML model inference:

```kusto
// Load and apply ONNX model
let model = externaldata(m:dynamic) [@'model.onnx'];
MyTable
| invoke predict_onnx_fl(model, features_cols, output_col)
```

### Built-in ML Plugins

| Plugin | Purpose |
|--------|---------|
| `autocluster` | Automatic clustering of data |
| `basket` | Association rule mining |
| `diffpatterns` | Pattern differentiation between datasets |
| `diffpatterns_text` | Text pattern differentiation |

**Documentation Paths:**
- `kusto/query/python-plugin.md`
- `kusto/functions-library/predict-onnx-fl.md`

---

## Logic Apps Integration

### Azure Kusto Logic App Connector

Logic Apps can execute KQL queries and management commands as part of automated workflows.

**Capabilities:**
- Execute KQL queries
- Run management commands
- Scheduled and triggered workflows
- Integration with 400+ other connectors

**Authentication Methods:**
- User credentials (OAuth)
- Service Principal (Azure AD App)
- Managed Identity

**Use Cases:**
- Automated data processing pipelines
- Alert-driven workflows
- Scheduled reporting
- Cross-service data orchestration

**Documentation Reference:** Referenced in `flow.md` as alternative to Power Automate

---

## Azure Functions Integration

### Integration Patterns

Azure Functions integrate with ADX through:

1. **Event Hub Triggers**: Process events and ingest to ADX
2. **Timer Triggers**: Scheduled queries and data processing
3. **HTTP Triggers**: API endpoints for ADX operations
4. **Blob Triggers**: Process new files and ingest to ADX

### SDK Support

All ADX SDKs support Azure Functions:
- .NET SDK (`Kusto.Data`, `Kusto.Ingest`)
- Python SDK (`azure-kusto-python`)
- Node.js SDK (`azure-kusto-node`)
- Java SDK (`azure-kusto-java`)

### Managed Identity Authentication

Functions can authenticate to ADX using managed identity:

```csharp
// .NET Example
var kustoUri = "https://mycluster.region.kusto.windows.net";
var builder = new KustoConnectionStringBuilder(kustoUri)
    .WithAadManagedIdentity("system"); // or user-assigned identity client ID
```

---

## Power Automate (Microsoft Flow) Integration

### Azure Data Explorer Connector

The ADX connector for Power Automate enables flow automation:

**Available Actions:**
| Action | Description |
|--------|-------------|
| Run KQL query | Execute a query and iterate over results |
| Run KQL query and render a chart | Visualize query results |
| Run async management command | Execute long-running commands |
| Run management command and render a chart | Visualize command output |
| Run show management command | Execute `.show` commands |

**Authentication:**
- User credentials (OAuth)
- Service Principal (Azure AD App)

**Limitations:**
- 50,000 records per request maximum
- 32 MB maximum data size per request
- 8-minute timeout for synchronous requests
- 60-minute timeout for async requests

**Documentation Paths:**
- `flow.md`
- `flow-usage.md`

### Common Use Cases

1. **Threshold Alerts**: Send notifications when metrics exceed limits
2. **Scheduled Reports**: Email daily/weekly reports with charts
3. **Data Export/Import**: Move data between ADX and other databases
4. **Conditional Logic**: Branch workflows based on query results
5. **Integration with Other Services**: Push data to Power BI, Slack, Teams

---

## Cross-Product Queries

### Log Analytics and Application Insights

ADX can query data from Azure Monitor Logs (Log Analytics) and Application Insights:

**Cross-Service Query Syntax:**
```kusto
// Query Log Analytics workspace
cluster('https://ade.loganalytics.io/subscriptions/<sub>/resourcegroups/<rg>/providers/microsoft.operationalinsights/workspaces/<workspace>').database('<workspace>')
| <query>

// Query Application Insights
cluster('https://ade.applicationinsights.io/subscriptions/<sub>/resourcegroups/<rg>/providers/microsoft.insights/components/<app>').database('<app>')
| <query>
```

### SQL Database Integration

Use `sql_request` plugin to query SQL databases inline:

```kusto
evaluate sql_request(
    'Server=tcp:contoso.database.windows.net,1433;'
    'Authentication="Active Directory Integrated";'
    'Initial Catalog=MyDatabase;',
    'SELECT * FROM [dbo].[MyTable]'
) : (Id:long, Name:string)
| where Id > 0
```

**Authentication Methods:**
- Azure AD Integrated
- Managed Identity
- Username/Password
- Azure AD Access Token

**Documentation Path:** `kusto/query/sql-request-plugin.md`

### Cosmos DB Integration

Use `cosmosdb_sql_request` plugin for Cosmos DB queries:

```kusto
evaluate cosmosdb_sql_request(
    'AccountEndpoint=https://myaccount.documents.azure.com/;'
    'Database=MyDatabase;Collection=MyCollection;',
    'SELECT c.Id, c.Name FROM c'
) : (Id:long, Name:string)
```

**Authentication Methods:**
- Managed Identity (Recommended)
- Azure Resource Manager resource ID
- Account Key
- Azure AD Token

**Documentation Path:** `kusto/query/cosmosdb-plugin.md`

### HTTP API Integration

Use `http_request` plugin for REST API calls:

```kusto
let Uri = "https://api.example.com/data";
evaluate http_request(Uri)
| project ResponseBody
| mv-expand data = ResponseBody.items
```

**Requirements:**
- Enable `http_request` plugin
- Configure callout policy for allowed URIs

**Documentation Path:** `kusto/query/http-request-plugin.md`

---

## Azure Data Factory Integration

### Available Activities

| Activity | Description |
|----------|-------------|
| Copy Activity | Transfer data to/from ADX |
| Lookup Activity | Execute queries and use results in pipelines |
| Command Activity | Run management commands |

### Copy vs Command Activity

**Copy Data FROM ADX:**
| Feature | Copy Activity | .export Command |
|---------|---------------|-----------------|
| Data Stores | Wide variety | ADLSv2, Blob, SQL |
| Performance | Centralized | Distributed (faster) |
| Server Limits | Configurable | Extended by default |

**Copy Data TO ADX:**
| Feature | Copy Activity | Ingest Commands |
|---------|---------------|-----------------|
| Sources | Wide variety | Limited |
| Performance | High volume optimized | Small datasets |
| Use Case | Production workloads | Quick operations |

**Documentation Paths:**
- `data-factory-integration.md`
- `data-factory-load-data.md`
- `data-factory-command-activity.md`

---

## External Tables

### Overview

External tables reference data stored outside ADX without ingestion:

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

### Querying External Tables

```kusto
external_table("MyExternalTable")
| where Timestamp > ago(30d)
| summarize count() by bin(Timestamp, 1h)
```

**Documentation Paths:**
- `external-table.md`
- `data-lake-query-data.md`
- `kusto/management/external-tables-azure-storage.md`

---

## Key Documentation Files

| Feature | Primary Documentation |
|---------|----------------------|
| Azure Monitor | `data-explorer-insights.md` |
| Spark Integration | `spark-connector.md` |
| Data Lake | `data-lake-query-data.md` |
| Power Automate | `flow.md`, `flow-usage.md` |
| Data Factory | `data-factory-integration.md` |
| SQL Plugin | `kusto/query/sql-request-plugin.md` |
| Cosmos DB Plugin | `kusto/query/cosmosdb-plugin.md` |
| HTTP Plugin | `kusto/query/http-request-plugin.md` |
| External Tables | `external-table.md` |

---

## Summary

Azure Data Explorer provides comprehensive integration capabilities:

| Integration Type | Primary Method | Key Benefits |
|-----------------|----------------|--------------|
| Azure Monitor | Native integration | Unified monitoring |
| Synapse/Spark | Kusto Spark Connector | Big data analytics |
| Azure ML | Python plugin, ONNX | In-database ML |
| Logic Apps | Logic App Connector | Workflow automation |
| Functions | SDKs + Managed Identity | Event-driven processing |
| Power Automate | ADX Connector | Low-code automation |
| Cross-service | Cluster() function | Unified querying |
| External data | External tables | Query without ingestion |

---

*Document generated from Azure Data Explorer documentation analysis*
*Phase: 3 - Feature In-Depth Research*
