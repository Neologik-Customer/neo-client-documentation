# Neologik Documentation

Welcome to the central documentation repository for Neologik's AI platform - covering both end-user applications and platform operations.

**Welcome to Neologik!** We're here to help you get the most out of our AI platform. If you have any questions or feedback, please don't hesitate to reach out.

---

## 📚 Documentation Overview

This repository contains two main documentation areas:

### 👥 **For AI Operations**: Neo NCE Application
Learn how to use the Neo NCE UI to manage AI agents, knowledge, and data connections.

### 🔧 **For Platform Operators**: Azure Operations
Runbooks and procedures for operating the Neologik platform infrastructure on Azure.

---

## 👥 Neo NCE (Neologik Context Engine)

**[→ View Complete NCE User Guide](./nce/User-Guide.md)**

Comprehensive documentation for the Neo NCE UI - a modern React-based interface for managing AI agents, knowledge indexes, and data connections.

### Key Topics

- **Getting Started**: Authentication, roles (Reader / Contributor / Admin), navigation
- **AI Agents**: Create and configure agents, sub-agents and workers; models, tools,
  connections and channels
- **Apps & MCP**: Registered applications and the tool servers agents call
- **Knowledge Management**:
  - Indexes for document search, and the environment's ingest model settings
  - Document uploads and ingestion
  - SQL database connections
  - External document connections (Confluence, SharePoint, Blob Storage)
- **Memory**: What agents remember, the Memory tab, and erasing a user's memory
- **Monitoring**: Analytics and the Home platform health panel
- **Administration**: Blueprints, the audit trail and role management (Admin role)
- **Troubleshooting**: Common issues and solutions

**Quick Links:**
- [User Guide](./nce/User-Guide.md)

---

## 🔧 Platform Operations

**[→ View Operations Guide](./platform-operations/README.md)**

Operational documentation for managing the Neologik AI platform on Azure. Designed for platform operators, DevOps engineers, and SREs.

### Platform Reference & Resilience
- **[Platform Architecture](./architecture/Platform-Architecture.md)** - Reference architecture, components, data stores, deployment profiles
- **[Runbooks](./runbooks/Runbooks.md)** - Incident-response playbooks
- **[Operational Procedures](./operational-procedures/Operational-Procedures.md)** - Routine operations, scaling, access, monitoring
- **[Disaster Recovery Plan](./disaster-recovery/Disaster-Recovery-Plan.md)** - Strategy, RTO/RPO, scenarios, roles
- **[Recovery Procedures](./disaster-recovery/Recovery-Procedures.md)** - Step-by-step component restores

### Infrastructure & Compute
- **[AKS Operations](./platform-operations/AKS-Operations.md)** - Kubernetes cluster management, upgrades, scaling, monitoring
- **[Bastion](./platform-operations/Bastion.md)** - Secure VM access and session monitoring

### Networking & Security
- **[Application Gateway & WAF](./platform-operations/Application-Gateway-WAF.md)** - Load balancing, web application firewall, AGIC
- **[Private Endpoints & DNS](./platform-operations/Private-Endpoints-DNS.md)** - Private Link configuration and DNS integration
- **[RBAC](./platform-operations/RBAC.md)** - Role-based access control and permissions

### Monitoring & Observability
- **[Azure Monitor & Alerts](./platform-operations/Azure-Monitor-Alerts.md)** - Platform monitoring, alerts, action groups
- **[Application Insights](./platform-operations/Application-Insights.md)** - Application performance monitoring (APM)

### AI & Data Services
- **[Azure AI Search](./platform-operations/Azure-AI-Search.md)** - Search service operations, capacity planning, security
- **[Azure OpenAI & Foundry](./platform-operations/Azure-OpenAI-Foundry.md)** - AI service networking and configuration

### What's Included
- ✅ Runbook-style procedures for common tasks
- ✅ Troubleshooting guides with diagnostic queries
- ✅ Azure CLI commands and examples
- ✅ KQL queries for log analysis
- ✅ Best practices and recommendations
- ✅ Monitoring and alerting guidance

---

## 🎯 About This Repository

This repository serves as the centralized knowledge base for the Neologik platform, providing:

### For End Users
- Step-by-step user guides
- Feature documentation
- Troubleshooting help
- Best practices for using Neo NCE

### For Platform Operators
- Day-to-day operations runbooks
- Monitoring and alerting procedures
- Incident response guides
- Capacity planning guidance
- Security and compliance documentation

