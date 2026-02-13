# Azure Data Explorer Business Continuity Examples

## Follower Database Setup

### Basic Follower Configuration (PowerShell)
```powershell
$LeaderCluster = Get-AzKustoCluster -Name "leader-cluster" `
    -ResourceGroupName "leader-rg" -SubscriptionId "xxx"

New-AzKustoAttachedDatabaseConfiguration `
    -ClusterName "follower-cluster" `
    -Name "SalesDB" `
    -ResourceGroupName "follower-rg" `
    -DatabaseName "SalesDB" `
    -ClusterResourceId $LeaderCluster.Id `
    -DefaultPrincipalsModificationKind 'Union' `
    -Location $LeaderCluster.Location
```

### Table-Level Sharing
```powershell
New-AzKustoAttachedDatabaseConfiguration `
    -ClusterName "follower-cluster" `
    -Name "SalesDBConfig" `
    -ResourceGroupName "follower-rg" `
    -DatabaseName "SalesDB" `
    -ClusterResourceId $LeaderCluster.Id `
    -DefaultPrincipalsModificationKind 'Union' `
    -Location $LeaderCluster.Location `
    -TableLevelSharingPropertyTablesToInclude "Sales*", "Inventory*" `
    -TableLevelSharingPropertyExternalTablesToExclude "*"
```

### Follow All Databases
```powershell
New-AzKustoAttachedDatabaseConfiguration `
    -ClusterName "follower-cluster" `
    -Name "AllDBsConfig" `
    -ResourceGroupName "follower-rg" `
    -DatabaseName '*' `
    -ClusterResourceId $LeaderCluster.Id `
    -DefaultPrincipalsModificationKind 'Union' `
    -Location $LeaderCluster.Location
```

---

## Follower Database Management

### Configure Caching Policy
```kusto
// Database-level caching
.alter follower database SalesDB policy caching hot = 7d

// Table-specific caching
.alter follower database SalesDB table RecentTransactions policy caching hot = 30d

// Replace leader's caching policy
.alter follower database SalesDB caching-policies-modification-kind = replace
```

### Manage Principals
```kusto
// Add viewer group
.add follower database SalesDB viewers ('aadgroup=analytics-team@contoso.com')

// Add admin
.add follower database SalesDB admins ('aaduser=admin@contoso.com')

// Use only follower-defined principals
.alter follower database SalesDB principals-modification-kind = replace

// View configuration
.show follower database SalesDB | evaluate narrow()
```

### Detach Follower Database
```powershell
$Configs = Get-AzKustoAttachedDatabaseConfiguration `
    -ClusterName "follower-cluster" -ResourceGroupName "follower-rg"
$ConfigToRemove = $Configs | Where-Object { $_.DatabaseName -eq 'SalesDB' }

Remove-AzKustoAttachedDatabaseConfiguration `
    -ClusterName "follower-cluster" `
    -Name ($ConfigToRemove.Name.Split("/")[1]) `
    -ResourceGroupName "follower-rg"
```

---

## Cross-Cluster Queries

### Basic Cross-Cluster Query
```kusto
cluster("analytics-cluster.westeurope.kusto.windows.net")
    .database("MetricsDB").ServerMetrics
| where Timestamp > ago(1h)
| summarize AvgCPU = avg(CPUPercent) by bin(Timestamp, 5m), ServerName
```

### Join Local and Remote Data
```kusto
let remoteData = cluster("remote-cluster").database("RemoteDB").RemoteTable;
LocalTable
| join kind=inner remoteData on $left.Id == $right.Id
| project LocalColumn, RemoteColumn, JoinedTimestamp
```

### Cross-Database Query (Same Cluster)
```kusto
database("InventoryDB").Products
| join kind=inner database("SalesDB").Transactions
    on $left.ProductId == $right.ProductId
| summarize TotalSales = sum(Amount) by ProductName
```

### Wildcard Union
```kusto
union cluster("prod-cluster").database("Logs*").*
| where Level == "Error"
| summarize ErrorCount = count() by Database = database_name()
```

### Clear Remote Schema Cache
```kusto
.clear cache remote-schema cluster("remote-cluster").database("UpdatedDB")
```

---

## Continuous Export for DR

### Setup Export to GRS Storage
```kusto
// Create external table
.create external table DRBackup_Transactions (
    TransactionId: guid, CustomerId: long, Amount: decimal,
    Timestamp: datetime, ProductId: long, Region: string
)
kind=blob
partition by (Region: string, Date: datetime = bin(Timestamp, 1d))
dataformat=parquet
(h@'https://drstorage.blob.core.windows.net/backups;...')

// Create continuous export
.create-or-alter continuous-export TransactionBackup
over (Transactions)
to table DRBackup_Transactions
with (intervalBetweenRuns=5m, forcedLatency=2m, sizeLimit=524288000)
<| Transactions
```

### Monitor Export Health
```kusto
// Check status
.show continuous-export TransactionBackup
| project Name, LastRunTime, LastRunResult, ExportedTo

// View lateness
.show continuous-export TransactionBackup
| project Name, LatencyMinutes = datetime_diff('minute', now(), ExportedTo)

// Check failures
.show continuous-export TransactionBackup failures | take 20

// View exported files
.show continuous-export TransactionBackup exported-artifacts
| where Timestamp > ago(1h)
| summarize FileCount = count(), TotalSizeBytes = sum(SizeInBytes)
```

### Export Historical Data
```kusto
// Get cursor
.show continuous-export TransactionBackup | project StartCursor

// Export data before cursor
.export async to table DRBackup_Transactions
with (sizeLimit=1073741824, distributed=true)
<| Transactions
| where cursor_before_or_at("636751928823156645")
| where Timestamp > ago(365d)
```

---

## Recovery Procedures

### Clone Schema to DR Cluster
```kusto
// SOURCE: Export schema
.show database SalesDB schema as csl script with (ShowObfuscatedStrings = true)

// TARGET: Execute schema
.execute database script <|
// Paste CSL script here
```

### Recover Dropped Table
```kusto
.show database SalesDB policy retention
.undo drop table DroppedTableName
```

### Restore from Continuous Export
```kusto
// Create table
.create table Transactions (
    TransactionId: guid, CustomerId: long, Amount: decimal,
    Timestamp: datetime, ProductId: long, Region: string
)

// Ingest from backup
.ingest async into table Transactions (
    h@'https://drstorage.blob.core.windows.net/backups/;...'
)
with (format='parquet')
```

### Validate DR Data Consistency
```kusto
let primaryCount = toscalar(
    cluster("primary-cluster").database("SalesDB").Transactions | count());
let drCount = toscalar(Transactions | count());
print Primary = primaryCount, DR = drCount,
      Difference = primaryCount - drCount,
      PercentDiff = round(100.0 * (primaryCount - drCount) / primaryCount, 2)
```

---

## Availability Zone Migration

```powershell
# Check available zones
$Cluster = Get-AzKustoCluster -Name "mycluster" -ResourceGroupName "myrg"

# Migrate to AZ (can take hours)
Update-AzKustoCluster -ResourceGroupName "myrg" -Name "mycluster" `
    -Zone "1", "2", "3"
```

---

## Application-Level Failover (C#)

```csharp
public class AdxBcdrClient : IDisposable
{
    private readonly List<ICslQueryProvider> _providers;
    private int _currentIndex = 0;

    public AdxBcdrClient(string[] clusterUris, string database)
    {
        _providers = clusterUris.Select(uri => {
            var csb = new KustoConnectionStringBuilder(uri, database)
                .WithAadUserPromptAuthentication();
            return KustoClientFactory.CreateCslQueryProvider(csb);
        }).ToList();
    }

    public async Task<IDataReader> ExecuteQueryAsync(string query)
    {
        for (int i = 0; i < _providers.Count; i++)
        {
            try
            {
                return await Task.Run(() => _providers[_currentIndex].ExecuteQuery(query));
            }
            catch
            {
                _currentIndex = (_currentIndex + 1) % _providers.Count;
            }
        }
        throw new Exception("All clusters unavailable");
    }

    public void Dispose() => _providers.ForEach(p => p?.Dispose());
}

// Usage
var clusters = new[] {
    "https://primary.westeurope.kusto.windows.net",
    "https://secondary.northeurope.kusto.windows.net"
};
using var client = new AdxBcdrClient(clusters, "SalesDB");
var result = await client.ExecuteQueryAsync("Transactions | take 10");
```

---

## Failover Ingestion

```kusto
// Verify streaming ingestion on DR cluster
.show database SalesDB policy streamingingestion

// Enable if needed
.alter database SalesDB policy streamingingestion enable
```
