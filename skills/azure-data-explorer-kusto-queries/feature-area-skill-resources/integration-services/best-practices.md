# Integration Services Best Practices

## Integration Selection Guide

| Scenario | Recommended Integration | Rationale |
|----------|------------------------|-----------|
| Real-time alerting | Power Automate | Low-code, quick setup, built-in notifications |
| Complex workflows | Logic Apps | Enterprise integration, error handling, retry logic |
| Event-driven processing | Azure Functions | Code flexibility, scalability, event triggers |
| Batch data movement | Azure Data Factory | Large volumes, scheduling, monitoring |
| Interactive analytics | Synapse Spark | Data science workloads, notebook experience |
| Cross-service queries | Plugins (sql_request, etc.) | In-query joins, real-time enrichment |
| Archival data access | External Tables | Query without ingestion cost |
| Monitoring/Operations | Azure Monitor | Native integration, comprehensive dashboards |

---

## Data Flow Patterns

### Pattern 1: Event-Driven Ingestion

**Use Case**: Real-time IoT, logs, streaming sources

```
Events --> Event Hub --> Azure Functions --> ADX (Streaming Ingestion)
```

**Best Practices:**
- Use streaming ingestion for low-latency (<10 seconds)
- Batch small events together when possible
- Implement dead-letter queues for failed ingestions
- Use managed identity for authentication

### Pattern 2: ETL Pipeline

**Use Case**: Regular data loading from external sources

```
Source Data --> Data Factory --> (Transform) --> ADX (Queued Ingestion)
```

**Best Practices:**
- Use queued ingestion for large volumes (optimized throughput)
- Implement incremental loading with watermarks
- Configure appropriate batching policies
- Monitor ingestion latency and success rates

### Pattern 3: Cross-Service Analytics

**Use Case**: Joining ADX data with external sources

```
ADX Query --> sql_request --> SQL Database
         --> cosmosdb_sql_request --> Cosmos DB
         --> cluster() --> Log Analytics / App Insights
```

**Best Practices:**
- Always specify output schema for better performance
- Filter data as early as possible in external queries
- Use materialized views for frequently joined dimension data
- Consider ingesting frequently queried external data

### Pattern 4: Automated Reporting

**Use Case**: Scheduled reports and alerts

```
Timer Trigger --> Power Automate --> KQL Query --> Email/Teams/Slack
```

**Best Practices:**
- Optimize queries for 8-minute timeout limit
- Use async commands for long-running operations
- Cache results when possible (stored query results)
- Implement retry logic for transient failures

### Pattern 5: Data Lake Federation

**Use Case**: Query historical/cold data without ingestion

```
Hot Data (ADX Tables) + Cold Data (External Tables)
         |                           |
         v                           v
    Ingested, indexed          Query on demand
         |                           |
         +----------> union <--------+
```

**Best Practices:**
- Ingest frequently queried data for best performance
- Use external tables only for historical/rarely queried data
- Implement proper partitioning for external tables
- Use columnar formats (Parquet) for external data

---

## Query Plugin Optimization

### sql_request Plugin

```kusto
// GOOD: Specify schema and filter in SQL
evaluate sql_request(
    'Server=tcp:server.database.windows.net,1433;'
    'Authentication="Active Directory Integrated";Initial Catalog=DB;',
    'SELECT Id, Name FROM Table WHERE Active = 1'
) : (Id:long, Name:string)

// BAD: No schema, no filtering
evaluate sql_request('...', 'SELECT * FROM Table')
```

**Guidelines:**
1. Always specify output schema
2. Push filters to SQL query
3. Select only needed columns
4. Use parameterized queries for dynamic values
5. Consider query timeout (~5 minutes default)

### cosmosdb_sql_request Plugin

```kusto
// GOOD: Specify schema and preferred locations
evaluate cosmosdb_sql_request(
    'AccountEndpoint=...;Database=DB;Collection=Col;',
    'SELECT c.Id, c.Name FROM c WHERE c.Active = true',
    dynamic(null),
    dynamic({'preferredLocations': ['East US']})
) : (Id:long, Name:string)
```

**Guidelines:**
1. Use managed identity authentication
2. Specify preferred locations for multi-region
3. Filter in Cosmos DB query, not ADX
4. Limit result sets with TOP clause

---

## External Table Best Practices

### Partitioning Strategy

```kusto
// Good: Partition by date for time-series data
.create external table Logs (Timestamp: datetime, Message: string)
kind=blob
partition by (Date: datetime = bin(Timestamp, 1d))
dataformat=parquet
(h@'https://storage/logs;key')

// Query with partition filter
external_table("Logs")
| where Date >= ago(7d)  // Partition pruning
| where Timestamp > ago(1d)  // Further filtering
```

### File Organization

| Aspect | Recommendation |
|--------|---------------|
| File Size | 100MB - 1GB per file |
| Format | Parquet (preferred) or ORC |
| Compression | Snappy (built-in) or Gzip |
| Partitioning | By date or tenant |
| Location | Same region as ADX cluster |

---

## Power Automate/Logic Apps Optimization

1. **Minimize Query Scope**
```kusto
// GOOD: Filter early, aggregate, limit results
MyTable
| where Timestamp > ago(1h)
| summarize Count=count() by Status
| take 100

// BAD: Fetch all data
MyTable
```

2. **Use Async for Long Operations**
   - Async commands: 60-minute timeout
   - Sync commands: 8-minute timeout
   - Poll `.show operations` for status

3. **Batch Operations**
   - Group multiple related queries
   - Use single flow with multiple actions
   - Aggregate results before sending

---

## Security Best Practices

### Authentication Hierarchy

| Method | Security Level | Use Case |
|--------|---------------|----------|
| Managed Identity | Highest | Azure-hosted services |
| Service Principal | High | Automated processes |
| User Delegation | Medium | Interactive scenarios |
| Connection Strings | Lower | Legacy systems |

### Callout Policy Configuration

```kusto
.alter cluster policy callout @'[
    {"CalloutType": "sql", "CalloutUriRegex": ".*\\.database\\.windows\\.net", "CanCall": true},
    {"CalloutType": "cosmosdb", "CalloutUriRegex": ".*\\.documents\\.azure\\.com", "CanCall": true},
    {"CalloutType": "webapi", "CalloutUriRegex": "https://api\\.example\\.com/.*", "CanCall": true}
]'
```

### Data Protection

1. **Use Obfuscated Strings**: Hide secrets in query logs with `h` prefix
2. **Minimize Data Exposure**: Filter sensitive columns
3. **Audit Integration Access**: Enable diagnostic logs
4. **Rotate Credentials**: Use short-lived tokens

---

## Error Handling

### Retry Strategy by Integration

| Integration | Retry Approach |
|-------------|---------------|
| Power Automate | Built-in retry policies |
| Logic Apps | Configure retry in action settings |
| Functions | Implement exponential backoff |
| Data Factory | Activity retry settings |

### Common Errors and Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| RequestTimeout | Query too slow | Optimize query, use async |
| TooManyRequests | Rate limiting | Implement backoff, reduce frequency |
| Unauthorized | Auth expired/invalid | Refresh tokens, check permissions |
| EntityNotFound | Missing table/database | Verify names, check access |
| QuotaExceeded | Result set too large | Filter data, paginate results |

### Dead Letter Pattern

```
Primary Flow --> Process Data --> Success
     |
     +--> On Failure --> Dead Letter Queue --> Alert + Manual Review
```

---

## Cost Optimization

| Integration | Cost Driver | Optimization |
|-------------|-------------|--------------|
| Power Automate | Actions/runs | Reduce flow frequency |
| Logic Apps | Actions/connectors | Consolidate actions |
| Functions | Executions/duration | Optimize code, batch |
| Data Factory | DIUs/duration | Efficient mappings |
| External Tables | Query scans | Partition effectively |

### Query Cost Reduction

1. Use materialized views for common aggregations
2. Implement caching with stored query results
3. Optimize partitioning for external tables
4. Filter early to reduce data movement

---

## Decision Flowchart

```
Real-time processing required?
  |
  +-- Yes --> High volume? --> Yes --> Azure Functions (Streaming)
  |                       --> No --> Power Automate / Functions
  |
  +-- No --> Batch/scheduled?
              |
              +-- Yes --> Large volume? --> Yes --> Data Factory
              |                         --> No --> Power Automate / Logic Apps
              |
              +-- No --> Cross-service join needed?
                          |
                          +-- Yes --> Query Plugins
                          +-- No --> Direct ADX Query / External Tables
```
