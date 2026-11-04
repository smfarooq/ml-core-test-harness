# 16 – Glossary

> Financial and technical terms used in this guide and in the repository. "(repo)" marks terms that appear literally in this repository's files.

## A

| Term | Meaning |
| --- | --- |
| **Abort / ABORTED** | Final failed state of a transfer. No money obligation is created and any reservation is reversed. (repo: `TransferState`) |
| **Account lookup** | Finding which FSP holds an identifier. Done by the Account Lookup Service with oracles. |
| **ACCOUNT_ID** | Party identifier type for a bank or FSP account number. (repo: `PartyIdType`) |
| **Acquirer** | The FSP that serves a merchant and receives card or merchant payments for it |
| **Admin API** | Central-ledger HTTP API for hub operators: participants, accounts, limits, endpoints, settlement models (repo: `central_admin_1.0`) |
| **Agent** | A person or shop providing cash-in and cash-out for an FSP (`initiatorType: AGENT`) |
| **ALIAS** | Generic alias identifier type, e.g. an LEI in the repo's merchant tests |
| **ALS** | Account Lookup Service (`account-lookup-service`) |
| **AML / KYC** | Anti-Money-Laundering / Know-Your-Customer checks. An FSP responsibility, not the switch's. |
| **amountType** | `SEND` (payer sends this amount) or `RECEIVE` (payee receives this amount) |
| **Asynchronous API** | The request gets `202 Accepted`; the result comes later as a callback |
| **Audit event** | Event SDK message describing an action. Sent to `topic-event-audit`. (repo) |

## B–C

| Term | Meaning |
| --- | --- |
| **Backend API** | SDK Scheme Adapter API that the FSP implements (`/parties`, `/quoterequests`, `/transfers`) |
| **Bulk transfer** | Many transfers submitted as one request (`/bulkQuotes`, `/bulkTransfers`) |
| **Callback** | The asynchronous response delivered to an FSP's registered URL (`PUT /resource/{ID}`) |
| **Callback URL / endpoint** | URL registered for each participant and resource type, e.g. `FSPIOP_CALLBACK_URL_TRANSFER_PUT` (repo) |
| **Central Ledger** | Core service that holds participants, positions and transfer state (`central-ledger`) |
| **Central Settlement** | Service managing settlement windows and settlements (`central-settlement`) |
| **Clearing** | Determining and recording obligations between institutions from payments |
| **Commit / COMMITTED** | Final successful transfer state. Irrevocable. |
| **Condition** | `base64url(SHA-256(fulfilment))`. Locks a transfer until the payee reveals the fulfilment. |
| **Config-modifier** | The harness script that deep-merges overrides into each image's `default.json` (repo: `docker/config-modifier`) |
| **Connector / Mojaloop Connector** | Adapter between an FSP and the switch. Usually the SDK Scheme Adapter. |
| **Consumer group** | Kafka concept: consumers sharing a group split the partitions of a topic |
| **Core banking system (CBS)** | The bank's system of record for accounts and balances |
| **Core Connector** | FSP-specific adapter between the CBS and the SDK |

## D–F

| Term | Meaning |
| --- | --- |
| **DFSP / FSP** | (Digital) Financial Service Provider: a bank, MMO or MFI participating in the scheme |
| **Discovery** | Phase 1: find the payee and its FSP (`/parties`) |
| **DNS (Deferred Net Settlement)** | Settlement model: net obligations are settled periodically, not per payment (repo: `DEFERREDNET…`) |
| **Disbursement** | One-to-many payments, e.g. salaries or social grants |
| **Duplicate check** | Switch compares the ID and a hash of the body to detect replays (3106 if modified) |
| **Endpoint (participant)** | See callback URL |
| **Event SDK** | Mojaloop library emitting audit, log and trace events (repo: `docker/event_sdk/config.json`) |
| **expiration** | Deadline on quotes and transfers. After it, the switch aborts the transfer (3303). |
| **extensionList** | Generic key/value extension mechanism on FSPIOP objects |
| **FSPIOP** | Financial Service Provider Interoperability Protocol: Mojaloop's API |
| **FSPIOP-Source / FSPIOP-Destination** | Headers naming the sending and receiving FSP |
| **FSPIOP-Signature** | JWS signature header |
| **Fulfilment** | Secret preimage revealed by the payee FSP to accept a transfer |
| **Funds-in / funds-out** | Recording deposits to and withdrawals from an FSP's settlement account at the hub (`recordFundsIn`, `recordFundsOutPrepareReserve`…) (repo) |
| **FX / FXP** | Foreign exchange / Foreign Exchange Provider, which converts currencies (`/fxQuotes`, `/fxTransfers`) |

## G–L

| Term | Meaning |
| --- | --- |
| **Golden path** | Mojaloop's standard regression test suite (repo: `collections/tests/golden_path`, label `std-gp`) |
| **Handler** | A Kafka consumer process inside a service (prepare, position, fulfil, notification…) |
| **Hub / Switch** | The central Mojaloop system run by the scheme operator. Participant name `Hub` in the harness. |
| **HUB_MULTILATERAL_SETTLEMENT / HUB_RECONCILIATION** | Hub ledger accounts per currency used for settlement and funds movements (repo) |
| **homeTransactionId** | The FSP's own reference in SDK APIs |
| **Idempotency** | Repeating a request has no extra effect. Mojaloop relies on client-generated IDs. |
| **ILP (Interledger Protocol)** | Protocol whose packet, condition and fulfilment concepts Mojaloop uses for transfer agreement |
| **ilpPacket** | Base64url-encoded packet with amount, address and transaction data |
| **Initiator / initiatorType** | Who started the transaction (`PAYER`/`PAYEE`) and what kind of party (`CONSUMER`, `AGENT`, `BUSINESS`, `DEVICE`) |
| **Interchange fee** | Fee between FSPs. Settlement can calculate it via rules (`INTERCHANGE_FEE` accounts). |
| **Interoperability** | Ability of customers of different providers to transact seamlessly |
| **JWS** | JSON Web Signature, used for `FSPIOP-Signature` |
| **Kafka** | Distributed log / message broker used inside the switch |
| **KRaft** | Kafka mode without ZooKeeper (repo: `KAFKA_ENABLE_KRAFT`) |
| **LEI** | Legal Entity Identifier, used as a merchant alias in the repo's LEI tests |
| **Liquidity** | Funds available to an FSP to cover its payment obligations |

## M–P

| Term | Meaning |
| --- | --- |
| **Merchant payment (P2M)** | Consumer pays a business (`scenario: PAYMENT`) |
| **MMO** | Mobile Money Operator |
| **MSISDN** | Mobile phone number in international format, the most common party identifier |
| **mTLS** | Mutual TLS: both sides authenticate with certificates |
| **Net Debit Cap (NDC)** | The maximum net amount an FSP may owe the scheme. Transfers beyond it are rejected (4001/4200). (repo: `NET_DEBIT_CAP`) |
| **Net position** | Sum of what an FSP owes (outgoing) minus what it is owed (incoming) |
| **Notification (PATCH)** | `PATCH /transfers/{ID}` from the switch to the payee with the final state |
| **On-us transfer** | Payer and payee at the same FSP (`ENABLE_ON_US_TRANSFERS`) |
| **Oracle** | Directory service mapping identifiers to FSPs (repo: `als-msisdn-oracle-svc`) |
| **P2P** | Person-to-person transfer (`scenario: TRANSFER`) |
| **Participant** | An FSP (or the Hub) registered in central-ledger |
| **Party** | The payer or payee person or business |
| **Payee / Payer** | Receiver / sender of funds |
| **Payee FSP / Payer FSP** | FSP of the receiver / sender |
| **PISP** | Payment Initiation Service Provider (third-party initiator; not in this harness) |
| **PM4ML** | Payment Manager for Mojaloop, a packaged FSP connector product |
| **Position** | An FSP's running obligation at the hub, updated on reserve, commit and abort |
| **Prefunding** | FSP deposits funds in advance to support its NDC |
| **Prepare** | First phase of a transfer: validate and reserve (`POST /transfers`) |
| **Profile (Compose)** | Named subset of services, e.g. `all-services`, `ttk-tests` (repo) |
| **Proxy (inter-scheme)** | Participant that represents another scheme. Supported via proxy cache config. |

## Q–S

| Term | Meaning |
| --- | --- |
| **Quote** | Agreement on amounts, fees, expiry and condition before a transfer (`POST /quotes`) |
| **Quoting service** | Service handling `/quotes`, `/fxQuotes`, `/bulkQuotes` |
| **Reconciliation** | Matching records between FSPs, the hub and the settlement bank |
| **Refund** | New transfer reversing a previous one economically (`scenario: REFUND`) |
| **Request to pay** | Payee-initiated request approved by the payer (`/transactionRequests`) |
| **Reserve / RESERVED** | Intermediate transfer state where the payer's position is reserved |
| **RTGS** | Real-Time Gross Settlement system, typically operated by the central bank |
| **Scheme** | The set of rules, participants and infrastructure for a payment system |
| **SDK Scheme Adapter** | Mojaloop's FSP connector (`mojaloop/sdk-scheme-adapter`) |
| **Settlement** | Actual movement of funds between FSPs' settlement accounts to discharge net obligations |
| **Settlement bank** | Institution holding FSP settlement accounts |
| **Settlement model** | Configuration of granularity (NET/GROSS), interchange (MULTILATERAL/BILATERAL) and delay (DEFERRED/IMMEDIATE) (repo: `POST /settlementModels`) |
| **Settlement window** | Time period grouping committed transfers for settlement (`OPEN`, `CLOSED`, `PENDING_SETTLEMENT`, `SETTLED`, `ABORTED`) |
| **SIMPLE_ROUTING_MODE** | Quoting-service mode that only forwards (true) or also validates and persists (false, as in the harness) |

## T–Z

| Term | Meaning |
| --- | --- |
| **Timeout handler** | Central-ledger job expiring transfers (every 15 s by upstream default) |
| **Topic** | Named Kafka stream, e.g. `topic-transfer-prepare` (repo: `provision.sh`) |
| **traceparent** | W3C trace-context header for distributed tracing |
| **Transaction request** | See request to pay |
| **transactionType.scenario** | `DEPOSIT`, `WITHDRAWAL`, `TRANSFER`, `PAYMENT`, `REFUND` |
| **Transfer** | Movement of value between FSPs through the switch (`POST /transfers`) |
| **TransferState** | External: `RECEIVED`, `RESERVED`, `COMMITTED`, `ABORTED`. Internal states are finer (see [07 §4](07-payment-transaction-lifecycle.md#4-transaction-state-management)). |
| **TTK** | Mojaloop Testing Toolkit: simulator, test runner and UI (repo) |
| **wait4** | Harness script that waits for dependencies before a service starts (repo) |
| **Wallet** | E-money account, typically at an MMO |
| **XXX / XTS / XDR** | ISO 4217 codes used as test currencies in the harness (XXX "no currency", XTS "testing", XDR IMF SDR) |

## Error-code quick reference

| Code | Meaning |
| --- | --- |
| 3001 | Unacceptable version |
| 3100–3107 | Validation errors (3106 = modified request / duplicate with different body) |
| 3201 | Destination FSP error |
| 3204 | Party not found |
| 3208 | Transfer ID not found |
| 3303 | Transfer expired |
| 4001 | Payer FSP insufficient liquidity |
| 4200 | Payer limit error |
| 5100–5106 | Payee rejections |

Full table: [08 §5](08-fspiop-protocol-and-apis.md#5-error-handling).
