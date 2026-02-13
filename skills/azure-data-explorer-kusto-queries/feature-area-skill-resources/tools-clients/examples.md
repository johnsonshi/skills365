# Tools and Clients Examples

## 1. Kusto.Cli Examples

### Basic Query
```bash
Kusto.Cli.exe "https://help.kusto.windows.net/Samples;Fed=true" -execute:"StormEvents | count"
```

### Export to CSV
```bash
Kusto.Cli.exe "https://help.kusto.windows.net/Samples;Fed=true" ^
  -execute:"#save c:\temp\results.csv" ^
  -execute:"StormEvents | summarize count() by State"
```

### Multi-Query Script
```bash
Kusto.Cli.exe "https://help.kusto.windows.net/Samples;Fed=true" ^
  -execute:"#save counts.csv" ^
  -execute:"StormEvents | count" ^
  -execute:"#save by_state.csv" ^
  -execute:"StormEvents | summarize count() by State"
```

### Script File Execution
**queries.kql:**
```kql
StormEvents | count

StormEvents
| summarize TotalDamage = sum(DamageProperty) by State
| top 10 by TotalDamage desc
```

**Command:**
```bash
Kusto.Cli.exe "https://help.kusto.windows.net/Samples;Fed=true" ^
  -script:"queries.kql" ^
  -lineMode:false ^
  -transcript:"output.log"
```

### CI/CD Pipeline Script
```bash
#!/bin/bash
CONNECTION="https://mycluster.kusto.windows.net/TestDB;Fed=true;AppClientId=${CLIENT_ID};AppKey=${CLIENT_SECRET};Authority Id=${TENANT_ID}"

Kusto.Cli.exe "${CONNECTION}" \
  -script:"test_queries.kql" \
  -scriptQuitOnError:true \
  -transcript:"test_output.log"

if [ $? -eq 0 ]; then
    echo "All queries passed"
else
    echo "Tests failed"
    exit 1
fi
```

### Query Parameters
```bash
Kusto.Cli.exe "https://help.kusto.windows.net/Samples;Fed=true" ^
  -execute:"#qp start_date=2007-01-01" ^
  -execute:"declare query_parameters(start_date:datetime); StormEvents | where StartTime >= start_date | count"
```

---

## 2. Kusto Emulator Examples

### Start Emulator
```bash
docker run -e ACCEPT_EULA=Y -m 4G -d -p 8080:8080 \
  mcr.microsoft.com/azuredataexplorer/kustainer-linux:latest
```

### With Persistent Volume
```bash
docker run -e ACCEPT_EULA=Y -m 4G -d -p 8080:8080 \
  -v kusto_data:/kusto/data \
  mcr.microsoft.com/azuredataexplorer/kustainer-linux:latest
```

### Docker Compose
```yaml
version: '3'
services:
  kusto:
    image: mcr.microsoft.com/azuredataexplorer/kustainer-linux:latest
    environment:
      - ACCEPT_EULA=Y
    ports:
      - "8080:8080"
    deploy:
      resources:
        limits:
          memory: 4G
    volumes:
      - kusto_data:/kusto/data

volumes:
  kusto_data:
```

### Setup Database and Table
```kql
// Connect to emulator
// Connection: http://localhost:8080

// Create database
.create database TestDB

// Create table
.create table Events (
    Timestamp: datetime,
    EventType: string,
    Value: real
)

// Ingest test data
.ingest inline into table Events <|
2024-01-01,TypeA,100
2024-01-02,TypeB,200
2024-01-03,TypeA,150
```

---

## 3. Connection String Examples

### User Interactive
```
https://mycluster.westus.kusto.windows.net/MyDatabase;Fed=true
```

### Service Principal with Secret
```
https://mycluster.westus.kusto.windows.net/MyDatabase;Fed=true;AppClientId=<app-id>;AppKey=<secret>;Authority Id=<tenant-id>
```

### Managed Identity
```
https://mycluster.westus.kusto.windows.net/MyDatabase;Fed=true;AzureRegion=westus;ManagedServiceIdentity=true
```

### Certificate Authentication
```
https://mycluster.westus.kusto.windows.net/MyDatabase;Fed=true;AppClientId=<app-id>;AppCert=<thumbprint>;Authority Id=<tenant-id>
```

---

## 4. Web UI Deep Links

### Basic Query Link
```
https://dataexplorer.azure.com/clusters/help/databases/Samples?query=StormEvents%20%7C%20take%2010
```

### With Time Range
```
https://dataexplorer.azure.com/clusters/help/databases/Samples?query=StormEvents%20%7C%20where%20StartTime%20%3E%20ago(7d)
```

### Generate in Kusto.Explorer
1. Run query
2. `Ctrl+Shift+C` to copy with deep link
3. Share the URL

---

## 5. Monaco-Kusto Integration

### HTML Setup
```html
<!DOCTYPE html>
<html>
<head>
  <style>
    #editor { width: 100%; height: 400px; }
  </style>
</head>
<body>
  <div id="editor"></div>
  <script src="https://unpkg.com/monaco-editor/min/vs/loader.js"></script>
  <script>
    require.config({ paths: { vs: 'https://unpkg.com/monaco-editor/min/vs' } });
    require(['vs/editor/editor.main'], function () {
      monaco.editor.create(document.getElementById('editor'), {
        value: 'StormEvents\n| where StartTime > ago(1h)\n| take 100',
        language: 'kusto',
        theme: 'vs-dark'
      });
    });
  </script>
</body>
</html>
```

### React Integration
```jsx
import React, { useRef, useEffect } from 'react';
import * as monaco from 'monaco-editor';

function KustoEditor({ initialValue, onChange }) {
  const editorRef = useRef(null);
  const containerRef = useRef(null);

  useEffect(() => {
    editorRef.current = monaco.editor.create(containerRef.current, {
      value: initialValue,
      language: 'kusto',
      theme: 'vs-dark',
      automaticLayout: true
    });

    editorRef.current.onDidChangeModelContent(() => {
      onChange?.(editorRef.current.getValue());
    });

    return () => editorRef.current?.dispose();
  }, []);

  return <div ref={containerRef} style={{ height: '400px' }} />;
}
```

---

## 6. Kusto.Explorer Scripted Setup

### Connection Script
```kql
// Save as: setup_connections.kql
#connect cluster('mycluster.westus').database('Production')

// Verify connection
.show database

// Set context for subsequent queries
print "Connected to Production"
```

### Multi-Environment Script
```kql
// development.kql
#connect cluster('mydev.westus').database('DevDB')
.show tables

// production.kql
#connect cluster('myprod.westus').database('ProdDB')
.show tables
```
