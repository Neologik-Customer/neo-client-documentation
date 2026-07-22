# Application Insights Guide

This guide covers Application Performance Monitoring (APM) using Azure Application Insights for the Neologik platform.

---

## Table of Contents

1. [Application Insights Overview](#application-insights-overview)
2. [Monitoring Applications](#monitoring-applications)
3. [Performance Monitoring](#performance-monitoring)
4. [Failure Analysis](#failure-analysis)
5. [Availability Testing](#availability-testing)
6. [Custom Telemetry](#custom-telemetry)
7. [Diagnostic Tools](#diagnostic-tools)
8. [Common Troubleshooting](#common-troubleshooting)

---

## Application Insights Overview

Application Insights is an APM service that monitors live applications, automatically detects performance anomalies, and helps diagnose issues.

### Key Features

- **Real-time monitoring**: Live metrics stream
- **Request tracking**: Response times and success rates
- **Dependency tracking**: External calls (databases, APIs)
- **Exception tracking**: Detailed error information
- **Custom events**: Application-specific telemetry
- **Smart Detection**: AI-powered anomaly detection

### Access Application Insights

**Via Azure Portal:**
1. Navigate to your Application Insights resource
2. Key sections:
   - **Overview**: Key metrics and health
   - **Performance**: Request and dependency performance
   - **Failures**: Exceptions and failed requests
   - **Availability**: Uptime monitoring
   - **Metrics**: Custom charts
   - **Logs**: Query telemetry data

**Via CLI:**
```bash
# List Application Insights resources
az monitor app-insights component show \
  --resource-group <rg-name> \
  --app <app-insights-name>

# Get instrumentation key
az monitor app-insights component show \
  --resource-group <rg-name> \
  --app <app-insights-name> \
  --query instrumentationKey -o tsv
```

---

## Monitoring Applications

### Live Metrics

View real-time telemetry as it arrives.

**Access Live Metrics:**
1. Navigate to Application Insights
2. Select **Live Metrics** from left menu
3. Monitor:
   - Incoming requests per second
   - Request duration
   - Failed requests
   - CPU and memory usage
   - Custom events

**Use cases:**
- Deployment validation
- Real-time troubleshooting
- Load testing verification
- Performance optimization

### Application Map

Visualize application topology and dependencies.

**Access Application Map:**
1. Navigate to **Investigate** → **Application Map**
2. View:
   - Application components
   - Dependencies (databases, external services)
   - Request volumes
   - Failure rates
   - Average response times

**Investigate issues:**
- Click on component to see metrics
- Identify slow dependencies
- Find failure points
- Trace request flow

### Transaction Search

Search and analyze individual transactions.

**Access Transaction Search:**
1. Navigate to **Investigate** → **Transaction search**
2. Filter by:
   - Time range
   - Event type (requests, dependencies, exceptions)
   - Result code
   - Custom properties

**View transaction details:**
- Full request timeline
- Related events
- Custom properties
- Stack traces

---

## Performance Monitoring

### Performance Dashboard

**Access Performance:**
1. Navigate to **Investigate** → **Performance**
2. Review:
   - **Operations tab**: Request performance by endpoint
   - **Dependencies tab**: External call performance
   - **Overall**: Aggregated metrics

### Analyze Slow Requests

```kusto
// Slowest requests in last 24 hours
requests
| where timestamp > ago(24h)
| where success == true
| order by duration desc
| take 20
| project timestamp, name, duration, url, resultCode

// Requests slower than 3 seconds
requests
| where timestamp > ago(1h)
| where duration > 3000
| summarize Count = count(), AvgDuration = avg(duration) by name
| order by Count desc
```

**Drill into slow requests:**
1. Click on operation name
2. Select **Drill into... samples**
3. View individual request details
4. Check **End-to-end transaction**

### Monitor Dependencies

```kusto
// Slow database calls
dependencies
| where timestamp > ago(1h)
| where type == "SQL"
| where duration > 1000
| summarize Count = count(), AvgDuration = avg(duration) by name
| order by AvgDuration desc

// Dependency failures
dependencies
| where timestamp > ago(24h)
| where success == false
| summarize Failures = count() by name, type, resultCode
| order by Failures desc
```

### Performance Alerts

**Create performance alert:**
1. Navigate to **Monitoring** → **Alerts**
2. **+ New alert rule**
3. Add condition:
   - Signal: Server response time
   - Threshold: > 3 seconds
   - Evaluation: Average over 5 minutes
4. Add action group
5. Create alert

---

## Failure Analysis

### Failures Dashboard

**Access Failures:**
1. Navigate to **Investigate** → **Failures**
2. View:
   - **Operations tab**: Failed requests by endpoint
   - **Dependencies tab**: Failed dependency calls
   - **Exceptions tab**: Application exceptions

### Analyze Exceptions

```kusto
// Top exceptions in last 24 hours
exceptions
| where timestamp > ago(24h)
| summarize Count = count() by type, outerMessage
| order by Count desc

// Exception details with stack trace
exceptions
| where timestamp > ago(1h)
| project timestamp, type, outerMessage, innermostMessage, details
| order by timestamp desc

// Exceptions by operation
exceptions
| where timestamp > ago(24h)
| join kind=inner (requests) on operation_Id
| summarize ExceptionCount = count() by operation_Name
| order by ExceptionCount desc
```

### Failed Requests

```kusto
// Failed requests by status code
requests
| where timestamp > ago(24h)
| where success == false
| summarize Count = count() by resultCode, name
| order by Count desc

// 5xx errors
requests
| where timestamp > ago(1h)
| where resultCode >= 500
| project timestamp, name, resultCode, duration, url
| order by timestamp desc
```

### Exception Tracking

**View exception details:**
1. Click on exception in Failures view
2. Review:
   - Stack trace
   - Failed operation
   - Custom properties
   - Related telemetry
3. Click **View all telemetry** for complete context

---

## Availability Testing

Availability tests monitor application uptime from global locations.

### Create URL Ping Test

**Via Portal:**
1. Navigate to **Availability**
2. Click **+ Add Standard test**
3. Configure:
   - **Test name**: Descriptive name
   - **URL**: Endpoint to test
   - **Test frequency**: 5 or 15 minutes
   - **Test locations**: Select multiple locations
   - **Success criteria**:
     - HTTP status code (default: 200-399)
     - Timeout: 30 seconds
     - SSL validation
4. **Create**

### Multi-Step Web Test

For complex test scenarios:
1. Create Visual Studio web test
2. Upload `.webtest` file
3. Configure in Application Insights

### Availability Alerts

Automatically created with availability tests:
- Alert if test fails from 2+ locations
- Email notifications to subscription admins

**Customize availability alerts:**
1. Navigate to **Alerts**
2. Find auto-created availability alert
3. Edit conditions:
   - Number of failed locations
   - Time window
4. Modify action groups

### Diagnose Test Failures

**View test results:**
1. Navigate to **Availability**
2. Select test
3. View:
   - Success/failure scatter plot
   - Geographic availability
   - Test details

**Common failure reasons:**
```kusto
// Availability test failures
availabilityResults
| where timestamp > ago(24h)
| where success == false
| summarize Count = count() by location, message
| order by Count desc
```

**Troubleshoot failures:**
- Check HTTP status code
- Verify DNS resolution
- Check SSL certificate validity
- Ensure firewall allows test IPs
- Review response content

---

## Custom Telemetry

### Track Custom Events

**Application code examples:**

**.NET:**
```csharp
var telemetry = new TelemetryClient();
telemetry.TrackEvent("UserLogin", new Dictionary<string, string> {
    { "UserId", userId },
    { "LoginMethod", "Azure AD" }
});
```

**Python:**
```python
from applicationinsights import TelemetryClient
tc = TelemetryClient('<instrumentation-key>')
tc.track_event('UserLogin', {'UserId': user_id, 'LoginMethod': 'Azure AD'})
tc.flush()
```

**Node.js:**
```javascript
const appInsights = require('applicationinsights');
appInsights.defaultClient.trackEvent({
    name: 'UserLogin',
    properties: { UserId: userId, LoginMethod: 'Azure AD' }
});
```

### Query Custom Events

```kusto
// Count custom events
customEvents
| where timestamp > ago(24h)
| summarize Count = count() by name
| order by Count desc

// Custom event properties
customEvents
| where timestamp > ago(1h)
| where name == "UserLogin"
| extend UserId = tostring(customDimensions.UserId)
| project timestamp, UserId, customDimensions
```

### Track Custom Metrics

**Application code:**
```csharp
telemetry.TrackMetric("QueueLength", queueLength);
telemetry.TrackMetric("ProcessingTime", processingTime, new Dictionary<string, string> {
    { "Queue", queueName }
});
```

**Query custom metrics:**
```kusto
customMetrics
| where timestamp > ago(1h)
| where name == "QueueLength"
| summarize Avg = avg(value), Max = max(value) by bin(timestamp, 5m)
| render timechart
```

---

## Diagnostic Tools

### End-to-End Transaction Diagnostics

**View complete transaction:**
1. Navigate to **Transaction search**
2. Select a request
3. Click **View all telemetry**
4. See timeline:
   - Request start/end
   - Dependencies called
   - Exceptions thrown
   - Custom events

### Profiler

Collect detailed performance traces.

**Enable Profiler:**
1. Navigate to **Performance**
2. Click **Configure Profiler**
3. Enable profiler
4. Set sampling rate

**View profiler traces:**
1. Navigate to **Performance**
2. Select operation
3. Click **Profiler traces**
4. Analyze call tree and timing

### Snapshot Debugger

Capture snapshots when exceptions occur (for .NET apps).

**Enable Snapshot Debugger:**
1. Navigate to **Failures**
2. Click **Configure Snapshot Debugger**
3. Enable for application

**View snapshots:**
1. Navigate to **Failures** → **Exceptions**
2. Select exception with snapshot
3. Click **Open debug snapshot**
4. View local variables and call stack

### Smart Detection

AI-powered anomaly detection alerts.

**Smart detection types:**
- Failure anomalies
- Performance anomalies
- Memory leak detection
- Slow page load
- Security detection pattern

**Configure smart detection:**
1. Navigate to **Smart Detection**
2. View detected anomalies
3. Click anomaly for details
4. Configure notification settings

---

## Common Troubleshooting

### High Response Times

**Diagnosis steps:**
```kusto
// Identify slow operations
requests
| where timestamp > ago(1h)
| summarize Avg = avg(duration), P95 = percentile(duration, 95) by name
| where Avg > 1000
| order by Avg desc

// Find slow dependencies
dependencies
| where timestamp > ago(1h)
| summarize Avg = avg(duration) by name, type
| where Avg > 500
| order by Avg desc
```

**Common causes:**
- Slow database queries
- External API latency
- Insufficient resources
- N+1 query problems
- Large response payloads

### High Failure Rate

**Diagnosis steps:**
```kusto
// Failure rate over time
requests
| where timestamp > ago(24h)
| summarize Total = count(), Failed = countif(success == false) by bin(timestamp, 5m)
| extend FailureRate = (Failed * 100.0) / Total
| render timechart

// Failed requests by endpoint
requests
| where timestamp > ago(1h)
| where success == false
| summarize Count = count() by name, resultCode
| order by Count desc
```

**Investigate:**
- Check exception details
- Review dependency failures
- Check recent deployments
- Verify service health

### Missing Telemetry

**Check instrumentation:**
```bash
# Verify connection string/instrumentation key
az monitor app-insights component show \
  --resource-group <rg-name> \
  --app <app-insights-name> \
  --query connectionString
```

**Verify data ingestion:**
```kusto
// Check recent requests
requests
| where timestamp > ago(15m)
| summarize count()

// Check all telemetry types
union requests, dependencies, exceptions, customEvents
| where timestamp > ago(15m)
| summarize count() by itemType
```

**Common issues:**
- Incorrect instrumentation key
- SDK not initialized
- Sampling too aggressive
- Network/firewall blocking
- Ingestion delay (2-3 minutes)

### Availability Test Failures

**Diagnose ping test failures:**
1. Check test configuration
2. Verify URL is accessible
3. Check HTTP status codes returned
4. Review SSL certificate validity
5. Check firewall rules

**Azure test agent IPs:**
Must allow traffic from Application Insights test agents. See [documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/app/availability-standard-tests#locations) for IP ranges.

---

## Best Practices

### Telemetry Collection
- Use sampling to control costs (default: 5 samples/second)
- Filter sensitive data before sending
- Add custom properties for context
- Use correlation IDs for tracking
- Set appropriate retention period (default: 90 days)

### Performance
- Monitor key user journeys
- Set performance budgets
- Track dependencies separately
- Use profiler for deep analysis
- Monitor from user perspective (availability tests)

### Alerting
- Alert on anomalies, not just thresholds
- Use smart detection alerts
- Configure appropriate severity levels
- Test action groups regularly
- Document alert response procedures

### Cost Optimization
- Configure sampling appropriately
- Use adaptive sampling
- Filter noisy telemetry
- Set data retention based on needs
- Monitor daily cap to avoid overages

---

## Useful KQL Queries

### User Analytics

```kusto
// Daily active users
customEvents
| where timestamp > ago(30d)
| where name == "PageView"
| summarize Users = dcount(user_Id) by bin(timestamp, 1d)
| render timechart

// User sessions
requests
| where timestamp > ago(24h)
| summarize Sessions = dcount(session_Id), Requests = count() by bin(timestamp, 1h)
| render timechart
```

### Performance Trends

```kusto
// Response time trend
requests
| where timestamp > ago(7d)
| summarize Avg = avg(duration), P95 = percentile(duration, 95) by bin(timestamp, 1h)
| render timechart

// Dependency duration by type
dependencies
| where timestamp > ago(24h)
| summarize Avg = avg(duration) by type, bin(timestamp, 1h)
| render timechart
```

### Error Analysis

```kusto
// Error rate percentage
requests
| where timestamp > ago(24h)
| summarize Total = count(), Errors = countif(success == false) 
    by bin(timestamp, 5m)
| extend ErrorRate = (Errors * 100.0) / Total
| render timechart

// New exceptions (not seen in previous week)
let previousExceptions = exceptions
    | where timestamp between (ago(14d) .. ago(7d))
    | distinct type, outerMessage;
exceptions
| where timestamp > ago(7d)
| distinct type, outerMessage
| join kind=leftanti (previousExceptions) on type, outerMessage
```

---

## Additional Resources

- [Application Insights Overview](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview)
- [Availability Tests](https://learn.microsoft.com/en-us/azure/azure-monitor/app/availability)
- [Application Insights FAQ](https://learn.microsoft.com/en-us/azure/azure-monitor/app/application-insights-faq)
- [Smart Detection](https://learn.microsoft.com/en-us/azure/azure-monitor/app/smart-detection)

---

*Last Updated: January 2026*
