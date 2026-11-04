# Diagrams

Standalone Mermaid sources for the larger diagrams in the learning guide. Render them with the VS Code "Markdown Preview Mermaid Support" extension, the Mermaid Live Editor, or `npx @mermaid-js/mermaid-cli -i <file>.mmd -o <file>.svg`.

| File | Shows | Used in |
| --- | --- | --- |
| [high-level-architecture.mmd](high-level-architecture.mmd) | FSPs, switch services, Kafka, databases, operations | [05 §1](../05-system-architecture.md#1-high-level-architecture) |
| [harness-topology.mmd](harness-topology.mmd) | What runs locally and who plays which FSP | [05 §2](../05-system-architecture.md#2-the-harness-topology-what-actually-runs-on-your-laptop) |
| [kafka-transfer-flow.mmd](kafka-transfer-flow.mmd) | Transfer through topics and handlers | [05 §5](../05-system-architecture.md#5-component-interaction-for-a-transfer-kafka-view) |
| [p2p-transfer-sequence.mmd](p2p-transfer-sequence.mmd) | Full discovery → quote → transfer sequence | [07 §2](../07-payment-transaction-lifecycle.md#2-end-to-end-sequence-with-switch-internals) |
| [transfer-state-machine.mmd](transfer-state-machine.mmd) | Internal transfer states | [07 §4](../07-payment-transaction-lifecycle.md#4-transaction-state-management) |
| [settlement-lifecycle.mmd](settlement-lifecycle.mmd) | Settlement window and settlement states | [07 §7](../07-payment-transaction-lifecycle.md#7-settlement-and-reconciliation) |
| [fx-transfer-sequence.mmd](fx-transfer-sequence.mmd) | Cross-currency payment via an FXP | [04 §3.9](../04-business-use-cases.md#39-cross-border-and-cross-currency-fx) |
| [fsp-adapter-architecture.mmd](fsp-adapter-architecture.mmd) | Core Connector + SDK adapter pattern | [12 §3](../12-adapter-development.md#3-conceptual-architecture) |
| [production-deployment.mmd](production-deployment.mmd) | Reference production architecture (recommendation) | [15 §3](../15-production-deployment.md#3-reference-production-architecture-recommendation) |
