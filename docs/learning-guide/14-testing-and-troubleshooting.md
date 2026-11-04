# 14 – Testing and Troubleshooting

> The testing assets in this repository, how to run them, how to simulate several FSPs, and how to debug a payment that doesn't complete.

---

## 1. Testing strategy at a glance

| Test type | Present here? | Tool / location | Command or how |
| --- | --- | --- | --- |
| Unit tests (repo) | **No**: `package.json` test scripts are `"true"` | – | – |
| Unit tests (switch services) | **[EXTERNAL]** in each upstream repo | upstream | – |
| Integration (service + its dependencies) | **Yes** | Compose profiles: `central-ledger`, `quoting-service`, `account-lookup-service`, `ml-api-adapter`, `discovery`, `agreement`, `transfer` | [10 §9.6](10-local-setup-and-execution.md#96-partial-stacks-by-phase-or-service) |
| API tests | **Yes** | TTK collections (`collections/tests/**`) | `ttk-tests`, `ttk-tests-gp` |
| Contract tests | **Yes** (implicit) | TTK validates traffic against the OpenAPI specs in `api_definitions/`; `rules_validation/`; header and version negative tests | Run any collection |
| End-to-end | **Yes** | `tests/p2p.json`, golden path, FX happy path | `ttk-tests`, `ttk-tests-gp`, `ttk-fx-tests` |
| Payment lifecycle | **Yes** | Golden path: duplicates, timeouts, aborts, PATCH notifications, funds-in/out, settlement | `ttk-tests-gp` |
| Failure scenarios | **Yes** | `transfer_negative_scenarios/`, `negative_scenarios/`, `block_transfer.json`, special TTK amounts | §5 |
| Performance / load | **Yes** | k6 (`packages/k6-tests`) + `docker-compose-perf.yml` + monitoring | §4 |
| Resilience (chaos) | **Partial** | "No callback" scenarios (k6 `*NoCallback*`, TTK amount 16, `noresponsepayeefsp`), timeouts. No fault injection into infrastructure. | §5 |
| Security | **Partial** | mTLS + JWS harness (`docker-compose-security.yml`); negative header tests; Grype in CI. No penetration tests. | [09](09-security-and-controls.md) |

### Test inventory (TTK)

- `tests/` contains **131 test cases with 614 requests** (counted during this analysis). Most frequent operations: `GET /participants/{name}/accounts` (126), `POST /quotes` (89), `POST /transfers` (58), `GET /settlements/{id}` (48).
- Labels select subsets ([tests/master.json](../../docker/ml-testing-toolkit/test-cases/collections/tests/master.json)): `std` → `p2p.json`; `std-gp` → `golden_path/`; `fx` → `fx/`; `fx-sdk` → `fx-sdk` (folder not present in this version). `lei/` is not referenced by `master.json`.

---

## 2. Test frameworks and commands

| Framework | Role | Command (from compose) |
| --- | --- | --- |
| **TTK CLI** (`ml-testing-toolkit-client-lib` v1.9.1) | Runs collections headless | `npm run cli -- -u http://mojaloop-testing-toolkit:5050 -l 2 -i <collections> -e <environment.json> --labels <labels> --report-format html --save-report true --report-target file://reports/<name>.html` |
| **TTK server** | Simulated FSPs, assertions, reports | Service `mojaloop-testing-toolkit` |
| **k6** | Load | `env K6_SCRIPT_CONFIG_FILE_NAME=<config>.json docker compose --project-name load -f docker-compose-load.yml up` |
| `wait-for-container.sh` | CI: wait for a test container and propagate its exit code | `bash wait-for-container.sh ttk-func-ttk-tests-1` |

**CI pattern used by upstream services** (README "Functional tests inside CICD"):

```bash
docker-compose --project-name ttk-func --profile all-services --profile ttk-provisioning --profile ttk-tests up -d
```

```bash
bash wait-for-container.sh ttk-func-ttk-tests-1
```

Then save the logs and reports (`reports/ttk-func-tests-report.html`, `reports/ttk-provisioning-report.html`) and run `docker-compose -p ttk-func down`.

**Run one collection file [RECOMMENDATION]** (reuses the `ttk-tests` service definition and its volume mounts; requires a running, provisioned stack):

```bash
docker compose --profile all-services --profile ttk-provisioning --profile ttk-tests run --rm --no-deps ttk-tests sh -c "npm run cli -- -u http://mojaloop-testing-toolkit:5050 -l 2 -i collections/tests/golden_path/negative_scenarios/parties_negative.json -e environments/default-env.json --report-format html --save-report true --report-target file://reports/parties-negative.html"
```

Keep all three `--profile` flags. With only `--profile ttk-tests`, Compose v5.6.1 refuses with "depends on undefined service", because the services `ttk-tests` depends on are defined in the other profiles (verified).

Many golden-path files assume the **golden-path provisioning** (`ttk-provisioning-gp`: `payerfsp`, `testfsp1..4`, `noresponsepayeefsp`, …). Provision with `std-gp` before running them.

---

## 3. Simulating multiple financial institutions

| Simulated FSP(s) | Technology | Profile | Callback target | Typical use |
| --- | --- | --- | --- | --- |
| `testingtoolkitdfsp` (payer), `payeefsp` (payee) | TTK, rule-based | `all-services` + `ttk-provisioning` | `http://mojaloop-testing-toolkit:4040` | Standard P2P |
| `payerfsp`, `payeefsp`, `testfsp1`–`testfsp4` | SDK Scheme Adapter + TTK backend | `all-services` + `ttk-provisioning-gp` | `http://<fsp>-sdk:4000` | Golden path, settlement between several FSPs |
| `noresponsepayeefsp` | TTK rule: never answers | golden path | TTK | Timeout tests |
| `ttkpayeefsp`, `ttkfxpayer`, `ttkfxpayee`, `ttkfxp1` | TTK | golden path / FX | TTK | Notifications, FX |
| `perffxp` (standard onboarding), `testfxp1`, `fxpayerdfsp`, `fxpayeedfsp` | TTK or FX SDKs | `fx`, `fx-sdk` | TTK / `fx-*-sdk` | FX |
| `perffsp-1` … `perffsp-8`, `perffxp` | callback-handler simulator | perf `2dfsp`/`4dfsp`/`8dfsp` | `http://sim-perffsp-N:3001/fspiop` | Load |
| `perfsdkpayer`, `perfsdkpayee`, `perfsdkfxp` | SDK + callback-handler backend | perf `fx-sdk-e2e` | SDKs | SDK load |
| `pinkbankfsp` → `greenbankfsp` | TTK mobile simulator | `all-services` | TTK | Visual demo |

**To execute transactions between them:** pick a payer FSP and a payee MSISDN registered by the provisioning (e.g. `testfsp1` MSISDN `17039811903` → `testfsp2` MSISDN `17039811904`). Then either run a golden-path P2P collection with those inputs, or call the payer's SDK outbound `POST /transfers` (publish its port 4001 via an override; see [11 §8.9](11-fsp-integration-guide.md#89-testing-the-reference-app-against-this-harness-recommendation)).

---

## 4. Performance and load testing

### 4.1 Stacks

| Command (from README) | Purpose |
| --- | --- |
| `docker compose --project-name ml-core -f docker-compose-perf.yml --profile als-test --profile ttk-provisioning-als up -d` | Discovery only (ALS) |
| `… --profile transfers-test --profile 8dfsp --profile ttk-provisioning-transfers up -d` | Transfers only |
| `… --profile quotes-test --profile 8dfsp --profile ttk-provisioning-quotes up -d` | Quotes only |
| `… --profile all-services --profile 8dfsp --profile ttk-provisioning-e2e up -d` | Full E2E (set `ALS_SWITCH_ENDPOINT` and `QS_SWITCH_ENDPOINT` in `perf.env` as described in the README) |
| `… --profile sdk-scheme-adapter up -d` | SDK alone |
| `docker compose --project-name monitoring -f docker-compose-monitoring.yml --profile <als-test, transfers-test or quotes-test> up -d` | Metrics (the profile adds the matching MySQL exporter; omit it for host and Kafka metrics only) |

Remember: `cp perf.override.sample.env perf.override.env` first, and make sure `mojaloop-net` exists ([10 §9.7](10-local-setup-and-execution.md#97-monitoring-stack)).

### 4.2 k6 scenarios (`packages/k6-tests/config/*.json`)

`fspiopDiscovery`, `fspiopQuotes`, `fspiopTransfers`, `fspiopTransfersUnidirectional`, `fspiopTransfersNoCallback*`, `fspiopE2E`, `fspiopSingleQuote`, `fspiopSingleTransfer`, `fxSendE2E`, `getTransfers`, `inboundSDKDiscovery|Quotes|Transfer`, `outboundSDKDiscovery|Quotes|Transfer`, `sdkSendE2E`, `sdkFxSendE2E`, `localhost`, `test`. The `config-k8s/` folder adds ramping-rate and parallel variants used by [perf-test.sh](../../perf-test.sh) (`t discovery`, `t transfers rate`, `t dfx rate`, `t e2e`, …).

How a k6 transfer test measures end-to-end latency: k6 opens a **WebSocket** to the payer simulator (`ws://sim-perffsp-1:3002/<traceId>/PUT/transfers/<transferId>`), sends `POST /transfers`, and the simulator pushes `SUCCESS_CALLBACK_RECEIVED` when the switch's callback arrives ([postTransfers.js](../../packages/k6-tests/scripts/postTransfers.js)). Metrics go to Prometheus by remote-write (`K6_OUT=experimental-prometheus-rw` in `perf.env`). The k6 web dashboard is on port 5665.

### 4.3 Automation and remote targets

- [automate_perf.sh](../../automate_perf.sh) runs a scenario and renders the Grafana dashboards to PNGs under `results/<date>/<scenario>/` (needs `automate_perf.env` with Grafana credentials; template `automate_perf_sample.env`).
- To test a **remote Kubernetes** Mojaloop: set endpoints in `perf.override.env`, customise `environments/remote-k8s-env.json`, run simulators with `--profile ttk-provisioning-remote-k8s`, then k6 (README "Testing Performance of Remote Mojaloop Deployment").
- Tuning knobs: replicas in `.env`, partitions in `perf.env`, the position-batch handler, Kafka-optimised configs (`*-kafka.js`), and Kubernetes patches in `k8s-mojaloop-perf-tuning/`.

---

## 5. Practical test scenarios

| # | Scenario | How to trigger (evidence) | Expected switch behaviour | Expected states / codes |
| --- | --- | --- | --- | --- |
| 1 | **Successful payment** | `tests/p2p.json` (`ttk-tests`) | Party found → quote → transfer committed. Payer position +amount, payee position −amount. | Transfer `RECEIVED_PREPARE → RESERVED → RECEIVED_FULFIL → COMMITTED`; callback `transferState: COMMITTED`; 31/31 assertions (the first run right after start-up may show 27/31: see the cold-start note in [10](10-local-setup-and-execution.md#steps-510--start-everything-databases-kafka-services-migrations-onboarding-tests)) |
| 2a | **Insufficient customer funds** | Payer FSP's own balance check | **Never reaches the switch.** The FSP rejects locally. | – (FSP-internal) |
| 2b | **Insufficient FSP liquidity / NDC exceeded** | [block_transfer.json](../../docker/ml-testing-toolkit/test-cases/collections/tests/golden_path/feature_tests/block_transfer.json) ("Block Transfer due to NDC limit") | Prepare rejected by the position handler; positions unchanged | `ABORTED_REJECTED`; error **4001** (insufficient liquidity) or **4200** (payer limit) to the payer FSP, `FSPIOP-Source` = switch |
| 3 | **Invalid beneficiary** | [parties_negative.json](../../docker/ml-testing-toolkit/test-cases/collections/tests/golden_path/negative_scenarios/parties_negative.json) ("Party info of unprovisioned party") | ALS cannot resolve the party | `PUT /parties/{Type}/{ID}/error` **3204** Party not found; no quote or transfer |
| 4 | **Destination FSP unavailable** | [payer_transfer_timeout.json](../../docker/ml-testing-toolkit/test-cases/collections/tests/golden_path/feature_tests/transfer_negative_scenarios/payer_transfer_timeout.json) (payee `noresponsepayeefsp`); or a transfer with amount **16** (TTK rule "No callback") | Transfer reserved, but no fulfil arrives; the timeout handler expires it (cron every 15 s) | `RESERVED → RESERVED_TIMEOUT → EXPIRED_RESERVED` (ABORTED); **3303** to payer **and** payee; positions restored |
| 5 | **Duplicate payment request** | [duplicate_handling/transfers/](../../docker/ml-testing-toolkit/test-cases/collections/tests/golden_path/feature_tests/duplicate_handling/transfers/) | Same ID + same body: ignored while `RECEIVED`, otherwise callback re-sent with the current state. Same ID + different body: rejected. | No second debit; **3106** Modified request for altered duplicates |
| 6 | Invalid fulfilment | Transfer with amount **15** (TTK rule), or [payee_invalid_fulfillment.json](../../docker/ml-testing-toolkit/test-cases/collections/tests/golden_path/feature_tests/transfer_negative_scenarios/payee_invalid_fulfillment.json) | Fulfil handler rejects the fulfilment | Aborted; payer gets error; payee gets `PATCH` with `ABORTED` |
| 7 | Payee rejects | [payee_abort_v1.1.json](../../docker/ml-testing-toolkit/test-cases/collections/tests/golden_path/feature_tests/transfer_negative_scenarios/payee_abort_v1.1.json) | Payee sends `transferState: ABORTED` | `RECEIVED_REJECT → ABORTED_REJECTED` |
| 8 | Inactive participant | [Active_Inactive_participants/](../../docker/ml-testing-toolkit/test-cases/collections/tests/golden_path/feature_tests/Active_Inactive_participants/) | Requests involving an inactive FSP or account are rejected | **3100** / **3201** ("Unsupported participant", "account is inactive") |
| 9 | Unknown transfer status | [get_transfers.json](../../docker/ml-testing-toolkit/test-cases/collections/tests/golden_path/feature_tests/get_transfers.json) | `GET /transfers/{unknownId}` | **3208** Transfer ID not found |
| 10 | Wrong API version | [fspiop_protocol_validation.json](../../docker/ml-testing-toolkit/test-cases/collections/tests/golden_path/feature_tests/backward_compatibility/fspiop_protocol_validation.json) | `Accept: …version=9.0` | **3001** Unacceptable version |

Special TTK amounts come from [rules_callback/default.json](../../docker/ml-testing-toolkit/spec_files/rules_callback/default.json): `POST /transfers` amount `15` → invalid fulfilment, `16` → no callback. `POST /fxTransfers` sourceAmount `15` / `16` behave the same way.

---

## 6. Troubleshooting a payment

### 6.1 Where to look, per phase

| Phase | Service logs | Kafka topics (debug UI) | DB evidence |
| --- | --- | --- | --- |
| Discovery | `account-lookup-service`, `als-msisdn-oracle-svc` | – (HTTP only) | `oracle_msisdn` records |
| Agreement | `quoting-service`, `quoting-service-handler` | `topic-quotes-post`, `topic-quotes-put` | quote tables in `central_ledger` |
| Transfer | `ml-api-adapter`, `central-ledger`, `central-handler-position-batch` | `topic-transfer-prepare`, `topic-transfer-position-batch`, `topic-transfer-fulfil`, `topic-notification-event` | `transfer`, `transferStateChange`, `participantPosition` |
| Callbacks | `ml-api-adapter` (notification handler), TTK | `topic-notification-event` | – |
| Audit | – | `topic-event-audit` | MongoDB `reporting` (Finance Portal profile) |

### 6.2 Symptom → cause → fix

| Symptom | Likely cause | How to confirm | Fix |
| --- | --- | --- | --- |
| `GET /parties` returns 202 but no callback | No oracle for the ID type; ID not registered; payee endpoint missing | ALS logs; `GET /oracles` on 4001; payee endpoints `GET /participants/{fsp}/endpoints` | Register oracle, ID or endpoints |
| 3204 for a known number | Registered under a different type or currency | Oracle DB / `GET /participants/MSISDN/{id}` | Re-register |
| Quote callback never arrives | Payee FSP `FSPIOP_CALLBACK_URL_QUOTES` missing or wrong; payee down | `quoting-service-handler` logs | Fix the endpoint |
| Transfer stuck `RESERVED` | Payee didn't fulfil | DB `transferStateChange`; TTK logs | Waits until expiry → 3303 (expected) |
| Immediate 4001 / 4200 | Payer position near NDC; no funds-in | `GET /participants/{fsp}/positions`, `/limits`, `/accounts` | Funds-in, raise NDC, or settle |
| 3106 on retry | Retry used the same ID with a different body (e.g. new `expiration`) | Compare payloads | Resend the identical body, or `GET` the status |
| 3001 / 3100 / 3102 | Version or header or body problem | Error description | Fix the client; check `Accept`/`Content-Type` versions |
| Callbacks reach no one | Callback URL not reachable from the switch's network | `ml-api-adapter` logs ("ECONNREFUSED") | Use DNS names reachable on `mojaloop-net` |
| Handlers not consuming | Kafka topics missing, or `kafka-provisioning` failed | `docker logs kafka-provisioning`; Kafka UI | Re-run provisioning; check broker health |
| Services restart in a loop | `wait4` gave up (DB or Kafka slow) or migration failed | `docker logs <svc>` | Increase Docker resources; restart; `down -v` for a clean DB |

### 6.3 A debugging routine [RECOMMENDATION]

1. Note the `transferId` (or `quoteId`) from the TTK report.
2. Search all logs: `docker compose logs --no-color | grep <transferId>` (PowerShell: `docker compose logs --no-color | Select-String <transferId>`).
3. Check its state history:

```bash
docker exec mysql mysql -ucentral_ledger -p<db-password> -D central_ledger -e "SELECT transferStateId, reason, createdDate FROM transferStateChange WHERE transferId='<transferId>' ORDER BY transferStateChangeId;"
```

4. Open the Kafka UI (`--profile debug`, <http://localhost:8080>) and search the transfer topics for the ID.
5. Ask the switch: `GET /transfers/{transferId}` from the TTK UI and inspect the callback.

Next: [15-production-deployment.md](15-production-deployment.md).
