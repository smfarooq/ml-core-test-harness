# Bank Briefing Outline (template for capstone C1)

> 12–15 slides, 30–40 minutes, for a mixed audience of bank executives, IT, risk and operations. Speaker notes in *italics*.

1. **Title**: "Joining <scheme>: instant, interoperable payments for <Bank>"
2. **Why now**: market problem: closed loops, wallet growth, customer demand. *Use the bank's own numbers if available.*
3. **What Mojaloop is, in one picture**: hub, FSPs, settlement bank. *"The hub never holds your customers' money."*
4. **What a payment looks like**: Discovery → Agreement → Transfer, with the customer's screens. *Show the name check and the fee shown before confirmation.*
5. **What changes for the bank**: new channel feature, connector, GL accounts, liquidity at the settlement bank.
6. **Money flows**: prefunding, Net Debit Cap, positions, deferred net settlement. *Show the worked example with numbers.*
7. **Risk controls built in**: NDC, cryptographic finality (condition/fulfilment), expiry, duplicate protection, signed messages.
8. **What the bank must still own**: customer authentication, AML/fraud, CBS postings, reconciliation, key management.
9. **Integration architecture**: channels → integration service → Mojaloop Connector → scheme. *Keep this to one diagram.*
10. **Security model**: mTLS, JWS, HSM, network zones.
11. **Operations model**: monitoring, settlement windows, support, incident handling.
12. **Implementation roadmap**: phases, certification, pilot, go-live.
13. **Team and RACI**: who does what across bank, vendor and scheme.
14. **Key decisions needed from the bank**: identifiers to publish, credit timing, liquidity amount, fee policy, pilot scope.
15. **Q&A and next steps**

Appendix: glossary, error-code policy, NFR targets, evidence from the local lab (screenshots of the TTK sequence diagram, trace report).
