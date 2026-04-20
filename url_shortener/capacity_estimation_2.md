# URL Shortener — Step 2: Estimation

**Problem:** Design a URL shortener like bit.ly / tinyurl
**Step:** 2 of 8 (Back-of-envelope estimation)

---

## 1. Estimation Philosophy

**Goal: Order of magnitude, NOT precision.**

Being off by 2x = fine. Being off by 100x = your architecture breaks.

Interviewers don't care about 11,574 vs 10,000 QPS. They care that you don't say 100 when the answer is 100,000.

---

## 2. Core Numbers to Memorize Cold

### Time
```
1 day ≈ 100,000 seconds
(actual: 86,400, round up for easy math)
```

### Storage Units
```
10^3  bytes = 1 KB
10^6  bytes = 1 MB
10^9  bytes = 1 GB
10^12 bytes = 1 TB
10^15 bytes = 1 PB
```

### DAU Anchors
```
Small startup:    100K DAU
Mid-size product: 1M DAU
Large product:    10M DAU
Massive scale:    100M+ DAU (WhatsApp, FB)
```

---

## 3. The 5 Estimation Formulas

**These 5 formulas cover EVERY HLD estimation.**

```
1. QPS        = daily requests / 100,000
2. Peak QPS   = QPS × 2
3. Storage    = records × size × days
4. Bandwidth  = QPS × size per request
5. Cache size = 20% × total storage (80/20 rule)
```

### Formula 1: QPS
Drop 5 zeros from daily requests → QPS.

```
1M/day     → 10 QPS
10M/day    → 100 QPS
100M/day   → 1,000 QPS
1B/day     → 10,000 QPS
```

### Formula 2: Peak QPS
- Traffic isn't flat across 24 hours
- Peak = 2x average (3x for spiky traffic like flash sales)
- **Design for peak, not average** — otherwise system crashes every evening

### Formula 3: Storage
- records/day × bytes/record × retention in days
- **Always watch units** (bytes → KB → MB → GB → TB)

### Formula 4: Bandwidth
- QPS × size per request
- **Ingress** = data coming in (writes/uploads)
- **Egress** = data going out (reads/downloads)

### Formula 5: Cache (80/20 rule)
- 80% of requests hit 20% of data
- Only cache the hot 20%
- Cache size = 20% × total storage

---

## 4. Unit Conversion Trick

**Count the zeros above 9 (for GB):**

```
Answer in bytes → divide by 10^9 for GB

7.5 × 10^11 bytes
Exponent 11 - 9 = 2 extra zeros after GB
= 7.5 × 10^2 GB
= 750 GB ✅
```

**For TB: subtract 12 instead of 9.**

### Common Mistakes
- Confusing `10^11 bytes` with 11 GB (it's actually 100 GB)
- Forgetting to apply the peak multiplier
- Not cross-checking with sanity math

### Sanity Check Every Calculation
```
Storage example:
1M records × 500 bytes  = 500 MB/day
500 MB × 365 days       = ~180 GB/year
180 GB × 5 years        = ~900 GB ✅
```

---

## 5. When to SKIP Bandwidth

**Bandwidth is NOT always a design driver.** Rule of thumb:

### SKIP bandwidth when:
- Response payload < 10 KB
- Read-heavy but payload-light
- Examples: URL Shortener, Rate Limiter, Notification System, ID Generator

### DO calculate bandwidth when:
- Response payload > 10 KB
- Media/video/image heavy
- Massive upload volume
- Examples: YouTube, Instagram, WhatsApp (media), Hotstar, Google Drive

### Bandwidth → Design Decisions Table

| Bandwidth | Design Implication |
|-----------|---------------------|
| < 10 MB/sec | Simple setup, no CDN, no tuning |
| 10 MB – 1 GB/sec | Multiple servers, LB matters |
| > 1 GB/sec | CDN mandatory, geo-distribution |
| > 10 GB/sec | Edge computing, precomputation |

---

## 6. URL Shortener — Full Estimation

### Inputs (Assumptions)

```
Shorten DAU:        1 million
Redirect DAU:       100 million (100:1 read:write)
Retention:          5 years (~1,500 days)
Record size:        ~500 bytes
Response size:      ~500 bytes (302 redirect)
```

### Record Size Breakdown (~500 bytes)

```
short_url       →    7 bytes
long_url        →  ~100 bytes (typical)
user_id         →    8 bytes
created_at      →    8 bytes
expiry_at       →    8 bytes
click_count     →    8 bytes
custom_alias    →   ~20 bytes (optional)
DB overhead     →  ~300 bytes (indexes, metadata)
                ──────────────
Total           ≈  500 bytes
```

### QPS Calculations

```
Shorten QPS  = 10^6 / 10^5     = 10/sec
Redirect QPS = 10^8 / 10^5     = 1,000/sec
```

### Peak QPS

```
Peak Shorten  = 10 × 2   = 20/sec
Peak Redirect = 1000 × 2 = 2,000/sec
```

**Insight:** 2,000 QPS is not huge. URL Shortener is NOT a QPS-bound problem — it's a **storage + low-latency** problem. This drives Step 3 design decisions.

### Storage

```
Storage = 10^6 records/day × 500 bytes × 1,500 days
        = 10^6 × 5 × 10^2 × 1.5 × 10^3
        = 7.5 × 10^11 bytes
        = 750 GB → round up to 1 TB
```

### Bandwidth — Sanity Check (Skipped)

```
Peak egress = 2,000 QPS × 500 bytes = 1 MB/sec
```

1 MB/sec is trivial → bandwidth is NOT a design driver. Skipped.

### Cache Memory

```
Cache = 20% × 750 GB = 150 GB
```

---

## 7. Final Estimation Snapshot

```
╔════════════════════════════════════════╗
║  URL SHORTENER — ESTIMATION            ║
╠════════════════════════════════════════╣
║  Shorten QPS (avg):   10               ║
║  Shorten QPS (peak):  20               ║
║  Redirect QPS (avg):  1,000            ║
║  Redirect QPS (peak): 2,000            ║
║                                        ║
║  Storage (5 years):   ~1 TB            ║
║  Cache size:          150 GB           ║
║  Bandwidth:           Not a driver     ║
╚════════════════════════════════════════╝
```

---

## 8. How Each Number Drives Design (Step 3 Preview)

**This is the key — every number must drive a decision.**

| Number | Design Implication |
|--------|---------------------|
| 2,000 QPS (peak redirect) | Single server can handle with caching. No heavy load balancing needed for reads. |
| 20 QPS (peak shorten) | Writes are trivial. Single primary DB node sufficient. |
| 1 TB storage | Manageable on single DB, but shard for HA + fault tolerance |
| 150 GB cache | Single Redis node maxes ~64-256 GB → need 3-5 node Redis cluster with consistent hashing |
| 100:1 read:write | **Cache-first architecture.** Read replicas > write replicas. |

**Interview soundbite:**
> *"For 150 GB of cache, we need a Redis cluster — single Redis node maxes around 64-256 GB. We'd use 3-5 Redis nodes with consistent hashing. This caches the hot 20% of URLs serving 80% of redirect traffic."*

---

## 9. Interview Best Practices for Estimation

### What to DO

1. **State assumptions openly** — "I'll assume 1M DAU for shortening. Sound reasonable?"
2. **Round for easy math** — 86,400 → 100,000 seconds, 500 bytes not 487 bytes
3. **Cross-check with sanity math** — catch unit errors
4. **Connect numbers to design** — "2,000 QPS means single server handles reads"
5. **Note what's NOT a driver** — "Bandwidth is 1 MB/sec, negligible. Skipping."

### What to AVOID

1. Don't do precise math (1,427,392 bytes) — round
2. Don't calculate bandwidth when payload is tiny
3. Don't calculate numbers you won't use
4. Don't forget peak multiplier
5. Don't mix up GB and TB (common mistake in exam stress)

### The Tier-2 Move

After every calculation, immediately tie it to design:

**Weak:**
> "Storage is 1 TB."

**Strong:**
> "Storage is 1 TB. This fits on a single DB node, but we'll shard for horizontal scaling and fault tolerance, not storage pressure."

---

## 10. Interview Soundbites

**On QPS:**
> *"2,000 QPS peak is modest. URL Shortener is bottlenecked by latency and storage, not throughput."*

**On the 80/20 cache:**
> *"80% of clicks go to the hot 20% of URLs — viral links, marketing campaigns. We only cache that 20%. At 150 GB, that's a Redis cluster, not a single node."*

**On storage:**
> *"1 TB over 5 years is comfortable for sharded SQL or a distributed key-value store. Storage isn't our scaling driver — latency is."*

**On bandwidth being skipped:**
> *"Peak egress is ~1 MB/sec — negligible. I'm skipping bandwidth because it doesn't drive any architectural decision here."*

---

## 11. Self-Assessment

### What I did well
- Identified 100:1 read:write early (consistent with Step 1 NFR)
- Derived QPS from DAU cleanly
- Used 80/20 rule without prompting
- Pushed back on unnecessary bandwidth calculation — senior move

### What to improve
- Unit conversion slip (10^11 → thought 75 GB, actual 750 GB)
- Fix: count zeros above 9 for GB conversion
- Always run sanity check — catches most unit errors

### Step 2 grade: 8/10

---

## 12. Next Step Preview — High-Level Design

With estimation locked, we now design:

1. **API Gateway + Load Balancer** — handles 2,000 QPS easily
2. **Application servers** — stateless, horizontally scalable
3. **Cache layer** — Redis cluster (3-5 nodes, 150 GB capacity)
4. **Primary DB** — sharded SQL or NoSQL (1 TB, read replicas)
5. **ID generation** — base62 encoder (covered in depth Step 3)
6. **Analytics pipeline** — async counter (not blocking redirects)

Design will be driven by the numbers above — nothing more, nothing less.