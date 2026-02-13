# Security and Access Control Reference

Quick reference for Azure Data Explorer security features: authentication, authorization, network security, and data protection.

---

## Authentication

### Microsoft Entra ID Methods

| Method | Use Case |
|--------|----------|
| User Interactive | Human users via browser sign-in |
| Managed Identity | Azure services (recommended - no secrets) |
| Service Principal + Secret | Automated services, background jobs |
| Service Principal + Certificate | Higher security automation |
| On-Behalf-Of (OBO) | Web APIs calling ADX |

### Managed Identity Types

| Type | Lifecycle | Use When |
|------|-----------|----------|
| System-Assigned | Tied to cluster | Single cluster access |
| User-Assigned | Independent resource | Multi-resource or cross-cluster |

### Principal Reference Syntax

```
aaduser=user@domain.com                    # User (implicit tenant)
aaduser=user@domain.com;TenantId           # User (explicit tenant)
aadgroup=GroupName;TenantId                # Security group
aadapp=AppId;TenantId                      # Service principal
aadapp=ObjectId;TenantId                   # Managed identity
```

---

## Authorization (RBAC)

### Role Hierarchy

```
Cluster
  └── Database
        ├── Table
        ├── External Table
        ├── Materialized View
        └── Function
```

### Database Roles

| Role | Permissions |
|------|-------------|
| Admin | Full control; manage principals, policies |
| User | Read data; create tables/functions |
| Viewer | Read data (except RestrictedViewAccess tables) |
| UnrestrictedViewer | Read all data including restricted tables |
| Ingestor | Ingest data only (no query access) |
| Monitor | Execute `.show` commands |

### Table Roles

| Role | Permissions | Requires |
|------|-------------|----------|
| Admin | Full control on table | Database User |
| Ingestor | Ingest data to table | Database User or Ingestor |

### Principal Management Commands

```kusto
// View principals
.show database <DB> principals
.show database <DB> principal roles

// Add principals
.add database <DB> <Role> ('aaduser=user@domain.com')
.add database <DB> <Role> ('aadgroup=Group;TenantId')

// Remove principals
.drop database <DB> <Role> ('aaduser=user@domain.com')

// Replace all principals in role
.set database <DB> <Role> ('aaduser=user1@domain.com', 'aaduser=user2@domain.com')

// Clear all principals from role
.set database <DB> <Role> none
```

---

## Row-Level Security (RLS)

### Key Functions

| Function | Returns |
|----------|---------|
| `current_principal()` | Principal making the request |
| `current_principal_details()` | Dict with UPN, ObjectId, etc. |
| `current_principal_is_member_of()` | Boolean group membership check |

### RLS Commands

```kusto
// Enable RLS
.alter table <Table> policy row_level_security enable "<Query>"

// Disable RLS (keeps policy)
.alter table <Table> policy row_level_security disable "<Query>"

// Show policy
.show table <Table> policy row_level_security

// Delete policy
.delete table <Table> policy row_level_security

// Test without enabling
set query_force_row_level_security;
<YourQuery>
```

### RLS Limitations

- Cannot reference external tables
- Cannot reference other RLS-enabled tables
- Cannot reference cross-database tables
- Incompatible with RestrictedViewAccess on same table
- Applies to ALL users including admins

---

## Restricted View Access

Requires UnrestrictedViewer role to access table data.

```kusto
// Enable
.alter table <Table> policy restricted_view_access true

// Disable
.alter table <Table> policy restricted_view_access false

// Check status
.show table <Table> policy restricted_view_access
```

---

## Network Security

### Public Access Options

| Setting | Description |
|---------|-------------|
| Enabled from all networks | Default; open access |
| Enabled from selected IPs | Firewall allow list |
| Disabled | Private endpoint only |

### Firewall Configuration

```json
{
    "publicNetworkAccess": "Enabled",
    "allowedIpRangeList": ["192.168.1.0/24", "10.0.0.1", "PowerBI", "LogicApps"]
}
```

### Service Tags

| Tag | Service |
|-----|---------|
| PowerBI | Power BI service |
| LogicApps | Azure Logic Apps |
| Storage.{Region} | Azure Storage (regional) |

### Private Endpoint DNS Zones

| Zone | Purpose |
|------|---------|
| `privatelink.<region>.kusto.windows.net` | Engine endpoints |
| `privatelink.blob.core.windows.net` | Blob ingestion |
| `privatelink.queue.core.windows.net` | Queue ingestion |
| `privatelink.table.core.windows.net` | Table ingestion |

---

## Data Protection

### Encryption Layers

| Layer | Key Management | Description |
|-------|----------------|-------------|
| Service Level | Microsoft or Customer | Primary encryption |
| Infrastructure Level | Microsoft | Optional double encryption |
| Disk Encryption | Microsoft | VM cache encryption |

### Customer-Managed Keys (CMK)

**Key Vault Requirements:**
- Soft Delete enabled (required)
- Purge Protection enabled (required)
- RSA/RSA-HSM keys: 2048, 3072, or 4096 bits
- Same region as cluster

**Required Permissions:**
- Get
- Wrap Key
- Unwrap Key

**Configuration:**
```json
{
    "keyVaultProperties": {
        "keyVaultUri": "https://<vault>.vault.azure.net/",
        "keyName": "<keyName>",
        "keyVersion": "<version>",
        "userIdentity": "<managedIdentityResourceId>"
    }
}
```

---

## Quick Reference: Common Operations

| Task | Command/Action |
|------|----------------|
| Grant read access | `.add database <DB> viewers ('aadgroup=Group;Tenant')` |
| Grant ingestion access | `.add database <DB> ingestors ('aadapp=AppId;Tenant')` |
| Restrict sensitive table | `.alter table <T> policy restricted_view_access true` |
| Filter by user identity | RLS with `current_principal_details()['UserPrincipalName']` |
| Filter by group | RLS with `current_principal_is_member_of()` |
| Limit network access | Configure firewall allow list or private endpoints |
| Enable CMK | Configure Key Vault properties on cluster |
