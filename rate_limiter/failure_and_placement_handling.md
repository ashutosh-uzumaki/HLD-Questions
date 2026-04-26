# HLD: Rate Limiter — Chunk 9: Placement + Failure Handling

---

# Part 1: Rate Limiter Placement

## Option 1: Client Side ❌

Client ke andar rate limiting logic.

**Problem:** Malicious client modify kar sakta hai — limits bypass easily.

```
Client SDK → self-throttle → but attacker ignores this completely
```

**Use case:** Only for internal trusted services / SDKs that want to avoid hitting server limits.

---

## Option 2: API Gateway ✅ (Most Common)

```
Client → Load Balancer → API Gateway (rate limit here) → Backend
```

**Pros:**
- Backend ko request pahunchti hi nahi if blocked → compute saved ✅
- Single place to configure + monitor ✅
- All downstream services protected ✅

**Cons:**
- Limited application context — user tier, subscription status directly nahi pata
- Sirf headers/API key se decisions lene padte hain

**Use case:** General API rate limiting — most common approach.

---

## Option 3: Application Level ⚠️

Rate limiting logic runs as middleware inside the application.

**Pros:**
- Full application context available — user tier, subscription, business rules ✅
- Complex business-specific rules implement kar sakte hain ✅

**Cons:**
- Request already backend tak pahunch gayi before rejection — compute waste
- Har service ko apna rate limiting implement karna padta hai — code duplication

**Use case:** Complex rate limiting that depends on application-specific data.

---

## Recommended: Multi-Layer Approach ✅

```
Layer 1: CDN/Edge     → coarse IP-based limits    → DDoS protection
Layer 2: API Gateway  → API key/user limits        → main enforcement
Layer 3: Application  → business-specific rules    → premium features
```

**Defense in depth** — har layer ek alag type ka abuse handle karta hai.

| Layer | Rate Limit Type | Purpose |
|-------|----------------|---------|
| CDN/Edge | IP-based, coarse | Block DDoS, obvious abuse |
| API Gateway | API key, user ID | Enforce subscription limits |
| Application | Business rules | Handle edge cases, premium features |

---

# Part 2: Failure Handling

## Fail Open vs Fail Closed

Rate Limiter **critical path** mein hai — har request iske through jaati hai.

### Fail Open (Allow All) — Default for Most APIs

Rate limiter unavailable → **allow all requests through.**

```
Rate Limiter fails + Fail Open =
Temporarily overloaded ho sakta hai
but legitimate users serve hote rehte hain ✅
```

**Why:** Rate limiter failure pe poora API down karna worse hai — 100% users block ho jaate hain including legitimate ones. Availability > strict rate enforcement.

**Use case:** Most APIs — e-commerce, social media, general purpose APIs.

---

### Fail Closed (Block All)

Rate limiter unavailable → **block all requests.**

```
Rate Limiter fails + Fail Closed =
Poora API down 😱 (legitimate users bhi blocked)
but security guaranteed ✅
```

**Why:** Kuch systems mein limit exceed hona catastrophic hai.

**Use case:** Financial transactions, OTP generation, billing systems — security > availability.

```
Example: OTP system
Fail open → attacker unlimited OTPs generate kare → security breach 😱
Fail closed better here
```

---

### Summary

| Mode | Behavior | Use Case |
|------|---------|---------|
| **Fail Open** | Rate limiter down → allow all | Most APIs — availability matters more |
| **Fail Closed** | Rate limiter down → block all | Financial, security-critical systems |

---

## Recommended: Fail Open with Fallbacks

```
Primary:    Distributed Redis cluster (accurate rate limiting)
    ↓ Redis fails
Fallback 1: Local in-memory rate limiting (less accurate but functional)
    ↓ Local cache also fails
Fallback 2: Fail open + aggressive monitoring + alerts
```

> "Pehle local cache se rate limit karo — Redis se accurate nahi hoga but better than nothing. Agar local cache bhi fail ho toh fail open karo with alerting."

---

## Circuit Breaker Pattern

Prevents cascading failures — automatically detects + recovers from failures.

Think of it like an **electrical switch:**
```
Switch CLOSED = circuit complete = current flows = bulb ON
Switch OPEN   = circuit broken   = no current   = bulb OFF
```

### 3 States

#### Closed (Normal) ✅
Circuit closed = requests flow through normally.
Rate limiter healthy — everything normal.

#### Open (Failed) ❌
Circuit open = requests blocked/bypassed.
Rate limiter unhealthy — use fallback (local cache or fail open).

#### Half-Open (Probing) 🔍
After a timeout — allow **few test requests** to check if system recovered.
```
Test requests →
  Success → back to CLOSED ✅
  Failure → back to OPEN ❌
```

### State Flow

```
CLOSED (healthy)
    ↓ failures exceed threshold
OPEN (unhealthy — bypass rate limiter)
    ↓ after timeout (e.g. 30 seconds)
HALF-OPEN (probe with few requests)
    ↓ success              ↓ failure
CLOSED ✅              OPEN again ❌
```

### Applied to Rate Limiter

```
CLOSED:    Redis available → normal Redis-based rate limiting
OPEN:      Redis down → local in-memory rate limiting or fail open
HALF-OPEN: Redis recovering → few requests tested on Redis
```

**Benefits:**
- Cascading failures prevent karta hai ✅
- Automatic recovery ✅
- System ko breathe karne deta hai ✅
- No manual intervention needed ✅

---

## Chunk 9 Summary

| Topic | Recommendation |
|-------|---------------|
| Placement | Multi-layer — CDN + Gateway + Application |
| Failure mode | Fail Open (most APIs) / Fail Closed (financial/security) |
| Fallback strategy | Redis → Local Cache → Fail Open |
| Failure detection | Circuit Breaker — Closed → Open → Half-Open |

---

*Next: Chunk 10 — Monitoring + Response Headers*