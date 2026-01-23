# Azure AI Search Guide

This guide covers Azure AI Search (formerly Azure Cognitive Search) operations, monitoring, and optimization for the Neologik platform.

---

## Table of Contents

1. [Azure AI Search Overview](#azure-ai-search-overview)
2. [Monitoring Search Service](#monitoring-search-service)
3. [Index Management](#index-management)
4. [Query Performance](#query-performance)
5. [Indexer Operations](#indexer-operations)
6. [Capacity Planning](#capacity-planning)
7. [Security Configuration](#security-configuration)
8. [Troubleshooting](#troubleshooting)

---

## Azure AI Search Overview

Azure AI Search provides AI-powered search capabilities for the Neologik knowledge base.

### Key Concepts

- **Search Service**: Top-level resource
- **Indexes**: Searchable data collections
- **Indexers**: Automate data ingestion from sources
- **Data Sources**: Origin of content (Blob, SQL, etc.)
- **Skillsets**: AI enrichment pipeline (OCR, entities, etc.)

### Service Tiers

- **Free**: Development/testing only
- **Basic**: Small production workloads
- **Standard (S1, S2, S3)**: Production workloads
- **Storage Optimized (L1, L2)**: Large data volumes

### View Service Information

```bash
# Show search service details
az search service show \
  --resource-group <rg-name> \
  --name <search-service-name>

# Get service endpoint
az search service show \
  --resource-group <rg-name> \
  --name <search-service-name> \
  --query "endpoint" -o tsv

# List admin keys
az search admin-key show \
  --resource-group <rg-name> \
  --service-name <search-service-name>
```

---

## Monitoring Search Service

### Key Metrics

**Via Portal:**
1. Navigate to Search service
2. Select **Monitoring** → **Metrics**
3. Key metrics:
   - **Search latency**: Query response time
   - **Search queries per second**: Query volume
   - **Throttled search queries percentage**: Rate limiting
   - **Document count**: Total indexed documents
   - **Index size**: Storage used
   - **Indexer execution count**: Indexing activity
   - **Indexer execution failed**: Indexing errors

### Query Monitoring

```bash
# Enable diagnostic logging
az monitor diagnostic-settings create \
  --resource <search-resource-id> \
  --name "Search-Diagnostics" \
  --workspace <log-analytics-workspace-id> \
  --logs '[
    {
      "category": "OperationLogs",
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

### Query Logs with KQL

```kusto
// Search query performance
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.SEARCH"
| where OperationName == "Query.Search"
| where TimeGenerated > ago(1h)
| project TimeGenerated, DurationMs, ResultCount, Query_s
| order by DurationMs desc

// Slow queries (>1 second)
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.SEARCH"
| where OperationName == "Query.Search"
| where DurationMs > 1000
| summarize Count = count(), AvgDuration = avg(DurationMs) by Query_s
| order by Count desc

// Throttled queries
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.SEARCH"
| where OperationName == "Query.Search"
| where HttpStatusCode_d == 503
| summarize Count = count() by bin(TimeGenerated, 5m)
| render timechart
```

### Indexer Monitoring

```kusto
// Indexer execution status
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.SEARCH"
| where OperationName == "Indexers.Status"
| where TimeGenerated > ago(24h)
| project TimeGenerated, IndexerName_s, Status_s, ErrorMessage_s
| order by TimeGenerated desc

// Indexer failures
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.SEARCH"
| where OperationName == "Indexers.Status"
| where Status_s == "Failed"
| summarize Count = count() by IndexerName_s, ErrorMessage_s
| order by Count desc
```

---

## Index Management

### List Indexes

**Using REST API:**
```bash
# Get indexes (using admin key)
curl -X GET \
  "https://<search-service-name>.search.windows.net/indexes?api-version=2024-07-01" \
  -H "api-key: <admin-key>"
```

**Using Azure Portal:**
1. Navigate to Search service
2. Select **Indexes**
3. View list of indexes with document counts and storage

### Index Statistics

```bash
# Get index statistics via REST API
curl -X GET \
  "https://<search-service-name>.search.windows.net/indexes/<index-name>/stats?api-version=2024-07-01" \
  -H "api-key: <admin-key>"
```

**Response includes:**
- Document count
- Storage size
- Number of vector indexes (if applicable)

### Delete Documents

```bash
# Delete documents from index (using REST API)
curl -X POST \
  "https://<search-service-name>.search.windows.net/indexes/<index-name>/docs/index?api-version=2024-07-01" \
  -H "Content-Type: application/json" \
  -H "api-key: <admin-key>" \
  -d '{
    "value": [
      { "@search.action": "delete", "id": "doc-id-1" },
      { "@search.action": "delete", "id": "doc-id-2" }
    ]
  }'
```

### Rebuild Index

**When to rebuild:**
- Schema changes (adding/removing fields)
- Analyzer changes
- Scoring profile modifications
- Major data quality issues

**Process:**
1. Create new index with updated schema
2. Configure indexer to target new index
3. Run indexer to populate new index
4. Validate new index
5. Update application to use new index
6. Delete old index

**Note**: Rebuilding can take significant time for large indexes.

---

## Query Performance

### Analyze Query Performance

```kusto
// Query performance by time of day
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.SEARCH"
| where OperationName == "Query.Search"
| where TimeGenerated > ago(7d)
| summarize AvgDuration = avg(DurationMs), 
    Count = count() by bin(TimeGenerated, 1h)
| render timechart

// Most expensive queries
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.SEARCH"
| where OperationName == "Query.Search"
| where TimeGenerated > ago(24h)
| summarize AvgDuration = avg(DurationMs), 
    Count = count() by Query_s
| where Count > 10
| order by AvgDuration desc
| take 20
```

### Query Optimization Tips

1. **Use filters before search**: Filters are faster than full-text search
2. **Limit fields returned**: Use `$select` parameter
3. **Set appropriate top**: Don't request more results than needed
4. **Use search mode**: `any` vs `all` affects performance
5. **Enable caching**: Configure `queryPlan` caching
6. **Avoid complex scoring**: Custom scoring profiles add overhead

### Example Optimized Query

```bash
# Optimized query with filters and field selection
curl -X POST \
  "https://<search-service-name>.search.windows.net/indexes/<index-name>/docs/search?api-version=2024-07-01" \
  -H "Content-Type: application/json" \
  -H "api-key: <query-key>" \
  -d '{
    "search": "AI platform",
    "filter": "status eq '\''active'\'' and category eq '\''documentation'\''",
    "select": "id,title,content,lastModified",
    "top": 50,
    "searchMode": "all"
  }'
```

---

## Indexer Operations

### List Indexers

```bash
# Get indexers via REST API
curl -X GET \
  "https://<search-service-name>.search.windows.net/indexers?api-version=2024-07-01" \
  -H "api-key: <admin-key>"
```

### Run Indexer

```bash
# Trigger indexer run
curl -X POST \
  "https://<search-service-name>.search.windows.net/indexers/<indexer-name>/run?api-version=2024-07-01" \
  -H "api-key: <admin-key>"
```

### Check Indexer Status

```bash
# Get indexer status
curl -X GET \
  "https://<search-service-name>.search.windows.net/indexers/<indexer-name>/status?api-version=2024-07-01" \
  -H "api-key: <admin-key>"
```

**Status values:**
- **success**: Completed successfully
- **inProgress**: Currently running
- **transientFailure**: Temporary error, will retry
- **persistentFailure**: Requires intervention
- **reset**: Indexer was reset

### Reset Indexer

Reset when indexer is stuck or needs full re-index:

```bash
# Reset indexer
curl -X POST \
  "https://<search-service-name>.search.windows.net/indexers/<indexer-name>/reset?api-version=2024-07-01" \
  -H "api-key: <admin-key>"

# Run after reset
curl -X POST \
  "https://<search-service-name>.search.windows.net/indexers/<indexer-name>/run?api-version=2024-07-01" \
  -H "api-key: <admin-key>"
```

### Indexer Schedules

**Configure schedule:**
```bash
# Update indexer with schedule (via REST API)
# Schedule format: ISO 8601 duration
# Example: Run every 30 minutes
curl -X PUT \
  "https://<search-service-name>.search.windows.net/indexers/<indexer-name>?api-version=2024-07-01" \
  -H "Content-Type: application/json" \
  -H "api-key: <admin-key>" \
  -d '{
    "name": "<indexer-name>",
    "schedule": {
      "interval": "PT30M",
      "startTime": "2026-01-23T00:00:00Z"
    }
  }'
```

---

## Capacity Planning

### Service Limits by Tier

| Tier | Max Indexes | Max Indexers | Max Storage | Max QPS |
|------|-------------|--------------|-------------|---------|
| Free | 3 | 3 | 50 MB | Variable |
| Basic | 15 | 15 | 2 GB | ~3 |
| S1 | 50 | 50 | 25 GB | ~15 |
| S2 | 200 | 200 | 100 GB | ~60 |
| S3 | 200+ | 200+ | 200 GB/partition | ~60/partition |

### Monitor Capacity

```kusto
// Storage usage trend
AzureMetrics
| where ResourceProvider == "MICROSOFT.SEARCH"
| where MetricName == "StorageUsed"
| where TimeGenerated > ago(30d)
| summarize AvgStorage = avg(Average) by bin(TimeGenerated, 1d)
| render timechart

// Query load trend
AzureMetrics
| where ResourceProvider == "MICROSOFT.SEARCH"
| where MetricName == "SearchQueriesPerSecond"
| where TimeGenerated > ago(7d)
| summarize AvgQPS = avg(Average), MaxQPS = max(Maximum) 
    by bin(TimeGenerated, 1h)
| render timechart
```

### Scale Search Service

```bash
# Scale replicas (for query performance)
az search service update \
  --resource-group <rg-name> \
  --name <search-service-name> \
  --replica-count 3

# Scale partitions (for storage and indexing)
az search service update \
  --resource-group <rg-name> \
  --name <search-service-name> \
  --partition-count 2
```

**Replicas vs Partitions:**
- **Replicas**: Improve query performance and availability
- **Partitions**: Increase storage capacity and indexing throughput

### Capacity Planning Guidelines

**When to add replicas:**
- Query latency increasing
- Queries per second approaching limits
- Need high availability (minimum 2 for SLA)

**When to add partitions:**
- Storage usage > 80%
- Large index sizes
- Need faster indexing throughput

**Cost optimization:**
- Start with minimum configuration
- Monitor metrics for 2-4 weeks
- Scale based on actual usage patterns
- Consider reserved capacity for predictable workloads

---

## Security Configuration

### Network Security

**Enable Private Endpoint:**
```bash
# Create private endpoint for search service
az network private-endpoint create \
  --resource-group <rg-name> \
  --name <pe-name> \
  --vnet-name <vnet-name> \
  --subnet <subnet-name> \
  --private-connection-resource-id <search-resource-id> \
  --group-id searchService \
  --connection-name <connection-name>
```

**Configure Firewall:**
```bash
# Allow specific IP ranges
az search service update \
  --resource-group <rg-name> \
  --name <search-service-name> \
  --ip-rules "203.0.113.0/24" "198.51.100.0/24"
```

### Managed Identity

**Enable system-assigned identity:**
```bash
az search service update \
  --resource-group <rg-name> \
  --name <search-service-name> \
  --identity-type SystemAssigned
```

**Use managed identity for data sources:**
- Blob Storage connections
- SQL Database connections
- Key Vault access for customer-managed keys

### API Key Management

**Regenerate admin keys:**
```bash
# Regenerate primary admin key
az search admin-key renew \
  --resource-group <rg-name> \
  --service-name <search-service-name> \
  --key-kind primary

# Regenerate secondary admin key
az search admin-key renew \
  --resource-group <rg-name> \
  --service-name <search-service-name> \
  --key-kind secondary
```

**Query key best practices:**
- Use query keys (not admin keys) for application queries
- Create separate query keys per application/environment
- Rotate keys regularly
- Delete unused keys

```bash
# Create query key
az search query-key create \
  --resource-group <rg-name> \
  --service-name <search-service-name> \
  --name "app-production"

# List query keys
az search query-key list \
  --resource-group <rg-name> \
  --service-name <search-service-name>

# Delete query key
az search query-key delete \
  --resource-group <rg-name> \
  --service-name <search-service-name> \
  --key-value <key-value>
```

---

## Troubleshooting

### High Query Latency

**Diagnose:**
```kusto
// Identify slow queries
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.SEARCH"
| where OperationName == "Query.Search"
| where DurationMs > 1000
| summarize Count = count(), AvgDuration = avg(DurationMs), 
    MaxDuration = max(DurationMs) by Query_s
| order by AvgDuration desc
```

**Common causes:**
- Insufficient replicas
- Complex queries
- Large result sets
- Heavy indexing activity
- Resource contention

**Solutions:**
- Add replicas for query workload
- Optimize queries (filters, field selection)
- Use pagination for large result sets
- Schedule heavy indexing during off-peak
- Consider higher tier if at capacity

### Indexer Failures

**Check indexer status:**
```bash
curl -X GET \
  "https://<search-service-name>.search.windows.net/indexers/<indexer-name>/status?api-version=2024-07-01" \
  -H "api-key: <admin-key>" | jq '.lastResult.errors'
```

**Common errors:**
- **Data source connection failed**: Check credentials, firewall
- **Document parsing error**: Invalid document format
- **Skill execution failed**: AI service error or quota
- **Indexing quota exceeded**: Reached indexing rate limit

**Solutions:**
- Verify data source connection strings
- Check network connectivity (private endpoints, firewalls)
- Validate document formats
- Review error messages in indexer status
- Reset and re-run indexer if needed

### Throttling (503 Errors)

**Detect throttling:**
```kusto
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.SEARCH"
| where HttpStatusCode_d == 503
| summarize Count = count() by bin(TimeGenerated, 5m)
| render timechart
```

**Causes:**
- Query rate exceeds QPS limit
- Indexing rate too high
- Insufficient replicas/partitions

**Solutions:**
- Implement retry logic with exponential backoff
- Add replicas to increase QPS limit
- Throttle application query rate
- Batch indexing operations
- Consider higher service tier

### Storage Limits

**Check storage usage:**
```kusto
AzureMetrics
| where ResourceProvider == "MICROSOFT.SEARCH"
| where MetricName == "StorageUsed"
| where TimeGenerated > ago(1h)
| summarize CurrentStorage = max(Maximum)
```

**Solutions:**
- Add partitions to increase storage
- Archive or delete old documents
- Optimize document size (remove unnecessary fields)
- Use separate indexes for different data sets
- Upgrade to storage-optimized tier (L1/L2)

---

## Best Practices

### Index Design
- Keep indexes focused on specific use cases
- Minimize field count in index schema
- Use appropriate field types
- Set retrievable=false for fields not needed in results
- Use analyzers appropriate for content language

### Performance
- Add replicas for query performance
- Add partitions for storage and indexing
- Monitor and optimize slow queries
- Use filters before full-text search
- Implement result caching in application

### Security
- Use query keys for application access
- Enable private endpoints for production
- Rotate API keys regularly
- Use managed identities for data sources
- Enable diagnostic logging
- Implement least-privilege access

### Operations
- Monitor key metrics daily
- Set up alerts for throttling and failures
- Schedule indexers during off-peak hours
- Test disaster recovery procedures
- Document index schemas and mappings
- Plan capacity based on growth trends

---

## Additional Resources

- [Monitor Search Queries](https://learn.microsoft.com/en-us/azure/search/search-monitor-queries)
- [Monitor Indexers](https://learn.microsoft.com/en-us/azure/search/search-monitor-indexers)
- [Capacity Planning](https://learn.microsoft.com/en-us/azure/search/search-capacity-planning)
- [Security Overview](https://learn.microsoft.com/en-us/azure/search/search-security-overview)
- [Service Limits](https://learn.microsoft.com/en-us/azure/search/search-limits-quotas-capacity)

---

*Last Updated: January 2026*
