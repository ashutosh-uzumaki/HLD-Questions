# URL Shortener — Step 3: High Level Design (HLD)

**Problem:** Design a URL shortener like bit.ly / tinyurl
**Step:** 3 of 8 (High Level Design)
**Teaching style:** For every decision → show ALL options with tradeoffs

---

## 1. HLD Philosophy

**Step 3's job:** Draw components + show how they connect.

**NOT Step 3's job:**
- Database schema (that's Step 5)
- API contracts (that's Step 4)
- Deep failure modes (that's Step 6)

**Golden rule:** Every component you add must earn its place. No component should be there "just in case." YAGNI applies.

---

## 2. Diagram Best Practices

### Single cluster box > Multiple individual boxes

**Preferred:**
```
┌──────────────────────┐
│  APP SERVER CLUSTER  │
│  (3+ instances,      │
│   stateless,         │
│   auto-scaled)       │
└──────────────────────┘
```

**Not preferred (cluttered):**
```
┌────┐  ┌────┐  ┌────┐
│ S1 │  │ S2 │  │ S3 │
└────┘  └────┘  └────┘
```

### When to use each

| Use Single Cluster Box | Use Multiple Boxes |
|------------------------|---------------------|
| HLD overview | Instances have distinct roles (Leader/Follower) |
| Redundancy is implicit | Deep dive on failover (Step 6) |
| Many similar instances | Specific sharding (Shard 1, Shard 2) |
| Cleaner, faster | When count explicitly matters |

### Label components richly

Labels teach the interviewer your thinking. Pack in Tier-2 signals:

```
"App Server Cluster
 (3-5 instances, stateless, auto-scaled)"

"Redis Cluster
 (3-5 nodes, ~150 GB, consistent hashing)"

"DB Cluster
 (Primary + 2 read replicas, ~1 TB sharded)"
```

---

## 3. The Two Flows to Design

```
FLOW 1: SHORTEN (write path)
============================
User → System → Short URL returned
Scale: 20 peak QPS (tiny)

FLOW 2: REDIRECT (read path)
============================
User clicks short URL → System → 302 redirect to long URL
Scale: 2,000 peak QPS + click counter update
```

**Design them separately.** Combining them creates confusion.

---

## 4. Decision 1: Load Balancer — Layer 4 vs Layer 7

### Option A: L4 Load Balancer (TCP level)
- Routes based on IP/port only
- No understanding of HTTP content
- Faster (less processing per packet)
- Less flexible
- Examples: AWS NLB, HAProxy TCP mode

### Option B: L7 Load Balancer (HTTP level)
- Routes based on URL path, headers, cookies
- Can do SSL termination, rate limiting, auth
- Slightly slower
- More flexible
- Examples: AWS ALB, Nginx, Cloudflare

### Option C: DNS-based Load Balancing
- Uses DNS to resolve to different IPs
- Very simple, geographic routing
- TTL-based (stale mappings possible)
- Examples: Route 53, Cloudflare DNS

### ✅ Recommendation for URL Shortener: L7 (AWS ALB / Nginx)

**Why:**
- Can route `/shorten` to shorten pool and `/{short_url}` to redirect pool separately (different scaling needs)
- SSL termination in one place
- Rate limiting per path possible
- Small perf cost is negligible at our scale

### Interview framing
> *"I'll use an L7 load balancer like AWS ALB. It lets us route shorten and redirect traffic differently, handle SSL, and apply per-path rate limits. L4 would be faster but overkill-simple for our needs."*

---

## 5. Decision 2: App Server Deployment — Stateful vs Stateless

### Option A: Stateful Servers
- Session/user data stored in server memory
- Sticky sessions needed (user always hits same server)
- Hard to scale horizontally
- Failure = user loses session
- **Avoid for distributed systems**

### Option B: Stateless Servers ✅
- No session/user state in server memory
- All state in external cache/DB
- Any server can handle any request
- Easy horizontal scaling, zero-downtime deploys

### ✅ Recommendation: Stateless

**Why mandatory for URL Shortener:**
- 99.99% availability NFR needs failover without session loss
- Horizontal auto-scaling requires statelessness
- Request has no "state" anyway (idempotent lookups)

### Interview framing
> *"App servers are stateless — no session or request state in memory. All state lives in Redis/DB. Any server handles any request, enabling horizontal scale, zero-downtime deploys, and easy failover."*

---

## 6. Decision 3: Cache — Local vs Centralized

### Option A: Local In-Memory Cache
- Cache lives inside app server (Caffeine, Guava)
- Fastest (no network hop, sub-ms)
- Each server has its own copy
- **Downsides:**
  - Cache inconsistency across servers
  - Low hit rate (each server sees 1/N traffic)
  - Lost on restart
  - 150 GB × 5 servers = 750 GB of duplicated memory

### Option B: Centralized Cache (Redis/Memcached) ✅
- Shared cache all app servers hit
- Network hop (~1-2 ms)
- Single source of truth
- Scales independently
- Survives app server restarts

### Option C: Hybrid (L1 + L2)
- L1 = local cache (small, hottest data)
- L2 = centralized (bigger, warm data)
- Used by Netflix, Instagram, YouTube
- Complex to keep in sync
- **Overkill for our scale**

### ✅ Recommendation: Centralized (Option B)

**Why:**
- 150 GB can't be duplicated on every server
- 100:1 read ratio — we need the high hit rate of centralized
- Click counter needs single source of truth

### Interview framing
> *"Centralized Redis cluster. Local would duplicate 150 GB across servers and fragment hit rates. Hybrid L1+L2 is how Netflix does it, but YAGNI for our scale — we can add L1 later if p99 needs tightening."*

---

## 7. Decision 4: Cache Technology — Redis vs Memcached

### Option A: Memcached
- Pure string key-value
- Multi-threaded per node
- No persistence, no replication built-in
- Slightly faster for simple GET/SET

### Option B: Redis ✅
- Multiple data structures (string, hash, sorted set, etc.)
- Atomic operations (INCR, DECR)
- Persistence (RDB, AOF)
- Replication + Cluster mode built-in
- Pub/Sub, streams, Lua scripting

### ✅ Recommendation: Redis

**Why for URL Shortener:**
- Need atomic INCR for click counter
- Need Cluster mode for 150 GB (Memcached requires client-side sharding)
- 99.99% availability needs replication (Redis built-in)

### Interview framing
> *"Redis over Memcached because we need atomic INCR for click counters, Cluster mode for 150 GB sharding, and built-in replication for HA. Memcached wins on pure GET/SET speed but those features are non-negotiable for us."*

### Companies using each
- **Memcached:** Facebook (news feed), Twitter (historical)
- **Redis:** Twitter (timelines), Instagram (feed cache), Uber (geospatial), Razorpay (rate limiting)

---

## 8. Decision 5: Cache Write Strategy — Write-through vs Write-around vs Write-back

### Option A: Write-through
- Write goes to cache AND DB simultaneously
- Cache always fresh
- Extra write latency
- Risk: cache/DB out of sync if one write fails

### Option B: Write-around ✅
- Write goes to DB only
- Cache populated on first read (cache-aside)
- Simpler
- First read after write is slow (cache miss)
- Used by most URL shorteners

### Option C: Write-back (Write-behind)
- Write goes to cache only; async to DB later
- Fastest writes
- Data loss risk if cache crashes before sync
- Used rarely (only when writes are very heavy)

### ✅ Recommendation: Write-around

**Why for URL Shortener:**
- Shorten is 20 QPS — write speed irrelevant
- Most URLs may never be clicked — no point pre-caching
- First-click latency difference is unnoticeable (~10ms)

### Interview framing
> *"Write-around (cache-aside). Shorten is 20 QPS so write latency doesn't matter. Most URLs may never be clicked — pre-caching would waste memory. First click takes a DB hit, subsequent clicks served from cache."*

---

## 9. Decision 6: Click Counter — HOW to Increment (Detailed Menu)

**This is the decision you asked for full treatment on.**

### Option A: Synchronous DB Update
- Every click → `UPDATE urls SET count = count + 1 WHERE ...`
- **Pros:** Simple, strong consistency
- **Cons:** 2,000 QPS of DB writes — kills DB
- **Verdict:** ❌ Don't do this

### Option B: Redis INCR
```
App Server → Redis: INCR click:short_url
App Server → User: 302 redirect
(Background job syncs Redis → DB every 60s)
```
- **Pros:**
  - Atomic, ~0.1 ms latency
  - No extra infrastructure (Redis already in stack)
  - Non-blocking
- **Cons:**
  - Small data loss risk if Redis crashes before sync
  - Eventual consistency
- **When it fits:** Simple counter, QPS < 10K, no downstream fan-out needed

### Option C: Kafka + Consumer Worker
```
App Server → Kafka: publish click event
App Server → User: 302 redirect
Kafka Consumer → aggregates → writes to DB
```
- **Pros:**
  - Durable (events persisted to disk)
  - Event replay capability
  - Multiple consumers can subscribe (analytics, fraud, ML)
  - Scales to millions of events/sec
- **Cons:**
  - Kafka cluster = operational overhead
  - ~5-10 ms latency to publish
  - Overkill for simple counter
- **When it fits:** QPS > 100K, event-driven architecture exists, multiple downstream systems

### Option D: Redis Streams
- Redis-based event log (Kafka-lite)
- Already have Redis — no new infra
- Less durable than Kafka
- Smaller ecosystem
- **When it fits:** Need event semantics but don't want Kafka overhead

### Option E: In-memory Queue + Batch Writer
- App holds in-memory queue, flushes to DB every N seconds
- **Pros:** Zero external dependencies
- **Cons:** Data loss on server crash
- **Verdict:** ❌ Fragile for production

### Option F: Message Queue (RabbitMQ, SQS)
- Similar to Kafka but simpler semantics
- Good for task queuing, not high-throughput event streaming
- **When it fits:** Moderate QPS, don't need replay

### ✅ Recommendation for URL Shortener: Option B (Redis INCR)

**Why:**
- 2,000 QPS is nowhere near Kafka-necessary scale
- Redis already in the system — reuse it
- No fan-out requirement (just a counter)
- Small data loss risk is acceptable for "click count"
- YAGNI on Kafka

### Interview framing — the Tier-2 answer
> *"I'll use Redis INCR. Synchronous DB update would kill the DB at 2K QPS. Kafka would work but it's overkill — we don't have multiple downstream consumers and 2K QPS doesn't justify the operational overhead of a Kafka cluster. Redis INCR is atomic, sub-ms, and we already have Redis. A background job syncs counters to DB every 60 seconds. If we later need analytics, fraud detection, or real-time dashboards, I'd add Kafka — but YAGNI."*

**Key phrase:** **"YAGNI"** (You Aren't Gonna Need It) — senior-level engineering.

---

## 10. Decision 7: Database Type (Preview Only — Decided in Step 5)

For HLD, we just say "DB Cluster" and defer the SQL/NoSQL decision to Step 5.

**Why defer?**
- DB choice is a detailed decision needing proper reasoning
- We might realize in Step 4 (API design) that we need transactions → changes DB choice
- Premature commitment is an interview anti-pattern

**For now:** Label it "DB Cluster" generically.

---

## 11. Decision 8: URL Generator Placement

### Option A: Logic Inside App Server ✅
- URL generation is a function call inside app server
- No network hop
- Simple

### Option B: Dedicated ID Generator Service (Snowflake-style)
- Separate deployed service
- Needed at extreme scale (10M+ IDs/sec, like Twitter)
- **Will cover deeply in Problem 3 (Unique ID Generator)**

### ✅ Recommendation: Option A (logic inside app server)

**Why:**
- 20 QPS writes — app server handles easily
- No need for separate service at this scale

### Interview framing
> *"URL generator is a function inside the app server — Base62 encoding of 7 characters. At 20 QPS writes, we don't need a separate ID service like Twitter's Snowflake. I'll detail the generation strategy in deep dives (Step 6)."*

---

## 12. Final Architecture — Two Flows

### WRITE PATH (Shorten)

```
  User
   ↓
 Load Balancer (L7, AWS ALB / Nginx)
   ↓
 App Server Cluster
 (stateless, 3+ instances, auto-scaled)
   │
   │ (generate 7-char base62 short URL)
   ↓
 DB Cluster (store: short_url → long_url)

Notes:
- Write-around strategy — cache NOT touched here
- Cache populated lazily on first read
- Validate custom alias collision if provided
```

### READ PATH (Redirect)

```
  User clicks short URL
   ↓
 Load Balancer (L7)
   ↓
 App Server Cluster (stateless)
   ↓↑
 Redis Cluster                     ← cache lookup
   │
   ├── HIT  → return long URL → 302 redirect
   │          (+ INCR click:short_url in Redis)
   │
   └── MISS → query DB
              → populate cache
              → return long URL → 302 redirect
              (+ INCR click counter)

Background job (every 60s):
  Redis click counters → sync to DB
```

---

## 13. Components Summary

| Component | Purpose | Scale |
|-----------|---------|-------|
| Load Balancer | L7 routing, SSL termination | AWS ALB or Nginx |
| App Server Cluster | Business logic, URL generation | 3-5 stateless instances, auto-scaled |
| Redis Cluster | URL cache + click counters | 3-5 nodes, ~150 GB, consistent hashing |
| DB Cluster | Persistent storage | Primary + 2 read replicas, ~1 TB |
| URL Generator | Base62 7-char encoding | Logic inside app server |
| Background Job | Sync Redis counters → DB | Runs every 60 seconds |

---

## 14. How Numbers Drove Every Decision

Connect Step 2 numbers → Step 3 decisions. Interviewers love this.

| Step 2 Number | Step 3 Decision |
|---------------|------------------|
| 100:1 read:write | Cache-first architecture, read replicas > write replicas |
| 2,000 peak QPS | Single LB + 3-5 app servers (not massive scale) |
| 20 write QPS | No separate ID generator service needed |
| 150 GB cache | Redis Cluster (not single node), consistent hashing |
| 1 TB storage | Shard DB horizontally (not for size but for HA) |
| 99.99% availability | Every component needs redundancy |
| p99 < 100ms redirect | Cache mandatory, async click counter |

---

## 15. Interview Best Practices for HLD

### DO
1. **Start simple, add complexity as needed** — draw user → server → DB first, then layer in cache, LB, etc.
2. **Label components richly** — "Redis Cluster (3-5 nodes, 150 GB)" > "Cache"
3. **Justify each component with a number** from Step 2
4. **Keep HLD under 7-8 components** — if more, you're overthinking
5. **Draw 2 flows separately** (write + read) — don't spaghetti them together
6. **Say "I'll cover this in deep dives"** for DB choice, ID generation, etc.
7. **Explicitly use "YAGNI"** when rejecting unnecessary complexity

### DON'T
1. Don't commit to SQL/NoSQL in Step 3 (that's Step 5)
2. Don't draw 3 individual server boxes — draw 1 cluster box
3. Don't add components "just because they're common" (Kafka without fan-out need)
4. Don't forget the async click counter — shows latency awareness
5. Don't mix write + read flows in one diagram

---

## 16. Interview Soundbites to Memorize

**On stateless:**
> *"All state in cache/DB, none in server memory. Any server handles any request — enables horizontal scale and zero-downtime deploys."*

**On centralized cache:**
> *"150 GB can't be duplicated across servers — centralized Redis gives single source of truth and high hit rate from aggregated traffic."*

**On Redis over Memcached:**
> *"Need atomic INCR for click counters and Cluster mode for 150 GB sharding. Memcached wins on pure speed but those features are non-negotiable."*

**On write-around:**
> *"Cache-aside / write-around. Shorten is 20 QPS — write latency doesn't matter. Most URLs may never be clicked, so lazy cache population is efficient."*

**On Redis INCR over Kafka:**
> *"At 2K QPS with no fan-out requirement, Kafka is over-engineering. Redis INCR is atomic, sub-ms, and reuses infra we already have. YAGNI on Kafka — add it when we have real event-driven needs."*

**On deferring DB choice:**
> *"I'll decide SQL vs NoSQL in database design. For now, it's a DB cluster — the access pattern (key-value lookup) suggests NoSQL, but I'll validate tradeoffs in Step 5."*

---

## 17. What I Got Wrong Initially (Self-Correction)

**Mistake:** I initially suggested Kafka for click counter without comparing alternatives.

**Why it was wrong:** 2,000 QPS doesn't justify Kafka's operational overhead. Redis INCR is simpler, faster, reuses existing infra.

**Lesson:** Always ask *"what's the simplest thing that works for OUR scale?"* Don't pattern-match to textbook solutions without reasoning from your numbers.

**Key phrase to lock:** **YAGNI** — You Aren't Gonna Need It. Over-engineering is worse than under-engineering at SDE-2 level.

---

## 18. Self-Assessment

### What I did well
- Separated write and read flows
- Correctly chose centralized cache over local
- Pushed back on Kafka — caught over-engineering instinct
- Asked about Memcached vs Redis (shows awareness of alternatives)
- Understood stateless + horizontal scaling connection

### What to improve
- First diagram had "NoSQL DB" (committed too early)
- Initial write path had 3 URL Generator boxes (confused component vs logic)
- Missed async click counter in first diagram pass

### Step 3 grade: 8/10

---

## 19. Concept Consolidation — Key Terms

Lock these interview terms:

| Term | Meaning |
|------|---------|
| **Stateless** | No session/state in server memory |
| **Cache-aside / Write-around** | Write to DB; populate cache on read miss |
| **Cache-hit / Cache-miss** | Data in cache (hit) vs needs DB fetch (miss) |
| **Consistent hashing** | Sharding algorithm that minimizes resharding on node add/remove |
| **YAGNI** | You Aren't Gonna Need It — don't add complexity prematurely |
| **Fan-out** | Single event → multiple consumers/systems |
| **L4 vs L7 LB** | Network level (TCP) vs Application level (HTTP) |
| **SSL termination** | Decrypt HTTPS at LB, internal traffic plain HTTP |

---

## 20. Next Step Preview — API Design (Step 4)

With components locked, we design the contract between client and server:

1. **POST /urls** — shorten a URL (request/response schema)
2. **GET /{short_url}** — redirect flow
3. **DELETE /urls/{id}** — delete URL (optional)
4. **HTTP methods**: which to use and why
5. **Idempotency**: how shorten stays idempotent on retries
6. **Error responses**: 400, 409, 404, 429, 500
7. **REST vs gRPC**: which for URL Shortener and why