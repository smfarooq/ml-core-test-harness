# ADR and RCA Templates

## Architecture Decision Record (ADR)

```markdown
# ADR-<nnn>: <decision title>

- Status: Proposed | Accepted | Superseded by ADR-<x>
- Date:
- Deciders:

## Context
<What forces are at play? Scheme rules, CBS capabilities, regulation, cost, timelines.>

## Options considered
1. <Option A> – pros / cons
2. <Option B> – pros / cons
3. <Option C> – pros / cons

## Decision
<The chosen option, stated plainly.>

## Consequences
<What becomes easier or harder. Follow-up work. Risks accepted.>

## Evidence
<Lab results, scheme confirmations, vendor statements, links to the journal.>
```

Suggested first ADRs for a bank integration:

| ADR | Decision |
| --- | --- |
| ADR-001 | Integration option: SDK Scheme Adapter vs direct FSPIOP vs packaged product |
| ADR-002 | Incoming credit timing: immediate COMMITTED vs RESERVED + PATCH confirmation |
| ADR-003 | Identifiers to publish (MSISDN, ACCOUNT_ID, IBAN) and consent model |
| ADR-004 | Idempotency and recovery strategy (keys, status polling, reconciliation) |
| ADR-005 | Key management (HSM vs software keystore) and rotation |
| ADR-006 | Hosting of the connector (on-prem DMZ vs cloud) |

## Incident Root-Cause Analysis (RCA) – used in M8 (D11)

```markdown
# RCA: <short title>

| Field | Value |
| --- | --- |
| Incident id / date | |
| Detected by | (alert, customer, reconciliation, test) |
| Severity | |
| Affected transfers (ids) | |
| Customer impact | |
| Financial impact | (e.g. none: positions restored) |

## Timeline
| Time | Event | Evidence (log line, DB row, topic message) |
| --- | --- | --- |

## What happened (facts only)

## Root cause
<Use the "5 whys". Separate the trigger from the underlying cause.>

## Why controls did / did not work
<NDC, expiry, duplicate check, monitoring …>

## Corrective and preventive actions
| Action | Owner | Due |
| --- | --- | --- |

## Lessons for the integration design
```
