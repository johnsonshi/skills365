# Security Best Practices for Azure Data Explorer

Actionable guidelines for securing ADX clusters and protecting data.

---

## 1. Access Control

### Use Least Privilege

| Task | Recommended Role | Avoid |
|------|------------------|-------|
| Read-only analytics | Database Viewer | Database User |
| Data ingestion only | Table/Database Ingestor | Database User |
| Create tables/functions | Database User | Database Admin |
| Full management | Database Admin | AllDatabasesAdmin |

### Prefer Groups Over Individual Users

**Do:**
```kusto
.add database MyDB viewers ('aadgroup=DataAnalysts;contoso.onmicrosoft.com')
```

**Avoid:**
```kusto
.add database MyDB viewers ('aaduser=user1@contoso.com')
.add database MyDB viewers ('aaduser=user2@contoso.com')
```

**Benefits:**
- Centralized permission management
- Easier onboarding/offboarding
- Integration with PIM for time-bound access
- Audit trail for group membership

### Service Principal Guidelines

| Scenario | Approach |
|----------|----------|
| Azure-hosted service | Use managed identity (no secrets) |
| Single table ingestion | Table Ingestor role |
| Multi-table ingestion | Database Ingestor role |
| Read-only queries | Database Viewer role |

**Credential Hygiene:**
- Rotate secrets regularly
- Store secrets in Key Vault
- Never embed credentials in code
- Use separate principals per service

### Time-Bound Access with PIM

1. Assign elevated permissions as "eligible" in PIM
2. Users activate when needed
3. Access expires automatically
4. Force cache refresh after changes:

```kusto
.clear cluster cache groupmembership with (group='aadGroup=ADX-Admins@domain.com')
```

---

## 2. Row-Level Security Design

### Effective RLS Patterns

**User-Based Filtering:**
```kusto
MyTable | where OwnerEmail == current_principal_details()["UserPrincipalName"]
```

**Group-Based Access:**
```kusto
let IsGlobalReader = current_principal_is_member_of('aadgroup=GlobalReaders@domain.com');
let IsRegionalReader = current_principal_is_member_of('aadgroup=USReaders@domain.com');
MyTable
| where IsGlobalReader or (IsRegionalReader and Region == "US")
```

**Manager Override:**
```kusto
let IsManager = current_principal_is_member_of('aadgroup=Managers@domain.com');
MyTable
| where IsManager or OwnerEmail == current_principal_details()["UserPrincipalName"]
```

**Data Masking:**
```kusto
let IsPrivileged = current_principal_is_member_of('aadgroup=PrivilegedUsers@domain.com');
Customers
| extend Email = iff(IsPrivileged, Email, strcat(substring(Email, 0, 3), "***@***.***"))
```

### RLS Performance Tips

1. **Evaluate boolean conditions first** - membership checks before data filtering
2. **Keep RLS queries simple** - avoid complex joins (impacts all queries)
3. **Consider partitioning** - on columns used in RLS filters
4. **Always test before enabling:**
   ```kusto
   set query_force_row_level_security;
   <YourQuery>
   ```

### RLS Anti-Patterns to Avoid

| Anti-Pattern | Issue | Better Approach |
|--------------|-------|-----------------|
| Complex joins | Performance degradation | Pre-compute access in lookup table |
| No admin bypass | Operational challenges | Include manager override clause |
| RLS as only security | False sense of security | Combine with proper RBAC |

---

## 3. Network Security

### Defense in Depth

```
Layer 1: Private Endpoints (VNet integration)
    |
Layer 2: Firewall Rules (IP allow list)
    |
Layer 3: Outbound Restrictions (Callout policies)
    |
Layer 4: Data Encryption (CMK + Disk Encryption)
```

### Private Endpoint Best Practices

- Use Azure Private DNS zones for name resolution
- Configure conditional forwarders for hybrid scenarios
- Test resolution from all client locations
- Consider hub-spoke VNet topology

### Firewall Rules Management

| Scenario | Configuration |
|----------|---------------|
| Corporate network | Corporate egress IPs/CIDR ranges |
| Azure services | Service tags (PowerBI, LogicApps) |
| Partners | Documented IP ranges with justification |

**Maintenance:**
- Document all allowed IPs with business justification
- Review quarterly; remove unused entries
- Use CIDR ranges over individual IPs
- Never use 0.0.0.0/0

### Outbound Access Control

- Configure callout policies for plugin destinations
- Use AllowedFQDNList to limit external table sources
- Deploy managed private endpoints for storage/Event Hub

---

## 4. Data Protection

### Encryption Strategy by Data Classification

| Classification | Protection |
|---------------|------------|
| Public | Standard encryption |
| Internal | Standard + RLS |
| Confidential | CMK + RLS + Private endpoints |
| Restricted | CMK + RLS + Private endpoints + Restricted View Access |

### Customer-Managed Keys

**Key Vault Setup:**
- Enable soft delete (required)
- Enable purge protection (required)
- Use RBAC for access control
- Enable logging and alerts

**Key Rotation:**
- Define rotation schedule (e.g., annually)
- Test in non-production first
- Document rollback procedures
- Monitor for rotation failures

**Access Control:**
- Limit access to cluster managed identity only
- Separate key vaults for prod/non-prod
- Implement break-glass procedures

---

## 5. Monitoring and Auditing

### Enable Logging

- Azure Activity Log (ARM operations)
- Diagnostic Logs (queries and commands)
- Microsoft Entra Sign-in Logs (authentication)
- Azure Monitor Alerts (anomaly detection)

### Useful Audit Queries

```kusto
// Recent security-related commands
.show commands
| where StartedOn > ago(7d)
| where CommandType has "Security" or CommandType has "Policy"

// Database principals
.show database MyDatabase principals

// Query patterns by user
.show queries
| where StartedOn > ago(1d)
| summarize QueryCount = count() by User
| order by QueryCount desc
```

### Review Schedule

| Review | Frequency | Focus |
|--------|-----------|-------|
| Access review | Quarterly | Remove unused principals, validate groups |
| Policy review | Quarterly | RLS policies, restricted view access |
| Network review | Quarterly | Firewall rules, private endpoints |
| Key rotation | Annually | CMK, service principal credentials |

---

## 6. Security Checklist

### Pre-Production

- [ ] Managed identities configured where possible
- [ ] Security groups created for role assignments
- [ ] RLS policies tested with multiple user contexts
- [ ] Private endpoints configured (if required)
- [ ] IP firewall rules documented and minimal
- [ ] CMK configured (if required)
- [ ] Diagnostic logging enabled

### Production Readiness

- [ ] No individual user assignments (groups only)
- [ ] Service principals use managed identities or short-lived credentials
- [ ] Outbound access restricted to required destinations
- [ ] Monitoring alerts configured
- [ ] Incident response procedures documented
- [ ] Break-glass access procedures tested

---

## Quick Decision Guide

| Question | Recommendation |
|----------|----------------|
| How to authenticate Azure services? | Managed identity |
| How to assign permissions? | Security groups, not individuals |
| How to protect sensitive tables? | Restricted View Access + RLS |
| How to filter data by user? | RLS with `current_principal_details()` |
| How to isolate network? | Private endpoints + firewall |
| How to protect encryption keys? | CMK with managed identity |
