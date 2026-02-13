# Azure Data Explorer API and SDK Best Practices

## Authentication

### Recommended Methods by Scenario

| Scenario | Method | SDK Call |
|----------|--------|----------|
| Azure-hosted workloads | Managed Identity | `WithAadSystemManagedIdentity()` |
| Background services | App key/certificate | `WithAadApplicationKeyAuthentication()` |
| Interactive apps | User prompt | `WithAadUserPromptAuthentication()` |
| CI/CD pipelines | Service principal | `WithAadApplicationKeyAuthentication()` |
| High-security | Certificate-based | `WithAadApplicationCertificateAuthentication()` |

### Key Principles

1. **Prefer Managed Identities** for Azure workloads - eliminates credential management
2. **Never hardcode credentials** - use Key Vault or environment variables
3. **Grant minimum permissions** - use table-level access when possible
4. **Cache tokens** and refresh before expiration
5. **Use certificate auth** for high-security scenarios (2048-bit minimum)

---

## Connection Management

### Client Reuse (Critical)

**Always reuse client instances** - never create per-request:

```csharp
// CORRECT: Single instance, reused
private static readonly ICslQueryProvider QueryClient = CreateClient();

// WRONG: New client per request
public void Query() {
    using var client = KustoClientFactory.CreateCslQueryProvider(kcsb); // Avoid!
}
```

### Rules

- **One query client per cluster** in your application
- **One ingest client per target database**
- **Dispose clients** at application shutdown
- **Dispose result readers** after processing

### Thread Safety

| Component | Thread-Safe |
|-----------|-------------|
| Query Client | Yes - reuse across threads |
| Ingest Client | Yes - reuse across threads |
| Result Reader (IDataReader) | No - single thread, dispose after |
| Connection String Builder | Yes |

### Database Parameter

Specify database per query (thread-safe):
```csharp
var response = client.ExecuteQuery("MyDatabase", query, null);
```

---

## Error Handling

### Exception Categories

| Category | HTTP Equivalent | Retry? |
|----------|-----------------|--------|
| Client errors | Pre-request | Depends on `IsPermanent` |
| Request errors (4xx) | 400-499 | No (permanent) |
| Service errors (5xx) | 500-599 | Usually yes |

### Common Exceptions

**Permanent (do not retry):**
- `SyntaxException` - Fix query syntax
- `SemanticException` - Fix query logic
- `DatabaseNotFoundException` - Verify database name
- `KustoRequestDeniedException` (403) - Fix permissions
- `MappingNotFoundException` - Create mapping first

**Transient (retry with backoff):**
- `KustoRequestThrottledException` (429) - Rate limited
- `KustoServiceTimeoutException` (504) - Server timeout
- Network failures

### Retry Pattern

```csharp
try {
    return await client.ExecuteQueryAsync(database, query, properties);
}
catch (KustoRequestThrottledException ex) {
    await Task.Delay(TimeSpan.FromSeconds(ex.RetryAfter ?? 30));
    return await client.ExecuteQueryAsync(database, query, properties);
}
catch (KustoServiceException ex) when (!ex.IsPermanent) {
    // Exponential backoff retry
}
catch (KustoRequestException ex) {
    // Permanent - fix the request, don't retry
    throw;
}
```

### Exponential Backoff

```csharp
var delay = baseDelayMs * Math.Pow(2, retryCount);
var jitter = Random.Shared.Next(0, (int)(delay * 0.1));
await Task.Delay((int)delay + jitter);
```

---

## Ingestion Best Practices

### Choose the Right Mode

| Mode | Use Case |
|------|----------|
| **Queued** | Production workloads (recommended) |
| Direct | Development/testing only |
| Streaming | Low-latency, small chunks (<4MB) |
| Managed Streaming | Streaming with queued fallback |

### Optimize Throughput

1. **Batch data** into large chunks (100 MB - 1 GB uncompressed)
2. **Use efficient formats**: CSV, Parquet, JSON, Avro
3. **Minimize table width**: Only ingest essential columns
4. **Colocate data sources**: Same region as cluster

### Optimize Cost

1. **Fewer large files** > many small files
2. **Provide uncompressed data size** estimate
3. **Avoid `FlushImmediately`**: Disrupts batching
4. **Minimize extent tags**: Reduces overhead

### Status Tracking

For high-volume, use `FailuresOnly`:

```csharp
var props = new KustoQueuedIngestionProperties(database, table) {
    ReportLevel = IngestionReportLevel.FailuresOnly,  // Not FailuresAndSuccesses
    ReportMethod = IngestionReportMethod.Queue
};
```

---

## Query Best Practices

### Use Client Request Properties

```csharp
var crp = new ClientRequestProperties();
crp.ClientRequestId = $"MyApp;{Guid.NewGuid()}";  // For tracing
crp.SetOption(ClientRequestProperties.OptionServerTimeout, TimeSpan.FromMinutes(5));
```

### Parameterize User Input (Security)

Always use query parameters to prevent injection:

```csharp
var query = @"
    declare query_parameters(state:string, minDamage:long);
    StormEvents
    | where State == state
    | where DamageProperty > minDamage";

var crp = new ClientRequestProperties();
crp.SetParameter("state", userInput);
crp.SetParameter("minDamage", "1000000");
```

### Streaming Response Handling

```csharp
using (var response = client.ExecuteQuery(database, query, null)) {
    while (response.Read()) {
        // Process row
    }
}  // Connection released when disposed
```

- **Dispose IDataReader** promptly
- Network connection held until disposal
- Process in single thread

---

## Performance Summary

| Area | Best Practice |
|------|---------------|
| Clients | Single instance per cluster/database |
| Authentication | Managed identity when possible |
| Connections | Dispose results, reuse clients |
| Queries | Parameterize input, set timeouts |
| Ingestion | Batch large, prefer queued mode |
| Errors | Exponential backoff for transient |
| Status Tracking | Failures only for high volume |

---

## Security Checklist

- [ ] No hardcoded credentials in source
- [ ] Managed identities for Azure workloads
- [ ] Certificate auth for high-security
- [ ] All user inputs parameterized
- [ ] Minimum required permissions granted
- [ ] Secrets rotated regularly
- [ ] Private endpoints for network isolation
