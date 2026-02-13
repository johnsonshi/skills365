# Azure Data Explorer Cluster Management Reference

## Overview

Azure Data Explorer (ADX) cluster management encompasses all operations related to creating, configuring, scaling, monitoring, and maintaining clusters. A cluster is the fundamental compute resource that hosts databases, processes queries, and manages data ingestion.

---

## Cluster Operations

### Create Cluster

Creating a cluster provisions compute and storage resources in a specified Azure region.

**Key Parameters:**
- **Cluster Name**: 4-22 lowercase alphanumeric characters, globally unique
- **Resource Group**: Azure resource group for the cluster
- **Region**: Azure region (affects SKU availability and latency)
- **SKU**: Compute SKU determining CPU, memory, and storage
- **Capacity**: Number of instances (nodes)

**Cluster Types:**
| Type | Description | SLA | Minimum Nodes |
|------|-------------|-----|---------------|
| Production | Full-featured with redundancy | Yes | 2 engine + 2 data management |
| Dev/Test | Single node, evaluation/testing | No | 1 engine + 1 data management |

**Provisioning Time**: Approximately 10 minutes

**Methods:**
- Azure Portal
- Azure CLI: `az kusto cluster create`
- PowerShell: `New-AzKustoCluster`
- ARM/Bicep Templates
- SDKs: C#, Python, Go, Java, Node.js

### Start/Stop Cluster

Clusters can be stopped to save costs while preserving data. Stopped clusters release compute resources but retain storage.

**Start Cluster:**
- Time to become available: ~10 minutes
- Hot cache needs time to reload

**Stop Cluster:**
- Compute costs cease immediately
- Storage costs continue
- No queries or ingestion while stopped

**Behavior Notes:**
- Data remains available after restart
- Queries and ingestion unavailable while stopped

### Delete Cluster

**Soft Delete Period**: 14 days for clusters active more than 14 days
- Cluster is recoverable during soft delete
- Cannot create cluster with same name during soft delete
- Open support ticket to restore

**Opt Out of Soft Delete**: Set tag `opt-out-of-soft-delete = true` for immediate permanent deletion

**Warning**: Deletion is permanent after soft delete period expires

### Auto-Stop Clusters

Clusters automatically stop after 5 days of inactivity (no ingestion or queries).

**Exceptions (clusters that do NOT auto-stop):**
- Leader clusters (with follower databases)
- Virtual Network deployed clusters
- Start-for-free clusters
- Clusters with Auto-Stop disabled

**Configuration:**
- Default: `enableAutoStop = true`
- Can be modified during or after creation
- Settings path: Cluster > Settings > Configurations > Auto-Stop cluster

---

## SKU Selection

### SKU Categories

**Compute Optimized:**
- High core-to-cache ratio
- Lowest cost per core
- Local SSD for low-latency I/O
- Best for high query volumes

| SKU Series | vCPU Options | Type | Premium Storage |
|------------|--------------|------|-----------------|
| Eadsv5, ECadsv5 | 2, 4, 8, 16 | AMD | No |
| Edv4, Edv5 | 2, 4, 8, 16 | Intel | No |
| Eav4 | 2, 4, 8, 16 | AMD | No |
| Dv2 | 2, 4, 8, 16 | Intel | No |

**Storage Optimized:**
- Larger storage (1-4 TB per node)
- Lowest cost per GB
- Best for large data volumes

| SKU Series | vCPU Options | Type | Premium Storage |
|------------|--------------|------|-----------------|
| Lasv3 | 8, 16, 32 | AMD | No |
| Lsv3 | 8, 16, 32 | Intel | No |
| Easv4, Easv5, ECasv5 | 8, 16 | AMD | Yes |
| Esv4, Esv5 | 8, 16 | Intel | Yes |
| DSv2 | 8, 16 | Intel | Yes |

**Dev/Test SKUs:**
- Any compute optimized SKU with 2 cores
- Example: `Dev(No SLA)_Standard_E2a_v4`
- No Azure Data Explorer markup charge
- No SLA

### SKU Selection Guidelines

| Attribute | Compute Optimized | Storage Optimized |
|-----------|-------------------|-------------------|
| Cost per GB | Higher | Lower |
| Cost per core | Lower | Higher |
| Best for | High query rates | Large cached data |
| Local SSD | Yes | Varies |

**Recommendations:**
- Start with smallest SKU meeting requirements
- Scale up (larger SKU) before scaling out (more nodes)
- Use Azure Advisor for SKU optimization suggestions
- Prefer larger VMs with more RAM for join-heavy queries

---

## Scaling

### Horizontal Scaling (Scale Out/In)

Adjusts the number of cluster instances.

**Scaling Methods:**

1. **Manual Scale**
   - Static capacity
   - Set via Instance count slider
   - Remains fixed until manually changed

2. **Optimized Autoscale (Recommended)**
   - Default setting for new clusters
   - Automatically adjusts based on load
   - Configure minimum and maximum instance count
   - Uses predictive and reactive logic

   **Metrics Monitored:**
   - CPU utilization
   - Cache utilization factor
   - Ingestion utilization

   **Scale Out Triggers** (any condition for 1+ hour):
   - High cache utilization
   - High CPU
   - High ingestion utilization

   **Scale In Requirements** (all must be met):
   - Cache utilization not high
   - CPU below average
   - Ingestion utilization below average
   - Keep alive metric healthy
   - No query throttling
   - Failed queries below threshold

3. **Custom Autoscale**
   - Rule-based scaling
   - Configure metrics, thresholds, actions
   - Supports schedule-based rules

### Vertical Scaling (Scale Up/Down)

Changes the cluster SKU to add or remove CPU/memory resources.

**Process:**
- New cluster resources prepared while old cluster serves requests
- Switchover takes 1-3 minutes
- Data preserved during migration
- Query performance may be affected during migration

**VNet Clusters**: May experience longer service disruption

**Configuration Path**: Cluster > Settings > Scale up

---

## Availability Zones

Availability zones provide intra-region redundancy by distributing resources across physically separate data centers.

**Benefits:**
- Protection against single zone failure
- No additional cost for compute
- ZRS storage automatically configured

**Configuration:**
- Enable during cluster creation
- Can migrate existing clusters to availability zones
- Requires minimum 3 zones for full protection

**Storage with Availability Zones:**
- Standard ZRS (Zone Redundant Storage) enabled automatically
- Higher durability than LRS

**Path**: Cluster creation > Availability zones toggle

---

## Free Cluster

A free Azure Data Explorer cluster for evaluation without Azure subscription or credit card.

**Specifications:**
| Item | Limit |
|------|-------|
| Storage (uncompressed) | ~100 GB (20 GB compressed) |
| Databases | Up to 10 |
| Tables per database | Up to 100 |
| Columns per table | Up to 200 |
| Materialized views per database | Up to 5 |

**Trial Period**: 1 year (may be extended)

**Features Available:**
- Full KQL support
- SDKs (all languages)
- Time series and ML functions
- Geospatial functions
- Streaming ingestion
- Event Hub connector
- Dashboards (Power BI, Web UI, Grafana)

**Features NOT Available:**
- External tables
- Continuous export
- Workload groups
- Purge
- Follower clusters
- Partitioning policy
- Python/R plugins
- Autoscale
- Azure Monitor/Insights
- ARM templates
- Enterprise security (CMK, VNet)

**Upgrade Path**: Can upgrade to full cluster at any time

---

## Monitoring Metrics

### Cluster Health Check

**KQL Command:**
```kusto
.show diagnostics
| project IsHealthy, NotHealthyReason, IsAttentionRequired, AttentionRequiredReason, IsScaleOutRequired
```

**Output Parameters:**
| Parameter | Description |
|-----------|-------------|
| IsHealthy | 1 = healthy, 0 = unhealthy |
| NotHealthyReason | Reason for unhealthy state |
| IsAttentionRequired | 1 = needs attention |
| AttentionRequiredReason | Reason attention needed |
| IsScaleOutRequired | 1 = scale out recommended |

### Key Metrics (Azure Monitor)

**Resource Utilization:**
- Keep Alive (cluster health)
- CPU utilization
- Cache utilization
- Memory utilization

**Ingestion Metrics:**
- Ingestion utilization
- Ingestion latency
- Ingestion volume
- Events received/processed

**Query Metrics:**
- Query duration
- Concurrent queries
- Throttled queries

### Azure Data Explorer Insights

Comprehensive monitoring workbook providing:
- At-scale view across subscriptions
- Drill-down analysis per cluster
- Query performance tracking
- Ingestion performance monitoring
- Cache analysis and recommendations
- Cluster boundaries assessment

**Required Diagnostic Logs:**
- Command
- Query
- SucceededIngestion
- FailedIngestion
- IngestionBatching
- TableDetails
- TableUsageStatistics

### Azure Advisor Integration

**Recommendation Types:**
- **Cost**: Unused clusters, SKU optimization, cache reduction, enable autoscale
- **Performance**: SKU changes, cache policy updates
- **Reliability**: Subnet delegation, IP configuration
- **Operational Excellence**: Cache policy alignment with usage

---

## Follower Databases

Attach databases from another cluster in read-only mode for data sharing.

**Use Cases:**
- Share data between teams/organizations
- Segregate production from non-production workloads
- Associate costs with query consumers

**Characteristics:**
- Read-only access
- Synchronization lag: seconds to minutes
- Uses leader cluster's storage
- Cannot modify data, tables, or policies (except caching)

**Limitations:**
- Leader and follower must be in same region
- Cannot delete attached databases directly

---

## Cluster Encryption

**Options:**
- **Disk Encryption**: Default Azure encryption
- **Double Encryption**: Platform + customer encryption
- **Customer-Managed Keys (CMK)**: Azure Key Vault integration

**Managed Identities:**
- System-assigned
- User-assigned
- Used for authentication to dependent services

---

## Document Information

- **Generated**: 2026-02-13
- **Source**: Azure Data Explorer documentation analysis
- **Phase**: 3 - Feature In-Depth Research
