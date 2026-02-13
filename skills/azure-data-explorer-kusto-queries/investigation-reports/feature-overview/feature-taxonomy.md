# Azure Data Explorer Feature Taxonomy

## Overview

This document provides a comprehensive hierarchical taxonomy of all major feature areas in Azure Data Explorer (ADX) based on analysis of the official documentation repository. The taxonomy is organized into primary categories with sub-features and relationships.

---

## Feature Taxonomy

### 1. Kusto Query Language (KQL)

The query language is the core feature of Azure Data Explorer with 645+ documented functions and operators.

#### 1.1 Query Entities
- **Databases** - Logical containers for tables and functions
- **Tables** - Primary data storage entities
- **Columns** - Typed fields within tables
- **Functions** - Stored query procedures
- **Views** - Virtual tables based on queries
- **External Tables** - References to external data sources

#### 1.2 Data Types
- **Scalar Types**
  - bool - Boolean values
  - datetime - Date and time values
  - decimal - High-precision numeric
  - dynamic - JSON-like structured data
  - guid - Globally unique identifiers
  - int - 32-bit integers
  - long - 64-bit integers
  - real - Floating-point numbers
  - string - Text values
  - timespan - Duration values

#### 1.3 Query Statements
- **let** - Variable and function definitions
- **alias** - Name aliasing
- **pattern** - Pattern declarations
- **restrict** - Access restrictions
- **set** - Query option settings
- **tabular expressions** - Table-producing expressions

#### 1.4 Tabular Operators
- **Filtering**
  - where - Row filtering
  - take/limit - Row count limiting
  - sample - Random sampling
  - distinct - Unique rows
- **Transformation**
  - extend - Add computed columns
  - project - Column selection/transformation
  - project-away - Column removal
  - project-rename - Column renaming
  - project-reorder - Column ordering
  - mv-expand - Multi-value expansion
  - mv-apply - Multi-value transformation
- **Aggregation**
  - summarize - Group and aggregate
  - count - Row counting
- **Joining**
  - join - Table joining (inner, outer, left, right, anti, semi)
  - union - Table concatenation
  - lookup - Dimension lookup
- **Sorting**
  - sort/order by - Row ordering
  - top - Top N rows
  - top-nested - Hierarchical top N
- **Graph Operators**
  - make-graph - Create graph from data
  - graph-match - Pattern matching on graphs
  - graph-to-table - Convert graph to table
  - graph-shortest-paths - Path finding
- **Set Operations**
  - intersect - Set intersection
  - except - Set difference

#### 1.5 Scalar Functions (500+)
- **String Functions**
  - strcat, strlen, substring, replace, split
  - contains, startswith, endswith, matches regex
  - extract, parse, parse_json
  - tolower, toupper, trim
- **DateTime Functions**
  - now, ago, datetime_add, datetime_diff
  - startofday, startofweek, startofmonth, startofyear
  - format_datetime, todatetime
  - dayofweek, dayofmonth, dayofyear
- **Math Functions**
  - abs, ceiling, floor, round
  - log, log10, log2, exp, pow, sqrt
  - sin, cos, tan, asin, acos, atan
  - min, max, sign
- **Array/Dynamic Functions**
  - array_length, array_concat, array_slice
  - bag_keys, bag_pack, bag_merge
  - set_difference, set_intersect, set_union
  - treepath, todynamic, tostring
- **Conditional Functions**
  - iff, iif, case, coalesce
  - isempty, isnull, isnotempty, isnotnull
  - max_of, min_of
- **Type Conversion Functions**
  - tostring, toint, tolong, todouble, tobool
  - todatetime, totimespan, toguid, todecimal
- **Hash Functions**
  - hash, hash_md5, hash_sha256
  - hash_combine, hash_many
- **Encoding Functions**
  - base64_encode_tostring, base64_decode_tostring
  - url_encode, url_decode
  - zlib_compress, zlib_decompress
- **Binary Functions**
  - binary_and, binary_or, binary_xor, binary_not
  - binary_shift_left, binary_shift_right

#### 1.6 Aggregation Functions
- **Count Functions**
  - count, countif, dcount, dcountif
  - count_distinct (alias for dcount)
- **Statistical Functions**
  - sum, sumif, avg, avgif
  - min, max, minif, maxif
  - stdev, stdevif, variance, varianceif
  - percentile, percentiles, percentile_array
- **Collection Functions**
  - make_list, make_set, make_bag
  - make_list_if, make_set_if
  - arg_min, arg_max
- **Binary Aggregations**
  - binary_all_and, binary_all_or, binary_all_xor

#### 1.7 Geospatial Functions
- **Point Operations**
  - geo_distance_point_to_point
  - geo_point_in_circle
  - geo_point_in_polygon
  - geo_point_to_geohash
  - geo_point_to_h3cell
  - geo_point_to_s2cell
- **Polygon Operations**
  - geo_polygon_area
  - geo_polygon_centroid
  - geo_polygon_perimeter
  - geo_polygon_simplify
- **Line Operations**
  - geo_line_length
  - geo_line_simplify
- **Geohash/H3/S2 Functions**
  - geo_geohash_to_polygon
  - geo_h3cell_to_polygon
  - geo_s2cell_to_polygon
  - geo_geohash_neighbors
  - geo_h3cell_children, geo_h3cell_parent
- **Union/Intersection**
  - geo_union, geo_intersection

#### 1.8 Time Series Analysis
- **Decomposition**
  - series_decompose - Seasonal decomposition
  - series_decompose_anomalies - Anomaly detection
  - series_decompose_forecast - Forecasting
- **Fitting**
  - series_fit_line - Linear regression
  - series_fit_2lines - Two-segment regression
  - series_fit_poly - Polynomial fitting
- **Statistics**
  - series_stats - Descriptive statistics
  - series_stats_dynamic - Dynamic statistics
  - series_pearson_correlation
  - series_periods_detect - Period detection
- **Transformation**
  - series_fir - Finite impulse response filter
  - series_iir - Infinite impulse response filter
  - series_fill_backward, series_fill_forward
  - series_fill_const, series_fill_linear
  - series_outliers - Outlier detection

#### 1.9 Window Functions
- **Navigation**
  - next - Next row value
  - prev - Previous row value
- **Ranking**
  - row_number - Sequential row number
  - row_rank - Rank with ties
  - row_dense_rank - Dense ranking
- **Cumulative**
  - row_cumsum - Cumulative sum
  - scan - Running aggregation

#### 1.10 Plugins
- **Machine Learning**
  - autocluster - Automatic clustering
  - basket - Association rules
  - diffpatterns - Pattern differentiation
  - narrow - Table pivoting
- **Language Plugins**
  - python - Python inline execution
  - r - R inline execution
- **Connectivity Plugins**
  - sql_request - SQL database queries
  - cosmosdb_sql_request - Cosmos DB queries
  - http_request - HTTP API calls
  - azure_digital_twins_query - Digital twins queries

---

### 2. Management Commands (Control Plane)

Management commands control cluster, database, and table configuration with 297+ documented commands.

#### 2.1 Schema Management
- **Cluster Management**
  - .show cluster
  - .show capacity
  - .show cluster policy
- **Database Management**
  - .create database
  - .alter database
  - .drop database
  - .show databases
  - .attach database
  - .detach database
- **Table Management**
  - .create table
  - .create-merge table
  - .alter table
  - .drop table
  - .rename table
  - .show tables
  - .undo drop table
- **Column Management**
  - .alter column
  - .drop column
  - .rename column
- **Function Management**
  - .create function
  - .alter function
  - .drop function
  - .show functions

#### 2.2 External Tables
- **Azure Storage External Tables**
  - .create external table (Azure Blob, ADLS)
  - .alter external table
  - .drop external table
  - .show external tables
- **Azure SQL External Tables**
  - .create external table sql
  - External table mappings
- **External Table Artifacts**
  - .show external table artifacts

#### 2.3 Materialized Views
- **Lifecycle Management**
  - .create materialized-view
  - .alter materialized-view
  - .drop materialized-view
  - .disable/.enable materialized-view
- **Monitoring**
  - .show materialized-views
  - .show materialized-view failures

#### 2.4 Persistent Graphs
- **Graph Management**
  - .create graph
  - .alter graph
  - .drop graph
  - .show graphs
- **Graph Snapshots**
  - .create graph snapshot
  - .show graph snapshots

#### 2.5 Stored Query Results
- **Management**
  - .set stored_query_result
  - .drop stored_query_result
  - .show stored_query_results

#### 2.6 Policies (30+ policy types)
- **Data Lifecycle Policies**
  - Retention policy
  - Caching policy
  - Soft delete policy
  - Auto delete policy
- **Performance Policies**
  - Partitioning policy
  - Row order policy
  - Merge policy
  - Sharding policy
- **Ingestion Policies**
  - Ingestion batching policy
  - Ingestion time policy
  - Encoding policy
  - Streaming ingestion policy
- **Security Policies**
  - Row level security policy
  - Restricted view access policy
  - Callout policy
  - Sandbox policy
- **Query Policies**
  - Query throttling policy
  - Query limit policy
  - Query consistency policy
  - Query weak consistency policy
- **Update Policies**
  - Update policy
  - Multi-table update policy

#### 2.7 Workload Groups
- **Request Classification**
  - Workload group definitions
  - Request classification rules
- **Resource Limits**
  - Memory limits
  - CPU limits
  - Concurrency limits
- **Queuing**
  - Queue policies
  - Priority settings

#### 2.8 Security Roles
- **Database Roles**
  - Database admin
  - Database user
  - Database viewer
  - Database ingestor
  - Database monitor
- **Table Roles**
  - Table admin
  - Table ingestor
- **Function Roles**
  - Function admin
- **Materialized View Roles**
  - Materialized view admin

#### 2.9 Data Ingestion Commands
- **Ingestion Mappings**
  - .create ingestion mapping (CSV, JSON, Avro, Parquet, ORC, W3CLOGFILE)
  - .alter ingestion mapping
  - .drop ingestion mapping
- **Inline Ingestion**
  - .ingest inline
  - .set-or-append
  - .set-or-replace
- **Queued Ingestion**
  - .ingest into table
  - Ingestion from storage

#### 2.10 Data Export
- **One-time Export**
  - .export to storage
  - .export to sql
- **Continuous Export**
  - .create continuous-export
  - .alter continuous-export
  - .show continuous-export
  - .show continuous-export failures

#### 2.11 Data Management
- **Data Purge**
  - .purge table (GDPR compliance)
  - .show purges
- **Extents/Shards**
  - .show extents
  - .drop extents
  - .merge extents
  - .move extents
  - .replace extents
- **Data Recovery**
  - .undo drop table

#### 2.12 System Information
- **Operations**
  - .show operations
  - .show operation details
- **Running Queries**
  - .show running queries
  - .cancel query
- **Commands**
  - .show commands
  - .show commands-and-queries
- **Journal**
  - .show journal
  - .show database schema
- **Cluster Diagnostics**
  - .show diagnostics
  - .show cluster databases
  - .show cluster extents

---

### 3. API and SDK Integration

Multi-language SDK support for programmatic access.

#### 3.1 REST API
- **Authentication**
  - Azure Active Directory (Microsoft Entra ID)
  - Managed identities
  - Service principals
- **Query API**
  - Query request format
  - Query response format
  - Progressive query results
  - Streaming responses
- **Management API**
  - Control command execution
  - Async operations
- **Ingestion API**
  - Streaming ingestion endpoint
  - Queued ingestion status

#### 3.2 .NET SDK (Kusto.Data, Kusto.Ingest)
- **Kusto.Data**
  - KustoConnectionStringBuilder
  - KustoClient/ICslQueryProvider
  - Client request properties
- **Kusto.Ingest**
  - KustoQueuedIngestClient
  - KustoDirectIngestClient
  - KustoStreamingIngestClient
  - Ingestion result reporting
- **Kusto.Language**
  - Query parsing
  - Semantic analysis
  - Intellisense support

#### 3.3 Python SDK (azure-kusto-python)
- **KustoClient**
  - Query execution
  - Management commands
- **KustoIngestClient**
  - File ingestion
  - Stream ingestion
  - Blob ingestion
- **DataFrame Integration**
  - Pandas integration
  - Result to DataFrame conversion

#### 3.4 Java SDK (azure-kusto-java)
- **Client Creation**
  - Connection string handling
  - Authentication providers
- **Query Execution**
  - Synchronous queries
  - Client request options
- **Ingestion**
  - IngestClient interface
  - Source info configuration

#### 3.5 Node.js SDK (azure-kusto-node)
- **KustoClient**
  - Query and management operations
- **IngestClient**
  - Ingestion operations
  - Streaming support

#### 3.6 Go SDK (azure-kusto-go)
- **Client**
  - Query execution
  - Iterator-based results
- **Ingestion**
  - Managed streaming
  - File ingestion

#### 3.7 R SDK (azure-kusto-r)
- **AzureKusto Package**
  - Query execution
  - Result handling
  - dplyr integration

#### 3.8 PowerShell Integration
- **Az.Kusto Module**
  - Cluster management
  - Database management
  - Principal assignment

---

### 4. Data Ingestion

Comprehensive data ingestion capabilities.

#### 4.1 Streaming Ingestion
- Real-time data flow
- Low latency requirements
- Direct ingestion to tables

#### 4.2 Queued/Batched Ingestion
- Batching policy configuration
- Optimized for throughput
- Automatic batching

#### 4.3 Connectors and Integrations
- **Azure Native**
  - Event Hubs
  - Event Grid
  - IoT Hub
  - Azure Storage (Blob, ADLS)
  - Cosmos DB
  - Azure Data Factory
- **Third-Party**
  - Apache Kafka
  - Apache Spark
  - Apache Flink
  - Logstash
  - Splunk
  - Telegraf
  - Fluent Bit

#### 4.4 One-Click Ingestion
- Web UI guided ingestion
- Schema inference
- Format detection

#### 4.5 Ingestion Formats
- CSV
- JSON
- Avro
- Parquet
- ORC
- W3C Extended Log Format
- TSV
- TXT
- Raw

---

### 5. Visualization and Dashboards

#### 5.1 Native Dashboards (Web UI)
- Dashboard creation
- Tile management
- Parameters and filters
- Cross-filtering
- Sharing and permissions

#### 5.2 Power BI Integration
- Native connector
- Import and DirectQuery modes
- KQL integration

#### 5.3 Third-Party Visualization
- Grafana (native data source)
- Tableau
- Excel
- Sisense
- Redash
- Kibana

---

### 6. Security and Access Control

#### 6.1 Authentication
- Microsoft Entra ID (Azure AD)
- Managed identities
- Service principals
- User principals

#### 6.2 Authorization
- Role-based access control (RBAC)
- Row-level security
- Column-level security
- Restricted view access

#### 6.3 Network Security
- Private endpoints
- Virtual network service endpoints
- Firewall rules
- DNS configuration

#### 6.4 Data Protection
- Customer-managed keys (CMK)
- Disk encryption
- Data masking
- Data purge (GDPR)

---

### 7. Cluster Management

#### 7.1 Cluster Operations
- Create cluster
- Scale up/down
- Scale out/in
- Start/Stop cluster
- Delete cluster

#### 7.2 Cluster Configuration
- SKU selection
- Availability zones
- Auto-scale configuration
- Engine and data management instances

#### 7.3 Free Cluster
- Trial cluster creation
- Limitations and quotas
- Upgrade path

#### 7.4 Cluster Monitoring
- Azure Monitor metrics
- Diagnostic logs
- Health checks
- Resource utilization

---

### 8. Business Continuity

#### 8.1 High Availability
- Availability zone support
- Multi-zone deployment
- Leader follower clusters

#### 8.2 Disaster Recovery
- Cross-region replication
- Geo-redundant backup
- Recovery procedures

#### 8.3 Backup and Restore
- Automated backups
- Point-in-time recovery
- Cross-cluster data sharing

---

### 9. Integration Services

#### 9.1 Azure Integration
- Azure Monitor integration
- Azure Synapse integration
- Azure Machine Learning
- Azure Logic Apps
- Azure Functions
- Power Automate

#### 9.2 Cross-Product Queries
- Azure Monitor logs query
- Application Insights query
- Resource Graph query

#### 9.3 MCP Servers
- Model Context Protocol integration
- AI/ML connectivity

---

### 10. User-Defined Functions Library

Pre-built functions for advanced analytics.

#### 10.1 Statistical Tests
- bartlett_test
- binomial_test_fl
- ks_test_fl (Kolmogorov-Smirnov)
- levene_test_fl
- mann_whitney_u_test_fl
- normality_test_fl
- t_test_fl (t-test variations)
- wilcoxon_test_fl

#### 10.2 Machine Learning
- kmeans_fl (K-means clustering)
- dbscan_fl (Density-based clustering)
- predict_fl
- predict_onnx_fl
- linear_regression_fl

#### 10.3 Time Series
- series_moving_avg_fl
- series_exp_smoothing_fl
- series_rolling_fl
- time_weighted_avg_fl

#### 10.4 Text Analytics
- log_reduce_fl
- log_reduce_predict_fl
- log_reduce_train_fl

#### 10.5 Graph Analytics
- graph_blast_radius_fl
- graph_centrality_fl
- pair_probabilities_fl

#### 10.6 Visualization
- plotly_anomaly_fl
- plotly_scatter3d_fl

---

### 11. Tools and Clients

#### 11.1 Kusto.Explorer
- Desktop query tool
- Query editing and execution
- Result visualization
- Connection management

#### 11.2 Kusto.Cli
- Command-line interface
- Scripting support
- Batch operations

#### 11.3 Web UI
- Browser-based query editor
- Dashboard management
- Data exploration
- Schema browsing

#### 11.4 Monaco Editor Integration
- VS Code extension
- Intellisense
- Syntax highlighting
- Query formatting

#### 11.5 Kusto Emulator
- Local development
- Testing environment
- Docker-based deployment

---

## Feature Relationships Diagram

```
                           +---------------------------+
                           |   Azure Data Explorer     |
                           |         (ADX)             |
                           +---------------------------+
                                        |
        +---------------+---------------+---------------+---------------+
        |               |               |               |               |
        v               v               v               v               v
+---------------+ +-------------+ +-------------+ +--------------+ +------------+
|    Kusto      | |  Management | |    Data     | | Visualization| |  Security  |
|Query Language | |   Commands  | |  Ingestion  | |  & Dashboards| |  & Access  |
+---------------+ +-------------+ +-------------+ +--------------+ +------------+
        |               |               |               |               |
        |               |               |               |               |
        v               v               v               v               v
+---------------+ +-------------+ +-------------+ +--------------+ +------------+
|  - Operators  | | - Schema    | | - Streaming | | - Native     | | - RBAC     |
|  - Functions  | | - Policies  | | - Batched   | | - Power BI   | | - RLS      |
|  - Plugins    | | - Security  | | - Connectors| | - Grafana    | | - CMK      |
|  - Types      | | - Export    | | - Mappings  | | - Excel      | | - Network  |
+---------------+ +-------------+ +-------------+ +--------------+ +------------+
                                        |
                                        v
                           +---------------------------+
                           |   SDK & API Integration   |
                           |  .NET | Python | Java     |
                           |  Node | Go | R | REST     |
                           +---------------------------+
```

---

## Summary Statistics

| Category | Sub-features | Documented Items |
|----------|--------------|------------------|
| KQL Query Functions | 10 | 645+ |
| Management Commands | 12 | 297+ |
| API/SDK Languages | 8 | 50+ |
| Data Connectors | 2 | 15+ |
| Policies | 1 | 30+ |
| UDF Library Functions | 6 | 73 |
| **Total Major Features** | **39** | **1,100+** |

---

## Document Information

- **Generated**: 2026-02-13
- **Source**: dataexplorer-docs repository analysis
- **Phase**: 2 - Feature Discovery
