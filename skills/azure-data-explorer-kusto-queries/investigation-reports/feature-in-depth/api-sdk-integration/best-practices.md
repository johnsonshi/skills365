# Azure Data Explorer API and SDK Integration Best Practices

## Overview

This document provides best practices for using Azure Data Explorer APIs and SDKs effectively, covering authentication patterns, connection management, error handling, and retry logic.

---

## Authentication Patterns

### Recommended Authentication Methods by Scenario

| Scenario | Recommended Method | SDK Method |
|----------|-------------------|------------|
| Interactive applications | User prompt authentication | `WithAadUserPromptAuthentication()` |
| Background services/daemons | Application key or certificate | `WithAadApplicationKeyAuthentication()` |
| Azure-hosted workloads | Managed Identity | `WithAadSystemManagedIdentity()` |
| CI/CD pipelines | Service principal with secret | `WithAadApplicationKeyAuthentication()` |
| High-security environments | Certificate-based auth | `WithAadApplicationCertificateAuthentication()` |
| Token already available | Token authentication | `WithAadUserTokenAuthentication()` |

### Authentication Best Practices

1. **Prefer Managed Identities** for Azure-hosted applications
   - Eliminates credential management
   - Automatic token rotation
   - System-assigned for single-purpose apps; user-assigned for shared identity

2. **Use Certificate Authentication** for high-security scenarios
   - Certificates should have at least 2048-bit public keys
   - Store certificates in secure locations (Azure Key Vault, local machine store)
   - Enable X5c for subject name and issuer authentication

3. **Never hardcode credentials** in source code
   - Use environment variables, Azure Key Vault, or configuration files
   - Leverage connection string builders rather than raw strings

4. **Scope permissions appropriately**
   - Grant minimum required permissions (Database User, Database Ingestor, Table Ingestor)
   - Use table-level permissions when database-level is too broad

### Token Management

```csharp
// Use token provider callback for custom token management
var kcsb = new KustoConnectionStringBuilder(clusterUri)
    .WithAadTokenProviderAuthentication(() => GetTokenFromCache());
```

- Cache tokens and refresh before expiration
- Handle token refresh failures gracefully
- Consider using Azure Identity library for automatic token management

---

## Connection Pooling and Client Reuse

### Use Single Client Instances

**Critical Best Practice:** Reuse client instances rather than creating new ones per request.

```csharp
// GOOD: Single instance, reused across requests
private static readonly ICslQueryProvider QueryClient = CreateClient();

// BAD: Creating new client per request
public void Query() {
    using var client = KustoClientFactory.CreateCslQueryProvider(kcsb); // Avoid!
}
```

**Why this matters:**
- Clients are thread-safe and designed for concurrent use
- Clients cache metadata retrieved during initial connection
- Creating multiple clients wastes resources and can overwhelm the service

### Connection Management Rules

1. **One query client per cluster** in your application
2. **One ingest client per target database** (queued ingestion)
3. **Dispose clients properly** when the application shuts down
4. **Dispose result readers** after processing to release network resources

### Thread Safety Considerations

| Component | Thread-Safe | Notes |
|-----------|-------------|-------|
| Query Client | Yes | Reuse across threads |
| Ingest Client | Yes | Reuse across threads |
| Result Reader (IDataReader) | No | Process in single thread, dispose after use |
| Connection String Builder | Yes | Can be shared |
| ClientRequestProperties.Database | No | Set at creation, don't modify |

### Database Parameter Handling

```csharp
// RECOMMENDED: Specify database per query
var response = client.ExecuteQuery("MyDatabase", query, null);

// AVOID: Relying on default database property (not thread-safe to modify)
```

---

## Error Handling

### Exception Hierarchy (.NET)

All SDK exceptions implement `ICloudPlatformException` with these properties:
- `FailureCode` - HTTP status code equivalent
- `FailureSubCode` - HTTP reason phrase equivalent
- `IsPermanent` - Whether retry is unlikely to succeed

### Exception Categories

| Category | Base Class | HTTP Equivalent | Retry? |
|----------|------------|-----------------|--------|
| Client errors | `KustoClientException` | N/A (pre-request) | Depends |
| Request errors | `KustoRequestException` | 4xx | No (permanent) |
| Service errors | `KustoServiceException` | 5xx | Usually yes |

### Common Exceptions and Handling

**Client Exceptions (pre-request failures):**

| Exception | Cause | Action |
|-----------|-------|--------|
| `KustoClientAuthenticationException` | Auth flow failure | Check credentials, refresh tokens |
| `KustoClientNameResolutionFailureException` | DNS failure | Check cluster URI, network |
| `KustoClientTimeoutException` | Client-side timeout | Increase timeout, check network |
| `KustoClientInvalidConnectionStringException` | Bad connection string | Fix connection string parameters |

**Request Exceptions (400-level, permanent):**

| Exception | Cause | Action |
|-----------|-------|--------|
| `DatabaseNotFoundException` | Database doesn't exist | Verify database name |
| `TableNotFoundException` | Table doesn't exist | Verify table name |
| `SyntaxException` | Invalid query syntax | Fix the query |
| `SemanticException` | Semantic error in query | Review query logic |
| `KustoRequestDeniedException` (403) | Insufficient permissions | Check RBAC assignments |
| `MappingNotFoundException` | Missing ingestion mapping | Create the mapping first |

**Service Exceptions (500-level, may retry):**

| Exception | Cause | Action |
|-----------|-------|--------|
| `KustoRequestThrottledException` (429) | Rate limited | Implement backoff, retry |
| `KustoServiceTimeoutException` (504) | Server timeout | Retry with exponential backoff |
| `KustoServicePartialQueryFailureException` | Query started but failed | Check resources, simplify query |

### Exception Handling Pattern

```csharp
try {
    var result = await client.ExecuteQueryAsync(database, query, properties);
}
catch (KustoClientException ex) when (!ex.IsPermanent) {
    // Transient client error - may retry
    await RetryWithBackoffAsync(() => client.ExecuteQueryAsync(database, query, properties));
}
catch (KustoRequestException ex) {
    // Permanent request error - fix the request
    logger.LogError($"Request error (permanent): {ex.FailureSubCode}");
    throw;
}
catch (KustoRequestThrottledException ex) {
    // Throttled - back off and retry
    await Task.Delay(TimeSpan.FromSeconds(ex.RetryAfter ?? 30));
    return await client.ExecuteQueryAsync(database, query, properties);
}
catch (KustoServiceException ex) when (!ex.IsPermanent) {
    // Transient service error - retry
    await RetryWithBackoffAsync(() => client.ExecuteQueryAsync(database, query, properties));
}
```

---

## Retry Logic

### Retry Decision Matrix

| Exception Type | IsPermanent | Should Retry | Strategy |
|----------------|-------------|--------------|----------|
| Authentication failure | True | No | Fix credentials |
| Syntax/Semantic error | True | No | Fix query |
| Permission denied (403) | True | No | Fix permissions |
| Throttled (429) | False | Yes | Exponential backoff |
| Server timeout (504) | False | Yes | Exponential backoff |
| Network failure | False | Yes | Immediate retry, then backoff |
| Partial query failure | Varies | Maybe | Depends on cause |

### Exponential Backoff Implementation

```csharp
public async Task<T> ExecuteWithRetryAsync<T>(
    Func<Task<T>> operation,
    int maxRetries = 3,
    int baseDelayMs = 1000)
{
    int retryCount = 0;
    while (true) {
        try {
            return await operation();
        }
        catch (Exception ex) when (ShouldRetry(ex) && retryCount < maxRetries) {
            var delay = baseDelayMs * Math.Pow(2, retryCount);
            var jitter = Random.Shared.Next(0, (int)(delay * 0.1));
            await Task.Delay((int)delay + jitter);
            retryCount++;
        }
    }
}

private bool ShouldRetry(Exception ex) {
    return ex switch {
        KustoRequestThrottledException => true,
        KustoServiceTimeoutException => true,
        KustoServiceException e when !e.IsPermanent => true,
        KustoClientException e when !e.IsPermanent => true,
        _ => false
    };
}
```

### Retry Configuration for Queued Ingestion

```csharp
var ingestClient = KustoIngestFactory.CreateQueuedIngestClient(kcsb);
// The QueueRetryPolicy property controls retry behavior
ingestClient.QueueRetryPolicy = new CustomRetryPolicy();
```

---

## Ingestion Best Practices

### Prefer Queued Ingestion

| Mode | Use Case | Recommendation |
|------|----------|----------------|
| Queued | Production workloads | Recommended |
| Direct | Development/testing | Not for production |
| Streaming | Low-latency, small chunks | Specialized scenarios |
| Managed Streaming | Streaming with fallback | Best of both worlds |

### Optimize Throughput

1. **Batch data into large chunks** (100 MB - 1 GB uncompressed)
2. **Use efficient formats**: CSV, Parquet, JSON, Avro
3. **Minimize table width**: Only ingest essential columns
4. **Colocate data sources**: Same region as cluster
5. **Avoid peak query times**: Ingestion competes with queries

### Optimize Cost

1. **Limit ingested chunk count**: Fewer large files > many small files
2. **Provide uncompressed data size**: Helps service optimize
3. **Avoid `FlushImmediately`**: Disrupts batching
4. **Minimize extent tags**: `ingest-by`/`drop-by` tags per extent

### Ingestion Status Tracking

```csharp
// For high-volume streams, limit status tracking
var props = new KustoQueuedIngestionProperties(database, table) {
    ReportLevel = IngestionReportLevel.FailuresOnly,  // Not FailuresAndSuccesses
    ReportMethod = IngestionReportMethod.Queue
};
```

**Warning:** Excessive tracking (`FailuresAndSuccesses`) can:
- Increase ingestion latency
- Cause non-responsiveness under load

---

## Query Best Practices

### Use Client Request Properties

```csharp
var crp = new ClientRequestProperties();
crp.ClientRequestId = $"MyApp.Query.{Guid.NewGuid()}";  // For tracing
crp.SetOption(ClientRequestProperties.OptionServerTimeout, "5m");  // Timeout
```

### Parameterized Queries for Security

**Always parameterize user input** to prevent injection:

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

### Resource Governance

- Set appropriate timeouts for different query types
- Use `x-ms-readonly` header for read-only operations
- Monitor and respect workload group limits

---

## Streaming Response Best Practices

When streaming is enabled (default), the SDK doesn't buffer response data:

```csharp
using (var response = client.ExecuteQuery(database, query, null)) {
    while (response.Read()) {
        // Process row
    }
}  // Connection released when disposed
```

**Important:**
- Always dispose `IDataReader` results promptly
- Network connection held open until disposal
- Process results in a single thread
- Consider disabling streaming for small result sets

---

## Performance Optimization Summary

| Area | Best Practice |
|------|---------------|
| Clients | Single instance per cluster/database |
| Authentication | Use managed identity where possible |
| Connections | Dispose results, reuse clients |
| Queries | Parameterize user input, set timeouts |
| Ingestion | Batch large, prefer queued mode |
| Errors | Implement exponential backoff |
| Tracking | Limit to failures for high-volume |

---

## Security Checklist

- [ ] Never hardcode credentials in source code
- [ ] Use managed identities for Azure workloads
- [ ] Implement certificate-based auth for high-security
- [ ] Parameterize all user inputs in queries
- [ ] Grant minimum required permissions
- [ ] Rotate secrets and certificates regularly
- [ ] Monitor failed authentication attempts
- [ ] Use private endpoints for network isolation
