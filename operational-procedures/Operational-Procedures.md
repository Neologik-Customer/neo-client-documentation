# Neologik Platform - Operational Procedures

> **STATUS: DRAFT.** Generic routine-operations procedures. No customer
> identifiers. Replace `<env>`, `<subId>`, `<rg>`, `<aks>` with your
> environment's values. Review before use.

Routine, planned operations for a running environment. For failures see
[Runbooks](../runbooks/Runbooks.md); for disasters see
[Recovery Procedures](../disaster-recovery/Recovery-Procedures.md).

**Operating model.** Your team operates the running environment day to day
(monitoring, user and access management, scaling requests, backup verification,
recovery) with Neologik support. **Deployments, releases and environment
configuration changes are performed by Neologik** - see OP-01.

## Golden rules

- **Verify the target environment before any change you make** (e.g. which
  cluster your `kubectl` context points at) - never assume.
- **Least privilege.** Never widen RBAC, open a public endpoint, or relax
  Conditional Access to "make it work" - flag the risk and use the scoped
  alternative. See [RBAC](../platform-operations/RBAC.md).
- **No unauthorised data writes.** Operational tasks are read-only against
  application data unless the change is explicitly approved.

---

## OP-01 - Requesting a deployment, change or rollback

Deployments and releases run through Neologik's CI/CD. To request a change,
deployment, or rollback, contact Neologik support with:
- the affected environment and service,
- the change required (or the incident, for a rollback),
- any deadline / maintenance window.

Neologik verifies the target environment and build provenance and executes the
deployment. Rollback policy is **forward-fix by default** (database migrations
are immutable and rolling them back can corrupt state), so a "rollback" is often
a fast forward-fix.

## OP-02 - Scaling

- **Pods:** replica counts and HPA are set in the workload configuration;
  request changes via Neologik. Confirm resource requests/limits are set.
- **Nodes:** user and system node pools autoscale (typically 1-4). For sustained
  load, the pool max or node SKU is raised by Neologik. See
  [AKS Operations](../platform-operations/AKS-Operations.md).
- **Model capacity:** raise the Azure OpenAI deployment TPM quota (request via
  Azure) rather than adding retries. See
  [Azure OpenAI & Foundry](../platform-operations/Azure-OpenAI-Foundry.md).
- Bot token/replica sizing scales with expected user count.

## OP-03 - TLS certificate rotation

Certificates are held in a central PFX Key Vault and bound at ingress deploy
time.
1. Obtain the renewed PFX for your hostname.
2. Provide it to Neologik, who uploads it and redeploys ingress to pull the new
   cert.
A same-domain wildcard renewal needs no configuration change. Alert at **30 days**
before expiry. See [Runbooks](../runbooks/Runbooks.md) RB-08.

## OP-04 - Secret / app-registration rotation

- All secrets live in Key Vault; never in configuration files or code.
- Rotating a bot/app-registration secret means updating **both** the Key Vault
  secret **and** the consuming configuration (e.g. the Bot Service OAuth
  connection). This is coordinated with Neologik - see [Runbooks](../runbooks/Runbooks.md)
  RB-02.
- After rotation, verify the dependent flow (bot sign-in, API auth) end-to-end.

## OP-05 - Environment hostname change

A hostname change touches DNS, TLS and several workloads. Raise it with Neologik;
you own the DNS + TLS prerequisites (new A record, certificate for the new
hostname), Neologik redeploys the affected workloads.

## OP-06 - User and access management

- **Bot access:** add the user to that agent's user group (Entra). Membership
  gates the Teams app.
- **NCE web access:** add the user to the **NCE User Group** and confirm that
  group is **app-assigned** to the NCE Auth enterprise app (the `groups` claim is
  only emitted for app-assigned groups; direct membership, not nested).
- **Admin/RBAC:** the **Admin User Group** grants Azure RBAC and is separate from
  the NCE User Group - never conflate them. See [RBAC](../platform-operations/RBAC.md).
- Offboarding: remove from all three group types; review any per-user role
  assignments.

## OP-07 - Environment start/stop (cost control, Small profile)

Cost-optimised environments stop AKS out of hours (e.g. 08:00-18:00 Mon-Fri) via
scheduled automation, for roughly a 65% compute saving. Serverless SQL/Cosmos
auto-pause. Confirm the schedule before diagnosing "morning" scheduling/HPA
alerts that are actually the cluster resuming.

## OP-08 - Backup verification (routine)

Monthly, confirm the safety nets are in place (a backup you have never restored
is a hope, not a backup):
- Azure SQL: PITR retention configured; run a test point-in-time restore to a
  throwaway DB.
- Cosmos: continuous backup tier confirmed; test-restore to a new account.
- Key Vault: soft-delete + purge protection **on**.
- Storage: soft-delete + versioning on; GRS if required.
- Record results; feed failures into the
  [DR plan](../disaster-recovery/Disaster-Recovery-Plan.md) testing log.

## OP-09 - Monitoring and health

- Live: `kubectl get pods -n neologik`; tail with `stern <service> -n neologik`.
  See [AKS Operations](../platform-operations/AKS-Operations.md).
- Historical/structured: [Application Insights](../platform-operations/Application-Insights.md) +
  Log Analytics (query by `cloud_RoleName`).
- Alerts and action groups: [Azure Monitor & Alerts](../platform-operations/Azure-Monitor-Alerts.md).
  Triage with the [Runbooks](../runbooks/Runbooks.md).

## Related

- [Runbooks](../runbooks/Runbooks.md) | [Disaster Recovery Plan](../disaster-recovery/Disaster-Recovery-Plan.md) | [Recovery Procedures](../disaster-recovery/Recovery-Procedures.md)
- [Platform Architecture](../architecture/Platform-Architecture.md)
