# Integration Services Best Practices

## Overview

This document provides best practices for integrating Azure Data Explorer with other Azure services and external systems. It covers when to use each integration method, data flow patterns, performance optimization, and architectural guidance.

---

## Integration Selection Guide

### Decision Matrix

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

### Integration Complexity vs. Flexibility

```
High Flexibility
     ^
     |  Azure Functions    Custom SDK Apps
     |       *                  *
     |                Logic Apps
     |                    *
     |           Data Factory
     |               *
     |    Power Automate
     |        *
     |________________________> Low Code
Low Flexibility
```

---

## Data Flow Patterns

### Pattern 1: Event-Driven Ingestion

**Use Case**: Real-time data ingestion from IoT, logs, or streaming sources

```
Events --> Event Hub --> Azure Functions --> ADX (Streaming Ingestion)
                              |
                              v
                         (Transform/Validate)
```

**Best Practices:**
- Use streaming ingestion for low-latency requirements (<10 seconds)
- Batch small events together when possible
- Implement dead-letter queues for failed ingestions
- Use managed identity for authentication

### Pattern 2: ETL Pipeline

**Use Case**: Regular data loading from external sources

```
Source Data --> Data Factory --> (Transform) --> ADX (Queued Ingestion)
                    |
                    v
               (Staging/Validation)
```

**Best Practices:**
- Use queued ingestion for large volumes (optimized for throughput)
- Implement incremental loading with watermarks
- Configure appropriate batching policies
- Monitor ingestion latency and success rates

### Pattern 3: Cross-Service Analytics

**Use Case**: Joining ADX data with external data sources

```
ADX Query --> sql_request plugin --> SQL Database
         --> cosmosdb_sql_request plugin --> Cosmos DB
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
            or
Timer Trigger --> Logic Apps --> Multiple Queries --> Aggregated Report
```

**Best Practices:**
- Optimize queries for 8-minute timeout limit
- Use async commands for long-running operations
- Cache results when possible (stored query results)
- Implement retry logic for transient failures

### Pattern 5: Data Lake Query Federation

**Use Case**: Query historical/cold data without ingestion

```
Hot Data (ADX Tables) + Cold Data (External Tables on ADLS)
         |                           |
         v                           v
    Ingested, indexed          Query on demand
         |                           |
         +----------> union <--------+
                        |
                        v
                  Combined Results
```

**Best Practices:**
- Ingest frequently queried data for best performance
- Use external tables only for historical/rarely queried data
- Implement proper partitioning for external tables
- Use columnar formats (Parquet) for external data

---

## When to Use Each Integration

### Power Automate

**Best For:**
- Simple, recurring automation tasks
- Alert notifications
- Scheduled reports
- Non-technical users

**Avoid When:**
- Complex error handling required
- High-volume data processing
- Sub-second latency needed
- Custom authentication scenarios

**Key Considerations:**
```
Pros:
+ No code required
+ 400+ connectors available
+ Built-in retry logic
+ Quick deployment

Cons:
- 8-minute query timeout
- 50,000 record limit per request
- Limited customization
- Higher cost at scale
```

### Logic Apps

**Best For:**
- Enterprise integration scenarios
- Complex workflow orchestration
- Long-running processes
- B2B integrations

**Avoid When:**
- Simple notifications suffice
- Cost is primary concern
- Real-time processing required

**Key Considerations:**
```
Pros:
+ Visual workflow designer
+ Enterprise connectors
+ Robust error handling
+ State management

Cons:
- Higher complexity than Power Automate
- Learning curve
- Consumption-based pricing
```

### Azure Functions

**Best For:**
- Event-driven processing
- Custom business logic
- High-volume scenarios
- Code-first teams

**Avoid When:**
- No development resources available
- Visual workflow preferred
- Simple automation tasks

**Key Considerations:**
```
Pros:
+ Full code control
+ Scale to millions of events
+ Multiple language support
+ Pay-per-execution

Cons:
- Requires development skills
- Cold start latency
- Monitoring complexity
```

### Azure Data Factory

**Best For:**
- Large-scale data movement
- Complex ETL pipelines
- Scheduled batch processing
- Data lake integration

**Avoid When:**
- Real-time processing required
- Simple queries suffice
- Small data volumes

**Key Considerations:**
```
Pros:
+ Visual pipeline designer
+ 90+ connectors
+ Built-in monitoring
+ Mapping data flows

Cons:
- Not for real-time
- Learning curve
- Cost for large pipelines
```

### Synapse/Spark Integration

**Best For:**
- Data science workloads
- Complex transformations
- Large-scale analytics
- ML model training

**Avoid When:**
- Simple queries suffice
- Real-time requirements
- Cost constraints

**Key Considerations:**
```
Pros:
+ Notebook experience
+ Advanced ML libraries
+ Distributed processing
+ Delta Lake support

Cons:
- Higher cost
- Cluster startup time
- Complexity overhead
```

---

## Performance Optimization

### Query Plugin Best Practices

#### sql_request Plugin

```kusto
// GOOD: Specify schema and filter in SQL
evaluate sql_request(
    'Server=tcp:server.database.windows.net,1433;Authentication="Active Directory Integrated";Initial Catalog=DB;',
    'SELECT Id, Name FROM Table WHERE Active = 1'
) : (Id:long, Name:string)

// BAD: No schema, no filtering
evaluate sql_request(
    '...',
    'SELECT * FROM Table'
)
```

**Best Practices:**
1. Always specify output schema
2. Push filters to SQL query
3. Select only needed columns
4. Use parameterized queries for dynamic values
5. Consider query timeout (default: ~5 minutes)

#### cosmosdb_sql_request Plugin

```kusto
// GOOD: Specify schema and use preferred locations
evaluate cosmosdb_sql_request(
    'AccountEndpoint=...;Database=DB;Collection=Col;',
    'SELECT c.Id, c.Name FROM c WHERE c.Active = true',
    dynamic(null),
    dynamic({'preferredLocations': ['East US']})
) : (Id:long, Name:string)
```

**Best Practices:**
1. Use managed identity authentication
2. Specify preferred locations for multi-region
3. Filter in Cosmos DB query, not ADX
4. Use explicit schema definitions
5. Limit result sets with TOP clause

### External Table Best Practices

#### Partitioning Strategy

```kusto
// Good: Partition by date for time-series data
.create external table Logs (
    Timestamp: datetime,
    Message: string
)
kind=blob
partition by (Date: datetime = bin(Timestamp, 1d))
dataformat=parquet
(h@'https://storage/logs;key')

// Query with partition filter
external_table("Logs")
| where Date >= ago(7d)  // Partition pruning
| where Timestamp > ago(1d)  // Further filtering
```

#### File Organization

| Aspect | Recommendation |
|--------|---------------|
| File Size | 100MB - 1GB per file |
| Format | Parquet (preferred) or ORC |
| Compression | Snappy (built-in) or Gzip |
| Partitioning | By date or tenant |
| Location | Same region as ADX cluster |

### Power Automate/Logic Apps Optimization

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

### Managed Identity Configuration

```kusto
// Grant ADX permissions to managed identity
.add database MyDB viewers ('aadapp=<managed-identity-object-id>')
```

### Callout Policy for Plugins

```kusto
// Configure allowed external endpoints
.alter cluster policy callout @'[
    {"CalloutType": "sql", "CalloutUriRegex": ".*\\.database\\.windows\\.net", "CanCall": true},
    {"CalloutType": "cosmosdb", "CalloutUriRegex": ".*\\.documents\\.azure\\.com", "CanCall": true},
    {"CalloutType": "webapi", "CalloutUriRegex": "https://api\\.example\\.com/.*", "CanCall": true}
]'
```

### Data Protection

1. **Use Obfuscated Strings**: Hide secrets in query logs
```kusto
evaluate sql_request(
    h'Server=tcp:server.database.windows.net,1433;User ID=...',  // 'h' prefix
    '...'
)
```

2. **Minimize Data Exposure**: Filter sensitive columns
3. **Audit Integration Access**: Enable diagnostic logs
4. **Rotate Credentials**: Use short-lived tokens

---

## Error Handling Patterns

### Retry Strategy

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

## Monitoring and Observability

### Key Metrics to Track

| Integration | Metrics |
|-------------|---------|
| Power Automate | Run success rate, duration, failures |
| Logic Apps | Execution history, action latency |
| Functions | Invocations, duration, errors |
| Data Factory | Pipeline runs, activity duration |
| ADX Queries | Duration, resource consumption |

### Azure Monitor Integration

Enable comprehensive monitoring:
1. Configure diagnostic logs for ADX
2. Use Azure Monitor Insights dashboard
3. Set up alerts for failures
4. Track query performance over time

### Health Check Queries

```kusto
// Check recent ingestion health
.show ingestion failures
| where FailedOn > ago(1h)

// Monitor query performance
.show queries
| where StartedOn > ago(1h)
| summarize
    AvgDuration=avg(Duration),
    MaxDuration=max(Duration),
    Count=count()
  by User
```

---

## Cost Optimization

### Integration Cost Considerations

| Integration | Cost Driver | Optimization |
|-------------|-------------|--------------|
| Power Automate | Actions/runs | Reduce flow frequency |
| Logic Apps | Actions/connectors | Consolidate actions |
| Functions | Executions/duration | Optimize code, batch |
| Data Factory | DIUs/duration | Efficient mappings |
| External Tables | Query scans | Partition effectively |

### Query Cost Reduction

1. **Use Materialized Views** for common aggregations
2. **Implement Caching** with stored query results
3. **Optimize Partitioning** for external tables
4. **Filter Early** to reduce data movement

---

## Summary: Decision Flowchart

```
Start
  |
  v
Is real-time processing required?
  |
  +-- Yes --> High volume? --> Yes --> Azure Functions with Streaming
  |                      --> No --> Power Automate (simple) / Functions (complex)
  |
  +-- No --> Is it batch/scheduled?
              |
              +-- Yes --> Large data volume? --> Yes --> Data Factory
              |                              --> No --> Power Automate / Logic Apps
              |
              +-- No --> Is it interactive/ad-hoc?
                          |
                          +-- Yes --> Cross-service join? --> Yes --> Query Plugins
                          |                               --> No --> Direct ADX Query
                          |
                          +-- No --> External Tables for archival
```

---

*Document generated from Azure Data Explorer documentation analysis*
*Phase: 3 - Feature In-Depth Research*
