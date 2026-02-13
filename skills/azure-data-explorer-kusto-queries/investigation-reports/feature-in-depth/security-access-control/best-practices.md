# Security and Access Control Best Practices

## Overview

This document outlines best practices for securing Azure Data Explorer (ADX) clusters, implementing effective access control, and protecting sensitive data. Following these practices helps ensure compliance, minimize attack surface, and maintain operational security.

---

## 1. Least Privilege Access Patterns

### 1.1 Role Assignment Guidelines

#### Principle of Least Privilege

Always grant the minimum permissions required for a task:

| Task | Recommended Role | Avoid |
|------|------------------|-------|
| Read-only analytics | Database Viewer | Database User |
| Data ingestion only | Database/Table Ingestor | Database User |
| Create tables/functions | Database User | Database Admin |
| Full database management | Database Admin | AllDatabasesAdmin |
| Monitoring/troubleshooting | Database Monitor | Database Admin |

#### Role Escalation Path

```
Viewer → User → Admin → AllDatabasesAdmin
   ↓        ↓       ↓
Ingestor  Monitor  Cluster Admin (ARM)
```

### 1.2 Group-Based Access

Prefer Microsoft Entra security groups over individual user assignments:

**Benefits:**
- Centralized permission management
- Easier onboarding/offboarding
- Audit trail for group membership changes
- Integration with Privileged Identity Management (PIM)

**Implementation Pattern:**
```
Security Group: ADX-Database-Readers
  └── Assigned Role: Database Viewer

Security Group: ADX-Database-Writers
  └── Assigned Role: Database User

Security Group: ADX-Database-Admins
  └── Assigned Role: Database Admin
```

### 1.3 Time-Bound Access

Use Microsoft Entra Privileged Identity Management (PIM) for elevated access:

1. Assign users to groups with elevated permissions as "eligible"
2. Users activate membership when needed
3. Access automatically expires after configured duration
4. Force group membership refresh when needed:

```kusto
.clear cluster cache groupmembership with (group='aadGroup=ADX-Admins@domain.com')
```

### 1.4 Service Principal Scoping

For automated services:

| Use Case | Recommended Scope |
|----------|------------------|
| Single table ingestion | Table Ingestor |
| Database-wide ingestion | Database Ingestor |
| Read-only queries | Database Viewer |
| Specific table queries | Create function with RLS |
| Administrative tasks | Database Admin (with rotation) |

**Credential Hygiene:**
- Use managed identities when possible (eliminates secret management)
- Rotate service principal secrets regularly
- Store secrets in Azure Key Vault
- Never embed credentials in code

---

## 2. Row-Level Security Design Patterns

### 2.1 User-Based Filtering

Filter data based on the current user's identity:

```kusto
// Simple user-based filtering
Sales | where SalesPersonEmail == current_principal_details()["UserPrincipalName"]
```

### 2.2 Group-Based Filtering

Grant different data access based on group membership:

```kusto
// Multi-group access pattern
let IsGlobalReader = current_principal_is_member_of('aadgroup=GlobalReaders@domain.com');
let IsRegionalReader = current_principal_is_member_of('aadgroup=USReaders@domain.com');
Sales
| where IsGlobalReader
    or (IsRegionalReader and Region == "US")
```

### 2.3 Manager Override Pattern

Allow managers to see all data while restricting individual contributors:

```kusto
let IsManager = current_principal_is_member_of('aadgroup=SalesManagers@domain.com');
let AllData = Sales | where IsManager;
let RestrictedData = Sales
    | where not(IsManager)
    | where SalesPersonEmail == current_principal_details()["UserPrincipalName"];
union AllData, RestrictedData
```

### 2.4 Data Masking Pattern

Mask sensitive columns for unauthorized users:

```kusto
let IsPrivileged = current_principal_is_member_of('aadgroup=PrivilegedUsers@domain.com');
Customers
| extend
    Email = iff(IsPrivileged, Email, strcat(substring(Email, 0, 3), "***@***.***")),
    SSN = iff(IsPrivileged, SSN, strcat("***-**-", substring(SSN, 7, 4))),
    CreditCard = iff(IsPrivileged, CreditCard, strcat("****-****-****-", substring(CreditCard, 15, 4)))
```

### 2.5 Multi-Tenant Isolation

Isolate tenant data in a shared database:

```kusto
// Tenant ID stored in user's custom claims or derived from domain
let TenantId = extract(@"@(.+)\.com", 1, tostring(current_principal_details()["UserPrincipalName"]));
MultiTenantData | where Tenant == TenantId
```

### 2.6 RLS Performance Best Practices

1. **Evaluate boolean conditions early**: Structure queries so membership checks happen first
2. **Use partitioning**: If RLS filters on a specific column, consider partitioning on that column
3. **Avoid complex joins**: Keep RLS queries simple; complex queries impact all table access
4. **Test with force flag**: Use `set query_force_row_level_security;` before enabling

```kusto
// Efficient pattern - membership check first
let HasAccess = current_principal_is_member_of('aadgroup=AccessGroup@domain.com');
MyTable | where HasAccess or (OwnerId == current_principal())
```

### 2.7 RLS Testing Strategy

Before enabling RLS in production:

1. Create the RLS function separately
2. Test with different user contexts
3. Measure query performance impact
4. Use `query_force_row_level_security` for validation
5. Enable in non-production first

---

## 3. Network Isolation Strategies

### 3.1 Defense in Depth Approach

Layer multiple network security controls:

```
Layer 1: Private Endpoints (VNet integration)
    ↓
Layer 2: Firewall Rules (IP allow list)
    ↓
Layer 3: Outbound Restrictions (Callout policies)
    ↓
Layer 4: Data Encryption (CMK + Disk Encryption)
```

### 3.2 Private Endpoint Architecture

**Recommended Design:**

```
On-Premises Network
    ↓ (ExpressRoute/VPN)
Hub VNet (Shared Services)
    ↓ (VNet Peering)
Spoke VNet (ADX)
    └── Private Endpoint → ADX Cluster
```

**DNS Configuration:**
- Use Azure Private DNS zones
- Configure conditional forwarders for hybrid scenarios
- Test resolution from all client locations

### 3.3 IP Firewall Strategy

When full private endpoint isolation isn't possible:

| Scenario | Configuration |
|----------|---------------|
| Corporate network only | Add corporate egress IPs |
| Azure services integration | Use service tags (PowerBI, LogicApps) |
| Partner access | Add partner IP ranges with documentation |
| Development/testing | Separate allow list for dev environments |

**Maintenance:**
- Document all allowed IPs with business justification
- Review quarterly and remove unused entries
- Use CIDR ranges instead of individual IPs when possible

### 3.4 Outbound Access Control

Restrict what external services ADX can call:

| Feature | Use Case | Configuration |
|---------|----------|---------------|
| Callout Policy | Control plugin destinations | Specify allowed FQDNs |
| AllowedFQDNList | Limit external table sources | Cluster-level setting |
| Managed Private Endpoints | Secure storage/Event Hub access | VNet integration |

### 3.5 Hybrid Network Considerations

For organizations with on-premises data centers:

1. **ExpressRoute**: Private connectivity to Azure
2. **VPN Gateway**: Backup connectivity option
3. **Private DNS**: Resolve ADX private endpoints from on-premises
4. **Network Security Groups**: Control traffic within VNets

---

## 4. Data Protection Best Practices

### 4.1 Encryption Strategy

**Minimum Configuration:**
- Azure Storage encryption (automatic, Microsoft-managed)

**Enhanced Security:**
- Customer-managed keys for data encryption
- Disk encryption for VM cache
- Double encryption for compliance requirements

### 4.2 Key Management Best Practices

For customer-managed keys:

1. **Key Vault Configuration:**
   - Enable soft delete (required)
   - Enable purge protection (required)
   - Use RBAC for Key Vault access control
   - Enable logging and alerts

2. **Key Rotation:**
   - Define rotation schedule (e.g., annually)
   - Test rotation in non-production first
   - Document rollback procedures
   - Monitor for rotation failures

3. **Access Control:**
   - Limit key access to cluster managed identity only
   - Use separate key vaults for production/non-production
   - Implement break-glass procedures for emergencies

### 4.3 Sensitive Data Handling

| Data Classification | Protection Measures |
|--------------------|---------------------|
| Public | Standard encryption |
| Internal | RLS + Standard encryption |
| Confidential | RLS + CMK + Private endpoints |
| Restricted | RLS + CMK + Private endpoints + Restricted View Access |

---

## 5. Monitoring and Auditing

### 5.1 Security Monitoring

Enable and monitor:

1. **Azure Activity Log**: ARM-level operations
2. **Diagnostic Logs**: Query and command execution
3. **Microsoft Entra Sign-in Logs**: Authentication events
4. **Azure Monitor Alerts**: Suspicious activity detection

### 5.2 Audit Queries

```kusto
// Who has access to this database?
.show database MyDatabase principals

// What commands were executed recently?
.show commands
| where StartedOn > ago(7d)
| where CommandType has "Security" or CommandType has "Policy"

// Failed authentication attempts (requires diagnostic logs)
ADXSecurityLogs
| where OperationName == "Authentication"
| where ResultType == "Failure"
| summarize FailedAttempts = count() by Principal, bin(TimeGenerated, 1h)
```

### 5.3 Regular Security Reviews

| Review Type | Frequency | Focus Areas |
|-------------|-----------|-------------|
| Access review | Quarterly | Remove unused principals, validate group memberships |
| Policy review | Quarterly | RLS policies, restricted view access |
| Network review | Quarterly | Firewall rules, private endpoint configurations |
| Key rotation | Annually | CMK rotation, service principal credentials |
| Compliance audit | Annually | Full security posture assessment |

---

## 6. Common Anti-Patterns to Avoid

### 6.1 Access Control Anti-Patterns

| Anti-Pattern | Issue | Better Approach |
|--------------|-------|-----------------|
| AllDatabasesAdmin for everyone | Excessive permissions | Use database-specific roles |
| Individual user assignments | Hard to manage at scale | Use security groups |
| Shared service principal | No accountability | Separate principals per service |
| No credential rotation | Increased compromise risk | Implement rotation schedule |

### 6.2 RLS Anti-Patterns

| Anti-Pattern | Issue | Better Approach |
|--------------|-------|-----------------|
| Complex joins in RLS | Performance degradation | Pre-compute access in separate table |
| RLS on high-cardinality filter | Slow queries | Use partitioning |
| No manager override | Operational challenges | Include admin bypass clause |
| Relying solely on RLS | False sense of security | Combine with proper RBAC |

### 6.3 Network Security Anti-Patterns

| Anti-Pattern | Issue | Better Approach |
|--------------|-------|-----------------|
| 0.0.0.0/0 in firewall | Open to internet | Specific IP ranges only |
| No outbound restrictions | Data exfiltration risk | Configure callout policies |
| Missing DNS integration | Resolution failures | Use Azure Private DNS zones |
| No network monitoring | Blind to attacks | Enable NSG flow logs |

---

## 7. Security Checklist

### Pre-Production Checklist

- [ ] Managed identities configured (avoid secrets where possible)
- [ ] Security groups created for role assignments
- [ ] RLS policies tested with multiple user contexts
- [ ] Private endpoints configured (if required)
- [ ] IP firewall rules documented and minimal
- [ ] Customer-managed keys configured (if required)
- [ ] Diagnostic logging enabled
- [ ] Access review process defined

### Production Readiness

- [ ] No individual user assignments (groups only)
- [ ] Service principals use managed identities or short-lived credentials
- [ ] Outbound access restricted to required destinations
- [ ] Encryption configuration matches data classification
- [ ] Monitoring alerts configured for security events
- [ ] Incident response procedures documented
- [ ] Break-glass access procedures tested

---

*Generated: 2026-02-13 | Source: Azure Data Explorer Documentation Analysis | Phase 3 - Security Deep Investigation*
