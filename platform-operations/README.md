# Neologik Platform Operations Guide

Welcome to the Neologik Platform Operations documentation. This guide provides practical runbook procedures and operational guidance for managing the Neologik AI platform on Azure.

---

## 📚 Documentation Overview

This operations guide covers the key Azure services and components that power the Neologik platform:

This guide covers the per-service Azure reference. For the cross-cutting
platform docs, see:
- **[Platform Architecture](../architecture/Platform-Architecture.md)** - reference architecture, components, data stores, deployment profiles
- **[Runbooks](../runbooks/Runbooks.md)** - incident-response playbooks (symptom -> diagnosis -> mitigation)
- **[Operational Procedures](../operational-procedures/Operational-Procedures.md)** - routine operations, scaling, access, monitoring
- **[Disaster Recovery Plan](../disaster-recovery/Disaster-Recovery-Plan.md)** - strategy, RTO/RPO, scenarios, roles
- **[Recovery Procedures](../disaster-recovery/Recovery-Procedures.md)** - step-by-step component restores

### Infrastructure & Compute
- **[AKS Operations](./AKS-Operations.md)** - Kubernetes cluster management, upgrades, scaling, and monitoring
- **[Bastion](./Bastion.md)** - Secure VM access and session monitoring

### Networking & Security
- **[Application Gateway & WAF](./Application-Gateway-WAF.md)** - Load balancing, web application firewall, and AGIC
- **[Private Endpoints & DNS](./Private-Endpoints-DNS.md)** - Private Link configuration and DNS integration
- **[RBAC](./RBAC.md)** - Role-based access control and common roles

### Monitoring & Observability
- **[Azure Monitor & Alerts](./Azure-Monitor-Alerts.md)** - Platform monitoring, alerts, and action groups
- **[Application Insights](./Application-Insights.md)** - Application performance monitoring (APM)

### AI & Data Services
- **[Azure AI Search](./Azure-AI-Search.md)** - Search service operations, capacity planning, and security
- **[Azure OpenAI & Foundry](./Azure-OpenAI-Foundry.md)** - AI service networking and configuration

---

## 🎯 About This Guide

This documentation is designed for:
- **Platform Operators**: Day-to-day operations and troubleshooting
- **DevOps Engineers**: Deployment, monitoring, and incident response
- **Site Reliability Engineers**: Service reliability and performance optimization

### Key Principles

1. **Runbook-First**: Focus on actionable procedures and common tasks
2. **Practical Examples**: Real-world scenarios and solutions
3. **Quick Reference**: Fast access to commands and configurations
4. **Best Practices**: Following Azure Well-Architected Framework

---

## 🚀 Quick Start

### Essential Tools

Before operating the platform, ensure you have:

- **Azure CLI**: `az --version` (v2.50+)
- **kubectl**: `kubectl version` (compatible with AKS version)
- **Helm**: `helm version` (v3.x)
- **Azure PowerShell** (optional): `Get-Module -ListAvailable Az`

### Authentication

```bash
# Login to Azure
az login

# Set subscription
az account set --subscription "your-subscription-id"

# Get AKS credentials
az aks get-credentials --resource-group <rg-name> --name <aks-name>

# Verify kubectl access
kubectl get nodes
```

### Common Resource Groups

The Neologik platform typically uses these resource group patterns:
- `rg-neologik-<env>-core` - Core infrastructure
- `rg-neologik-<env>-aks` - AKS cluster resources
- `rg-neologik-<env>-network` - Networking resources
- `rg-neologik-<env>-ai` - AI services (OpenAI, Search)
- `rg-neologik-<env>-monitoring` - Monitoring resources

*Note: Actual naming may vary by deployment*

---

## 📞 Getting Help

If you encounter issues not covered in this guide:

1. **Check Service Health**: [Azure Status](https://status.azure.com/)
2. **Review Alerts**: Check Azure Monitor for active alerts
3. **Check Logs**: Review Application Insights and Log Analytics
4. **Escalate**: Contact the Neologik support team with:
   - Service/component affected
   - Error messages and timestamps
   - Steps already taken

---

## 🔄 Regular Maintenance Tasks

### Daily
- Monitor active alerts in Azure Monitor
- Review Application Insights for errors/exceptions
- Check AKS node health and pod status

### Weekly
- Review capacity metrics (CPU, memory, storage)
- Check backup status and retention
- Review security advisories and CVE reports

### Monthly
- Review access logs and RBAC assignments
- Verify disaster recovery procedures ([backup verification](../operational-procedures/Operational-Procedures.md#op-08---backup-verification-routine))
- Plan AKS cluster upgrades
- Review cost optimization opportunities

### Quarterly
- Test disaster recovery procedures ([DR plan](../disaster-recovery/Disaster-Recovery-Plan.md#9-testing-and-maintenance))
- Review and update runbook documentation
- Security audit and compliance review
- Capacity planning and forecasting

---

## 📖 Additional Resources

- [Neologik Client Documentation](../nce/User-Guide.md) - End-user guide for the NCE UI
- [Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/)
- [Azure Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/)
- [Neologik Platform Website](https://www.neologik.ai/)

---

*Last Updated: July 2026*
