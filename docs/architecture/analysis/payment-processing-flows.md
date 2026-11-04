# Payment Processing Flows

> Traced through the configuration in this repository, the upstream service source at the pinned tags, and **live runs on 2026-11-04** (Kafka messages, database rows and callbacks were captured). Diagram: [07](../diagrams/07-end-to-end-payment-flow.svg).
> Status labels are defined in the [README](../README.md#status-labels).

---

## 1. Transaction types and where they stand here

| Transaction type | API resources | Status in this repository |
| --- | --- | --- |
| **P2P transfer** (payer-initiated, same currency) | `/parties`, `/quotes`, `/transfers` | Implemented (upstream), verified at runtime: P2P 31/31 |
| Golden-path variants (duplicates, timeouts, aborts, PATCH notifications, funds in/out, settlement) | same + admin + settlement APIs | Verified: 124 test cases, 0 failed |
| **FX transfer** (cross-currency via an FX provider) | `/fxQuotes`, `/fxTransfers` | Verified: FX tests 0 failed (`fx` profile) |
| Funds in / funds out (FSP prefunding) | central-ledger admin `/participants/{name}/accounts/{id}` | Verified (labs) |
| Settlement | central-settlement `/v2/settlementWindows`, `/v2/settlements` | Verified (T8 lab) |
| Request-to-pay (merchant / payee-initiated) | `/transactionRequests`, `/authorizations` | **Not deployed** (no Transaction Requests Service; tests skipped) |
| Bulk transfers | `/bulkQuotes`, `/bulkTransfers` | **Not deployed** at the edge (no bulk API adapter) |

---

## 2. P2P transfer: the three phases

Every FSP ↔ switch interaction is **asynchronous**: the receiver answers `202 Accepted` at once and the result arrives later as a callback (`PUT …`) to the caller's registered endpoint. The client-generated IDs (`transferId`, `quoteId`) correlate callbacks and make retries safe.

### Phase 1 – Discovery (who is the payee?)

| # | From → To | Message | What happens |
| --- | --- | --- | --- |
| 1 | Payer FSP → ALS | `GET /parties/MSISDN/{id}` | ALS answers 202 |
| 2 | ALS → Oracle | `GET /participants/MSISDN/{id}` (sync) | Oracle returns the `fspId` from `oracleMSISDN` |
| 3 | ALS → central-ledger | Participant endpoints (sync, cached) | Resolves the payee FSP's `/parties` callback URL |
| 4 | ALS → Payee FSP | `GET /parties/MSISDN/{id}` | Payee FSP looks up its customer |
| 5 | Payee FSP → ALS | `PUT /parties/MSISDN/{id}` | Party name, FSP id |
| 6 | ALS → Payer FSP | `PUT /parties/MSISDN/{id}` | Payer shows the name to the customer for confirmation |

Error: number unknown to the oracle → `PUT /parties/…/error` **3204** from the Hub (verified).

### Phase 2 – Agreement (quote)

| # | From → To | Message | What happens |
| --- | --- | --- | --- |
| 7 | Payer FSP → quoting-service | `POST /quotes` (`quoteId`, `transactionId`, amount, payer, payee) | 202; published to `topic-quotes-post` |
| 8 | quoting-service-handler | consume | Validates participants and currency, **persists** the quote, checks duplicates |
| 9 | handler → Payee FSP | `POST /quotes` | Payee FSP computes fees and creates the ILP packet and **condition** |
| 10 | Payee FSP → quoting-service | `PUT /quotes/{quoteId}` | `transferAmount`, fees, `expiration`, `ilpPacket`, `condition` → `topic-quotes-put` |
| 11 | handler → Payer FSP | `PUT /quotes/{quoteId}` | Payer shows fees; customer accepts |

### Phase 3 – Transfer (clearing)

| # | From → To | Message / topic | Ledger effect | State (`transferStateChange`) |
| --- | --- | --- | --- | --- |
| 12 | Payer FSP → ML API adapter | `POST /transfers` (`transferId`, amount, `ilpPacket`, `condition`, `expiration`) | – | – |
| 13 | ML API adapter → Kafka | `topic-transfer-prepare` (202 to the payer) | – | – |
| 14 | Prepare handler | validate, duplicate check, save | `transfer`, `transferParticipant`, `ilpPacket` rows | `RECEIVED_PREPARE` |
| 15 | → Kafka | `topic-transfer-position-batch`, **key = payer's position account** | – | – |
| 16 | Position batch handler | liquidity + Net Debit Cap check; payer position += amount | `participantPosition`, `participantPositionChange` | `RESERVED` |
| 17 | → Kafka → notification handler | `topic-notification-event` | – | – |
| 18 | Notification handler → Payee FSP | `POST /transfers` | – | – |
| 19 | Payee FSP → ML API adapter | `PUT /transfers/{transferId}` (`fulfilment`, `transferState: COMMITTED`) | Payee FSP credits its customer | – |
| 20 | ML API adapter → Kafka | `topic-transfer-fulfil` | – | – |
| 21 | Fulfil handler | SHA-256(fulfilment) == condition? expiry? | `transferFulfilment` | `RECEIVED_FULFIL` |
| 22 | → Kafka | `topic-transfer-position-batch`, **key = payee's position account** | – | – |
| 23 | Position batch handler | payee position −= amount | position change rows | `COMMITTED` |
| 24 | → notification handler → Payer FSP | `PUT /transfers/{transferId}` (`COMMITTED`) | Payer FSP finalises the debit | – |
| 25 | → Payee FSP | `PATCH /transfers/{transferId}` if the payee answered `RESERVED` (a `PUT` confirmation is configurable: `SEND_TRANSFER_CONFIRMATION_TO_PAYEE`) | – | – |

Measured on a warm stack: prepare → final notification **≈ 830 ms**, of which ≈ 410 ms was the simulated payee's turn-around (Kafka broker timestamps).

Every service also emits **audit events** to `topic-event-audit` (23 event types were seen for one transfer in the M7 lab). Positions are double-entry: the payer's and payee's position changes sum to zero (verified).

---

## 3. Correlation identifiers

| Identifier | Created by | Used for |
| --- | --- | --- |
| `transferId` | Payer FSP (UUID) | Transfer correlation, Kafka message content, ledger primary key, duplicate check |
| `quoteId`, `transactionId` | Payer FSP | Quote correlation; `transactionId` links quote and transfer (inside the ILP packet) |
| ILP `condition` / `fulfilment` | Payee FSP (from its `ILP_SECRET`) | Cryptographic proof that the payee accepted |
| `FSPIOP-Source`, `FSPIOP-Destination` | Sender | Routing and participant identification |
| `traceparent` (W3C) | First service / FSP | Distributed tracing context in headers and event metadata |
| `homeTransactionId` | FSP's own system (SDK API) | Links the FSP's internal record to the Mojaloop transfer |

---

## 4. Alternative and error scenarios (all verified at runtime)

| Scenario | Trigger | Switch behaviour | Final internal state | Error code to payer |
| --- | --- | --- | --- | --- |
| Payee not found | Unregistered MSISDN | Oracle has no record | (no transfer) | **3204** |
| NDC / liquidity breach | Amount beyond the payer's Net Debit Cap or liquidity | Rejected by the position handler; position unchanged | `ABORTED_REJECTED` | **4200** / **4001** |
| Invalid fulfilment | Payee returns a fulfilment that doesn't hash to the condition | Fulfil handler rejects; the payer's reservation is reversed | `RECEIVED_ERROR` → `ABORTED_ERROR` | **3100** |
| Payee rejects | Payee sends `transferState: ABORTED` | Reservation reversed | `RECEIVED_REJECT` → `ABORTED_REJECTED` | payee's code |
| Payee never answers | No fulfil before `expiration` | Timeout handler (every 15 s) expires it; both FSPs notified | `RESERVED` → `RESERVED_TIMEOUT` → `EXPIRED_RESERVED` | **3303** |
| Duplicate, same body | Same `transferId` and body resent | No second debit; callback re-sent with the current state | unchanged | – |
| Duplicate, changed body | Same ID, different body | Rejected | unchanged | **3106** |
| Unknown transfer queried | `GET /transfers/{unknown}` | – | – | **3208** |
| Bad headers or body | Missing/invalid header, wrong `Date` weekday, bad version | Rejected at the edge | – | **3100–3102**, **3001** |
| Late core-banking credit (FSP-side defect) | Payee FSP credits after expiry | Switch has already expired the transfer: **reconciliation break** at the FSP | `EXPIRED_RESERVED` | 3303 |

---

## 5. FX transfer (summary)

The payer FSP asks an **FX provider (FXP)** for a conversion quote (`POST /fxQuotes`), then reserves the conversion (`POST /fxTransfers`) before the main transfer. The switch keeps positions in both currencies and commits the conversion together with the dependent transfer (upstream design; the transfer state enumeration includes `RECEIVED_FULFIL_DEPENDENT` for this). Verified: FX provisioning 223/223 and FX tests passed (0 failed).

## 6. Settlement (deferred net)

| Step | API | State |
| --- | --- | --- |
| Transfers commit into the open window | – | Window `OPEN` |
| Operator closes the window | `POST /v2/settlementWindows/{id}` | `PROCESSING` → `CLOSED`; a new window opens |
| Operator creates a settlement | `POST /v2/settlements` | Settlement `PENDING_SETTLEMENT` |
| Settlement bank moves the money (external) and the operator records it per participant | `PUT /v2/settlements/{id}` | `PS_TRANSFERS_RECORDED` → `PS_TRANSFERS_RESERVED` → `PS_TRANSFERS_COMMITTED` → `SETTLED` |

Ledger effect (verified): positions are reset through MLNS transfers between each FSP's `POSITION` account and the Hub's `HUB_MULTILATERAL_SETTLEMENT` account. The FSPs' `SETTLEMENT` accounts are **not** changed; real money moves outside the switch and is recorded as funds in/out.

## 7. Where the participant is authenticated

| Step | As-is | Target (Proposed) |
| --- | --- | --- |
| Transport | None (plain HTTP) | mTLS at the gateway with the FSP's client certificate |
| API access | None | OAuth2 client credentials token |
| Message origin | Switch signs its own messages; FSP signatures forwarded, not validated | Validate FSP JWS against the key registered for `FSPIOP-Source` |
| Business identity | `FSPIOP-Source` must be a registered, active participant | Same, bound to the certificate identity |
