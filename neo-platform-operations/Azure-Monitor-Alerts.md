# Azure Monitor & Alerts Guide

This guide covers Azure Monitor configuration, alert management, and action groups for the Neologik platform.

---

## Table of Contents

1. [Azure Monitor Overview](#azure-monitor-overview)
2. [Metrics and Logs](#metrics-and-logs)
3. [Alert Types](#alert-types)
4. [Creating Alerts](#creating-alerts)
5. [Action Groups](#action-groups)
6. [Alert Management](#alert-management)
7. [Common Alert Rules](#common-alert-rules)
8. [Troubleshooting Alerts](#troubleshooting-alerts)

---

## Azure Monitor Overview

Azure Monitor collects, analyzes, and acts on telemetry data from the Neologik platform.

### Key Components

- **Metrics**: Numerical time-series data (CPU, memory, requests)
- **Logs**: Text-based records of events (Log Analytics)
- **Alerts**: Notifications based on metric or log conditions
- **Action Groups**: Define who gets notified and how
- **Workbooks**: Interactive reports and dashboards

### Access Azure Monitor

**Via Azure Portal:**
1. Navigate to **Monitor** in the portal
2. Key sections:
   - **Alerts**: View and manage alerts
   - **Metrics**: Explore metrics across resources
   - **Logs**: Query Log Analytics data
   - **Workbooks**: View dashboards
   - **Service Health**: Azure service status

---

## Metrics and Logs

### View Metrics

**Via Portal:**
1. Navigate to resource (AKS, App Gateway, etc.)
2. Select **Monitoring** → **Metrics**
3. Add metrics to chart:
   - Select metric namespace
   - Choose metric
   - Select aggregation (Avg, Max, Sum, Count)
   - Apply filters/splits

**Via Azure CLI:**
```bash
# List available metrics for resource
az monitor metrics list-definitions \
  --resource <resource-id>

# Get metric values
az monitor metrics list \
  --resource <resource-id> \
  --metric "Percentage CPU" \
  --start-time 2026-01-23T00:00:00Z \
  --end-time 2026-01-23T23:59:59Z \
  --interval PT1M
```

### Query Logs with KQL

**Access Log Analytics:**
1. Navigate to **Monitor** → **Logs**
2. Select scope (subscription, resource group, workspace)
3. Write KQL query

**Example queries:**
```kusto
// All errors in last 24 hours
AzureDiagnostics
| where TimeGenerated > ago(24h)
| where Level == "Error"
| project TimeGenerated, Resource, OperationName, Message
| order by TimeGenerated desc

// HTTP 5xx errors
AzureDiagnostics
| where TimeGenerated > ago(1h)
| where httpStatus_d >= 500
| summarize Count = count() by Resource, httpStatus_d
| order by Count desc

// Resource usage trends
Perf
| where TimeGenerated > ago(7d)
| where CounterName == "% Processor Time"
| summarize AvgCPU = avg(CounterValue) by bin(TimeGenerated, 1h), Computer
| render timechart
```

### Log Analytics Workspaces

```bash
# List workspaces
az monitor log-analytics workspace list -o table

# Show workspace details
az monitor log-analytics workspace show \
  --resource-group <rg-name> \
  --workspace-name <workspace-name>

# Query workspace
az monitor log-analytics query \
  --workspace <workspace-id> \
  --analytics-query "AzureDiagnostics | where TimeGenerated > ago(1h) | take 10"
```

---

## Alert Types

### Metric Alerts

Monitor numeric values from metrics.

**Use cases:**
- CPU/Memory thresholds
- Request rates
- Error rates
- Latency measurements

**Characteristics:**
- Near real-time (1-minute granularity)
- Simple threshold or dynamic conditions
- Auto-resolved when condition clears

### Log Alerts

Based on Log Analytics queries.

**Use cases:**
- Complex conditions across logs
- Pattern detection
- Custom business logic
- Aggregations and correlations

**Characteristics:**
- Query-based evaluation
- Flexible time windows
- Can measure number of results or metric values

### Activity Log Alerts

Triggered by Azure activity events.

**Use cases:**
- Resource creation/deletion
- Configuration changes
- Service health events
- Security events

**Characteristics:**
- Subscription or resource-level
- No delay (event-driven)
- Administrative actions

### Smart Detection Alerts

AI-powered anomaly detection (Application Insights).

**Use cases:**
- Unusual failure rates
- Performance degradation
- Memory leaks
- Security issues

---

## Creating Alerts

### Create Metric Alert (Portal)

1. Navigate to **Monitor** → **Alerts**
2. Click **+ Create** → **Alert rule**
3. Select **Scope**: Choose target resource(s)
4. Configure **Condition**:
   - Select signal (metric)
   - Set threshold type (Static or Dynamic)
   - Set threshold value
   - Configure aggregation and evaluation frequency
5. Add **Action Group** (or create new)
6. Configure **Alert Details**:
   - Severity (0-4)
   - Rule name and description
   - Resource group
7. **Review + Create**

### Create Metric Alert (CLI)

```bash
# Create metric alert
az monitor metrics alert create \
  --name "High CPU Alert" \
  --resource-group <rg-name> \
  --scopes <resource-id> \
  --condition "avg Percentage CPU > 80" \
  --description "Alert when CPU exceeds 80%" \
  --evaluation-frequency 5m \
  --window-size 15m \
  --severity 2 \
  --action <action-group-id>
```

### Create Log Alert (Portal)

1. Navigate to **Monitor** → **Alerts** → **+ Create** → **Alert rule**
2. Select **Scope**: Choose Log Analytics workspace
3. Configure **Condition**:
   - Click **Custom log search**
   - Write KQL query
   - Set threshold (number of results or metric value)
   - Configure evaluation frequency
4. Add **Action Group**
5. Configure **Alert Details**
6. **Review + Create**

### Create Log Alert (CLI)

```bash
# Create log search alert
az monitor scheduled-query create \
  --name "High Error Rate" \
  --resource-group <rg-name> \
  --scopes <workspace-id> \
  --condition "count 'Heartbeat | where TimeGenerated > ago(5m)' > 100" \
  --description "Alert on high error rate" \
  --evaluation-frequency 5m \
  --window-size 5m \
  --severity 2 \
  --action-groups <action-group-id>
```

---

## Action Groups

Action groups define notification and remediation actions when alerts fire.

### Action Types

1. **Email/SMS/Push/Voice**: Notify people
2. **Azure Function**: Trigger custom code
3. **Logic App**: Execute workflow
4. **Webhook**: Call external URL
5. **ITSM**: Create incident in ticketing system
6. **Automation Runbook**: Execute PowerShell/Python
7. **Secure Webhook**: Authenticated webhook

### Create Action Group (Portal)

1. Navigate to **Monitor** → **Alerts** → **Action groups**
2. Click **+ Create**
3. Configure **Basics**:
   - Subscription
   - Resource group
   - Action group name
   - Display name
4. Configure **Notifications**:
   - Type (Email, SMS, Push, Voice)
   - Name
   - Details (email address, phone number)
5. Configure **Actions** (optional):
   - Type (Webhook, Function, Logic App, etc.)
   - Name
   - Details
6. **Review + Create**

### Create Action Group (CLI)

```bash
# Create action group with email notification
az monitor action-group create \
  --name "ops-team-notifications" \
  --resource-group <rg-name> \
  --short-name "OpsTeam" \
  --email-receiver name="OpsEmail" email="ops@neologik.ai"

# Add webhook action
az monitor action-group update \
  --name "ops-team-notifications" \
  --resource-group <rg-name> \
  --add-action webhook \
    name="OpsWebhook" \
    serviceUri="https://your-webhook-url.com/alert"
```

### Test Action Group

```bash
# Test action group
az monitor action-group test-notifications create \
  --action-group <action-group-name> \
  --resource-group <rg-name> \
  --alert-type servicehealth
```

---

## Alert Management

### View Active Alerts

**Via Portal:**
1. Navigate to **Monitor** → **Alerts**
2. Filter by:
   - Time range
   - Severity
   - Monitor condition
   - Alert state

**Via CLI:**
```bash
# List alerts
az monitor metrics alert list --resource-group <rg-name> -o table

# Show alert details
az monitor metrics alert show \
  --name <alert-name> \
  --resource-group <rg-name>
```

### Alert States

- **New**: Alert fired but not acknowledged
- **Acknowledged**: Alert acknowledged by operator
- **Closed**: Alert resolved (manually or auto-resolved)

### Manage Alert State

**Via Portal:**
1. Navigate to **Monitor** → **Alerts**
2. Select alert
3. Click **Change user response**
4. Select **Acknowledged** or **Closed**
5. Add optional comment

**Via CLI:**
```bash
# Update alert state (requires alert instance ID)
az monitor metrics alert update \
  --name <alert-name> \
  --resource-group <rg-name> \
  --enabled false
```

### Suppress Alerts (Maintenance)

```bash
# Disable alert rule
az monitor metrics alert update \
  --name <alert-name> \
  --resource-group <rg-name> \
  --enabled false

# Re-enable after maintenance
az monitor metrics alert update \
  --name <alert-name> \
  --resource-group <rg-name> \
  --enabled true
```

---

## Common Alert Rules

### AKS Cluster Alerts

```bash
# Node CPU > 80%
az monitor metrics alert create \
  --name "AKS-High-Node-CPU" \
  --resource-group <rg-name> \
  --scopes <aks-resource-id> \
  --condition "avg node_cpu_usage_percentage > 80" \
  --window-size 15m \
  --evaluation-frequency 5m \
  --severity 2

# Node Memory > 85%
az monitor metrics alert create \
  --name "AKS-High-Node-Memory" \
  --resource-group <rg-name> \
  --scopes <aks-resource-id> \
  --condition "avg node_memory_working_set_percentage > 85" \
  --window-size 15m \
  --evaluation-frequency 5m \
  --severity 2

# Pod failures (log alert)
# KQL: KubePodInventory | where PodStatus == "Failed" | summarize count()
```

### Application Gateway Alerts

```bash
# Backend response time > 3s
az monitor metrics alert create \
  --name "AppGW-High-Latency" \
  --resource-group <rg-name> \
  --scopes <appgw-resource-id> \
  --condition "avg BackendLastByteResponseTime > 3000" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 3

# Unhealthy backend hosts
az monitor metrics alert create \
  --name "AppGW-Unhealthy-Backend" \
  --resource-group <rg-name> \
  --scopes <appgw-resource-id> \
  --condition "avg UnhealthyHostCount > 0" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 1
```

### Azure AI Search Alerts

```bash
# High search latency
az monitor metrics alert create \
  --name "Search-High-Latency" \
  --resource-group <rg-name> \
  --scopes <search-resource-id> \
  --condition "avg SearchLatency > 1000" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 3

# Throttled requests
az monitor metrics alert create \
  --name "Search-Throttled-Requests" \
  --resource-group <rg-name> \
  --scopes <search-resource-id> \
  --condition "total ThrottledSearchQueriesPercentage > 5" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2
```

### OpenAI Service Alerts

```bash
# HTTP 429 (Rate Limited)
# Use log alert with KQL query:
# AzureDiagnostics | where httpStatusCode_d == 429 | summarize count()

# Token usage > 80% of quota
az monitor metrics alert create \
  --name "OpenAI-High-Token-Usage" \
  --resource-group <rg-name> \
  --scopes <openai-resource-id> \
  --condition "total ProcessedPromptTokens > 800000" \
  --window-size 1h \
  --evaluation-frequency 15m \
  --severity 2
```

---

## Troubleshooting Alerts

### Alert Not Firing

**Check:**
1. Alert rule is enabled
2. Metric/log data is being collected
3. Threshold and time window are correct
4. Evaluation frequency is appropriate
5. Resource is in scope

**Validate query (log alerts):**
```kusto
// Run the alert query manually in Log Analytics
YourAlertQuery
| where TimeGenerated > ago(15m)
```

### Too Many False Positives

**Solutions:**
1. Adjust threshold values
2. Increase time window for aggregation
3. Use dynamic thresholds instead of static
4. Add additional conditions to query
5. Suppress alerts during known maintenance

### Alert Fired But No Notification

**Check:**
1. Action group is configured correctly
2. Email/SMS not blocked or filtered
3. Webhook endpoint is accessible
4. Action group has not hit rate limits
5. Check action group execution history

**View action history:**
1. Navigate to **Monitor** → **Alerts**
2. Select alert instance
3. View **Actions** tab
4. Check execution status

### Alert Latency

**Factors affecting latency:**
- Evaluation frequency (5m minimum)
- Metric ingestion delay (2-3 minutes typical)
- Log ingestion delay (varies)
- Action group processing time

**Reduce latency:**
- Use metric alerts for time-sensitive scenarios
- Increase evaluation frequency (costs more)
- Use Application Insights for app-level alerts

---

## Best Practices

### Alert Design
- **Actionable**: Only alert on conditions requiring action
- **Clear naming**: Include resource and condition in name
- **Appropriate severity**: Match to business impact
- **Avoid alert fatigue**: Don't over-alert
- **Document**: Add descriptions explaining the alert

### Severities
- **Sev 0 (Critical)**: Service down, immediate action required
- **Sev 1 (Error)**: Major functionality impaired
- **Sev 2 (Warning)**: Degraded performance, investigate
- **Sev 3 (Informational)**: Potential issues, monitor
- **Sev 4 (Verbose)**: Diagnostic information

### Action Groups
- Separate action groups by severity and team
- Test action groups regularly
- Keep contact information updated
- Use multiple notification channels
- Implement escalation procedures

### Maintenance
- Review and tune alerts monthly
- Remove or disable unused alerts
- Update thresholds based on trends
- Document alert response procedures
- Test during maintenance windows

---

## Useful KQL Queries

### Performance Issues

```kusto
// Slow requests
AppRequests
| where TimeGenerated > ago(1h)
| where Duration > 3000
| project TimeGenerated, Name, Duration, ResultCode, Url
| order by Duration desc

// High CPU by resource
Perf
| where TimeGenerated > ago(1h)
| where CounterName == "% Processor Time"
| summarize AvgCPU = avg(CounterValue) by Computer
| where AvgCPU > 70
| order by AvgCPU desc
```

### Error Analysis

```kusto
// Error distribution
AppExceptions
| where TimeGenerated > ago(24h)
| summarize Count = count() by Type, OuterMessage
| order by Count desc

// Failed requests by endpoint
AppRequests
| where TimeGenerated > ago(1h)
| where Success == false
| summarize Failures = count() by Name, ResultCode
| order by Failures desc
```

### Availability

```kusto
// Service availability
AppAvailabilityResults
| where TimeGenerated > ago(24h)
| summarize AvailabilityPct = 100.0 * countif(Success == true) / count() by Name
| order by AvailabilityPct asc
```

---

## Additional Resources

- [Azure Monitor Overview](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/overview)
- [Alert Types](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-types)
- [Action Groups](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/action-groups)
- [KQL Reference](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/)

---

*Last Updated: January 2026*
