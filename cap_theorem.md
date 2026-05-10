# CAP Theorem — Complete Deep Dive
> Created by @kriti.sde | All content original.
> Part of the UNDERSTANDING-SYSTEMS series.

---

## What is CAP Theorem?

CAP Theorem states:

> A distributed system can only guarantee 2 out of 3 properties simultaneously — Consistency, Availability, and Partition Tolerance.

Proposed by Eric Brewer at PODC 2000.
Formally proven by Gilbert and Lynch at MIT in 2002.

This is not a guideline. Not a best practice. Not a suggestion.
It is a **mathematical proof.**

Every distributed system ever built obeys CAP theorem — whether the engineer knew it or not.

---

## The Three Properties

### C — Consistency

Every node in the distributed system returns the **most recent write** for every read.

No node ever returns stale, old, or outdated data. Every read reflects the latest write — regardless of which node receives the request.

#### Technical Definition
```
Linearizability:
Every operation appears to execute instantaneously
at some point between its invocation and completion.

In simple terms:
The system behaves as if it has a single node
even though it has many.
```

#### What Consistency Looks Like

```
Write: User updates balance → $500 → hits Node A
       Node A replicates to Node B, Node C immediately

Read:  Any subsequent read from Node A, B, or C
       returns $500. Always. No exceptions.
```

#### What Consistency Violation Looks Like

```
Write: User updates balance → $500 → hits Node A
       Replication to Node B is delayed by 200ms

Read:  User reads from Node B within those 200ms
       Node B returns $300 (old value)
       Consistency is violated.
       User sees stale data.
```

#### Real World Consistency Requirement

| System | Why Consistency Matters |
|---|---|
| Bank balance | Wrong balance shown = overdraft undetected |
| Stock price | Stale price = wrong trade executed |
| Inventory | Old count = overselling |
| Seat booking | Stale availability = double booking |
| Medical records | Wrong dosage = patient risk |

---

### A — Availability

Every request receives a **non-error response** within a reasonable time — even if some nodes are failing.

The system never refuses to answer. It always responds with something.

#### Technical Definition
```
Total Availability:
For every request that reaches a non-failing node,
the system must return a non-error response
within a finite time bound.

Key: non-error response.
Returning an error IS a violation of availability.
```

#### What Availability Looks Like

```
System: 10 nodes
Failure: 3 nodes go down simultaneously

With availability guaranteed:
Remaining 7 nodes keep responding.
Every request gets a response.
No 503 errors. No timeouts.
App keeps working.
```

#### What Availability Violation Looks Like

```
System: 10 nodes
Failure: Network partition between Node A and Node B

Without availability guarantee:
System detects inconsistency risk.
Returns 503 Service Unavailable.
Users see error page.
Availability is violated.
```

#### Availability in Numbers

| Availability | Downtime/year | Downtime/month | Downtime/week |
|---|---|---|---|
| 90% | 36.5 days | 72 hours | 16.8 hours |
| 99% | 3.65 days | 7.2 hours | 1.68 hours |
| 99.9% | 8.76 hours | 43.8 minutes | 10.1 minutes |
| 99.99% | 52.6 minutes | 4.38 minutes | 1.01 minutes |
| 99.999% | 5.26 minutes | 26.3 seconds | 6.05 seconds |

---

### P — Partition Tolerance

The system continues to operate even when **network communication between nodes is lost.**

A network partition = nodes are alive but cannot communicate with each other. Messages are dropped. Sync is impossible.

#### Technical Definition
```
Network Partition:
A scenario where the network fails to deliver
messages between nodes.

Nodes are running.
Nodes are processing requests.
But nodes cannot communicate with each other.
```

#### What a Partition Looks Like

```
Normal operation:
Node A (US-East) ←──────────→ Node B (EU-West)
Both communicating. Data in sync.

During partition:
Node A (US-East)  ✗──────────✗  Node B (EU-West)
Network link broken.
Both nodes still running.
Both serving requests.
Cannot sync data.
```

#### Why Partitions Happen in Real Systems

| Cause | Frequency | Example |
|---|---|---|
| Network cable cut | Rare but happens | AWS fiber cut 2019 |
| Switch/router failure | Occasional | Data center hardware failure |
| BGP routing failure | Rare, catastrophic | Cloudflare outage 2022 |
| Cross-region latency spike | Common | US-EU link saturation |
| DDoS attack | Increasing | Targeted infrastructure attack |
| Software bug in network stack | Occasional | Linux kernel TCP bug |
| Power failure in DC | Rare | AWS us-east-1 2021 |

**Partitions are not rare. They happen constantly across large distributed systems.**

---

## The Proof — Why You Cannot Have All 3

### The Setup

```
System: 2 nodes — Node A (US-East), Node B (EU-West)
Data: User balance, replicated on both nodes
Initial state: Both nodes show balance = $1,000
```

### The Sequence

```
Step 1 — Write occurs:
User withdraws $500. Request hits Node A.
Node A updates: balance = $500
Node A begins replicating to Node B.
Replication takes ~150ms (speed of light, US to EU)

Step 2 — Partition occurs:
At 50ms into replication — network breaks.
Replication never completes.
Node B still shows: balance = $1,000
Node A shows: balance = $500

Step 3 — Read request hits Node B:
Another user (or the same user from EU)
reads their balance from Node B.
```

### The Impossible Choice

```
YOU MUST CHOOSE ONE:

Option A — Respond immediately (choose Availability):
Node B returns $1,000 (its last known value)
User gets a response ✓ → Availability preserved
User gets wrong data ✗ → Consistency violated

Option B — Refuse to respond (choose Consistency):
Node B detects it cannot verify latest data
Node B returns error: "503 Service Unavailable"
User gets correct data ✓ → Consistency preserved
User gets no response ✗ → Availability violated
```

### Why There is No Third Option

```
Could Node B just wait for partition to heal?

Partition could last:
- Milliseconds (brief network blip)
- Minutes (routing failure)
- Hours (hardware failure)
- Days (major outage)

"Wait" means unavailable for the duration.
"Respond" means potentially returning stale data.

No third option.
This is why it is a theorem.
Gilbert and Lynch proved this formally in 2002.
```

---

## Why P is Not Optional

The most common misconception about CAP:

> "I will just choose CA — Consistency + Availability — and ignore Partition Tolerance."

**This is not possible in a distributed system.**

### The Mathematical Reason

```
A distributed system = multiple nodes communicating over a network.
A network = a medium that can and will fail.
Therefore: every distributed system will experience partitions.

CA system = a system that assumes no network failure will ever occur.
Network failure will occur.
Therefore: CA systems cannot exist in distributed systems.
```

### The Practical Reason

Networks fail constantly. At scale, this is not rare — it is routine.

| Infrastructure | Failure Rate |
|---|---|
| Single HDD | 1-5% annual failure rate |
| Single server | 1-3% annual failure rate |
| Network link | Failures happen daily at scale |
| Cross-region network | Disruptions happen weekly |
| Cloud provider region | Major outage 2-4 times per year |

Google's internal study found that in their infrastructure, **thousands of network partitions occur every year.**

### The Conclusion

```
The moment you have 2 nodes → distributed system
The moment you have a distributed system → partitions will occur
The moment partitions occur → you must handle them

P is not optional.
CA is a myth in distributed systems.

The real choice is ALWAYS:
CP or AP.
```

---

## CP vs AP — The Real Trade-off

Since P is mandatory, the only real decision is:

**When a partition occurs — what do you sacrifice?**

### CP — Consistency + Partition Tolerance

| Dimension | Detail |
|---|---|
| Guarantee | Every read returns most recent write |
| Sacrifice | Availability during partition |
| User experience | Error returned during partition |
| Data guarantee | Never returns stale data |
| Recovery | Responds again once partition heals |
| Use case | Financial, medical, booking systems |

#### What Happens During a Partition in a CP System

```
Partition detected between Node A and Node B.

Request arrives at Node B.
Node B checks: "Can I verify I have the latest data?"
Answer: No — partition prevents sync with Node A.

Node B response: 503 Service Unavailable

User sees: Error message
Data: Never returned incorrect data
```

#### CP Databases and Why

| Database | Why CP | Use Case |
|---|---|---|
| PostgreSQL | ACID transactions, sync replication | Financial systems, ERP |
| MySQL (sync mode) | Master-slave with sync replication | Banking, e-commerce |
| HBase | Single master for writes | Hadoop analytics |
| Zookeeper | Quorum-based writes | Config management, leader election |
| Google Spanner | TrueTime API with atomic clocks | Global financial data |
| etcd | Raft consensus algorithm | Kubernetes config store |

#### Google Spanner — CP at Global Scale

```
Challenge: Global database with strong consistency
Physics problem: Speed of light = minimum 150ms US-EU latency
Solution: TrueTime API

TrueTime uses:
- GPS receivers in each datacenter
- Atomic clocks in each datacenter
- Uncertainty bounds on timestamps

Result: Globally consistent transactions
        with known timestamp uncertainty (1-7ms)

During partition: Returns error
Consistency: Never violated
Used for: Google Ads billing (billions of dollars)
```

---

### AP — Availability + Partition Tolerance

| Dimension | Detail |
|---|---|
| Guarantee | Every request gets a response |
| Sacrifice | Consistency during partition |
| User experience | Response returned (may be stale) |
| Data guarantee | Eventually consistent — corrects after partition heals |
| Recovery | Data syncs automatically once partition heals |
| Use case | Social media, DNS, caches, shopping carts |

#### What Happens During a Partition in an AP System

```
Partition detected between Node A and Node B.

Request arrives at Node B.
Node B checks: "Can I verify I have the latest data?"
Answer: No — partition prevents sync with Node A.

Node B response: Returns last known data

User sees: Response (may be slightly stale)
Data: May be stale for duration of partition
After partition heals: Node B syncs with Node A
                       Data becomes consistent again
```

This is called **Eventual Consistency.**

#### AP Databases and Why

| Database | Why AP | Use Case |
|---|---|---|
| Cassandra | Leaderless replication, always available | Social feeds, IoT, messaging |
| DynamoDB | Multi-region active-active | Shopping cart, gaming, ad tech |
| CouchDB | Multi-master replication | Mobile sync, offline-first apps |
| Riak | Consistent hashing, high availability | Session storage, user profiles |
| DNS | TTL-based cache invalidation | Domain resolution for entire internet |

#### Cassandra — AP at Facebook Scale

```
Built by: Facebook (2008) for inbox search
Problem: Search across billions of messages
         Must always be available
         Brief staleness acceptable

Design decisions:
- Leaderless replication (no single point of failure)
- Tunable consistency (can choose per query)
- Write to any node, replicate asynchronously

During partition:
- Both sides of partition keep accepting writes
- After partition heals — conflict resolution via
  Last Write Wins (LWW) or application logic

Result: 
Facebook Messenger: Always available
Instagram: Always loads
Netflix: Viewer history always accessible
```

---

## Consistency Models — The Spectrum

Consistency is not binary. Between strong consistency and no consistency, there is a full spectrum:

```
STRONG ─────────────────────────────────────── WEAK
  │              │              │                │
Strong      Sequential      Eventual           Weak
Consistency  Consistency   Consistency      Consistency
  │              │              │                │
Always        Ordered        Eventually       No
correct       correct         correct        guarantee
  │              │              │                │
Slowest        Slow           Fast            Fastest
  │              │              │                │
Banks        Financial      Social media      Gaming,
payments      reports        DNS, caches      analytics
```

### Strong Consistency
```
Every read returns the most recent write. Always.
System may refuse request rather than return stale data.

Technical implementation:
- 2-Phase Locking (2PL)
- Paxos consensus
- Raft consensus
- Two-Phase Commit (2PC)

Examples: PostgreSQL, Google Spanner, Zookeeper
Cost: Highest latency, lowest availability
```

### Sequential Consistency
```
All operations appear to execute in some sequential order.
Each node's operations appear in the order they were issued.
But global order may differ from real-time order.

Example: Operations appear sequential but
         may not reflect real-time global order.
Cost: Lower latency than strong, higher than eventual
```

### Eventual Consistency
```
All nodes will eventually have the same data.
Reads may return stale data temporarily.
System always responds (high availability).

Technical implementation:
- Vector clocks
- CRDTs (Conflict-free Replicated Data Types)
- Last Write Wins (LWW)
- Read-repair

Examples: Cassandra, DynamoDB, DNS
Cost: May briefly return stale data
```

### Weak Consistency
```
After a write, reads may or may not see it.
No guarantee when or if data will be consistent.
Maximum performance and availability.

Examples: 
- Live video streams (missed frames are gone)
- Multiplayer game positions (interpolated)
- Real-time analytics counters

Cost: Data may be permanently inconsistent
```

---

## How Real Systems Chose

| System | Choice | Reason | Database |
|---|---|---|---|
| Google Ads billing | CP | Wrong billing = direct revenue loss | Google Spanner |
| Amazon shopping cart | AP | Cart must always load on Prime Day | DynamoDB |
| Facebook social feed | AP | Seeing post 2s late is fine | Cassandra |
| WhatsApp messages | AP | Always delivered, order eventually consistent | Mnesia + custom |
| Uber ride matching | CP | Wrong driver assignment = safety issue | MySQL + Schemaless |
| Netflix recommendations | AP | Slightly stale recommendations = fine | Cassandra |
| DNS resolution | AP | Cached IP until TTL expires | BIND, PowerDNS |
| Kubernetes config | CP | Wrong config to cluster = cascading failure | etcd (Raft) |
| Instagram likes | AP | Approximate count is acceptable | Cassandra |
| Bank transactions | CP | Wrong balance = money lost | PostgreSQL |

### The Instagram Architecture Decision

```
Instagram uses BOTH CP and AP — for different data:

User authentication → CP (PostgreSQL)
Wrong auth = security breach

Payment processing → CP (PostgreSQL)  
Wrong payment = financial loss

Photo feed → AP (Cassandra)
Seeing photo 1 second late = fine

Like counts → AP (Redis with async sync)
Approximate count = fine

Comment ordering → Eventually consistent
Comments appearing slightly out of order = acceptable

Same company. Same system.
Different CAP choice for different data types.
```

---

## How to Choose CP or AP for Your System

### The Decision Framework

```
For every data type in your system, ask:

"If this data is wrong for 2 seconds —
 what is the worst that happens?"
```

| Answer | Choice | Reasoning |
|---|---|---|
| Money is lost | CP | Financial damage is unacceptable |
| Legal liability created | CP | Compliance requires correctness |
| Safety risk introduced | CP | Human safety non-negotiable |
| Double booking occurs | CP | Operational damage unacceptable |
| Inventory oversold | CP | Fulfillment failure |
| User sees stale feed | AP | Acceptable, corrects itself |
| Like count is approximate | AP | No real harm |
| Recommendation is old | AP | Acceptable degradation |
| Cart syncs 1s late | AP | Acceptable UX impact |

### The Practical Rule

```
Money and critical operations → CP
Everything else → AP with eventual consistency

Most modern systems:
90% of data → AP (social, cache, recommendations, feeds)
10% of data → CP (payments, auth, inventory, booking)
```

### The Tunable Consistency Approach

Some databases (Cassandra, DynamoDB) allow **tunable consistency** — you choose per query:

```
Cassandra Consistency Levels:

ONE   → Read from 1 replica → fastest, least consistent
QUORUM → Read from majority → balanced
ALL   → Read from all replicas → slowest, most consistent

// Strong consistency read (CP behavior)
SELECT * FROM payments WHERE id = ? 
WITH CONSISTENCY QUORUM;

// Fast read (AP behavior)  
SELECT * FROM feed WHERE user_id = ?
WITH CONSISTENCY ONE;

Same database. Different consistency per query.
```

---

## CAP in System Design Interviews

When an interviewer asks: **"Why did you choose Cassandra?"**

❌ Wrong answer:
> "Because it's fast and scalable."

✅ Right answer:
> "This system stores social feed data. The data can tolerate eventual consistency — seeing a post one second late is acceptable. Downtime is more damaging than brief staleness. Cassandra is an AP system — it sacrifices consistency for availability during network partitions. That trade-off is correct for this use case."

---

When an interviewer asks: **"What happens if your database goes down?"**

❌ Wrong answer:
> "We have backups."

✅ Right answer:
> "We use PostgreSQL with synchronous replication for payment data — CP choice. During partition, we return 503 rather than risk incorrect payment processing. For feed data, we use Cassandra — AP choice. During partition, both sides keep serving cached data. After partition heals, Cassandra reconciles via Last Write Wins. Recovery is automatic."

---

## Quick Reference

### CAP Summary Table

| Property | Guarantee | Violated when | Technical term |
|---|---|---|---|
| Consistency | Latest write returned always | Node returns stale data | Linearizability |
| Availability | Every request gets response | System returns error | Total availability |
| Partition Tolerance | Works despite network failure | System halts on partition | Split-brain handling |

### CP vs AP Summary

| Dimension | CP | AP |
|---|---|---|
| Sacrifices | Availability | Consistency |
| During partition | Returns error | Returns stale data |
| Data guarantee | Always correct | Eventually correct |
| Use case | Financial, medical, booking | Social, DNS, cache |
| Examples | PostgreSQL, Spanner, Zookeeper | Cassandra, DynamoDB, DNS |
| Consistency model | Strong | Eventual |

### The 5 Numbers to Remember

```
1. CAP proven in: 2002 (Gilbert and Lynch, MIT)
2. Google partitions per year: Thousands
3. AWS major outages per year: 2-4 per region
4. Cassandra replication: Async, tunable
5. Spanner clock uncertainty: 1-7ms (TrueTime)
```

---

## What Comes Next

```
CAP Theorem governs the consistency vs availability trade-off.

But consistency itself is not binary.
There is a full spectrum between strong and weak.

Next → Consistency Models
Strong consistency · Sequential consistency
Eventual consistency · Weak consistency

How each model works technically.
Which systems use which model.
How to choose for your system.
```

---

*All content original. Created by @kriti.sde*
*github.com/rai-kriti/UNDERSTANDING-SYSTEMS*
