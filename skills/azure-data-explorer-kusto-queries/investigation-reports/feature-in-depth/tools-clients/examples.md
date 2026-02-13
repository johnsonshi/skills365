# Azure Data Explorer Tools and Clients Examples

## Overview

This document provides practical examples for CLI scripts, emulator setup, connection configurations, and common tool workflows.

---

## 1. Kusto.Cli Script Examples

### Basic Query Execution

```bash
# Simple query execution
Kusto.Cli.exe "https://help.kusto.windows.net/Samples;Fed=true" -execute:"StormEvents | count"
```

### Export Results to CSV

```bash
# Export query results to CSV file
Kusto.Cli.exe "https://help.kusto.windows.net/Samples;Fed=true" ^
  -execute:"#save c:\temp\storm_summary.csv" ^
  -execute:"StormEvents | summarize EventCount=count() by State | order by EventCount desc"
```

### Multi-Query Batch Script

```bash
# Execute multiple queries sequentially
Kusto.Cli.exe "https://help.kusto.windows.net/Samples;Fed=true" ^
  -execute:"#save c:\temp\counts.csv" ^
  -execute:"StormEvents | count" ^
  -execute:"#save c:\temp\by_state.csv" ^
  -execute:"StormEvents | summarize count() by State" ^
  -execute:"#save c:\temp\by_type.csv" ^
  -execute:"StormEvents | summarize count() by EventType"
```

### Script File Execution

**File: queries.kql**
```kql
// Query 1: Event counts
StormEvents | count

// Query 2: Damage summary
StormEvents
| summarize TotalDamage = sum(DamageProperty)
by State
| order by TotalDamage desc
| take 10
```

**Command:**
```bash
Kusto.Cli.exe "https://help.kusto.windows.net/Samples;Fed=true" ^
  -script:"c:\scripts\queries.kql" ^
  -lineMode:false ^
  -transcript:"c:\logs\output.log"
```

### Connection Context Script

**File: multi_db_script.kql**
```kql
// Connect to first database
#connect cluster('help').database('Samples')

// Query Samples database
StormEvents | count

// Switch to different database
#dbcontext ContosoSales

// Query ContosoSales database
Sales | summarize sum(Amount) by Region
```

### CI/CD Pipeline Script

```bash
#!/bin/bash
# ci_test_queries.sh - Run query tests in CI/CD pipeline

CLUSTER="https://mycluster.kusto.windows.net"
DATABASE="TestDB"
CONNECTION="${CLUSTER}/${DATABASE};Fed=true;AppClientId=${CLIENT_ID};AppKey=${CLIENT_SECRET};Authority Id=${TENANT_ID}"

# Run test queries and capture output
Kusto.Cli.exe "${CONNECTION}" \
  -script:"test_queries.kql" \
  -scriptQuitOnError:true \
  -transcript:"test_output.log" \
  -lineMode:false

# Check exit code
if [ $? -eq 0 ]; then
    echo "All queries passed"
    exit 0
else
    echo "Query tests failed - see test_output.log"
    exit 1
fi
```

### Query Parameters Script

```bash
# Using query parameters
Kusto.Cli.exe "https://help.kusto.windows.net/Samples;Fed=true" ^
  -execute:"#qp start_date=2007-01-01" ^
  -execute:"#qp end_date=2007-12-31" ^
  -execute:"declare query_parameters(start_date:datetime, end_date:datetime); StormEvents | where StartTime between (start_date .. end_date) | count"
```

### Scheduled Report Script

**File: daily_report.cmd**
```batch
@echo off
REM Daily report generation script

set CLUSTER=https://mycluster.kusto.windows.net
set DATABASE=Production
set OUTPUT_DIR=C:\Reports\%date:~-4,4%%date:~-10,2%%date:~-7,2%

REM Create output directory
mkdir %OUTPUT_DIR% 2>nul

REM Generate reports
Kusto.Cli.exe "%CLUSTER%/%DATABASE%;Fed=true" ^
  -execute:"#save %OUTPUT_DIR%\summary.csv" ^
  -execute:"DailyMetrics | where Timestamp > ago(1d) | summarize avg(Value) by MetricName" ^
  -execute:"#save %OUTPUT_DIR%\errors.csv" ^
  -execute:"ErrorLog | where Timestamp > ago(1d) | summarize count() by ErrorType" ^
  -logToConsole:false

echo Reports generated in %OUTPUT_DIR%
```

---

## 2. Kusto Emulator Setup Examples

### Basic Docker Setup

```powershell
# Pull and start emulator
docker run -e ACCEPT_EULA=Y -m 4G -d -p 8080:8080 -t mcr.microsoft.com/azuredataexplorer/kustainer-linux:latest

# Verify running
docker ps

# Test connection
Invoke-WebRequest -Method post -ContentType 'application/json' -Body '{"csl":".show cluster"}' http://localhost:8080/v1/rest/mgmt
```

### Persistent Data Setup

```powershell
# Create local directory for persistence
New-Item -ItemType Directory -Path "C:\KustoData" -Force

# Start emulator with mounted volume
docker run `
  --name kusto-local `
  -v C:\KustoData:/kustodata `
  -e ACCEPT_EULA=Y `
  -m 4G `
  -d `
  -p 8080:8080 `
  -t mcr.microsoft.com/azuredataexplorer/kustainer-linux:latest
```

### Database and Table Setup

```kql
// Create persistent database
.create database DevDB persist (
    @"/kustodata/dbs/DevDB/md",
    @"/kustodata/dbs/DevDB/data"
)

// Create tables
.create table Logs (
    Timestamp: datetime,
    Level: string,
    Message: string,
    Source: string
)

.create table Metrics (
    Timestamp: datetime,
    MetricName: string,
    Value: real,
    Tags: dynamic
)

// Create ingestion mapping
.create table Logs ingestion json mapping 'LogsMapping' '[
    {"column":"Timestamp","path":"$.timestamp","datatype":"datetime"},
    {"column":"Level","path":"$.level","datatype":"string"},
    {"column":"Message","path":"$.message","datatype":"string"},
    {"column":"Source","path":"$.source","datatype":"string"}
]'
```

### Sample Data Ingestion

**File: sample_logs.csv**
```csv
Timestamp,Level,Message,Source
2024-01-01T10:00:00Z,INFO,Application started,AppService
2024-01-01T10:05:00Z,WARNING,High memory usage,Monitor
2024-01-01T10:10:00Z,ERROR,Connection timeout,Database
```

```kql
// Ingest from mounted volume
.ingest into table Logs (@"/kustodata/sample_logs.csv") with (format="csv", ignoreFirstRecord=true)

// Verify data
Logs | take 10
```

### Docker Compose Setup

**File: docker-compose.yml**
```yaml
version: '3.8'
services:
  kusto-emulator:
    image: mcr.microsoft.com/azuredataexplorer/kustainer-linux:latest
    container_name: kusto-emulator
    environment:
      - ACCEPT_EULA=Y
    ports:
      - "8080:8080"
    volumes:
      - kusto-data:/kustodata
      - ./test-data:/testdata:ro
    deploy:
      resources:
        limits:
          memory: 4G
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/v1/rest/mgmt"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

volumes:
  kusto-data:
```

**Commands:**
```bash
# Start
docker-compose up -d

# View logs
docker-compose logs -f kusto-emulator

# Stop and clean
docker-compose down
docker-compose down -v  # Remove volumes too
```

### CI/CD Integration Example

**File: .github/workflows/kusto-tests.yml**
```yaml
name: Kusto Query Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      kusto:
        image: mcr.microsoft.com/azuredataexplorer/kustainer-linux:latest
        env:
          ACCEPT_EULA: Y
        ports:
          - 8080:8080
        options: >-
          --memory=4g
          --health-cmd "curl -f http://localhost:8080/v1/rest/mgmt"
          --health-interval 30s
          --health-timeout 10s
          --health-retries 5

    steps:
      - uses: actions/checkout@v3

      - name: Wait for Kusto
        run: |
          timeout 120 sh -c 'until curl -s http://localhost:8080/v1/rest/mgmt; do sleep 5; done'

      - name: Setup Database
        run: |
          curl -X POST http://localhost:8080/v1/rest/mgmt \
            -H "Content-Type: application/json" \
            -d '{"csl":".create database TestDB"}'

      - name: Run Tests
        run: |
          ./scripts/run_kusto_tests.sh
```

---

## 3. Connection String Examples

### Kusto.Explorer Connections

#### Azure AD Interactive
```
https://mycluster.kusto.windows.net
```

#### Azure AD with Tenant
```
Data Source=https://mycluster.kusto.windows.net;Initial Catalog=MyDatabase;AAD Federated Security=True;Authority Id=contoso.com
```

#### Azure AD with User Identity
```
Data Source=https://mycluster.kusto.windows.net;Initial Catalog=MyDatabase;AAD Federated Security=True;Authority Id=contoso.com;User=user@example.com
```

#### Service Principal
```
Data Source=https://mycluster.kusto.windows.net;Initial Catalog=MyDatabase;AAD Federated Security=True;Application Client Id=<app-id>;Application Key=<app-key>;Authority Id=<tenant-id>
```

#### Managed Identity (System-Assigned)
```
Data Source=https://mycluster.kusto.windows.net;Initial Catalog=MyDatabase;AAD Federated Security=True;AadFederatedSecurity=True;dSTS Federated Security=True;Fed=True;AppToken=<token>
```

### Kusto Emulator Connection

#### HTTP Connection (No Auth)
```
http://localhost:8080
```

#### In Kusto.Explorer
1. **Tools > Options > Connections > Allow unsafe connections**: Enable
2. **Add Connection**:
   - Cluster Connection: `http://localhost:8080`
   - Security: Remove `AAD Federated Security=True`

### CLI Connection Strings

```bash
# Azure AD interactive
Kusto.Cli.exe "https://mycluster.kusto.windows.net/MyDatabase;Fed=true"

# Service principal
Kusto.Cli.exe "https://mycluster.kusto.windows.net/MyDatabase;Fed=true;AppClientId=<id>;AppKey=<secret>;Authority Id=<tenant>"

# With specific tenant
Kusto.Cli.exe "https://mycluster.kusto.windows.net/MyDatabase;AAD Federated Security=True;Authority Id=<tenant-id>"
```

---

## 4. Web UI Deep Link Examples

### Basic Query Link

```
https://dataexplorer.azure.com/clusters/help/databases/Samples?query=StormEvents%20%7C%20count
```

### Parameterized Query Link

```
https://help.kusto.windows.net/Samples?web=0&query=KustoLogs+%7c+where+Timestamp+>+ago({Period})+%7c+count&Period=1h
```

### Encoded Query Link (for long queries)

```csharp
// C# code to generate short links
string query = "StormEvents | where StartTime > ago(7d) | summarize count() by State | order by count_ desc";
string encoded = CslCommandGenerator.EncodeQueryAsBase64Url(query);
string url = $"https://help.kusto.windows.net/Samples?query={encoded}";
```

---

## 5. Monaco-Kusto Integration Examples

### Basic HTML Integration

```html
<!DOCTYPE html>
<html>
<head>
    <title>KQL Editor</title>
    <style>
        #editor { width: 100%; height: 400px; border: 1px solid #ccc; }
    </style>
</head>
<body>
    <div id="editor"></div>

    <script src="monaco-editor/min/vs/loader.js"></script>
    <script src="monaco-editor/min/vs/language/kusto/bridge.min.js"></script>
    <script src="monaco-editor/min/vs/language/kusto/kusto.javascript.client.min.js"></script>
    <script src="monaco-editor/min/vs/language/kusto/newtonsoft.json.min.js"></script>
    <script src="monaco-editor/min/vs/language/kusto/Kusto.Language.Bridge.min.js"></script>

    <script>
        require.config({ paths: { vs: 'monaco-editor/min/vs' }});
        require(['vs/editor/editor.main'], function() {
            require(['vs/language/kusto/monaco.contribution'], function() {
                const editor = monaco.editor.create(document.getElementById('editor'), {
                    value: 'StormEvents | take 10',
                    language: 'kusto',
                    theme: 'vs-dark'
                });
            });
        });
    </script>
</body>
</html>
```

### React Integration

```javascript
// KustoEditor.jsx
import React, { useEffect, useRef } from 'react';
import * as monaco from 'monaco-editor';

export function KustoEditor({ value, onChange, schema }) {
    const editorRef = useRef(null);
    const containerRef = useRef(null);

    useEffect(() => {
        // Initialize Monaco environment
        window.MonacoEnvironment = {
            getWorkerUrl: function(moduleId, label) {
                if (label === 'kusto') {
                    return './kusto.worker.js';
                }
                return './editor.worker.js';
            }
        };

        // Create editor
        editorRef.current = monaco.editor.create(containerRef.current, {
            value: value,
            language: 'kusto',
            theme: 'vs-dark',
            automaticLayout: true
        });

        // Setup change handler
        editorRef.current.onDidChangeModelContent(() => {
            onChange?.(editorRef.current.getValue());
        });

        return () => editorRef.current?.dispose();
    }, []);

    useEffect(() => {
        if (schema && editorRef.current) {
            setEditorSchema(editorRef.current, schema);
        }
    }, [schema]);

    return <div ref={containerRef} style={{ height: '400px' }} />;
}

async function setEditorSchema(editor, schema) {
    const workerAccessor = await monaco.languages.kusto.getKustoWorker();
    const model = editor.getModel();
    const worker = await workerAccessor(model.uri);
    worker.setSchemaFromShowSchema(schema, schema.cluster.connectionString, schema.database.name);
}
```

### Schema Setup from Cluster

```javascript
// Fetch schema from cluster
async function fetchSchema(clusterUrl, database, token) {
    const response = await fetch(`${clusterUrl}/v1/rest/mgmt`, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${token}`
        },
        body: JSON.stringify({
            csl: '.show schema as json',
            db: database
        })
    });

    const result = await response.json();
    return JSON.parse(result.Tables[0].Rows[0][0]);
}

// Set schema in editor
async function setupEditor(editor, clusterUrl, database, token) {
    const schema = await fetchSchema(clusterUrl, database, token);
    const workerAccessor = await monaco.languages.kusto.getKustoWorker();
    const model = editor.getModel();
    const worker = await workerAccessor(model.uri);
    worker.setSchemaFromShowSchema(schema, clusterUrl, database);
}
```

---

## 6. Avrotize Schema Conversion Examples

### Kusto to Avro

```bash
# Export all tables in database to Avro schema
avrotize k2a --kusto-uri https://help.kusto.windows.net --kusto-database Samples --avsc samples.avsc

# Export with CloudEvents wrapper
avrotize k2a --kusto-uri https://mycluster.kusto.windows.net --kusto-database Events --avsc events.xreg.json --emit-cloudevents-xregistry --avro-namespace com.mycompany.events
```

### Avro to Kusto

```bash
# Convert Avro schema to KQL table declarations
avrotize a2k samples.avsc --out create_tables.kql
```

**Output (create_tables.kql):**
```kql
.create table StormEvents (
    StartTime: datetime,
    EndTime: datetime,
    EpisodeId: int,
    EventId: int,
    State: string,
    EventType: string,
    DamageProperty: real,
    DamageCrops: real
)
```

### JSON Schema to Kusto

```bash
# Convert JSON Schema to Kusto via Avro
avrotize j2a user_event.json | avrotize a2k --out user_event.kql
```

---

## 7. Complete Workflow Examples

### Local Development Setup

```powershell
# 1. Start emulator
docker run --name kusto-dev -v C:\Dev\kustodata:/kustodata -e ACCEPT_EULA=Y -m 4G -d -p 8080:8080 -t mcr.microsoft.com/azuredataexplorer/kustainer-linux:latest

# 2. Wait for startup
Start-Sleep -Seconds 30

# 3. Create database via REST API
$body = @{csl=".create database DevDB persist (@'/kustodata/dbs/DevDB/md', @'/kustodata/dbs/DevDB/data')"} | ConvertTo-Json
Invoke-RestMethod -Method Post -Uri "http://localhost:8080/v1/rest/mgmt" -ContentType "application/json" -Body $body

# 4. Connect Kusto.Explorer
# Open Kusto.Explorer, enable unsafe connections, add http://localhost:8080

# 5. Develop and test queries
# Use Kusto.Explorer for interactive development

# 6. Export final queries
# Save to .kql files for version control
```

### Automated Testing Pipeline

```bash
#!/bin/bash
# test_pipeline.sh - Complete testing workflow

# Configuration
EMULATOR_PORT=8080
DB_NAME="TestDB"
SCRIPT_DIR="./test_queries"
OUTPUT_DIR="./test_output"

# Start emulator
echo "Starting emulator..."
docker run --name kusto-test -e ACCEPT_EULA=Y -m 4G -d -p ${EMULATOR_PORT}:8080 -t mcr.microsoft.com/azuredataexplorer/kustainer-linux:latest

# Wait for startup
echo "Waiting for emulator..."
sleep 30
until curl -s http://localhost:${EMULATOR_PORT}/v1/rest/mgmt > /dev/null; do
    sleep 5
done

# Create database
echo "Creating database..."
curl -s -X POST http://localhost:${EMULATOR_PORT}/v1/rest/mgmt \
    -H "Content-Type: application/json" \
    -d "{\"csl\":\".create database ${DB_NAME}\"}"

# Run setup script
echo "Running setup..."
Kusto.Cli.exe "http://localhost:${EMULATOR_PORT}/${DB_NAME}" \
    -script:"${SCRIPT_DIR}/setup.kql" \
    -lineMode:false

# Run test queries
echo "Running tests..."
mkdir -p ${OUTPUT_DIR}
Kusto.Cli.exe "http://localhost:${EMULATOR_PORT}/${DB_NAME}" \
    -script:"${SCRIPT_DIR}/tests.kql" \
    -transcript:"${OUTPUT_DIR}/test_results.log" \
    -scriptQuitOnError:true \
    -lineMode:false

TEST_EXIT=$?

# Cleanup
echo "Cleaning up..."
docker stop kusto-test
docker rm kusto-test

exit $TEST_EXIT
```

### Production Deployment Script

```powershell
# deploy_schema.ps1 - Deploy schema changes to production

param(
    [string]$ClusterUrl = "https://mycluster.kusto.windows.net",
    [string]$Database = "Production",
    [string]$ScriptPath = ".\schema_changes.kql"
)

# Validate script exists
if (-not (Test-Path $ScriptPath)) {
    Write-Error "Script not found: $ScriptPath"
    exit 1
}

# Run schema changes
Write-Host "Deploying schema changes to $ClusterUrl/$Database..."

$output = & Kusto.Cli.exe "$ClusterUrl/$Database;Fed=true" `
    -script:"$ScriptPath" `
    -lineMode:false `
    -scriptQuitOnError:true `
    2>&1

if ($LASTEXITCODE -ne 0) {
    Write-Error "Deployment failed:"
    Write-Error $output
    exit 1
}

Write-Host "Deployment successful"
Write-Host $output
```

---

## Document Information

- **Generated**: 2026-02-13
- **Source**: Azure Data Explorer documentation analysis
- **Phase**: 3 - Feature In-Depth Investigation
