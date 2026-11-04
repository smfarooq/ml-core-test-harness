# Data Architecture

> Stores, ownership, main entities, data movement and consistency rules. Diagram: [09](../diagrams/09-data-architecture.svg).
> Table names come from the upstream migrations at the pinned tags and were checked against the live databases on 2026-11-04 (`central_ledger` had **108 tables**). Status labels are defined in the [README](../README.md#status-labels).

---

## 1. Data stores and ownership

| Store (container) | Database / keyspace | Owner (writes schema) | Other writers / readers | Status |
| --- | --- | --- | --- | --- |
| MySQL `mysql` | `central_ledger` | **central-ledger** (migrations) | central-settlement (writes settlement data), quoting-service (writes quotes), reporting API (reads, Finance Portal) | Configured |
| MySQL `mysql-als` | `account_lookup` | **account-lookup-service** | – | Configured |
| MySQL `mysql-als` | `oracle_msisdn` | **als-msisdn-oracle-svc** (init SQL + migrations) | – | Configured |
| MongoDB `objstore` | `mlos` | central-ledger object store (bulk payloads) | – | Configured |
| MongoDB `objstore` | `ttk` | Testing Toolkit (history, reports) | – | Configured |
| MongoDB `objstore` | `reporting` | reporting-events-processor (audit events) | reporting API | Optional |
| Redis `redis` | proxy cache | central-ledger, ALS, ML API adapter | – | Configured |
| Redis `*-redis` | per connector | SDK Scheme Adapter (in-flight transfer state) | – | Configured |
| Kafka `kafka` | 23 topics | All producers | All consumer groups | Configured |

**Shared database:** central-ledger, central-settlement and the quoting service all use the `central_ledger` schema. That simplifies consistency, but couples their releases and makes this one MySQL instance the most critical component of the switch.

## 2. Main entities in `central_ledger`

| Domain | Entities (tables) | Purpose |
| --- | --- | --- |
| Participants | `participant`, `participantCurrency`, `participantEndpoint`, `participantLimit`, `participantPosition`, `participantPositionChange` | Who the FSPs are, their accounts per currency (`POSITION`, `SETTLEMENT`, Hub accounts), callback URLs, Net Debit Caps, current positions and every change |
| Transfers | `transfer`, `transferParticipant`, `transferStateChange`, `transferFulfilment`, `transferError`, `ilpPacket`, `transferExtension` | The transfer, its parties and amounts, **append-only state history**, the fulfilment, errors and the ILP packet |
| Quotes | `quote`, `quoteParty`, `quoteResponse` (and related) | Agreed terms (when `SIMPLE_ROUTING_MODE` is false, as here) |
| Settlement | `settlementModel`, `settlementWindow`, `settlementWindowStateChange`, `settlement`, `settlementStateChange`, `settlementParticipantCurrency` | Windows, settlements and per-participant net amounts |
| Reference data | `transferState`, `ledgerAccountType`, `ledgerEntryType`, `participantLimitType`, `endpointType`, … | Enumerations seeded by migrations |
| FX | FX quote and FX transfer tables (upstream migrations) | Conversion terms and FX transfer states |

Directory data: `account_lookup.oracleEndpoint` (which oracle serves which party ID type), `oracle_msisdn.oracleMSISDN(id, subId, fspId)`.

## 3. Data movement

| Flow | From → To | Mechanism | Data |
| --- | --- | --- | --- |
| Request intake | FSP → API services | HTTP (FSPIOP JSON) | Party, quote, transfer requests |
| Edge → processing | API services → Kafka → handlers | Kafka messages (envelope `id`, `from`, `to`, `type`, `content`, `metadata.event`) | Full request content |
| Ledger writes | Handlers → MySQL | SQL transactions | Transfers, states, positions |
| Settlement | central-settlement ↔ MySQL | SQL | Window and settlement records, MLNS transfers |
| Directory | ALS → oracle → MySQL | HTTP, SQL | ID → FSP mapping |
| Audit | All services → `topic-event-audit` → reporting-events-processor → MongoDB `reporting` | Kafka, Event SDK | Audit events |
| Reporting | MySQL + MongoDB → reporting API (GraphQL) → Finance Portal | SQL, GraphQL | Transfers, positions |
| Connector state | SDK ↔ its Redis | Redis | In-flight transfer state |
| Party data | Payee FSP → ALS → payer FSP | HTTP relay | Names (not stored by ALS) |

## 4. Consistency rules

| Rule | How it is achieved | Evidence |
| --- | --- | --- |
| A transfer has exactly one authoritative state | Append-only `transferStateChange`; the latest row is the state | Lab DB queries |
| Positions balance | Each commit changes payer and payee positions by equal and opposite amounts (zero-sum) | Lab (T2) |
| No double processing | Duplicate check by `transferId` + request hash; position messages keyed per account preserve order | Lab (T5) |
| Atomic position updates | The position batch handler applies a batch of changes in one DB transaction | Upstream source |
| Both FSPs agree on the outcome | ILP condition/fulfilment + final notifications; `GET /transfers/{id}` resolves doubt | Lab |
| Kafka delivery | Producers use `request.required.acks: all` for events; topics have replication factor 1 here, so a broker loss loses unconsumed messages | [event_sdk/config.json](../../../docker/event_sdk/config.json), [provision.sh](../../../docker/kafka/scripts/provision.sh) |

## 5. Retention and protection

| Aspect | As-is | Target (Proposed) |
| --- | --- | --- |
| Ledger retention | Not configured (grows forever) | Scheme rule (often several years); archive closed settlement windows |
| Kafka retention | Broker default (not configured) | Short (hours–days); Kafka is transport, not the record |
| Audit retention | MongoDB `reporting`, not configured | Immutable storage, retention per regulation |
| Personal data | Party names relayed; quote parties stored | Minimise; encrypt at rest; access logging |
| Backups | **None**; no persistent volumes | Point-in-time recovery for MySQL; snapshot MongoDB; tested restores |
| Encryption at rest | None | Storage and DB-level encryption, keys in KMS |
