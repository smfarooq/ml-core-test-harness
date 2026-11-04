# Technology Stack

> Versions are the image tags pinned in [`.env`](../../../.env) at the analysed commit. Status labels are defined in the [README](../README.md#status-labels).

## 1. Switch services (pre-built Mojaloop images)

| Layer | Technology | Version | Used by | Status |
| --- | --- | --- | --- | --- |
| Runtime | Node.js | per image (repo tooling pins Node via [.nvmrc](../../../.nvmrc)) | All Mojaloop services | Configured |
| HTTP framework | Hapi (upstream source) | per image | central-ledger, ml-api-adapter, ALS, quoting, settlement | Configured |
| Kafka client | node-rdkafka (librdkafka) | per image | All Kafka producers and consumers | Configured |
| DB access | Knex (MySQL) | per image | central-ledger, quoting, ALS, settlement | Configured |
| Services | central-ledger | v19.4.4 | | Configured |
| | ml-api-adapter | v16.4.2 | | Configured |
| | account-lookup-service | v17.9.1 | | Configured |
| | als-msisdn-oracle-svc | v0.0.18 | | Configured |
| | quoting-service | v17.8.0 | | Configured |
| | central-settlement | v17.0.6 | | Configured |
| | sdk-scheme-adapter | v24.7.0 | FSP side | Configured |

## 2. Infrastructure dependencies

| Technology | Version | Role | Status |
| --- | --- | --- | --- |
| Apache Kafka (KRaft, Bitnami legacy image) | 3.9.0 | Messaging backbone | Configured |
| MySQL Server | 8.0 | Ledger, directory | Configured |
| MongoDB | 8.0.6 | Object store, TTK, reporting | Configured |
| Redis | 6.2.4-alpine | Cache, connector state | Configured |
| Docker Compose | v2 syntax (tested on Compose v5.6.1) | Local orchestration with profiles | Configured |
| Kubernetes, Helm, ArgoCD, Istio | not pinned here | Target platform seen in [k8s-mojaloop-perf-tuning](../../../k8s-mojaloop-perf-tuning/) | External |
| Redpanda Console | `latest` (unpinned) | Kafka UI (`debug`, monitoring) | Optional |

## 3. Operations and observability

| Technology | Version | Role | Status |
| --- | --- | --- | --- |
| Prometheus | v2.45.0 | Metrics | Optional |
| Grafana (+ image renderer) | 10.0.2 (3.7.1) | Dashboards | Optional |
| node-exporter, cAdvisor, kafka-exporter | `latest` (unpinned) | Host, container and Kafka metrics | Optional |
| mysqld-exporter | v0.15.0 | MySQL metrics for the perf databases (`mysql-als`, `mysql-cl`), only under the monitoring profiles `als-test`, `transfers-test`, `quotes-test` | Optional |
| Mojaloop Event SDK | per image | Audit events to Kafka | Configured |
| Finance Portal (reporting events processor, GraphQL API, micro-frontends, nginx) | v3.0.0 / v2.1.0 / v2.2.3, v1.7.2, v0.0.6 | Back office | Optional |

## 4. Testing

| Technology | Version | Role | Status |
| --- | --- | --- | --- |
| Mojaloop Testing Toolkit (server, UI, CLI) | v18.11.2, v16.2.0, v1.9.1 | Simulated FSPs, certification tests | Configured |
| Legacy simulator | v12.2.4 | Simple payer/payee/oracle | Configured |
| Callback handler simulator | v0.1.5 | Load-test FSPs | Optional |
| k6 / k6-operator runner | 1.0.0 / runner-v1.2.0 | Load testing (Docker and Kubernetes) | Implemented (test code) |

## 5. Protocols and standards

| Standard | Where | Status |
| --- | --- | --- |
| FSPIOP API (definitions `fspiop_1.0`, `fspiop_1.1`, `fspiop_2.0` in the TTK) | FSP ↔ switch | Configured (validated by tests) |
| FSPIOP 2.0 over ISO 20022 messages (`fspiop_2.0_iso20022` definition) | `API_TYPE=iso20022` for the services in [perf.env](../../../perf.env) | Optional (perf runs only) |
| Interledger (ILP) packet, SHA-256 condition/fulfilment | Transfer agreement and commit | Implemented (upstream; verified at runtime) |
| JWS (RS256) `FSPIOP-Signature` | Message signing | Configured (switch signs) |
| W3C `traceparent` | Trace context in headers and events | Configured (trace export off) |
| OAuth2 client credentials | SDK → scheme gateway | Optional (disabled) |
| mTLS | SDK ↔ SDK in the security harness | Optional |
| GraphQL | Finance Portal reporting API | Optional |

## 6. Build and delivery

| Tool | Evidence | Status |
| --- | --- | --- |
| CircleCI with the `mojaloop/build` orb | [.circleci/config.yml](../../../.circleci/config.yml): one `build/workflow` job on release tags | Configured |
| Vulnerability scanning (Grype) | [.grype.yaml](../../../.grype.yaml) | Configured |
| SBOM | [sbom-v2.15.1.csv](../../../sbom-v2.15.1.csv) | Configured |
| Container registry | Docker Hub (`mojaloop/*`), GHCR (k6-operator), quay.io (node-exporter), gcr.io (cAdvisor) | External |
