# Azure Data Explorer Cluster Management Best Practices

## Sizing Guidelines

### Initial Sizing Strategy

1. Start with smallest SKU meeting requirements
2. Enable Optimized Autoscale from creation
3. Monitor usage for 2-4 weeks before adjustments
4. Use Azure Advisor for optimization recommendations

### SKU Selection Decision Tree

```
Development/Testing/PoC?
  Yes -> Dev/Test SKU (e.g., Dev(No SLA)_Standard_E2a_v4)
  No -> Production workload
        Primary constraint?
          High query rates -> Compute Optimized (Eadsv5, Edv5)
          Large data volumes -> Storage Optimized (Lasv3, Lsv3)
          Balanced -> Start compute optimized, adjust based on metrics
```

### Capacity Planning

**Formula**: `Required Nodes = max(CPU needs, Cache needs, Ingestion needs)`

**Cache Calculation**:
```
Required Cache (GB) = Daily Ingestion (GB) x Hot Cache Days x (1 / Compression Ratio)
Nodes for Cache = Required Cache / Cache per Node
```

**Compression Ratios**:
- Structured logs: 10:1 to 15:1
- JSON/dynamic columns: 5:1 to 8:1
- High cardinality (GUIDs): 3:1 to 5:1

### Memory Guidance

Choose larger SKUs (more RAM) when workloads have:
- Heavy joins across large tables
- Complex aggregations
- High concurrent query loads
- Large result sets

**Rule**: Scale up (larger SKU) before scaling out (more nodes) for RAM constraints.

---

## Cost Optimization

### Cost Components

1. Engine Instances (70-80% of total)
2. Data Management Instances
3. Storage
4. Network egress
5. ADX Markup (not applied to dev clusters)

### Utilization Targets

| Metric | Target | If Below | If Above |
|--------|--------|----------|----------|
| CPU | 40-70% | Scale in/down | Scale out/up |
| Cache | 60-80% | Reduce cache policy | Increase capacity |
| Ingestion | 40-70% | Reduce batching | Scale out |

### Cost Reduction Strategies

**1. Enable Optimized Autoscale**
```
Minimum: 2 (production)
Maximum: Peak load + 20% buffer
```

**2. Optimize Cache Policies**
```kusto
// Check query lookback patterns
.show queries
| where StartedOn > ago(30d)
| summarize QueryCount = count() by bin(MinDataScannedTime, 1d)
```

**3. Use Reserved Capacity**
- 1-year: 30-40% savings
- 3-year: 50-60% savings

**4. Use Dev/Test Clusters**
- No ADX markup
- Single node
- Full query functionality

**5. Enable Auto-Stop**
- Stops after 5 days of inactivity
- Preserves data, stops compute costs

### Cost Monitoring

- Set budget alerts at 80%, 90%, 100%
- Enable anomaly detection
- Track cost per query/ingestion
- Monthly cost analysis

---

## Scaling Strategies

### When to Scale Out (Add Nodes)

**Indicators**:
- CPU >70% sustained
- Query throttling
- High ingestion latency
- Cache at maximum

**Configuration**:
- Set autoscale maximum for peak load
- Allow 1-day evaluation before scale-in

### When to Scale Up (Larger SKU)

**Indicators**:
- Memory pressure during queries
- Complex joins timing out
- Scale-out not improving performance

**Process**:
1. Test new SKU in non-production
2. Schedule during low activity
3. Monitor post-migration

### When to Scale Down

**Indicators**:
- CPU <30% sustained
- Low cache utilization
- Azure Advisor recommendations

**Safe Process**:
1. Reduce maximum autoscale first
2. Monitor 1-2 weeks
3. Consider smaller SKU if stable

### Scaling Patterns

| Pattern | Strategy | Configuration |
|---------|----------|---------------|
| Daily peaks (9-6) | Optimized Autoscale | Min=2, Max=10 |
| Bursty/unpredictable | Reactive autoscale | Min=4, Max=20 |
| Steady 24/7 | Manual with review | Fixed count, quarterly review |

---

## High Availability

### Availability Zones

- Enable for production workloads
- No additional compute cost
- Automatic ZRS storage

### Minimum Instances

| Workload Type | Minimum Instances |
|---------------|-------------------|
| Production | 2 |
| Critical | 3+ across zones |

### Health Monitoring Alerts

| Metric | Threshold |
|--------|-----------|
| Keep Alive | <0.9 |
| CPU | >80% sustained |
| Cache | >90% |
| Failed queries | Spike detection |

```kusto
// Regular health check
.show diagnostics
| project IsHealthy, IsScaleOutRequired, AttentionRequiredReason
```

---

## Operational Excellence

### Infrastructure as Code

Use ARM/Bicep/Terraform for:
- Consistent deployments
- Version control
- Disaster recovery preparation

### Diagnostic Logs to Enable

- Command (admin operations)
- Query (performance)
- Ingestion events
- Table statistics

### Change Management

1. Document current configuration
2. Test in non-production
3. Schedule during low activity
4. Prepare rollback plan

---

## Anti-Patterns to Avoid

### Sizing

| Anti-Pattern | Solution |
|--------------|----------|
| Over-provisioning | Start small, use autoscale |
| Under-provisioning | Monitor and scale proactively |
| Ignoring SKU recommendations | Review Azure Advisor monthly |
| Fixed capacity without autoscale | Enable Optimized Autoscale |

### Operations

| Anti-Pattern | Solution |
|--------------|----------|
| No monitoring | Enable insights and alerts |
| Manual scaling only | Enable autoscale |
| No health checks | Regular `.show diagnostics` |
| Single region | Consider multi-region for DR |

### Cost

| Anti-Pattern | Solution |
|--------------|----------|
| Unused running clusters | Enable auto-stop |
| Production SKU for dev | Use Dev/Test SKUs |
| No reserved capacity | Purchase reservations for stable workloads |
| Unoptimized cache | Align with usage patterns |

---

## Quick Checklists

### New Cluster Setup

- [ ] Choose appropriate SKU (compute vs storage optimized)
- [ ] Enable availability zones (production)
- [ ] Enable Optimized Autoscale
- [ ] Set min/max instance counts
- [ ] Configure auto-stop
- [ ] Enable diagnostic logs
- [ ] Set up monitoring alerts

### Monthly Review

- [ ] Review Azure Advisor recommendations
- [ ] Check utilization metrics
- [ ] Validate cache policy alignment
- [ ] Review cost trends
- [ ] Update autoscale bounds if needed

### Quarterly Planning

- [ ] Assess capacity needs
- [ ] Evaluate reserved capacity opportunities
- [ ] Review SKU appropriateness
- [ ] Plan for growth
