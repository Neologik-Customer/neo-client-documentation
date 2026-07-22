# Neologik Platform - Operational Procedures

> **STATUS: DRAFT.** Generic operating-model and responsibilities document. No
> customer identifiers. Review with your Neologik contact before it is adopted;
> the exact division of responsibilities is confirmed per engagement.

This document sets out **who operates what** across the life of your Neologik
platform environment - both the day-to-day running of the application and the
rollout of updates and upgrades.

**Operating model.** Neologik runs the platform. We handle the day-to-day
operation of the application and every update and upgrade - the bulk of the
operational work. **Your responsibilities are deliberately small** and limited
to the actions only your own tenant administrators can perform.

## Responsibility summary

| Area | Customer | Neologik |
|---|---|---|
| Initial environment provisioning (onboarding script) | **Runs it** | Prepares script + config, supports |
| Entra permissions / group membership (e.g. adding a developer) | **Owns** | Advises |
| Tenant admin consent for platform apps | **Owns** (Global Admin) | Provides consent URLs |
| DNS records for your domain | **Owns** | Provides target IP |
| TLS certificate supply / renewal | **Provides** | Uploads + binds |
| Application monitoring & health | Visibility | **Owns** |
| Incident response | - | **Owns** ([Runbooks](../runbooks/Runbooks.md)) |
| Scaling (pods / nodes / model capacity) | - | **Owns** |
| Backups, verification, disaster recovery & restores | Authorises | **Owns** ([DR Plan](../disaster-recovery/Disaster-Recovery-Plan.md)) |
| Secret & certificate rotation (execution) | - | **Owns** |
| Deployments, releases, rollbacks | Requests | **Owns** |
| Cluster & component upgrades | - | **Owns** |
| Infrastructure & configuration changes | - | **Owns** |

---

## 1. Customer responsibilities

Your operational role is limited to the following. Everything else is handled by
Neologik (section 2).

### 1.1 Run the onboarding script (one-time, during setup)

Initial provisioning of your environment happens in **your own tenant** and is
run by you, because it requires **Owner** on the subscription and **Global
Administrator** in Entra ID - roles Neologik does not hold.

- Run `Install-NeologikEnvironment.ps1` from the onboarding repository. It
  creates the resource group, security groups, app registrations, Key Vault,
  storage account and managed identities.
- Generate and upload your TLS certificate with `Export-NeologikCertificate.ps1`
  (your private key never leaves your tenant; Neologik never receives it).
- Hand the generated configuration JSON to Neologik so deployment can proceed.

Full step-by-step guidance is provided in your onboarding pack.

### 1.2 Entra permissions and group membership (occasional)

Membership and consent live in your tenant, so these are performed by your
Entra administrators. Typical tasks:

- **Add a new developer or user** - invite the account (if external) and add it
  to the appropriate group(s).
- **Group membership**:
  - the per-agent **bot user group** gates Teams access to that agent;
  - the **NCE User Group** gates the NCE web tool - it must remain **assigned to
    the NCE Auth application** (the sign-in `groups` claim is only emitted for
    app-assigned groups; use direct membership, not nested groups);
  - the **Admin User Group** grants Azure RBAC and is **separate** from the NCE
    User Group - do not conflate them.
- **Grant tenant admin consent** for the bot and NCE applications when asked
  (Global Admin; Neologik supplies the consent URLs).
- **Offboarding** - remove the leaver from all relevant groups.

### 1.3 Tenant-side actions on request (rare)

Only you can change these, so Neologik will ask you to action them during a
hostname change or an environment rebuild:

- **DNS** - create/update the A record for your hostname (Neologik provides the
  ingress IP).
- **TLS certificate** - provide a renewed certificate before expiry (alerting is
  Neologik's; the 30-day warning will come to you as a request).

---

## 2. Neologik responsibilities (the bulk of the work)

Neologik operates the platform end to end. This section is for transparency - no
action is required from you unless noted above.

### 2.1 Day-to-day running of the application

- **Monitoring & health** - pod/cluster health, Application Insights, Log
  Analytics, and Azure Monitor alerting. See
  [Platform Operations](../platform-operations/README.md).
- **Incident response** - triage and resolution against the
  [Runbooks](../runbooks/Runbooks.md).
- **Scaling** - pod replicas / HPA, node pools, and Azure OpenAI model capacity,
  adjusted to load.
- **Backups & disaster recovery** - backup verification, and restore/rebuild per
  the [Disaster Recovery Plan](../disaster-recovery/Disaster-Recovery-Plan.md)
  and [Recovery Procedures](../disaster-recovery/Recovery-Procedures.md)
  (production restores are run with your authorisation).
- **Secret & certificate rotation** - execution and validation of the dependent
  flows.

### 2.2 Rolling out updates and upgrades

- **Deployments & releases** - all service and platform releases run through
  Neologik's CI/CD, with target-environment and build-provenance verification.
- **Rollback / forward-fix** - the default response to a bad release is a
  fast forward-fix (database migrations are immutable and unsafe to roll back).
- **Cluster & component upgrades** - AKS version upgrades, node image updates,
  and dependency upgrades, in planned windows.
- **Infrastructure & configuration changes** - all environment configuration is
  managed and applied by Neologik.

---

## 3. Requesting a change

For anything in section 2 - a deployment, an upgrade, a scaling change, a
hostname change, or to report an incident - contact Neologik support with:

- the affected environment and service,
- the change required (or the incident details, for a fix),
- any deadline or preferred maintenance window.

## Related

- [Runbooks](../runbooks/Runbooks.md) | [Disaster Recovery Plan](../disaster-recovery/Disaster-Recovery-Plan.md) | [Recovery Procedures](../disaster-recovery/Recovery-Procedures.md)
- [Platform Architecture](../architecture/Platform-Architecture.md) | [Platform Operations](../platform-operations/README.md)
