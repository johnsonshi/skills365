# Security Configuration Examples

Ready-to-use examples for Azure Data Explorer security and access control.

---

## 1. Principal Management

### Add Database Principals

```kusto
// Add user as viewer
.add database MyDB viewers ('aaduser=analyst@contoso.com')

// Add security group as viewers
.add database MyDB viewers ('aadgroup=DataAnalysts;contoso.onmicrosoft.com')

// Add service principal as ingestor
.add database MyDB ingestors ('aadapp=12345678-1234-1234-1234-123456789012;contoso.onmicrosoft.com')

// Add managed identity as ingestor
.add database MyDB ingestors ('aadapp=87654321-4321-4321-4321-210987654321;contoso.onmicrosoft.com')

// Add database admin
.add database MyDB admins ('aaduser=admin@contoso.com')
```

### Add Table Principals

```kusto
// Add table admin
.add table SalesData admins ('aaduser=sales_lead@contoso.com')

// Add table ingestor for pipeline
.add table IoTEvents ingestors ('aadapp=12345678-1234-1234-1234-123456789012;contoso.onmicrosoft.com')
```

### Remove and Replace Principals

```kusto
// Remove user
.drop database MyDB viewers ('aaduser=former_employee@contoso.com')

// Replace all admins
.set database MyDB admins ('aaduser=admin1@contoso.com', 'aaduser=admin2@contoso.com')

// Clear all ingestors
.set database MyDB ingestors none
```

### View Principals

```kusto
.show database MyDB principals
.show database MyDB principal roles
.show table SalesData principals
```

---

## 2. Row-Level Security

### Filter by User Email

```kusto
.alter table CustomerData policy row_level_security enable
"CustomerData | where SalesRepEmail == current_principal_details()['UserPrincipalName']"
```

### Filter by Group Membership

```kusto
.alter table SensitiveData policy row_level_security enable ```
SensitiveData
| where current_principal_is_member_of('aadgroup=DataAdmins;contoso.onmicrosoft.com')
    or DepartmentId == toint(current_principal_details()['DepartmentId'])
```
```

### Manager Override Pattern

```kusto
.alter table TeamData policy row_level_security enable ```
let IsManager = current_principal_is_member_of('aadgroup=Managers;contoso.onmicrosoft.com');
TeamData
| where IsManager or OwnerEmail == current_principal_details()['UserPrincipalName']
```
```

### Multi-Region Access

```kusto
.alter table RegionalSales policy row_level_security enable ```
let IsGlobal = current_principal_is_member_of('aadgroup=GlobalAnalysts;contoso.onmicrosoft.com');
let IsUS = current_principal_is_member_of('aadgroup=USAnalysts;contoso.onmicrosoft.com');
let IsEU = current_principal_is_member_of('aadgroup=EUAnalysts;contoso.onmicrosoft.com');
RegionalSales
| where IsGlobal
    or (IsUS and Region == "US")
    or (IsEU and Region == "EU")
```
```

### Data Masking

```kusto
.alter table Customers policy row_level_security enable ```
let IsPrivileged = current_principal_is_member_of('aadgroup=PrivilegedUsers;contoso.onmicrosoft.com');
Customers
| extend
    Email = iff(IsPrivileged, Email, strcat(substring(Email, 0, 3), "***@***.***")),
    SSN = iff(IsPrivileged, SSN, strcat("***-**-", substring(SSN, 7, 4))),
    Phone = iff(IsPrivileged, Phone, strcat(substring(Phone, 0, 3), "-***-****"))
```
```

### Test RLS Before Enabling

```kusto
set query_force_row_level_security;
SensitiveData | take 100
```

### Manage RLS Policies

```kusto
// Show policy
.show table CustomerData policy row_level_security

// Disable (keeps definition)
.alter table CustomerData policy row_level_security disable
"CustomerData | where SalesRepEmail == current_principal_details()['UserPrincipalName']"

// Delete policy
.delete table CustomerData policy row_level_security
```

---

## 3. Restricted View Access

```kusto
// Enable on sensitive tables
.alter table FinancialData policy restricted_view_access true
.alter table EmployeeSalaries policy restricted_view_access true

// Grant access via unrestricted viewer role
.add database HRDatabase unrestrictedviewers ('aaduser=hr_analyst@contoso.com')
.add database HRDatabase unrestrictedviewers ('aadgroup=HRManagers;contoso.onmicrosoft.com')

// Check status
.show table FinancialData policy restricted_view_access

// Disable
.alter table FinancialData policy restricted_view_access false
```

---

## 4. Authentication Code Examples

### C# - Managed Identity

```csharp
var connectionString = new KustoConnectionStringBuilder(
    "https://mycluster.westus.kusto.windows.net",
    "MyDatabase")
    .WithAadManagedIdentity();

using var client = KustoClientFactory.CreateCslQueryProvider(connectionString);
```

### C# - Service Principal with Secret

```csharp
var connectionString = new KustoConnectionStringBuilder(
    "https://mycluster.westus.kusto.windows.net",
    "MyDatabase")
    .WithAadApplicationKeyAuthentication(
        applicationClientId: "12345678-1234-1234-1234-123456789012",
        applicationKey: "your-client-secret",
        authority: "contoso.onmicrosoft.com");

using var client = KustoClientFactory.CreateCslQueryProvider(connectionString);
```

### C# - Certificate Authentication

```csharp
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

### Python - Various Methods

```python
from azure.kusto.data import KustoClient, KustoConnectionStringBuilder

cluster = "https://mycluster.westus.kusto.windows.net"

# Managed identity (system-assigned)
kcsb = KustoConnectionStringBuilder.with_aad_managed_service_identity_authentication(cluster)

# Managed identity (user-assigned)
kcsb = KustoConnectionStringBuilder.with_aad_managed_service_identity_authentication(
    cluster, client_id="user-assigned-mi-client-id")

# Service principal with secret
kcsb = KustoConnectionStringBuilder.with_aad_application_key_authentication(
    cluster,
    aad_app_id="12345678-1234-1234-1234-123456789012",
    app_key="your-client-secret",
    authority_id="contoso.onmicrosoft.com")

client = KustoClient(kcsb)
response = client.execute("MyDatabase", "MyTable | take 10")
```

---

## 5. Network Security

### Azure CLI - Firewall Configuration

```bash
# Allow specific IPs and service tags
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

### PowerShell - Network Configuration

```powershell
Update-AzKustoCluster `
    -ResourceGroupName "myresourcegroup" `
    -Name "mycluster" `
    -PublicNetworkAccess "Enabled" `
    -AllowedIpRangeList @("192.168.1.0/24", "10.0.0.1", "PowerBI")
```

### ARM Template - Private Endpoint

```json
{
    "type": "Microsoft.Network/privateEndpoints",
    "apiVersion": "2021-05-01",
    "name": "[concat(parameters('clusterName'), '-pe')]",
    "location": "[resourceGroup().location]",
    "properties": {
        "privateLinkServiceConnections": [{
            "name": "[concat(parameters('clusterName'), '-plsc')]",
            "properties": {
                "privateLinkServiceId": "[resourceId('Microsoft.Kusto/clusters', parameters('clusterName'))]",
                "groupIds": ["cluster"]
            }
        }],
        "subnet": {
            "id": "[resourceId('Microsoft.Network/virtualNetworks/subnets', parameters('vnetName'), parameters('subnetName'))]"
        }
    }
}
```

---

## 6. Customer-Managed Keys

### Azure CLI

```bash
az kusto cluster update \
    --name mycluster \
    --resource-group myresourcegroup \
    --key-vault-properties key-name="adx-cmk" \
                          key-vault-uri="https://mykeyvault.vault.azure.net/" \
                          key-version="abc123"
```

### PowerShell with Managed Identity

```powershell
# Create user-assigned managed identity
$identity = New-AzUserAssignedIdentity `
    -ResourceGroupName "myresourcegroup" `
    -Name "adx-cmk-identity"

# Grant Key Vault permissions
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

## 7. Auditing Queries

### Security Command History

```kusto
.show commands
| where StartedOn > ago(7d)
| where CommandText has_any (".add", ".drop", ".set")
    and CommandText has_any ("admins", "users", "viewers", "ingestors")
| project StartedOn, User, CommandText, State
| order by StartedOn desc
```

### Query Activity by User

```kusto
.show queries
| where StartedOn > ago(1d)
| summarize QueryCount = count(), AvgDuration = avg(Duration), Databases = make_set(Database)
    by User
| order by QueryCount desc
```

---

## 8. Complete Security Setup

```kusto
// Step 1: Configure admin team
.add database ProductionDB admins (
    'aaduser=lead_admin@contoso.com',
    'aadgroup=DatabaseAdmins;contoso.onmicrosoft.com'
)

// Step 2: Configure analyst access
.add database ProductionDB viewers (
    'aadgroup=DataAnalysts;contoso.onmicrosoft.com'
)

// Step 3: Configure data pipelines
.add database ProductionDB ingestors (
    'aadapp=pipeline-app-id;contoso.onmicrosoft.com'
)

// Step 4: Protect sensitive tables
.alter table EmployeeData policy restricted_view_access true
.alter table FinancialData policy restricted_view_access true

// Step 5: Grant privileged access
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
