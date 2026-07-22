# RBAC Guide

This guide covers Azure Role-Based Access Control (RBAC) for managing access to Neologik platform resources.

---

## Table of Contents

1. [RBAC Overview](#rbac-overview)
2. [Built-in Roles](#built-in-roles)
3. [Role Assignments](#role-assignments)
4. [Custom Roles](#custom-roles)
5. [Best Practices](#best-practices)
6. [Troubleshooting](#troubleshooting)

---

## RBAC Overview

Azure RBAC provides fine-grained access management for Azure resources.

### Key Concepts

- **Security Principal**: Who (user, group, service principal, managed identity)
- **Role Definition**: What (permissions)
- **Scope**: Where (subscription, resource group, resource)
- **Role Assignment**: Ties together who, what, and where

### RBAC Model

```
Security Principal + Role Definition + Scope = Role Assignment
```

**Example:**
- **Who**: ops-team@neologik.ai (Azure AD group)
- **What**: Contributor role
- **Where**: rg-neologik-prod-aks (resource group)

### Scope Hierarchy

```
Management Group
  └── Subscription
       └── Resource Group
            └── Resource
```

Permissions inherit down the hierarchy.

---

## Built-in Roles

### Common General Roles

#### Owner
- **Full access** to all resources
- Can **grant access** to others
- Use for: Admins managing subscriptions/resource groups

```bash
# Assign Owner at resource group
az role assignment create \
  --assignee <user-or-group-id> \
  --role "Owner" \
  --resource-group <rg-name>
```

#### Contributor
- **Create and manage** all resources
- **Cannot grant access** to others
- Use for: DevOps engineers, developers

```bash
# Assign Contributor at resource group
az role assignment create \
  --assignee <user-or-group-id> \
  --role "Contributor" \
  --resource-group <rg-name>
```

#### Reader
- **View all resources**
- Cannot make changes
- Use for: Auditors, read-only access

```bash
# Assign Reader at subscription
az role assignment create \
  --assignee <user-or-group-id> \
  --role "Reader" \
  --scope "/subscriptions/<subscription-id>"
```

### AKS-Specific Roles

#### Azure Kubernetes Service Cluster Admin
- Full access to cluster
- Can download **admin** kubeconfig
- Use for: Platform administrators

```bash
az role assignment create \
  --assignee <user-or-group-id> \
  --role "Azure Kubernetes Service Cluster Admin Role" \
  --scope <aks-resource-id>
```

#### Azure Kubernetes Service Cluster User
- Can download **user** kubeconfig
- Permissions in cluster controlled by Kubernetes RBAC
- Use for: Developers, operators

```bash
az role assignment create \
  --assignee <user-or-group-id> \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope <aks-resource-id>
```

#### Azure Kubernetes Service RBAC Admin
- Manage all resources in cluster
- Cannot modify cluster or node pools
- Use for: Kubernetes administrators

#### Azure Kubernetes Service RBAC Cluster Admin
- Super-user access in cluster
- Use for: Break-glass scenarios

### Monitoring Roles

#### Monitoring Contributor
- Read monitoring data
- Create/modify alerts and diagnostic settings
- Use for: Monitoring team

```bash
az role assignment create \
  --assignee <user-or-group-id> \
  --role "Monitoring Contributor" \
  --resource-group <rg-name>
```

#### Monitoring Reader
- Read all monitoring data
- Cannot modify settings
- Use for: Read-only monitoring access

#### Log Analytics Contributor
- Read log data
- Configure workspace settings
- Use for: Log Analytics administrators

#### Log Analytics Reader
- Read log data only
- Use for: Analysts, developers

### Storage Roles

#### Storage Blob Data Contributor
- Read, write, delete blob data
- Use for: Applications, data engineers

```bash
az role assignment create \
  --assignee <user-or-group-id> \
  --role "Storage Blob Data Contributor" \
  --scope <storage-account-resource-id>
```

#### Storage Blob Data Reader
- Read blob data only
- Use for: Read-only data access

#### Storage Account Contributor
- Manage storage account (not data)
- Use for: Infrastructure team

### Key Vault Roles

#### Key Vault Administrator
- Full access to Key Vault and data
- Use for: Security administrators

```bash
az role assignment create \
  --assignee <user-or-group-id> \
  --role "Key Vault Administrator" \
  --scope <keyvault-resource-id>
```

#### Key Vault Secrets User
- Read secret contents only
- Use for: Applications via managed identity

#### Key Vault Certificates Officer
- Manage certificates (not secrets/keys)
- Use for: Certificate administrators

### Network Roles

#### Network Contributor
- Manage network resources
- Use for: Network team

```bash
az role assignment create \
  --assignee <user-or-group-id> \
  --role "Network Contributor" \
  --resource-group <network-rg-name>
```

### AI Services Roles

#### Cognitive Services Contributor
- Manage AI services (not data)
- Use for: AI service administrators

#### Cognitive Services User
- Use AI services (make API calls)
- Use for: Applications via managed identity

```bash
az role assignment create \
  --assignee <managed-identity-id> \
  --role "Cognitive Services User" \
  --scope <openai-resource-id>
```

#### Search Service Contributor
- Manage search service
- Use for: Search administrators

#### Search Index Data Contributor
- Read/write search index data
- Use for: Applications managing indexes

---

## Role Assignments

### List Role Assignments

```bash
# List all assignments in resource group
az role assignment list \
  --resource-group <rg-name> \
  -o table

# List assignments for specific user
az role assignment list \
  --assignee <user-email> \
  --all \
  -o table

# List assignments for resource
az role assignment list \
  --scope <resource-id> \
  -o table
```

### Create Role Assignment

**For user:**
```bash
az role assignment create \
  --assignee user@domain.com \
  --role "Contributor" \
  --resource-group <rg-name>
```

**For Azure AD group:**
```bash
# Get group object ID first
GROUP_ID=$(az ad group show --group "ops-team" --query "id" -o tsv)

az role assignment create \
  --assignee $GROUP_ID \
  --role "Contributor" \
  --resource-group <rg-name>
```

**For service principal:**
```bash
az role assignment create \
  --assignee <service-principal-app-id> \
  --role "Contributor" \
  --resource-group <rg-name>
```

**For managed identity:**
```bash
# Get managed identity principal ID
MI_PRINCIPAL_ID=$(az identity show \
  --resource-group <rg-name> \
  --name <identity-name> \
  --query principalId -o tsv)

az role assignment create \
  --assignee $MI_PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope <storage-account-resource-id>
```

### Delete Role Assignment

```bash
# Delete specific assignment
az role assignment delete \
  --assignee <user-or-group-id> \
  --role "Contributor" \
  --resource-group <rg-name>

# List and delete by ID
ASSIGNMENT_ID=$(az role assignment list \
  --assignee <user-email> \
  --resource-group <rg-name> \
  --query "[0].id" -o tsv)

az role assignment delete --ids $ASSIGNMENT_ID
```

### Scope Examples

**Subscription scope:**
```bash
az role assignment create \
  --assignee <user-id> \
  --role "Reader" \
  --scope "/subscriptions/<subscription-id>"
```

**Resource group scope:**
```bash
az role assignment create \
  --assignee <user-id> \
  --role "Contributor" \
  --scope "/subscriptions/<subscription-id>/resourceGroups/<rg-name>"
```

**Resource scope:**
```bash
az role assignment create \
  --assignee <user-id> \
  --role "Virtual Machine Contributor" \
  --scope "/subscriptions/<subscription-id>/resourceGroups/<rg-name>/providers/Microsoft.Compute/virtualMachines/<vm-name>"
```

---

## Custom Roles

### When to Use Custom Roles

Create custom roles when built-in roles:
- Grant too many permissions (over-privileged)
- Don't grant enough permissions
- Don't match your security requirements

### View Role Definition

```bash
# View built-in role definition
az role definition list --name "Contributor" --output json

# Save to file for reference
az role definition list --name "Contributor" > contributor-role.json
```

### Create Custom Role

**From JSON file:**
```json
{
  "Name": "Neologik AKS Operator",
  "Description": "Can manage AKS workloads but not infrastructure",
  "Actions": [
    "Microsoft.ContainerService/managedClusters/read",
    "Microsoft.ContainerService/managedClusters/listClusterUserCredential/action"
  ],
  "NotActions": [
    "Microsoft.ContainerService/managedClusters/delete",
    "Microsoft.ContainerService/managedClusters/write"
  ],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": [
    "/subscriptions/<subscription-id>/resourceGroups/rg-neologik-prod-aks"
  ]
}
```

```bash
# Create custom role from JSON
az role definition create --role-definition custom-role.json
```

**Using Azure CLI:**
```bash
az role definition create --role-definition '{
  "Name": "Neologik Monitoring Reader",
  "Description": "Read monitoring data and logs",
  "Actions": [
    "Microsoft.Insights/*/read",
    "Microsoft.OperationalInsights/workspaces/*/read"
  ],
  "NotActions": [],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": ["/subscriptions/<subscription-id>"]
}'
```

### Update Custom Role

```bash
# Get existing role definition
az role definition list --name "Neologik AKS Operator" > custom-role.json

# Edit JSON file
# Update the role
az role definition update --role-definition custom-role.json
```

### Delete Custom Role

```bash
az role definition delete --name "Neologik AKS Operator"
```

---

## Best Practices

### Principle of Least Privilege
- Grant minimum permissions required
- Use built-in roles when possible
- Scope assignments narrowly (resource > RG > subscription)
- Review and remove unnecessary permissions regularly

### Group-Based Access
- Assign roles to **Azure AD groups**, not individual users
- Easier to manage as team membership changes
- Better audit trail
- Consistent permissions across team

```bash
# Good: Assign to group
az role assignment create \
  --assignee-object-id <ops-team-group-id> \
  --assignee-principal-type Group \
  --role "Contributor" \
  --resource-group <rg-name>

# Avoid: Assigning to many individual users
```

### Role Assignment Strategy

**By environment:**
```
Production:
  - ops-prod-admins → Owner (subscription)
  - ops-prod-team → Contributor (resource groups)
  - developers-prod → Reader (resource groups)

Development:
  - developers → Contributor (subscription)
  - ops-team → Owner (subscription)
```

**By function:**
```
Infrastructure:
  - infra-team → Contributor on network RG
  - infra-team → Network Contributor on shared RG

Application:
  - app-team → Contributor on app RG
  - app-team → AKS Cluster User on AKS

Monitoring:
  - ops-team → Monitoring Contributor (all RGs)
  - developers → Monitoring Reader (all RGs)
```

### Managed Identities

Use managed identities instead of service principals:
- No credential management required
- Automatic rotation
- Easier RBAC assignment

```bash
# Assign role to managed identity
MI_ID=$(az identity show -n <identity-name> -g <rg-name> --query principalId -o tsv)

az role assignment create \
  --assignee $MI_ID \
  --role "Storage Blob Data Contributor" \
  --scope <storage-resource-id>
```

### Auditing and Monitoring

**Enable activity logging:**
```bash
# Review role assignment changes
az monitor activity-log list \
  --resource-group <rg-name> \
  --offset 7d \
  --query "[?contains(authorization.action, 'roleAssignments')]" \
  -o table
```

**Query with KQL:**
```kusto
// Role assignment changes
AzureActivity
| where TimeGenerated > ago(30d)
| where OperationNameValue contains "Microsoft.Authorization/roleAssignments"
| project TimeGenerated, Caller, OperationNameValue, 
    ActivityStatusValue, Properties
| order by TimeGenerated desc

// Failed authorization attempts
AzureActivity
| where TimeGenerated > ago(7d)
| where ActivityStatusValue == "Failure"
| where Authorization contains "Denied"
| summarize Count = count() by Caller, OperationNameValue
| order by Count desc
```

### Regular Reviews

- **Monthly**: Review role assignments per resource group
- **Quarterly**: Audit all assignments at subscription level
- **Annually**: Review custom roles for relevance
- **On departure**: Remove access immediately

```bash
# Generate access report
az role assignment list \
  --all \
  --query "[].{Principal:principalName, Role:roleDefinitionName, Scope:scope}" \
  -o table > access-report-$(date +%Y%m%d).txt
```

---

## Troubleshooting

### Access Denied Errors

**Check current user permissions:**
```bash
# List your current role assignments
az role assignment list \
  --assignee $(az ad signed-in-user show --query id -o tsv) \
  --all \
  -o table
```

**Check required permissions:**
```bash
# View permissions for a role
az role definition list --name "Contributor" \
  --query "[].permissions[].actions"
```

**Common issues:**
- Role assigned at wrong scope
- Inherited deny assignment
- Conditional access blocking
- Just-In-Time access expired

### Role Assignment Not Working

**Wait for replication:**
- Role assignments can take up to **10 minutes** to replicate
- Cache may take **30 minutes** to update in some services

**Force refresh:**
```bash
# Re-authenticate to get fresh token
az logout
az login

# For AKS, re-fetch credentials
az aks get-credentials --resource-group <rg-name> --name <aks-name> --overwrite-existing
```

### Permission Denied for Managed Identity

**Verify identity enabled:**
```bash
az webapp identity show \
  --resource-group <rg-name> \
  --name <app-name>
```

**Check role assignment:**
```bash
# Get identity principal ID
PRINCIPAL_ID=$(az webapp identity show \
  --resource-group <rg-name> \
  --name <app-name> \
  --query principalId -o tsv)

# List assignments
az role assignment list \
  --assignee $PRINCIPAL_ID \
  --all \
  -o table
```

### Deny Assignments

Deny assignments override role assignments (rare, system-managed).

```bash
# List deny assignments
az role assignment list \
  --resource-group <rg-name> \
  --include-inherited \
  --include-deny-assignments
```

---

## Common Runbook Actions

### 1. Grant Developer Access to Development Environment

```bash
# Get developer group ID
DEV_GROUP_ID=$(az ad group show --group "developers" --query id -o tsv)

# Assign Contributor to dev resource groups
az role assignment create \
  --assignee $DEV_GROUP_ID \
  --role "Contributor" \
  --resource-group rg-neologik-dev

# Assign AKS Cluster User
az role assignment create \
  --assignee $DEV_GROUP_ID \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope <aks-dev-resource-id>

# Assign Monitoring Reader
az role assignment create \
  --assignee $DEV_GROUP_ID \
  --role "Monitoring Reader" \
  --resource-group rg-neologik-dev
```

### 2. Grant Application Access to Storage

```bash
# Enable managed identity on app
az webapp identity assign \
  --resource-group <rg-name> \
  --name <app-name>

# Get managed identity principal ID
MI_PRINCIPAL_ID=$(az webapp identity show \
  --resource-group <rg-name> \
  --name <app-name> \
  --query principalId -o tsv)

# Assign Storage Blob Data Contributor
az role assignment create \
  --assignee $MI_PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/<sub-id>/resourceGroups/<rg-name>/providers/Microsoft.Storage/storageAccounts/<storage-name>"
```

### 3. Onboard New Operations Team Member

```bash
# Add user to ops team group
az ad group member add \
  --group "ops-team-prod" \
  --member-id <user-object-id>

# Verify group membership
az ad group member list --group "ops-team-prod" -o table

# User now inherits all role assignments from group
```

### 4. Offboard Team Member

```bash
# List all role assignments
az role assignment list \
  --assignee user@domain.com \
  --all \
  -o table

# Remove from groups
az ad group member remove \
  --group "ops-team" \
  --member-id <user-object-id>

# Remove direct assignments (if any)
az role assignment delete \
  --assignee user@domain.com \
  --role "Contributor" \
  --resource-group <rg-name>
```

### 5. Audit Access for Compliance

```bash
# Generate comprehensive access report
az role assignment list \
  --all \
  --include-inherited \
  --query "sort_by([].{Principal:principalName, PrincipalType:principalType, Role:roleDefinitionName, Scope:scope}, &Scope)" \
  -o table > rbac-audit-$(date +%Y%m%d).txt

# Query audit logs
az monitor activity-log list \
  --start-time 2026-01-01 \
  --offset 30d \
  --query "[?contains(authorization.action, 'roleAssignments')]" \
  -o json > rbac-changes-$(date +%Y%m%d).json
```

---

## Additional Resources

- [Azure Built-in Roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles)
- [RBAC Documentation](https://learn.microsoft.com/en-us/azure/role-based-access-control/)
- [Custom Roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/custom-roles)
- [Best Practices](https://learn.microsoft.com/en-us/azure/role-based-access-control/best-practices)

---

*Last Updated: January 2026*
