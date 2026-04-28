# Distributed State Patterns

Working reference of patterns for managing state across machines without losing your mind.

## Eventual consistency

The default of most distributed systems. Writes propagate over time; reads may see stale data. Eventually all replicas agree.

Application implications:

- **Reads can be stale.** Your code calls "get user" right after "update user" — it may get the old value.
- **Writes can be reordered** when seen from different observers. Two writes A then B may appear B then A.
- **Recently-written data may not appear in queries.** Indexes are updated asynchronously.

Design responses:

- **Read-your-writes consistency.** A user who just wrote sees their own write. Achieved by routing the user's reads to the same replica or by including a session token that requires the latest data.
- **Causal consistency.** If event A causes event B, all observers see A before B. Often what's actually wanted; harder than eventual but cheaper than strong.
- **Bounded staleness.** "Reads may be at most N seconds old." A useful contract for caches and read replicas.
- **Versioning.** Each write produces a new version; readers can detect and reconcile divergence.

The trap: using a database with eventual consistency and writing application code that assumes strong consistency. The bug only shows up under load or partition.

## CRDTs (Conflict-free Replicated Data Types)

Data structures designed for distributed editing without coordination. Multiple replicas can be updated independently and concurrently; merging is automatic and convergent.

Examples:
- **G-counter** (grow-only counter). Each replica counts its own increments; the total is the sum.
- **PN-counter** (positive/negative counter). Two G-counters: one for adds, one for removes.
- **OR-set** (observed-remove set). Add/remove a value; "remove" only removes elements observed-on-add.
- **LWW-register** (last-write-wins register). Highest-timestamp wins. Simple but can lose updates.
- **RGA / Logoot** (sequence CRDTs for collaborative text editing).

When useful:
- Real-time collaborative editing (Google Docs, Figma).
- Distributed counters where strong consistency is too expensive.
- Offline-first apps that sync when connection returns.

When NOT useful:
- Anything where data loss is unacceptable (LWW silently drops concurrent updates).
- Anything with strict invariants that must hold across replicas (e.g., "balance never negative" — can't be enforced with a CRDT).

## Sagas (long-running distributed transactions)

When a business operation spans multiple services and you can't use a distributed transaction, a saga is a sequence of local transactions with compensating actions for rollback.

Example: book-flight-and-hotel.

1. Reserve flight (local transaction in flight service).
2. Reserve hotel (local transaction in hotel service).
3. Charge card (local transaction in payment service).

If step 3 fails, compensate: cancel hotel reservation, cancel flight reservation.

Two flavors:
- **Choreography**: each service publishes events; other services react. No central coordinator. Simpler to start, harder to reason about.
- **Orchestration**: one orchestrator service drives the saga. Easier to reason about; the orchestrator is a coordination point.

Sagas are not transactions. They have weaker semantics:
- The middle of a saga is observable (someone might see the flight reserved while the hotel reservation is still in progress).
- Compensation may not perfectly undo (hotel might charge a cancellation fee).
- The compensating action might also fail.

Use sagas when distributed transactions are unavailable but coordinated multi-service operations are needed. Document the partial-state behavior; design the UX to tolerate it.

## Outbox pattern

Problem: a service needs to atomically (a) update its own database and (b) publish an event to a queue. If you do them sequentially, you can crash between them and lose the event.

Solution: write the event to an "outbox" table in the *same* database transaction as the state change. A separate process polls the outbox and publishes to the queue. After publishing, mark the row processed.

Now the state change and the intent-to-publish are atomic (one transaction). The publish can be retried independently. Duplicates are possible (publish, then crash before marking processed → republish on next poll), so consumers must be idempotent.

This is the standard solution for "transactionally update DB and publish event." Use it; don't try to coordinate two systems atomically without it (that path leads to two-phase commit, which is rarely worth it).

## Inbox pattern

The receiving side of the outbox. When a consumer receives a message, it records the message id in an "inbox" table within the same transaction as processing the message. Subsequent receipts of the same id are no-ops.

This is the standard idempotency mechanism for at-least-once message delivery. Combined with outbox, you have effectively-exactly-once semantics for the application.

## Two-phase commit (and why to mostly avoid it)

Distributed transactions across multiple resource managers. Phase 1: all participants prepare (vote yes/no). Phase 2: if all yes, all commit; if any no, all rollback.

The good: gives you distributed atomicity.

The bad: blocks while waiting (a slow participant stalls everyone), can deadlock under coordinator failure, has well-known protocol bugs around the in-doubt state. Rarely supported by modern infrastructure.

The verdict: only use 2PC if your platform supports it natively (some XA databases, some message brokers) and you have very strong reasons to need atomicity. Otherwise: sagas + outbox + idempotent consumers.

## Leader election / consensus

When a distributed system needs one (and only one) actor to do something, you need leader election. The well-tested algorithms are Paxos, Raft, and Multi-Paxos.

Don't implement these. Use a library (etcd, Zookeeper, Consul) or a system that provides them as a primitive. The number of subtle bugs in homegrown leader election is enormous; the failure mode is "two leaders" and the consequences are usually data corruption.

## Quorum reads/writes

For a system with N replicas, write to W of them and read from R of them. As long as W + R > N, every read intersects with at least one replica that has the latest write — strongly consistent.

Trade-offs:
- High W: writes are slow (waits for many replicas) but recent writes are immediately visible.
- High R: reads are slow but always consistent.
- Both moderate: balanced.

Cassandra/Dynamo-style systems expose these as tunables per query. The application chooses per-operation what consistency level to pay for.

## CAP / PACELC, practically

CAP: in a partition, choose Consistency or Availability. Not both.

PACELC (Daniel Abadi): even when there's no partition (Else), choose Latency or Consistency.

Most applications:
- **AP under partition** (return possibly-stale data, keep serving). Failover databases, caches, eventually consistent stores.
- **EL** (low latency at the cost of consistency). Read replicas with replication lag.

Some applications need:
- **CP under partition** (refuse to answer if not certain). Financial ledgers, identity systems. Use a database that does this honestly (Spanner, FoundationDB, CockroachDB, etcd).

The wrong move: using an AP system and writing code that assumes CP semantics. Eventually you'll see the partition (or just a slow replica) and the assumption breaks silently.

## Tombstones, garbage collection, and the "delete" problem

Distributed systems can't truly delete. Other replicas may not know yet. So "delete" usually means "write a tombstone" (a marker that says "deleted at time T"). Tombstones must propagate; eventually they can be garbage-collected.

Implications:
- A "delete" is a write, not the absence of a write.
- Naive "remove from set then re-add" can fail to remove if the remove arrives before the original add.
- Tombstone retention must be longer than the maximum partition / failure / replication delay, or the deleted item resurrects.

If you're designing distributed deletion logic, consult the literature — this is one of the trickiest areas. The "Delete is hard" problem comes up in sync engines, distributed databases, distributed file systems, and gossip protocols.

## When to escalate to a real distributed-systems expert

- Designing a new distributed protocol.
- Choosing between consensus algorithms.
- Anything involving "we're going to handle the consistency manually."
- A bug that involves "we sometimes see inconsistent state."
- A system whose correctness depends on event ordering across machines.

The cost of getting these wrong is data corruption that's discovered months later. The cost of an expert review is far smaller.

The Jepsen test reports (jepsen.io) are humbling reading: even production-grade distributed databases written by experts have shipped with subtle correctness bugs. Don't expect to do better with an afternoon's effort.
