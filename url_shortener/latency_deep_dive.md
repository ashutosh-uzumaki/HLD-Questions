# URL Shortener — Step 6: Latency Deep Dive

**Problem:** Design a URL shortener like bit.ly / tinyurl
**Step:** 6 of 8 (Deep Dives — Topic 1: Latency)
**Teaching style:** Every concept built from simple → technical

---

## 1. What is Latency?

**Simple definition:**
```
Latency = time taken from request sent to response received.
```

How long the user waits. Nothing more.

### Example
User clicks short.ly/abc1234 → browser opens long URL after 50 ms.
→ **Latency = 50 ms**

---

## 2. Why Latency Matters — User Psychology

```
0-100 ms     → feels instant ✨
100-300 ms   → slight delay (still OK)
300-1000 ms  → user notices waiting
1000+ ms     → user frustrated, may leave
```

**Our NFR from Step 1:** p99 < 100 ms (feels instant).

### Why p99, not average?
- **Average hides outliers.** 99 requests at 10 ms + 1 at 10 s = 100 ms avg (looks fine)
- **p99** = the 99th percentile = worst 1% of requests
- At 10M requests/day, 1% = 100K slow experiences daily
- **p99 is what users actually experience at scale**

Standard percentiles:
- **p50 (median)** — typical experience
- **p95** — most users
- **p99** — tail latency (the bad cases)
- **p99.9** — worst cases (matters at massive scale)

---

## 3. Where Latency Comes From — The Decomposition

Latency is **many small delays added up** across components.

```
When user clicks short.ly/abc1234:
===================================

Step 1: Request travels over network to server
Step 2: Server processes request
Step 3: Server reads data from cache/DB
Step 4: Server builds response
Step 5: Response travels back to user

Total latency = sum of all these.
```

### Pizza analogy
```
Ordering pizza:
  Call placed (1 min)
  Order taken (2 min)
  Pizza cooked (15 min)
  Delivery (20 min)
  Total: 38 min

Delivery dominates. That's where optimization pays off.
```

---

## 4. The Key Insight — Attack the LONGEST Step

**To reduce total latency, target the biggest step first.**

Don't waste effort optimizing tiny steps:
- Reducing 1 min → 30 sec saves 30 sec
- Reducing 20 min → 10 min saves 10 min ✨

---

## 5. What Dominates Latency — NETWORK

```
REALISTIC BREAKDOWN FOR A REDIRECT
===================================

Client → LB (TLS reused):     20-50 ms   ← NETWORK
LB → App server:              1-2 ms
App processing:               1-2 ms
App → Redis (hit):            0.5-1 ms
App → Client response:        20-50 ms   ← NETWORK
                              ───────────
TOTAL:                        ~45-100 ms

Of which:
  NETWORK:  ~80% of total
  COMPUTE:  ~5% of total
  STORAGE:  ~15% of total
```

**Network dominates. Compute and storage are usually fast.**

---

## 6. Why Network Is So Slow — PHYSICS

Data moves through fiber optic cables at ~200,000 km/sec (speed of light through fiber).

### Real network latencies (round-trip)

| Route | Distance | RTT |
|-------|----------|-----|
| Same city (Mumbai → Mumbai) | <50 km | 1-5 ms |
| Same country (Mumbai → Bangalore) | 1,000 km | 15-30 ms |
| Asia (Mumbai → Singapore) | 4,000 km | 60 ms |
| Cross-continent (Mumbai → Frankfurt) | 7,000 km | 130 ms |
| Across globe (Mumbai → New York) | 12,000 km | 250 ms |

### The hard truth
```
You cannot break the speed of light.
Longer distance = more latency.
No amount of code optimization beats physics.
```

---

## 7. The TLS Handshake Cost

For HTTPS connections, there's setup overhead:

```
FIRST CONNECTION (cold)
========================
TCP handshake:  1 round-trip
TLS handshake:  2 round-trips
HTTP request:   1 round-trip
Total: 4 round-trips × RTT

For a NY→Mumbai connection (250 ms RTT):
  First request: 4 × 250 = ~1000 ms before server even works!

REUSED CONNECTION (warm)
=========================
TCP + TLS already established.
Only HTTP request: 1 × 250 = 250 ms
```

**Connection pooling** reuses connections → saves 60-90 ms per subsequent request.

---

## 8. The Problem for Global Users

**Scenario:** Mumbai server + New York user

```
Network alone = 120 ms (one-way)
Round trip = 250 ms

Can we hit p99 < 100 ms?
NO. Impossible. Physics won't allow it.
```

**Solution:** Put a server CLOSER to the user. This is what CDN does.

---

## 9. CDN — The Biggest Latency Win

### What CDN is
**Content Delivery Network** — third-party service with edge servers spread globally.

Examples:
- **Cloudflare** — 300+ cities
- **AWS CloudFront** — 450+ edge locations
- **Akamai** — 4,100+ locations

### How it works

```
WITHOUT CDN
============
ALL users → your Mumbai server
NY user pays 250 ms

WITH CDN
=========
NY user → nearest CDN edge (NY) = 5 ms
Edge has cached response → returns directly
Only CACHE MISSES go to origin

Result:
  Global users: 10-20 ms
  Origin sees 1% of traffic
```

### Why CDN Works for URL Shortener

URL Shortener responses are **IMMUTABLE**:
```
GET /abc1234 → 302 redirect to "https://..."

Same input → same output always.
Perfect for caching at edges.
```

Compare to Twitter home feed — different response per user. CDN can't cache easily.

### Pizza analogy
```
Without CDN:   One pizza shop in Mumbai serves the world.
With CDN:      Pizza shops in every city.
               Each has its own copies of popular pizzas.
```

---

## 10. CDN vs Multi-Region Deployment

**Similar problems, different tools.**

### CDN
```
Third-party service with edge locations
Caches YOUR responses at edges
Serves CACHEABLE content
Low ops overhead, cheap
```

### Multi-Region
```
YOUR OWN infrastructure in multiple regions
Each region: LB + app servers + Redis + DB
Serves LIVE, WRITE-able, PERSONALIZED content
High ops overhead, expensive
```

### When to use which

| Use Case | Tool |
|----------|------|
| Cacheable reads (URL redirect, static files) | CDN |
| Global writes | Multi-region |
| Personalized content | Multi-region |
| Compliance (data residency) | Multi-region |
| Best of both | CDN + Multi-region |

### For URL Shortener
**Phase 1 (launch):** CDN + single-region origin
- CDN handles 99% of reads (cacheable redirects)
- Single region for writes
- Distant writers accept ~250 ms for one-time action

**Phase 2 (scale):** Add multi-region
- CDN globally
- Multi-region writes with Cassandra multi-DC replication

---

## 11. The Cold Cache Problem

**Reality: CDN edges are empty initially.**

```
First user from NY:
  → NY edge cache empty
  → Forwards to Mumbai origin (250 ms)
  → Response cached at NY edge
  → User gets: ~500 ms

Subsequent NY users for same URL:
  → NY edge cache HIT
  → Returns in ~10 ms ✅
```

### Four cold cache problems

1. **First user pays the tax** — someone's always first, they see 250 ms
2. **Rare URLs never warm up** — long-tail URLs have spread clicks across time, never cache-hot
3. **Cache eviction** — CDN has limited memory, URLs get evicted
4. **Deploy / restart** — edge restarts = entire cache empty again

### Math — cold cache impact

```
CDN edge caches top 50K URLs out of 200K popular ones.
At 2,000 QPS:

Hits:   60% × 2000 = 1200 QPS × 10 ms
Misses: 40% × 2000 =  800 QPS × 250 ms

Weighted avg:
  (1200×10 + 800×250) / 2000 ≈ 106 ms

BARELY meeting p99 < 100 ms — cache hit rate matters massively.
```

---

## 12. Solutions to Cold Cache

### Solution 1: TTL-based caching (default)
- CDN caches responses for X minutes/hours
- First request warms, subsequent requests fast
- Tradeoff: longer TTL = more stale, shorter = more origin hits

### Solution 2: Pre-warming (proactive)
- Script fires requests for top URLs before peak traffic
- Used by e-commerce sales, media events

### Solution 3: Cache warming on publish
- When new URL created, origin pushes to CDN via API
- Complex per-CDN, but effective

### Solution 4: Long TTL + async invalidation
- TTL = 24 hours (long)
- On delete/expire, call CDN API to purge

### Solution 5: Origin Shield ✅ (CDN feature)

```
Without shield:
  N edges have N cache misses → N hits to origin

With shield:
  All edges hit a SHIELD region first
  Shield caches once
  Only 1 miss ever reaches origin

Benefits:
  - Origin sees fewer requests
  - Cache warms once, all edges share
  - Thundering herd at origin prevented
```

Available in: AWS CloudFront, Cloudflare (Argo), Fastly

---

## 13. Latency Optimization Arsenal — Full Menu

From biggest impact → smallest:

| # | Technique | Gain | Notes |
|---|-----------|------|-------|
| 1 | **CDN at edge** | -200 ms | For global users |
| 2 | **Redis cache (in-region)** | -5 ms | Hot path |
| 3 | **Connection pooling** | -60 ms | TLS reuse (first req) |
| 4 | **HTTP/2 or HTTP/3** | -10-20 ms | Concurrent requests |
| 5 | **Compression (gzip)** | -2-5 ms | Larger payloads |
| 6 | **Database indexes** | -5-50 ms | On cache miss |
| 7 | **Async click counter** | -5 ms | Don't block redirect |
| 8 | **Persistent connections** | -30 ms | Avoid reconnect |
| 9 | **Regional deployment** | -100-200 ms | Multi-region |
| 10 | **Pre-computation** | varies | For computed responses |

### Zones of optimization

```
ZONE 1: USER ↔ EDGE (geographic)
  Tools: CDN, regional deployment
  Biggest factor: distance

ZONE 2: EDGE ↔ ORIGIN (cache miss path)
  Tools: connection pooling, HTTP/2, Origin Shield

ZONE 3: ORIGIN WORK (compute + storage)
  Tools: Redis cache, DB indexes, async operations
```

---

## 14. Tail Latency — The p99 Problem

**Average hides outliers. p99 reveals them.**

### Common tail latency causes

1. **Garbage Collection pauses** (JVM)
   - Full GC can pause app 100-500 ms
   - Mitigation: tune GC, use Shenandoah/ZGC

2. **Noisy neighbor** (cloud)
   - Other tenant hogs CPU
   - Mitigation: dedicated hosts

3. **Thread pool saturation**
   - All threads busy, requests queue
   - Mitigation: size pools, use async I/O

4. **Cold cache**
   - First requests hit DB
   - Mitigation: cache warming on deploy

5. **Lock contention**
   - Threads waiting on synchronized code
   - Mitigation: concurrent data structures

6. **Network jitter**
   - Sudden RTT spike
   - Mitigation: hedged requests (fire duplicate, take first)

7. **Slow backend**
   - One slow DB call blocks request
   - Mitigation: timeouts, circuit breakers

### Hedged Requests (advanced technique)

```
If p99 of a backend is much worse than p50:

1. Send primary request
2. After short delay (say p95 latency), send duplicate to another replica
3. Take whichever response arrives first
4. Cancel the other

Used by: Google, used for tail latency reduction.
Tradeoff: more load on backends.
```

---

## 15. Cache Hit Rate — The Key Metric

**Real-world cache hit rates:**

| Quality | Hit Rate |
|---------|----------|
| Perfect (doesn't exist) | 100% |
| Excellent | 95%+ |
| Good | 80-90% |
| Acceptable | 70-80% |
| Bad (investigate) | <50% |

**Golden rule:** Some % of requests ALWAYS miss cache. Design origin to handle full load gracefully.

---

## 16. For URL Shortener — Priority List

```
MUST HAVE (SLA-critical)
=========================
1. CDN at edge (global coverage)
2. Redis cache (in-region origin cache)

NICE TO HAVE (polish)
======================
3. Connection pooling (TLS reuse)
4. HTTP/2
5. gzip compression
6. Async click counter
7. Origin Shield on CDN

DON'T WORRY ABOUT (diminishing returns)
=========================================
- Protobuf vs JSON (small payloads anyway)
- Custom serialization
- JVM tuning (unless hitting GC issues)
```

---

## 17. Calculating Realistic Latencies

### Scenario A: Mumbai user, cache hit at CDN

```
User → Mumbai CDN edge (reused connection): 5 ms
Edge cache HIT: returns directly
TOTAL: ~5 ms ✅ excellent
```

### Scenario B: NY user, cache hit at CDN

```
User → NY CDN edge: 5 ms
Edge cache HIT: returns directly
TOTAL: ~10 ms ✅ excellent
```

### Scenario C: NY user, cache miss at CDN (cold)

```
User → NY edge: 5 ms
NY edge → Mumbai origin: 250 ms
Origin → Redis: 1 ms
Origin → NY edge: 250 ms
NY edge → user: 5 ms
TOTAL: ~511 ms ❌ miss SLA

Mitigation:
  - Origin Shield (collapse N misses → 1)
  - Multi-region (NY origin too)
  - TTL tuning (fewer cold misses)
```

---

## 18. Interview Soundbites

**On latency dominated by network:**
> *"Network is the biggest part of latency, limited by the speed of light through fiber. Mumbai to New York = 250 ms RTT minimum. No code optimization beats physics."*

**On CDN:**
> *"CDN caches immutable redirects at edge locations globally. User hits nearest edge in 5-10 ms. Only cache misses go to origin. For URL Shortener, redirects are perfect for edge caching — same input → same output always."*

**On CDN vs multi-region:**
> *"CDN caches responses at third-party edges — great for cacheable reads. Multi-region deployment puts YOUR infrastructure (DBs, apps) in multiple places — needed for writes, personalization, compliance. Different tools for different problems, often combined."*

**On cold cache:**
> *"CDN only helps for cached URLs. First request from a region pays origin round-trip. Mitigations: Origin Shield collapses N edge misses to 1 origin hit, 1-hour TTL keeps warmed URLs warm, pre-warming for known hot URLs. Target cache hit rate: 80%+."*

**On tail latency:**
> *"P99 is what users actually experience at scale. Common spikes: GC pauses, thread saturation, cold caches, slow backends. Mitigations: async I/O, proper thread pool sizing, timeouts, hedged requests. Never forget: average hides outliers."*

**On meeting p99 < 100 ms globally:**
> *"Two layers. First, geographic — CDN at edge brings global users to ~10 ms. Redirects are immutable, perfect for caching. Second, origin work — Redis cache for sub-ms lookups, connection pooling to skip TLS handshake, async click counter so we don't block the redirect. CDN alone isn't enough — we also keep origin work fast for cache misses."*

---

## 19. Key Terms

| Term | Meaning |
|------|---------|
| **Latency** | Time from request to response |
| **p50 / p95 / p99** | 50th / 95th / 99th percentile latency |
| **Tail latency** | The slow outliers (p99 and beyond) |
| **RTT (Round-Trip Time)** | Time for packet to go to server and back |
| **TLS handshake** | Secure connection setup; costs 2 round-trips |
| **CDN** | Edge servers globally distributed |
| **Edge caching** | Caching at CDN, before origin |
| **Origin server** | Your actual backend (vs CDN edges) |
| **Origin Shield** | CDN feature — extra cache layer between edges and origin |
| **Cold cache** | Cache empty; every request misses |
| **Cache hit rate** | % of requests served from cache (target 80%+) |
| **TTL (Time To Live)** | How long cache keeps a value |
| **Anycast** | Same IP routed to nearest server automatically |
| **Hedged request** | Fire duplicate, take first response |
| **GC pause** | JVM stop-the-world during garbage collection |
| **Connection pooling** | Reuse TCP/TLS connections across requests |

---

## 20. Self-Assessment

### What I understood
- ✅ Latency basics
- ✅ Why p99 matters over average
- ✅ Network dominates latency (physics)
- ✅ Speed of light as physical limit
- ✅ TLS handshake cost
- ✅ CDN role for global users
- ✅ Why CDN works for URL Shortener (immutable responses)
- ✅ CDN vs multi-region distinction
- ✅ Cold cache problem (first user pays tax)
- ✅ Origin Shield mitigation
- ✅ Cache hit rate as a metric

### What to improve
- Initial latency estimates were off on network (too optimistic)
- Will build intuition with more problems

---

## 21. Recap — The 3 Questions You Can Answer Now

**Q1:** What is latency?
> Time from request sent to response received.

**Q2:** Why is network the biggest part of latency?
> Data travels through fiber optic cables limited by speed of light. Distance = time. Mumbai to NY = 250 ms minimum, no optimization beats physics.

**Q3:** How does CDN reduce latency?
> CDN places cached copies of responses at edge locations globally. User connects to the nearest edge, gets response directly — no trip to origin. For cacheable/immutable content (like URL redirects), this is the biggest latency win.

---

## 22. Next Deep Dives in Step 6

```
✅ Latency                    (this file)
⏳ Throughput / bottlenecks
⏳ Fault tolerance
⏳ CAP application
⏳ ID generation
⏳ Rate limiting
⏳ Security
⏳ Observability
```

Latency deep dive — COMPLETE.