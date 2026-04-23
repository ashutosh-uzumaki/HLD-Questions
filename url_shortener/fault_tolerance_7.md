# URL Shortener — Step 6: Fault Tolerance Deep Dive

**Problem:** Design a URL shortener like bit.ly / tinyurl
**Step:** 6 of 8 (Deep Dives — Topic 3: Fault Tolerance)
**Teaching style:** Every concept built simple → technical → interview-ready

---

## 1. Why Fault Tolerance Matters

**Simple fact:** Everything fails eventually.

Servers crash. Networks blip. Disks fail. Deploys go bad. Data centers lose power.

If your system doesn't tolerate failure → downtime. Downtime → missed SLA → unhappy users → lost revenue.

### Our NFR
```
99.99% availability = ~52 min downtime/year
```

To hit that number, fault tolerance must be **baked in from the start**, not added later.

---

## 2. Types of Failures — The Full Map

```
FAILURE CATEGORIES (ranked by frequency)
==========================================

1. DEPLOYMENT / CONFIG FAILURES       ~60-70%
   - Bad code deploy
   - Config change broke things
   - Schema migration gone wrong

2. SINGLE NODE / HARDWARE FAILURES    ~15-20%
   - App server dies
   - Cassandra node crashes
   - Disk fails

3. NETWORK FAILURES                   ~5-10%
   - Network partition (split brain)
   - Packet loss
   - High latency

4. DEPENDENCY FAILURES                ~5-10%
   - CDN down
   - DNS failure
   - External service timeout

5. CAPACITY FAILURES                  ~3-5%
   - DDoS
   - Traffic spike
   - Bot traffic

6. DATA CORRUPTION                    ~1-2%
```

**Interview focus:** types 2-5 (design concerns). Type 1 is DevOps domain, usually not asked in HLD.

### Real-world insight
Most outages come from **humans changing things**. That's why industry obsesses over:
- Blue-green deployments
- Canary releases
- Feature flags
- Rollback automation

---

## 3. The 3 Layers of Fault Tolerance

Every fault-tolerant system has these 3 layers:

### Layer 1: REDUNDANCY
> Have extra copies so one failure doesn't kill the system.

```
NEVER have a single point of failure (SPOF).
```

### Layer 2: DETECTION
> Know quickly when something fails.

```
Detect failures in SECONDS, not hours.
```

### Layer 3: RECOVERY
> Heal without manual intervention.

```
System heals itself automatically.
```

---

## 4. What We Already Built Into URL Shortener

Fault tolerance isn't a separate feature — it's already implicit in earlier steps. Let me make it explicit:

### Layer 1: Redundancy (built in from Step 3)

| Component | Redundancy |
|-----------|------------|
| Load Balancer | Managed service, auto-scales |
| App Servers | 3-5 stateless instances |
| Redis | Redis Cluster (3-5 nodes) |
| Cassandra | 5 nodes with RF=3 (each row on 3 nodes) |
| CDN | Thousands of edge locations |

**Key:** Any single node can die, system stays up.

### Layer 2: Detection (built into tools)

| Mechanism | Where |
|-----------|-------|
| Health checks | Load Balancer pings app servers |
| Gossip protocol | Cassandra nodes gossip every second |
| Heartbeats | Redis Sentinel monitors master/replica |
| Metrics + alerts | Prometheus / CloudWatch |

**Key:** Failures detected in ~10 seconds without human intervention.

### Layer 3: Recovery (auto-healing)

| Mechanism | How it recovers |
|-----------|-----------------|
| LB removes dead server | Traffic redirected to live ones |
| Cassandra hinted handoff | Missed writes replayed when node returns |
| Redis Cluster failover | Replica promoted if master dies |
| Stateless apps | Any request goes to any server |

---

## 5. The Big Insight — It's About NOT Being Stupid

```
"Most fault tolerance design is about NOT doing stupid things."

Stupid:
  ❌ Single Redis node
  ❌ RF=1 in Cassandra
  ❌ Sticky sessions on app servers
  ❌ Synchronous calls to all downstreams
  ❌ No timeouts
  ❌ No circuit breakers

Smart:
  ✅ Redis Cluster (not single node)
  ✅ RF=3 (not RF=1)
  ✅ Stateless app servers
  ✅ Async where possible (Kafka)
  ✅ Timeouts everywhere
  ✅ Circuit breakers on every external call
```

**When interviewer asks about fault tolerance, show awareness of these traps.**

---

## 6. Circuit Breaker Pattern ⭐ (Must Know)

### The problem it solves

```
WITHOUT CIRCUIT BREAKER
========================

App → calls Redis (slow, not dead)
Each request waits 30 sec for timeout.

More requests pile up...
Threads all blocked waiting on Redis...
Thread pool exhausted...
App appears "down" to LB...
LB removes app from pool...
More load on remaining apps...

CASCADING FAILURE — entire cluster dies 💥
```

This is what killed many companies' production systems. Circuit breaker is the fix.

### How circuit breaker works — 3 states

```
CLOSED (normal)
================
Requests flow through.
Breaker monitors failure rate.

If failure rate > threshold (e.g., 50% in 10 sec):
  → Trip to OPEN state

OPEN (circuit broken)
======================
ALL requests rejected IMMEDIATELY (0 ms).
Don't even try to call the sick service.
Return error or fallback response.

After timeout (e.g., 30 sec):
  → Move to HALF-OPEN

HALF-OPEN (testing recovery)
==============================
Let ONE request through as a test.

Success → CLOSE (service recovered)
Failure → back to OPEN (still sick)
```

### Analogy
Think of electrical circuit breaker in your home:
- Too much current → flips → circuit off
- Prevents fire (cascading damage)

Software circuit breaker = same idea. Too many failures → "open" → stop calling.

### Applied to URL Shortener

```
CIRCUIT BREAKER LOCATIONS
==========================

App → Redis:
  If Redis fails 50% in 10 sec → OPEN
  For 30 sec, skip Redis, go straight to Cassandra
  Degrades gracefully (slower but works)

App → Cassandra:
  If Cassandra fails → OPEN
  Return error OR serve from Redis cache only
  Accept partial outage

App → Kafka (click events):
  If Kafka fails → OPEN
  Drop click events (non-critical)
  Redirect still works
```

### Tools

| Language | Library |
|----------|---------|
| Java | **Resilience4j** (modern, recommended) |
| Java (legacy) | Hystrix (Netflix, deprecated) |
| Go | gobreaker, hystrix-go |
| Infrastructure | Istio (service mesh does it per service) |

### Why "fail fast" beats "retry endlessly"

```
Fail fast prevents THREAD EXHAUSTION.

If we wait 30s per request:
  Threads stay blocked
  Thread pool fills up
  Service appears down

If we fail in 0ms:
  Thread freed instantly
  Service handles next request
  Service stays healthy even when downstream is sick
```

**Lock the term: "Thread exhaustion."**

### Interview soundbite
> *"Circuit breaker pattern. If Redis fails above threshold — say 50% errors in 10 seconds — the breaker opens and we stop calling it for 30 seconds. Prevents cascading failures where one sick service exhausts our thread pool. Half-open state tests recovery. We'd use Resilience4j in Java. Netflix popularized this with Hystrix."*

---

## 7. Retry with Backoff ⭐ (Must Know)

### When to retry

```
RETRY WHEN:
============
✅ Transient failures (network blip, brief overload)
✅ Eventually-consistent systems (replica lag)

DON'T RETRY WHEN:
==================
❌ Downstream is overloaded
   (retries make it worse — "retry storm")
❌ Request is NOT idempotent
   (retrying might double-create resources)
```

### The naive retry problem — Retry Storm

```
BAD PATTERN: retry immediately
================================

Call fails → retry
Still fails → retry
Still fails → retry

If downstream is overloaded:
  1 failed request → 10 requests (after retries)
  10 failed → 100 (more retries)
  
Exponential load increase → downstream dies harder.

This is RETRY STORM. 🔥
```

### Fix 1: Exponential Backoff

```
Double the delay each retry:

Attempt 1: immediate
Attempt 2: wait 100ms
Attempt 3: wait 200ms
Attempt 4: wait 400ms
Attempt 5: wait 800ms
Attempt 6: wait 1600ms
...

Gives downstream time to recover.
```

### Fix 2: Add Jitter (even better)

**Problem with pure backoff:** 1000 clients fail at same time → all retry at 100ms together → ANOTHER spike.

**Fix: randomness spreads retries in time.**

```
EXPONENTIAL BACKOFF + JITTER
==============================

Attempt 1: immediate
Attempt 2: wait (100ms + random 0-100ms)
Attempt 3: wait (200ms + random 0-200ms)
Attempt 4: wait (400ms + random 0-400ms)

1000 clients retry at different times → no wave.
```

### Applied to URL Shortener

```
RETRY POLICY
=============

App → Redis:
  Max retries: 3
  Backoff: 50ms, 100ms, 200ms (with jitter)

App → Cassandra:
  Max retries: 2
  Backoff: 100ms, 300ms

App → External service:
  Max retries: 5
  Backoff: 100ms to 5s (exponential)

CRITICAL RULE:
  Only retry IDEMPOTENT operations.
  GET: safe
  PUT/DELETE: safe
  POST shorten: needs Idempotency-Key to be retry-safe
```

### Interview soundbite
> *"Retries use exponential backoff with jitter. Doubling delay prevents retry storms that overload recovering services. Jitter spreads timing so 1000 clients don't retry simultaneously. Max 3-5 retries before giving up. Only retry idempotent operations — non-idempotent POST requests need Idempotency-Key to be safe."*

---

## 8. Bulkhead Pattern (Tier-2 Vocab)

### The problem

```
One slow downstream exhausts ALL app server threads.
Even unrelated requests fail.
```

### The fix — isolate thread pools

```
BULKHEAD PATTERN
==================

Separate thread pools for each downstream:

App Server:
  Redis calls pool:     10 threads max
  Cassandra pool:       10 threads max
  Kafka pool:           5 threads max

If Redis is slow:
  → Only Redis pool saturates
  → Cassandra/Kafka pools still work
  → Other requests still served

Ship bulkheads analogy:
  One breach doesn't sink the ship.
  Compartments isolated.
```

### Used by
- Netflix (Hystrix had this built-in)
- Most production Java services
- Service meshes (Istio per-service isolation)

### Interview soundbite
> *"Bulkhead pattern isolates thread pools per downstream. If Redis is slow, only Redis's pool saturates — Cassandra calls and unrelated paths still work. Named after ship compartments — one breach doesn't sink the ship."*

---

## 9. Graceful Degradation ⭐

### Core principle

```
When something fails, serve a REDUCED response
rather than failing completely.
```

### Critical vs optional paths

```
URL SHORTENER PATH CLASSIFICATION
===================================

CRITICAL PATH (must work):
  - Redirect (GET /{short_url}) — core product
  - Shorten (POST /urls) — core product
  
  → Needs redundancy
  → Cannot fail

OPTIONAL PATH (can fail gracefully):
  - Click counter
  - Analytics
  - Notifications
  - Admin dashboard
  
  → Can fail silently
  → System still works
```

### Degradation scenarios

| Component Down | Normal Behavior | Degraded Behavior |
|---------------|-----------------|-------------------|
| Redis cache | App → Redis (1ms) | App → Cassandra (5ms) — slower but works |
| Kafka (click) | Publish click event | Drop click event — redirect still works |
| Analytics service | Dashboard shows stats | Dashboard shows "unavailable" |
| Cassandra | Reads hit DB on cache miss | Serve from Redis only (~90% success) |

### Real-world examples

```
NETFLIX
  Recommendations fail → show popular movies
  Play history fails → still can watch
  Ratings fail → hide ratings section

UBER
  Surge pricing fails → show base price
  ETA fails → show "calculating..."
  Map tiles fail → show basic map

FLIPKART
  Reviews fail → hide reviews section
  Price comparison fails → show base price
  Recommendations fail → show category browse
```

### Design rule

```
Ask for EVERY dependency: "What if this fails?"

If critical path → need redundancy + fallback
If optional path → fail silently with degraded response

NEVER let an optional service take down a critical path.
```

### Interview soundbite
> *"Graceful degradation — critical paths have redundancy, optional features fail silently. For URL Shortener, redirect is critical, click counter is optional. If Kafka is down, I still redirect — just lose that click. If Redis is down, I fall back to Cassandra — slower but works. If Cassandra is down, I serve from Redis cache — ~90% success. Netflix does this — recommendations fail, you still watch movies."*

---

## 10. Timeout Configuration

Often overlooked. Setting timeouts correctly is fault tolerance 101.

```
TIMEOUT LAYERS
================

Request coming in to app:
  Total request timeout: 5 sec (end-to-end budget)

App → Redis:
  Connection timeout: 100ms
  Read timeout: 200ms
  
App → Cassandra:
  Connection timeout: 500ms
  Read timeout: 1 sec
  
App → External service (rare):
  Connection timeout: 2 sec
  Read timeout: 3 sec
```

### Key rule

```
Downstream timeout < my timeout

If my total budget is 5 sec:
  No single downstream can take > 5 sec
  Account for retries in the budget

Example math:
  My budget: 5000ms
  Redis call: 200ms × 3 retries = 600ms
  Cassandra: 1000ms × 2 retries = 2000ms
  Total: 2600ms (within budget) ✓
```

### Never have infinite timeouts

```
❌ Default HTTP client timeout = infinity (dangerous)
✅ Always set explicit timeouts
```

### Interview soundbite
> *"Every external call has explicit connection + read timeouts. Downstream timeout must be less than my own request budget, accounting for retries. Never infinite timeouts — threads stay blocked forever, exhausting pool."*

---

## 11. Connecting All Patterns Together

A full fault-tolerant call looks like this:

```
App Server calls Redis:

1. Check circuit breaker state
   If OPEN → skip, return fallback (go to Cassandra)
   If CLOSED → proceed

2. Use bulkhead thread pool (Redis-specific)
   If pool exhausted → reject, return fallback

3. Set timeout (connection + read)
   If timeout → throw exception

4. Execute call
   If failure → record in circuit breaker

5. On failure:
   - Increment circuit breaker counter
   - Retry with exponential backoff + jitter
   - Max 3 retries
   - After retries exhausted → return fallback

6. On success:
   - Reset circuit breaker counter
   - Return response
```

**5 patterns in one call:** Circuit Breaker + Bulkhead + Timeout + Retry with Backoff + Graceful Fallback.

---

## 12. CAP Theorem Tie-In

Fault tolerance forces CAP decisions per component:

```
NETWORK PARTITION HAPPENS
==========================

Redis cluster splits:
  Option A: Keep serving from available shards (AP)
            Risk: stale/incorrect data
  Option B: Reject requests on affected shards (CP)
            Risk: unavailability

Cassandra with RF=3:
  Option A: Serve reads with CL=ONE (AP)
            May be stale during partition
  Option B: Require CL=QUORUM (CP)
            Fails if 2 replicas unreachable

Our URL Shortener choice:
  Writes: CL=QUORUM (lean CP)
  Reads: CL=ONE (lean AP, stale OK)
  → Per-operation CAP tuning
```

**We discussed this in Step 5.** Fault tolerance = deciding CAP per operation.

---

## 13. Chaos Engineering (Bonus — Tier-1 Concept)

**Name-drop for senior-level signaling.**

```
CHAOS ENGINEERING
==================

Pioneered by Netflix (Chaos Monkey, 2011).

Idea: DELIBERATELY inject failures in production.
  - Random server kills
  - Network delays
  - CPU throttling
  - DB latency spikes

Why?
  If you only see failures in production emergencies,
  you fear them.
  If you inject them weekly, you design for them.

Tools:
  - Chaos Monkey (Netflix)
  - Gremlin (commercial)
  - Litmus (Kubernetes native)

Tier-2 mention:
  "I'd advocate for chaos engineering post-MVP —
   regular fault injection ensures our fault-tolerance
   mechanisms actually work in production."
```

**Don't go deep in interview unless asked. But knowing it exists = senior signal.**

---

## 14. Checklist for Every External Call

Use this mental checklist for every component-to-component call:

```
□ Timeout set (connection + read)?
□ Retry policy (exponential backoff + jitter)?
□ Circuit breaker wrapping the call?
□ Fallback/degraded response defined?
□ Idempotent (for retries to be safe)?
□ Thread pool isolated (bulkhead)?
□ Error logged for observability?
□ Metrics emitted (latency, errors, timeouts)?
```

**A call without these = production outage waiting to happen.**

---

## 15. Named Patterns Summary (Interview Vocabulary)

```
PATTERN                  | WHAT IT DOES                  | WHEN TO USE
══════════════════════════════════════════════════════════════════════════════
Circuit Breaker          | Fail fast when downstream sick | Always for external calls
Retry with Backoff       | Recover from transient fails   | Idempotent ops only
Bulkhead                 | Isolate failure (thread pools) | Multiple downstreams
Graceful Degradation     | Reduced > broken               | Optional features
Timeout                  | Cap wait time                  | EVERY external call
Idempotency              | Safe retries                   | Any write operation
Dead Letter Queue (DLQ)  | Capture failed messages        | Async processing
```

**These are the words interviewers listen for.**

---

## 16. Interview Soundbites

**On fault tolerance overall:**
> *"Three layers — redundancy (multiple nodes/replicas), detection (health checks, gossip), recovery (auto-failover, hinted handoff). Plus four patterns on every external call: circuit breaker, retry with backoff, bulkhead isolation, and graceful degradation for optional features."*

**On circuit breaker:**
> *"Circuit breaker prevents cascading failures. If Redis fails 50% in 10 seconds, breaker opens and we skip it for 30 seconds, returning fallback. Half-open tests recovery. Resilience4j in Java, Hystrix from Netflix popularized this."*

**On retry:**
> *"Exponential backoff with jitter. Doubling delay prevents retry storms; jitter spreads timing so concurrent clients don't retry simultaneously. Max 3-5 retries, only on idempotent ops."*

**On graceful degradation:**
> *"Critical paths have redundancy, optional features fail silently. For URL Shortener, redirect is critical, click counter is optional. Kafka down? Drop the click event — user still gets redirected. Netflix does this — recommendations fail, you still watch movies."*

**On CAP:**
> *"Fault tolerance forces per-operation CAP choices. Our writes use QUORUM (lean CP — correctness). Reads use ONE (lean AP — speed, stale OK). Different operations, different tradeoffs in the same system."*

---

## 17. Self-Assessment

### What I covered
- ✅ Types of failures (6 categories)
- ✅ 3 layers: redundancy, detection, recovery
- ✅ What's already built in (redundancy everywhere)
- ✅ Circuit Breaker (3 states, cascading failure prevention)
- ✅ Retry with exponential backoff + jitter
- ✅ Bulkhead pattern
- ✅ Graceful degradation (critical vs optional)
- ✅ Timeout configuration
- ✅ CAP theorem tie-in
- ✅ Chaos engineering (bonus name-drop)
- ✅ Named patterns vocabulary

### Key insight locked
- Thread exhaustion is the real killer
- Fault tolerance = NOT being stupid + applying patterns
- Patterns combine — one call uses 5 patterns together

### Grade: 9/10

---

## 18. Key Terms

| Term | Meaning |
|------|---------|
| **SPOF** | Single Point of Failure |
| **Cascading Failure** | One failure propagating through the system |
| **Thread Exhaustion** | All threads blocked waiting on slow downstream |
| **Retry Storm** | Synchronized retries overwhelming recovering service |
| **Circuit Breaker** | Pattern to fail fast on repeated failures |
| **Bulkhead** | Pattern to isolate failure via separate pools |
| **Graceful Degradation** | Reduced service instead of total failure |
| **Hinted Handoff** | Cassandra feature to replay missed writes |
| **Exponential Backoff** | Doubling delay between retries |
| **Jitter** | Randomness added to timing to prevent waves |
| **Chaos Engineering** | Deliberate fault injection to test resilience |
| **DLQ** | Dead Letter Queue — capture for failed messages |

---

## 19. Connection to Earlier Steps

| Step | Fault Tolerance Connection |
|------|---------------------------|
| Step 1 NFR | 99.99% availability = drives everything |
| Step 3 HLD | Redundancy (3+ app servers, Redis Cluster) |
| Step 4 API | Idempotency-Key = safe retries |
| Step 5 DB | RF=3 = survives 2 node failures, QUORUM for writes |
| Step 5 Replication Lag | Route idempotency reads to primary |
| Step 6 Latency | Origin Shield prevents thundering herd |
| Step 6 Throughput | Memory limits force sharding (no SPOF) |

**Fault tolerance ties all previous steps together.**

---

## 20. Next Deep Dive — ID Generation

Next up: **How does a 7-char base62 short URL actually get generated?**

Topics we'll cover:
1. Counter-based vs random vs hash approaches
2. Base62 encoding math (why 7 chars = 3.5 trillion URLs)
3. Snowflake-style distributed IDs
4. Collision handling
5. Custom alias namespace strategy
6. Leader election for machine_id assignment

**This topic transfers ~70% to Problem 3 (Unique ID Generator).** Deep coverage here saves time later.