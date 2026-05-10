# Core Concepts of System Design — Complete Deep Dive
> Created by @kriti.sde | All content original.

---

## Introduction — Why These Concepts Come Before Everything Else

Before you draw a single component in HLD.
Before you choose a database.
Before you pick an architecture pattern.

You must understand the **forces** that act on every distributed system.

These are not optional concepts. They are the **physics of system design.**

Just like a civil engineer cannot design a bridge without understanding gravity and load — a system designer cannot design a system without understanding these 8 forces.

Every decision you make in HLD traces back to one of these concepts.

---

## The 8 Core Concepts

```
┌─────────────────────────────────────────────────┐
│           CORE CONCEPTS MAP                     │
│                                                 │
│  SPEED GROUP          TRUST GROUP               │
│  ├── Performance      ├── Availability          │
│  ├── Scalability      ├── Reliability           │
│  ├── Latency          └── Consistency           │
│  └── Throughput                                 │
│                                                 │
│  TRADE-OFF GROUP                                │
│  └── Availability vs Consistency                │
└─────────────────────────────────────────────────┘
```

---

## Concept 1 — Performance vs Scalability

### The Core Confusion

> "My system is fast. That means it scales."

This is the most common misconception in system design.
Performance and scalability are **two completely different problems** requiring **two completely different solutions.**

---

### Performance — Defined

**Performance** is how fast your system serves **one request** at **one moment in time.**

It is a measurement of speed for a **single operation** in isolation.

#### Technical Definition
```
Performance = f(code efficiency, algorithm complexity, hardware speed)
```

#### How It Is Measured

| Metric | What it measures | Example |
|---|---|---|
| Response time | Time for one request to complete | API responds in 200ms |
| Latency | Delay in the system for one operation | DB query takes 20ms |
| CPU time | Processor time consumed per request | Function takes 5ms CPU |
| Memory usage | RAM consumed per operation | 50MB per request |

#### What Causes Poor Performance

| Root Cause | Technical Reason | Fix |
|---|---|---|
| Slow DB query | Missing index, full table scan | Add index, optimize query |
| Unoptimized algorithm | O(n²) instead of O(n log n) | Rewrite algorithm |
| Excessive memory | Memory leak, large objects | Profile and fix memory |
| Blocking I/O | Thread waiting for disk/network | Async I/O |
| N+1 query problem | 1 query per record instead of JOIN | Use JOIN or batch query |

#### Real Technical Example — N+1 Query Problem

```sql
-- BAD PERFORMANCE: N+1 queries
-- 1 query to get all users
SELECT * FROM users;  -- returns 1000 users

-- Then 1 query PER user to get their orders
-- This runs 1000 times
SELECT * FROM orders WHERE user_id = ?;

-- Total: 1001 queries. Slow.

-- GOOD PERFORMANCE: 1 query with JOIN
SELECT users.*, orders.*
FROM users
LEFT JOIN orders ON users.id = orders.user_id;

-- Total: 1 query. Fast.
```

**Result:** Response time drops from 2000ms to 20ms.
That is a **performance fix.** The system is now fast for one user.

---

### Scalability — Defined

**Scalability** is whether that performance **holds** as load increases from 1 user to 1,000,000 users.

A system is scalable if adding more users does not proportionally degrade performance.

#### Technical Definition
```
Scalability = ability to handle increasing load
             by adding resources proportionally
```

#### The Scalability Equation

```
Ideal scalability (linear):
  2x resources → 2x capacity

Reality (due to coordination overhead):
  2x resources → 1.7x capacity (Amdahl's Law)

Bad scalability:
  2x resources → 1.1x capacity (bottleneck exists)
```

#### Amdahl's Law — The Physics of Scalability

```
Speedup = 1 / (S + (1-S)/N)

Where:
  S = fraction of work that CANNOT be parallelized
  N = number of processors/servers
  (1-S) = fraction that CAN be parallelized
```

**Example:**

If 20% of your system cannot be parallelized (S = 0.20):

| Servers (N) | Max Speedup | Efficiency |
|---|---|---|
| 1 | 1.0x | 100% |
| 2 | 1.67x | 83% |
| 4 | 2.5x | 63% |
| 8 | 3.33x | 42% |
| 16 | 4.0x | 25% |
| ∞ | 5.0x | ~0% |

**Key insight:** Even with infinite servers, you can only get 5x speedup if 20% of work is serial.
**This is why scalability is an architecture problem, not a code problem.**

#### What Causes Poor Scalability

| Root Cause | Technical Reason | Fix |
|---|---|---|
| Single database | All requests hit one DB | Sharding, read replicas |
| No caching | Every request hits storage | Redis, Memcached |
| Shared mutable state | All servers share one lock | Stateless services |
| Synchronous calls | Services wait for each other | Async, message queues |
| Monolithic deployment | Cannot scale one part | Microservices |

#### Real Technical Example — Instagram 2011

```
BEFORE SCALE EVENT:
System: Single PostgreSQL + 3 app servers
Load: ~100,000 users
Performance: 200ms API response ✓
Scalability: Unknown — never tested

THE EVENT:
Featured on App Store → 1,000,000 users in 12 hours

WHAT HAPPENED TECHNICALLY:
PostgreSQL max connections: 100
Simultaneous requests: 50,000
Each request tried to get a DB connection
Connection pool exhausted in milliseconds
Requests queued → waited → timed out
Response time: 200ms → 30,000ms → timeout

THE FIX (scalability, not performance):
- Sharded PostgreSQL across multiple servers
- Added Memcached layer (reduced DB hits by 80%)
- Added more read replicas
- Moved to async processing for non-critical operations
```

**Note:** The queries were still fast (200ms). The **code** was not the problem. The **architecture** was.

---

### Performance vs Scalability — Direct Comparison

| Dimension | Performance | Scalability |
|---|---|---|
| Question it answers | Is my code fast? | Does fast survive at scale? |
| Measured by | Response time, latency | Requests/sec, concurrent users |
| Problem lives in | Code, algorithms, queries | Architecture, infrastructure |
| Fixed by | Optimize code, add index | Shard DB, add servers, cache |
| Tested with | Single user benchmark | Load testing (JMeter, k6) |
| Example tool | Query profiler, APM | Load balancer, auto-scaling |
| When it breaks | Always (one user is slow) | Under high concurrent load |

---

### The Key Rule

```
Performance = is your code fast?
Scalability = does your architecture survive?

Fast code inside a broken architecture
will fail at scale every single time.

You need both.
They need completely different solutions.
```

---

## Concept 2 — Latency vs Throughput

### The Core Confusion

> "Low latency means high throughput."

Wrong. They are **independent axes.** You can have any combination of the two.

---

### Latency — Defined

**Latency** is the time it takes for **one request** to travel from the user to the system and back.

It is the measurement of **delay** for a single operation.

#### Technical Definition
```
Latency = Time from request sent → response received
        = Network latency + Processing latency + Queue latency
```

#### Latency Breakdown — Where Time Is Spent

```
User clicks button
     │
     ▼ ← DNS lookup: ~20-120ms
     ▼ ← TCP handshake: ~50-200ms
     ▼ ← TLS handshake: ~50-200ms
     ▼ ← Request travels network: ~1-150ms
     ▼ ← Server processes: ~1-5000ms
     ▼ ← Database query: ~1-2000ms
     ▼ ← Response travels back: ~1-150ms
     │
User sees response
```

#### Latency Numbers Every Engineer Must Know

| Operation | Typical Latency | Notes |
|---|---|---|
| L1 cache reference | 0.5 ns | Fastest possible |
| L2 cache reference | 7 ns | |
| RAM reference | 100 ns | 200x slower than L1 |
| SSD random read | 150 µs | 150,000 ns |
| HDD random read | 10 ms | 10,000,000 ns |
| Same datacenter network | 0.5 ms | LAN |
| Cross-region (US-EU) | 150 ms | Speed of light limit |
| Redis GET | ~0.5 ms | In-memory |
| PostgreSQL indexed query | ~5-10 ms | With good indexing |
| PostgreSQL full table scan | ~100ms-10s | Depends on table size |

#### The Speed of Light Problem

```
Distance: New York → London = 5,570 km
Speed of light in fiber: ~200,000 km/s

Minimum theoretical latency = 5,570 / 200,000 = 27.85ms ONE WAY
Round trip minimum = ~56ms

Reality (with routing, processing): ~80-150ms

This is a physics limit.
No amount of code optimization can beat it.
This is why CDNs and edge servers exist.
```

#### What Causes High Latency

| Root Cause | Added Latency | Fix |
|---|---|---|
| Slow database query | 100ms-10s | Index, optimize, cache |
| No CDN (static files from origin) | 100-300ms | Add CDN |
| Synchronous external API calls | 200-2000ms | Cache, async, timeout |
| N+1 queries | Multiplied latency | Batch queries, JOIN |
| No connection pooling | 50-200ms per request | Add connection pool |
| Large payload size | 50-500ms | Compress, paginate |

---

### Throughput — Defined

**Throughput** is how many requests your system can process **per unit of time.**

It is the measurement of **volume** the system can handle.

#### Technical Definition
```
Throughput = Number of requests processed / Time period
           = Requests per second (RPS) or
             Transactions per second (TPS) or
             Messages per second (MPS)
```

#### Little's Law — The Physics of Throughput

```
L = λ × W

Where:
  L = average number of requests in the system
  λ = average arrival rate (requests per second)
  W = average time a request spends in the system (latency)

Rearranged:
  λ = L / W
  Throughput = Concurrency / Latency
```

**Example:**

```
Your system handles 100 concurrent requests (L = 100)
Each request takes 100ms to process (W = 0.1 seconds)

Throughput = 100 / 0.1 = 1,000 requests per second

To get 2,000 RPS you can either:
  → Double concurrency to 200 (add more servers/threads)
  → Halve latency to 50ms (optimize processing)
```

#### What Limits Throughput

| Bottleneck | What it limits | Fix |
|---|---|---|
| Single-threaded server | CPU cores unused | Use all cores, async |
| Database connections | DB can't accept more | Connection pooling |
| Network bandwidth | Bytes in/out saturated | Compress, CDN |
| Single queue consumer | Messages pile up | Add consumers, partition |
| Synchronous processing | One thing at a time | Async, parallel processing |

#### Real Technical Example

```
PAYMENT PROCESSING SYSTEM

Latency: Each payment takes 800ms to process
         (bank verification + fraud check + DB write)

Throughput needed: 10,000 transactions per second

Using Little's Law:
  Concurrent requests needed = Throughput × Latency
  = 10,000 × 0.8
  = 8,000 concurrent requests

This means: 8,000 threads/connections handling payments simultaneously

Solution: Message queue (Kafka)
  → Payments arrive → drop into Kafka topic
  → 1,000 consumer workers each handle 10 payments/sec
  → Total: 10,000 TPS achieved
  → Latency per payment: still 800ms
  → Throughput: 10,000 TPS ✓
```

**High latency (800ms) + High throughput (10,000 TPS) — proven possible.**

---

### Latency vs Throughput — Direct Comparison

| Dimension | Latency | Throughput |
|---|---|---|
| Question it answers | How long does ONE request take? | How many requests per second? |
| Unit | Milliseconds (ms) | Requests/sec (RPS), TPS |
| Measured per | Single request | Time window |
| Problem lives in | Processing speed, network | Concurrency, parallelism |
| Fixed by | Cache, optimize, CDN | Add workers, queues, scale |
| Test tool | Single request timing | Load test (k6, JMeter) |
| Physics limit | Speed of light (network) | Amdahl's Law (parallelism) |

#### All Four Combinations Exist

| Latency | Throughput | Real Example |
|---|---|---|
| Low (fast) | Low (few) | Internal admin tool, 10 users |
| Low (fast) | High (many) | Google Search — fast AND millions of queries/sec |
| High (slow) | High (many) | Batch video encoding — slow per video, thousands parallel |
| High (slow) | Low (few) | Legacy mainframe system — slow AND limited capacity |

---

### The Key Rule

```
Latency = how long does one request take?
Throughput = how many requests per second?

They are independent axes.
Optimizing one does NOT automatically fix the other.

Slow response time? → Latency problem.
Server choking under load? → Throughput problem.

Always identify which one you are solving.
They need different solutions.
```

---

## Concept 3 — Availability vs Reliability

### The Core Confusion

> "If my system is available it must be reliable."

Wrong. A system can be **up and answering** while giving **completely wrong answers.**

---

### Availability — Defined

**Availability** is whether your system is **up and reachable** when a user tries to access it.

It is a binary question at any given moment: **is the system responding or not?**

#### Technical Definition
```
Availability = (Total time - Downtime) / Total time × 100%
             = Uptime / Total time × 100%
```

#### The 9s of Availability — Calculated

| Availability | Downtime per year | Downtime per month | Downtime per week |
|---|---|---|---|
| 90% (one 9) | 36.5 days | 72 hours | 16.8 hours |
| 99% (two 9s) | 3.65 days | 7.2 hours | 1.68 hours |
| 99.9% (three 9s) | 8.76 hours | 43.8 minutes | 10.1 minutes |
| 99.99% (four 9s) | 52.6 minutes | 4.38 minutes | 1.01 minutes |
| 99.999% (five 9s) | 5.26 minutes | 26.3 seconds | 6.05 seconds |

#### How To Calculate Availability

```
System with 3 components in SERIES (all must work):
  Availability = A1 × A2 × A3

Example:
  Web server: 99.9%
  App server: 99.9%
  Database:   99.9%

  System availability = 0.999 × 0.999 × 0.999
                      = 0.997 = 99.7%

  This is WORSE than any individual component.
  Each component added in series reduces availability.

System with components in PARALLEL (any can work):
  Availability = 1 - (1-A1) × (1-A2)

Example:
  Two databases, each 99.9%:
  = 1 - (0.001 × 0.001)
  = 1 - 0.000001
  = 99.9999%

  This is WHY redundancy improves availability.
```

#### What Causes Poor Availability

| Root Cause | Impact | Fix |
|---|---|---|
| Single server crashes | 100% downtime | Redundant servers |
| Bad deployment | Minutes-hours downtime | Blue-green deploy |
| Hardware failure | Hours downtime | Multi-AZ deployment |
| DDoS attack | Variable downtime | CDN, rate limiting |
| Database goes down | Full downtime | Read replicas, failover |
| No health checks | Slow detection | Health check endpoints |

---

### Reliability — Defined

**Reliability** is whether your system gives **correct and consistent results** every time it responds.

A system is reliable if: same input → same correct output, every time, under any conditions.

#### Technical Definition
```
Reliability = P(system functions correctly over time T)
            = e^(-λT)     [exponential distribution]

Where:
  λ = failure rate (failures per unit time)
  T = time period

MTBF = Mean Time Between Failures = 1/λ
MTTR = Mean Time To Recover
```

#### Reliability Metrics

| Metric | Formula | What it measures |
|---|---|---|
| MTBF | Total uptime / Number of failures | Average time between failures |
| MTTR | Total repair time / Number of failures | Average time to fix a failure |
| Error rate | Failed requests / Total requests × 100% | % of wrong responses |
| SLO | Target reliability percentage | e.g. 99.9% correct responses |

#### Real Technical Example — Payment API

```
SCENARIO:
Payment API is running. 99.99% availability.

AVAILABILITY CHECK:
Total requests in 1 month: 10,000,000
System responded: 9,999,000 times (99.99%)
System was DOWN: 1,000 requests failed completely
Availability: 99.99% ✓ GOOD

RELIABILITY CHECK:
Of the 9,999,000 responses:
  - Correct payment processed: 9,499,050 (95%)
  - Wrong amount charged: 499,950 (5%)
  - Duplicate charges: included in above

Reliability: 95% ✗ TERRIBLE

5% of 10,000,000 = 500,000 wrong transactions.
System was available. Not reliable.
This is a financial disaster.
```

#### What Causes Poor Reliability

| Root Cause | Wrong output type | Fix |
|---|---|---|
| Race condition | Inconsistent data | Locks, transactions |
| Bug in business logic | Wrong calculation | Better testing, code review |
| Bad deployment | Corrupted state | Staged rollout, feature flags |
| Data corruption | Garbage output | Data validation, checksums |
| Timeout without retry | Missing data | Retry with idempotency key |
| Floating point errors | Wrong amounts | Use decimal not float for money |

#### The Float Money Bug — A Reliability Killer

```javascript
// WRONG — floating point arithmetic
const price = 1.10;
const quantity = 3;
const total = price * quantity;
console.log(total); // 3.3000000000000003 ← WRONG

// RIGHT — use integers (store cents not dollars)
const priceInCents = 110;
const quantity = 3;
const totalInCents = priceInCents * quantity;
console.log(totalInCents); // 330 ← CORRECT
console.log(totalInCents / 100); // 3.30 ← display only
```

**This single bug has caused millions of dollars in wrong charges across the industry.**

---

### Availability vs Reliability — Direct Comparison

| Dimension | Availability | Reliability |
|---|---|---|
| Question it answers | Is the system UP? | Is the system CORRECT? |
| Measured by | Uptime % (99.9%, 99.99%) | Error rate %, MTBF |
| Failure mode | System doesn't respond | System responds wrongly |
| User experience | "The app is down" | "The app shows wrong data" |
| Fixed by | Redundancy, failover, load balancing | Testing, validation, transactions |
| Detected by | Health checks, uptime monitors | Error tracking, data audits |
| Example tool | PagerDuty, Pingdom | Sentry, DataDog error tracking |

#### The Four Combinations

| Availability | Reliability | Example | Impact |
|---|---|---|---|
| High | High | Google Search | Ideal — always up, always correct |
| High | Low | Buggy payment API | Dangerous — always up, wrong answers |
| Low | High | Scheduled maintenance window | Acceptable — down sometimes, correct when up |
| Low | Low | Legacy system in crisis | Critical failure |

---

### The Key Rule

```
Availability = the system responds.
Reliability = the system responds correctly.

A system can be:
  Available but not reliable (up but giving wrong answers)
  Reliable but not always available (correct but sometimes down)

You need both.
They need completely different solutions.

Availability fix: add redundancy, remove single points of failure.
Reliability fix: add testing, validation, transactions, monitoring.
```

---

## Concept 4 — Availability vs Consistency

### The Core Confusion

> "Can't I just have both availability and consistency?"

In a distributed system: **mathematically, no.**

This is not an engineering limitation. It is a **fundamental property of distributed systems** — proven by the CAP theorem (covered in the next post).

---

### Availability (in distributed context) — Defined

**Availability** means every request receives a response — even if some nodes in the system are failing or unreachable.

The system **never refuses** to answer. It always responds with something.

#### Technical Definition
```
Availability: For every request,
              the system returns a response
              (not an error) within a reasonable time.
```

---

### Consistency (in distributed context) — Defined

**Consistency** means every node in the distributed system returns the **most recent write** for any read.

All users see the **same data at the same time.**

#### Technical Definition
```
Consistency: Every read receives the most recent write
             or an error — never stale data.
```

---

### Why You Cannot Have Both — The Physics

```
DISTRIBUTED SYSTEM:
3 servers, data replicated across all 3.

Server A (US-East) ──── Server B (US-West)
         │                      │
         └──── Server C (EU) ───┘

User writes: "username = kriti" → hits Server A

Server A updates immediately.
Server A starts replicating to B and C.
Replication takes 150ms (speed of light, US to EU)

DURING THOSE 150ms, another user reads from Server C:

AVAILABILITY CHOICE:
Server C responds immediately with old data.
User gets a response. ✓ (available)
User sees old username. ✗ (not consistent)

CONSISTENCY CHOICE:
Server C waits for replication from A to complete.
User gets correct data. ✓ (consistent)
But what if the network between A and C breaks?
Server C must refuse the request or wait forever. ✗ (not available)

YOU CANNOT HAVE BOTH.
Network replication takes time.
During that time — available means stale, consistent means slow/unavailable.
```

---

### Real World — How Systems Choose

#### Instagram / Facebook — Choose Availability

```
Scenario:
You post a photo.
Write goes to Server A (US-East).
Your friend in EU reads from Server C.

What happens:
Your photo replicates from A → C in ~150ms.
During those 150ms:
  Your friend refreshes Instagram.
  Server C responds immediately (availability).
  Your friend sees your old profile, not the new photo yet.
  1 second later — replication complete.
  Your friend refreshes again — sees your photo.

This is "eventual consistency."
Available always. Consistent eventually.
Acceptable for social media.
Your friend not seeing your photo for 1 second is fine.
```

#### Banking Systems — Choose Consistency

```
Scenario:
You have $1,000 in your account.
You withdraw $800 from ATM in New York.
Write goes to Server A.
Replication to Server B (London) takes 150ms.

WHAT WOULD HAPPEN with availability choice:
During those 150ms:
  Your partner withdraws $800 from ATM in London.
  Server B responds immediately (available).
  Server B shows $1,000 (stale data — not replicated yet).
  Your partner gets $800.
  You got $800.
  Total withdrawn: $1,600 from $1,000 account.
  BANK LOSES $600.

WHAT HAPPENS with consistency choice:
Your withdrawal locks the account.
London ATM waits for replication.
  If network works → London sees $200, can only withdraw $200. ✓
  If network breaks → London ATM returns error: "Service unavailable." ✓
  No double spend. ✓
  Sometimes unavailable. ✓ (acceptable for money)
```

---

### Consistency Models — The Spectrum

Consistency is not binary. It exists on a spectrum:

```
STRONG ──────────────────────────────────────── WEAK
  │                    │                          │
Strong            Eventual                      Weak
Consistency      Consistency                Consistency
  │                    │                          │
Always           Eventually               No guarantee
correct           correct                  at all
  │                    │                          │
Slower             Faster                  Fastest
  │                    │                          │
Banks,           Social media,            DNS, Caches,
payments         Instagram, Twitter       Gaming scores
```

#### Strong Consistency
```
Every read returns the most recent write.
No stale data ever.
System may refuse request rather than return stale data.

Example: Bank balance, stock trades, inventory (last item)
Cost: Higher latency, lower availability
```

#### Eventual Consistency
```
All nodes will eventually have the same data.
Reads may return stale data temporarily.
System always responds (high availability).

Example: Instagram likes, Twitter followers count,
         Amazon product reviews, DNS records
Cost: Users may briefly see stale data
```

#### Weak Consistency
```
After a write, reads may or may not see it.
No guarantee when or if data will be consistent.
Maximum performance and availability.

Example: Live video stream (you don't need to
         see every frame if you missed it),
         multiplayer game positions,
         real-time analytics counters
Cost: Data may be permanently inconsistent
```

---

### Availability vs Consistency — Direct Comparison

| Dimension | Availability | Consistency |
|---|---|---|
| Question it answers | Does the system always respond? | Does every user see the same data? |
| Priority | Keep responding no matter what | Keep data correct no matter what |
| User experience | App never shows error | App always shows correct data |
| Failure mode | Returns stale data | Returns error or times out |
| Use case | Social media, DNS, caches | Banking, inventory, booking |
| Implementation | Multiple replicas respond independently | Distributed locks, consensus protocols |
| Protocol example | Cassandra, DynamoDB (AP) | PostgreSQL, Zookeeper (CP) |

---

### The Key Rule

```
Availability → always responds, may return stale data.
Consistency → always correct, may refuse to respond.

In a distributed system you cannot fully guarantee both
when a network partition occurs.

Choose based on what failure is acceptable:
  Wrong data acceptable? → Choose availability.
  No response acceptable? → Choose consistency.

For most systems: choose eventual consistency.
It gives you availability with acceptable staleness.
```

---

## How All 4 Concepts Connect

```
Every HLD decision you make traces back to one of these forces:

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Adding Redis cache?                                    │
│  → Solving LATENCY problem                             │
│                                                         │
│  Adding more servers behind load balancer?              │
│  → Solving SCALABILITY + THROUGHPUT problem            │
│                                                         │
│  Adding health checks + auto failover?                  │
│  → Solving AVAILABILITY problem                        │
│                                                         │
│  Adding data validation + transactions?                 │
│  → Solving RELIABILITY problem                         │
│                                                         │
│  Choosing Cassandra over PostgreSQL?                    │
│  → Choosing AVAILABILITY over CONSISTENCY              │
│                                                         │
│  Choosing eventual consistency for likes counter?       │
│  → Trading CONSISTENCY for AVAILABILITY + PERFORMANCE  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## The Master Comparison Table

| Concept | Question | Measured by | Fixed by | Trade-off |
|---|---|---|---|---|
| Performance | Is my code fast? | Response time, ms | Optimize code, index, cache | - |
| Scalability | Does fast hold at scale? | RPS, concurrent users | Shard, replicate, distribute | Cost, complexity |
| Latency | How long per request? | ms per request | CDN, cache, optimize | Throughput |
| Throughput | How many per second? | RPS, TPS | Add workers, queues, scale | Latency |
| Availability | Is the system up? | Uptime %, 9s | Redundancy, failover | Cost, consistency |
| Reliability | Is the response correct? | Error rate, MTBF | Testing, validation, monitoring | Development time |
| Availability (dist.) | Does it always respond? | Uptime in partitions | AP systems (Cassandra) | Consistency |
| Consistency (dist.) | Does everyone see same data? | Staleness window | CP systems (PostgreSQL) | Availability |

---

## Quick Reference — The 8 Numbers To Know

```
1. 99.9% availability    = 8.76 hours downtime per year
2. 99.99% availability   = 52.6 minutes downtime per year
3. L1 cache latency      = 0.5 nanoseconds
4. RAM latency           = 100 nanoseconds
5. SSD latency           = 150 microseconds
6. Same DC network       = 0.5 milliseconds
7. Cross-region (US-EU)  = ~150 milliseconds (physics limit)
8. HDD seek time         = 10 milliseconds
```

---

## What Comes Next

Now that you understand the 4 core confusions:

```
Next post → CAP Theorem
The formal mathematical proof that you cannot have
Consistency + Availability + Partition Tolerance
simultaneously in a distributed system.

Every database, every distributed system, every
architecture pattern is a response to CAP theorem.

Understanding CAP makes every HLD decision clear.
```

---

*All content original. Created by @kriti.sde*
