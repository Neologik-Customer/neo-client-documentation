# Azure Bastion Guide

This guide covers Azure Bastion operations and session monitoring for secure VM access in the Neologik platform.

---

## Table of Contents

1. [Bastion Overview](#bastion-overview)
2. [Connecting to VMs](#connecting-to-vms)
3. [Session Monitoring](#session-monitoring)
4. [Bastion Configuration](#bastion-configuration)
5. [Troubleshooting](#troubleshooting)
6. [Best Practices](#best-practices)

---

## Bastion Overview

Azure Bastion provides secure RDP/SSH connectivity to VMs without exposing public IP addresses.

### Key Features

- **No public IPs required**: VMs don't need public IPs
- **SSL protection**: Encrypted RDP/SSH over 443
- **No NSG changes**: Works through existing firewall rules
- **Portal integration**: Connect directly from Azure Portal
- **Session recording**: Audit trail of connections
- **MFA support**: Azure AD multi-factor authentication

### Bastion SKUs

- **Basic**: Standard features, manual scaling
- **Standard**: Advanced features, autoscaling, IP-based connection

### View Bastion Configuration

```bash
# Show Bastion details
az network bastion show \
  --resource-group <rg-name> \
  --name <bastion-name>

# Get Bastion DNS name
az network bastion show \
  --resource-group <rg-name> \
  --name <bastion-name> \
  --query "dnsName" -o tsv
```

---

## Connecting to VMs

### Connect via Azure Portal

**For Windows VMs (RDP):**
1. Navigate to target VM
2. Select **Connect** → **Bastion**
3. Enter credentials:
   - **Username**: Local admin or domain user
   - **Password**: User password
   - Or use **Azure AD** authentication
4. Click **Connect**
5. Browser opens new tab with RDP session

**For Linux VMs (SSH):**
1. Navigate to target VM
2. Select **Connect** → **Bastion**
3. Choose authentication:
   - **SSH Private Key from Local File**
   - **SSH Private Key from Azure Key Vault**
   - **Password**
4. Enter username
5. Provide key or password
6. Click **Connect**

### Connect to VM by IP (Standard SKU)

**Requirements:**
- Bastion Standard SKU
- IP-based connection feature enabled
- Target VM in same virtual network or peered network

**Steps:**
1. Navigate to Bastion resource
2. Select **Connect**
3. Enter target VM private IP
4. Choose protocol (RDP/SSH)
5. Enter credentials
6. Click **Connect**

### Connect Using Azure CLI

**Native client support (Standard SKU):**
```bash
# Connect via RDP using native client
az network bastion rdp \
  --resource-group <rg-name> \
  --name <bastion-name> \
  --target-resource-id <vm-resource-id>

# Connect via SSH using native client
az network bastion ssh \
  --resource-group <rg-name> \
  --name <bastion-name> \
  --target-resource-id <vm-resource-id> \
  --auth-type password \
  --username <username>
```

**Tunnel for custom clients:**
```bash
# Create SSH tunnel
az network bastion tunnel \
  --resource-group <rg-name> \
  --name <bastion-name> \
  --target-resource-id <vm-resource-id> \
  --resource-port 22 \
  --port 50022

# Then connect using local SSH client
ssh <username>@127.0.0.1 -p 50022
```

---

## Session Monitoring

### Enable Diagnostic Logging

```bash
# Enable Bastion diagnostic logs
az monitor diagnostic-settings create \
  --resource <bastion-resource-id> \
  --name "Bastion-Diagnostics" \
  --workspace <log-analytics-workspace-id> \
  --logs '[
    {
      "category": "BastionAuditLogs",
      "enabled": true
    }
  ]'
```

### View Active Sessions

**Via Portal:**
1. Navigate to Bastion resource
2. Select **Monitoring** → **Sessions**
3. View active sessions:
   - User
   - Target VM
   - Protocol
   - Start time
   - Duration

### Query Session Logs

```kusto
// All Bastion sessions in last 24 hours
MicrosoftAzureBastionAuditLogs
| where TimeGenerated > ago(24h)
| project TimeGenerated, UserName, TargetResourceId, 
    Protocol, SourceIPAddress, Duration
| order by TimeGenerated desc

// Sessions by user
MicrosoftAzureBastionAuditLogs
| where TimeGenerated > ago(7d)
| summarize SessionCount = count(), 
    TotalDuration = sum(Duration) by UserName
| order by SessionCount desc

// Failed connection attempts
MicrosoftAzureBastionAuditLogs
| where TimeGenerated > ago(24h)
| where ResultType == "Failure"
| project TimeGenerated, UserName, TargetResourceId, 
    Message, SourceIPAddress
| order by TimeGenerated desc

// Sessions to specific VM
MicrosoftAzureBastionAuditLogs
| where TimeGenerated > ago(7d)
| where TargetResourceId contains "<vm-name>"
| project TimeGenerated, UserName, Protocol, Duration
| order by TimeGenerated desc
```

### Session Monitoring Alerts

**Alert on unusual access patterns:**
```bash
# Create log alert for after-hours access
az monitor scheduled-query create \
  --name "Bastion-AfterHours-Access" \
  --resource-group <rg-name> \
  --scopes <log-analytics-workspace-id> \
  --condition "count 'MicrosoftAzureBastionAuditLogs | where TimeGenerated > ago(5m) | where hourofday(TimeGenerated) < 6 or hourofday(TimeGenerated) > 20' > 0" \
  --description "Alert on after-hours Bastion access" \
  --evaluation-frequency 5m \
  --window-size 5m \
  --severity 2
```

---

## Bastion Configuration

### Upgrade to Standard SKU

```bash
# Upgrade Bastion from Basic to Standard
az network bastion update \
  --resource-group <rg-name> \
  --name <bastion-name> \
  --sku Standard
```

### Enable Features (Standard SKU)

```bash
# Enable IP-based connection
az network bastion update \
  --resource-group <rg-name> \
  --name <bastion-name> \
  --enable-ip-connect true

# Enable native client support
az network bastion update \
  --resource-group <rg-name> \
  --name <bastion-name> \
  --enable-tunneling true

# Enable session recording
az network bastion update \
  --resource-group <rg-name> \
  --name <bastion-name> \
  --enable-session-recording true
```

### Scale Bastion (Standard SKU)

```bash
# Set scale units (2-50 instances)
az network bastion update \
  --resource-group <rg-name> \
  --name <bastion-name> \
  --scale-units 5
```

**Scale units:**
- 2 units = ~8 concurrent sessions
- Each additional unit = ~4 more sessions
- Maximum 50 units = ~200 sessions

### Configure Custom Domain

**Requirements:**
- Custom TLS certificate in Key Vault
- DNS CNAME record pointing to Bastion

```bash
# Configure custom domain (via ARM template or Portal)
# Currently requires Azure Portal or ARM template
```

---

## Troubleshooting

### Cannot Connect to VM

**Check connectivity:**
```bash
# Verify Bastion status
az network bastion show \
  --resource-group <rg-name> \
  --name <bastion-name> \
  --query "provisioningState"

# Check if VM is running
az vm get-instance-view \
  --resource-group <rg-name> \
  --name <vm-name> \
  --query "instanceView.statuses[?starts_with(code, 'PowerState/')].displayStatus" -o tsv
```

**Common issues:**

1. **VM not in same VNet**
   - Solution: Verify VM is in Bastion VNet or peered VNet

2. **NSG blocking traffic**
   - Required: Allow inbound 443 to Bastion subnet
   - Required: Allow outbound 3389 (RDP) or 22 (SSH) to target VM

3. **Bastion subnet too small**
   - Required: /26 or larger subnet
   - Solution: Expand Bastion subnet

4. **VM agent not running**
   - Check VM agent status in Portal
   - Restart VM agent if needed

### Connection Drops or Slow

**Check session logs:**
```kusto
MicrosoftAzureBastionAuditLogs
| where TimeGenerated > ago(1h)
| where ResultType == "Failure" or Duration < 10
| project TimeGenerated, UserName, TargetResourceId, Message
```

**Common causes:**
- Network latency to Azure region
- Bastion scale units insufficient
- VM performance issues
- Browser or client issues

**Solutions:**
- Check browser compatibility (latest Chrome/Edge recommended)
- Increase Bastion scale units if at capacity
- Verify VM is not resource-constrained
- Test from different network

### Authentication Failures

**Check audit logs:**
```kusto
MicrosoftAzureBastionAuditLogs
| where TimeGenerated > ago(24h)
| where ResultType == "Failure"
| where Message contains "authentication"
| summarize Count = count() by UserName, Message
```

**Common causes:**
- Incorrect credentials
- Account locked/disabled
- MFA not completed
- Azure AD authentication not configured

**Solutions:**
- Verify credentials are correct
- Check account status in AD
- Complete MFA if required
- Configure Azure AD authentication properly

---

## Best Practices

### Security
- **Use Azure AD authentication** when possible
- **Enable MFA** for all administrative access
- **Use Just-In-Time (JIT) access** with Azure Defender
- **Rotate local admin passwords** regularly
- **Monitor session logs** for unusual activity
- **Restrict Bastion subnet** with NSG rules
- **Enable session recording** for compliance

### Operations
- **Deploy Bastion in hub VNet** for centralized access
- **Use Standard SKU** for production environments
- **Configure appropriate scale units** based on concurrent users
- **Set up alerts** for failed connections and after-hours access
- **Document VM access procedures** for team
- **Test DR procedures** regularly

### Cost Optimization
- **Use Basic SKU** for dev/test environments
- **Right-size scale units** based on actual usage
- **Consider shared Bastion** across multiple VNets via peering
- **Monitor usage** and adjust scaling accordingly

### Compliance
- **Enable diagnostic logging** to Log Analytics
- **Retain logs** according to compliance requirements
- **Review access logs** periodically
- **Implement approval workflows** for production access
- **Document who accessed what and when**

---

## Monitoring and Alerts

### Key Metrics to Monitor

**Via Portal:**
1. Navigate to Bastion
2. Select **Monitoring** → **Metrics**
3. Available metrics:
   - **Session count**: Active sessions
   - **Total sessions**: Cumulative sessions
   - **Session duration**: Average duration

### Recommended Alerts

```bash
# Alert on high session count
az monitor metrics alert create \
  --name "Bastion-High-Session-Count" \
  --resource-group <rg-name> \
  --scopes <bastion-resource-id> \
  --condition "avg sessions > 40" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2

# Alert on failed connections
az monitor scheduled-query create \
  --name "Bastion-Failed-Connections" \
  --resource-group <rg-name> \
  --scopes <log-analytics-workspace-id> \
  --condition "count 'MicrosoftAzureBastionAuditLogs | where TimeGenerated > ago(15m) | where ResultType == \"Failure\"' > 5" \
  --description "Multiple failed Bastion connections" \
  --evaluation-frequency 15m \
  --window-size 15m \
  --severity 2
```

---

## Common Queries

### Access Audit Report

```kusto
// Monthly access report
MicrosoftAzureBastionAuditLogs
| where TimeGenerated between (startofmonth(now()) .. now())
| summarize 
    TotalSessions = count(),
    UniqueSessions = dcount(SessionId),
    UniqueUsers = dcount(UserName),
    UniqueVMs = dcount(TargetResourceId),
    AvgDuration = avg(Duration)
| project TotalSessions, UniqueUsers, UniqueVMs, 
    AvgDurationMinutes = AvgDuration / 60
```

### Security Analysis

```kusto
// Failed connection attempts by source IP
MicrosoftAzureBastionAuditLogs
| where TimeGenerated > ago(7d)
| where ResultType == "Failure"
| summarize FailedAttempts = count() by SourceIPAddress, UserName
| where FailedAttempts > 3
| order by FailedAttempts desc

// Unusual access times
MicrosoftAzureBastionAuditLogs
| where TimeGenerated > ago(30d)
| extend Hour = hourofday(TimeGenerated)
| where Hour < 6 or Hour > 20
| summarize Sessions = count() by UserName, Hour
| order by Sessions desc
```

### Usage Patterns

```kusto
// Most accessed VMs
MicrosoftAzureBastionAuditLogs
| where TimeGenerated > ago(30d)
| summarize Sessions = count() by TargetResourceId
| extend VMName = split(TargetResourceId, '/')[-1]
| order by Sessions desc
| take 10

// Session duration distribution
MicrosoftAzureBastionAuditLogs
| where TimeGenerated > ago(7d)
| summarize Sessions = count() by DurationBucket = 
    case(Duration < 300, "<5min",
         Duration < 1800, "5-30min",
         Duration < 3600, "30-60min",
         ">60min")
```

---

## Additional Resources

- [Bastion Overview](https://learn.microsoft.com/en-us/azure/bastion/bastion-overview)
- [Monitor Bastion](https://learn.microsoft.com/en-us/azure/bastion/monitor-bastion)
- [Session Monitoring](https://learn.microsoft.com/en-us/azure/bastion/session-monitoring)
- [Bastion Audit Logs](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/microsoftazurebastionauditlogs)

---

*Last Updated: January 2026*
