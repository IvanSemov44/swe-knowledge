# Distributed Systems
<!-- level: Senior | CAP theorem, eventual consistency, distributed transactions, CRDTs, consensus, consistent hashing -->

## CAP Theorem (Deep)

Every distributed system can provide at most **two** of these three guarantees simultaneously:

- **Consistency (C)** — Every read returns the most recent write or an error. This is *linearizability*: the system behaves as if there is a single, atomic copy of the data.
- **Availability (A)** — Every request receives a non-error response (not necessarily the most recent data).
- **Partition Tolerance (P)** — The system continues operating even when network messages are dropped or delayed between nodes.

**Why P is not optional:** Networks always partition. Routers fail, cables are cut, GCP us-east loses connectivity to us-west for 30 seconds. If you don't tolerate partitions, your system goes fully offline the moment a packet is lost. Every real distributed system must choose P — which means the real trade-off is **C vs A during a partition**.

| Type | Examples | During partition: |
|------|----------|-------------------|
| CP | HBase, ZooKeeper, etcd, Consul | Returns an error rather than stale data |
| AP | Cassandra, DynamoDB, CouchDB | Returns potentially stale data rather than an error |
| CA | Single-node SQL | Not a real category in distributed systems |

**The "CA" myth:** A CA system only makes sense when there are no partitions — i.e., a single node. The moment you add replication, you must handle partitions. "CA" is not a valid choice for distributed systems.

**PACELC model (practical extension):** CAP only describes behavior *during a partition*. PACELC adds the trade-off that exists *even without a partition*:

- During a **P**artition: choose **A**vailability or **C**onsistency (same as CAP)
- **E**lse (no partition): choose **L**atency or **C**onsistency

DynamoDB is PA/EL: available during partition, low latency during normal operation (at the cost of consistency). etcd is PC/EC: consistent during partition, consistent during normal operation (at higher latency cost).

**Interview trap — "Where does SQL Server sit?"**
It depends. A single SQL Server node is effectively CP (it doesn't partition). With synchronous replication, you're CP — the primary waits for acknowledgment, sacrificing availability. With asynchronous replication (log shipping, read replicas), you're AP — replicas may serve stale reads but won't go down. The interviewer wants you to explain *it depends on the replication config*, not give a one-word answer.

---

## BASE vs ACID

### ACID (Relational DBs)
- **Atomicity** — Transaction succeeds completely or not at all. No partial writes.
- **Consistency** — DB transitions from one valid state to another. Constraints, triggers, cascades all run.
- **Isolation** — Concurrent transactions don't interfere. Levels control *how* isolated.
- **Durability** — Committed data survives crashes (write-ahead log, fsync).

**Isolation levels** (weakest to strongest):

| Level | Dirty Read | Non-repeatable Read | Phantom Read |
|-------|-----------|--------------------|-|
| Read Uncommitted | Yes | Yes | Yes |
| Read Committed | No | Yes | Yes |
| Repeatable Read | No | No | Yes |
| Serializable | No | No | No |

SQL Server default: Read Committed. EF Core default on SQL Server: Read Committed.

### BASE (Distributed Systems)
- **Basically Available** — System remains available but may return stale or approximate data.
- **Soft state** — State may change over time even without new input (due to replication convergence).
- **Eventually consistent** — Given no new updates, all replicas converge to the same value.

| Property | ACID | BASE |
|----------|------|------|
| Consistency | Strong (immediate) | Eventual |
| Availability | May sacrifice during conflict | Prioritized |
| Transactions | Multi-row, atomic | Per-document or saga-based |
| Latency | Higher (sync coordination) | Lower (async replication) |
| Use case | Financial, inventory, reservations | Social feeds, analytics, carts |

You don't choose one model globally — use ACID where correctness is non-negotiable (payments), BASE where scale and availability matter more (product catalog reads).

---

## Eventual Consistency Patterns

Eventual consistency is a spectrum. These patterns let you reason about which guarantees you actually get:

**Read-your-writes:** After a user writes data, their subsequent reads always return that write or a newer version. Achieved by routing that user's reads to the primary (sticky routing) or by reading with a causally consistent token. DynamoDB: strongly consistent read on the record you just wrote.

**Monotonic reads:** If a client has seen version N of a value, it will never see an older version N-1. Achieved with session affinity — route all reads for a session to the same replica. Prevents seeing "time travel" when a load balancer sends requests to different replicas.

**Causal consistency:** If operation B causally depends on operation A (e.g., B reads a value written by A), then all nodes see A before B. This is stronger than eventual but weaker than strong consistency. Implemented via vector clocks or logical timestamps.

**Strong Eventual Consistency (SEC):** Any two nodes that receive the *same set* of updates will be in the same state, regardless of the order in which updates arrived. This is the key property of CRDTs. There is no conflict — the data structure design guarantees convergence.

**DynamoDB read consistency modes:**
- **Eventually consistent reads** (default): reads from any replica, may be up to a few seconds stale, costs 0.5x read capacity units.
- **Strongly consistent reads**: always reads from the leader, guarantees latest data, costs 1x read capacity units, higher latency. Use for order totals, inventory checks — not for product listings.

---

## Two-Phase Commit (2PC)

2PC is a distributed protocol for achieving atomicity across multiple participants (services, databases).

**Roles:** Coordinator (the service or middleware orchestrating the transaction) and Participants (the services or DBs doing the work).

**Phase 1 — Prepare:**
1. Coordinator sends `PREPARE` to all participants.
2. Each participant executes the transaction locally, writes to a redo log, but does **not** commit.
3. Each participant responds `YES` (ready) or `NO` (abort).

**Phase 2 — Commit or Abort:**
1. If all participants said YES: coordinator sends `COMMIT`. Each participant commits and releases locks.
2. If any participant said NO: coordinator sends `ABORT`. Each participant rolls back.

**The blocking problem:** If the coordinator crashes *after* sending `PREPARE` but *before* sending `COMMIT`, participants are stuck in a prepared state — they hold locks, cannot proceed, cannot safely abort on their own. They are blocked until the coordinator recovers. This can take minutes in practice.

**Why 2PC is avoided in microservices:**
- Requires synchronous coordination between all participants — high latency, tight coupling.
- A single slow participant blocks all others.
- Coordinator is a single point of failure.
- Participants must hold database locks across network round trips.

**Three-Phase Commit (3PC):** Adds a `PRE-COMMIT` phase between Prepare and Commit so participants know the coordinator's decision before it crashes. Reduces blocking. In practice, it's rarely used because it's complex, still doesn't work under arbitrary network partitions, and Sagas solve the problem more practically.

---

## Saga Pattern

A Saga is a sequence of local transactions. Each local transaction updates one service and publishes a message or event that triggers the next step. If any step fails, compensating transactions undo the previous steps.

**Compensating transactions:** Must be defined for every step. They are not rollbacks — the original transaction already committed. They are new transactions that semantically reverse the effect. Example: if `ReserveInventory` succeeded, the compensation is `ReleaseInventory`.

### Choreography Sagas
Each service emits events and reacts to events from others. No central coordinator.

```
OrderService → OrderCreated event
  → PaymentService processes → PaymentProcessed event
  → InventoryService processes → InventoryReserved event
  → ShippingService processes → ShipmentScheduled event
```

- Pro: decoupled, no single point of failure.
- Con: hard to track overall saga state, difficult to debug distributed flow, risk of cyclic event chains.
- Use for: simple 2–3 step flows with low complexity.

### Orchestration Sagas
A central saga orchestrator (a state machine) sends commands to services and waits for replies.

```csharp
// MassTransit saga state machine (simplified)
public class OrderSagaStateMachine : MassTransitStateMachine<OrderSagaState>
{
    public State PaymentPending { get; private set; }
    public State InventoryReserving { get; private set; }
    public State Completed { get; private set; }
    public State Compensating { get; private set; }

    public Event<OrderCreated> OrderCreated { get; private set; }
    public Event<PaymentProcessed> PaymentProcessed { get; private set; }
    public Event<PaymentFailed> PaymentFailed { get; private set; }

    public OrderSagaStateMachine()
    {
        InstanceState(x => x.CurrentState);

        Initially(
            When(OrderCreated)
                .Then(ctx => ctx.Saga.OrderId = ctx.Message.OrderId)
                .TransitionTo(PaymentPending)
                .Send(new Uri("queue:payment"), ctx => new ProcessPaymentCommand
                {
                    OrderId = ctx.Saga.OrderId,
                    Amount = ctx.Message.TotalAmount
                }));

        During(PaymentPending,
            When(PaymentProcessed)
                .TransitionTo(InventoryReserving)
                .Send(new Uri("queue:inventory"), ctx => new ReserveInventoryCommand
                {
                    OrderId = ctx.Saga.OrderId
                }),
            When(PaymentFailed)
                .TransitionTo(Compensating)
                .Send(new Uri("queue:order"), ctx => new CancelOrderCommand
                {
                    OrderId = ctx.Saga.OrderId,
                    Reason = ctx.Message.FailureReason
                }));
    }
}
```

- Pro: clear visibility into saga state, easy to add steps, centralized error handling.
- Con: orchestrator is coupled to all participant services.
- Use for: complex multi-step flows requiring explicit compensation and observability.

---

## Idempotency at Scale

**Natural idempotency:**
- `GET` — reading the same resource twice has no side effects.
- `DELETE` — deleting an already-deleted resource returns 404 or 200; same end state.
- `PUT` — replacing a resource with the same data is idempotent by definition.

**POST idempotency — the hard case:** POST is not naturally idempotent. If the client retries due to a timeout, the server may execute the operation twice (double charge, duplicate order).

**Solution: Idempotency keys**

```csharp
[HttpPost("orders")]
public async Task<IActionResult> CreateOrder(
    [FromHeader(Name = "Idempotency-Key")] string idempotencyKey,
    [FromBody] CreateOrderRequest request,
    CancellationToken cancellationToken = default)
{
    // Check if we've already processed this key
    var cached = await _idempotencyStore.GetAsync(idempotencyKey, cancellationToken);
    if (cached is not null)
        return Ok(cached); // Return the previously computed response

    var result = await _mediator.Send(new CreateOrderCommand(request), cancellationToken);
    if (result.IsFailure)
        return BadRequest(result.Error);

    // Store the result with TTL (e.g., 24 hours)
    await _idempotencyStore.SetAsync(idempotencyKey, result.Value, TimeSpan.FromHours(24), cancellationToken);
    return Ok(result.Value);
}
```

The idempotency store is typically Redis. Key: `idempotency:{userId}:{idempotencyKey}`. TTL prevents unbounded growth.

**Outbox + inbox deduplication:** When using the outbox pattern, include a `MessageId` (UUID) in every outgoing message. The consumer maintains an `InboxMessages` table with `MessageId` as the primary key. Before processing, insert the `MessageId`; if the insert fails (duplicate key), skip processing. This achieves effectively-once processing on top of at-least-once delivery.

**At-least-once + idempotent consumer = effectively exactly-once.** True exactly-once delivery across distributed systems is theoretically impossible without heavy coordination (and 2PC). The practical answer is idempotent consumers.

---

## Vector Clocks

**The problem:** Wall clocks in distributed systems drift. Two events on two nodes may have the same timestamp, or the ordering may be wrong. You cannot use wall time to determine causality.

**Vector clock:** Each node maintains an array of counters, one per node in the system. Example for 3 nodes A, B, C: `[A:0, B:0, C:0]`.

**Rules:**
1. On any local event, increment your own counter: `A: [A:1, B:0, C:0]`.
2. When sending a message, include your current vector clock.
3. On receiving a message with clock `V_msg`, merge: `V_local[i] = max(V_local[i], V_msg[i])` for all i, then increment your own counter.

**Comparing clocks:**
- `V1 < V2` (V1 happened-before V2): every element of V1 ≤ corresponding element of V2, and at least one is strictly less.
- **Concurrent:** neither V1 < V2 nor V2 < V1 — this means conflict.

**Example conflict:** Node A writes key X with clock `[A:2, B:1]`. Node B writes key X with clock `[A:1, B:2]`. Neither dominates. This is a concurrent write — both nodes acted on the same previous state. The system must resolve the conflict (LWW by timestamp, or surface it to the client).

**Used in:** DynamoDB and Riak use vector clocks (or their variants) for detecting write conflicts on the same key. DynamoDB surfaces conflicts as multiple values for a key; the client or application logic resolves them.

---

## CRDTs (Conflict-free Replicated Data Types)

CRDTs provide **Strong Eventual Consistency**: any two nodes that have received the same set of updates will be in identical states, regardless of the order updates were received or applied. No coordination required.

### G-Counter (Grow-only counter)
Each node has its own slot. Value = sum of all slots. No node can decrement another node's slot.

```csharp
public class GCounter
{
    private readonly Dictionary<string, int> _counts = new();

    public void Increment(string nodeId) =>
        _counts[nodeId] = _counts.GetValueOrDefault(nodeId) + 1;

    public int Value() => _counts.Values.Sum();

    // Merge: take max of each slot — safe to call in any order
    public void Merge(GCounter other)
    {
        foreach (var (node, count) in other._counts)
            _counts[node] = Math.Max(_counts.GetValueOrDefault(node), count);
    }
}
```

### PN-Counter
Two G-Counters: one for increments (P), one for decrements (N). Value = P.Value() - N.Value().

### LWW-Register (Last Write Wins)
Each update carries a timestamp. On merge, the highest timestamp wins. Simple but requires loosely synchronized clocks and loses concurrent writes.

### OR-Set (Observed-Remove Set)
Each `add(element)` generates a unique tag. `remove(element)` removes all known tags for that element. If `add` and `remove` are concurrent, `add` wins — because the remove can only remove tags it has observed, not the new tag from the concurrent add.

**Where used:** Redis Enterprise (CRDT mode), Riak, Figma's multiplayer document state, Notion blocks, collaborative text editors (Peritext, Automerge). Anywhere you need offline-capable, conflict-free merging.

---

## Raft Consensus

Raft solves state machine replication: get a cluster of nodes to agree on an ordered sequence of log entries, so every node runs the same commands in the same order and arrives at the same state.

**Three roles:**
- **Leader** — handles all client writes. Exactly one leader per term.
- **Follower** — passive. Responds to Leader and Candidate RPCs.
- **Candidate** — temporarily during an election.

**Leader election:**
1. Follower receives no heartbeat within election timeout (150–300ms randomized).
2. Follower increments its term, becomes Candidate, votes for itself, sends `RequestVote` RPCs.
3. A node votes YES if: the candidate's term ≥ its own term AND the candidate's log is at least as up-to-date as its own.
4. Candidate wins if it gets votes from a **majority** (quorum) of nodes.
5. Randomized timeouts prevent split votes from cycling.

**Log replication:**
1. Client sends command to Leader.
2. Leader appends entry to its log (uncommitted).
3. Leader sends `AppendEntries` RPC to all Followers in parallel.
4. Once a **majority** acknowledge the entry, Leader commits it and applies it to its state machine.
5. Leader notifies Followers of commit in the next heartbeat; they commit too.

**Term numbers:** Monotonically increasing integers. Every RPC includes the sender's term. If a node receives a message with a higher term, it immediately reverts to Follower. This prevents stale leaders.

**Safety guarantee:** A committed entry (acknowledged by majority) will never be lost. Any new leader will have all committed entries because it must win a majority vote, and at least one voter in that majority holds the committed entry.

**Used in:** etcd (Kubernetes control plane), CockroachDB, Consul, TiKV (TiDB storage), InfluxDB Clustered.

---

## Gossip Protocol

Gossip (epidemic) protocols spread information through a cluster without central coordination. Each node periodically picks a random set of peers and exchanges state.

**How it works:**
1. Node A has new information (e.g., node D just joined).
2. A picks 3 random nodes and tells them.
3. Each of those nodes picks 3 more random nodes next round.
4. Information spreads exponentially — O(log N) rounds to reach all N nodes.

**Properties:**
- Eventually all nodes converge to the same information.
- Resistant to node failures — if a node is down, gossip routes around it.
- No single point of failure.
- Scales to thousands of nodes (Cassandra rings, Consul clusters).

**Failure detection:** Each node maintains a heartbeat counter per peer. If node A hasn't heard from node B (directly or via gossip) for N rounds, it marks B as *suspect*. If B remains suspect for another threshold, it's marked *dead* and the ring redistributes B's token ranges (in Cassandra) or removes B from the service registry (in Consul).

**Used in:** Cassandra (ring membership, schema changes), Consul/Serf (cluster membership), Amazon S3 (object metadata gossip internally), Bitcoin (block propagation).

---

## Consistent Hashing

**Problem with naive hashing:** With `key % N` (N nodes), adding or removing a node causes nearly all keys to remap. In a cache cluster, this means a thundering herd — every key misses and hits the database simultaneously.

**Consistent hashing solution:**
1. Map the hash space to a ring `[0, 2^32)`.
2. Place each node on the ring by hashing its identifier (IP:port or name).
3. Map each key to the ring by hashing it; assign it to the first node clockwise from that position.
4. When a node is added: only keys between the new node and its predecessor remap (~1/N keys).
5. When a node is removed: only its keys move to the next node clockwise.

**Virtual nodes:** A single physical node maps to multiple points on the ring (e.g., 150 virtual nodes per physical node). This:
- Smooths load distribution (avoids hot spots from uneven ring placement).
- Makes adding a new node incrementally drain load from many existing nodes rather than one.
- Allows heterogeneous nodes (more powerful nodes get more virtual nodes).

```
Ring:  0 ─── vNode-A1 ─── vNode-B1 ─── vNode-A2 ─── vNode-C1 ─── vNode-B2 ─── 2^32
Key X hashes to position between B1 and A2 → goes to A2 (next clockwise)
```

**Used in:** Amazon DynamoDB (partition key routing), Apache Cassandra (token-based ring), CDN cache routing (which edge node caches an asset), Redis Cluster (hash slots — a discrete variant), Memcached client-side sharding.

---

## Key Interview Questions

1. **"Can you build a CA distributed system?"** No — partition tolerance is mandatory in any real distributed system; CA is only possible on a single node.

2. **"Why don't microservices use 2PC?"** The coordinator is a single point of failure, participants hold locks across network round trips, and a coordinator crash can block the system indefinitely — Sagas solve this with local transactions and compensations.

3. **"How does Cassandra handle two nodes writing the same key simultaneously?"** Last-write-wins by timestamp at the column level; vector clocks (or client timestamps) determine the winner — which is why NTP clock sync matters in Cassandra clusters.

4. **"What's the difference between a choreography saga and an orchestration saga?"** Choreography: services react to each other's events with no central coordinator — decoupled but hard to track. Orchestration: a central state machine issues commands — visible and controllable but coupled to participants.

5. **"How does consistent hashing help when a cache node dies?"** Only the keys owned by the dead node remap to its successor (~1/N of total keys), so the cache warm-up impact is limited rather than remapping all keys as with modulo hashing.
