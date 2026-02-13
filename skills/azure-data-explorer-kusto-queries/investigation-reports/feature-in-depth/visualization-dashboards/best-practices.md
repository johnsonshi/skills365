# Visualization and Dashboards Best Practices

## Overview

This document provides best practices for designing effective Azure Data Explorer dashboards, choosing appropriate visualizations, and optimizing performance.

---

## Dashboard Design Principles

### 1. Purpose-Driven Design

**Define Clear Objectives:**
- Identify primary questions the dashboard should answer
- Target specific user personas (executives, analysts, operators)
- Limit each page to a single theme or workflow

**Information Hierarchy:**
- Place most critical metrics at top-left
- Use larger tiles for primary KPIs
- Group related visualizations together
- Use pages to separate concerns

### 2. Layout Guidelines

**Tile Organization:**
- Maximum 6-8 tiles per page for readability
- Maintain consistent tile sizes where possible
- Leave whitespace for visual breathing room
- Use pages for different analysis depths (summary vs. detail)

**Responsive Considerations:**
- Test dashboards at different screen sizes
- Prioritize horizontal layouts for wide screens
- Consider mobile viewing if needed

### 3. Naming Conventions

**Dashboard Names:**
- Use descriptive names: `Sales-Performance-Q4-2024`
- Include scope and timeframe when relevant
- Avoid generic names like "Dashboard 1"

**Tile Names:**
- Be specific: "Daily Active Users by Region" not "DAU"
- Include aggregation method if not obvious
- Keep under 50 characters

---

## Visualization Selection Guide

### Choose the Right Chart Type

| Data Pattern | Recommended Visual | Avoid |
|--------------|-------------------|-------|
| Trend over time | timechart, linechart | piechart |
| Part-to-whole | piechart, treemap | linechart |
| Comparison | barchart, columnchart | piechart |
| Distribution | scatterchart, heatmap | linechart |
| Correlation | scatterchart | barchart |
| Single KPI | card, stat | timechart |
| Geographic | map (scatterchart with kind=map) | table |
| Hierarchical | treemap, funnel | linechart |
| Anomalies | anomalychart | columnchart |

### Visualization-Specific Tips

**Timecharts:**
```kusto
// Good: Appropriate time binning
| summarize count() by bin(Timestamp, 1h)
| render timechart

// Bad: Too granular for long periods
| summarize count() by bin(Timestamp, 1s)  // Performance issue
```

**Piecharts:**
- Limit to 5-7 slices maximum
- Use "Other" category for small values
- Avoid when comparing precise values

```kusto
// Good: Limited categories
| summarize Count = count() by EventType
| top 5 by Count
| render piechart
```

**Bar/Column Charts:**
- Sort by value when ranking matters
- Sort alphabetically when scanning for specific items
- Use stacked variants for part-to-whole with categories

```kusto
// Good: Sorted for ranking
| summarize Revenue = sum(Amount) by Product
| order by Revenue desc
| render barchart
```

**Heatmaps:**
- Use for discovering patterns across two dimensions
- Choose color palettes with clear gradients
- Specify min/max values for consistent scaling

---

## Parameter Best Practices

### Design Effective Filters

**Parameter Naming:**
```
// Good: Descriptive with underscore prefix
Variable name: _eventType
Label: Event Type

// Bad: Cryptic abbreviations
Variable name: et
```

**Parameter Types Selection:**

| Scenario | Parameter Type |
|----------|----------------|
| Exact value selection | Single selection |
| Multiple filters at once | Multiple selection |
| Range filtering | Time range |
| User-defined input | Free text |
| Data source switching | Data source parameter |

### Query-Based Parameters

**Performance Optimization:**
```kusto
// Good: Limited parameter values with filtering
StormEvents
| where StartTime > ago(30d)  // Limit scope
| summarize by State
| top 20 by State  // Limit options
| project State
```

**Cascading Parameters:**
```kusto
// State parameter depends on selected EventType
StormEvents
| where EventType in (_eventType) or isempty(_eventType)
| summarize by State
| project State
```

### Default Time Range

Always use `_startTime` and `_endTime` parameters:
```kusto
StormEvents
| where StartTime between (_startTime.._endTime)
| summarize TotalEvents = count() by State
```

---

## Cross-Filtering and Drillthrough

### Cross-Filter Design

**When to Use:**
- Exploring relationships between visuals
- Interactive data discovery
- Same-page filtering

**Configuration:**
1. Enable cross-filters in Visual > Interactions
2. Specify interaction type: Point or Drag
3. Map column to parameter

**Best Practices:**
- Use parameters of matching data types
- Provide clear visual feedback on selection
- Include reset button instruction in dashboard notes

### Drillthrough Design

**When to Use:**
- Moving from summary to detail views
- Different analysis contexts
- Multi-page navigation

**Setup Checklist:**
1. Create target page with detail visualizations
2. Add parameters for drillthrough values
3. Configure drillthrough on source visual
4. Test navigation flow

---

## Performance Optimization

### Query-Level Optimization

**Pre-Aggregate in Queries:**
```kusto
// Good: Aggregate before visualization
StormEvents
| summarize DailyCount = count() by bin(StartTime, 1d), State
| render timechart

// Bad: Let visualization aggregate raw data
StormEvents
| project StartTime, State
| render timechart
```

**Limit Data Volume:**
```kusto
// Good: Filter early, limit results
StormEvents
| where StartTime > ago(7d)
| where State in (_selectedStates)
| summarize count() by EventType
| top 10 by count_
```

**Use Parameters Early:**
```kusto
// Good: Filter with parameter at start
StormEvents
| where State == _state  // Filter immediately
| summarize count() by EventType

// Bad: Filter after aggregation
StormEvents
| summarize count() by EventType, State
| where State == _state  // Late filtering
```

### Dashboard-Level Optimization

**Auto-Refresh Settings:**
- Set appropriate minimum intervals (5m+)
- Disable for dashboards with heavy queries
- Use longer defaults for non-critical dashboards

**Query Results Caching:**
```json
"Query results cache max age": "5m"
```
- Enable for queries that don't need real-time data
- Balance freshness vs. performance
- Use data source-level caching configuration

**Page Loading:**
- Use pages to separate heavy visualizations
- Load on-demand rather than all at once
- Consider parameter-driven data loading

---

## Power BI Best Practices

### Data Loading Strategy

**For Large Datasets:**
```
Dimension tables -> Import mode (small, static)
Fact tables -> DirectQuery mode (large, dynamic)
```

**Composite Model Pattern:**
1. Import aggregated summaries for top-level views
2. Use DirectQuery for drill-down to raw data
3. Create relationships between import and DirectQuery tables

### Query Design for Power BI

**Optimize at Source:**
```kusto
// Good: Return only needed columns and rows
StormEvents
| where StartTime > ago(30d)
| summarize EventCount = count() by bin(StartTime, 1d), State
| project Date = StartTime, State, EventCount
```

**Avoid in Power BI:**
- Complex DAX calculations on large DirectQuery tables
- Real-time filters without query reduction enabled
- Loading more data than needed for visualization

### Performance Settings

**Query Reduction:**
- Enable "Reduce number of queries sent by" option
- Use "Apply" button for slicer changes
- Disable cross-filtering on heavy visuals

**Concurrent Connections:**
- Increase from default 10 for better parallelism
- Monitor cluster load when increasing

---

## Grafana Best Practices

### Query Optimization

**Enable Results Cache:**
```
Query Optimizations > Use dynamic caching: Off
Cache Max Age: 5 (minutes)
```

**Weak Consistency:**
- Enable for improved rendering performance
- Accept 1-2 minute data lag
- Use for dashboards where real-time isn't critical

### Dashboard Design

**Variable (Parameter) Design:**
```
// Use query-based variables for dynamic options
StormEvents
| distinct State
| order by State asc
```

**Alert Configuration:**
- Set meaningful thresholds based on baseline data
- Use appropriate notification channels
- Avoid alert fatigue with proper intervals

---

## Conditional Formatting Best Practices

### Color by Condition

**Rule Design:**
- Use intuitive colors (red=bad, green=good)
- Limit to 3-4 rules per visual
- Order rules from most to least specific

**Example Configuration:**
| Condition | Color | Use Case |
|-----------|-------|----------|
| Value > 90% | Red | Critical threshold |
| Value > 75% | Yellow | Warning threshold |
| Value <= 75% | Green | Normal range |

### Color by Value

**Theme Selection:**
- Traffic lights: Performance/status metrics
- Cold/Warm: Temperature, intensity
- Single color: Simple high/low gradients

**Range Configuration:**
- Set explicit min/max for consistent scaling
- Use reverse colors when lower is better

---

## Accessibility Considerations

### Color Choices

- Don't rely solely on color to convey meaning
- Use high-contrast color combinations
- Test with color blindness simulators
- Include legends and labels

### Text and Labels

- Use readable font sizes (12pt minimum)
- Provide descriptive titles and axis labels
- Include units in labels (%, ms, count)
- Avoid abbreviations without explanation

---

## Security Best Practices

### Permission Management

**Least Privilege:**
- Grant "Can view" by default
- Reserve "Can edit" for dashboard maintainers
- Review permissions periodically

**Data Access:**
- Dashboard permissions don't grant data access
- Users need both dashboard AND database permissions
- Consider row-level security for sensitive data

### Cross-Tenant Sharing

- Only enable when necessary
- Monitor shared dashboard access
- Set appropriate invitation expiry
- Revoke access promptly when no longer needed

---

## Maintenance Best Practices

### Version Control

- Export dashboard JSON regularly
- Store in source control with meaningful commits
- Document significant changes

### Testing

- Test after adding new tiles or parameters
- Verify performance with production data volumes
- Check mobile/different screen size rendering

### Documentation

- Include README page or tile with dashboard purpose
- Document parameter meanings
- Note data refresh schedules and limitations

---

*Generated from Azure Data Explorer documentation analysis - Phase 3*
