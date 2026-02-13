# UDF Functions Library Examples

## Overview

This document provides practical, ready-to-use examples for the most common UDF functions in Azure Data Explorer. Each example includes the complete function definition for query-defined usage.

---

## Statistical Tests Examples

### Example 1: Two-Sample T-Test - A/B Testing

Compare conversion rates between two website variants.

```kusto
// Define the function
let two_sample_t_test_fl = (tbl:(*), data1:string, data2:string, test_statistic:string, p_value:string, equal_var:bool=true)
{
    let kwargs = bag_pack('data1', data1, 'data2', data2, 'test_statistic', test_statistic, 'p_value', p_value, 'equal_var', equal_var);
    let code = ```if 1:
        from scipy import stats
        import pandas
        data1 = kargs["data1"]
        data2 = kargs["data2"]
        test_statistic = kargs["test_statistic"]
        p_value = kargs["p_value"]
        equal_var = kargs["equal_var"]
        def func(row):
            statistics = stats.ttest_ind(row[data1], row[data2], equal_var=equal_var)
            return statistics[0], statistics[1]
        result = df
        result[[test_statistic, p_value]] = df.apply(func, axis=1, result_type = "expand")
    ```;
    tbl
    | evaluate python(typeof(*), code, kwargs)
};
// A/B test example: Compare response times between two server configurations
datatable(test_name:string, config_a_times:dynamic, config_b_times:dynamic) [
    'API Response Test', dynamic([125, 132, 118, 145, 139, 128, 135, 142]), dynamic([98, 105, 112, 95, 108, 102, 99, 115]),
    'Database Query Test', dynamic([45, 52, 48, 55, 47, 51, 49, 53]), dynamic([42, 48, 45, 50, 44, 47, 46, 49]),
    'Page Load Test', dynamic([1.2, 1.4, 1.3, 1.5, 1.35, 1.45, 1.25, 1.38]), dynamic([0.9, 1.1, 0.95, 1.05, 0.98, 1.02, 0.92, 1.08])
]
| extend t_stat = 0.0, p_val = 0.0
| invoke two_sample_t_test_fl('config_a_times', 'config_b_times', 't_stat', 'p_val')
| extend is_significant = p_val < 0.05
| project test_name, t_stat, p_val, is_significant,
    interpretation = iff(is_significant, "Significant difference detected", "No significant difference")
```

---

### Example 2: Kolmogorov-Smirnov Test - Distribution Comparison

Test if data distributions are consistent across time periods.

```kusto
// Define the function
let ks_test_fl = (tbl:(*), data1:string, data2:string, test_statistic:string, p_value:string)
{
    let kwargs = bag_pack('data1', data1, 'data2', data2, 'test_statistic', test_statistic, 'p_value', p_value);
    let code = ```if 1:
        from scipy import stats
        data1 = kargs["data1"]
        data2 = kargs["data2"]
        test_statistic = kargs["test_statistic"]
        p_value = kargs["p_value"]
        def func(row):
            statistics = stats.ks_2samp(row[data1], row[data2])
            return statistics[0], statistics[1]
        result = df
        result[[test_statistic, p_value]] = df.apply(func, axis=1, result_type = "expand")
    ```;
    tbl
    | evaluate python(typeof(*), code, kwargs)
};
// Compare transaction amount distributions between months
datatable(comparison:string, month1_amounts:dynamic, month2_amounts:dynamic) [
    'Jan vs Feb', dynamic([120, 85, 200, 150, 95, 180, 110, 165, 90, 145]), dynamic([125, 90, 195, 155, 100, 175, 115, 160, 95, 140]),
    'Feb vs Mar', dynamic([125, 90, 195, 155, 100, 175, 115, 160, 95, 140]), dynamic([250, 180, 320, 280, 195, 290, 220, 275, 185, 260]),
    'Mar vs Apr', dynamic([250, 180, 320, 280, 195, 290, 220, 275, 185, 260]), dynamic([245, 185, 315, 275, 200, 285, 225, 270, 190, 255])
]
| extend ks_stat = 0.0, p_val = 0.0
| invoke ks_test_fl('month1_amounts', 'month2_amounts', 'ks_stat', 'p_val')
| extend distribution_changed = p_val < 0.05
| project comparison, ks_stat, p_val, distribution_changed,
    alert = iff(distribution_changed, "ALERT: Distribution shift detected", "Normal")
```

---

### Example 3: Normality Test - Data Quality Check

Validate if data follows normal distribution before parametric analysis.

```kusto
// Define the function
let normality_test_fl = (tbl:(*), data:string, test_statistic:string, p_value:string)
{
    let kwargs = bag_pack('data', data, 'test_statistic', test_statistic, 'p_value', p_value);
    let code = ```if 1:
        from scipy import stats
        data = kargs["data"]
        test_statistic = kargs["test_statistic"]
        p_value = kargs["p_value"]
        def func(row):
            statistics = stats.normaltest(row[data])
            return statistics[0], statistics[1]
        result = df
        result[[test_statistic, p_value]] = df.apply(func, axis=1, result_type = "expand")
    ```;
    tbl
    | evaluate python(typeof(*), code, kwargs)
};
// Test normality of sensor readings from different devices
datatable(device_id:string, readings:dynamic) [
    'Sensor-001', dynamic([23.5, 24.1, 23.8, 24.3, 23.9, 24.0, 23.7, 24.2, 23.6, 24.4, 23.8, 24.1]),
    'Sensor-002', dynamic([25.0, 22.0, 28.0, 21.0, 30.0, 20.0, 27.0, 23.0, 29.0, 19.0, 26.0, 24.0]),  // Uniform-ish
    'Sensor-003', dynamic([50.0, 51.2, 50.5, 52.0, 50.8, 51.5, 50.3, 51.8, 50.6, 51.0, 50.9, 51.3])
]
| extend test_stat = 0.0, p_val = 0.0
| invoke normality_test_fl('readings', 'test_stat', 'p_val')
| extend is_normal = p_val > 0.05
| project device_id, test_stat, p_val, is_normal,
    recommendation = iff(is_normal, "Use parametric tests", "Use non-parametric tests")
```

---

## Clustering Examples

### Example 4: K-Means Clustering - Customer Segmentation

Segment customers based on behavior metrics.

```kusto
// Define the function
let kmeans_fl = (tbl:(*), k:int, features:dynamic, cluster_col:string)
{
    let kwargs = bag_pack('k', k, 'features', features, 'cluster_col', cluster_col);
    let code = ```if 1:
        from sklearn.cluster import KMeans
        k = kargs["k"]
        features = kargs["features"]
        cluster_col = kargs["cluster_col"]
        km = KMeans(n_clusters=k)
        df1 = df[features]
        km.fit(df1)
        result = df
        result[cluster_col] = km.labels_
    ```;
    tbl
    | evaluate python(typeof(*), code, kwargs)
};
// Customer segmentation based on purchase behavior
datatable(customer_id:string, total_spend:real, purchase_frequency:real, avg_order_value:real) [
    'C001', 5000.0, 25.0, 200.0,   // High value, frequent
    'C002', 4800.0, 24.0, 200.0,
    'C003', 500.0, 50.0, 10.0,     // Low value, very frequent
    'C004', 450.0, 45.0, 10.0,
    'C005', 2000.0, 4.0, 500.0,    // Medium value, infrequent, high AOV
    'C006', 2200.0, 5.0, 440.0,
    'C007', 100.0, 2.0, 50.0,      // Low value, infrequent
    'C008', 150.0, 3.0, 50.0,
    'C009', 4500.0, 22.0, 205.0,   // High value
    'C010', 480.0, 48.0, 10.0,     // Low value, frequent
    'C011', 1900.0, 4.0, 475.0,    // Medium value, infrequent
    'C012', 120.0, 2.0, 60.0       // Low value, infrequent
]
| invoke kmeans_fl(4, dynamic(["total_spend", "purchase_frequency", "avg_order_value"]), "segment")
| extend segment_name = case(
    segment == 0, "High-Value Loyalists",
    segment == 1, "Bargain Hunters",
    segment == 2, "Big Spenders",
    segment == 3, "Occasional Buyers",
    "Unknown")
| project customer_id, total_spend, purchase_frequency, avg_order_value, segment, segment_name
| order by segment asc
```

---

### Example 5: DBSCAN Clustering - Anomaly Detection

Find clusters and identify outliers in network traffic data.

```kusto
// Define the function
let dbscan_fl = (tbl:(*), features:dynamic, cluster_col:string, epsilon:double, min_samples:int=10,
                 metric:string='minkowski', metric_params:dynamic=dynamic({'p': 2}))
{
    let kwargs = bag_pack('features', features, 'cluster_col', cluster_col, 'epsilon', epsilon, 'min_samples', min_samples,
                          'metric', metric, 'metric_params', metric_params);
    let code = ```if 1:
        from sklearn.cluster import DBSCAN
        from sklearn.preprocessing import StandardScaler
        features = kargs["features"]
        cluster_col = kargs["cluster_col"]
        epsilon = kargs["epsilon"]
        min_samples = kargs["min_samples"]
        metric = kargs["metric"]
        metric_params = kargs["metric_params"]
        df1 = df[features]
        mat = df1.values
        scaler = StandardScaler()
        mat = scaler.fit_transform(mat)
        dbscan = DBSCAN(eps=epsilon, min_samples=min_samples, metric=metric, metric_params=metric_params)
        labels = dbscan.fit_predict(mat)
        result = df
        result[cluster_col] = labels
    ```;
    tbl
    | evaluate python(typeof(*), code, kwargs)
};
// Network traffic anomaly detection
datatable(connection_id:string, bytes_sent:real, bytes_received:real, duration_sec:real) [
    'conn_001', 1024.0, 2048.0, 5.0,      // Normal cluster 1
    'conn_002', 1100.0, 2100.0, 5.5,
    'conn_003', 980.0, 1950.0, 4.8,
    'conn_004', 1050.0, 2000.0, 5.2,
    'conn_005', 50000.0, 100.0, 1.0,      // Anomaly: data exfiltration pattern
    'conn_006', 5000.0, 5500.0, 30.0,     // Normal cluster 2 (larger transfers)
    'conn_007', 5200.0, 5300.0, 32.0,
    'conn_008', 4900.0, 5600.0, 29.0,
    'conn_009', 1020.0, 2040.0, 5.1,      // Normal cluster 1
    'conn_010', 100.0, 80000.0, 2.0,      // Anomaly: large download
    'conn_011', 1080.0, 2080.0, 5.3,      // Normal cluster 1
    'conn_012', 5100.0, 5400.0, 31.0      // Normal cluster 2
]
| extend cluster_id = int(null)
| invoke dbscan_fl(dynamic(["bytes_sent", "bytes_received", "duration_sec"]), "cluster_id", epsilon=0.8, min_samples=3)
| extend is_anomaly = cluster_id == -1
| project connection_id, bytes_sent, bytes_received, duration_sec, cluster_id, is_anomaly,
    status = iff(is_anomaly, "INVESTIGATE", "Normal")
| order by is_anomaly desc
```

---

### Example 6: Visual Clustering Results

Combine clustering with visualization.

```kusto
// Define the function
let kmeans_fl = (tbl:(*), k:int, features:dynamic, cluster_col:string)
{
    let kwargs = bag_pack('k', k, 'features', features, 'cluster_col', cluster_col);
    let code = ```if 1:
        from sklearn.cluster import KMeans
        k = kargs["k"]
        features = kargs["features"]
        cluster_col = kargs["cluster_col"]
        km = KMeans(n_clusters=k)
        df1 = df[features]
        km.fit(df1)
        result = df
        result[cluster_col] = km.labels_
    ```;
    tbl
    | evaluate python(typeof(*), code, kwargs)
};
// Generate sample data with 3 clear clusters
union
    (range i from 1 to 50 step 1 | extend x = rand() * 2 + 1, y = rand() * 2 + 1),    // Cluster near (2, 2)
    (range i from 1 to 50 step 1 | extend x = rand() * 2 + 6, y = rand() * 2 + 1),    // Cluster near (7, 2)
    (range i from 1 to 50 step 1 | extend x = rand() * 2 + 3.5, y = rand() * 2 + 6)   // Cluster near (4.5, 7)
| invoke kmeans_fl(3, dynamic(["x", "y"]), "cluster")
| render scatterchart with (series=cluster, xcolumn=x, ycolumns=y)
```

---

## Prediction Examples

### Example 7: Using Pre-Trained ML Models

Score new data using a stored scikit-learn model.

```kusto
// Define the prediction function
let predict_fl = (samples:(*), models_tbl:(name:string, timestamp:datetime, model:string),
                  model_name:string, features_cols:dynamic, pred_col:string)
{
    let model_str = toscalar(models_tbl | where name == model_name | top 1 by timestamp desc | project model);
    let kwargs = bag_pack('smodel', model_str, 'features_cols', features_cols, 'pred_col', pred_col);
    let code = ```if 1:
        import pickle
        import binascii
        smodel = kargs["smodel"]
        features_cols = kargs["features_cols"]
        pred_col = kargs["pred_col"]
        bmodel = binascii.unhexlify(smodel)
        clf1 = pickle.loads(bmodel)
        df1 = df[features_cols]
        predictions = clf1.predict(df1)
        result = df
        result[pred_col] = pd.DataFrame(predictions, columns=[pred_col])
    ```;
    samples
    | evaluate python(typeof(*), code, kwargs)
};
// Example: Room occupancy prediction (requires pre-stored model)
// First, set up the model table:
// .set ML_Models <| datatable(name:string, timestamp:datetime, model:string) ['Occupancy', now(), '<hex_model>']
//
// Then predict:
// OccupancyDetection
// | where Test == 1
// | extend pred_Occupancy = false
// | invoke predict_fl(ML_Models, 'Occupancy',
//     pack_array('Temperature', 'Humidity', 'Light', 'CO2', 'HumidityRatio'), 'pred_Occupancy')
// | summarize n=count() by Occupancy, pred_Occupancy
```

---

## Log Analysis Examples

### Example 8: Log Pattern Discovery

Extract common patterns from application logs.

```kusto
// Define the function
let log_reduce_fl = (tbl:(*), reduce_col:string, use_logram:bool=true, use_drain:bool=true,
                     custom_regexes:dynamic=dynamic([]), custom_regexes_policy:string='prepend',
                     delimiters:dynamic=dynamic(' '), similarity_th:double=0.5, tree_depth:int=4,
                     trigram_th:int=10, bigram_th:int=15)
{
    let default_regex_table = pack_array(
        '(/|)([0-9]+\\.){3}[0-9]+(:[0-9]+|)(:|)', '<IP>',
        '([0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12})', '<GUID>',
        '(?<=[^A-Za-z0-9])(\\-?\\+?\\d+)(?=[^A-Za-z0-9])|[0-9]+$', '<NUM>');
    let kwargs = bag_pack('reduced_column', reduce_col, 'delimiters', delimiters, 'output_column', 'LogReduce',
                          'parameters_column', '', 'trigram_th', trigram_th, 'bigram_th', bigram_th,
                          'default_regexes', default_regex_table, 'custom_regexes', custom_regexes,
                          'custom_regexes_policy', custom_regexes_policy, 'tree_depth', tree_depth,
                          'similarity_th', similarity_th, 'use_drain', use_drain, 'use_logram', use_logram,
                          'save_regex_tuples_in_output', true, 'regex_tuples_column', 'RegexesColumn',
                          'output_type', 'summary');
    let code = ```if 1:
        from log_cluster import log_reduce
        result = log_reduce.log_reduce(df, kargs)
    ```;
    tbl
    | extend LogReduce = ''
    | evaluate python(typeof(Count:int, LogReduce:string, example:string), code, kwargs)
};
// Example with sample log data
datatable(log_message:string) [
    '2026-02-13 10:15:23 INFO Server started on port 8080',
    '2026-02-13 10:15:24 INFO Server started on port 8081',
    '2026-02-13 10:15:25 INFO Server started on port 8082',
    '2026-02-13 10:16:01 ERROR Connection failed to 192.168.1.100:5432 - timeout after 30s',
    '2026-02-13 10:16:15 ERROR Connection failed to 192.168.1.101:5432 - timeout after 30s',
    '2026-02-13 10:16:30 ERROR Connection failed to 192.168.1.102:5432 - timeout after 30s',
    '2026-02-13 10:17:00 INFO User abc123 logged in from 10.0.0.50',
    '2026-02-13 10:17:05 INFO User def456 logged in from 10.0.0.51',
    '2026-02-13 10:17:10 INFO User ghi789 logged in from 10.0.0.52',
    '2026-02-13 10:18:00 WARN Request 550e8400-e29b-41d4-a716-446655440000 took 5234ms',
    '2026-02-13 10:18:01 WARN Request 6ba7b810-9dad-11d1-80b4-00c04fd430c8 took 4521ms'
]
| invoke log_reduce_fl('log_message')
| order by Count desc
```

---

## Graph Analytics Examples

### Example 9: Blast Radius Analysis for Security

Calculate potential impact of compromised nodes.

```kusto
// Define the function
let graph_blast_radius_fl = (T:(*), sourceIdColumnName:string, targetIdColumnName:string,
                             targetWeightColumnName:string='noWeightsColumn',
                             resultCountLimit:long=100000, listedIdsLimit:long=50)
{
    let paths = (
        T
        | extend sourceId = column_ifexists(sourceIdColumnName, '')
        | extend targetId = column_ifexists(targetIdColumnName, '')
        | extend targetWeight = tolong(column_ifexists(targetWeightColumnName, 0))
    );
    let aggregatedPaths = (
        paths
        | sort by sourceId, targetWeight desc
        | summarize
            blastRadiusList = array_slice(make_set_if(targetId, isnotempty(targetId)), 0, (listedIdsLimit - 1)),
            blastRadiusScore = dcountif(targetId, isnotempty(targetId)),
            blastRadiusScoreWeighted = sum(targetWeight)
          by sourceId
        | extend isBlastRadiusListCapped = (blastRadiusScore > listedIdsLimit)
    );
    aggregatedPaths
    | top resultCountLimit by blastRadiusScore desc
};
// Network access analysis - what can each host reach?
let network_connections = datatable(SourceHost:string, TargetHost:string, TargetCriticality:int) [
    'workstation-001', 'file-server', 3,
    'workstation-001', 'print-server', 1,
    'workstation-001', 'mail-server', 2,
    'workstation-002', 'file-server', 3,
    'workstation-002', 'db-server', 5,
    'admin-laptop', 'domain-controller', 5,
    'admin-laptop', 'db-server', 5,
    'admin-laptop', 'file-server', 3,
    'admin-laptop', 'mail-server', 2,
    'admin-laptop', 'backup-server', 4,
    'web-server', 'db-server', 5,
    'web-server', 'cache-server', 2,
    'db-server', 'backup-server', 4
];
network_connections
| invoke graph_blast_radius_fl('SourceHost', 'TargetHost', 'TargetCriticality')
| extend risk_level = case(
    blastRadiusScoreWeighted >= 15, "CRITICAL",
    blastRadiusScoreWeighted >= 10, "HIGH",
    blastRadiusScoreWeighted >= 5, "MEDIUM",
    "LOW")
| project sourceId, blastRadiusScore, blastRadiusScoreWeighted, risk_level, blastRadiusList
| order by blastRadiusScoreWeighted desc
```

---

## Time Series Examples

### Example 10: Moving Average Smoothing

Smooth noisy time series data.

```kusto
// Define the function (uses native series_fir under the hood)
let series_moving_avg_fl = (y_series:dynamic, n:int, center:bool=false)
{
    series_fir(y_series, repeat(1, n), true, center)
};
// Apply to sample time series
range TimeStamp from datetime(2026-02-01) to datetime(2026-02-10) step 1h
| extend value = rand() * 10 + 50 + (hourofday(TimeStamp) - 12) * 0.5  // Noisy data with daily pattern
| summarize timestamps = make_list(TimeStamp), values = make_list(value)
| extend smoothed_values = series_moving_avg_fl(values, 12, true)  // 12-hour centered moving average
| mv-expand timestamps to typeof(datetime), values to typeof(real), smoothed_values to typeof(real)
| project TimeStamp = timestamps, Original = values, Smoothed = smoothed_values
| render timechart
```

---

### Example 11: Time Series Forecasting with Native Function

For most forecasting needs, use the native function:

```kusto
// Native forecasting (preferred for most cases)
range TimeStamp from datetime(2026-01-01) to datetime(2026-02-01) step 1h
| extend value = 100 + rand() * 10 + sin(2 * pi() * hourofday(TimeStamp) / 24) * 20
| summarize values = make_list(value), timestamps = make_list(TimeStamp)
| extend forecast = series_decompose_forecast(values, 168)  // Forecast 1 week (168 hours)
| mv-expand timestamps, values, forecast
| render timechart
```

---

## Visualization Examples

### Example 12: Interactive Anomaly Chart with Plotly

Create interactive anomaly visualizations.

```kusto
// First, copy the PlotlyTemplate table:
// .set PlotlyTemplate <| cluster('help.kusto.windows.net').database('Samples').PlotlyTemplate

// Define the function
let plotly_anomaly_fl = (tbl:(*), time_col:string, val_col:string, baseline_col:string,
                         time_high_col:string, val_high_col:string, size_high_col:string,
                         time_low_col:string, val_low_col:string, size_low_col:string,
                         chart_title:string='Anomaly chart', series_name:string='Metric', val_name:string='Value')
{
    let anomaly_chart = toscalar(PlotlyTemplate | where name == "anomaly" | project plotly);
    let tbl_ex = tbl
        | extend _timestamp = column_ifexists(time_col, datetime(null)),
                 _values = column_ifexists(val_col, 0.0),
                 _baseline = column_ifexists(baseline_col, 0.0),
                 _high_timestamp = column_ifexists(time_high_col, datetime(null)),
                 _high_values = column_ifexists(val_high_col, 0.0),
                 _high_size = column_ifexists(size_high_col, 1),
                 _low_timestamp = column_ifexists(time_low_col, datetime(null)),
                 _low_values = column_ifexists(val_low_col, 0.0),
                 _low_size = column_ifexists(size_low_col, 1);
    tbl_ex
    | extend plotly = anomaly_chart
    | extend plotly = replace_string(plotly, '$TIME_STAMPS$', tostring(_timestamp))
    | extend plotly = replace_string(plotly, '$SERIES_VALS$', tostring(_values))
    | extend plotly = replace_string(plotly, '$BASELINE_VALS$', tostring(_baseline))
    | extend plotly = replace_string(plotly, '$TIME_STAMPS_HIGH_ANOMALIES$', tostring(_high_timestamp))
    | extend plotly = replace_string(plotly, '$HIGH_ANOMALIES_VALS$', tostring(_high_values))
    | extend plotly = replace_string(plotly, '$HIGH_ANOMALIES_MARKER_SIZE$', tostring(_high_size))
    | extend plotly = replace_string(plotly, '$TIME_STAMPS_LOW_ANOMALIES$', tostring(_low_timestamp))
    | extend plotly = replace_string(plotly, '$LOW_ANOMALIES_VALS$', tostring(_low_values))
    | extend plotly = replace_string(plotly, '$LOW_ANOMALIES_MARKER_SIZE$', tostring(_low_size))
    | extend plotly = replace_string(plotly, '$TITLE$', chart_title)
    | extend plotly = replace_string(plotly, '$SERIES_NAME$', series_name)
    | extend plotly = replace_string(plotly, '$Y_NAME$', val_name)
    | project plotly
};
// Usage example (requires PlotlyTemplate table and sample data):
// demo_make_series2
// | make-series num=avg(num) on TimeStamp from min_t to max_t step dt by sid
// | where sid == 'TS1'
// | extend (anomalies, score, baseline) = series_decompose_anomalies(num, 1.5, -1, 'linefit')
// | ... (process anomalies)
// | invoke plotly_anomaly_fl(...)
// | render plotly
```

---

## Combined Workflow Examples

### Example 13: End-to-End Analysis Pipeline

Combine multiple UDFs in an analysis workflow.

```kusto
// Step 1: Test if data is normally distributed
let normality_test_fl = (tbl:(*), data:string, test_statistic:string, p_value:string)
{
    let kwargs = bag_pack('data', data, 'test_statistic', test_statistic, 'p_value', p_value);
    let code = ```if 1:
        from scipy import stats
        data = kargs["data"]
        test_statistic = kargs["test_statistic"]
        p_value = kargs["p_value"]
        def func(row):
            statistics = stats.normaltest(row[data])
            return statistics[0], statistics[1]
        result = df
        result[[test_statistic, p_value]] = df.apply(func, axis=1, result_type = "expand")
    ```;
    tbl | evaluate python(typeof(*), code, kwargs)
};
// Step 2: Perform appropriate test based on normality
let two_sample_t_test_fl = (tbl:(*), data1:string, data2:string, test_statistic:string, p_value:string, equal_var:bool=true)
{
    let kwargs = bag_pack('data1', data1, 'data2', data2, 'test_statistic', test_statistic, 'p_value', p_value, 'equal_var', equal_var);
    let code = ```if 1:
        from scipy import stats
        data1 = kargs["data1"]
        data2 = kargs["data2"]
        test_statistic = kargs["test_statistic"]
        p_value = kargs["p_value"]
        equal_var = kargs["equal_var"]
        def func(row):
            statistics = stats.ttest_ind(row[data1], row[data2], equal_var=equal_var)
            return statistics[0], statistics[1]
        result = df
        result[[test_statistic, p_value]] = df.apply(func, axis=1, result_type = "expand")
    ```;
    tbl | evaluate python(typeof(*), code, kwargs)
};
// Analysis: Compare performance metrics between two system versions
datatable(test_name:string, version_a:dynamic, version_b:dynamic) [
    'Latency Test', dynamic([45, 48, 52, 47, 50, 49, 46, 51, 48, 47]), dynamic([35, 38, 42, 37, 40, 39, 36, 41, 38, 37])
]
// First check normality of both samples
| mv-expand with_itemindex=idx v = pack_array(version_a, version_b)
| extend sample_name = iff(idx == 0, "version_a", "version_b"), sample_data = v
| extend norm_stat = 0.0, norm_p = 0.0
| invoke normality_test_fl('sample_data', 'norm_stat', 'norm_p')
| summarize
    all_normal = min(norm_p) > 0.05,
    version_a = take_any(iff(sample_name == "version_a", sample_data, dynamic(null))),
    version_b = take_any(iff(sample_name == "version_b", sample_data, dynamic(null)))
  by test_name
// Then perform t-test
| extend t_stat = 0.0, t_p = 0.0
| invoke two_sample_t_test_fl('version_a', 'version_b', 't_stat', 't_p')
| project test_name, all_normal, t_stat, t_p,
    conclusion = iff(t_p < 0.05, "Version B is significantly faster", "No significant difference")
```

---

## Quick Reference: Function Signatures

### Statistical Tests
```kusto
two_sample_t_test_fl(data1, data2, test_statistic, p_value, equal_var=true)
ks_test_fl(data1, data2, test_statistic, p_value)
normality_test_fl(data, test_statistic, p_value)
mann_whitney_u_test_fl(data1, data2, test_statistic, p_value)
wilcoxon_test_fl(data1, data2, test_statistic, p_value)
```

### Clustering
```kusto
kmeans_fl(k, features, cluster_col)
kmeans_dynamic_fl(k, features_col, cluster_col)
dbscan_fl(features, cluster_col, epsilon, min_samples=10, metric='minkowski', metric_params={'p':2})
```

### Prediction
```kusto
predict_fl(models_tbl, model_name, features_cols, pred_col)
predict_onnx_fl(models_tbl, model_name, features_cols, pred_col)
```

### Log Analysis
```kusto
log_reduce_fl(reduce_col, use_logram=true, use_drain=true, similarity_th=0.5)
```

### Graph Analytics
```kusto
graph_blast_radius_fl(sourceIdColumnName, targetIdColumnName, targetWeightColumnName='noWeightsColumn')
```

### Time Series
```kusto
series_moving_avg_fl(y_series, n, center=false)
series_fbprophet_forecast_fl(ts_series, y_series, y_pred_series, points, y_pred_low_series='', y_pred_high_series='')
```

---

## Document Information

- **Generated**: 2026-02-13
- **Source**: ADX Functions Library Examples
- **Phase**: 3 - Feature Deep Dive
