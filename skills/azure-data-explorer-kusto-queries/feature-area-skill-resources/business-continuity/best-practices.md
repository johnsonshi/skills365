# Azure Data Explorer Business Continuity Best Practices

## DR Planning Fundamentals

### Define Recovery Objectives First
| Objective | Definition |
|-----------|------------|
| **RTO** | Maximum acceptable downtime |
| **RPO** | Maximum acceptable data loss |
| **Scope** | What needs recovery (all data, recent, specific databases) |

### Map Requirements to Configurations
| Requirement | Configuration |
|-------------|---------------|
| Zero outage tolerance | Active-Active-Active |
| Near-zero downtime | Active-Active |
| Some downtime OK, moderate budget | Active-Hot Standby |
| Cost-sensitive, higher RTO/RPO OK | On-Demand Recovery |

---

## Infrastructure Best Practices

### 1. Deploy Across Availability Zones
```powershell
New-AzKustoCluster -ResourceGroupName "myRG" -Name "mycluster" `
    -Location "West Europe" -SkuName "Standard_E16ads_v5" `
    -Zone "1", "2", "3"
```
- Survives datacenter failures within region
- Automatic failover, no application changes

### 2. Use Azure Paired Regions
| Primary | Secondary |
|---------|-----------|
| East US | West US |
| North Europe | West Europe |
| Southeast Asia | East Asia |

Benefits: Staggered updates, shared infrastructure resilience, prioritized recovery.

### 3. Match SKUs for Active-Active
Use identical SKUs for primary and DR clusters in Active-Active.
Exception: Hot standby can use smaller SKUs with autoscale configured.

---

## Data Replication Best Practices

### 1. Implement Parallel Ingestion
For Active-Active: ingest to all clusters simultaneously.
```
Data Sources --> Event Hubs (Region A) --> ADX Cluster A
            \-> Event Hubs (Region B) --> ADX Cluster B
```

### 2. Use GRS Storage for Continuous Export
```kusto
.create-or-alter continuous-export BackupExport
to table ExternalBackup
with (intervalBetweenRuns=5m)
<| MyTable
```

### 3. Configure Appropriate Export Intervals
| Scenario | Interval |
|----------|----------|
| Near-real-time | 1-5 minutes |
| Standard analytics | 15-30 minutes |
| Historical/archive | 1 hour+ |

---

## Schema and Configuration Management

### 1. Version Control Everything
- Table schemas, functions, policies
- Ingestion mappings, materialized views
- Security role assignments

```kusto
.show database schema as csl script with (ShowObfuscatedStrings = true)
```

### 2. Automate Deployments
Use CI/CD pipelines (Azure DevOps, GitHub Actions, ARM/Bicep).

### 3. Validate Cross-Cluster Sync
```kusto
let localCount = MyTable | count;
let remoteCount = cluster("backup-cluster").database("MyDB").MyTable | count;
print Local=toscalar(localCount), Remote=toscalar(remoteCount),
      Difference=toscalar(localCount) - toscalar(remoteCount)
```

---

## Failover Strategy Best Practices

### 1. Use DNS-Based Routing
| Service | Best For |
|---------|----------|
| Azure Traffic Manager | DNS routing, simple setup |
| Azure Front Door | Global load balancing, SSL offloading |

### 2. Build Application-Level Failover
Implement retry logic across cluster endpoints in client code.

### 3. Configure Autoscale on Standby
```powershell
Set-AzKustoCluster -ResourceGroupName "myRG" -Name "standby-cluster" `
    -OptimizedAutoscale_IsEnabled $true `
    -OptimizedAutoscale_Minimum 2 `
    -OptimizedAutoscale_Maximum 10
```

---

## RTO/RPO Optimization

### Minimize RTO
| Strategy | Impact |
|----------|--------|
| Pre-warmed standby clusters | Minutes |
| Automated failover with Traffic Manager | Minutes |
| Continuous data replication | Near-zero |
| Pre-deployed schema in DR region | Minutes |

### Minimize RPO
| Strategy | Impact |
|----------|--------|
| Dual ingestion (Active-Active) | Zero |
| Streaming ingestion to both clusters | Sub-second |
| Frequent continuous export | Minutes |

---

## Testing and Monitoring

### Conduct Quarterly DR Drills
- [ ] Failover to secondary
- [ ] Verify data consistency
- [ ] Test client failover
- [ ] Measure actual RTO/RPO
- [ ] Test failback procedure

### Monitor Key Metrics
| Metric | Target | Alert |
|--------|--------|-------|
| Continuous export lateness | < 15 min | > 30 min |
| Cross-cluster row count diff | < 0.1% | > 1% |
| Follower database lag | < 5 min | > 15 min |
| Export failure rate | 0% | Any failure |

---

## Cost Optimization

### Right-Size Standby Clusters
Use smaller SKUs with autoscale for standby (scales up during failover).

### Use On-Demand Recovery For
- Non-critical analytics
- Historical data that can be reprocessed
- Dev/test environments

### Optimize Storage
- Use hot/cold tiers appropriately
- Set retention policies for auto-delete
- Use Parquet format for exports
- Apply storage lifecycle policies

---

## Documentation Requirements

1. **Architecture Diagram** - Current DR topology
2. **Failover Runbook** - Step-by-step procedures
3. **Recovery Checklist** - Post-failover validation
4. **Contact List** - Escalation paths
5. **Dependency Map** - Upstream/downstream systems
