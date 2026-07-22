# Neologik Platform - Architecture

## 1. Purpose and scope

This document describes the reference architecture of the Neologik AI platform -
the governed AI workflow layer that connects your AI, data and systems inside
your own Azure tenant. It is the entry point for the platform operations
documentation set:

- **Architecture** (this document) - what is deployed and how it fits together.
- [Runbooks](../runbooks/Runbooks.md) - incident-response playbooks.
- [Operational Procedures](../operational-procedures/Operational-Procedures.md) - routine operations.
- [Disaster Recovery Plan](../disaster-recovery/Disaster-Recovery-Plan.md) - strategy, RTO/RPO, scenarios.
- [Recovery Procedures](../disaster-recovery/Recovery-Procedures.md) - step-by-step restores.

Scope is a single environment. Every environment is deployed into **your own
Azure subscription and Entra ID tenant**; Neologik does not host your data
centrally.

## 2. Architecture at a glance

The reference architecture diagram is maintained as a draw.io file alongside this
document: [`neo-reference-architecture.drawio`](./neo-reference-architecture.drawio)
(open at [app.diagrams.net](https://app.diagrams.net)). It has three pages, one
per deployment option (see section 4).

**Placeholder key** (used in the diagram and this document):

| Placeholder | Meaning | Example |
|---|---|---|
| `{org}` | 3-char organisation code | `abc` |
| `{env}` | environment short name | `dev`, `prd` |
| `{e}` | single-letter environment class | `d` (dev), `p` (prod) |
| `<customer-domain>` | your public DNS domain | `contoso.com` |
| region | Azure region (CAF short) | `uks` (UK South) |

## 3. Logical components

The platform is a set of containerised services on AKS plus managed Azure PaaS.
The runtime request flow: operators configure agents, indexes, knowledge and
connections in the **NCE UI** -> **NCE API** (persisted to Azure SQL); documents
arrive via **neo-file** and **neo-ingest-api** (driven asynchronously off
**Redis**) and are chunked, entity-extracted and vector-indexed into **Azure AI
Search**; end users chat through **neo-bot-service** (Teams / web), which reads
agent + index config from SQL, runs hybrid search and calls the **MCP tool
servers**.

| Component | Role | Stack | Container port |
|---|---|---|---|
| neo-nce-ui | Operator web app (agents, indexes, knowledge, connections) | React + Vite | 50053 |
| neo-nce-api | Central config / CRUD API; ingestion control | Python / FastAPI | 50055 |
| neo-bot-service | User-facing Teams / web bot; hybrid search, multi-model | Python / aiohttp + Agents SDK + Semantic Kernel | 50060 |
| neo-ingest-api | Chunk -> entity-extract -> vector-index pipeline | Python / FastAPI | 50068 |
| neo-file | Multi-source file transfer (SharePoint / Blob) | Python / FastAPI | 50057 |
| neo-redis-queue | Redis Streams broker (Sentinel HA) for async jobs | Redis 7 / K8s | - |
| neo-database-sql | Azure SQL schema (metadata, document tracking) | Python / SQL | - |
| neo-mcp-sql | MCP server: natural language -> SQL | Python / FastAPI / MCP | 50062 |
| neo-mcp-document-creation | MCP server: Word doc generation | Python / FastMCP | 50068 |
| neo-mcp-etc | Any bespoke MCP solution (tailored to each client) | Varies | Varies |

Ingress routing and the underlying infrastructure underpin all of the above; the
whole environment is deployed and orchestrated by Neologik (see section 9).

## 4. Deployment options (infrastructure profiles)

Three profiles are supported. The choice is made with you at onboarding based on
your security and cost posture. The private-networking and DNS layout for the
secure profiles is described in
[Private Endpoints & DNS](../platform-operations/Private-Endpoints-DNS.md); per-environment network
address plans are provided to you separately.

| | Option A - Secure + App Gateway | Option B - Secure + NGINX | Option C - Small |
|---|---|---|---|
| Network posture | Private (all PaaS on Private Endpoints) | Private (all PaaS on Private Endpoints) | Public endpoints (firewall + Entra ID / RBAC) |
| Ingress | Application Gateway WAF_v2 (AGIC) | NGINX Ingress Controller (in-cluster) | NGINX (Web App Routing add-on) |
| AKS API server | Private | Private | Public |
| Bastion + jumpbox + runner VM | Yes | Yes | No |
| CI/CD runner | Self-hosted runner VM (in VNet) | Self-hosted runner VM (in VNet) | GitHub-hosted |
| Log retention (typical) | 90 days | 90 days | 30 days |
| Cost profile | Highest | High | Cost-optimised (AKS start/stop) |
| Typical use | Regulated production | Production, no WAF appliance | Dev / PoV / cost-sensitive |

Options A and B share the same secure network layout; they differ only in the
ingress resource. Option C omits private networking entirely.

## 5. Data stores and where state lives

Understanding which components hold state is the foundation of the DR and
recovery model (most of the platform is stateless and redeployable from code).

| Store | Holds | Stateful? | Recovery source |
|---|---|---|---|
| Azure SQL | Agent / index / knowledge / connection config; document tracking metadata | **Yes** | Automated PITR backups |
| Azure Cosmos DB | Bot chat history | **Yes** | Continuous backup (PITR) |
| Azure Storage (x4: files, templates, foundry, output) | Source docs, templates, generated outputs | **Yes** | Soft-delete / versioning; re-transferable from source (SharePoint) |
| Azure AI Search | Vector + keyword indexes | Derived | **Rebuild by re-running ingestion** (no native backup) |
| Azure Key Vault | Secrets, certificates | **Yes** | Soft-delete + purge protection |
| Azure Container Registry | Service images | Derived | Re-push from CI / geo-replication (Premium) |
| AKS | Workloads (stateless pods) | No | Redeploy from infrastructure-as-code |

The critical consequence: **AI Search indexes are not backed up** by Azure and
must be rebuilt from source data after a loss (see
[Recovery Procedures](../disaster-recovery/Recovery-Procedures.md)).

## 6. Identity and access

- **Workload identity (federated to the AKS OIDC issuer)** - one user-assigned
  managed identity per service; pods obtain Azure tokens with no secrets on disk.
  RBAC is scoped per service (Key Vault Secrets User, Storage Blob roles, Azure
  OpenAI User, Search Index Contributor, Cosmos SQL role).
- **Deployment service principal** - resource-group-scoped roles only; no
  subscription-scoped roles. Used by Neologik to deploy and operate the
  environment.
- **End-user auth** - Entra ID. Bot access is gated by per-agent user groups;
  NCE web access is gated by the NCE User Group (app-assigned so the `groups`
  claim is emitted). Admin RBAC is a separate Admin User Group. See
  [Operational Procedures](../operational-procedures/Operational-Procedures.md) for user and access
  management.
- **Secrets** - all in Key Vault; surfaced to pods via the Key Vault Secrets
  Provider (CSI). TLS certificates are pulled from a central PFX vault at ingress
  deploy time.

## 7. Networking and traffic flow

- **Inbound**: user HTTPS (443) -> ingress (App Gateway WAF_v2 / NGINX) -> AKS
  service -> pod. Teams traffic reaches the bot via Azure Bot Service.
- **Outbound**: all egress leaves through a **NAT Gateway** with a consistent
  public IP (allow-listable by your firewalls).
- **PaaS access** (Options A/B): over **Private Link** only; public network
  access is disabled on every PaaS service. Private DNS zones resolve the
  `privatelink.*` records inside the VNet - see
  [Private Endpoints & DNS](../platform-operations/Private-Endpoints-DNS.md).
- **Admin / ops** (Options A/B): via [Azure Bastion](../platform-operations/Bastion.md) -> jumpbox VM
  -> private AKS API; no public management plane.

## 8. Observability

- App stdout -> pod logs **and** Application Insights (OpenTelemetry).
- Azure Monitor: Log Analytics workspace + managed Prometheus (Azure Monitor
  Workspace) + Application Insights per environment.
- `cloud_RoleName` in Application Insights matches the pod's service name
  (`neo-bot`, `neo-nce-api`, ...). See
  [Application Insights](../platform-operations/Application-Insights.md) for query patterns and
  [Azure Monitor & Alerts](../platform-operations/Azure-Monitor-Alerts.md) for alerting.

## 9. Deployment and operations model

- The environment is **deployed, released and operated by Neologik** using
  infrastructure-as-code (Bicep) and Helm, orchestrated centrally. Neologik runs
  the environment day to day - monitoring, incident response, scaling, backups
  and recovery. Your operational role is limited to tenant-side actions (Entra
  permissions and group membership, admin consent, DNS, certificate supply) -
  see [Operational Procedures](../operational-procedures/Operational-Procedures.md).
- Two versioning models sit behind a release: container-image services deploy
  from image tags; infrastructure-style services deploy from git tags. To
  request a deployment, change or rollback, contact Neologik support.
- Secure environments run the pipeline on a self-hosted runner VM inside the
  VNet so it can reach the private control plane; Small environments use
  cloud-hosted runners.

## 10. Change log

| Version | Date | Author | Summary |
|---|---|---|---|
| 0.1 (draft) | 2026-07-20 | Neologik | Initial generic reference architecture derived from the secure / small deployments. |

## Sources

Azure service capabilities cited above were verified against Microsoft Learn
(July 2026):
- [AKS backup and recovery](https://learn.microsoft.com/en-us/azure/architecture/operator-guides/aks/aks-backup-and-recovery)
- [Azure SQL automated backups](https://learn.microsoft.com/en-us/azure/azure-sql/database/automated-backups-overview?view=azuresql)
- [Cosmos DB continuous backup / PITR](https://learn.microsoft.com/en-us/azure/cosmos-db/continuous-backup-restore-introduction)
- [Key Vault recovery (soft-delete / purge protection)](https://learn.microsoft.com/en-us/azure/key-vault/general/key-vault-recovery)
- [Azure AI Search FAQ (no built-in backup)](https://learn.microsoft.com/en-us/azure/search/search-faq-frequently-asked-questions)
