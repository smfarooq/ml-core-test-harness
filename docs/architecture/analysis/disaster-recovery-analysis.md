# Disaster Recovery Analysis

> **As-Is:** this repository has **no** backup, replication or disaster-recovery configuration (diagram [04a](../diagrams/04a-resilience-as-is.svg)).
> **Target:** a **Proposed** business-continuity and DR design (diagram [04b](../diagrams/04b-high-availability-disaster-recovery-target.svg)).
> RPO and RTO are **not defined anywhere in the repository**. They are shown as design decisions, never as numbers. Status labels are defined in the [README](../README.md#status-labels).

---

## 1. As-Is

| Capability | As-is | Status |
| --- | --- | --- |
| Database backups | None | Unverified (none found) |
| Database replication | None (single MySQL instances) | Configured (single instance) |
| Persistent volumes | None for switch data; `down -v` deletes everything | Configured |
| Message persistence | Kafka on container storage, RF 1 | Configured |
| DR site / regional recovery | None | – |
| Transaction recovery | Protocol-level: expiry (3303), idempotent resend, `GET` status queries | Implemented (verified) |
| RPO / RTO | Not defined | – |

In the harness, losing the `mysql` container loses the ledger. That is fine for testing; the provisioning collections rebuild a clean state in minutes.

## 2. What must be protected (target)

| Data | Why | Recovery source |
| --- | --- | --- |
| `central_ledger` (participants, positions, transfers, settlement) | Legal and financial record; positions are money | Replicated DB + point-in-time backups |
| `account_lookup`, oracle data | Directory; rebuildable but slow | DB backups |
| Configuration (Helm values, participant endpoints, limits) | Needed to rebuild the switch | Git (GitOps) + DB |
| Keys and certificates | Without them, no FSP can connect | HSM / vault replication |
| Audit events | Regulatory evidence | Immutable storage replicated off-site |
| Kafka in-flight messages | Short-lived | **Not** a recovery source; the protocol recovers via expiry and status queries |

## 3. Target DR design — Proposed

| Element | Proposal |
| --- | --- |
| Topology | Primary site (3 zones) **active**; DR site **warm standby** with the same Helm release deployed by GitOps, scaled down |
| MySQL | Synchronous replication inside the primary site; **asynchronous** replication to the DR site; point-in-time backups to object storage in a third location |
| Kafka | Independent cluster per site; no cross-site mirroring needed for correctness (DB is the record). Topics recreated by provisioning. |
| MongoDB / audit | Replica set; backups and immutable archive off-site |
| Secrets and keys | Replicated vault / HSM in the DR site |
| DNS / gateway | Global load balancer or DNS failover for the FSPIOP endpoints; FSPs must trust the DR site's certificates in advance |
| Runbooks | Declare disaster → freeze → promote DR DB → scale up DR services → switch DNS → reconcile with FSPs → resume |

## 4. Scenario walk-through

| # | Scenario | Detection | Recovery | Data loss window |
| --- | --- | --- | --- | --- |
| 1 | Normal | – | – | – |
| 2 | Service failure | Probes, alerts | Kubernetes restart; replicas continue | None |
| 3 | Worker node failure | Node not ready | Reschedule pods | None |
| 4 | Database failure (primary) | DB health, error rates | Automatic in-site failover | None with synchronous replication |
| 5 | Message broker failure (one broker) | Under-replicated partitions | Leader election (RF 3) | None |
| 6 | Complete cluster or site failure | Site monitoring, synthetic transfers | DR runbook: promote DR DB, scale up DR site, switch DNS | **RPO = design decision** (asynchronous replication lag) |
| 7 | Restoration and failback | – | Rebuild primary from DR, resynchronise, planned switch-back in a maintenance window | – |

## 5. Transaction recovery after a disaster

Transfers in flight at the moment of failure end in one of three states:

1. **Committed in the replicated DB** → final; FSPs learn it via `GET /transfers/{id}`.
2. **Reserved but not committed** → expire after failover (3303); funds released.
3. **Not yet recorded** (lost in Kafka or the replication gap) → the payer FSP's status query returns **3208** (not found) and the FSP treats it as failed.

Afterwards, every FSP reconciles its records against the switch by `transferId`, and the hub operator reconciles positions before the next settlement window closes.

## 6. Decisions the scheme must record

| Decision | Options to consider |
| --- | --- |
| **RPO** (acceptable data loss) | 0 (synchronous cross-site, higher latency) vs. seconds (asynchronous) |
| **RTO** (acceptable downtime) | Minutes (automated warm standby) vs. hours (manual restore) |
| DR test frequency | e.g. twice a year, with FSP participation |
| Settlement during DR | Suspend window closure until reconciliation completes |
| Communication | Who informs FSPs, regulators and the settlement bank, and when |
