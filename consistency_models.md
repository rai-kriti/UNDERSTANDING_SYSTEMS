# Consistency Models — Complete Deep Dive
> Created by @kriti.sde | All content original.
> Part of the UNDERSTANDING-SYSTEMS series.

---

## Prerequisites

Before reading this — make sure you understand:
- CAP Theorem (what C, A, P mean)
- The difference between Availability and Consistency
- What a distributed system is (multiple nodes, data replicated)

If not — read the CAP Theorem README first.
`github.com/rai-kriti/UNDERSTANDING-SYSTEMS`

---

## What is a Consistency Model?

A consistency model is a **contract** between a distributed database and the developer.

It answers one question:

> "After I write data to this system — what is any user guaranteed to read back, from any node, at any point in time?"

Without this contract — you are guessing. And guessing in distributed systems leads to bugs that only appear under load, at scale, in production, at 3am on a Sunday.

---

## Why Consistency Models Exist

### The Single Node World

```
Single server. Single database.

User writes: balance = $500
User reads:  balance = $500

Same machine. Same memory.
Write and read always consistent.
No problem.
```

### The Distributed World

```
3 nodes. Data replicated across all 3.

Node A (US-East) ←──── replicate ────→ Node B (EU-West)
                                              │
                                        Node C (Asia)

User writes: balance = $500 → hits Node A
Replication to Node B takes: ~150ms (speed of light, US to EU)
Replication to Node C takes: ~200ms (speed of light, US to Asia)

Another user reads from Node B at 50ms after write:
What does Node B return?
$500 (new value) or $1,000 (old value)?

The consistency model answers this question.
```

### The Core Tension

```
You want: correct data + fast responses + always available

Reality: you can only optimize for some of these at once.

More consistent = more coordination between nodes
                = higher latency
                = lower availability

Less consistent = less coordination between nodes
               = lower latency
               = higher availability

The consistency model is where you define
exactly where on this spectrum your system sits.
```

---

## The Consistency Spectrum

```
STRONG ──────────────────────────────────────── WEAK
  │              │               │                │
Strong      Sequential       Eventual            Weak
Consistency  Consistency    Consistency       Consistency
  │              │               │                │
Always        Ordered        Eventually         No
correct       correct         correct          guarantee
  │              │               │                │
Slowest        Slow            Fast             Fastest
  │              │               │                │
Least         Low-Med          High              Most
available    available        available         available
  │              │               │                │
Banks        Financial       Social media       Gaming
Payments      reports         DNS, Caches        VoIP
```

No model is universally best. Each is right for a different problem.

---

## Model 1 — Strong Consistency

### Definition

Every read returns the **most recent write**. Always. From any node. Instantly.

No node ever returns stale data. No exceptions.

### Technical Term: Linearizability

```
Formal definition:
Every operation appears to execute instantaneously
at a single point in real time.

In simple terms:
The system behaves as if it has exactly ONE node —
even though it has many.

Every operation has a clear before/after relationship
that matches real-world time ordering.
```

### How It Works Technically

```
Step 1 — Write arrives:
User writes balance = $500 → Node A

Step 2 — Coordination begins:
Node A sends update to Node B and Node C
Node A WAITS for confirmation from both

Step 3 — Confirmation received:
Node B confirms: updated ✓
Node C confirms: updated ✓

Step 4 — Write acknowledged:
Node A tells user: write successful

Step 5 — Read now safe:
Any read from any node returns $500

Total time: write latency + replication time to ALL nodes
```

### What the User Experiences

```
Scenario: Online banking

User A withdraws $500 from ATM in New York.
Balance updates: $1,000 → $500

With strong consistency:
- ATM in London immediately shows $500
- ATM in Tokyo immediately shows $500
- Mobile app immediately shows $500
- No node has old balance

The system locked all nodes during the update.
No read was served until all nodes confirmed.
```

### What Happens During a Partition

```
Partition occurs between Node A and Node B.

Write arrives at Node A.
Node A tries to replicate to Node B.
Cannot reach Node B — partition.

Strong consistency response:
Return error: "503 Service Unavailable"
Refuse to acknowledge write
until partition heals and all nodes confirm.

User experience: Error message.
Data guarantee: Never returned incorrect data.
```

### Implementation Techniques

| Technique | How it works | Used in |
|---|---|---|
| Two-Phase Locking (2PL) | Lock all nodes before write, release after | PostgreSQL |
| Paxos | Consensus algorithm — majority must agree | Google Chubby |
| Raft | Leader-based consensus — simpler than Paxos | etcd, CockroachDB |
| Two-Phase Commit (2PC) | Coordinator asks all nodes to prepare, then commit | Distributed SQL |
| TrueTime (Google) | Atomic clocks + GPS for global time ordering | Google Spanner |

### Google Spanner — Strong Consistency at Global Scale

```
Challenge: 
Global database. Nodes in US, EU, Asia.
Must be strongly consistent.
But speed of light = minimum 150ms US-EU latency.

Problem:
How do you order operations globally
when clocks on different servers
are never perfectly synchronized?

Solution: TrueTime API

TrueTime uses:
- GPS receivers in every Google datacenter
- Atomic clocks in every Google datacenter
- Returns time as an interval [earliest, latest]
  rather than a single value
- Uncertainty: 1-7ms

How writes work:
1. Write arrives
2. Spanner assigns timestamp T
3. Spanner WAITS 7ms (uncertainty window)
4. Any write that could conflict is ordered
5. Write committed with guaranteed global order

Result:
Globally consistent reads and writes
across US, EU, Asia simultaneously.
Every node agrees on the order of every operation.

Used for: Google Ads billing, Google Cloud SQL
Cost: ~7ms added latency per write (the wait)
Worth it: Wrong billing data = direct revenue loss
```

### Real Databases Using Strong Consistency

| Database | Type | How it achieves strong consistency | Use case |
|---|---|---|---|
| PostgreSQL | Relational SQL | ACID transactions, 2PL | Banking, ERP, e-commerce |
| MySQL (sync replication) | Relational SQL | Synchronous master-slave | Financial systems |
| Google Spanner | Distributed SQL | TrueTime, Paxos | Global financial data |
| CockroachDB | Distributed SQL | Raft consensus | Multi-region banking |
| Zookeeper | Coordination service | ZAB protocol (Paxos variant) | Config management |
| etcd | Key-value store | Raft consensus | Kubernetes config |
| HBase | Column-family | Single master for writes | Consistent analytics |

### When to Use Strong Consistency

Use strong consistency when **wrong data causes real damage:**

| Use Case | Why Strong Consistency |
|---|---|
| Bank balance | Stale balance = overdraft undetected |
| Payment processing | Duplicate charge = customer loses money |
| Inventory (last item) | Overselling = unfulfillable order |
| Seat/hotel booking | Double booking = legal and operational nightmare |
| Stock trading | Stale price = wrong trade, financial loss |
| Authentication tokens | Stale token = security vulnerability |
| Medical records | Wrong dosage = patient safety risk |
| Distributed locks | Stale lock state = multiple processes in critical section |

### Cost of Strong Consistency

```
1. Highest latency
   Every write waits for ALL nodes to confirm.
   Write latency = slowest node's confirmation time.

2. Lowest throughput
   Coordination overhead limits concurrent writes.

3. Reduced availability
   Returns error during partition.
   Users see error instead of stale data.

4. Complexity
   Implementing distributed consensus is hard.
   Bugs in consensus = data corruption.
```

---

## Model 2 — Sequential Consistency

### Definition

All operations appear to execute in **some sequential order** that all nodes agree on. Each node's operations appear **in the order they were issued** — but that order may not match real-world clock time.

### The Key Difference from Strong Consistency

```
Strong Consistency:
Operations reflect REAL-TIME global order.
If A happened before B in wall-clock time →
every node sees A before B.
Matches the actual clock.

Sequential Consistency:
Operations are in a consistent order —
but not necessarily wall-clock order.
All nodes see THE SAME order
but it may not match the actual clock.

Example:
Thread 1: write X=1 at time 10:00:00.000
Thread 2: write X=2 at time 10:00:00.001

Strong: every node sees X=1 then X=2 (clock order)
Sequential: every node sees the SAME order
            but might see X=2 then X=1
            if that order was decided by the system
```

### How It Works Technically

```
Each node maintains a local operation log.
Operations within one node are always in order.
A global sequencer orders operations across nodes.
But the sequencer does not need to match wall-clock time.

Thread 1 operations: [write X=1, write X=3, read Y]
Thread 2 operations: [write Y=2, read X]

Sequential consistency guarantees:
- Thread 1's ops appear in order: X=1, X=3, read Y
- Thread 2's ops appear in order: Y=2, read X
- All threads agree on the interleaving
- But the interleaving may not match clock time
```

### What the User Experiences

```
Scenario: Collaborative document editing

User A types "Hello" then "World" in a doc.
User B reads the doc.

Sequential consistency guarantees:
User B never sees "World" before "Hello"
(User A's operations are ordered)

But User B might see:
- Neither word (stale)
- "Hello" only
- "Hello World"

Never: "World" without "Hello"
The per-thread order is always preserved.
```

### Real World Usage

```
Where sequential consistency appears:

1. Multi-core CPU memory models
   Each CPU core sees memory operations
   in a consistent sequential order.
   Not always real-time global order.
   Reason: cache coherence protocols

2. Some distributed databases
   in relaxed transaction modes

3. Message queues
   Messages from one producer
   always consumed in order.
   But ordering between producers
   not guaranteed in real-time.

4. Version control systems
   Git commits are sequential per branch.
   Merge order may not match clock time.
```

### When to Use Sequential Consistency

```
Use when:
- Operations from the same source must be ordered
- Real-time global ordering is not critical
- Lower latency than strong consistency is needed
- Multi-threaded systems where per-thread order matters

Examples:
- Message queues (Kafka partitions)
- Multi-core CPU operations
- Collaborative tools where per-user order matters
- Log aggregation systems
```

---

## Model 3 — Eventual Consistency

### Definition

All nodes will **eventually** have the same data. Reads may return stale data temporarily. But given enough time without new writes — all nodes converge to the same value.

The most widely used consistency model in modern distributed systems.

### Technical Definition

```
Formal definition:
If no new updates are made to a data item —
eventually all accesses to that item
will return the last updated value.

Key word: eventually.
No specific time bound is guaranteed.
Could be milliseconds.
Could be seconds.
Depends on network conditions and system load.
```

### How It Works Technically

```
Step 1 — Write arrives:
User writes balance = $500 → Node A
Node A updates immediately: balance = $500
Node A returns success to user immediately ✓

Step 2 — Async replication:
Node A sends replication message to Node B and C
Does NOT wait for confirmation
Continues serving other requests

Step 3 — Replication completes:
Node B receives replication: updates to $500
Node C receives replication: updates to $500

Step 4 — All nodes converged:
Any read from any node now returns $500

Time between Step 1 and Step 4:
Could be 10ms. Could be 2 seconds.
During this window: Node B and C may return old value.
```

### Conflict Resolution — The Hard Part

```
What happens when two nodes get conflicting writes?

Scenario:
User A writes X=1 → Node A
User B writes X=2 → Node B
(simultaneously, during partition)

After partition heals:
Node A has X=1
Node B has X=2
Which one wins?

Conflict resolution strategies:

1. Last Write Wins (LWW)
   Whichever write has the later timestamp wins.
   Problem: Clock skew between nodes.
   Cassandra uses this by default.

2. Vector Clocks
   Each write carries a version vector.
   System tracks causal relationships.
   Can detect true conflicts.
   Amazon DynamoDB uses vector clocks.

3. CRDTs (Conflict-free Replicated Data Types)
   Data structures designed to merge automatically.
   No conflicts by design.
   Example: Counter that only increments
            can merge by taking max value.
   Used in: Redis, Riak, collaborative editors.

4. Application-level resolution
   Database detects conflict, returns both values.
   Application decides which to keep.
   Amazon shopping cart uses this.
```

### The Shopping Cart Problem — Amazon's Solution

```
Amazon shopping cart design (Werner Vogels, 2007):

Problem:
User adds item to cart on mobile (Node A).
User removes item on laptop (Node B).
Partition occurs.

With Last Write Wins:
One operation silently overwrites the other.
User loses items without knowing.
Terrible user experience.

Amazon's solution: Eventual consistency + union merge
Cart = set of additions + set of deletions
On conflict: take union of both sets

Result:
Cart may have extra items temporarily.
Users can always remove unwanted items.
Users never silently lose items.

"Surprising a customer by
having their cart be smaller than expected
is worse than surprising them by
having it be larger than expected."
— Amazon engineering blog
```

### Read Repair — How Eventual Systems Self-Heal

```
Read Repair is a technique to speed up convergence.

When a read request arrives:
1. Query multiple nodes
2. Compare their responses
3. If responses differ — stale node detected
4. Repair the stale node in the background
5. Return latest value to user

Example in Cassandra:
Read from Node A: balance = $500 (latest)
Read from Node B: balance = $1,000 (stale)

Read repair:
Return $500 to user.
Async update Node B to $500.
Next read from Node B: $500 (repaired).

Result: Convergence happens faster
        during normal read traffic.
```

### Real Databases Using Eventual Consistency

| Database | Replication type | Conflict resolution | Use case |
|---|---|---|---|
| Cassandra | Async, leaderless | Last Write Wins, tunable | Social feeds, IoT, messaging |
| DynamoDB | Async, multi-region | Vector clocks, app-level | Shopping cart, gaming, ad tech |
| CouchDB | Multi-master async | CRDTs, app-level | Mobile sync, offline-first |
| Riak | Leaderless, async | CRDTs, Last Write Wins | Session storage, user profiles |
| Redis (async replication) | Master-replica async | Last Write Wins | Cache, leaderboards |
| DNS | TTL-based propagation | Last write propagates | Domain resolution |
| MongoDB (default) | Async replica set | Primary wins | General purpose |

### DNS — Eventual Consistency at Internet Scale

```
DNS is the largest eventually consistent system ever built.

How it works:
1. You update DNS record: mysite.com → new IP
2. Change hits your authoritative DNS server
3. Propagation begins to 13 root DNS servers worldwide
4. Each DNS resolver has cached old record
5. Cache expires based on TTL (Time To Live)
   Common TTL: 3600 seconds (1 hour)

During propagation (up to 48 hours):
- Some users get old IP (stale)
- Some users get new IP (latest)
- Different users around world see different versions

After TTL expires everywhere:
- All DNS resolvers have new IP
- All users reach new server
- Eventual consistency achieved

Cost: Up to 48 hours of inconsistency
Benefit: Entire internet never goes down for DNS updates
         Zero coordination needed
         Infinitely scalable
```

### Tunable Consistency in Cassandra

```
Cassandra allows you to choose consistency per query.
This is called Tunable Consistency.

Consistency levels:
ONE    → Read/write from 1 replica  → fastest, least consistent
TWO    → Read/write from 2 replicas → medium
QUORUM → Read/write from majority   → balanced, most common
ALL    → Read/write from all        → slowest, most consistent
LOCAL_QUORUM → Majority in local DC → good for multi-region

// Eventual consistency read (fast, may be stale)
SELECT * FROM social_feed WHERE user_id = ?
WITH CONSISTENCY ONE;

// Strong-ish consistency read (slower, less stale)
SELECT * FROM inventory WHERE product_id = ?
WITH CONSISTENCY QUORUM;

// Maximum consistency (slowest, never stale)
SELECT * FROM payments WHERE transaction_id = ?
WITH CONSISTENCY ALL;

Same database. Different consistency per query.
Use the right level for each data type.
```

### When to Use Eventual Consistency

Use eventual consistency when **brief staleness causes no real harm:**

| Use Case | Why Eventual Consistency |
|---|---|
| Social media feed | Seeing post 2 seconds late = fine |
| Like and follower counts | Approximate count = acceptable |
| Shopping cart | Slight sync delay across devices = fine |
| DNS records | Propagation over hours = acceptable |
| Product recommendations | Slightly stale = fine |
| User profiles | 1 second delay = acceptable |
| Search indexes | New content indexed after delay = fine |
| Analytics dashboards | Approximate real-time = acceptable |
| Notification delivery | Slight delay = acceptable |
| Message read receipts | 1 second delay = fine |

---

## Model 4 — Weak Consistency

### Definition

After a write — reads **may or may not** see it. No guarantee of when. No guarantee of if. The system makes no promise about what you will read back after a write.

### Technical Definition

```
Formal definition:
After a write to data item X —
subsequent reads of X are not guaranteed
to return the written value.

The system does not attempt to propagate
the write to other nodes immediately
or guarantee when or if it will arrive.
```

### Why Would Anyone Use This?

```
Sounds terrible. Here is why it exists:

Zero coordination overhead.
Maximum throughput.
Maximum availability.
Minimum latency.

Used ONLY when:
Missing or stale data causes absolutely no harm.
The use case naturally tolerates data loss.
Speed and availability are the ONLY priorities.
```

### Real World Use Cases — Explained

#### Live Video Streaming

```
Twitch / YouTube Live / Netflix Live

Each video frame is a write.
30 frames per second = 30 writes per second.
1080p frame = ~2MB of data.

With strong consistency:
All nodes confirm each frame before serving next.
30 frames/sec × confirmation latency = impossible.
Video would be unwatchable.

With weak consistency:
Frame sent to nearest CDN node.
Served immediately.
If packet drops → frame is gone.
Not retransmitted.
Next frame continues.

Why acceptable:
Human eye cannot distinguish 1 missing frame.
Video keeps playing.
Retransmitting old frame would cause worse experience.
(You don't want to see old frame after new one)
```

#### Multiplayer Gaming

```
Fortnite / Call of Duty / League of Legends

Player position updates: ~60 times per second.
100 players in one game = 6,000 position updates/sec.

With strong consistency:
All 100 players must confirm your position.
Round trip to all players globally: 200ms+.
Game would be unplayable.

With weak consistency:
Your position sent to server.
Server broadcasts to nearby players.
No confirmation waited.
If update lost → next update covers it.

Client-side interpolation:
Your client predicts positions between updates.
Fills gaps where updates are missing or late.
You see smooth movement even with weak consistency.

Why acceptable:
Game positions are approximate by nature.
50ms position lag is normal and expected.
Exact real-time position of all players
is physically impossible at global scale.
```

#### VoIP and Real-Time Communication

```
WhatsApp calls / Zoom audio / Google Meet

Audio packets: 50 packets per second.
Each packet = 20ms of audio.

With strong consistency:
Dropped packet → wait for retransmission.
Retransmission adds 200ms+ delay.
Voice becomes echo-y and confusing.
Conversation impossible.

With weak consistency (UDP protocol):
Dropped packet → skip it.
Next packet continues.
Brief silence (20ms) = barely noticeable.
Conversation stays real-time.

Why UDP not TCP for VoIP:
TCP guarantees delivery (eventual consistency).
UDP does not guarantee delivery (weak consistency).
For real-time audio: UDP is correct choice.
Silence < delay. Every time.
```

#### Real-Time Analytics

```
Google Analytics / Mixpanel / Amplitude

Counting page views, active users, events.

With strong consistency:
Every counter increment requires global lock.
1 million simultaneous page views.
All waiting for lock.
Counter update system collapses.

With weak consistency:
Each node increments its local counter.
Counters merged periodically (every 10 seconds).
Dashboard shows approximate count.

Result:
Dashboard shows 10,042 active users.
Real number might be 10,039 or 10,047.
Difference of 3-8 users → irrelevant.
Real-time approximate beats delayed exact.
```

### Implementation Techniques

```
1. Fire and forget
   Write sent to one node.
   No confirmation waited.
   No replication guaranteed.

2. UDP-based communication
   User Datagram Protocol.
   No delivery guarantee.
   No ordering guarantee.
   Maximum speed.

3. In-memory only storage
   Data stored in RAM.
   Not replicated.
   Not persisted.
   Lost on node restart.
   Acceptable for ephemeral data.

4. Approximate data structures
   HyperLogLog → approximate cardinality counting
   Count-Min Sketch → approximate frequency counting
   Bloom filters → approximate set membership
   Used when exact counts are unnecessary.
```

### When to Use Weak Consistency

| Use Case | Why Weak Consistency Works |
|---|---|
| Live video frames | Missing frame → next frame, no harm |
| VoIP audio packets | Missing packet → brief silence, acceptable |
| Game player positions | Approximate position + interpolation = fine |
| Real-time analytics counters | Approximate count = acceptable |
| Sensor data streams | Missing one reading = fine, trend still visible |
| Live sports scores | 1 second delay = acceptable |
| Ad impression counting | Approximate count within margin = fine |
| CDN cache warming | Stale cache for brief period = acceptable |

---

## Side by Side Comparison

| Dimension | Strong | Sequential | Eventual | Weak |
|---|---|---|---|---|
| Latest write returned | Always ✓ | Ordered ✓ | Eventually ✓ | No guarantee |
| Stale data possible | Never | Briefly | Yes, temporarily | Always |
| Write latency | Highest | High | Low | Lowest |
| Read latency | Highest | High | Low | Lowest |
| Availability | Lowest | Low-Medium | High | Highest |
| Partition behavior | Returns error | Returns error sometimes | Returns stale data | Returns stale data |
| Conflict resolution | No conflicts possible | Ordered, no conflicts | LWW, CRDTs, app-level | Not applicable |
| Real databases | PostgreSQL, Spanner | CPU memory models | Cassandra, DynamoDB | Game servers, VoIP |
| Use case | Banking, payments | Multi-core, ordered logs | Social, DNS, cache | Streaming, gaming |
| Technical term | Linearizability | Sequential ordering | Eventual convergence | Best effort |

---

## How Real Systems Mix Consistency Models

The biggest insight about consistency models:

**You do not choose ONE model for your entire system.**

You choose the right model for each **data type** in your system.

### Instagram — Multiple Models in One System

```
Instagram uses different consistency models
for different types of data:

Authentication tokens → Strong Consistency (PostgreSQL)
Wrong auth = security breach. Never acceptable.

Payment data → Strong Consistency (PostgreSQL)
Wrong charge = financial loss. Never acceptable.

User profile (name, bio) → Eventual Consistency (Cassandra)
Profile showing 1 second old = fine.

Photo feed → Eventual Consistency (Cassandra)
Seeing photo 1 second late = fine.

Like counts → Eventual Consistency (Redis async)
Showing 10,492 vs 10,491 = nobody cares.

Live video frames → Weak Consistency (CDN, UDP)
Missing frame = not noticeable.

Story view counts → Eventual Consistency
Approximate real-time count = acceptable.
```

### The Decision Per Data Type

```
For every piece of data in your system, ask:

1. "If this data is wrong for 2 seconds —
    what is the worst that happens?"

   Catastrophic → Strong Consistency
   Significant → Sequential or Strong
   Minor → Eventual Consistency
   Irrelevant → Weak Consistency

2. "What is more damaging —
    wrong data or no response?"

   Wrong data more damaging → CP → Strong/Sequential
   No response more damaging → AP → Eventual/Weak
```

---

## Common Mistakes Engineers Make

### Mistake 1 — Strong Consistency Everywhere

```
"PostgreSQL for everything. Safety first."

What happens at scale:
Social feed queries hitting PostgreSQL.
1 million concurrent reads.
Every read acquiring shared locks.
Lock contention increases.
Queries queue up.
Response time: 200ms → 2 seconds → timeout.
System collapses under read load.

Fix:
Use eventual consistency (Cassandra/Redis)
for read-heavy, tolerance-for-staleness data.
Reserve strong consistency for critical data only.
```

### Mistake 2 — Eventual Consistency Everywhere

```
"Cassandra scales better. Use it for everything."

What happens with payment data:
User clicks Pay Now.
Write hits Node A → success returned.
User immediately redirected → reads from Node B.
Node B has not replicated yet.
Shows: "No recent payment found."
User clicks Pay Now again.
Duplicate charge created.

Fix:
Use strong consistency (PostgreSQL with transactions)
for payment processing.
Implement idempotency keys for retry safety.
```

### Mistake 3 — Not Knowing Your Database Default

```
MongoDB consistency history:
- Before 3.2: eventual consistency by default
- After 3.2: strong consistency by default for reads
             from primary

Engineers who upgraded without checking:
Read behavior changed silently.
Performance degraded unexpectedly.
(Stronger consistency = more coordination = slower reads)

Always check:
- What is the default consistency level of your DB?
- Did it change between versions?
- Does your application rely on the default?
```

### Mistake 4 — Ignoring Replication Lag

```
Common pattern (broken):
Write to master PostgreSQL → success
Immediately redirect user → read from replica
User sees old data → reports bug
Developer cannot reproduce (single node test)
Bug only appears under load in production

Fix options:
1. Read-your-own-writes consistency:
   Route write + immediate read to same node (master)
   Route all other reads to replicas

2. Session consistency:
   After a write, route all reads from that
   session to master for 1 second
   Then fall back to replica reads

3. Accept the lag:
   Design UI to show "Processing..." after write
   Give replication time to propagate
   Then show result from replica
```

---

## Quick Reference

### Choosing a Consistency Model

```
Data type → Question → Model → Database

Financial data
"Wrong = money lost"
→ Strong Consistency
→ PostgreSQL, Spanner, CockroachDB

Configuration data
"Wrong = system misconfigured"
→ Strong Consistency
→ Zookeeper, etcd

Social feed
"Stale for 1 second = fine"
→ Eventual Consistency
→ Cassandra, DynamoDB

Like/follower counts
"Approximate = fine"
→ Eventual Consistency
→ Redis (async), Cassandra

Live video
"Missing frame = fine"
→ Weak Consistency
→ CDN, UDP streaming

Real-time gaming
"Approximate position = fine"
→ Weak Consistency
→ Custom UDP game server

DNS records
"Propagation delay = fine"
→ Eventual Consistency
→ DNS infrastructure
```

### The 5 Numbers to Know

```
1. DNS propagation time:    up to 48 hours (eventual)
2. Cassandra replication:   typically < 100ms (eventual)
3. Google Spanner wait:     7ms per write (TrueTime uncertainty)
4. PostgreSQL 2PL overhead: 1-5ms per write (strong)
5. UDP packet loss rate:    0.1-5% on internet (weak = acceptable)
```

---

## What Comes Next

```
You now understand:
✓ What consistency models are
✓ The full spectrum: Strong → Sequential → Eventual → Weak
✓ How each model works technically
✓ Which databases use which model
✓ How to choose per data type
✓ Common mistakes to avoid

Next → Fail-over vs Replication
How distributed systems stay alive
when nodes crash, networks fail,
and everything goes wrong.

The mechanisms that give you
those 99.99% availability numbers.
```

---

*All content original. Created by @kriti.sde*
*github.com/rai-kriti/UNDERSTANDING-SYSTEMS*
