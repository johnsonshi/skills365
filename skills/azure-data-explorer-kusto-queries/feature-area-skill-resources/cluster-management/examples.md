# Azure Data Explorer Cluster Management Examples

## Azure CLI

### Create Cluster

```bash
# Production cluster
az kusto cluster create \
  --cluster-name myadxcluster \
  --resource-group myResourceGroup \
  --location eastus2 \
  --sku name="Standard_E8ads_v5" tier="Standard" capacity=2 \
  --enable-streaming-ingest true \
  --enable-auto-stop true

# Dev/Test cluster
az kusto cluster create \
  --cluster-name myadxdevcluster \
  --resource-group myResourceGroup \
  --location eastus2 \
  --sku name="Dev(No SLA)_Standard_E2a_v4" tier="Basic" capacity=1
```

### Create Database

```bash
az kusto database create \
  --cluster-name myadxcluster \
  --resource-group myResourceGroup \
  --database-name maindb \
  --read-write-database soft-delete-period=P365D hot-cache-period=P31D location=eastus2
```

### Cluster Operations

```bash
# Start cluster
az kusto cluster start \
  --cluster-name myadxcluster \
  --resource-group myResourceGroup

# Stop cluster
az kusto cluster stop \
  --cluster-name myadxcluster \
  --resource-group myResourceGroup

# Show cluster details
az kusto cluster show \
  --cluster-name myadxcluster \
  --resource-group myResourceGroup

# Delete cluster
az kusto cluster delete \
  --cluster-name myadxcluster \
  --resource-group myResourceGroup \
  --yes
```

### Update Cluster

```bash
# Update SKU (vertical scaling)
az kusto cluster update \
  --cluster-name myadxcluster \
  --resource-group myResourceGroup \
  --sku name="Standard_E16ads_v5" tier="Standard" capacity=2

# Update capacity (horizontal scaling)
az kusto cluster update \
  --cluster-name myadxcluster \
  --resource-group myResourceGroup \
  --sku name="Standard_E8ads_v5" tier="Standard" capacity=4

# Enable/disable auto-stop
az kusto cluster update \
  --cluster-name myadxcluster \
  --resource-group myResourceGroup \
  --enable-auto-stop true
```

### Enable Autoscale via REST

```bash
az rest --method patch \
  --uri "https://management.azure.com/subscriptions/{subscription-id}/resourceGroups/{resource-group}/providers/Microsoft.Kusto/clusters/{cluster-name}?api-version=2023-08-15" \
  --body '{
    "properties": {
      "optimizedAutoscale": {
        "version": 1,
        "isEnabled": true,
        "minimum": 2,
        "maximum": 15
      }
    }
  }'
```

---

## PowerShell

### Create Cluster

```powershell
# Production cluster
New-AzKustoCluster `
  -ResourceGroupName "myResourceGroup" `
  -Name "myadxcluster" `
  -Location "eastus2" `
  -SkuName "Standard_E8ads_v5" `
  -SkuTier "Standard" `
  -SkuCapacity 2 `
  -EnableStreamingIngest $true `
  -EnableAutoStop $true

# Dev/Test cluster
New-AzKustoCluster `
  -ResourceGroupName "myResourceGroup" `
  -Name "myadxdevcluster" `
  -Location "eastus2" `
  -SkuName "Dev(No SLA)_Standard_E2a_v4" `
  -SkuTier "Basic" `
  -SkuCapacity 1
```

### Create Database

```powershell
New-AzKustoDatabase `
  -ResourceGroupName "myResourceGroup" `
  -ClusterName "myadxcluster" `
  -Name "maindb" `
  -Kind "ReadWrite" `
  -SoftDeletePeriod (New-TimeSpan -Days 365) `
  -HotCachePeriod (New-TimeSpan -Days 31)
```

### Cluster Operations

```powershell
# Get cluster
Get-AzKustoCluster `
  -ResourceGroupName "myResourceGroup" `
  -Name "myadxcluster"

# Start cluster
Start-AzKustoCluster `
  -ResourceGroupName "myResourceGroup" `
  -Name "myadxcluster"

# Stop cluster
Stop-AzKustoCluster `
  -ResourceGroupName "myResourceGroup" `
  -Name "myadxcluster"

# Remove cluster
Remove-AzKustoCluster `
  -ResourceGroupName "myResourceGroup" `
  -Name "myadxcluster"
```

### Update Cluster

```powershell
# Update SKU
Update-AzKustoCluster `
  -ResourceGroupName "myResourceGroup" `
  -Name "myadxcluster" `
  -SkuName "Standard_E16ads_v5" `
  -SkuTier "Standard" `
  -SkuCapacity 2

# Disable auto-stop
Update-AzKustoCluster `
  -ResourceGroupName "myResourceGroup" `
  -Name "myadxcluster" `
  -EnableAutoStop $false
```

### Attach Follower Database

```powershell
$LeaderCluster = Get-AzKustoCluster `
  -Name "leader-cluster" `
  -ResourceGroupName "leader-rg"

New-AzKustoAttachedDatabaseConfiguration `
  -ClusterName "follower-cluster" `
  -Name "shared-database-config" `
  -ResourceGroupName "follower-rg" `
  -DatabaseName "shared-database" `
  -ClusterResourceId $LeaderCluster.Id `
  -DefaultPrincipalsModificationKind "Union" `
  -Location $LeaderCluster.Location
```

---

## ARM Templates

### Production Cluster

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "clusterName": { "type": "string", "minLength": 4, "maxLength": 22 },
    "location": { "type": "string", "defaultValue": "[resourceGroup().location]" },
    "skuName": { "type": "string", "defaultValue": "Standard_E8ads_v5" },
    "capacity": { "type": "int", "defaultValue": 2, "minValue": 2 },
    "autoScaleMin": { "type": "int", "defaultValue": 2 },
    "autoScaleMax": { "type": "int", "defaultValue": 10 }
  },
  "resources": [
    {
      "name": "[parameters('clusterName')]",
      "type": "Microsoft.Kusto/clusters",
      "apiVersion": "2023-08-15",
      "location": "[parameters('location')]",
      "sku": {
        "name": "[parameters('skuName')]",
        "tier": "Standard",
        "capacity": "[parameters('capacity')]"
      },
      "zones": ["1", "2", "3"],
      "identity": { "type": "SystemAssigned" },
      "properties": {
        "enableAutoStop": true,
        "enableStreamingIngest": true,
        "engineType": "V3",
        "optimizedAutoscale": {
          "version": 1,
          "isEnabled": true,
          "minimum": "[parameters('autoScaleMin')]",
          "maximum": "[parameters('autoScaleMax')]"
        }
      }
    }
  ],
  "outputs": {
    "clusterUri": { "type": "string", "value": "[reference(parameters('clusterName')).uri]" },
    "dataIngestionUri": { "type": "string", "value": "[reference(parameters('clusterName')).dataIngestionUri]" }
  }
}
```

### Dev/Test Cluster

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "clusterName": { "type": "string" },
    "location": { "type": "string", "defaultValue": "[resourceGroup().location]" }
  },
  "resources": [
    {
      "name": "[parameters('clusterName')]",
      "type": "Microsoft.Kusto/clusters",
      "apiVersion": "2023-08-15",
      "location": "[parameters('location')]",
      "sku": {
        "name": "Dev(No SLA)_Standard_E2a_v4",
        "tier": "Basic",
        "capacity": 1
      },
      "properties": {
        "enableAutoStop": true,
        "enableStreamingIngest": true,
        "engineType": "V3"
      }
    }
  ]
}
```

---

## Bicep Templates

### Cluster with Database

```bicep
@minLength(4)
@maxLength(22)
param clusterName string

param location string = resourceGroup().location

@allowed(['Standard_E8ads_v5', 'Standard_E16ads_v5', 'Standard_L8as_v3'])
param skuName string = 'Standard_E8ads_v5'

@minValue(2)
param capacity int = 2

param databaseName string = 'maindb'
param hotCachePeriodInDays int = 31
param softDeletePeriodInDays int = 365

resource cluster 'Microsoft.Kusto/clusters@2023-08-15' = {
  name: clusterName
  location: location
  sku: {
    name: skuName
    tier: 'Standard'
    capacity: capacity
  }
  zones: ['1', '2', '3']
  identity: { type: 'SystemAssigned' }
  properties: {
    enableAutoStop: true
    enableStreamingIngest: true
    engineType: 'V3'
    optimizedAutoscale: {
      version: 1
      isEnabled: true
      minimum: 2
      maximum: 10
    }
  }
}

resource database 'Microsoft.Kusto/clusters/databases@2023-08-15' = {
  parent: cluster
  name: databaseName
  location: location
  kind: 'ReadWrite'
  properties: {
    softDeletePeriod: 'P${softDeletePeriodInDays}D'
    hotCachePeriod: 'P${hotCachePeriodInDays}D'
  }
}

output clusterUri string = cluster.properties.uri
output dataIngestionUri string = cluster.properties.dataIngestionUri
output principalId string = cluster.identity.principalId
```

---

## KQL Health Check Commands

```kusto
// Basic health check
.show diagnostics
| project IsHealthy, NotHealthyReason, IsAttentionRequired, IsScaleOutRequired

// Cluster capacity overview
.show capacity

// Check cluster policies
.show cluster policy caching
.show cluster policy capacity

// Recent operations summary
.show operations
| where StartedOn > ago(1d)
| summarize count() by State, OperationType
| order by count_ desc

// Running queries
.show running queries

// Database sizes
.show databases
| project DatabaseName, PersistentStorage, SizeInMB
```

---

## Monitoring Alert Rules (ARM)

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "clusterName": { "type": "string" },
    "actionGroupId": { "type": "string" }
  },
  "variables": {
    "clusterId": "[resourceId('Microsoft.Kusto/clusters', parameters('clusterName'))]"
  },
  "resources": [
    {
      "type": "Microsoft.Insights/metricAlerts",
      "apiVersion": "2018-03-01",
      "name": "ADX-HighCPU-Alert",
      "location": "global",
      "properties": {
        "description": "Alert when CPU exceeds 80%",
        "severity": 2,
        "enabled": true,
        "scopes": ["[variables('clusterId')]"],
        "evaluationFrequency": "PT5M",
        "windowSize": "PT15M",
        "criteria": {
          "odata.type": "Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria",
          "allOf": [{
            "name": "HighCPU",
            "metricName": "CPU",
            "operator": "GreaterThan",
            "threshold": 80,
            "timeAggregation": "Average"
          }]
        },
        "actions": [{ "actionGroupId": "[parameters('actionGroupId')]" }]
      }
    },
    {
      "type": "Microsoft.Insights/metricAlerts",
      "apiVersion": "2018-03-01",
      "name": "ADX-LowKeepAlive-Alert",
      "location": "global",
      "properties": {
        "description": "Alert when cluster health degrades",
        "severity": 1,
        "enabled": true,
        "scopes": ["[variables('clusterId')]"],
        "evaluationFrequency": "PT1M",
        "windowSize": "PT5M",
        "criteria": {
          "odata.type": "Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria",
          "allOf": [{
            "name": "LowKeepAlive",
            "metricName": "KeepAlive",
            "operator": "LessThan",
            "threshold": 0.9,
            "timeAggregation": "Average"
          }]
        },
        "actions": [{ "actionGroupId": "[parameters('actionGroupId')]" }]
      }
    }
  ]
}
```

---

## Custom Autoscale Rules

```json
{
  "profiles": [{
    "name": "DefaultProfile",
    "capacity": { "minimum": "2", "maximum": "10", "default": "2" },
    "rules": [
      {
        "metricTrigger": {
          "metricName": "CPU",
          "metricResourceUri": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Kusto/clusters/{cluster}",
          "timeGrain": "PT1M",
          "statistic": "Average",
          "timeWindow": "PT10M",
          "timeAggregation": "Average",
          "operator": "GreaterThan",
          "threshold": 70
        },
        "scaleAction": { "direction": "Increase", "type": "ChangeCount", "value": "1", "cooldown": "PT5M" }
      },
      {
        "metricTrigger": {
          "metricName": "CPU",
          "metricResourceUri": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Kusto/clusters/{cluster}",
          "timeGrain": "PT1M",
          "statistic": "Average",
          "timeWindow": "PT30M",
          "timeAggregation": "Average",
          "operator": "LessThan",
          "threshold": 30
        },
        "scaleAction": { "direction": "Decrease", "type": "ChangeCount", "value": "1", "cooldown": "PT30M" }
      },
      {
        "metricTrigger": {
          "metricName": "CacheUtilization",
          "metricResourceUri": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Kusto/clusters/{cluster}",
          "timeGrain": "PT1M",
          "statistic": "Average",
          "timeWindow": "PT60M",
          "timeAggregation": "Average",
          "operator": "GreaterThan",
          "threshold": 80
        },
        "scaleAction": { "direction": "Increase", "type": "ChangeCount", "value": "1", "cooldown": "PT10M" }
      }
    ]
  }]
}
```

---

## Follower Database (ARM)

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "followerClusterName": { "type": "string" },
    "leaderClusterResourceId": { "type": "string" },
    "databaseName": { "type": "string" },
    "location": { "type": "string" }
  },
  "resources": [{
    "type": "Microsoft.Kusto/clusters/attachedDatabaseConfigurations",
    "apiVersion": "2023-08-15",
    "name": "[concat(parameters('followerClusterName'), '/', parameters('databaseName'), '-config')]",
    "location": "[parameters('location')]",
    "properties": {
      "databaseName": "[parameters('databaseName')]",
      "clusterResourceId": "[parameters('leaderClusterResourceId')]",
      "defaultPrincipalsModificationKind": "Union"
    }
  }]
}
```

---

## REST API

### Create Cluster

```bash
curl -X PUT \
  "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroup}/providers/Microsoft.Kusto/clusters/{clusterName}?api-version=2023-08-15" \
  -H "Authorization: Bearer {accessToken}" \
  -H "Content-Type: application/json" \
  -d '{
    "location": "eastus2",
    "sku": { "name": "Standard_E8ads_v5", "tier": "Standard", "capacity": 2 },
    "properties": {
      "enableStreamingIngest": true,
      "enableAutoStop": true,
      "optimizedAutoscale": { "version": 1, "isEnabled": true, "minimum": 2, "maximum": 10 }
    }
  }'
```

### Update Auto-Stop

```bash
curl -X PATCH \
  "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroup}/providers/Microsoft.Kusto/clusters/{clusterName}?api-version=2023-08-15" \
  -H "Authorization: Bearer {accessToken}" \
  -H "Content-Type: application/json" \
  -d '{ "properties": { "enableAutoStop": false } }'
```
