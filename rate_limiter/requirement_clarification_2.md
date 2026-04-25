# HLD: Rate Limiter — Chunk 2: Clarifying Questions & Requirements

---

## Clarifying Questions to Ask in Interview

### Q1 — What is the request limit per client?
Drives scale estimation. Are we talking 100 req/min or 10,000 req/sec?

### Q2 — What identifier are we using to track clients?
IP address, user ID, API key, phone number?
→ Directly impacts Redis key design: `ratelimit:{identifier}:{resource}:{window}`

### Q3 — Is the limit global or per-service/per-endpoint?
Can each API endpoint define its own limit?
→ Leads to "configurable rules" component in design.

### Q4 — Do we allow burst traffic?
Can a client use saved-up quota all at once?
→ Drives algorithm choice: Token Bucket allows bursts, Fixed Window doesn't.

### Q5 — What happens when limit is exceeded? ⚠️ Don't miss this
Silent drop, or return error with headers?
→ Answer: HTTP 429 with `Retry-After` and `X-RateLimit-Remaining` headers.

### Q6 — What if the rate limiter itself goes down? ⚠️ Don't miss this
Fail open (allow all traffic through) or fail closed (block all)?
→ Classic availability tradeoff. Most APIs choose **fail open** — availability > strict enforcement.

---

## Summarized Requirements

### Functional Requirements
| # | Requirement |
|---|-------------|
| 1 | Limit requests per client within a configurable time window |
| 2 | Support multiple identifier types — user ID, API key, IP address |
| 3 | Per-service / per-endpoint configurable limits |
| 4 | Support different limits for different user tiers (free vs premium) |
| 5 | Return HTTP 429 with `Retry-After` + `X-RateLimit-Remaining` headers when blocked |

### Non-Functional Requirements
| # | Requirement | Target |
|---|-------------|--------|
| 1 | Low Latency | p99 < 5ms — rate limiter is in the hot path of every request |
| 2 | High Availability | 99.99% uptime — must not be a single point of failure |
| 3 | Scalability | Handle millions of requests/sec across distributed nodes |
| 4 | Accuracy | Approximate is acceptable for performance tradeoff |
| 5 | Fail Behavior | Fail open — if rate limiter is down, allow requests through |

---

## Key Interview Insight

> The rate limiter sits in the **critical path** of every API call.  
> A slow rate limiter = slow API for everyone.  
> A failed rate limiter = your choice: ghost town or flood.  
> **Most systems choose flood (fail open) over ghost town (fail closed).**

---

*Next: Chunk 3 — Back of Envelope Estimation*