# UDF Functions Library Examples

Ready-to-use examples for Azure Data Explorer UDF functions.

---

## Statistical Tests

### Two-Sample T-Test - A/B Testing

```kusto
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
    tbl
    | evaluate python(typeof(*), code, kwargs)
};
// Compare response times between configurations
datatable(test_name:string, config_a:dynamic, config_b:dynamic) [
    'API Test', dynamic([125, 132, 118, 145, 139]), dynamic([98, 105, 112, 95, 108])
]
| extend t_stat = 0.0, p_val = 0.0
| invoke two_sample_t_test_fl('config_a', 'config_b', 't_stat', 'p_val')
| extend is_significant = p_val < 0.05
```

### Normality Test

```kusto
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
// Test normality of sensor readings
datatable(device_id:string, readings:dynamic) [
    'Sensor-001', dynamic([23.5, 24.1, 23.8, 24.3, 23.9, 24.0, 23.7, 24.2])
]
| extend test_stat = 0.0, p_val = 0.0
| invoke normality_test_fl('readings', 'test_stat', 'p_val')
| extend is_normal = p_val > 0.05
```

---

## Clustering

### K-Means - Customer Segmentation

```kusto
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
// Segment customers
datatable(customer_id:string, total_spend:real, frequency:real) [
    'C001', 5000.0, 25.0,
    'C002', 500.0, 50.0,
    'C003', 2000.0, 4.0,
    'C004', 100.0, 2.0
]
| invoke kmeans_fl(3, dynamic(["total_spend", "frequency"]), "segment")
```

### DBSCAN - Anomaly Detection

```kusto
let dbscan_fl = (tbl:(*), features:dynamic, cluster_col:string, epsilon:double, min_samples:int=5)
{
    let kwargs = bag_pack('features', features, 'cluster_col', cluster_col, 'epsilon', epsilon, 'min_samples', min_samples);
    let code = ```if 1:
        from sklearn.cluster import DBSCAN
        from sklearn.preprocessing import StandardScaler
        features = kargs["features"]
        cluster_col = kargs["cluster_col"]
        epsilon = kargs["epsilon"]
        min_samples = kargs["min_samples"]
        scaler = StandardScaler()
        X = scaler.fit_transform(df[features])
        db = DBSCAN(eps=epsilon, min_samples=min_samples)
        result = df
        result[cluster_col] = db.fit_predict(X)
    ```;
    tbl
    | evaluate python(typeof(*), code, kwargs)
};
// Find anomalies in network traffic
NetworkTraffic
| invoke dbscan_fl(dynamic(["bytes_sent", "bytes_received", "packet_count"]), "cluster", 0.5, 5)
| where cluster == -1  // Outliers
```

---

## Log Analysis

### Log Reduce - Pattern Discovery

```kusto
let log_reduce_fl = (tbl:(*), reduce_col:string, pattern_col:string, regex_flags:string='i')
{
    let kwargs = bag_pack('reduce_col', reduce_col, 'pattern_col', pattern_col, 'regex_flags', regex_flags);
    let code = ```if 1:
        import re
        reduce_col = kargs["reduce_col"]
        pattern_col = kargs["pattern_col"]
        regex_flags = kargs["regex_flags"]

        def reduce_message(msg):
            # Replace numbers with <NUM>
            msg = re.sub(r'\b\d+\b', '<NUM>', msg)
            # Replace IPs with <IP>
            msg = re.sub(r'\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b', '<IP>', msg)
            # Replace GUIDs with <GUID>
            msg = re.sub(r'[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}', '<GUID>', msg, flags=re.I)
            return msg

        result = df
        result[pattern_col] = df[reduce_col].apply(reduce_message)
    ```;
    tbl
    | evaluate python(typeof(*), code, kwargs)
};
// Find common log patterns
AppLogs
| take 10000
| invoke log_reduce_fl('Message', 'Pattern')
| summarize Count = count(), Sample = any(Message) by Pattern
| order by Count desc
```

---

## Time Series

### Moving Average

```kusto
let series_moving_avg_fl = (tbl:(*), y_series:string, y_ma:string, n:int, center:bool=true)
{
    let kwargs = bag_pack('y_series', y_series, 'y_ma', y_ma, 'n', n, 'center', center);
    let code = ```if 1:
        import pandas as pd
        y_series = kargs["y_series"]
        y_ma = kargs["y_ma"]
        n = kargs["n"]
        center = kargs["center"]
        result = df
        result[y_ma] = df[y_series].apply(lambda x: pd.Series(x).rolling(n, center=center).mean().tolist())
    ```;
    tbl
    | evaluate python(typeof(*), code, kwargs)
};
// Smooth noisy metrics
Metrics
| make-series value=avg(Value) on Timestamp step 1h
| invoke series_moving_avg_fl('value', 'smoothed', 5)
```

### Exponential Smoothing

```kusto
let series_exp_smoothing_fl = (tbl:(*), y_series:string, y_smooth:string, alpha:double=0.3)
{
    let kwargs = bag_pack('y_series', y_series, 'y_smooth', y_smooth, 'alpha', alpha);
    let code = ```if 1:
        y_series = kargs["y_series"]
        y_smooth = kargs["y_smooth"]
        alpha = kargs["alpha"]
        def ema(values):
            result = [values[0]]
            for v in values[1:]:
                result.append(alpha * v + (1 - alpha) * result[-1])
            return result
        result = df
        result[y_smooth] = df[y_series].apply(ema)
    ```;
    tbl
    | evaluate python(typeof(*), code, kwargs)
};
```

---

## Graph Analytics

### Blast Radius Analysis

```kusto
let graph_blast_radius_fl = (nodesTable:(id:string), edgesTable:(source:string, target:string), startNode:string, maxHops:int)
{
    let nodes = nodesTable | project id;
    let edges = edgesTable | project source, target;
    range hop from 0 to maxHops step 1
    | mv-apply with_itemindex = i to typeof(int) on (
        // Traverse graph from start node
        let reachable = case(
            i == 0, toscalar(pack_array(startNode)),
            i > 0, // Union of previous hop's neighbors
                (edges
                | where source in (prev_reachable)
                | distinct target
                | summarize make_list(target)))
        | project hop = i, reachable
    )
    | mv-expand reachable to typeof(string)
    | distinct node = reachable
};
// Find all nodes within 3 hops of compromised server
let Nodes = datatable(id:string)['server1', 'server2', 'server3', 'server4'];
let Edges = datatable(source:string, target:string)[
    'server1', 'server2',
    'server2', 'server3',
    'server3', 'server4'
];
invoke graph_blast_radius_fl(Nodes, Edges, 'server1', 3)
```

---

## Visualization

### Plotly Anomaly Chart

```kusto
let plotly_anomaly_fl = (tbl:(*), time_col:string, val_col:string, baseline_col:string, title:string='Anomaly Detection')
{
    let kwargs = bag_pack('time_col', time_col, 'val_col', val_col, 'baseline_col', baseline_col, 'title', title);
    let code = ```if 1:
        import plotly.graph_objects as go
        import plotly.io as pio
        time_col = kargs["time_col"]
        val_col = kargs["val_col"]
        baseline_col = kargs["baseline_col"]
        title = kargs["title"]

        fig = go.Figure()
        fig.add_trace(go.Scatter(x=df[time_col], y=df[val_col], name='Actual', mode='lines'))
        fig.add_trace(go.Scatter(x=df[time_col], y=df[baseline_col], name='Baseline', mode='lines', line=dict(dash='dash')))
        fig.update_layout(title=title)
        result = df
        result['chart'] = pio.to_html(fig, full_html=False)
    ```;
    tbl
    | evaluate python(typeof(*), code, kwargs)
};
```

---

## Combined Workflow: End-to-End Analysis

```kusto
// 1. Get raw metrics
let raw_data = Metrics
| where Timestamp > ago(7d)
| make-series value=avg(Value) on Timestamp step 1h by DeviceId;

// 2. Apply smoothing
let smoothed = raw_data
| invoke series_moving_avg_fl('value', 'smoothed_value', 5);

// 3. Detect anomalies using native function
let with_anomalies = smoothed
| extend (anomalies, score, baseline) = series_decompose_anomalies(smoothed_value);

// 4. Get alert-worthy anomalies
with_anomalies
| mv-expand Timestamp, smoothed_value, anomalies, score
| where anomalies != 0
| project Timestamp, DeviceId, Value = smoothed_value, AnomalyScore = score
| order by abs(AnomalyScore) desc
```
