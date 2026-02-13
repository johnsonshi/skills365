# UDF Functions Library Best Practices

## When to Use UDFs vs Native Functions

### Decision Matrix

| Scenario | Use Native | Use UDF |
|----------|------------|---------|
| Simple aggregations | Yes | - |
| Time series decomposition | `series_decompose()` | - |
| Basic forecasting | `series_decompose_forecast()` | - |
| Two-sample t-test (unequal var) | `welch_test()` | - |
| Simple moving average | `series_fir()` | - |
| Static anomaly charts | `render anomalychart` | - |
| K-Means/DBSCAN clustering | - | Yes |
| Advanced statistical tests | - | Yes |
| Prophet forecasting (holidays) | - | Yes |
| Log pattern extraction | - | Yes |
| ONNX model inference | - | Yes |
| Interactive Plotly charts | - | Yes |
| Graph blast radius analysis | - | Yes |

---

## Installation Patterns

### Pattern 1: Query-Defined (Inline)

**When to use:** One-time analysis, testing, sharing with external users, no stored function permissions

```kusto
let kmeans_fl = (tbl:(*), k:int, features:dynamic, cluster_col:string)
{
    // Function implementation
};
MyTable | invoke kmeans_fl(3, dynamic(["col1", "col2"]), "cluster")
```

**Pros:** Self-contained, portable, no permissions needed
**Cons:** Verbose queries, repeated definitions

### Pattern 2: Stored Functions

**When to use:** Repeated use, team standardization, production workloads, dashboards

```kusto
.create-or-alter function with (folder = "Packages\\ML", docstring = "K-Means clustering")
kmeans_fl(tbl:(*), k:int, features:dynamic, cluster_col:string)
{
    // Function implementation
}
```

**Recommended folder structure:**
```
Packages/
  ML/           - Machine learning functions
  Stats/        - Statistical tests
  Series/       - Time series processing
  Text/         - Text analytics
  Cybersecurity/ - Security functions
  Plotly/       - Visualization functions
```

### Pattern 3: Cross-Database References

**When to use:** Central function library for organization

```kusto
cluster('mycluster').database('FunctionLib').kmeans_fl(...)
```

---

## Performance Optimization

### 1. Filter Before UDF

```kusto
// Bad: Filter after Python
MyTable | invoke expensive_udf() | where Category == "A"

// Good: Filter first
MyTable | where Category == "A" | invoke expensive_udf()
```

### 2. Aggregate Before Processing

```kusto
// Bad: Clustering raw data
RawMetrics | invoke kmeans_fl(5, dynamic(["value"]), "cluster")

// Good: Cluster aggregated data
RawMetrics
| summarize avg_value = avg(value) by EntityId
| invoke kmeans_fl(5, dynamic(["avg_value"]), "cluster")
```

### 3. Pre-Cast Data Types

```kusto
MyTable
| extend feature1 = todouble(feature1), feature2 = todouble(feature2)
| invoke kmeans_fl(3, dynamic(["feature1", "feature2"]), "cluster")
```

### 4. Batch Large Datasets

```kusto
MyTable
| partition by DateBucket
(
    invoke log_reduce_fl("logMessage")
)
```

### 5. Cache Expensive Results

```kusto
.set-or-replace ClusterResults <|
MyTable | invoke kmeans_fl(5, dynamic(["f1", "f2"]), "cluster")
```

---

## Input Validation

### Clean Data Before UDF

```kusto
MyTable
| where isnotnull(feature1) and isnotnull(feature2)
| where isfinite(feature1) and isfinite(feature2)
| invoke kmeans_fl(3, dynamic(["feature1", "feature2"]), "cluster")
```

### Test with Samples First

```kusto
MyTable
| take 1000
| invoke kmeans_fl(3, dynamic(["f1", "f2"]), "cluster")
```

### Pre-Create Output Columns

```kusto
MyTable
| extend cluster_id = int(null)
| invoke kmeans_fl(3, dynamic(["f1", "f2"]), "cluster_id")
```

---

## Debugging

### Check Python Plugin Status

```kusto
.show plugins | where PluginName == "python"
```

### Add Debug Output

```kusto
let debug_udf = (tbl:(*))
{
    let code = ```if 1:
        print(f"Input shape: {df.shape}")
        print(f"Columns: {df.columns.tolist()}")
        result = df
    ```;
    tbl | evaluate python(typeof(*), code)
};
```

---

## Security Considerations

### External Artifacts

- Use SAS tokens with minimal permissions (read-only)
- Set token expiration
- Use private blob containers

```kusto
external_artifacts=bag_pack('prophet.zip',
    'https://storage.blob.core.windows.net/packages/prophet.zip?sv=...&se=2026-12-31&sr=b&sp=r&sig=...')
```

### Model Security

- Store models in secured tables
- Audit model updates
- Validate model provenance

### Sensitive Data

- Review data residency for external service UDFs (Azure Cognitive Services)
- Consider data masking before processing

---

## UDF Selection by Use Case

### Statistical Analysis

| Use Case | Recommended |
|----------|-------------|
| Compare two group means | `two_sample_t_test_fl()` or native `welch_test()` |
| Compare distributions | `ks_test_fl()` |
| Check normality | `normality_test_fl()` |
| Non-parametric comparison | `mann_whitney_u_test_fl()` |
| Paired samples | `wilcoxon_test_fl()` |

### Clustering

| Data Characteristics | Recommended |
|---------------------|-------------|
| Known cluster count, spherical | `kmeans_fl()` |
| Unknown clusters, need outliers | `dbscan_fl()` |
| Features as array column | `*_dynamic_fl()` variants |

### Time Series

| Task | Recommended |
|------|-------------|
| Simple smoothing | `series_moving_avg_fl()` or native `series_fir()` |
| Non-linear trends | `series_fit_lowess_fl()` |
| Basic forecasting | Native `series_decompose_forecast()` |
| Advanced forecasting | `series_fbprophet_forecast_fl()` |

### Log Analysis

| Scenario | Recommended |
|----------|-------------|
| Discover patterns | `log_reduce_fl()` |
| Reusable model | `log_reduce_train_fl()` + `log_reduce_predict_fl()` |

### Security/Graph

| Analysis | Recommended |
|----------|-------------|
| Attack surface | `graph_blast_radius_fl()` |
| Crown jewel exposure | `graph_exposure_perimeter_fl()` |
| Key nodes | `graph_node_centrality_fl()` |

---

## Common Pitfalls

| Problem | Solution |
|---------|----------|
| Memory issues with large data | Aggregate first, process in partitions |
| Missing output columns | Pre-extend columns with correct types |
| Type mismatches | Explicitly cast with `todouble()`, `tolong()` |
| Slow DBSCAN | Reduce epsilon, increase min_samples, sample first |
| Prophet package not found | Upload package to blob, configure external_artifacts |

---

## Monitoring

### Track UDF Usage

```kusto
.show commands-and-queries
| where Text contains "invoke" and Text contains "_fl"
| summarize count() by bin(StartedOn, 1d), User
```

### Review Performance

```kusto
.show commands-and-queries
| where Text contains "kmeans_fl"
| summarize avg(Duration), max(Duration), count() by bin(StartedOn, 1h)
```

### Update Functions Safely

1. Test new version with different name
2. Validate results match expectations
3. Use `.create-or-alter` for atomic updates
4. Document changes in function docstring
