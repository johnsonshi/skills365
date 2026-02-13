# Azure Data Explorer API and SDK Integration Reference

## Overview

Azure Data Explorer provides multiple ways to interact with the service programmatically:
- REST API for direct HTTP communication
- Client SDKs for various programming languages
- PowerShell integration via .NET libraries

---

## REST API

### Endpoints

Azure Data Explorer exposes two main endpoints:

| Endpoint Type | Description | URI Pattern |
|---------------|-------------|-------------|
| Engine (Cluster URI) | Query and management operations | `https://<cluster>.<region>.kusto.windows.net` |
| Data Management (Ingestion URI) | Data ingestion operations | `https://ingest-<cluster>.<region>.kusto.windows.net` |

### Supported Actions

| Action | HTTP Verb | Resource Path | Description |
|--------|-----------|---------------|-------------|
| Query (v1) | GET/POST | `/v1/rest/query` | Execute KQL queries |
| Query (v2) | GET/POST | `/v2/rest/query` | Execute KQL queries (improved response format) |
| Management | POST | `/v1/rest/mgmt` | Execute management commands |
| Streaming Ingest | POST | `/v1/rest/ingest` | Direct streaming ingestion |

### Authentication

All REST API calls require a Microsoft Entra ID (Azure AD) bearer token in the `Authorization` header:

```
Authorization: Bearer <access_token>
```

The resource for token acquisition:
- Azure Data Explorer: `https://api.kusto.windows.net`
- Microsoft Fabric: `https://api.fabric.microsoft.com`

### Request Format

**Headers:**
| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | Yes | Bearer token for authentication |
| `Content-Type` | Yes | `application/json; charset=utf-8` |
| `Accept` | Yes | `application/json` |
| `x-ms-client-request-id` | Recommended | Unique request identifier for tracing |
| `x-ms-app` | Optional | Application name for telemetry |
| `x-ms-user` | Optional | User name for telemetry |

**Body (POST):**
```json
{
  "db": "<database_name>",
  "csl": "<query_or_command>",
  "properties": {
    "Options": { ... },
    "Parameters": { ... }
  }
}
```

---

## .NET SDK

### Packages

| Package | NuGet | Purpose |
|---------|-------|---------|
| Microsoft.Azure.Kusto.Data | [NuGet](https://www.nuget.org/packages/Microsoft.Azure.Kusto.Data/) | Query execution, management commands |
| Microsoft.Azure.Kusto.Ingest | [NuGet](https://www.nuget.org/packages/Microsoft.Azure.Kusto.Ingest/) | Data ingestion |
| Microsoft.Azure.Kusto.Language | [NuGet](https://www.nuget.org/packages/Microsoft.Azure.Kusto.Language/) | Query parsing, semantic analysis |
| Azure.ResourceManager.Kusto | [NuGet](https://www.nuget.org/packages/Azure.ResourceManager.Kusto/) | ARM/Resource management |

### Kusto.Data (Query Library)

**Key Classes:**
- `KustoConnectionStringBuilder` - Build connection strings with authentication
- `KustoClientFactory` - Factory for creating query/admin providers
- `ICslQueryProvider` - Interface for query execution
- `ICslAdminProvider` - Interface for management command execution
- `ClientRequestProperties` - Request options (timeout, parameters, etc.)

**Factory Methods:**
```csharp
// Create query provider
var queryProvider = KustoClientFactory.CreateCslQueryProvider(connectionString);

// Create admin provider (for management commands)
var adminProvider = KustoClientFactory.CreateCslAdminProvider(connectionString);
```

### Kusto.Ingest (Ingestion Library)

**Ingestion Client Types:**

| Client Type | Class/Method | Use Case |
|-------------|--------------|----------|
| Queued | `KustoIngestFactory.CreateQueuedIngestClient()` | Production workloads, batched ingestion |
| Direct | `KustoIngestFactory.CreateDirectIngestClient()` | Development/testing only |
| Streaming | `KustoIngestFactory.CreateStreamingIngestClient()` | Low-latency, small data |
| Managed Streaming | `KustoIngestFactory.CreateManagedStreamingIngestClient()` | Streaming with fallback to queued |

**Key Interfaces:**
- `IKustoIngestClient` - Base ingestion interface
- `IKustoQueuedIngestClient` - Queued ingestion with status tracking
- `KustoIngestionProperties` - Ingestion configuration (database, table, format, mapping)
- `KustoQueuedIngestionProperties` - Extended properties for queued ingestion

**Supported Data Sources:**
- `IngestFromStorageAsync()` - Files, blobs, ADLS
- `IngestFromStreamAsync()` - In-memory streams
- `IngestFromDataReaderAsync()` - IDataReader objects

---

## Python SDK

### Packages

| Package | PyPI | Purpose |
|---------|------|---------|
| azure-kusto-data | [PyPI](https://pypi.org/project/azure-kusto-data/) | Query execution |
| azure-kusto-ingest | [PyPI](https://pypi.org/project/azure-kusto-ingest/) | Data ingestion |
| azure-mgmt-kusto | [PyPI](https://pypi.org/project/azure-mgmt-kusto/) | Resource management |

### GitHub Repository
[Azure/azure-kusto-python](https://github.com/Azure/azure-kusto-python)

### Key Classes

**azure-kusto-data:**
- `KustoClient` - Main client for queries and commands
- `KustoConnectionStringBuilder` - Connection string builder with authentication methods
- `ClientRequestProperties` - Request options

**azure-kusto-ingest:**
- `QueuedIngestClient` - Queued ingestion client
- `IngestionProperties` - Ingestion configuration
- `DataFormat` - Enum for data formats (CSV, JSON, Parquet, etc.)

### Compatibility
- Python 2.x and 3.x compatible
- Supports Python DB API interface
- Pandas DataFrame integration for result handling

---

## Java SDK

### Packages

| Package | Maven | Purpose |
|---------|-------|---------|
| kusto-data | [Maven](https://mvnrepository.com/artifact/com.microsoft.azure.kusto/kusto-data) | Query execution |
| kusto-ingest | [Maven](https://mvnrepository.com/artifact/com.microsoft.azure.kusto/kusto-ingest) | Data ingestion |

### GitHub Repository
[Azure/azure-kusto-java](https://github.com/Azure/azure-kusto-java)

### Key Classes

**Query:**
- `Client` - Main query client
- `ClientFactory` - Factory for creating clients
- `ConnectionStringBuilder` - Connection string with authentication
- `KustoOperationResult` - Query result container
- `KustoResultSetTable` - Result table iterator

**Ingestion:**
- `QueuedIngestClient` - Queued ingestion
- `IngestClientFactory` - Factory for ingest clients
- `IngestionProperties` - Ingestion configuration
- `FileSourceInfo` / `BlobSourceInfo` / `StreamSourceInfo` - Data source descriptors

---

## Node.js SDK

### Packages

| Package | npm | Purpose |
|---------|-----|---------|
| azure-kusto-data | [npm](https://www.npmjs.com/package/azure-kusto-data) | Query execution |
| azure-kusto-ingest | [npm](https://www.npmjs.com/package/azure-kusto-ingest) | Data ingestion |
| azure-arm-kusto | [npm](https://www.npmjs.com/package/azure-arm-kusto) | Resource management |

### GitHub Repository
[Azure/azure-kusto-node](https://github.com/Azure/azure-kusto-node)

### Key Classes
- `Client` (KustoClient) - Query and management client
- `KustoConnectionStringBuilder` - Connection builder
- `IngestClient` - Ingestion client
- `IngestionProperties` - Ingestion configuration

### Compatibility
- Node.js LTS v6.14+
- Built with ES6
- Browser support (TypeScript)

---

## Go SDK

### Package
[github.com/Azure/azure-kusto-go](https://github.com/Azure/azure-kusto-go)

### Installation
```bash
go get github.com/Azure/azure-kusto-go/kusto
```

### Minimum Requirements
- Go 1.13+

### Key Components
- `kusto.Client` - Query execution with iterator-based results
- `azkustoingest` - Ingestion package
- Resource management: [azure-sdk-for-go/kusto](https://github.com/Azure/azure-sdk-for-go/tree/main/sdk/resourcemanager/kusto)

### Documentation
[GoDoc](https://godoc.org/github.com/Azure/azure-kusto-go)

---

## R SDK

### Package
[AzureKusto](https://cran.r-project.org/web/packages/AzureKusto/index.html) (CRAN)

### GitHub Repository
Part of the [cloudyr project](https://github.com/cloudyr/AzureKusto)

### Features
- Query execution
- Result handling
- dplyr integration for data manipulation

---

## PowerShell Integration

### Package
[Microsoft.Azure.Kusto.Tools](https://www.nuget.org/packages/Microsoft.Azure.Kusto.Tools/) (NuGet)

PowerShell uses the .NET libraries directly. The tools package includes all necessary assemblies.

### Azure Module
[Az.Kusto](https://www.powershellgallery.com/packages/Az.Kusto/) - For cluster and database management via Azure Resource Manager

### Setup Steps
1. Download Microsoft.Azure.Kusto.Tools NuGet package
2. Extract the package
3. Load the appropriate DLL based on PowerShell version:
   - PowerShell 5.1: Use `net472` folder
   - PowerShell 7+: Use any folder except `net472`
4. Load assembly: `[System.Reflection.Assembly]::LoadFrom("<path>\Kusto.Data.dll")`

### Key Objects
- `Kusto.Data.KustoConnectionStringBuilder` - Connection string builder
- `Kusto.Data.Net.Client.KustoClientFactory` - Factory for query/admin providers
- `Kusto.Data.Common.ClientRequestProperties` - Request options

---

## Connection String Format

Connection strings follow ADO.NET format with semicolon-delimited name-value pairs:

```
Data Source=https://cluster.region.kusto.windows.net;Initial Catalog=database;Fed=true
```

### Key Properties

| Property | Aliases | Description |
|----------|---------|-------------|
| Data Source | Addr, Address, Server | Cluster URI |
| Initial Catalog | Database | Default database |
| Fed / AADFed | Federated Security | Enable Microsoft Entra auth |
| Authority ID | TenantId | Azure AD tenant |
| Application Client ID | AppClientId | App registration ID |
| Application Key | AppKey | App secret |
| User ID | UID, User | User principal hint |

### Shorthand Format
Some tools support: `@ClusterName/Database`
Example: `@help/Samples` expands to `https://help.kusto.windows.net/Samples;Fed=true`

---

## SDK Feature Comparison

| Feature | .NET | Python | Java | Node.js | Go | R | PowerShell |
|---------|------|--------|------|---------|-----|---|------------|
| Query | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| Management Commands | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| Queued Ingestion | Yes | Yes | Yes | Yes | Yes | No | Yes |
| Streaming Ingestion | Yes | Yes | Yes | Yes | Yes | No | Yes |
| Managed Streaming | Yes | No | No | No | No | No | Yes |
| ARM/Resource Mgmt | Yes | Yes | Yes | Yes | Yes | No | Yes (Az.Kusto) |
| DataFrame Integration | No | Yes | No | No | No | Yes | No |

---

## Version Information

All SDKs follow semantic versioning and are actively maintained. Check the respective package managers for the latest versions:
- .NET: NuGet.org
- Python: PyPI.org
- Java: Maven Central
- Node.js: npmjs.com
- Go: GitHub releases
- R: CRAN

---

## Related Resources

- [REST API Reference](/rest/api/azurerekusto/)
- [Client Libraries Overview](https://learn.microsoft.com/azure/data-explorer/kusto/api/client-libraries)
- [Sample App Generator Wizard](https://learn.microsoft.com/azure/data-explorer/sample-app-generator-wizard)
