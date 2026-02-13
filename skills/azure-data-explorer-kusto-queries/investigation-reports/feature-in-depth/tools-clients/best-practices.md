# Azure Data Explorer Tools and Clients Best Practices

## Overview

This guide provides tool selection guidance, keyboard shortcuts for productivity, and workflow optimization tips for Azure Data Explorer tools.

---

## 1. Tool Selection Guide

### Decision Matrix: Which Tool to Use

| Use Case | Recommended Tool | Rationale |
|----------|------------------|-----------|
| **Ad-hoc data exploration** | Web UI | Quick access, no installation, visual results |
| **Complex query development** | Kusto.Explorer | Full IntelliSense, code refactoring, multiple tabs |
| **Automated batch operations** | Kusto.Cli | Scriptable, CI/CD friendly, no GUI required |
| **Local development/testing** | Kusto Emulator | Free, offline, fast iteration |
| **Custom application embedding** | Monaco-Kusto | Full flexibility, branded experience |
| **Schema conversion** | Avrotize | Automated schema migration |

### Environment-Based Recommendations

#### Development Environment
- **Primary**: Kusto.Explorer for query writing and testing
- **Secondary**: Kusto Emulator for local integration tests
- **Support**: Monaco-Kusto for custom tool development

#### Production Monitoring
- **Primary**: Web UI dashboards for visualization
- **Secondary**: Kusto.Explorer for deep-dive investigations
- **Automation**: Kusto.Cli for scheduled reports

#### CI/CD Pipelines
- **Primary**: Kusto.Cli for automated testing
- **Secondary**: Kusto Emulator as test environment
- **Validation**: Scripts for schema verification

---

## 2. Kusto.Explorer Productivity Tips

### Essential Keyboard Shortcuts

#### Application-Level Shortcuts

| Action | Shortcut | Use Case |
|--------|----------|----------|
| **Toggle full view** | `F11` | Maximize workspace |
| **New query tab** | `Ctrl+N` | Start new query |
| **Close tab** | `Ctrl+W` | Clean up workspace |
| **Save query** | `Ctrl+S` | Preserve work |
| **Cancel query** | `ESC` or `Shift+F5` | Stop long-running queries |
| **Open options** | `Ctrl+Shift+O` | Configure settings |
| **Toggle theme** | `Shift+F12` | Switch light/dark mode |
| **Switch tabs** | `Ctrl+1` to `Ctrl+7` | Navigate query documents |

#### Query Execution Shortcuts

| Action | Shortcut | Description |
|--------|----------|-------------|
| **Run query** | `F5` or `Shift+Enter` | Execute selected query |
| **Preview results** | `Ctrl+F5` | Show few results with count |
| **Fetch from cache** | `F8` | Reuse previous results |
| **Force IntelliSense** | `Ctrl+Space` | Show completion options |
| **New line with pipe** | `Ctrl+Enter` | Add pipe continuation |

#### Code Editing Shortcuts

| Action | Shortcut | Description |
|--------|----------|-------------|
| **Prettify query** | `Alt+Shift+F` or `Ctrl+K, Ctrl+F` | Format query |
| **Comment lines** | `Ctrl+K, Ctrl+C` | Add comments |
| **Uncomment lines** | `Ctrl+K, Ctrl+U` | Remove comments |
| **Go to line** | `Ctrl+G` | Navigate to specific line |
| **Find** | `Ctrl+F` | Search in query |
| **Replace** | `Ctrl+H` | Find and replace |
| **Rename symbol** | `Ctrl+R, Ctrl+R` | Refactor variable names |
| **Extract to let** | `Alt+Ctrl+M` or `Ctrl+.` | Create let statement |
| **Go to definition** | `F12` | Navigate to symbol |
| **Find references** | `Ctrl+F12` | Find all usages |

#### Results Handling Shortcuts

| Action | Shortcut | Description |
|--------|----------|-------------|
| **Copy query + results** | `Ctrl+Shift+C` | Copy with deep link |
| **Column chart** | `Alt+C` | Visualize as columns |
| **Time chart** | `Alt+T` | Visualize timeline |
| **Pie chart** | `Alt+P` | Visualize as pie |
| **Pivot chart** | `Alt+V` | Create pivot |
| **Toggle result panel** | `Ctrl+J` | Show/hide results |

### Productivity Workflows

#### Efficient Query Development

1. **Use Search++ Mode** for initial data exploration
   - Select Home > Query dropdown > Search++
   - Search across multiple tables to understand data distribution
   - Use heat maps to identify relevant tables

2. **Leverage IntelliSense**
   - Type table names for auto-complete
   - Press `Ctrl+Space` for context-aware suggestions
   - Hover over functions for inline documentation

3. **Use Code Analyzer**
   - Check the Issues tab after running queries
   - Apply suggested optimizations
   - Run `Ctrl+F6` for static analysis

4. **Manage Multiple Queries**
   - Use tabs for different query contexts
   - Rename tabs (`Ctrl+F2`) for organization
   - Use Work Documents pane for quick access

#### Connection Management Best Practices

```kql
// Scripted connection setup
#connect cluster('help').database('Samples')

// Set query context before running scripts
StormEvents | take 10
```

- **Group connections** by environment (Dev, Test, Prod)
- **Export connections** before reinstalling
- **Use connection profiles** for team sharing

---

## 3. Web UI Productivity Tips

### Essential Keyboard Shortcuts

#### Query Editor Shortcuts

| Action | Windows | Mac |
|--------|---------|-----|
| **Run query** | `Shift+Enter` | `Shift+Enter` |
| **Cancel query** | `Ctrl+Shift+Enter` | `Cmd+Shift+Enter` |
| **New tab** | `Ctrl+J` | `Cmd+J` |
| **Find** | `Ctrl+F` | `Cmd+F` |
| **Format all** | `Shift+Alt+F` | `Shift+Option+F` |
| **Copy link** | `Shift+Alt+L` | `Shift+Option+L` |
| **Toggle comment** | `Ctrl+/` | `Cmd+/` |
| **Fold all** | `Ctrl+K, Ctrl+0` | `Cmd+K, Cmd+0` |
| **Unfold all** | `Ctrl+K, Ctrl+J` | `Cmd+K, Cmd+J` |
| **Recall result** | `F8` | `F8` |
| **Show palette** | `F1` | `F1` |

#### Results Grid Shortcuts

| Action | Shortcut |
|--------|----------|
| **Add filter from cell** | `Ctrl+Shift+Space` |
| **Toggle column sort** | `Enter` (on header) |
| **Open column menu** | `Shift+Enter` (on header) |

### Productivity Workflows

#### Efficient Data Exploration

1. **Use Data Profile** for quick insights
   - Right-click table in connection pane
   - View column statistics and top values
   - Identify data patterns without queries

2. **Leverage Quick Fix**
   - Watch for underlined suggestions
   - Press `Ctrl+.` for quick fix options
   - Extract values to variables for reusability

3. **Use Tab Management**
   - Rename tabs by double-clicking
   - Each tab maintains its own context
   - Use tabs list (top right) for overview

4. **Share Effectively**
   - Use **KQL tools > Copy link** for sharing
   - Pin important queries to dashboards
   - Export results to Excel for stakeholders

---

## 4. Kusto.Cli Productivity Tips

### Efficient Script Patterns

#### Basic Script Structure

```bash
// Enable echo for debugging
#echo:true

// Set database context
#dbcontext MyDatabase

// Run queries
StormEvents | count

// Save results
#save output.csv
StormEvents | take 100
```

#### Batch Processing Pattern

```bash
// Process multiple queries efficiently
Kusto.Cli.exe "https://cluster.kusto.windows.net/MyDB;Fed=true" ^
  -execute:"#save result1.csv" ^
  -execute:"Table1 | count" ^
  -execute:"#save result2.csv" ^
  -execute:"Table2 | summarize count() by State"
```

#### CI/CD Script Pattern

```bash
// Script with error handling (recommended for block mode)
Kusto.Cli.exe "https://cluster.kusto.windows.net/MyDB;Fed=true" ^
  -script:"queries.kql" ^
  -scriptQuitOnError:true ^
  -transcript:"output.log" ^
  -lineMode:false
```

### Best Practices

| Practice | Recommendation |
|----------|----------------|
| **Use block mode for scripts** | `-lineMode:false` prevents newline issues |
| **Quote connection strings** | Prevents shell interpretation of semicolons |
| **Enable transcripts** | `-transcript:file.log` for debugging |
| **Use save directive** | `#save` before each query for CSV export |

---

## 5. Kusto Emulator Productivity Tips

### Development Workflow

#### Initial Setup

```powershell
# Create persistent data directory
mkdir C:\kustodata

# Start emulator with persistence
docker run -v C:\kustodata:/kustodata -e ACCEPT_EULA=Y -m 4G -d -p 8080:8080 -t mcr.microsoft.com/azuredataexplorer/kustainer-linux:latest
```

#### Database Setup Script

```kql
// Create database (run once)
.create database TestDB persist (
    @"/kustodata/dbs/TestDB/md",
    @"/kustodata/dbs/TestDB/data"
)

// Create test table
.create table Events (
    Timestamp: datetime,
    EventType: string,
    Value: real
)

// Ingest test data
.ingest into table Events (@"/kustodata/testdata.csv")
```

### Testing Patterns

#### Unit Test Setup

```powershell
# Start fresh container for test isolation
docker run --name kusto-test -e ACCEPT_EULA=Y -m 4G -d -p 8080:8080 -t mcr.microsoft.com/azuredataexplorer/kustainer-linux:latest

# Run tests
# ... your test commands ...

# Cleanup
docker stop kusto-test
docker rm kusto-test
```

#### Integration Test Pattern

1. **Start emulator** in CI/CD pipeline
2. **Create database** with test schema
3. **Load fixture data** from test files
4. **Execute queries** and validate results
5. **Tear down** container

### Best Practices

| Practice | Recommendation |
|----------|----------------|
| **Mount volumes** | Persist data across container restarts |
| **Name containers** | Use `--name` for easier management |
| **Limit memory** | Use `-m 4G` to prevent resource exhaustion |
| **Use stable tag** | `:stable` for production-like testing |

---

## 6. Monaco-Kusto Integration Tips

### Performance Optimization

```javascript
// Lazy load schema for large databases
const workerAccessor = await monaco.languages.kusto.getKustoWorker();
const worker = await workerAccessor(model.uri);

// Set schema only when editor is focused
editor.onDidFocusEditorText(async () => {
    if (!schemaLoaded) {
        await worker.setSchemaFromShowSchema(schema, clusterURI, database);
        schemaLoaded = true;
    }
});
```

### Best Practices

| Practice | Recommendation |
|----------|----------------|
| **Cache schema** | Avoid re-fetching on every page load |
| **Use web workers** | Keep UI responsive during parsing |
| **Lazy load** | Load monaco-kusto only when needed |
| **Handle errors** | Wrap API calls in try-catch |

---

## 7. General Productivity Recommendations

### Query Development Workflow

1. **Explore**: Use Web UI or Search++ mode for initial exploration
2. **Develop**: Switch to Kusto.Explorer for complex query development
3. **Test**: Use Kusto Emulator for local validation
4. **Automate**: Convert to Kusto.Cli scripts for production
5. **Monitor**: Deploy to Web UI dashboards for visualization

### Performance Tips

| Tip | Benefit |
|-----|---------|
| **Use `take` first** | Preview data before full queries |
| **Add time filters early** | Reduce data scanned |
| **Use `project` to limit columns** | Reduce network transfer |
| **Cache frequent queries** | Use `F8` to recall results |
| **Monitor query statistics** | Optimize based on metrics |

### Collaboration Tips

| Tip | How |
|-----|-----|
| **Share queries via deep links** | `Ctrl+Shift+C` in Kusto.Explorer |
| **Export connection profiles** | File > Export Profile |
| **Use consistent formatting** | `Alt+Shift+F` before sharing |
| **Document complex queries** | Use comments (`//`) |

---

## Quick Reference Card

### Kusto.Explorer Most-Used Shortcuts

| Action | Shortcut |
|--------|----------|
| Run query | `F5` |
| New tab | `Ctrl+N` |
| Save | `Ctrl+S` |
| Format | `Alt+Shift+F` |
| Comment | `Ctrl+K, Ctrl+C` |
| Find | `Ctrl+F` |
| Cancel | `ESC` |
| Full screen | `F11` |

### Web UI Most-Used Shortcuts

| Action | Shortcut |
|--------|----------|
| Run query | `Shift+Enter` |
| New tab | `Ctrl+J` |
| Format | `Shift+Alt+F` |
| Toggle comment | `Ctrl+/` |
| Find | `Ctrl+F` |
| Cancel | `ESC` |
| Show palette | `F1` |

---

## Document Information

- **Generated**: 2026-02-13
- **Source**: Azure Data Explorer documentation analysis
- **Phase**: 3 - Feature In-Depth Investigation
