# Tools and Clients Reference

Quick reference for Azure Data Explorer tools and clients.

---

## Tool Comparison

| Tool | Platform | Best For | Installation |
|------|----------|----------|--------------|
| **Kusto.Explorer** | Windows | Complex query development, power users | https://aka.ms/ke |
| **Web UI** | Browser | Quick exploration, dashboards | https://dataexplorer.azure.com |
| **Kusto.Cli** | Cross-platform | Automation, CI/CD, scripting | NuGet: Microsoft.Azure.Kusto.Tools |
| **Kusto Emulator** | Docker | Local dev/testing | mcr.microsoft.com/azuredataexplorer/kustainer-linux |
| **Monaco-Kusto** | Web | Custom app embedding | npm: @kusto/monaco-kusto |

---

## 1. Kusto.Explorer

### Installation
- Download: https://aka.ms/ke
- ClickOnce via browser: `https://<cluster>/?web=0`

### UI Components
| Component | Purpose |
|-----------|---------|
| **Connections Panel** | Browse clusters, databases, tables |
| **Script Panel** | Write KQL queries |
| **Results Panel** | View results, charts |

### Query Modes
- **Query Mode**: Standard (default) - write and save scripts
- **Search Mode**: Immediate execution
- **Search++ Mode**: Cross-table search with heat maps

### Visualization Types
Area, Column, Bar, Stacked Area, Time Chart, Line, Anomaly, Pie, Scatter, Pivot, Time Pivot, Time Ladder

### Connection Script
```kql
#connect cluster('help').database('Samples')
```

---

## 2. Kusto.Cli

### Installation
```bash
# Extract from NuGet package
nuget install Microsoft.Azure.Kusto.Tools -OutputDirectory ./tools
```

### Operating Modes
| Mode | Usage |
|------|-------|
| **REPL** | Interactive: `Kusto.Cli.exe "connection"` |
| **Execute** | One query: `-execute:"query"` |
| **Script** | File: `-script:"file.kql"` |

### Key Switches
| Switch | Description |
|--------|-------------|
| `-execute:<query>` | Run query |
| `-script:<file>` | Run script file |
| `-transcript:<file>` | Log output to file |
| `-lineMode:false` | Block mode (queries separated by blank lines) |
| `-scriptQuitOnError:true` | Exit on error |

### Directives
| Directive | Description |
|-----------|-------------|
| `#connect` | Connect to cluster |
| `#dbcontext` | Change database |
| `#save <file>` | Save next results to CSV |
| `#qp name=value` | Set query parameter |

---

## 3. Web UI

**URL**: https://dataexplorer.azure.com

### Features
- Query editor with IntelliSense
- Results grid with filtering/grouping
- Visualizations and dashboards
- Share queries via deep links
- Data profile for quick stats

### Deep Link Format
```
https://dataexplorer.azure.com/clusters/<cluster>/databases/<db>?query=<encoded-query>
```

---

## 4. Kusto Emulator

### Docker Setup
```bash
docker run -e ACCEPT_EULA=Y -m 4G -d -p 8080:8080 \
  mcr.microsoft.com/azuredataexplorer/kustainer-linux:latest
```

### Connection
```
http://localhost:8080
```

### Limitations
- Single database
- No persistent storage (by default)
- Subset of management commands
- No cluster-level features

### Create Database
```kql
.create database TestDB
```

---

## 5. Monaco-Kusto

### Installation
```bash
npm install @kusto/monaco-kusto monaco-editor
```

### Basic Integration
```javascript
import * as monaco from 'monaco-editor';
import { getKustoWorker } from '@kusto/monaco-kusto';

const editor = monaco.editor.create(container, {
  value: 'StormEvents | take 10',
  language: 'kusto'
});
```

---

## Connection Strings

### Format
```
Data Source=https://<cluster>.kusto.windows.net;Initial Catalog=<database>;AAD Federated Security=True
```

### Authentication Methods
| Method | Connection String Addition |
|--------|---------------------------|
| User Interactive | `Fed=true` |
| App + Secret | `AppClientId=<id>;AppKey=<secret>;Authority Id=<tenant>` |
| Managed Identity | `Fed=true;AzureRegion=<region>;ManagedServiceIdentity=true` |
| Certificate | `AppClientId=<id>;AppCert=<thumbprint>;Authority Id=<tenant>` |

### Shorthand
```
@help/Samples
# Expands to: https://help.kusto.windows.net/Samples;Fed=true
```
