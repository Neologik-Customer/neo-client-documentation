# Neologik Platform - Operational Runbooks

## How to use this document

A **runbook** is a set of precise, repeatable steps for a known failure mode.
Each one below follows the same shape: **Symptom / alert -> Severity ->
Diagnosis -> Mitigation -> Verification -> Escalation**. Start with the triage
checklist, then jump to the matching runbook.

**These are the incident-response procedures Neologik carries out** to keep your
environment healthy; they are shared with you for transparency, not as tasks for
your team. Where a step needs resource-level detail, it links down to the
relevant service guide (for example
[AKS Operations](../platform-operations/AKS-Operations.md) or
[Application Gateway & WAF](../platform-operations/Application-Gateway-WAF.md)).
A few steps need an action only your tenant administrators can take (for example
granting admin consent, or providing a certificate) - those are marked
**Customer action**.

Conventions:
- Always confirm the cluster first: `kubectl config current-context`.
- The app label is `app=<service>` (no `neo-` prefix); pod/service names include
  `neo-`. Filter health-probe noise with `grep -vE "GET /healthz|OPTIONS"`.
- Severity: **Sev1** customer-facing outage / data risk; **Sev2** degraded or
  single-service; **Sev3** minor / cosmetic.

### Triage checklist ("a service is broken")

1. **Pod running?** `kubectl get pods -n neologik | grep <service>` - check
   `RESTARTS` and `AGE`. High restarts = crashloop (check `--previous`). See
   [AKS Operations](../platform-operations/AKS-Operations.md).
2. **Startup clean?** First ~50 log lines - Redis / Cosmos / Key Vault
   connection errors.
3. **Traffic arriving?** nginx access lines in the ingress/app pod; non-2xx
   status codes. See [Application Gateway & WAF](../platform-operations/Application-Gateway-WAF.md).
4. **App errors?** Strip healthz, search for `ERROR` / `CRITICAL`.
5. **No stdout error but broken?** [Application Insights](../platform-operations/Application-Insights.md)
   `exceptions` table - it often captures what stdout swallowed. Trace across
   services with `operation_Id`.

---

## RB-01 - Pod crashloop / service will not start

**Symptom:** `kubectl get pods` shows `CrashLoopBackOff` or high `RESTARTS`;
5xx from the service. **Severity:** Sev1 if user-facing (bot, nce-ui, nce-api),
else Sev2.

**Diagnosis**
```bash
kubectl get pods -n neologik -l app=<service>
kubectl logs -n neologik -l app=<service> --previous --tail=100 | grep -vE "GET /healthz|OPTIONS"
kubectl describe pod -n neologik <pod>    # Events: ImagePullBackOff, probe failures, OOMKilled
```
Common causes: (a) bad image tag / `ImagePullBackOff`; (b) failed dependency at
startup (Redis / Cosmos / SQL / Key Vault); (c) probe port mismatch; (d)
`OOMKilled` (needs a higher memory limit). See
[AKS Operations](../platform-operations/AKS-Operations.md) for pod recovery and rollout detail.

**Mitigation**
- Bad deploy -> Neologik redeploys the last-good image (see
  [Operational Procedures](../operational-procedures/Operational-Procedures.md)).
- Dependency down -> Neologik resolves that dependency first (RB-05 / RB-06).
- Config/secret missing -> Neologik confirms the Key Vault secret and the CSI
  mount.

**Verification:** pod `Running` with `0` recent restarts; `/healthz` returns 200;
synthetic request succeeds.

**Escalation:** within Neologik, the owning service team; persistent
`ImagePullBackOff` -> Neologik platform/infra.

---

## RB-02 - Teams bot sign-in failure (`signin/failure`)

**Symptom:** user signs in from Teams, browser closes, nothing happens; or a
sign-in card loops. **Severity:** Sev1 (bot unusable for affected tenant).

> `signin/failure` is opaque - missing admin consent and an invalid OAuth
> connection secret look identical. Work the chain top-down: **TLS -> Bot
> Framework auth -> OAuth connection -> user sign-in -> Redis state.**

**Diagnosis (fastest first)**
1. **Clear stale Redis auth state** - a prior failure can leave a
   `teams_sso_unavailable` flag (24h TTL) that masks whether a real fix worked
   (see [Recovery Procedures](../disaster-recovery/Recovery-Procedures.md) -> clear bot auth state).
   Do this before deeper diagnosis.
2. **Test Connection** in Azure Portal -> Bot resource -> Configuration -> OAuth
   Connection Settings. This calls the token service directly and returns the
   real error: `ServiceError: Login failed` = bad/missing client secret; consent
   error = admin consent not granted.
3. **No POST traffic at all** (only `GET /healthz` in nginx logs) -> TLS cert
   SANs do not include the bot hostname -> Bot Framework rejects TLS. Check the
   served cert (RB-03).

**Mitigation**
- Bad secret -> Neologik rotates the app-registration secret and updates both the
  Key Vault secret and the Bot Service OAuth connection.
- Missing consent -> **Customer action:** your Global Admin grants admin consent
  in the **signing-in user's** home tenant:
  `https://login.microsoftonline.com/<userTenantId>/adminconsent?client_id=<botAppId>`
  (two-parameter form, no `redirect_uri`).
- Missing redirect URI -> the app registration must include
  `https://token.botframework.com/.auth/web/redirect`.

**Verification:** Test Connection succeeds; a fresh Teams sign-in completes and
the bot replies.

**Escalation:** your identity/tenant admin for consent (Customer action); within
Neologik for secret rotation.

---

## RB-03 - Ingress / TLS: no traffic or certificate errors

**Symptom:** browser TLS warning, or bot receives no traffic (only health
probes). **Severity:** Sev1. See
[Application Gateway & WAF](../platform-operations/Application-Gateway-WAF.md) for ingress detail.

**Diagnosis**

The TLS secret name depends on the ingress type: **`keyvault-neo-ingress`** on
NGINX profiles, **`appgw-tls`** on Application Gateway (AGIC/AGC) profiles.
```bash
# what cert is the ingress serving? (use appgw-tls on App Gateway profiles)
kubectl get secret keyvault-neo-ingress -n neologik -o json | \
  python3 -c "import sys,json,base64;from cryptography import x509;from cryptography.hazmat.backends import default_backend;d=json.load(sys.stdin);c=x509.load_pem_x509_certificate(base64.b64decode(d['data']['tls.crt']),default_backend());print('Subject:',c.subject);print('SANs:',[str(n.value) for n in c.extensions.get_extension_for_class(x509.SubjectAlternativeName).value])"
```
If the SANs do not cover the hostname (classic symptom: a self-signed
placeholder cert), the correct cert was never bound.

**Mitigation:** Neologik uploads the correct PFX for the hostname to the central
PFX Key Vault and redeploys ingress. See
[Recovery Procedures](../disaster-recovery/Recovery-Procedures.md) -> restore/replace ingress
certificate.

**Verification:** `openssl s_client -connect <host>:443 -servername <host>` shows
the correct SANs; browser padlock valid.

---

## RB-04 - Ingestion pipeline stuck (documents not appearing)

**Symptom:** uploaded documents never become searchable; queue depth grows.
**Severity:** Sev2 (config/data plane; chat still works on existing indexes).

**Diagnosis**
```bash
kubectl logs -n neologik -l app=neo-ingest --tail=200 | grep -vE "GET /healthz"
kubectl logs -n neologik -l app=neo-file --tail=200 | grep -vE "GET /healthz"
kubectl get pods -n neologik -l app=neo-redis    # broker healthy?
```
Common causes: Redis unavailable (jobs not dequeued); ingest failing on a
specific document (e.g. password-protected PDF) and blocking; Azure OpenAI
throttling (429) on embeddings (RB-07); AI Search index/quota issue (see
[Azure AI Search](../platform-operations/Azure-AI-Search.md)).

**Mitigation:** resolve the dependency; for a poison document, identify it in the
ingest logs and skip/quarantine it. If an index is corrupt, rebuild it
([Recovery Procedures](../disaster-recovery/Recovery-Procedures.md) -> rebuild AI Search index).

**Verification:** queue drains; a test document becomes searchable end-to-end.

---

## RB-05 - Redis unavailable

**Symptom:** bot and ingest errors referencing Redis; async jobs not processed.
**Severity:** Sev1 (bot state + job broker).

**Diagnosis**
```bash
kubectl get pods -n neologik -l app=neo-redis --all-containers
kubectl logs -n neologik -l app=neo-redis --tail=100    # Sentinel failover?
```

**Mitigation:** allow Sentinel to complete failover; if a pod is wedged, delete
it to force reschedule; confirm the PVC bound. Do not flush Redis in production
without explicit authorisation.

**Verification:** `redis-cli -h <sentinel> ping` = `PONG`; bot state reads/writes
succeed.

---

## RB-06 - Database (Azure SQL / Cosmos) connectivity

**Symptom:** nce-api / bot errors on config reads or chat-history writes.
**Severity:** Sev1.

**Diagnosis**
- Private endpoint / DNS: confirm `privatelink.database.windows.net` and
  `privatelink.documents.azure.com` resolve inside the VNet (see
  [Private Endpoints & DNS](../platform-operations/Private-Endpoints-DNS.md)).
- Auth: SQL uses Entra ID (workload identity); confirm the managed identity has
  its DB role. Cosmos uses its data-plane SQL role assignment.

**Mitigation:** fix the missing role assignment or DNS record. Never issue writes
to diagnose - reads only.

**Verification:** a read query from the pod succeeds; app recovers.

---

## RB-07 - Azure OpenAI throttling (HTTP 429 / quota)

**Symptom:** bot slow/failing; ingest embeddings stalling; `429` in logs.
**Severity:** Sev2 (degraded). See
[Azure OpenAI & Foundry](../platform-operations/Azure-OpenAI-Foundry.md).

**Diagnosis:** Application Insights `dependencies` / `traces` for `429` and
`Retry-After`; check the model deployment's TPM quota in Azure OpenAI.

**Mitigation:** confirm client-side retry/backoff is working; request a quota
increase for the deployment; reduce concurrency on ingest if it is starving the
bot. Consider spreading load across model deployments.

**Verification:** 429 rate returns to baseline; latency normalises.

---

## RB-08 - Certificate expiry (proactive)

**Symptom:** TLS cert within 30 days of expiry (alert), or expired (Sev1 outage).

**Mitigation:** **Customer action:** provide the renewed certificate. Neologik
then uploads the new PFX and redeploys ingress. A same-domain wildcard renewal
needs no manifest change.

**Verification:** served cert shows the new `notAfter`.

---

## Related

- [Disaster Recovery Plan](../disaster-recovery/Disaster-Recovery-Plan.md) - when an incident is a disaster.
- [Recovery Procedures](../disaster-recovery/Recovery-Procedures.md) - the concrete restore steps referenced above.
- [Operational Procedures](../operational-procedures/Operational-Procedures.md) - scaling, access, monitoring, and how to request deployments.
- [Azure Monitor & Alerts](../platform-operations/Azure-Monitor-Alerts.md) - where alerts come from.

## Sources

- [NIST SP 800-61 Rev. 3 - Incident Response](https://csrc.nist.gov/pubs/sp/800/61/r3/final)
- [Rootly - Incident response runbooks: templates & guide](https://rootly.com/incident-response/runbooks)
