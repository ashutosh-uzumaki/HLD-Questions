# URL Shortener — Step 5: Database Design

**Problem:** Design a URL shortener like bit.ly / tinyurl
**Step:** 5 of 8 (Database Design)
**Teaching style:** Every decision → all options with tradeoffs → pick with reasoning

---

## 1. Database Design Philosophy

**Step 5's job:** Pick a DB and design the data layer to support our access patterns.

Covers:
- SQL vs NoSQL decision (with reasoning, not buzzwords)
- Specific DB choice
- Schema design
- Sharding / partitioning strategy
- Replication strategy
- Capacity planning

**Golden rule:** Every DB decision must trace back to Step 1 NFRs or Step 2 numbers.

---

## 2. Decision 1: SQL vs NoSQL

### The 5-question framework

```
Q1: What's the access pattern?
   Key-value lookup?    → NoSQL advantage
   Complex joins?       → SQL advantage
   Aggregations?        → SQL or OLAP

Q2: Is schema stable?
   Stable?              → SQL
   Evolving?            → NoSQL

Q3: Transactions needed?
   Multi-row ACID?      → SQL (or NewSQL)
   Single-row OK?       → NoSQL

Q4: Scale?
   <1 TB, <10K QPS?     → SQL fits
   >100 TB or >100K QPS? → NoSQL mandatory

Q5: Consistency?
   Strong always?       → SQL
   Eventual OK?         → NoSQL has more options
```

### Applied to URL Shortener

| Question | URL Shortener Answer | Leaning |
|----------|---------------------|---------|
| Access pattern | Pure key-value (short_url → long_url) | NoSQL |
| Schema stable | Yes, unchanging | Either |
| Transactions | No | NoSQL |
| Scale | 4.5 TB, 2K QPS now, growth expected | NoSQL |
| Consistency | Single-row, eventual OK | NoSQL |

**Verdict:** NoSQL.

### Why at OUR scale, SQL also "works" but we pick NoSQL

At 1,000 QPS and 750 GB, SQL is sufficient. But:

1. **Access pattern is pure key-value** — NoSQL purpose-built for this
2. **Horizontal scaling for 10x spike** — NoSQL handles automatically; SQL requires painful sharding
3. **Growth trajectory** — won't hit migration pain later

### Interview framing
> *"At 2,000 QPS and 750 GB, SQL would work — MySQL with read replicas handles it. I'm picking NoSQL because: access pattern is pure key-value (NoSQL's specialty), built-in horizontal scaling handles our 10x spike NFR without manual sharding, and future growth to 10x-100x scale doesn't require architectural rewrites. SQL would force a painful migration later."*

---

## 3. Decision 2: Which NoSQL — Candidate Comparison

### Candidate A: DynamoDB (AWS managed KV)
- **Pros:** Serverless, zero ops, 99.999% SLA, TTL support, auto-scaling
- **Cons:** AWS lock-in, expensive at scale, limited query flexibility
- **Used by:** Lyft, Airbnb, Disney+

### Candidate B: Cassandra (open-source wide-column) ✅
- **Pros:**
  - Masterless architecture → natural 99.99% availability
  - Tunable consistency per query
  - Horizontal scale with no architectural change
  - Strong Indian company usage (Razorpay, PhonePe)
- **Cons:**
  - Operational complexity (self-managed cluster)
  - Query-driven modeling (painful for evolving queries)
  - Eventually consistent by default
- **Used by:** Instagram, Netflix, Uber, Discord, Razorpay, PhonePe

### Candidate C: Redis (as primary DB)
- **Pros:** Fastest (in-memory), atomic operations, already in our stack
- **Cons:** All data in RAM expensive for 4.5 TB, not built as primary DB
- **Used by:** Twitter (timelines), not usually as sole DB

### ✅ Chosen: Cassandra

**Why:**
- Access pattern fits wide-column model perfectly
- Masterless architecture naturally matches 99.99% availability NFR
- Tunable consistency for per-operation CAP choice
- Target companies (Razorpay, PhonePe) use it heavily

### Interview framing
> *"Cassandra. Wide-column fits our key-value access pattern perfectly. Masterless architecture means no leader election — any node handles any request, naturally supporting 99.99% availability. Tunable consistency per query lets us pick strong for writes and eventual for reads. Instagram, Discord, Netflix, and PhonePe all use Cassandra for similar workloads."*

---

## 4. Cassandra Internals — Why It's Fast for Our Workload

### Read path for a point lookup

```
1. Client → coordinator node (~0.5 ms)
2. Coordinator hashes partition key → target node
3. Target node:
   a. Check memtable (RAM, ~0.1 ms)
   b. Check row cache (RAM, ~0.1 ms)
   c. Check key cache (which SSTable has it)
   d. Check Bloom filters (all SSTables, ~0.01 ms total)
   e. Read matching SSTable(s) (SSD, ~1-2 ms)
4. Return to client

TOTAL: 2-5 ms for point lookup
```

### Key optimizations
- **Partition key** routes directly to one node — no search
- **Bloom filters** skip SSTables that don't have the key
- **Memtable + row cache** serve hot data from RAM
- **Leveled Compaction Strategy (LCS)** guarantees most reads hit 1 SSTable

### Myth debunked: "NoSQL is slower than SQL"
| Query type | MySQL | Cassandra |
|-----------|-------|-----------|
| Point lookup (indexed) | 1-5 ms | 2-5 ms |
| Complex query (joins, aggregates) | 50-500 ms | ❌ not supported |

For point lookups, they're roughly equal. Cassandra is built for exactly this pattern.

### Why we cache even though Cassandra is fast

1. **Latency win:** 2ms → 0.5ms (small)
2. **Throughput win:** Redis handles 10-20x more QPS per node (huge)
3. **Cost win:** Cache absorbs traffic spikes 10x cheaper than scaling Cassandra
4. **Hot key protection:** Viral URLs served from Redis, not Cassandra

**Caching is primarily a cost + hot-spot protection, not a latency fix.**

---

## 5. Schema Design

### Philosophy: Query-driven modeling

**SQL philosophy:** normalize data, then query
**Cassandra philosophy:** design tables specifically for queries, duplicate data

### Queries to support

```
CORE (stable):
  Q1: Given short_url → get long_url (redirect)
  Q2: Given short_url → increment click count
  Q3: Given (user_id, idempotency_key) → check retry

LIKELY FUTURE:
  Q4: List URLs by user (dashboard)
  Q5: Cleanup expired URLs

UNLIKELY:
  Q6: Full-text search → use Elasticsearch separately
```

### Cassandra tables

```cql
-- Table 1: Redirect lookup (hot path)
CREATE TABLE urls_by_short_url (
  short_url       TEXT PRIMARY KEY,
  long_url        TEXT,
  user_id         BIGINT,
  custom_alias    TEXT,
  status          TEXT,
  expiry_at       TIMESTAMP,
  created_at      TIMESTAMP
);

-- Table 2: User's URLs (dashboard)
CREATE TABLE urls_by_user (
  user_id         BIGINT,
  created_at      TIMESTAMP,
  short_url       TEXT,
  long_url        TEXT,
  status          TEXT,
  PRIMARY KEY (user_id, created_at, short_url)
) WITH CLUSTERING ORDER BY (created_at DESC);

-- Table 3: Idempotency check
CREATE TABLE idempotency_keys (
  user_id         BIGINT,
  idempotency_key TEXT,
  short_url       TEXT,
  created_at      TIMESTAMP,
  PRIMARY KEY ((user_id, idempotency_key))
) WITH default_time_to_live = 86400;  -- 24h

-- Table 4: Click counters (Cassandra requires separate table for COUNTER)
CREATE TABLE url_click_counts (
  short_url   TEXT PRIMARY KEY,
  click_count COUNTER
);
```

### Tradeoff acknowledged

Query-driven modeling means adding a new query often requires a new table + backfill. For URL Shortener's stable query patterns, this is manageable. For evolving query needs, we'd add PostgreSQL for admin/analytics (polyglot persistence).

---

## 6. Analytics Strategy — Polyglot Persistence

### Why analytics breaks Cassandra
Analytics queries (top 10 clicked, date ranges, aggregations) don't fit query-driven modeling.

### Solution: Layered architecture

```
Clicks → Cassandra (live DB, hot path)
       → Kafka (event stream)
       → ClickHouse / Snowflake / BigQuery (OLAP)
       → Dashboards, reports
```

### OLTP vs OLAP — key distinction

| OLTP (Transactional) | OLAP (Analytical) |
|---------------------|-------------------|
| Cassandra, DynamoDB, MySQL | Snowflake, BigQuery, ClickHouse |
| Fast writes + point lookups | Aggregations on huge data |
| Latency: ms | Latency: seconds-minutes OK |
| Millions of small ops | Few massive queries |

**Rule:** Never use OLTP for analytics. Never use OLAP for live serving.

### Interview soundbite
> *"For v1, click_count lives on the URL row for 'total clicks per URL'. For rich analytics, I add async Kafka event stream → ClickHouse/Snowflake for OLAP queries. Cassandra stays hot path; OLAP handles ad-hoc analytics. This is polyglot persistence — different DBs for different jobs."*

---

## 7. Sharding — Partition Key Strategy

### What sharding is
Splitting data across multiple nodes so one node doesn't hold everything.

### Partitioning vs sharding (quick disambiguation)
- **Partitioning:** logical grouping of data
- **Sharding:** physical distribution across machines
- In most interviews, used interchangeably
- Cassandra says "partition key" but IS sharding

### How Cassandra shards — Consistent Hashing

```
Nodes placed on a logical ring (0 to 2^64).
For each row:
  1. hash(partition_key) → position on ring
  2. Walk clockwise → first node = primary
  3. Continue clockwise → next (RF-1) nodes = replicas

Same key → same nodes (deterministic, not random)
```

### Why consistent hashing (vs simple `hash % N`)

- Adding/removing a node remaps only ~1/N of keys
- Simple `hash % N` would remap ~100% of keys on resize
- Minimal data movement during scaling

### Partition key candidates

| Option | Distribution | Query support | Verdict |
|--------|-------------|---------------|---------|
| short_url ✅ | Even (hash of random base62) | Redirect O(1) ✅ | Winner |
| user_id | Uneven (power users) | Redirect needs full scan ❌ | No |
| created_at (time) | Write hotspot on "today" | Redirect needs full scan ❌ | No |
| (user_id, short_url) | Even | Redirect doesn't know user_id ❌ | No |

### ✅ Chosen: Partition by `short_url`

**Why:**
- Redirect queries hit exactly one node (O(1))
- Even distribution from base62 hashing
- Each partition is small (1 row)

### Interview framing
> *"Partition by short_url. Hash produces uniform distribution. Redirects hit one node — O(1) lookup. Alternatives fail: user_id creates hot partitions for power users and doesn't support redirect queries. Time-based partitioning creates write hotspots."*

---

## 8. Hot Key / Hot Partition Problem

### The problem
Viral URL: `short.ly/razorpay-diwali-offer` gets 100K clicks/sec.
All clicks hash to the same partition → one node gets overloaded → melts.

### Solutions menu

**1. Redis cache (our primary defense) ✅**
- First click: cache miss → Cassandra
- Subsequent clicks: cache hit → 0 load on Cassandra
- Redis shields Cassandra from hot keys

**2. Request coalescing (singleflight)**
- 100 simultaneous cache misses → 1 DB query, 99 wait
- Prevents thundering herd

**3. Key splitting (sub-partitioning)**
- For extreme cases (flash sales): `hash(short_url + random(0..9))`
- Spreads load across 10 partitions
- Read amplification — query all 10
- Instagram uses this for celebrity accounts

**4. Read replicas (natural from RF=3)**
- With RF=3, each partition has 3 replicas
- Reads can go to any replica → load splits

### Recommendation: Solutions 1 + 2 sufficient for URL Shortener

### Interview framing
> *"Hot keys handled at cache layer. Viral URLs get cached in Redis — first click populates, subsequent clicks served at sub-ms latency. Cassandra only sees cache-miss traffic. For extreme cases like flash sale products, we could sub-partition with random suffix — Instagram does this — but YAGNI for our scale."*

---

## 9. Replication Strategy

### Why replicate
Node failure = data loss. Replication = multiple copies across nodes.

### Replication Factor (RF)

```
RF = 3 → each row copied to 3 nodes

Why 3:
  RF=1: Node dies → data lost
  RF=2: Minimal safety (2 fail → lost)
  RF=3: Survives 2 failures ✅ Industry standard
  RF=5: Overkill for most cases
```

### How Cassandra places replicas
On the consistent hash ring, from hash position:
- Primary = next node clockwise
- Replica 2 = next node after
- Replica 3 = node after that

Deterministic — same key always goes to same 3 nodes.

### How writes replicate

```
Client write → coordinator node
Coordinator sends to ALL 3 replicas in parallel
(NOT sequential — all simultaneous)
```

### Masterless architecture
- No leader / primary
- All replicas are equal
- Any node can serve any read/write
- No failover dance when a node dies

### Failure handling
- Node down → other replicas continue serving
- Missed writes stored as "hints" on live nodes
- Hinted handoff replays writes when node returns
- Read repair fixes inconsistencies during reads
- No manual intervention, no downtime

---

## 10. Consistency Levels (CL) — The Key Mechanism

### What CL does
Tells Cassandra how many replicas must confirm before returning "success."

### The 3 main values

| CL | Behavior | Speed | Safety |
|----|----------|-------|--------|
| ONE | 1 replica confirms | Fastest | Eventual consistency |
| QUORUM | Majority confirms | Balanced | Strong consistency (with paired read) |
| ALL | All replicas confirm | Slowest | Strongest, but fragile |

### QUORUM formula
```
QUORUM = (RF / 2) + 1

RF=3 → QUORUM = 2
RF=5 → QUORUM = 3
```

### The magic formula

```
W + R > RF → STRONG CONSISTENCY

Write QUORUM (2) + Read QUORUM (2) = 4
RF = 3
4 > 3 ✅ Strong consistency guaranteed
```

Why it works: write hits 2 replicas, read asks 2 replicas → at least 1 overlap → latest data guaranteed visible.

### Per-operation tuning for URL Shortener

| Operation | CL | Reasoning |
|-----------|-----|-----------|
| Write (POST /urls) | QUORUM | Safe; must be visible to idempotency checks |
| Read (redirect) | ONE | Fast; stale data by seconds is fine |
| Read (idempotency) | QUORUM | Must see latest to detect retries |
| Read (click counter) | ONE | Eventual consistency fine |

### Interview soundbite
> *"Cassandra is masterless — no primary to route to. Instead, I tune consistency level per operation using the W + R > RF formula. For URL Shortener: writes use QUORUM (safe), redirect reads use ONE (fast, stale OK), idempotency reads use QUORUM (must be fresh). Different operations, different CAP choices in the same system."*

---

## 11. Capacity Planning

### Inputs
```
Storage:     1.5 TB (corrected from 500 bytes/record assumption)
RF:          3
Read QPS:    2,000 peak
Write QPS:   20 peak
```

### Total storage with RF
```
1.5 TB × 3 = 4.5 TB total cluster storage
```

### Node sizing — why 1-2 TB per node

Industry rule: keep Cassandra nodes under 2 TB for operational safety.

**Drivers:**
1. **Repair time:** 1 TB ≈ 3 hours, 10 TB ≈ 30 hours
2. **Compaction pressure:** larger nodes = longer compactions
3. **Network bandwidth:** repair streams at ~100 MB/s
4. **Blast radius:** larger node = harder to replace

**Rule of thumb:**
- 500 GB – 2 TB: sweet spot
- 2-4 TB: acceptable with strong infra
- >5 TB: avoid (per Datastax, Netflix)

### Our sizing

```
4.5 TB ÷ 1 TB per node = 5 nodes minimum

Choice: 5 nodes × 1 TB each
Total capacity: 5 TB (holds our 4.5 TB with headroom)
```

### QPS capacity check

```
Each Cassandra node: ~10,000 reads/sec

5 nodes × 10K = 50,000 reads/sec capacity
Our peak read: 2,000 reads/sec

Utilization: 4% — massively over-provisioned on QPS.
```

### Key insight: URL Shortener is STORAGE-bound, not QPS-bound
- Storage forces 5 nodes
- QPS could run on 1 node
- Storage is the binding constraint

### Why still provision 25x QPS headroom?

**Cache-independent sizing rule:**

```
NEVER size DB based on cache-hit numbers.

If Redis is down / cold / bypassed:
  ALL 2,000 QPS hits Cassandra directly.
  → Cassandra must handle full load.

Cache is optimization, not dependency.
If DB can't serve without cache → 1 Redis outage = full downtime.
```

### Interview soundbite
> *"I size Cassandra for worst-case — Redis down, cache cold, viral traffic bypassing cache. Cache is optimization, not dependency. Otherwise one Redis outage = full downtime. For 4.5 TB storage at 1 TB/node, I use 5 nodes. That gives 25x QPS headroom — plenty for cache-miss spikes."*

---

## 12. Growth Planning

### When to add nodes
```
Trigger at 70% storage usage:
  5 TB × 70% = 3.5 TB

When cluster hits 3.5 TB, add a 6th node.
Cassandra rebalances automatically (consistent hashing minimizes data movement).
```

### Operational headroom
- QPS: 25x headroom (Cassandra) + 100x (Redis cache)
- Storage: 10% headroom before growth trigger
- Fault tolerance: survives 2 node failures (RF=3)

---

## 13. Final DB Design Snapshot

```
╔════════════════════════════════════════════╗
║  URL SHORTENER — DATABASE DESIGN           ║
╠════════════════════════════════════════════╣
║                                            ║
║  Primary DB:         Cassandra             ║
║  Cache:              Redis (150-300 GB)    ║
║  Analytics:          Kafka → ClickHouse    ║
║                      (future layer)        ║
║                                            ║
║  SHARDING                                  ║
║  Partition key:      short_url             ║
║  Algorithm:          Consistent hashing    ║
║  Hot key defense:    Redis cache           ║
║                                            ║
║  REPLICATION                               ║
║  Replication Factor: 3                     ║
║  Write CL:           QUORUM                ║
║  Read CL (redirect): ONE                   ║
║  Read CL (idempot.): QUORUM                ║
║  Read CL (counter):  ONE                   ║
║                                            ║
║  CAPACITY                                  ║
║  Nodes:              5                     ║
║  Storage/node:       1 TB                  ║
║  Total capacity:     5 TB                  ║
║  QPS headroom:       25x (cache-safe)      ║
║  Fault tolerance:    Survives 2 failures   ║
║                                            ║
║  SCHEMA                                    ║
║  4 tables (query-driven)                   ║
║  - urls_by_short_url  (redirect)           ║
║  - urls_by_user       (dashboard)          ║
║  - idempotency_keys   (retries)            ║
║  - url_click_counts   (counter)            ║
║                                            ║
╚════════════════════════════════════════════╝
```

---

## 14. How Every Decision Ties to Earlier Steps

| Decision | Traces back to |
|----------|---------------|
| NoSQL (Cassandra) | Step 1 NFRs (99.99% avail), Step 2 (access pattern) |
| RF = 3 | Step 1 NFR (99.99% availability) |
| Partition by short_url | Step 2 (redirect is 100x the volume) |
| Write CL = QUORUM | Step 4 (idempotency needs freshness) |
| Read CL = ONE | Step 1 (p99 <100ms), Step 2 (2000 QPS) |
| 5 nodes, 1 TB each | Step 2 (1.5 TB storage × RF=3 = 4.5 TB) |
| Redis cache in front | Step 2 (100:1 read:write, hot key defense) |
| Polyglot analytics | Step 1 (OLAP not mentioned; add when needed) |

**Never justify a DB decision with "that's the best." Always cite a number or requirement.**

---

## 15. Interview Best Practices

### DO
1. Ask clarifying questions about access patterns BEFORE picking a DB
2. Justify NoSQL with pattern + scalability, not "it's faster"
3. Pick Cassandra for key-value-at-scale; MySQL for complex queries
4. Use RF=3 as default (industry standard)
5. Tune consistency per-operation (ONE vs QUORUM)
6. Size for worst case (cache down), not happy path
7. Acknowledge tradeoffs explicitly (query-driven pain, eventual consistency)
8. Separate OLTP from OLAP — polyglot persistence

### DON'T
1. Don't say "NoSQL is faster than SQL" (myth for point lookups)
2. Don't use user_id as partition key for URL Shortener (hot partitions + bad queries)
3. Don't forget hot key defense (cache + coalescing)
4. Don't size Cassandra based on cache-hit numbers
5. Don't commit to CL ALL for URL Shortener (availability drops)
6. Don't ignore replication lag (Step 4 lesson)
7. Don't try to force analytics onto Cassandra (use OLAP layer)

---

## 16. Interview Soundbites

**On DB choice:**
> *"Cassandra — masterless architecture for 99.99% availability, wide-column fits key-value access, tunable consistency per query, used by Instagram/Netflix/Razorpay for similar workloads."*

**On sharding:**
> *"Partition by short_url using consistent hashing. Hash is uniform, redirects hit one node, each partition is tiny (1 row), no hot partitions from heavy users."*

**On hot keys:**
> *"Cache shields Cassandra. Viral URLs get cached — first click populates Redis, subsequent clicks served sub-ms. For extreme cases, sub-partitioning with random suffix like Instagram's celebrity handling — YAGNI for our scale."*

**On consistency:**
> *"W+R>RF formula. Writes QUORUM, redirect reads ONE (fast + stale OK), idempotency reads QUORUM (must be fresh). Per-operation tuning, same system."*

**On sizing:**
> *"Size for cache failure. All QPS hits Cassandra when Redis is down. Storage is our binding constraint — 5 nodes at 1 TB each handles 4.5 TB with RF=3, plus 25x QPS headroom."*

**On analytics:**
> *"Polyglot persistence. Cassandra hot path, Kafka stream to ClickHouse for OLAP. OLTP and OLAP are fundamentally different workloads — never conflate them."*

---

## 17. Self-Assessment

### What went well
- Pushed back on "500 bytes per record" assumption → caught at interview-grade depth
- Asked about read/write path separation (CQRS awareness)
- Asked "if analytics comes later?" (architectural thinking)
- Challenged "is NoSQL slower" (first-principles)
- Asked about Bloom filter overhead (deep internals)
- Asked about partitioning vs sharding terminology

### To improve
- Initial record size (500 bytes) didn't account for 2048-char URLs — fixed mid-way
- Initially unsure about partition key for URL Shortener — got there with guidance
- Still developing intuition for consistency levels — formula will help

### Step 5 grade: 9/10

---

## 18. Key Terms

| Term | Meaning |
|------|---------|
| Sharding | Physical distribution of data across nodes |
| Partitioning | Logical grouping of data (often used interchangeably with sharding) |
| Partition key | Field Cassandra hashes to decide node placement |
| Consistent hashing | Ring-based algorithm where +/- nodes only remap 1/N keys |
| Replication Factor (RF) | Number of copies of each row |
| Masterless | No leader node; all replicas equal |
| Consistency Level (CL) | How many replicas must confirm for success |
| QUORUM | (RF/2)+1 replicas; majority |
| Hot key | Single key with disproportionate traffic |
| Hinted handoff | Cassandra's mechanism to replay missed writes |
| LCS | Leveled Compaction Strategy — optimal for read-heavy |
| Polyglot persistence | Using different DBs for different purposes |
| OLTP | Transactional DB (Cassandra, MySQL) |
| OLAP | Analytical DB (Snowflake, ClickHouse) |
| Read repair | Cassandra auto-fixes stale replicas on read |

---

## 19. Next Step — Deep Dives (Step 6)

Step 6 covers:
1. **Latency** — redirect path breakdown, tail latency, p99 strategies
2. **Throughput** — bottleneck per component, scale-per-component
3. **Fault tolerance** — circuit breakers, retries, bulkheads
4. **CAP application** — per-component CAP choice
5. **ID generation** — how 7-char base62 actually works
6. **Rate limiting** — token bucket, sliding window, where to place
7. **Security** — XSS, phishing abuse, DoS
8. **Observability** — what to monitor, alerts

This is where we stress-test the design.