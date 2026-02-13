# Azure Data Explorer Business Continuity Examples

## Overview

This document provides practical examples for implementing business continuity features in Azure Data Explorer, including follower database setup, cross-cluster queries, and recovery procedures.

---

## Follower Database Setup

### Example 1: Basic Follower Database Configuration (PowerShell)

Attach a single database from a leader cluster to a follower cluster.

```powershell
# Variables
$FollowerClusterName = 'follower-cluster'
$FollowerSubscriptionId = 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
$FollowerResourceGroupName = 'follower-rg'
$LeaderClusterName = 'leader-cluster'
$LeaderSubscriptionId = 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
$LeaderResourceGroupName = 'leader-rg'
$DatabaseName = 'SalesDB'

# Get leader cluster details
$LeaderCluster = Get-AzKustoCluster -Name $LeaderClusterName `
    -ResourceGroupName $LeaderResourceGroupName `
    -SubscriptionId $LeaderSubscriptionId

# Create attached database configuration
New-AzKustoAttachedDatabaseConfiguration `
    -ClusterName $FollowerClusterName `
    -Name $DatabaseName `
    -ResourceGroupName $FollowerResourceGroupName `
    -SubscriptionId $FollowerSubscriptionId `
    -DatabaseName $DatabaseName `
    -ClusterResourceId $LeaderCluster.Id `
    -DefaultPrincipalsModificationKind 'Union' `
    -Location $LeaderCluster.Location
```

### Example 2: Follower with Table-Level Sharing (PowerShell)

Share only specific tables from the leader database.

```powershell
# Follow only tables starting with "Sales" and "Inventory"
New-AzKustoAttachedDatabaseConfiguration `
    -ClusterName $FollowerClusterName `
    -Name 'SalesDBConfig' `
    -ResourceGroupName $FollowerResourceGroupName `
    -SubscriptionId $FollowerSubscriptionId `
    -DatabaseName 'SalesDB' `
    -ClusterResourceId $LeaderCluster.Id `
    -DefaultPrincipalsModificationKind 'Union' `
    -Location $LeaderCluster.Location `
    -TableLevelSharingPropertyTablesToInclude "Sales*", "Inventory*" `
    -TableLevelSharingPropertyExternalTablesToExclude "*"
```

### Example 3: Follower Database with Different Name (C#)

Attach a database with a different name on the follower cluster.

```csharp
using Azure.Identity;
using Azure.ResourceManager;
using Azure.ResourceManager.Kusto;
using Azure.ResourceManager.Kusto.Models;

// Configuration
var followerSubscriptionId = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx";
var followerResourceGroup = "follower-rg";
var followerClusterName = "follower-cluster";
var leaderSubscriptionId = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx";
var leaderResourceGroup = "leader-rg";
var leaderClusterName = "leader-cluster";
var leaderDatabaseName = "ProductionDB";
var followerDatabaseName = "ProductionDB_ReadOnly"; // Different name

// Create ARM client
var credential = new DefaultAzureCredential();
var armClient = new ArmClient(credential);

// Get follower cluster
var followerClusterId = KustoClusterResource.CreateResourceIdentifier(
    followerSubscriptionId, followerResourceGroup, followerClusterName);
var followerCluster = armClient.GetKustoClusterResource(followerClusterId);

// Get leader cluster resource ID
var leaderClusterId = KustoClusterResource.CreateResourceIdentifier(
    leaderSubscriptionId, leaderResourceGroup, leaderClusterName);

// Create attached database configuration
var attachedDbConfigs = followerCluster.GetKustoAttachedDatabaseConfigurations();
var configData = new KustoAttachedDatabaseConfigurationData
{
    ClusterResourceId = leaderClusterId,
    DatabaseName = leaderDatabaseName,
    DatabaseNameOverride = followerDatabaseName, // Override name
    DefaultPrincipalsModificationKind = KustoDatabaseDefaultPrincipalsModificationKind.Union,
    Location = AzureLocation.WestEurope
};

await attachedDbConfigs.CreateOrUpdateAsync(
    WaitUntil.Completed,
    "ProductionDBConfig",
    configData);

Console.WriteLine($"Follower database '{followerDatabaseName}' attached successfully");
```

### Example 4: Follow All Databases (PowerShell)

Follow all databases from a leader cluster.

```powershell
# Use '*' to follow all databases
$ConfigName = $FollowerClusterName + 'AllDBsConfig'

New-AzKustoAttachedDatabaseConfiguration `
    -ClusterName $FollowerClusterName `
    -Name $ConfigName `
    -ResourceGroupName $FollowerResourceGroupName `
    -SubscriptionId $FollowerSubscriptionId `
    -DatabaseName '*' `
    -ClusterResourceId $LeaderCluster.Id `
    -DefaultPrincipalsModificationKind 'Union' `
    -Location $LeaderCluster.Location

# Note: Table-level sharing not supported with '*' notation
```

---

## Follower Database Management

### Example 5: Configure Caching Policy on Follower

Override the leader's caching policy on the follower.

```kusto
// Set database-level caching policy (7 days hot cache)
.alter follower database SalesDB policy caching hot = 7d

// Set table-specific caching (override for specific tables)
.alter follower database SalesDB table RecentTransactions policy caching hot = 30d
.alter follower database SalesDB table HistoricalData policy caching hot = 1d

// Replace leader's caching policy entirely
.alter follower database SalesDB caching-policies-modification-kind = replace
```

### Example 6: Manage Follower Principals

Add additional principals to the follower database.

```kusto
// Add a viewer group to the follower database
.add follower database SalesDB viewers ('aadgroup=analytics-team@contoso.com') 'Analytics Team'

// Add an admin user
.add follower database SalesDB admins ('aaduser=admin@contoso.com')

// Use replace mode to use only follower-defined principals
.alter follower database SalesDB principals-modification-kind = replace

// View current follower configuration
.show follower database SalesDB
| evaluate narrow()
```

### Example 7: Detach Follower Database

Remove a follower database attachment.

```powershell
# Get the attached database configuration name
$Configs = Get-AzKustoAttachedDatabaseConfiguration `
    -ClusterName $FollowerClusterName `
    -ResourceGroupName $FollowerResourceGroupName `
    -SubscriptionId $FollowerSubscriptionId

# Find the config for SalesDB
$ConfigToRemove = $Configs | Where-Object { $_.DatabaseName -eq 'SalesDB' }
$ConfigName = $ConfigToRemove.Name.Split("/")[1]

# Remove the attachment
Remove-AzKustoAttachedDatabaseConfiguration `
    -ClusterName $FollowerClusterName `
    -Name $ConfigName `
    -ResourceGroupName $FollowerResourceGroupName `
    -SubscriptionId $FollowerSubscriptionId
```

---

## Cross-Cluster Queries

### Example 8: Basic Cross-Cluster Query

Query data from a remote cluster.

```kusto
// Query a table from another cluster
cluster("analytics-cluster.westeurope.kusto.windows.net")
    .database("MetricsDB")
    .ServerMetrics
| where Timestamp > ago(1h)
| summarize AvgCPU = avg(CPUPercent) by bin(Timestamp, 5m), ServerName

// Join local and remote data
let remoteData = cluster("remote-cluster").database("RemoteDB").RemoteTable;
LocalTable
| join kind=inner remoteData on $left.Id == $right.Id
| project LocalColumn, RemoteColumn, JoinedTimestamp
```

### Example 9: Cross-Database Query (Same Cluster)

Query across databases in the same cluster.

```kusto
// Reference another database
database("InventoryDB").Products
| join kind=inner database("SalesDB").Transactions
    on $left.ProductId == $right.ProductId
| summarize TotalSales = sum(Amount) by ProductName

// Use union across databases
union database("DB1").Table1, database("DB2").Table2, database("DB3").Table3
| where Timestamp > ago(24h)
| count
```

### Example 10: Cross-Cluster Function Call

Call a function defined on a remote cluster.

```kusto
// Remote cluster has function: GetActiveUsers(timeRange:timespan)
cluster("user-analytics-cluster").database("UserDB").GetActiveUsers(1h)
| summarize Count = count() by Region

// Note: Remote functions must return tabular data
// Scalar functions cannot be called cross-cluster
```

### Example 11: Wildcard Cross-Database Union

Query multiple databases and tables using wildcards.

```kusto
// Union all tables from databases matching pattern
union cluster("prod-cluster").database("Logs*").*
| where Level == "Error"
| summarize ErrorCount = count() by Database = database_name(), Table = table_name()

// Union specific table pattern across databases
union database("*").EventLogs
| where EventType == "SecurityAlert"
| take 100
```

### Example 12: Clear Remote Schema Cache

Refresh cached schema after remote changes.

```kusto
// Clear cache for specific database
.clear cache remote-schema
    cluster("remote-cluster.kusto.windows.net")
    .database("UpdatedDB")

// Useful when:
// - Columns added/removed on remote cluster
// - New tables created
// - Schema modifications made
```

---

## Continuous Export for DR

### Example 13: Setup Continuous Export to GRS Storage

Configure continuous export for disaster recovery backup.

```kusto
// Step 1: Create external table pointing to GRS storage
.create external table DRBackup_Transactions (
    TransactionId: guid,
    CustomerId: long,
    Amount: decimal,
    Timestamp: datetime,
    ProductId: long,
    Region: string
)
kind=blob
partition by (Region: string, Date: datetime = bin(Timestamp, 1d))
dataformat=parquet
(
    h@'https://drstorage.blob.core.windows.net/backups;...'
)
with (folder='dr-backup')

// Step 2: Create continuous export
.create-or-alter continuous-export TransactionBackup
over (Transactions)
to table DRBackup_Transactions
with (
    intervalBetweenRuns = 5m,
    forcedLatency = 2m,
    sizeLimit = 524288000
)
<| Transactions
```

### Example 14: Monitor Continuous Export Health

Track export status and troubleshoot issues.

```kusto
// Check export status
.show continuous-export TransactionBackup
| project Name, ExternalTableName, Query,
          LastRunTime, LastRunResult,
          ExportedTo, IsDisabled

// View export lateness (how far behind real-time)
.show continuous-export TransactionBackup
| project Name,
          LatencyMinutes = datetime_diff('minute', now(), ExportedTo)

// Check for failures
.show continuous-export TransactionBackup failures
| take 20

// View exported artifacts (files created)
.show continuous-export TransactionBackup exported-artifacts
| where Timestamp > ago(1h)
| summarize FileCount = count(), TotalSizeBytes = sum(SizeInBytes)
```

### Example 15: Export Historical Data Before Continuous Export

Backfill data not covered by continuous export.

```kusto
// Get the cursor from continuous export
.show continuous-export TransactionBackup | project StartCursor

// Export historical data (before the cursor)
// Replace cursor value with actual value from above
.export async to table DRBackup_Transactions
with (sizeLimit=1073741824, distributed=true)
<| Transactions
| where cursor_before_or_at("636751928823156645")
| where Timestamp > ago(365d)
```

---

## Recovery Procedures

### Example 16: Clone Database Schema to DR Cluster

Export and recreate schema on disaster recovery cluster.

```kusto
// On SOURCE cluster: Export schema
.show database SalesDB schema as csl script
with (ShowObfuscatedStrings = true)

// Copy the output (CSL script)
// On TARGET cluster: Execute the schema script
.execute database script <|
// Paste the entire CSL script here
// Example content:
.create table Transactions (
    TransactionId: guid,
    CustomerId: long,
    Amount: decimal,
    Timestamp: datetime
) with (folder='Sales')

.create table Customers (
    CustomerId: long,
    Name: string,
    Email: string,
    Region: string
) with (folder='Sales')

// ... rest of schema
```

### Example 17: Recover Dropped Table

Restore an accidentally dropped table.

```kusto
// Check if table can be recovered
// (depends on retention policy recoverability setting)
.show database SalesDB policy retention

// Recover the table
.undo drop table DroppedTableName

// If original name conflicts, recover with new name
// Note: This requires additional steps and may need support
```

### Example 18: Restore Data from Continuous Export

Re-ingest data from external storage after disaster.

```kusto
// Step 1: Create the table with same schema
.create table Transactions (
    TransactionId: guid,
    CustomerId: long,
    Amount: decimal,
    Timestamp: datetime,
    ProductId: long,
    Region: string
)

// Step 2: Ingest from external storage (exported backup)
.ingest into table Transactions (
    h@'https://drstorage.blob.core.windows.net/backups/Region=US/Date=2024-01-15/part_0.parquet;...'
)
with (format='parquet')

// Step 3: For bulk restore, use async ingest from multiple files
.ingest async into table Transactions (
    h@'https://drstorage.blob.core.windows.net/backups/;...'
)
with (
    format='parquet',
    creationTime='2024-01-15'
)
```

### Example 19: Validate DR Cluster Data Consistency

Compare data between primary and DR clusters.

```kusto
// Compare row counts
let primaryCount = toscalar(
    cluster("primary-cluster").database("SalesDB").Transactions
    | summarize count()
);
let drCount = toscalar(Transactions | summarize count());
print Primary = primaryCount, DR = drCount,
      Difference = primaryCount - drCount,
      PercentDiff = round(100.0 * (primaryCount - drCount) / primaryCount, 2)

// Compare recent data hash (simple consistency check)
let timeWindow = ago(1h);
let primaryHash = toscalar(
    cluster("primary-cluster").database("SalesDB").Transactions
    | where Timestamp > timeWindow
    | summarize hash(make_list(TransactionId))
);
let drHash = toscalar(
    Transactions
    | where Timestamp > timeWindow
    | summarize hash(make_list(TransactionId))
);
print PrimaryHash = primaryHash, DRHash = drHash,
      Consistent = (primaryHash == drHash)
```

### Example 20: Failover Ingestion to DR Cluster

Redirect ingestion endpoints during failover.

```kusto
// On DR cluster: Verify streaming ingestion is enabled
.show database SalesDB policy streamingingestion

// Enable if needed
.alter database SalesDB policy streamingingestion enable

// Verify ingestion endpoint
// DR cluster ingestion URL: https://ingest-dr-cluster.region.kusto.windows.net

// Application code should be updated to use DR ingestion endpoint
// Or use Traffic Manager for automatic DNS-based failover
```

---

## Availability Zone Migration

### Example 21: Migrate Cluster to Availability Zones

Add availability zone support to existing cluster.

```powershell
# Get cluster resource ID
$Cluster = Get-AzKustoCluster -Name "mycluster" `
    -ResourceGroupName "myrg"

# Check available zones in the region
$Location = $Cluster.Location
$Bearer = (Get-AzAccessToken).Token
$SubscriptionId = (Get-AzContext).Subscription.Id

$SkuInfo = Invoke-RestMethod `
    -Uri "https://management.azure.com/subscriptions/$SubscriptionId/providers/Microsoft.Kusto/locations/$Location/skus?api-version=2023-05-02" `
    -Headers @{Authorization="Bearer $Bearer"}

$Zones = $SkuInfo.value |
    Where-Object { $_.name -eq $Cluster.SkuName } |
    Select-Object -ExpandProperty locationInfo |
    Select-Object -ExpandProperty zones

Write-Host "Available zones: $($Zones -join ', ')"

# Migrate to availability zones
Update-AzKustoCluster -ResourceGroupName "myrg" `
    -Name "mycluster" `
    -Zone "1", "2", "3"

# Note: Migration can take hours for storage data
# Cluster remains operational during migration
```

---

## Application-Level Failover

### Example 22: C# BCDR Client Implementation

Client that automatically fails over between clusters.

```csharp
using Kusto.Data;
using Kusto.Data.Common;
using Kusto.Data.Net.Client;
using System;
using System.Collections.Generic;
using System.Data;
using System.Threading.Tasks;

public class AdxBcdrClient : IDisposable
{
    private readonly List<ICslQueryProvider> _queryProviders;
    private readonly List<string> _clusterUris;
    private int _currentClusterIndex = 0;

    public AdxBcdrClient(string[] clusterUris, string databaseName)
    {
        _clusterUris = new List<string>(clusterUris);
        _queryProviders = new List<ICslQueryProvider>();

        foreach (var uri in clusterUris)
        {
            var connectionString = new KustoConnectionStringBuilder(uri, databaseName)
                .WithAadUserPromptAuthentication();
            _queryProviders.Add(KustoClientFactory.CreateCslQueryProvider(connectionString));
        }
    }

    public async Task<IDataReader> ExecuteQueryAsync(string query)
    {
        var startIndex = _currentClusterIndex;
        var attempts = 0;

        while (attempts < _queryProviders.Count)
        {
            try
            {
                var provider = _queryProviders[_currentClusterIndex];
                return await Task.Run(() => provider.ExecuteQuery(query));
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Cluster {_clusterUris[_currentClusterIndex]} failed: {ex.Message}");
                _currentClusterIndex = (_currentClusterIndex + 1) % _queryProviders.Count;
                attempts++;
            }
        }

        throw new Exception("All clusters unavailable");
    }

    public void Dispose()
    {
        foreach (var provider in _queryProviders)
        {
            provider?.Dispose();
        }
    }
}

// Usage
var clusters = new[] {
    "https://primary-cluster.westeurope.kusto.windows.net",
    "https://secondary-cluster.northeurope.kusto.windows.net"
};

using var client = new AdxBcdrClient(clusters, "SalesDB");
var result = await client.ExecuteQueryAsync("Transactions | take 10");
```

---

## Document Information

- **Generated**: 2026-02-13
- **Source**: Azure Data Explorer documentation analysis
- **Phase**: 3 - Feature Deep Dive (Business Continuity Examples)
