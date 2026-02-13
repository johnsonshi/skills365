# Azure Data Explorer - Feature Priority Analysis for Phase 3 Investigation

## Executive Summary

This document prioritizes features from the Azure Data Explorer documentation repository for Phase 3 deep investigation. Features are categorized into CORE, ADVANCED, and REFERENCE tiers based on frequency of use, importance to skill users, and documentation complexity.

---

## Priority Tier Definitions

| Tier | Description | Investigation Depth |
|------|-------------|---------------------|
| **CORE** | Essential for basic usage, most commonly needed | Deep investigation with examples |
| **ADVANCED** | Power user features, specialized use cases | Moderate investigation with patterns |
| **REFERENCE** | API docs, SDK references, administrative details | Light investigation, link-based |

---

## CORE Features (Priority 1)

These features are fundamental to using Azure Data Explorer and KQL effectively. They should receive the most detailed investigation.

### 1.1 KQL Query Language Fundamentals
**Rationale:** The query language is the primary interface for all data analysis tasks. ~645 query documentation files indicate this is the heart of ADX functionality.

| Feature Area | File Count | Priority Rationale |
|--------------|------------|-------------------|
| **Tabular Operators** | 50+ | Most frequently used: `where`, `summarize`, `join`, `extend`, `project`, `sort` |
| **Aggregation Functions** | 30+ | Essential for data analysis: `count`, `sum`, `avg`, `dcount`, `percentile` |
| **String Functions** | 40+ | Common data manipulation: `contains`, `startswith`, `parse`, `extract` |
| **DateTime Functions** | 25+ | Time-series analysis foundation: `ago`, `between`, `bin`, `datetime_part` |
| **Data Types** | 10+ | Core understanding: `dynamic`, `datetime`, `timespan`, `string`, `real` |

**Key Files to Investigate:**
- `data-explorer/kusto/query/toc.yml` - Main query navigation (1800+ lines)
- `data-explorer/kusto/query/index.md` - Query reference entry point
- Tabular operators in `data-explorer/kusto/query/`

### 1.2 Data Ingestion
**Rationale:** Getting data into ADX is a prerequisite for all analysis. Multiple ingestion methods documented across ~50 files.

| Feature Area | Priority Rationale |
|--------------|-------------------|
| **Web UI Ingestion** | Simplest entry point for new users |
| **Event Hubs** | Most common streaming source |
| **Event Grid** | Azure blob storage integration |
| **SDK Ingestion** | Programmatic data loading |
| **Ingestion Mappings** | Data transformation during ingest |

**Key Files:**
- Top-level `data-explorer/` contains ~20 ingestion-related articles
- `data-explorer/kusto/management/data-ingestion/` directory

### 1.3 Data Visualization (Dashboards)
**Rationale:** Dashboards are the primary output mechanism for data insights. Documentation shows significant media assets (~128 subdirs including dashboard screenshots).

| Feature Area | Priority Rationale |
|--------------|-------------------|
| **Web UI Dashboards** | Native ADX visualization |
| **Power BI Integration** | Enterprise reporting standard |
| **Query Results Rendering** | Inline visualization in queries |

**Key Files:**
- `data-explorer/azure-data-explorer-dashboards.md`
- Dashboard-related files in `data-explorer/`
- `data-explorer/media/adx-dashboards/`

---

## ADVANCED Features (Priority 2)

These features serve specialized use cases and power users. They require moderate investigation.

### 2.1 Time Series Analysis
**Rationale:** Core ADX differentiator for IoT and monitoring scenarios.

| Feature Area | Priority Rationale |
|--------------|-------------------|
| **series_decompose** | Trend/seasonality analysis |
| **Anomaly Detection** | series_decompose_anomalies, series_outliers |
| **Forecasting** | series_fit_line, series_fit_2lines |
| **Smoothing** | series_fir, series_iir |

**Key Files:**
- Time series functions in `data-explorer/kusto/query/`
- `data-explorer/kusto/functions-library/` - 73 UDF files including time series

### 2.2 Machine Learning Integration
**Rationale:** Enables advanced analytics beyond basic queries.

| Feature Area | Priority Rationale |
|--------------|-------------------|
| **Python Plugin** | sklearn, pandas integration |
| **R Plugin** | Statistical analysis |
| **ONNX Prediction** | ML model inference |
| **Clustering** | kmeans, dbscan functions |

**Key Files:**
- `data-explorer/kusto/query/` - plugin documentation
- `data-explorer/kusto/functions-library/` - clustering, predict functions

### 2.3 Management Commands
**Rationale:** ~297 management command files indicate extensive operational control. Critical for administrators.

| Feature Area | File Count | Priority Rationale |
|--------------|------------|-------------------|
| **Schema Management** | 50+ | Tables, columns, functions |
| **Policies** | 30+ types | Caching, retention, partitioning, security |
| **Materialized Views** | 15+ | Pre-aggregated data optimization |
| **External Tables** | 20+ | Query external storage |

**Key Files:**
- `data-explorer/kusto/management/toc.yml` (1000+ lines)
- `data-explorer/kusto/management/index.md`
- Policy subdirectories

### 2.4 Graph Operations
**Rationale:** Emerging feature area with dedicated documentation.

| Feature Area | Priority Rationale |
|--------------|-------------------|
| **make-graph** | Graph construction from tabular data |
| **graph-match** | Pattern matching in graphs |
| **Graph Snapshots** | Persistent graph storage |

**Key Files:**
- `data-explorer/kusto/management/graph/`
- Graph operators in query documentation

### 2.5 Geospatial Functions
**Rationale:** Specialized but growing use case for location analytics.

| Feature Area | Priority Rationale |
|--------------|-------------------|
| **geo_distance** | Point-to-point calculations |
| **geo_point_in_polygon** | Geofencing |
| **H3/S2/Geohash** | Spatial indexing |

---

## REFERENCE Features (Priority 3)

These features are primarily reference documentation. Light investigation with links to official docs.

### 3.1 SDK Documentation
**Rationale:** 7 SDK languages documented. Users typically reference official SDK docs.

| SDK | Location |
|-----|----------|
| .NET | `data-explorer/kusto/api/netfx/` (~19 files) |
| Python | `data-explorer/kusto/api/python/` |
| Java | `data-explorer/kusto/api/java/` |
| Node.js | `data-explorer/kusto/api/node/` |
| Go | `data-explorer/kusto/api/golang/` |
| R | `data-explorer/kusto/api/r/` |
| PowerShell | `data-explorer/kusto/api/powershell/` |

### 3.2 REST API
**Rationale:** Low-level API, typically accessed via SDKs.

**Key Files:**
- `data-explorer/kusto/api/rest/` (~12 files)

### 3.3 Access Control
**Rationale:** Security configuration, typically one-time setup.

**Key Files:**
- `data-explorer/kusto/access-control/` (~6 files)

### 3.4 Tools Documentation
**Rationale:** Kusto.Explorer and CLI are well-documented but rarely need skill assistance.

**Key Files:**
- `data-explorer/kusto/tools/` (~11 files)

### 3.5 Connection Strings
**Rationale:** Reference format, straightforward usage.

**Key Files:**
- `data-explorer/kusto/api/connection-strings/`

---

## Phase 3 Investigation Recommendations

### Immediate Priority (Sprint 1)
1. **KQL Tabular Operators** - Focus on top 20 most-used operators
2. **Aggregation Functions** - Complete coverage of core aggregations
3. **DateTime Operations** - Time-based filtering and binning
4. **Basic Ingestion** - Web UI and Event Hubs patterns

### Secondary Priority (Sprint 2)
1. **Time Series Analysis** - Decomposition and anomaly detection
2. **Dashboard Creation** - Native dashboard patterns
3. **Join Operations** - All join flavors and optimization
4. **Policy Management** - Caching, retention, partitioning

### Tertiary Priority (Sprint 3)
1. **ML/Python Integration** - Plugin patterns
2. **Graph Operations** - Basic graph queries
3. **Geospatial Functions** - Common location patterns
4. **External Tables** - Data lake querying

---

## Feature Cross-Reference Matrix

| Feature | Doc Files | Media Assets | Complexity | User Impact |
|---------|-----------|--------------|------------|-------------|
| KQL Query | 645+ | High | High | Critical |
| Management | 297+ | Medium | High | High |
| Functions Library | 73 | Medium | Medium | Medium |
| API/SDK | 50+ | Low | Medium | Medium |
| Access Control | 6 | Low | Low | Low |
| Tools | 11 | Medium | Low | Low |

---

## Prioritization Rationale Summary

1. **Most Referenced:** KQL query documentation (~645 files) is by far the largest content area, indicating central importance
2. **User Journey:** Ingestion -> Query -> Visualize is the primary user path
3. **Differentiation:** Time series and ML capabilities distinguish ADX from other databases
4. **Investigation Depth:** CORE features need example-rich documentation; REFERENCE features need accurate linking
5. **Skill User Needs:** Query writing and data visualization are the most common assistance requests

---

## Statistics

| Tier | Feature Areas | Estimated Files | Investigation Effort |
|------|---------------|-----------------|---------------------|
| CORE | 3 | ~750 | High (40%) |
| ADVANCED | 5 | ~450 | Medium (35%) |
| REFERENCE | 5 | ~100 | Low (25%) |
| **Total** | **13** | **~1300** | **100%** |
