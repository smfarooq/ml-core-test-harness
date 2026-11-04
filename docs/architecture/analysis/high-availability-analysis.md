# High Availability Analysis

> **Part A – As-Is** (diagram [04a](../diagrams/04a-resilience-as-is.svg)): how the harness behaves when parts fail, and which resilience mechanisms are built into the services.
> **Part B – Target** (diagram [04b](../diagrams/04b-high-availability-disaster-recovery-target.svg)): a **Proposed** HA design.
> Status labels are defined in the [README](../README.md#status-labels).

---

## Part A – As-Is

### A1. Single points of failure

Everything in the harness runs as **one instance**, and the switch services and data stores have **no restart policy**. Each of these is a single point of failure:

| Component | If it fails | Data at risk |
| --- | --- | --- |
| `kafka` (1 broker, RF 1) | All quotes and transfers stop; unconsumed messages are lost with the container | In-flight messages |
| `mysql` (`central_ledger`) | Ledger, positions, quoting and settlement stop | **Everything** if the container is removed (no persistent volume) |
| `mysql-als` | Lookups stop | Directory data |
| `central-ledger` (API + handlers in one process) | Prepare, fulfil, get, admin and timeouts all stop | – |
| `central-handler-position-batch` | Transfers stay `RECEIVED_PREPARE`, then expire | – |
| `ml-api-adapter` (API + notifications) | No transfer intake and no callbacks | – |
| `account-lookup-service`, oracle | No payee discovery | – |
| `quoting-service` / handler | No quotes | – |
| `redis` | Proxy cache lost (rebuilt) | – |
| SDK + its Redis (FSP side) | The FSP can't send or receive; in-flight SDK state lost | In-flight state |

### A2. Resilience mechanisms built into the services (verified)

| Mechanism | What it does | Evidence | Status |
| --- | --- | --- | --- |
| Asynchronous decoupling | API answers 202 after publishing to Kafka; processing continues independently | Lab (T5) | Implemented |
| Transfer expiry | Every transfer carries an `expiration`; the timeout handler (cron every 15 s) aborts expired ones and releases the reserved funds (3303) | Lab (M8): 2 s expiry → `EXPIRED_RESERVED` | Implemented |
| Idempotency | Resending the same request does not double-process; the current state is re-sent | Lab (T4, T5) | Implemented |
| Status query | `GET /transfers/{id}` and `GET /quotes/{id}` let an FSP recover after a timeout | Lab (M9) | Implemented |
| Ordered position processing | Position messages keyed per account | Lab (T5) | Implemented |
| Health endpoints | `/health` on every service; 17 Compose health checks | compose | Configured |
| Start-up dependency waiting | `wait4` retries dependencies (60 × 10 s) | [wait4.config.js](../../../docker/wait4/wait4.config.js) | Implemented |
| Connector timeouts | SDK request timeout (`REQUEST_PROCESSING_TIMEOUT_SECONDS=30` in the harness) and transfer expiry (`EXPIRY_SECONDS=60`) | `envs/*-sdk.env`; lab (M10: SDK gave up at ~44 s, switch expired at ~62 s) | Configured |
| Dead-letter queues | **Not found** in configuration or topics | – | Unverified |
| Retries of failed callbacks | Not configured in this repository | – | Unverified |

### A3. What scales in the perf stack

[docker-compose-perf.yml](../../../docker-compose-perf.yml) splits every handler into its own container with a replica variable, and [perf.env](../../../perf.env) raises Kafka partitions for position topics to 25 and sets `partition.assignment.strategy=cooperative-sticky` (smoother consumer-group rebalances). This shows the **intended** scaling model: stateless services scale horizontally; handler parallelism is bounded by partitions.

---

## Part B – Target HA design — Proposed

### B1. Principles

1. **No single instance of anything** on the transaction path.
2. **The database is the source of truth.** Kafka is transport; the protocol's expiry and status query make lost in-flight messages recoverable (the transfer expires and both FSPs are told).
3. **Spread across 3 availability zones**, with anti-affinity and PodDisruptionBudgets.
4. **Fail fast and visibly**: health probes, alerts and runbooks.

### B2. Component HA

| Component | HA mechanism | Failure handling |
| --- | --- | --- |
| API services | ≥ 2 replicas per zone set, readiness probes, load-balanced | Kubernetes restarts failed pods; the load balancer drains them |
| Kafka handlers | ≥ 2 replicas per consumer group | Group rebalance moves partitions to healthy consumers |
| Timeout handler | Run as a singleton (1 replica) with fast restart, or confirm upstream support for multiple replicas | Short outage only delays expiry |
| Kafka | 3 brokers across zones, RF 3, `min.insync.replicas=2`, `acks=all` producers | Leader election; no data loss with one broker down |
| MySQL | Synchronous replication cluster (3 nodes) with automatic primary failover, or a managed HA service | Brief write pause; in-flight transfers may expire and are reported (3303) |
| MongoDB | 3-node replica set | Automatic election |
| Redis | Sentinel or cluster | Failover; caches rebuild |
| Ingress / gateway | ≥ 2 instances per zone behind the load balancer | – |
| FSP side | 2+ connectors with HA Redis; idempotent backend | FSP's responsibility; status query after timeouts |

### B3. Failure scenarios (target behaviour)

| # | Scenario | Expected behaviour | Customer impact |
| --- | --- | --- | --- |
| 1 | Normal operation | All replicas healthy across 3 zones | – |
| 2 | One service pod fails | Restarted by Kubernetes; other replicas serve; consumer group rebalances | None or seconds of latency |
| 3 | One worker node fails | Pods rescheduled to other nodes; PDBs keep minimum replicas | Seconds |
| 4 | Database primary fails | Automatic failover to a replica | Seconds of failed writes; some in-flight transfers expire (3303), FSPs reconcile |
| 5 | One Kafka broker fails | Partition leaders move; RF 3 keeps data | Brief latency |
| 6 | Whole cluster or site fails | Disaster recovery (see [disaster-recovery-analysis.md](disaster-recovery-analysis.md)) | Outage until failover (RTO) |

Validate each scenario with chaos tests in pre-production, using the golden-path suite to confirm correctness afterwards.
