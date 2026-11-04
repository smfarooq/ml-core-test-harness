# M7 – Trace the Transaction Through the Microservices

> **Phase 3 · How the switch works (synthesis)** · Time box: 5–6 h · Prerequisite: [T1–T6](05-topic-01-fspiop.md) · Next: [M8 – Investigate a failed transaction](12-milestone-08-investigate-failure.md)
> Reference reading: [05 §5](../05-system-architecture.md#5-component-interaction-for-a-transfer-kafka-view), [diagrams/p2p-transfer-sequence.mmd](../diagrams/p2p-transfer-sequence.mmd)

---

## Why this matters to a solution architect

A trace turns architecture diagrams into **evidence**. With it you can show a bank exactly where time is spent, which component owns which step, and how you would find a payment that a customer complains about. It is also the raw material for latency and capacity discussions.

## Learning objectives

1. Collect evidence for one payment from every layer: HTTP (TTK), service logs, Kafka topics, the audit stream and the database.
2. Build a timeline with per-hop latencies.
3. Map each hop to the component, the topic, the table and the state change.
4. Draw conclusions for NFRs and operations.

---

## The evidence map

| Layer | Source | What it proves | Key to search by |
| --- | --- | --- | --- |
| FSP view | TTK monitor and request details (<http://localhost:9660>) | What each FSP sent and received, with headers | `quoteId`, `transferId` |
| Service logs | `docker logs <container>` | Processing steps and errors per service | `transferId` (also present in quote logs as `transactionId`) |
| Messaging | Kafka topics (`topic-quotes-*`, `topic-transfer-*`, `topic-notification-event`) | Hand-offs between components, message keys, event type and action | `transferId` / `quoteId` |
| Audit stream | `topic-event-audit` (Event SDK `AUDIT: kafka` in [event_sdk/config.json](../../../docker/event_sdk/config.json)) | Cross-service audit events for the same transaction | `transferId` |
| Ledger | MySQL `central_ledger` | Authoritative states, position movements, fulfilment, window | `transferId`, `quoteId` |
| Directory | MySQL `oracle_msisdn` / ALS | Which FSP owned the identifier | MSISDN |

---

## Lab M7 – Trace one payment

### Step 1 – Make a payment that is easy to find

In the TTK UI, run your Bank A → Wallet B test case with a distinctive amount, e.g. `37` (avoid 15 and 16, which trigger TTK failure rules). Note the start time, and copy `quoteId` and `transferId` into your journal.

### Step 2 – Collect service logs into one file (PowerShell)

```powershell
$id = "<transferId>"
$out = "..\my-lab\trace-$id.txt"
foreach ($c in "account-lookup-service","quoting-service","quoting-service-handler","ml-api-adapter","central-ledger","central-handler-position-batch") {
  "===== $c =====" | Out-File -Append $out
  docker logs --timestamps $c 2>&1 | Select-String $id | Out-File -Append $out
}
```

Party lookups are keyed by MSISDN, not `transferId`. Collect those separately:

```powershell
docker logs --timestamps account-lookup-service 2>&1 | Select-String "27713809999" | Out-File -Append $out
```

### Step 3 – Collect Kafka evidence with broker timestamps

```bash
docker exec kafka /opt/bitnami/kafka/bin/kafka-console-consumer.sh --bootstrap-server kafka:29092 --topic topic-transfer-prepare --from-beginning --property print.timestamp=true --property print.key=true --timeout-ms 10000
```

(Git Bash: `export MSYS_NO_PATHCONV=1` first, otherwise the `/opt/...` path is rewritten and the command fails silently.)

Repeat for `topic-transfer-position-batch`, `topic-transfer-fulfil`, `topic-notification-event`, `topic-quotes-post` and `topic-quotes-put`. Pipe each through `Select-String <transferId>` (Git Bash: `grep`). Note the broker timestamp (`CreateTime:`) of each message.

### Step 4 – Collect the audit trail

```bash
docker exec kafka /opt/bitnami/kafka/bin/kafka-console-consumer.sh --bootstrap-server kafka:29092 --topic topic-event-audit --from-beginning --property print.timestamp=true --timeout-ms 15000
```

Filter for your `transferId`. Identify which services emitted audit events and at which steps. Verified for one Bank A → Wallet B payment: **23** audit events. Their `metadata.trace.service` names read like a map of the switch: `QuotesPost`, `qs_quote_forwardQuoteRequest`, `transfers`, `cl_transfer_prepare`, `cl_transfer_position` (×2: prepare and commit), `ml_notification_event` (×4), `TransfersByIDPut`, `cl_transfer_fulfil`, `cs_process_transfer_settlement_window` (central-settlement assigning the window), and, if you queried status, `TransfersByIDGet` and `cl_transfer_get`.

**Tip – finding the quote from a transfer:** `quote.transactionReferenceId` holds the transaction id, which equals the `transferId`:

```bash
docker exec mysql mysql -uroot -N -D central_ledger -e "SELECT quoteId FROM quote WHERE transactionReferenceId='<transferId>';"
```

### Step 5 – Collect ledger evidence

```bash
docker exec mysql mysql -uroot -D central_ledger -e "SELECT 'quote' AS src, createdDate AS ts, quoteId AS ref, NULL AS state FROM quote WHERE quoteId='<quoteId>' UNION ALL SELECT 'quoteResponse', createdDate, quoteId, NULL FROM quoteResponse WHERE quoteId='<quoteId>' UNION ALL SELECT 'transferState', createdDate, transferId, transferStateId FROM transferStateChange WHERE transferId='<transferId>' UNION ALL SELECT 'fulfilment', createdDate, transferId, CONCAT('window ', settlementWindowId) FROM transferFulfilment WHERE transferId='<transferId>' ORDER BY ts;"
```

(Columns are as in central-ledger v19.4.4 migrations. `createdDate` precision depends on the column definition, so use the Kafka broker timestamps from step 3 for sub-second deltas.)

### Step 6 – Build the timeline

Fill a table like this in your journal:

| # | Time | Component | Evidence | Event | Δ from previous |
| --- | --- | --- | --- | --- | --- |
| 1 | | TTK (Bank A) | TTK monitor | `GET /parties` sent | – |
| 2 | | ALS → oracle → TTK (Wallet B) → ALS | ALS log | party resolved | |
| 3 | | quoting-service | `topic-quotes-post` | quote accepted | |
| 4 | | quoting-service-handler | `quote` row | quote persisted and forwarded | |
| 5 | | quoting-service-handler | `quoteResponse` row, `topic-quotes-put` | quote response forwarded | |
| 6 | | ml-api-adapter | `topic-transfer-prepare` | prepare accepted (202) | |
| 7 | | central-ledger prepare handler | `RECEIVED_PREPARE` row | validated | |
| 8 | | position-batch handler | `RESERVED` row | position reserved | |
| 9 | | notification handler | `topic-notification-event` | `POST /transfers` to Wallet B | |
| 10 | | ml-api-adapter | `topic-transfer-fulfil` | fulfil received | |
| 11 | | central-ledger fulfil handler | `RECEIVED_FULFIL` row | fulfilment valid | |
| 12 | | position-batch handler | `COMMITTED` row, fulfilment window | committed | |
| 13 | | notification handler | TTK monitor | `PUT /transfers` to Bank A | |

**Reference numbers from verification** (warm stack, TTK as both FSPs; broker timestamps): prepare → position reserved **78 ms** → forwarded to payee **135 ms** → payee fulfil received **413 ms** → position committed **56 ms** → payer notified **149 ms**. That is **about 0.83 s** end to end for the transfer phase. The service-log line counts for the transfer id were: ml-api-adapter 13, central-ledger 7, position-batch 4, quoting-service 2, quoting-service-handler 2, account-lookup-service 0 (lookups are keyed by MSISDN).

### Step 7 – Draw it

Annotate a copy of [diagrams/p2p-transfer-sequence.mmd](../diagrams/p2p-transfer-sequence.mmd) with your measured Δ values.

---

## Interpreting what you see

| Observation | What it tells an architect |
| --- | --- |
| Most time is spent in the payee's turn-around (TTK here, a real core banking system in production) | The scheme's end-to-end SLA depends on the slowest FSP. Schemes set payee response-time rules. |
| Several Kafka hops per transfer | Each hop adds milliseconds but buys decoupling, durability and independent scaling |
| Position changes are serialized per FSP account | One very busy FSP can become a hotspot; partitioning and batching matter |
| The audit stream has events from several services | It is the basis for a scheme-wide audit and reporting store (the Finance Portal profile consumes it) |
| Timestamps come from different clocks | In production, clock sync (NTP) and distributed tracing (`traceparent`) are prerequisites for reliable traces |

---

## Checkpoint

<details><summary>1. Which evidence source is authoritative for "was the payment completed?"</summary>
The central ledger (transferStateChange showing COMMITTED), not logs or Kafka.
</details>

<details><summary>2. Why does grepping by transferId also find quote-service log lines?</summary>
Because the quote's transactionId becomes the transferId, so the same value appears in the quote payload.
</details>

<details><summary>3. What would you add in production to make tracing reliable across organisations?</summary>
W3C trace context (traceparent) propagation at the gateway, switch and connectors, centralised logs and traces, and synchronized clocks.
</details>

---

## Deliverable D10 – Trace report

Two pages: the timeline table, the annotated sequence diagram, latency observations, and three operational recommendations (e.g. dashboards, alerts, required log fields).

Next: [M8 – Investigate a failed transaction](12-milestone-08-investigate-failure.md).
