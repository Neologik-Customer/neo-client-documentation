# Neologik Platform - Support Handbook

This handbook describes how support for a Neologik platform environment works:
how to raise a request, when we are available, how quickly we respond, how
incidents are escalated, what Neologik owns, and how releases and changes are
managed. It is deliberately simple. Support is delivered by email; there is no
customer facing ticketing portal.

Related documents: [Operational Procedures](../operational-procedures/Operational-Procedures.md)
(who operates what), [Runbooks](../runbooks/Runbooks.md) (Neologik's incident
procedures), [Disaster Recovery Plan](../disaster-recovery/Disaster-Recovery-Plan.md),
[Platform Architecture](../architecture/Platform-Architecture.md).

---

## 1. How to get support

All support requests, incidents and change requests are raised by email to
**support@neologik.ai**. Every email to that address is logged by Neologik as a
ticket and acknowledged within the response target for its severity (section 3).

Include the following in every request so it can be actioned without a
round trip:

| Field | Example |
|---|---|
| Environment | production, test |
| Service or agent affected | NCE web tool, Teams agent name, document ingestion |
| What happened | error text, unexpected behaviour, missing result |
| When it happened | date and time, UK time |
| Who is affected | one user, a team, everyone |
| Evidence | screenshots, the message that failed, document names |
| Steps already taken | retried, signed out and in, checked with another user |
| Your reference | your own service desk ticket number, if any |

Requests without an environment and a description of impact are acknowledged
but cannot be prioritised until that information is supplied.

### Support contact

**support@neologik.ai** is the single point of contact for all support:
incidents, requests and changes. Neologik does not publish individual names or
addresses; the mailbox is monitored throughout support hours and routes to the
responsible engineer. Escalation is described in section 4.

Store a copy of this handbook outside the platform so it is available during an
outage.

---

## 2. Support hours

| | |
|---|---|
| Standard support hours | Monday to Friday, 09:00 to 17:30 UK time, excluding English bank holidays |
| Extended support | Not offered at this stage |
| Out of hours | Not offered at this stage. Requests received outside standard hours are picked up at the start of the next business day |
| Emergency cover | Within standard hours only. See section 3 for major incident handling |

All response and resolution targets in this handbook are measured in business
hours within the standard support hours above.

---

## 3. Incident severity, response targets and major incidents

### Severity definitions

| Severity | Definition | Examples |
|---|---|---|
| **Sev1** | Customer facing outage or data risk. The platform or an agent is unusable for all users of an environment, or data is at risk | Agent does not respond in Teams for anyone; NCE web tool unreachable; sign in fails for all users; TLS certificate expired |
| **Sev2** | Degraded or single service. A function is unavailable or materially degraded but users can still work | Document ingestion stuck; one agent slow or erroring intermittently; one integration failing |
| **Sev3** | Minor or cosmetic. Questions, how to requests, small defects with a workaround | Formatting issue in a response; a settings question; a request to add a user group |

Neologik assigns the severity on receipt and confirms it in the acknowledgement.
Where the customer disagrees, the customer's view of business impact prevails.

### Response and resolution targets

| Severity | Acknowledgement | Progress updates | Resolution target |
|---|---|---|---|
| Sev1 | 2 business hours | Every 2 business hours until resolved or a workaround is in place | 1 business day (workaround) |
| Sev2 | 1 business day | Daily | 3 business days |
| Sev3 | 3 business days | On change of status | Next scheduled release, or as agreed |

Acknowledgement and update targets are commitments. Resolution targets are
targets: the platform depends on Microsoft Azure services and AI model
providers, and resolution of issues originating there follows the provider's
timescales. Neologik still owns the ticket and the communication throughout.

### Major incidents

A major incident is a Sev1 affecting a whole environment, or a defect affecting
multiple customer environments. Handling:

1. Neologik declares the major incident and names a single incident lead.
2. The incident lead emails the customer's nominated contact with the
   declaration, the known impact and the next update time.
3. Updates are sent every 2 business hours until service is restored.
4. Recovery follows Neologik's [Runbooks](../runbooks/Runbooks.md) and, where
   data or infrastructure must be restored, the
   [Disaster Recovery Plan](../disaster-recovery/Disaster-Recovery-Plan.md).
   Production restores are run with the customer's authorisation.
5. Within 5 business days of closure, Neologik sends a short written incident
   summary: what happened, impact, root cause, fix, and any follow up actions.

---

## 4. Escalation

Escalation is by email, in this order. Quote the original request in each step.

| Level | Contact | When |
|---|---|---|
| 1. Support | support@neologik.ai | All incidents, requests and changes (section 1) |
| 2. Technical escalation | Bryan Lloyd, bryan.lloyd@neologik.ai | A Sev1 is not acknowledged within the Sev1 target, or a Sev1 has no workaround after 1 business day, or the support response is otherwise unsatisfactory |
| 3. Management escalation | Victoria Pent, victoria.pent@neologik.ai | Level 2 has not resolved the concern within 1 business day, or dissatisfaction with how an incident is being handled |

Escalation inside Neologik (to the engineer owning the affected service, and
onward to management) is automatic for every Sev1 and is Neologik's
responsibility; the customer does not need to trigger it.

---

## 5. Service scope and ownership

### What Neologik owns

Neologik operates the platform end to end. The customer's operational role is
limited to actions only its own tenant administrators can perform (see
[Operational Procedures](../operational-procedures/Operational-Procedures.md)).

| Area | Neologik | Customer |
|---|---|---|
| Platform operations, day to day running of the application | **Owns** | |
| Monitoring, alerting and health | **Owns** | Visibility on request |
| Incident response and platform defects | **Owns** | Reports incidents |
| Platform maintenance, including AKS cluster and node upgrades | **Owns** | |
| Releases, upgrades and hotfixes | **Owns** | Receives release notes |
| Infrastructure and configuration changes | **Owns** | Requests changes |
| Application security patches, platform vulnerabilities, base image and dependency updates | **Owns** | |
| Backups, disaster recovery and restores | **Owns** | Authorises production restores |
| Secret and certificate rotation (execution) | **Owns** | Supplies renewed certificates |
| AI behaviour: system prompts, prompt engineering, model selection and configuration, agent configuration delivered by Neologik, output quality of platform and Neologik built agents | **Owns** | |
| Tenant administration: Entra groups and membership, admin consent, DNS records, TLS certificate supply | Advises | **Owns** |
| Content and data: quality, currency and permissions of source documents and data | Advises | **Owns** |

### AI behaviour

Neologik is responsible for the behaviour of the platform's agents: the prompts,
the model configuration, retrieval settings and the quality of responses. Agent
quality is measured with Neologik's evaluation and red teaming suites before
release (see the platform Testing Strategy). Two limits apply:

- Responses are only as good as the source content. Missing, outdated or
  incorrectly permissioned documents produce poor answers; correcting the
  content is the customer's action, identifying the cause is Neologik's.
- Where the customer edits an agent's own configuration in the NCE tool
  (instructions, knowledge sources, settings), responsibility for the effect of
  those edits is shared: Neologik advises and can restore the delivered
  configuration on request.

### Security fixes

Neologik owns application level security fixes, platform vulnerability
remediation, container base image and dependency updates, and AKS version
upgrades. Critical vulnerabilities are patched as a hotfix; others ride the next
release. The customer remains responsible for the security posture of its own
tenant, network, identities and end user devices.

### Not in scope

The following are outside Neologik support. Neologik will help identify when an
issue lies in one of these areas and hand over with the evidence.

- The customer's Microsoft 365 tenant, Entra ID, Teams and SharePoint service
  availability and configuration
- The customer's network, firewalls, proxies, DNS and end user devices
- Licensing and consumption charges for the customer's Azure subscription and
  Microsoft 365
- Outages or behaviour changes at AI model providers (Microsoft Azure OpenAI,
  Anthropic and others) beyond raising and tracking the provider incident
- Quality, completeness and permissions of the customer's source data and
  documents
- Bespoke applications or integrations not listed in the customer's Statement of
  Work
- Training beyond the knowledge transfer included in the Statement of Work

---

## 6. Releases and maintenance

**Maintenance is continuous.** Neologik deploys releases, hotfixes, security
patches and cluster upgrades as and when appropriate, during standard support
hours, designed for no or minimal user visible interruption. There are no fixed
maintenance windows and no customer approval gate for platform releases.

| Aspect | Practice |
|---|---|
| Planned releases | Platform releases are cut centrally, tested on Neologik's staging environment, then deployed to customer environments. Customer production environments receive tagged releases only |
| Change notification | Release notes are emailed to the customer's nominated contact when a release or hotfix is deployed to production. Releases that change user visible behaviour are notified in advance |
| Testing approach | Every release passes unit tests, dependency and static security analysis, API integration tests, regression tests and AI evaluations on staging before production. Penetration testing and red teaming are run periodically |
| Version management | Releases are versioned `MAJOR.MINOR.PATCH`. Every environment's deployed version is recorded by Neologik and can be quoted on request |
| Rollback | The default response to a failed release is a fast forward fix delivered as a hotfix. Redeploying the previous release is available where the change is reversible; database changes are designed to be backward compatible so this remains possible. Rollback decisions are Neologik's and are communicated to the customer |
| Emergency changes | Hotfixes for Sev1 incidents and critical vulnerabilities are deployed as soon as tested, with notification on deployment |

---

## 7. Change requests

Requests for change (a new agent, a new data source, a configuration change, a
scaling change, a hostname change) are sent to support@neologik.ai like any
other request, stating the change required, the environment and any deadline.

- Changes within the existing service scope are scheduled and confirmed by
  email.
- Changes to scope, or chargeable work, follow the Change Control Process in
  the Statement of Work: Neologik documents the change and its impact on a
  Change Request form, which is approved by an authorised representative of
  both parties before work starts.

---

## 8. Operational acceptance

Before an environment enters support, Neologik follows the checklist below with
the customer. Every item is confirmed for each environment.

- Environment deployed and available on the current release
- Smoke test of agents and the NCE tool passed
- Monitoring alerts routed to Neologik
- Customer nominated contacts recorded by Neologik
- Support handbook issued to the customer
- Recovery objectives (RTO and RPO) agreed
- Tenant side responsibilities confirmed with the customer's administrators
- Environment accepted into support and the date recorded
