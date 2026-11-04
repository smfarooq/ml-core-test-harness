# Mojaloop Learning Guide (ml-core-test-harness)

This guide teaches Mojaloop from first principles. It starts with what a payment switch is and why interoperability matters. It then covers how this repository runs a complete Mojaloop switch on your laptop, and how a bank or wallet provider would build its own integration.

It is written for someone exploring Mojaloop in depth for the first time. Every technical claim is tied back to a file in this repository, or clearly marked as coming from the upstream Mojaloop projects and documentation.

---

## 1. The single most important thing to know first

> **This repository is not the source code of the Mojaloop switch.**
>
> `ml-core-test-harness` is a **Docker Compose test harness**. It downloads **pre-built Docker images** of the real Mojaloop services from Docker Hub, including `mojaloop/central-ledger`, `mojaloop/ml-api-adapter`, `mojaloop/account-lookup-service` and `mojaloop/quoting-service`. It wires them together with configuration overrides, onboards test financial institutions, and runs automated payment tests and load tests against them.

What lives **in this repository**:

| Lives here | Lives elsewhere (upstream Mojaloop repos) |
| --- | --- |
| `docker-compose*.yml` files that assemble the switch | The Node.js source code of every switch service |
| Version pins for every image (`.env`) | Helm charts for Kubernetes production deployments |
| Configuration overrides (`docker/config-modifier/configs/*.js`) | The FSPIOP API specification documents (copies are here as test fixtures) |
| Database / Kafka bootstrap scripts | Database migrations and seed data (they run inside the images) |
| Mojaloop Testing Toolkit (TTK) collections: onboarding plus 131 functional test cases | |
| k6 load-test scripts, Grafana dashboards, Prometheus config | |
| Sample certificates and keys for security testing | |

That makes this repository the ideal **first** hands-on step. You can run a real switch, send a real payment through it, and watch every message, all without a Kubernetes cluster. To read the switch's **internal code**, you will follow the pinned image tags to the upstream repositories (see [06-microservices-deep-dive.md](06-microservices-deep-dive.md)).

---

## 2. Repository snapshot analysed

| Item | Value | Evidence |
| --- | --- | --- |
| Repository | `mojaloop/ml-core-test-harness` | [README.md](../../README.md) |
| Package version | `2.15.1` | [package.json](../../package.json) |
| Latest tag | `v2.15.1` (3 commits ahead: `v2.15.1-3-g2187520`) | `git describe --tags` |
| Commit | `2187520` – "Merge pull request #145 … ttk-ui-hardcoded-path" | `git log` |
| Analysed on | 2026-11-04 | – |
| Core switch images | central-ledger `v19.4.4`, ml-api-adapter `v16.4.2`, account-lookup-service `v17.9.1`, quoting-service `v17.8.0`, central-settlement `v17.0.6`, als-msisdn-oracle-svc `v0.0.18` | [.env](../../.env) |
| Connector image | sdk-scheme-adapter `v24.7.0` | [.env](../../.env) |
| Testing images | ml-testing-toolkit `v18.11.2`, TTK UI `v16.2.0`, TTK CLI `v1.9.1`, simulator `v12.2.4`, callback-handler simulator `v0.1.5` | [.env](../../.env) |

---

## 3. How to read this guide

Read the documents in order the first time. Each one builds on the vocabulary of the previous ones.

| # | Document | What you will learn |
| --- | --- | --- |
| 00 | **00-README.md** (this file) | Orientation, evidence conventions, quick answers |
| 01 | [01-mojaloop-fundamentals.md](01-mojaloop-fundamentals.md) | What Mojaloop is, payment switches, interoperability, the actors |
| 02 | [02-repository-structure.md](02-repository-structure.md) | Every directory in this repo and why it exists |
| 03 | [03-technology-stack.md](03-technology-stack.md) | Every technology used, why, where, mandatory or optional |
| 04 | [04-business-use-cases.md](04-business-use-cases.md) | Which payment use cases exist, which run in this harness, plus 6 real-world scenarios |
| 05 | [05-system-architecture.md](05-system-architecture.md) | High-level and component architecture, communication patterns |
| 06 | [06-microservices-deep-dive.md](06-microservices-deep-dive.md) | Service-by-service technical reference |
| 07 | [07-payment-transaction-lifecycle.md](07-payment-transaction-lifecycle.md) | A payment from tap to settlement, states, timeouts, idempotency |
| 08 | [08-fspiop-protocol-and-apis.md](08-fspiop-protocol-and-apis.md) | The FSPIOP protocol, headers, API catalogue, annotated examples |
| 09 | [09-security-and-controls.md](09-security-and-controls.md) | JWS, mTLS, participant identity, what the switch does vs. what the FSP must do |
| 10 | [10-local-setup-and-execution.md](10-local-setup-and-execution.md) | Install, run, verify, and execute your first payment |
| 11 | [11-fsp-integration-guide.md](11-fsp-integration-guide.md) | How "Bank A" connects to Mojaloop; a .NET reference design |
| 12 | [12-adapter-development.md](12-adapter-development.md) | FSPIOP adapters, the SDK Scheme Adapter, mapping and idempotency |
| 13 | [13-customization-and-extension.md](13-customization-and-extension.md) | Adding FSPs, currencies, oracles, monitoring, extending services |
| 14 | [14-testing-and-troubleshooting.md](14-testing-and-troubleshooting.md) | Testing strategy, test scenarios, common problems |
| 15 | [15-production-deployment.md](15-production-deployment.md) | Production architecture, Kubernetes, HA, operations, team roles |
| 16 | [16-glossary.md](16-glossary.md) | Financial and technical terms |
| 17 | [17-learning-roadmap.md](17-learning-roadmap.md) | A five-level learning plan with exercises and completion criteria |

### Structured, hands-on path: the Solution Architect Learning Project

If you want a guided programme with labs, checkpoints and portfolio deliverables, follow **[sa-learning-project/](sa-learning-project/00-project-charter.md)**. It covers FSPIOP → Central Ledger → Account Lookup → Quoting → Transfers → Fulfilment → FSP Adapter → Settlement → Kubernetes, interleaved with ten hands-on milestones from business architecture to connecting a real bank, and ends with a capstone (bank briefing, solution design, implementation plan). The documents below are its reference material.

Larger diagrams are also stored as standalone Mermaid files under [diagrams/](diagrams/). Open them with any Mermaid viewer, the VS Code Mermaid extension, or GitHub's renderer.

---

## 4. Evidence conventions used throughout

Every capability is tagged with **where the evidence comes from**, so you never confuse "Mojaloop can do this" with "this repository runs this":

| Tag | Meaning |
| --- | --- |
| **[IMPLEMENTED]** | Runs in this harness with the provided compose files and test collections. |
| **[PARTIAL]** | Some pieces are present (API spec, topics, tests or config) but a required service is missing, or it needs external code. |
| **[CONFIG]** | Behaviour is switched on or off, or tuned, purely through configuration in this repo. |
| **[EXTERNAL]** | Lives in another Mojaloop repository or product (for example Helm charts or upstream service source code). |
| **[UPSTREAM-DOC]** | Taken from official Mojaloop documentation or upstream source, not verifiable from this repo alone. |
| **[RECOMMENDATION]** | Advice from this guide's author. Not defined by the repository. |

File references are relative links from this folder, for example [docker-compose.yml](../../docker-compose.yml). Where a line matters, it is cited as `docker-compose.yml:9`.

**Secrets policy:** the repository contains test keys, test certificates and sample passwords (for example `docker/secrets/jwsSigningKey.key`, `docker/security/**`, values in `envs/*.env`). This guide names those files but **never reproduces their contents**. Examples use `<placeholder>` values instead.

---

## 5. Quick answers to the key questions

These are short answers. Each links to the full explanation.

1. **What exactly is Mojaloop?** An open-source, real-time payment switch, the "hub" of a payment scheme. It lets different banks, mobile-money operators and other financial service providers send money to each other's customers. See [01](01-mojaloop-fundamentals.md).
2. **What business problems does it solve?** Closed-loop wallets that cannot pay each other, expensive bilateral integrations, slow and costly settlement, and the lack of affordable infrastructure for financial inclusion. See [01 §2](01-mojaloop-fundamentals.md#2-why-the-mojaloop-foundation-created-it).
3. **How does a payment switch work?** It routes payment messages between institutions, guarantees that both sides agree on the outcome, tracks what each institution owes, and supports later settlement of those obligations. See [01 §3](01-mojaloop-fundamentals.md#3-what-is-a-payment-switch).
4. **Which use cases does Mojaloop support?** P2P, merchant payments, cash-in and cash-out, bulk, request-to-pay, FX (cross-currency) and more. This harness runs **P2P, FX, settlement, funds-in/funds-out and duplicate/timeout handling**. See [04](04-business-use-cases.md).
5. **How do banks and wallets communicate through Mojaloop?** Through the asynchronous REST **FSPIOP API**, in three phases: **Discovery** (`/parties`), **Agreement** (`/quotes`) and **Transfer** (`/transfers`). The switch forwards each request and the matching callback. See [08](08-fspiop-protocol-and-apis.md).
6. **What happens internally when a payment is initiated?** The ML API Adapter accepts the request, Kafka carries it to the Central Ledger handlers (prepare, position, fulfil), the ledger reserves and commits positions in MySQL, and the notification handler delivers callbacks. See [07](07-payment-transaction-lifecycle.md).
7. **What are the responsibilities of each microservice?** See the table in [05 §3](05-system-architecture.md#3-component-responsibilities) and the deep dive in [06](06-microservices-deep-dive.md).
8. **Which technologies are used?** Node.js services in Docker, Kafka (KRaft), MySQL 8, MongoDB, Redis, OpenAPI/FSPIOP, ILP conditions, JWS, the Testing Toolkit, k6, Prometheus and Grafana. See [03](03-technology-stack.md).
9. **How do all the components interact?** HTTP at the edges, Kafka inside the transfer and quote pipelines, MySQL as the system of record, callbacks back to FSPs. See [05](05-system-architecture.md).
10. **How do I install and run Mojaloop locally?** `docker compose --profile all-services --profile ttk-provisioning --profile ttk-tests up`. See [10](10-local-setup-and-execution.md).
11. **How do I execute an end-to-end payment?** The `ttk-tests` container runs [tests/p2p.json](../../docker/ml-testing-toolkit/test-cases/collections/tests/p2p.json) automatically. You can also use the TTK web UI or the mobile simulator at `http://localhost:9660`. See [10 §8](10-local-setup-and-execution.md#8-execute-a-sample-payment).
12. **How do I integrate a bank's core banking system?** Put a connector (usually the **SDK Scheme Adapter**) between the core system and the switch. Implement its simple backend API and call its outbound API. See [11](11-fsp-integration-guide.md).
13. **How do I build an FSP adapter?** See [12](12-adapter-development.md).
14. **How do I develop a new application using Mojaloop?** See the .NET 10 reference design in [11 §8](11-fsp-integration-guide.md#8-reference-application-bank-a--integration-service--mojaloop--bank-b).
15. **How do I customize the payment switch?** Mostly through configuration and onboarding API calls: participants, currencies, limits, oracles, settlement models. Code changes go upstream. See [13](13-customization-and-extension.md).
16. **How do I deploy it in production?** On Kubernetes, using the upstream Mojaloop Helm charts or IaC, with highly available Kafka and MySQL, mTLS and JWS, and full observability. This repo does **not** contain production deployment assets. See [15](15-production-deployment.md).
17. **What would I need to build my own interoperable ecosystem?** A scheme operator, rules and governance, a settlement bank, onboarded FSPs with connectors, oracles for identifiers, PKI and security, operations, and monitoring. See [15 §10](15-production-deployment.md#10-what-it-takes-to-run-your-own-interoperable-ecosystem).

---

## 6. Known limitations of this repository (summary)

Detailed in the relevant documents; collected here so you are not surprised:

- **No switch source code.** Services are consumed as Docker images (see §1).
- **No Helm charts or production manifests.** `k8s-mojaloop-perf-tuning/` only patches an *existing* Kubernetes deployment for performance experiments.
- **Some README sections are stale.** "Settlements – TODO" and "Bulk – TODO": `central-settlement` *is* now part of `all-services`, but no bulk API adapter exists. The README also refers to `packages/k6-tests/scripts/test.js`, which does not exist.
- **Transaction Requests Service is not defined** in any compose file. The legacy simulator and the golden-path collections reference `transaction-requests-service`, so request-to-pay flows cannot complete here.
- **The `merchant-registry` profile is a template.** Its services build from placeholder paths (`your/local/path/to/...`), so `docker compose --profile '*' config` fails until you replace them. The standard profiles validate cleanly.
- **`central-settlement` health check targets port 3001**, but the service listens on **3007** (upstream `config/default.json` at `v17.0.6`). Expect `docker ps` to show it as unhealthy even when it works.
- **The perf and load stacks require a `perf.override.env` file**, which is git-ignored. Copy it from `perf.override.sample.env` first, or Compose fails with "env file … not found".
- **The monitoring compose treats `mojaloop-net` as an external network**, but the README's quick start launches monitoring first. On a fresh machine, create the network (`docker network create mojaloop-net`) or start a switch stack first.
- **Switch service ports are not published to the host** (they are commented out). Check health with `docker exec` or through the TTK ([10 §6](10-local-setup-and-execution.md#6-verification)).
- **`docker-clean-all.sh` is destructive.** It removes **every** container and volume on the machine, not just this project's.

Found when the labs were run end to end on 2026-11-04 (Windows 11, Docker Desktop, Compose v5.6.1); details in [10](10-local-setup-and-execution.md) and [SA 03](sa-learning-project/03-milestones-03-04-setup-and-run.md):

- **The README's test re-run command fails on current Compose** (`--project-name ttk-test-only --profile ttk-tests up --no-deps` → "depends on undefined service"). Use `docker compose --profile all-services --profile ttk-provisioning --profile ttk-tests run --rm --no-deps ttk-tests`.
- **`docker compose down -v` without the profiles removes nothing**, because every service belongs to a profile. Repeat the same `--profile` flags you started with. The monitoring stack is a separate project and needs its own `down`.
- **The first test run after a cold start can show 27/31** (transfer-callback timeouts while handlers warm up). A re-run on the warm stack passes 31/31.
- **`central-settlement` serves its health endpoint at `/v2/health`**, not `/health`.
- **FSPIOP `Date` headers are validated, weekday included.** A hand-typed date with the wrong weekday gets 400 / 3101.
- **Git Bash rewrites container paths** such as `/opt/...` in `docker exec` arguments. Prefix the command with `MSYS_NO_PATHCONV=1`, or use PowerShell.
- **The monitoring stack fails on Docker Desktop for Windows.** `node-exporter`'s `rslave` root mount is rejected, so `up -d` exits before Prometheus and Grafana start. Its cAdvisor also clashes with the `debug` profile on port 8080. A working subset is in [10 §9.7](10-local-setup-and-execution.md#97-monitoring-stack).
- **The finance portal is only half wired.** Transfers and Financial Positions work. Roles and Settlements fail to load, because their UI containers (ports 9301, 9303) aren't defined in compose ([10 §9.5](10-local-setup-and-execution.md#95-finance-portal-back-office-ui)).
- **Settlement does not change the FSPs' `SETTLEMENT` accounts.** It resets positions through the Hub's multilateral settlement account; moving real money (funds in or out) is a separate step.

---

## 7. Recommended first hands-on exercise

Run the standard stack, watch the P2P test pass, then replay it in the TTK UI and the mobile simulator while you follow the sequence diagram in [07](07-payment-transaction-lifecycle.md). Step-by-step instructions are in [10-local-setup-and-execution.md](10-local-setup-and-execution.md) and [17-learning-roadmap.md](17-learning-roadmap.md) (Level 3, Exercise 3.1).
