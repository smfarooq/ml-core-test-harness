# Architecture Evidence Register

> Every component and connection drawn in the diagrams, with the evidence behind it. Paths are relative to the repository root.
> **Status labels** ([README](../README.md#status-labels)): **Implemented** – verified in source code (this repo, or the upstream service at the pinned tag) · **Configured** – present in deployment/configuration · **External** – depends on a system outside this repo · **Optional** – supported but not enabled by default · **Proposed** – recommended, not present · **Unverified** – insufficient evidence.
> "✓ runtime" = observed in the lab runs of 2026-11-04.
>
> Diagram keys: 01 enterprise · 02 microservices · 03a/03b infrastructure · 04a/04b resilience & DR · 05a/05b security · 06a/06b identity & RBAC · 07 payment flow · 08 interactions · 09 data · 10 observability.

## 1. Components

| Component | Diagrams | Repository location | Evidence | Status |
| --- | --- | --- | --- | --- |
| ML API Adapter (API + notification handler) | 01, 02, 03a, 07, 08, 10 | `docker-compose.yml` (`ml-api-adapter`), `docker/config-modifier/configs/ml-api-adapter.js` | Image `v16.4.2`; JWS signing on; ✓ runtime | Configured |
| Central Ledger API + handlers | 01, 02, 03a, 07, 08, 09 | `docker-compose.yml` (`central-ledger`), `configs/central-ledger.js` | Image `v19.4.4`; all position actions → `topic-transfer-position-batch`; ✓ runtime | Configured |
| Position batch handler | 02, 03a, 07, 08 | `docker-compose.yml` (`central-handler-position-batch`) | `handler --positionbatch`; ✓ runtime | Configured |
| Prepare / fulfil / get / admin / timeout handlers | 02, 07, 08 | In-process in `central-ledger`; separate in `docker-compose-perf.yml` | Upstream `src/handlers/index.js` flags; ✓ runtime (states observed) | Configured |
| Bulk handlers | 02 | central-ledger (in-process), topics `topic-bulk-*` | No bulk API adapter deployed | Configured (unused) |
| Account Lookup Service | 01, 02, 07, 08 | `docker-compose.yml`, `configs/account-lookup-service.js` | Image `v17.9.1`; ✓ runtime (3204) | Configured |
| ALS MSISDN Oracle | 01, 02, 07, 08, 09 | `docker-compose.yml`, `docker/sql-als-init/02_oracle_msisdn.sql` | Image `v0.0.18`; table `oracleMSISDN` ✓ runtime | Configured |
| Quoting Service + handler | 01, 02, 07, 08 | `docker-compose.yml`, `configs/quoting-service.js` | Image `v17.8.0`; `SIMPLE_ROUTING_MODE: false`; ✓ runtime | Configured |
| Central Settlement | 01, 02, 08, 09 | `docker-compose.yml`, `configs/central-settlement.js` | Image `v17.0.6`; `/v2/health` ✓ runtime; health check port bug | Configured |
| SDK Scheme Adapter (Mojaloop Connector) | 01, 02, 05a, 07, 08 | `docker-compose.yml` (`*-sdk`), `envs/*-sdk.env` | Image `v24.7.0`; ✓ runtime (M9) | Configured |
| FSP core banking / wallet | 01, 07, 08 | `BACKEND_ENDPOINT` in `envs/*-sdk.env` (TTK simulates it) | Not in repo | External |
| Kafka (KRaft, 1 broker) | 01, 02, 03a, 04a, 08, 09, 10 | `docker-compose.yml` (`kafka`), `docker/kafka/scripts/provision.sh` | 23 topics, 1 partition, RF 1; PLAINTEXT | Configured |
| MySQL `mysql` / `mysql-als` | 01, 02, 03a, 04a, 09 | `docker-compose.yml`, `docker/sql-*-init/` | MySQL 8.0; 108 tables in `central_ledger` ✓ runtime; no volume | Configured |
| MongoDB `objstore` | 02, 03a, 09 | `docker-compose.yml` | `mongo:8.0.6`; DBs `mlos`, `ttk`, `reporting` | Configured |
| Redis (hub + per SDK) | 02, 03a, 09 | `docker-compose.yml` | `redis:6.2.4-alpine`; ports published | Configured |
| Finance Portal (events processor, GraphQL API, shell, transfers UI, positions UI, fp-proxy) | 01, 02, 06a, 09, 10 | `docker-compose.yml` (profile `finance-portal`), `docker/reporting-hub-bop-api-svc/`, `docker/fp-proxy/nginx.conf` | ✓ runtime: transfers and positions work | Optional |
| Finance Portal Roles and Settlements UIs | 06a | Shell remote list (ports 9301, 9303) | Not defined in compose; ✓ runtime load error | External |
| Prometheus | 01, 10 | `docker-compose-monitoring.yml`, `docker/prometheus/prometheus.yml` | v2.45.0; 10 s scrape; ✓ runtime (3/73 targets up) | Optional |
| Grafana (+ renderer) | 01, 10 | `docker-compose-monitoring.yml`, `docker/grafana/provisioning/` | 10.0.2; 12 dashboards; ✓ runtime | Optional |
| node-exporter, cAdvisor, kafka-exporter, mysqld-exporters | 10 | `docker-compose-monitoring.yml` | node-exporter fails on Docker Desktop; cAdvisor port clash ✓ runtime | Optional |
| Kafka UIs (Redpanda Console) | 10 | `kafka-debug-ui` (`debug` profile), `kafka-mon-ui` (monitoring) | ✓ runtime | Optional |
| Alertmanager / alert rules | 10 | `docker-compose-monitoring.yml` (commented), `docker/prometheus/alert.rules` (commented), `docker/alertmanager/config.yml` | Present only as commented-out configuration | Unverified (not active) |
| Log aggregation, tracing backend | 10 | – | Not found; Event SDK `LOG`/`TRACE` off | Unverified (absent) |
| Event SDK audit stream | 02, 08, 09, 10 | `docker/event_sdk/config.json` | `AUDIT: kafka` → `topic-event-audit`; ✓ runtime (23 event types) | Configured |
| Testing Toolkit (server, UI, CLI) | 01, 03a, 05a, 06a | `docker-compose.yml`, `docker/ml-testing-toolkit/` | v18.11.2 / v16.2.0 / v1.9.1; ✓ runtime | Configured |
| Legacy simulator, callback-handler simulators | 03a | `docker-compose.yml`, `docker-compose-perf.yml` | Images v12.2.4, v0.1.5 | Configured / Optional |
| k6 load tests + k6-operator runner | 03a, 10 | `packages/k6-tests/`, `Dockerfile`, `docker-compose-load.yml` | Test code in repo | Implemented |
| config-modifier, wait4 | 02, 03a, 04a | `docker/config-modifier/run.js`, `docker/wait4/wait4.js` | Source in repo | Implemented |
| Security harness (mTLS + JWS between SDKs) | 05a | `docker-compose-security.yml`, `envs/payer-sdk-scheme-adapter-api-svc.env`, `docker/security/` | Test keys and certificates | Optional |
| Kubernetes namespace `mojaloop`, Helm release `moja` | 03a | `k8s-mojaloop-perf-tuning/patch.sh` | Deployment names patched with `kubectl` | Configured (external cluster) |
| Namespace `stateful-resources` (Kafka, MongoDB) | 03a | `k8s-mojaloop-perf-tuning/redpanda-app.yaml`, `reporting-service-config.json` | Service DNS names | Configured (external cluster) |
| ArgoCD | 03a, 03b | `redpanda-app.yaml`, `job-setup.yaml`, `job-gp.yaml` | `Application`, `PostSync` hooks | Configured (external cluster) |
| Istio ingress gateway | 03a, 05a, 03b, 05b | `redpanda-service.yaml` | `VirtualService` → `istio-ingress-int/internal-wildcard-gateway` | Configured (external cluster) |
| ConfigMap `config-override`, migration init container | 03a | `config-override.yaml`, `ledger-service.yaml` | rc-file mounts; `initContainers: run-migration` | Configured (external cluster) |
| Mojaloop Helm charts | 03a, 03b | Deployment names only | Charts not in repo | External |
| Settlement bank | 01, 04b | – | Funds in/out recorded via admin API; bank itself not in repo | External |
| Scheme API gateway (mTLS, OAuth2) | 01, 05a, 05b, 06b | SDK OAuth settings (`envs/*-sdk.env`), TTK OAuth block | Clients exist, gateway absent | External |
| OIDC IdP / Keycloak | 05a, 05b, 06a, 06b | TTK `KEYCLOAK` block (`ENABLED: false`) | Disabled | Optional (off) |
| Ory Keto + identity-aware proxy (`x-user`) | 05a, 06a, 05b, 06b | `docker/reporting-hub-bop-api-svc/config/default.json` | `ORY_KETO_READ_URL: ""`, `AUTH_CHECK_PARTICIPANTS: false`, `USER_ID_HEADER: x-user` | Optional (off) |
| Mojaloop Connection Manager | 05a, 05b | TTK `CONNECTION_MANAGER` (`ENABLED: false`); SDK `PM4ML_ENABLED=false` | Disabled | External |
| Transaction Requests Service, bulk API adapter | 07 | Simulator env, golden-path collections | Not deployed; tests skipped ✓ runtime | External |
| CI/CD (CircleCI, Grype, SBOM) | 03a | `.circleci/config.yml`, `.grype.yaml`, `sbom-v2.15.1.csv` | `mojaloop/build` orb on release tags | Configured |

## 2. Connections

| Connection | Diagrams | Protocol / pattern | Evidence | Status |
| --- | --- | --- | --- | --- |
| FSP connector → ALS `/parties`, `/participants` | 01, 07, 08 | HTTP, FSPIOP async (202 + `PUT` callback) | `ALS_ENDPOINT` in `envs/*-sdk.env`; ✓ runtime | Configured |
| ALS → oracle `/participants/{Type}/{ID}` | 02, 07, 08 | HTTP, synchronous | `hub_setup_prod.json` registers MSISDN oracle; ✓ runtime | Configured |
| ALS, quoting, ML API adapter → central-ledger endpoints | 02, 08 | HTTP, synchronous (cached) | `SWITCH_ENDPOINT` / `ENDPOINT_SOURCE_URL` in config overrides | Configured |
| ALS → payee FSP `/parties` (forward) and → payer FSP (callback) | 07, 08 | HTTP, async | ✓ runtime | Configured |
| FSP → quoting-service `/quotes` | 01, 07, 08 | HTTP, async | ✓ runtime | Configured |
| quoting-service ↔ handler via `topic-quotes-*` | 02, 08 | Kafka | `provision.sh`; ✓ runtime | Configured |
| quoting handler → MySQL | 08, 09 | SQL | `configs/quoting-service.js` | Configured |
| quoting handler → FSPs (`POST`/`PUT /quotes`) | 07, 08 | HTTP | ✓ runtime | Configured |
| FSP → ML API adapter `/transfers` | 01, 07, 08 | HTTP, async | ✓ runtime | Configured |
| ML API adapter → `topic-transfer-prepare` / `-fulfil` / `-get` | 02, 07, 08 | Kafka produce | ✓ runtime (messages captured) | Configured |
| prepare / fulfil handlers → `topic-transfer-position-batch` | 02, 07, 08 | Kafka | `configs/central-ledger.js`; ✓ runtime (keys observed) | Configured |
| position batch → `topic-notification-event` | 02, 07, 08 | Kafka | ✓ runtime | Configured |
| notification handler → FSP callbacks (`POST`/`PUT`/`PATCH /transfers`) | 02, 07, 08 | HTTP | ✓ runtime | Configured |
| Handlers → MySQL `central_ledger` | 02, 08, 09 | SQL transactions | ✓ runtime (rows queried) | Configured |
| central-ledger → MongoDB `mlos` | 02, 09 | MongoDB | `configs/central-ledger.js` (`MONGODB` enabled) | Configured |
| Services → Redis proxy cache | 02, 09 | Redis | `PROXY_CACHE` in config overrides | Configured |
| central-settlement ↔ MySQL, Kafka | 02, 08, 09 | SQL, Kafka | `configs/central-settlement.js` | Configured |
| Hub operator → admin APIs (3001, 4001, 3007) | 01, 06a, 08 | HTTP, synchronous, unauthenticated | ✓ runtime | Configured |
| SDK ↔ FSP backend | 01, 07, 08 | HTTP, synchronous (Backend API) | `BACKEND_ENDPOINT`; ✓ runtime (M9) | Configured |
| SDK ↔ its Redis | 02, 09 | Redis | `CACHE_URL` | Configured |
| All services → `topic-event-audit` | 02, 08, 10 | Kafka (Event SDK) | `docker/event_sdk/config.json`; ✓ runtime | Configured |
| `topic-event-audit` → reporting-events-processor → MongoDB `reporting` | 09, 10 | Kafka, MongoDB | `configs/reporting-events-processor-svc.js` | Optional |
| Reporting API → MySQL + MongoDB; portal → reporting API | 09, 10 | SQL, GraphQL over HTTP | reporting API config; ✓ runtime | Optional |
| Prometheus → `/metrics` | 10 | HTTP pull, 10 s | `prometheus.yml` | Optional |
| Switch JWS signature on outbound messages | 05a, 07 | `FSPIOP-Signature` header | `JWS_SIGN: true`; tests assert presence | Configured |

## 3. Proposed elements (target diagrams only)

| Element | Diagrams | Basis | Status |
| --- | --- | --- | --- |
| Multi-zone cluster, node pools, namespaces, HPA, PDBs, storage classes | 03b, 04b | Kubernetes good practice; namespaces aligned with 03a evidence | Proposed |
| Kafka 3 brokers RF 3, MySQL HA cluster, MongoDB replica set, Redis Sentinel | 03b, 04b | Standard HA patterns | Proposed |
| DR site (warm standby), backups, PITR, DNS failover | 04b | Standard DR patterns; RPO/RTO left as decisions | Proposed |
| API gateway with mTLS + OAuth2, WAF, mesh mTLS, NetworkPolicies, HSM, vault, cert-manager, SIEM | 05b | Security good practice; hooks exist in SDK/TTK/reporting configs | Proposed |
| Operational roles and permission matrix | 06b | Operating-model recommendation | Proposed |
| Alerting, log aggregation, tracing backend | 03b | Observability gaps found in 10 | Proposed |
