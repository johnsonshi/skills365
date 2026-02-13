# Azure Data Explorer Cluster Management Best Practices

## Overview

This document provides best practices for Azure Data Explorer cluster management, covering sizing guidelines, cost optimization strategies, and scaling approaches.

---

## Sizing Guidelines

### Initial Cluster Sizing

**Start Small, Scale Smart:**
1. Begin with the smallest SKU that meets initial requirements
2. Enable Optimized Autoscale from the start
3. Monitor usage patterns for 2-4 weeks before making sizing decisions
4. Use Azure Advisor recommendations for optimization

**Sizing Factors to Consider:**
- Daily ingestion volume (GB/TB)
- Hot cache retention period
- Query concurrency requirements
- Query complexity (joins, aggregations)
- Peak vs. average load patterns

### SKU Selection Decision Tree

```
Is this for development/testing/PoC?
├── Yes → Use Dev/Test SKU (e.g., Dev(No SLA)_Standard_E2a_v4)
└── No → Production workload
         │
         What is your primary constraint?
         ├── High query rates → Compute Optimized SKUs
         │   └── Start with Eadsv5 or Edv5 series
         ├── Large data volumes → Storage Optimized SKUs
         │   └── Start with Lasv3 or Lsv3 series
         └── Balanced workload → Start with compute optimized, adjust based on metrics
```

### Capacity Planning Formula

**Estimated Nodes = max(CPU needs, Cache needs, Ingestion needs)**

**Cache Calculation:**
```
Required Cache (GB) = Daily Ingestion (GB) × Hot Cache Days × (1 / Compression Ratio)
Nodes for Cache = Required Cache / Cache per Node
```

**Typical Compression Ratios:**
- Structured logs: 10:1 to 15:1
- JSON/dynamic columns: 5:1 to 8:1
- GUIDs/high cardinality: 3:1 to 5:1

### Memory Considerations

**When to choose larger SKUs (more RAM):**
- Heavy use of joins across large tables
- Complex aggregations
- High concurrent query loads
- Large result sets

**Rule of Thumb:** Prefer scaling up (larger SKU) before scaling out (more nodes) when RAM is the constraint.

---

## Cost Optimization

### Infrastructure Cost Components

1. **Engine Instances**: Primary cost driver (70-80% of total)
2. **Data Management Instances**: Managed automatically, scales with engine
3. **Storage**: Persistent storage costs
4. **Networking**: Egress costs for cross-region traffic
5. **ADX Markup**: Premium support charge (not applied to dev clusters)

### Cost Reduction Strategies

#### 1. Right-Size Your Cluster

**Use Azure Advisor:**
- Review SKU recommendations regularly
- Check for unused running clusters
- Identify underutilized capacity

**Cluster Utilization Targets:**
| Metric | Target Range | Action if Below | Action if Above |
|--------|--------------|-----------------|-----------------|
| CPU | 40-70% | Scale in/down | Scale out/up |
| Cache | 60-80% | Reduce cache policy | Increase capacity |
| Ingestion | 40-70% | Reduce batching | Scale out |

#### 2. Enable Optimized Autoscale

**Configuration:**
```
Minimum Instances: 2 (production minimum)
Maximum Instances: Based on peak load + 20% buffer
```

**Benefits:**
- Automatic scale-in during low usage
- Predictive scaling for known patterns
- Reduces manual intervention

#### 3. Optimize Cache Policies

**Align cache with actual usage:**
```kusto
// Check actual query lookback patterns
.show queries
| where StartedOn > ago(30d)
| summarize QueryCount = count() by bin(MinDataScannedTime, 1d)
```

**Reduce cache where possible:**
- Set table-level cache policies based on access patterns
- Use shorter cache for infrequently queried tables
- Consider hot windows for time-based queries

#### 4. Use Reserved Capacity

**Savings Options:**
- **Reserved Instances (1-year)**: 30-40% savings
- **Reserved Instances (3-year)**: 50-60% savings
- **Savings Plans**: For VMs used by cluster

**When to Use:**
- Stable, predictable workloads
- Long-term commitments
- Production environments

#### 5. Dev/Test Clusters

**Use Dev/Test for:**
- Development environments
- Proof of concepts
- Testing and validation
- Training/learning

**Benefits:**
- No ADX markup charge
- Single node (lowest compute cost)
- Full query functionality

#### 6. Stop Unused Clusters

**Enable Auto-Stop:**
- Automatically stops after 5 days of inactivity
- Preserves data while stopping compute costs

**Manual Stop for:**
- Known idle periods (weekends, holidays)
- Scheduled maintenance windows

### Cost Monitoring

**Set Up Alerts:**
- Budget alerts at 80%, 90%, 100%
- Anomaly detection for unexpected spikes
- Daily/weekly cost reports

**Review Regularly:**
- Monthly cost analysis
- Compare actual vs. estimated
- Track cost per query/ingestion

---

## Scaling Strategies

### When to Scale Out (Add Nodes)

**Indicators:**
- Consistently high CPU utilization (>70%)
- Query throttling occurring
- High ingestion latency
- Cache utilization at maximum

**Best Practices:**
- Set autoscale maximum to accommodate peaks
- Allow 1-day evaluation before scale-in
- Consider predictive patterns for seasonal workloads

### When to Scale Up (Larger SKU)

**Indicators:**
- Memory pressure during query execution
- Complex joins timing out
- Scale-out not improving performance
- RAM-bound workloads

**Process:**
1. Review workload characteristics
2. Test new SKU in non-production
3. Schedule migration during low-activity period
4. Monitor post-migration performance

### When to Scale Down

**Indicators:**
- Consistent low CPU (<30%)
- Low cache utilization
- Over-provisioned for actual workload
- Azure Advisor cost recommendations

**Safe Scale-Down Process:**
1. Reduce maximum autoscale instances first
2. Monitor for 1-2 weeks
3. Consider smaller SKU if stable
4. Keep buffer for unexpected peaks

### Scaling Patterns

#### Predictable Load Pattern
```
Pattern: Daily peaks 9 AM - 6 PM
Strategy: Optimized Autoscale with predictive logic
Config: Min=2, Max=10, let autoscale handle daily cycles
```

#### Bursty/Unpredictable Pattern
```
Pattern: Sporadic high-volume queries
Strategy: Optimized Autoscale with reactive scaling
Config: Min=4, Max=20, quick scale-out triggers
```

#### Steady-State Pattern
```
Pattern: Consistent load 24/7
Strategy: Manual scaling with periodic review
Config: Fixed instance count, quarterly optimization review
```

---

## High Availability Best Practices

### Enable Availability Zones

**Benefits:**
- Protection against zone failures
- No additional compute cost
- Automatic ZRS storage

**When to Enable:**
- Production workloads
- Business-critical applications
- SLA requirements

### Configure Appropriate Minimum Instances

**Production Clusters:**
- Minimum 2 instances for redundancy
- Ensure capacity for single-node failure

**Critical Workloads:**
- Minimum 3+ instances
- Distribute across availability zones

### Monitor Health Proactively

**Set Up Alerts For:**
- Keep Alive drops below 0.9
- CPU sustained above 80%
- Cache utilization above 90%
- Failed queries spike

**Regular Health Checks:**
```kusto
.show diagnostics
| project IsHealthy, IsScaleOutRequired, AttentionRequiredReason
```

---

## Operational Excellence

### Infrastructure as Code

**Use ARM/Bicep/Terraform for:**
- Consistent deployments
- Version control
- Repeatable environments
- Disaster recovery preparation

### Automated Monitoring

**Enable Diagnostic Logs:**
- Command (admin operations)
- Query (query performance)
- Ingestion events
- Table statistics

**Configure Azure Monitor:**
- Metric alerts
- Log-based alerts
- Workbooks for visualization

### Change Management

**Before Making Changes:**
1. Document current configuration
2. Test in non-production
3. Schedule during low-activity windows
4. Have rollback plan

**Track Changes:**
- Activity logs
- Version-controlled templates
- Change documentation

---

## Anti-Patterns to Avoid

### Sizing Anti-Patterns

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| Over-provisioning | Wasted costs | Start small, use autoscale |
| Under-provisioning | Poor performance | Monitor and scale proactively |
| Ignoring SKU recommendations | Suboptimal cost/performance | Review Azure Advisor monthly |
| Fixed capacity without autoscale | Inefficient resource use | Enable Optimized Autoscale |

### Operational Anti-Patterns

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| No monitoring | Blind to issues | Enable insights and alerts |
| Manual scaling only | Slow response | Enable autoscale |
| No health checks | Late issue detection | Regular .show diagnostics |
| Single region | No DR capability | Consider multi-region |

### Cost Anti-Patterns

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| Unused running clusters | Wasted compute | Enable auto-stop |
| Production SKU for dev | Excessive dev costs | Use Dev/Test SKUs |
| No reserved capacity | Higher costs | Purchase reservations |
| Unoptimized cache | Over-provisioned storage | Align with usage patterns |

---

## Quick Reference Checklist

### New Cluster Setup
- [ ] Choose appropriate SKU category (compute vs. storage optimized)
- [ ] Enable availability zones (production)
- [ ] Enable Optimized Autoscale
- [ ] Set appropriate min/max instance counts
- [ ] Configure auto-stop (if applicable)
- [ ] Enable diagnostic logs
- [ ] Set up monitoring alerts

### Monthly Review
- [ ] Review Azure Advisor recommendations
- [ ] Check cluster utilization metrics
- [ ] Validate cache policy alignment
- [ ] Review cost trends
- [ ] Update autoscale bounds if needed

### Quarterly Planning
- [ ] Assess capacity needs
- [ ] Evaluate reserved capacity opportunities
- [ ] Review SKU appropriateness
- [ ] Plan for growth or optimization

---

## Document Information

- **Generated**: 2026-02-13
- **Source**: Azure Data Explorer documentation analysis
- **Phase**: 3 - Feature In-Depth Research
