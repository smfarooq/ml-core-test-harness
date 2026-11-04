# Infrastructure Inventory

> Two perspectives, kept apart on purpose:
> - **Part A – As-Is**: what this repository actually provides (diagram [03a](../diagrams/03a-infrastructure-as-is.svg)).
> - **Part B – Target**: a proposed production deployment (diagram [03b](../diagrams/03b-kubernetes-infrastructure-target.svg)). Everything in Part B is **Proposed** unless marked otherwise.
>
> Status labels are defined in the [README](../README.md#status-labels).

---

## Part A – As-Is (this repository)

### A1. Deployment model: Docker Compose with profiles — Configured

| Compose file | Purpose | Services |
| --- | --- | --- |
| [docker-compose.yml](../../../docker-compose.yml) | Functional switch, simulated FSPs, tests, finance portal, debug UI, merchant registry template | 54 services across profiles |
| [docker-compose-perf.yml](../../../docker-compose-perf.yml) | Performance stack: every handler in its own scalable container, separate DBs (`mysql-cl`, `mysql-als`), 8 simulated FSPs | 43 services |
| [docker-compose-monitoring.yml](../../../docker-compose-monitoring.yml) | Prometheus, Grafana, exporters, Kafka UI (separate Compose project) | 9 services |
| [docker-compose-load.yml](../../../docker-compose-load.yml) | k6 load generator | 1 service |
| [docker-compose-security.yml](../../../docker-compose-security.yml) | Two SDK adapters with mTLS + JWS talking directly, to measure security overhead | 4 services |

Main profiles: `all-services`, `ttk-provisioning`, `ttk-tests`, `ttk-provisioning-gp`, `ttk-tests-gp`, `fx`, `ttk-provisioning-fx`, `ttk-fx-tests`, `finance-portal`, `debug`, `merchant-registry`.

### A2. Hosts, networks and storage

| Resource | As-is value | Evidence | Status |
| --- | --- | --- | --- |
| Hosts | One machine running Docker (tested: Windows 11 + Docker Desktop/WSL2) | Lab run | Configured |
| Networks | One bridge network `mojaloop-net`; monitoring adds `monitoring-net` and treats `mojaloop-net` as external | compose files | Configured |
| Service discovery | Docker DNS by container name (`kafka:29092`, `mysql`, `central-ledger:3001`) | config overrides | Configured |
| Persistent storage | **None for switch data.** MySQL, Kafka, MongoDB and Redis keep data in container storage or anonymous volumes; `down -v` deletes it. Only the merchant-registry MinIO has a named volume. | compose `volumes:` | Configured |
| Published host ports | TTK 9440/9550/9660; MongoDB 27017; Redis 6379 and 6381–6391; finance portal 9400–9401, 9302, 9304, 8082; Kafka UI 8080 (`debug`) | compose | Configured |
| Switch service ports | **Not published** (commented out). Reach them with `docker exec` or through the TTK. | compose | Configured |
| Container registry | Public registries: Docker Hub `mojaloop/*`, `bitnamilegacy/kafka`, `mysql/mysql-server`, `mongo`, `redis`; quay.io; gcr.io; GHCR | `.env`, compose | External |

### A3. Compute, memory and disk (measured, not specified)

The repository defines **no CPU or memory requests or limits** anywhere (compose or Kubernetes files). These figures were measured on 2026-11-04 and are observations, not requirements:

| Stack | Containers | Memory at idle (`docker stats`) | Notes |
| --- | --- | --- | --- |
| Standard (`all-services` + `ttk-provisioning` + `ttk-tests`) | 31 services | ~4.5 GiB | First image pull ~11.2 GB, ~25 min |
| Finance portal (+ `fx`, `finance-portal`, `debug`) | 37 running | ~5.4 GiB | cAdvisor alone ~800 MiB |
| Monitoring subset (5 containers that ran) | 5 | ~215 MiB | |

### A4. Replicas, scaling and availability

| Item | As-is | Evidence | Status |
| --- | --- | --- | --- |
| Replicas | 1 of everything in the main compose. Perf compose reads `QS_REPLICAS`, `ALS_REPLICAS`, `CENTRAL_LEDGER_*_REPLICAS`, `ML_API_ADAPTER_REPLICAS`, `ML_NOTIFICATION_REPLICAS` (all `1` in `.env`). | [.env](../../../.env), perf compose | Configured |
| Restart policy | **None** on the switch services and data stores (Compose default `no`). Only some optional containers have one. | compose | Configured |
| Health checks | 17 Compose health checks (`/health` endpoints, DB pings). `central-settlement`'s targets the wrong port. | compose | Configured |
| Kafka | 1 broker (KRaft combined), 23 topics, 1 partition, replication factor 1. Perf: 25 partitions for position topics (`perf.env`). | [provision.sh](../../../docker/kafka/scripts/provision.sh) | Configured |
| Databases | Single MySQL instances; no replication; no backups | compose | Configured |
| Autoscaling (HPA/VPA) | **Not present** | – | Unverified (none found) |

### A5. Kubernetes evidence (patches against an existing cluster) — Configured

[k8s-mojaloop-perf-tuning/](../../../k8s-mojaloop-perf-tuning/) does **not** deploy Mojaloop. It patches an existing deployment for performance experiments. What it reveals about the cluster it was written for:

| Evidence | What it shows | File |
| --- | --- | --- |
| Namespace `mojaloop`, Helm release `moja` | Deployments `moja-centralledger-service`, `moja-centralledger-handler-transfer-prepare`, `-transfer-fulfil`, `-transfer-get`, `-admin-transfer`, `-timeout`, `moja-handler-pos-batch`, `moja-ml-api-adapter-service`, `moja-ml-api-adapter-handler-notification`, `moja-account-lookup-service`, `moja-quoting-service`, `moja-quoting-service-handler` | [patch.sh](../../../k8s-mojaloop-perf-tuning/patch.sh) |
| Namespace `stateful-resources` | Kafka at `mojaloop-kafka-svc.stateful-resources.svc.cluster.local:9092`; MongoDB `mongodb-reporting-events-mongodb` | [redpanda-app.yaml](../../../k8s-mojaloop-perf-tuning/redpanda-app.yaml), [reporting-service-config.json](../../../k8s-mojaloop-perf-tuning/reporting-service-config.json) |
| GitOps with **ArgoCD** | `Application` in namespace `argocd`; TTK jobs run as ArgoCD `PostSync` hooks | redpanda-app.yaml, [job-setup.yaml](../../../k8s-mojaloop-perf-tuning/job-setup.yaml), [job-gp.yaml](../../../k8s-mojaloop-perf-tuning/job-gp.yaml) |
| **Istio** service mesh | `VirtualService` bound to gateway `istio-ingress-int/internal-wildcard-gateway`; `sidecar.istio.io/inject: 'false'` on test jobs | [redpanda-service.yaml](../../../k8s-mojaloop-perf-tuning/redpanda-service.yaml) |
| On-premises bare-metal dev cluster | Internal host name pattern `*.int.dev1.devbaremetal.moja-onprem.net` | redpanda-service.yaml |
| Config injection | `ConfigMap config-override` mounted as `.CLEDGrc`, `.MLAPIrc`, `.QUOTErc`, `.EVENT_SDKrc` | [config-override.yaml](../../../k8s-mojaloop-perf-tuning/config-override.yaml) |
| Migrations as init container | `initContainers: run-migration` on the ledger service; then `kubectl scale … --replicas=8` | [ledger-service.yaml](../../../k8s-mojaloop-perf-tuning/ledger-service.yaml), patch.sh |
| Event SDK sidecar toggle | `EVENT_SDK_SIDECAR_DISABLED` per deployment | handler patch files |
| Secrets referenced, not defined | `secretRef: moja-ml-ttk-test-setup-aws-creds` | job-setup.yaml |
| In-cluster load testing | k6-operator runner image | [Dockerfile](../../../Dockerfile), [config-k8s](../../../packages/k6-tests/config-k8s/) |

**Not in the repository** (so not shown as existing): Helm charts or values, node pools, control-plane layout, StorageClasses, PersistentVolumes, NetworkPolicies, HPA/VPA, PodDisruptionBudgets, resource requests/limits, ingress controllers for the FSPIOP APIs.

---

## Part B – Target production infrastructure — Proposed

These are design recommendations, sized by **design decision**, not numbers from the repository. Validate every choice with the performance harness (`docker-compose-perf.yml`, k6) before committing to it.

### B1. Cluster topology

| Element | Proposal |
| --- | --- |
| Clusters | One production cluster in the primary site spanning **3 availability zones**, plus a **DR cluster** in a second site (see [disaster-recovery-analysis.md](disaster-recovery-analysis.md)). Separate non-production clusters. |
| Control plane | Managed or 3 control-plane nodes across zones, etcd backed up |
| Worker pools | `edge` (gateways), `switch` (stateless Mojaloop services), `stateful` (Kafka, databases, if self-hosted), `ops` (monitoring, GitOps) |
| Namespaces | `ingress` / `istio-ingress`, `mojaloop`, `stateful-resources`, `monitoring`, `security` (cert-manager, external secrets), `argocd`, `ttk` (non-prod only) |
| Deployment | **Mojaloop Helm charts** (External) delivered by ArgoCD, matching the namespace and release names seen in A5 |

### B2. Workload types

| Workload | Kind | Replicas (starting point, to be tuned) | Scaling |
| --- | --- | --- | --- |
| API services (ALS, quoting, ML API adapter, central-ledger admin, settlement) | Deployment | ≥ 2, spread across zones | HPA on CPU / request rate |
| Kafka handlers (prepare, fulfil, get, position batch, notification, quoting handler, timeout) | Deployment | ≥ 2 per consumer group (timeout: 1 active, leader-elected or singleton by design) | HPA bounded by topic partitions |
| Migrations | Job / init container | 1 | – |
| Kafka | StatefulSet (operator) or managed service | 3 brokers, RF 3, `min.insync.replicas=2` | Add partitions and brokers |
| MySQL | StatefulSet (operator: InnoDB Cluster / Group Replication) or managed service | 3 nodes | Vertical, read replicas for reporting |
| MongoDB | Replica set | 3 | – |
| Redis | Sentinel or cluster | 3 | – |
| Log/metrics agents | DaemonSet | 1 per node | – |

### B3. Network and storage

| Resource | Proposal |
| --- | --- |
| Load balancers | External L4 load balancer → API gateway / Istio ingress for FSPIOP; internal load balancer for operator APIs |
| Ingress / gateway | API gateway with mTLS and OAuth2 at the edge; Istio ingress gateway inside (internal gateway already evidenced in A5) |
| Egress | Egress gateway for callbacks to FSP endpoints, with the hub's client certificate |
| Storage classes | Replicated SSD-backed class for databases and Kafka; separate class for backups |
| Persistent volumes | One per Kafka broker, DB node and Mongo node; sized from retention and volume (design decision) |
| Registry | Private registry mirroring pinned, scanned images (no `latest` tags) |

### B4. Resource sizing

Not defined in the repository. Derive requests and limits from perf runs (the harness's k6 scenarios and Grafana dashboards exist for exactly this), then set HPA targets and PodDisruptionBudgets. Record them as design decisions.
