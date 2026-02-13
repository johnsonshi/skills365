# Visualization and Dashboards Reference

## Overview

Azure Data Explorer provides comprehensive visualization capabilities through native dashboards, the render operator, and integrations with third-party tools like Power BI and Grafana.

---

## Native ADX Dashboards

### Dashboard Architecture

ADX dashboards are web-based, integrated directly into the Azure Data Explorer web UI. Each dashboard consists of:

- **Tiles**: Individual visualizations with underlying KQL queries
- **Pages**: Optional containers to organize tiles into logical groups
- **Parameters**: Dashboard-level filters that can be used across multiple tiles
- **Data Sources**: Connections to one or more ADX clusters

### Creating Dashboards

```
Navigation: Dashboards > New dashboard > Enter name > Create
```

**Adding Data Sources:**
1. Select **Data sources** from toolbar
2. Click **+ Add**
3. Enter cluster URI and select database
4. Optionally configure query results cache max age

**Adding Tiles:**
1. Click **Add tile** from canvas or toolbar
2. Select data source
3. Write KQL query and run
4. Select **Visual** tab
5. Choose visual type
6. Click **Apply changes**

### Dashboard Export/Import

Dashboards can be exported as JSON for version control, templating, or backup:

```json
{
  "id": "{GUID}",
  "title": "Dashboard title",
  "tiles": [
    {
      "id": "{GUID}",
      "title": "Tile title",
      "query": "{QUERY}",
      "layout": { "x": 0, "y": 7, "width": 6, "height": 5 },
      "visualType": "line",
      "visualOptions": { ... }
    }
  ],
  "dataSources": [ {} ],
  "parameters": [ {} ],
  "pages": [ { "name": "Primary", "id": "{GUID}" } ],
  "autoRefresh": { "enabled": true, "defaultInterval": "15m", "minInterval": "5m" }
}
```

**Export**: File > Download dashboard to file
**Import**: New dashboard > Import dashboard from file

### Dashboard Sharing

**Permission Levels:**
- **Can view**: Read-only access
- **Can edit**: Full editing permissions

**Sharing Steps:**
1. Toggle to Editing mode
2. Select Share > Manage access
3. Add Microsoft Entra users or groups
4. Assign permission level

**Cross-Tenant Sharing:**
- Disabled by default (tenant admin must enable)
- Send invitation link to external users
- Invitations expire after 3 days

---

## Render Operator

The render operator instructs the client to visualize query results. It must be the last operator in a query.

### Syntax

```kusto
T | render visualization [with (propertyName = propertyValue [, ...])]
```

### Visualization Types

| Type | Description |
|------|-------------|
| `timechart` | Line graph with datetime x-axis |
| `linechart` | Standard line graph |
| `barchart` | Horizontal bar chart |
| `columnchart` | Vertical column chart |
| `piechart` | Pie/donut chart |
| `scatterchart` | Scatter plot (supports map kind) |
| `areachart` | Area graph |
| `stackedareachart` | Stacked area graph |
| `table` | Default tabular display |
| `card` | Single-value card display |
| `anomalychart` | Timechart with anomaly highlighting |
| `treemap` | Hierarchical rectangles |
| `pivotchart` | Interactive pivot table |
| `ladderchart` | Timeline ladder visualization |

### Render Properties

| Property | Description |
|----------|-------------|
| `title` | Chart title |
| `xtitle`, `ytitle` | Axis titles |
| `xcolumn` | Column for x-axis |
| `ycolumns` | Columns for y-axis values |
| `series` | Columns defining data series |
| `legend` | `visible` or `hidden` |
| `ymin`, `ymax` | Y-axis range |
| `xaxis`, `yaxis` | Scale: `linear` or `log` |
| `kind` | Sub-type (e.g., `stacked`, `map`) |
| `ysplit` | `none`, `axes`, or `panels` |
| `accumulate` | Cumulative values (`true`/`false`) |

### Kind Property Values

| Visual | Kind | Description |
|--------|------|-------------|
| areachart | `stacked` | Stack areas to the right |
| areachart | `stacked100` | Stack areas with equal width |
| barchart | `stacked` | Stack bars |
| columnchart | `stacked100` | Stack columns to same height |
| scatterchart | `map` | Render as geographic map |
| piechart | `map` | Geographic pie charts |

---

## Dashboard-Specific Visuals

These visualizations are only available in ADX dashboards (not via render operator):

### Funnel Chart

Visualizes sequential process stages showing drop-off between steps.

```kusto
// Example: Server request funnel
let funnelData =
    union
    (base | summarize Count = count() | extend Stage = "Sessions"),
    (base | summarize Count = countif(completed) | extend Stage = "Completed");
funnelData | project Stage, Count
```

### Heatmap

Displays values as a grid of colored squares across two axes.

**Requirements:**
- Three columns: X value, Y value, numeric Value
- If X is string, Y must be string
- If X is datetime, Y must be numeric

```kusto
TransformedServerMetrics
| summarize Count = count() by SQLMetrics, MetricType
| project X = MetricType, Y = SQLMetrics, Value = Count
```

### Map Visual

Geographic visualization for location-based data. Define location by:
- **Infer**: Automatic detection
- **Latitude and longitude**: Explicit coordinates
- **Geo point**: GeoJSON point

### Multi Stat

Multiple statistics in a configurable grid layout (1x1 to 5x5).

---

## Power BI Integration

### Connectivity Modes

| Mode | Use Case |
|------|----------|
| **Import** | Small datasets, don't need real-time data, perform aggregations in Kusto |
| **DirectQuery** | Large datasets, need near real-time data, frequently updated |

### Connection Methods

**Method 1: From ADX Web UI**
1. Run query in ADX web UI
2. Export > Query to Power BI
3. Paste in Power BI Desktop > Transform data

**Method 2: Power BI Connector**
1. Power BI Desktop > Get Data > Azure Data Explorer (Kusto)
2. Enter cluster URL
3. Select database and table
4. Choose Import or DirectQuery mode

### Connection Settings

| Setting | Description |
|---------|-------------|
| Cluster | `https://{ClusterName}.{Region}.kusto.windows.net` |
| Database | Target database name |
| Table name | Table or KQL query |
| Limit query result record number | Max records to return |
| Limit query result data size | Max bytes to return |

### Power BI Best Practices

- **Travel light**: Only bring data needed for reports
- **Composite model**: Combine aggregated and raw data
- **Dimension tables**: Use Import mode (small, stable)
- **Fact tables**: Use DirectQuery mode (large, raw)
- **Parallelism**: Increase concurrent DirectQuery connections
- **Weak consistency**: Enable for better performance (1-2 min lag)
- **Sync slicers**: Prevent premature data loading
- **Effective filters**: Focus ADX search on relevant shards

---

## Grafana Integration

### Setup Options

| Environment | Authentication |
|-------------|----------------|
| **Azure Managed Grafana** | System-assigned managed identity |
| **Self-hosted Grafana** | Service principal |

### Azure Managed Grafana Setup

1. Add managed identity to ADX database Viewer role
2. In Grafana: Settings > Data Sources > Azure Data Explorer Datasource
3. Enter cluster URL
4. Save & test

### Self-hosted Grafana Setup

1. Create service principal with Reader role
2. Add service principal to ADX database Viewer role
3. Configure data source with:
   - Cluster URL
   - Tenant ID
   - Client ID (Application ID)
   - Client secret (Password)

### Query Optimization Settings

| Setting | Description |
|---------|-------------|
| **Query Results Cache** | Cache results for specified duration |
| **Cache Max Age** | Duration to keep cached results |
| **Data Consistency** | Strong (default) or Weak (better performance) |

### Grafana Query Modes

**Query Builder Mode:**
- Visual interface for building queries
- Select database, table, filters
- Auto-generates KQL

**Raw Mode:**
- Write KQL directly
- Full query flexibility
- Edit KQL button to switch modes

### Creating Alerts

1. Alerting > Notification channels > Create channel
2. On dashboard panel: Edit > Alert tab > Create Alert
3. Configure alert conditions and notification

---

## Geospatial Visualizations

### Map Rendering

```kusto
// Points on map
StormEvents
| project BeginLon, BeginLat
| render scatterchart with (kind = map)

// Points with series
StormEvents
| project BeginLon, BeginLat, EventType
| render scatterchart with (kind = map)

// Bubble map (aggregated)
StormEvents
| summarize count() by hash = geo_point_to_s2cell(BeginLon, BeginLat)
| project geo_s2cell_to_central_point(hash), count_
| render piechart with (kind = map)
```

### GeoJSON Support

```kusto
// Using GeoJSON points
StormEvents
| project geo_point_to_s2cell(BeginLon, BeginLat, 5)
| project geo_s2cell_to_central_point(hash)
| render scatterchart with (kind = map)
```

---

## Plotly Visualization (Preview)

Supports ~80 chart types including 3D, animations, scientific, and financial charts.

### Methods

**Method 1: Python Plugin**
```kusto
OccupancyDetection
| evaluate python(typeof(plotly:string),
```if 1:
    import plotly.express as px
    fig = px.scatter_3d(df, x='Temperature', y='Humidity', z='CO2', color='Occupancy')
    plotly_obj = fig.to_json()
    result = pd.DataFrame(data = [plotly_obj], columns = ["plotly"])
```)
```

**Method 2: Preprepared Templates**
- `plotly_anomaly_fl()`: Anomaly visualization
- `plotly_scatter3d_fl()`: 3D scatter plots

```kusto
Iris
| invoke plotly_scatter3d_fl(x_col='SepalLength', y_col='PetalLength',
    z_col='SepalWidth', aggr_col='Class')
| render plotly
```

---

## Additional Visualization Tools

| Tool | Connection Method |
|------|-------------------|
| **Tableau** | ODBC connector |
| **Qlik** | ODBC connector |
| **Sisense** | JDBC connector |
| **Redash** | Native ADX data source |
| **Excel** | Native Excel connector |
| **Kibana** | K2Bridge open source connector |

---

## Documentation References

- Main dashboards: `azure-data-explorer-dashboards.md`
- Render operator: `kusto/query/render-operator.md`
- Dashboard visuals: `dashboard-visuals.md`
- Dashboard parameters: `dashboard-parameters.md`
- Power BI connector: `power-bi-data-connector.md`
- Grafana: `grafana.md`
- Geospatial: `kusto/query/geospatial-visualizations.md`
- Plotly: `kusto/query/visualization-plotly.md`

---

*Generated from Azure Data Explorer documentation analysis - Phase 3*
