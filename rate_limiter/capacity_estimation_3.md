# HLD: Rate Limiter — Chunk 3: Back of Envelope + Why Redis

---

## Scale Assumptions

| Metric | Value |
|--------|-------|
| Total requests | 10M/sec |
| Peak load (3x) | 30M/sec |
| Active clients per day | 10M |
| Concurrent clients at any moment | 1M (assuming 10% concurrency) |
| Operations per request | 2 (1 read + 1 write) |
| Peak operations/sec | 30M × 2 = **60M ops/sec** |

---

## Storage Estimation Per Client

### How Redis stores rate limit state

**Algorithm determines structure** — not one-size-fits-all.

### Fixed Window / Sliding Window Counter
```
KEY:   ratelimit:{client_id}:{resource}:{window_timestamp}
VALUE: 42   ← just an integer counter
TTL:   set via EXPIRE (e.g. 60s)
```

`window_timestamp` = start of current time bucket  
```
window_start = floor(current_time / window_size) * window_size
```
All requests in the same window share the same key → same counter.

### Token Bucket
```
KEY:   ratelimit:{client_id}:{resource}
VALUE: { tokens: 7.5, last_refill: 1703980842 }
TTL:   expires after inactivity
```
No window concept — bucket refills continuously.  
`last_refill` stored in value because every request needs:
```
time_elapsed  = now - last_refill
tokens_to_add = time_elapsed × refill_rate
tokens        = min(capacity, old_tokens + tokens_to_add)
```

### Per-Client Memory Breakdown

| Field | Size |
|-------|------|
| Key (client_id + resource + window) | ~50 bytes |
| Value (counter) | ~8 bytes |
| TTL + Redis internal metadata | ~34 bytes |
| **Total per client** | **~100 bytes** |

**1M concurrent clients × 100 bytes = 100 MB**  
→ Easily fits in memory. No disk needed.

---

## Why Redis? (Elimination Round)

### ❌ SQL (PostgreSQL/MySQL)
- Disk I/O on every request — **p99 < 5ms impossible** at 10M req/sec
- No built-in TTL/expiry
- ACID, joins, schema — none of this is needed here
- **Verdict: Too slow, wrong tool**

### ❌ Cassandra / NoSQL
- Still disk-based — latency better than SQL but ~1-5ms just for storage
- Atomic increment support is limited
- 100MB data ke liye pura Cassandra cluster = over-engineered
- **Verdict: Overkill for this data size**

### ❌ Local Application Memory (HashMap)
- Distributed system mein 4 servers = 4 separate counters → clients bypass limits easily
- Data lost on server restart
- No TTL support
- **Verdict: Breaks in distributed setup**

### ✅ Redis — Perfect Fit

| Requirement | How Redis Satisfies It |
|-------------|----------------------|
| Sub-millisecond latency | In-memory — ~0.1ms reads/writes |
| Atomic operations | `INCR` is atomic; Lua scripts for complex atomicity |
| Auto expiry | `EXPIRE` built-in — no manual cleanup needed |
| 100MB data | Trivially fits in memory |
| Distributed | Redis Cluster — horizontal sharding across nodes |
| High availability | Master-replica replication |

---

## Key Insight for Interview

> Rate limiter is a **read-heavy, latency-sensitive** system where **all data fits in memory**.  
> This is Redis's sweet spot.  
> If data were 100GB → Redis gets expensive → consider Cassandra.  
> At 100MB → Redis is the obvious, justified choice.

---

*Next: Chunk 4 — High Level Design*