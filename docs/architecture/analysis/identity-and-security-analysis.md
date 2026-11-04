# Identity and Security Analysis

> **Part A – As-Is** (diagram [05a](../diagrams/05a-security-as-is.svg)) lists only what the repository configures or what was observed at runtime.
> **Part B – Target** (diagram [05b](../diagrams/05b-security-target.svg)) is a **Proposed** design. Product names in Part B are examples, not requirements.
> No secret values are reproduced in this document. Status labels are defined in the [README](../README.md#status-labels).

---

## Part A – As-Is

### A1. Summary

The harness is **deliberately insecure** so it is easy to run and test. The switch services themselves implement **protocol-level** controls (validation, duplicate detection, expiry, ILP hash-lock, JWS signing, financial risk limits). **Who** is calling is not authenticated anywhere in the main harness: that job belongs to infrastructure (gateway, mTLS, IAM) that is **external** to this repository.

### A2. Network security

| Control | As-is | Evidence | Status |
| --- | --- | --- | --- |
| Network boundary | One flat Docker bridge network `mojaloop-net`; every container can reach every other | compose | Configured |
| Public vs private endpoints | Switch APIs not published; TTK, MongoDB, Redis and portal ports published on the host | compose | Configured |
| TLS between services | **None**: all internal HTTP is plain | config overrides | Configured |
| Kafka transport | PLAINTEXT listeners, no SASL, no ACLs | `KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP` | Configured |
| Network policies, firewalls, egress control | Not present | – | Unverified (none found) |
| Kubernetes ingress | Internal Istio gateway `istio-ingress-int/internal-wildcard-gateway` (for the Kafka console only) | [redpanda-service.yaml](../../../k8s-mojaloop-perf-tuning/redpanda-service.yaml) | Configured (external cluster) |

### A3. Application security

| Control | As-is | Evidence | Status |
| --- | --- | --- | --- |
| API authentication (FSPIOP) | **None.** Placeholder `Authorization` headers are sent by tests and not checked. | TTK `OAUTH.AUTH_ENABLED: false` | Configured (off) |
| Participant identification | `FSPIOP-Source` must be a registered, active participant (3100 / 3201) | golden-path tests | Implemented (upstream; verified by tests) |
| Header and schema validation | Missing or invalid headers and bodies → 3100/3101/3102; a wrong weekday in `Date` → 400/3101 | negative tests; lab | Implemented (verified at runtime) |
| Duplicate / replay protection | ID + body hash: resend gives the current state again; changed body gives 3106 | lab | Implemented (verified at runtime) |
| Expiry | Timeout handler expires transfers (3303) | lab | Implemented (verified at runtime) |
| Message integrity | ILP condition = base64url(SHA-256(fulfilment)); wrong fulfilment → 3100 → `ABORTED_ERROR` | lab | Implemented (verified at runtime) |
| JWS signing by the switch | **On** for switch-originated messages; key mounted from `docker/secrets/` | [configs/ml-api-adapter.js](../../../docker/config-modifier/configs/ml-api-adapter.js) | Configured |
| JWS signing / validation by FSPs | TTK signs; SDKs do not; nobody validates | TTK env, `envs/*-sdk.env` | Configured (mostly off) |
| Mutual TLS | Only in the security harness (SDK ↔ SDK direct, no switch) | [docker-compose-security.yml](../../../docker-compose-security.yml) | Optional |
| OAuth2 client credentials | SDK supports a token endpoint (`OAUTH_TOKEN_ENDPOINT`, refresh 3600 s); empty | `envs/*-sdk.env` | Optional (off) |
| Financial controls | Net Debit Cap and liquidity checks (4200 / 4001 → `ABORTED_REJECTED`), participant deactivation | lab | Implemented (verified at runtime) |
| Encryption at rest | Not configured | – | Unverified (none found) |
| Secrets management | **Test keys and passwords committed** (switch JWS key, FSP JWS and TLS keys, CA keys, DB passwords, sample `ILP_SECRET`); not reproduced here | `docker/secrets/`, `docker/security/`, `envs/`, compose | Configured (test-only) |
| Certificate management | Static files from `keygen.sh` / `openssl ca` in the security harness | README "SDK Security Overhead Testing" | Configured (test-only) |
| Secure configuration | Containers run as `root`; `MYSQL_ALLOW_EMPTY_PASSWORD=true`; MongoDB and Redis without auth; nginx CORS `*`; some `latest` tags | compose, [nginx.conf](../../../docker/fp-proxy/nginx.conf), `.env` | Configured |
| Vulnerability management | Grype scanning in CI with an ignore list; SBOM published | [.grype.yaml](../../../.grype.yaml), `sbom-v2.15.1.csv` | Configured |

### A4. Identity and access management

| Mechanism | As-is | Evidence | Status |
| --- | --- | --- | --- |
| OAuth 2.0 / OpenID Connect provider | **None deployed** | – | External |
| Keycloak | TTK integration block, `ENABLED: false` | [system_config.json](../../../docker/ml-testing-toolkit/spec_files/system_config.json) | Optional (off) |
| WSO2 API gateway tokens | SDK settings mention WSO2 bearer tokens; endpoint empty | `envs/*-sdk.env` | Optional (off) |
| Ory Keto (permission checks) | Reporting API: `ORY_KETO_READ_URL: ""`, `AUTH_CHECK_PARTICIPANTS: false` | [reporting API config](../../../docker/reporting-hub-bop-api-svc/config/default.json) | Optional (off) |
| Identity-aware proxy | Reporting API reads the user id from header `x-user`, set by an upstream proxy | same | External |
| User authentication (portals) | Finance Portal `AUTH_ENABLED=false`; TTK UI `AUTH_ENABLED=FALSE` | compose | Configured (off) |
| Machine-to-machine | FSPs identified only by headers; SDK OAuth client off | – | Configured (off) |
| Service identities | None (no service accounts, no mesh identity in compose) | – | Unverified |
| Token issuance, expiry, refresh | Only the SDK's client-side refresh setting (3600 s) | `envs/*-sdk.env` | Optional (off) |
| Role enforcement | None in the switch APIs; see [rbac-and-permissions.md](rbac-and-permissions.md) | – | Unverified |

### A5. Consequences

Without mTLS, JWS validation and a gateway, any container on `mojaloop-net` (or any host that can reach a published port) could send FSPIOP messages as any FSP, call the admin API to change limits, or read MongoDB and Redis. This is acceptable for a laptop test harness and **must not** be carried into any shared environment.

---

## Part B – Target security architecture — Proposed

### B1. Zones

| Zone | Contents | Controls |
| --- | --- | --- |
| Participant networks | FSP connectors, FXPs | Private links / VPN, IP allow-lists |
| Edge | DDoS protection, WAF, external L4 load balancer | TLS 1.2+, rate limits |
| API gateway | FSPIOP entry points | **mTLS** with FSP client certificates; **OAuth2 client credentials**; bind certificate identity to `FSPIOP-Source`; schema pre-validation; quotas |
| Switch services (namespace `mojaloop`) | Mojaloop services | **Service mesh mTLS (STRICT)**, default-deny NetworkPolicies, mesh AuthorizationPolicies; **JWS validation of FSP messages** and JWS signing of switch messages |
| Data zone (`stateful-resources`) | Kafka, MySQL, MongoDB, Redis | TLS, SASL/ACLs for Kafka, per-service DB users with least privilege, encryption at rest, no public ports |
| Operator zone | Admin APIs, Finance Portal, settlement | SSO (OIDC) with MFA, identity-aware proxy (sets `x-user`), fine-grained permissions (e.g. Ory Keto), separate internal gateway |
| Management | Kubernetes API, GitOps, CI/CD | Kubernetes RBAC, SSO for `kubectl`/ArgoCD, signed images, admission policies |

### B2. Keys, secrets and certificates

| Asset | Proposal |
| --- | --- |
| Hub JWS signing key | Held in an **HSM** or cloud KMS; rotated with overlapping verification keys |
| FSP JWS public keys and TLS certificates | Exchanged through an onboarding process (e.g. Mojaloop Connection Manager, External); stored in the gateway and service config |
| Internal certificates | Issued and rotated automatically (e.g. cert-manager + internal CA; mesh-issued workload certificates) |
| Secrets (DB passwords, OAuth client secrets) | Secrets manager (e.g. Vault or cloud secret manager) synced by External Secrets; never in Git |
| `ILP_SECRET` | FSP-side secret, unique per FSP, in the FSP's vault |

### B3. Monitoring and assurance

Security events (gateway denials, JWS failures, admin changes) go to a SIEM. Audit events from `topic-event-audit` are retained in tamper-evident storage. Penetration tests and scheme certification run in a pre-production environment.
