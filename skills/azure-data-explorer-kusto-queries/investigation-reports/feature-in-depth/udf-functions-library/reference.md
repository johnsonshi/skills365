# UDF Functions Library Reference

## Overview

The Azure Data Explorer UDF (User-Defined Functions) Library provides pre-built functions for advanced analytics scenarios that extend beyond native KQL capabilities. These functions leverage the Python plugin to implement sophisticated statistical, machine learning, and visualization algorithms.

---

## Statistical and Probability Functions

### Hypothesis Testing

#### two_sample_t_test_fl()
Performs the two-sample t-test to determine if two samples have significantly different means.

**Syntax:**
```kusto
T | invoke two_sample_t_test_fl(data1, data2, test_statistic, p_value, equal_var)
```

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| data1 | string | Yes | Column name containing first sample (dynamic array) |
| data2 | string | Yes | Column name containing second sample (dynamic array) |
| test_statistic | string | Yes | Output column for test statistic |
| p_value | string | Yes | Output column for p-value |
| equal_var | bool | No | Assume equal variances (default: true) |

**When to use:** Comparing means between two groups, A/B testing, before/after comparisons.

**Note:** For unequal variances, consider the native `welch_test()` function instead.

---

#### ks_test_fl()
Performs the Kolmogorov-Smirnov test to compare two sample distributions.

**Syntax:**
```kusto
T | invoke ks_test_fl(data1, data2, test_statistic, p_value)
```

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| data1 | string | Yes | Column name containing first sample |
| data2 | string | Yes | Column name containing second sample |
| test_statistic | string | Yes | Output column for KS statistic |
| p_value | string | Yes | Output column for p-value |

**When to use:** Testing if two samples come from the same distribution, validating data consistency across time periods.

---

#### normality_test_fl()
Tests whether a sample follows a normal distribution using D'Agostino-Pearson's test.

**Syntax:**
```kusto
T | invoke normality_test_fl(data, test_statistic, p_value)
```

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| data | string | Yes | Column name containing sample data |
| test_statistic | string | Yes | Output column for test statistic |
| p_value | string | Yes | Output column for p-value |

**When to use:** Validating normality assumptions before parametric tests, quality control.

---

#### bartlett_test_fl()
Tests homogeneity of variances across multiple groups (Bartlett's test).

**When to use:** Checking variance equality assumption before ANOVA.

---

#### levene_test_fl()
Tests equality of variances (Levene's test) - more robust to non-normality than Bartlett's.

**When to use:** Checking variance homogeneity when data may not be normal.

---

#### mann_whitney_u_test_fl()
Non-parametric test comparing two independent samples.

**When to use:** Comparing groups when normality assumption is violated.

---

#### wilcoxon_test_fl()
Non-parametric test for paired samples (Wilcoxon signed-rank test).

**When to use:** Paired comparisons when normality cannot be assumed.

---

#### binomial_test_fl()
Tests whether observed success rate differs from expected probability.

**When to use:** Testing success/failure rates, conversion rate analysis.

---

### Mathematical Functions

#### comb_fl()
Calculates C(n,k) - the number of combinations.

#### perm_fl()
Calculates P(n,k) - the number of permutations.

#### factorial_fl()
Calculates n! (factorial).

#### entropy_fl()
Calculates Shannon entropy of probability vectors.

#### percentiles_linear_fl()
Calculates percentiles using linear interpolation between closest ranks.

#### pair_probabilities_fl()
Calculates various probabilities and metrics for categorical variable pairs.

#### pairwise_dist_fl()
Calculates pairwise distances between entities based on multiple variables.

---

## Machine Learning Functions

### Clustering

#### kmeans_fl()
Performs K-Means clustering on features in separate columns.

**Syntax:**
```kusto
T | invoke kmeans_fl(k, features, cluster_col)
```

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| k | int | Yes | Number of clusters |
| features | dynamic | Yes | Array of feature column names |
| cluster_col | string | Yes | Output column for cluster IDs |

**When to use:** Segmenting customers, grouping similar entities, pattern discovery.

**Implementation:** Uses scikit-learn's KMeans algorithm.

---

#### kmeans_dynamic_fl()
K-Means clustering where features are in a single dynamic column.

**Syntax:**
```kusto
T | invoke kmeans_dynamic_fl(k, features_col, cluster_col)
```

**When to use:** When features are already stored as arrays/vectors.

---

#### dbscan_fl()
Density-based clustering (DBSCAN) for features in separate columns.

**Syntax:**
```kusto
T | invoke dbscan_fl(features, cluster_col, epsilon, min_samples, metric, metric_params)
```

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| features | dynamic | Yes | Array of feature column names |
| cluster_col | string | Yes | Output column for cluster IDs |
| epsilon | real | Yes | Maximum distance for neighborhood |
| min_samples | int | No | Minimum samples for core point (default: 10) |
| metric | string | No | Distance metric (default: 'minkowski') |
| metric_params | dynamic | No | Metric parameters (default: {'p': 2}) |

**When to use:** Finding clusters of arbitrary shape, outlier detection (cluster_id = -1).

**Key differences from K-Means:**
- Does not require specifying number of clusters
- Can find non-spherical clusters
- Identifies noise/outliers
- Automatically scales features using StandardScaler

---

#### dbscan_dynamic_fl()
DBSCAN clustering where features are in a single dynamic column.

---

### Prediction

#### predict_fl()
Makes predictions using a trained scikit-learn model stored as a serialized string.

**Syntax:**
```kusto
T | invoke predict_fl(models_tbl, model_name, features_cols, pred_col)
```

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| models_tbl | string | Yes | Table containing serialized models |
| model_name | string | Yes | Name of the model to use |
| features_cols | dynamic | Yes | Array of feature column names |
| pred_col | string | Yes | Output column for predictions |

**Model table schema:**
- `name`: Model name (string)
- `timestamp`: Training timestamp (datetime)
- `model`: Hex-encoded pickled model (string)

**When to use:** Inference using pre-trained sklearn models (classification, regression).

---

#### predict_onnx_fl()
Makes predictions using ONNX-format models for cross-platform ML inference.

**Syntax:**
```kusto
T | invoke predict_onnx_fl(models_tbl, model_name, features_cols, pred_col)
```

**When to use:**
- Using models trained in different frameworks (PyTorch, TensorFlow, etc.)
- Production deployments requiring optimized inference
- Cross-platform model portability

**Requirements:** Model must be converted to ONNX format and stored as hex-encoded string.

---

### Text Embeddings

#### slm_embeddings_fl()
Generates text embeddings using local Small Language Models (SLM).

**When to use:** Semantic similarity, text clustering, vector search preparation.

---

## Time Series Processing Functions

### Smoothing and Filtering

#### series_moving_avg_fl()
Applies a simple moving average filter to a time series.

**Syntax:**
```kusto
series_moving_avg_fl(y_series, n, center)
```

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| y_series | dynamic | Yes | Input numeric array |
| n | int | Yes | Window width |
| center | bool | No | Center the window (default: false) |

**Implementation:** Wrapper around native `series_fir()` function.

---

#### series_exp_smoothing_fl()
Basic exponential smoothing for trend estimation.

#### series_dbl_exp_smoothing_fl()
Double exponential smoothing (Holt's method) for trend and level.

---

### Advanced Analysis

#### series_fit_lowess_fl()
Fits a local polynomial using LOWESS (Locally Weighted Scatterplot Smoothing).

**When to use:** Non-linear trend extraction, robust smoothing.

---

#### series_fit_poly_fl()
Fits a polynomial to series using regression analysis.

---

#### series_fbprophet_forecast_fl()
Time series forecasting using Facebook's Prophet algorithm.

**Syntax:**
```kusto
T | invoke series_fbprophet_forecast_fl(ts_series, y_series, y_pred_series, points, y_pred_low_series, y_pred_high_series)
```

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| ts_series | string | Yes | Column with timestamps |
| y_series | string | Yes | Column with values |
| y_pred_series | string | Yes | Output column for predictions |
| points | int | Yes | Number of points to forecast |
| y_pred_low_series | string | No | Output column for lower confidence |
| y_pred_high_series | string | No | Output column for upper confidence |

**Requirements:** Prophet package must be installed via external_artifacts.

**When to use:** Business forecasting with seasonality, holiday effects.

**Note:** For simpler, faster forecasting, use native `series_decompose_forecast()`.

---

### Time-Weighted Calculations

#### time_weighted_avg_fl()
Calculates time-weighted average using fill-forward interpolation.

#### time_weighted_avg2_fl()
Calculates time-weighted average using linear interpolation.

#### time_weighted_val_fl()
Calculates time-weighted value using linear interpolation.

#### time_window_rolling_avg_fl()
Calculates rolling average over a constant-duration time window.

---

### Anomaly Detection

#### series_clean_anomalies_fl()
Replaces anomalies in a series with interpolated values.

#### series_monthly_decompose_anomalies_fl()
Detects anomalies in series with monthly seasonality.

#### series_uv_anomalies_fl()
Anomaly detection using Azure Cognitive Services Univariate API.

#### series_uv_change_points_fl()
Change point detection using Azure Cognitive Services API.

#### series_mv_ee_anomalies_fl()
Multivariate anomaly detection using elliptical envelope model.

#### series_mv_if_anomalies_fl()
Multivariate anomaly detection using isolation forest model.

#### series_mv_oc_anomalies_fl()
Multivariate anomaly detection using one-class SVM model.

---

### Series Utilities

#### series_rolling_fl()
Applies rolling aggregation function on series.

#### series_lag_fl()
Creates lagged versions of a time series.

#### series_downsample_fl()
Downsamples time series by an integer factor.

#### series_shapes_fl()
Detects positive/negative trends or jumps in series.

#### series_cosine_similarity_fl()
Calculates cosine similarity between two numerical vectors.

#### series_dot_product_fl()
Calculates dot product of two numerical vectors.

#### quantize_fl()
Quantizes metric columns for binning.

---

## PromQL Functions

For Prometheus metrics analysis using the Prometheus data model.

#### series_metric_fl()
Selects and retrieves time series stored in Prometheus format.

#### series_rate_fl()
Calculates average rate of counter metric increase per second.

---

## Text Analytics Functions

### Log Analysis

#### log_reduce_fl()
Finds common patterns in textual logs and outputs a summary table.

**Syntax:**
```kusto
T | invoke log_reduce_fl(reduce_col, use_logram, use_drain, custom_regexes, custom_regexes_policy, delimiters, similarity_th, tree_depth, trigram_th, bigram_th)
```

**Key Parameters:**
| Name | Type | Default | Description |
|------|------|---------|-------------|
| reduce_col | string | Required | Column containing log text |
| use_logram | bool | true | Enable Logram algorithm |
| use_drain | bool | true | Enable Drain algorithm |
| similarity_th | real | 0.5 | Similarity threshold for Drain |
| tree_depth | int | 4 | Prefix tree depth for Drain |

**Algorithms:**
- **Logram**: Uses n-gram frequency analysis, good for large scale, parallelizable
- **Drain**: Uses prefix tree for pattern matching, good for general log parsing

**Output columns:**
- `Count`: Number of matching log lines
- `LogReduce`: Extracted pattern with wildcards
- `example`: Example matching log line

**When to use:** Log aggregation, pattern discovery, log template extraction.

---

#### log_reduce_full_fl()
Same as log_reduce_fl but outputs full table with pattern assignments.

#### log_reduce_train_fl()
Trains a log reduction model that can be reused.

#### log_reduce_predict_fl()
Applies a trained model to classify new logs.

#### log_reduce_predict_full_fl()
Full output version of log_reduce_predict_fl.

---

### Tokenization

#### tokenize_fl()
Tokenizes semi-structured text strings into separate columns.

---

## Graph Analytics Functions

### Security-Focused Graph Analysis

#### graph_blast_radius_fl()
Calculates the Blast Radius (reachability) of source nodes.

**Syntax:**
```kusto
T | invoke graph_blast_radius_fl(sourceIdColumnName, targetIdColumnName, targetWeightColumnName, resultCountLimit, listedIdsLimit)
```

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| sourceIdColumnName | string | Yes | Column with source node IDs |
| targetIdColumnName | string | Yes | Column with target node IDs |
| targetWeightColumnName | string | No | Column with target weights (criticality) |
| resultCountLimit | long | No | Max rows returned (default: 100000) |
| listedIdsLimit | long | No | Max targets per source (default: 50) |

**Output columns:**
- `sourceId`: Source node identifier
- `blastRadiusList`: List of reachable targets
- `blastRadiusScore`: Count of reachable targets
- `blastRadiusScoreWeighted`: Sum of target weights
- `isBlastRadiusListCapped`: Whether list was truncated

**When to use:** Security analysis, identifying high-impact nodes, attack path analysis.

---

#### graph_exposure_perimeter_fl()
Calculates Exposure Perimeter (reverse blast radius) of target nodes.

**When to use:** Identifying which targets are most exposed, crown jewel analysis.

---

#### graph_node_centrality_fl()
Calculates node centrality metrics (degree, betweenness).

**When to use:** Identifying influential nodes in networks.

---

#### graph_path_discovery_fl()
Discovers valid paths between source and target endpoints.

**When to use:** Attack path discovery, access chain analysis.

---

## Cybersecurity Functions

### Anomaly Detection for Security

#### detect_anomalous_access_cf_fl()
Detects anomalous access patterns using collaborative filtering.

**When to use:** Detecting unusual user-resource access patterns.

---

#### detect_anomalous_new_entity_fl()
Detects appearance of anomalous new entities in timestamped data.

**When to use:** New account detection, first-seen analysis.

---

#### detect_anomalous_spike_fl()
Detects anomalous spikes in numeric variables.

**When to use:** Alert volume spikes, traffic anomalies.

---

## Visualization Functions (Plotly)

### Interactive Charts

#### plotly_anomaly_fl()
Renders interactive anomaly charts using Plotly templates.

**Syntax:**
```kusto
T | invoke plotly_anomaly_fl(time_col, val_col, baseline_col, time_high_col, val_high_col, size_high_col, time_low_col, val_low_col, size_low_col, chart_title, series_name, val_name)
```

**Prerequisites:**
```kusto
.set PlotlyTemplate <| cluster('help.kusto.windows.net').database('Samples').PlotlyTemplate
```

**When to use:** Interactive dashboards, detailed anomaly exploration.

**Note:** For non-interactive charts, use native `render anomalychart`.

---

#### plotly_scatter3d_fl()
Renders interactive 3D scatter plots.

**When to use:** Multi-dimensional data visualization, cluster visualization.

---

#### plotly_gauge_fl()
Renders gauge/speedometer charts.

**When to use:** KPI dashboards, status indicators.

---

#### plotly_graph_fl()
Renders interactive graph visualizations.

**When to use:** Network visualization, relationship mapping.

---

## General Utility Functions

#### geoip_fl()
Retrieves geographic information for IP addresses.

#### get_packages_version_fl()
Returns version information for Python engine and packages.

---

## Function Installation Patterns

### Query-Defined (Inline)
```kusto
let function_name = (parameters) { ... };
// Your query using the function
```

### Stored Function
```kusto
.create-or-alter function with (folder = "Packages\\Category", docstring = "Description")
function_name(parameters)
{
    // Function body
}
```

---

## Python Plugin Requirements

Most UDF functions require:
1. Python plugin enabled on the cluster
2. For stored functions: Database User permissions or higher
3. For external packages: Proper sandbox configuration and external_artifacts

Common Python packages used:
- `scikit-learn` - ML algorithms
- `scipy` - Statistical tests
- `pandas` - Data manipulation
- `numpy` - Numerical operations
- `onnxruntime` - ONNX model inference

---

## Document Information

- **Generated**: 2026-02-13
- **Source**: ADX Functions Library Documentation
- **Phase**: 3 - Feature Deep Dive
