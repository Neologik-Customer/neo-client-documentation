# Private Endpoints & DNS Guide

This guide covers Azure Private Link, Private Endpoints, and DNS integration for secure connectivity in the Neologik platform.

---

## Table of Contents

1. [Private Link Overview](#private-link-overview)
2. [Private Endpoint Configuration](#private-endpoint-configuration)
3. [DNS Integration](#dns-integration)
4. [Azure Monitor Private Link Scope (AMPLS)](#azure-monitor-private-link-scope-ampls)
5. [Troubleshooting](#troubleshooting)
6. [Best Practices](#best-practices)

---

## Private Link Overview

Azure Private Link provides private connectivity to Azure services over the Microsoft backbone network.

### Key Concepts

- **Private Endpoint**: Network interface with private IP in your VNet
- **Private Link Service**: Your service made available via Private Link
- **Private DNS Zone**: DNS resolution for private endpoints
- **AMPLS**: Private Link scope for Azure Monitor resources

### Benefits

- **Network security**: Services accessed via private IPs only
- **No internet exposure**: Traffic stays on Microsoft backbone
- **Cross-region access**: Connect to services in other regions
- **Simplified network architecture**: No need for VPN or ExpressRoute for many scenarios

### Supported Services

Common services in Neologik platform:
- Azure Storage (Blob, File, Table, Queue)
- Azure SQL Database
- Azure Cosmos DB
- Azure Key Vault
- Azure Container Registry
- Azure AI Search
- Azure OpenAI / AI Foundry
- Azure Monitor (via AMPLS)
- Azure App Configuration

---

## Private Endpoint Configuration

### Create Private Endpoint

```bash
# Create private endpoint for storage account
az network private-endpoint create \
  --resource-group <rg-name> \
  --name <pe-name> \
  --vnet-name <vnet-name> \
  --subnet <subnet-name> \
  --private-connection-resource-id <resource-id> \
  --group-id blob \
  --connection-name <connection-name> \
  --location <location>
```

**Group IDs for common services:**
- Storage Blob: `blob`
- Storage File: `file`
- SQL Database: `sqlServer`
- Key Vault: `vault`
- Container Registry: `registry`
- AI Search: `searchService`
- OpenAI: `account`

### Create with DNS Integration

```bash
# Create private endpoint with automatic DNS integration
az network private-endpoint create \
  --resource-group <rg-name> \
  --name <pe-name> \
  --vnet-name <vnet-name> \
  --subnet <subnet-name> \
  --private-connection-resource-id <resource-id> \
  --group-id blob \
  --connection-name <connection-name> \
  --private-dns-zone <private-dns-zone-id>
```

### List Private Endpoints

```bash
# List all private endpoints in resource group
az network private-endpoint list \
  --resource-group <rg-name> \
  -o table

# Show private endpoint details
az network private-endpoint show \
  --resource-group <rg-name> \
  --name <pe-name>

# Get private IP address
az network private-endpoint show \
  --resource-group <rg-name> \
  --name <pe-name> \
  --query "customDnsConfigs[0].ipAddresses[0]" -o tsv
```

### Approve Private Endpoint Connection

```bash
# List pending connections
az network private-endpoint-connection list \
  --resource-group <rg-name> \
  --name <resource-name> \
  --type Microsoft.Storage/storageAccounts

# Approve connection
az network private-endpoint-connection approve \
  --resource-group <rg-name> \
  --resource-name <resource-name> \
  --name <connection-name> \
  --type Microsoft.Storage/storageAccounts \
  --description "Approved for production use"
```

### Subnet Requirements

Private endpoint subnet must:
- Have network policies disabled for private endpoints
- Have sufficient IP addresses available
- Not have any applied route tables (optional for some scenarios)

```bash
# Disable network policies on subnet
az network vnet subnet update \
  --resource-group <rg-name> \
  --vnet-name <vnet-name> \
  --name <subnet-name> \
  --disable-private-endpoint-network-policies true
```

---

## DNS Integration

### Private DNS Zones

Each Azure service has a corresponding private DNS zone.

**Common private DNS zones:**
```
Azure Storage Blob:     privatelink.blob.core.windows.net
Azure Storage File:     privatelink.file.core.windows.net
Azure SQL Database:     privatelink.database.windows.net
Azure Key Vault:        privatelink.vaultcore.azure.net
Azure Container Registry: privatelink.azurecr.io
Azure AI Search:        privatelink.search.windows.net
Azure OpenAI:           privatelink.openai.azure.com
Azure Monitor:          See AMPLS section
```

### Create Private DNS Zone

```bash
# Create private DNS zone
az network private-dns zone create \
  --resource-group <rg-name> \
  --name "privatelink.blob.core.windows.net"

# Link DNS zone to VNet
az network private-dns link vnet create \
  --resource-group <rg-name> \
  --zone-name "privatelink.blob.core.windows.net" \
  --name <link-name> \
  --virtual-network <vnet-name> \
  --registration-enabled false
```

### Create DNS Records

```bash
# Create A record for private endpoint
az network private-dns record-set a add-record \
  --resource-group <rg-name> \
  --zone-name "privatelink.blob.core.windows.net" \
  --record-set-name <storage-account-name> \
  --ipv4-address <private-endpoint-ip>
```

### Automatic DNS Integration

When creating private endpoint with `--private-dns-zone` parameter, Azure automatically:
1. Creates A record in private DNS zone
2. Links DNS zone to VNet (if not already linked)
3. Configures DNS for private endpoint

### Verify DNS Resolution

```bash
# Test DNS resolution from VM in VNet
nslookup <storage-account-name>.blob.core.windows.net

# Expected result: Private IP address
# Without private endpoint: Public IP address
```

```powershell
# PowerShell test
Resolve-DnsName <storage-account-name>.blob.core.windows.net

# Should return private IP (10.x.x.x or 192.168.x.x)
```

### DNS Configuration Patterns

**Hub-and-spoke topology:**
1. Create private DNS zones in hub VNet resource group
2. Link DNS zones to hub VNet
3. Link DNS zones to each spoke VNet
4. DNS resolution works across all VNets

**Single VNet:**
1. Create private DNS zones in same resource group
2. Link to VNet
3. DNS resolution works within VNet

---

## Azure Monitor Private Link Scope (AMPLS)

AMPLS allows private access to Azure Monitor, Application Insights, and Log Analytics.

### Create AMPLS

```bash
# Create Azure Monitor Private Link Scope
az monitor private-link-scope create \
  --resource-group <rg-name> \
  --name <ampls-name> \
  --location global
```

### Add Resources to AMPLS

```bash
# Add Log Analytics workspace
az monitor private-link-scope scoped-resource create \
  --resource-group <rg-name> \
  --scope-name <ampls-name> \
  --name <scoped-resource-name> \
  --linked-resource <workspace-resource-id>

# Add Application Insights
az monitor private-link-scope scoped-resource create \
  --resource-group <rg-name> \
  --scope-name <ampls-name> \
  --name <app-insights-scoped-resource-name> \
  --linked-resource <app-insights-resource-id>
```

### Create Private Endpoint for AMPLS

```bash
# Create private endpoint for AMPLS
az network private-endpoint create \
  --resource-group <rg-name> \
  --name <ampls-pe-name> \
  --vnet-name <vnet-name> \
  --subnet <subnet-name> \
  --private-connection-resource-id <ampls-resource-id> \
  --group-id azuremonitor \
  --connection-name <connection-name>
```

### AMPLS DNS Zones

AMPLS requires multiple private DNS zones:

```bash
# Create DNS zones for Azure Monitor
zones=(
  "privatelink.monitor.azure.com"
  "privatelink.oms.opinsights.azure.com"
  "privatelink.ods.opinsights.azure.com"
  "privatelink.agentsvc.azure-automation.net"
  "privatelink.blob.core.windows.net"
)

for zone in "${zones[@]}"; do
  az network private-dns zone create \
    --resource-group <rg-name> \
    --name "$zone"
  
  az network private-dns link vnet create \
    --resource-group <rg-name> \
    --zone-name "$zone" \
    --name "${zone}-link" \
    --virtual-network <vnet-name> \
    --registration-enabled false
done
```

### Configure AMPLS Network Settings

```bash
# Update AMPLS to block public access
# (Currently requires Portal or ARM template)
# Set "Public Network Access" to "Disabled" in Portal
```

### Test AMPLS Connectivity

```bash
# From VM in VNet, test Log Analytics endpoint
curl -v https://<workspace-id>.ods.opinsights.azure.com/

# Should connect via private IP
# Check certificate matches *.ods.opinsights.azure.com
```

---

## Troubleshooting

### DNS Resolution Issues

**Symptom:** Cannot resolve private endpoint address

**Diagnosis:**
```bash
# Check DNS zone link
az network private-dns link vnet list \
  --resource-group <rg-name> \
  --zone-name "privatelink.blob.core.windows.net" \
  -o table

# Check A record exists
az network private-dns record-set a list \
  --resource-group <rg-name> \
  --zone-name "privatelink.blob.core.windows.net" \
  -o table

# Test resolution from VM
nslookup <resource-name>.blob.core.windows.net
```

**Common causes:**
- DNS zone not linked to VNet
- A record missing or incorrect
- Custom DNS servers not forwarding to Azure DNS (168.63.129.16)
- Split-brain DNS configuration issues

**Solutions:**
- Link DNS zone to VNet
- Verify A record points to correct private IP
- Configure custom DNS to forward to 168.63.129.16
- Check DNS forwarder configuration

### Connection Timeouts

**Symptom:** Cannot connect to resource via private endpoint

**Diagnosis:**
```bash
# Check private endpoint status
az network private-endpoint show \
  --resource-group <rg-name> \
  --name <pe-name> \
  --query "provisioningState"

# Check connection state
az network private-endpoint show \
  --resource-group <rg-name> \
  --name <pe-name> \
  --query "privateLinkServiceConnections[0].privateLinkServiceConnectionState"

# Test connectivity from VM
Test-NetConnection -ComputerName <resource-name>.blob.core.windows.net -Port 443
```

**Common causes:**
- NSG blocking traffic to private endpoint
- Service firewall blocking VNet
- Connection not approved
- Route table issues

**Solutions:**
- Allow traffic in NSG to private endpoint subnet
- Add VNet to service firewall allow list
- Approve private endpoint connection
- Verify route tables allow traffic

### AMPLS Issues

**Symptom:** Cannot access Log Analytics or Application Insights

**Diagnosis:**
```kusto
// Check for connection errors in Log Analytics
Operation
| where TimeGenerated > ago(1h)
| where Detail contains "connection" or Detail contains "network"
| project TimeGenerated, Detail
```

**Common causes:**
- Not all required DNS zones configured
- Resource not added to AMPLS scope
- Public network access enabled on resource
- Data ingestion endpoint not in AMPLS

**Solutions:**
- Configure all 5 required DNS zones for AMPLS
- Add resource to AMPLS scope
- Disable public network access on resource
- Ensure AMPLS includes ingestion endpoints

### Split-Brain DNS

**Symptom:** Different resolution inside vs outside VNet

**Scenario:** Need public resolution for some clients, private for others

**Solution:**
```
1. Use conditional DNS forwarding
2. Configure on-premises DNS to forward Azure zones to Azure DNS
3. Test from both networks
4. Document which networks resolve to private vs public
```

---

## Best Practices

### Network Design
- **Centralize DNS zones**: Create in hub or shared services VNet
- **Use dedicated subnet**: Separate subnet for private endpoints
- **Plan IP addressing**: Ensure sufficient IPs for private endpoints
- **Document mappings**: Track which resources use private endpoints
- **Test failover**: Verify connectivity during outages

### DNS Configuration
- **Link DNS zones to all VNets**: Hub and all spokes
- **Use Azure DNS (168.63.129.16)**: For VMs custom DNS configuration
- **Forward from on-premises**: Configure conditional forwarders
- **Monitor DNS resolution**: Alert on resolution failures
- **Keep DNS zones organized**: Use naming conventions

### Security
- **Disable public access**: After private endpoint configured
- **Use NSGs**: Control traffic to private endpoint subnet
- **Monitor access**: Log and alert on private endpoint traffic
- **Regular audits**: Review private endpoint configurations
- **Document exceptions**: When public access required

### AMPLS
- **Create per environment**: Separate AMPLS for prod/dev
- **Include all resources**: Add all monitoring resources to scope
- **Configure all DNS zones**: Don't skip any of the 5 zones
- **Test thoroughly**: Verify data ingestion and queries work
- **Monitor connectivity**: Alert on connection failures

### Operations
- **Automate deployment**: Use IaC (Bicep, Terraform)
- **Version control**: Track DNS zone and PE configurations
- **Change management**: Document all changes
- **Disaster recovery**: Document recovery procedures
- **Cost monitoring**: Track private endpoint costs

---

## Common Runbook Actions

### 1. Add Private Endpoint for New Service

```bash
# 1. Create private endpoint
az network private-endpoint create \
  --resource-group <rg-name> \
  --name "pe-${service_name}" \
  --vnet-name <vnet-name> \
  --subnet <pe-subnet-name> \
  --private-connection-resource-id <resource-id> \
  --group-id <group-id> \
  --connection-name "pe-connection"

# 2. Create DNS zone if not exists
az network private-dns zone create \
  --resource-group <dns-rg-name> \
  --name "privatelink.<service>.azure.com"

# 3. Link DNS zone to VNets
az network private-dns link vnet create \
  --resource-group <dns-rg-name> \
  --zone-name "privatelink.<service>.azure.com" \
  --name "link-to-<vnet-name>" \
  --virtual-network <vnet-id> \
  --registration-enabled false

# 4. Verify DNS resolution
nslookup <service-name>.<service>.azure.com
```

### 2. Troubleshoot DNS Resolution

```bash
# Check from VM in VNet
$dnsName = "<service-name>.blob.core.windows.net"
Resolve-DnsName $dnsName

# Expected: 
# - NameHost shows privatelink address
# - IP Address shows private IP (10.x or 192.168.x)

# If public IP returned:
# 1. Check DNS zone linked to VNet
az network private-dns link vnet list \
  --resource-group <rg-name> \
  --zone-name "privatelink.blob.core.windows.net"

# 2. Check A record exists
az network private-dns record-set a list \
  --resource-group <rg-name> \
  --zone-name "privatelink.blob.core.windows.net"

# 3. Check VM DNS settings use Azure DNS
ipconfig /all  # Should show 168.63.129.16
```

### 3. Update AMPLS Configuration

```bash
# Add new Log Analytics workspace to AMPLS
az monitor private-link-scope scoped-resource create \
  --resource-group <rg-name> \
  --scope-name <ampls-name> \
  --name "<workspace-name>-scoped" \
  --linked-resource <workspace-resource-id>

# Verify workspace in AMPLS
az monitor private-link-scope scoped-resource list \
  --resource-group <rg-name> \
  --scope-name <ampls-name>
```

---

## Monitoring Queries

### Private Endpoint Connections

```kusto
// Private endpoint connection events
AzureActivity
| where TimeGenerated > ago(7d)
| where OperationNameValue contains "privateEndpoint"
| project TimeGenerated, Caller, OperationNameValue, 
    ResourceGroup, Resource, ActivityStatusValue
| order by TimeGenerated desc
```

### DNS Query Analysis

```kusto
// DNS query patterns (requires DNS Analytics)
DnsEvents
| where TimeGenerated > ago(24h)
| where QueryType == "A"
| where Name contains "privatelink"
| summarize Queries = count() by Name, IPAddresses
| order by Queries desc
```

---

## Additional Resources

- [Private Endpoint Overview](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview)
- [Private Endpoint DNS](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns)
- [Private Endpoint DNS Integration](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns-integration)
- [AMPLS Configure](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/private-link-configure)

---

*Last Updated: January 2026*
