# Azure Data Explorer Business Continuity Reference

## Overview

Business continuity and disaster recovery (BCDR) in Azure Data Explorer enables continuous operation during disruptions. This document covers high availability configurations, disaster recovery strategies, cross-region replication, backup/restore mechanisms, leader-follower clusters, and data sharing capabilities.

---

## High Availability Architecture

### Persistence Layer

Azure Data Explorer uses Azure Storage as its durable persistence layer with automatic fault tolerance.

| Storage Type | Description | Use Case |
|--------------|-------------|----------|
| **LRS (Locally Redundant Storage)** | 3 replicas within a single datacenter | Default, cost-effective |
| **ZRS (Zone Redundant Storage)** | Replicas across availability zones | High availability, automatic with AZ deployment |

**Key Points:**
- Storage automatically provides fault tolerance
- If a replica is lost, another is deployed without disruption
- ZRS is automatically configured when cluster deploys to availability zones

### Compute Layer

Azure Data Explorer is a distributed computing platform with configurable node distribution.

**Availability Zone Configuration:**
- Distribute nodes across zones for maximum intra-region resiliency
- Zone failure causes performance degradation, not complete outage
- Recovery automatic when zone recovers

**Enable Availability Zones at Cluster Creation:**
```powershell
# PowerShell example
New-AzKustoCluster -ResourceGroupName "myRG" `
    -Name "mycluster" `
    -Location "North Europe" `
    -SkuName "Standard_E16ads_v5" `
    -SkuTier "Standard" `
    -Zone "1", "2", "3"
```

**Migrate Existing Cluster to Availability Zones:**
```powershell
Update-AzKustoCluster -SubscriptionId "{subscriptionId}" `
    -ResourceGroupName "{resourceGroupName}" `
    -Name "{clusterName}" `
    -Zone "1", "2", "3"
```

---

## Leader-Follower Clusters

The follower database feature allows attaching a database from a leader cluster to a follower cluster in read-only mode.

### Capabilities

| Feature | Description |
|---------|-------------|
| **Read-only access** | Query data without ingestion overhead |
| **Automatic sync** | Changes in leader (create, append, drop) sync to follower |
| **Same storage** | Follower views data without separate ingestion |
| **Data lag** | Few seconds to minutes depending on metadata size |
| **Cost segregation** | Associate compute costs with query consumers |

### Follower Configuration Options

A cluster can:
- Follow one database, several databases, or all databases from a leader
- Follow databases from multiple leader clusters
- Contain both follower and leader databases

### Table-Level Sharing

Control which tables are shared using `TableLevelSharingProperties`:

| Array | Purpose |
|-------|---------|
| `tablesToInclude` | Tables to include (supports wildcards) |
| `tablesToExclude` | Tables to exclude |
| `externalTablesToInclude` | External tables to include |
| `externalTablesToExclude` | External tables to exclude |
| `materializedViewsToInclude` | Materialized views to include |
| `materializedViewsToExclude` | Materialized views to exclude |
| `functionsToInclude` | Functions to include |
| `functionsToExclude` | Functions to exclude |

**Maximum:** 100 entries total across all arrays

### Principal Management

| Modification Kind | Behavior |
|-------------------|----------|
| **Union** (default) | Combine leader principals with follower overrides |
| **Replace** | Use only follower-defined principals |
| **None** | Use only leader principals |

### Caching Policy Override

| Modification Kind | Behavior |
|-------------------|----------|
| **Union** (default) | Combine leader and follower caching policies |
| **Replace** | Use only follower caching policies |
| **None** | Use leader caching policies only |

### Limitations

- Leader and follower must be in the same region
- Streaming ingestion requires follower cluster to enable it
- Cannot delete attached databases without detaching first
- Table-level sharing not supported when following all databases (`*`)
- Cross-tenant follower requires managed identity on follower cluster

---

## Cross-Cluster Queries

Query data across clusters and databases without data movement.

### Syntax

**Cross-database (same cluster):**
```kusto
database("DatabaseName").TableName
```

**Cross-cluster:**
```kusto
cluster("ClusterName").database("DatabaseName").TableName
```

### Wildcards with Union

```kusto
union withsource=TableName *,
      database("OtherDb*").*Table,
      cluster("OtherCluster").database("*").*
```

### Schema Caching

Remote schema is cached after first query. Clear cache after schema changes:
```kusto
.clear cache remote-schema cluster("ClusterName").database("DatabaseName")
```

### Cross-Cluster Function Limitations

- Remote functions must return tabular schema
- Scalar functions only accessible in same cluster
- Remote functions accept only scalar arguments
- Result schema must be fixed (known before execution)

---

## Data Sharing with Azure Data Share

Azure Data Share provides managed data sharing across organizations using the follower database mechanism.

### Flow

1. **Provider** creates share, adds database/tables, sends invitation
2. **Receiver** accepts invitation via email
3. **Receiver** maps shared database to their cluster
4. Data syncs automatically (few seconds to minutes lag)

### Sharing Options

| Level | Description |
|-------|-------------|
| **Full database** | Share entire database (default) |
| **Table-level** | Share specific tables using ARM template |

### Cross-Tenant Sharing

Azure Data Share enables sharing across different Microsoft Entra tenants.

---

## Continuous Data Export

Export data continuously to external storage for backup and data lake scenarios.

### Configuration

```kusto
.create-or-alter continuous-export MyExport
over (FactTable)
to table ExternalBlob
with (
    intervalBetweenRuns=1h,
    forcedLatency=10m,
    sizeLimit=104857600
)
<| FactTable | where Timestamp > ago(30d)
```

### Properties

| Property | Description | Default |
|----------|-------------|---------|
| `intervalBetweenRuns` | Time between executions | Required (min: 1 minute) |
| `forcedLatency` | Delay to ensure all records ingested | None |
| `sizeLimit` | Max artifact size before compression | 100 MB |
| `distributed` | Enable distributed export | true |
| `isDisabled` | Disable export | false |

### Exactly-Once Guarantee

- Uses database cursors to track exported data
- Requires `IngestionTime` policy enabled on source tables
- Guarantee applies to files in `.show exported artifacts` output

### Monitoring

```kusto
// Show continuous export status
.show continuous-export MyExport

// Show export failures
.show continuous-export MyExport failures

// Show exported artifacts
.show continuous-export MyExport exported-artifacts
```

---

## Backup and Restore Mechanisms

### Database Schema Backup

Export schema as CSL script:
```kusto
.show database schema as csl script with (ShowObfuscatedStrings = true)
```

### Schema Restore

Execute schema script on target database:
```kusto
.execute database script <|
// Paste CSL script here
```

### Data Recovery from Continuous Export

Historical data in external storage can be re-ingested:
```kusto
// Get cursor from continuous export
.show continuous-export MyExport | project StartCursor

// Export historical data before cursor
.export async to table ExternalBlob
<| T | where cursor_before_or_at("636751928823156645")
```

### Table Recovery

Recover accidentally dropped tables (if recoverability enabled):
```kusto
.undo drop table TableName
```

**Prerequisites:**
- Enable `recoverability` in retention policy
- Execute within soft-delete period

---

## Resource Protection

### Delete Locks

Prevent accidental cluster or database deletion:
- Apply Azure Resource Manager delete locks
- Available at resource level via Azure portal or CLI

```azurecli
az lock create --name PreventDelete \
    --resource-group myRG \
    --resource-name mycluster \
    --resource-type Microsoft.Kusto/clusters \
    --lock-type CanNotDelete
```

### Soft Delete for External Storage

Enable Azure Storage soft delete for external tables:
- Protects against accidental blob deletion
- Configurable retention period

---

## Disaster Recovery Configurations Summary

| Configuration | RPO | RTO | Effort | Cost |
|---------------|-----|-----|--------|------|
| **Active-Active-Active** | 0 hours | 0 hours | Lower | Highest |
| **Active-Active** | 0 hours | 0 hours | Lower | High |
| **Active-Hot Standby** | 0 hours | Low | Medium | Medium |
| **On-Demand Recovery** | Highest | Highest | Highest | Lowest |

---

## Document Information

- **Generated**: 2026-02-13
- **Source**: Azure Data Explorer documentation analysis
- **Phase**: 3 - Feature Deep Dive (Business Continuity)
