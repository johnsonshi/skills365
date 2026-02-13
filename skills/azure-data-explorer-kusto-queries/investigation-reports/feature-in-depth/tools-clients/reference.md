# Azure Data Explorer Tools and Clients Reference

## Overview

Azure Data Explorer provides multiple tools and clients for querying, managing, and developing KQL solutions. Each tool serves different use cases - from desktop development to command-line automation to browser-based exploration.

---

## 1. Kusto.Explorer (Desktop Application)

### Description

Kusto.Explorer is a free Windows desktop application for querying and analyzing data using KQL. It provides a rich IDE experience with advanced features for power users.

### Installation

- Download from: https://aka.ms/ke
- ClickOnce-enabled browser: `https://<your_cluster>/?web=0`
- For Chrome, install the ClickOnce extension
- Microsoft Edge supports ClickOnce natively via edge://flags/#edge-click-once

### User Interface Components

| Component | Purpose |
|-----------|---------|
| **Menu Panel** | Access all tool functions via ribbon tabs |
| **Connections Panel** | Manage cluster connections, browse schema |
| **Script Panel** | Write and edit KQL queries |
| **Results Panel** | View query results and visualizations |
| **Work Documents Pane** | Organize work folders and tracked KQL files |

### Key Features

#### Query Modes
- **Query Mode**: Standard query writing with script saving (default)
- **Search Mode**: Immediate command execution with results
- **Search++ Mode**: Search for terms across one or more tables using heat maps

#### Visualization Types
- Area Chart, Column Chart, Bar Chart, Stacked Area Chart
- Time Chart, Line Chart, Anomaly Chart (ML-based anomaly detection)
- Pie Chart, Scatter Chart, Pivot Chart, Time Pivot
- Time Ladder for timeline navigation

#### Code Features
- **Code Refactoring**: Rename variables/columns (`Ctrl+R`)
- **Extract to let**: Convert literals/tabular expressions (`Alt+Ctrl+M`)
- **Code Navigation**: Go to definition (`F12`), find references (`Ctrl+F12`)
- **Code Analyzer**: Automatic improvement recommendations via Issues tab

#### Data Export Options
- CSV, JSON, Excel (XLSX), Text, KQL Script, QRES files
- Direct clipboard copy with charts as bitmaps

### Configuration Options

| Category | Key Options |
|----------|-------------|
| **General** | Tool Mode (beta/alpha features), Display theme (light/dark) |
| **Query Editor** | Auto-save, Track external changes, Font settings, Tab replacement |
| **IntelliSense** | Enable/disable, Syntax highlighting, Issues list, Tooltips |
| **Formatter** | Version (V1/V2), Indentation, Bracketing style, Pipe operator placement |
| **Results Viewer** | Font size, Max lines per cell, Numeric formatting, Color scheme |
| **Connections** | Query server timeout, Admin command timeout, Lazy schema exploration |

### Connection Management

```kql
// Connect via script command
#connect cluster('help').database('Samples')

// Connection string format
Data Source=https://<ClusterName>.kusto.windows.net;Initial Catalog=<DatabaseName>;AAD Federated Security=True
```

### Troubleshooting

#### Common Issues and Solutions

| Issue | Solution |
|-------|----------|
| Always downloads even when no updates | Clear ClickOnce store: `rd /q /s %userprofile%\appdata\local\apps\2.0` |
| Cannot start application | Run `rundll32 %windir%\system32\dfshim.dll CleanOnlineAppCache` |
| Administrator blocked error | Reset ClickOnce trust prompt settings |

#### Complete Reset Procedure
1. Uninstall via Programs and Features
2. Delete `%LOCALAPPDATA%\Kusto.Explorer` (connections, history, settings)
3. Delete `%APPDATA%\Kusto` (token cache)
4. Reinstall from https://aka.ms/ke

---

## 2. Kusto.Cli (Command Line Interface)

### Description

Kusto.Cli is a command-line utility for running queries and control commands against a Kusto cluster. Ideal for scripting, automation, and batch operations.

### Installation

Part of the NuGet package `Microsoft.Azure.Kusto.Tools`. Download for .NET and extract the `tools` folder. No additional installation required (xcopy-installable).

### Operating Modes

| Mode | Description |
|------|-------------|
| **REPL Mode** | Interactive read/eval/print/loop for ad-hoc queries |
| **Execute Mode** | Run queries/commands from command-line arguments |
| **Script Mode** | Run queries/commands from a script file |

### Command Syntax

```bash
Kusto.Cli.exe <ConnectionString> [Switches]

# Basic connection
Kusto.Cli.exe "https://help.kusto.windows.net/Samples;Fed=true"

# Execute mode
Kusto.Cli.exe "https://help.kusto.windows.net/Samples;Fed=true" -execute:"StormEvents | take 10"

# Script mode
Kusto.Cli.exe "https://help.kusto.windows.net/Samples;Fed=true" -script:"c:\mycommands.txt"
```

### Command-Line Switches

| Switch | Description |
|--------|-------------|
| `-execute:<query>` | Run specified query or command |
| `-script:<file>` | Load and run script file (line-delimited) |
| `-scriptml:<file>` | Run entire script as single query |
| `-keepRunning:<true/false>` | Enter REPL after execute/script |
| `-echo:<true/false>` | Echo commands in output |
| `-transcript:<file>` | Write output to file |
| `-logToConsole:<true/false>` | Display output on console |
| `-lineMode:<true/false>` | Line vs block input mode |
| `-scriptQuitOnError:<true/false>` | Quit on script errors (default: true) |

### Built-in Directives

| Directive | Description |
|-----------|-------------|
| `?`, `#h`, `#help` | Show help message |
| `q`, `#quit`, `#exit` | Exit the tool |
| `#connect [conn]` | Connect to different cluster |
| `#dbcontext [db]` | Change database context |
| `#crp [name=value]` | Set client request properties |
| `#qp [name=value]` | Set query parameters |
| `#save <filename>` | Save next query results to CSV |
| `#clip` | Copy next results to clipboard |
| `#script <file>` | Execute script file |

### Input Modes

**Line Input Mode** (default):
- Each newline delimits a query/command
- Use `&` at line end to continue on next line
- Use `&&` to ignore newline and continue

**Block Input Mode** (`-lineMode:false` or `#blockmode`):
- Queries delimited by empty lines
- Recommended for script files

### Output Options

| Command | Description |
|---------|-------------|
| `#timeon` / `#timeoff` | Show/hide request timing |
| `#tableon` / `#tableoff` | Format as tables |
| `#markdownon` / `#markdownoff` | Format as Markdown |
| `#csvheaderson` / `#csvheadersoff` | Include CSV headers |
| `#prettyon` / `#prettyoff` | Clean up error messages |

---

## 3. Web UI (Azure Data Explorer Portal)

### Description

The Azure Data Explorer web UI (https://dataexplorer.azure.com) provides a browser-based environment for data exploration, querying, and dashboard creation.

### URL

- Main URL: https://dataexplorer.azure.com
- Help cluster: https://dataexplorer.azure.com/clusters/help/databases/Samples

### Key Features

| Feature | Description |
|---------|-------------|
| **Connection Pane** | Browse clusters, databases, tables, functions |
| **Query Editor** | Write KQL with IntelliSense and autocompletion |
| **Results Grid** | Sort, filter, group, visualize query results |
| **Multiple Tabs** | Work on queries in different contexts simultaneously |
| **Data Profile** | Quick insights into table columns and statistics |

### Query Editor Features

- KQL IntelliSense and autocomplete
- Inline documentation preview
- Quick fix suggestions and warnings
- Extract value into variable
- Define functions inline
- Format queries

### Running Queries

| Action | Shortcut |
|--------|----------|
| Run query (all results) | `Shift + Enter` |
| Preview results (50 rows) | `Alt + Shift + Enter` |
| Cancel query | `Ctrl + Shift + Enter` or `ESC` |
| Recall previous result | `F8` |

### Data Management

- **Add clusters/databases** to favorites with star icon
- **Group clusters** for easier management
- **Right-click database** for Ingest data, Create table options
- **Query statistics** show duration, CPU, memory, data scanned

### Sharing and Export

- Pin queries to dashboards
- Copy query link to clipboard
- Copy query and link together
- Export to Power BI, Excel, CSV formats

---

## 4. Monaco Editor / VS Code Extension

### Description

The Monaco Editor with Kusto Query Language support (monaco-kusto) enables embedding KQL editing capabilities into custom web applications.

### Package Information

- GitHub: https://github.com/Azure/monaco-kusto
- npm: `@kusto/monaco-kusto`

### Features

- Syntax highlighting and colorization
- IntelliSense and autocompletion
- Code refactoring and renaming
- Go-to-definition navigation
- Schema-aware suggestions

### Integration Options

| Method | Use Case |
|--------|----------|
| **AMD Module** | Traditional script-based loading |
| **ESM (webpack)** | Modern bundled applications |
| **IFrame Embed** | Quick integration with limited customization |

### Installation

```bash
# Install Monaco Editor
npm i monaco-editor

# Install monaco-kusto
npm i @kusto/monaco-kusto
```

### Basic Setup

```javascript
// Define MonacoEnvironment
window.MonacoEnvironment = {
    globalAPI: true,
    getWorkerUrl: function() { return "<path_to_kusto_worker>" }
};

// Create editor
const editor = monaco.editor.create(editorElement, editorConfig);

// Register Kusto language
import('monaco.contribution').then(async (contribution) => {
    const model = monaco.editor.createModel("", 'kusto');
    editor.setModel(model);
    const workerAccessor = await monaco.languages.kusto.getKustoWorker();
    const worker = await workerAccessor(model.uri);
});
```

### Schema Integration

```javascript
// Get schema from cluster
.show schema as json

// Set schema in editor
worker.setSchemaFromShowSchema(schema, clusterURI, database);
// OR
worker.setSchema(schema);
```

---

## 5. Kusto Emulator (Docker-based Local Development)

### Description

The Kusto Emulator is a Docker container image that encapsulates the Kusto query engine for local development and automated testing without requiring Azure services.

### Container Image

- Image: `mcr.microsoft.com/azuredataexplorer/kustainer-linux:latest`
- Tags: `latest`, `stable`
- License: https://aka.ms/adx.emulator.license

### Prerequisites

| Requirement | Details |
|-------------|---------|
| OS | Windows Server 2019/2022, Windows 11, Linux with Docker |
| Processor | SSE4.2/AVX2 support (ARM not supported) |
| RAM | 2 GB minimum, 4 GB+ recommended |
| Software | Docker Client for Linux or Windows |

### Quick Start

```powershell
# Start the emulator
docker run -e ACCEPT_EULA=Y -m 4G -d -p 8080:8080 -t mcr.microsoft.com/azuredataexplorer/kustainer-linux:latest

# Verify running
docker ps

# Test endpoint
Invoke-WebRequest -Method post -ContentType 'application/json' -Body '{"csl":".show cluster"}' http://localhost:8080/v1/rest/mgmt
```

### Run Options

```powershell
# Mount local folder for persistence
docker run -v d:\host\local:/kustodata -e ACCEPT_EULA=Y -m 4G -d -p 8080:8080 -t mcr.microsoft.com/azuredataexplorer/kustainer-linux:latest

# Use different port
docker run -e ACCEPT_EULA=Y -m 4G -d -p 9000:8080 -t mcr.microsoft.com/azuredataexplorer/kustainer-linux:latest
```

### Database Operations

```kql
// Create persistent database
.create database MyDB persist (
    @"/kustodata/dbs/MyDB/md",
    @"/kustodata/dbs/MyDB/data"
)

// Attach existing database
.attach database MyDB from @"/kustodata/dbs/MyDB/md"

// Detach database (preserves data)
.detach database MyDB
```

### Connecting from Kusto.Explorer

1. Enable unsafe connections: **Tools > Options > Connections > Allow unsafe connections**
2. Use HTTP (not HTTPS): `http://localhost:8080`
3. Remove AAD authentication from connection string

### Limitations

| Category | Limitation |
|----------|------------|
| **Security** | No authentication, no HTTPS, no encryption |
| **Ingestion** | No managed pipelines (Event Hubs, IoT Hub, Event Grid) |
| **Features** | No streaming ingestion, no Python plugin |
| **Data** | No extent merging, retention/partitioning policies not honored |
| **Format** | No compatibility guarantee between versions |

### Use Cases

| Scenario | Suitability |
|----------|-------------|
| Local development | Excellent (disconnected, free) |
| Automated testing | Excellent (fast provisioning, no Azure principal needed) |
| CI/CD pipelines | Excellent |
| Production workloads | Not suitable |

---

## 6. Avrotize Tool (Schema Conversion)

### Description

Avrotize is an open-source tool for converting data and schema formats between Kusto tables and Apache Avro.

### Installation

```bash
pip install avrotize
```

### Commands

```bash
# Kusto to Avro schema
avrotize k2a --kusto-uri <Uri> --kusto-database <DatabaseName> --avsc <AvroFilename.avsc>

# Avro to Kusto table declaration
avrotize a2k <AvroFilename.avsc> --out <KustoFilename.kql>

# JSON Schema to Kusto (via Avro)
avrotize j2a address.json | avrotize a2k --out address.kql
```

### Supported Conversions

- Kusto to Avro
- Avro to Kusto
- JSON Schema to Avro to Kusto
- XML Schema to Avro to Kusto
- Protobuf to Avro to Kusto

---

## Tool Comparison Matrix

| Feature | Kusto.Explorer | Kusto.Cli | Web UI | Emulator |
|---------|----------------|-----------|--------|----------|
| **Platform** | Windows | Cross-platform | Browser | Docker |
| **Best For** | Power users | Automation | Quick exploration | Local dev/testing |
| **Visualization** | Rich charts | Text/CSV | Charts, dashboards | N/A |
| **IntelliSense** | Full | None | Full | Via tools |
| **Offline** | Partial | Partial | No | Yes |
| **Cost** | Free | Free | Free | Free |
| **Authentication** | AAD | AAD | AAD | None |

---

## Document Information

- **Generated**: 2026-02-13
- **Source**: Azure Data Explorer documentation analysis
- **Phase**: 3 - Feature In-Depth Investigation
