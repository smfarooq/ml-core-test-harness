# 06 – Microservices Deep Dive

> A component-by-component reference. Wiring, ports, configuration and commands come from this repository. Internal behaviour comes from upstream source code at the image tags pinned in [.env](../../.env) **[UPSTREAM-DOC]**.
>
> **Reading the code:** clone `https://github.com/mojaloop/<repo>` and run `git checkout <tag>`, using the tag from `.env`. For example, `central-ledger` → `v19.4.4`.

---

## How every switch container starts (common pattern)

From [docker-compose.yml](../../docker-compose.yml), each switch service's `command:` is a chain:

```text
/opt/app/config-modifier/run.js  <image default.json>  <repo override .js>  <image default.json>   # 1. patch config
&& /opt/app/wait4/wait4.js <service-name>                                                             # 2. wait for deps
&& [npm run migrate]                                                                                  # 3. (some) DB schema
&& node src/... | npm run start...                                                                    # 4. start
```

- Step 1 lets the harness use stock images. Only hostnames, Kafka brokers, cache and feature flags are overridden.
- Step 2 is configured in [docker/wait4/wait4.config.js](../../docker/wait4/wait4.config.js) (60 retries × 10 s).
- Each image ships an `.EVENT_SDKrc` replaced by [docker/event_sdk/config.json](../../docker/event_sdk/config.json) (audit events to Kafka).
- Containers run as `user: root` in the harness. This is a test convenience, **not** a production practice.

Most services also accept **environment-variable overrides** using a per-service prefix and `__` for nesting, for example `CLEDG_DATABASE__HOST=mysql-cl` in [perf.env](../../perf.env). Prefixes seen in this repo: `CLEDG_` (central-ledger), `MLAPI_` (ML API adapter), `QUOTE_` (quoting service), `ALS_` (account lookup), `CSTTLMNT` (settlement, in `wait4.config.js`).

---

## 1. Central Ledger (`central-ledger`)

| Attribute | Description |
| --- | --- |
| Component name | `mojaloop/central-ledger:v19.4.4`. Containers: `central-ledger` (API + handlers) and `central-handler-position-batch` |
| Business responsibility | The **clearing ledger**: holds participant registrations, ledger accounts, limits, positions and the authoritative state of every transfer |
| Technical responsibility | Admin HTTP API; Kafka handlers for **prepare**, **fulfil**, **get**, **position / position-batch**, **timeout**, **admin** (funds-in/out) and **bulk** (prepare/fulfil/processing/get); DB migrations |
| Technology | Node.js (Hapi **[UPSTREAM-DOC]**), Knex + MySQL, node-rdkafka, MongoDB object store |
| APIs | Admin API on **3001**: `/participants`, `/participants/{name}/accounts`, `/limits`, `/positions`, `/endpoints`, `/initialPositionAndLimits`, `/settlementModels`, `/ledgerAccountTypes`, `/enums`, `/health`, `/metrics`, `/transactions/{id}` ([central_admin_1.0/api_spec.yaml](../../docker/ml-testing-toolkit/spec_files/api_definitions/central_admin_1.0/api_spec.yaml)) |
| Dependencies | Kafka, MySQL, MongoDB (`objstore`), Redis (proxy cache) |
| Database | MySQL `central_ledger` (user `central_ledger`; password from compose defaults, test only) |
| Messaging | Consumes `topic-transfer-prepare`, `-fulfil`, `-get`, `-position(-batch)`, `topic-admin-transfer`, `topic-bulk-*`. Produces to position and notification topics. |
| Configuration | [configs/central-ledger.js](../../docker/config-modifier/configs/central-ledger.js): `HUB_PARTICIPANT {ID:1, NAME:"Hub"}`, `DATABASE.HOST=mysql`, `MONGODB` enabled on `objstore`, `PROXY_CACHE` on Redis, all Kafka brokers `kafka:29092`, **all POSITION actions routed to `topic-transfer-position-batch`**. Upstream defaults ([v19.4.4 `config/default.json`](https://github.com/mojaloop/central-ledger/blob/v19.4.4/config/default.json)): `PORT 3001`, `HANDLERS.TIMEOUT.TIMEXP "*/15 * * * * *"`, `AMOUNT {PRECISION:18, SCALE:4}`, `ENABLE_ON_US_TRANSFERS false`, `MIGRATIONS.DISABLED false`. |
| Deployment (harness) | `central-ledger`: `node src/api/index.js` → API **plus** all handlers in one process (upstream `runHandlers: !Config.HANDLERS_DISABLED`). `central-handler-position-batch`: `node src/handlers/index.js handler --positionbatch`. Health: `GET /health` on 3001. |
| Deployment (perf / production-like) | Split into one container per handler: `--prepare`, `--position`, `--positionbatch`, `--fulfil`, `--get`, `--admin` (timeout commented out), each scalable ([docker-compose-perf.yml](../../docker-compose-perf.yml)). A `central-ledger-migrator` runs `npm run migrate` once. The API container uses `central-ledger-disabled-handlers.js` (`HANDLERS.DISABLED: true`). |

**Handler CLI flags** (upstream `src/handlers/index.js` at v19.4.4): `--prepare`, `--position`, `--positionbatch`, `--get`, `--fulfil`, `--timeout`, `--admin`, `--bulkprepare`, `--bulkfulfil`, `--bulkprocessing`, `--bulkget`.

**What each handler does [UPSTREAM-DOC]:**

| Handler | Consumes | Responsibility |
| --- | --- | --- |
| Prepare | `topic-transfer-prepare` | Validate the payer/payee FSPs (exist, active, currency accounts), amount scale, expiry, condition; **duplicate check** (hash of the request); persist the transfer as `RECEIVED_PREPARE`; emit a position-prepare message |
| Position / Position batch | `topic-transfer-position(-batch)` | On prepare: check **liquidity** and the **Net Debit Cap**, then increase the payer FSP's position and set `RESERVED`; or reject (`ABORTED`, error 4001 / 4200). On commit: decrease the payee FSP's position (`COMMITTED`). On abort or timeout: reverse the payer's reservation. |
| Fulfil | `topic-transfer-fulfil` | Validate the fulfilment against the condition, check expiry, handle reject/error callbacks, emit a position commit or abort |
| Get | `topic-transfer-get` | Answer `GET /transfers/{ID}` with the current state |
| Timeout | (cron) | Every 15 s, find expired transfers (`EXPIRED_PREPARED`, `RESERVED_TIMEOUT` → `EXPIRED_RESERVED`) and emit aborts and notifications |
| Admin | `topic-admin-transfer` | Funds in / funds out (`recordFundsIn`, `recordFundsOutPrepareReserve`, `…Commit`, `…Abort`) |
| Bulk handlers | `topic-bulk-*` | Split and track bulk transfers (needs the bulk-api-adapter at the edge, not present here) |

**Source map** (upstream repo, tag v19.4.4): `src/api/` (admin routes), `src/handlers/transfers/` (prepare/fulfil), `src/handlers/positions/handler.js` and `handlerBatch.js`, `src/handlers/timeouts/`, `src/handlers/admin/`, `src/domain/{participant,position,transfer,settlement,fx}`, `seeds/` (enumerations such as `transferState.js`), `migrations/`.

---

## 2. ML API Adapter (`ml-api-adapter`)

| Attribute | Description |
| --- | --- |
| Component name | `mojaloop/ml-api-adapter:v16.4.2` |
| Business responsibility | The **transfers front door** for FSPs and the **delivery service** that tells FSPs about transfer outcomes |
| Technical responsibility | Validates FSPIOP headers and bodies for `/transfers` and `/fxTransfers`, publishes to Kafka, returns 202. Its **notification handler** consumes `topic-notification-event`, resolves the destination FSP's callback URL, and calls it (POST/PUT/PATCH). Signs switch-originated messages with JWS. |
| Technology | Node.js, Kafka, Redis |
| APIs | **3000**: `POST /transfers`, `GET /transfers/{ID}`, `PUT /transfers/{ID}`, `PUT /transfers/{ID}/error`, `PATCH` (outbound only), FX equivalents `/fxTransfers`; `/health`, `/metrics` |
| Dependencies | Kafka; central-ledger (`ENDPOINT_SOURCE_URL=http://central-ledger:3001`) for participant endpoints; Redis proxy cache |
| Database | None (stateless; endpoint cache in memory) |
| Messaging | Produces `topic-transfer-prepare`, `-fulfil`, `-get`; consumes `topic-notification-event` |
| Configuration | [configs/ml-api-adapter.js](../../docker/config-modifier/configs/ml-api-adapter.js): `ENDPOINT_SECURITY.JWS.JWS_SIGN: true`, `JWS_SIGNING_KEY_PATH: secrets/jwsSigningKey.key` (mounted from `docker/secrets/`), `TLS.rejectUnauthorized: true`. Upstream default `TRANSFERS.SEND_TRANSFER_CONFIRMATION_TO_PAYEE: true`. |
| Deployment | Harness: one container, `node src/api/index.js` (API + notification handler). Perf: separate `ml-handler-notification` (`node src/handlers/index.js handler --notification`) and API with `ml-api-adapter-disabled-handlers.js`. |

---

## 3. Account Lookup Service (`account-lookup-service`)

| Attribute | Description |
| --- | --- |
| Component name | `mojaloop/account-lookup-service:v17.9.1` |
| Business responsibility | **Payee discovery**: map identifiers to FSPs and relay party information |
| Technical responsibility | FSPIOP `/participants` and `/parties` (port **4002**); admin `/oracles` (**4001**). Picks the oracle by `partyIdType` (from its `oracleEndpoint` table), queries it, then forwards `GET /parties` to the owning FSP and relays the `PUT` back. |
| Technology | Node.js, MySQL, Redis |
| APIs | FSPIOP: `GET/POST/PUT/DELETE /participants/{Type}/{ID}[/{SubId}]`, `POST /participants` (batch), `GET/PUT /parties/{Type}/{ID}[/{SubId}]` (+ `/error`). Admin: `GET/POST /oracles`, `PUT/DELETE /oracles/{ID}`, `/health` ([als_admin_1.1](../../docker/ml-testing-toolkit/spec_files/api_definitions/als_admin_1.1/api_spec.yaml)) |
| Dependencies | `mysql-als`; central-ledger (`SWITCH_ENDPOINT=http://central-ledger:3001`) for FSP endpoints; oracles; Redis (proxy cache); waits for `simulator` |
| Database | MySQL `account_lookup` (`npm run migrate` at start) |
| Messaging | None for the core flow (HTTP only). Emits audit events. |
| Configuration | [configs/account-lookup-service.js](../../docker/config-modifier/configs/account-lookup-service.js): `DATABASE.HOST=mysql-als`, `SCHEMA=account_lookup`, `SWITCH_ENDPOINT`, `PROXY_CACHE`. Env `ALS_DISPLAY_ROUTES=false`. |
| Deployment | `npm run migrate && npm run start:all` (API + admin). Health: `GET :4001/health`. |

**Oracle registration in this harness:** `hub_setup_prod.json` registers `MSISDN → {HOST_ORACLE_MSISDN} = http://als-msisdn-oracle-svc:3000`. `hub_setup.json` registers `ACCOUNT_ID → http://moja-simulator/oracle`, a host that does **not** exist in compose.

## 4. ALS MSISDN Oracle (`als-msisdn-oracle-svc`)

| Attribute | Description |
| --- | --- |
| Component | `mojaloop/als-msisdn-oracle-svc:v0.0.18` (TypeScript, compiled to `dist/`) |
| Business responsibility | Authoritative directory: which FSP serves this phone number |
| APIs | Oracle API on **3000**: `GET/POST/PUT/DELETE /participants/{Type}/{ID}`, `/health` |
| Database | MySQL `oracle_msisdn` on `mysql-als` (created by [02_oracle_msisdn.sql](../../docker/sql-als-init/02_oracle_msisdn.sql)) |
| Configuration | [configs/als-msisdn-oracle-svc.js](../../docker/config-modifier/configs/als-msisdn-oracle-svc.js) |
| Deployment | `npm run migrate && node ./dist/src/cli.js` |

---

## 5. Quoting Service (`quoting-service`, `quoting-service-handler`)

| Attribute | Description |
| --- | --- |
| Component name | `mojaloop/quoting-service:v17.8.0` |
| Business responsibility | **Agreement phase**: get the payee FSP's terms (amounts, fees, expiry, condition) to the payer FSP |
| Technical responsibility | API container receives `/quotes`, `/fxQuotes` and `/bulkQuotes` and publishes to Kafka. Handler container consumes, validates (participants, currencies, rules), **persists** the quote (when `SIMPLE_ROUTING_MODE` is false), detects duplicates, and forwards over HTTP to the counterparty. |
| Technology | Node.js, Knex/MySQL, Kafka |
| APIs | **3002**: `POST /quotes`, `GET/PUT /quotes/{ID}`, `PUT /quotes/{ID}/error`; FX: `POST /fxQuotes`, `GET/PUT /fxQuotes/{ID}`; bulk: `/bulkQuotes`. Handler monitoring/health on **3003**. |
| Dependencies | MySQL (`central_ledger` DB on host `mysql`), Kafka, central-ledger (`SWITCH_ENDPOINT`) |
| Messaging | `topic-quotes-post/put/get`, `topic-fx-quotes-*`, `topic-bulkquotes-*` (consumer groups `group-quotes-handler-post`, …) |
| Configuration | [configs/quoting-service.js](../../docker/config-modifier/configs/quoting-service.js): `SIMPLE_ROUTING_MODE: false` (upstream default is `true`); `CACHE.*_EXPIRES_IN_MS: 1` (caches effectively off, so test changes apply instantly); `UV_THREADPOOL_SIZE=20`. Perf variant adds `ORIGINAL_PAYLOAD_STORAGE: "kafka"`. |
| Deployment | API: `node src/index.js`. Handlers: `npm run start:handlers`. |

`SIMPLE_ROUTING_MODE` matters: in simple mode the service only forwards quotes. With it off (as here), it also validates and stores quote data, which supports duplicate detection and rule-based quoting **[UPSTREAM-DOC]**.

---

## 6. Central Settlement (`central-settlement`)

| Attribute | Description |
| --- | --- |
| Component name | `mojaloop/central-settlement:v17.0.6` |
| Business responsibility | **Settlement**: close windows, compute multilateral net positions, track settlement until each FSP is settled |
| APIs | v2 base `http://central-settlement:3007/v2` (from `HOST_CENTRAL_SETTLEMENT` in `default-env.json`): `GET /settlementWindows`, `GET/POST /settlementWindows/{id}`, `GET/POST /settlements`, `GET/PUT /settlements/{id}`, `GET/PUT /settlements/{sid}/participants/{pid}[/accounts/{aid}]`, `/v2/health` (verified; `/health` returns 404) ([settlements_2.0](../../docker/ml-testing-toolkit/spec_files/api_definitions/settlements_2.0/api_spec.yaml)) |
| Dependencies | MySQL `central_ledger` (shared with the ledger), Kafka (deferred-settlement close and notification topics) |
| Configuration | [configs/central-settlement.js](../../docker/config-modifier/configs/central-settlement.js): DB host, Kafka consumers `DEFERREDSETTLEMENT.CLOSE` and `NOTIFICATION.EVENT` |
| Deployment | `node src/api/index.js` |
| **Known issue** | Its compose **healthcheck calls port 3001**, but upstream `config/default.json` at `v17.0.6` sets **`PORT: 3007`** and the harness does not override it. **Verified:** port 3001 refuses connections and the container shows *health: starting*, then *unhealthy*, while `http://127.0.0.1:3007/v2/health` returns `"status":"OK"` (version 17.0.6). Nothing in the standard profiles depends on its health. |

---

## 7. SDK Scheme Adapter / Mojaloop Connector (`*-sdk`)

| Attribute | Description |
| --- | --- |
| Component name | `mojaloop/sdk-scheme-adapter:v24.7.0` (Nx monorepo; `yarn start:api-svc` or `yarn nx run modules-api-svc:start`) |
| Business responsibility | Lets an FSP integrate with Mojaloop **without** implementing FSPIOP's asynchronous callbacks, JWS, ILP and so on itself |
| Technical responsibility | **Inbound** FSPIOP server (**4000**): receives switch requests and calls the FSP's **backend API**. **Outbound** API (**4001**): the FSP makes one simple call (for example `POST /transfers`), and the SDK runs discovery → quote → transfer, waits for callbacks, and returns a final result. **Test API** (**4002**). Generates ILP packet, condition and fulfilment. Optional JWS and mTLS. |
| Dependencies | Redis (state), switch endpoints (`ALS_ENDPOINT`, `QUOTES_ENDPOINT`, `TRANSFERS_ENDPOINT`, `FX_*`), the FSP backend (`BACKEND_ENDPOINT`) |
| Configuration | [envs/payerfsp-sdk.env](../../envs/payerfsp-sdk.env) (and similar): `DFSP_ID`, `PEER_ENDPOINT`, `BACKEND_ENDPOINT`, `CACHE_URL`, `AUTO_ACCEPT_PARTY`, `AUTO_ACCEPT_QUOTES`, `EXPIRY_SECONDS=60`, `JWS_SIGN`, `VALIDATE_INBOUND_JWS`, `*_MUTUAL_TLS_ENABLED`, `CHECK_ILP`, `ALLOW_TRANSFER_WITHOUT_QUOTE`, `RESOURCE_VERSIONS`, `API_TYPE=fspiop`, `ILP_SECRET` (**secret**; never reuse the sample value) |
| Instances in harness | `payerfsp-sdk`, `payeefsp-sdk`, `testfsp1..4-sdk` (main, each with its own Redis); `fx-payerdfsp-sdk`, `fx-payeedfsp-sdk`, `fx-provider1-sdk` (`fx-sdk`); `perf-sdk-payer/payee/fxp`, `sdk-scheme-adapter-api-svc` (perf); `payer-` and `payee-sdk-scheme-adapter-api-svc` (security) |

How a bank uses it: see [11](11-fsp-integration-guide.md) and [12](12-adapter-development.md).

---

## 8. Mojaloop Testing Toolkit (TTK)

| Attribute | Description |
| --- | --- |
| Components | `mojaloop-testing-toolkit` (server), `mojaloop-testing-toolkit-ui`, `ttk-*` CLI runners (`ml-testing-toolkit-client-lib`) |
| Business responsibility | Stand in for FSPs and the hub operator; certify and regression-test the switch |
| Technical responsibility | Loads the API definitions listed in [system_config.json](../../docker/ml-testing-toolkit/spec_files/system_config.json). Validates inbound requests against the OpenAPI specs. Answers with **response rules**, then sends **callbacks** generated by **callback rules** ([rules_callback/default.json](../../docker/ml-testing-toolkit/spec_files/rules_callback/default.json), 73 rules). Forwards some callbacks to hub services (`CALLBACK_RESOURCE_ENDPOINTS` in `user_config.json`). Runs collections with JavaScript assertions. Produces HTML reports. |
| Ports | 4040 (FSPIOP mock / backend, host 9440), 5050 (control API, host 9550), UI 6060 (host 9660) |
| Storage | MongoDB `objstore`, DB `ttk` |
| Notable rules | `amount = 15` → invalid fulfilment; `amount = 16` → **no callback** (simulates an unresponsive payee); `FSPIOP-Destination: noresponsepayeefsp` → no callback; `amount = 101..104` with `ttkpayeefsp` → PATCH-notification scenarios |
| Labels | `std`, `std-gp`, `perf`, `fx`, `fx-sdk`, `release-tests` |

## 9. Legacy Simulator (`simulator`)

`mojaloop/simulator:v12.2.4` on **8444**. A simple payer/payee/oracle simulator configured with `TRANSFERS_ENDPOINT`, `QUOTES_ENDPOINT`, `PARTIES_ENDPOINT` and `TRANSACTION_REQUESTS_ENDPOINT=http://transaction-request-service:4003`. That last service is not defined in compose. ALS waits for the simulator to be healthy before starting.

## 10. Callback Handler Simulator (`sim-*`, perf only)

`mojaloop/callback-handler-simulator-svc:v0.1.5`. Fast FSP simulators for load tests (`sim-perffsp-1..8`, `sim-perffxp`, `sim-perf-backend-*`). Callbacks arrive on 3001 (`/fspiop`, `/fxp`, `/backend`). A WebSocket on 3002 notifies k6 when a callback for a given trace and transfer arrives. The security harness uses the similar `mojaloop/callback-handler-svc` image.

## 11. Finance Portal (profile `finance-portal`, optional)

| Service | Image | Role |
| --- | --- | --- |
| `reporting-events-processor-svc` | v3.0.0 | Consumes `topic-event-audit` into MongoDB `reporting` |
| `reporting-hub-bop-api-svc` | v2.1.0 | GraphQL API over MySQL `central_ledger` + MongoDB `reporting` (host port 9401) |
| `reporting-hub-bop-shell` | v2.2.3 | Micro-frontend shell, served through `fp-proxy` on **9400** |
| `reporting-hub-bop-trx-ui` | v1.7.2 | Transfers search UI (9302) |
| `reporting-hub-bop-positions-ui` | v0.0.6 | Positions UI (9304), using central-ledger admin through `fp-proxy` `/central-admin/` |
| `fp-proxy` | nginx:alpine | Routes `/central-admin/` → central-ledger:3001, `/graphql` → reporting API, `/` → shell ([nginx.conf](../../docker/fp-proxy/nginx.conf)) |

The shell's menu also lists **Roles** (`reporting_hub_bop_role_ui`, 9301) and **Settlements** (`reporting_hub_bop_settlements_ui`, 9303). No compose service provides them, so both fail with a "Webpack Loading Error" (verified). The profile also starts its own `cadvisor` on host port 8082.

## 12. Merchant Registry (profile `merchant-registry`, experimental)

`acquirer-backend`, `acquirer-frontend` and `registry-oracle` (built from **external source**; placeholder build contexts), plus `rabbitmq`, `merchant-db` and `minio`. It provides LEI-based merchant onboarding and a registry oracle for `ALIAS` lookups ([lei-ml-core-test-harness.md](../../lei-ml-core-test-harness.md)). It does not run as shipped.

---

## 13. Services Mojaloop has that this harness does **not** run [EXTERNAL]

| Service | Purpose | Effect of absence here |
| --- | --- | --- |
| transaction-requests-service | `/transactionRequests`, `/authorizations` (request-to-pay, OTP) | Payee-initiated flows unavailable |
| bulk-api-adapter | `/bulkTransfers` edge | Bulk transfers unavailable end-to-end |
| central-event-processor, email-notifier | NDC threshold alerts and emails | Notification emails registered in provisioning are never sent |
| thirdparty-api-svc, auth-service, consent oracle | PISP (third-party initiation) | Not available (wait4 config still lists them) |
| inter-scheme proxy adapter | Connecting two Mojaloop schemes | Only config and TTK rules exist |
| Mojaloop Connection Manager (MCM), Payment Manager (PM4ML) | Certificate onboarding, FSP portal | Not included (`PM4ML_ENABLED=false`) |

Next: [07-payment-transaction-lifecycle.md](07-payment-transaction-lifecycle.md).
