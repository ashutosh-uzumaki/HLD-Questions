# HLD: Rate Limiter — Chunk 8: Distributed Challenges

---

## Overview

3 main challenges in distributed rate limiting:
1. Race Conditions
2. Clock Skew
3. Cross-DC Synchronization

---

## Challenge 1: Race Conditions

### Problem

Multiple Rate Limiter instances reading + incrementing same counter simultaneously.

```
Counter = 99, Limit = 100

Instance 1: reads 99 → increments → 100 → allows ✅
Instance 2: reads 99 → increments → 101 → allows ✅ (WRONG!)

Final counter = 101 — limit exceeded! 😱
```

### Why INCR alone doesn't fix it

```
Instance 1: INCR → 100 (atomic ✅)
Instance 2: INCR → 101 (atomic ✅)
Instance 2: check 101 <= 100? → reject

Counter = 101 — already incremented before check!
```

INCR is atomic — but **INCR + CHECK** together are not.

### Solution: Redis Lua Script

Lua scripts execute **atomically** in Redis — entire script runs as single operation. Nothing can interrupt in between.

```lua
-- KEYS[1] = counter key
-- ARGV[1] = window TTL in seconds
-- ARGV[2] = limit

local count = redis.call('INCR', KEYS[1])
redis.call('EXPIRE', KEYS[1], ARGV[1])

if count > tonumber(ARGV[2]) then
    return 0  -- reject
else
    return 1  -- allow
end
```

INCR + CHECK = one atomic operation → no race condition ✅

### Sliding Window Counter Lua Script

```lua
-- KEYS[1] = current window key
-- KEYS[2] = previous window key
-- ARGV[1] = limit
-- ARGV[2] = window size in seconds
-- ARGV[3] = current timestamp

local curr_count = tonumber(redis.call('GET', KEYS[1]) or 0)
local prev_count = tonumber(redis.call('GET', KEYS[2]) or 0)
local window_size = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local window_start = math.floor(now / window_size) * window_size
local elapsed = now - window_start
local overlap = 1 - (elapsed / window_size)
local weighted = (prev_count * overlap) + curr_count

if weighted < tonumber(ARGV[1]) then
    redis.call('INCR', KEYS[1])
    redis.call('EXPIRE', KEYS[1], window_size * 2)
    return {1, weighted + 1}  -- allowed
else
    return {0, weighted}       -- rejected
end
```

---

## Challenge 2: Clock Skew

### Problem

Window timestamp is calculated from **application server clock** — but clocks on different servers can differ slightly.

```
Server 1 clock: 12:00:59.900  ← thinks window 12:00 still active
Server 2 clock: 12:01:00.050  ← thinks window 12:01 started

Same request on different servers:
Server 1 → key: ratelimit:user_123:/api/orders:1703980800  (window 12:00)
Server 2 → key: ratelimit:user_123:/api/orders:1703980860  (window 12:01)
```

**Two different keys = two different counters** → client gets double quota at boundary! 😱

### Solution: Redis TIME Command

Don't rely on application server clocks — use **Redis server time** for all timestamp calculations.

```lua
local time = redis.call('TIME')
local timestamp = time[1]  -- Unix timestamp from Redis server
```

All Rate Limiter instances use same Redis cluster → same timestamp for everyone → no clock skew. ✅

---

## Challenge 3: Cross-DC Synchronization

### Problem

Enterprise clients (e.g. Amazon) have servers in multiple data centers using the same API key.

```
Amazon Mumbai    → Razorpay Mumbai DC    → Redis Mumbai    → 60 req/min
Amazon Singapore → Razorpay Singapore DC → Redis Singapore → 60 req/min

Combined = 120 req/min — but global limit = 100! 😱
```

### Solution 1: Centralized Redis Cluster ❌ (Too slow)

All DCs use one global Redis.

```
Mumbai DC    → Rate Limiter → Global Redis (Singapore)
Singapore DC → Rate Limiter → Global Redis (Singapore)
```

**Pros:** Perfect accuracy ✅
**Cons:** Mumbai → Singapore = ~150ms latency. Rate limiter budget = 5ms. Not feasible. ❌

---

### Solution 2: Split Limits ✅ (Simple default)

Divide global limit across data centers.

```
Global limit = 100 req/min, 2 DCs
Mumbai DC    → 50 req/min (local Redis)
Singapore DC → 50 req/min (local Redis)
```

**Pros:** No cross-DC coordination, fast ✅
**Cons:** Unfair if traffic unevenly distributed — Mumbai idle = Singapore can't use Mumbai's quota

---

### Solution 3: Async Sync with Local Enforcement ✅ (Best balance)

Each DC maintains local counters + periodically syncs globally.

```
Step 1: Each DC uses local counter for decisions (fast!)
Step 2: Every 5-10 seconds → aggregate global count
Step 3: If global count approaches limit → local limiters become stricter
```

**Pros:** Low latency ✅, reasonable accuracy ✅
**Cons:** Can temporarily exceed limits during sync window — approximate

---

### Solution 4: Sticky Sessions (Simple but availability risk)

Route all requests from same client to same DC using consistent hashing on client ID.

```
user_123 → always → Mumbai DC
user_456 → always → Singapore DC
```

**Pros:** Simple, no sync needed ✅
**Cons:** If Mumbai DC goes down → user_123 affected. Reduces availability. ❌

---

## Recommended Interview Answer

> "For cross-DC sync, main **Split Limits** as default use karunga — simple aur fast. Agar traffic unevenly distributed ho toh **Async Sync** approach prefer karunga — background mein global count har 5-10 seconds mein sync karta rahunga. Slight approximation acceptable hai for performance."

---

## Chunk 8 Summary

| Challenge | Root Cause | Solution |
|-----------|-----------|---------|
| Race Condition | Read + increment not atomic | Redis Lua Script — atomic INCR + CHECK |
| Clock Skew | Different server clocks → different window keys | Redis TIME command — use Redis server time |
| Cross-DC Sync | Separate counters per DC | Split Limits (default) or Async Sync |

---

*Next: Chunk 9 — Rate Limiter Placement + Failure Handling (Fail open/closed, Circuit Breaker)*