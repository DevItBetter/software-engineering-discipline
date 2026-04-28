# Distributed Systems Deep Dive

Working reference of the distributed systems concepts that come up most often in architecture review. Drawn from Kleppmann's *Designing Data-Intensive Applications*, Lamport, the Jepsen reports, and the practical literature.

## CAP and PACELC

**CAP** (Brewer, 2000): in the presence of a network **P**artition, you must choose between **C**onsistency and **A**vailability. You cannot have both.

The catch in the popular framing: CAP is about *behavior under partition*. Most systems aren't partitioned most of the time, so CAP underspecifies the trade-off they're actually making.

**PACELC** (Daniel Abadi, 2010): under **P**artition, choose **A**vailability or **C**onsistency; **E**lse, choose **L**atency or **C**onsistency. This is more useful — it captures the latency-vs-consistency trade-off you face every millisecond, not just during the rare partition.

Most systems are:
- **PA/EL**: available under partition, low latency normally, accepting eventual consistency. Cassandra, DynamoDB, most caches.
- **PC/EC**: consistent under partition (refuses to serve when uncertain), consistent always but at higher latency. Spanner, FoundationDB, etcd, Zookeeper.
- **PA/EC**: available under partition (eventually consistent then), consistent normally. Some configurations of Riak, MongoDB.

For your design, name the choice explicitly. "We chose PA/EL because cart updates can tolerate seconds of staleness; we accept that the user might briefly see a stale total."

## Consistency models

A spectrum from strongest to weakest:

- **Linearizability** (single-key): every operation appears to happen instantaneously at some point between its start and end; reads see the most recent write. The strongest single-object guarantee.
- **Serializability** (multi-key transactions): the result is as if transactions executed one at a time in some serial order. The strongest multi-object guarantee.
- **Strict serializability** = serializability + linearizability. Spanner provides this. Hard.
- **Snapshot isolation**: each transaction sees a consistent snapshot of the database at the time it started. Common; cheap; some anomalies (write skew) possible.
- **Read committed**: see only committed writes. Doesn't prevent reads from seeing different values within a single transaction.
- **Causal consistency**: if A causally precedes B, all observers see A before B. Doesn't constrain unrelated operations.
- **Eventual consistency**: replicas eventually agree. No bound on when.

Honest assessment of what your storage actually provides: don't assume; check the docs and the Jepsen tests. Some databases provide stronger guarantees than they advertise; many provide weaker guarantees than people assume.

## Consensus

Getting a group of machines to agree on a value despite failures. Foundational for replication, leader election, distributed locks, and atomic commit.

The well-tested algorithms:
- **Paxos** (Lamport, 1989): the original; notoriously hard to implement correctly.
- **Multi-Paxos**: extension for multiple decisions in sequence.
- **Raft** (Ongaro, 2014): designed to be understandable; the modern default for most consensus needs. Used in etcd, Consul, CockroachDB.

Consensus algorithms have inherent costs:
- **A majority of nodes must be reachable.** With 5 nodes, you can lose 2 and still operate; with 3 nodes, 1.
- **Latency is bounded below by the slowest replica in the majority.**
- **Throughput is limited** by the leader's ability to coordinate.

Don't implement consensus from scratch. Use a system that provides it (etcd, Zookeeper, Consul) or a database that uses it internally (Spanner, CockroachDB, FoundationDB).

## Replication

How do you have multiple copies of data?

- **Single-leader (primary-replica)**: writes go to the leader; reads from any replica. Simple. Asynchronous replicas are eventually consistent; synchronous replicas are strongly consistent but slower.
- **Multi-leader**: writes can go to any leader; leaders replicate to each other. Higher write availability; conflicts must be resolved.
- **Leaderless** (Dynamo-style): writes go to multiple nodes; reads go to multiple nodes; quorum determines what's considered "committed". Tunable consistency.

Each has failure modes worth knowing:
- **Single-leader**: leader failure → election delay; writes blocked during election.
- **Multi-leader**: write conflicts → lost updates if not handled; complex conflict resolution.
- **Leaderless**: read repair, hinted handoff, anti-entropy — all to converge eventually. Subtle bugs around quorum semantics.

## Partitioning (sharding)

How do you distribute data across machines when one machine can't hold it?

- **Range partitioning**: consecutive keys go together. Good for range queries; can produce hot spots if data is skewed.
- **Hash partitioning**: hash of the key determines partition. Even distribution; bad for range queries.
- **Composite**: hash of one field, then range within. Good when "all of user X's data" should be together.

Re-partitioning (changing the partition scheme) is expensive. Choose carefully. **Consistent hashing** minimizes data movement when partitions change.

Hot spots are the enemy: a partition scheme that produces them under realistic data shape will dominate latency. Test with production-shape data, not synthetic uniform.

## Transactions, distributed and otherwise

A transaction has the ACID properties (when fully provided):
- **A**tomicity: all-or-nothing.
- **C**onsistency: invariants preserved (application-level).
- **I**solation: concurrent transactions don't see each other's intermediate state.
- **D**urability: committed data survives.

A single-node SQL database with serializable isolation gives you all four.

A distributed transaction (across multiple resource managers) gives you fewer:
- **Two-phase commit (2PC)** can give you atomicity and durability across resources, but blocks under coordinator failure and is widely avoided.
- **Sagas** give you eventual atomicity (compensating transactions), no isolation in the middle.
- **Reservations / two-phase reservations**: reserve in step 1; commit in step 2; release if commit fails. Used for booking systems.
- **Outbox pattern + idempotent consumers**: practical "effectively-once" message delivery without distributed transactions.

For most architectures: keep transactions inside a service (where one database can give you ACID), and use sagas / outbox / idempotency for cross-service workflows.

## Time, clocks, and ordering

Distributed systems can't agree on time. A few key ideas:

- **Wall clocks drift.** Two machines disagree by milliseconds (NTP) to seconds (untracked). Never compare wall-clock timestamps across machines.
- **Monotonic clocks** are stable on one machine but uncorrelated across machines.
- **Logical clocks** (Lamport timestamps) capture causality but not real time. If event A causally precedes event B, A's Lamport timestamp < B's.
- **Vector clocks** capture concurrent vs causally-ordered events more precisely than Lamport timestamps.
- **Hybrid logical clocks** combine wall clock and logical, getting some ordering and some real-time meaning.
- **TrueTime** (Spanner): bounded-uncertainty wall clock from atomic clocks and GPS. Lets Spanner provide global linearizability with bounded latency.

For most application code: don't use timestamps for ordering decisions. Use a logical sequence (a monotonic counter, a UUID v7, a database-assigned sequence) when you need ordering.

## Failure modes

Things that fail in production and the patterns that catch them:

- **Cascade failure**: one slow service causes upstream services to slow, which cause their callers to slow, which... Mitigated by timeouts (don't wait forever), bulkheads (separate resource pools), circuit breakers (stop calling failing services).
- **Thundering herd**: many clients retry simultaneously after a failure. Mitigated by jittered backoff.
- **Cache stampede**: cold cache + concurrent requests for the same value = N parallel computes. Mitigated by single-flight pattern.
- **Hot key**: one key gets disproportionate traffic. Mitigated by client-side caching, request coalescing, splitting the hot key.
- **Slow loris**: clients holding open connections without making requests. Mitigated by connection limits and idle timeouts.
- **Retry storm**: a downstream blip causes everyone to retry, which makes it worse. Mitigated by circuit breakers and adaptive retry budgets.
- **Memory leak in production**: usually shows as slowly degrading latency until OOM. Mitigated by heap monitoring, regular restarts.
- **Backpressure failure**: a fast producer overwhelming a slow consumer fills queues until OOM. Mitigated by bounded queues and explicit backpressure signals.

For each architecture: walk through this list. Which of these can happen here? What stops them?

## The trap: "We'll add it later"

Many architecture failures share the pattern: we built without X, scaled, then needed X, and X is hard to add now.

The X is often:
- **Multi-tenancy.** Adding tenant isolation to a system that wasn't designed for it requires touching every query.
- **Auditability.** Adding audit logs after the fact misses everything that already happened.
- **Soft deletes.** Reversibility built in late means data already gone.
- **Versioning.** Adding versions to a schema with existing data requires migration.
- **Encryption at rest.** Encrypting a non-encrypted system requires re-keying everything.
- **Region failover.** Building one region, then adding two, requires data layer rework.

Decide which of these you actually need at architecture time. Adding them later is much more expensive than building them in.

## How to write a good architecture decision

For substantial decisions, an Architecture Decision Record (ADR) captures the choice and the reasoning. A good ADR has:

1. **Title** (numbered, descriptive): "ADR 0023: Use Kafka for the order-events stream."
2. **Status**: proposed / accepted / superseded.
3. **Context**: what's the situation? What constraints apply?
4. **Decision**: what we decided.
5. **Consequences**: what becomes easier; what becomes harder; what trade-offs we accepted.
6. **Alternatives considered**: what we rejected and why.

Future readers (including future you) can read the ADR, understand the context, and decide whether the decision still holds. Without it, the why is lost — and the wrong people start arguing for changes whose reasoning was already considered.
