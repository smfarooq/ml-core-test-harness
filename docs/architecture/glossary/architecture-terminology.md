# Architecture Terminology

> Terms and abbreviations used in the architecture diagrams and analysis. For a longer Mojaloop glossary see [learning-guide/16-glossary.md](../../learning-guide/16-glossary.md).

## Mojaloop and payments

| Term | Meaning |
| --- | --- |
| **Mojaloop** | Open-source software for interoperable instant payment schemes, maintained by the Mojaloop Foundation |
| **Hub / Switch** | The central Mojaloop system that routes messages between FSPs and keeps the clearing ledger |
| **Scheme** | The rules, participants and operator of a payment network that runs on the hub |
| **FSP / DFSP** | (Digital) Financial Service Provider: a bank, mobile money operator or PSP that joins the scheme |
| **Payer FSP / Payee FSP** | The FSP sending the money / the FSP receiving it |
| **FXP** | Foreign Exchange Provider: a participant that converts currencies in FX transfers |
| **MMO** | Mobile Money Operator |
| **PSP** | Payment Service Provider |
| **FSPIOP** | FSP Interoperability API: the asynchronous REST API between FSPs and the hub (`/parties`, `/quotes`, `/transfers`, …) |
| **ISO 20022 mode** | FSPIOP 2.0 carried as ISO 20022 messages (`API_TYPE=iso20022`, used in the perf configuration) |
| **Discovery** | Phase 1 of a payment: finding which FSP serves an identifier and who the party is |
| **Agreement (quote)** | Phase 2: the payee FSP's terms, fees and the ILP condition |
| **Transfer (clearing)** | Phase 3: reserve funds, payee accepts, commit |
| **Settlement** | Moving real money between FSPs' settlement accounts for the net amounts of a settlement window |
| **Settlement window** | A period whose committed transfers are settled together (`OPEN → CLOSED → … → SETTLED`) |
| **MLNS** | Multilateral net settlement: each FSP pays or receives its net position against all others |
| **Position** | An FSP's running net obligation in the ledger (increases when it pays, decreases when it receives) |
| **NDC** | Net Debit Cap: the maximum position an FSP may reach; breaches are rejected (4200) |
| **Liquidity check** | Rejects a transfer the FSP's prefunded settlement balance cannot cover (4001) |
| **Funds in / funds out** | Recording real money deposited into or withdrawn from an FSP's settlement account |
| **ILP** | Interledger Protocol; Mojaloop uses its packet format and condition/fulfilment |
| **Condition / fulfilment** | Hash-lock: the payee reveals the fulfilment; the hub commits only if SHA-256(fulfilment) = condition |
| **Oracle** | A directory service mapping an identifier type (e.g. MSISDN) to the FSP that serves it |
| **MSISDN** | A mobile phone number used as a payment identifier |
| **ALS** | Account Lookup Service |
| **Callback** | The asynchronous result of an FSPIOP request, sent to the requester's registered endpoint (`PUT …`) |
| **PATCH notification** | A final-state notification the hub sends to a payee FSP that answered `RESERVED` |
| **Request-to-pay / TRS** | Payee-initiated payment via the Transaction Requests Service (not deployed in this repository) |
| **Bulk transfer** | Many transfers in one request via the bulk API adapter (not deployed in this repository) |
| **SDK Scheme Adapter / Mojaloop Connector** | FSP-side component that hides FSPIOP's asynchronous complexity behind a simple API |
| **Backend API** | The API an FSP's core system implements so the SDK can ask it about parties, quotes and transfers |
| **Core banking system** | The FSP's system of record for customer accounts |
| **TTK** | Mojaloop Testing Toolkit: simulates FSPs and the hub operator and runs test collections |
| **Golden path** | Mojaloop's regression suite of TTK test cases |
| **Finance Portal / BOP** | Business Operations Portal for hub operators (transfers, positions, roles, settlements) |
| **PM4ML** | Payment Manager for Mojaloop: a packaged FSP-side connector solution |
| **Connection Manager (MCM)** | Mojaloop tool for exchanging certificates and keys during onboarding |

## Architecture, infrastructure and operations

| Term | Meaning |
| --- | --- |
| **As-Is** | What this repository actually provides or configures |
| **Target** | A proposed production design; not present in the repository |
| **Docker Compose profile** | A named group of services started together (`--profile all-services`) |
| **Handler** | A Kafka consumer process inside a Mojaloop service (prepare, fulfil, position batch, notification, …) |
| **Consumer group** | Kafka consumers sharing a topic's partitions; one consumer per partition at a time |
| **Partition / key** | Kafka splits a topic into partitions; messages with the same key keep their order |
| **RF** | Replication factor: how many brokers hold a copy of each partition |
| **min.insync.replicas** | Minimum replicas that must acknowledge a write when `acks=all` |
| **DLQ** | Dead-letter queue: where unprocessable messages are parked (none configured here) |
| **KRaft** | Kafka's built-in consensus mode, replacing ZooKeeper |
| **Namespace** | A Kubernetes grouping of workloads (`mojaloop`, `stateful-resources`, …) |
| **Deployment / StatefulSet / DaemonSet / Job** | Kubernetes workload types: stateless replicas / stable-identity stateful pods / one pod per node / run-to-completion task |
| **HPA / VPA** | Horizontal / Vertical Pod Autoscaler |
| **PDB** | PodDisruptionBudget: keeps a minimum number of pods during voluntary disruptions |
| **PV / StorageClass** | PersistentVolume and the class of storage it is provisioned from |
| **Helm** | Kubernetes package manager; Mojaloop publishes Helm charts |
| **GitOps / ArgoCD** | Deploying by syncing a cluster to configuration stored in Git; ArgoCD is the tool evidenced here |
| **Service mesh / Istio** | Infrastructure layer for service-to-service traffic, mTLS and policies |
| **Ingress / egress gateway** | Controlled entry and exit points for cluster traffic |
| **AZ** | Availability zone: an isolated data-centre location within a region |
| **HA** | High availability: continuing service through component failures |
| **DR** | Disaster recovery: restoring service after losing a site or cluster |
| **BCP** | Business continuity planning |
| **RPO** | Recovery Point Objective: how much data loss is acceptable (a design decision here) |
| **RTO** | Recovery Time Objective: how much downtime is acceptable (a design decision here) |
| **PITR** | Point-in-time recovery of a database |
| **Warm standby** | A DR site that is deployed but scaled down, ready to take over |
| **SPOF** | Single point of failure |
| **SLO** | Service-level objective |
| **Exporter** | A process that exposes metrics of a system (Kafka, MySQL, host) in Prometheus format |
| **Health / readiness / liveness probe** | Checks that a service is up / ready for traffic / still alive |

## Security and identity

| Term | Meaning |
| --- | --- |
| **TLS / mTLS** | Transport Layer Security / mutual TLS, where both sides present certificates |
| **JWS** | JSON Web Signature; in FSPIOP the `FSPIOP-Signature` header signs selected headers and the body |
| **FSPIOP-Source / -Destination** | Headers naming the sending and receiving participant |
| **OAuth 2.0 client credentials** | Machine-to-machine token flow: a client authenticates with its own ID and secret |
| **OIDC** | OpenID Connect: identity layer on OAuth 2.0 for user sign-in |
| **IdP** | Identity provider (e.g. Keycloak) |
| **SSO / MFA** | Single sign-on / multi-factor authentication |
| **Identity-aware proxy** | A proxy that authenticates users and passes their identity (here the `x-user` header) to the backend |
| **Ory Keto / Oathkeeper** | Open-source permission service / identity-aware proxy; the reporting API supports Keto |
| **RBAC** | Role-based access control |
| **Least privilege** | Granting only the permissions a role or service needs |
| **Four-eyes (maker-checker)** | A change needs a second person to approve it |
| **Break-glass access** | Time-limited, logged emergency access |
| **HSM** | Hardware security module: tamper-resistant key storage and signing |
| **KMS / Vault** | Key management service / secrets manager |
| **PKI / CA / CSR** | Public key infrastructure / certificate authority / certificate signing request |
| **WAF** | Web application firewall |
| **SIEM** | Security information and event management |
| **SBOM** | Software bill of materials |
| **Grype** | Container image vulnerability scanner used in this repository's CI |

## Status labels used in this documentation

| Label | Meaning |
| --- | --- |
| **Implemented** | Verified in source code (this repository, or the upstream service at the pinned image tag) |
| **Configured** | Present in deployment or configuration files in this repository |
| **External** | Depends on a system outside this repository |
| **Optional** | Supported but not enabled by default |
| **Proposed** | Recommended design; not present in the repository |
| **Unverified** | Insufficient evidence |
| **Not present** (diagram badge) | Looked for and not found |
