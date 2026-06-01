# System Design Maths — Complete Deep Dive
> Created by @kritirai.sde | All content original.
> Part of the UNDERSTANDING-SYSTEMS series.

---

## Prerequisites

Before reading this — make sure you understand:
- What a distributed system is
- What latency and throughput mean
- What availability means
- The 9s (99.9%, 99.99% etc.)

If not — read the Core Concepts README first.
`github.com/rai-kriti/UNDERSTANDING-SYSTEMS`

---

## Why Math Comes Before Design

Every system design interview starts the same way:

> "Design Instagram."
> "Design Twitter."
> "Design WhatsApp."

Most engineers immediately jump to drawing boxes.
Load balancer here. Database there. Cache somewhere.

**This is wrong.**

Before drawing a single box — you must answer:

```
How many users?
How many requests per second?
How much data per day?
How much storage after 5 years?
How much bandwidth per second?
How much memory for cache?
```

These numbers tell you **what architecture you need.**

Without them — you are guessing.
With them — every decision has a reason.

---

## Part 1 — The Numbers You Must Memorise

---

### Powers of 2 — Storage Units

Every storage calculation starts here.
You do not need to memorise exact values.
You need to know the **order of magnitude.**

#### The Units

| Unit | Symbol | Exact value | Approximate value |
|---|---|---|---|
| Bit | b | 0 or 1 | smallest unit |
| Byte | B | 8 bits | 1 character |
| Kilobyte | KB | 1,024 bytes | ~10³ bytes |
| Megabyte | MB | 1,024 KB | ~10⁶ bytes |
| Gigabyte | GB | 1,024 MB | ~10⁹ bytes |
| Terabyte | TB | 1,024 GB | ~10¹² bytes |
| Petabyte | PB | 1,024 TB | ~10¹⁵ bytes |
| Exabyte | EB | 1,024 PB | ~10¹⁸ bytes |
| Zettabyte | ZB | 1,024 EB | ~10²¹ bytes |

#### In System Design We Simplify

```
In system design interviews — use approximate values.
Do not waste time on exact conversions.

1 KB ≈ 1,000 bytes     (actually 1,024)
1 MB ≈ 1,000 KB        (actually 1,024)
1 GB ≈ 1,000 MB        (actually 1,024)
1 TB ≈ 1,000 GB        (actually 1,024)
1 PB ≈ 1,000 TB        (actually 1,024)

The 2.4% error from rounding does not matter
in back of envelope estimation.
Being in the right order of magnitude matters.
```

#### Real World Reference — What Takes How Much Space

| Content | Size | Notes |
|---|---|---|
| 1 ASCII character | 1 Byte | a, b, c, 1, 2, 3 |
| 1 Unicode character | 2-4 Bytes | emoji, Chinese characters |
| 1 tweet | ~280 Bytes | max 280 characters |
| 1 SMS | ~160 Bytes | max 160 characters |
| 1 UUID | 16 Bytes | unique identifier |
| 1 integer | 4 Bytes | 32-bit int |
| 1 long integer | 8 Bytes | 64-bit long |
| 1 timestamp | 8 Bytes | Unix timestamp |
| 1 row in SQL table | ~100-1000 Bytes | depends on columns |
| 1 webpage (HTML) | ~2 MB | without images |
| 1 photo (uncompressed) | ~5-10 MB | raw camera photo |
| 1 photo (JPEG compressed) | ~3-5 MB | typical Instagram photo |
| 1 photo (thumbnail 200px) | ~200 KB | compressed thumbnail |
| 1 minute audio (MP3) | ~1 MB | 128kbps |
| 1 minute video (480p) | ~15 MB | compressed |
| 1 minute video (720p) | ~50 MB | compressed |
| 1 minute video (1080p) | ~100 MB | compressed |
| 1 minute video (4K) | ~375 MB | compressed |

#### Why Powers of 2 Matter — Example

```
Question: How much storage does Instagram need per day?

Given:
500 million DAU
Each user uploads 2 photos/day
Average photo = 3 MB

Calculation:
500,000,000 users
× 2 photos/user/day
× 3 MB/photo
= 3,000,000,000 MB/day
= 3,000,000 GB/day
= 3,000 TB/day
= 3 PB/day

Without knowing these units —
you cannot even state the answer.
With them — calculation takes 30 seconds.
```

---

### Latency Numbers Every Engineer Must Know

Latency numbers tell you **where time is being spent** in your system.
They guide every caching, database, and network decision you make.

#### The Core Numbers

| Operation | Latency | Notes |
|---|---|---|
| L1 cache read | 0.5 ns | Fastest possible — CPU cache |
| Branch misprediction | 5 ns | CPU pipeline flush |
| L2 cache read | 7 ns | Still CPU cache |
| Mutex lock/unlock | 25 ns | Thread synchronisation |
| RAM read | 100 ns | Main memory |
| Compress 1KB (Zippy) | 10,000 ns = 10 µs | In-memory compression |
| Send 1KB over 1Gbps network | 10,000 ns = 10 µs | LAN transfer |
| SSD random read | 150,000 ns = 150 µs | NVMe SSD |
| Read 1MB sequentially from RAM | 250,000 ns = 250 µs | Sequential RAM read |
| Round trip within datacenter | 500,000 ns = 0.5 ms | Same DC ping |
| Read 1MB from SSD | 1,000,000 ns = 1 ms | Sequential SSD read |
| HDD seek time | 10,000,000 ns = 10 ms | Mechanical disk |
| Read 1MB from HDD | 20,000,000 ns = 20 ms | Sequential HDD read |
| Send packet US → EU | 150,000,000 ns = 150 ms | Cross-region network |
| DNS lookup | 20-120 ms | Domain resolution |
| TCP handshake | 50-200 ms | Connection setup |
| TLS handshake | 100-300 ms | Secure connection |
| Redis GET | 0.5 ms | In-memory database |
| PostgreSQL indexed query | 5-10 ms | With good indexing |
| PostgreSQL unindexed query | 100ms-10s | Full table scan |

#### How to Read This Table

```
ns  = nanosecond  = 0.000000001 seconds = 10⁻⁹ seconds
µs  = microsecond = 0.000001 seconds    = 10⁻⁶ seconds
ms  = millisecond = 0.001 seconds       = 10⁻³ seconds

Conversions:
1 ms = 1,000 µs = 1,000,000 ns

In plain terms:
1 millisecond is to 1 second
what 1 second is to 16.7 minutes
```

#### The Key Comparisons

```
RAM vs SSD:
RAM read:  100 ns
SSD read:  150,000 ns
RAM is 1,500x faster than SSD

SSD vs HDD:
SSD read:  150,000 ns
HDD read:  10,000,000 ns
SSD is 67x faster than HDD

Same datacenter vs Cross-region:
Same DC:      0.5 ms
Cross-region: 150 ms
Same DC is 300x faster than cross-region

Redis vs PostgreSQL:
Redis GET:     0.5 ms
Postgres query: 5-10 ms
Redis is 10-20x faster than Postgres
```

#### Why Latency Numbers Matter for Design Decisions

```
Decision 1: Should I add a cache?

Without cache:
Every request → PostgreSQL → 10ms
1,000 requests/sec × 10ms = system at 100% load

With Redis cache:
Cache hit → Redis → 0.5ms (20x faster)
Cache miss → PostgreSQL → 10ms
80% cache hit rate:
800 req × 0.5ms + 200 req × 10ms = much lower load

Decision: YES add cache. Always.

─────────────────────────────────────────

Decision 2: Should I put my database in another region?

Without cross-region awareness:
"I will put database in EU for European users"

Reality check:
Every US user request → US server → EU database
Round trip: 150ms added per query
1,000 queries per page load × 150ms = 150 seconds

Decision: NO. Keep database in same region as servers.
Use read replicas in each region instead.

─────────────────────────────────────────

Decision 3: Should I store user sessions in memory or disk?

Session lookup on HDD: 10ms
Session lookup in Redis: 0.5ms

Every page load checks session.
1 million users × 10ms vs 0.5ms
= 10,000 seconds vs 500 seconds total latency saved

Decision: YES store sessions in Redis.
```

#### The Physics Limit — Speed of Light

```
No engineering can beat physics.

Speed of light in fiber optic cable:
~200,000 km/second

Distance New York to London: 5,570 km

Minimum theoretical one-way latency:
5,570 / 200,000 = 0.028 seconds = 28ms

Round trip minimum: ~56ms
Real world (with routing, switches): 80-150ms

This is why:
- CDNs exist (serve from closest location)
- Edge computing exists (process near user)
- Regional databases exist (data near users)

You cannot make New York-London faster than 80ms.
Physics says no.
```

---

### The 9s of Availability

When a system claims "99.99% availability" — what does that actually mean in real downtime?

#### How to Calculate Downtime

```
Total minutes in a year:
365 days × 24 hours × 60 minutes = 525,600 minutes/year

Total seconds in a year:
525,600 × 60 = 31,536,000 seconds/year

Downtime = (1 - availability) × total time

Example for 99.99%:
Downtime = (1 - 0.9999) × 525,600
         = 0.0001 × 525,600
         = 52.56 minutes per year
```

#### The Complete 9s Table

| Availability | Downtime/year | Downtime/month | Downtime/week | Downtime/day |
|---|---|---|---|---|
| 90% (one 9) | 36.5 days | 72 hours | 16.8 hours | 2.4 hours |
| 99% (two 9s) | 3.65 days | 7.2 hours | 1.68 hours | 14.4 minutes |
| 99.9% (three 9s) | 8.76 hours | 43.8 minutes | 10.1 minutes | 1.44 minutes |
| 99.99% (four 9s) | 52.6 minutes | 4.38 minutes | 1.01 minutes | 8.64 seconds |
| 99.999% (five 9s) | 5.26 minutes | 26.3 seconds | 6.05 seconds | 0.864 seconds |
| 99.9999% (six 9s) | 31.5 seconds | 2.63 seconds | 0.605 seconds | 86.4 ms |

#### What Each Level Means in Practice

```
90% availability:
36.5 days of downtime per year.
Completely unacceptable for any production system.
Only acceptable during early development.

99% availability:
3.65 days of downtime per year.
Unacceptable for most products.
Users will leave.

99.9% availability (three 9s):
8.76 hours of downtime per year.
Minimum acceptable for most startups.
~43 minutes downtime per month.
Most users tolerate this.

99.99% availability (four 9s):
52.6 minutes of downtime per year.
Standard target for mature products.
~4 minutes downtime per month.
Most SLAs target this.

99.999% availability (five 9s):
5.26 minutes of downtime per year.
Required for: banking, healthcare, infrastructure.
Extremely expensive to achieve.
Requires massive redundancy.

99.9999% availability (six 9s):
31.5 seconds of downtime per year.
Almost never required.
Only for: air traffic control, nuclear systems,
          life-critical infrastructure.
```

#### The Cost of Each Additional 9

```
Going from 99% to 99.9%:
10x reduction in downtime.
But requires: redundant servers, load balancing,
             health checks, auto-failover.
Cost increase: ~2-3x infrastructure cost.

Going from 99.9% to 99.99%:
Another 10x reduction in downtime.
But requires: multi-AZ deployment, database replication,
             zero-downtime deployments, chaos engineering.
Cost increase: ~3-5x infrastructure cost.

Going from 99.99% to 99.999%:
Another 10x reduction in downtime.
But requires: multi-region deployment, global load balancing,
             hot standby systems, dedicated SRE team.
Cost increase: ~10x infrastructure cost.

Rule of thumb:
Each additional 9 costs 10x more than the previous.
Match your availability target to your business need.
A blog does not need five 9s.
A hospital system does.
```

#### How Availability Compounds in Systems

```
When components are in SERIES (all must work):
Overall availability = A1 × A2 × A3 × ... × An

Example:
Web server:  99.9%
App server:  99.9%
Database:    99.9%
Cache:       99.9%

Overall = 0.999 × 0.999 × 0.999 × 0.999
        = 0.996 = 99.6%

WORSE than any individual component!
Every component added in series reduces availability.
This is why redundancy is needed.

─────────────────────────────────────────

When components are in PARALLEL (any can work):
Overall availability = 1 - (1-A1) × (1-A2) × ... × (1-An)

Example:
Two databases, each 99.9%:
= 1 - (1-0.999) × (1-0.999)
= 1 - (0.001 × 0.001)
= 1 - 0.000001
= 0.999999 = 99.9999%

BETTER than any individual component!
This is why redundancy (replication, failover) works.
Adding parallel components dramatically improves availability.
```

---

## Part 2 — How to Estimate Any System

---

### The 4 Estimations Framework

Every system design estimation follows this order:

```
1. Define scale (assumptions)
   ↓
2. Traffic estimation (RPS)
   ↓
3. Storage estimation (bytes)
   ↓
4. Memory estimation (cache)
   ↓
5. Bandwidth estimation (GB/sec)
   ↓
6. Decisions from numbers
```

---

### The Key Number to Memorise

```
1 day = 86,400 seconds

In system design — round to 100,000 seconds/day
(only 16% error, acceptable for estimation)

This simplifies calculations enormously:

1 billion requests/day ÷ 100,000 = 10,000 RPS
1 million requests/day ÷ 100,000 = 10 RPS
```

---

### Step 1 — Define Scale (Assumptions)

Before any calculation — state your assumptions.

In an interview this shows structured thinking.
Interviewers give you credit for this.

```
Always state:
- Total registered users
- Daily Active Users (DAU)
- Monthly Active Users (MAU)
- Key user behaviours (posts/day, views/day, messages/day)
- Content sizes (photo size, video size, message size)
- Data retention period
- Availability target
- Read/Write ratio (if known)

Example template:
"I will assume:
- 500M DAU
- Each user uploads 2 photos per day
- Each user views 100 photos per day
- Average photo size is 3MB
- Data retained forever
- Target availability: 99.99%
Is that reasonable?"

Always ask if your assumptions are reasonable.
Interviewer may correct you.
That is fine — shows collaborative thinking.
```

---

### Step 2 — Traffic Estimation

Traffic tells you how many requests your system handles per second.

#### The Formula

```
Requests Per Second (RPS) = DAU × requests per user per day
                                  ÷ 86,400 seconds

Always calculate reads and writes separately.
```

#### Read vs Write Traffic

```
Read traffic:
DAU × reads per user per day ÷ 86,400 = Read RPS

Write traffic:
DAU × writes per user per day ÷ 86,400 = Write RPS

Read/Write ratio = Read RPS : Write RPS
```

#### Common Read/Write Ratios

| System | Read/Write ratio | Reason |
|---|---|---|
| Twitter | 100:1 | Millions read each tweet, few write |
| Instagram | 50:1 | Users browse more than upload |
| WhatsApp | 1:1 | Every message sent is read once |
| Google Search | 1000:1 | Billions of searches, few index updates |
| Stock trading | 10:1 | Many price checks, fewer trades |
| Email | 5:1 | Read inbox more than send |

#### What Read/Write Ratio Tells You

```
Read heavy (>10:1):
→ Optimise for reads
→ Add read replicas
→ Cache aggressively
→ CDN for static content
→ Denormalise data for fast reads

Write heavy (<5:1):
→ Optimise for writes
→ Message queues to buffer writes
→ Batch writes together
→ Write-optimised databases (Cassandra)
→ Async processing

Balanced (1:1 to 5:1):
→ Balance both
→ Consider CQRS pattern
   (separate read and write models)
```

#### Traffic Estimation Example — Twitter

```
Assumptions:
DAU: 300 million
Each user reads 100 tweets/day
Each user writes 1 tweet/day

Read RPS:
300,000,000 × 100 ÷ 86,400
= 30,000,000,000 ÷ 86,400
= 347,222 reads/second
≈ 350,000 reads/second

Write RPS:
300,000,000 × 1 ÷ 86,400
= 300,000,000 ÷ 86,400
= 3,472 writes/second
≈ 3,500 writes/second

Read/Write ratio:
350,000 : 3,500 = 100:1

Decisions:
→ Heavily read optimised
→ Cache timelines aggressively
→ Read replicas for database
→ CDN for media
→ Write path and read path separate services
```

---

### Step 3 — Storage Estimation

Storage tells you how much data your system accumulates over time.

#### The Formula

```
Daily storage = data size per object
              × objects created per day

Total storage = daily storage
              × retention period (days)
              × replication factor (usually 3)
```

#### Replication Factor

```
Why multiply by 3?

In distributed systems — data is replicated
across multiple nodes for durability.

Standard replication factor: 3
(one primary + two replicas)

If one server fails → two copies remain.
If two servers fail → one copy remains.
No data lost.

Always account for replication in storage estimates.
Raw storage × 3 = actual storage needed.
```

#### Storage Estimation Example — WhatsApp Messages

```
Assumptions:
DAU: 2 billion
Each user sends 40 messages/day
Average message size: 100 bytes
Media messages: 10% of messages
Average media size: 500 KB
Retention: 5 years

Text message storage:
2,000,000,000 × 40 × 90% × 100 bytes
= 7,200,000,000,000 bytes/day
= 7,200 GB/day
= 7.2 TB/day (text only)

Media message storage:
2,000,000,000 × 40 × 10% × 500,000 bytes
= 4,000,000,000,000,000 bytes/day
= 4,000,000 GB/day
= 4 PB/day (media)

Total daily storage:
7.2 TB + 4,000 TB ≈ 4,007 TB/day ≈ 4 PB/day

5 year total:
4 PB × 365 × 5 = 7,300 PB = 7.3 Exabytes

With replication (×3):
7.3 EB × 3 = 21.9 Exabytes over 5 years

Decisions:
→ Distributed object storage mandatory
→ Media is 99.8% of storage problem
→ Text messages relatively cheap
→ Compress media aggressively
→ Cold storage for old messages (cheaper HDD)
→ Hot storage for recent messages (fast SSD)
```

#### Tiered Storage Strategy

```
Not all data is accessed equally.
Store based on access frequency.

HOT storage (SSD — fast, expensive):
Data accessed in last 30 days.
~20% of total data.
~80% of all accesses.

WARM storage (HDD — medium speed, medium cost):
Data accessed in last 1 year.
~30% of total data.
~15% of all accesses.

COLD storage (Tape/Glacier — slow, very cheap):
Data older than 1 year.
~50% of total data.
~5% of all accesses.

Cost savings: Cold storage is 10-100x cheaper than SSD.
Trade-off: Retrieval from cold storage takes minutes-hours.
Acceptable for: archived photos, old messages, audit logs.
```

---

### Step 4 — Memory Estimation

Memory estimation tells you how much RAM your caching layer needs.

#### The 80/20 Rule of Caching

```
In almost every system:
80% of all requests access
only 20% of the data.

This is called the Pareto Principle
applied to data access patterns.

Real examples:
- Twitter: 0.1% of tweets get 80% of views
- YouTube: top 20% of videos = 80% of watches
- Amazon: top 20% of products = 80% of sales
- Instagram: celebrity accounts = majority of views

Implication:
Cache only the top 20% of data.
Serve 80% of requests from fast RAM.
Massively reduce database load.
```

#### The Formula

```
Cache size = total daily active data × 20%

Or more precisely:
Cache size = daily active objects
           × object size
           × cache hit ratio target
```

#### Cache Hit Ratio

```
Cache hit ratio = % of requests served from cache
                  (not going to database)

Target: 80-95% cache hit ratio

Example:
100,000 RPS total
80% hit ratio = 80,000 requests served from cache
20% miss ratio = 20,000 requests hit database

Without cache: 100,000 DB queries/second (impossible)
With cache: 20,000 DB queries/second (manageable)

The cache absorbs 80% of load from the database.
```

#### Memory Estimation Example — YouTube

```
Assumptions:
2 billion DAU
Each user watches 5 videos/day
Average video metadata: 1 KB
Average video thumbnail: 200 KB
Popular videos (top 20%): 400 million

Metadata cache:
400 million popular videos × 1 KB metadata
= 400,000,000 KB
= 400,000 MB
= 400 GB

Thumbnail cache:
400 million popular videos × 200 KB thumbnail
= 80,000,000,000 KB
= 80,000,000 MB
= 80,000 GB
= 80 TB

Total cache: ~80 TB (dominated by thumbnails)

Decisions:
→ Distributed Redis cluster (multiple nodes)
→ Each Redis node: up to 100 GB RAM
→ Need ~800+ Redis nodes for thumbnails
→ CDN better than Redis for thumbnails
   (CDN serves at edge, closer to user)
→ Redis for metadata (~400 GB = 4-5 nodes)
→ CDN for thumbnails and video chunks
```

#### Cache Eviction Policies

```
When cache is full — which data to remove?

LRU (Least Recently Used):
Remove data not accessed recently.
Best for: general purpose caching.
Used by: Redis (default), Memcached.

LFU (Least Frequently Used):
Remove data accessed least often overall.
Best for: data with stable popularity patterns.
Used by: some Redis configurations.

TTL (Time To Live):
Remove data after a fixed time period.
Best for: data that becomes stale (news, prices).
Example: Cache Twitter trends for 60 seconds.
         After 60s → fetch fresh from DB.

FIFO (First In First Out):
Remove oldest cached data.
Simplest. Rarely optimal.
Best for: streaming data, logs.
```

---

### Step 5 — Bandwidth Estimation

Bandwidth tells you how much data flows in and out of your system per second.

#### Two Directions Always

```
Ingress = data flowing INTO your system
          (uploads, writes, incoming requests)

Egress  = data flowing OUT of your system
          (downloads, reads, responses, streaming)

Egress is almost always larger than ingress.
Plan for both.
Bill for both (cloud providers charge for egress).
```

#### The Formula

```
Ingress bandwidth = Write RPS × average write size

Egress bandwidth  = Read RPS × average response size
```

#### Bandwidth Estimation Example — Netflix

```
Assumptions:
DAU: 200 million
Each user watches 2 hours of video/day
Average video bitrate: 5 Mbps (1080p streaming)
Peak hours: 3x average load (evenings)

Average egress:
200,000,000 users × 2 hours × 3,600 seconds
= 1,440,000,000,000 seconds of video/day

Per second egress:
200,000,000 users × 5 Mbps ÷ 8 (bits to bytes)
= 200,000,000 × 0.625 MB/sec
= 125,000,000 MB/sec
= 125,000 GB/sec
= 125 TB/sec average

Peak egress (3x):
125 TB/sec × 3 = 375 TB/sec peak

In context:
Netflix reportedly accounts for
~15% of global internet traffic.
Total global internet traffic ~600 TB/sec.
Netflix at peak: ~90 TB/sec.
(Our estimate is in the right order of magnitude)

Decisions:
→ CDN is not optional — it is the entire delivery strategy
→ Netflix uses its own CDN: Open Connect
→ Places servers INSIDE ISP networks
→ Pre-positions popular content during off-peak
→ Adaptive bitrate streaming
   (reduces quality when bandwidth is limited)
→ Multiple encoding formats per video
   (different resolutions, bitrates, codecs)
```

#### Why Bandwidth Matters for Architecture

```
High egress → CDN mandatory

Without CDN:
All video streamed from Netflix origin servers.
375 TB/sec from one location.
Impossible. Network would collapse.

With CDN:
Video cached on servers worldwide.
User in Mumbai gets video from CDN in Mumbai.
User in New York gets video from CDN in New York.
Origin server load: dramatically reduced.
User latency: dramatically reduced.

CDN is not a nice-to-have.
At Netflix/YouTube/Instagram scale —
CDN IS the delivery infrastructure.
```

---

## Part 3 — Full Estimation Walkthrough

---

### Estimating Instagram at Scale

Let's put everything together.

#### Step 1 — Define Scale

```
Assumptions (state these in interview):

Users:
Total registered: 1 billion
Daily Active Users (DAU): 500 million
Monthly Active Users (MAU): 800 million

User behaviour:
Photo uploads per user per day: 2
Photo views per user per day: 100
Story views per user per day: 50
DMs sent per user per day: 5
Search queries per user per day: 2

Content sizes:
Raw photo: 3 MB
Compressed photo (JPEG): 2 MB
Thumbnail (200px): 200 KB
Story (compressed): 5 MB
DM text: 1 KB
Photo metadata: 500 bytes

System requirements:
Read/Write ratio: ~50:1 (heavily read)
Data retention: Forever (photos, stories 24h)
Availability target: 99.99%
Consistency: Eventual (social data), Strong (auth/payments)
```

#### Step 2 — Traffic Estimation

```
WRITE TRAFFIC:

Photo uploads:
500,000,000 × 2 photos/day ÷ 86,400
= 1,000,000,000 ÷ 86,400
= 11,574 uploads/second
≈ 12,000 photo writes/second

DM messages:
500,000,000 × 5 DMs/day ÷ 86,400
= 2,500,000,000 ÷ 86,400
= 28,935 DMs/second
≈ 29,000 DM writes/second

Total write RPS:
12,000 + 29,000 = 41,000 writes/second

─────────────────────────────────────────

READ TRAFFIC:

Photo views:
500,000,000 × 100 views/day ÷ 86,400
= 50,000,000,000 ÷ 86,400
= 578,703 photo views/second
≈ 580,000 photo reads/second

Story views:
500,000,000 × 50 views/day ÷ 86,400
= 25,000,000,000 ÷ 86,400
= 289,351 story views/second
≈ 290,000 story reads/second

Total read RPS:
580,000 + 290,000 = 870,000 reads/second

─────────────────────────────────────────

Read/Write ratio:
870,000 : 41,000 ≈ 21:1

Decision: Heavily read optimised system.
Cache aggressively. Read replicas essential.
```

#### Step 3 — Storage Estimation

```
PHOTO STORAGE:

Daily upload volume:
12,000 uploads/second × 86,400 seconds
= 1,036,800,000 photos/day
≈ 1 billion photos/day

Daily raw storage:
1,000,000,000 × 3 MB
= 3,000,000,000 MB/day
= 3,000,000 GB/day
= 3,000 TB/day
= 3 PB/day

With replication (×3):
3 PB × 3 = 9 PB/day actual

5 year total:
9 PB × 365 × 5 = 16,425 PB ≈ 16 Exabytes

─────────────────────────────────────────

THUMBNAIL STORAGE:

Each photo → 1 thumbnail generated
1 billion thumbnails/day × 200 KB
= 200,000,000,000 KB/day
= 200,000,000 MB/day
= 200,000 GB/day
= 200 TB/day thumbnails

─────────────────────────────────────────

METADATA STORAGE:

1 billion photos/day × 500 bytes metadata
= 500,000,000,000 bytes/day
= 500 GB/day metadata

5 year metadata:
500 GB × 365 × 5 = 912,500 GB ≈ 912 TB

─────────────────────────────────────────

STORY STORAGE:

Stories expire after 24 hours.
Maximum concurrent stories:
500,000,000 users × 2 stories/user × 5 MB
= 5,000,000,000 MB active at any time
= 5,000,000 GB
= 5 PB of active story data

─────────────────────────────────────────

DECISIONS FROM STORAGE NUMBERS:

→ AWS S3 or equivalent object storage mandatory
→ Single server cannot hold this
→ Tiered storage:
   Recent photos (30 days) → SSD (fast access)
   Older photos → HDD (cheaper)
   Very old photos → Glacier (cheapest)
→ CDN caches thumbnails (avoids hitting S3 for every view)
→ Metadata in distributed database (Cassandra)
→ Stories in separate service (different retention policy)
```

#### Step 4 — Memory Estimation

```
PHOTO FEED CACHE:

80/20 rule: top 20% of photos get 80% of views

Active photos (last 7 days):
1 billion/day × 7 days = 7 billion photos
Top 20%: 1.4 billion photos

Cache metadata per photo: 500 bytes
1,400,000,000 × 500 bytes
= 700,000,000,000 bytes
= 700,000 MB
= 700 GB metadata cache

Cache thumbnails for hot photos:
Top 1% of photos (most viral): 14 million photos
14,000,000 × 200 KB
= 2,800,000,000 KB
= 2,800,000 MB
= 2,800 GB ≈ 3 TB thumbnail cache

Total cache:
700 GB metadata + 3 TB thumbnails ≈ 3.7 TB

─────────────────────────────────────────

DECISIONS FROM MEMORY NUMBERS:

→ 3.7 TB fits in a Redis cluster
→ ~37 Redis nodes at 100 GB each
→ Consistent hashing to distribute keys
→ LRU eviction for thumbnails
→ TTL: 24 hours for feed cache
→ CDN serves thumbnails at edge
   (CDN cache >> Redis cache for media)
```

#### Step 5 — Bandwidth Estimation

```
INGRESS (data coming in):

Photo uploads:
12,000 uploads/sec × 3 MB/upload
= 36,000 MB/second
= 36 GB/second incoming

DM messages:
29,000 DMs/sec × 1 KB/DM
= 29,000 KB/second
= 29 MB/second (negligible vs photos)

Total ingress: ~36 GB/second

─────────────────────────────────────────

EGRESS (data going out):

Photo views (thumbnails):
580,000 views/sec × 200 KB/thumbnail
= 116,000,000 KB/second
= 116,000 MB/second
= 116 GB/second

Story views:
290,000 views/sec × 200 KB/story thumbnail
= 58,000,000 KB/second
= 58 GB/second

Total egress:
116 + 58 = 174 GB/second outgoing

─────────────────────────────────────────

DECISIONS FROM BANDWIDTH NUMBERS:

→ 174 GB/second outgoing is enormous
→ CDN is absolutely mandatory
→ Serve thumbnails from CDN edge (not origin)
→ Progressive loading: blur-up → full thumbnail
→ Full photo (3 MB) only loaded on tap
→ Video content via adaptive bitrate streaming
→ Image compression pipeline on upload
   (HEIC/WebP instead of JPEG for 30-50% size reduction)
```

#### Step 6 — The Summary and Architecture Decisions

```
INSTAGRAM SCALE SUMMARY:

Traffic:
Write RPS:   41,000/second
Read RPS:    870,000/second
Read/Write:  21:1

Storage:
Photos/day:  3 PB raw, 9 PB with replication
5 year total: ~16 Exabytes
Metadata:    ~912 TB over 5 years

Memory:
Cache size:  ~3.7 TB (Redis cluster)
CDN cache:   Petabytes at edge globally

Bandwidth:
Ingress:     ~36 GB/second
Egress:      ~174 GB/second

Availability target: 99.99% (52 min downtime/year)

─────────────────────────────────────────

ARCHITECTURE DECISIONS FORCED BY NUMBERS:

Object storage:
→ AWS S3 (or Instagram's own distributed storage)
→ Cannot use a single server or traditional NAS

Photo service:
→ Separate microservice for upload
→ Async thumbnail generation (message queue)
→ Multiple thumbnail sizes generated on upload

Database:
→ User profiles: PostgreSQL (strong consistency)
→ Social graph (followers): Cassandra (AP, eventual)
→ Feed data: Cassandra (AP, high read throughput)
→ Auth/payments: PostgreSQL (CP, strong consistency)

Cache:
→ Redis cluster for feed metadata
→ CDN (Cloudfront/Fastly) for thumbnails globally
→ In-memory cache on app servers for hot data

Message queue:
→ Kafka for async thumbnail generation
→ Kafka for feed fanout (celebrity posts)
→ Kafka for notification delivery

CDN:
→ Mandatory for 174 GB/sec egress
→ Cache thumbnails at edge in every region
→ Reduce origin server load by 90%+

Availability:
→ Multi-AZ deployment (3 availability zones)
→ Auto-scaling based on RPS
→ Circuit breakers between services
→ Health checks every 10 seconds
→ Automated failover < 60 seconds
```

---

## Quick Reference Cheat Sheet

### The Numbers to Know

```
Storage:
1 KB = 1,000 bytes
1 MB = 1,000 KB
1 GB = 1,000 MB
1 TB = 1,000 GB
1 PB = 1,000 TB

Time:
1 day = 86,400 seconds ≈ 100,000 seconds
1 month = 30 days = 2,592,000 seconds ≈ 3,000,000 seconds
1 year = 365 days = 31,536,000 seconds ≈ 30,000,000 seconds

Latency order:
RAM (100ns) << SSD (150µs) << HDD (10ms) << Network (0.5ms-150ms)

The 9s:
99.9%   = 8.76 hours/year downtime
99.99%  = 52.6 minutes/year downtime
99.999% = 5.26 minutes/year downtime
```

### The Estimation Framework

```
1. State assumptions (DAU, behaviour, sizes)
2. Traffic = DAU × requests/day ÷ 86,400
3. Storage = objects/day × size × retention × 3
4. Memory = active data × 20% (80/20 rule)
5. Bandwidth = RPS × response size
6. Let numbers drive architecture decisions
```

### Common Content Sizes

```
Text message: 100 bytes
Tweet: 280 bytes
SQL row: 100-1,000 bytes
Photo (compressed): 3-5 MB
Thumbnail: 200 KB
Audio (1 min): 1 MB
Video 720p (1 min): 50 MB
Video 1080p (1 min): 100 MB
```

### Common System Scales

```
Small startup: 10,000 DAU → ~1 RPS
Growing startup: 1M DAU → ~100 RPS
Mid-size product: 10M DAU → ~1,000 RPS
Large product: 100M DAU → ~10,000 RPS
FAANG scale: 1B DAU → ~100,000+ RPS
```

---

## Common Mistakes in Estimation

### Mistake 1 — Forgetting Replication

```
Wrong:
"We need 3 PB of storage per day."

Right:
"We need 3 PB raw storage.
With 3x replication: 9 PB actual storage needed."

Always multiply raw storage by replication factor (3).
```

### Mistake 2 — Not Separating Reads and Writes

```
Wrong:
"We need to handle 600,000 RPS."

Right:
"We have 580,000 read RPS and 20,000 write RPS.
Read/write ratio is 29:1.
We need to optimise heavily for reads."

Different problems. Different solutions.
Never combine them.
```

### Mistake 3 — Ignoring Peak Traffic

```
Average traffic vs Peak traffic:

Average: 580,000 read RPS
Peak (during events): 3-5x average
Peak RPS: 1,740,000 - 2,900,000 RPS

Always design for peak, not average.
Super Bowl, Black Friday, breaking news
= traffic spikes 5-10x normal.
If your system handles average only —
it fails exactly when it matters most.
```

### Mistake 4 — Forgetting Metadata

```
Engineers estimate content storage.
They forget metadata.

Instagram photo: 3 MB (easy to estimate)
Photo metadata: 500 bytes (easy to forget)

At 1 billion photos/day:
Content: 3 PB/day (obvious)
Metadata: 500 GB/day (forgotten)

Over 5 years:
Metadata alone: ~912 TB

Metadata needs a different database than content.
Content → object storage (S3)
Metadata → distributed database (Cassandra)
Forgetting metadata = wrong database choice.
```

### Mistake 5 — Exact Numbers

```
Wrong approach:
"We need exactly 3,472.22 writes per second."

Right approach:
"We need approximately 3,500 writes per second."

Back of envelope = rough estimate.
Order of magnitude = what matters.

3,500 vs 35,000 = completely different systems.
3,500 vs 3,472 = the same system.

Round aggressively. State assumptions.
Show reasoning. Numbers will be off.
That is acceptable and expected.
```

---

## What Comes Next

```
You now know:
✓ Powers of 2 — storage units and real world sizes
✓ Latency numbers — from L1 cache to cross-region
✓ The 9s — what each availability level actually means
✓ Traffic estimation — RPS calculation
✓ Storage estimation — with replication
✓ Memory estimation — 80/20 rule
✓ Bandwidth estimation — ingress and egress
✓ Full system estimation — Instagram walkthrough

Next → Availability Patterns
How systems actually achieve 99.99% availability.
Fail-over patterns (Active-Passive, Active-Active).
Replication strategies.
The engineering behind the 9s.
```

---

*All content original. Created by @kriti.sde*
*github.com/rai-kriti/UNDERSTANDING-SYSTEMS*
