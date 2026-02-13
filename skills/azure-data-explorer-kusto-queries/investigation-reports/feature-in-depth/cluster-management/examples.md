# Azure Data Explorer Cluster Management Examples

## Overview

This document provides practical examples for Azure Data Explorer cluster management operations, including ARM/Bicep templates, Azure CLI commands, PowerShell scripts, and scaling configurations.

---

## Cluster Creation Examples

### ARM Template - Production Cluster

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "clusterName": {
            "type": "string",
            "minLength": 4,
            "maxLength": 22,
            "metadata": {
                "description": "Name of the Azure Data Explorer cluster"
            }
        },
        "location": {
            "type": "string",
            "defaultValue": "[resourceGroup().location]",
            "metadata": {
                "description": "Location for the cluster"
            }
        },
        "skuName": {
            "type": "string",
            "defaultValue": "Standard_E8ads_v5",
            "allowedValues": [
                "Standard_E2ads_v5",
                "Standard_E4ads_v5",
                "Standard_E8ads_v5",
                "Standard_E16ads_v5",
                "Standard_L8as_v3",
                "Standard_L16as_v3",
                "Standard_L32as_v3"
            ],
            "metadata": {
                "description": "SKU name for the cluster"
            }
        },
        "skuTier": {
            "type": "string",
            "defaultValue": "Standard",
            "allowedValues": ["Standard", "Basic"],
            "metadata": {
                "description": "SKU tier"
            }
        },
        "capacity": {
            "type": "int",
            "defaultValue": 2,
            "minValue": 2,
            "maxValue": 1000,
            "metadata": {
                "description": "Number of instances"
            }
        },
        "enableAutoScale": {
            "type": "bool",
            "defaultValue": true,
            "metadata": {
                "description": "Enable optimized autoscale"
            }
        },
        "autoScaleMin": {
            "type": "int",
            "defaultValue": 2,
            "metadata": {
                "description": "Minimum instances for autoscale"
            }
        },
        "autoScaleMax": {
            "type": "int",
            "defaultValue": 10,
            "metadata": {
                "description": "Maximum instances for autoscale"
            }
        },
        "enableAvailabilityZones": {
            "type": "bool",
            "defaultValue": true,
            "metadata": {
                "description": "Enable availability zones"
            }
        }
    },
    "variables": {
        "zones": "[if(parameters('enableAvailabilityZones'), createArray('1', '2', '3'), createArray())]"
    },
    "resources": [
        {
            "name": "[parameters('clusterName')]",
            "type": "Microsoft.Kusto/clusters",
            "apiVersion": "2023-08-15",
            "location": "[parameters('location')]",
            "sku": {
                "name": "[parameters('skuName')]",
                "tier": "[parameters('skuTier')]",
                "capacity": "[parameters('capacity')]"
            },
            "zones": "[variables('zones')]",
            "identity": {
                "type": "SystemAssigned"
            },
            "properties": {
                "enableAutoStop": true,
                "enableStreamingIngest": true,
                "enablePurge": false,
                "enableDoubleEncryption": false,
                "engineType": "V3",
                "optimizedAutoscale": {
                    "version": 1,
                    "isEnabled": "[parameters('enableAutoScale')]",
                    "minimum": "[parameters('autoScaleMin')]",
                    "maximum": "[parameters('autoScaleMax')]"
                },
                "publicNetworkAccess": "Enabled"
            },
            "tags": {
                "environment": "production",
                "createdBy": "ARM-template"
            }
        }
    ],
    "outputs": {
        "clusterUri": {
            "type": "string",
            "value": "[reference(parameters('clusterName')).uri]"
        },
        "dataIngestionUri": {
            "type": "string",
            "value": "[reference(parameters('clusterName')).dataIngestionUri]"
        }
    }
}
```

### ARM Template - Dev/Test Cluster

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "clusterName": {
            "type": "string",
            "metadata": {
                "description": "Name of the dev cluster"
            }
        },
        "location": {
            "type": "string",
            "defaultValue": "[resourceGroup().location]"
        }
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
            },
            "tags": {
                "environment": "development"
            }
        }
    ]
}
```

### Bicep Template - Cluster with Database

```bicep
@description('Name of the Azure Data Explorer cluster')
@minLength(4)
@maxLength(22)
param clusterName string

@description('Location for all resources')
param location string = resourceGroup().location

@description('SKU name')
@allowed([
  'Standard_E8ads_v5'
  'Standard_E16ads_v5'
  'Standard_L8as_v3'
  'Standard_L16as_v3'
])
param skuName string = 'Standard_E8ads_v5'

@description('Number of instances')
@minValue(2)
@maxValue(100)
param capacity int = 2

@description('Database name')
param databaseName string = 'maindb'

@description('Hot cache period in days')
param hotCachePeriodInDays int = 31

@description('Soft delete period in days')
param softDeletePeriodInDays int = 365

// Cluster resource
resource cluster 'Microsoft.Kusto/clusters@2023-08-15' = {
  name: clusterName
  location: location
  sku: {
    name: skuName
    tier: 'Standard'
    capacity: capacity
  }
  zones: [
    '1'
    '2'
    '3'
  ]
  identity: {
    type: 'SystemAssigned'
  }
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

// Database resource
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

// Outputs
output clusterUri string = cluster.properties.uri
output dataIngestionUri string = cluster.properties.dataIngestionUri
output principalId string = cluster.identity.principalId
```

---

## Azure CLI Examples

### Create Cluster

```bash
# Create resource group
az group create \
  --name myResourceGroup \
  --location eastus2

# Create production cluster
az kusto cluster create \
  --cluster-name myadxcluster \
  --resource-group myResourceGroup \
  --location eastus2 \
  --sku name="Standard_E8ads_v5" tier="Standard" capacity=2 \
  --enable-streaming-ingest true \
  --enable-auto-stop true

# Create dev/test cluster
az kusto cluster create \
  --cluster-name myadxdevcluster \
  --resource-group myResourceGroup \
  --location eastus2 \
  --sku name="Dev(No SLA)_Standard_E2a_v4" tier="Basic" capacity=1
```

### Create Database

```bash
# Create database with retention and cache policies
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

# List clusters in resource group
az kusto cluster list \
  --resource-group myResourceGroup

# Delete cluster
az kusto cluster delete \
  --cluster-name myadxcluster \
  --resource-group myResourceGroup \
  --yes
```

### Update Cluster Configuration

```bash
# Enable/disable auto-stop
az kusto cluster update \
  --cluster-name myadxcluster \
  --resource-group myResourceGroup \
  --enable-auto-stop true

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
```

---

## PowerShell Examples

### Create Cluster

```powershell
# Connect to Azure
Connect-AzAccount

# Set subscription
Set-AzContext -SubscriptionId "your-subscription-id"

# Create resource group
New-AzResourceGroup -Name "myResourceGroup" -Location "eastus2"

# Create production cluster
New-AzKustoCluster `
  -ResourceGroupName "myResourceGroup" `
  -Name "myadxcluster" `
  -Location "eastus2" `
  -SkuName "Standard_E8ads_v5" `
  -SkuTier "Standard" `
  -SkuCapacity 2 `
  -EnableStreamingIngest $true `
  -EnableAutoStop $true

# Create dev cluster
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
# Create database
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
# Get cluster details
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

---

## Autoscale Configuration Examples

### ARM Template - Optimized Autoscale

```json
{
    "properties": {
        "optimizedAutoscale": {
            "version": 1,
            "isEnabled": true,
            "minimum": 2,
            "maximum": 10
        }
    }
}
```

### Azure CLI - Enable Autoscale

```bash
# Enable optimized autoscale via REST API
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

### Custom Autoscale Rules (Azure Monitor)

```json
{
    "autoscale_setting_resource_name": "adx-custom-autoscale",
    "profiles": [
        {
            "name": "DefaultProfile",
            "capacity": {
                "minimum": "2",
                "maximum": "10",
                "default": "2"
            },
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
                    "scaleAction": {
                        "direction": "Increase",
                        "type": "ChangeCount",
                        "value": "1",
                        "cooldown": "PT5M"
                    }
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
                    "scaleAction": {
                        "direction": "Decrease",
                        "type": "ChangeCount",
                        "value": "1",
                        "cooldown": "PT30M"
                    }
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
                    "scaleAction": {
                        "direction": "Increase",
                        "type": "ChangeCount",
                        "value": "1",
                        "cooldown": "PT10M"
                    }
                }
            ]
        }
    ]
}
```

---

## Monitoring and Health Check Examples

### KQL Health Check Commands

```kusto
// Basic health check
.show diagnostics
| project IsHealthy, NotHealthyReason, IsAttentionRequired, AttentionRequiredReason, IsScaleOutRequired

// Cluster capacity overview
.show capacity

// Check cluster policies
.show cluster policy caching
.show cluster policy capacity

// View recent operations
.show operations
| where StartedOn > ago(1d)
| summarize count() by State, OperationType
| order by count_ desc

// Check running queries
.show running queries

// Database sizes
.show databases
| project DatabaseName, PersistentStorage, HotDataSize = SizeInMB * 1024 * 1024
```

### Azure Monitor Alert Rules (ARM Template)

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
                    "allOf": [
                        {
                            "name": "HighCPU",
                            "metricName": "CPU",
                            "operator": "GreaterThan",
                            "threshold": 80,
                            "timeAggregation": "Average"
                        }
                    ]
                },
                "actions": [
                    {
                        "actionGroupId": "[parameters('actionGroupId')]"
                    }
                ]
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
                    "allOf": [
                        {
                            "name": "LowKeepAlive",
                            "metricName": "KeepAlive",
                            "operator": "LessThan",
                            "threshold": 0.9,
                            "timeAggregation": "Average"
                        }
                    ]
                },
                "actions": [
                    {
                        "actionGroupId": "[parameters('actionGroupId')]"
                    }
                ]
            }
        }
    ]
}
```

---

## Follower Database Configuration

### ARM Template - Attach Follower Database

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
    "resources": [
        {
            "type": "Microsoft.Kusto/clusters/attachedDatabaseConfigurations",
            "apiVersion": "2023-08-15",
            "name": "[concat(parameters('followerClusterName'), '/', parameters('databaseName'), '-config')]",
            "location": "[parameters('location')]",
            "properties": {
                "databaseName": "[parameters('databaseName')]",
                "clusterResourceId": "[parameters('leaderClusterResourceId')]",
                "defaultPrincipalsModificationKind": "Union",
                "tableLevelSharingProperties": {
                    "tablesToInclude": [],
                    "tablesToExclude": [],
                    "externalTablesToInclude": [],
                    "externalTablesToExclude": []
                }
            }
        }
    ]
}
```

### PowerShell - Attach Follower Database

```powershell
$FollowerClusterName = 'follower-cluster'
$FollowerResourceGroup = 'follower-rg'
$LeaderClusterName = 'leader-cluster'
$LeaderResourceGroup = 'leader-rg'
$LeaderSubscriptionId = 'leader-subscription-id'
$DatabaseName = 'shared-database'

# Get leader cluster details
$LeaderCluster = Get-AzKustoCluster `
  -Name $LeaderClusterName `
  -ResourceGroupName $LeaderResourceGroup `
  -SubscriptionId $LeaderSubscriptionId

# Attach database
New-AzKustoAttachedDatabaseConfiguration `
  -ClusterName $FollowerClusterName `
  -Name "$DatabaseName-config" `
  -ResourceGroupName $FollowerResourceGroup `
  -DatabaseName $DatabaseName `
  -ClusterResourceId $LeaderCluster.Id `
  -DefaultPrincipalsModificationKind "Union" `
  -Location $LeaderCluster.Location
```

---

## CI/CD Pipeline Example

### Azure DevOps Pipeline (YAML)

```yaml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - infrastructure/adx/*

variables:
  azureSubscription: 'Azure-ServiceConnection'
  resourceGroupName: 'adx-rg'
  location: 'eastus2'
  clusterName: 'adxcluster$(Build.BuildId)'

stages:
  - stage: Deploy_Infrastructure
    displayName: 'Deploy ADX Cluster'
    jobs:
      - job: DeployCluster
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: AzureResourceManagerTemplateDeployment@3
            displayName: 'Deploy ADX Cluster'
            inputs:
              deploymentScope: 'Resource Group'
              azureResourceManagerConnection: '$(azureSubscription)'
              subscriptionId: '$(subscriptionId)'
              action: 'Create Or Update Resource Group'
              resourceGroupName: '$(resourceGroupName)'
              location: '$(location)'
              templateLocation: 'Linked artifact'
              csmFile: 'infrastructure/adx/cluster.bicep'
              overrideParameters: '-clusterName $(clusterName)'
              deploymentMode: 'Incremental'

          - task: AzureCLI@2
            displayName: 'Create Database'
            inputs:
              azureSubscription: '$(azureSubscription)'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                az kusto database create \
                  --cluster-name $(clusterName) \
                  --resource-group $(resourceGroupName) \
                  --database-name maindb \
                  --read-write-database soft-delete-period=P365D hot-cache-period=P31D location=$(location)

  - stage: Configure_Schema
    displayName: 'Configure Database Schema'
    dependsOn: Deploy_Infrastructure
    jobs:
      - job: ConfigureSchema
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: AzureCLI@2
            displayName: 'Run Schema Script'
            inputs:
              azureSubscription: '$(azureSubscription)'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                # Install Kusto CLI tools
                pip install azure-kusto-data azure-kusto-ingest

                # Run schema script
                python scripts/deploy_schema.py \
                  --cluster $(clusterName).$(location).kusto.windows.net \
                  --database maindb \
                  --script infrastructure/adx/schema.kql
```

---

## REST API Examples

### Create Cluster via REST

```bash
# Create cluster
curl -X PUT \
  "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroup}/providers/Microsoft.Kusto/clusters/{clusterName}?api-version=2023-08-15" \
  -H "Authorization: Bearer {accessToken}" \
  -H "Content-Type: application/json" \
  -d '{
    "location": "eastus2",
    "sku": {
      "name": "Standard_E8ads_v5",
      "tier": "Standard",
      "capacity": 2
    },
    "properties": {
      "enableStreamingIngest": true,
      "enableAutoStop": true,
      "optimizedAutoscale": {
        "version": 1,
        "isEnabled": true,
        "minimum": 2,
        "maximum": 10
      }
    }
  }'
```

### Update Auto-Stop via REST

```bash
# Disable auto-stop
curl -X PATCH \
  "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroup}/providers/Microsoft.Kusto/clusters/{clusterName}?api-version=2023-08-15" \
  -H "Authorization: Bearer {accessToken}" \
  -H "Content-Type: application/json" \
  -d '{
    "properties": {
      "enableAutoStop": false
    }
  }'
```

---

## Document Information

- **Generated**: 2026-02-13
- **Source**: Azure Data Explorer documentation analysis
- **Phase**: 3 - Feature In-Depth Research
