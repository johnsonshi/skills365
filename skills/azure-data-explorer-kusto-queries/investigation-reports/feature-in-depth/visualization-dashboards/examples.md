# Visualization and Dashboards Examples

## Overview

This document provides practical examples for creating dashboards, using render operators, configuring parameters, and integrating with Power BI and Grafana.

---

## Dashboard Creation Examples

### Example 1: Basic Dashboard Setup

**Step 1: Create Dashboard**
```
1. Navigate to Dashboards > New dashboard
2. Enter name: "Storm Events Analysis"
3. Click Create
```

**Step 2: Add Data Source**
```
1. Click Data sources in toolbar
2. Click + Add
3. Data source name: "StormEventsDB"
4. Cluster URI: https://help.kusto.windows.net
5. Database: Samples
6. Click Create
```

**Step 3: Add First Tile**
```kusto
// Events by state - bar chart
StormEvents
| where StartTime > ago(365d)
| summarize EventCount = count() by State
| top 10 by EventCount
```

Visual type: Bar chart

### Example 2: Multi-Page Dashboard

**Page 1: Executive Summary**
```kusto
// Total events card
StormEvents
| where StartTime between (_startTime.._endTime)
| count

// Events trend timechart
StormEvents
| where StartTime between (_startTime.._endTime)
| summarize Events = count() by bin(StartTime, 1d)
| render timechart

// Damage summary pie chart
StormEvents
| where StartTime between (_startTime.._endTime)
| summarize TotalDamage = sum(DamageProperty + DamageCrops) by EventType
| top 5 by TotalDamage
| render piechart
```

**Page 2: Detailed Analysis**
```kusto
// Events table with details
StormEvents
| where StartTime between (_startTime.._endTime)
| where State == _selectedState
| project StartTime, State, EventType, DamageProperty, DamageCrops
| order by StartTime desc
| take 100

// State comparison column chart
StormEvents
| where StartTime between (_startTime.._endTime)
| where State in (_selectedStates)
| summarize Events = count() by State
| render columnchart
```

---

## Render Operator Examples

### Basic Chart Types

**Timechart with Decomposition:**
```kusto
let min_t = datetime(2007-01-01);
let max_t = datetime(2007-12-31);
StormEvents
| where StartTime between (min_t..max_t)
| summarize Events = count() by bin(StartTime, 1d)
| extend (baseline, seasonal, trend, residual) = series_decompose(Events)
| render timechart with (title='Storm Events Decomposition')
```

**Stacked Column Chart:**
```kusto
StormEvents
| where StartTime between (datetime(2007-01-01)..datetime(2007-12-31))
| summarize Events = count() by bin(StartTime, 30d), EventType
| render columnchart with (kind=stacked, title='Monthly Events by Type')
```

**Multi-Panel Timechart:**
```kusto
StormEvents
| where State in ("TEXAS", "FLORIDA", "CALIFORNIA")
| summarize Events = count() by State, bin(StartTime, 7d)
| render timechart with (ysplit=panels, title='Weekly Events by State')
```

**Scatter Chart:**
```kusto
StormEvents
| where DamageProperty > 0 and DamageCrops > 0
| project DamageProperty, DamageCrops, EventType
| render scatterchart with (
    title='Property vs Crop Damage',
    xtitle='Property Damage ($)',
    ytitle='Crop Damage ($)'
)
```

### Anomaly Chart

```kusto
let min_t = datetime(2007-01-01);
let max_t = datetime(2007-12-31);
StormEvents
| where StartTime between (min_t..max_t)
| summarize Events = count() by bin(StartTime, 1d)
| render anomalychart with (
    title='Storm Events with Anomaly Detection',
    anomalycolumns=Events
)
```

### Card Visualization

```kusto
// Single KPI card
StormEvents
| where StartTime > ago(30d)
| summarize
    TotalEvents = count(),
    TotalDamage = sum(DamageProperty),
    AffectedStates = dcount(State)
| project Metric = "Total Events", Value = TotalEvents
| render card
```

### Treemap

```kusto
StormEvents
| summarize Events = count() by State, EventType
| render treemap with (title='Events by State and Type')
```

---

## Geographic Visualization Examples

### Points on Map

```kusto
// Simple point map
StormEvents
| where BeginLat != 0 and BeginLon != 0
| take 1000
| project BeginLon, BeginLat
| render scatterchart with (kind=map)
```

### Colored Points by Category

```kusto
// Points colored by event type
StormEvents
| where BeginLat != 0 and BeginLon != 0
| take 500
| project BeginLon, BeginLat, EventType
| render scatterchart with (kind=map, title='Storm Events by Type')
```

### Bubble Map with Aggregation

```kusto
// Aggregated bubbles using S2 cells
StormEvents
| where BeginLat != 0 and BeginLon != 0
| summarize Events = count() by hash = geo_point_to_s2cell(BeginLon, BeginLat, 5)
| project geo_s2cell_to_central_point(hash), Events
| render piechart with (kind=map, title='Event Density')
```

### Pie Charts on Map

```kusto
// Event type distribution by location
StormEvents
| where BeginLat != 0 and BeginLon != 0
| where geo_point_in_circle(BeginLon, BeginLat, -95.0, 37.0, 500000)
| summarize Events = count() by EventType, hash = geo_point_to_s2cell(BeginLon, BeginLat, 4)
| project geo_s2cell_to_central_point(hash), EventType, Events
| render piechart with (kind=map)
```

---

## Dashboard Parameter Examples

### Time Range Parameter

**Query using built-in time range:**
```kusto
StormEvents
| where StartTime between (_startTime.._endTime)
| summarize Events = count() by bin(StartTime, 1h)
| render timechart
```

### Single Selection Parameter

**Parameter Configuration:**
- Label: Event Type
- Variable name: `_eventType`
- Parameter type: Single selection
- Source: Fixed values
- Values: Thunderstorm Wind, Hail, Flash Flood, Tornado

**Query using parameter:**
```kusto
StormEvents
| where StartTime between (_startTime.._endTime)
| where EventType == _eventType
| summarize Events = count() by State
| top 10 by Events
| render barchart
```

### Multiple Selection Parameter

**Parameter Configuration:**
- Label: States
- Variable name: `_selectedStates`
- Parameter type: Multiple selection
- Source: Query
- Query:
```kusto
StormEvents
| distinct State
| order by State asc
```

**Query using parameter:**
```kusto
StormEvents
| where StartTime between (_startTime.._endTime)
| where State in (_selectedStates) or isempty(_selectedStates)
| summarize Events = count() by State
| render columnchart
```

### Cascading Parameters

**First parameter (Region):**
```kusto
// Source query for Region parameter
StormEvents
| extend Region = case(
    State in ("CALIFORNIA", "OREGON", "WASHINGTON"), "West",
    State in ("TEXAS", "FLORIDA", "GEORGIA"), "South",
    State in ("NEW YORK", "NEW JERSEY", "MASSACHUSETTS"), "Northeast",
    "Other")
| distinct Region
```

**Second parameter (State) - filtered by Region:**
```kusto
// Source query for State parameter
StormEvents
| extend Region = case(
    State in ("CALIFORNIA", "OREGON", "WASHINGTON"), "West",
    State in ("TEXAS", "FLORIDA", "GEORGIA"), "South",
    State in ("NEW YORK", "NEW JERSEY", "MASSACHUSETTS"), "Northeast",
    "Other")
| where Region == _selectedRegion or isempty(_selectedRegion)
| distinct State
| order by State asc
```

### Free Text Parameter

**Parameter Configuration:**
- Label: Search Term
- Variable name: `_searchTerm`
- Parameter type: Free text
- Data type: String

**Query using parameter:**
```kusto
StormEvents
| where StartTime between (_startTime.._endTime)
| where EventNarrative contains _searchTerm or isempty(_searchTerm)
| project StartTime, State, EventType, EventNarrative
| take 100
```

---

## Cross-Filter Examples

### Setup Cross-Filter

**Tile 1: State Overview (enable cross-filter)**
```kusto
StormEvents
| where StartTime between (_startTime.._endTime)
| summarize Events = count() by State
| top 10 by Events
```

Configuration:
- Visual > Interactions > Cross-filters: On
- Interaction type: Point
- Column: State
- Parameter: _selectedState

**Tile 2: Event Details (filtered by cross-filter)**
```kusto
StormEvents
| where StartTime between (_startTime.._endTime)
| where State == _selectedState or isempty(_selectedState)
| summarize Events = count() by EventType
| render piechart
```

### Drag-to-Select Time Range

**Time series with drag cross-filter:**
```kusto
StormEvents
| summarize Events = count() by bin(StartTime, 1d)
| render timechart
```

Configuration:
- Interaction type: Drag
- Column: StartTime
- Parameter: _timeRange

---

## Drillthrough Examples

### Summary to Detail Drillthrough

**Page 1: Summary (source)**
```kusto
// State summary tile with drillthrough enabled
StormEvents
| where StartTime between (_startTime.._endTime)
| summarize
    Events = count(),
    TotalDamage = sum(DamageProperty)
by State
| top 20 by Events
| render barchart
```

Drillthrough configuration:
- Destination page: StateDetails
- Column: State
- Parameter: _drillState

**Page 2: StateDetails (target)**
```kusto
// Events over time for selected state
StormEvents
| where StartTime between (_startTime.._endTime)
| where State == _drillState
| summarize Events = count() by bin(StartTime, 1d)
| render timechart with (title=strcat('Events in ', _drillState))

// Event type breakdown
StormEvents
| where StartTime between (_startTime.._endTime)
| where State == _drillState
| summarize Events = count() by EventType
| render piechart

// Event details table
StormEvents
| where StartTime between (_startTime.._endTime)
| where State == _drillState
| project StartTime, EventType, DamageProperty, DamageCrops, BeginLocation
| order by StartTime desc
| take 50
```

---

## Power BI Query Examples

### Export Query from ADX

```kusto
// Query to export to Power BI
StormEvents
| where StartTime between (datetime(2007-01-01)..datetime(2007-12-31))
| summarize
    TotalEvents = count(),
    TotalPropertyDamage = sum(DamageProperty),
    TotalCropDamage = sum(DamageCrops)
by State, EventType
| order by TotalEvents desc
```

**Export Steps:**
1. Run query in ADX web UI
2. Click Export > Query to Power BI
3. Open Power BI Desktop
4. Transform data > Paste query

### DirectQuery Optimized Query

```kusto
// Efficient query for DirectQuery mode
let selectedDate = todatetime(_selectedDate);
StormEvents
| where StartTime between (selectedDate..selectedDate+1d)
| summarize Events = count() by bin(StartTime, 1h), State
| project Timestamp = StartTime, State, EventCount = Events
```

### Import Mode with Aggregations

```kusto
// Pre-aggregated for Import mode
StormEvents
| summarize
    DailyEvents = count(),
    MaxDamage = max(DamageProperty),
    AvgDamage = avg(DamageProperty)
by Date = startofday(StartTime), State, EventType
```

---

## Grafana Query Examples

### Query Builder Mode Steps

1. Select Database: Samples
2. From: StormEvents
3. Where (filter):
   - Column: StartTime
   - Operator: >
   - Value: ago(30d)
4. Value columns:
   - Column: count()
   - Aggregation: -
5. Group by: bin(StartTime, 1d)

### Raw KQL Mode

```kusto
// Events by day
StormEvents
| where StartTime > ago(30d)
| summarize Events = count() by bin(StartTime, 1d)
```

### Grafana Variables

**Query for dropdown variable:**
```kusto
// Returns list of states for dropdown
StormEvents
| where StartTime > ago(365d)
| distinct State
| order by State asc
```

**Using variable in query:**
```kusto
StormEvents
| where StartTime > ago(30d)
| where State == '$state'
| summarize Events = count() by bin(StartTime, 1d)
```

### Alert Query

```kusto
// Query for alert threshold
StormEvents
| where StartTime > ago(1h)
| summarize Events = count()
| where Events > 100
```

---

## Conditional Formatting Examples

### Table with Color Conditions

**Query:**
```kusto
StormEvents
| where StartTime between (_startTime.._endTime)
| summarize
    Events = count(),
    TotalDamage = sum(DamageProperty)
by State
| order by TotalDamage desc
| take 20
```

**Conditional Formatting Rules:**

Rule 1: High Damage
- Rule type: Color by condition
- Column: TotalDamage
- Operator: >
- Value: 1000000
- Color: Red
- Tag: Critical

Rule 2: Medium Damage
- Rule type: Color by condition
- Column: TotalDamage
- Operator: between
- Value: 100000 and 1000000
- Color: Yellow
- Tag: Warning

### Heatmap Color by Value

**Query:**
```kusto
StormEvents
| where StartTime between (_startTime.._endTime)
| extend Hour = hourofday(StartTime)
| extend DayOfWeek = dayofweek(StartTime) / 1d
| summarize Events = count() by Hour, DayOfWeek
| project X = Hour, Y = DayOfWeek, Value = Events
```

**Conditional Formatting:**
- Rule type: Color by value
- Column: Value
- Theme: Warm
- Min value: 0
- Max value: (auto)

---

## Plotly Advanced Examples

### 3D Scatter Plot with Python Plugin

```kusto
OccupancyDetection
| project Temperature, Humidity, CO2, Occupancy
| where rand() < 0.1
| evaluate python(typeof(plotly:string),
```if 1:
    import plotly.express as px
    fig = px.scatter_3d(df, x='Temperature', y='Humidity', z='CO2',
                        color='Occupancy',
                        title='Occupancy Detection - 3D View')
    fig.update_layout(scene=dict(
        xaxis_title='Temperature',
        yaxis_title='Humidity',
        zaxis_title='CO2'))
    plotly_obj = fig.to_json()
    result = pd.DataFrame(data=[plotly_obj], columns=['plotly'])
```)
| render plotly
```

### Anomaly Chart with Plotly Template

```kusto
let dt = 1h;
demo_make_series2
| make-series num=avg(num) on TimeStamp from datetime(2017-01-05) to datetime(2017-02-03) step dt
| extend (anomalies, score, baseline) = series_decompose_anomalies(num, 1.5)
| invoke plotly_anomaly_fl('TimeStamp', 'num', 'anomalies', 'Anomaly Detection')
| render plotly
```

### Using plotly_scatter3d_fl

```kusto
Iris
| invoke plotly_scatter3d_fl(
    x_col='SepalLength',
    y_col='PetalLength',
    z_col='SepalWidth',
    aggr_col='Class',
    chart_title='Iris Dataset - 3D Scatter')
| render plotly
```

---

## Dashboard JSON Export Example

```json
{
  "title": "Storm Events Dashboard",
  "tiles": [
    {
      "title": "Events Over Time",
      "query": "StormEvents | where StartTime between (_startTime.._endTime) | summarize Events = count() by bin(StartTime, 1d)",
      "visualType": "timechart",
      "layout": { "x": 0, "y": 0, "width": 12, "height": 4 }
    },
    {
      "title": "Top States",
      "query": "StormEvents | where StartTime between (_startTime.._endTime) | summarize Events = count() by State | top 10 by Events",
      "visualType": "barchart",
      "layout": { "x": 0, "y": 4, "width": 6, "height": 4 }
    },
    {
      "title": "Event Types",
      "query": "StormEvents | where StartTime between (_startTime.._endTime) | summarize Events = count() by EventType | top 5 by Events",
      "visualType": "piechart",
      "layout": { "x": 6, "y": 4, "width": 6, "height": 4 }
    }
  ],
  "parameters": [
    {
      "label": "Time Range",
      "variableName": "_startTime",
      "type": "timerange"
    },
    {
      "label": "State",
      "variableName": "_selectedState",
      "type": "single_selection",
      "source": "query",
      "query": "StormEvents | distinct State | order by State"
    }
  ],
  "autoRefresh": {
    "enabled": true,
    "defaultInterval": "15m",
    "minInterval": "5m"
  }
}
```

---

## Auto-Refresh Configuration Example

**For Operational Dashboard:**
```
Minimum time interval: 1m
Default refresh rate: 5m
```

**For Analytical Dashboard:**
```
Minimum time interval: 15m
Default refresh rate: 1h
```

**Disable for Heavy Queries:**
```
Auto refresh: Disabled
```

---

*Generated from Azure Data Explorer documentation analysis - Phase 3*
