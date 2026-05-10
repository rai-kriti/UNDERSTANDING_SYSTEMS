# Consistency Models — Complete Deep Dive
> Created by @kriti.sde | All content original.
> Part of the UNDERSTANDING-SYSTEMS series.

---

## What is a Consistency Model?

A consistency model is a **contract** between a distributed database and the developer.

It answers one fundamental question:

> "After I write data to the system — what is any user guaranteed to read back? From which node? At what time?"

Without understanding consistency models — you cannot reason about what your users will see when your system is under load, experiencing failures, or replicating data across multiple nodes.

---

## Why Consistency Models Exist

### The Single Node World

```
Single server. Single database.
User writes X = 500.
User reads X.
Returns 500. Always.
No consistency problem.
No distributed system.
```

In a single node system — consistency is automatic. There is only one copy of the data. Every read sees the latest write.

### The Distributed World

```
Node A (US-East) ←── 150ms ──→ Node B (EU-West)

User writes X = 500 → Node A
Node A updates immediately.
Node A starts replicating to Node B.
Replication takes 150ms.

During those 150ms:
Another user reads X from Node B.
What does Node B return?

→ 500 (latest write) ?
→ 300 (old value) ?
→ Error ?

The answer depends on the consistency model.
```

The moment you have more than one node — you have a distributed system. The moment you have a distributed system — you need a consistency model.

---

## The Core Tension

```
STRONG CONSISTENCY                    WEAK CONSISTENCY
       │                                     │
Always correct data                  Always fast response
       │                                     │
Requires coordination                Zero coordination
between all nodes                    between nodes
       │                                     │
Slower                               Faster
       │                                     │
Less available                       More available
       │                                     │
Returns error during                 Returns stale data
network partition                    during partition
```

You cannot have both ends simultaneously.
Every consistency model is a trade-off between correctness and speed.

---

## The Four Consistency Models

```
STRONGEST ─────────────────────────────────── WEAKEST
     │              │              │               │
   Strong       Sequential      Eventual          Weak
Consistency    Consistency    Consistency      Consistency
     │              │              │               │
  Always          Ordered       Eventually        No
  correct         correct        correct        guarantee
     │              │              │               │
  Slowest          Slow           Fast           Fastest
     │              │              │               │
 Banks, payments  Multi-core   Social feeds,    Live video,
 bookings         CPU, some    DNS, caches      gaming, VoIP
                  strict DBs
```

---

## Model 1 — Strong Consistency

### Definition

Every read returns the **most recent write** — always — from any node — instantly.

No node ever returns stale data. The system behaves as if it has exactly one node even though it has many.

### Technical Term — Linearizability

```
Linearizability means:
Every operation appears to execute instantaneously
at a single point in real time.

If write W completes before read R begins —
R must return the value written by W.
No exceptions. No stale reads.
```

### How Strong Consistency Works Internally

```
Step 1 — Write request arrives:
User writes balance = $500 → Node A

Step 2 — System acquires distributed lock:
Node A sends lock request to Node B, Node C
All nodes acknowledge lock
No reads served during this period

Step 3 — All nodes update:
Node A updates → $500
Node B updates → $500
Node C updates → $500
All confirmed.

Step 4 — Lock released:
Reads now served from any node
Every node returns $500

Total time: Write latency + Replication latency + Lock overhead
```

### What the User Experiences

```
User A withdraws $500 from ATM in New York.
Write completes. Balance = $500.

User A checks balance on mobile app
routed to Node B in EU.

With strong consistency:
Node B returns $500 immediately.
Correct. Always.

With strong consistency during partition:
Node B cannot sync with Node A.
Node B returns: "Service temporarily unavailable."
User must retry.
Never returns wrong balance.
```

### Technical Implementations

| Algorithm | How it works | Used in |
|---|---|---|
| Two-Phase Locking (2PL) | Acquire all locks before write, release after | PostgreSQL, MySQL |
| Paxos | Consensus protocol, majority quorum | Google Chubby, Zookeeper |
| Raft | Leader-based consensus, easier to understand | etcd, CockroachDB |
| Two-Phase Commit (2PC) | Coordinator asks all nodes to prepare then commit | Distributed transactions |
| TrueTime (Google) | Atomic clocks + GPS for timestamp certainty | Google Spanner |

### Real World Systems Using Strong Consistency

| System | Why Strong Consistency | What Happens During Partition |
|---|---|---|
| PostgreSQL | ACID transactions for financial data | Returns error, rolls back transaction |
| Google Spanner | Global financial data, ads billing | Returns error, never serves stale data |
| Zookeeper | Distributed config, leader election | Returns error, waits for quorum |
| etcd | Kubernetes config store | Returns error, requires quorum |
| Apache HBase | Hadoop analytics requiring correctness | Returns error from non-leader nodes |

### Google Spanner — Strong Consistency at Global Scale

```
Challenge: 
Global database spanning multiple continents.
Strong consistency required for financial data.
Physics problem: US to EU = minimum 150ms round trip.

Solution: TrueTime API

TrueTime uses:
→ GPS receivers in every Google datacenter
→ Atomic clocks in every Google datacenter
→ Uncertainty bounds on every timestamp (1-7ms)

How it achieves global consistency:
Every write gets a TrueTime timestamp.
System waits out the uncertainty window.
Guarantees: if write W has timestamp T,
any read after T returns W.

Result:
Globally consistent transactions.
Used for: Google Ads billing (billions of dollars).
During partition: returns error, never stale data.

Cost: Higher latency due to uncertainty wait.
Worth it: Wrong ad billing = direct revenue loss.
```

### When to Use Strong Consistency

Use strong consistency when **wrong data causes real damage:**

| Use Case | Why Strong | Risk of Eventual |
|---|---|---|
| Bank balance | Wrong balance = overdraft undetected | Money lost |
| Payment processing | Duplicate payment risk | Customer charged twice |
| Inventory (last item) | Overselling risk | Order cannot be fulfilled |
| Seat/hotel booking | Double booking risk | Legal and operational problem |
| Stock trading | Stale price = wrong trade | Financial loss |
| Authentication tokens | Stale token = security breach | Unauthorized access |
| Medical dosage records | Wrong dosage shown | Patient safety risk |

### Cost of Strong Consistency

```
Latency: Highest of all models
         Write must wait for all nodes to confirm

Throughput: Lowest of all models
            Distributed locks reduce concurrency

Availability: Lowest of all models
              Returns error during partition
              rather than serve stale data

Scalability: Hardest to scale
             More nodes = more coordination overhead
             Amdahl's Law limits parallelism
```

---

## Model 2 — Sequential Consistency

### Definition

All operations appear to execute in **some sequential order.** Each node's operations appear in the order they were issued — but that order may not match real-time global clock order.

### The Key Difference from Strong Consistency

```
STRONG CONSISTENCY:
Operations reflect real wall-clock time order.
If A happened at 10:00:01.000
and B happened at 10:00:01.001 —
every node sees A before B.
Timestamps are meaningful globally.

SEQUENTIAL CONSISTENCY:
Operations are in a consistent order —
but not necessarily wall-clock order.
All nodes see the SAME sequence —
but that sequence may not match
the real-time order of events.
Timestamps may not be globally meaningful.
```

### Example to Understand the Difference

```
Real-time order of events:
10:00:00.001 → Thread 1 writes X = 1
10:00:00.002 → Thread 1 writes X = 2
10:00:00.003 → Thread 2 reads X

STRONG CONSISTENCY result:
Thread 2 must read X = 2
(the most recent write at that timestamp)

SEQUENTIAL CONSISTENCY result:
Thread 2 might read X = 1 or X = 2
Both are valid as long as:
→ Thread 2 never sees X = 2 before X = 1
→ The order within Thread 1 is preserved
→ All threads agree on the same sequence
```

### Technical Implementation

```
Sequential consistency is implemented through:

Memory barriers / fences:
→ Prevent CPU from reordering instructions
→ Ensure writes are visible in order

Message ordering protocols:
→ FIFO ordering between node pairs
→ Causal ordering (happens-before relationship)

Vector clocks:
→ Track causal relationships between events
→ Ensure causally related operations are ordered
```

### Where Sequential Consistency Appears

| System | Context | Why Sequential |
|---|---|---|
| Multi-core CPU | Shared memory between threads | Hardware implements memory model |
| Java Memory Model | Thread communication | JVM guarantees sequential per thread |
| x86 processors | Store-load ordering | TSO (Total Store Order) memory model |
| Some distributed DBs | Relaxed transaction mode | Balance between strong and eventual |

### Real World Example — Multi-Core CPU

```
CPU has 4 cores. Each core has its own cache.

Core 1 writes X = 1 to its cache.
Core 1 writes X = 2 to its cache.
Core 2 reads X from shared memory.

Sequential consistency guarantees:
Core 2 never sees X = 2 before X = 1.
The write order within Core 1 is preserved.

But Core 2 might still see X = 1
even after Core 1 wrote X = 2 —
because cache synchronization takes time.

This is why concurrent programming is hard.
This is why volatile, synchronized,
and memory barriers exist in Java/C++.
```

### When to Use Sequential Consistency

```
Use when:
→ Operation ordering matters
→ Real-time global ordering not required
→ Stronger than eventual but weaker than strong needed
→ Multi-threaded programming guarantees

Common in:
→ CPU memory models
→ Multi-threaded application design
→ Some distributed databases in relaxed mode
→ Message ordering systems
```

---

## Model 3 — Eventual Consistency

### Definition

All nodes will **eventually** have the same data — given enough time and no new writes.

Reads may return stale data temporarily. But eventually all nodes converge to the same value.

### Technical Definition

```
Eventual consistency guarantees:

If no new updates are made to a data item —
eventually all accesses to that item
will return the last updated value.

Key word: eventually.
No time bound specified.
Could be milliseconds.
Could be seconds.
Depends on network conditions and replication lag.
```

### How Eventual Consistency Works

```
Step 1 — Write arrives:
User writes likes = 1,042 → Node A (US-East)
Node A updates immediately.
Node A responds to user: "success"

Step 2 — Async replication begins:
Node A sends replication to Node B (EU-West)
Node A sends replication to Node C (Asia)
Replication happens in background.
User does not wait for it.

Step 3 — During replication:
Read from Node B → returns 1,041 (stale) ← briefly inconsistent
Read from Node A → returns 1,042 (correct)

Step 4 — Replication completes (~150ms later):
Node B updated → 1,042
Node C updated → 1,042
All nodes now consistent.

Read from Node B → returns 1,042 ← eventually consistent
```

### Conflict Resolution — What Happens When Two Nodes Get Different Writes

```
Problem scenario:
User A updates profile picture → Node A
User B updates same user's profile picture
(admin override) → Node B simultaneously

Both nodes accept the write.
Network between A and B is partitioned.
After partition heals — both nodes have different values.

Which one wins?

Resolution strategies:

1. Last Write Wins (LWW):
   Compare timestamps. Latest timestamp wins.
   Simple. Risk: clock skew can cause wrong winner.
   Used by: Cassandra (default)

2. Vector Clocks:
   Track causal history of writes.
   Detect true conflicts vs causal ordering.
   Used by: Amazon DynamoDB, Riak

3. CRDTs (Conflict-free Replicated Data Types):
   Design data structures that merge automatically.
   No conflicts possible by design.
   Example: A counter that only increments
            can be merged by adding values.
   Used by: Redis, some Cassandra data types

4. Application-level resolution:
   Expose conflict to application.
   Application decides which version wins.
   Used by: CouchDB (returns all versions)
```

### Replication Lag — The Numbers

```
Replication lag = time between write on primary
                  and availability on replica

Typical replication lag values:

Same datacenter:      1-5ms
Cross-region (async): 50-200ms
Cross-continent:      100-500ms
During high load:     500ms-5 seconds
During partition:     Until partition heals

This is the window during which
stale data can be returned.

For most applications — this is acceptable.
For financial applications — this is not.
```

### Real World Systems Using Eventual Consistency

| System | Data Type | Why Eventual | Stale Duration |
|---|---|---|---|
| Cassandra (Instagram) | Social feed posts | Seeing post 1s late = fine | Milliseconds |
| DynamoDB (Amazon) | Shopping cart | Cart sync delay = fine | Milliseconds |
| DNS | Domain IP records | Cached IP until TTL = fine | Minutes to hours |
| Cassandra (Netflix) | Viewing history | History 1s stale = fine | Milliseconds |
| Redis (async) | Session cache | Slight staleness = fine | Milliseconds |
| CouchDB | Mobile app sync | Offline sync = fine | Minutes |
| Cassandra (Uber) | Ride history | History 1s stale = fine | Milliseconds |

### DNS — Eventual Consistency Since 1983

```
DNS is the oldest and most widely used
eventually consistent system on the internet.

How it works:
You update your domain's IP address.
Root DNS servers updated immediately.
Authoritative DNS servers updated: minutes
ISP DNS caches: up to TTL value (hours)
Browser DNS cache: up to TTL value (minutes)

Result:
Some users see new IP immediately.
Some users see old IP for hours.
Everyone sees new IP eventually.

TTL (Time To Live) = the consistency window.
Lower TTL = faster convergence = more DNS load.
Higher TTL = slower convergence = less DNS load.

Every website on the internet
accepts this eventual consistency.
The alternative (strong consistency for DNS)
would make the internet unbearably slow.
```

### When to Use Eventual Consistency

Use eventual consistency when **brief staleness causes no real harm:**

| Use Case | Acceptable Stale Duration | Why |
|---|---|---|
| Social media feed | 1-5 seconds | Seeing post slightly late = fine |
| Like/follower counts | Seconds | Approximate count = fine |
| Product recommendations | Minutes | Slightly stale = fine |
| Shopping cart | Seconds | Sync delay across devices = fine |
| DNS records | Minutes to hours | Cached IP until TTL = fine |
| User profiles | Seconds | Name change appears slightly late = fine |
| Analytics dashboards | Minutes | Approximate metrics = fine |
| Notification delivery | Seconds | Slight delay = fine |

---

## Model 4 — Weak Consistency

### Definition

After a write — reads **may or may not** see it. No guarantee. No timeline. No promise.

The system makes zero commitment about what any read will return after a write.

### Technical Definition

```
Weak consistency:
After a write to data item X —
subsequent reads of X are not guaranteed
to return the written value.

There is no guarantee of:
→ When the value becomes visible
→ Whether it ever becomes visible
→ Which nodes will ever see it

The system optimizes purely for:
→ Maximum throughput
→ Minimum latency
→ Maximum availability
```

### Why Weak Consistency Exists

```
Strong consistency cost:
→ Distributed coordination overhead
→ Network round trips for confirmation
→ Locks and quorums
→ Added latency on every operation

For some use cases:
This cost is completely unnecessary.
The data does not need to be consistent.
Missing it or seeing old value = zero harm.

Weak consistency eliminates:
→ All coordination overhead
→ All replication waiting
→ All conflict resolution

Result:
Fastest possible reads and writes.
Maximum possible availability.
Zero consistency guarantees.
```

### Real World Examples

#### Live Video Streaming

```
Twitch / YouTube Live:

Video encoded into frames → 30 or 60 per second.
Frames sent over network.
Network drops a packet.
Frame is lost.

Weak consistency behavior:
Lost frame is NOT retransmitted.
Viewer sees brief glitch or frozen frame.
Video continues from current frame.

Why not retransmit?
Retransmitting takes time.
By the time it arrives — 10 more frames have passed.
Viewer would see video stutter and jump backward.
That is worse than a brief glitch.

Missing data = acceptable.
Stale data = unacceptable in live context.
Weak consistency = correct choice here.
```

#### Multiplayer Gaming

```
Fortnite / Call of Duty / CS:GO:

100 players. Each moving constantly.
Each player's position sent 20-60 times per second.

Network conditions vary per player.
Some packets arrive late.
Some packets are lost.

Weak consistency behavior:
Your position on my screen
may be 50-100ms behind reality.
My game client interpolates between
your last known positions.
Fills the gap with prediction.

Strong consistency alternative:
Wait for confirmed position before rendering.
Result: 100-200ms of added lag.
Game becomes unplayable.

Approximate position = acceptable.
Playable game = critical.
Weak consistency = correct choice.
```

#### Real-Time Analytics

```
Twitter trending topics:
Exact tweet count per topic not required.
#WorldCup showing 1,042,847 vs 1,042,851 = identical UX.

Implementation:
Each server increments its own local counter.
Counters periodically synced to central store.
Central store shows approximate sum.

Strong consistency alternative:
Every tweet requires distributed lock on counter.
Millions of tweets per minute.
Global lock = complete system bottleneck.

Approximate count = acceptable.
System availability = critical.
Weak consistency = correct choice.
```

#### VoIP Calls

```
WhatsApp Voice / Zoom / Google Meet:

Audio encoded into packets → 50 packets per second.
Network drops a packet.
Audio for 20ms is lost.

Weak consistency behavior:
Lost packet NOT retransmitted.
Brief silence or static for 20ms.
Call continues in real time.

Why not retransmit?
By the time retransmitted packet arrives —
conversation has moved 200ms forward.
Playing old audio = incoherent conversation.

Missing 20ms of audio = acceptable.
Real-time conversation = critical.
Weak consistency = correct choice.
```

### When to Use Weak Consistency

| Use Case | Why Weak | Alternative Cost |
|---|---|---|
| Live video frames | Lost frame acceptable | Retransmit = stuttering |
| VoIP audio | Dropped audio acceptable | Retransmit = incoherent |
| Game positions | Interpolated position fine | Strong = unplayable lag |
| Real-time counters | Approximate is fine | Strong = global bottleneck |
| Sensor telemetry | Missing reading acceptable | Strong = IoT at scale impossible |
| Live sports scores | 1 second delay fine | Strong = massive coordination cost |

---

## Comparison — All Four Models

| Dimension | Strong | Sequential | Eventual | Weak |
|---|---|---|---|---|
| Latest write returned | Always | Ordered | Eventually | Never guaranteed |
| Stale data possible | Never | Briefly | Yes, briefly | Always possible |
| Data loss possible | No | No | No (eventually) | Yes |
| Latency | Highest | High | Low | Lowest |
| Throughput | Lowest | Low-Medium | High | Highest |
| Availability | Lowest | Low-Medium | High | Highest |
| Partition behavior | Returns error | Returns error sometimes | Returns stale | Returns stale |
| Conflict resolution | Not needed (locks prevent) | Not needed (ordered) | LWW, CRDTs, Vectors | Not needed (no guarantee) |
| Implementation cost | Highest | High | Medium | Lowest |
| Real example | PostgreSQL, Spanner | CPU memory model | Cassandra, DNS | Game servers, VoIP |
| Use case | Banking, payments | Multi-threaded systems | Social, caches | Live media, gaming |

---

## Real Systems — Consistency Model Choices

### Instagram — Multiple Models in One System

```
Instagram uses different consistency models
for different data types within the same system:

User authentication → Strong Consistency (PostgreSQL)
  Wrong auth = security breach
  Must always be correct

Payment processing → Strong Consistency (PostgreSQL)
  Wrong payment = financial loss
  Must always be correct

Photo storage metadata → Strong Consistency
  Corrupted photo metadata = permanent data loss
  Must always be correct

Social feed → Eventual Consistency (Cassandra)
  Seeing photo 1 second late = fine
  Availability more important

Like counts → Eventual Consistency (Redis async)
  Approximate count = fine
  Speed more important

Story views → Eventual Consistency
  Approximate view count = fine

Live video frames → Weak Consistency
  Dropped frame = fine
  Real-time more important

Same company. Same app.
Four different consistency models.
Each chosen deliberately for each data type.
```

### Amazon — Eventual Consistency for Shopping Cart

```
Amazon shopping cart uses DynamoDB with eventual consistency.

Why not strong consistency?

Shopping cart requirements:
→ Must always be accessible (Prime Day: 300M users)
→ Slight sync delay across devices = fine
→ Adding item on mobile, seeing it on desktop
  1 second later = fine

Strong consistency cost:
→ During any network partition: cart unavailable
→ Prime Day partition = millions in lost revenue per minute
→ Unacceptable

Eventual consistency behavior:
→ Add item on mobile → immediate response
→ DynamoDB replicates async across regions
→ Desktop refreshes 1 second later → item appears
→ Cart always available even during partition

One edge case: adding same item on two devices simultaneously
→ Both writes accepted (AP behavior)
→ DynamoDB reconciles via Last Write Wins
→ User sees item once (deduplication logic in app)
→ Acceptable outcome

Eventual consistency = correct choice for shopping cart.
```

---

## How to Choose Your Consistency Model

### The Decision Framework

```
For every data type in your system:

Question 1:
"What is the worst that happens
if a user sees stale data for 2 seconds?"

Catastrophic (money, safety, legal) → Strong Consistency
Significant but recoverable → Sequential Consistency  
Minor and temporary → Eventual Consistency
Completely irrelevant → Weak Consistency
```

### The Decision Table

| Worst case of stale data | Model | Database |
|---|---|---|
| Money lost | Strong | PostgreSQL, Spanner |
| Legal liability | Strong | PostgreSQL, MySQL |
| Safety risk | Strong | PostgreSQL, etcd |
| Double booking | Strong | PostgreSQL, MySQL |
| Overselling | Strong | PostgreSQL, MySQL |
| Wrong order shown | Sequential/Strong | Relational DB |
| User sees old profile | Eventual | Cassandra, DynamoDB |
| Like count is off by 1 | Eventual | Redis, Cassandra |
| Feed post appears 1s late | Eventual | Cassandra |
| Video frame missing | Weak | Custom, game servers |
| Counter approximate | Weak | In-memory, Redis |

### The Practical Rule

```
10% of your data → Strong Consistency
(money, auth, inventory, booking, medical)

90% of your data → Eventual Consistency
(feeds, counts, recommendations, profiles, caches)

<1% of your data → Weak Consistency
(live media, gaming, real-time telemetry)

Most engineers over-use strong consistency.
Most production bugs come from
accidentally using eventual where strong was needed.
```

---

## Common Mistakes

### Mistake 1 — Strong Consistency Everywhere

```
"PostgreSQL for everything. Safety first."

Problem:
Social feed on PostgreSQL with distributed locks.
1 million concurrent readers.
Every read waiting for lock confirmation.
Latency spikes. Throughput collapses.
System cannot scale.

Fix:
Use eventual consistency (Cassandra/DynamoDB)
for feed data.
Reserve strong consistency for financial data only.
```

### Mistake 2 — Eventual Consistency Everywhere

```
"Cassandra scales better. Use it for everything."

Problem:
Payment service on Cassandra with eventual consistency.
User clicks pay → write hits Node A.
User immediately redirected → read from Node B.
Node B not yet synced.
User sees "payment pending" — clicks pay again.
Duplicate charge.

Fix:
Use strong consistency (PostgreSQL) for payments.
Idempotency keys to prevent duplicate charges.
```

### Mistake 3 — Not Knowing Your Database Default

```
"I am using MongoDB so I am fine."

Problem:
MongoDB default consistency changed between versions.
Pre-4.0: eventual consistency by default.
Post-4.0: strong consistency available but not default.
Engineers assumed strong. Got eventual.
Bugs in production only visible under load.

Fix:
Always explicitly configure and verify
your database's consistency level.
Never assume.
Read the documentation for your specific version.

MongoDB:
db.collection.find().readConcern("linearizable")  // strong
db.collection.find().readConcern("majority")       // strong-ish  
db.collection.find().readConcern("local")          // eventual (default)
```

### Mistake 4 — Ignoring Replication Lag

```
"We have replication so we are consistent."

Problem:
Primary PostgreSQL in US-East.
Read replica in EU-West.
Async replication lag: 200ms.

Application reads from replica immediately after write.
Gets stale data.
Engineers confused — "we have PostgreSQL, it's ACID!"

ACID applies to the primary.
Read replicas with async replication = eventual consistency.

Fix:
Route writes AND immediate reads to primary.
Route all other reads to replica.
Or use sync replication (higher latency, no lag).
```

---

## Quick Reference

### Consistency Models Cheat Sheet

```
STRONG     → Always correct. Slow. Use for money/safety.
SEQUENTIAL → Ordered. Not real-time. Use for threading.
EVENTUAL   → Fast. Briefly stale. Use for most things.
WEAK       → No guarantee. Fastest. Use for live media.
```

### Database Consistency Quick Reference

| Database | Default Consistency | Configurable? |
|---|---|---|
| PostgreSQL | Strong (ACID) | Yes (read replicas = eventual) |
| MySQL | Strong (InnoDB) | Yes (async replica = eventual) |
| Google Spanner | Strong | No (always strong) |
| Cassandra | Eventual (tunable) | Yes (ONE to ALL) |
| DynamoDB | Eventual | Yes (strongly consistent reads) |
| MongoDB | Eventual (local) | Yes (linearizable available) |
| Redis | Eventual (async) | Yes (WAIT command) |
| Zookeeper | Strong | No (always strong) |
| etcd | Strong (Raft) | No (always strong) |
| DNS | Eventual (TTL) | Partial (TTL controls window) |

---

## What Comes Next

```
You now understand:
→ What consistency models are
→ All 4 models in depth
→ How real systems choose
→ How to choose for your system

Next → Fail-over vs Replication
How do systems stay alive when nodes fail?
What is the difference between
backing up data and handling failure?
How do the 9s of availability connect
to fail-over and replication strategies?
```

---

*All content original. Created by @kriti.sde*
*github.com/rai-kriti/UNDERSTANDING-SYSTEMS*
