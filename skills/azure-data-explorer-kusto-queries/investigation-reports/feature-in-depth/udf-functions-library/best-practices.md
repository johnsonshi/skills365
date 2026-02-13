# UDF Functions Library Best Practices

## Overview

This document provides guidance on when and how to effectively use User-Defined Functions (UDFs) in Azure Data Explorer, including performance considerations, installation patterns, and decision frameworks.

---

## When to Use UDFs vs Native Functions

### Decision Framework

| Scenario | Recommendation | Rationale |
|----------|----------------|-----------|
| Simple aggregations | Native functions | Built-in functions are optimized |
| Time series decomposition | `series_decompose()` native | Faster, scales better |
| Simple forecasting | `series_decompose_forecast()` native | Simpler, no Python overhead |
| Advanced ML clustering | `kmeans_fl()` / `dbscan_fl()` | Python sklearn algorithms |
| Statistical hypothesis tests | UDF statistical functions | SciPy provides robust implementations |
| Complex forecasting | `series_fbprophet_forecast_fl()` | Prophet handles holidays, changepoints |
| Log pattern extraction | `log_reduce_fl()` | Specialized algorithms (Drain, Logram) |
| ONNX model inference | `predict_onnx_fl()` | Cross-platform model support |
| Interactive visualization | Plotly UDFs | Rich interactivity beyond native charts |
| Graph blast radius | `graph_blast_radius_fl()` | Security-specific calculations |

### Native Alternatives to Consider First

Before using a UDF, check if a native function meets your needs:

| UDF Function | Native Alternative | When Native is Sufficient |
|--------------|-------------------|---------------------------|
| `series_moving_avg_fl()` | `series_fir()` | Simple filtering needs |
| `two_sample_t_test_fl()` | `welch_test()` | Unequal variance assumption |
| `series_fbprophet_forecast_fl()` | `series_decompose_forecast()` | Basic seasonality, no holidays |
| `kmeans_fl()` | `autocluster` plugin | Pattern mining, not strict clustering |
| `plotly_anomaly_fl()` | `render anomalychart` | Static anomaly visualization |

---

## Installation Patterns

### Pattern 1: Query-Defined (Inline)

**When to use:**
- One-time analysis
- Exploring/testing a UDF
- Sharing queries with external users
- No permissions to create stored functions

**Syntax:**
```kusto
let kmeans_fl = (tbl:(*), k:int, features:dynamic, cluster_col:string)
{
    // Function implementation
};
// Use immediately after
MyTable
| invoke kmeans_fl(3, dynamic(["col1", "col2"]), "cluster")
```

**Advantages:**
- Self-contained, portable queries
- No permissions required
- Easy to share and version control

**Disadvantages:**
- Query becomes verbose
- Function definition repeated in every query
- Harder to maintain across many queries

---

### Pattern 2: Stored Functions

**When to use:**
- Repeated use across multiple queries
- Team-wide standardization
- Production workloads
- Dashboard queries

**Installation:**
```kusto
.create-or-alter function with (folder = "Packages\\ML", docstring = "K-Means clustering")
kmeans_fl(tbl:(*), k:int, features:dynamic, cluster_col:string)
{
    // Function implementation
}
```

**Usage:**
```kusto
MyTable
| invoke kmeans_fl(3, dynamic(["col1", "col2"]), "cluster")
```

**Advantages:**
- Clean, readable queries
- Single point of maintenance
- Shared across team
- Can organize in folders

**Disadvantages:**
- Requires Database User permissions or higher
- Need to manage function versions

**Recommended folder structure:**
```
Packages/
  ML/           - Machine learning functions
  Stats/        - Statistical tests
  Series/       - Time series processing
  Text/         - Text analytics
  Cybersecurity/ - Security-focused functions
  Plotly/       - Visualization functions
```

---

### Pattern 3: Cross-Database Function References

**When to use:**
- Central function library for multiple databases
- Standardized functions across organization

```kusto
// Reference function from another database
cluster('mycluster').database('FunctionLib').kmeans_fl(...)
```

---

## Performance Best Practices

### 1. Minimize Data Sent to Python Plugin

**Bad:**
```kusto
MyTable
| invoke expensive_python_udf()  // Sends all data to Python
| where Category == "A"          // Filter happens after Python
```

**Good:**
```kusto
MyTable
| where Category == "A"          // Filter first
| invoke expensive_python_udf()  // Less data to Python
```

### 2. Aggregate Before Python Processing

**Bad:**
```kusto
RawMetrics
| invoke kmeans_fl(5, dynamic(["value"]), "cluster")  // Clustering millions of rows
```

**Good:**
```kusto
RawMetrics
| summarize avg_value = avg(value) by EntityId
| invoke kmeans_fl(5, dynamic(["avg_value"]), "cluster")  // Clustering aggregated data
```

### 3. Use Appropriate Data Types

Pre-process columns to expected types before invoking UDFs:

```kusto
MyTable
| extend
    feature1 = todouble(feature1),
    feature2 = todouble(feature2)
| invoke kmeans_fl(3, dynamic(["feature1", "feature2"]), "cluster")
```

### 4. Batch Processing for Large Datasets

For very large datasets, process in batches:

```kusto
MyTable
| partition by DateBucket
(
    invoke log_reduce_fl("logMessage")
)
```

### 5. Cache Results for Expensive Operations

Store expensive UDF results using materialized views or stored query results:

```kusto
// Store clustering results
.set-or-replace ClusterResults <|
MyTable
| invoke kmeans_fl(5, dynamic(["f1", "f2"]), "cluster")
```

---

## Error Handling and Debugging

### 1. Validate Input Data

Check for nulls and invalid values before UDF:

```kusto
MyTable
| where isnotnull(feature1) and isnotnull(feature2)
| where isfinite(feature1) and isfinite(feature2)
| invoke kmeans_fl(3, dynamic(["feature1", "feature2"]), "cluster")
```

### 2. Test with Small Samples First

```kusto
MyTable
| take 1000  // Test with sample
| invoke kmeans_fl(3, dynamic(["f1", "f2"]), "cluster")
```

### 3. Check Python Plugin Status

```kusto
// Verify Python plugin is enabled
.show plugins
| where PluginName == "python"
```

### 4. Debug Python Code

Add print statements (visible in query results for debugging):

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

### 1. External Artifacts

When using external packages (like Prophet), secure your blob storage:

- Use SAS tokens with minimal permissions (read-only)
- Set token expiration
- Use private blob containers

```kusto
external_artifacts=bag_pack('prophet.zip',
    'https://mystorageaccount.blob.core.windows.net/packages/prophet.zip?sv=2020-08-04&se=2026-12-31&sr=b&sp=r&sig=...')
```

### 2. Model Security

For `predict_fl()` and `predict_onnx_fl()`:

- Store models in secured tables
- Audit model updates
- Validate model provenance

### 3. Sensitive Data

Be cautious with UDFs that send data to external services:

- `series_uv_anomalies_fl()` uses Azure Cognitive Services
- Review data residency requirements
- Consider data masking before processing

---

## UDF Selection Guidelines by Use Case

### Statistical Analysis

| Use Case | Recommended UDF | Alternative |
|----------|-----------------|-------------|
| Compare two group means | `two_sample_t_test_fl()` | Native `welch_test()` |
| Compare distributions | `ks_test_fl()` | - |
| Check normality | `normality_test_fl()` | - |
| Non-parametric comparison | `mann_whitney_u_test_fl()` | - |
| Paired samples | `wilcoxon_test_fl()` | - |
| Variance homogeneity | `levene_test_fl()` or `bartlett_test_fl()` | - |

### Clustering

| Data Characteristics | Recommended UDF |
|---------------------|-----------------|
| Known number of clusters | `kmeans_fl()` |
| Unknown clusters, need outlier detection | `dbscan_fl()` |
| Features as array column | `*_dynamic_fl()` variants |
| Spherical clusters | `kmeans_fl()` |
| Arbitrary shaped clusters | `dbscan_fl()` |

### Time Series

| Task | Recommended Approach |
|------|---------------------|
| Simple moving average | `series_moving_avg_fl()` or native `series_fir()` |
| Trend smoothing | `series_exp_smoothing_fl()` |
| Non-linear trends | `series_fit_lowess_fl()` |
| Basic forecasting | Native `series_decompose_forecast()` |
| Advanced forecasting | `series_fbprophet_forecast_fl()` |
| Prometheus metrics | `series_metric_fl()`, `series_rate_fl()` |

### Log Analysis

| Scenario | Recommended UDF |
|----------|-----------------|
| Discover log patterns | `log_reduce_fl()` |
| Full pattern assignment | `log_reduce_full_fl()` |
| Reusable pattern model | `log_reduce_train_fl()` + `log_reduce_predict_fl()` |

### Graph/Security Analysis

| Analysis Type | Recommended UDF |
|--------------|-----------------|
| Attack surface analysis | `graph_blast_radius_fl()` |
| Crown jewel exposure | `graph_exposure_perimeter_fl()` |
| Key node identification | `graph_node_centrality_fl()` |
| Path discovery | `graph_path_discovery_fl()` |
| Anomalous access | `detect_anomalous_access_cf_fl()` |

---

## Common Pitfalls and Solutions

### Pitfall 1: Memory Issues with Large Data

**Problem:** Python sandbox has memory limits.

**Solution:**
- Aggregate data before UDF
- Process in partitions
- Increase sandbox memory limits (cluster setting)

### Pitfall 2: Missing Output Columns

**Problem:** UDF output columns not appearing.

**Solution:** Pre-extend output columns with correct types:

```kusto
MyTable
| extend cluster_id = int(null)  // Pre-create column with correct type
| invoke kmeans_fl(3, dynamic(["f1", "f2"]), "cluster_id")
```

### Pitfall 3: Type Mismatches

**Problem:** Python expects specific types.

**Solution:** Explicitly cast columns:

```kusto
| extend value = todouble(value)
```

### Pitfall 4: Slow DBSCAN

**Problem:** DBSCAN is slow on large datasets.

**Solution:**
- Reduce epsilon for faster neighborhood search
- Increase min_samples to reduce noise processing
- Sample data first for parameter tuning

### Pitfall 5: Prophet Package Not Found

**Problem:** `series_fbprophet_forecast_fl()` fails.

**Solution:**
1. Download prophet package zip
2. Upload to blob storage
3. Generate SAS token
4. Update external_artifacts URL

---

## Monitoring and Maintenance

### Track UDF Usage

```kusto
.show commands-and-queries
| where Text contains "invoke" and Text contains "_fl"
| summarize count() by bin(StartedOn, 1d), User
```

### Review Function Performance

```kusto
.show commands-and-queries
| where Text contains "kmeans_fl"
| summarize avg(Duration), max(Duration), count() by bin(StartedOn, 1h)
```

### Update Functions Safely

1. Test new version with different name
2. Validate results match expectations
3. Use `.create-or-alter` to update atomically
4. Keep change log in function docstring

---

## Document Information

- **Generated**: 2026-02-13
- **Source**: ADX Functions Library Best Practices
- **Phase**: 3 - Feature Deep Dive
