# Azure Data Explorer API and SDK Reference

## REST API

### Endpoints

| Endpoint | URI Pattern | Purpose |
|----------|-------------|---------|
| Engine (Query) | `https://<cluster>.<region>.kusto.windows.net` | Query and management |
| Data Management | `https://ingest-<cluster>.<region>.kusto.windows.net` | Ingestion |

### API Routes

| Action | Method | Path |
|--------|--------|------|
| Query (v1) | POST | `/v1/rest/query` |
| Query (v2) | POST | `/v2/rest/query` |
| Management | POST | `/v1/rest/mgmt` |
| Streaming Ingest | POST | `/v1/rest/ingest` |

### Authentication

All requests require Microsoft Entra ID bearer token:
```
Authorization: Bearer <access_token>
```

Token resource: `https://api.kusto.windows.net`

### Required Headers

| Header | Value |
|--------|-------|
| `Authorization` | `Bearer <token>` |
| `Content-Type` | `application/json; charset=utf-8` |
| `Accept` | `application/json` |
| `x-ms-client-request-id` | Unique ID for tracing (recommended) |

### Request Body

```json
{
  "db": "<database>",
  "csl": "<query_or_command>",
  "properties": {
    "Options": { "servertimeout": "00:05:00" },
    "Parameters": { "param1": "value1" }
  }
}
```

---

## .NET SDK

### Packages

| Package | Purpose |
|---------|---------|
| `Microsoft.Azure.Kusto.Data` | Query execution, management commands |
| `Microsoft.Azure.Kusto.Ingest` | Data ingestion |
| `Microsoft.Azure.Kusto.Language` | Query parsing, semantic analysis |
| `Azure.ResourceManager.Kusto` | ARM resource management |

### Key Classes

**Query:**
- `KustoConnectionStringBuilder` - Connection with auth
- `KustoClientFactory` - Factory for providers
- `ICslQueryProvider` - Query execution
- `ICslAdminProvider` - Management commands
- `ClientRequestProperties` - Request options

**Ingestion:**
- `KustoIngestFactory` - Factory for ingest clients
- `IKustoIngestClient` - Base ingestion interface
- `KustoIngestionProperties` - Ingestion config

### Ingestion Client Types

| Client | Factory Method | Use Case |
|--------|----------------|----------|
| Queued | `CreateQueuedIngestClient()` | Production workloads |
| Direct | `CreateDirectIngestClient()` | Dev/test only |
| Streaming | `CreateStreamingIngestClient()` | Low-latency, small data |
| Managed Streaming | `CreateManagedStreamingIngestClient()` | Streaming with fallback |

---

## Python SDK

### Packages

| Package | Purpose |
|---------|---------|
| `azure-kusto-data` | Query execution |
| `azure-kusto-ingest` | Data ingestion |
| `azure-mgmt-kusto` | Resource management |

### Key Classes

- `KustoClient` - Query and commands
- `KustoConnectionStringBuilder` - Connection builder
- `ClientRequestProperties` - Request options
- `QueuedIngestClient` - Queued ingestion
- `IngestionProperties` - Ingestion config
- `DataFormat` - Format enum (CSV, JSON, Parquet)

### Features

- Python 2.x and 3.x compatible
- Pandas DataFrame integration
- Python DB API interface support

---

## Java SDK

### Packages (Maven)

| Package | Purpose |
|---------|---------|
| `com.microsoft.azure.kusto:kusto-data` | Query execution |
| `com.microsoft.azure.kusto:kusto-ingest` | Data ingestion |

### Key Classes

**Query:**
- `Client` - Main query client
- `ClientFactory` - Client factory
- `ConnectionStringBuilder` - Connection builder
- `KustoOperationResult` - Result container
- `KustoResultSetTable` - Result iterator

**Ingestion:**
- `QueuedIngestClient` - Queued ingestion
- `IngestClientFactory` - Ingest factory
- `IngestionProperties` - Ingestion config
- `FileSourceInfo` / `BlobSourceInfo` / `StreamSourceInfo` - Data sources

---

## Node.js SDK

### Packages (npm)

| Package | Purpose |
|---------|---------|
| `azure-kusto-data` | Query execution |
| `azure-kusto-ingest` | Data ingestion |
| `azure-arm-kusto` | Resource management |

### Key Classes

- `Client` (KustoClient) - Query and management
- `KustoConnectionStringBuilder` - Connection builder
- `IngestClient` - Ingestion client
- `IngestionProperties` - Ingestion config

### Requirements

- Node.js LTS v6.14+
- ES6 / TypeScript support

---

## Go SDK

### Package

`github.com/Azure/azure-kusto-go/kusto`

### Installation

```bash
go get github.com/Azure/azure-kusto-go/kusto
```

### Key Components

- `kusto.Client` - Query execution with iterators
- `azkustoingest` - Ingestion package
- `kusto.NewConnectionStringBuilder()` - Connection builder

### Requirements

- Go 1.13+

---

## PowerShell

### Setup

1. Download `Microsoft.Azure.Kusto.Tools` from NuGet
2. Extract and load DLL:
   - PowerShell 5.1: Use `net472` folder
   - PowerShell 7+: Use other folders
3. Load assembly: `[System.Reflection.Assembly]::LoadFrom("<path>\Kusto.Data.dll")`

### Key Objects

- `Kusto.Data.KustoConnectionStringBuilder`
- `Kusto.Data.Net.Client.KustoClientFactory`
- `Kusto.Data.Common.ClientRequestProperties`

---

## Connection String Format

ADO.NET style with semicolon-delimited pairs:

```
Data Source=https://cluster.region.kusto.windows.net;Initial Catalog=database;Fed=true
```

### Key Properties

| Property | Aliases | Description |
|----------|---------|-------------|
| Data Source | Addr, Server | Cluster URI |
| Initial Catalog | Database | Default database |
| Fed | AADFed | Enable Entra auth |
| Authority ID | TenantId | Azure AD tenant |
| Application Client ID | AppClientId | App registration ID |
| Application Key | AppKey | App secret |

### Shorthand

`@ClusterName/Database` expands to full URI with Fed=true

---

## SDK Feature Matrix

| Feature | .NET | Python | Java | Node.js | Go |
|---------|------|--------|------|---------|-----|
| Query | Yes | Yes | Yes | Yes | Yes |
| Management Commands | Yes | Yes | Yes | Yes | Yes |
| Queued Ingestion | Yes | Yes | Yes | Yes | Yes |
| Streaming Ingestion | Yes | Yes | Yes | Yes | Yes |
| Managed Streaming | Yes | No | No | No | No |
| DataFrame Integration | No | Yes | No | No | No |
| ARM/Resource Mgmt | Yes | Yes | Yes | Yes | Yes |
