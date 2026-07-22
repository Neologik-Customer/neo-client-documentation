# Azure OpenAI & Foundry Guide

This guide covers Azure OpenAI Service and Azure AI Foundry networking, security, and operations for the Neologik platform.

---

## Table of Contents

1. [Service Overview](#service-overview)
2. [Network Configuration](#network-configuration)
3. [Private Endpoints](#private-endpoints)
4. [Security & Access Control](#security--access-control)
5. [Monitoring & Usage](#monitoring--usage)
6. [Quota Management](#quota-management)
7. [Troubleshooting](#troubleshooting)
8. [Best Practices](#best-practices)

---

## Service Overview

### Azure OpenAI Service

Provides REST API access to OpenAI models (GPT-4, GPT-3.5, embeddings, etc.) with enterprise security.

### Azure AI Foundry

Next-generation platform for building and deploying AI applications.

**Key Features:**
- Model deployment and management
- Prompt engineering and testing
- Data integration (indexes, datastores)
- Enterprise security and compliance
- Model fine-tuning

### View Service Details

```bash
# List OpenAI accounts
az cognitiveservices account list \
  --resource-group <rg-name> \
  --query "[?kind=='OpenAI']" \
  -o table

# Show OpenAI account details
az cognitiveservices account show \
  --resource-group <rg-name> \
  --name <openai-name>

# Get endpoint
az cognitiveservices account show \
  --resource-group <rg-name> \
  --name <openai-name> \
  --query properties.endpoint -o tsv

# List deployed models
az cognitiveservices account deployment list \
  --resource-group <rg-name> \
  --name <openai-name> \
  -o table
```

---

## Network Configuration

### Network Access Options

1. **Public endpoint** (default): Accessible from internet
2. **Public endpoint with firewall**: Restrict to specific IPs/VNets
3. **Private endpoint**: Fully private access via VNet

### Configure Public Network Access

```bash
# Disable public access
az cognitiveservices account update \
  --resource-group <rg-name> \
  --name <openai-name> \
  --public-network-access Disabled

# Enable public access
az cognitiveservices account update \
  --resource-group <rg-name> \
  --name <openai-name> \
  --public-network-access Enabled
```

### Configure IP Firewall

```bash
# Add allowed IP ranges
az cognitiveservices account network-rule add \
  --resource-group <rg-name> \
  --name <openai-name> \
  --ip-address 203.0.113.0/24

# Add VNet/subnet to allowed networks
az cognitiveservices account network-rule add \
  --resource-group <rg-name> \
  --name <openai-name> \
  --vnet-name <vnet-name> \
  --subnet <subnet-name>

# List network rules
az cognitiveservices account show \
  --resource-group <rg-name> \
  --name <openai-name> \
  --query networkAcls
```

### Network Rule Set

```bash
# Set default action to Deny (must explicitly allow access)
az cognitiveservices account update \
  --resource-group <rg-name> \
  --name <openai-name> \
  --default-action Deny

# Allow Azure services bypass
az cognitiveservices account update \
  --resource-group <rg-name> \
  --name <openai-name> \
  --bypass AzureServices
```

---

## Private Endpoints

### Create Private Endpoint

```bash
# Create private endpoint for OpenAI
az network private-endpoint create \
  --resource-group <rg-name> \
  --name <pe-name> \
  --vnet-name <vnet-name> \
  --subnet <subnet-name> \
  --private-connection-resource-id <openai-resource-id> \
  --group-id account \
  --connection-name "openai-pe-connection"
```

### DNS Configuration

**Create private DNS zone:**
```bash
# Create DNS zone for OpenAI
az network private-dns zone create \
  --resource-group <rg-name> \
  --name "privatelink.openai.azure.com"

# Link to VNet
az network private-dns link vnet create \
  --resource-group <rg-name> \
  --zone-name "privatelink.openai.azure.com" \
  --name "openai-dns-link" \
  --virtual-network <vnet-name> \
  --registration-enabled false
```

**Get private IP:**
```bash
# Get private endpoint IP
az network private-endpoint show \
  --resource-group <rg-name> \
  --name <pe-name> \
  --query "customDnsConfigs[0].ipAddresses[0]" -o tsv
```

**Create DNS A record:**
```bash
# Get OpenAI account name
OPENAI_NAME="<openai-account-name>"

# Create A record
az network private-dns record-set a add-record \
  --resource-group <rg-name> \
  --zone-name "privatelink.openai.azure.com" \
  --record-set-name $OPENAI_NAME \
  --ipv4-address <private-endpoint-ip>
```

### Verify Private Endpoint Connectivity

```bash
# Test DNS resolution (from VM in VNet)
nslookup <openai-name>.openai.azure.com

# Should resolve to private IP (10.x.x.x)

# Test API connectivity
curl https://<openai-name>.openai.azure.com/openai/deployments?api-version=2024-02-01 \
  -H "api-key: <api-key>"
```

### Multi-Region Private Endpoints

For redundancy, create private endpoints in multiple regions:

```bash
# Primary region
az network private-endpoint create \
  --resource-group <rg-name> \
  --name "pe-openai-eastus" \
  --vnet-name <vnet-eastus> \
  --subnet <subnet-name> \
  --private-connection-resource-id <openai-resource-id> \
  --group-id account

# Secondary region
az network private-endpoint create \
  --resource-group <rg-name> \
  --name "pe-openai-westus" \
  --vnet-name <vnet-westus> \
  --subnet <subnet-name> \
  --private-connection-resource-id <openai-resource-id> \
  --group-id account
```

---

## Security & Access Control

### API Key Management

```bash
# List keys
az cognitiveservices account keys list \
  --resource-group <rg-name> \
  --name <openai-name>

# Regenerate primary key
az cognitiveservices account keys regenerate \
  --resource-group <rg-name> \
  --name <openai-name> \
  --key-name key1

# Regenerate secondary key
az cognitiveservices account keys regenerate \
  --resource-group <rg-name> \
  --name <openai-name> \
  --key-name key2
```

**Best practices:**
- Rotate keys regularly (quarterly)
- Use Key Vault to store keys
- Use secondary key during rotation
- Never commit keys to source control

### Managed Identity Authentication

**Enable system-assigned identity:**
```bash
# Enable on OpenAI account
az cognitiveservices account identity assign \
  --resource-group <rg-name> \
  --name <openai-name>
```

**Use managed identity from application:**
```bash
# Grant managed identity access
MI_PRINCIPAL_ID=$(az webapp identity show \
  --resource-group <rg-name> \
  --name <app-name> \
  --query principalId -o tsv)

az role assignment create \
  --assignee $MI_PRINCIPAL_ID \
  --role "Cognitive Services User" \
  --scope <openai-resource-id>
```

**Application code example (Python):**
```python
from azure.identity import DefaultAzureCredential
from openai import AzureOpenAI

credential = DefaultAzureCredential()
token = credential.get_token("https://cognitiveservices.azure.com/.default")

client = AzureOpenAI(
    azure_endpoint="https://<openai-name>.openai.azure.com",
    api_version="2024-02-01",
    azure_ad_token=token.token
)
```

### RBAC Roles

```bash
# Cognitive Services User - Use API
az role assignment create \
  --assignee <user-or-app-id> \
  --role "Cognitive Services User" \
  --scope <openai-resource-id>

# Cognitive Services Contributor - Manage resource
az role assignment create \
  --assignee <user-id> \
  --role "Cognitive Services Contributor" \
  --scope <openai-resource-id>

# Cognitive Services OpenAI User - Use OpenAI models
az role assignment create \
  --assignee <user-or-app-id> \
  --role "Cognitive Services OpenAI User" \
  --scope <openai-resource-id>
```

---

## Monitoring & Usage

### Enable Diagnostic Logging

```bash
# Enable diagnostics to Log Analytics
az monitor diagnostic-settings create \
  --resource <openai-resource-id> \
  --name "OpenAI-Diagnostics" \
  --workspace <log-analytics-workspace-id> \
  --logs '[
    {
      "category": "Audit",
      "enabled": true
    },
    {
      "category": "RequestResponse",
      "enabled": true
    }
  ]' \
  --metrics '[
    {
      "category": "AllMetrics",
      "enabled": true
    }
  ]'
```

### Key Metrics

**Via Portal:**
1. Navigate to OpenAI resource
2. Select **Monitoring** → **Metrics**
3. Key metrics:
   - **Total Calls**: Request count
   - **Successful Calls**: Successful requests
   - **Total Errors**: Failed requests
   - **Total Tokens**: Token consumption
   - **Processed Prompt Tokens**: Input tokens
   - **Generated Completion Tokens**: Output tokens

### Usage Monitoring Queries

```kusto
// Request volume over time
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.COGNITIVESERVICES"
| where Category == "RequestResponse"
| where TimeGenerated > ago(24h)
| summarize Requests = count() by bin(TimeGenerated, 1h), model_s
| render timechart

// Error rate analysis
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.COGNITIVESERVICES"
| where Category == "RequestResponse"
| where TimeGenerated > ago(24h)
| summarize Total = count(), 
    Errors = countif(httpStatusCode_d >= 400) by bin(TimeGenerated, 5m)
| extend ErrorRate = (Errors * 100.0) / Total
| render timechart

// Token consumption by model
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.COGNITIVESERVICES"
| where Category == "RequestResponse"
| where TimeGenerated > ago(7d)
| extend PromptTokens = toint(prompt_tokens_d)
| extend CompletionTokens = toint(completion_tokens_d)
| extend TotalTokens = PromptTokens + CompletionTokens
| summarize 
    TotalRequests = count(),
    PromptTokensSum = sum(PromptTokens),
    CompletionTokensSum = sum(CompletionTokens),
    TotalTokensSum = sum(TotalTokens)
    by model_s
| order by TotalTokensSum desc

// HTTP 429 (Rate Limited) events
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.COGNITIVESERVICES"
| where Category == "RequestResponse"
| where httpStatusCode_d == 429
| summarize RateLimitEvents = count() by bin(TimeGenerated, 5m), model_s
| render timechart

// Average response time
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.COGNITIVESERVICES"
| where Category == "RequestResponse"
| where TimeGenerated > ago(24h)
| extend Duration = todouble(duration_d)
| summarize AvgDuration = avg(Duration), 
    P95Duration = percentile(Duration, 95) by model_s
```

### Cost Monitoring

```kusto
// Token-based cost estimation (example pricing)
let gpt4_prompt_price = 0.03 / 1000;  // $ per 1K tokens
let gpt4_completion_price = 0.06 / 1000;
let gpt35_prompt_price = 0.0015 / 1000;
let gpt35_completion_price = 0.002 / 1000;
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.COGNITIVESERVICES"
| where TimeGenerated > startofmonth(now())
| extend PromptTokens = toint(prompt_tokens_d)
| extend CompletionTokens = toint(completion_tokens_d)
| extend EstimatedCost = case(
    model_s contains "gpt-4", 
        (PromptTokens * gpt4_prompt_price) + (CompletionTokens * gpt4_completion_price),
    model_s contains "gpt-35-turbo",
        (PromptTokens * gpt35_prompt_price) + (CompletionTokens * gpt35_completion_price),
    0.0
  )
| summarize TotalEstimatedCost = sum(EstimatedCost) by model_s
| order by TotalEstimatedCost desc
```

---

## Quota Management

### View Quotas

```bash
# List deployments with capacity
az cognitiveservices account deployment list \
  --resource-group <rg-name> \
  --name <openai-name> \
  --query "[].{Name:name, Model:properties.model.name, Capacity:sku.capacity}" \
  -o table
```

**Quota types:**
- **TPM (Tokens Per Minute)**: Rate limit per deployment
- **RPM (Requests Per Minute)**: Request count limit
- **PTU (Provisioned Throughput Units)**: Reserved capacity

### Manage Deployment Capacity

```bash
# Update deployment capacity
az cognitiveservices account deployment update \
  --resource-group <rg-name> \
  --name <openai-name> \
  --deployment-name <deployment-name> \
  --sku-capacity 50 \
  --sku-name "Standard"
```

### Handle Rate Limiting

**Implement retry logic with exponential backoff:**

```python
import time
import openai
from openai import RateLimitError

def call_openai_with_retry(max_retries=3):
    for attempt in range(max_retries):
        try:
            response = client.chat.completions.create(
                model="gpt-4",
                messages=[{"role": "user", "content": "Hello"}]
            )
            return response
        except RateLimitError as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt  # Exponential backoff
                time.sleep(wait_time)
            else:
                raise
```

### Monitor Quota Usage

```kusto
// Requests approaching quota limit
let QuotaLimit = 60000;  // TPM limit
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.COGNITIVESERVICES"
| where TimeGenerated > ago(1h)
| extend TotalTokens = toint(prompt_tokens_d) + toint(completion_tokens_d)
| summarize TokensPerMinute = sum(TotalTokens) by bin(TimeGenerated, 1m), model_s
| extend QuotaUsagePercent = (TokensPerMinute * 100.0) / QuotaLimit
| where QuotaUsagePercent > 80
| render timechart
```

---

## Troubleshooting

### Connection Issues with Private Endpoint

**Diagnose:**
```bash
# Check private endpoint status
az network private-endpoint show \
  --resource-group <rg-name> \
  --name <pe-name> \
  --query "provisioningState"

# Test DNS resolution
nslookup <openai-name>.openai.azure.com
# Should return private IP

# Test connectivity
curl -v https://<openai-name>.openai.azure.com
```

**Common causes:**
- DNS not configured correctly
- NSG blocking traffic
- Private endpoint not approved
- VNet peering not configured

**Solutions:**
- Verify DNS A record points to private IP
- Check NSG allows outbound 443 to private endpoint subnet
- Approve private endpoint connection
- Verify VNet peering configuration

### HTTP 401 Unauthorized

**Common causes:**
- Invalid API key
- Expired managed identity token
- Insufficient RBAC permissions
- Wrong API endpoint

**Solutions:**
```bash
# Verify API key
az cognitiveservices account keys list \
  --resource-group <rg-name> \
  --name <openai-name>

# Check RBAC assignments
az role assignment list \
  --scope <openai-resource-id> \
  -o table

# Verify endpoint URL
az cognitiveservices account show \
  --resource-group <rg-name> \
  --name <openai-name> \
  --query properties.endpoint
```

### HTTP 429 Too Many Requests

**Causes:**
- Exceeded TPM quota
- Exceeded RPM quota
- Burst traffic

**Solutions:**
- Implement retry with exponential backoff
- Increase deployment capacity
- Distribute load across multiple deployments
- Request quota increase from Azure support

```bash
# Increase deployment capacity
az cognitiveservices account deployment update \
  --resource-group <rg-name> \
  --name <openai-name> \
  --deployment-name <deployment-name> \
  --sku-capacity 100
```

### HTTP 503 Service Unavailable

**Causes:**
- Azure service outage
- Deployment not ready
- Resource temporarily unavailable

**Solutions:**
- Check [Azure Status](https://status.azure.com/)
- Verify deployment status
- Wait and retry
- Implement failover to another region

---

## Best Practices

### Network Security
- **Use private endpoints** for production workloads
- **Disable public access** when using private endpoints
- **Implement IP allowlisting** if private endpoints not feasible
- **Use VNet service endpoints** where private endpoints not available
- **Monitor network rules** regularly

### Authentication & Authorization
- **Use managed identities** instead of API keys when possible
- **Rotate API keys** quarterly
- **Store keys in Key Vault**
- **Use RBAC** for fine-grained access control
- **Audit access** regularly

### Performance & Reliability
- **Deploy in multiple regions** for high availability
- **Implement retry logic** with exponential backoff
- **Monitor quota usage** proactively
- **Set up alerts** for rate limiting
- **Cache responses** when appropriate
- **Use streaming** for long responses

### Cost Optimization
- **Monitor token usage** by application/team
- **Right-size deployments** based on actual usage
- **Use GPT-3.5** where GPT-4 not required
- **Implement request caching**
- **Set max_tokens** appropriately
- **Review and optimize prompts** for efficiency

### Monitoring & Operations
- **Enable diagnostic logging** to Log Analytics
- **Set up dashboards** for key metrics
- **Alert on rate limiting**
- **Monitor error rates**
- **Track costs** by deployment
- **Regular security reviews**

### Data Privacy
- **Review data residency** requirements
- **Understand data retention** policies
- **Disable logging** for sensitive data if needed
- **Use customer-managed keys** for encryption at rest
- **Implement data filtering** in applications

---

## Common Runbook Actions

### 1. Deploy New Model

```bash
# Create model deployment
az cognitiveservices account deployment create \
  --resource-group <rg-name> \
  --name <openai-name> \
  --deployment-name "gpt-4-production" \
  --model-name "gpt-4" \
  --model-version "0613" \
  --model-format "OpenAI" \
  --sku-capacity 50 \
  --sku-name "Standard"
```

### 2. Rotate API Keys

```bash
# Update application to use secondary key
# Then regenerate primary key
az cognitiveservices account keys regenerate \
  --resource-group <rg-name> \
  --name <openai-name> \
  --key-name key1

# After applications updated, regenerate secondary
az cognitiveservices account keys regenerate \
  --resource-group <rg-name> \
  --name <openai-name> \
  --key-name key2
```

### 3. Scale Deployment for Peak Load

```bash
# Increase capacity before expected high load
az cognitiveservices account deployment update \
  --resource-group <rg-name> \
  --name <openai-name> \
  --deployment-name <deployment-name> \
  --sku-capacity 120

# Scale down after peak
az cognitiveservices account deployment update \
  --resource-group <rg-name> \
  --name <openai-name> \
  --deployment-name <deployment-name> \
  --sku-capacity 50
```

### 4. Troubleshoot Rate Limiting

```kusto
// Identify applications hitting rate limits
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.COGNITIVESERVICES"
| where httpStatusCode_d == 429
| where TimeGenerated > ago(1h)
| summarize RateLimitCount = count() by 
    CallerIpAddress, 
    userAgent_s,
    model_s
| order by RateLimitCount desc
```

---

## Additional Resources

- [Azure OpenAI Networking](https://learn.microsoft.com/en-us/azure/ai-foundry/openai/how-to/network)
- [Azure OpenAI On Your Data](https://learn.microsoft.com/en-us/azure/ai-foundry/openai/how-to/on-your-data-configuration)
- [Azure OpenAI Quotas](https://learn.microsoft.com/en-us/azure/ai-services/openai/quotas-limits)
- [Azure OpenAI Best Practices](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/fine-tuning-considerations)

---

*Last Updated: January 2026*
