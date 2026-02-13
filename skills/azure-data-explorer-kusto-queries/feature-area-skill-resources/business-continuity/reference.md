# Azure Data Explorer Business Continuity Reference

## High Availability Architecture

### Storage Layer
| Type | Description | Use Case |
|------|-------------|----------|
| **LRS** | 3 replicas in single datacenter | Default, cost-effective |
| **ZRS** | Replicas across availability zones | Auto-enabled with AZ deployment |

### Compute Layer (Availability Zones)
- Distribute nodes across zones for intra-region resiliency
- Zone failure causes degradation, not complete outage
- Recovery automatic when zone recovers

```powershell
# Enable AZ at cluster creation
New-AzKustoCluster -ResourceGroupName "myRG" -Name "mycluster" `
    -Location "North Europe" -SkuName "Standard_E16ads_v5" `
    -Zone "1", "2", "3"

# Migrate existing cluster to AZ
Update-AzKustoCluster -ResourceGroupName "myRG" -Name "mycluster" `
    -Zone "1", "2", "3"
```

---

## Leader-Follower Clusters

### Capabilities
| Feature | Description |
|---------|-------------|
| Read-only access | Query without ingestion overhead |
| Automatic sync | Changes sync with seconds-to-minutes lag |
| Same storage | No separate ingestion required |
| Cost segregation | Associate compute with query consumers |

### Configuration Options
- Follow one, several, or all (`*`) databases from a leader
- Follow databases from multiple leader clusters
- Cluster can contain both follower and leader databases

### Table-Level Sharing Properties
| Array | Purpose |
|-------|---------|
| `tablesToInclude/Exclude` | Filter tables (wildcards supported) |
| `externalTablesToInclude/Exclude` | Filter external tables |
| `materializedViewsToInclude/Exclude` | Filter materialized views |
| `functionsToInclude/Exclude` | Filter functions |

**Limit:** 100 entries total across all arrays

### Principal/Caching Modification Kinds
| Kind | Behavior |
|------|----------|
| **Union** (default) | Combine leader + follower definitions |
| **Replace** | Use only follower definitions |
| **None** | Use only leader definitions |

### Limitations
- Leader and follower must be in same region
- Streaming ingestion requires enablement on follower
- Cannot delete attached databases without detaching
- Table-level sharing not supported with `*` notation

---

## Cross-Cluster Queries

```kusto
// Cross-database (same cluster)
database("DatabaseName").TableName

// Cross-cluster
cluster("ClusterName").database("DatabaseName").TableName

// Wildcard union
union cluster("OtherCluster").database("*").*

// Clear schema cache after remote changes
.clear cache remote-schema cluster("ClusterName").database("DatabaseName")
```

**Limitations:** Remote functions must return tabular schema; scalar functions only accessible in same cluster.

---

## Continuous Data Export

```kusto
.create-or-alter continuous-export MyExport
over (SourceTable)
to table ExternalBlob
with (intervalBetweenRuns=1h, forcedLatency=10m, sizeLimit=104857600)
<| SourceTable | where Timestamp > ago(30d)
```

| Property | Description |
|----------|-------------|
| `intervalBetweenRuns` | Time between executions (min: 1 minute) |
| `forcedLatency` | Delay to ensure records ingested |
| `sizeLimit` | Max artifact size before compression |

**Monitoring:**
```kusto
.show continuous-export MyExport
.show continuous-export MyExport failures
.show continuous-export MyExport exported-artifacts
```

---

## Backup and Restore

### Schema Backup/Restore
```kusto
// Export schema
.show database schema as csl script with (ShowObfuscatedStrings = true)

// Restore schema
.execute database script <|
// CSL script content
```

### Table Recovery
```kusto
// Recover dropped table (requires recoverability enabled in retention policy)
.undo drop table TableName
```

---

## Resource Protection

```azurecli
# Apply delete lock
az lock create --name PreventDelete \
    --resource-group myRG \
    --resource-name mycluster \
    --resource-type Microsoft.Kusto/clusters \
    --lock-type CanNotDelete
```

---

## DR Configuration Comparison

| Configuration | RPO | RTO | Effort | Cost |
|---------------|-----|-----|--------|------|
| **Active-Active-Active** | 0 hrs | 0 hrs | Lower | Highest |
| **Active-Active** | 0 hrs | 0 hrs | Lower | High |
| **Active-Hot Standby** | 0 hrs | Low | Medium | Medium |
| **On-Demand Recovery** | Highest | Highest | Highest | Lowest |

---

## Azure Data Share

Managed cross-organization data sharing using follower database mechanism:
1. Provider creates share, adds database/tables, sends invitation
2. Receiver accepts and maps to their cluster
3. Data syncs automatically (seconds-to-minutes lag)

Supports full database or table-level sharing; enables cross-tenant sharing.
