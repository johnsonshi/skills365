# Azure Data Explorer SDK Code Examples

## .NET SDK

### Basic Query

```csharp
using Kusto.Data;
using Kusto.Data.Net.Client;

var kcsb = new KustoConnectionStringBuilder(
    "https://mycluster.westus.kusto.windows.net", "MyDatabase")
    .WithAadManagedIdentity();

using var client = KustoClientFactory.CreateCslQueryProvider(kcsb);

var query = @"
    StormEvents
    | summarize EventCount = count() by State
    | top 10 by EventCount desc";

var reader = client.ExecuteQuery(query);
```

### Parameterized Query

```csharp
using Kusto.Data.Common;

var query = @"
    declare query_parameters (targetState:string, minCount:int);
    StormEvents
    | where State == targetState
    | summarize EventCount = count() by EventType
    | where EventCount >= minCount";

var crp = new ClientRequestProperties();
crp.SetParameter("targetState", "TEXAS");
crp.SetParameter("minCount", "100");
crp.ClientRequestId = $"MyApp;{Guid.NewGuid()}";
crp.SetOption(ClientRequestProperties.OptionServerTimeout, TimeSpan.FromMinutes(5));

var reader = client.ExecuteQuery(query, crp);
```

### Management Command

```csharp
using var admin = KustoClientFactory.CreateCslAdminProvider(kcsb);

// Show database schema
admin.ExecuteControlCommand(".show database schema");

// Create table
admin.ExecuteControlCommand(@"
    .create table Events (
        Timestamp: datetime,
        EventName: string,
        Value: real
    )");
```

### Queued Ingestion

```csharp
using Kusto.Ingest;

var ingestKcsb = new KustoConnectionStringBuilder(
    "https://ingest-mycluster.westus.kusto.windows.net", "MyDatabase")
    .WithAadApplicationKeyAuthentication("app-id", "app-secret", "tenant-id");

using var client = KustoIngestFactory.CreateQueuedIngestClient(ingestKcsb);

var props = new KustoIngestionProperties("MyDatabase", "MyTable") {
    Format = DataSourceFormat.json,
    IngestionMapping = new IngestionMapping { IngestionMappingReference = "JsonMapping" }
};

await client.IngestFromStorageAsync(blobUri, props);
```

### Streaming Ingestion

```csharp
using var client = KustoIngestFactory.CreateStreamingIngestClient(kcsb);

var props = new KustoIngestionProperties("MyDatabase", "MyTable") {
    Format = DataSourceFormat.json,
    IngestionMapping = new IngestionMapping { IngestionMappingReference = "JsonMapping" }
};

var jsonData = JsonSerializer.Serialize(events);
using var stream = new MemoryStream(Encoding.UTF8.GetBytes(jsonData));

await client.IngestFromStreamAsync(stream, props);
```

---

## Python SDK

### Basic Query

```python
from azure.kusto.data import KustoClient, KustoConnectionStringBuilder

kcsb = KustoConnectionStringBuilder.with_aad_application_key_authentication(
    "https://mycluster.westus.kusto.windows.net",
    "app-id", "app-secret", "tenant-id"
)

client = KustoClient(kcsb)

query = """
StormEvents
| summarize EventCount = count() by State
| top 10 by EventCount desc
"""

response = client.execute("MyDatabase", query)

for row in response.primary_results[0]:
    print(f"{row['State']}: {row['EventCount']}")
```

### Query to Pandas DataFrame

```python
from azure.kusto.data.helpers import dataframe_from_result_table

response = client.execute("MyDatabase", query)
df = dataframe_from_result_table(response.primary_results[0])

print(df.head(20))
df.to_csv("output.csv", index=False)
```

### Parameterized Query

```python
from azure.kusto.data import ClientRequestProperties
from datetime import timedelta
import uuid

properties = ClientRequestProperties()
properties.client_request_id = f"MyApp;{uuid.uuid4()}"
properties.set_option(
    ClientRequestProperties.request_timeout_option_name,
    timedelta(minutes=10)
)
properties.set_parameter("startDate", "2024-01-01")
properties.set_parameter("eventType", "Tornado")

query = """
declare query_parameters(startDate:string, eventType:string);
StormEvents
| where StartTime >= datetime(startDate)
| where EventType == eventType
"""

response = client.execute("MyDatabase", query, properties)
```

### Queued Ingestion

```python
from azure.kusto.ingest import QueuedIngestClient, IngestionProperties, DataFormat

kcsb = KustoConnectionStringBuilder.with_aad_application_key_authentication(
    "https://ingest-mycluster.westus.kusto.windows.net",
    "app-id", "app-secret", "tenant-id"
)

client = QueuedIngestClient(kcsb)

props = IngestionProperties(
    database="MyDatabase",
    table="MyTable",
    data_format=DataFormat.CSV,
    ingestion_mapping_reference="CsvMapping"
)

# From blob
client.ingest_from_blob(blob_uri, ingestion_properties=props)

# From local file
client.ingest_from_file("data.csv", ingestion_properties=props)
```

### Ingest DataFrame

```python
import pandas as pd

df = pd.DataFrame({
    "Timestamp": pd.date_range(start="2024-01-01", periods=100, freq="H"),
    "SensorId": ["sensor-" + str(i % 10) for i in range(100)],
    "Temperature": [20 + (i * 0.1) for i in range(100)]
})

props = IngestionProperties(
    database="MyDatabase",
    table="SensorReadings",
    data_format=DataFormat.CSV
)

client.ingest_from_dataframe(df, ingestion_properties=props)
```

---

## Java SDK

### Basic Query

```java
import com.microsoft.azure.kusto.data.*;

ConnectionStringBuilder csb = ConnectionStringBuilder
    .createWithAadApplicationCredentials(
        "https://mycluster.westus.kusto.windows.net",
        "app-id", "app-secret", "tenant-id"
    );

try (Client client = ClientFactory.createClient(csb)) {
    KustoOperationResult result = client.execute("MyDatabase", "StormEvents | take 10");
    KustoResultSetTable table = result.getPrimaryResults();

    while (table.next()) {
        String state = table.getString("State");
        System.out.println(state);
    }
}
```

### Queued Ingestion

```java
import com.microsoft.azure.kusto.ingest.*;

ConnectionStringBuilder csb = ConnectionStringBuilder
    .createWithAadApplicationCredentials(
        "https://ingest-mycluster.westus.kusto.windows.net",
        "app-id", "app-secret", "tenant-id"
    );

try (IngestClient client = IngestClientFactory.createClient(csb)) {
    IngestionProperties props = new IngestionProperties("MyDatabase", "MyTable");
    props.setDataFormat(IngestionProperties.DataFormat.CSV);
    props.setIngestionMapping("CsvMapping", IngestionMapping.IngestionMappingKind.CSV);

    BlobSourceInfo blobSource = new BlobSourceInfo(blobUri, 1024 * 1024);
    client.ingestFromBlob(blobSource, props);
}
```

### Streaming Ingestion

```java
try (StreamingIngestClient client = IngestClientFactory.createStreamingIngestClient(csb)) {
    IngestionProperties props = new IngestionProperties("MyDatabase", "MyTable");
    props.setDataFormat(IngestionProperties.DataFormat.JSON);

    String jsonData = "[{\"ts\":\"2024-01-01\",\"name\":\"event1\",\"val\":42.5}]";
    InputStream stream = new ByteArrayInputStream(jsonData.getBytes("UTF-8"));

    client.ingestFromStream(new StreamSourceInfo(stream), props);
}
```

---

## Node.js SDK

### Basic Query

```javascript
const { Client, KustoConnectionStringBuilder } = require("azure-kusto-data");

const kcsb = KustoConnectionStringBuilder.withAadApplicationKeyAuthentication(
    "https://mycluster.westus.kusto.windows.net",
    "app-id", "app-secret", "tenant-id"
);

const client = new Client(kcsb);

const query = `
    StormEvents
    | summarize EventCount = count() by State
    | top 10 by EventCount desc
`;

const response = await client.execute("MyDatabase", query);

for (const row of response.primaryResults[0].rows()) {
    console.log(`${row.State}: ${row.EventCount}`);
}
```

### TypeScript with Parameters

```typescript
import { Client, KustoConnectionStringBuilder, ClientRequestProperties } from "azure-kusto-data";

const kcsb = KustoConnectionStringBuilder.withAadManagedIdentity(cluster);
const client = new Client(kcsb);

const properties = new ClientRequestProperties();
properties.setParameter("minCount", "100");

const query = `
    declare query_parameters(minCount:long);
    StormEvents
    | summarize EventCount = count() by State
    | where EventCount >= minCount
`;

const response = await client.execute("MyDatabase", query, properties);
```

### Queued Ingestion

```javascript
const { IngestClient, IngestionProperties, DataFormat } = require("azure-kusto-ingest");

const kcsb = KustoConnectionStringBuilder.withAadApplicationKeyAuthentication(
    "https://ingest-mycluster.westus.kusto.windows.net",
    "app-id", "app-secret", "tenant-id"
);

const client = new IngestClient(kcsb);

const props = new IngestionProperties({
    database: "MyDatabase",
    table: "MyTable",
    format: DataFormat.CSV,
    ingestionMappingReference: "CsvMapping"
});

await client.ingestFromBlob(blobUri, props);
await client.ingestFromFile("./data.csv", props);
```

---

## Go SDK

### Basic Query

```go
package main

import (
    "context"
    "github.com/Azure/azure-kusto-go/kusto"
)

func main() {
    kcsb := kusto.NewConnectionStringBuilder("https://mycluster.westus.kusto.windows.net").
        WithManagedIdentity()

    client, _ := kusto.New(kcsb)
    defer client.Close()

    ctx := context.Background()
    query := "StormEvents | summarize count() by State | top 10 by count_"

    iter, _ := client.Query(ctx, "MyDatabase", kusto.NewStmt(query))
    defer iter.Stop()

    for {
        row, _, err := iter.NextRowOrError()
        if err != nil {
            break
        }
        // Process row
    }
}
```

### Ingestion

```go
import "github.com/Azure/azure-kusto-go/kusto/ingest"

kcsb := kusto.NewConnectionStringBuilder("https://ingest-mycluster.westus.kusto.windows.net").
    WithAadAppKey("app-id", "app-secret", "tenant-id")

client, _ := ingest.New(kcsb, ingest.Database("MyDatabase"), ingest.Table("MyTable"))
defer client.Close()

ctx := context.Background()
client.FromFile(ctx, blobUri, ingest.FileFormat(ingest.CSV))
```

---

## REST API (cURL)

### Query

```bash
ACCESS_TOKEN=$(az account get-access-token \
    --resource https://api.kusto.windows.net \
    --query accessToken -o tsv)

curl -X POST \
    "https://mycluster.westus.kusto.windows.net/v2/rest/query" \
    -H "Authorization: Bearer $ACCESS_TOKEN" \
    -H "Content-Type: application/json" \
    -H "x-ms-client-request-id: $(uuidgen)" \
    -d '{"db": "MyDatabase", "csl": "StormEvents | take 10"}'
```

### Management Command

```bash
curl -X POST \
    "https://mycluster.westus.kusto.windows.net/v1/rest/mgmt" \
    -H "Authorization: Bearer $ACCESS_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"db": "MyDatabase", "csl": ".show tables"}'
```

### Streaming Ingestion

```bash
curl -X POST \
    "https://mycluster.westus.kusto.windows.net/v1/rest/ingest/MyDatabase/MyTable?streamFormat=json&mappingName=JsonMapping" \
    -H "Authorization: Bearer $ACCESS_TOKEN" \
    -H "Content-Type: application/json" \
    -d '[{"ts":"2024-01-01T00:00:00Z","name":"event1","val":42.5}]'
```

---

## PowerShell

### Query

```powershell
Add-Type -Path "C:\KustoTools\Kusto.Data.dll"

$kcsb = New-Object Kusto.Data.KustoConnectionStringBuilder(
    "https://mycluster.westus.kusto.windows.net", "MyDatabase")
$kcsb = $kcsb.WithAadApplicationKeyAuthentication("app-id", "app-secret", "tenant-id")

$client = [Kusto.Data.Net.Client.KustoClientFactory]::CreateCslQueryProvider($kcsb)

$query = "StormEvents | summarize count() by State | top 10 by count_"
$reader = $client.ExecuteQuery($query)

$dataTable = New-Object System.Data.DataTable
$dataTable.Load($reader)
$dataTable | Format-Table -AutoSize
```

### Ingestion with Status

```powershell
Add-Type -Path "C:\KustoTools\Kusto.Ingest.dll"

$kcsb = New-Object Kusto.Data.KustoConnectionStringBuilder(
    "https://ingest-mycluster.westus.kusto.windows.net", "MyDatabase")
$kcsb = $kcsb.WithAadApplicationKeyAuthentication("app-id", "app-secret", "tenant-id")

$client = [Kusto.Ingest.KustoIngestFactory]::CreateQueuedIngestClient($kcsb)

$props = New-Object Kusto.Ingest.KustoQueuedIngestionProperties("MyDatabase", "MyTable")
$props.Format = [Kusto.Data.Common.DataSourceFormat]::csv
$props.ReportLevel = [Kusto.Ingest.IngestionReportLevel]::FailuresAndSuccesses

$result = $client.IngestFromStorageAsync("C:\Data\events.csv", $props).Result

Start-Sleep -Seconds 30
$result.GetIngestionStatusCollection() | ForEach-Object {
    Write-Host "Status: $($_.Status)"
}
```

---

## Advanced Patterns

### Connection Pool (.NET DI)

```csharp
public static class KustoClientExtensions
{
    public static IServiceCollection AddKustoClient(
        this IServiceCollection services, string clusterUri, string database)
    {
        services.AddSingleton<ICslQueryProvider>(sp =>
        {
            var kcsb = new KustoConnectionStringBuilder(clusterUri, database)
                .WithAadManagedIdentity();
            return KustoClientFactory.CreateCslQueryProvider(kcsb);
        });
        return services;
    }
}
```

### Retry with Tenacity (Python)

```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type
from azure.kusto.data.exceptions import KustoServiceError

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=30),
    retry=retry_if_exception_type(KustoServiceError)
)
def execute_query_with_retry(client, database, query):
    return client.execute(database, query)
```
