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

**[→ View Complete NCE User Guide](./neo-nce/User-Guide.md)**

Comprehensive documentation for the Neo NCE UI - a modern React-based interface for managing AI agents, knowledge indexes, and data connections.

### Key Topics

- **Getting Started**: Authentication, navigation, dashboard overview
- **AI Agents**: Create and configure main agents and specialized subagents
- **Plugins & Functions**: Extend agent capabilities with custom functions
- **Knowledge Management**: 
  - Indexes for document search
  - Document uploads and ingestion
  - SQL database connections
  - External document connections (Confluence, SharePoint, Blob Storage)
- **Monitoring**: Agent performance and usage tracking
- **Troubleshooting**: Common issues and solutions

**Quick Links:**
- [User Guide](./neo-nce/User-Guide.md)

---

## 🔧 Platform Operations

**[→ View Operations Guide](./neo-platform-operations/README.md)**

Operational documentation for managing the Neologik AI platform on Azure. Designed for platform operators, DevOps engineers, and SREs.

### Infrastructure & Compute
- **[AKS Operations](./neo-platform-operations/AKS-Operations.md)** - Kubernetes cluster management, upgrades, scaling, monitoring
- **[Bastion](./neo-platform-operations/Bastion.md)** - Secure VM access and session monitoring

### Networking & Security
- **[Application Gateway & WAF](./neo-platform-operations/Application-Gateway-WAF.md)** - Load balancing, web application firewall, AGIC
- **[Private Endpoints & DNS](./neo-platform-operations/Private-Endpoints-DNS.md)** - Private Link configuration and DNS integration
- **[RBAC](./neo-platform-operations/RBAC.md)** - Role-based access control and permissions

### Monitoring & Observability
- **[Azure Monitor & Alerts](./neo-platform-operations/Azure-Monitor-Alerts.md)** - Platform monitoring, alerts, action groups
- **[Application Insights](./neo-platform-operations/Application-Insights.md)** - Application performance monitoring (APM)

### AI & Data Services
- **[Azure AI Search](./neo-platform-operations/Azure-AI-Search.md)** - Search service operations, capacity planning, security
- **[Azure OpenAI & Foundry](./neo-platform-operations/Azure-OpenAI-Foundry.md)** - AI service networking and configuration

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

