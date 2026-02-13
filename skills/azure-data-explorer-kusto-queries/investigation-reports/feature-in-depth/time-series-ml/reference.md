# Time Series and Machine Learning Reference

## Overview

Azure Data Explorer (ADX) provides comprehensive native support for time series analysis and machine learning. KQL can create, manipulate, and analyze thousands of time series in seconds, enabling near real-time monitoring solutions.

---

## Series Creation: make-series Operator

The `make-series` operator is the foundation for time series analysis. It creates regular time series from tabular data.

### Syntax

```kusto
T | make-series [Column =] Aggregation [default = DefaultValue]
    on AxisColumn from start to end step step
    [by GroupExpression]
```

### Key Parameters

| Parameter | Description |
|-----------|-------------|
| `Aggregation` | Aggregation function (count, avg, sum, min, max, percentile, etc.) |
| `AxisColumn` | Column for ordering (typically datetime) |
| `from/to/step` | Time range and bin size |
| `default` | Fill value for missing bins (default: 0) |
| `by` | Grouping expression for creating multiple series |

### Supported Aggregation Functions

- `count()`, `countif()` - Row counting
- `avg()`, `avgif()` - Average calculations
- `sum()`, `sumif()` - Sum calculations
- `min()`, `max()`, `minif()`, `maxif()` - Extremes
- `percentile()`, `percentiles()` - Percentile calculations
- `stdev()`, `variance()` - Statistical measures
- `dcount()`, `dcountif()` - Distinct counts
- `covariance()`, `covariancep()` - Covariance calculations

### Array Limit

Series arrays are limited to 1,048,576 values (2^20).

---

## Series Decomposition Functions

### series_decompose()

Decomposes a time series into seasonal, trend, and residual components.

```kusto
series_decompose(Series, [Seasonality], [Trend], [Test_points], [Seasonality_threshold])
```

**Parameters:**
- `Seasonality`: `-1` (auto-detect), positive integer (known period), or `0` (no seasonality)
- `Trend`: `avg` (default), `linefit` (linear regression), or `none`
- `Test_points`: Points to exclude from learning (for forecasting)
- `Seasonality_threshold`: Auto-detect threshold (default: 0.6)

**Returns:**
- `baseline`: Predicted value (seasonal + trend)
- `seasonal`: Seasonal component
- `trend`: Trend component
- `residual`: Residual (original - baseline)

### series_decompose_anomalies()

Detects anomalies based on decomposition model.

```kusto
series_decompose_anomalies(Series, [Threshold], [Seasonality], [Trend], [Test_points], [AD_method], [Seasonality_threshold])
```

**Additional Parameters:**
- `Threshold`: Anomaly threshold (default: 1.5 for mild anomalies, 3.0 for strong)
- `AD_method`: `ctukey` (default, 10th-90th percentile) or `tukey` (25th-75th percentile)

**Returns:**
- `ad_flag`: Ternary series (+1 up anomaly, -1 down anomaly, 0 normal)
- `ad_score`: Anomaly score
- `baseline`: Predicted baseline

### series_decompose_forecast()

Forecasts future values using decomposition.

```kusto
series_decompose_forecast(Series, Points, [Seasonality], [Trend], [Seasonality_threshold])
```

**Key Parameter:**
- `Points`: Number of points to forecast at the end of the series

---

## Series Analysis Functions

### Fitting and Regression

| Function | Description |
|----------|-------------|
| `series_fit_line()` | Linear regression, returns slope, intercept, rsquare, variance |
| `series_fit_line_dynamic()` | Same as above, returns dynamic object |
| `series_fit_2lines()` | Two-segment regression for detecting level changes |
| `series_fit_2lines_dynamic()` | Same as above, returns dynamic object |
| `series_fit_poly()` | Polynomial fitting |

### Period Detection

| Function | Description |
|----------|-------------|
| `series_periods_detect()` | Auto-detect periodic patterns |
| `series_periods_validate()` | Validate expected periods exist |

### Statistics

| Function | Description |
|----------|-------------|
| `series_stats()` | Returns min, max, avg, stdev, variance |
| `series_stats_dynamic()` | Same as above in dynamic format |
| `series_outliers()` | Scores anomaly points using Tukey's fence test |
| `series_pearson_correlation()` | Pearson correlation between two series |

### Filtering (Signal Processing)

| Function | Description |
|----------|-------------|
| `series_fir()` | Finite Impulse Response filter (moving average, differentiation) |
| `series_iir()` | Infinite Impulse Response filter (exponential smoothing) |
| `series_fft()` | Fast Fourier Transform |
| `series_ifft()` | Inverse Fast Fourier Transform |

### Interpolation (Gap Filling)

| Function | Description |
|----------|-------------|
| `series_fill_backward()` | Backward fill interpolation |
| `series_fill_forward()` | Forward fill interpolation |
| `series_fill_const()` | Fill with constant value |
| `series_fill_linear()` | Linear interpolation |

### Element-wise Operations

| Function | Description |
|----------|-------------|
| `series_add()` | Add two series element-wise |
| `series_subtract()` | Subtract series |
| `series_multiply()` | Multiply series |
| `series_divide()` | Divide series |
| `series_equals()`, `series_greater()`, etc. | Comparison operations |
| `series_abs()`, `series_log()`, `series_exp()` | Mathematical operations |
| `series_sin()`, `series_cos()`, `series_tan()` | Trigonometric operations |

---

## Native ML Plugins

### autocluster Plugin

Finds common patterns of discrete attributes in data.

```kusto
T | evaluate autocluster([SizeWeight], [WeightColumn], [NumSeeds], [CustomWildcard])
```

**Parameters:**
- `SizeWeight`: Balance between coverage and specificity (0-1, default: 0.5)
- `NumSeeds`: Number of initial search points (default: 25)

**Returns:** Patterns with SegmentId, Count, Percent, and attribute values.

### basket Plugin

Finds frequent patterns using the Apriori algorithm.

```kusto
T | evaluate basket([Threshold], [WeightColumn], [MaxDimensions], [CustomWildcard])
```

**Parameters:**
- `Threshold`: Minimum frequency ratio (0.015-1, default: 0.05)
- `MaxDimensions`: Maximum dimensions per basket (default: 5)

### diffpatterns Plugin

Compares two datasets and finds patterns that differentiate them.

```kusto
T | evaluate diffpatterns(SplitColumn, SplitValue1, SplitValue2)
```

---

## Python Plugin

The Python plugin enables running custom Python code within KQL queries.

### Syntax

```kusto
T | evaluate python(output_schema, script, [script_parameters], [external_artifacts])
```

### Reserved Variables

- `df`: Input data as pandas DataFrame
- `kargs`: Script parameters as Python dictionary
- `result`: Output pandas DataFrame

### Key Features

- Access to numpy (`np`) and pandas (`pd`) pre-imported
- Supports custom package installation via `external_artifacts`
- Can run distributed (`hint.distribution = per_node`)
- Runs in isolated sandbox environment

### Package Installation

1. Create zip file with wheel files
2. Upload to blob storage
3. Configure callout policy
4. Reference via `external_artifacts` parameter

```kusto
| evaluate python(typeof(*), code, kwargs,
    external_artifacts=bag_pack('package.zip', 'https://storage.blob.../package.zip?SAS'))
```

---

## R Plugin (Preview)

Similar to Python plugin but for R scripts.

### Syntax

```kusto
T | evaluate r(output_schema, script, [script_parameters], [external_artifacts])
```

### Reserved Variables

- `df`: Input data as R DataFrame
- `kargs`: Script parameters as R dictionary
- `result`: Output R DataFrame

### Notes

- Based on R 3.4.4 with Anaconda R Essentials
- Must be enabled on cluster
- Supports custom package installation

---

## Functions Library (UDFs)

### Time Series UDFs

| Function | Description |
|----------|-------------|
| `series_moving_avg_fl()` | Moving average calculation |
| `series_exp_smoothing_fl()` | Exponential smoothing |
| `series_dbl_exp_smoothing_fl()` | Double exponential smoothing |
| `series_rolling_fl()` | Rolling window calculations |
| `series_fbprophet_forecast_fl()` | Facebook Prophet forecasting |

### Anomaly Detection UDFs

| Function | Description |
|----------|-------------|
| `series_uv_anomalies_fl()` | Azure Cognitive Services Anomaly Detector |
| `series_uv_change_points_fl()` | Change point detection |
| `series_mv_ee_anomalies_fl()` | Multivariate elliptic envelope |
| `series_mv_if_anomalies_fl()` | Multivariate isolation forest |
| `series_mv_oc_anomalies_fl()` | Multivariate one-class SVM |
| `series_clean_anomalies_fl()` | Clean anomalies from series |

### Clustering UDFs

| Function | Description |
|----------|-------------|
| `kmeans_fl()` | K-means clustering |
| `dbscan_fl()` | DBSCAN density-based clustering |

### ML Prediction UDFs

| Function | Description |
|----------|-------------|
| `predict_fl()` | General prediction with sklearn |
| `predict_onnx_fl()` | ONNX model inference |

### Statistical Tests UDFs

| Function | Description |
|----------|-------------|
| `bartlett_test_fl()` | Bartlett's test for variance homogeneity |
| `ks_test_fl()` | Kolmogorov-Smirnov test |
| `mann_whitney_u_test_fl()` | Mann-Whitney U test |
| `t_test_fl()`, `two_sample_t_test_fl()` | Student's t-test |
| `wilcoxon_test_fl()` | Wilcoxon signed-rank test |
| `normality_test_fl()` | Normality tests |

---

## Visualization

### anomalychart Render Type

Specialized chart for displaying anomalies.

```kusto
| render anomalychart with(anomalycolumns=ad_flag)
```

### timechart Render Type

Standard time series visualization.

```kusto
| render timechart with(title='My Chart', ysplit=panels)
```

---

## Key Documentation Files

| Topic | Documentation Path |
|-------|-------------------|
| make-series | `kusto/query/make-series-operator.md` |
| series_decompose | `kusto/query/series-decompose-function.md` |
| Anomaly Detection | `kusto/query/anomaly-detection.md` |
| Time Series Analysis | `kusto/query/time-series-analysis.md` |
| Python Plugin | `kusto/query/python-plugin.md` |
| R Plugin | `kusto/query/r-plugin.md` |
| Functions Library | `kusto/functions-library/functions-library.md` |

---

*Generated from Azure Data Explorer documentation analysis - Phase 3 In-Depth Research*
