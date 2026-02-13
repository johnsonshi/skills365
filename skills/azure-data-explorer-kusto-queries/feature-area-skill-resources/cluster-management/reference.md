# Azure Data Explorer Cluster Management Reference

## Cluster Operations

### Create Cluster

| Parameter | Description | Constraints |
|-----------|-------------|-------------|
| Cluster Name | Globally unique identifier | 4-22 lowercase alphanumeric |
| Region | Azure region | Affects SKU availability |
| SKU | Compute configuration | See SKU categories below |
| Capacity | Number of instances | Production: min 2 |

**Cluster Types:**
- **Production**: Full SLA, minimum 2 engine + 2 data management nodes
- **Dev/Test**: No SLA, single node for evaluation

**Provisioning Time**: ~10 minutes

**Methods**: Azure Portal, CLI (`az kusto cluster create`), PowerShell (`New-AzKustoCluster`), ARM/Bicep, SDKs

### Start/Stop Cluster

| Operation | Time | Behavior |
|-----------|------|----------|
| Start | ~10 minutes | Hot cache reloads |
| Stop | Immediate | Compute stops, storage persists |

### Delete Cluster

- **Soft Delete**: 14-day recovery period for clusters active >14 days
- **Opt Out**: Tag `opt-out-of-soft-delete = true` for immediate deletion

### Auto-Stop

Clusters stop after 5 days of inactivity.

**Exceptions** (do NOT auto-stop):
- Leader clusters with follower databases
- VNet-deployed clusters
- Start-for-free clusters
- Clusters with Auto-Stop disabled

---

## SKU Categories

### Compute Optimized

Best for high query volumes. High core-to-cache ratio, lowest cost per core.

| Series | vCPU Options | Type |
|--------|--------------|------|
| Eadsv5, ECadsv5 | 2, 4, 8, 16 | AMD |
| Edv4, Edv5 | 2, 4, 8, 16 | Intel |
| Eav4 | 2, 4, 8, 16 | AMD |
| Dv2 | 2, 4, 8, 16 | Intel |

### Storage Optimized

Best for large data volumes. 1-4 TB per node, lowest cost per GB.

| Series | vCPU Options | Type | Premium Storage |
|--------|--------------|------|-----------------|
| Lasv3 | 8, 16, 32 | AMD | No |
| Lsv3 | 8, 16, 32 | Intel | No |
| Easv4, Easv5, ECasv5 | 8, 16 | AMD | Yes |
| Esv4, Esv5 | 8, 16 | Intel | Yes |

### Dev/Test SKUs

- Example: `Dev(No SLA)_Standard_E2a_v4`
- No ADX markup charge, no SLA
- Any compute optimized SKU with 2 cores

### Selection Guidelines

| Factor | Compute Optimized | Storage Optimized |
|--------|-------------------|-------------------|
| High query rates | Preferred | - |
| Large cached data | - | Preferred |
| Cost per core | Lower | Higher |
| Cost per GB | Higher | Lower |

**Recommendation**: Scale up (larger SKU) before scaling out (more nodes) for RAM-constrained workloads.

---

## Scaling

### Horizontal Scaling (Out/In)

**Methods:**

1. **Manual Scale**: Fixed instance count
2. **Optimized Autoscale** (Recommended): Automatic, predictive + reactive
3. **Custom Autoscale**: Rule-based with schedule support

**Optimized Autoscale Triggers:**

| Action | Condition |
|--------|-----------|
| Scale Out | High cache, CPU, or ingestion utilization (1+ hour) |
| Scale In | All metrics below threshold, no throttling, healthy keep-alive |

### Vertical Scaling (Up/Down)

Changes cluster SKU. Process takes 1-3 minutes switchover with data preserved.

**Note**: VNet clusters may experience longer disruption.

---

## Availability Zones

- Distributes across physically separate data centers
- No additional compute cost
- ZRS storage enabled automatically
- Enable during creation or migrate existing clusters

---

## Free Cluster

Trial cluster without Azure subscription.

| Limit | Value |
|-------|-------|
| Storage | ~100 GB uncompressed |
| Databases | 10 |
| Tables per database | 100 |
| Trial period | 1 year |

**Available**: Full KQL, all SDKs, streaming ingestion, dashboards

**Not Available**: External tables, continuous export, workload groups, follower clusters, partitioning policy, Python/R plugins, autoscale, CMK, VNet

---

## Monitoring

### Health Check (KQL)

```kusto
.show diagnostics
| project IsHealthy, NotHealthyReason, IsAttentionRequired, IsScaleOutRequired
```

### Key Metrics (Azure Monitor)

| Category | Metrics |
|----------|---------|
| Health | Keep Alive |
| Resource | CPU, Cache, Memory utilization |
| Ingestion | Utilization, Latency, Volume |
| Query | Duration, Concurrent, Throttled |

### Azure Advisor Recommendations

- **Cost**: Unused clusters, SKU optimization, cache reduction
- **Performance**: SKU changes, cache policy updates
- **Reliability**: Subnet delegation, IP configuration

---

## Follower Databases

Attach databases from another cluster in read-only mode.

**Characteristics:**
- Read-only access
- Synchronization lag: seconds to minutes
- Leader and follower must be in same region
- Cannot modify data, tables, or policies (except caching)

---

## Encryption Options

| Option | Description |
|--------|-------------|
| Disk Encryption | Default Azure encryption |
| Double Encryption | Platform + customer encryption |
| Customer-Managed Keys | Azure Key Vault integration |

**Managed Identities**: System-assigned or user-assigned for authentication to dependent services.
