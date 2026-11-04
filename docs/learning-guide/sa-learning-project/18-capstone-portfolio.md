# Capstone – Mojaloop Solution Architect Portfolio

> **Phase 6** · Time box: 10–15 h · Prerequisite: all modules · Templates: [templates/](templates/)

---

## Purpose

Prove the three capabilities defined in the [charter](00-project-charter.md#1-project-goal-and-success-criteria): **explain**, **design**, **lead**. You do so by producing three artefacts for a realistic engagement, and defending them in a mock review.

## The engagement brief (fictional)

> **Bank A** is a mid-sized commercial bank: 2 million customers; mobile app, internet banking and USSD. The core banking system offers a SOAP posting API and a REST hold API (200–800 ms), with a nightly batch from 23:00 to 01:00. The country's central bank has launched **"PayLink"**, an instant-payment scheme running Mojaloop, operated by the national switch company. Settlement is deferred multilateral net settlement on the central bank's RTGS, three windows per day. Bank A must go live in 9 months with P2P send and receive by phone number and account number, and plans merchant payments in phase 2. The CIO has asked you to brief the executive committee, produce the integration design, and lead the implementation.

## Deliverables

| # | Artefact | Built from | Template |
| --- | --- | --- | --- |
| **C1** | Executive and IT briefing (12–15 slides with speaker notes) | D1, D2, D10, D11, D15 | [bank-briefing-outline.md](templates/bank-briefing-outline.md) |
| **C2** | Integration Solution Design for Bank A, with ADR-001 to ADR-006 | D5–D9, D12–D14, D16 | [solution-design-template.md](templates/solution-design-template.md), [adr-and-rca-templates.md](templates/adr-and-rca-templates.md) |
| **C3** | Implementation plan: phases, workstreams, RACI, risks, certification and cutover | D3, D4, D13, D15, D16 | [implementation-plan-template.md](templates/implementation-plan-template.md) |

### Portfolio index (your evidence)

| ID | Deliverable | Module |
| --- | --- | --- |
| D1 | Business architecture one-pager | [M1](01-milestone-01-business-architecture.md) |
| D2 | Annotated P2P sequence | [M2](02-milestone-02-p2p-lifecycle.md) |
| D3 | Environment baseline record | [M3/M4](03-milestones-03-04-setup-and-run.md) |
| D4 | Onboarding sheet + first-payment evidence | [M5/M6](04-milestones-05-06-two-fis-first-payment.md) |
| D5 | FSPIOP message catalogue | [T1](05-topic-01-fspiop.md) |
| D6 | Ledger and position worksheet | [T2](06-topic-02-central-ledger.md) |
| D7 | Directory strategy note | [T3](07-topic-03-account-lookup.md) |
| D8 | Fee and quote policy | [T4](08-topic-04-quoting.md) |
| D9a / D9b | Transfer risk controls; finality and timeout policy | [T5](09-topic-05-transfers.md), [T6](10-topic-06-fulfilment.md) |
| D10 | Trace report | [M7](11-milestone-07-trace-transaction.md) |
| D11 | Incident RCA | [M8](12-milestone-08-investigate-failure.md) |
| D12 | Adapter design | [T7](13-topic-07-fsp-adapter.md) |
| D13 | Prototype + test evidence | [M9](14-milestone-09-build-fsp-integration.md) |
| D14 | Bank integration solution design (draft) | [M10](15-milestone-10-connect-real-bank.md) |
| D15 | Settlement and reconciliation operating model | [T8](16-topic-08-settlement.md) |
| D16 | Deployment architecture | [T9](17-topic-09-kubernetes.md) |

---

## Mock review: questions you must be able to answer

Prepare for each stakeholder. A strong answer cites **evidence from your labs** where possible.

| Stakeholder | Question | Where your answer comes from |
| --- | --- | --- |
| CEO | "Why should we join, and what does it cost us?" | D1, briefing slides 2–5 |
| CFO / Treasury | "How much liquidity do we need, and when does money actually move?" | D6 (NDC formula), D15 |
| CRO | "What stops a payment going wrong, and who carries the loss?" | D9a, D9b, D11 (controls worked as designed) |
| CISO | "How is our connection secured, and where are our keys?" | C2 security section, ADR-005, reference [09](../09-security-and-controls.md) |
| CIO | "What do we build vs reuse, and what is the hardest part?" | ADR-001, D12, D13, the M10 gap table |
| Head of Core Banking | "Our core has a nightly batch and can't reverse credits. Can we still join?" | ADR-002 (`RESERVED` + `PATCH` / provisional credit), M10 spike |
| Head of Operations | "What happens at 02:00 when a payment is stuck?" | D11 runbook, D10 trace method, monitoring requirements |
| Compliance | "How do we handle consent for publishing phone numbers, and AML screening?" | D7, C2 functional design |
| Scheme operator | "How will you pass certification, and what is your go-live plan?" | C3 phases, certification path (M10), golden-path experience |
| Platform team | "What do you need from us?" | D16 platform checklist |

---

## Assessment rubric

| Criterion | Needs work | Competent | Excellent |
| --- | --- | --- | --- |
| Accuracy | Facts conflict with the protocol or the ledger | Facts correct | Facts correct and backed by lab evidence and versions |
| Business clarity | Technical jargon for executives | Clear value and risk story | Tailored to Bank A's strategy and constraints |
| Integration design | Happy path only | Failure paths and idempotency covered | Unknown outcomes, reconciliation and credit timing designed and tested |
| Security | Generic statements | mTLS, JWS and key custody covered | Key lifecycle, network zones and identity binding to `FSPIOP-Source` |
| Operations | Missing | Monitoring and runbooks listed | Alert thresholds, support model, settlement operations and RCA process |
| Delivery leadership | Vague timeline | Phases and RACI | Risks with mitigations, certification plan, cutover and rollback |
| Evidence discipline | Assumptions unmarked | Mostly tagged | Every claim tagged *[VERIFIED]* or *[ASSUMPTION]* |

**Pass:** "Competent" or better in every row. **Lead-ready:** "Excellent" in at least four rows, including integration design and delivery leadership.

---

## After the capstone: where to go deeper

| Topic | Why | Start from |
| --- | --- | --- |
| ISO 20022 mode | Many schemes mandate it | `fspiop_2.0_iso20022` spec; perf stack with `API_TYPE=iso20022` |
| FX and cross-border | Remittance corridors | `fx` profiles; [diagrams/fx-transfer-sequence.mmd](../diagrams/fx-transfer-sequence.mmd) |
| Request-to-pay and merchant | Phase 2 for Bank A | Upstream transaction-requests-service; the merchant-registry profile |
| Bulk and disbursements | Government and salary payments | Upstream bulk-api-adapter; SDK `bulkTransactions` |
| Inter-scheme connectivity | Cross-network payments | Proxy cache configuration; upstream inter-scheme proxy adapter |
| Community | Stay current, contribute | Mojaloop Foundation community, upstream repositories |

Congratulations: by completing this portfolio you have done the core work of a Mojaloop solution architect on a bank integration.
