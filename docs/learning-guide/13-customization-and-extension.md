# 13 – Customization and Extension

> How to adapt Mojaloop to your scheme. Most changes are **configuration and onboarding API calls**. Code changes belong in the upstream service repositories.
> Commands in this document are **[RECOMMENDATION]** unless cited from the repository. Keep the original files untouched: put your changes in **new** files (override compose files, new collections, new env files).

---

## 1. Configuration or code? Decide first

| You want to… | Configuration / API call | Source-code change |
| --- | --- | --- |
| Add an FSP, currency, limit or callback URL | ✔ Admin API | – |
| Add an identifier type (IBAN, NATIONAL_ID…) | ✔ Register an oracle (ALS admin API) | Build the oracle service (new code, outside the switch) |
| Change timeouts, caches, routing modes, Kafka settings | ✔ Config overrides / env vars | – |
| Add fees | ✔ FSP quote logic (payee fee); settlement interchange scripts | Scheme fee engine (new code) |
| Add a new FSPIOP field | ✔ `extensionList` key/value pairs | Spec change (scheme governance + upstream PR) |
| New business flow (e.g. a new scenario) | ✔ `transactionType` values already exist | New resources: upstream code |
| Different settlement model | ✔ `POST /settlementModels` | – |
| Change ledger logic | – | Fork central-ledger (avoid if possible) |

---

## 2. Add a new financial institution (register an FSP)

**Template in the repo:** [participant_sender.json](../../docker/ml-testing-toolkit/test-cases/collections/provisioning/participant_sender.json) shows the full onboarding sequence.

| Step | API (central-ledger admin, `:3001`) | Body example |
| --- | --- | --- |
| 1. Create the participant with its first currency | `POST /participants` | `{"name":"banka","currency":"PKR"}` |
| 2. Set the Net Debit Cap and initial position | `POST /participants/banka/initialPositionAndLimits` | `{"currency":"PKR","limit":{"type":"NET_DEBIT_CAP","value":1000000},"initialPosition":0}` |
| 3. Register callback endpoints (one per type, see [12 §7](12-adapter-development.md#7-callback-handling)) | `POST /participants/banka/endpoints` | `{"type":"FSPIOP_CALLBACK_URL_TRANSFER_PUT","value":"https://banka-connector.example/transfers/{{transferId}}"}` |
| 4. Register notification emails | same | `{"type":"NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL","value":"treasury@banka.example"}` |
| 5. Find the settlement account id | `GET /participants/banka/accounts` | – |
| 6. Fund it (funds-in) | `POST /participants/banka/accounts/{id}` | `{"transferId":"<uuid>","externalReference":"...","action":"recordFundsIn","reason":"Initial prefunding","amount":{"amount":"1000000","currency":"PKR"}}` |
| 7. Register its customers' identifiers | FSPIOP `POST /participants/{Type}/{ID}` (FSPIOP-Source: banka) | `{"fspId":"banka","currency":"PKR"}` |
| 8. Activate or deactivate | `PUT /participants/banka` | `{"isActive":true}` |

**How to run it locally (two options):**
1. **TTK collection (preferred):** copy `participant_sender.json` to a **new** file (e.g. `my-collections/banka.json`), replace `{$inputs.payerfsp}` with your FSP name, then import and run it from the TTK UI (<http://localhost:9660> → Test Runner → Import).
2. **curl from inside the Docker network** (switch ports aren't published):

```bash
docker run --rm --network mojaloop-net curlimages/curl:8.10.1 -s -X POST http://central-ledger:3001/participants -H "Content-Type: application/json" -d "{\"name\":\"banka\",\"currency\":\"XXX\"}"
```

**Testing:** check `GET /participants/banka`, `/accounts`, `/limits`, `/endpoints`, then run a P2P test with `fromFspId=banka`. Plus non-functional checks: certificate exchange and gateway access **[EXTERNAL]**.

---

## 3. Add a new currency

| Step | API | Notes |
| --- | --- | --- |
| Hub accounts for the currency | `POST /participants/Hub/accounts` `{"type":"HUB_MULTILATERAL_SETTLEMENT","currency":"PKR"}` and `{"type":"HUB_RECONCILIATION","currency":"PKR"}` | As in [hub_setup.json](../../docker/ml-testing-toolkit/test-cases/collections/provisioning/hub_setup.json) ("Second Currency" requests) |
| Settlement model for the currency | `POST /settlementModels` `{"name":"DEFERREDNETPKR","settlementGranularity":"NET","settlementInterchange":"MULTILATERAL","settlementDelay":"DEFERRED","requireLiquidityCheck":true,"ledgerAccountType":"POSITION","autoPositionReset":true,"currency":"PKR","settlementAccountType":"SETTLEMENT"}` | Or rely on `DEFAULTDEFERREDNET` (no currency) |
| Each FSP that will use it | `POST /participants` `{"name":"banka","currency":"PKR"}` + `initialPositionAndLimits` + funds-in | Same pattern as "Add participant with second currency" in `participant_sender.json` |
| Connector | `supportedCurrencies` in backend party responses; FX config if cross-currency | |
| Tests | Change `currency` / `currency2` inputs in a **copy** of `default-env.json` | Harness defaults: `XXX`, `XTS`, `XDR` |

Currency codes are validated against the ISO 4217 list seeded by central-ledger **[UPSTREAM-DOC]**. Amount precision is capped at 4 decimal places (`AMOUNT.SCALE`).

---

## 4. Configure new payment use cases

| Use case | What to configure | Status here |
| --- | --- | --- |
| Merchant payments | FSPs send `scenario: PAYMENT`; register merchant IDs (`BUSINESS`/`ALIAS`) in an oracle | Oracle needed (§5) |
| Cash-in / cash-out | `scenario: DEPOSIT` / `WITHDRAWAL`, `initiatorType: AGENT` | Works with the existing flow |
| Cross-currency | Onboard an FXP with accounts in both currencies and FX callbacks ([fxp.json](../../docker/ml-testing-toolkit/test-cases/collections/provisioning/fxp.json)); payer SDK `FX_QUOTES_ENDPOINT`, `FX_TRANSFERS_ENDPOINT` | **[IMPLEMENTED]** |
| On-us transfers | Upstream central-ledger `ENABLE_ON_US_TRANSFERS` (default `false`) via a config override | **[CONFIG]** |
| Request-to-pay, OTP | Add `mojaloop/transaction-requests-service` to an override compose file; register `FSPIOP_CALLBACK_URL_TRX_REQ_SERVICE` | Not in harness; you would add and test it |
| Bulk / disbursements | Add `mojaloop/bulk-api-adapter`; bulk topics and central-ledger bulk handlers already exist | Not in harness |

Adding a missing service: create `docker-compose.extra.yml` with the upstream image on `mojaloop-net`, a config override in a **new** `configs/<service>.js`, and run `docker compose -f docker-compose.yml -f docker-compose.extra.yml …`. Follow the upstream service README for its configuration **[EXTERNAL]**.

---

## 5. Add an oracle for a new identifier type

**How discovery routing works:** ALS keeps a table of oracles keyed by `oracleIdType` (and optionally currency). On `GET /parties/IBAN/…`, it calls the oracle registered for `IBAN`.

| Step | Detail |
| --- | --- |
| 1. Build or choose an oracle service | Implements the **Oracle API**: `GET/POST/PUT/DELETE /participants/{Type}/{ID}` (+ `/health`), returning `{"partyList":[{"fspId":"..."}]}` for GET **[UPSTREAM-DOC]**. `als-msisdn-oracle-svc` is a reference implementation for MSISDN. |
| 2. Run it on `mojaloop-net` | Override compose file, own database |
| 3. Register it with ALS | `POST http://account-lookup-service:4001/oracles` `{"oracleIdType":"IBAN","endpoint":{"value":"http://iban-oracle:3000","endpointType":"URL"},"isDefault":true}` (same shape as [hub_setup_prod.json](../../docker/ml-testing-toolkit/test-cases/collections/provisioning/hub_setup_prod.json)) |
| 4. FSPs register their identifiers | `POST /participants/IBAN/{iban}` |
| 5. Test | `GET /parties/IBAN/{iban}` returns the right FSP; negative test for an unknown IBAN → 3204 |

> Note: the harness registers `ACCOUNT_ID` against `http://moja-simulator/oracle`, which doesn't exist in compose. Re-register `ACCOUNT_ID` with `PUT /oracles/{ID}` to a real oracle if you need it.

---

## 6. Customise transaction routing

Mojaloop routing is **data-driven**, not rule-driven:

| Routing decision | Mechanism | Where |
| --- | --- | --- |
| Which FSP owns an identifier | Oracle lookup | ALS + oracle DB |
| Where to deliver a message | Participant **endpoints** (callback URLs) | central-ledger `endpoints` |
| Who is the destination of a quote | `FSPIOP-Destination` set by the payer FSP | Request header |
| Quote handling mode | `SIMPLE_ROUTING_MODE` (`false` in harness: validate and persist) | [configs/quoting-service.js](../../docker/config-modifier/configs/quoting-service.js) |
| FX routing | `GET /services/FXP` + payer's choice; SDK `USE_QUOTE_SOURCE_FSP_AS_TRANSFER_PAYEE_FSP` | SDK env |
| Cross-scheme routing | Proxy participants + `PROXY_CACHE` (Redis) | central-ledger, ALS, ML API adapter configs |

"Least-cost routing" or "route by amount" are **not** built-in switch features. Implement them in the oracle (which FSP to return) or in the payer FSP's logic.

---

## 7. Extend the APIs

- **Without spec changes:** every major FSPIOP object accepts `extensionList: {extension: [{key, value}]}`. The SDK passes them through (`quoteRequestExtensions`, `transferRequestExtensions` in the outbound API). Agree on keys in the scheme rulebook.
- **With spec changes:** propose them to the Mojaloop community (API Change Control Board **[UPSTREAM-DOC]**). Implement in upstream services. Add the new spec to the TTK by placing it under `spec_files/api_definitions/<name>_<ver>/` and listing it in `system_config.json` `API_DEFINITIONS` (that is how this repo adds `fspiop_2.0_iso20022`, `fx-api_2.0` and others).

---

## 8. Integrate a core banking system or mobile wallet

See [11-fsp-integration-guide.md](11-fsp-integration-guide.md) and [12-adapter-development.md](12-adapter-development.md). Differences for wallets: identifiers are usually MSISDN, credits are real-time, and agents generate `DEPOSIT`/`WITHDRAWAL` scenarios. Banks often need IBAN/ACCOUNT_ID oracles and provisional crediting.

---

## 9. Add monitoring

| Task | How | Files |
| --- | --- | --- |
| Start the stack | `docker compose --project-name monitoring -f docker-compose-monitoring.yml up -d` | [docker-compose-monitoring.yml](../../docker-compose-monitoring.yml) |
| Add scrape targets | Add `host:port` to the `mojaloop-services` job (services expose `/metrics`) | [prometheus.yml](../../docker/prometheus/prometheus.yml). Copy it and mount your copy via an override. |
| Add a dashboard | Drop a Grafana JSON into a provisioning folder | `docker/grafana/provisioning/dashboards/<folder>/` (picked up by `dashboard.yml`) |
| Alerts | Write rules in a copy of `alert.rules`; enable Alertmanager (commented out); configure receivers | `docker/prometheus/alert.rules`, `docker/alertmanager/config.yml` |
| Tracing and logs | Event SDK: set `LOG`/`TRACE` from `off` to `kafka` in a copy of `event_sdk/config.json` | `docker/event_sdk/config.json` |
| Kafka lag | Already exported by `kafka-exporter` (9308) | monitoring compose |

---

## 10. Customise deployment configuration

| Mechanism | Example | Where |
| --- | --- | --- |
| Image versions | `CENTRAL_LEDGER_VERSION=v19.4.4` → another tag | [.env](../../.env) (or a copy with `--env-file my.env`) |
| Per-service config overrides | New `docker/config-modifier/configs/central-ledger-custom.js`, referenced from an override compose file's `command:` | [config-modifier](../../docker/config-modifier/) |
| Env-var overrides | `CLEDG_*`, `MLAPI_*`, `QUOTE_*`, `ALS_*` with `__` nesting | [perf.env](../../perf.env) shows many |
| Replicas | `QS_REPLICAS`, `ALS_REPLICAS`, `CENTRAL_LEDGER_*_REPLICAS`, `ML_*_REPLICAS` | `.env` (used by perf compose) |
| Config variant switching | `CONF_CL_HANDLER_PREPARE=central-handlers-kafka.js` etc. | `perf.env` |
| Kafka partitions | `KAFKA_*_PARTITIONS_NUM`; partition spread check with `packages/kafaka-partitioner-calculator` | `perf.env` |
| Position batching | Use `topic-transfer-position-batch` mapping + `CENTRAL_LEDGER_POSITION_BATCH_REPLICAS` (README) | README, `configs/central-ledger.js` |
| Kubernetes patches | `k8s-mojaloop-perf-tuning/patch.sh` variants (`audit`, `direct`, `disabled`, `schema`, `init`) | that folder |

**Testing version upgrades:** change one image tag, then run `ttk-provisioning` + `ttk-tests`, and then the golden path. This is exactly how upstream services use this harness in their CI (README "Functional tests inside CICD").

---

## 11. Extend existing services (source-code changes)

1. Fork the upstream repo (e.g. `mojaloop/quoting-service`) and check out the tag pinned in `.env`.
2. Make the change. Run the upstream unit and integration tests **[EXTERNAL]**.
3. Build an image: `docker build -t myorg/quoting-service:v17.8.0-custom.1 .`
4. In an override compose file, set `image: myorg/quoting-service:v17.8.0-custom.1` (or change the `.env` value in a copy).
5. Run the harness provisioning, the P2P test and the golden path. Add new TTK test cases for your feature.
6. Contribute upstream if it's generally useful. Long-lived forks of a payment switch are expensive to maintain.

Known configuration-level extension point: **central-settlement rules scripts** (`HANDLERS.SETTINGS.RULES.SCRIPTS_FOLDER: ./scripts/transferSettlementTemp` in upstream `config/default.json` v17.0.6) for interchange-fee calculation **[UPSTREAM-DOC]**. The golden-path settlement tests check `INTERCHANGE_FEE` accounts.

---

## 12. Build a new microservice that integrates with the switch

| Integration style | Use it for | How |
| --- | --- | --- |
| **Participant** (via SDK) | A new product acting as an FSP (e.g. a merchant aggregator) | Onboard as FSP; use the SDK APIs |
| **Oracle** | A new identifier directory | §5 |
| **Read-only event consumer** | Analytics, fraud monitoring, notifications | Consume `topic-event-audit` (audit events) or `topic-notification-event` with your **own consumer group**. The Finance Portal's `reporting-events-processor-svc` does exactly this (`KAFKA_TOPIC_EVENT=topic-event-audit`). |
| **Admin automation** | Onboarding portal, liquidity bot | Call central-ledger admin and settlement APIs |
| **Reporting** | Regulatory reports | Read replicas of MySQL `central_ledger`; or the reporting GraphQL API |

**Minimal steps for a new consumer service [RECOMMENDATION]:**
1. Create the service in its own repo (any language with a Kafka client).
2. Connect to `kafka:29092`, topic `topic-event-audit`, consumer group `my-analytics`. **Never** reuse a switch handler's group id, or you will steal its messages.
3. Add it to an override compose file on `mojaloop-net`.
4. Run the P2P test and confirm your service receives the events.
5. In production: Kafka ACLs (read-only), separate credentials, monitoring of consumer lag.

---

## 13. Summary matrix

| Extension | Source dirs (upstream) | Config files (this repo) | APIs | Dependencies | Testing |
| --- | --- | --- | --- | --- | --- |
| New FSP | – | new TTK collection | central-ledger admin, FSPIOP `/participants` | SDK, certificates | Provisioning report + P2P |
| New currency | – | env copy | admin + settlement models | Hub accounts | P2P in new currency, settlement test |
| New use case | FSP code | SDK env | FSPIOP | Possibly TRS / bulk adapter | New TTK cases |
| Routing / oracle | new oracle repo | override compose | ALS admin, Oracle API | DB | Parties tests |
| API extension | upstream services | TTK `system_config.json` copy | FSPIOP | Governance | Contract tests |
| CBS / wallet | your adapter | SDK env | SDK backend/outbound | CBS | [11 §8.9](11-fsp-integration-guide.md#89-testing-the-reference-app-against-this-harness-recommendation) |
| Monitoring | – | monitoring compose, Grafana JSON | `/metrics` | Prometheus | Dashboards populated |
| Deployment config | – | `.env`, config-modifier, `perf.env` | – | – | Golden path |
| Service change | upstream repo at tag | override compose `image:` | – | Build pipeline | Unit + golden path |
| New microservice | your repo | override compose | Kafka, admin APIs | Kafka ACLs | Event consumption |

Next: [14-testing-and-troubleshooting.md](14-testing-and-troubleshooting.md).
