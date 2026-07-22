# Application Gateway & WAF Guide

This guide covers Azure Application Gateway, Web Application Firewall (WAF), and Application Gateway Ingress Controller (AGIC) operations for the Neologik platform.

---

## Table of Contents

1. [Application Gateway Overview](#application-gateway-overview)
2. [Monitoring Application Gateway](#monitoring-application-gateway)
3. [Web Application Firewall (WAF)](#web-application-firewall-waf)
4. [AGIC Operations](#agic-operations)
5. [Diagnostics and Logging](#diagnostics-and-logging)
6. [Performance Optimization](#performance-optimization)
7. [Troubleshooting](#troubleshooting)
8. [Common Runbook Actions](#common-runbook-actions)

---

## Application Gateway Overview

Application Gateway is a Layer 7 load balancer with features including:
- HTTP/HTTPS load balancing
- SSL/TLS termination
- URL-based routing
- Session affinity
- Web Application Firewall (WAF)
- Autoscaling

### Key Components

- **Frontend IP**: Public and/or private IP addresses
- **Listeners**: Accept traffic on specific port/protocol
- **Rules**: Route requests to backend pools
- **Backend pools**: Target servers or applications
- **HTTP settings**: Backend protocol and port configuration
- **Health probes**: Monitor backend health

### View Configuration

```bash
# Show Application Gateway details
az network application-gateway show \
  --resource-group <rg-name> \
  --name <appgw-name>

# List backend pools
az network application-gateway address-pool list \
  --resource-group <rg-name> \
  --gateway-name <appgw-name> \
  -o table

# Show frontend IPs
az network application-gateway frontend-ip list \
  --resource-group <rg-name> \
  --gateway-name <appgw-name> \
  -o table
```

---

## Monitoring Application Gateway

### Key Metrics

**Via Portal:**
1. Navigate to Application Gateway
2. Select **Monitoring** → **Metrics**
3. Key metrics to monitor:
   - **Throughput**: Bytes per second
   - **Healthy/Unhealthy host count**: Backend health
   - **Total Requests**: Request volume
   - **Failed Requests**: Request errors
   - **Response Status**: 2xx, 3xx, 4xx, 5xx counts
   - **Backend Response Time**: Latency
   - **Capacity Units**: Auto-scaling metric
   - **Compute Units**: CPU usage

### View Metrics via CLI

```bash
# List available metrics
az monitor metrics list-definitions \
  --resource <appgw-resource-id>

# Get backend response time
az monitor metrics list \
  --resource <appgw-resource-id> \
  --metric "BackendResponseTime" \
  --start-time 2026-01-23T00:00:00Z \
  --end-time 2026-01-23T23:59:59Z \
  --interval PT5M \
  --aggregation Average
```

### Health Monitoring

```bash
# Check backend health
az network application-gateway show-backend-health \
  --resource-group <rg-name> \
  --name <appgw-name>
```

**Interpret health status:**
- **Healthy**: Backend responding to health probes
- **Unhealthy**: Backend failing health probes
- **Unknown**: Health probe not yet completed
- **Draining**: Backend marked for removal

### Monitor Backend Health

**Via Portal:**
1. Navigate to Application Gateway
2. Select **Monitoring** → **Backend health**
3. View health status of all backend pools
4. Click on backend for detailed probe information

---

## Web Application Firewall (WAF)

WAF protects applications from common exploits and vulnerabilities.

### WAF Modes

- **Detection**: Logs threats but doesn't block
- **Prevention**: Blocks malicious requests

### View WAF Configuration

```bash
# Show WAF policy
az network application-gateway waf-policy show \
  --resource-group <rg-name> \
  --name <waf-policy-name>

# Show WAF configuration (if using built-in policy)
az network application-gateway waf-config show \
  --resource-group <rg-name> \
  --gateway-name <appgw-name>
```

### WAF Metrics

**Key metrics to monitor:**
- **Blocked requests**: Requests blocked by WAF
- **Matched rule count**: Rules triggered
- **Request count by rule group**: Which rules are triggering

**Query WAF metrics:**
```bash
az monitor metrics list \
  --resource <appgw-resource-id> \
  --metric "ApplicationGatewayTotalRequestCount" \
  --filter "BackendSettingsPool eq '*'"
```

### WAF Rule Groups

**OWASP Core Rule Set (CRS) groups:**
- **REQUEST-911**: Method Enforcement
- **REQUEST-913**: Scanner Detection
- **REQUEST-920**: Protocol Enforcement
- **REQUEST-921**: Protocol Attack
- **REQUEST-930**: Application Attack LFI
- **REQUEST-931**: Application Attack RFI
- **REQUEST-932**: Application Attack RCE
- **REQUEST-933**: Application Attack PHP
- **REQUEST-941**: Application Attack XSS
- **REQUEST-942**: Application Attack SQLi
- **REQUEST-943**: Application Attack Session Fixation

### View WAF Logs

```kusto
// WAF blocked requests
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS"
| where Category == "ApplicationGatewayFirewallLog"
| where action_s == "Blocked"
| project TimeGenerated, hostname_s, requestUri_s, Message, ruleId_s
| order by TimeGenerated desc

// Top triggered rules
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS"
| where Category == "ApplicationGatewayFirewallLog"
| summarize Count = count() by ruleId_s, Message
| order by Count desc
| take 20
```

### Manage WAF Rules

**Disable specific rule:**
```bash
# Disable rule in WAF policy
az network application-gateway waf-policy rule-set rule disable \
  --resource-group <rg-name> \
  --policy-name <waf-policy-name> \
  --type OWASP \
  --rule-set-version 3.2 \
  --rule-group-name REQUEST-942-APPLICATION-ATTACK-SQLI \
  --rule-id 942100
```

**Create custom exclusion:**
```bash
# Add exclusion for request header
az network application-gateway waf-policy managed-rule exclusion add \
  --resource-group <rg-name> \
  --policy-name <waf-policy-name> \
  --match-variable RequestHeaderNames \
  --selector-match-operator Contains \
  --selector "User-Agent"
```

### WAF Best Practices

1. **Start in Detection mode**: Monitor before enforcing
2. **Tune false positives**: Create exclusions for legitimate traffic
3. **Monitor regularly**: Review WAF logs weekly
4. **Update CRS**: Keep rule sets current
5. **Use custom rules**: Add application-specific protection
6. **Test thoroughly**: Verify exclusions don't weaken security

---

## AGIC Operations

Application Gateway Ingress Controller manages Application Gateway configuration from Kubernetes.

### Check AGIC Status

```bash
# Get AGIC pods
kubectl get pods -n default -l app=ingress-azure

# View AGIC logs
kubectl logs -n default -l app=ingress-azure --tail=100

# Follow AGIC logs
kubectl logs -n default -l app=ingress-azure -f
```

### View AGIC Configuration

```bash
# Get ingress resources
kubectl get ingress --all-namespaces

# Describe ingress
kubectl describe ingress <ingress-name> -n <namespace>

# Get AGIC helm values
helm get values ingress-azure -n default
```

### AGIC Annotations

Common annotations for Kubernetes ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    # SSL redirect
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    
    # Backend protocol
    appgw.ingress.kubernetes.io/backend-protocol: "https"
    
    # Session affinity
    appgw.ingress.kubernetes.io/cookie-based-affinity: "true"
    
    # Backend path prefix
    appgw.ingress.kubernetes.io/backend-path-prefix: "/api"
    
    # WAF policy
    appgw.ingress.kubernetes.io/waf-policy-for-path: "/subscriptions/.../wafPolicies/mypolicy"
    
    # Health probe
    appgw.ingress.kubernetes.io/health-probe-path: "/health"
    appgw.ingress.kubernetes.io/health-probe-interval: "30"
    appgw.ingress.kubernetes.io/health-probe-timeout: "5"
```

### Restart AGIC

```bash
# Delete AGIC pod (will be recreated)
kubectl delete pod -n default -l app=ingress-azure

# Or restart deployment
kubectl rollout restart deployment/ingress-azure -n default
```

### AGIC Troubleshooting

**Check AGIC logs for errors:**
```bash
kubectl logs -n default -l app=ingress-azure | grep -i error
```

**Common AGIC issues:**
- **Configuration not applied**: Check AGIC has permissions
- **Backend pool empty**: Verify service endpoints exist
- **SSL certificate errors**: Check certificate in Key Vault
- **Health probe failures**: Verify backend health endpoint

---

## Diagnostics and Logging

### Enable Diagnostic Settings

```bash
# Create diagnostic setting
az monitor diagnostic-settings create \
  --resource <appgw-resource-id> \
  --name "AppGW-Diagnostics" \
  --workspace <log-analytics-workspace-id> \
  --logs '[
    {
      "category": "ApplicationGatewayAccessLog",
      "enabled": true
    },
    {
      "category": "ApplicationGatewayPerformanceLog",
      "enabled": true
    },
    {
      "category": "ApplicationGatewayFirewallLog",
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

### Access Logs

```kusto
// Recent requests
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS"
| where Category == "ApplicationGatewayAccessLog"
| where TimeGenerated > ago(1h)
| project TimeGenerated, host_s, requestUri_s, httpStatus_d, 
    clientIP_s, timeTaken_d
| order by TimeGenerated desc

// Slow requests
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS"
| where Category == "ApplicationGatewayAccessLog"
| where timeTaken_d > 3000
| summarize Count = count(), AvgTime = avg(timeTaken_d) by requestUri_s
| order by Count desc

// 5xx errors
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS"
| where Category == "ApplicationGatewayAccessLog"
| where httpStatus_d >= 500
| summarize Count = count() by httpStatus_d, requestUri_s
| order by Count desc
```

### Performance Logs

```kusto
// Backend connection time
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS"
| where Category == "ApplicationGatewayPerformanceLog"
| summarize AvgBackendConnectTime = avg(backendConnectTime_d),
    AvgBackendResponseTime = avg(backendResponseTime_d)
    by bin(TimeGenerated, 5m)
| render timechart
```

---

## Performance Optimization

### Autoscaling Configuration

```bash
# Show autoscale configuration
az network application-gateway show \
  --resource-group <rg-name> \
  --name <appgw-name> \
  --query "autoscaleConfiguration"

# Update autoscale settings
az network application-gateway update \
  --resource-group <rg-name> \
  --name <appgw-name> \
  --min-capacity 2 \
  --max-capacity 10
```

### Connection Draining

Enable graceful shutdown of backend pool members:

```bash
# Enable connection draining
az network application-gateway http-settings update \
  --resource-group <rg-name> \
  --gateway-name <appgw-name> \
  --name <http-settings-name> \
  --connection-draining-timeout 60
```

### SSL/TLS Optimization

**Best practices:**
- Use TLS 1.2 minimum
- Disable weak cipher suites
- Enable HTTP/2

```bash
# Set SSL policy
az network application-gateway ssl-policy set \
  --resource-group <rg-name> \
  --gateway-name <appgw-name> \
  --name AppGwSslPolicy20170401S \
  --policy-type Predefined
```

### Health Probe Configuration

**Optimal health probe settings:**
```bash
az network application-gateway probe update \
  --resource-group <rg-name> \
  --gateway-name <appgw-name> \
  --name <probe-name> \
  --protocol Https \
  --path "/health" \
  --interval 30 \
  --timeout 5 \
  --threshold 3
```

**Settings explanation:**
- **Interval**: Time between probes (30 seconds)
- **Timeout**: Probe timeout (5 seconds)
- **Threshold**: Failed probes before marking unhealthy (3)

---

## Troubleshooting

### Backend Returns 502 Bad Gateway

**Common causes:**
1. Backend not responding
2. Health probe failing
3. NSG blocking traffic
4. Backend timeout too short

**Diagnosis steps:**
```bash
# Check backend health
az network application-gateway show-backend-health \
  --resource-group <rg-name> \
  --name <appgw-name>

# Review access logs
# Look for errorInfo_s field in logs
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS"
| where Category == "ApplicationGatewayAccessLog"
| where httpStatus_d == 502
| project TimeGenerated, errorInfo_s, backendPoolName_s
```

**Solutions:**
- Verify backend is running and responsive
- Check health probe configuration
- Verify NSG rules allow traffic
- Increase backend timeout if needed

### Backend Returns 504 Gateway Timeout

**Causes:**
- Backend taking too long to respond
- Backend timeout configured too low

**Solutions:**
```bash
# Increase backend timeout (max 86400 seconds)
az network application-gateway http-settings update \
  --resource-group <rg-name> \
  --gateway-name <appgw-name> \
  --name <http-settings-name> \
  --timeout 180
```

### Unhealthy Backend Hosts

```bash
# Check backend health details
az network application-gateway show-backend-health \
  --resource-group <rg-name> \
  --name <appgw-name> \
  --query "backendAddressPools[].backendHttpSettingsCollection[].servers[]"
```

**Common issues:**
- Health probe path doesn't exist (404)
- Health probe timeout too short
- Backend SSL certificate issues
- NSG/firewall blocking health probe

**Solutions:**
- Verify health probe path returns 200 OK
- Increase probe timeout
- Add backend certificate to trusted root (if using HTTPS probe)
- Allow Application Gateway subnet in NSG

### SSL/TLS Certificate Issues

**Check certificate:**
```bash
# List SSL certificates
az network application-gateway ssl-cert list \
  --resource-group <rg-name> \
  --gateway-name <appgw-name> \
  -o table
```

**Common issues:**
- Certificate expired
- Certificate not trusted
- Hostname mismatch
- Missing intermediate certificates

### AGIC Not Updating Configuration

**Check AGIC status:**
```bash
# View AGIC logs for errors
kubectl logs -n default -l app=ingress-azure --tail=50

# Check AGIC has access to Application Gateway
kubectl describe pod -n default -l app=ingress-azure
```

**Common causes:**
- AGIC identity lacks permissions
- Application Gateway in different subscription
- Ingress annotations incorrect
- AGIC version incompatible

---

## Common Runbook Actions

### 1. Add Backend to Pool

```bash
# Add backend server to pool
az network application-gateway address-pool update \
  --resource-group <rg-name> \
  --gateway-name <appgw-name> \
  --name <backend-pool-name> \
  --servers <ip-address-1> <ip-address-2>
```

### 2. Drain Backend Before Maintenance

```bash
# Mark backend as draining (stop new connections)
# Requires connection draining enabled on HTTP settings
# Remove backend from pool temporarily
az network application-gateway address-pool update \
  --resource-group <rg-name> \
  --gateway-name <appgw-name> \
  --name <backend-pool-name> \
  --servers <remaining-backends>

# Wait for connection drain timeout period
# Perform maintenance
# Re-add backend to pool
```

### 3. Update SSL Certificate

```bash
# Upload new certificate to Key Vault
az keyvault certificate import \
  --vault-name <keyvault-name> \
  --name <cert-name> \
  --file certificate.pfx

# Update Application Gateway listener
az network application-gateway ssl-cert update \
  --resource-group <rg-name> \
  --gateway-name <appgw-name> \
  --name <cert-name> \
  --key-vault-secret-id "https://<keyvault-name>.vault.azure.net/secrets/<cert-name>"
```

### 4. Restart Application Gateway

```bash
# Stop Application Gateway
az network application-gateway stop \
  --resource-group <rg-name> \
  --name <appgw-name>

# Start Application Gateway
az network application-gateway start \
  --resource-group <rg-name> \
  --name <appgw-name>
```

**Note**: Restart causes downtime. Use only when necessary.

### 5. Clear WAF False Positive

```kusto
// Identify rule causing false positive
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS"
| where Category == "ApplicationGatewayFirewallLog"
| where action_s == "Blocked"
| where hostname_s == "yourapp.neologik.ai"
| summarize Count = count() by ruleId_s, Message
| order by Count desc
```

```bash
# Disable problematic rule or add exclusion
az network application-gateway waf-policy managed-rule exclusion add \
  --resource-group <rg-name> \
  --policy-name <waf-policy-name> \
  --match-variable RequestArgNames \
  --selector-match-operator Contains \
  --selector "userId"
```

### 6. Scale Application Gateway

```bash
# Increase minimum capacity
az network application-gateway update \
  --resource-group <rg-name> \
  --name <appgw-name> \
  --min-capacity 5
```

---

## Best Practices

### High Availability
- Deploy in availability zones
- Configure autoscaling
- Use health probes on all backends
- Enable connection draining
- Implement retry logic in applications

### Security
- Enable WAF in Prevention mode
- Use TLS 1.2 minimum
- Store certificates in Key Vault
- Enable diagnostic logging
- Regularly review WAF logs
- Use custom rules for application-specific threats

### Performance
- Enable HTTP/2
- Use SSL offload
- Configure appropriate timeouts
- Implement backend keep-alive
- Use session affinity when needed
- Monitor capacity units

### Monitoring
- Alert on unhealthy backends
- Monitor 5xx error rates
- Track backend response times
- Review WAF logs regularly
- Set up availability tests

---

## Additional Resources

- [Application Gateway Diagnostics](https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-diagnostics)
- [Monitor Application Gateway](https://learn.microsoft.com/en-us/azure/application-gateway/monitor-application-gateway)
- [WAF Best Practices](https://learn.microsoft.com/en-us/azure/web-application-firewall/ag/best-practices)
- [AGIC Overview](https://learn.microsoft.com/en-us/azure/application-gateway/ingress-controller-overview)
- [WAF CRS Rule Groups](https://learn.microsoft.com/en-us/azure/web-application-firewall/ag/application-gateway-crs-rulegroups-rules)

---

*Last Updated: January 2026*
