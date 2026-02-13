# Security and Access Control Reference

## Overview

Azure Data Explorer (ADX) provides comprehensive security features across four key areas: authentication, authorization, network security, and data protection. This reference documents the security model, available configurations, and implementation details.

---

## 1. Authentication

Authentication validates the identity of security principals making requests to ADX resources.

### 1.1 Microsoft Entra ID (Azure AD)

Microsoft Entra ID is the recommended identity provider for ADX authentication.

#### Supported Authentication Scenarios

| Scenario | Description | Use Case |
|----------|-------------|----------|
| **User Authentication** | Interactive sign-in via browser | Human users querying data |
| **Application Authentication** | Non-interactive sign-in with credentials | Automated services, background jobs |
| **On-Behalf-Of (OBO)** | Token exchange for downstream services | Web APIs calling ADX |
| **Single Page Application (SPA)** | Client-side authentication | Browser-based applications |

#### User Authentication Methods

- Interactive sign-in through the user interface
- Microsoft Entra token issued for Azure Data Explorer
- Token exchange via OBO authentication

#### Application Authentication Methods

- **Managed Identity**: Azure-managed credentials (recommended for Azure services)
- **X.509v2 Certificate**: Certificate-based authentication
- **Application ID + Key**: Client ID and secret (like username/password)
- **Pre-obtained Token**: Previously acquired Microsoft Entra token

### 1.2 Managed Identities

Managed identities eliminate the need for credential management in code.

#### Types of Managed Identities

| Type | Description | Lifecycle |
|------|-------------|-----------|
| **System-Assigned** | Tied to the cluster resource | Deleted when cluster is deleted |
| **User-Assigned** | Standalone Azure resource | Independent of cluster lifecycle |

#### Key Properties

- **PrincipalId**: Unique identifier for the identity in Microsoft Entra ID
- **TenantId**: Microsoft Entra tenant the identity belongs to
- **ClientId**: Application ID for runtime use (user-assigned only)

#### Managed Identity Use Cases

- Accessing Azure Key Vault for customer-managed keys
- Connecting to external tables in Azure Storage
- Authenticating continuous export to storage accounts
- Ingestion from secured Event Hubs

### 1.3 Service Principals

Service principals are Microsoft Entra applications used for non-interactive authentication.

#### Creating a Service Principal

1. Register an application in Microsoft Entra ID
2. Generate client credentials (secret or certificate)
3. Grant the service principal access to ADX resources

#### Required Information

| Property | Purpose |
|----------|---------|
| Application (Client) ID | Identifies the application |
| Directory (Tenant) ID | Identifies the Microsoft Entra tenant |
| Client Secret or Certificate | Proves the application's identity |

---

## 2. Authorization (RBAC)

ADX uses role-based access control (RBAC) where principals are granted permissions through assigned security roles.

### 2.1 Role Hierarchy

Roles can be defined at different scopes:

```
Cluster
  └── Database
        ├── Table
        ├── External Table
        ├── Materialized View
        └── Function
```

Roles defined at a higher level (cluster, database) apply to all entities within that scope.

### 2.2 Cluster-Level Roles

| Role | Permissions |
|------|-------------|
| **AllDatabasesAdmin** | Full permission to all databases; can show/alter cluster policies |
| **AllDatabasesViewer** | Read all data and metadata of any database |
| **AllDatabasesMonitor** | Execute `.show` commands in any database context |

> Note: Cluster-level roles are configured via Azure portal, not management commands.

### 2.3 Database-Level Roles

| Role | Permissions |
|------|-------------|
| **Admin** | Full permission; includes all lower-level permissions |
| **User** | Read data/metadata; create tables and functions |
| **Viewer** | Read data/metadata (except RestrictedViewAccess tables) |
| **UnrestrictedViewer** | Read all data including RestrictedViewAccess tables |
| **Ingestor** | Ingest data without query access |
| **Monitor** | Execute `.show` commands on database and child entities |

### 2.4 Table-Level Roles

| Role | Permissions | Dependencies |
|------|-------------|--------------|
| **Admin** | Full permission on the table | Requires Database User |
| **Ingestor** | Ingest data without query access | Requires Database User or Ingestor |

### 2.5 Other Entity Roles

| Entity | Role | Permissions |
|--------|------|-------------|
| External Table | Admin | Full permission on the external table |
| Materialized View | Admin | Alter, delete, grant admin permissions |
| Function | Admin | Alter, delete, grant admin permissions |
| Graph | GraphAdmin | Full permission on the graph model |

### 2.6 Referencing Security Principals

#### Microsoft Entra ID Principals

| Entity Type | Syntax |
|-------------|--------|
| User (implicit tenant) | `aaduser=user@domain.com` |
| User (explicit tenant) | `aaduser=user@domain.com;TenantId` |
| Group | `aadgroup=GroupName;TenantId` |
| Application | `aadapp=AppId;TenantId` |
| Managed Identity | `aadapp=ObjectId;TenantId` |

#### Microsoft Account (MSA)

```
msauser=user@live.com
```

### 2.7 Row-Level Security (RLS)

RLS provides row-level filtering based on user identity or group membership.

#### How RLS Works

1. A KQL query is defined as the RLS policy
2. All access to the table is replaced by this query
3. The query can filter rows based on the current principal
4. The restriction applies to ALL users, including admins

#### Key Functions for RLS

| Function | Purpose |
|----------|---------|
| `current_principal()` | Returns the principal making the request |
| `current_principal_details()` | Returns detailed principal information (UPN, ObjectId, etc.) |
| `current_principal_is_member_of()` | Checks group membership |

#### RLS Limitations

- Cannot be configured on external tables
- Cannot reference other RLS-enabled tables
- Cannot reference tables in other databases
- Cannot coexist with RestrictedViewAccess policy on the same table

### 2.8 Restricted View Access Policy

Controls which principals can view a table's data.

| Policy State | Viewer Requirements |
|--------------|---------------------|
| Disabled (default) | Standard Viewer role sufficient |
| Enabled | Requires UnrestrictedViewer role |

> The UnrestrictedViewer role operates at the database level and grants access to ALL tables with RestrictedViewAccess enabled.

---

## 3. Network Security

### 3.1 Private Endpoints

Private endpoints connect ADX clusters to virtual networks using private IP addresses.

#### Architecture

- Uses Azure Private Link
- Traffic traverses Microsoft backbone network
- Eliminates public internet exposure
- Requires private DNS zone integration

#### Required DNS Zones

| Zone | Purpose |
|------|---------|
| `privatelink.<region>.kusto.windows.net` | Engine and data management endpoints |
| `privatelink.blob.core.windows.net` | Blob storage for ingestion |
| `privatelink.queue.core.windows.net` | Queue storage for ingestion |
| `privatelink.table.core.windows.net` | Table storage for ingestion |

#### Private Endpoint Features

| Feature | Implementation |
|---------|----------------|
| Inbound IP filtering | Manage public access settings |
| Transitive access to services | Managed private endpoints |
| Outbound access control | Callout policies / AllowedFQDNList |

### 3.2 Public Access Management

Three options for public network access:

| Option | Description |
|--------|-------------|
| **Enabled from all networks** | Default; allows all public access |
| **Enabled from selected IP addresses** | Firewall allow list with IP/CIDR/service tags |
| **Disabled** | Requires private endpoint for all connections |

#### Firewall Configuration

```json
"properties": {
    "publicNetworkAccess": "Enabled",
    "allowedIpRangeList": [
        "192.168.1.10",
        "192.168.2.0/24",
        "PowerBI",
        "LogicApps"
    ]
}
```

### 3.3 Service Tags

Pre-defined groups of IP address prefixes for Azure services:

| Service Tag | Use Case |
|-------------|----------|
| `PowerBI` | Allow Power BI service connections |
| `LogicApps` | Allow Azure Logic Apps connections |
| `Storage.WestUS` | Allow Azure Storage (regional) |

---

## 4. Data Protection

### 4.1 Encryption at Rest

All ADX data is encrypted at rest using Azure Storage encryption.

#### Encryption Layers

| Layer | Key Type | Description |
|-------|----------|-------------|
| Service Level | Microsoft-managed or Customer-managed | Primary encryption layer |
| Infrastructure Level | Microsoft-managed | Optional double encryption |

#### Key Requirements for CMK

- Azure Key Vault with Soft Delete enabled
- Azure Key Vault with Do Not Purge enabled
- RSA or RSA-HSM keys (2048, 3072, or 4096 bits)
- Key vault and cluster in the same region

### 4.2 Customer-Managed Keys (CMK)

CMK provides additional control over encryption keys.

#### Required Key Vault Permissions

| Permission | Purpose |
|------------|---------|
| Get | Retrieve key metadata |
| Wrap Key | Encrypt the data encryption key |
| Unwrap Key | Decrypt the data encryption key |

#### CMK Configuration Properties

```json
{
    "keyVaultProperties": {
        "keyVaultUri": "https://<keyVaultName>.vault.azure.net/",
        "keyName": "<keyName>",
        "keyVersion": "<keyVersion>",
        "userIdentity": "<managedIdentityResourceId>"
    }
}
```

### 4.3 Disk Encryption

Virtual machine cache storage encryption for hot cache data.

| Component | Encryption |
|-----------|------------|
| OS Disk | Microsoft-managed keys |
| Data Volumes | Microsoft-managed keys |
| Hot Cache | Encrypted when disk encryption enabled |

### 4.4 Double Encryption

Protects against compromise of a single encryption algorithm or key.

- Service-level encryption (Microsoft or customer-managed)
- Infrastructure-level encryption (always Microsoft-managed)
- Uses different algorithms and keys at each layer

---

## 5. Security Management Commands

### 5.1 Principal Management

```kusto
// Show all principals on a database
.show database <DatabaseName> principals

// Show your roles on a database
.show database <DatabaseName> principal roles

// Add principals to a role
.add database <DatabaseName> <Role> ('aaduser=user@domain.com')

// Remove principals from a role
.drop database <DatabaseName> <Role> ('aaduser=user@domain.com')

// Replace all principals in a role
.set database <DatabaseName> <Role> ('aaduser=user1@domain.com', 'aaduser=user2@domain.com')

// Remove all principals from a role
.set database <DatabaseName> <Role> none
```

### 5.2 RLS Policy Management

```kusto
// Enable RLS policy
.alter table <TableName> policy row_level_security enable "<Query>"

// Disable RLS policy
.alter table <TableName> policy row_level_security disable "<Query>"

// Show RLS policy
.show table <TableName> policy row_level_security

// Test RLS without enabling
set query_force_row_level_security;
<YourQuery>
```

### 5.3 Restricted View Access Policy

```kusto
// Enable restricted view access
.alter table <TableName> policy restricted_view_access true

// Disable restricted view access
.alter table <TableName> policy restricted_view_access false

// Show policy
.show table <TableName> policy restricted_view_access
```

---

## 6. Related Documentation Files

| Topic | Documentation Path |
|-------|-------------------|
| Access Control Overview | `kusto/access-control/index.md` |
| Role-Based Access Control | `kusto/access-control/role-based-access-control.md` |
| Security Roles | `kusto/management/security-roles.md` |
| Row Level Security | `kusto/management/row-level-security-policy.md` |
| Restricted View Access | `kusto/management/restricted-view-access-policy.md` |
| Managed Identities | `configure-managed-identities-cluster.md` |
| Customer-Managed Keys | `customer-managed-keys.md` |
| Network Security | `security-network-overview.md` |
| Private Endpoints | `security-network-private-endpoint.md` |

---

*Generated: 2026-02-13 | Source: Azure Data Explorer Documentation Analysis | Phase 3 - Security Deep Investigation*
