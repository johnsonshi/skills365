# Azure Data Explorer API and SDK Integration Examples

This document provides practical code examples for integrating with Azure Data Explorer using various SDKs and the REST API.

---

## 1. .NET SDK Examples

### 1.1 Basic Query Execution

```csharp
using Kusto.Data;
using Kusto.Data.Net.Client;

public async Task<IDataReader> ExecuteQueryAsync()
{
    var connectionString = new KustoConnectionStringBuilder(
        "https://mycluster.westus.kusto.windows.net",
        "MyDatabase")
        .WithAadUserPromptAuthentication();

    using var client = KustoClientFactory.CreateCslQueryProvider(connectionString);

    var query = @"
        StormEvents
        | where StartTime > ago(7d)
        | summarize EventCount = count() by State
        | top 10 by EventCount desc";

    return client.ExecuteQuery(query);
}
```

### 1.2 Query with Parameters

```csharp
using Kusto.Data;
using Kusto.Data.Common;
using Kusto.Data.Net.Client;

public async Task<IDataReader> ExecuteParameterizedQueryAsync(string stateName, int minEvents)
{
    var connectionString = new KustoConnectionStringBuilder(
        "https://mycluster.westus.kusto.windows.net",
        "MyDatabase")
        .WithAadApplicationKeyAuthentication(
            "app-id", "app-secret", "tenant-id");

    using var client = KustoClientFactory.CreateCslQueryProvider(connectionString);

    // Define query with parameters
    var query = @"
        declare query_parameters (targetState:string, minCount:int);
        StormEvents
        | where State == targetState
        | summarize EventCount = count() by EventType
        | where EventCount >= minCount
        | order by EventCount desc";

    // Set up client request properties with parameters
    var crp = new ClientRequestProperties();
    crp.SetParameter("targetState", stateName);
    crp.SetParameter("minCount", minEvents.ToString());
    crp.ClientRequestId = $"MyApp;{Guid.NewGuid()}";
    crp.SetOption(ClientRequestProperties.OptionServerTimeout, TimeSpan.FromMinutes(5));

    return client.ExecuteQuery(query, crp);
}
```

### 1.3 Execute Management Command

```csharp
using Kusto.Data;
using Kusto.Data.Net.Client;

public async Task ExecuteManagementCommandAsync()
{
    var connectionString = new KustoConnectionStringBuilder(
        "https://mycluster.westus.kusto.windows.net",
        "MyDatabase")
        .WithAadManagedIdentity();

    using var admin = KustoClientFactory.CreateCslAdminProvider(connectionString);

    // Show database schema
    var reader = admin.ExecuteControlCommand(".show database schema");

    // Create table
    var createTableCommand = @"
        .create table NewTable (
            Timestamp: datetime,
            EventName: string,
            Value: real,
            Tags: dynamic
        )";
    admin.ExecuteControlCommand(createTableCommand);

    // Create ingestion mapping
    var mappingCommand = @"
        .create table NewTable ingestion json mapping 'JsonMapping'
        '[{""column"":""Timestamp"",""path"":""$.ts"",""datatype"":""datetime""},
          {""column"":""EventName"",""path"":""$.name""},
          {""column"":""Value"",""path"":""$.val""},
          {""column"":""Tags"",""path"":""$.tags""}]'";
    admin.ExecuteControlCommand(mappingCommand);
}
```

### 1.4 Queued Ingestion

```csharp
using Kusto.Data;
using Kusto.Ingest;

public async Task IngestFromBlobAsync(string blobUri)
{
    var connectionString = new KustoConnectionStringBuilder(
        "https://ingest-mycluster.westus.kusto.windows.net",
        "MyDatabase")
        .WithAadApplicationKeyAuthentication(
            "app-id", "app-secret", "tenant-id");

    using var client = KustoIngestFactory.CreateQueuedIngestClient(connectionString);

    var ingestionProperties = new KustoIngestionProperties(
        databaseName: "MyDatabase",
        tableName: "MyTable")
    {
        Format = DataSourceFormat.json,
        IngestionMapping = new IngestionMapping
        {
            IngestionMappingReference = "JsonMapping"
        },
        AdditionalTags = new List<string> { "source:blob", "env:production" }
    };

    var sourceOptions = new StorageSourceOptions
    {
        Size = 1024 * 1024 * 100, // 100 MB estimated
        DeleteSourceOnSuccess = false
    };

    await client.IngestFromStorageAsync(blobUri, ingestionProperties, sourceOptions);
}
```

### 1.5 Streaming Ingestion

```csharp
using Kusto.Data;
using Kusto.Ingest;
using System.Text;

public async Task StreamIngestAsync(IEnumerable<MyEvent> events)
{
    var connectionString = new KustoConnectionStringBuilder(
        "https://mycluster.westus.kusto.windows.net",
        "MyDatabase")
        .WithAadManagedIdentity();

    using var client = KustoIngestFactory.CreateStreamingIngestClient(connectionString);

    var ingestionProperties = new KustoIngestionProperties(
        databaseName: "MyDatabase",
        tableName: "EventsTable")
    {
        Format = DataSourceFormat.json,
        IngestionMapping = new IngestionMapping
        {
            IngestionMappingReference = "JsonMapping"
        }
    };

    // Convert events to JSON stream
    var jsonData = System.Text.Json.JsonSerializer.Serialize(events);
    using var stream = new MemoryStream(Encoding.UTF8.GetBytes(jsonData));

    await client.IngestFromStreamAsync(stream, ingestionProperties, leaveOpen: false);
}
```

### 1.6 Managed Streaming Ingestion (with Fallback)

```csharp
using Kusto.Data;
using Kusto.Ingest;

public async Task ManagedStreamIngestAsync(Stream dataStream)
{
    var engineConnectionString = new KustoConnectionStringBuilder(
        "https://mycluster.westus.kusto.windows.net",
        "MyDatabase")
        .WithAadManagedIdentity();

    var dmConnectionString = new KustoConnectionStringBuilder(
        "https://ingest-mycluster.westus.kusto.windows.net",
        "MyDatabase")
        .WithAadManagedIdentity();

    // Managed streaming tries streaming first, falls back to queued on failure
    using var client = KustoIngestFactory.CreateManagedStreamingIngestClient(
        engineConnectionString,
        dmConnectionString);

    var ingestionProperties = new KustoIngestionProperties(
        databaseName: "MyDatabase",
        tableName: "MyTable")
    {
        Format = DataSourceFormat.csv
    };

    await client.IngestFromStreamAsync(dataStream, ingestionProperties);
}
```

### 1.7 Track Ingestion Status

```csharp
using Kusto.Data;
using Kusto.Ingest;

public async Task IngestWithStatusTrackingAsync(string filePath)
{
    var connectionString = new KustoConnectionStringBuilder(
        "https://ingest-mycluster.westus.kusto.windows.net",
        "MyDatabase")
        .WithAadApplicationKeyAuthentication(
            "app-id", "app-secret", "tenant-id");

    using var client = KustoIngestFactory.CreateQueuedIngestClient(connectionString);

    var ingestionProperties = new KustoQueuedIngestionProperties(
        databaseName: "MyDatabase",
        tableName: "MyTable")
    {
        Format = DataSourceFormat.csv,
        ReportLevel = IngestionReportLevel.FailuresAndSuccesses,
        ReportMethod = IngestionReportMethod.Table
    };

    var result = await client.IngestFromStorageAsync(filePath, ingestionProperties);

    // Poll for status
    var statusTable = result.GetIngestionStatusCollection();
    while (statusTable.Any(s => s.Status == Status.Pending))
    {
        await Task.Delay(TimeSpan.FromSeconds(10));
        statusTable = result.GetIngestionStatusCollection();
    }

    foreach (var status in statusTable)
    {
        Console.WriteLine($"Ingestion {status.IngestionSourceId}: {status.Status}");
        if (status.Status == Status.Failed)
        {
            Console.WriteLine($"  Error: {status.ErrorCode} - {status.Details}");
        }
    }
}
```

---

## 2. Python SDK Examples

### 2.1 Basic Query

```python
from azure.kusto.data import KustoClient, KustoConnectionStringBuilder

cluster = "https://mycluster.westus.kusto.windows.net"
database = "MyDatabase"

# Using service principal
kcsb = KustoConnectionStringBuilder.with_aad_application_key_authentication(
    cluster,
    "app-id",
    "app-secret",
    "tenant-id"
)

client = KustoClient(kcsb)

query = """
StormEvents
| where StartTime > ago(7d)
| summarize EventCount = count() by State
| top 10 by EventCount desc
"""

response = client.execute(database, query)

# Process results
for row in response.primary_results[0]:
    print(f"{row['State']}: {row['EventCount']}")
```

### 2.2 Query with Pandas DataFrame

```python
from azure.kusto.data import KustoClient, KustoConnectionStringBuilder
from azure.kusto.data.helpers import dataframe_from_result_table
import pandas as pd

kcsb = KustoConnectionStringBuilder.with_aad_managed_service_identity_authentication(
    "https://mycluster.westus.kusto.windows.net"
)

client = KustoClient(kcsb)

query = """
StormEvents
| summarize
    TotalEvents = count(),
    TotalDamage = sum(DamageProperty),
    AvgDamage = avg(DamageProperty)
    by State, EventType
| order by TotalDamage desc
"""

response = client.execute("MyDatabase", query)

# Convert to Pandas DataFrame
df = dataframe_from_result_table(response.primary_results[0])

# Now you can use standard Pandas operations
print(df.head(20))
print(df.describe())

# Export to various formats
df.to_csv("storm_analysis.csv", index=False)
df.to_parquet("storm_analysis.parquet")
```

### 2.3 Queued Ingestion

```python
from azure.kusto.data import KustoConnectionStringBuilder
from azure.kusto.ingest import QueuedIngestClient, IngestionProperties, DataFormat

ingest_cluster = "https://ingest-mycluster.westus.kusto.windows.net"

kcsb = KustoConnectionStringBuilder.with_aad_application_key_authentication(
    ingest_cluster,
    "app-id",
    "app-secret",
    "tenant-id"
)

client = QueuedIngestClient(kcsb)

ingestion_props = IngestionProperties(
    database="MyDatabase",
    table="MyTable",
    data_format=DataFormat.CSV,
    ingestion_mapping_reference="CsvMapping"
)

# Ingest from blob
blob_uri = "https://mystorageaccount.blob.core.windows.net/container/file.csv"
client.ingest_from_blob(blob_uri, ingestion_properties=ingestion_props)

# Ingest from local file
client.ingest_from_file("local_data.csv", ingestion_properties=ingestion_props)
```

### 2.4 Ingest Pandas DataFrame

```python
from azure.kusto.data import KustoConnectionStringBuilder
from azure.kusto.ingest import QueuedIngestClient, IngestionProperties, DataFormat
import pandas as pd

kcsb = KustoConnectionStringBuilder.with_aad_managed_service_identity_authentication(
    "https://ingest-mycluster.westus.kusto.windows.net"
)

client = QueuedIngestClient(kcsb)

# Create DataFrame
df = pd.DataFrame({
    "Timestamp": pd.date_range(start="2024-01-01", periods=100, freq="H"),
    "SensorId": ["sensor-" + str(i % 10) for i in range(100)],
    "Temperature": [20 + (i * 0.1) for i in range(100)],
    "Humidity": [50 + (i * 0.2) for i in range(100)]
})

ingestion_props = IngestionProperties(
    database="MyDatabase",
    table="SensorReadings",
    data_format=DataFormat.CSV
)

# Ingest DataFrame
client.ingest_from_dataframe(df, ingestion_properties=ingestion_props)
```

### 2.5 Query with Request Properties

```python
from azure.kusto.data import KustoClient, KustoConnectionStringBuilder, ClientRequestProperties
from datetime import timedelta
import uuid

kcsb = KustoConnectionStringBuilder.with_aad_application_key_authentication(
    "https://mycluster.westus.kusto.windows.net",
    "app-id",
    "app-secret",
    "tenant-id"
)

client = KustoClient(kcsb)

# Set up request properties
properties = ClientRequestProperties()
properties.client_request_id = f"MyApp;{uuid.uuid4()}"
properties.set_option(
    ClientRequestProperties.results_defer_partial_query_failures_option_name,
    True
)
properties.set_option(
    ClientRequestProperties.request_timeout_option_name,
    timedelta(minutes=10)
)

# Set query parameters
properties.set_parameter("startDate", "2024-01-01")
properties.set_parameter("eventType", "Tornado")

query = """
declare query_parameters(startDate:string, eventType:string);
StormEvents
| where StartTime >= datetime(startDate)
| where EventType == eventType
| summarize count() by bin(StartTime, 1d)
"""

response = client.execute("MyDatabase", query, properties)
```

---

## 3. Java SDK Examples

### 3.1 Basic Query

```java
import com.microsoft.azure.kusto.data.*;
import com.microsoft.azure.kusto.data.auth.*;

public class KustoQueryExample {
    public static void main(String[] args) throws Exception {
        String cluster = "https://mycluster.westus.kusto.windows.net";
        String database = "MyDatabase";

        ConnectionStringBuilder csb = ConnectionStringBuilder
            .createWithAadApplicationCredentials(
                cluster,
                "app-id",
                "app-secret",
                "tenant-id"
            );

        try (Client client = ClientFactory.createClient(csb)) {
            String query = "StormEvents | take 10";

            KustoOperationResult result = client.execute(database, query);
            KustoResultSetTable primaryResults = result.getPrimaryResults();

            while (primaryResults.next()) {
                String state = primaryResults.getString("State");
                String eventType = primaryResults.getString("EventType");
                System.out.println(state + ": " + eventType);
            }
        }
    }
}
```

### 3.2 Queued Ingestion

```java
import com.microsoft.azure.kusto.data.*;
import com.microsoft.azure.kusto.ingest.*;

public class KustoIngestExample {
    public static void main(String[] args) throws Exception {
        String ingestCluster = "https://ingest-mycluster.westus.kusto.windows.net";

        ConnectionStringBuilder csb = ConnectionStringBuilder
            .createWithAadApplicationCredentials(
                ingestCluster,
                "app-id",
                "app-secret",
                "tenant-id"
            );

        try (IngestClient client = IngestClientFactory.createClient(csb)) {
            IngestionProperties props = new IngestionProperties(
                "MyDatabase",
                "MyTable"
            );
            props.setDataFormat(IngestionProperties.DataFormat.CSV);
            props.setIngestionMapping("CsvMapping", IngestionMapping.IngestionMappingKind.CSV);

            // Ingest from blob
            BlobSourceInfo blobSource = new BlobSourceInfo(
                "https://storage.blob.core.windows.net/container/file.csv",
                1024 * 1024 // size estimate
            );

            client.ingestFromBlob(blobSource, props);

            // Ingest from file
            FileSourceInfo fileSource = new FileSourceInfo(
                "/path/to/local/file.csv",
                1024 * 1024
            );

            client.ingestFromFile(fileSource, props);
        }
    }
}
```

### 3.3 Streaming Ingestion

```java
import com.microsoft.azure.kusto.data.*;
import com.microsoft.azure.kusto.ingest.*;
import java.io.*;

public class KustoStreamingExample {
    public static void main(String[] args) throws Exception {
        String cluster = "https://mycluster.westus.kusto.windows.net";

        ConnectionStringBuilder csb = ConnectionStringBuilder
            .createWithAadManagedIdentity(cluster);

        try (StreamingIngestClient client = IngestClientFactory.createStreamingIngestClient(csb)) {
            IngestionProperties props = new IngestionProperties(
                "MyDatabase",
                "MyTable"
            );
            props.setDataFormat(IngestionProperties.DataFormat.JSON);
            props.setIngestionMapping("JsonMapping", IngestionMapping.IngestionMappingKind.JSON);

            String jsonData = "[{\"ts\":\"2024-01-01\",\"name\":\"event1\",\"val\":42.5}]";
            InputStream stream = new ByteArrayInputStream(jsonData.getBytes("UTF-8"));

            StreamSourceInfo streamSource = new StreamSourceInfo(stream);
            client.ingestFromStream(streamSource, props);
        }
    }
}
```

---

## 4. Node.js SDK Examples

### 4.1 Basic Query

```javascript
const { Client, KustoConnectionStringBuilder } = require("azure-kusto-data");

async function runQuery() {
    const cluster = "https://mycluster.westus.kusto.windows.net";
    const database = "MyDatabase";

    const kcsb = KustoConnectionStringBuilder.withAadApplicationKeyAuthentication(
        cluster,
        "app-id",
        "app-secret",
        "tenant-id"
    );

    const client = new Client(kcsb);

    const query = `
        StormEvents
        | where StartTime > ago(7d)
        | summarize EventCount = count() by State
        | top 10 by EventCount desc
    `;

    const response = await client.execute(database, query);

    const primaryResults = response.primaryResults[0];
    for (const row of primaryResults.rows()) {
        console.log(`${row.State}: ${row.EventCount}`);
    }
}

runQuery().catch(console.error);
```

### 4.2 Queued Ingestion

```javascript
const { IngestClient, IngestionProperties, DataFormat } = require("azure-kusto-ingest");
const { KustoConnectionStringBuilder } = require("azure-kusto-data");

async function ingestData() {
    const ingestCluster = "https://ingest-mycluster.westus.kusto.windows.net";

    const kcsb = KustoConnectionStringBuilder.withAadApplicationKeyAuthentication(
        ingestCluster,
        "app-id",
        "app-secret",
        "tenant-id"
    );

    const client = new IngestClient(kcsb);

    const ingestionProps = new IngestionProperties({
        database: "MyDatabase",
        table: "MyTable",
        format: DataFormat.CSV,
        ingestionMappingReference: "CsvMapping"
    });

    // Ingest from blob
    const blobUri = "https://storage.blob.core.windows.net/container/file.csv";
    await client.ingestFromBlob(blobUri, ingestionProps);

    // Ingest from local file
    await client.ingestFromFile("./data.csv", ingestionProps);

    console.log("Ingestion queued successfully");
}

ingestData().catch(console.error);
```

### 4.3 TypeScript with Async/Await

```typescript
import { Client, KustoConnectionStringBuilder, ClientRequestProperties } from "azure-kusto-data";

interface StormEvent {
    State: string;
    EventType: string;
    EventCount: number;
}

async function queryWithTypes(): Promise<StormEvent[]> {
    const cluster = "https://mycluster.westus.kusto.windows.net";
    const database = "MyDatabase";

    const kcsb = KustoConnectionStringBuilder.withAadManagedIdentity(cluster);
    const client = new Client(kcsb);

    const properties = new ClientRequestProperties();
    properties.setParameter("minCount", "100");

    const query = `
        declare query_parameters(minCount:long);
        StormEvents
        | summarize EventCount = count() by State, EventType
        | where EventCount >= minCount
        | order by EventCount desc
    `;

    const response = await client.execute(database, query, properties);

    const results: StormEvent[] = [];
    for (const row of response.primaryResults[0].rows()) {
        results.push({
            State: row.State as string,
            EventType: row.EventType as string,
            EventCount: row.EventCount as number
        });
    }

    return results;
}
```

---

## 5. Go SDK Examples

### 5.1 Basic Query

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/Azure/azure-kusto-go/kusto"
    "github.com/Azure/azure-kusto-go/kusto/data/table"
    "github.com/Azure/azure-kusto-go/kusto/unsafe"
)

func main() {
    cluster := "https://mycluster.westus.kusto.windows.net"
    database := "MyDatabase"

    // Using managed identity
    kcsb := kusto.NewConnectionStringBuilder(cluster).
        WithManagedIdentity()

    client, err := kusto.New(kcsb)
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()

    ctx := context.Background()

    query := `
        StormEvents
        | summarize EventCount = count() by State
        | top 10 by EventCount desc
    `

    iter, err := client.Query(ctx, database, kusto.NewStmt("", kusto.UnsafeStmt(unsafe.Stmt{
        Query: query,
    })))
    if err != nil {
        log.Fatal(err)
    }
    defer iter.Stop()

    for {
        row, inlineErr, err := iter.NextRowOrError()
        if err != nil {
            break
        }
        if inlineErr != nil {
            continue
        }

        var state string
        var count int64

        if err := row.ToStruct(&struct {
            State      string `kusto:"State"`
            EventCount int64  `kusto:"EventCount"`
        }{&state, &count}); err == nil {
            fmt.Printf("%s: %d\n", state, count)
        }
    }
}
```

### 5.2 Ingestion

```go
package main

import (
    "context"
    "log"

    "github.com/Azure/azure-kusto-go/kusto"
    "github.com/Azure/azure-kusto-go/kusto/ingest"
)

func main() {
    cluster := "https://ingest-mycluster.westus.kusto.windows.net"
    database := "MyDatabase"
    table := "MyTable"

    kcsb := kusto.NewConnectionStringBuilder(cluster).
        WithAadAppKey("app-id", "app-secret", "tenant-id")

    client, err := ingest.New(kcsb, ingest.Database(database), ingest.Table(table))
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()

    ctx := context.Background()

    // Ingest from blob
    blobUri := "https://storage.blob.core.windows.net/container/file.csv"
    if _, err := client.FromFile(ctx, blobUri, ingest.FileFormat(ingest.CSV)); err != nil {
        log.Fatal(err)
    }

    // Ingest from local file
    if _, err := client.FromFile(ctx, "./data.csv", ingest.FileFormat(ingest.CSV)); err != nil {
        log.Fatal(err)
    }
}
```

---

## 6. REST API Examples

### 6.1 Query via cURL

```bash
# Get access token
ACCESS_TOKEN=$(az account get-access-token \
    --resource https://api.kusto.windows.net \
    --query accessToken -o tsv)

# Execute query
curl -X POST \
    "https://mycluster.westus.kusto.windows.net/v2/rest/query" \
    -H "Authorization: Bearer $ACCESS_TOKEN" \
    -H "Content-Type: application/json" \
    -H "Accept: application/json" \
    -H "x-ms-client-request-id: $(uuidgen)" \
    -d '{
        "db": "MyDatabase",
        "csl": "StormEvents | take 10"
    }'
```

### 6.2 Management Command via cURL

```bash
ACCESS_TOKEN=$(az account get-access-token \
    --resource https://api.kusto.windows.net \
    --query accessToken -o tsv)

# Execute management command
curl -X POST \
    "https://mycluster.westus.kusto.windows.net/v1/rest/mgmt" \
    -H "Authorization: Bearer $ACCESS_TOKEN" \
    -H "Content-Type: application/json" \
    -H "Accept: application/json" \
    -d '{
        "db": "MyDatabase",
        "csl": ".show tables"
    }'
```

### 6.3 Streaming Ingestion via REST

```bash
ACCESS_TOKEN=$(az account get-access-token \
    --resource https://api.kusto.windows.net \
    --query accessToken -o tsv)

# Stream ingest JSON data
curl -X POST \
    "https://mycluster.westus.kusto.windows.net/v1/rest/ingest/MyDatabase/MyTable?streamFormat=json&mappingName=JsonMapping" \
    -H "Authorization: Bearer $ACCESS_TOKEN" \
    -H "Content-Type: application/json" \
    -d '[{"ts":"2024-01-01T00:00:00Z","name":"event1","val":42.5}]'
```

### 6.4 Python REST API Client

```python
import requests
from azure.identity import DefaultAzureCredential

def execute_kusto_query(cluster: str, database: str, query: str) -> dict:
    """Execute a Kusto query via REST API."""

    # Get token
    credential = DefaultAzureCredential()
    token = credential.get_token("https://api.kusto.windows.net/.default")

    # Build request
    url = f"{cluster}/v2/rest/query"
    headers = {
        "Authorization": f"Bearer {token.token}",
        "Content-Type": "application/json",
        "Accept": "application/json"
    }

    payload = {
        "db": database,
        "csl": query,
        "properties": {
            "Options": {
                "servertimeout": "00:10:00"
            }
        }
    }

    response = requests.post(url, json=payload, headers=headers)
    response.raise_for_status()

    return response.json()

# Usage
result = execute_kusto_query(
    "https://mycluster.westus.kusto.windows.net",
    "MyDatabase",
    "StormEvents | take 10"
)

# Parse v2 response
for frame in result:
    if frame.get("FrameType") == "DataTable":
        columns = frame["Columns"]
        rows = frame["Rows"]
        for row in rows:
            print(dict(zip([c["ColumnName"] for c in columns], row)))
```

---

## 7. PowerShell Examples

### 7.1 Query with Service Principal

```powershell
# Download and extract Kusto.Tools
$toolsPath = "C:\KustoTools"
# Load assembly
Add-Type -Path "$toolsPath\Kusto.Data.dll"

# Build connection string
$cluster = "https://mycluster.westus.kusto.windows.net"
$database = "MyDatabase"

$kcsb = New-Object Kusto.Data.KustoConnectionStringBuilder($cluster, $database)
$kcsb = $kcsb.WithAadApplicationKeyAuthentication(
    "app-id",
    "app-secret",
    "tenant-id"
)

# Create client and execute query
$queryProvider = [Kusto.Data.Net.Client.KustoClientFactory]::CreateCslQueryProvider($kcsb)

$query = @"
StormEvents
| summarize EventCount = count() by State
| top 10 by EventCount desc
"@

$reader = $queryProvider.ExecuteQuery($query)

# Convert to DataTable for easy processing
$dataTable = New-Object System.Data.DataTable
$dataTable.Load($reader)

# Display results
$dataTable | Format-Table -AutoSize
```

### 7.2 Batch Query Execution

```powershell
Add-Type -Path "C:\KustoTools\Kusto.Data.dll"

$cluster = "https://mycluster.westus.kusto.windows.net"
$databases = @("Database1", "Database2", "Database3")

foreach ($database in $databases) {
    $kcsb = New-Object Kusto.Data.KustoConnectionStringBuilder($cluster, $database)
    $kcsb = $kcsb.WithAadManagedIdentity()

    $queryProvider = [Kusto.Data.Net.Client.KustoClientFactory]::CreateCslQueryProvider($kcsb)

    $query = ".show database $database extents | summarize TotalSize = sum(OriginalSize)"

    try {
        $reader = $queryProvider.ExecuteQuery($query)
        $result = New-Object System.Data.DataTable
        $result.Load($reader)

        Write-Host "$database : $($result.Rows[0]['TotalSize'] / 1GB) GB"
    }
    catch {
        Write-Warning "Failed to query $database : $_"
    }
    finally {
        $queryProvider.Dispose()
    }
}
```

### 7.3 Ingestion with Status Tracking

```powershell
Add-Type -Path "C:\KustoTools\Kusto.Data.dll"
Add-Type -Path "C:\KustoTools\Kusto.Ingest.dll"

$ingestCluster = "https://ingest-mycluster.westus.kusto.windows.net"
$database = "MyDatabase"
$table = "MyTable"

$kcsb = New-Object Kusto.Data.KustoConnectionStringBuilder($ingestCluster, $database)
$kcsb = $kcsb.WithAadApplicationKeyAuthentication("app-id", "app-secret", "tenant-id")

$ingestClient = [Kusto.Ingest.KustoIngestFactory]::CreateQueuedIngestClient($kcsb)

# Set up ingestion properties with status reporting
$ingestionProps = New-Object Kusto.Ingest.KustoQueuedIngestionProperties($database, $table)
$ingestionProps.Format = [Kusto.Data.Common.DataSourceFormat]::csv
$ingestionProps.ReportLevel = [Kusto.Ingest.IngestionReportLevel]::FailuresAndSuccesses
$ingestionProps.ReportMethod = [Kusto.Ingest.IngestionReportMethod]::Table

# Ingest file
$filePath = "C:\Data\events.csv"
$result = $ingestClient.IngestFromStorageAsync($filePath, $ingestionProps).Result

# Check status
Start-Sleep -Seconds 30
$statusCollection = $result.GetIngestionStatusCollection()

foreach ($status in $statusCollection) {
    Write-Host "Source: $($status.IngestionSourceId)"
    Write-Host "Status: $($status.Status)"
    if ($status.Status -eq [Kusto.Ingest.Status]::Failed) {
        Write-Host "Error: $($status.ErrorCode) - $($status.Details)"
    }
    Write-Host "---"
}

$ingestClient.Dispose()
```

---

## 8. Advanced Integration Patterns

### 8.1 Connection Pooling (.NET)

```csharp
using Kusto.Data;
using Kusto.Data.Net.Client;
using Microsoft.Extensions.DependencyInjection;

// Register as singleton for connection pooling
public static class KustoClientExtensions
{
    public static IServiceCollection AddKustoClient(
        this IServiceCollection services,
        string clusterUri,
        string database)
    {
        services.AddSingleton<ICslQueryProvider>(sp =>
        {
            var kcsb = new KustoConnectionStringBuilder(clusterUri, database)
                .WithAadManagedIdentity();

            return KustoClientFactory.CreateCslQueryProvider(kcsb);
        });

        services.AddSingleton<ICslAdminProvider>(sp =>
        {
            var kcsb = new KustoConnectionStringBuilder(clusterUri, database)
                .WithAadManagedIdentity();

            return KustoClientFactory.CreateCslAdminProvider(kcsb);
        });

        return services;
    }
}
```

### 8.2 Retry Pattern (Python)

```python
from azure.kusto.data import KustoClient, KustoConnectionStringBuilder
from azure.kusto.data.exceptions import KustoServiceError
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=30),
    retry=retry_if_exception_type(KustoServiceError)
)
def execute_query_with_retry(client: KustoClient, database: str, query: str):
    """Execute query with exponential backoff retry."""
    return client.execute(database, query)

# Usage
kcsb = KustoConnectionStringBuilder.with_aad_managed_service_identity_authentication(
    "https://mycluster.westus.kusto.windows.net"
)
client = KustoClient(kcsb)

try:
    result = execute_query_with_retry(client, "MyDatabase", "MyTable | take 10")
except KustoServiceError as e:
    print(f"Query failed after retries: {e}")
```

### 8.3 Batch Ingestion Manager

```csharp
using Kusto.Data;
using Kusto.Ingest;
using System.Collections.Concurrent;

public class BatchIngestionManager : IDisposable
{
    private readonly IKustoQueuedIngestClient _client;
    private readonly KustoIngestionProperties _properties;
    private readonly ConcurrentQueue<string> _pendingFiles;
    private readonly SemaphoreSlim _semaphore;
    private readonly int _maxConcurrency;

    public BatchIngestionManager(
        string ingestCluster,
        string database,
        string table,
        int maxConcurrency = 10)
    {
        var kcsb = new KustoConnectionStringBuilder(ingestCluster, database)
            .WithAadManagedIdentity();

        _client = KustoIngestFactory.CreateQueuedIngestClient(kcsb);
        _properties = new KustoIngestionProperties(database, table)
        {
            Format = DataSourceFormat.csv
        };
        _pendingFiles = new ConcurrentQueue<string>();
        _semaphore = new SemaphoreSlim(maxConcurrency);
        _maxConcurrency = maxConcurrency;
    }

    public async Task IngestFilesAsync(IEnumerable<string> filePaths)
    {
        var tasks = filePaths.Select(async filePath =>
        {
            await _semaphore.WaitAsync();
            try
            {
                await _client.IngestFromStorageAsync(filePath, _properties);
                Console.WriteLine($"Queued: {filePath}");
            }
            finally
            {
                _semaphore.Release();
            }
        });

        await Task.WhenAll(tasks);
    }

    public void Dispose()
    {
        _client?.Dispose();
        _semaphore?.Dispose();
    }
}
```

---

*Generated: 2026-02-13 | Source: Azure Data Explorer Documentation Analysis | Phase 3 - API and SDK Deep Investigation*
