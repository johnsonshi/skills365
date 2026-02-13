# Azure Data Explorer Documentation - Content Organization Analysis

## Overview

The dataexplorer-docs repository follows a **task-based and feature-oriented hybrid organization pattern**. Content is organized primarily by:

1. **User tasks/workflows** (Get data, Query data, Visualize data, Admin)
2. **Technical domain** (Kusto Query Language, Management Commands, API)
3. **User journey stages** (Get started, Secure, Develop, Integrate)

This organization pattern enables users to find content based on what they want to accomplish while also providing comprehensive reference documentation organized by technical domain.

---

## TOC Structure Analysis

### Main Documentation Hierarchy

The repository has **11 toc.yml files** that define the navigation structure:

| TOC File | Purpose | Scope |
|----------|---------|-------|
| `data-explorer/toc.yml` | Main Azure Data Explorer documentation | Primary entry point |
| `data-explorer/kusto/toc.yml` | Kusto language documentation hub | KQL reference |
| `data-explorer/kusto/query/toc.yml` | Query language reference | 1800+ lines, comprehensive |
| `data-explorer/kusto/management/toc.yml` | Management commands reference | 1000+ lines |
| `data-explorer/kusto/api/toc.yml` | API and SDK reference | REST, .NET, Python, etc. |
| `data-explorer/kusto/functions-library/toc.yml` | User-defined functions library | Statistical, ML functions |
| `data-explorer/breadcrumb/toc.yml` | Navigation breadcrumbs | Cross-linking |
| `data-explorer/kusto/breadcrumb/toc.yml` | Kusto breadcrumbs | Cross-linking |
| `data-explorer/kusto-tocs/query/toc.yml` | Query TOC reference | Alias |
| `data-explorer/kusto-tocs/management/toc.yml` | Management TOC reference | Alias |
| `data-explorer/kusto-tocs/api/toc.yml` | API TOC reference | Alias |

### Main TOC Structure (`data-explorer/toc.yml`)

```
Azure Data Explorer documentation
|
+-- Overview
|   +-- What is Azure Data Explorer?
|   +-- How Azure Data Explorer works
|   +-- Get started for free
|   +-- What's new
|
+-- Get started
|   +-- Plan (Solution architectures, Multi-tenant, POC playbook)
|   +-- Set up (Create cluster/database, Create tables, Migrate)
|   +-- Secure (Security, Access control, Network, Encryption, Managed identities)
|   +-- Get data (Ingestion overview, Web UI, SDKs, Event Hubs, Event Grid, IoT Hub, etc.)
|
+-- Query data
|   +-- Query integrations overview
|   +-- Query in web UI
|   +-- External tables, Data Lake, Azure Monitor
|   +-- Python, T-SQL, SQL Server emulation
|   +-- Time series analysis and ML
|
+-- Visualize data
|   +-- Dashboards in web UI
|   +-- Power BI
|   +-- Excel, Grafana, Tableau, Sisense, Redash, Kibana
|
+-- Kusto Query Language --> (links to kusto/query/toc.yml)
|
+-- Management commands --> (links to kusto/management/toc.yml)
|
+-- Admin
|   +-- Manage (Cluster, Free cluster, Database, Table, Data)
|   +-- Monitor
|   +-- Business continuity
|   +-- Share data
|
+-- Develop
|   +-- API overview
|   +-- Client libraries
|   +-- Write code with SDKs
|   +-- Connection strings
|   +-- Query editor integration
|   +-- Kusto emulator
|
+-- Integrate
|   +-- Power Automate, Logic Apps, Azure Pipelines
|   +-- Notebooks
|   +-- Azure Functions
|   +-- MCP servers
|
+-- Reference
|   +-- API (links to kusto/api/toc.yml)
|   +-- Azure CLI, Policy, ARM templates, PowerShell
|   +-- Troubleshooting
|
+-- Resources
    +-- Forums, Product feedback, Pricing, Service updates
```

---

## Section Relationship Diagram

```
                    +---------------------------+
                    |   Azure Data Explorer     |
                    |   Documentation (Main)    |
                    +---------------------------+
                               |
        +----------------------+----------------------+
        |                      |                      |
        v                      v                      v
+---------------+    +------------------+    +----------------+
|  Azure ADX    |    |      Kusto       |    |   Reference    |
|  Service Docs |    |   Language Docs  |    |   & Resources  |
+---------------+    +------------------+    +----------------+
        |                      |
        |            +---------+---------+
        |            |         |         |
        v            v         v         v
+---------------+  +-----+  +-----+  +-----+
| Get started   |  |Query|  |Mgmt |  | API |
| Query data    |  | toc |  | toc |  | toc |
| Visualize     |  +-----+  +-----+  +-----+
| Admin         |     |         |         |
| Develop       |     v         v         v
| Integrate     |  +-----+   +-----+   +-----+
+---------------+  |Func |   |Schema|  |REST |
                   |Lib  |   |Policy|  |SDKs |
                   +-----+   |Data  |   +-----+
                             |Export|
                             +-----+
```

---

## Key Content Areas Identified

### 1. Kusto Query Language (KQL) Reference
**Location:** `data-explorer/kusto/query/`
**File count:** 647 files

Major categories:
- **Entities** (databases, tables, columns, functions, views)
- **Data types** (bool, datetime, decimal, dynamic, guid, int, long, real, string, timespan)
- **Query statements** (let, alias, pattern, restrict, set, tabular expressions)
- **Tabular operators** (where, summarize, join, union, extend, project, sort, etc.)
- **Scalar functions** (500+ functions including string, datetime, math, array, JSON)
- **Aggregation functions** (count, sum, avg, min, max, dcount, percentile, etc.)
- **Graph operators** (make-graph, graph-match, graph-to-table, etc.)
- **Geospatial functions** (geo_distance, geo_point_in_polygon, H3, S2, Geohash)
- **Time series analysis** (series_decompose, series_fit_line, anomaly detection)
- **Plugins** (Python, R, ML plugins, query connectivity plugins)
- **Window functions** (next, prev, row_cumsum, row_number, row_rank)

### 2. Management Commands Reference
**Location:** `data-explorer/kusto/management/`
**File count:** 299 files

Major categories:
- **Workload groups** (request classification, limits, queuing policies)
- **Schema management** (clusters, databases, tables, columns, functions)
- **External tables** (Azure Storage, Azure SQL)
- **Materialized views**
- **Graphs** (persistent graphs, graph models, graph snapshots)
- **Stored query results**
- **Policies** (30+ policy types including caching, retention, partitioning, ingestion batching, row-level security)
- **Security roles** (database, table, function, materialized view roles)
- **Data ingestion** (mappings, streaming, queued ingestion)
- **Data export** (continuous export, export to storage/SQL)
- **System information** (operations, queries, commands, statistics)
- **Advanced data management** (data purge, soft delete, extents/shards)

### 3. API and SDK Reference
**Location:** `data-explorer/kusto/api/`
**File count:** 17 directories + files

Major categories:
- **REST API** (authentication, query request/response, streaming ingest)
- **.NET SDK** (Kusto.Data, Kusto.Ingest, Kusto.Language)
- **Python SDK** (azure-kusto-python)
- **R SDK** (azure-kusto-r)
- **Java SDK** (azure-kusto-java)
- **Node SDK** (azure-kusto-node)
- **Go SDK** (azure-kusto-go)
- **PowerShell** integration

### 4. Functions Library
**Location:** `data-explorer/kusto/functions-library/`
**File count:** 73 files

User-defined function categories:
- **Statistical tests** (bartlett, binomial, ks_test, normality, levene, mann-whitney, t-test, wilcoxon)
- **Clustering** (dbscan, kmeans)
- **Security/Cybersecurity** (anomaly detection, graph analysis)
- **Machine learning** (predict, predict_onnx)
- **Time series** (smoothing, forecasting, anomaly detection)
- **Text analytics** (log_reduce, tokenize)
- **Visualization** (Plotly charts)
- **Graph functions** (blast radius, centrality, path discovery)

### 5. Azure Data Explorer Service Documentation
**Location:** `data-explorer/` (root level .md files)
**File count:** ~200 markdown files

Major categories:
- **Cluster management** (create, scale, encrypt, monitor)
- **Data ingestion** (Event Hubs, Event Grid, IoT Hub, Cosmos DB, Data Factory)
- **Data connectors** (Kafka, Spark, Flink, Logstash, Splunk, Telegraf)
- **Visualization** (Dashboards, Power BI, Grafana, Tableau)
- **Security** (private endpoints, managed identities, encryption)
- **Integration** (Power Automate, Logic Apps, Azure Pipelines)

---

## Content Organization Patterns

### Pattern 1: Task-Based Organization (Main TOC)
The main TOC organizes content by user tasks:
- "Get started" -> Setup tasks
- "Query data" -> Data analysis tasks
- "Visualize data" -> Reporting tasks
- "Admin" -> Operational tasks
- "Develop" -> Programming tasks
- "Integrate" -> Automation tasks

### Pattern 2: Reference-Based Organization (Kusto TOCs)
The Kusto documentation follows a comprehensive reference pattern:
- Organized by language construct type (operators, functions, statements)
- Alphabetically sorted within categories
- Extensive cross-referencing via displayName metadata

### Pattern 3: Policy-Centric Organization (Management TOC)
Management commands are organized around:
- Object types (cluster, database, table, column)
- Policy types (30+ distinct policy categories)
- Operations (create, alter, drop, show)

### Pattern 4: Cross-Linking Strategy
- Breadcrumb TOCs define navigation paths
- `kusto-tocs/` directory contains alias references
- External links to Azure documentation (Azure Monitor, Time Series Insights)
- `href` attributes with `?view=azure-data-explorer&preserve-view=true` for cross-product linking

---

## Directory Structure Summary

```
dataexplorer-docs/
+-- data-explorer/                    # Main documentation root
    +-- toc.yml                       # Main navigation
    +-- index.yml                     # Landing page
    +-- docfx.json                    # Build configuration
    +-- *.md                          # ~200 Azure ADX service docs
    +-- breadcrumb/                   # Navigation breadcrumbs
    +-- context/                      # Context configuration
    +-- kusto/                        # Kusto language docs
    |   +-- toc.yml                   # Kusto hub navigation
    |   +-- query/                    # Query language reference (647 files)
    |   |   +-- toc.yml               # Query TOC (1800+ lines)
    |   +-- management/               # Management commands (299 files)
    |   |   +-- toc.yml               # Management TOC (1000+ lines)
    |   +-- api/                      # API reference (17+ items)
    |   |   +-- toc.yml               # API TOC
    |   +-- functions-library/        # UDF library (73 files)
    |   |   +-- toc.yml               # Functions TOC
    |   +-- access-control/           # Security docs
    |   +-- concepts/                 # Conceptual docs
    |   +-- tools/                    # Kusto.Explorer, Kusto.Cli
    |   +-- includes/                 # Reusable content snippets
    |   +-- media/                    # Images
    +-- kusto-tocs/                   # TOC aliases/references
        +-- query/toc.yml
        +-- management/toc.yml
        +-- api/toc.yml
```

---

## Key Insights

1. **Dual audience support**: The documentation serves both end-users (task-based Azure ADX docs) and developers (reference-based Kusto docs).

2. **Comprehensive KQL reference**: The query documentation is exceptionally thorough with 600+ function/operator pages.

3. **Policy-heavy management**: 30+ distinct policy types indicate significant operational configurability.

4. **Multi-SDK support**: First-class SDK support for .NET, Python, Java, Node, Go, R, and PowerShell.

5. **Cross-product integration**: Strong integration with Azure Monitor, Power BI, Synapse, and other Azure services.

6. **Functions library**: Advanced user-defined functions for ML, statistics, and security analytics.
