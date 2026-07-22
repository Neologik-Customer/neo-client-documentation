# Neologik Platform - Disaster Recovery Plan

## 1. Purpose

This plan defines how your Neologik platform environment is recovered after a
disaster - a loss of service, data, or region that ordinary
[runbooks](../runbooks/Runbooks.md) cannot resolve. It sets recovery objectives, assigns
roles, and points to the concrete [Recovery Procedures](./Recovery-Procedures.md).

It complements, and does not replace, your organisation's own business
continuity plan.

## 2. Scope and the core recovery principle

Scope: one environment (its Azure subscription + Entra tenant), as described in
the [Platform Architecture](../architecture/Platform-Architecture.md).

**Core principle: the platform is mostly stateless and defined as code.**
Compute (AKS, all service pods), networking, ingress and identity are
**redeployable from code** by Neologik. Disaster recovery therefore concentrates
on the handful of **stateful stores** and on rebuilding what is derived:

| Tier | Components | Recovery approach | Led by |
|---|---|---|---|
| Redeployable from code | AKS, ingress, VNet, workload identities, service pods | Re-run infrastructure + workload deploys | Neologik |
| Must be restored | Azure SQL, Cosmos DB, Azure Storage, Key Vault | Restore from backup / soft-delete | Neologik (customer authorises) |
| Must be rebuilt | Azure AI Search indexes | Re-run ingestion from source data | Neologik (customer authorises) |
| Re-pushable | Container images | Rebuild from CI, or Premium geo-replication | Neologik |

## 3. Recovery objectives

RTO = maximum acceptable downtime; RPO = maximum acceptable data loss (time).
These are **service-level building blocks**; the environment RTO is the sum of
the critical path (typically: restore data stores in parallel, redeploy
platform, rebuild indexes).

| Asset | Backup mechanism | RPO | RTO (component) | Notes |
|---|---|---|---|---|
| Azure SQL (config, tracking) | Automated PITR (point-in-time restore) | Within the PITR retention window | Within-region restore: minutes-hours | Geo-restore / failover group available as options (section 7) |
| Cosmos DB (chat history) | Continuous backup (PITR), 30-day tier | To the second, within 30 days | Minutes-hours (restore to new account) | Restored to a new account |
| Azure Storage (blobs) | Soft-delete + versioning; GRS optional | Near-zero with versioning | Minutes | Source docs also re-transferable from SharePoint via neo-file |
| Key Vault (secrets/certs) | Soft-delete (90d) + purge protection | Near-zero | Minutes | RBAC assignments NOT auto-restored - recreate |
| AI Search (indexes) | **None native** | = source data currency | Hours (full re-index) | Rebuild by re-running ingestion |
| Container images | Re-push from CI / Premium geo-replication | 0 (source in CI) | Minutes-hours | Latest image tags |
| AKS + workloads | Infrastructure-as-code + Helm | 0 (code) | ~15-30 min infra + service deploys | Full-env deploy ~2-3h from scratch |

**Proposed environment-level targets (to be ratified for your environment):**
- **Production RTO: 4-8 hours** (single-region rebuild + data restore + re-index).
- **Production RPO: <= 1 hour** for config/chat state; AI Search content is only
  as current as the last successful ingestion of source data.

For a lower production RTO/RPO, adopt the enhancements in section 7.

## 4. Roles and responsibilities

| Role | Responsibility |
|---|---|
| Incident Commander (Neologik on-call) | Declares the disaster, owns comms, coordinates recovery |
| Neologik platform / infra engineer | Redeploys infrastructure, ingress, restores Key Vault |
| Neologik data engineer | Restores SQL, Cosmos, Storage; rebuilds AI Search |
| Customer IT contact | DNS changes, Entra consent, tenant-side approvals, restore authorisation |
| Customer business owner | Confirms acceptance criteria and go-live |

Maintain a current contact list (primary + backup) for each role. Store it
outside the platform (so it survives a platform outage).

## 5. Disaster scenarios and response

| Scenario | Classification | Response summary |
|---|---|---|
| Single service failure | Incident, not disaster | [Runbooks](../runbooks/Runbooks.md) |
| Accidental data deletion (SQL / Cosmos) | Disaster (data) | PITR restore to just before the event ([Recovery Procedures](./Recovery-Procedures.md)) |
| AI Search index loss/corruption | Disaster (derived data) | Rebuild index from source ([Recovery Procedures](./Recovery-Procedures.md)) |
| Key Vault deletion | Disaster (secrets) | Recover soft-deleted vault/objects; recreate RBAC |
| Cluster loss / corruption | Disaster (compute) | Redeploy AKS + workloads from code (Neologik) |
| Full region outage | Disaster (region) | Rebuild environment in paired region; restore data via geo-redundant backups |
| Ransomware / tenant compromise | Disaster (security) | Isolate, rotate all secrets, restore from immutable backups; involve your security team |

## 6. Recovery sequence (region rebuild - worst case)

Ordered to respect dependencies (mirrors the deployment order). Infrastructure
and workload redeployment are **led by Neologik**; the customer owns DNS, Entra
consent, and restore authorisation.

1. **Provision infrastructure** in the target region (Neologik): AKS, VNet,
   identities, PaaS resources.
2. **Restore secrets** to Key Vault (recover soft-deleted, or re-seed) and
   **recreate RBAC role assignments** (not restored automatically).
3. **Redis** deploy.
4. **Restore Azure SQL** (geo-restore / failover group) and **Cosmos DB** (PITR)
   - can run in parallel with steps 5-6.
5. **Deploy data-plane services**: ingest-api, file.
6. **Restore Storage** content (or confirm GRS failover / re-transfer from
   source).
7. **Deploy** nce-api, bot-service, nce-ui, mcp-*.
8. **Rebuild AI Search indexes** by re-running ingestion.
9. **Deploy ingress**; pull TLS cert; update **DNS** to the new ingress IP
   (customer DNS action).
10. **Re-grant Entra consent** if app registrations changed (customer Global
    Admin).
11. **Verify** end-to-end (section 8).

## 7. Optional resilience enhancements (cost trade-off)

- **Azure SQL failover group** (RPO 5s / RTO ~1h) instead of geo-restore.
- **Cosmos multi-region** write/read for regional resilience.
- **Storage GRS/RA-GRS** and blob versioning + immutability (ransomware).
- **Container registry Premium geo-replication.**
- **AKS Backup** with a geo-redundant Backup Vault + Cross Region Restore for
  cluster/PV state (paired regions; vault-tier RPO up to ~24h primary / ~36h to
  secondary) - useful where stateful PVs exist.
- **Warm standby** in the paired region for the lowest RTO (highest cost).

## 8. Verification (recovery acceptance)

- [ ] All pods `Running`, health probes green.
- [ ] TLS valid for the hostname; ingress routes.
- [ ] NCE admin portal loads; operator can read config (SQL restored).
- [ ] Bot responds in Teams; chat history reads (Cosmos restored).
- [ ] Document search returns results (AI Search rebuilt).
- [ ] File upload/download works (Storage restored).
- [ ] Application Insights receiving telemetry.
- [ ] Data currency within the agreed RPO.

## 9. Testing and maintenance

- **Tabletop exercise: quarterly** (walk a scenario without touching prod).
- **Technical DR drill: at least annually**; for regulated environments
  (FCA-aligned financial services) run technical restore tests **quarterly**. A
  freshly provisioned, not-yet-live environment is the ideal drill target.
- **After any material architecture change**, re-validate the affected recovery
  procedure.
- Follow the **3-2-1** principle for any backup you control (3 copies, 2 media,
  1 offsite/offtenant).
- Review this plan on every drill and after every real incident; record RTO/RPO
  actually achieved vs target.

## 10. Change log

| Version | Date | Author | Summary |
|---|---|---|---|
| 0.1 (draft) | 2026-07-20 | Neologik | Initial generic DR plan; per-service RTO/RPO from Microsoft Learn. |

## Sources

- [Azure Well-Architected - Disaster recovery](https://learn.microsoft.com/en-us/azure/well-architected/reliability/disaster-recovery)
- [Azure SQL automated backups & geo-restore / failover groups](https://learn.microsoft.com/en-us/azure/azure-sql/database/automated-backups-overview?view=azuresql)
- [Cosmos DB continuous backup / PITR](https://learn.microsoft.com/en-us/azure/cosmos-db/continuous-backup-restore-introduction)
- [Key Vault recovery](https://learn.microsoft.com/en-us/azure/key-vault/general/key-vault-recovery)
- [AKS backup & cross-region restore](https://learn.microsoft.com/en-us/azure/backup/azure-kubernetes-service-backup-overview)
- [Azure AI Search FAQ (rebuild from source)](https://learn.microsoft.com/en-us/azure/search/search-faq-frequently-asked-questions)
