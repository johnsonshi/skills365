# Azure Data Explorer Data Ingestion Reference

## Overview

Data ingestion is the process of loading data into Azure Data Explorer tables, making it available for query. ADX validates data, converts formats, performs schema matching, indexing, encoding, and compression during ingestion.

---

## Ingestion Methods

### 1. Streaming Ingestion

Real-time ingestion with sub-second latency for small data volumes.

**When to use:**
- Latency requirements under 1 second
- Many tables with small data streams (few records/second each)
- Overall high volume distributed across tables

**Limitations:**
- Data size limit: 4 MB per request
- Requires cluster-level enablement
- Schema changes may take up to 5 minutes to propagate
- Consumes SSD disk space even when not in use
- Mappings must be pre-created (no inline mappings)
- Extent tags cannot be set
- Update policy restrictions apply

**Enable streaming ingestion:**
```kusto
// Enable on table
.alter table MyTable policy streamingingestion enable

// Enable on database (applies to all tables)
.alter database MyDatabase policy streamingingestion enable
```

**Performance considerations:**
- 6 concurrent requests per core (e.g., D14/L16 = 96 concurrent requests)
- Database cursor updates may lag up to 60 seconds

---

### 2. Queued (Batched) Ingestion

Optimized for high throughput with automatic batching.

**Default batching thresholds:**
| Property | Default | Min | Max |
|----------|---------|-----|-----|
| MaximumBatchingTimeSpan | 5 minutes | 10 seconds | 30 minutes |
| MaximumNumberOfItems | 500 | 1 | 25,000 |
| MaximumRawDataSizeMB | 1024 MB | 100 MB | 4096 MB |

**Batch sealing triggers:**
- Size limit reached
- Item count limit reached
- Time limit expired
- `FlushImmediately` flag set

**Configure batching policy:**
```kusto
// Set table-level policy
.alter table MyTable policy ingestionbatching
```
'{'
'  "MaximumBatchingTimeSpan": "00:00:30",'
'  "MaximumNumberOfItems": 500,'
'  "MaximumRawDataSizeMB": 1024'
'}'
```

---

### 3. Direct Ingestion (Management Commands)

For exploration, prototyping, and testing only. Not for production.

#### Inline Ingestion
```kusto
.ingest inline into table MyTable <|
    value1,value2,value3
    value4,value5,value6
```

#### Ingest from Query
```kusto
// Create and populate table
.set MyTable <| OtherTable | where Timestamp > ago(1h)

// Append to existing table
.append MyTable <| OtherTable | where Timestamp > ago(1h)

// Create or append
.set-or-append MyTable <| OtherTable | where Timestamp > ago(1h)

// Create or replace
.set-or-replace MyTable <| OtherTable | where Timestamp > ago(1h)
```

#### Ingest from Storage
```kusto
.ingest into table MyTable ('https://storage.blob.core.windows.net/container/file.csv')
    with (format='csv', ingestionMappingReference='MyMapping')
```

---

## Data Connectors

### Azure Native Connectors

#### Event Hubs
- Continuous ingestion from Azure Event Hubs
- Supports streaming and queued ingestion
- System properties mapping available
- Dynamic routing via event properties

**Create connection:**
```csharp
// Set routing properties on event
eventData.Properties.Add("Table", "MyTable");
eventData.Properties.Add("Format", "json");
eventData.Properties.Add("IngestionMappingReference", "MyMapping");
```

#### Event Grid
- Triggers on blob created/renamed in Azure Storage
- Supports Azure Blob Storage and ADLS Gen2
- Metadata-based routing

**Blob metadata properties:**
- `kustoTable` - Target table name
- `kustoDataFormat` - Data format
- `kustoIngestionMappingReference` - Mapping name
- `kustoDatabase` - Target database (multi-database connections)

#### IoT Hub
- Continuous ingestion from Azure IoT Hub
- Similar to Event Hubs with IoT-specific properties
- Device-to-cloud message ingestion

### Third-Party Connectors

#### Apache Kafka
Configuration (adx-sink-config.json):
```json
{
    "connector.class": "com.microsoft.azure.kusto.kafka.connect.sink.KustoSinkConnector",
    "flush.size.bytes": 10000,
    "flush.interval.ms": 10000,
    "topics": "my-topic",
    "kusto.tables.topics.mapping": "[{'topic':'my-topic','db':'MyDB','table':'MyTable','format':'csv'}]",
    "kusto.ingestion.url": "https://ingest-cluster.region.kusto.windows.net",
    "kusto.query.url": "https://cluster.region.kusto.windows.net"
}
```

#### Apache Spark
- Native Spark connector
- Supports batch and streaming modes
- Integrates with Spark DataFrames

#### Apache Flink
- Streaming data pipeline integration
- Supports exactly-once semantics

#### Logstash
- Log aggregation and processing
- JSON format support
- Plugin-based configuration

#### Telegraf
- Metrics collection agent
- Time-series data ingestion

#### Fluent Bit / Fluentd
- Log forwarding and processing
- Lightweight deployment

#### Splunk
- Enterprise log management integration
- Universal Forwarder support

---

## Ingestion Mappings

Mappings define how source data maps to table columns.

### Supported Mapping Types

| Data Format | Mapping Type |
|-------------|--------------|
| CSV, TSV, PSV, TXT | CSV Mapping |
| JSON | JSON Mapping |
| Avro, ApacheAvro | AVRO Mapping |
| Parquet | Parquet Mapping |
| ORC | ORC Mapping |
| W3CLOGFILE | W3CLOGFILE Mapping |

### Create Mapping

**CSV Mapping:**
```kusto
.create table MyTable ingestion csv mapping 'MyCsvMapping'
'['
'  {"column":"col1","Properties":{"Ordinal":"0"}},'
'  {"column":"col2","Properties":{"Ordinal":"1"}},'
'  {"column":"col3","Properties":{"Ordinal":"2"}}'
']'
```

**JSON Mapping:**
```kusto
.create table MyTable ingestion json mapping 'MyJsonMapping'
'['
'  {"column":"timestamp","Properties":{"path":"$.timestamp"}},'
'  {"column":"deviceId","Properties":{"path":"$.device.id"}},'
'  {"column":"value","Properties":{"path":"$.metrics.value"}}'
']'
```

**Parquet Mapping:**
```kusto
.create table MyTable ingestion parquet mapping 'MyParquetMapping'
'['
'  {"column":"col1","Properties":{"path":"$.field1"}},'
'  {"column":"col2","Properties":{"path":"$.field2"}}'
']'
```

### Mapping Transformations

| Transformation | Description |
|---------------|-------------|
| PropertyBagArrayToDictionary | Convert JSON array to dictionary |
| SourceLocation | Include storage artifact name |
| SourceLineNumber | Include line number offset |
| DateTimeFromUnixSeconds | Convert Unix timestamp (seconds) |
| DateTimeFromUnixMilliseconds | Convert Unix timestamp (milliseconds) |
| DateTimeFromUnixMicroseconds | Convert Unix timestamp (microseconds) |
| DateTimeFromUnixNanoseconds | Convert Unix timestamp (nanoseconds) |
| DropMappedFields | Exclude already-mapped nested fields |
| BytesAsBase64 | Convert byte array to Base64 string |

---

## Supported Data Formats

### Text Formats
| Format | Extension | Description |
|--------|-----------|-------------|
| CSV | .csv | Comma-separated values |
| TSV | .tsv | Tab-separated values |
| PSV | .psv | Pipe-separated values |
| SCSV | .scsv | Semicolon-separated values |
| TXT | .txt | Line-delimited text |
| RAW | .raw | Single string value |
| JSON | .json | JSON Lines (JSONL) |
| MultiJSON | .multijson | JSON array or whitespace-delimited objects |
| W3CLOGFILE | .log | W3C Extended Log Format |

### Binary Formats
| Format | Extension | Description |
|--------|-----------|-------------|
| Avro | .avro | Legacy Avro implementation |
| ApacheAvro | .avro | Apache Avro with logical types |
| Parquet | .parquet | Apache Parquet columnar |
| ORC | .orc | Apache ORC columnar |

### Compression
| Type | Extension |
|------|-----------|
| gzip | .gz |
| zip | .zip |

---

## Ingestion Properties

Key properties for controlling ingestion behavior:

| Property | Description |
|----------|-------------|
| format | Data format (csv, json, parquet, etc.) |
| ingestionMappingReference | Pre-created mapping name |
| ingestionMapping | Inline mapping definition |
| ignoreFirstRecord | Skip header row (CSV) |
| tags | Extent tags for the ingested data |
| creationTime | Override extent creation time |
| ingestIfNotExists | Conditional ingestion based on tags |
| validationPolicy | Data validation settings |
| zipPattern | Pattern for files in ZIP archives |

---

## Authentication Methods

### Managed Identity (Recommended)
- System-assigned or user-assigned managed identity
- No secrets to manage
- Automatic credential rotation

### Service Principal
- Azure AD application credentials
- Client ID and secret or certificate

### SAS Token
- Shared Access Signature for storage
- Time-limited access

---

## Document Information

- **Generated**: 2026-02-13
- **Source**: Azure Data Explorer documentation analysis
- **Phase**: 3 - Feature Deep Dive (Data Ingestion)
