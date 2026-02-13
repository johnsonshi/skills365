# Integration Services Examples

## Overview

This document provides practical, ready-to-use examples for integrating Azure Data Explorer with Azure Monitor, Synapse Analytics, Logic Apps, Power Automate, and cross-service queries.

---

## Azure Monitor Queries

### Query ADX Diagnostic Logs in Log Analytics

When ADX diagnostic logs are sent to a Log Analytics workspace, you can query them:

```kusto
// Query ADX command execution logs
ADXCommand
| where TimeGenerated > ago(1h)
| summarize CommandCount=count(), AvgDuration=avg(Duration) by User, CommandType
| order by CommandCount desc

// Query ADX query performance
ADXQuery
| where TimeGenerated > ago(24h)
| where State == "Completed"
| summarize
    QueryCount=count(),
    AvgDuration=avg(Duration),
    P95Duration=percentile(Duration, 95),
    TotalScanMB=sum(TotalExtentsScannedMB)
  by bin(TimeGenerated, 1h)
| render timechart

// Analyze failed ingestions
ADXIngestionBatching
| where TimeGenerated > ago(1d)
| where Status != "Success"
| summarize FailedCount=count() by Table, FailureKind
| order by FailedCount desc
```

### Cross-Query from ADX to Log Analytics

```kusto
// Query Log Analytics workspace from ADX
let workspaceId = "your-workspace-guid";
cluster('https://ade.loganalytics.io/subscriptions/{subscription-id}/resourcegroups/{resource-group}/providers/microsoft.operationalinsights/workspaces/{workspace-name}').database(workspaceId)
.Heartbeat
| where TimeGenerated > ago(1h)
| summarize LastHeartbeat=max(TimeGenerated) by Computer

// Join ADX data with Log Analytics data
let vmPerformance =
    cluster('https://ade.loganalytics.io/...').database('workspace')
    .Perf
    | where TimeGenerated > ago(1h)
    | where CounterName == "% Processor Time"
    | summarize AvgCPU=avg(CounterValue) by Computer;
MyADXTable
| join kind=leftouter vmPerformance on $left.ServerName == $right.Computer
| project ServerName, MyMetric, AvgCPU
```

### Cross-Query from ADX to Application Insights

```kusto
// Query Application Insights from ADX
cluster('https://ade.applicationinsights.io/subscriptions/{sub}/resourcegroups/{rg}/providers/microsoft.insights/components/{app-name}').database('{app-name}')
.requests
| where timestamp > ago(1h)
| summarize RequestCount=count(), AvgDuration=avg(duration) by name
| order by RequestCount desc

// Correlate ADX telemetry with App Insights traces
let appInsightsErrors =
    cluster('https://ade.applicationinsights.io/...').database('myapp')
    .exceptions
    | where timestamp > ago(1h)
    | project operation_Id, problemId, timestamp;
MyServiceLogs
| where Timestamp > ago(1h)
| join kind=inner appInsightsErrors on $left.CorrelationId == $right.operation_Id
| project Timestamp, ServiceMessage, problemId
```

---

## Synapse/Spark Integration Examples

### Read from ADX in Spark (PySpark)

```python
# Basic read from ADX
df = spark.read \
    .format("com.microsoft.kusto.spark.datasource") \
    .option("kustoCluster", "https://mycluster.region.kusto.windows.net") \
    .option("kustoDatabase", "mydb") \
    .option("kustoQuery", "MyTable | where Timestamp > ago(1d)") \
    .option("kustoAadAppId", "<app-id>") \
    .option("kustoAadAppSecret", "<app-secret>") \
    .option("kustoAadAuthorityID", "<tenant-id>") \
    .load()

# Display results
df.show()

# With managed identity
df = spark.read \
    .format("com.microsoft.kusto.spark.datasource") \
    .option("kustoCluster", "https://mycluster.region.kusto.windows.net") \
    .option("kustoDatabase", "mydb") \
    .option("kustoQuery", "MyTable | take 1000") \
    .option("kustoAadUseDeviceAuth", "true") \
    .load()
```

### Write to ADX from Spark (PySpark)

```python
# Write DataFrame to ADX
df.write \
    .format("com.microsoft.kusto.spark.datasource") \
    .option("kustoCluster", "https://mycluster.region.kusto.windows.net") \
    .option("kustoDatabase", "mydb") \
    .option("kustoTable", "TargetTable") \
    .option("kustoAadAppId", "<app-id>") \
    .option("kustoAadAppSecret", "<app-secret>") \
    .option("kustoAadAuthorityID", "<tenant-id>") \
    .mode("Append") \
    .save()

# Streaming write to ADX
streamingDf.writeStream \
    .format("com.microsoft.kusto.spark.datasource") \
    .option("kustoCluster", "https://mycluster.region.kusto.windows.net") \
    .option("kustoDatabase", "mydb") \
    .option("kustoTable", "StreamingTable") \
    .option("checkpointLocation", "/checkpoint/path") \
    .start()
```

### Spark Scala Examples

```scala
// Read from ADX
val df = spark.read
    .format("com.microsoft.kusto.spark.datasource")
    .option("kustoCluster", "https://mycluster.region.kusto.windows.net")
    .option("kustoDatabase", "mydb")
    .option("kustoQuery", "MyTable | where Value > 100 | take 1000")
    .load()

// Write to ADX with ingestion properties
df.write
    .format("com.microsoft.kusto.spark.datasource")
    .option("kustoCluster", "https://mycluster.region.kusto.windows.net")
    .option("kustoDatabase", "mydb")
    .option("kustoTable", "TargetTable")
    .option("tableCreateOptions", "CreateIfNotExist")
    .mode(SaveMode.Append)
    .save()
```

---

## External Table Examples

### Create External Table on Azure Data Lake

```kusto
// Create external table for Parquet files in ADLS Gen2
.create external table SalesArchive (
    TransactionDate: datetime,
    ProductId: string,
    CustomerId: string,
    Quantity: int,
    Revenue: decimal
)
kind=blob
partition by (Year: int = getfield(TransactionDate, "Year"), Month: int = getfield(TransactionDate, "Month"))
dataformat=parquet
(
    h@'https://mydatalake.dfs.core.windows.net/archive/sales;StorageAccountKey'
)

// Create external table for CSV files with partitioning
.create external table LogsArchive (
    Timestamp: datetime,
    Level: string,
    Message: string,
    Source: string
)
kind=blob
partition by (Date: datetime = bin(Timestamp, 1d))
dataformat=csv
(
    h@'https://mystorage.blob.core.windows.net/logs;SharedAccessSignature'
)
with (csvheader = true)
```

### Query External Tables

```kusto
// Query with partition pruning
external_table("SalesArchive")
| where Year == 2024 and Month >= 10  // Uses partition columns
| where TransactionDate > datetime(2024-10-15)
| summarize TotalRevenue=sum(Revenue) by ProductId
| top 10 by TotalRevenue

// Join external and internal tables
let archiveData = external_table("SalesArchive") | where Year == 2024;
let recentData = SalesTable | where TransactionDate >= startofyear(now());
union archiveData, recentData
| summarize TotalRevenue=sum(Revenue) by ProductId
| order by TotalRevenue desc

// Query JSON files with mapping
.create external table ApiLogs (
    Timestamp: datetime,
    RequestId: guid,
    Endpoint: string,
    StatusCode: int,
    Duration: real
)
kind=blob
dataformat=multijson
(
    h@'https://storage/apilogs;key'
)

.create external table ApiLogs json mapping 'LogMapping'
'['
'   {"Column": "Timestamp", "Properties": {"Path": "$.timestamp"}},'
'   {"Column": "RequestId", "Properties": {"Path": "$.request.id"}},'
'   {"Column": "Endpoint", "Properties": {"Path": "$.request.path"}},'
'   {"Column": "StatusCode", "Properties": {"Path": "$.response.status"}},'
'   {"Column": "Duration", "Properties": {"Path": "$.metrics.duration_ms"}}'
']'
```

---

## SQL Database Integration Examples

### Basic SQL Query

```kusto
// Query Azure SQL Database with explicit schema
evaluate sql_request(
    'Server=tcp:myserver.database.windows.net,1433;'
    'Authentication="Active Directory Integrated";'
    'Initial Catalog=ProductsDB;',
    'SELECT ProductId, ProductName, Category, Price FROM Products WHERE IsActive = 1'
) : (ProductId: int, ProductName: string, Category: string, Price: decimal)

// With parameters
evaluate sql_request(
    'Server=tcp:myserver.database.windows.net,1433;'
    'Authentication="Active Directory Integrated";'
    'Initial Catalog=OrdersDB;',
    'SELECT OrderId, CustomerId, TotalAmount FROM Orders WHERE OrderDate >= @startDate',
    dynamic({'@startDate': ago(30d)})
) : (OrderId: long, CustomerId: string, TotalAmount: decimal)
```

### Join ADX Data with SQL Data

```kusto
// Enrich ADX events with SQL dimension data
let customers =
    evaluate sql_request(
        'Server=tcp:myserver.database.windows.net,1433;'
        'Authentication="Active Directory Integrated";'
        'Initial Catalog=CRM;',
        'SELECT CustomerId, CustomerName, Region, Tier FROM Customers'
    ) : (CustomerId: string, CustomerName: string, Region: string, Tier: string);
TransactionEvents
| where Timestamp > ago(1d)
| lookup kind=leftouter customers on CustomerId
| summarize
    TransactionCount=count(),
    TotalValue=sum(Amount)
  by Region, Tier
| order by TotalValue desc
```

### SQL with Managed Identity

```kusto
// Using managed identity authentication
evaluate sql_request(
    'Server=tcp:myserver.database.windows.net,1433;'
    'Authentication="Active Directory Managed Identity";'
    'User Id=<managed-identity-client-id>;'
    'Initial Catalog=DataWarehouse;',
    'SELECT * FROM DimProduct WHERE ModifiedDate > @lastSync',
    dynamic({'@lastSync': ago(1h)})
) : (ProductKey: int, ProductName: string, Category: string, ModifiedDate: datetime)
```

---

## Cosmos DB Integration Examples

### Basic Cosmos DB Query

```kusto
// Query Cosmos DB with account key
evaluate cosmosdb_sql_request(
    'AccountEndpoint=https://mycosmosdb.documents.azure.com/;'
    'Database=ProductCatalog;'
    'Collection=Products;'
    h'AccountKey=<your-account-key>',
    'SELECT c.id, c.name, c.category, c.price FROM c WHERE c.isActive = true'
) : (id: string, name: string, category: string, price: real)

// With managed identity (recommended)
evaluate cosmosdb_sql_request(
    'AccountEndpoint=https://mycosmosdb.documents.azure.com/;'
    'Database=UserProfiles;'
    'Collection=Profiles;'
    'Authentication="Active Directory Managed Identity";'
    'User Id=<managed-identity-object-id>;',
    'SELECT c.userId, c.preferences FROM c'
) : (userId: string, preferences: dynamic)
```

### Join ADX with Cosmos DB

```kusto
// Enrich telemetry with user preferences from Cosmos DB
let userPrefs =
    evaluate cosmosdb_sql_request(
        'AccountEndpoint=https://mydb.documents.azure.com/;'
        'Database=Users;Collection=Preferences;',
        'SELECT c.userId, c.theme, c.language FROM c',
        dynamic(null),
        dynamic({'armResourceId': '/subscriptions/.../databaseAccounts/mydb'})
    ) : (userId: string, theme: string, language: string);
UserActivityEvents
| where Timestamp > ago(1h)
| join kind=leftouter userPrefs on $left.UserId == $right.userId
| summarize EventCount=count() by theme, language
| render piechart
```

### Cosmos DB with Region Preference

```kusto
// Query specific region for lower latency
evaluate cosmosdb_sql_request(
    'AccountEndpoint=https://mycosmosdb.documents.azure.com/;'
    'Database=Sessions;Collection=ActiveSessions;',
    'SELECT c.sessionId, c.userId, c.startTime FROM c WHERE c.status = "active"',
    dynamic(null),
    dynamic({'preferredLocations': ['West US 2', 'East US']})
) : (sessionId: string, userId: string, startTime: datetime)
```

---

## HTTP API Integration Examples

### Query REST API

```kusto
// Query Azure pricing API
let pricingUri = 'https://prices.azure.com/api/retail/prices?$filter=serviceName eq \'Azure Kubernetes Service\'';
evaluate http_request(pricingUri)
| project prices = ResponseBody.Items
| mv-expand prices
| evaluate bag_unpack(prices)
| project armRegionName, meterName, retailPrice, unitOfMeasure

// Query custom REST endpoint with headers
evaluate http_request(
    'https://api.myservice.com/v1/metrics',
    dynamic({'Authorization': 'Bearer <token>', 'Accept': 'application/json'})
)
| project ResponseBody
| mv-expand metric = ResponseBody.metrics
| project
    MetricName = tostring(metric.name),
    Value = todouble(metric.value),
    Timestamp = todatetime(metric.timestamp)
```

### Integrate with External Service

```kusto
// Get exchange rates from external API
let rates =
    evaluate http_request('https://api.exchangerate-api.com/v4/latest/USD')
    | project ResponseBody.rates;
TransactionTable
| where Currency != 'USD'
| extend rates_data = toscalar(rates)
| extend ExchangeRate = todouble(rates_data[Currency])
| extend AmountUSD = Amount * ExchangeRate
| summarize TotalUSD=sum(AmountUSD) by Region
```

---

## Power Automate Flow Examples

### Alert Flow: Query and Email

**Flow Configuration:**

1. **Trigger**: Recurrence (Every 5 minutes)
2. **Action**: Run KQL query
   - Cluster: `https://mycluster.region.kusto.windows.net`
   - Database: `monitoring`
   - Query:
   ```kusto
   Alerts
   | where Timestamp > ago(5m)
   | where Severity == "Critical"
   | summarize AlertCount=count() by AlertType, AffectedResource
   | where AlertCount > 0
   ```
3. **Condition**: If AlertCount > 0
4. **Action (True)**: Send email
   - Subject: `Critical Alert: @{body('Run_KQL_query')?['AlertType']}`
   - Body: `@{body('Run_KQL_query')}`

### Report Flow: Daily Summary with Chart

**Flow Configuration:**

1. **Trigger**: Recurrence (Daily at 8:00 AM)
2. **Action**: Run KQL query and render a chart
   - Chart Type: Timechart
   - Query:
   ```kusto
   RequestLogs
   | where Timestamp > ago(24h)
   | summarize RequestCount=count(), AvgLatency=avg(Duration) by bin(Timestamp, 1h)
   ```
3. **Action**: Send email (V2)
   - Body: `@{body('Run_KQL_query_and_render_a_chart')?['BodyHtml']}`
   - Attachments: `@{body('Run_KQL_query_and_render_a_chart')?['AttachmentContent']}`

### Data Sync Flow: ADX to SQL

**Flow Configuration:**

1. **Trigger**: Recurrence (Every hour)
2. **Action**: Run KQL query
   - Query:
   ```kusto
   ProcessedEvents
   | where Timestamp > ago(1h)
   | summarize Count=count(), AvgValue=avg(Value) by Category
   ```
3. **Action**: Apply to each (on query results)
4. **Action (inside loop)**: SQL - Insert row
   - Table: EventSummary
   - Category: `@{items('Apply_to_each')?['Category']}`
   - Count: `@{items('Apply_to_each')?['Count']}`
   - AvgValue: `@{items('Apply_to_each')?['AvgValue']}`

### Management Command Flow: Async Operation

**Flow Configuration:**

1. **Trigger**: HTTP request (manual trigger)
2. **Action**: Run async management command
   - Query:
   ```kusto
   .export async to csv (
       h@'https://storage.blob.core.windows.net/exports;key'
   ) <| ArchiveData | where Timestamp > ago(30d)
   ```
3. **Action**: Initialize variable `OperationId` = `@{body('Run_async_command')?['OperationId']}`
4. **Action**: Do Until (Status == "Completed")
   - **Action**: Run show management command
     - Query: `.show operations @{variables('OperationId')}`
   - **Action**: Delay 30 seconds
5. **Action**: Send completion notification

---

## Logic Apps Integration Patterns

### Pattern: Multi-Stage Data Pipeline

```json
{
  "definition": {
    "triggers": {
      "Recurrence": {
        "type": "Recurrence",
        "recurrence": {
          "frequency": "Hour",
          "interval": 1
        }
      }
    },
    "actions": {
      "Query_Source_Data": {
        "type": "ApiConnection",
        "inputs": {
          "host": {"connection": {"name": "@parameters('$connections')['kusto']['connectionId']"}},
          "method": "post",
          "path": "/ListKustoResults/",
          "body": {
            "clusterUrl": "https://source.kusto.windows.net",
            "databaseName": "sourcedb",
            "query": "SourceTable | where ModifiedTime > ago(1h)"
          }
        }
      },
      "Transform_Data": {
        "type": "Compose",
        "inputs": "@body('Query_Source_Data')",
        "runAfter": {"Query_Source_Data": ["Succeeded"]}
      },
      "Insert_To_Target": {
        "type": "ApiConnection",
        "inputs": {
          "host": {"connection": {"name": "@parameters('$connections')['kusto']['connectionId']"}},
          "method": "post",
          "path": "/ListKustoResults/",
          "body": {
            "clusterUrl": "https://target.kusto.windows.net",
            "databaseName": "targetdb",
            "query": ".ingest inline into table TargetTable <| @{outputs('Transform_Data')}"
          }
        },
        "runAfter": {"Transform_Data": ["Succeeded"]}
      }
    }
  }
}
```

---

## Azure Functions Examples

### Event Hub Trigger - Ingest to ADX (C#)

```csharp
using Azure.Core;
using Kusto.Data;
using Kusto.Ingest;
using Microsoft.Azure.WebJobs;
using Microsoft.Extensions.Logging;
using System.IO;
using System.Text;

public static class EventHubIngestFunction
{
    private static readonly IKustoIngestClient IngestClient;

    static EventHubIngestFunction()
    {
        var kustoUri = "https://mycluster.region.kusto.windows.net";
        var database = "mydb";

        var builder = new KustoConnectionStringBuilder(kustoUri, database)
            .WithAadManagedIdentity("system");

        IngestClient = KustoIngestFactory.CreateQueuedIngestClient(builder);
    }

    [FunctionName("IngestFromEventHub")]
    public static async Task Run(
        [EventHubTrigger("events", Connection = "EventHubConnection")] EventData[] events,
        ILogger log)
    {
        var properties = new KustoIngestionProperties("mydb", "EventsTable")
        {
            Format = DataSourceFormat.json,
            IngestionMapping = new IngestionMapping
            {
                IngestionMappingReference = "JsonMapping"
            }
        };

        foreach (var eventData in events)
        {
            var body = Encoding.UTF8.GetString(eventData.Body.Array);
            using var stream = new MemoryStream(Encoding.UTF8.GetBytes(body));
            await IngestClient.IngestFromStreamAsync(stream, properties);
        }

        log.LogInformation($"Processed {events.Length} events");
    }
}
```

### Timer Trigger - Scheduled Query (Python)

```python
import azure.functions as func
from azure.kusto.data import KustoClient, KustoConnectionStringBuilder
from azure.kusto.data.helpers import dataframe_from_result_table
import pandas as pd

def main(mytimer: func.TimerRequest) -> None:
    cluster = "https://mycluster.region.kusto.windows.net"
    database = "mydb"

    # Use managed identity
    kcsb = KustoConnectionStringBuilder.with_aad_managed_service_identity_authentication(
        cluster,
        client_id="<managed-identity-client-id>"  # Optional for system-assigned
    )

    client = KustoClient(kcsb)

    query = """
    Metrics
    | where Timestamp > ago(1h)
    | summarize AvgValue=avg(Value) by MetricName
    | where AvgValue > 100
    """

    response = client.execute(database, query)
    df = dataframe_from_result_table(response.primary_results[0])

    # Process results - send alerts, store summary, etc.
    if not df.empty:
        for _, row in df.iterrows():
            # Send alert for each high metric
            logging.info(f"High metric: {row['MetricName']} = {row['AvgValue']}")
```

---

## Complete Integration Scenario

### End-to-End: IoT Data Pipeline

This example shows a complete integration pattern from IoT Hub to ADX with external enrichment:

```
IoT Hub --> Event Hub --> Azure Function --> ADX (Streaming)
                                |
                                v
                         SQL (Device Registry)
                                |
                                v
                         Power Automate (Alerts)
```

**Step 1: Azure Function (Event Processing)**
```csharp
// Process IoT events, enrich with SQL, ingest to ADX
[FunctionName("ProcessIoTEvents")]
public async Task Run([EventHubTrigger("iothub", Connection = "IoTHub")] EventData[] events)
{
    // Batch process events
    var enrichedEvents = await EnrichWithDeviceInfo(events);
    await IngestToADX(enrichedEvents);
}
```

**Step 2: ADX Query (Analysis)**
```kusto
// Analyze enriched IoT data
IoTEvents
| where Timestamp > ago(1h)
| join kind=leftouter (
    evaluate sql_request(
        'Server=tcp:devices.database.windows.net;...',
        'SELECT DeviceId, Location, Owner FROM Devices'
    ) : (DeviceId: string, Location: string, Owner: string)
) on DeviceId
| summarize AvgTemperature=avg(Temperature) by Location, Owner
| where AvgTemperature > 80
```

**Step 3: Power Automate (Alerting)**
- Trigger: Every 5 minutes
- Query: Above query for high temperature
- Action: Send Teams notification to Owner

---

*Document generated from Azure Data Explorer documentation analysis*
*Phase: 3 - Feature In-Depth Research*
