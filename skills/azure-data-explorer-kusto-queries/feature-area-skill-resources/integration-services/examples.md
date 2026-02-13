# Integration Services Examples

## Azure Monitor Cross-Queries

### Query Log Analytics from ADX

```kusto
// Query Log Analytics workspace
let workspaceId = "your-workspace-guid";
cluster('https://ade.loganalytics.io/subscriptions/{sub}/resourcegroups/{rg}/providers/microsoft.operationalinsights/workspaces/{workspace}').database(workspaceId)
.Heartbeat
| where TimeGenerated > ago(1h)
| summarize LastHeartbeat=max(TimeGenerated) by Computer

// Join ADX data with Log Analytics VM performance
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

### Query Application Insights from ADX

```kusto
// Query App Insights requests
cluster('https://ade.applicationinsights.io/subscriptions/{sub}/resourcegroups/{rg}/providers/microsoft.insights/components/{app}').database('{app}')
.requests
| where timestamp > ago(1h)
| summarize RequestCount=count(), AvgDuration=avg(duration) by name
| order by RequestCount desc

// Correlate ADX logs with App Insights exceptions
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

## Spark Integration

### Read from ADX (PySpark)

```python
# Basic read
df = spark.read \
    .format("com.microsoft.kusto.spark.datasource") \
    .option("kustoCluster", "https://mycluster.region.kusto.windows.net") \
    .option("kustoDatabase", "mydb") \
    .option("kustoQuery", "MyTable | where Timestamp > ago(1d)") \
    .option("kustoAadAppId", "<app-id>") \
    .option("kustoAadAppSecret", "<app-secret>") \
    .option("kustoAadAuthorityID", "<tenant-id>") \
    .load()

# With managed identity
df = spark.read \
    .format("com.microsoft.kusto.spark.datasource") \
    .option("kustoCluster", "https://mycluster.region.kusto.windows.net") \
    .option("kustoDatabase", "mydb") \
    .option("kustoQuery", "MyTable | take 1000") \
    .option("kustoAadUseDeviceAuth", "true") \
    .load()
```

### Write to ADX (PySpark)

```python
# Batch write
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

# Streaming write
streamingDf.writeStream \
    .format("com.microsoft.kusto.spark.datasource") \
    .option("kustoCluster", "https://mycluster.region.kusto.windows.net") \
    .option("kustoDatabase", "mydb") \
    .option("kustoTable", "StreamingTable") \
    .option("checkpointLocation", "/checkpoint/path") \
    .start()
```

### Spark Scala

```scala
// Read with ingestion properties
val df = spark.read
    .format("com.microsoft.kusto.spark.datasource")
    .option("kustoCluster", "https://mycluster.region.kusto.windows.net")
    .option("kustoDatabase", "mydb")
    .option("kustoQuery", "MyTable | where Value > 100 | take 1000")
    .load()

// Write with auto table creation
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

## External Tables

### Create External Table on ADLS

```kusto
// Parquet files with partitioning
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

// CSV files with date partitioning
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
// Partition pruning for performance
external_table("SalesArchive")
| where Year == 2024 and Month >= 10  // Uses partition columns
| where TransactionDate > datetime(2024-10-15)
| summarize TotalRevenue=sum(Revenue) by ProductId
| top 10 by TotalRevenue

// Union external and internal tables
let archiveData = external_table("SalesArchive") | where Year == 2024;
let recentData = SalesTable | where TransactionDate >= startofyear(now());
union archiveData, recentData
| summarize TotalRevenue=sum(Revenue) by ProductId
| order by TotalRevenue desc
```

### JSON Files with Mapping

```kusto
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

## SQL Database Integration

### Basic SQL Query

```kusto
// Query with explicit schema
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

### Join ADX with SQL

```kusto
// Enrich events with SQL dimension data
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
| summarize TransactionCount=count(), TotalValue=sum(Amount) by Region, Tier
| order by TotalValue desc
```

### SQL with Managed Identity

```kusto
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

## Cosmos DB Integration

### Basic Query

```kusto
// With account key
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
// Enrich telemetry with user preferences
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

### Regional Preference

```kusto
evaluate cosmosdb_sql_request(
    'AccountEndpoint=https://mycosmosdb.documents.azure.com/;'
    'Database=Sessions;Collection=ActiveSessions;',
    'SELECT c.sessionId, c.userId, c.startTime FROM c WHERE c.status = "active"',
    dynamic(null),
    dynamic({'preferredLocations': ['West US 2', 'East US']})
) : (sessionId: string, userId: string, startTime: datetime)
```

---

## HTTP API Integration

### Query REST API

```kusto
// Azure Pricing API
let pricingUri = 'https://prices.azure.com/api/retail/prices?$filter=serviceName eq \'Azure Kubernetes Service\'';
evaluate http_request(pricingUri)
| project prices = ResponseBody.Items
| mv-expand prices
| evaluate bag_unpack(prices)
| project armRegionName, meterName, retailPrice, unitOfMeasure

// Custom REST endpoint with headers
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

---

## Power Automate Flows

### Alert Flow: Critical Alerts to Email

**Configuration:**
1. Trigger: Recurrence (Every 5 minutes)
2. Action: Run KQL query
   ```kusto
   Alerts
   | where Timestamp > ago(5m)
   | where Severity == "Critical"
   | summarize AlertCount=count() by AlertType, AffectedResource
   | where AlertCount > 0
   ```
3. Condition: If AlertCount > 0
4. Action (True): Send email with alert details

### Report Flow: Daily Summary with Chart

**Configuration:**
1. Trigger: Recurrence (Daily at 8:00 AM)
2. Action: Run KQL query and render chart (Timechart)
   ```kusto
   RequestLogs
   | where Timestamp > ago(24h)
   | summarize RequestCount=count(), AvgLatency=avg(Duration) by bin(Timestamp, 1h)
   ```
3. Action: Send email with chart attachment

### Data Sync Flow: ADX to SQL

**Configuration:**
1. Trigger: Recurrence (Every hour)
2. Action: Run KQL query
   ```kusto
   ProcessedEvents
   | where Timestamp > ago(1h)
   | summarize Count=count(), AvgValue=avg(Value) by Category
   ```
3. Action: Apply to each result
4. Action (loop): SQL - Insert row

### Async Export Flow

**Configuration:**
1. Trigger: HTTP request
2. Action: Run async management command
   ```kusto
   .export async to csv (
       h@'https://storage.blob.core.windows.net/exports;key'
   ) <| ArchiveData | where Timestamp > ago(30d)
   ```
3. Action: Store OperationId
4. Action: Do Until (Status == "Completed")
   - Run: `.show operations <OperationId>`
   - Delay: 30 seconds
5. Action: Send completion notification

---

## Azure Functions

### Event Hub Trigger - Ingest to ADX (C#)

```csharp
using Kusto.Data;
using Kusto.Ingest;

public static class EventHubIngestFunction
{
    private static readonly IKustoIngestClient IngestClient;

    static EventHubIngestFunction()
    {
        var kustoUri = "https://mycluster.region.kusto.windows.net";
        var builder = new KustoConnectionStringBuilder(kustoUri, "mydb")
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
            IngestionMapping = new IngestionMapping { IngestionMappingReference = "JsonMapping" }
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

def main(mytimer: func.TimerRequest) -> None:
    cluster = "https://mycluster.region.kusto.windows.net"
    kcsb = KustoConnectionStringBuilder.with_aad_managed_service_identity_authentication(
        cluster, client_id="<managed-identity-client-id>"
    )
    client = KustoClient(kcsb)

    query = """
    Metrics
    | where Timestamp > ago(1h)
    | summarize AvgValue=avg(Value) by MetricName
    | where AvgValue > 100
    """
    response = client.execute("mydb", query)
    df = dataframe_from_result_table(response.primary_results[0])

    if not df.empty:
        for _, row in df.iterrows():
            logging.info(f"High metric: {row['MetricName']} = {row['AvgValue']}")
```

---

## End-to-End: IoT Data Pipeline

### Architecture

```
IoT Hub --> Event Hub --> Azure Function --> ADX (Streaming)
                              |
                              v
                         SQL (Device Registry)
                              |
                              v
                         Power Automate (Alerts)
```

### Step 1: Azure Function (Event Processing)

```csharp
[FunctionName("ProcessIoTEvents")]
public async Task Run([EventHubTrigger("iothub", Connection = "IoTHub")] EventData[] events)
{
    var enrichedEvents = await EnrichWithDeviceInfo(events);
    await IngestToADX(enrichedEvents);
}
```

### Step 2: ADX Query (Analysis with SQL Enrichment)

```kusto
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

### Step 3: Power Automate (Alerting)

- Trigger: Every 5 minutes
- Query: Above query for high temperature
- Action: Send Teams notification to Owner
