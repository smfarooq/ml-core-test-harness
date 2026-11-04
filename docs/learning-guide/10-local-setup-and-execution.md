# 10 – Local Setup and Execution

> How to run a complete Mojaloop switch on your machine with this repository, verify it, and execute payments.
> Commands come from [README.md](../../README.md) and the compose files. Where this guide adds a command that is not in the README (health checks, inspection), it is marked **[RECOMMENDATION]** and is read-only.
>
> **Validation status:** verified end to end on 2026-11-04 on Windows 11 with Docker Desktop 4.97.0 (engine 29.9.2, Compose v5.6.1; 8 CPUs and 11.5 GiB for Docker). The standard stack, onboarding, the P2P test, health checks and inspection commands were run, and the corrections from that run are folded in below.

---

## 1. Which execution approaches exist?

| Approach | Supported by this repo? | Notes |
| --- | --- | --- |
| **Docker Compose** (this harness) | **Yes, the primary purpose** | Everything in this document |
| Running individual services from source | **No** (not here) | Each upstream repo (`mojaloop/central-ledger`, …) has its own README and dev compose **[EXTERNAL]**. This harness can still provide the dependencies; see §10. |
| Kubernetes | **Partly** | `k8s-mojaloop-perf-tuning/` only patches an **existing** cluster. The k6 runner `Dockerfile` targets k6-operator. |
| Official Mojaloop deployment tooling (Helm charts, IaC) | **No** | **[EXTERNAL]**. See [15](15-production-deployment.md). |

---

## 2. Prerequisites

| Requirement | Required version / note | Source | How to verify |
| --- | --- | --- | --- |
| **git** | any recent | README | `git --version` |
| **Docker Engine** | a recent version that supports Compose v2 | README ("docker") | `docker --version`, `docker info` |
| **Docker Compose** | Must support **profiles** and `depends_on.condition: service_completed_successfully`, so use Compose **v2** (`docker compose`). The README uses both `docker-compose` and `docker compose`. | README | `docker compose version` |
| Operating system | Not stated. Scripts are POSIX shell; images are **amd64**. Linux and macOS work; Windows works with Docker Desktop (WSL2 backend). Apple Silicon runs amd64 images under emulation (see `docker_arm_rebuild/`). | `docker_arm_rebuild/README.md` | – |
| Node.js | **Not required** to run the harness (`.nvmrc` 22.22.0 is only for release tooling) | `.nvmrc` | – |
| Kubernetes, Helm | Not required | – | – |
| Databases, Kafka | **Not installed manually.** They run as containers. | compose | – |
| Hardware | **Not documented in the repo.** **Measured:** the standard stack uses **about 4.5 GiB** of container memory at idle (`docker stats`), and its 15 images take **about 11.2 GB** of disk (the first pull took ~25 min). **[RECOMMENDATION]** Give Docker at least **8 GB RAM** (12–16 GB if you add the finance portal or monitoring), 4 CPUs and ~20 GB of free disk. | – | Docker Desktop → Settings → Resources; `docker stats --no-stream` |
| Free host ports | 9440, 9550, 9660 (TTK), 27017 (MongoDB), 6379, 6381, 6383, 6385, 6387, 6389, 6391 (Redis) | `docker-compose.yml` | `netstat -ano` (Windows) / `lsof -i` (macOS/Linux) |

**Windows note:** [.gitattributes](../../.gitattributes) disables CRLF conversion for `*.sh` and `*.js`, so scripts mounted into Linux containers keep LF line endings even with `core.autocrlf=true`. For the bash helper scripts (`perf-test.sh`, `automate_perf.sh`) use Git Bash or WSL. PowerShell examples are given where syntax differs.

---

## 3. Step-by-step installation and first run

### Step 1 – Clone the repository

```bash
git clone https://github.com/mojaloop/ml-core-test-harness.git
```

```bash
cd ml-core-test-harness
```

- **What / why:** get the compose files, configs and test collections.
- **Expected:** a folder with `docker-compose.yml`, `.env`, `docker/`, …
- **Errors:** corporate proxy blocking GitHub → configure `git config --global http.proxy`.

### Step 2 – Confirm prerequisites

```bash
docker compose version
```

- **Expected:** `Docker Compose version v2.x` or later.
- **Error:** `docker: 'compose' is not a docker command` → install or upgrade Docker Desktop / the compose plugin.

### Step 3 – Configure environment variables (normally nothing to change)

- [.env](../../.env) pins all image versions (`CENTRAL_LEDGER_VERSION=v19.4.4`, …). Compose reads it automatically.
- Keep the defaults for your first run. Change versions only deliberately ([13 §10](13-customization-and-extension.md#10-customise-deployment-configuration)).

### Step 4 – (Optional) Pre-pull images

```bash
docker compose --profile all-services --profile ttk-provisioning --profile ttk-tests pull
```

- **What / why:** downloads all images up front (several GB), so the first `up` is not slowed down or timed out by downloads.
- **Expected:** `Pulled` lines for each image.
- **Errors:** `pull access denied` or `manifest unknown` → check the version in `.env`; check network or proxy; check Docker Hub rate limits (`docker login`).

### Steps 5–10 – Start everything (databases, Kafka, services, migrations, onboarding, tests)

From the README:

```bash
docker compose --profile all-services --profile ttk-provisioning --profile ttk-tests up
```

**What this one command does, in order** (driven by `depends_on` conditions in [docker-compose.yml](../../docker-compose.yml)):

| Order | Container(s) | Action | Configured by |
| --- | --- | --- | --- |
| 1 | `kafka` | Start a KRaft broker (internal listener `kafka:29092`) | compose env |
| 2 | `kafka-provisioning` | **Configure messaging**: create 23 topics, then exit 0 | [provision.sh](../../docker/kafka/scripts/provision.sh) |
| 3 | `mysql`, `mysql-als`, `objstore`, `redis*` | **Configure databases**: run `docker/sql-init/*.sql`, `docker/sql-als-init/*.sql` | SQL init files |
| 4 | `central-ledger` | Patch config → wait for Kafka and MySQL → **run DB migrations** (upstream default `MIGRATIONS.DISABLED=false`) → start API + handlers | config-modifier, wait4 |
| 5 | `central-handler-position-batch`, `ml-api-adapter`, `quoting-service`(+handler), `central-settlement` | Start | config-modifier |
| 6 | `simulator` → `account-lookup-service`, `als-msisdn-oracle-svc` | `npm run migrate` then start | wait4 |
| 7 | `mojaloop-testing-toolkit`, UI, SDK adapters | Start the simulated FSPs | TTK spec files, `envs/*.env` |
| 8 | `ttk-provisioning` | When all core services are **healthy**: **seed data**. Creates hub accounts, settlement models, oracles, participants `testingtoolkitdfsp` and `payeefsp`, callback URLs, NDC limits, funds-in, FXP. Report: `reports/ttk-provisioning-report.html` | `collections/provisioning` (`--labels std`) |
| 9 | `ttk-tests` | When provisioning completes successfully: **runs the P2P payment test**. Report: `reports/ttk-func-tests-report.html` | `collections/tests` (`--labels std`) |

- **Where to run:** the repository root (where `docker-compose.yml` is).
- **Expected time (measured):** about 5.5 minutes from `up` until the test container starts, with all images already pulled. Health checks use 30 s intervals with many retries.
- **Expected final output** in the `ttk-tests` logs. At this commit the P2P test has **31 assertions**; the README's sample, from 2022, shows 27. This is the verified output of a passing run:

```text
┌───────────────────────────────────────────────────┐
│                      SUMMARY                      │
├───────────────────┬───────────────────────────────┤
│ Total assertions  │ 31                            │
│ Passed assertions │ 31                            │
│ Failed assertions │ 0                             │
│ Total requests    │ 4                             │
│ Total test cases  │ 1                             │
│ Passed percentage │ 100.00%                       │
└───────────────────┴───────────────────────────────┘
```

The 4 requests are: register the payee in ALS, look up the party, request the quote, execute the transfer ([tests/p2p.json](../../docker/ml-testing-toolkit/test-cases/collections/tests/p2p.json)).

> **Cold-start caveat (observed during verification):** the **first** `ttk-tests` run, straight after start-up, reported **27/31** and exited with code 1. The 4 failures were all on the transfer callback. The ledger showed the transfer `COMMITTED`, but the final callback arrived 21 s after the prepare, which is longer than the TTK's 15 s wait, while Kafka consumer groups were still settling. Re-running on the warm stack (Step 12) gave **31/31**. Treat a first-run transfer-callback timeout as start-up noise, and check the ledger before assuming a defect.

**Detached mode [RECOMMENDATION]:** add `-d` to get your terminal back, then follow the logs of one container:

```bash
docker compose --profile all-services --profile ttk-provisioning --profile ttk-tests up -d
```

```bash
docker compose logs -f ttk-tests
```

### Step 11 – Verify service health

See [§6](#6-verification). Quick check:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
```

- **Expected:** `(healthy)` for central-ledger, ml-api-adapter, quoting-service, quoting-service-handler, account-lookup-service, als-msisdn-oracle-svc, simulator, mojaloop-testing-toolkit and central-handler-position-batch. `kafka-provisioning`, `ttk-provisioning` and `ttk-tests` should be **Exited (0)**.
- **Known (verified):** `central-settlement` shows *health: starting*, then **unhealthy**, because its health check probes port 3001 (connection refused) while the service listens on **3007**, under the path `/v2/health` ([06 §6](06-microservices-deep-dive.md#6-central-settlement-central-settlement)).
- `up -d` with these profiles does **not** return immediately. It waits until onboarding has exited, then starts `ttk-tests`.

### Step 12 – Run automated tests again (with the stack still running)

**Verified working command**, in a second terminal:

```bash
docker compose --profile all-services --profile ttk-provisioning --profile ttk-tests run --rm --no-deps ttk-tests
```

- **What:** runs a fresh one-off `ttk-tests` container in the same project, without starting or recreating its dependencies.
- **Expected (verified):** `Passed assertions 31 / Failed 0`, exit code 0; a new HTML report in `reports/`.
- **Note:** `--no-deps` does not wait for anything, so make sure the switch is healthy first.
- **Do not use the README's version** (`docker compose --project-name ttk-test-only --profile ttk-tests up --no-deps`). On Docker Compose v5.6.1 it fails with `service "ttk-tests" depends on undefined service "quoting-service": invalid compose project`, because with only the `ttk-tests` profile the services it depends on aren't defined.

---

## 4. Where to see results

| What | Where |
| --- | --- |
| Test reports (HTML) | `./reports/ttk-provisioning-report.html`, `./reports/ttk-func-tests-report.html` |
| TTK report history | <http://localhost:9660/admin/reports> |
| TTK UI | <http://localhost:9660> |
| Mobile simulator (live sequence diagram) | <http://localhost:9660/mobilesimulator> |
| Container logs | `docker compose logs -f <service>` |

---

## 5. Environment configuration reference

### 5.1 Configuration files

| File | Controls | Edit it when |
| --- | --- | --- |
| [.env](../../.env) | Image versions, replica counts, dependency versions | Testing another Mojaloop release |
| `docker/config-modifier/configs/<service>.js` | Each service's `default.json` overrides: DB host, Kafka brokers, caches, JWS, routing modes | Changing service behaviour |
| `docker/event_sdk/config.json` | Audit, log and trace event outputs | Turning tracing on or off |
| `envs/<fsp>-sdk.env` | Each SDK connector: FSP id, endpoints, JWS/mTLS, auto-accept flags, timeouts | Changing connector behaviour |
| `docker/ml-testing-toolkit/test-cases/environments/default-env.json` | Test inputs: FSP ids, MSISDNs, currencies, URLs, amounts | Changing test data |
| `docker/ml-testing-toolkit/spec_files/user_config.json`, `system_config.json` | TTK behaviour (callbacks, JWS, validation, labels, DB) | Changing simulation |
| `perf.env`, `perf.override.env` | Perf and load stacks (must create `perf.override.env`) | Performance testing |

### 5.2 Important settings, with placeholders for secrets

```dotenv
# ---- .env (image pins) ----
CENTRAL_LEDGER_VERSION=v19.4.4
ML_API_ADAPTER_VERSION=v16.4.2
ACCOUNT_LOOKUP_SERVICE_VERSION=v17.9.1
QUOTING_SERVICE_VERSION=v17.8.0
CENTRAL_SETTLEMENT_VERSION=v17.0.6
SDK_SCHEME_ADAPTER_VERSION=v24.7.0
DEP_KAFKA_VERSION=3.9.0
DEP_MYSQL_VERSION=8.0

# ---- Database connection (compose defaults; override via env) ----
DBUSER=central_ledger
DBPASS=<db-password>

# ---- Per-service env override pattern (prefix + __ nesting) ----
CLEDG_DATABASE__HOST=mysql
CLEDG_DATABASE__USER=central_ledger
CLEDG_DATABASE__PASSWORD=<db-password>
CLEDG_KAFKA__CONSUMER__TRANSFER__PREPARE__config__rdkafkaConf__metadata.broker.list=kafka:29092

# ---- SDK connector (envs/<fsp>-sdk.env) ----
DFSP_ID=banka
ALS_ENDPOINT=account-lookup-service:4002
QUOTES_ENDPOINT=quoting-service:3002
TRANSFERS_ENDPOINT=ml-api-adapter:3000
BACKEND_ENDPOINT=<core-banking-adapter-host>:<port>
CACHE_URL=redis://<redis-host>:6379
ILP_SECRET=<long-random-secret>
JWS_SIGN=true
JWS_SIGNING_KEY_PATH=secrets/jwsSigningKey.key
INBOUND_MUTUAL_TLS_ENABLED=true
OUTBOUND_MUTUAL_TLS_ENABLED=true
AUTO_ACCEPT_PARTY=false
AUTO_ACCEPT_QUOTES=false
```

| Concern | Where it is set in the harness |
| --- | --- |
| DB connection | `DATABASE.HOST` in `configs/*.js` (`mysql`, `mysql-als`); users and passwords from compose env and `docker/sql-*` |
| Service URLs | `SWITCH_ENDPOINT` / `ENDPOINT_SOURCE_URL` = `http://central-ledger:3001`; TTK `CALLBACK_RESOURCE_ENDPOINTS` |
| Broker | `metadata.broker.list: kafka:29092` everywhere |
| API endpoints for FSPs | Registered per participant through `POST /participants/{name}/endpoints` (provisioning) |
| Certificates and secrets | `docker/secrets/`, `docker/security/` (test) |
| Feature flags | `SIMPLE_ROUTING_MODE`, `PROXY_CACHE.enabled`, `MONGODB.DISABLED`, `HANDLERS.DISABLED`, `JWS_SIGN`, SDK `AUTO_ACCEPT_*`, `ENABLE_TEST_FEATURES`, `API_TYPE` |

---

## 6. Verification

Switch ports are **not published** to the host (they are commented out in compose), so check health **inside** the containers. Their own health checks use `wget`, so it is available. **[RECOMMENDATION]** (read-only commands):

| Check | Command | Expected |
| --- | --- | --- |
| Central ledger | `docker exec central-ledger wget -qO- http://127.0.0.1:3001/health` | JSON with `"status":"OK"` and sub-service statuses (DB, broker) |
| ML API adapter | `docker exec ml-api-adapter wget -qO- http://127.0.0.1:3000/health` | `"status":"OK"` |
| ALS | `docker exec account-lookup-service wget -qO- http://127.0.0.1:4001/health` | `"status":"OK"` |
| Quoting service | `docker exec quoting-service wget -qO- http://127.0.0.1:3002/health` | `"status":"OK"` |
| Settlement | `docker exec central-settlement wget -qO- http://127.0.0.1:3007/v2/health` | `"status":"OK"` (note port **3007** and the **`/v2`** prefix; plain `/health` returns 404) |
| Participants onboarded | `docker exec central-ledger wget -qO- http://127.0.0.1:3001/participants` | List including `Hub`, `testingtoolkitdfsp`, `payeefsp` |
| Positions | `docker exec central-ledger wget -qO- http://127.0.0.1:3001/participants/testingtoolkitdfsp/positions` | Position per currency |
| Database | `docker exec mysql mysql -ucentral_ledger -p<db-password> -D central_ledger -e "SELECT transferId, transferStateId FROM transferStateChange ORDER BY transferStateChangeId DESC LIMIT 5;"` | Recent `COMMITTED` rows after the test |
| Kafka topics | `docker exec kafka /opt/bitnami/kafka/bin/kafka-topics.sh --bootstrap-server kafka:29092 --list` | The 23 topics |
| Kafka messages (UI) | Start with `--profile debug`, open <http://localhost:8080> | Redpanda Console showing topics |
| Logs | `docker compose logs -f central-ledger ml-api-adapter` | No repeated errors |
| Monitoring | See §9.7 | Dashboards on <http://localhost:9999> |
| Sample payment | §8 | `transferState: COMMITTED` |

(The DB password default is in `docker-compose.yml`'s `DBPASS` default and `docker/sql-init/`. It is a test value; substitute it yourself.)

---

## 7. Common errors and troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `mysql` fails or restarts on the very first `up` | Possible per the compose comment at `docker-compose.yml:178` ("…docker-compose up will fail on first attempt"); **not** observed in verification | Run the same `up` command again |
| Docker Desktop "unable to start"; log `%LOCALAPPDATA%\Docker\log\host\com.docker.backend.exe.log` shows `MountDisk/HCS/E_ACCESSDENIED` | Docker's disk image location was moved to another drive, and Windows' virtual-machine group can't open it (seen during verification) | Move it back via Settings → Resources → Advanced → *Disk image location*, or (as administrator) `icacls "<folder>" /grant "*S-1-5-83-0:(OI)(CI)F" /T`, then `wsl --shutdown` and restart Docker Desktop |
| `docker compose down -v` "works" but everything keeps running | Every service is behind a profile. Without `--profile` flags Compose sees an **empty project** and removes nothing (verified) | Repeat the same `--profile` flags you used for `up` |
| Commands silently fail in **Git Bash** (e.g. `docker exec kafka /opt/bitnami/...`) | Git Bash rewrites Linux-style paths (`/opt/...`, `/data`) into Windows paths | `export MSYS_NO_PATHCONV=1`, or use PowerShell |
| `400 / 3101 "Invalid date header"` on hand-made FSPIOP requests | The `Date` header's weekday doesn't match the date (the date only has to be valid, not recent) | Generate it: PowerShell `[DateTime]::UtcNow.ToString('r')`, bash `date -u '+%a, %d %b %Y %H:%M:%S GMT'` |
| First P2P test run fails 4 transfer-callback assertions | Cold start (see Steps 5–10) | Re-run with Step 12 |
| `ttk-provisioning` never starts | A dependency is not `healthy` (health checks retry for up to ~15 min) | `docker ps`; check the unhealthy container's logs |
| Provisioning assertions fail with "400 if exists" | You re-ran provisioning on existing volumes | Expected and tolerated: tests accept 201 *or* 400/500 for "already exists" |
| Port already allocated (27017, 6379, 9660…) | Another local MongoDB, Redis or app is running | Stop it, or change the host port mapping |
| `Out of memory` / containers killed (exit 137) | Docker memory too low | Raise the Docker Desktop memory limit |
| Very slow on Apple Silicon | amd64 emulation | Use `docker_arm_rebuild/rebuild.sh` (builds arm64 images from source) |
| `env file …/perf.override.env not found` | Perf/load stacks need the git-ignored file | Copy `perf.override.sample.env` to `perf.override.env` (it can stay all-commented) |
| `service "merchant-db" refers to undefined volume your_local_path_to_merchant_registry_sql_scripts` | You enabled `--profile '*'` or `merchant-registry` | Don't use those profiles, or replace the placeholder paths |
| `network mojaloop-net declared as external, but could not be found` | Monitoring (or load) compose started before any switch stack | `docker network create mojaloop-net`, or start the switch first |
| TTK UI loads but shows "cannot connect" | The UI calls `API_BASE_URL=http://localhost:9550` from your browser | Use the browser on the same machine, or change `API_BASE_URL` to the host IP |
| `central-settlement` unhealthy | Health check on the wrong port | Ignore, or check port 3007 manually |
| Golden-path report shows many "skipped" assertions | Tests for transaction requests, authorizations and some JWS checks are skipped by their own conditions, because those services or features are not in this harness | Expected. The verified run had 453 skipped and **0 failed** (§9.4) |

Container won't start? `docker compose logs <service>` and look for `config-modifier Error` (bad override), `wait4 Error` (dependency unreachable) or migration errors.

---

## 8. Execute a sample payment

### 8.1 Automatically (already done by `ttk-tests`)

Look at `reports/ttk-func-tests-report.html`. Each request shows the request, the immediate response and the **callback** received by the TTK.

### 8.2 Through the TTK web UI (README procedure)

1. Open <http://localhost:9660> → **Test Runner**.
2. **Collection Manager** → **Import Folder** → choose `docker/ml-testing-toolkit/test-cases/collections/tests`.
3. Select `p2p.json`, then click the masked area on the right to close the manager.
4. Click **Run** (top right). All tests should pass.
5. Click **Edit** next to the test case to explore each request, response and callback.

### 8.3 Through the mobile simulator

Open <http://localhost:9660/mobilesimulator>. You will see two virtual phones (sender and receiver) and a **live sequence diagram** of every Mojaloop message. The sender and receiver FSPs are `pinkbankfsp` and `greenbankfsp` (`mobileSimPayerFsp`, `mobileSimPayeeFsp` in `default-env.json`). This is the best way to *see* the three phases.

### 8.4 Expected successful result

- Party callback with `fspId: payeefsp`.
- Quote callback with `transferAmount`, `ilpPacket`, `condition`.
- Transfer callback with `transferState: COMMITTED`.
- The payer's position increased and the payee's decreased by the amount (`GET /participants/{name}/positions`).

---

## 9. Other run modes

### 9.1 Core services only (no onboarding)

```bash
docker compose --profile all-services up
```

### 9.2 With Kafka debug UI

```bash
docker compose --profile all-services --profile debug up
```

### 9.3 FX (cross-currency) tests

```bash
docker compose --profile testing-toolkit --profile fx --profile ttk-provisioning-fx --profile ttk-fx-tests --profile debug up -d
```

Stop and remove volumes:

```bash
docker compose --profile testing-toolkit --profile fx --profile ttk-provisioning-fx --profile ttk-fx-tests --profile debug down -v
```

### 9.4 Golden-path regression

```bash
docker compose --profile all-services --profile ttk-provisioning-gp --profile ttk-tests-gp up
```

To re-run the suite on a stack that is already up and provisioned:

```bash
docker compose --profile all-services --profile ttk-provisioning-gp --profile ttk-tests-gp run --rm --no-deps ttk-tests-gp
```

**Verified (2026-11-04):** provisioning passed **460/460** assertions. The full suite then ran **124 test cases** (583 requests, 2,625 assertions) in about **7.5 minutes**: **2,172 passed, 453 skipped, 0 failed**. The summary table counts skipped assertions as passed, so it shows "2625 passed, 100%". The skips are in the transaction-request, authorization and JWS-validation tests, which depend on services or features this harness doesn't run.

### 9.5 Finance Portal (back-office UI)

```bash
docker compose --profile all-services --profile fx --profile finance-portal --profile ttk-provisioning-fx --profile ttk-fx-tests --profile debug up -d
```

Open <http://localhost:9400> (shell), with APIs proxied by `fp-proxy`.

**Verified (2026-11-04), starting from an empty project:**

| Check | Result |
| --- | --- |
| `up -d` | Exit 0 after **~14.5 min**, including pulling 8 extra images. It returns only after FX provisioning has finished. |
| FX provisioning (`ttk-provisioning-fx`) | 15 test cases, **223/223** assertions |
| FX tests (`ttk-fx-tests`) | 4 test cases, 77 assertions: **73 passed, 4 skipped, 0 failed** (36 s) |
| Memory (`docker stats`) | 37 containers, **~5.4 GiB** at idle. cAdvisor alone uses ~800 MiB. |
| Portal → **Transfers** | Works: lists the FX test transfers, with filters and charts by currency, payer and payee |
| Portal → **Financial Positions** | Works: balance, position and NDC per DFSP and currency, with update buttons |
| Portal → **Roles** and **Settlements** | **Fail**: "Webpack Loading Error: Error loading from http://localhost:9301/app.js" (9303 for Settlements). The shell lists four apps, but compose defines only the transfers (9302) and positions (9304) UIs. |
| Other ports | Kafka UI (`debug` profile) on 8080; the finance-portal profile's own cAdvisor on 8082; TTK UI on 9660 |

### 9.6 Partial stacks by phase or service

```bash
docker compose --profile discovery up
```

```bash
docker compose --profile quoting-service --profile central-ledger up
```

Other values: `agreement`, `transfer`, `central-ledger`, `ml-api-adapter`, `account-lookup-service` (the last three need `--profile central-ledger` too).

### 9.7 Monitoring stack

```bash
docker compose --project-name monitoring -f docker-compose-monitoring.yml up -d
```

Grafana <http://localhost:9999> (admin password in `docker/grafana/config.env`), Prometheus <http://localhost:9090>, Kafka console <http://localhost:9080>.

The monitoring compose declares `mojaloop-net` as **`external: true`**, so that network must already exist. On a fresh machine, start a switch stack first (it creates `mojaloop-net`), or create the network once yourself **[RECOMMENDATION]**:

```bash
docker network create mojaloop-net
```

(The README's own TODO notes this: "add note about network being created by docker-compose-perf.yml, or it can be done manually". The load compose additionally needs `monitoring-net`, which the monitoring compose creates.) Prometheus targets are named for the **perf** stack (`ml-core-*` containers), so most Mojaloop dashboards fill only with `docker-compose-perf.yml`. cAdvisor uses host port **8080**, which clashes with the `debug` profile's Kafka UI.

**Verified on Windows / Docker Desktop (2026-11-04)**, started next to the finance-portal stack (§9.5):

| Check | Result |
| --- | --- |
| `up -d` | **Exit 1** after ~6 min (mostly image pulls): `path / is mounted on / but it is not a shared or slave mount`. `node-exporter` mounts the host root with `rslave` propagation, which Docker Desktop doesn't support. Because the command aborted, **cAdvisor, Prometheus and Grafana were left in `Created`**. Only `kafka-exporter`, `kafka-mon-ui` and `renderer` ran. |
| cAdvisor, started alone | Fails: `Bind for 0.0.0.0:8080 failed: port is already allocated` (the `debug` profile's Kafka UI). The clash is real. |
| Prometheus and Grafana, started with `--no-deps` | Both run: Prometheus ready on 9090, Grafana healthy on 9999. **3 of 73 scrape targets are up** (Prometheus, kafka-exporter, central-ledger). The rest point to perf-stack container names. |
| Memory | The five containers that ran use about 215 MiB in total |

To get a working subset on Windows without changing any file **[RECOMMENDATION]**:

```bash
docker compose --project-name monitoring -f docker-compose-monitoring.yml up -d --no-deps prometheus grafana kafka-exporter kafka-mon-ui
```

To stop it (it is a separate Compose project, so the switch stack's `down` doesn't touch it). Run this **before** stopping the switch stack, because its containers hold on to `mojaloop-net`:

```bash
docker compose --project-name monitoring -f docker-compose-monitoring.yml down -v
```

### 9.8 Performance and load

Create the override file first (bash shown; PowerShell: `Copy-Item perf.override.sample.env perf.override.env`):

```bash
cp perf.override.sample.env perf.override.env
```

Then, per the README quick start:

```bash
docker compose --project-name monitoring -f docker-compose-monitoring.yml up -d
```

```bash
docker compose --project-name ml-core -f docker-compose-perf.yml --profile all-services --profile 8dfsp --profile fx-sdk-e2e --profile ttk-provisioning-e2e up -d
```

```bash
env K6_SCRIPT_CONFIG_FILE_NAME=fxSendE2E.json docker compose --project-name load -f docker-compose-load.yml up
```

PowerShell equivalent of the last command: `$env:K6_SCRIPT_CONFIG_FILE_NAME='fxSendE2E.json'; docker compose --project-name load -f docker-compose-load.yml up`. More scenarios: [14 §4](14-testing-and-troubleshooting.md#4-performance-and-load-testing).

### 9.9 Security harness

```bash
docker compose --project-name security -f docker-compose-security.yml --profile security-sdk-scheme-adapter up
```

---

## 10. Running an individual service from source (developer workflow) [RECOMMENDATION]

The harness does not do this, but you can combine it with upstream code:

1. Start only the dependencies, e.g. `docker compose --profile central-ledger up`, which gives Kafka, MySQL, MongoDB and central-ledger.
2. Stop the container you want to replace (`docker stop central-ledger`).
3. Clone the upstream repo at the pinned tag (`git clone https://github.com/mojaloop/central-ledger && git checkout v19.4.4`), `npm ci`, and point its config at `localhost`. **Note:** MySQL and Kafka ports are **not** published by default; uncomment the `ports:` entries in your local copy, or run your service as a container on `mojaloop-net`.
4. Follow the upstream repo's README for exact commands **[EXTERNAL]**.

---

## 11. Stopping and cleaning up

```bash
docker compose --profile all-services --profile ttk-provisioning --profile ttk-tests down -v
```

- `down` removes this project's containers and network. `-v` also removes its **volumes** (database data), giving a clean slate next time.

> **Warning:** `./docker-clean-all.sh` runs `docker rm -f $(docker ps -a -q)` and `docker volume rm $(docker volume ls -q)`. That deletes **every container and volume on your machine**, including other projects. Prefer the scoped `down -v` above.

Next: [11-fsp-integration-guide.md](11-fsp-integration-guide.md).
