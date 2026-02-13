# Time Series and ML Best Practices

## Overview

This document provides best practices for leveraging Azure Data Explorer's time series analysis and machine learning capabilities effectively.

---

## When to Use Time Series Analysis

### Ideal Use Cases

1. **Monitoring and Alerting**
   - Service health monitoring
   - Infrastructure metrics tracking
   - Application performance monitoring
   - IoT sensor data analysis

2. **Anomaly Detection**
   - Detecting service outages
   - Identifying unusual traffic patterns
   - Security threat detection
   - Quality control deviations

3. **Forecasting**
   - Capacity planning
   - Demand prediction
   - Resource allocation
   - Budget forecasting

4. **Pattern Recognition**
   - Seasonality detection (daily, weekly, monthly)
   - Trend analysis
   - Periodic behavior validation
   - Baseline establishment

### When NOT to Use

- Data without temporal ordering
- One-time analysis of non-recurring events
- When simpler aggregations suffice
- Data with insufficient history for pattern detection

---

## Choosing the Right Approach

### Native Functions vs Python/R Plugins

| Criteria | Native Functions | Python/R Plugins |
|----------|------------------|------------------|
| **Performance** | Highly optimized, fastest | Slower, additional overhead |
| **Scalability** | Processes thousands of series | Best for smaller datasets |
| **Complexity** | Simple to moderate algorithms | Any algorithm possible |
| **Dependencies** | None | Requires plugin enablement |
| **Use When** | Standard analysis patterns | Custom ML models, specialized libraries |

**Recommendation:** Always start with native functions. Use plugins only when native capabilities are insufficient.

### Native vs UDF Library Functions

| Criteria | Native Functions | UDF Library |
|----------|------------------|-------------|
| `series_decompose_forecast()` | Fast, scalable | - |
| `series_fbprophet_forecast_fl()` | - | More sophisticated, handles holidays |
| `series_decompose_anomalies()` | Fast, scalable | - |
| `series_uv_anomalies_fl()` | - | Uses Azure Cognitive Services |

**Recommendation:** Use native functions for production workloads at scale. Use UDFs for specialized needs or when the native approach produces insufficient results.

---

## make-series Best Practices

### 1. Define Appropriate Time Bins

```kusto
// Good: Match bin size to data granularity and analysis needs
| make-series count() on Timestamp from min_t to max_t step 1h

// Avoid: Too granular bins create sparse data
| make-series count() on Timestamp from min_t to max_t step 1s  // Likely too fine
```

**Guidelines:**
- Use 1-hour bins for daily/weekly patterns
- Use 1-day bins for monthly/yearly patterns
- Ensure each bin has sufficient data points

### 2. Handle Missing Data Appropriately

```kusto
// For metrics where zero is meaningful (e.g., counts)
| make-series count() default=0 on Timestamp ...

// For metrics where missing means unknown
| make-series avg(Value) default=double(null) on Timestamp ...
| extend filled = series_fill_linear(metric)  // Then interpolate
```

### 3. Partition Wisely

```kusto
// Good: Create meaningful groups
| make-series count() on Timestamp ... by Region, Service

// Avoid: Creating too many series (memory/performance impact)
| make-series count() on Timestamp ... by UserId  // Millions of users!
```

### 4. Use Explicit Time Ranges

```kusto
// Good: Explicit ranges for consistency
let min_t = datetime(2024-01-01);
let max_t = datetime(2024-01-31);
| make-series count() on Timestamp from min_t to max_t step 1h

// Avoid: Relying on data boundaries (can vary)
| make-series count() on Timestamp step 1h  // Implicit range
```

---

## Anomaly Detection Best Practices

### 1. Tune the Threshold

```kusto
// Default threshold (1.5) catches mild anomalies
| extend (anomalies, score, baseline) = series_decompose_anomalies(metric)

// Higher threshold (2.5-3.0) for fewer, stronger anomalies
| extend (anomalies, score, baseline) = series_decompose_anomalies(metric, 2.5)

// Lower threshold (1.0) for more sensitive detection
| extend (anomalies, score, baseline) = series_decompose_anomalies(metric, 1.0)
```

**Guidelines:**
- Start with default (1.5)
- Increase if too many false positives
- Decrease if missing important anomalies

### 2. Account for Trends

```kusto
// When data has an upward/downward trend, use linefit
| extend (anomalies, score, baseline) = series_decompose_anomalies(metric, 1.5, -1, 'linefit')

// For stationary data, avg is sufficient (default)
| extend (anomalies, score, baseline) = series_decompose_anomalies(metric)
```

### 3. Specify Known Seasonality

```kusto
// If you know the period, specify it (more reliable)
// Weekly period with hourly bins = 24 * 7 = 168
| extend (anomalies, score, baseline) = series_decompose_anomalies(metric, 1.5, 168)

// Auto-detect when period is unknown (slower, may be inaccurate)
| extend (anomalies, score, baseline) = series_decompose_anomalies(metric, 1.5, -1)
```

### 4. Ensure Sufficient History

- **Minimum:** 2 full cycles of the seasonal pattern
- **Recommended:** 3-4 full cycles for robust baseline
- **Example:** For weekly patterns, need at least 2-3 weeks of data

---

## Forecasting Best Practices

### 1. Exclude Forecast Points from Training

```kusto
// Forecast 7 days ahead (with hourly bins = 168 points)
| extend forecast = series_decompose_forecast(metric, 168, -1, 'linefit')
```

### 2. Extend the Series Range

```kusto
// Create series with extra space for forecast
let horizon = 7d;
| make-series metric=avg(Value) on Timestamp from min_t to max_t+horizon step 1h
| extend forecast = series_decompose_forecast(metric, toint(horizon/1h))
```

### 3. Use Appropriate Trend Setting

```kusto
// For data with clear trend, use linefit
| extend forecast = series_decompose_forecast(metric, points, -1, 'linefit')

// For stationary data, avg works better
| extend forecast = series_decompose_forecast(metric, points, -1, 'avg')
```

---

## ML Integration Patterns

### Pattern 1: Clustering for Segmentation

```kusto
// Use autocluster for failure analysis
Errors
| where Timestamp > ago(7d)
| project ErrorType, Component, Region, Environment
| evaluate autocluster(0.5)
```

### Pattern 2: Frequent Pattern Mining

```kusto
// Use basket for market basket analysis
Transactions
| project Item1, Item2, Item3, Category
| evaluate basket(0.1)
```

### Pattern 3: Python for Custom ML

```kusto
// K-means clustering with sklearn
let kmeans_fl=(tbl:(*), k:int, features:dynamic, cluster_col:string)
{
    let code = ```
from sklearn.cluster import KMeans
km = KMeans(n_clusters=kargs["k"])
df1 = df[kargs["features"]]
km.fit(df1)
result = df
result[kargs["cluster_col"]] = km.labels_
    ```;
    tbl | evaluate python(typeof(*), code, kwargs)
};
```

### Pattern 4: ONNX Model Inference

```kusto
// Score data with pre-trained ONNX model
MyData
| extend pred_col=bool(0)
| invoke predict_onnx_fl(ML_Models, 'MyModel', pack_array('feature1', 'feature2'), 'pred_col')
```

---

## Performance Optimization

### 1. Reduce Data Before Analysis

```kusto
// Good: Filter early
MyTable
| where Timestamp > ago(30d)
| where Region == "US-East"
| make-series count() on Timestamp step 1h by Service

// Avoid: Creating series then filtering
MyTable
| make-series count() on Timestamp step 1h by Region, Service
| where Region == "US-East"  // Too late, already computed all series
```

### 2. Limit Series Count

```kusto
// Process top N most active series
| make-series count() on Timestamp step 1h by Service
| extend total = series_sum(metric)
| top 100 by total
| extend (anomalies, score, baseline) = series_decompose_anomalies(metric)
```

### 3. Use Appropriate Bin Size

- Larger bins = fewer points = faster processing
- Balance between detail and performance

### 4. For Python Plugin

```kusto
// Use per_node distribution when possible
| evaluate hint.distribution = per_node python(...)

// Project only needed columns
| project col1, col2, col3
| evaluate python(...)
```

---

## Common Pitfalls to Avoid

### 1. Insufficient Data

**Problem:** Too little history for pattern detection.
**Solution:** Ensure 2-4 cycles of seasonal patterns minimum.

### 2. Wrong Seasonality Setting

**Problem:** Auto-detect fails or detects wrong period.
**Solution:** Specify known period explicitly.

### 3. Ignoring Trends

**Problem:** Baseline doesn't follow upward/downward data.
**Solution:** Use `linefit` trend parameter.

### 4. Over-Granular Bins

**Problem:** Sparse data with many empty bins.
**Solution:** Use larger bins appropriate to data volume.

### 5. Too Many Series

**Problem:** Memory exhaustion or slow queries.
**Solution:** Filter data, limit grouping dimensions.

### 6. Threshold Too Sensitive

**Problem:** Too many false positive anomalies.
**Solution:** Increase threshold from 1.5 to 2.0-3.0.

---

## Decision Tree: Choosing Functions

```
Need to analyze time series?
|
+-- Create series first --> make-series
|
+-- Need baseline/components?
|   |
|   +-- For understanding data --> series_decompose()
|   +-- For anomaly detection --> series_decompose_anomalies()
|   +-- For forecasting --> series_decompose_forecast()
|
+-- Need pattern detection?
|   |
|   +-- Periodic patterns --> series_periods_detect()
|   +-- Validate known periods --> series_periods_validate()
|
+-- Need regression?
|   |
|   +-- Single linear fit --> series_fit_line()
|   +-- Detect level changes --> series_fit_2lines()
|   +-- Complex curves --> series_fit_poly()
|
+-- Need filtering/smoothing?
|   |
|   +-- Moving average --> series_fir()
|   +-- Exponential smoothing --> series_iir()
|
+-- Need custom ML?
    |
    +-- Standard sklearn --> Python plugin with kmeans_fl, etc.
    +-- Pre-trained model --> predict_onnx_fl()
    +-- Advanced forecasting --> series_fbprophet_forecast_fl()
```

---

*Generated from Azure Data Explorer documentation analysis - Phase 3 In-Depth Research*
