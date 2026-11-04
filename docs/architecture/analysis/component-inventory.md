# Component Inventory

> Every component this repository deploys or references, with the attributes an architecture review needs.
> Status labels (**Implemented, Configured, External, Optional, Proposed, Unverified**) are defined in the [README](../README.md#status-labels).
> Image versions are the tags pinned in [`.env`](../../../.env). "Verified at runtime" means the behaviour was observed in the lab runs of 2026-11-04 (Windows 11, Docker Desktop, Compose v5.6.1).

**Read this first.** This repository is the **Mojaloop core test harness**. It does **not** contain the source code of the switch services. It deploys them as **pre-built Docker images** and configures them. So for switch services:

- *what* runs, its ports, wiring and configuration are **Configured** (evidence: compose files and config overrides here);
- *how* a service behaves internally is taken from the upstream source at the exact pinned tag, and, where marked, verified at runtime.

The only source code in this repository is harness tooling: the config-modifier, `wait4`, the Kafka provisioning script, the TTK collections, the k6 load tests and the Kafka partition calculator.

---

## Index

| # | Component | Domain | Status |
| --- | --- | --- | --- |
| 1 | [ML API Adapter](#1-ml-api-adapter) (incl. notification handler) | API layer / notifications | Configured |
| 2 | [Central Ledger](#2-central-ledger-api-and-handlers) (API and handlers) | Transaction processing / ledger | Configured |
| 3 | [Position batch handler](#3-position-batch-handler) | Transaction processing / risk | Configured |
| 4 | [Account Lookup Service](#4-account-lookup-service-als) | Routing / participant resolution | Configured |
| 5 | [ALS MSISDN Oracle](#5-als-msisdn-oracle) | Directory | Configured |
| 6 | [Quoting Service](#6-quoting-service-api-and-handler) (API and handler) | Agreement / quoting | Configured |
| 7 | [Central Settlement](#7-central-settlement) | Settlement | Configured |
| 8 | [SDK Scheme Adapter (Mojaloop Connector)](#8-sdk-scheme-adapter-mojaloop-connector) | FSP integration | Configured |
| 9 | [FSP core banking / wallet system](#9-fsp-core-banking-or-wallet-system) | Participant | External |
| 10 | [Kafka](#10-kafka) | Messaging | Configured |
| 11 | [MySQL (`mysql`, `mysql-als`)](#11-mysql) | Persistence | Configured |
| 12 | [MongoDB (`objstore`)](#12-mongodb-objstore) | Persistence | Configured |
| 13 | [Redis](#13-redis) | Cache / connector state | Configured |
| 14 | [Finance Portal (Business Operations Portal)](#14-finance-portal-business-operations-portal) | Operations / reporting | Optional |
| 15 | [Monitoring stack](#15-monitoring-stack) | Observability | Optional |
| 16 | [Mojaloop Testing Toolkit (TTK)](#16-mojaloop-testing-toolkit-ttk) | Testing / simulated participants | Configured |
| 17 | [Simulators and load tools](#17-simulators-and-load-tools) | Testing | Optional |
| 18 | [Harness glue](#18-harness-glue-config-modifier-wait4-provisioning) (config-modifier, wait4, provisioning) | Configuration / start-up | Implemented |
| 19 | [Components referenced but not deployed](#19-components-referenced-but-not-deployed) | – | External / Unverified |

---

## 1. ML API Adapter

| Attribute | Value |
| --- | --- |
| Component name | `ml-api-adapter` (`mojaloop/ml-api-adapter:v16.4.2`). Its notification handler runs in the same process here; it is a separate deployment in the perf compose and in Kubernetes (`moja-ml-api-adapter-handler-notification`). |
| Business responsibility | The transfers front door for FSPs, and the service that tells FSPs the outcome of their transfers |
| Technical responsibility | Validates FSPIOP headers and bodies for `/transfers` and `/fxTransfers`, publishes the request to Kafka and answers **202 Accepted**. The notification handler consumes `topic-notification-event`, looks up the destination FSP's callback URL, and calls it (`POST`/`PUT`/`PATCH`). Signs switch-originated messages with JWS. |
| Technology / framework | Node.js (Hapi, upstream), node-rdkafka, Redis client |
| Deployment model | One container (`node src/api/index.js`); perf compose and Kubernetes split API and notification handler, each scalable (`ML_API_ADAPTER_REPLICAS`, `ML_NOTIFICATION_REPLICAS`) |
| Dependencies | Kafka; central-ledger admin API (participant endpoints); Redis (proxy cache) |
| APIs exposed | Port **3000**: `POST /transfers`, `GET/PUT /transfers/{ID}`, `PUT /transfers/{ID}/error`, `/fxTransfers` equivalents, `/health`, `/metrics` |
| APIs consumed | central-ledger `GET /participants/{name}/endpoints`; FSP callback URLs (outbound) |
| Communication protocol | HTTP/1.1 REST (FSPIOP JSON) in and out; Kafka produce/consume |
| Data stored or processed | No database. Processes transfer messages and holds an in-memory endpoint cache. |
| Security mechanisms | FSPIOP header validation; **JWS signing of its own messages: on** (`JWS_SIGN: true`, key mounted from `docker/secrets/`); TLS for outbound calls when FSP URLs are `https` (`rejectUnauthorized: true`). No inbound authentication in the harness. |
| Scaling characteristics | Stateless; scale horizontally behind a load balancer. Notification throughput is bounded by `topic-notification-event` partitions. |
| High availability | Run ≥ 2 replicas of the API and the notification handler (**Proposed**). The harness runs one, with no restart policy. |

## 2. Central Ledger (API and handlers)

| Attribute | Value |
| --- | --- |
| Component name | `central-ledger` (`mojaloop/central-ledger:v19.4.4`) |
| Business responsibility | The clearing ledger and participant registry: participants, accounts, limits, positions, and the authoritative state of every transfer |
| Technical responsibility | Admin HTTP API, plus Kafka handlers: **prepare**, **fulfil**, **get**, **admin** (funds in/out), **timeout** (cron every 15 s) and **bulk** (prepare/fulfil/processing/get). Runs DB migrations at start. |
| Technology / framework | Node.js (Hapi), Knex + MySQL, node-rdkafka, MongoDB object store |
| Deployment model | Harness: one container, `node src/api/index.js` → API **and** all handlers in one process (except position batch, see 3). Perf compose / Kubernetes: one deployment per handler (`--prepare`, `--fulfil`, `--get`, `--admin`, `--timeout`) and a migration step (`central-ledger-migrator` / init container `run-migration`). |
| Dependencies | Kafka, MySQL `central_ledger`, MongoDB (`mlos`), Redis (proxy cache) |
| APIs exposed | Port **3001** admin API: `/participants`, `/participants/{name}/accounts`, `/limits`, `/positions`, `/endpoints`, `/initialPositionAndLimits`, `/settlementModels`, `/ledgerAccountTypes`, `/health`, `/metrics` |
| APIs consumed | None over HTTP for the core flow |
| Communication protocol | HTTP REST (admin); Kafka consume/produce (transfer topics) |
| Data stored or processed | Participants, accounts (`participantCurrency`), limits, positions and position changes, transfers and their state history, ILP packets, fulfilments, errors, settlement models |
| Security mechanisms | Validation of participants (exist, active), amounts, expiry, condition, duplicate hash; Net Debit Cap and liquidity checks. **Admin API is unauthenticated in the harness** (verified). |
| Scaling characteristics | Handlers scale with Kafka partitions (one consumer per partition per group). The admin API is stateless. |
| High availability | Multiple handler replicas per consumer group (**Proposed**); MySQL is the stateful dependency. |

## 3. Position batch handler

| Attribute | Value |
| --- | --- |
| Component name | `central-handler-position-batch` (same central-ledger image, `node src/handlers/index.js handler --positionbatch`); Kubernetes deployment `moja-handler-pos-batch` |
| Business responsibility | Inter-FSP risk control: reserve the payer's position on prepare, enforce the Net Debit Cap, move the payee's position on commit, release on abort or timeout |
| Technical responsibility | Consumes `topic-transfer-position-batch` (the harness routes **all** position actions there, [configs/central-ledger.js](../../../docker/config-modifier/configs/central-ledger.js)); processes many messages per DB transaction; produces to `topic-notification-event` |
| Technology / framework | Node.js, Knex/MySQL, node-rdkafka |
| Deployment model | Separate container; replicas via `CENTRAL_LEDGER_POSITION_BATCH_REPLICAS` in perf |
| Dependencies | Kafka, MySQL |
| APIs exposed | `/health`, `/metrics` on 3001 |
| APIs consumed | – |
| Communication protocol | Kafka |
| Data stored or processed | `participantPosition`, `participantPositionChange`, `transferStateChange` |
| Security mechanisms | Rejects with **4001** (liquidity) or **4200** (NDC) → `ABORTED_REJECTED` (verified at runtime) |
| Scaling characteristics | Messages are keyed by the participant account whose position changes (payer on prepare, payee on commit; verified), so per-account ordering is kept and parallelism is per partition. The [partition calculator](../../../packages/kafaka-partitioner-calculator/) exists because uneven key spread limits throughput. |
| High availability | Consumer-group failover with ≥ 2 replicas (**Proposed**) |

## 4. Account Lookup Service (ALS)

| Attribute | Value |
| --- | --- |
| Component name | `account-lookup-service` (`mojaloop/account-lookup-service:v17.9.1`) |
| Business responsibility | Payee discovery: which FSP serves this phone number, account or ID, and who is the party |
| Technical responsibility | FSPIOP `/participants` and `/parties` (port 4002); admin `/oracles` (port 4001). Chooses the oracle by party ID type, asks it for the FSP, then forwards `GET /parties` to that FSP and relays its `PUT` callback. |
| Technology / framework | Node.js, MySQL, Redis |
| Deployment model | One container (`npm run migrate && npm run start:all`); perf replicas `ALS_REPLICAS`; Kubernetes `moja-account-lookup-service` |
| Dependencies | MySQL `account_lookup` (`mysql-als`), oracles (HTTP), central-ledger (FSP endpoints), Redis |
| APIs exposed | FSPIOP 4002: `GET/POST/PUT/DELETE /participants/{Type}/{ID}`, `GET/PUT /parties/{Type}/{ID}` (+ `/error`). Admin 4001: `GET/POST /oracles`, `/health`. |
| APIs consumed | Oracle API; central-ledger endpoints; FSP `/parties` callback URLs |
| Communication protocol | HTTP REST (no Kafka in the core lookup flow) |
| Data stored or processed | Oracle registry (`oracleEndpoint`), party ID types; party data is relayed, not stored |
| Security mechanisms | FSPIOP validation; **3204** party not found (verified). Admin API needs `Content-Type` and `Date` headers but no credentials (verified). |
| Scaling characteristics | Stateless; horizontally scalable |
| High availability | ≥ 2 replicas plus a resilient oracle and DB (**Proposed**) |

## 5. ALS MSISDN Oracle

| Attribute | Value |
| --- | --- |
| Component name | `als-msisdn-oracle-svc` (`mojaloop/als-msisdn-oracle-svc:v0.0.18`, TypeScript) |
| Business responsibility | Authoritative directory: MSISDN → FSP |
| Technical responsibility | Oracle API `/participants/{Type}/{ID}` on port 3000 |
| Technology / framework | Node.js (TypeScript), MySQL |
| Deployment model | One container (`npm run migrate && node ./dist/src/cli.js`) |
| Dependencies | MySQL `oracle_msisdn` on `mysql-als` |
| APIs exposed | `GET/POST/PUT/DELETE /participants/{Type}/{ID}`, `/health` |
| APIs consumed | – |
| Communication protocol | HTTP REST, synchronous |
| Data stored or processed | Table `oracleMSISDN(id, subId, fspId)` (verified) |
| Security mechanisms | None configured (internal-only service) |
| Scaling characteristics | Stateless API over a small table |
| High availability | Replicas plus DB HA (**Proposed**). Other ID types need other oracles (`ACCOUNT_ID` points to a non-existent host in `hub_setup.json`). |

## 6. Quoting Service (API and handler)

| Attribute | Value |
| --- | --- |
| Component name | `quoting-service` and `quoting-service-handler` (`mojaloop/quoting-service:v17.8.0`) |
| Business responsibility | Agreement phase: deliver the payee FSP's terms (amounts, fees, expiry, ILP condition) to the payer FSP |
| Technical responsibility | API (3002) receives `/quotes`, `/fxQuotes` and `/bulkQuotes` and publishes to Kafka. The handler (health on 3003) validates, **persists** the quote (because `SIMPLE_ROUTING_MODE: false`), detects duplicates, and forwards over HTTP to the counterparty. |
| Technology / framework | Node.js, Knex/MySQL, Kafka |
| Deployment model | Two containers; perf replicas `QS_REPLICAS`; Kubernetes `moja-quoting-service`, `moja-quoting-service-handler` |
| Dependencies | Kafka, MySQL `central_ledger` (host `mysql`), central-ledger (endpoints) |
| APIs exposed | `POST /quotes`, `GET/PUT /quotes/{ID}`, `PUT /quotes/{ID}/error`, `/fxQuotes`, `/bulkQuotes`, `/health` |
| APIs consumed | FSP callback URLs; central-ledger endpoints |
| Communication protocol | HTTP REST in/out; Kafka between API and handler |
| Data stored or processed | Quote, quote party and quote response records |
| Security mechanisms | FSPIOP validation; duplicate detection (resend gives a re-sent callback; changed body gives **3106**, verified) |
| Scaling characteristics | API stateless; handler scales with `topic-quotes-*` partitions |
| High availability | Replicas per tier (**Proposed**) |

## 7. Central Settlement

| Attribute | Value |
| --- | --- |
| Component name | `central-settlement` (`mojaloop/central-settlement:v17.0.6`) |
| Business responsibility | Net settlement: close settlement windows, compute multilateral net positions, track each FSP until settled |
| Technical responsibility | Settlement windows and settlements API (base `/v2`, port 3007) |
| Technology / framework | Node.js, MySQL, Kafka |
| Deployment model | One container (`node src/api/index.js`) |
| Dependencies | MySQL `central_ledger` (shared with central-ledger), Kafka |
| APIs exposed | `GET /v2/settlementWindows`, `POST /v2/settlementWindows/{id}` (close), `GET/POST /v2/settlements`, `PUT /v2/settlements/{id}`, `/v2/health` (verified; `/health` is 404) |
| APIs consumed | – |
| Communication protocol | HTTP REST; Kafka |
| Data stored or processed | Settlement windows and state changes, settlements, settlement participant currencies; MLNS transfers between FSP positions and the Hub multilateral settlement account (verified) |
| Security mechanisms | None in the harness (unauthenticated operator API) |
| Scaling characteristics | Low volume (operator-driven) |
| High availability | 2 replicas (**Proposed**). Note: its compose health check targets the wrong port (3001), so it always shows unhealthy. |

## 8. SDK Scheme Adapter (Mojaloop Connector)

| Attribute | Value |
| --- | --- |
| Component name | `*-sdk` containers (`mojaloop/sdk-scheme-adapter:v24.7.0`): `payerfsp-sdk`, `payeefsp-sdk`, `testfsp1..4-sdk`, FX and perf variants |
| Business responsibility | Lets an FSP join Mojaloop without implementing FSPIOP's asynchronous callbacks, ILP and JWS itself |
| Technical responsibility | Inbound FSPIOP server (4000) → calls the FSP's **Backend API**. Outbound API (4001): one synchronous call runs discovery → quote → transfer and waits for callbacks. Test API (4002). Generates ILP packet, condition and fulfilment from `ILP_SECRET`. |
| Technology / framework | Node.js (Nx monorepo), Redis |
| Deployment model | One container plus its own Redis per FSP; **runs at the FSP**, not the hub |
| Dependencies | Redis (state), switch endpoints (`ALS_ENDPOINT`, `QUOTES_ENDPOINT`, `TRANSFERS_ENDPOINT`), FSP backend (`BACKEND_ENDPOINT`) |
| APIs exposed | FSPIOP inbound (4000), Outbound API (4001), test API (4002) |
| APIs consumed | Switch FSPIOP APIs; FSP Backend API (`/parties`, `/quoterequests`, `/transfers`) |
| Communication protocol | HTTP REST (FSPIOP async to the switch; sync to the backend and the FSP's caller) |
| Data stored or processed | In-flight transfer state in Redis; party, quote and transfer data in transit |
| Security mechanisms | **Optional** JWS signing/validation, mTLS (in/out), OAuth2 client credentials to a gateway (`OAUTH_TOKEN_ENDPOINT`). All **off** in the main harness; mTLS + JWS **on** in [docker-compose-security.yml](../../../docker-compose-security.yml). |
| Scaling characteristics | Stateless process with shared Redis; horizontally scalable |
| High availability | 2+ instances with HA Redis (**Proposed**, FSP's responsibility) |

## 9. FSP core banking or wallet system

| Attribute | Value |
| --- | --- |
| Component name | The participant's own system of record (bank core, wallet platform) |
| Business responsibility | Customer accounts, authentication, AML/fraud screening, holds, debits and credits |
| Technical responsibility | Implements the SDK Backend API (as payee) and calls the SDK Outbound API (as payer) |
| Technology / framework | Any. Simulated here by the **TTK backend** (`BACKEND_ENDPOINT=mojaloop-testing-toolkit:4040/backend`). A .NET example is built in the learning guide (M9). |
| Deployment model | **External** to the hub |
| Dependencies | SDK Scheme Adapter |
| APIs exposed / consumed | Backend API (exposed); Outbound API (consumed) |
| Communication protocol | HTTP REST, synchronous |
| Data stored or processed | Customer and account data (never stored by the hub) |
| Security mechanisms | Bank IAM, channel authentication (External) |
| Scaling / HA | Bank's responsibility |

## 10. Kafka

| Attribute | Value |
| --- | --- |
| Component name | `kafka` (`bitnamilegacy/kafka:3.9.0`, KRaft: combined broker + controller) and `kafka-provisioning` (topic creation script) |
| Business responsibility | Decouples the API edge from processing and carries every transfer and quote step |
| Technical responsibility | 23 topics created by [docker/kafka/scripts/provision.sh](../../../docker/kafka/scripts/provision.sh) with **1 partition, replication factor 1**. Audit events go to `topic-event-audit`. |
| Technology / framework | Apache Kafka 3.9 (KRaft) |
| Deployment model | Single broker; PLAINTEXT listeners (`kafka:29092` internal). Kubernetes evidence: `mojaloop-kafka-svc.stateful-resources.svc.cluster.local:9092`. |
| Dependencies | – |
| APIs exposed | Kafka protocol 29092 (internal), 9092 (host, not published) |
| APIs consumed | – |
| Communication protocol | Kafka binary protocol |
| Data stored or processed | Transient transfer, quote, notification and audit messages (default retention; not configured here) |
| Security mechanisms | **None** (no TLS, SASL or ACLs) |
| Scaling characteristics | Throughput set by partitions; perf uses `KAFKA_POSITION_PARTITIONS_NUM=25` ([perf.env](../../../perf.env)) |
| High availability | **Single point of failure** in the harness. Production: ≥ 3 brokers, RF 3, `min.insync.replicas=2` (**Proposed**). |

## 11. MySQL

| Attribute | Value |
| --- | --- |
| Component name | `mysql` (DB `central_ledger`) and `mysql-als` (DBs `account_lookup`, `oracle_msisdn`), image `mysql/mysql-server:8.0` |
| Business responsibility | System of record for the ledger, positions, quotes, settlement and the directory |
| Technical responsibility | Schemas created by the services' migrations (108 tables in `central_ledger`, verified); init SQL in `docker/sql-init`, `docker/sql-als-init` |
| Technology / framework | MySQL 8.0 |
| Deployment model | Two single instances, **no persistent volume** (data is in the container's anonymous volume; `down -v` deletes it) |
| Dependencies | – |
| APIs exposed | MySQL 3306 (not published) |
| Data stored or processed | Participants, accounts, limits, positions, transfers, state history, quotes, settlement; oracle data |
| Security mechanisms | Static passwords from compose defaults; `MYSQL_ALLOW_EMPTY_PASSWORD=true`; no TLS, no encryption at rest |
| Scaling characteristics | Vertical; the position path is write-heavy |
| High availability | Single point of failure here. Production: replicated cluster with automatic failover and backups (**Proposed**). |

## 12. MongoDB (`objstore`)

| Attribute | Value |
| --- | --- |
| Component name | `objstore` (`mongo:8.0.6`) |
| Business responsibility | Document store for large or audit payloads |
| Technical responsibility | DB `mlos` (central-ledger object store, used for bulk payloads), `ttk` (TTK history and reports), `reporting` (audit events for the Finance Portal) |
| Deployment model | Single instance, host port **27017 published**, no auth |
| Security mechanisms | None |
| High availability | Replica set (**Proposed**). Kubernetes evidence: `mongodb-reporting-events-mongodb.stateful-resources…` in [reporting-service-config.json](../../../k8s-mojaloop-perf-tuning/reporting-service-config.json). |

## 13. Redis

| Attribute | Value |
| --- | --- |
| Component name | `redis` (`redis:6.2.4-alpine`, port 6379 published) and one Redis per SDK (`payerfsp-redis` 6383, `payeefsp-redis` 6381, `testfsp1..4-redis` 6385–6391) |
| Business responsibility | Hub: proxy cache for inter-scheme routing. SDK: in-flight transaction state. |
| Deployment model | Single instances, no auth, ports published to the host |
| High availability | Sentinel or cluster mode (**Proposed**); the SDK's Redis holds state that a failure would lose mid-transfer |

## 14. Finance Portal (Business Operations Portal)

| Attribute | Value |
| --- | --- |
| Component name | Profile `finance-portal`: `reporting-events-processor-svc` v3.0.0, `reporting-hub-bop-api-svc` v2.1.0, `reporting-hub-bop-shell` v2.2.3, `reporting-hub-bop-trx-ui` v1.7.2, `reporting-hub-bop-positions-ui` v0.0.6, `fp-proxy` (nginx), plus its own `cadvisor` |
| Business responsibility | Hub operator back office: find transfers, view and update financial positions |
| Technical responsibility | The events processor consumes `topic-event-audit` into MongoDB `reporting`. The GraphQL API reads MySQL `central_ledger` and MongoDB. Micro-frontends are loaded by the shell (port 9400, via `fp-proxy`). |
| Deployment model | **Optional** profile |
| APIs exposed | 9400 (shell via proxy), 9401 (reporting API), 9302 (transfers UI), 9304 (positions UI) |
| Security mechanisms | **Off**: `AUTH_ENABLED=false`; the reporting API supports Ory Keto checks (`ORY_KETO_READ_URL`, `AUTH_CHECK_PARTICIPANTS: false`) and reads the user from header `x-user`, so it expects an identity-aware proxy in front of it ([config](../../../docker/reporting-hub-bop-api-svc/config/default.json)). CORS `*` in [nginx.conf](../../../docker/fp-proxy/nginx.conf). |
| Verified behaviour | Transfers and Financial Positions work; **Roles (9301) and Settlements (9303) fail to load**: their UIs are not defined in compose |

## 15. Monitoring stack

| Attribute | Value |
| --- | --- |
| Component name | [docker-compose-monitoring.yml](../../../docker-compose-monitoring.yml): Prometheus v2.45.0, Grafana 10.0.2 (+ image renderer 3.7.1), node-exporter, cAdvisor, kafka-exporter, `kafka-mon-ui` (Redpanda Console); `mysqlexporter-als` / `mysqlexporter-cl` only under the perf-test profiles |
| Business responsibility | Operational visibility for performance work |
| Technical responsibility | Prometheus scrapes every 10 s (jobs `prometheus`, `cadvisor`, `node-exporter`, `mojaloop-services`, `support-services`); Grafana provisions 12 dashboards (Kafka, MySQL, Node.js application, ALS, central-ledger DB, central services, central services characterisation, ML API adapter, quoting service, k6, node monitoring, callback handler) |
| Deployment model | **Optional**, separate Compose project; network `mojaloop-net` is external |
| Verified behaviour | On Docker Desktop for Windows `up -d` fails (`node-exporter` `rslave` mount); cAdvisor clashes with the `debug` profile on port 8080; only 3 of 73 targets reachable outside the perf stack |
| Not present | Alertmanager and alert rules are **commented out**; no log aggregation; no tracing backend |

## 16. Mojaloop Testing Toolkit (TTK)

| Attribute | Value |
| --- | --- |
| Component name | `mojaloop-testing-toolkit` v18.11.2, `mojaloop-testing-toolkit-ui` v16.2.0, CLI runners `ttk-*` (`ml-testing-toolkit-client-lib` v1.9.1) |
| Business responsibility | Plays the hub operator and simulated FSPs; certifies and regression-tests the switch |
| Technical responsibility | Validates requests against the API specs, answers with rules, sends callbacks, runs test collections (provisioning, P2P, golden path, FX) |
| APIs exposed | 4040 FSPIOP mock / backend (host 9440), 5050 control API (host 9550), UI 6060 (host 9660) |
| Security mechanisms | OAuth, Keycloak and Connection Manager integrations exist in [system_config.json](../../../docker/ml-testing-toolkit/spec_files/system_config.json) but are **disabled** |
| Verified behaviour | P2P 31/31; golden path 2,625 assertions, 0 failed; FX 0 failed |

## 17. Simulators and load tools

| Component | Purpose | Status |
| --- | --- | --- |
| `simulator` (v12.2.4) | Legacy payer/payee/oracle simulator; ALS waits for it | Configured |
| `callback-handler-simulator-svc` (v0.1.5) | Fast FSP simulators for load tests (`sim-perffsp-1..8`) | Optional (perf) |
| k6 (1.0.0) + [packages/k6-tests](../../../packages/k6-tests/) | Load scenarios (discovery, quotes, transfers, FX, SDK); `config-k8s/` for running inside Kubernetes with the **k6-operator** ([Dockerfile](../../../Dockerfile)) | Implemented (test code) |
| [Kafka partition calculator](../../../packages/kafaka-partitioner-calculator/) | Plans participant-to-partition spread | Implemented (tool) |

## 18. Harness glue (config-modifier, wait4, provisioning)

| Item | What it does | Evidence |
| --- | --- | --- |
| config-modifier | Deep-merges a per-service override (`configs/*.js`) into the image's `default.json` before start. This is how stock images are configured. | [docker/config-modifier/run.js](../../../docker/config-modifier/run.js) |
| wait4 | Blocks start-up until dependencies answer (60 retries × 10 s) | [docker/wait4/wait4.js](../../../docker/wait4/wait4.js) |
| Kafka provisioning | Creates the 23 topics | [docker/kafka/scripts/provision.sh](../../../docker/kafka/scripts/provision.sh) |
| Event SDK config | Sends audit events to Kafka; `LOG` and `TRACE` off | [docker/event_sdk/config.json](../../../docker/event_sdk/config.json) |
| DB init SQL | Creates DB users and the oracle schema | `docker/sql-*-init/` |

## 19. Components referenced but not deployed

| Component | Where referenced | Status |
| --- | --- | --- |
| Transaction Requests Service (request-to-pay) | Simulator env, golden-path collections | External (not deployed; those tests are skipped) |
| Bulk API adapter | Bulk handlers and topics exist; no adapter | External |
| Finance Portal Roles and Settlements UIs | Shell menu (9301, 9303) | External (not deployed) |
| Mojaloop Helm charts | Deployment names in [k8s-mojaloop-perf-tuning/patch.sh](../../../k8s-mojaloop-perf-tuning/patch.sh) | External |
| Scheme API gateway / IAM (e.g. WSO2, Ory, Keycloak) | SDK OAuth settings, TTK `KEYCLOAK`, reporting API Keto settings | External, disabled |
| Mojaloop Connection Manager (certificate exchange) | TTK `CONNECTION_MANAGER` (disabled), SDK `PM4ML_ENABLED=false` | External |
| Merchant registry | `merchant-registry` profile with placeholder build paths | Optional, not runnable as shipped |
