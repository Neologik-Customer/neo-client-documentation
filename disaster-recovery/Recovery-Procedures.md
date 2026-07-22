# Neologik Platform - Recovery Procedures

> **STATUS: DRAFT.** Generic, step-by-step recovery procedures for individual
> components. No customer identifiers. Replace `<env>`, `<subId>`, `<rg>`,
> `<server>`, `<account>`, `<vault>` with your environment's values.
> **These procedures change or restore live resources - each carries an explicit
> authorisation gate. Get approval before running any restore against a
> production environment, and never write to application data without it.**

These are the concrete restores referenced by the [Runbooks](../runbooks/Runbooks.md) and
sequenced by the [Disaster Recovery Plan](./Disaster-Recovery-Plan.md). Each
follows: **When to use -> Pre-checks -> Steps -> Verify -> Post-actions.**
Cluster and region rebuilds (RP-06) are **led by Neologik**; the remaining
procedures can be run by your platform operators with authorisation, or jointly
with Neologik support.

## Authorisation gate (read first)

A recovery action is a deliberate, often destructive-adjacent change. Before
running any procedure below:
1. Confirm the incident classification and that recovery (not a runbook) is the
   right response.
2. Get **explicit sign-off** from the Incident Commander and, for production,
   the customer business owner.
3. Prefer restoring to a **new/parallel** resource and cutting over, rather than
   overwriting the original, so the damaged copy remains for forensics.

---

## RP-01 - Restore Azure SQL (point-in-time)

**When:** accidental data deletion/corruption in config or document-tracking
data; SQL loss.

**Pre-checks:** confirm PITR retention covers the target time; identify the exact
timestamp just **before** the corrupting event.

**Steps (restore to a new database - non-destructive):**
```bash
az sql db restore -g <rg> -s <server> -n <sourceDb> \
  --dest-name <sourceDb>-restore --time "<YYYY-MM-DDTHH:MM:SSZ>" --subscription <subId>
```
For a full server/region loss, use **geo-restore** (from geo-redundant backup) or
fail over the **failover group** if configured.

**Verify:** query the restored DB for the expected pre-incident state.

**Post-actions:** validate with the data owner, then cut the app over
(reconfigure connection / rename) under change control - coordinate with
Neologik. Re-confirm the workload managed identity has its DB role on the
restored database.

---

## RP-02 - Restore Cosmos DB (chat history, PITR)

**When:** chat-history loss/corruption.

**Pre-checks:** account must be on **continuous backup** (7 or 30-day tier);
choose a timestamp within the window (restore is to the second).

**Steps:** restore to a **new account** via Portal / CLI / PowerShell / ARM,
targeting the chosen point in time. (Continuous-backup restore always creates a
new account; you cannot overwrite in place.)

**Verify:** the bot reads recent history from the restored account in a test
conversation.

**Post-actions:** repoint the bot config to the restored account (coordinate with
Neologik); retain the old account until verified.

---

## RP-03 - Rebuild Azure AI Search index (no native backup)

**When:** index deleted/corrupted, or after a search-service loss. Azure AI
Search has **no built-in backup** - the index is rebuilt from source data. See
[Azure AI Search](../platform-operations/Azure-AI-Search.md) for service-level detail.

**Pre-checks:** source documents are intact in Storage / SharePoint; ingestion
services (neo-file, neo-ingest-api) and their dependencies (Redis, Azure OpenAI
embeddings) are healthy.

**Steps:**
1. Recreate the index definition (from platform config; the schema is owned by
   the platform, not hand-authored).
2. Re-run ingestion for the affected knowledge source(s) via the NCE ingestion
   control, so documents are re-chunked, entity-extracted and re-indexed.
3. Monitor `neo-ingest` logs until the queue drains.

**Verify:** document search returns expected results; document counts reconcile
against source.

**Post-actions:** record re-index duration (feeds the DR RTO figure).

---

## RP-04 - Recover Azure Key Vault (secrets / certificates)

**When:** vault or secret/cert deleted.

**Pre-checks:** soft-delete retains deleted objects (default 90 days); purge
protection may block permanent deletion until retention elapses.

**Steps:**
```bash
az keyvault list-deleted --subscription <subId>
az keyvault recover --name <vault> --subscription <subId>              # whole vault
az keyvault secret recover --vault-name <vault> --name <secret>        # a secret
az keyvault certificate recover --vault-name <vault> --name <cert>     # a cert
```

**Verify:** secret/cert values resolve; the Key Vault Secrets Provider (CSI)
mounts them into pods.

**Post-actions:** **recreate RBAC role assignments and Event Grid subscriptions -
they are NOT restored automatically** when a vault is recovered. Re-grant the
deployment SP and workload identities their Key Vault roles (coordinate with
Neologik).

---

## RP-05 - Restore Azure Storage content

**When:** blobs (source files, templates, outputs) deleted/corrupted.

**Steps:**
- **Soft-delete / versioning:** undelete blobs or promote a previous version
  (Portal or `az storage blob`).
- **GRS failover** (if enabled): initiate account failover to the secondary
  region.
- **Re-transfer from source:** for source documents originating in SharePoint,
  re-run the neo-file transfer to repopulate central storage.

**Verify:** file upload/download works; ingestion can read the restored blobs.

**Post-actions:** if content fed AI Search, follow RP-03 to rebuild indexes.

---

## RP-06 - Rebuild AKS cluster / redeploy workloads (Neologik-led)

**When:** cluster loss/corruption, or region rebuild. This is performed by
**Neologik** using infrastructure-as-code; see [AKS Operations](../platform-operations/AKS-Operations.md)
for cluster-level context.

**Steps (Neologik):**
1. Redeploy infrastructure to recreate AKS, VNet, identities.
2. Redeploy workloads in order (redis -> database -> ingest -> file -> nce-api ->
   bot -> nce-ui -> mcp-*).
3. Deploy ingress last; confirm TLS cert pull.

**Verify:** all pods `Running`; health probes green; ingress routes.

**Post-actions:** restore/rebuild data stores (RP-01/02/03/05); update DNS
(RP-08 - customer action); re-grant Entra consent if app registrations changed
(customer Global Admin).

---

## RP-07 - Restore / replace ingress TLS certificate (Neologik-led)

**When:** wrong/placeholder cert served, or cert expired (see
[Runbooks](../runbooks/Runbooks.md) RB-03).

**Steps:**
1. Provide the correct PFX for the hostname to Neologik; Neologik uploads it to
   the central PFX Key Vault.
2. Neologik redeploys ingress to pull and bind the cert.

**Verify:** `openssl s_client -connect <host>:443 -servername <host>` shows the
correct SANs and validity; browser padlock valid; bot receives POST traffic.

---

## RP-08 - Repoint DNS after a rebuild (customer action)

**When:** ingress public IP changed (new cluster/region).

**Pre-checks:** a custom domain is changed by your DNS owner. Neologik will
provide the new ingress IP.

**Steps:**
1. Obtain the new ingress public IP from Neologik.
2. Update the A record: `<host>.<domain>` -> new ingress IP (TTL >= 300s).

**Verify:** DNS resolves to the new IP; site reachable over HTTPS.

---

## RP-09 - Clear bot auth state (Teams SSO recovery)

**When:** [Runbooks](../runbooks/Runbooks.md) RB-02 - a prior failure left stale Redis auth
state that masks whether a fix worked. **This deletes Redis keys - authorisation
required.**

**Steps:** delete the stale keys per `BOT_SHORT_NAME` and user id (Teams
`29:...`):
```
bot:<shortname>:teams_sso_unavailable:<userId>
bot:<shortname>:state:auth:_SignInState:msteams:<userId>
bot:<shortname>:state:auth/msteams/<userId>/GRAPH
```

**Verify:** the user gets a fresh sign-in card and completes sign-in; the bot
replies.

**Post-actions:** if failures persist, the root cause is upstream (consent /
secret / TLS) - return to RB-02.

---

## Post-recovery (all procedures)

- Run the [DR plan verification checklist](./Disaster-Recovery-Plan.md#8-verification-recovery-acceptance).
- Confirm data currency meets the agreed RPO.
- Record what happened, RTO/RPO actually achieved, and any procedure gaps; update
  these procedures and the DR plan.
- For security-related recoveries, rotate all potentially exposed secrets and
  involve your security team.

## Sources

- [Azure SQL - restore from backup](https://learn.microsoft.com/en-us/azure/azure-sql/database/recovery-using-backups?view=azuresql)
- [Cosmos DB - restore continuous backup](https://learn.microsoft.com/en-us/azure/cosmos-db/continuous-backup-restore-introduction)
- [Key Vault recovery (RBAC not auto-restored)](https://learn.microsoft.com/en-us/azure/key-vault/general/key-vault-recovery)
- [Azure AI Search - rebuild an index](https://learn.microsoft.com/en-us/azure/search/search-howto-reindex)
- [Storage account failover](https://learn.microsoft.com/en-us/azure/storage/common/storage-disaster-recovery-guidance)
