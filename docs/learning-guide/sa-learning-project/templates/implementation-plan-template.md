# <Bank name> – Mojaloop Integration Implementation Plan (template)

> Capstone C3. A plan the solution architect can present to a steering committee.

## 1. Phases and milestones

| Phase | Key activities | Exit criteria | Indicative duration |
| --- | --- | --- | --- |
| 0. Mobilise | Governance, team, scheme agreements, environments requested | Charter signed; scheme sandbox access | |
| 1. Design | Solution design (C2), security design, GL design, ADRs | Design approved by bank IT, security, finance, scheme | |
| 2. Build | Integration service, connector config, CBS changes, monitoring | Unit and integration tests green | |
| 3. Scheme certification | TTK/golden-path-style certification against the scheme sandbox | Scheme certificate issued | |
| 4. UAT and NFR | Business UAT, performance, security, DR tests | Sign-offs | |
| 5. Pilot | Limited customers or limits; hypercare | Pilot KPIs met | |
| 6. Go-live and hypercare | Full launch, monitoring, daily reconciliation | Stable for N weeks | |

## 2. Workstreams

| Workstream | Scope | Lead |
| --- | --- | --- |
| Architecture and integration | Integration service, connector, mappings | |
| Core banking | Holds, postings, GL accounts, alias data | |
| Channels | App or USSD journeys, customer messages | |
| Security | PKI, JWS keys, HSM, network, pen-test | |
| Treasury and finance | Prefunding, NDC, settlement, reconciliation | |
| Operations and support | Runbooks, monitoring, L1–L3 support | |
| Testing | Test strategy, certification, UAT, performance | |
| Compliance and legal | Scheme rules, AML/KYC, data protection | |

## 3. RACI (R = responsible, A = accountable, C = consulted, I = informed)

| Activity | Solution architect | Bank IT | Core banking vendor | Security | Treasury | Operations | Scheme operator |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Integration solution design | A/R | C | C | C | C | C | C |
| Connector deployment | C | A/R | I | C | I | C | I |
| CBS posting changes | C | A | R | I | C | I | I |
| Certificates and JWS keys | C | R | I | A | I | I | C |
| Prefunding and NDC | C | I | I | I | A/R | I | C |
| Scheme certification | R | R | C | C | I | C | A |
| Go-live decision | C | R | I | C | C | C | C (A = bank exec) |

## 4. Risk register

| # | Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- | --- |
| 1 | CBS cannot post in real time | | | Provisional credit + suspense GL; answer RESERVED | |
| 2 | Duplicate debits on retries | | | Idempotency keys; GET-before-retry policy | |
| 3 | Liquidity exhausted (NDC breach) at peak | | | Threshold alerts, top-up runbook | |
| 4 | Certificate or key expiry | | | Expiry monitoring; rotation procedure | |
| 5 | Lost callbacks cause unknown outcomes | | | Status polling, daily reconciliation | |

## 5. Cutover and go-live checklist

- [ ] Production certificates and keys installed and verified
- [ ] Participant onboarded in production (accounts, NDC, endpoints, funds-in)
- [ ] Aliases registered (bulk load + ongoing)
- [ ] Monitoring and alerts live; on-call rota published
- [ ] Reconciliation job tested on pilot data
- [ ] Rollback plan (disable channel feature flag; keep inbound running)
