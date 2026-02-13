# Feature to Documentation Mapping

## Overview

This document maps Azure Data Explorer (ADX) feature areas to their corresponding documentation files in the `dataexplorer-docs` repository. The mapping is organized by major feature category with key file paths and cross-references.

**Base Path:** `dataexplorer-docs/data-explorer/`

---

## 1. Kusto Query Language (KQL) - Core Query Features

### 1.1 Query Operators (Tabular)

| Feature | Documentation Path | File Count |
|---------|-------------------|------------|
| **Filter Operations** | `kusto/query/where-operator.md`, `kusto/query/filter-operator.md` | 2 |
| **Aggregation (summarize)** | `kusto/query/summarize-operator.md`, `kusto/query/aggregation-functions.md` | 2+ |
| **Join Operations** | `kusto/query/join-operator.md`, `kusto/query/broadcast-join.md`, `kusto/query/join-cross-cluster.md` | 3+ |
| **Union Operations** | `kusto/query/union-operator.md` | 1 |
| **Extend/Project** | `kusto/query/extend-operator.md`, `kusto/query/project-operator.md`, `kusto/query/project-away-operator.md`, `kusto/query/project-keep-operator.md`, `kusto/query/project-rename-operator.md`, `kusto/query/project-reorder-operator.md` | 6 |
| **Sort/Order** | `kusto/query/sort-operator.md`, `kusto/query/order-operator.md`, `kusto/query/top-operator.md`, `kusto/query/top-nested-operator.md` | 4 |
| **Distinct/Sample** | `kusto/query/distinct-operator.md`, `kusto/query/sample-operator.md`, `kusto/query/sample-distinct-operator.md` | 3 |
| **Take/Limit** | `kusto/query/take-operator.md`, `kusto/query/limit-operator.md` | 2 |
| **Render (Visualization)** | `kusto/query/render-operator.md` | 1 |
| **Parse Operations** | `kusto/query/parse-operator.md`, `kusto/query/parse-where-operator.md`, `kusto/query/parse-kv-operator.md` | 3 |
| **Pivot/Evaluate** | `kusto/query/pivot-plugin.md`, `kusto/query/evaluate-operator.md` | 2 |
| **Lookup** | `kusto/query/lookup-operator.md` | 1 |
| **Serialize** | `kusto/query/serialize-operator.md` | 1 |

**Key Files:**
- `kusto/query/index.md` - Main KQL query reference index
- `kusto/query/best-practices.md` - Query optimization best practices
- `kusto/query/batches.md` - Query batching

### 1.2 Scalar Functions

| Category | Sample Files | Approx Count |
|----------|--------------|--------------|
| **String Functions** | `kusto/query/strcat-function.md`, `kusto/query/substring-function.md`, `kusto/query/replace-function.md`, `kusto/query/split-function.md`, `kusto/query/trim-function.md` | 50+ |
| **DateTime Functions** | `kusto/query/datetime-add-function.md`, `kusto/query/datetime-diff-function.md`, `kusto/query/format-datetime-function.md`, `kusto/query/ago-function.md`, `kusto/query/now-function.md` | 30+ |
| **Mathematical Functions** | `kusto/query/abs-function.md`, `kusto/query/round-function.md`, `kusto/query/ceiling-function.md`, `kusto/query/floor-function.md`, `kusto/query/pow-function.md`, `kusto/query/log-function.md` | 40+ |
| **Array/Dynamic Functions** | `kusto/query/array-concat-function.md`, `kusto/query/array-length-function.md`, `kusto/query/array-slice-function.md`, `kusto/query/bag-keys-function.md`, `kusto/query/bag-merge-function.md` | 30+ |
| **Conversion Functions** | `kusto/query/tostring-function.md`, `kusto/query/toint-function.md`, `kusto/query/todatetime-function.md`, `kusto/query/tobool-function.md` | 15+ |
| **Conditional Functions** | `kusto/query/case-function.md`, `kusto/query/iff-function.md`, `kusto/query/coalesce-function.md`, `kusto/query/isnull-function.md` | 10+ |
| **Hash Functions** | `kusto/query/hash-function.md`, `kusto/query/hash-sha256-function.md`, `kusto/query/hash-sha1-function.md` | 5+ |
| **Binary Functions** | `kusto/query/binary-and-function.md`, `kusto/query/binary-or-function.md`, `kusto/query/binary-xor-function.md`, `kusto/query/binary-not-function.md` | 10+ |

### 1.3 Aggregation Functions

| Feature | Documentation Path |
|---------|-------------------|
| **Basic Aggregations** | `kusto/query/count-aggregation-function.md`, `kusto/query/sum-aggregation-function.md`, `kusto/query/avg-aggregation-function.md`, `kusto/query/min-aggregation-function.md`, `kusto/query/max-aggregation-function.md` |
| **Distinct Count** | `kusto/query/dcount-aggregation-function.md`, `kusto/query/dcountif-aggregation-function.md`, `kusto/query/dcount-hll-function.md` |
| **Percentiles** | `kusto/query/percentile-aggregation-function.md`, `kusto/query/percentiles-aggregation-function.md`, `kusto/query/percentile-tdigest-function.md` |
| **Make Functions** | `kusto/query/make-list-aggregation-function.md`, `kusto/query/make-set-aggregation-function.md`, `kusto/query/make-bag-aggregation-function.md` |
| **Arg Min/Max** | `kusto/query/arg-max-aggregation-function.md`, `kusto/query/arg-min-aggregation-function.md` |
| **Statistical** | `kusto/query/stdev-aggregation-function.md`, `kusto/query/variance-aggregation-function.md` |

### 1.4 Data Types

| Type | Documentation Path |
|------|-------------------|
| **Overview** | `kusto/query/scalar-data-types/index.md` |
| **Boolean** | `kusto/query/scalar-data-types/bool.md` |
| **DateTime** | `kusto/query/scalar-data-types/datetime.md` |
| **Decimal** | `kusto/query/scalar-data-types/decimal.md` |
| **Dynamic** | `kusto/query/scalar-data-types/dynamic.md` |
| **GUID** | `kusto/query/scalar-data-types/guid.md` |
| **Int/Long** | `kusto/query/scalar-data-types/int.md`, `kusto/query/scalar-data-types/long.md` |
| **Real** | `kusto/query/scalar-data-types/real.md` |
| **String** | `kusto/query/scalar-data-types/string.md` |
| **Timespan** | `kusto/query/scalar-data-types/timespan.md` |
| **Null Values** | `kusto/query/scalar-data-types/null-values.md` |

---

## 2. Time Series Analysis & Machine Learning

### 2.1 Built-in Time Series Functions

| Feature | Documentation Path |
|---------|-------------------|
| **Series Creation** | `kusto/query/make-series-operator.md` |
| **Decomposition** | `kusto/query/series-decompose-function.md`, `kusto/query/series-decompose-anomalies-function.md`, `kusto/query/series-decompose-forecast-function.md` |
| **Seasonal Patterns** | `kusto/query/series-seasonal-function.md`, `kusto/query/series-periods-detect-function.md`, `kusto/query/series-periods-validate-function.md` |
| **Statistical Analysis** | `kusto/query/series-stats-function.md`, `kusto/query/series-stats-dynamic-function.md`, `kusto/query/series-outliers-function.md` |
| **Fitting/Regression** | `kusto/query/series-fit-line-function.md`, `kusto/query/series-fit-2lines-function.md`, `kusto/query/series-fit-line-dynamic-function.md`, `kusto/query/series-fit-2lines-dynamic-function.md` |
| **Operations** | `kusto/query/series-add-function.md`, `kusto/query/series-subtract-function.md`, `kusto/query/series-multiply-function.md`, `kusto/query/series-divide-function.md`, `kusto/query/series-fir-function.md`, `kusto/query/series-iir-function.md` |
| **Interpolation** | `kusto/query/series-fill-backward-function.md`, `kusto/query/series-fill-const-function.md`, `kusto/query/series-fill-forward-function.md`, `kusto/query/series-fill-linear-function.md` |

### 2.2 Anomaly Detection

| Feature | Documentation Path |
|---------|-------------------|
| **Overview** | `kusto/query/anomaly-detection.md` |
| **Diagnosis** | `kusto/query/anomaly-diagnosis.md` |
| **Series Anomalies** | `kusto/query/series-decompose-anomalies-function.md` |

### 2.3 Machine Learning Plugins

| Feature | Documentation Path |
|---------|-------------------|
| **Autocluster** | `kusto/query/autocluster-plugin.md` |
| **Basket** | `kusto/query/basket-plugin.md` |
| **Diffpatterns** | `kusto/query/diffpatterns-plugin.md`, `kusto/query/diffpatterns-text-plugin.md` |
| **Predict** | `kusto/query/predict-function.md` |
| **Narrow** | `kusto/query/narrow-plugin.md` |

### 2.4 Functions Library (UDFs for ML/Statistics)

| Category | Key Files |
|----------|-----------|
| **Overview** | `kusto/functions-library/functions-library.md` |
| **Clustering** | `kusto/functions-library/kmeans-fl.md`, `kusto/functions-library/kmeans-dynamic-fl.md`, `kusto/functions-library/dbscan-fl.md`, `kusto/functions-library/dbscan-dynamic-fl.md` |
| **Statistical Tests** | `kusto/functions-library/bartlett-test-fl.md`, `kusto/functions-library/binomial-test-fl.md`, `kusto/functions-library/ks-test-fl.md`, `kusto/functions-library/levene-test-fl.md`, `kusto/functions-library/mann-whitney-u-test-fl.md`, `kusto/functions-library/normality-test-fl.md`, `kusto/functions-library/two-sample-t-test-fl.md`, `kusto/functions-library/wilcoxon-test-fl.md` |
| **Time Series** | `kusto/functions-library/series-exp-smoothing-fl.md`, `kusto/functions-library/series-dbl-exp-smoothing-fl.md`, `kusto/functions-library/series-moving-avg-fl.md`, `kusto/functions-library/series-rolling-fl.md`, `kusto/functions-library/series-fbprophet-forecast-fl.md` |
| **Anomaly Detection** | `kusto/functions-library/series-uv-anomalies-fl.md`, `kusto/functions-library/series-uv-change-points-fl.md`, `kusto/functions-library/series-mv-ee-anomalies-fl.md`, `kusto/functions-library/series-mv-if-anomalies-fl.md`, `kusto/functions-library/series-mv-oc-anomalies-fl.md`, `kusto/functions-library/detect-anomalous-spike-fl.md`, `kusto/functions-library/detect-anomalous-new-entity-fl.md` |
| **ML Prediction** | `kusto/functions-library/predict-fl.md`, `kusto/functions-library/predict-onnx-fl.md` |
| **Text Analytics** | `kusto/functions-library/log-reduce-fl.md`, `kusto/functions-library/log-reduce-full-fl.md`, `kusto/functions-library/log-reduce-train-fl.md`, `kusto/functions-library/log-reduce-predict-fl.md`, `kusto/functions-library/tokenize-fl.md` |
| **Visualization (Plotly)** | `kusto/functions-library/plotly-anomaly-fl.md`, `kusto/functions-library/plotly-gauge-fl.md`, `kusto/functions-library/plotly-graph-fl.md`, `kusto/functions-library/plotly-scatter3d-fl.md` |

---

## 3. Graph Analytics

### 3.1 Graph Operators

| Feature | Documentation Path |
|---------|-------------------|
| **Graph Creation** | `kusto/query/make-graph-operator.md` |
| **Graph Matching** | `kusto/query/graph-match-operator.md` |
| **Graph to Table** | `kusto/query/graph-to-table-operator.md` |
| **Graph Functions** | `kusto/query/all-graph-function.md`, `kusto/query/any-graph-function.md`, `kusto/query/inner-graph-function.md` |

### 3.2 Graph Management

| Feature | Documentation Path |
|---------|-------------------|
| **Overview** | `kusto/management/graph/` (directory) |
| **Graph Commands** | `kusto/management/graph/` |

### 3.3 Graph Functions Library

| Feature | Documentation Path |
|---------|-------------------|
| **Blast Radius** | `kusto/functions-library/graph-blast-radius-fl.md` |
| **Exposure Perimeter** | `kusto/functions-library/graph-exposure-perimeter-fl.md` |
| **Node Centrality** | `kusto/functions-library/graph-node-centrality-fl.md` |
| **Path Discovery** | `kusto/functions-library/graph-path-discovery-fl.md` |

---

## 4. Geospatial Analysis

### 4.1 Geospatial Functions

| Category | Key Files |
|----------|-----------|
| **Point Functions** | `kusto/query/geo-point-in-circle-function.md`, `kusto/query/geo-point-in-polygon-function.md`, `kusto/query/geo-point-to-geohash-function.md`, `kusto/query/geo-point-to-h3cell-function.md`, `kusto/query/geo-point-to-s2cell-function.md` |
| **Distance Functions** | `kusto/query/geo-distance-2points-function.md`, `kusto/query/geo-distance-point-to-line-function.md`, `kusto/query/geo-distance-point-to-polygon-function.md` |
| **Polygon Functions** | `kusto/query/geo-polygon-area-function.md`, `kusto/query/geo-polygon-centroid-function.md`, `kusto/query/geo-polygon-densify-function.md`, `kusto/query/geo-polygon-perimeter-function.md`, `kusto/query/geo-polygon-simplify-function.md` |
| **Line Functions** | `kusto/query/geo-line-buffer-function.md`, `kusto/query/geo-line-centroid-function.md`, `kusto/query/geo-line-densify-function.md`, `kusto/query/geo-line-length-function.md`, `kusto/query/geo-line-simplify-function.md` |
| **H3 Functions** | `kusto/query/geo-h3cell-*.md` (multiple files) |
| **S2 Functions** | `kusto/query/geo-s2cell-*.md` (multiple files) |
| **Geohash Functions** | `kusto/query/geo-geohash-*.md` (multiple files) |
| **Intersection** | `kusto/query/geo-intersection-*.md` (multiple files) |

---

## 5. Management Commands (Control Plane)

### 5.1 Schema Management

| Feature | Documentation Path |
|---------|-------------------|
| **Tables** | `kusto/management/create-table-command.md`, `kusto/management/alter-table-command.md`, `kusto/management/drop-table-command.md`, `kusto/management/show-tables.md`, `kusto/management/rename-table-command.md` |
| **Columns** | `kusto/management/columns.md`, `kusto/management/alter-column.md`, `kusto/management/alter-column-docstrings.md`, `kusto/management/drop-column.md`, `kusto/management/rename-column.md` |
| **Functions** | `kusto/management/create-function.md`, `kusto/management/alter-function.md`, `kusto/management/drop-function.md`, `kusto/management/show-function.md`, `kusto/management/create-alter-function.md` |
| **Ingestion Mappings** | `kusto/management/create-ingestion-mapping-command.md`, `kusto/management/alter-ingestion-mapping-command.md`, `kusto/management/drop-ingestion-mapping-command.md`, `kusto/management/show-ingestion-mapping-command.md` |
| **Mapping Types** | `kusto/management/json-mapping.md`, `kusto/management/csv-mapping.md`, `kusto/management/avro-mapping.md`, `kusto/management/parquet-mapping.md`, `kusto/management/orc-mapping.md`, `kusto/management/w3clogfile-mapping.md` |

### 5.2 Policies (30+ Policy Types)

| Policy Category | Key Files |
|-----------------|-----------|
| **Caching** | `kusto/management/cache-policy.md`, `kusto/management/alter-table-cache-policy-command.md`, `kusto/management/alter-database-cache-policy-command.md`, `kusto/management/alter-cluster-cache-policy-command.md` |
| **Retention** | `kusto/management/retention-policy.md`, `kusto/management/alter-table-retention-policy-command.md`, `kusto/management/alter-database-retention-policy-command.md` |
| **Partitioning** | `kusto/management/partitioning-policy.md`, `kusto/management/alter-table-partitioning-policy-command.md` |
| **Merge** | `kusto/management/merge-policy.md`, `kusto/management/alter-table-merge-policy-command.md`, `kusto/management/alter-database-merge-policy-command.md` |
| **Sharding** | `kusto/management/sharding-policy.md`, `kusto/management/alter-table-sharding-policy-command.md`, `kusto/management/alter-database-sharding-policy-command.md` |
| **Ingestion Batching** | `kusto/management/batching-policy.md`, `kusto/management/alter-table-ingestion-batching-policy.md`, `kusto/management/alter-database-ingestion-batching-policy.md` |
| **Streaming Ingestion** | `kusto/management/streaming-ingestion-policy.md`, `kusto/management/alter-table-streaming-ingestion-policy-command.md`, `kusto/management/alter-database-streaming-ingestion-policy-command.md` |
| **Update Policy** | `kusto/management/update-policy.md`, `kusto/management/alter-table-update-policy-command.md` |
| **Row Level Security** | `kusto/management/row-level-security-policy.md`, `kusto/management/alter-table-row-level-security-policy-command.md` |
| **Restricted View Access** | `kusto/management/restricted-view-access-policy.md`, `kusto/management/alter-table-restricted-view-access-policy-command.md` |
| **Auto Delete** | `kusto/management/auto-delete-policy.md`, `kusto/management/alter-auto-delete-policy-command.md` |
| **Callout** | `kusto/management/callout-policy.md`, `kusto/management/alter-callout-policy-command.md` |
| **Capacity** | `kusto/management/capacity-policy.md`, `kusto/management/alter-capacity-policy-command.md` |
| **Sandbox** | `kusto/management/sandbox-policy.md`, `kusto/management/alter-cluster-sandbox-policy-command.md` |
| **Encoding** | `kusto/management/encoding-policy.md`, `kusto/management/alter-encoding-policy.md` |
| **Extent Tags Retention** | `kusto/management/extent-tags-retention-policy.md`, `kusto/management/alter-extent-tags-retention-policy.md` |
| **Row Order** | `kusto/management/row-order-policy.md`, `kusto/management/alter-table-row-order-policy-command.md` |
| **Mirroring** | `kusto/management/mirroring-policy.md`, `kusto/management/alter-merge-mirroring-policy-command.md` |
| **Query Acceleration** | `kusto/management/query-acceleration-policy.md`, `kusto/management/alter-query-acceleration-policy-command.md` |

### 5.3 Materialized Views

| Feature | Documentation Path |
|---------|-------------------|
| **Overview** | `kusto/management/materialized-views/materialized-view-overview.md` |
| **Use Cases** | `kusto/management/materialized-views/materialized-view-use-cases.md` |
| **Create** | `kusto/management/materialized-views/materialized-view-create.md`, `kusto/management/materialized-views/materialized-view-create-or-alter.md` |
| **Alter** | `kusto/management/materialized-views/materialized-view-alter.md`, `kusto/management/materialized-views/materialized-view-alter-lookback.md`, `kusto/management/materialized-views/materialized-view-alter-autoupdateschema.md` |
| **Drop** | `kusto/management/materialized-views/materialized-view-drop.md` |
| **Show** | `kusto/management/materialized-views/materialized-view-show-command.md`, `kusto/management/materialized-views/materialized-view-show-details-command.md`, `kusto/management/materialized-views/materialized-view-show-extents-command.md`, `kusto/management/materialized-views/materialized-view-show-failures-command.md`, `kusto/management/materialized-views/materialized-view-show-schema-command.md` |
| **Policies** | `kusto/management/materialized-views/materialized-view-policies.md` |
| **Enable/Disable** | `kusto/management/materialized-views/materialized-view-enable-disable.md` |
| **Monitoring** | `kusto/management/materialized-views/materialized-views-monitoring.md` |
| **Limitations** | `kusto/management/materialized-views/materialized-views-limitations.md` |

### 5.4 Data Export

| Feature | Documentation Path |
|---------|-------------------|
| **Overview** | `kusto/management/data-export/index.md` |
| **Continuous Export** | `kusto/management/data-export/continuous-data-export.md`, `kusto/management/data-export/create-alter-continuous.md`, `kusto/management/data-export/drop-continuous-export.md`, `kusto/management/data-export/disable-enable-continuous.md` |
| **Export to Storage** | `kusto/management/data-export/export-data-to-storage.md` |
| **Export to External Table** | `kusto/management/data-export/export-data-to-an-external-table.md` |
| **Export to SQL** | `kusto/management/data-export/export-data-to-sql.md` |
| **Show Commands** | `kusto/management/data-export/show-continuous-export.md`, `kusto/management/data-export/show-continuous-artifacts.md`, `kusto/management/data-export/show-continuous-failures.md` |
| **Managed Identity** | `kusto/management/data-export/continuous-export-with-managed-identity.md` |

### 5.5 External Tables

| Feature | Documentation Path |
|---------|-------------------|
| **Azure Storage** | `kusto/management/external-tables-azurestorage-azuredatalake.md`, `external-azure-storage-tables-query.md` |
| **Azure SQL** | `kusto/management/external-sql-tables.md` |
| **Commands** | `kusto/management/external-table-show-command.md`, `kusto/management/external-table-drop-command.md` |
| **Managed Identities** | `external-tables-managed-identities.md` |

### 5.6 Workload Management

| Feature | Documentation Path |
|---------|-------------------|
| **Workload Groups** | `kusto/management/workload-groups.md`, `kusto/management/create-or-alter-workload-group-command.md`, `kusto/management/alter-merge-workload-group-command.md` |
| **Request Classification** | `kusto/management/request-classification-policy.md`, `kusto/management/alter-cluster-policy-request-classification-command.md` |
| **Query Limits** | `kusto/management/request-limits-policy.md`, `kusto/management/request-rate-limit-policy.md`, `kusto/management/request-queuing-policy.md` |

### 5.7 Security & Access Control

| Feature | Documentation Path |
|---------|-------------------|
| **Overview** | `kusto/access-control/index.md` |
| **Role-Based Access Control** | `kusto/access-control/role-based-access-control.md` |
| **Microsoft Entra ID** | `kusto/access-control/provision-entra-id-app.md` |
| **Principals** | `add-cluster-principal.md`, `add-database-principal.md`, `kusto/management/security-roles.md` |
| **Row Level Security** | `kusto/management/row-level-security-policy.md` |

### 5.8 Data Management

| Feature | Documentation Path |
|---------|-------------------|
| **Data Purge** | `kusto/concepts/data-purge.md`, `data-purge-portal.md` |
| **Soft Delete** | `kusto/concepts/data-soft-delete.md` |
| **Delete Data** | `kusto/concepts/delete-data.md` |
| **Extents (Shards)** | `kusto/management/extents-overview.md`, `kusto/management/show-extents.md`, `kusto/management/alter-extent.md`, `kusto/management/drop-extents.md`, `kusto/management/move-extents.md`, `kusto/management/merge-extents.md` |

---

## 6. Data Ingestion

### 6.1 Ingestion Overview

| Feature | Documentation Path |
|---------|-------------------|
| **Overview** | `ingest-data-overview.md`, `how-it-works.md` |
| **Ingestion Properties** | `kusto/management/ingestion-properties.md` |
| **Streaming Ingestion** | `kusto/management/streaming-ingestion-policy.md`, `ingest-data-streaming.md` |
| **Queued Ingestion** | `kusto/management/batching-policy.md` |

### 6.2 Data Connectors

| Connector | Documentation Path |
|-----------|-------------------|
| **Event Hubs** | `create-event-hubs-connection.md`, `create-event-hubs-connection-sdk.md` |
| **Event Grid** | `create-event-grid-connection.md`, `create-event-grid-connection-sdk.md` |
| **IoT Hub** | `create-iot-hub-connection.md`, `create-iot-hub-connection-sdk.md` |
| **Kafka** | `ingest-data-kafka.md`, `includes/cross-repo/ingest-data-kafka*.md` |
| **Apache Flink** | `includes/cross-repo/ingest-data-flink*.md` |
| **Spark** | `spark-connector.md`, `includes/cross-repo/ingest-data-spark.md` |
| **Fluent Bit** | `fluent-bit.md`, `includes/cross-repo/fluent-bit*.md` |
| **Logstash** | `ingest-data-logstash.md` |
| **Telegraf** | `ingest-data-telegraf.md` |
| **Cribl** | `includes/cross-repo/ingest-data-cribl*.md` |
| **Serilog** | `includes/cross-repo/ingest-data-serilog*.md` |
| **NLog** | `includes/cross-repo/ingest-nlog-sink*.md` |
| **Log4j2** | `apache-log4j2-connector.md`, `includes/cross-repo/ingest-data-log4j2.md` |

### 6.3 Get Data Wizard

| Feature | Documentation Path |
|---------|-------------------|
| **From File** | `get-data-file.md` |
| **From Storage** | `get-data-storage.md` |
| **From Amazon S3** | `get-data-amazon-s3.md` |
| **Create Table Wizard** | `create-table-wizard.md` |

---

## 7. APIs & SDKs

### 7.1 API Overview

| Feature | Documentation Path |
|---------|-------------------|
| **Overview** | `kusto/api/index.md` |
| **Client Libraries** | `kusto/api/client-libraries.md` |

### 7.2 REST API

| Feature | Documentation Path |
|---------|-------------------|
| **Overview** | `kusto/api/rest/index.md` |
| **Authentication** | `kusto/api/rest/authentication.md`, `kusto/api/rest/authenticate-with-msal.md` |
| **Request/Response** | `kusto/api/rest/request.md`, `kusto/api/rest/response.md`, `kusto/api/rest/response-v2.md` |
| **Request Properties** | `kusto/api/rest/request-properties.md` |
| **Streaming Ingest** | `kusto/api/rest/streaming-ingest.md` |
| **T-SQL** | `kusto/api/rest/t-sql.md` |
| **Deep Linking** | `kusto/api/rest/deeplink.md` |

### 7.3 SDK Documentation by Language

| SDK | Documentation Path |
|-----|-------------------|
| **.NET SDK** | `kusto/api/netfx/about-the-sdk.md`, `kusto/api/netfx/about-kusto-data.md`, `kusto/api/netfx/about-kusto-ingest.md`, `kusto/api/netfx/about-kusto-language.md`, `kusto/api/netfx/kusto-data-best-practices.md`, `kusto/api/netfx/kusto-ingest-best-practices.md`, `kusto/api/netfx/kusto-ingest-client-*.md` |
| **Python SDK** | `kusto/api/python/kusto-python-client-library.md` |
| **Java SDK** | `kusto/api/java/kusto-java-client-library.md` |
| **Node.js SDK** | `kusto/api/node/kusto-node-client-library.md` |
| **Go SDK** | `kusto/api/golang/kusto-golang-client-library.md` |
| **R SDK** | `kusto/api/r/kusto-r-client-library.md` |
| **PowerShell** | `kusto/api/powershell/powershell.md` |

### 7.4 Connection Strings

| Feature | Documentation Path |
|---------|-------------------|
| **Overview** | `kusto/api/connection-strings/index.md` |
| **Kusto Connection** | `kusto/api/connection-strings/kusto.md` |
| **Storage Connection** | `kusto/api/connection-strings/storage-connection-strings.md` |
| **SQL Connection** | `kusto/api/connection-strings/sql-connection-strings.md` |
| **SAS Token Generation** | `kusto/api/connection-strings/generate-sas-token.md` |

### 7.5 Getting Started Tutorials

| Tutorial | Documentation Path |
|----------|-------------------|
| **App Setup** | `kusto/api/get-started/app-set-up.md` |
| **Hello Kusto** | `kusto/api/get-started/app-hello-kusto.md` |
| **Basic Query** | `kusto/api/get-started/app-basic-query.md` |
| **Management Commands** | `kusto/api/get-started/app-management-commands.md` |
| **Queued Ingestion** | `kusto/api/get-started/app-queued-ingestion.md` |
| **Streaming Ingestion** | `kusto/api/get-started/app-managed-streaming-ingest.md` |
| **Authentication** | `kusto/api/get-started/app-authentication-methods.md` |
| **Network Restrictions** | `kusto/api/get-started/app-client-network-restrictions.md` |

---

## 8. Visualization & Dashboards

### 8.1 ADX Dashboards

| Feature | Documentation Path |
|---------|-------------------|
| **Overview** | `azure-data-explorer-dashboards.md` |
| **Sharing** | `azure-data-explorer-dashboard-share.md` |
| **Parameters** | `dashboard-parameters.md` |
| **Visuals** | `dashboard-visuals.md`, `dashboard-customize-visuals.md` |
| **Conditional Formatting** | `dashboard-conditional-formatting.md` |
| **Explore Data** | `dashboard-explore-data.md` |
| **Query Visualization** | `add-query-visualization.md` |
| **Base Query** | `base-query.md` |

### 8.2 External Visualization Tools

| Tool | Documentation Path |
|------|-------------------|
| **Power BI** | `power-bi-*.md` (multiple files) |
| **Grafana** | `grafana.md` |
| **Excel** | `excel.md` |
| **Tableau** | `tableau.md` |
| **Sisense** | `sisense.md` |
| **Redash** | `redash.md` |

---

## 9. Tools & Utilities

### 9.1 Kusto Tools

| Tool | Documentation Path |
|------|-------------------|
| **Kusto.Explorer** | `kusto/tools/kusto-explorer.md`, `kusto/tools/kusto-explorer-using.md`, `kusto/tools/kusto-explorer-code-features.md`, `kusto/tools/kusto-explorer-options.md`, `kusto/tools/kusto-explorer-shortcuts.md`, `kusto/tools/kusto-explorer-troubleshooting.md` |
| **Kusto CLI** | `kusto/tools/kusto-cli.md` |
| **Avrotize** | `kusto/tools/avrotize.md` |

### 9.2 Monaco Integration

| Feature | Documentation Path |
|---------|-------------------|
| **Overview** | `kusto/api/monaco/monaco-overview.md` |
| **Monaco-Kusto** | `kusto/api/monaco/monaco-kusto.md` |
| **Host in iframe** | `kusto/api/monaco/host-web-ux-in-iframe.md` |

---

## 10. Cluster & Database Management

### 10.1 Cluster Operations

| Feature | Documentation Path |
|---------|-------------------|
| **Create Cluster** | `create-cluster-database.md`, `create-cluster-and-database.md` |
| **Delete Cluster** | `delete-cluster.md` |
| **Check Health** | `check-cluster-health.md` |
| **Auto Stop** | `auto-stop-clusters.md` |
| **Scaling** | `manage-cluster-horizontal-scaling.md`, `manage-cluster-vertical-scaling.md` |
| **Follower** | `follower.md`, `kusto/management/cluster-follower.md` |

### 10.2 Database Operations

| Feature | Documentation Path |
|---------|-------------------|
| **Delete Database** | `delete-database.md` |
| **Clone Schema** | `clone-database-schema.md` |
| **Database Script** | `database-script.md` |
| **Database Policies** | `database-table-policies.md` |

### 10.3 Encryption & Security

| Feature | Documentation Path |
|---------|-------------------|
| **Overview** | `cluster-encryption-overview.md` |
| **Disk Encryption** | `cluster-encryption-disk.md` |
| **Double Encryption** | `cluster-encryption-double.md` |
| **Customer Managed Keys** | `customer-managed-keys.md` |
| **Managed Identities** | `configure-managed-identities-cluster.md` |
| **Confidential Compute** | `confidential-compute.md` |

---

## 11. Business Continuity & Operations

### 11.1 Business Continuity

| Feature | Documentation Path |
|---------|-------------------|
| **Overview** | `business-continuity-overview.md` |
| **Create Solution** | `business-continuity-create-solution.md` |

### 11.2 Monitoring & Diagnostics

| Feature | Documentation Path |
|---------|-------------------|
| **Insights** | `data-explorer-insights.md` |
| **Azure Advisor** | `azure-advisor.md` |
| **Error Codes** | `error-codes.md` |

### 11.3 Concepts

| Feature | Documentation Path |
|---------|-------------------|
| **Query Limits** | `kusto/concepts/query-limits.md` |
| **Query Consistency** | `kusto/concepts/query-consistency.md` |
| **Resource Consumption** | `kusto/concepts/query-resource-consumption.md` |
| **Result Truncation** | `kusto/concepts/result-truncation.md` |
| **Runaway Queries** | `kusto/concepts/runaway-queries.md` |
| **Partial Failures** | `kusto/concepts/partial-query-failures.md` |
| **Overflow** | `kusto/concepts/overflow.md` |
| **Sandboxes** | `kusto/concepts/sandboxes.md` |
| **Fact/Dimension Tables** | `kusto/concepts/fact-and-dimension-tables.md` |

---

## 12. Integration Services

### 12.1 Azure Integration

| Service | Documentation Path |
|---------|-------------------|
| **Data Factory** | `data-factory-integration.md`, `data-factory-load-data.md`, `data-factory-command-activity.md`, `data-factory-template.md` |
| **Power Automate** | `flow.md`, `flow-usage.md` |
| **Data Share** | `data-share.md` |
| **Data Lake** | `data-lake-query-data.md` |

### 12.2 DevOps

| Feature | Documentation Path |
|---------|-------------------|
| **DevOps Integration** | `devops.md` |
| **Automated Deployment** | `automated-deploy-overview.md` |

### 12.3 Cross-Tenant/Cluster

| Feature | Documentation Path |
|---------|-------------------|
| **Cross-Tenant** | `cross-tenant-query-and-commands.md` |
| **Cross-Cluster** | `kusto/query/join-cross-cluster.md` |

---

## 13. AI/LLM Integration (New Features)

| Feature | Documentation Path |
|---------|-------------------|
| **Chat Completion Plugin** | `kusto/query/ai-chat-completion-plugin.md` |
| **Chat Completion Prompt** | `kusto/query/ai-chat-completion-prompt-plugin.md` |
| **Embeddings Plugin** | `kusto/query/ai-embeddings-plugin.md` |
| **SLM Embeddings (UDF)** | `kusto/functions-library/slm-embeddings-fl.md` |

---

## 14. Plugins & Extensibility

### 14.1 Query Plugins

| Plugin Type | Key Files |
|-------------|-----------|
| **Python Plugin** | `kusto/query/python-plugin.md` |
| **R Plugin** | `kusto/query/r-plugin.md` |
| **CosmosDB Plugin** | `kusto/query/cosmosdb-plugin.md` |
| **Azure Digital Twins** | `kusto/query/azure-digital-twins-query-request-plugin.md` |
| **SQL Request Plugin** | `kusto/query/sql-request-plugin.md` |
| **HTTP Request Plugin** | `kusto/query/http-request-plugin.md` |

### 14.2 Data Format Plugins

| Plugin | Documentation Path |
|--------|-------------------|
| **Bag Unpack** | `kusto/query/bag-unpack-plugin.md` |
| **Narrow** | `kusto/query/narrow-plugin.md` |
| **Pivot** | `kusto/query/pivot-plugin.md` |

---

## Cross-Reference Matrix

| Feature Area | Query Docs | Management Docs | API Docs | Service Docs |
|--------------|------------|-----------------|----------|--------------|
| **Time Series** | `kusto/query/series-*` | - | - | - |
| **Materialized Views** | - | `kusto/management/materialized-views/` | - | - |
| **Ingestion** | - | `kusto/management/batching-policy.md` | `kusto/api/get-started/app-*-ingestion.md` | `create-*-connection.md` |
| **Security** | - | `kusto/management/security-roles.md`, `row-level-security-policy.md` | - | `add-*-principal.md`, `kusto/access-control/` |
| **External Tables** | `kusto/query/schema-entities/external-tables.md` | `kusto/management/external-*` | - | `external-table.md` |
| **Functions** | `kusto/query/functions/` | `kusto/management/create-function.md` | - | - |
| **Dashboards** | - | - | - | `azure-data-explorer-dashboards.md`, `dashboard-*.md` |

---

## Summary Statistics

| Category | File Count |
|----------|------------|
| KQL Query Reference | ~645 files |
| Management Commands | ~297 files |
| Functions Library (UDFs) | ~69 files |
| API Documentation | ~51 files |
| Service Documentation | ~200 files |
| Concepts | ~12 files |
| Access Control | ~3 files |
| Tools | ~8 files |
| **Total Markdown Files** | ~1,491 files |

---

*Generated from Phase 1 repository layout analysis and documentation exploration.*
