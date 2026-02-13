# Azure Data Explorer Data Ingestion Best Practices

## Overview

This guide provides practical recommendations for optimizing data ingestion into Azure Data Explorer, covering method selection, performance tuning, and error handling.

---

## Choosing the Right Ingestion Method

### Decision Matrix

| Scenario | Recommended Method | Rationale |
|----------|-------------------|-----------|
| Real-time monitoring (<1s latency) | Streaming | Sub-second latency requirement |
| High-volume continuous data | Queued (Event Hubs/Event Grid) | Throughput optimized, automatic batching |
| Historical data backfill | Queued + LightIngest | Bulk loading, timestamp override support |
| Log aggregation | Queued (Logstash/Fluent Bit) | Existing pipeline integration |
| Spark/data lake processing | Spark Connector | Native DataFrame integration |
| One-time file upload | Get Data experience | UI-guided, schema inference |
| Testing/prototyping | Direct (.ingest inline) | Quick validation |

### Streaming vs Queued Decision Guide

**Use Streaming when:**
- Sub-second query latency is critical
- Data volume per table is small (< 4 GB/hour per table)
- Many tables with distributed small streams
- Total concurrent requests within cluster limits

**Use Queued when:**
- Throughput is the priority
- Data volume exceeds 4 GB/hour per table
- Cost optimization is important
- Automatic retry/reliability is needed

---

## Performance Optimization

### Optimize for Throughput

1. **Batch Size**: Send 100 MB to 1 GB uncompressed per batch
   ```
   Ideal: 100 MB - 1 GB uncompressed
   Maximum: 6 GB per queued ingestion command
   ```

2. **Data Format Selection**:
   - **Best**: CSV, TSV, Parquet, JSON
   - **Avoid**: Many small files, uncompressed large files

3. **Table Width**: Only ingest essential columns
   - Wider tables = lower throughput
   - Use mappings to select specific fields

4. **Data Locality**: Keep data in same region as cluster
   - Cross-region reads significantly slower

5. **Compression**: Use gzip for text formats
   ```
   MyData.csv.gz  // Good: compressed CSV
   MyData.json.gz // Good: compressed JSON
   ```

### Batching Policy Tuning

**For Low Latency (20-30 second E2E):**
```kusto
.alter table MyTable policy ingestionbatching
```
'{'
'  "MaximumBatchingTimeSpan": "00:00:20"'
'}'
```

**For High Throughput:**
```kusto
.alter table MyTable policy ingestionbatching
```
'{'
'  "MaximumBatchingTimeSpan": "00:05:00",'
'  "MaximumNumberOfItems": 2000,'
'  "MaximumRawDataSizeMB": 1024'
'}'
```

**Warning**: Very low batching times on low-volume tables cause:
- Increased COGS (cost)
- More small extents requiring merging
- Potentially higher actual latency

### Client Library Best Practices

1. **Use Single Client Instance**
   ```csharp
   // Good: Reuse single instance
   private static readonly IKustoIngestClient _client =
       KustoIngestFactory.CreateQueuedIngestClient(connectionString);

   // Bad: Creating new client per request
   using var client = KustoIngestFactory.CreateQueuedIngestClient(connectionString);
   ```

2. **Prefer Queued over Direct**
   - Queued: Production workloads
   - Direct: Testing only

3. **Limit Operation Tracking**
   - Excessive tracking increases latency
   - Use only when needed for debugging

---

## Cost Optimization

### Reduce Storage Transactions

1. **Batch before sending**: Combine small files
2. **Avoid FlushImmediately**: Let batching work
3. **Minimize extent tags**: Use sparingly
4. **Provide uncompressed size**: Avoid estimation overhead

### Efficient Ingestion Patterns

**Good:**
```csharp
// Large batch, single operation
await client.IngestFromStreamAsync(largeDataStream, properties);
```

**Avoid:**
```csharp
// Many small operations
foreach (var record in records)
{
    await client.IngestFromStreamAsync(ToStream(record), properties);
}
```

### Cost vs Latency Trade-off

| Setting | Lower Cost | Lower Latency |
|---------|------------|---------------|
| Batch time | Longer (5 min) | Shorter (20-30s) |
| Batch size | Larger (1 GB) | Smaller (100 MB) |
| File count | Fewer large files | More acceptable |

---

## Error Handling

### View Ingestion Failures

```kusto
// All recent failures
.show ingestion failures

// Failures for specific table
.show ingestion failures
| where Table == "MyTable"
| where FailedOn > ago(1h)

// Failures by error type
.show ingestion failures
| summarize count() by ErrorCode, FailureKind
```

### Common Error Types

| FailureKind | Description | Action |
|-------------|-------------|--------|
| Permanent | Unrecoverable error | Fix data/schema issue |
| Transient | Temporary failure | Automatic retry applies |

### Common Error Codes

| Error Code | Cause | Resolution |
|------------|-------|------------|
| Stream_ClosingQuoteMissing | Malformed CSV | Validate CSV format |
| Download_Forbidden | Access denied | Check SAS token/permissions |
| BadRequest_EmptyBlob | Empty file | Filter empty files |
| Schema_MappingError | Schema mismatch | Update mapping/schema |
| General_InternalServerError | Service issue | Retry or contact support |

### Handling Update Policy Failures

Update policies can cause ingestion failures:

```kusto
// Check if failure is from update policy
.show ingestion failures
| where OriginatesFromUpdatePolicy == true
```

**Best practices:**
- Test update policy queries independently
- Use transactional: false for non-critical transformations
- Monitor materialized view health

---

## Reliability Patterns

### Retry Configuration

Queued ingestion includes automatic retry:
- Up to 48 hours retry window
- Exponential backoff
- "At least once" semantics

### Data Validation

```kusto
// Ingestion with validation
.ingest into table MyTable ('https://...')
    with (
        format='csv',
        validationPolicy='{"ValidationOptions":1,"ValidationImplications":1}'
    )
```

Validation options:
- 0: No validation
- 1: Validate CSV column count

### Deduplication

Use `ingest-by` tags to prevent duplicate ingestion:

```kusto
// Ingest only if tag doesn't exist
.ingest into table MyTable ('https://...')
    with (
        tags='["ingest-by:batch-2024-01-15"]',
        ingestIfNotExists='["batch-2024-01-15"]'
    )
```

---

## Monitoring and Diagnostics

### Key Metrics to Monitor

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| Ingestion latency | Time from request to queryable | > batching policy time |
| Ingestion volume | Data size ingested | Anomaly detection |
| Ingestion success rate | % successful ingestions | < 99% |
| Batch duration | Time to seal batches | > expected |

### Diagnostic Queries

```kusto
// Ingestion volume by table (last 24h)
.show commands
| where StartedOn > ago(24h)
| where CommandType == "DataIngestPull"
| summarize TotalSize=sum(ResourcesUtilization.IngestionVolume) by TableName

// Failed ingestions summary
.show ingestion failures
| where FailedOn > ago(1h)
| summarize FailureCount=count() by Table, ErrorCode
| order by FailureCount desc
```

---

## Connector-Specific Recommendations

### Event Hubs

- Use dedicated consumer group per ADX connection
- Set partition count for long-term scale (Premium/Dedicated)
- Enable managed identity authentication
- Same region as cluster for best performance

### Event Grid

- Use BlockBlob, not AppendBlob
- Avoid CopyBlob with hierarchical namespace
- Set blob metadata before upload, not after
- Configure lifecycle management for cleanup

### Kafka

- Configure flush.size.bytes ~100 KB or higher
- Use managed identity when possible
- Match topic mapping to table schema

---

## Security Best Practices

1. **Use Managed Identity** over SAS tokens or keys
2. **Principle of least privilege**: Table Ingestor role is sufficient
3. **Network isolation**: Use Private Endpoints
4. **Rotate credentials** regularly if using service principals
5. **Audit access**: Monitor who ingests data

---

## Summary Checklist

- [ ] Choose appropriate ingestion method for use case
- [ ] Configure batching policy based on latency/throughput needs
- [ ] Use managed identity for authentication
- [ ] Batch data before sending (100 MB - 1 GB)
- [ ] Monitor ingestion failures regularly
- [ ] Set up alerts for anomalies
- [ ] Use deduplication for exactly-once scenarios
- [ ] Test update policies thoroughly
- [ ] Validate data format before ingestion

---

## Document Information

- **Generated**: 2026-02-13
- **Source**: Azure Data Explorer documentation analysis
- **Phase**: 3 - Feature Deep Dive (Data Ingestion)
