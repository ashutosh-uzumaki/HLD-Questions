# URL Shortener — Step 4: API Design

**Problem:** Design a URL shortener like bit.ly / tinyurl
**Step:** 4 of 8 (API Design)
**Teaching style:** Every decision → all options with tradeoffs → pick with reasoning

---

## 1. API Design Philosophy

**Step 4's job:** Define the **contract** between client and server.

**Not just "list endpoints."** Real API design covers:
- Style choice (REST / gRPC / GraphQL)
- HTTP methods + status codes (semantically correct)
- Idempotency behavior
- Error contracts
- Versioning
- Validation
- Rate limiting
- Auth (if applicable)
- Behavior under failure (replication lag, downtime)

**Golden rule:** Every API design decision has multiple valid options. Always know the alternatives, pick with reasoning.

---

## 2. Decision 1: API Style — REST vs gRPC vs GraphQL

### Option A: REST ✅
- HTTP + JSON, stateless
- Browser-native (can do 302 redirects)
- HTTP caching works
- Less efficient payload than protobuf
- Best for: public APIs, browser/mobile clients

### Option B: gRPC
- HTTP/2 + Protocol Buffers (binary)
- 5-7x smaller payload, lower latency
- No native browser support (needs gRPC-web proxy)
- Strong typing via .proto files
- Best for: internal service-to-service, low-latency needs

### Option C: GraphQL
- Single endpoint, client selects fields
- No over-fetching
- HTTP caching is harder
- N+1 query problem
- Best for: frontend-heavy apps with complex nested data

### Recommendation: REST

Why:
- URL Shortener is public-facing (browsers hit it)
- Redirect flow needs HTTP 302 native to REST — gRPC can't do browser redirects
- Simple CRUD — no over-fetching problem for GraphQL to solve
- HTTP caching helps for metadata/stats endpoints

### Interview framing
> "REST over HTTP. The redirect flow requires HTTP 302 which is native to REST — gRPC can't do browser redirects. And shorten is a simple CRUD operation, no need for gRPC efficiency or GraphQL flexibility."

---

## 3. Decision 2: Path Structure — Collision Avoidance

### The problem
User visits `short.ly/abc1234` → server must route `/abc1234` to redirect.
But `/urls` is an API endpoint. What if user's custom alias is "urls"? Collision.

### Option A: Subdomain separation (industry standard)
```
short.ly/{short_url}       → redirects
api.short.ly/v1/urls/...   → API endpoints
```

### Option B: `/api/` prefix
```
/{short_url}       → redirect
/api/v1/urls/...   → API
```

### Option C: Blacklist reserved aliases
Block "urls", "api", "admin", etc. from being custom aliases.

### Recommendation: Subdomain separation (Option A)

Why: zero collision risk, what bit.ly / tinyurl / rebrandly all do.
If easier to demo: `/api/v1/...` prefix (Option B).

### Interview framing
> "I'll separate concerns with subdomains — short.ly for redirects, api.short.ly/v1 for the REST API. Eliminates path collision with custom aliases. Same pattern as bit.ly and tinyurl."

---

## 4. Decision 3: API Versioning

### Option A: URI versioning (recommended)
`/api/v1/urls` → `/api/v2/urls`

### Option B: Header versioning
`Header: API-Version: 1`

### Option C: Accept header versioning
`Header: Accept: application/vnd.shortly.v1+json`

### Recommendation: URI versioning

Visible in logs, browser, curl — debuggable. Industry standard (Razorpay, Stripe, GitHub).

---

## 5. Final API Endpoint List

### 5.1 Shorten URL

```
POST /api/v1/urls

Headers:
  Idempotency-Key: <client-generated-uuid>   (required for retry safety)
  Content-Type: application/json

Request:
{
  "long_url":     "https://example.com/very/long/url",
  "custom_alias": "my-promo",              // optional
  "expiry_at":    "2026-12-31T00:00:00Z"   // optional
}

Response 201 Created:
{
  "short_url":    "abc1234",
  "long_url":     "https://example.com/very/long/url",
  "custom_alias": "my-promo",
  "expiry_at":    "2026-12-31T00:00:00Z",
  "created_at":   "2025-04-20T12:00:00Z"
}

Errors:
  400 - Invalid URL format / scheme / length
  409 - Custom alias already exists
  422 - Valid JSON but semantic error (e.g., expiry in past)
  429 - Rate limit exceeded
  503 - Service unavailable (Redis + DB both down)
```

### 5.2 Redirect (browser-facing)

```
GET /{short_url}

Response 302 Found:
  Headers: Location: https://example.com/very/long/url
  Body: (empty)

Errors:
  404 - Short URL not found
  410 - URL expired (Gone)
```

Note: 302 (not 301) to preserve click tracking — 301 gets cached by browsers.

### 5.3 Get URL Metadata

```
GET /api/v1/urls/{short_url}

Response 200 OK:
{
  "short_url":    "abc1234",
  "long_url":     "https://...",
  "custom_alias": "my-promo",
  "expiry_at":    "...",
  "created_at":   "...",
  "click_count":  12453
}

Errors: 404
```

### 5.4 Update URL (extend expiry only)

```
PATCH /api/v1/urls/{short_url}

Request: { "expiry_at": "2027-12-31T00:00:00Z" }
Response 200 OK: (full updated resource)

Errors:
  400 - Invalid expiry (past, or >1 year out)
  404 - Not found
```

Why only PATCH for expiry? long_url and custom_alias are immutable — changing them would break every existing short link in circulation.

### 5.5 Delete URL

```
DELETE /api/v1/urls/{short_url}
Response 204 No Content (empty body)
Errors: 404
```

### 5.6 Click Stats (optional)

```
GET /api/v1/urls/{short_url}/stats

Response 200 OK:
{
  "short_url":       "abc1234",
  "click_count":     12453,
  "last_clicked_at": "..."
}
```

---

## 6. HTTP Status Code Reference

### 2xx Success
| Code | Meaning | Use |
|------|---------|-----|
| 200 OK | Success + body | GET, PATCH |
| 201 Created | Resource created | POST |
| 204 No Content | Success, no body | DELETE |

### 3xx Redirects
| Code | Meaning | Use |
|------|---------|-----|
| 301 Moved Permanently | Cached by browser | DON'T USE (breaks tracking) |
| 302 Found | Temporary redirect | Redirect flow |

### 4xx Client Errors
| Code | Use |
|------|-----|
| 400 Bad Request | Invalid URL format |
| 401 Unauthorized | Not authenticated |
| 403 Forbidden | Authenticated but not allowed |
| 404 Not Found | short_url doesn't exist |
| 409 Conflict | Custom alias taken (state conflict) |
| 410 Gone | URL expired |
| 422 Unprocessable | Valid JSON, bad semantics |
| 429 Too Many Requests | Rate limit hit |

### 5xx Server Errors
| Code | Use |
|------|-----|
| 500 | Generic failure |
| 502 | DB unreachable |
| 503 | Overload/maintenance |
| 504 | DB didn't respond |

Trap: Duplicate email on signup → 409 Conflict (not 422).

---

## 7. Idempotency — The Deep Topic

### 7.1 Why idempotency matters
Networks fail. Clients retry. Without dedup, retries create duplicates.

### 7.2 HTTP method idempotency (from spec)
| Method | Idempotent? | Explanation |
|--------|-------------|-------------|
| GET    | Yes | Reads never mutate |
| PUT    | Yes | Replace with same body = same state |
| DELETE | Yes | Delete twice = still deleted |
| PATCH  | By convention | `{op: "increment"}` is NOT idempotent |
| POST   | NO | Creates new resource each time |

POST needs explicit idempotency mechanism — the Idempotency-Key pattern.

### 7.3 The Idempotency-Key Pattern

```
Client generates UUID per logical request, sends as header.

POST /api/v1/urls
Header: Idempotency-Key: uuid-abc-123

Server logic:
  1. Look up the key
  2. If exists → return cached response (retry case)
  3. If new → process, cache response, return
```

Duration: 24 hours. Retries happen within seconds/minutes.

### 7.4 Scenarios — Retry vs New Request

| Scenario | Same Idempotency Key? | Correct Behavior |
|----------|----------------------|-------------------|
| User retries after timeout | Yes | Return cached response |
| Two users shorten same URL | No (different keys) | Create 2 short URLs |
| Same user shortens twice | No (different keys) | Create 2 short URLs |
| Same custom_alias, different keys | No | Return 409 (conflict is real) |

Key insight: Uniqueness constraints (custom alias) are SEPARATE from retry detection. Don't confuse them.

### 7.5 Real-World Examples
| Company | Where |
|---------|-------|
| Razorpay | Payment creation (`X-Razorpay-Idempotency-Key`) |
| Stripe | Charge creation (`Idempotency-Key`) |
| AWS, GitHub, PayPal | Same pattern universally |

### Soundbite
> "POST /urls requires Idempotency-Key header. Server caches response against the key for 24h. Retries with same key return cached response — no duplicate URLs. Same pattern as Razorpay payment creation."

---

## 8. Idempotency Implementation — Full Menu

### Approach 1: Dedicated idempotency_keys table
```sql
CREATE TABLE idempotency_keys (
  key VARCHAR(64) PRIMARY KEY,
  response_body JSON,
  response_status INT,
  user_id BIGINT,
  created_at TIMESTAMP,
  expires_at TIMESTAMP
);
```
Use: flexible for multi-table operations.

### Approach 2: Unique constraint on business table (CHOSEN)
```sql
CREATE TABLE urls (
  id BIGINT PRIMARY KEY,
  short_url VARCHAR(8) UNIQUE,
  long_url TEXT,
  user_id BIGINT,
  idempotency_key VARCHAR(64),
  ...
  UNIQUE(user_id, idempotency_key)
);

INSERT INTO urls (...)
VALUES (...)
ON CONFLICT (user_id, idempotency_key)
DO NOTHING
RETURNING *;
```
Use: simple single-table operations. Chosen for URL Shortener — simplest, atomic, no extra table.

### Approach 3: Hybrid (Redis + DB) — production-grade
```
1. Check Redis (fast path, ~1ms)
2. HIT → return cached
3. MISS → check DB (slower, ~10ms)
4. If DB has it → cache in Redis + return
5. If not → process + store in BOTH
```
Use: production systems where speed + durability matter.

### Why UNIQUE constraint beats "SELECT then INSERT"

SELECT → INSERT has a race condition:
```
Request 1: SELECT key → not found
Request 2: SELECT key → not found  (concurrent!)
Request 1: INSERT key → success
Request 2: INSERT key → success (duplicate!)
```

UNIQUE constraint is atomic:
```
Request 1: INSERT → success
Request 2: INSERT → constraint violation (caught)
```

Rule: Never "check then write" for idempotency. Always use DB uniqueness.

### Store the response, not just the key

Wrong: only store the key. Retry comes → know it's duplicate but have no response.
Right: store (key, response_body, response_status). Retry returns exact same response first call got.

---

## 9. Idempotency Failure — What if Stores Are Down?

### Three options on failure
| Option | Behavior | Tradeoff |
|--------|----------|----------|
| A. Fail open | Accept, risk duplicates | High availability, weak consistency |
| B. Fail closed | Reject (503) | Strong consistency, low availability |
| C. Fallback store | Redis → DB → last resort | Best of both, complex |

### CAP theorem lens
- Fail open = AP (available, eventually consistent)
- Fail closed = CP (consistent, temporarily unavailable)

### Recommendation for URL Shortener: Option A (Fail Open)

Reasoning — Blast Radius Principle:
| System | Duplicate impact | Rigor |
|--------|------------------|-------|
| URL Shortener | Minor UX annoyance | Fail open OK |
| Notification | Duplicate email | Fail open OK |
| Payment | Double charge | Must never fail open |
| Seat booking | Double booking | Must never fail open |

Rule: Idempotency strictness scales with blast radius.

### Soundbite
> "Fail-open on idempotency for URL Shortener — worst case is a duplicate short URL, minor UX issue. For Razorpay payments, fail-open would be catastrophic (double charge). Strictness scales with blast radius — tune it per operation, not per system."

---

## 10. Replication Lag + Idempotency — Advanced

### The problem
```
T=0ms:  Request 1 → Primary → writes key → commits
T=50ms: Retry → Read Replica (lag) → reads "key not found"
        → processes again → DUPLICATE
```

This is the "read-your-own-write" consistency problem.

### Five fixes

1. **Read from Primary for idempotency** — regular reads go to replicas, idempotency reads go to primary. Simple. (Razorpay, Stripe)

2. **Synchronous replication** — primary waits for replica ack. Zero lag, higher write latency.

3. **Session pinning** — after a write, that user's reads go to primary for N seconds. (Facebook profiles)

4. **Wait for replication on write** — after commit, poll until replicas catch up. High write latency.

5. **Deterministic key + atomic UPSERT** — INSERT ON CONFLICT DO NOTHING. All writes go to primary. DB enforces uniqueness atomically. No replica read involved. **This is what we're using (Approach 2) — naturally avoids the problem.**

### Combine Fix 1 + Fix 5 for URL Shortener

```
Routing:
  PRIMARY
    - POST /urls (write + idempotency)
    - DELETE /urls/{id}
    - PATCH /urls/{id}
    - Uniqueness checks (custom alias, idempotency key)

  REPLICAS
    - GET /{short_url}      (redirect — stale OK)
    - GET /urls/{id}        (metadata — stale OK)
    - GET /urls/{id}/stats  (click count — stale OK)
```

### The rule — which reads MUST go to primary
| Primary (stale = correctness bug) | Replicas (stale = minor UX) |
|-----------------------------------|----------------------------|
| Idempotency check | URL redirect lookup |
| Uniqueness check before insert | Admin dashboard |
| Auth / permission check | Stats / analytics |
| Financial balance check | List queries |
| Inventory check before order | - |

### Soundbite
> "Replication lag means idempotency reads from a replica can return 'not found' for a freshly written key — causing duplicates. Fix: idempotency reads always go to primary. In practice, INSERT ON CONFLICT DO NOTHING is atomic at primary and doesn't involve replicas. Regular GETs still hit replicas for scale."

---

## 11. URL Validation — What Is "Invalid"?

### Six levels (cheap → expensive)
| Level | Check | Cost | Where |
|-------|-------|------|-------|
| 1 | Format (well-formed) | microseconds | App server |
| 2 | Scheme whitelist (http/https only) | microseconds | App server |
| 3 | Length (10-2048 chars) | microseconds | App server |
| 4 | Blacklist (known malicious) | ~1ms | Redis cache |
| 5 | Domain reputation | ~100ms | Async |
| 6 | Content scanning | ~500-2000ms | Async only |

### Minimum viable (Levels 1-3) — MANDATORY

```java
public boolean isValidUrl(String url) {
    if (url == null || url.length() < 10 || url.length() > 2048)
        return false;

    try {
        URI uri = new URI(url);
        if (uri.getHost() == null) return false;

        String scheme = uri.getScheme();
        return "http".equalsIgnoreCase(scheme) ||
               "https".equalsIgnoreCase(scheme);
    } catch (URISyntaxException e) {
        return false;
    }
}
```

### Security-critical: javascript: attack prevention
User submits `javascript:alert('xss')` → scheme whitelist rejects at Level 2 → 400.
URL shorteners are historical abuse targets for phishing.

### Production additions (Level 4-6)
- Sync blacklist — Redis cache of known malicious domains
- Async Google Safe Browsing — disable URL if flagged
- Content scanning — out of v1 scope

### Soundbite
> "Validation has 3 mandatory layers: syntactic (URI parser), semantic (scheme whitelist — reject javascript: for XSS prevention), and constraints (max 2048 chars). Production adds async abuse checks via Google Safe Browsing. All sync failures return 400 with structured error."

---

## 12. Structured Error Responses

### Bad (unstructured)
```json
{ "status": "FAILED", "error": "bad url" }
```

### Good (structured)
```json
{
  "error": {
    "code":    "INVALID_URL",
    "message": "Long URL must be a valid http/https URL under 2048 characters",
    "field":   "long_url"
  }
}
```

### Why structured?
- code → programmatic handling
- field → highlight bad input in UI
- message → human-readable debugging
- Enables i18n

### Industry patterns
| Company | Shape |
|---------|-------|
| Razorpay | `{error: {code, description, source, step, reason}}` |
| Stripe | `{error: {type, code, message, param}}` |
| GitHub | `{message, errors: [{resource, field, code}]}` |

---

## 13. Anti-pattern: Redundant "status" in Body

### Don't
```json
POST /urls → 201
{ "status": "SUCCESS", "short_url": "abc1234" }
```

### Why wrong
HTTP status code (201) already says success. Body duplicating is redundant.

### Do instead
```json
POST /urls → 201
{ "short_url": "abc1234", "long_url": "...", "created_at": "..." }
```

Rule: HTTP status for status. Body for data.

---

## 14. Rate Limiting (Preview — Step 6)

Rate limiting is heavy enough to be its own problem (P3 in Wave 1).

Quick: 429 Too Many Requests, Token Bucket / Sliding Window, Redis, at API Gateway.

Interview mention: "I'd rate-limit at the API gateway — 100 shortens/min per user, 1000 redirects/min per IP. Using Redis sliding window. Detailed design in deep dives."

---

## 15. Authentication (Out of Scope)

Per Step 1, auth is out of scope for v1. For production:
- Public redirect endpoint → no auth
- API endpoints → API key or OAuth2 Bearer
- Admin endpoints → stronger auth, role-based

---

## 16. Interview Best Practices

### DO
1. REST for browser-facing APIs (needs 302)
2. Version from day 1 (/api/v1/...)
3. Separate subdomains when possible
4. Always Idempotency-Key for POST
5. Return full resource on create
6. Use status codes semantically (204 for DELETE, 302 for redirect)
7. Validate at multiple layers
8. Return structured errors (code + message + field)
9. Route idempotency reads to primary
10. Tie fail-open/closed to blast radius

### DON'T
1. Don't use 301 redirects (breaks click tracking)
2. Don't use 201 for DELETE
3. Don't add "status": "SUCCESS" in body
4. Don't use "SELECT then INSERT" for idempotency
5. Don't route idempotency reads to replicas
6. Don't confuse 409 (conflict) with 400 (bad input)
7. Don't skip versioning
8. Don't share path namespace for redirect + API

---

## 17. Interview Soundbites

**REST choice:**
> "REST over gRPC — 302 redirects are native, browsers can't speak gRPC without proxies. GraphQL has no over-fetching to solve in simple CRUD."

**Idempotency:**
> "POST /urls requires Idempotency-Key header. Server caches response for 24h. Retries return cached response. Same as Razorpay payment creation."

**DB-based idempotency:**
> "Using INSERT ON CONFLICT DO NOTHING on unique(user_id, idempotency_key). Atomic at primary — no race conditions, no replica lag impact."

**Fail-open:**
> "Fail-open on Redis outage for URL Shortener — duplicate URL is minor. For payments, never fail open — double charge is catastrophic. Strictness scales with blast radius."

**Replication lag:**
> "Idempotency reads always go to primary to avoid replica-lag duplicates. Redirect GETs go to replicas for scale. Different operations, different CAP choices."

**Validation:**
> "Three layers: format, scheme whitelist (block javascript: URIs), length cap. Async Google Safe Browsing for abuse. 400 with structured error."

**Structured errors:**
> "Error body has code, message, field. Code for programmatic handling, message for humans, field for UI highlighting. Standard pattern at Razorpay and Stripe."

---

## 18. Self-Assessment

### What went well
- Asked about API prefix / collision — Tier-2 instinct
- Asked about PATCH update semantics — good product thinking
- Asked about Redis-down scenario — senior question
- Asked about replication lag — Tier-1 question

### To improve
- Initially used `GET /urls/{short_url}` for redirect (should be `GET /{short_url}`)
- Initially used 201 for DELETE (should be 204)
- Included redundant `"status": "SUCCESS"` in responses
- Didn't include Idempotency-Key in first pass

### Step 4 grade: 9/10

---

## 19. Key Terms

| Term | Meaning |
|------|---------|
| Idempotent | Same operation multiple times = same result |
| Idempotency Key | Client-generated UUID identifying a logical request |
| Read-your-own-write | Consistency guarantee user sees their own writes |
| Replication lag | Time between write on primary and availability on replica |
| Fail-open / Fail-closed | On failure, accept (open) or reject (closed) |
| Blast radius | Extent of damage caused by a failure |
| UPSERT | Insert if not exists, update if exists (atomic) |

---

## 20. Next Step — DB Design (Step 5)

With APIs locked, Step 5 covers:
1. SQL vs NoSQL with full tradeoff menu
2. Specific DB choice (MySQL / PostgreSQL / Cassandra / DynamoDB)
3. Schema design — tables, columns, indexes
4. Sharding strategy — partition key + hot shard mitigation
5. Replication strategy — primary-replica, multi-master, quorum
6. Capacity planning — shards, replicas, growth projection

Step 5 is where Step 1 NFRs + Step 2 numbers pay off. Every design choice traces back to a specific number.