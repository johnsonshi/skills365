# Azure Data Explorer Data Ingestion Examples

## Overview

This document provides practical examples for common data ingestion scenarios in Azure Data Explorer, including command examples, connector setup, and SDK code samples.

---

## Direct Ingestion Commands

### Inline Ingestion (Testing Only)

```kusto
// Simple CSV inline ingestion
.ingest inline into table SensorData <|
    2024-01-15T10:30:00Z,sensor-001,temperature,23.5
    2024-01-15T10:30:00Z,sensor-001,humidity,65.2
    2024-01-15T10:31:00Z,sensor-002,temperature,24.1

// With format specification
.ingest inline into table Events with (format='csv') <|
    event1,2024-01-15,user123
    event2,2024-01-15,user456
```

### Ingest from Storage

```kusto
// Basic CSV ingestion from blob storage
.ingest into table MyTable (
    'https://mystorageaccount.blob.core.windows.net/container/data.csv'
    h'?sv=2021-06-08&st=2024-01-15&se=2024-01-16&sr=b&sp=r&sig=...'
)
with (format='csv', ignoreFirstRecord=true)

// JSON ingestion with mapping
.ingest into table Events (
    'https://mystorageaccount.blob.core.windows.net/container/events.json'
    h'?sv=...'
)
with (
    format='json',
    ingestionMappingReference='EventsMapping'
)

// Parquet ingestion
.ingest into table Analytics (
    'https://mystorageaccount.blob.core.windows.net/container/data.parquet'
    h'?sv=...'
)
with (format='parquet')

// Compressed file ingestion
.ingest into table Logs (
    'https://mystorageaccount.blob.core.windows.net/container/logs.csv.gz'
    h'?sv=...'
)
with (format='csv')

// Using managed identity
.ingest into table MyTable (
    'https://mystorageaccount.blob.core.windows.net/container/data.csv;managed_identity=system'
)
with (format='csv')
```

### Ingest from Query

```kusto
// Create table from query results
.set NewTable <|
    SourceTable
    | where Timestamp > ago(7d)
    | summarize Count=count() by bin(Timestamp, 1h), Category

// Append to existing table
.append DailyAggregates <|
    RawEvents
    | where Timestamp between(datetime(2024-01-15) .. datetime(2024-01-16))
    | summarize EventCount=count() by UserId

// Create or append with tags
.set-or-append HistoricalData
    with (tags='["ingest-by:batch-20240115"]', ingestIfNotExists='["batch-20240115"]') <|
    SourceTable
    | where Timestamp > ago(30d)

// Replace table contents
.set-or-replace DailySummary <|
    RawData
    | where Timestamp >= startofday(now())
    | summarize Total=sum(Value) by Category

// Distributed ingestion for large datasets
.set async LargeTable with (distributed=true) <|
    VeryLargeSourceTable
    | where Timestamp > ago(90d)
```

---

## Ingestion Mapping Examples

### CSV Mapping

```kusto
// Create CSV mapping with ordinal positions
.create table SensorData ingestion csv mapping 'SensorDataMapping'
```
'['
'  {"column":"Timestamp","Properties":{"Ordinal":"0"},"DataType":"datetime"},'
'  {"column":"DeviceId","Properties":{"Ordinal":"1"},"DataType":"string"},'
'  {"column":"MetricName","Properties":{"Ordinal":"2"},"DataType":"string"},'
'  {"column":"MetricValue","Properties":{"Ordinal":"3"},"DataType":"real"}'
']'
```

### JSON Mapping

```kusto
// Create JSON mapping with JSONPath
.create table Events ingestion json mapping 'EventsMapping'
```
'['
'  {"column":"EventTime","Properties":{"path":"$.timestamp"}},'
'  {"column":"EventType","Properties":{"path":"$.event.type"}},'
'  {"column":"UserId","Properties":{"path":"$.user.id"}},'
'  {"column":"Properties","Properties":{"path":"$.event.properties"}}'
']'
```

### JSON Mapping with Transformations

```kusto
// Unix timestamp conversion
.create table MetricsFromUnix ingestion json mapping 'UnixTimestampMapping'
```
'['
'  {"column":"Timestamp","Properties":{"path":"$.ts","transform":"DateTimeFromUnixSeconds"}},'
'  {"column":"Value","Properties":{"path":"$.value"}}'
']'
```

### Nested JSON Mapping

```kusto
// Map nested objects
.create table IoTData ingestion json mapping 'IoTMapping'
```
'['
'  {"column":"DeviceId","Properties":{"path":"$.device.id"}},'
'  {"column":"Location","Properties":{"path":"$.device.location.name"}},'
'  {"column":"Latitude","Properties":{"path":"$.device.location.lat"}},'
'  {"column":"Longitude","Properties":{"path":"$.device.location.lon"}},'
'  {"column":"Readings","Properties":{"path":"$.readings"}}'
']'
```

### Parquet Mapping

```kusto
// Parquet field mapping
.create table Analytics ingestion parquet mapping 'AnalyticsMapping'
```
'['
'  {"column":"EventDate","Properties":{"path":"$.event_date"}},'
'  {"column":"UserId","Properties":{"path":"$.user_id"}},'
'  {"column":"SessionId","Properties":{"path":"$.session_id"}},'
'  {"column":"PageViews","Properties":{"path":"$.metrics.page_views"}}'
']'
```

### Avro Mapping

```kusto
// Avro field mapping
.create table EventHubData ingestion avro mapping 'AvroMapping'
```
'['
'  {"column":"EventId","Properties":{"path":"$.EventId"}},'
'  {"column":"EventBody","Properties":{"path":"$.Body"}},'
'  {"column":"EnqueuedTime","Properties":{"path":"$.EnqueuedTimeUtc"}}'
']'
```

---

## SDK Ingestion Examples

### C# - Queued Ingestion

```csharp
using Kusto.Data;
using Kusto.Ingest;

// Create connection string
var connectionString = new KustoConnectionStringBuilder(
    "https://ingest-mycluster.region.kusto.windows.net")
    .WithAadApplicationKeyAuthentication(appId, appKey, tenantId);

// Create queued ingest client (reuse this instance!)
using var ingestClient = KustoIngestFactory.CreateQueuedIngestClient(connectionString);

// Define ingestion properties
var ingestionProperties = new KustoIngestionProperties(
    databaseName: "MyDatabase",
    tableName: "MyTable")
{
    Format = DataSourceFormat.csv,
    IngestionMapping = new IngestionMapping
    {
        IngestionMappingReference = "MyCsvMapping"
    }
};

// Ingest from file
await ingestClient.IngestFromStorageAsync(
    "https://storage.blob.core.windows.net/container/file.csv?sas=...",
    ingestionProperties);

// Ingest from stream
using var stream = File.OpenRead("data.csv");
await ingestClient.IngestFromStreamAsync(stream, ingestionProperties);
```

### C# - Streaming Ingestion

```csharp
using Kusto.Data;
using Kusto.Ingest;

// Create connection string (use engine endpoint, not ingest)
var connectionString = new KustoConnectionStringBuilder(
    "https://mycluster.region.kusto.windows.net")
    .WithAadApplicationKeyAuthentication(appId, appKey, tenantId);

// Create streaming ingest client
using var ingestClient = KustoIngestFactory.CreateStreamingIngestClient(connectionString);

var ingestionProperties = new KustoIngestionProperties("MyDatabase", "MyTable")
{
    Format = DataSourceFormat.json,
    IngestionMapping = new IngestionMapping
    {
        IngestionMappingReference = "MyJsonMapping"
    }
};

// Ingest from stream
using var stream = new MemoryStream(Encoding.UTF8.GetBytes(jsonData));
var sourceOptions = new StreamSourceOptions { CompressionType = DataSourceCompressionType.None };
await ingestClient.IngestFromStreamAsync(stream, ingestionProperties, sourceOptions);
```

### Python - Queued Ingestion

```python
from azure.kusto.data import KustoConnectionStringBuilder
from azure.kusto.ingest import (
    QueuedIngestClient,
    IngestionProperties,
    DataFormat
)

# Create connection string
cluster = "https://ingest-mycluster.region.kusto.windows.net"
kcsb = KustoConnectionStringBuilder.with_aad_application_key_authentication(
    cluster, app_id, app_key, tenant_id
)

# Create client
client = QueuedIngestClient(kcsb)

# Define ingestion properties
ingestion_props = IngestionProperties(
    database="MyDatabase",
    table="MyTable",
    data_format=DataFormat.CSV,
    ingestion_mapping_reference="MyCsvMapping"
)

# Ingest from file
client.ingest_from_file("data.csv", ingestion_properties=ingestion_props)

# Ingest from blob
client.ingest_from_blob(
    "https://storage.blob.core.windows.net/container/file.csv?sas=...",
    ingestion_properties=ingestion_props
)

# Ingest from DataFrame
import pandas as pd
df = pd.DataFrame({"col1": [1, 2, 3], "col2": ["a", "b", "c"]})
client.ingest_from_dataframe(df, ingestion_properties=ingestion_props)
```

### Python - Streaming Ingestion

```python
from azure.kusto.data import KustoConnectionStringBuilder, DataFormat
from azure.kusto.ingest import KustoStreamingIngestClient, IngestionProperties

# Use engine endpoint for streaming
cluster = "https://mycluster.region.kusto.windows.net"
kcsb = KustoConnectionStringBuilder.with_aad_application_key_authentication(
    cluster, app_id, app_key, tenant_id
)

client = KustoStreamingIngestClient(kcsb)

ingestion_props = IngestionProperties(
    database="MyDatabase",
    table="MyTable",
    data_format=DataFormat.JSON,
    ingestion_mapping_reference="MyJsonMapping"
)

# Ingest from file (auto-detects compression)
client.ingest_from_file("data.json.gz", ingestion_properties=ingestion_props)
```

### Node.js - Queued Ingestion

```javascript
import { KustoConnectionStringBuilder } from "azure-kusto-data";
import { IngestClient, IngestionProperties, DataFormat } from "azure-kusto-ingest";

// Create connection string
const cluster = "https://ingest-mycluster.region.kusto.windows.net";
const kcsb = KustoConnectionStringBuilder.withAadApplicationKeyAuthentication(
    cluster, appId, appKey, tenantId
);

// Create client
const client = new IngestClient(kcsb);

// Define properties
const ingestionProps = new IngestionProperties({
    database: "MyDatabase",
    table: "MyTable",
    format: DataFormat.JSON,
    ingestionMappingReference: "MyJsonMapping"
});

// Ingest from file
await client.ingestFromFile("data.json", ingestionProps);

// Ingest from blob
await client.ingestFromBlob(
    new BlobDescriptor("https://storage.blob.core.windows.net/container/file.json?sas=..."),
    ingestionProps
);
```

### Java - Queued Ingestion

```java
import com.microsoft.azure.kusto.data.auth.ConnectionStringBuilder;
import com.microsoft.azure.kusto.ingest.*;
import com.microsoft.azure.kusto.ingest.source.FileSourceInfo;

// Create connection string
String cluster = "https://ingest-mycluster.region.kusto.windows.net";
ConnectionStringBuilder csb = ConnectionStringBuilder.createWithAadApplicationCredentials(
    cluster, appId, appKey, tenantId
);

// Create client
IngestClient client = IngestClientFactory.createClient(csb);

// Define properties
IngestionProperties props = new IngestionProperties("MyDatabase", "MyTable");
props.setDataFormat(DataFormat.CSV);
props.setIngestionMapping("MyCsvMapping", IngestionMappingKind.CSV);

// Ingest from file
FileSourceInfo fileInfo = new FileSourceInfo("data.csv", 0);
client.ingestFromFile(fileInfo, props);
```

### Go - Streaming Ingestion

```go
package main

import (
    "context"
    "github.com/Azure/azure-kusto-go/kusto"
    "github.com/Azure/azure-kusto-go/kusto/ingest"
    "github.com/Azure/go-autorest/autorest/azure/auth"
)

func main() {
    cluster := "https://mycluster.region.kusto.windows.net"

    // Create authorization
    authorizer := kusto.Authorization{
        Config: auth.NewClientCredentialsConfig(appId, appKey, tenantId),
    }

    // Create client
    client, _ := kusto.New(cluster, authorizer)
    defer client.Close()

    // Create ingestion instance
    ingestor, _ := ingest.New(client, "MyDatabase", "MyTable")

    // Stream ingestion (max 4 MB for Go)
    data := []byte(`{"id": 1, "name": "test"}`)
    err := ingestor.Stream(context.Background(), data, ingest.JSON, "MyJsonMapping")
}
```

---

## Connector Setup Examples

### Event Hubs Connection (C#)

```csharp
using Azure.Messaging.EventHubs;
using Azure.Messaging.EventHubs.Producer;

// Create producer client
await using var producer = new EventHubProducerClient(connectionString, eventHubName);

// Create event with routing properties
var eventData = new EventData(Encoding.UTF8.GetBytes(jsonPayload));

// Set dynamic routing properties
eventData.Properties["Table"] = "MyTable";
eventData.Properties["Format"] = "json";
eventData.Properties["IngestionMappingReference"] = "MyMapping";

// For multi-database routing
eventData.Properties["Database"] = "AlternateDB";

// Send event
await producer.SendAsync(new[] { eventData });
```

### Event Grid Blob Upload (C#)

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;

var containerClient = new BlobContainerClient(connectionString, containerName);
var blobClient = containerClient.GetBlobClient("data/events-2024-01-15.json");

// Set ingestion metadata BEFORE upload
var metadata = new Dictionary<string, string>
{
    { "kustoTable", "Events" },
    { "kustoDataFormat", "json" },
    { "kustoIngestionMappingReference", "EventsMapping" },
    { "rawSizeBytes", "4096" }
};

// Upload with metadata
await blobClient.UploadAsync(
    BinaryData.FromString(jsonContent),
    new BlobUploadOptions { Metadata = metadata }
);
```

### Kafka Sink Configuration

```json
{
    "name": "kusto-sink-connector",
    "config": {
        "connector.class": "com.microsoft.azure.kusto.kafka.connect.sink.KustoSinkConnector",
        "tasks.max": "3",
        "topics": "events-topic,metrics-topic",

        "kusto.tables.topics.mapping": "[
            {'topic':'events-topic','db':'MyDatabase','table':'Events','format':'json','mapping':'EventsMapping'},
            {'topic':'metrics-topic','db':'MyDatabase','table':'Metrics','format':'csv','mapping':'MetricsMapping'}
        ]",

        "kusto.ingestion.url": "https://ingest-mycluster.region.kusto.windows.net",
        "kusto.query.url": "https://mycluster.region.kusto.windows.net",

        "aad.auth.authority": "your-tenant-id",
        "aad.auth.appid": "your-app-id",
        "aad.auth.appkey": "your-app-secret",

        "flush.size.bytes": "1000000",
        "flush.interval.ms": "30000",

        "key.converter": "org.apache.kafka.connect.storage.StringConverter",
        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
        "value.converter.schemas.enable": "false"
    }
}
```

---

## Policy Configuration Examples

### Streaming Ingestion Policy

```kusto
// Enable on specific table
.alter table MyTable policy streamingingestion enable

// Enable on entire database
.alter database MyDatabase policy streamingingestion enable

// Disable streaming ingestion
.delete table MyTable policy streamingingestion
```

### Ingestion Batching Policy

```kusto
// Low latency configuration (20-30 seconds)
.alter table RealTimeData policy ingestionbatching
```
'{'
'  "MaximumBatchingTimeSpan": "00:00:20",'
'  "MaximumNumberOfItems": 500,'
'  "MaximumRawDataSizeMB": 256'
'}'
```

// High throughput configuration
.alter database MyDatabase policy ingestionbatching
```
'{'
'  "MaximumBatchingTimeSpan": "00:05:00",'
'  "MaximumNumberOfItems": 1000,'
'  "MaximumRawDataSizeMB": 1024'
'}'
```

---

## Monitoring and Troubleshooting Examples

### Check Ingestion Failures

```kusto
// Recent failures
.show ingestion failures
| where FailedOn > ago(1h)
| project FailedOn, Database, Table, ErrorCode, Details

// Failures by error type
.show ingestion failures
| where FailedOn > ago(24h)
| summarize Count=count() by ErrorCode, FailureKind
| order by Count desc

// Specific table failures
.show ingestion failures
| where Table == "MyTable"
| where FailedOn > ago(1h)
| project FailedOn, IngestionSourcePath, ErrorCode, Details
```

### Check Ingestion Operations

```kusto
// Recent ingestion commands
.show commands
| where StartedOn > ago(1h)
| where CommandType has "Ingest"
| project StartedOn, CommandType, State, Duration, Text
| order by StartedOn desc
```

### Validate Mapping

```kusto
// Show all mappings for a table
.show table MyTable ingestion mappings

// Show specific mapping
.show table MyTable ingestion json mapping 'MyJsonMapping'
```

---

## Document Information

- **Generated**: 2026-02-13
- **Source**: Azure Data Explorer documentation analysis
- **Phase**: 3 - Feature Deep Dive (Data Ingestion)
