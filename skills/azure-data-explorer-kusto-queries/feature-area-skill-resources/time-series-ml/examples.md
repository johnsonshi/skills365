# Time Series and ML Examples

## Overview

This document provides practical, actionable examples for time series analysis, anomaly detection, forecasting, and ML workflows in Azure Data Explorer.

---

## Time Series Creation Examples

### Basic Time Series from Event Data

```kusto
// Count events per hour by category
let min_t = datetime(2024-01-01);
let max_t = datetime(2024-01-31);
Events
| make-series EventCount=count() default=0
    on Timestamp from min_t to max_t step 1h
    by Category
| render timechart
```

### Metric Aggregation with Multiple Series

```kusto
// Average response time per service
let min_t = toscalar(Requests | summarize min(Timestamp));
let max_t = toscalar(Requests | summarize max(Timestamp));
Requests
| make-series AvgLatency=avg(ResponseTime), P95Latency=percentile(ResponseTime, 95)
    on Timestamp from min_t to max_t step 5m
    by ServiceName
| render timechart with(title='Service Latency Over Time')
```

### Handling Missing Data

```kusto
// Fill gaps with linear interpolation
SensorData
| make-series Temperature=avg(Value) default=double(null)
    on Timestamp from datetime(2024-01-01) to datetime(2024-01-07) step 1h
    by SensorId
| extend Temperature_filled = series_fill_linear(Temperature)
| render timechart
```

---

## Anomaly Detection Examples

### Basic Anomaly Detection

```kusto
// Detect anomalies in web traffic
let min_t = datetime(2024-01-01);
let max_t = datetime(2024-01-31 23:00);
WebTraffic
| make-series Requests=count() default=0
    on Timestamp from min_t to max_t step 1h
| extend (anomalies, score, baseline) = series_decompose_anomalies(Requests)
| render anomalychart with(anomalycolumns=anomalies, title='Web Traffic Anomalies')
```

### Anomaly Detection with Trend Handling

```kusto
// For data with upward/downward trend
ServiceMetrics
| make-series CPU=avg(CpuPercent) default=0
    on Timestamp from ago(30d) to now() step 1h
    by ServerName
| extend (anomalies, score, baseline) = series_decompose_anomalies(CPU, 1.5, -1, 'linefit')
| mv-expand Timestamp to typeof(datetime), CPU to typeof(double),
            anomalies to typeof(int), score to typeof(double), baseline to typeof(double)
| where anomalies != 0
| project Timestamp, ServerName, CPU, anomalies, score
| order by score desc
```

### Detecting Strong Anomalies Only

```kusto
// Higher threshold for fewer, more significant anomalies
ErrorLogs
| make-series ErrorCount=count() default=0
    on Timestamp from ago(7d) to now() step 1h
    by ErrorType
| extend (anomalies, score, baseline) = series_decompose_anomalies(ErrorCount, 2.5, -1, 'linefit')
| where array_sum(series_abs(anomalies)) > 0  // Only series with anomalies
| render anomalychart with(anomalycolumns=anomalies)
```

### Weekly Seasonality Detection and Anomalies

```kusto
// Specify known weekly pattern (168 hourly bins)
let weekly_period = 24 * 7;  // 168 hours
Transactions
| make-series Amount=sum(Value) default=0
    on Timestamp from ago(30d) to now() step 1h
| extend (anomalies, score, baseline) = series_decompose_anomalies(Amount, 1.5, weekly_period, 'linefit')
| render anomalychart with(anomalycolumns=anomalies, title='Transaction Anomalies (Weekly Pattern)')
```

### Multi-Series Anomaly Detection at Scale

```kusto
// Analyze thousands of time series, find top anomalies
let min_t = toscalar(Telemetry | summarize min(Timestamp));
let max_t = toscalar(Telemetry | summarize max(Timestamp));
Telemetry
| make-series Metric=avg(Value) default=0
    on Timestamp from min_t to max_t step 1h
    by DeviceId, MetricName
| extend (anomalies, score, baseline) = series_decompose_anomalies(Metric)
| extend anomaly_count = array_sum(series_abs(anomalies))
| where anomaly_count > 0
| top 20 by anomaly_count desc
| render timechart
```

---

## Forecasting Examples

### Basic Forecasting

```kusto
// Forecast next 7 days of traffic
let horizon = 7d;
let min_t = datetime(2024-01-01);
let max_t = datetime(2024-01-31) + horizon;  // Extend range for forecast
WebTraffic
| make-series Requests=count() default=0
    on Timestamp from min_t to max_t step 1h
| extend Forecast = series_decompose_forecast(Requests, toint(horizon/1h))
| render timechart with(title='Traffic Forecast')
```

### Forecasting with Trend

```kusto
// Revenue forecast with trend analysis
let horizon = 14d;
let step = 1d;
Sales
| make-series Revenue=sum(Amount) default=0
    on Date from ago(90d) to now()+horizon step step
| extend Forecast = series_decompose_forecast(Revenue, toint(horizon/step), -1, 'linefit')
| render timechart with(title='14-Day Revenue Forecast')
```

### Comparing Multiple Forecast Methods

```kusto
// Compare native forecast vs Prophet
let min_t = datetime(2024-01-01);
let max_t = datetime(2024-02-01);
let horizon = 7d;
let step = 2h;
ServiceMetrics
| make-series Metric=avg(Value) on Timestamp from min_t to max_t+horizon step step by ServiceId
| extend native_forecast = series_decompose_forecast(Metric, toint(horizon/step))
| extend prophet_forecast = dynamic(null)
// Prophet requires series_fbprophet_forecast_fl UDF
// | invoke series_fbprophet_forecast_fl('Timestamp', 'Metric', 'prophet_forecast', toint(horizon/step))
| render timechart
```

---

## Decomposition Examples

### Full Series Decomposition

```kusto
// Decompose into seasonal, trend, residual components
Metrics
| make-series Value=avg(Metric) on Timestamp from ago(30d) to now() step 1h
| extend (baseline, seasonal, trend, residual) = series_decompose(Value, -1, 'linefit')
| render timechart with(ysplit=panels, title='Time Series Decomposition')
```

### Extracting and Analyzing Trend

```kusto
// Analyze trend component separately
Sales
| make-series Revenue=sum(Amount) on Date from ago(365d) to now() step 1d
| extend (baseline, seasonal, trend, residual) = series_decompose(Revenue, -1, 'linefit')
| extend (slope, intercept, rsquare, variance) = series_fit_line(trend)
| project slope, rsquare,
    trend_direction = iff(slope > 0, "Growing", "Declining"),
    daily_change = slope
```

### Period Detection

```kusto
// Automatically detect seasonal periods
WebTraffic
| make-series Requests=count() on Timestamp from ago(30d) to now() step 1h
| project (periods, scores) = series_periods_detect(Requests, 0., 14d/1h, 2)
| mv-expand periods, scores
| extend period_hours = toint(periods), period_days = todouble(periods)/24
| project period_hours, period_days, confidence_score = scores
```

---

## Pattern Analysis Examples

### Autocluster for Failure Analysis

```kusto
// Find common patterns in failures
FailedRequests
| where Timestamp > ago(7d)
| project ErrorCode, ServiceName, Region, Environment, ClientVersion
| evaluate autocluster(0.5)
| project-rename FailurePattern=SegmentId, FailureCount=Count, FailurePercent=Percent
```

### Basket Analysis for Feature Co-occurrence

```kusto
// Find frequently co-occurring features
UserActions
| where Timestamp > ago(30d)
| summarize Features = make_set(FeatureName) by SessionId
| mv-expand Features to typeof(string)
| evaluate basket(0.1)
```

### Diffpatterns for Comparing Time Periods

```kusto
// Compare patterns between normal and incident periods
Telemetry
| extend Period = iff(Timestamp between(datetime(2024-01-15)..datetime(2024-01-16)), "Incident", "Normal")
| project ErrorType, ServiceName, Region, Period
| evaluate diffpatterns(Period, "Normal", "Incident")
```

---

## ML Workflow Examples

### K-Means Clustering

```kusto
// Cluster customers by behavior
let kmeans_fl=(tbl:(*), k:int, features:dynamic, cluster_col:string)
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
    tbl | evaluate python(typeof(*), code, kwargs)
};
CustomerMetrics
| project CustomerId, TotalSpend, OrderCount, AvgOrderValue, DaysSinceLastOrder
| invoke kmeans_fl(4, dynamic(["TotalSpend", "OrderCount", "AvgOrderValue"]), "Segment")
| summarize Count=count(), AvgSpend=avg(TotalSpend), AvgOrders=avg(OrderCount) by Segment
```

### Linear Regression with Python

```kusto
// Build linear regression model
let linear_regression_fl=(tbl:(*), y_col:string, x_cols:dynamic, pred_col:string)
{
    let kwargs = bag_pack('y_col', y_col, 'x_cols', x_cols, 'pred_col', pred_col);
    let code = ```if 1:
from sklearn.linear_model import LinearRegression
y_col = kargs["y_col"]
x_cols = kargs["x_cols"]
pred_col = kargs["pred_col"]
X = df[x_cols].values
y = df[y_col].values
model = LinearRegression()
model.fit(X, y)
result = df
result[pred_col] = model.predict(X)
    ```;
    tbl | evaluate python(typeof(*), code, kwargs)
};
SalesData
| project Revenue, AdSpend, Promotions, Seasonality
| extend PredictedRevenue = 0.0
| invoke linear_regression_fl('Revenue', dynamic(['AdSpend', 'Promotions', 'Seasonality']), 'PredictedRevenue')
| project Revenue, PredictedRevenue, Error = Revenue - PredictedRevenue
```

### ONNX Model Scoring

```kusto
// Score data with pre-trained classification model
let predict_onnx_fl=(samples:(*), models_tbl:(name:string, timestamp:datetime, model:string),
                     model_name:string, features_cols:dynamic, pred_col:string)
{
    let model_str = toscalar(models_tbl | where name == model_name | top 1 by timestamp desc | project model);
    let kwargs = bag_pack('smodel', model_str, 'features_cols', features_cols, 'pred_col', pred_col);
    let code = ```if 1:
import binascii
import onnxruntime as rt
smodel = kargs["smodel"]
features_cols = kargs["features_cols"]
pred_col = kargs["pred_col"]
bmodel = binascii.unhexlify(smodel)
sess = rt.InferenceSession(bmodel)
input_name = sess.get_inputs()[0].name
label_name = sess.get_outputs()[0].name
df1 = df[features_cols]
predictions = sess.run([label_name], {input_name: df1.values.astype(np.float32)})[0]
result = df
result[pred_col] = pd.DataFrame(predictions, columns=[pred_col])
    ```;
    samples | evaluate python(typeof(*), code, kwargs)
};
// Usage
SensorReadings
| extend Prediction = bool(false)
| invoke predict_onnx_fl(ML_Models, 'EquipmentFailureModel',
    dynamic(['Temperature', 'Pressure', 'Vibration', 'Runtime']), 'Prediction')
| where Prediction == true
| project Timestamp, EquipmentId, Prediction
```

---

## Signal Processing Examples

### Moving Average Smoothing

```kusto
// 5-point moving average
Metrics
| make-series Value=avg(Metric) on Timestamp from ago(7d) to now() step 1h
| extend Smoothed = series_fir(Value, repeat(1, 5), true, true)
| render timechart
```

### Change Detection with Differentiation

```kusto
// First derivative to detect sudden changes
Metrics
| make-series Value=avg(Metric) on Timestamp from ago(7d) to now() step 1h
| extend Derivative = series_fir(Value, dynamic([1, -1]), false, false)
| extend AbsChange = series_abs(Derivative)
| render timechart
```

### Frequency Analysis with FFT

```kusto
// Analyze frequency components
Sensors
| make-series Value=avg(Reading) on Timestamp from ago(24h) to now() step 1m
| extend FFT = series_fft(Value)
| extend Magnitude = series_magnitude(FFT)
| mv-expand Timestamp, Value, Magnitude
| project Timestamp, Value=todouble(Value), Magnitude=todouble(Magnitude)
```

---

## Regression Analysis Examples

### Linear Trend Detection

```kusto
// Detect if metric is increasing or decreasing
Metrics
| make-series Value=avg(Metric) on Timestamp from ago(30d) to now() step 1h by ServiceName
| extend (rsquare, slope, variance, rvariance, interception, line_fit) = series_fit_line(Value)
| project ServiceName,
    Trend = iff(slope > 0, "Increasing", "Decreasing"),
    Slope = slope,
    Confidence = rsquare
| order by abs(Slope) desc
```

### Level Change Detection

```kusto
// Detect sudden level shifts
Metrics
| make-series Value=avg(Metric) on Timestamp from ago(7d) to now() step 1h
| extend (rsquare, split_idx, variance, rvariance,
          line_fit, right_rsquare, right_slope, right_intercept, right_variance, right_rvariance, right_line_fit)
    = series_fit_2lines(Value)
| extend SplitTime = Timestamp[split_idx]
| extend LevelChange = right_intercept - (line_fit[0])
| project SplitTime, LevelChange, Confidence = rsquare
```

---

## Real-World Scenario Examples

### Scenario 1: Service Outage Detection

```kusto
// Detect service outages by monitoring error rates
let threshold = 2.0;
let min_t = ago(7d);
let max_t = now();
ServiceLogs
| where Level == "Error"
| make-series ErrorCount=count() default=0 on Timestamp from min_t to max_t step 5m by ServiceName
| extend (anomalies, score, baseline) = series_decompose_anomalies(ErrorCount, threshold)
| extend max_anomaly_score = series_max(score)
| where max_anomaly_score > threshold
| order by max_anomaly_score desc
| project ServiceName, max_anomaly_score
| take 10
```

### Scenario 2: Capacity Planning Forecast

```kusto
// Forecast resource usage for capacity planning
let history = 90d;
let forecast_days = 30;
let step = 1d;
ResourceUsage
| make-series
    CPU=avg(CpuPercent),
    Memory=avg(MemoryPercent),
    Storage=avg(StorageUsedGB)
    on Timestamp from ago(history) to now()+forecast_days step step
    by ClusterName
| extend
    CPU_Forecast = series_decompose_forecast(CPU, forecast_days, -1, 'linefit'),
    Memory_Forecast = series_decompose_forecast(Memory, forecast_days, -1, 'linefit'),
    Storage_Forecast = series_decompose_forecast(Storage, forecast_days, -1, 'linefit')
| mv-expand Timestamp to typeof(datetime),
    CPU to typeof(double), CPU_Forecast to typeof(double),
    Memory to typeof(double), Memory_Forecast to typeof(double),
    Storage to typeof(double), Storage_Forecast to typeof(double)
| where Timestamp > now()  // Only forecast period
| extend Alert = iff(CPU_Forecast > 80 or Memory_Forecast > 90, "CapacityWarning", "OK")
| where Alert == "CapacityWarning"
```

### Scenario 3: IoT Sensor Anomaly Monitoring

```kusto
// Monitor thousands of IoT sensors for anomalies
let min_t = ago(24h);
let max_t = now();
IoTTelemetry
| make-series
    Temperature=avg(Temperature),
    Humidity=avg(Humidity),
    Pressure=avg(Pressure)
    on Timestamp from min_t to max_t step 5m
    by DeviceId, Location
| extend
    (temp_anomalies, temp_score, temp_baseline) = series_decompose_anomalies(Temperature, 2.0),
    (humidity_anomalies, humidity_score, humidity_baseline) = series_decompose_anomalies(Humidity, 2.0),
    (pressure_anomalies, pressure_score, pressure_baseline) = series_decompose_anomalies(Pressure, 2.0)
| extend
    total_anomalies = array_sum(series_abs(temp_anomalies)) +
                      array_sum(series_abs(humidity_anomalies)) +
                      array_sum(series_abs(pressure_anomalies))
| where total_anomalies > 0
| project DeviceId, Location, total_anomalies,
    max_temp_score = series_max(temp_score),
    max_humidity_score = series_max(humidity_score),
    max_pressure_score = series_max(pressure_score)
| order by total_anomalies desc
| take 50
```

### Scenario 4: User Behavior Pattern Analysis

```kusto
// Analyze user activity patterns and detect unusual behavior
UserActivity
| make-series
    Actions=count(),
    UniquePages=dcount(PageUrl)
    on Timestamp from ago(30d) to now() step 1h
    by UserId
| extend
    (periods, scores) = series_periods_detect(Actions, 0., 7d/1h, 2)
| extend
    (anomalies, anomaly_score, baseline) = series_decompose_anomalies(Actions, 2.5)
| extend
    unusual_activity = array_sum(series_abs(anomalies))
| where unusual_activity > 5
| project UserId, unusual_activity, periods, scores
| order by unusual_activity desc
```

---

*Generated from Azure Data Explorer documentation analysis - Phase 3 In-Depth Research*
