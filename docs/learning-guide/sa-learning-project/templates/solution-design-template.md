# <Bank name> – Mojaloop Integration Solution Design (template)

> Use for deliverables D12, D14 and capstone C2. Replace every `<…>`. Tag statements as *[VERIFIED]* (tested in the harness or confirmed by the scheme) or *[ASSUMPTION]* (to be confirmed).

| Document control | |
| --- | --- |
| Version / date | |
| Author (solution architect) | |
| Reviewers (bank IT, security, operations, scheme) | |
| Status | Draft / Review / Approved |

## 1. Context and scope

- Business objective (e.g. "offer instant transfers to any wallet in <country>")
- In scope: use cases (P2P send/receive, merchant, cash-out …), channels, currencies
- Out of scope
- Scheme facts: operator, API version(s) (`fspiop` 1.1 / 2.0, ISO 20022?), settlement model, NDC policy, certification process

## 2. Architecture overview

- Context diagram: channels → bank systems → integration service → connector → scheme
- Component list with owner (bank / vendor / scheme)
- Deployment zones (internal, DMZ) and connectivity to the scheme

## 3. Functional design

| Flow | Bank as payer | Bank as payee |
| --- | --- | --- |
| Alias registration | | |
| Account lookup | | |
| Quote and fees | | |
| Transfer and posting | | |
| Notifications to customers | | |
| Status inquiry and recovery | | |

Include sequence diagrams for each flow, including failure branches.

## 4. Data and mapping

- Identifier types published (MSISDN, ACCOUNT_ID, IBAN…) and the oracle used
- Field mapping FSPIOP/SDK ↔ CBS (amount precision ≤ 4 dp, currency, names, narrative)
- ID management (`homeTransactionId`, `transferId`, `quoteId`, CBS reference)
- Error mapping table (CBS → FSPIOP, FSPIOP → customer message)

## 5. Accounting design

- GL accounts (customer, outgoing clearing, incoming clearing, settlement nostro, fee income)
- Posting rules per event (hold, commit, abort, settlement, funds-in)
- Provisional vs final crediting decision for incoming payments

## 6. Non-functional requirements

| NFR | Target | How achieved |
| --- | --- | --- |
| Availability | | |
| Throughput (TPS peak) | | |
| Latency (end-to-end p95) | | |
| Timeouts (quote, transfer, CBS) | | |
| Idempotency and recovery | | |
| Observability (logs, metrics, traces) | | |

## 7. Security design

- mTLS certificates (issuer, rotation), JWS keys (generation, storage, rotation), ILP secret
- Network: firewall rules, IP allow-lists, DMZ placement of the connector
- Secrets management, HSM usage, access control for operations
- Data protection (party names, DOB) and logging rules

## 8. Liquidity, settlement and reconciliation

- Prefunding amount and NDC, top-up triggers and approvals
- Settlement windows and bank-side process; reconciliation jobs and break handling

## 9. Operations

- Monitoring and alerting, support model, runbooks (stuck transfer, NDC breach, scheme outage, key rotation)

## 10. Testing and certification

- Unit / integration / scheme certification (TTK) / UAT / performance / security tests and entry-exit criteria

## 11. Risks, assumptions, decisions

- Risk register (see the implementation plan template)
- Architecture Decision Records (see [adr-and-rca-templates.md](adr-and-rca-templates.md))
