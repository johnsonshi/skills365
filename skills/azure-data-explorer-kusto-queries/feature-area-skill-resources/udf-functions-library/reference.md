# UDF Functions Library Reference

## Overview

Azure Data Explorer UDFs (User-Defined Functions) extend KQL with Python-powered statistical, ML, and visualization capabilities. Functions follow the `*_fl()` naming convention and require the Python plugin.

---

## Statistical Functions

### Hypothesis Testing

| Function | Purpose | When to Use |
|----------|---------|-------------|
| `two_sample_t_test_fl()` | Compare means of two samples | A/B testing, before/after comparisons |
| `ks_test_fl()` | Compare two distributions (Kolmogorov-Smirnov) | Data consistency validation across periods |
| `normality_test_fl()` | Test for normal distribution | Validate assumptions before parametric tests |
| `mann_whitney_u_test_fl()` | Non-parametric two-sample comparison | When normality assumption is violated |
| `wilcoxon_test_fl()` | Non-parametric paired samples test | Paired comparisons without normality |
| `bartlett_test_fl()` | Test variance homogeneity | Pre-ANOVA assumption checking |
| `levene_test_fl()` | Robust variance equality test | Variance checking with non-normal data |
| `binomial_test_fl()` | Test success rate vs expected | Conversion rate analysis |

### Mathematical Utilities

| Function | Purpose |
|----------|---------|
| `comb_fl()` | Calculate combinations C(n,k) |
| `perm_fl()` | Calculate permutations P(n,k) |
| `factorial_fl()` | Calculate n! |
| `entropy_fl()` | Shannon entropy of probability vectors |
| `percentiles_linear_fl()` | Percentiles with linear interpolation |
| `pairwise_dist_fl()` | Pairwise distances between entities |

---

## Machine Learning Functions

### Clustering

| Function | Algorithm | Best For |
|----------|-----------|----------|
| `kmeans_fl()` | K-Means | Known cluster count, spherical clusters |
| `kmeans_dynamic_fl()` | K-Means (array input) | Features stored as dynamic arrays |
| `dbscan_fl()` | DBSCAN | Unknown clusters, outlier detection, arbitrary shapes |
| `dbscan_dynamic_fl()` | DBSCAN (array input) | Features stored as dynamic arrays |

**Key Parameters:**
- K-Means: `k` (clusters), `features` (column names), `cluster_col` (output)
- DBSCAN: `epsilon` (neighborhood radius), `min_samples` (core point threshold), cluster_id=-1 indicates outliers

### Prediction

| Function | Model Format | Use Case |
|----------|--------------|----------|
| `predict_fl()` | Pickled scikit-learn | Classification/regression with sklearn models |
| `predict_onnx_fl()` | ONNX format | Cross-platform models (PyTorch, TensorFlow) |

**Model Table Schema:** `name` (string), `timestamp` (datetime), `model` (hex-encoded)

### Text Embeddings

| Function | Purpose |
|----------|---------|
| `slm_embeddings_fl()` | Generate embeddings using local SLMs |

---

## Time Series Functions

### Smoothing and Filtering

| Function | Method | Use Case |
|----------|--------|----------|
| `series_moving_avg_fl()` | Simple moving average | Basic noise reduction |
| `series_exp_smoothing_fl()` | Exponential smoothing | Trend estimation |
| `series_dbl_exp_smoothing_fl()` | Double exponential (Holt) | Trend and level tracking |
| `series_fit_lowess_fl()` | LOWESS | Non-linear trend extraction |
| `series_fit_poly_fl()` | Polynomial regression | Polynomial trend fitting |

### Forecasting

| Function | Algorithm | When to Use |
|----------|-----------|-------------|
| `series_fbprophet_forecast_fl()` | Facebook Prophet | Complex seasonality, holidays, changepoints |
| Native `series_decompose_forecast()` | Built-in | Simple forecasting (preferred for basic needs) |

### Time-Weighted Calculations

| Function | Interpolation Method |
|----------|---------------------|
| `time_weighted_avg_fl()` | Fill-forward |
| `time_weighted_avg2_fl()` | Linear |
| `time_weighted_val_fl()` | Linear |
| `time_window_rolling_avg_fl()` | Rolling window |

### Anomaly Detection

| Function | Approach |
|----------|----------|
| `series_clean_anomalies_fl()` | Replace anomalies with interpolated values |
| `series_monthly_decompose_anomalies_fl()` | Monthly seasonality detection |
| `series_uv_anomalies_fl()` | Azure Cognitive Services Univariate API |
| `series_uv_change_points_fl()` | Change point detection via Azure API |
| `series_mv_ee_anomalies_fl()` | Multivariate elliptical envelope |
| `series_mv_if_anomalies_fl()` | Multivariate isolation forest |
| `series_mv_oc_anomalies_fl()` | Multivariate one-class SVM |

### Series Utilities

| Function | Purpose |
|----------|---------|
| `series_rolling_fl()` | Rolling aggregation |
| `series_lag_fl()` | Create lagged versions |
| `series_downsample_fl()` | Downsample by integer factor |
| `series_shapes_fl()` | Detect trends/jumps |
| `series_cosine_similarity_fl()` | Cosine similarity between vectors |
| `series_dot_product_fl()` | Dot product calculation |
| `quantize_fl()` | Bin metric columns |

### PromQL Functions

| Function | Purpose |
|----------|---------|
| `series_metric_fl()` | Retrieve Prometheus-format time series |
| `series_rate_fl()` | Calculate counter metric rate per second |

---

## Text Analytics Functions

### Log Analysis

| Function | Purpose | Output |
|----------|---------|--------|
| `log_reduce_fl()` | Extract common log patterns | Summary table with counts |
| `log_reduce_full_fl()` | Pattern extraction with full assignment | Full table with pattern IDs |
| `log_reduce_train_fl()` | Train reusable pattern model | Model for later prediction |
| `log_reduce_predict_fl()` | Apply trained model | Pattern assignments |

**Key Parameters:** `reduce_col` (log column), `use_logram` (n-gram frequency), `use_drain` (prefix tree), `similarity_th` (Drain threshold)

**Algorithms:**
- **Logram**: N-gram frequency analysis, parallelizable, good for scale
- **Drain**: Prefix tree matching, general-purpose log parsing

### Tokenization

| Function | Purpose |
|----------|---------|
| `tokenize_fl()` | Split semi-structured text into columns |

---

## Graph Analytics Functions

### Security Analysis

| Function | Analysis Type | Output |
|----------|--------------|--------|
| `graph_blast_radius_fl()` | Forward reachability from sources | Reachable targets, weighted scores |
| `graph_exposure_perimeter_fl()` | Reverse blast radius to targets | Exposure metrics |
| `graph_node_centrality_fl()` | Node importance metrics | Degree, betweenness centrality |
| `graph_path_discovery_fl()` | Path finding between endpoints | Valid paths |

**Blast Radius Output:**
- `blastRadiusList`: Reachable target IDs
- `blastRadiusScore`: Count of reachable targets
- `blastRadiusScoreWeighted`: Sum of target weights (criticality)

---

## Cybersecurity Functions

| Function | Detection Type |
|----------|---------------|
| `detect_anomalous_access_cf_fl()` | Unusual user-resource access patterns |
| `detect_anomalous_new_entity_fl()` | New entity appearance detection |
| `detect_anomalous_spike_fl()` | Numeric variable spike detection |

---

## Visualization Functions (Plotly)

| Function | Chart Type | Use Case |
|----------|-----------|----------|
| `plotly_anomaly_fl()` | Interactive anomaly chart | Detailed anomaly exploration |
| `plotly_scatter3d_fl()` | 3D scatter plot | Multi-dimensional visualization |
| `plotly_gauge_fl()` | Gauge/speedometer | KPI dashboards |
| `plotly_graph_fl()` | Interactive graph | Network visualization |

**Prerequisite:** Copy PlotlyTemplate table from help cluster
```kusto
.set PlotlyTemplate <| cluster('help.kusto.windows.net').database('Samples').PlotlyTemplate
```

---

## Utility Functions

| Function | Purpose |
|----------|---------|
| `geoip_fl()` | Geographic info for IP addresses |
| `get_packages_version_fl()` | Python engine and package versions |

---

## Native Alternatives

Before using UDFs, check if native functions suffice:

| UDF | Native Alternative | When Native Works |
|-----|-------------------|-------------------|
| `series_moving_avg_fl()` | `series_fir()` | Simple filtering |
| `two_sample_t_test_fl()` | `welch_test()` | Unequal variance assumption |
| `series_fbprophet_forecast_fl()` | `series_decompose_forecast()` | Basic seasonality, no holidays |
| `kmeans_fl()` | `autocluster` plugin | Pattern mining vs strict clustering |
| `plotly_anomaly_fl()` | `render anomalychart` | Static visualization |

---

## Requirements

- Python plugin enabled on cluster
- Database User permissions (for stored functions)
- External packages via sandbox configuration for Prophet, etc.
- Common packages: scikit-learn, scipy, pandas, numpy, onnxruntime
