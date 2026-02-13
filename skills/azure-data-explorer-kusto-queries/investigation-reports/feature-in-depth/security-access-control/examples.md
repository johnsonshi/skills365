# Security and Access Control Examples

This document provides practical examples for implementing security and access control in Azure Data Explorer.

---

## 1. Principal Management Examples

### 1.1 Database Role Management

```kusto
// View all principals assigned to a database
.show database MyDatabase principals

// View your own roles on a database
.show database MyDatabase principal roles

// Add a user as database viewer
.add database MyDatabase viewers ('aaduser=analyst@contoso.com')

// Add a user as database admin
.add database MyDatabase admins ('aaduser=admin@contoso.com')

// Add a Microsoft Entra group as viewers
.add database MyDatabase viewers ('aadgroup=DataAnalysts;contoso.onmicrosoft.com')

// Add a service principal as database user
.add database MyDatabase users ('aadapp=12345678-1234-1234-1234-123456789012;contoso.onmicrosoft.com')

// Add managed identity as ingestor
.add database MyDatabase ingestors ('aadapp=87654321-4321-4321-4321-210987654321;contoso.onmicrosoft.com')
```

### 1.2 Table Role Management

```kusto
// Add a user as table admin
.add table SalesData admins ('aaduser=sales_lead@contoso.com')

// Add ingestor role to a service principal for specific table
.add table IoTEvents ingestors ('aadapp=12345678-1234-1234-1234-123456789012;contoso.onmicrosoft.com')

// View table-level principals
.show table SalesData principals
```

### 1.3 Removing and Replacing Principals

```kusto
// Remove a user from database viewers
.drop database MyDatabase viewers ('aaduser=former_employee@contoso.com')

// Replace all database admins (sets exact list)
.set database MyDatabase admins ('aaduser=admin1@contoso.com', 'aaduser=admin2@contoso.com')

// Remove all ingestors from a database
.set database MyDatabase ingestors none
```

---

## 2. Row-Level Security (RLS) Examples

### 2.1 Basic RLS - Filter by User Email

```kusto
// Create RLS policy that filters data by user's email
.alter table CustomerData policy row_level_security enable
"CustomerData | where SalesRepEmail == current_principal_details()['UserPrincipalName']"
```

### 2.2 RLS with Group Membership

```kusto
// Create a lookup table for user-region assignments
.set-or-append UserRegionAccess <|
datatable(UserPrincipal:string, Region:string) [
    "user1@contoso.com", "North America",
    "user1@contoso.com", "Europe",
    "user2@contoso.com", "Asia Pacific"
]

// Apply RLS using lookup table
.alter table RegionalSales policy row_level_security enable ```
RegionalSales
| where Region in (
    (UserRegionAccess | where UserPrincipal == current_principal_details()['UserPrincipalName'] | project Region)
)
```
```

### 2.3 RLS with Microsoft Entra Group Check

```kusto
// Allow full access for members of admin group, filter for others
.alter table SensitiveData policy row_level_security enable ```
SensitiveData
| where current_principal_is_member_of('aadgroup=DataAdmins;contoso.onmicrosoft.com')
    or DepartmentId == toint(current_principal_details()['DepartmentId'])
```
```

### 2.4 Testing RLS Before Enabling

```kusto
// Test RLS query without enabling it
set query_force_row_level_security;
SensitiveData
| take 100
```

### 2.5 View and Manage RLS Policies

```kusto
// Show RLS policy for a table
.show table CustomerData policy row_level_security

// Disable RLS (keeps policy definition)
.alter table CustomerData policy row_level_security disable
"CustomerData | where SalesRepEmail == current_principal_details()['UserPrincipalName']"

// Delete RLS policy entirely
.delete table CustomerData policy row_level_security
```

---

## 3. Restricted View Access Examples

### 3.1 Enable Restricted View Access

```kusto
// Enable restricted view access on sensitive table
.alter table FinancialData policy restricted_view_access true

// Enable on multiple tables
.alter table EmployeeSalaries policy restricted_view_access true
.alter table ExecutiveCompensation policy restricted_view_access true
```

### 3.2 Grant Unrestricted Viewer Access

```kusto
// Add unrestricted viewer role to allow access to restricted tables
.add database HRDatabase unrestrictedviewers ('aaduser=hr_analyst@contoso.com')

// Add unrestricted viewer role to a group
.add database HRDatabase unrestrictedviewers ('aadgroup=HRManagers;contoso.onmicrosoft.com')
```

### 3.3 Check and Disable Restricted View Access

```kusto
// Check policy status
.show table FinancialData policy restricted_view_access

// Disable restricted view access
.alter table FinancialData policy restricted_view_access false
```

---

## 4. Authentication Examples

### 4.1 C# - User Interactive Authentication

```csharp
using Kusto.Data;
using Kusto.Data.Net.Client;

// Interactive user login
var connectionString = new KustoConnectionStringBuilder(
    "https://mycluster.westus.kusto.windows.net",
    "MyDatabase")
    .WithAadUserPromptAuthentication();

using var client = KustoClientFactory.CreateCslQueryProvider(connectionString);
var results = client.ExecuteQuery("MyTable | take 10");
```

### 4.2 C# - Application with Client Secret

```csharp
using Kusto.Data;
using Kusto.Data.Net.Client;

var connectionString = new KustoConnectionStringBuilder(
    "https://mycluster.westus.kusto.windows.net",
    "MyDatabase")
    .WithAadApplicationKeyAuthentication(
        applicationClientId: "12345678-1234-1234-1234-123456789012",
        applicationKey: "your-client-secret",
        authority: "contoso.onmicrosoft.com");

using var client = KustoClientFactory.CreateCslQueryProvider(connectionString);
```

### 4.3 C# - Application with Certificate

```csharp
using Kusto.Data;
using Kusto.Data.Net.Client;
using System.Security.Cryptography.X509Certificates;

var certificate = new X509Certificate2("path/to/certificate.pfx", "password");

var connectionString = new KustoConnectionStringBuilder(
    "https://mycluster.westus.kusto.windows.net",
    "MyDatabase")
    .WithAadApplicationCertificateAuthentication(
        applicationClientId: "12345678-1234-1234-1234-123456789012",
        certificate: certificate,
        authority: "contoso.onmicrosoft.com");

using var client = KustoClientFactory.CreateCslQueryProvider(connectionString);
```

### 4.4 C# - Managed Identity Authentication

```csharp
using Kusto.Data;
using Kusto.Data.Net.Client;

// System-assigned managed identity
var connectionString = new KustoConnectionStringBuilder(
    "https://mycluster.westus.kusto.windows.net",
    "MyDatabase")
    .WithAadManagedIdentity();

// User-assigned managed identity
var connectionStringUserAssigned = new KustoConnectionStringBuilder(
    "https://mycluster.westus.kusto.windows.net",
    "MyDatabase")
    .WithAadManagedIdentity("client-id-of-user-assigned-identity");

using var client = KustoClientFactory.CreateCslQueryProvider(connectionString);
```

### 4.5 Python - Various Authentication Methods

```python
from azure.kusto.data import KustoClient, KustoConnectionStringBuilder

cluster = "https://mycluster.westus.kusto.windows.net"
database = "MyDatabase"

# Interactive user authentication
kcsb_user = KustoConnectionStringBuilder.with_aad_user_prompt_authentication(cluster, database)

# Application with client secret
kcsb_secret = KustoConnectionStringBuilder.with_aad_application_key_authentication(
    cluster,
    database,
    aad_app_id="12345678-1234-1234-1234-123456789012",
    app_key="your-client-secret",
    authority_id="contoso.onmicrosoft.com"
)

# Managed identity (system-assigned)
kcsb_msi = KustoConnectionStringBuilder.with_aad_managed_service_identity_authentication(cluster, database)

# Managed identity (user-assigned)
kcsb_msi_user = KustoConnectionStringBuilder.with_aad_managed_service_identity_authentication(
    cluster,
    database,
    client_id="user-assigned-mi-client-id"
)

# Create client and execute query
client = KustoClient(kcsb_secret)
response = client.execute(database, "MyTable | take 10")
```

### 4.6 Java - Service Principal Authentication

```java
import com.microsoft.azure.kusto.data.*;
import com.microsoft.azure.kusto.data.auth.*;

String cluster = "https://mycluster.westus.kusto.windows.net";
String database = "MyDatabase";

// Application with client secret
ConnectionStringBuilder csb = ConnectionStringBuilder
    .createWithAadApplicationCredentials(
        cluster,
        "12345678-1234-1234-1234-123456789012", // App ID
        "your-client-secret",                    // Secret
        "contoso.onmicrosoft.com"               // Tenant
    );

Client client = ClientFactory.createClient(csb);
KustoOperationResult result = client.execute(database, "MyTable | take 10");
```

---

## 5. PowerShell Examples

### 5.1 Query with Service Principal

```powershell
# Load Kusto assemblies
[System.Reflection.Assembly]::LoadFrom("C:\KustoTools\Kusto.Data.dll")

# Build connection string with app authentication
$cluster = "https://mycluster.westus.kusto.windows.net"
$database = "MyDatabase"
$appId = "12345678-1234-1234-1234-123456789012"
$appKey = "your-client-secret"
$tenant = "contoso.onmicrosoft.com"

$kcsb = [Kusto.Data.KustoConnectionStringBuilder]::new($cluster, $database)
$kcsb = $kcsb.WithAadApplicationKeyAuthentication($appId, $appKey, $tenant)

# Create client and execute query
$queryProvider = [Kusto.Data.Net.Client.KustoClientFactory]::CreateCslQueryProvider($kcsb)
$reader = $queryProvider.ExecuteQuery("MyTable | take 10")

# Process results
$dataTable = New-Object System.Data.DataTable
$dataTable.Load($reader)
$dataTable | Format-Table
```

### 5.2 Manage Principals with PowerShell

```powershell
# Connect and add a principal
$adminProvider = [Kusto.Data.Net.Client.KustoClientFactory]::CreateCslAdminProvider($kcsb)

# Add database viewer
$command = ".add database MyDatabase viewers ('aaduser=analyst@contoso.com')"
$result = $adminProvider.ExecuteControlCommand($command)

# List all principals
$command = ".show database MyDatabase principals"
$reader = $adminProvider.ExecuteControlCommand($command)
$principals = New-Object System.Data.DataTable
$principals.Load($reader)
$principals | Format-Table
```

---

## 6. Network Security Examples

### 6.1 Azure CLI - Configure Firewall Rules

```bash
# Enable firewall and allow specific IPs
az kusto cluster update \
    --name mycluster \
    --resource-group myresourcegroup \
    --allowed-ip-range-list "192.168.1.0/24" "10.0.0.1" "PowerBI" "LogicApps"

# Disable public access (private endpoint only)
az kusto cluster update \
    --name mycluster \
    --resource-group myresourcegroup \
    --public-network-access Disabled
```

### 6.2 PowerShell - Configure Network Access

```powershell
# Allow specific IP ranges and service tags
Update-AzKustoCluster `
    -ResourceGroupName "myresourcegroup" `
    -Name "mycluster" `
    -PublicNetworkAccess "Enabled" `
    -AllowedIpRangeList @("192.168.1.0/24", "10.0.0.1", "PowerBI")
```

### 6.3 ARM Template - Private Endpoint Configuration

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "clusterName": { "type": "string" },
        "vnetName": { "type": "string" },
        "subnetName": { "type": "string" }
    },
    "resources": [
        {
            "type": "Microsoft.Network/privateEndpoints",
            "apiVersion": "2021-05-01",
            "name": "[concat(parameters('clusterName'), '-pe')]",
            "location": "[resourceGroup().location]",
            "properties": {
                "privateLinkServiceConnections": [
                    {
                        "name": "[concat(parameters('clusterName'), '-plsc')]",
                        "properties": {
                            "privateLinkServiceId": "[resourceId('Microsoft.Kusto/clusters', parameters('clusterName'))]",
                            "groupIds": ["cluster"]
                        }
                    }
                ],
                "subnet": {
                    "id": "[resourceId('Microsoft.Network/virtualNetworks/subnets', parameters('vnetName'), parameters('subnetName'))]"
                }
            }
        }
    ]
}
```

---

## 7. Customer-Managed Keys (CMK) Examples

### 7.1 Azure CLI - Enable CMK

```bash
# Enable customer-managed key encryption
az kusto cluster update \
    --name mycluster \
    --resource-group myresourcegroup \
    --key-vault-properties key-name="adx-cmk" \
                          key-vault-uri="https://mykeyvault.vault.azure.net/" \
                          key-version="abc123"
```

### 7.2 PowerShell - Configure CMK with Managed Identity

```powershell
# Create user-assigned managed identity
$identity = New-AzUserAssignedIdentity `
    -ResourceGroupName "myresourcegroup" `
    -Name "adx-cmk-identity"

# Assign Key Vault permissions to the identity
Set-AzKeyVaultAccessPolicy `
    -VaultName "mykeyvault" `
    -ObjectId $identity.PrincipalId `
    -PermissionsToKeys get, wrapKey, unwrapKey

# Configure cluster with CMK
Update-AzKustoCluster `
    -ResourceGroupName "myresourcegroup" `
    -Name "mycluster" `
    -KeyVaultPropertyKeyName "adx-cmk" `
    -KeyVaultPropertyKeyVaultUri "https://mykeyvault.vault.azure.net/" `
    -KeyVaultPropertyKeyVersion "abc123" `
    -KeyVaultPropertyUserIdentity $identity.Id
```

---

## 8. Auditing and Monitoring Examples

### 8.1 Query Security-Related Operations from Audit Logs

```kusto
// Find all principal changes in the last 7 days
.show commands
| where StartedOn > ago(7d)
| where CommandText has_any (".add", ".drop", ".set")
    and CommandText has_any ("admins", "users", "viewers", "ingestors")
| project StartedOn, User, CommandText, State
| order by StartedOn desc
```

### 8.2 Monitor Access Patterns

```kusto
// Track queries by user over time
.show queries
| where StartedOn > ago(1d)
| summarize QueryCount = count(),
            AvgDuration = avg(Duration),
            Databases = make_set(Database)
    by User
| order by QueryCount desc
```

### 8.3 Find Failed Authentication Attempts

```kusto
// Query diagnostic logs for auth failures (if sent to ADX)
AzureDiagnostics
| where ResourceType == "CLUSTERS"
| where Category == "FailedIngestion" or Category == "Command"
| where ResultType != "Success"
| where Message has "auth" or Message has "unauthorized"
| project TimeGenerated, OperationName, ResultType, Message
| order by TimeGenerated desc
```

---

## 9. Complete Security Setup Example

### 9.1 Comprehensive Database Security Configuration

```kusto
// Step 1: Set up admin team
.add database ProductionDB admins (
    'aaduser=lead_admin@contoso.com',
    'aadgroup=DatabaseAdmins;contoso.onmicrosoft.com'
)

// Step 2: Set up viewers for analysts
.add database ProductionDB viewers (
    'aadgroup=DataAnalysts;contoso.onmicrosoft.com'
)

// Step 3: Set up ingestors for data pipelines
.add database ProductionDB ingestors (
    'aadapp=pipeline-app-id;contoso.onmicrosoft.com',
    'aadapp=managed-identity-object-id;contoso.onmicrosoft.com'
)

// Step 4: Enable restricted view access on sensitive tables
.alter table EmployeeData policy restricted_view_access true
.alter table FinancialData policy restricted_view_access true

// Step 5: Grant unrestricted access to HR team
.add database ProductionDB unrestrictedviewers (
    'aadgroup=HRTeam;contoso.onmicrosoft.com'
)

// Step 6: Configure RLS for regional data
.alter table RegionalSales policy row_level_security enable ```
RegionalSales
| where current_principal_is_member_of('aadgroup=GlobalAnalysts;contoso.onmicrosoft.com')
    or Region in (
        (UserRegionMapping
         | where UserEmail == current_principal_details()['UserPrincipalName']
         | project Region)
    )
```

// Step 7: Verify configuration
.show database ProductionDB principals
.show table EmployeeData policy restricted_view_access
.show table RegionalSales policy row_level_security
```

---

*Generated: 2026-02-13 | Source: Azure Data Explorer Documentation Analysis | Phase 3 - Security Deep Investigation*
