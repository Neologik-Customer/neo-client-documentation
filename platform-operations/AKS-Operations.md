# AKS Operations Guide

This guide covers day-to-day operations, monitoring, and maintenance of Azure Kubernetes Service (AKS) clusters running the Neologik platform.

---

## Table of Contents

1. [Cluster Health Monitoring](#cluster-health-monitoring)
2. [Node Management](#node-management)
3. [Pod Operations](#pod-operations)
4. [Cluster Autoscaling](#cluster-autoscaling)
5. [Cluster Upgrades](#cluster-upgrades)
6. [Monitoring with Container Insights](#monitoring-with-container-insights)
7. [Troubleshooting](#troubleshooting)
8. [Common Runbook Actions](#common-runbook-actions)

---

## Cluster Health Monitoring

### Check Cluster Status

```bash
# Get cluster information
az aks show --resource-group <rg-name> --name <aks-name> --output table

# Check cluster health
az aks show --resource-group <rg-name> --name <aks-name> --query "powerState.code"

# List all nodes
kubectl get nodes -o wide

# Check node conditions
kubectl describe nodes | grep -A 5 "Conditions:"
```

### Monitor Cluster Components

```bash
# Check system pods
kubectl get pods -n kube-system

# Check AGIC (Application Gateway Ingress Controller)
kubectl get pods -n default -l app=ingress-azure

# Check all namespaces
kubectl get pods --all-namespaces

# View cluster events
kubectl get events --all-namespaces --sort-by='.lastTimestamp'
```

### Resource Usage

```bash
# Node resource usage
kubectl top nodes

# Pod resource usage
kubectl top pods --all-namespaces

# Specific namespace
kubectl top pods -n <namespace>
```

---

## Node Management

### View Node Information

```bash
# List nodes with details
kubectl get nodes -o wide

# Describe specific node
kubectl describe node <node-name>

# Check node labels
kubectl get nodes --show-labels

# Check node taints
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

### Node Pools

```bash
# List node pools
az aks nodepool list --resource-group <rg-name> --cluster-name <aks-name> -o table

# Show node pool details
az aks nodepool show --resource-group <rg-name> --cluster-name <aks-name> --name <nodepool-name>

# Scale node pool manually
az aks nodepool scale \
  --resource-group <rg-name> \
  --cluster-name <aks-name> \
  --name <nodepool-name> \
  --node-count <count>
```

### Cordon and Drain Nodes

```bash
# Cordon node (mark unschedulable)
kubectl cordon <node-name>

# Drain node (evict pods safely)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Uncordon node (make schedulable again)
kubectl uncordon <node-name>
```

### Restart Node Pool

```bash
# For VM Scale Set node pools
az vmss restart --resource-group <node-rg> --name <vmss-name>

# Get node resource group
az aks show --resource-group <rg-name> --name <aks-name> --query nodeResourceGroup -o tsv
```

---

## Pod Operations

### View Pods

```bash
# List all pods in namespace
kubectl get pods -n <namespace>

# List all pods with node assignment
kubectl get pods -n <namespace> -o wide

# Watch pods in real-time
kubectl get pods -n <namespace> --watch

# List pods with specific label
kubectl get pods -l app=<app-name> -n <namespace>
```

### Pod Details and Logs

```bash
# Describe pod
kubectl describe pod <pod-name> -n <namespace>

# View pod logs
kubectl logs <pod-name> -n <namespace>

# Follow logs in real-time
kubectl logs -f <pod-name> -n <namespace>

# Previous container logs (for crashed pods)
kubectl logs <pod-name> -n <namespace> --previous

# Logs from specific container in multi-container pod
kubectl logs <pod-name> -c <container-name> -n <namespace>
```

### Restart Pods

```bash
# Delete pod (will be recreated by deployment)
kubectl delete pod <pod-name> -n <namespace>

# Restart deployment (rolling restart)
kubectl rollout restart deployment/<deployment-name> -n <namespace>

# Scale deployment to 0 and back
kubectl scale deployment/<deployment-name> --replicas=0 -n <namespace>
kubectl scale deployment/<deployment-name> --replicas=<original-count> -n <namespace>
```

### Execute Commands in Pods

```bash
# Get shell access
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash

# Run single command
kubectl exec <pod-name> -n <namespace> -- <command>

# For specific container in pod
kubectl exec -it <pod-name> -c <container-name> -n <namespace> -- /bin/bash
```

---

## Cluster Autoscaling

### Cluster Autoscaler Overview

The cluster autoscaler automatically adjusts the number of nodes based on pending pod resource requests.

### Check Autoscaler Status

```bash
# View autoscaler configuration
az aks nodepool show \
  --resource-group <rg-name> \
  --cluster-name <aks-name> \
  --name <nodepool-name> \
  --query "enableAutoScaling"

# Check autoscaler logs
kubectl logs -n kube-system -l app=cluster-autoscaler
```

### Enable Autoscaling

```bash
# Enable autoscaling on existing node pool
az aks nodepool update \
  --resource-group <rg-name> \
  --cluster-name <aks-name> \
  --name <nodepool-name> \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 10
```

### Update Autoscaler Settings

```bash
# Update min/max node count
az aks nodepool update \
  --resource-group <rg-name> \
  --cluster-name <aks-name> \
  --name <nodepool-name> \
  --update-cluster-autoscaler \
  --min-count 3 \
  --max-count 15
```

### Horizontal Pod Autoscaler (HPA)

```bash
# List HPAs
kubectl get hpa --all-namespaces

# Describe HPA
kubectl describe hpa <hpa-name> -n <namespace>

# Create HPA for deployment
kubectl autoscale deployment <deployment-name> \
  --cpu-percent=70 \
  --min=2 \
  --max=10 \
  -n <namespace>
```

---

## Cluster Upgrades

### Check Available Upgrades

```bash
# Get current version
az aks show --resource-group <rg-name> --name <aks-name> --query "kubernetesVersion" -o tsv

# List available upgrades
az aks get-upgrades --resource-group <rg-name> --name <aks-name> -o table
```

### Upgrade Planning

**Best Practices:**
1. Review [AKS release notes](https://github.com/Azure/AKS/releases) for breaking changes
2. Test upgrades in non-production environment first
3. Schedule during maintenance window
4. Ensure backup/snapshot is available
5. Notify stakeholders

### Upgrade Control Plane

```bash
# Upgrade control plane only
az aks upgrade \
  --resource-group <rg-name> \
  --name <aks-name> \
  --kubernetes-version <version> \
  --control-plane-only
```

### Upgrade Node Pool

```bash
# Upgrade specific node pool
az aks nodepool upgrade \
  --resource-group <rg-name> \
  --cluster-name <aks-name> \
  --name <nodepool-name> \
  --kubernetes-version <version>
```

### Full Cluster Upgrade

```bash
# Upgrade both control plane and nodes
az aks upgrade \
  --resource-group <rg-name> \
  --name <aks-name> \
  --kubernetes-version <version>
```

### Monitor Upgrade Progress

```bash
# Check upgrade status
az aks show --resource-group <rg-name> --name <aks-name> --query "provisioningState"

# Watch node status
kubectl get nodes --watch

# Check for issues
kubectl get events --all-namespaces --sort-by='.lastTimestamp'
```

---

## Monitoring with Container Insights

### Enable Container Insights

```bash
# Enable on existing cluster
az aks enable-addons \
  --resource-group <rg-name> \
  --name <aks-name> \
  --addons monitoring \
  --workspace-resource-id <log-analytics-workspace-id>
```

### View Monitoring Data

**Via Azure Portal:**
1. Navigate to AKS cluster
2. Select **Monitoring** → **Insights**
3. Review:
   - Cluster health
   - Nodes performance
   - Controllers performance
   - Containers performance

### Query Logs with KQL

```kusto
// Pod failures in last 24 hours
KubePodInventory
| where TimeGenerated > ago(24h)
| where PodStatus == "Failed"
| project TimeGenerated, Namespace, Name, PodStatus, Computer

// High CPU usage nodes
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "K8SNode"
| where CounterName == "cpuUsageNanoCores"
| summarize AvgCPU = avg(CounterValue) by Computer
| where AvgCPU > 80

// Container restarts
KubePodInventory
| where TimeGenerated > ago(24h)
| where RestartCount > 0
| summarize Restarts = sum(RestartCount) by ContainerName, Namespace
| order by Restarts desc
```

### Configure Alerts

Key metrics to alert on:
- Node CPU > 80%
- Node Memory > 85%
- Pod restart count > threshold
- Failed pods
- Node not ready status

---

## Troubleshooting

### Pod Won't Start

**Check pod status:**
```bash
kubectl describe pod <pod-name> -n <namespace>
```

**Common issues:**
- **ImagePullBackOff**: Check image name, registry credentials
- **CrashLoopBackOff**: Check application logs
- **Pending**: Check resource requests vs. available capacity
- **CreateContainerConfigError**: Check ConfigMaps/Secrets

### Node Issues

```bash
# Check node conditions
kubectl describe node <node-name> | grep -A 10 "Conditions:"

# Check node events
kubectl describe node <node-name> | grep -A 20 "Events:"

# Check kubelet logs (requires SSH or AKS run-command)
az aks command invoke \
  --resource-group <rg-name> \
  --name <aks-name> \
  --command "journalctl -u kubelet -n 100"
```

### Network Issues

```bash
# Test pod-to-pod connectivity
kubectl run -it --rm debug --image=busybox --restart=Never -- sh
# Inside container: ping <other-pod-ip>

# Check service endpoints
kubectl get endpoints -n <namespace>

# Verify DNS
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup <service-name>
```

### Storage Issues

```bash
# List persistent volumes
kubectl get pv

# List persistent volume claims
kubectl get pvc --all-namespaces

# Describe PVC for details
kubectl describe pvc <pvc-name> -n <namespace>
```

---

## Common Runbook Actions

### 1. Restart Stuck Pods

```bash
# Identify stuck pods
kubectl get pods -n <namespace> | grep -E 'CrashLoop|Error|Pending'

# Delete pod to force recreation
kubectl delete pod <pod-name> -n <namespace>

# Or restart entire deployment
kubectl rollout restart deployment/<deployment-name> -n <namespace>
```

### 2. Clear Failed Pods

```bash
# Delete all failed pods in namespace
kubectl delete pods --field-selector status.phase=Failed -n <namespace>

# Delete all evicted pods
kubectl get pods -n <namespace> | grep Evicted | awk '{print $1}' | xargs kubectl delete pod -n <namespace>
```

### 3. Check Resource Pressure

```bash
# Check node pressure
kubectl describe nodes | grep -A 5 "Allocated resources:"

# Check which pods are using most resources
kubectl top pods --all-namespaces --sort-by=memory
kubectl top pods --all-namespaces --sort-by=cpu
```

### 4. Scale Application

```bash
# Scale up
kubectl scale deployment/<deployment-name> --replicas=5 -n <namespace>

# Verify scaling
kubectl get deployment/<deployment-name> -n <namespace>
kubectl get pods -n <namespace> -l app=<app-name>
```

### 5. View Recent Events

```bash
# Last 20 events across cluster
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -20

# Warning events only
kubectl get events --field-selector type=Warning --all-namespaces
```

### 6. Diagnose Deployment Issues

```bash
# Check deployment status
kubectl rollout status deployment/<deployment-name> -n <namespace>

# View deployment history
kubectl rollout history deployment/<deployment-name> -n <namespace>

# Rollback to previous version
kubectl rollout undo deployment/<deployment-name> -n <namespace>
```

### 7. Update AKS Credentials

```bash
# Refresh AKS credentials
az aks get-credentials --resource-group <rg-name> --name <aks-name> --overwrite-existing

# Verify access
kubectl get nodes
```

---

## Best Practices

### Operations
- Use namespaces to organize workloads
- Label resources consistently for easier filtering
- Implement resource requests and limits on all pods
- Enable cluster autoscaling for production workloads
- Regularly review and clean up unused resources

### Upgrades
- Stay within N-2 supported versions
- Upgrade control plane before node pools
- Test upgrades in dev/staging environments
- Schedule upgrades during maintenance windows
- Monitor cluster health during and after upgrades

### Monitoring
- Enable Container Insights on all clusters
- Configure alerts for critical metrics
- Regularly review logs and metrics
- Set up log retention policies
- Monitor resource utilization trends

### Security
- Use Azure Policy for governance
- Enable RBAC and Azure AD integration
- Regularly update node images
- Scan container images for vulnerabilities
- Use network policies for pod-to-pod communication

---

## Additional Resources

- [AKS Day-2 Operations Guide](https://learn.microsoft.com/en-us/azure/architecture/operator-guides/aks/day-2-operations-guide)
- [AKS Upgrade Best Practices](https://learn.microsoft.com/en-us/azure/architecture/operator-guides/aks/aks-upgrade-practices)
- [Monitor AKS](https://learn.microsoft.com/en-us/azure/aks/monitor-aks)
- [Cluster Autoscaler](https://learn.microsoft.com/en-us/azure/aks/cluster-autoscaler)

---

*Last Updated: January 2026*
