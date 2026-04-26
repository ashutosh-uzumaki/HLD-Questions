# HLD: Rate Limiter — Chunk 10: Monitoring + Response Headers

---

# Part 1: Response Headers

When rate limit responses are returned, include headers that help clients understand their limits.

## Standard Headers

```http
X-RateLimit-Limit:     100          ← max requests allowed per window
X-RateLimit-Remaining: 42           ← requests remaining in current window
X-RateLimit-Reset:     1703980800   ← unix timestamp when window resets
Retry-After:           30           ← seconds to wait before retry (429 ONLY)
```

### Naming Convention Rules
- Rate limit headers use `X-RateLimit-` prefix
- `Retry-After` has NO `X-` prefix — it is an official HTTP spec header
- `X-RateLimit-Reset` = Unix timestamp (exact time)
- `Retry-After` = seconds (duration) — only present on 429 responses

---

## Example 200 Response (Allowed)

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1703980800

{ "data": "..." }
```

---

## Example 429 Response (Blocked)

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1703980800
Retry-After: 30

{
  "error": "rate_limit_exceeded",
  "message": "You have exceeded your rate limit. Retry after 30 seconds.",
  "retry_after": 30
}
```

---

## Logging vs Monitoring — Key Difference

| | Logging | Monitoring |
|--|---------|-----------|
| What | Record of every request — who, what, when | System health metrics |
| Purpose | Audit trail, debugging | Alerting, capacity planning |
| Example | "user_123 hit /api/orders at 13:01:05" | "p99 latency = 8ms" |

---

# Part 2: Monitoring Metrics

## Key Metrics to Track

| Metric | Description | Alert Threshold |
|--------|-------------|----------------|
| `rate_limit.allowed` | Requests allowed per second | N/A — baseline |
| `rate_limit.rejected` | Requests rejected per second | > 1% of traffic |
| `rate_limit.latency_p99` | 99th percentile check latency | > 10ms |
| `rate_limit.redis_errors` | Redis connection failures | > 0 per minute |
| `rate_limit.fallback_used` | Fallback rate limiter activations | > 0 |

---

## Alerting Rules

**1. High Rejection Rate**
```
> 5% requests rejected
→ Investigate potential DDoS or misconfigured limits
```

**2. Latency Spike**
```
p99 > 10ms
→ Redis might be overloaded or network issues
```

**3. Redis Unavailable**
```
Redis errors > 0
→ Failover is active — investigate immediately
```

**4. Single Client Domination**
```
One client > 50% of all rejected requests
→ Possible abuse or bot attack
```

---

# Complete Rate Limiter — Revision Summary

## System at a Glance

```
Client → Load Balancer → API Gateway
                              ↓ POST /ratelimit/check
                         Rate Limiter Service
                           ├── Rules Cache (lookup rules)
                           └── Redis Cluster (INCR/GET counters)
                              ↓ allowed         ↓ blocked
                         Backend Services    429 Response

Admin → Rules Service → Rules DB (PostgreSQL)
              └── notify → Rules Cache (refresh)
```

---

## Quick Reference

### APIs
| API | Purpose |
|-----|---------|
| `POST /ratelimit/check` | Check if request allowed — called on every request |
| `GET /ratelimit/status/{client_id}` | Read-only status check — no quota consumed |
| `PUT /ratelimit/rules` | Admin — configure rate limit rules |

### Redis Key Structure
```
Fixed/Sliding Window: ratelimit:{client_id}:{resource}:{window_timestamp}
Token Bucket:         ratelimit:{client_id}:{resource}
```

### Algorithm Selection
| Need | Algorithm |
|------|-----------|
| Default / General purpose | Sliding Window Counter |
| Allow burst traffic | Token Bucket |
| Constant output rate | Leaky Bucket |
| Exact accuracy, small scale | Sliding Window Log |
| Simplest possible | Fixed Window Counter |

### Failure Strategy
```
Primary:    Redis Cluster (accurate)
Fallback 1: Local in-memory (approximate)
Fallback 2: Fail open + alert
```

### Circuit Breaker States
```
CLOSED    → Redis healthy → normal rate limiting
OPEN      → Redis down   → local cache or fail open
HALF-OPEN → recovering   → test few requests on Redis
```

### Distributed Challenges
| Challenge | Solution |
|-----------|---------|
| Race condition | Redis Lua Script — atomic INCR + CHECK |
| Clock skew | Redis TIME command |
| Cross-DC sync | Split limits (default) or Async sync |

---

*Rate Limiter Complete! ✅*
*Next Problem: WhatsApp / Chat System*