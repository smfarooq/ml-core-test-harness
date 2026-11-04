# RBAC and Permissions

> **Part A – As-Is** (diagram [06a](../diagrams/06a-identity-rbac-as-is.svg)): which roles and permissions the repository actually implements.
> **Part B – Target** (diagram [06b](../diagrams/06b-identity-rbac-target.svg)): **proposed operational roles**. They are not existing Mojaloop roles.
> Status labels are defined in the [README](../README.md#status-labels).

---

## Part A – As-Is

### A1. Does Mojaloop implement native RBAC here?

**No, not in the switch services as deployed by this repository.** Authorization is either absent (test harness) or delegated to external components that are switched off:

| Surface | Who uses it | Authentication | Authorization | Evidence | Status |
| --- | --- | --- | --- | --- | --- |
| FSPIOP APIs (ALS 4002, quoting 3002, ML API adapter 3000) | FSP systems (machine) | None; identity is the `FSPIOP-Source` header | Business rules only: the participant must exist and be active, accounts must exist, limits apply | lab, golden-path tests | Implemented (business rules) |
| Central-ledger admin API (3001) | Hub operator | None | None: anyone on the network can create participants, change limits, record funds in/out (verified) | lab | Configured (open) |
| ALS admin API (4001) | Hub operator | None | None | lab | Configured (open) |
| Central-settlement API (3007, `/v2`) | Hub operator / settlement team | None | None | lab | Configured (open) |
| Finance Portal (9400) | Hub operations staff | `AUTH_ENABLED=false` | Roles UI (`reporting_hub_bop_role_ui`, 9301) **not deployed** | compose, lab | Optional (off) |
| Reporting API (GraphQL, 9401) | Finance Portal | Expects `x-user` from a proxy | Ory Keto participant checks supported, `AUTH_CHECK_PARTICIPANTS: false` | [config](../../../docker/reporting-hub-bop-api-svc/config/default.json) | Optional (off) |
| TTK | Testers | `OAUTH.AUTH_ENABLED: false` | Role names defined (`Application/MTA`, `Application/PTA`, `Internal/everyone`), not enforced | [system_config.json](../../../docker/ml-testing-toolkit/spec_files/system_config.json) | Optional (off) |
| Kubernetes | Platform engineers | Not in repo | Not in repo (no `Role`/`RoleBinding`/`ServiceAccount` manifests) | – | Unverified |
| MySQL | Services | Static users per schema (`central_ledger`, `account_lookup`); reporting API reuses the `central_ledger` user | Schema-level grants from init SQL; root with empty password allowed | `docker/sql-*-init/`, compose | Configured |
| MongoDB, Redis, Kafka | Services | None | None | compose | Configured |
| Secrets | Anyone with repository access | – | Test keys committed in Git; Kubernetes job references `secretRef` only | `docker/secrets/`, [job-setup.yaml](../../../k8s-mojaloop-perf-tuning/job-setup.yaml) | Configured (test-only) |

### A2. Machine roles that do exist (implicit)

| Machine role | How it is recognised | Example |
| --- | --- | --- |
| Hub participant | Participant named `Hub` (`HUB_PARTICIPANT {ID:1, NAME:"Hub"}`) with Hub accounts | [configs/central-ledger.js](../../../docker/config-modifier/configs/central-ledger.js) |
| DFSP participant | Registered participant with currency accounts, limits and callback endpoints | `payerfsp`, `testfsp1`, `banka` (labs) |
| FXP participant | Participant with accounts in two currencies, used in FX flows | `perffxp`, `testfxp1` |
| Ledger account roles | `transferParticipantRoleType` (payer / payee DFSP) and ledger account types (`POSITION`, `SETTLEMENT`, `HUB_MULTILATERAL_SETTLEMENT`, …) | lab DB queries |

These are **business roles inside the ledger**, not access-control roles.

---

## Part B – Proposed operational roles — Proposed

### B1. Roles

| Role | Type | Scope |
| --- | --- | --- |
| Platform Administrator | Human | Owns the Mojaloop deployment (Helm values, GitOps), upgrades |
| Infrastructure Administrator | Human | Clusters, nodes, networks, storage, databases, Kafka |
| Security Administrator | Human | IAM, PKI, keys, secrets, policies, audit review |
| Hub Operations Engineer | Human | Onboards participants, manages endpoints, limits, oracles; first-line incidents |
| Settlement / Finance Operator | Human | Closes windows, runs settlements, records funds in/out, reconciles |
| Developer | Human | Non-production deploys, logs and metrics in non-production |
| FI (DFSP) Administrator | Human (at the FSP) | Views its own transfers and positions, manages its own users and certificates |
| Payment Participant (DFSP system) | Machine | Calls FSPIOP APIs as itself only |
| API Client | Machine | Other integrations (reporting, scheme services) with scoped tokens |
| Service Account | Machine | Kubernetes workloads with least-privilege identities |
| Monitoring & Audit | Human (read-only) | Dashboards, logs, audit trails; no changes |

### B2. Role-permission matrix

Legend: **F** full, **W** write/operate, **R** read, **O** own participant's data only, **–** none. Changes marked † need four-eyes approval.

| Permission ↓ / Role → | Platform Admin | Infra Admin | Security Admin | Hub Ops | Settlement Op | Developer | FI Admin | DFSP system | API Client | Service Account | Monitoring & Audit |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| FSPIOP APIs (`/parties`, `/quotes`, `/transfers`) | – | – | – | – | – | R (non-prod) | – | **O** | – | – | – |
| Participant onboarding, endpoints, oracles | R | – | R | W† | R | – | R (O) | – | – | – | R |
| Limits (NDC) and funds in/out | – | – | R | W† | W† | – | R (O) | – | – | – | R |
| Settlement windows and settlements | – | – | R | R | W† | – | R (O) | – | – | – | R |
| Finance Portal: transfers and positions | R | – | R | R | R | – | O | – | R (scoped) | – | R |
| Kubernetes cluster (`cluster-admin`) | F | F | R | – | – | – | – | – | – | – | – |
| Namespace deploy (GitOps) | W | R | R | – | – | W (non-prod) | – | – | – | – | – |
| Secrets and keys (vault, HSM) | – | – | F | – | – | – | – | – | – | R (own secrets) | – |
| Database direct access | – | W (break-glass) | R | – | – | R (non-prod) | – | – | – | W (own schema) | R (audit views) |
| Kafka administration | – | W | R | – | – | – | – | – | – | W (own topics/groups) | – |
| Monitoring dashboards and logs | R | R | R | R | R | R (non-prod) | – | – | – | – | R |
| Audit trail | R | R | F | R | R | – | R (O) | – | – | – | R |

### B3. How to enforce it

| Layer | Mechanism (examples) |
| --- | --- |
| FSPIOP | Gateway: mTLS certificate + OAuth2 client → bound to `FSPIOP-Source`; JWS validation in the switch |
| Operator APIs and portal | OIDC SSO with MFA → identity-aware proxy → `x-user` → permission checks (Ory Keto is already supported by the reporting API) |
| Kubernetes | RBAC `Role`/`ClusterRole` bound to SSO groups; per-workload `ServiceAccount`s; admission policies |
| Data | Per-service DB users, Kafka ACLs per consumer group and topic, read-only audit views |
| Secrets | Vault policies per role and per service account |
