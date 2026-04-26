# HLD: Rate Limiter — Chunk 4: Core API Design

---

## Overview

Rate limiter needs 3 APIs:
1. `POST /ratelimit/check` — core runtime API (called on every request)
2. `GET /ratelimit/status/{client_id}` — read-only status check
3. `PUT /ratelimit/rules` — admin API to configure rules

---

## 1. POST /ratelimit/check

Called by API Gateway on **every incoming request** to decide allow/deny.

### Request Body
```json
{
  "client_id": "user_123",
  "client_type": "user_id",
  "resource": "/api/orders",
  "tokens": 1
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `client_id` | ✅ | The identifier value — user_123, 192.168.1.1, api_key_xyz |
| `client_type` | ✅ | Type of identifier — `user_id`, `ip`, `api_key` |
| `resource` | ✅ | API endpoint being accessed |
| `tokens` | ❌ | Units to consume, default 1. Expensive ops can consume more. |

### Response — 200 OK (Allowed)
```json
{
  "allowed": true,
  "tokens_remaining": 42,
  "reset_at": 1703980800
}
```

### Response — 429 Too Many Requests (Blocked)
```json
{
  "allowed": false,
  "tokens_remaining": 0,
  "reset_at": 1703980800,
  "retry_after": 30
}
```

| Field | Description |
|-------|-------------|
| `allowed` | Gateway ko bas yahi chahiye — forward kare ya 429 return kare |
| `tokens_remaining` | Kitna quota bacha hai current window mein |
| `reset_at` | Exact Unix timestamp jab window reset hogi — client dashboard pe show karo |
| `retry_after` | **Seconds** mein wait time before retry — sirf 429 pe present hota hai |

### retry_after vs reset_at — Key Difference
- `retry_after: 30` → "30 seconds baad retry karo" — simple, SDK mein `Thread.sleep(30000)` type logic
- `reset_at: 1703980800` → exact timestamp — "Your limit resets at 12:01:00 PM" — better UX for dashboards

---

## 2. GET /ratelimit/status/{client_id}

Read-only — **quota consume nahi hota**. Client apna current status check kare.

### Use Cases
- Developer dashboard — "Aapka 80% quota use ho gaya"
- Proactive throttling — SDK check kare before sending request
- Customer support — agent check kare client ka current quota
- Internal monitoring + alerting

### Response — 200 OK
```json
{
  "limits": [
    {
      "resource": "/api/orders",
      "tokens_remaining": 100,
      "reset_at": 1703980800
    },
    {
      "resource": "/api/search",
      "tokens_remaining": 450,
      "reset_at": 1703980800
    }
  ]
}
```

Array of limits kyunki ek client ke multiple resources pe alag alag limits hoti hain.

---

## 3. PUT /ratelimit/rules (Admin API)

Admin configure karta hai — kaunse endpoint pe kaunse tier ke liye kitna limit.

### Request Body
```json
{
  "rules": [
    {
      "resource": "/api/dashboard",
      "tier": "free",
      "limit": 100,
      "window_seconds": 30,
      "algorithm": "token_bucket"
    }
  ]
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `resource` | ✅ | API endpoint pattern — supports wildcards like `/api/v1/*` |
| `tier` | ❌ | User tier — `free`, `premium`, `enterprise`. NULL = applies to all |
| `limit` | ✅ | Maximum requests allowed per window |
| `window_seconds` | ✅ | Time window duration in seconds |
| `algorithm` | ❌ | `token_bucket`, `sliding_window_counter`, etc. Default applied if omitted |

### How Rules Work at Runtime
1. Request aati hai — `client_id: user_123, resource: /api/orders`
2. Rate Limiter Service — "user_123 ka tier kya hai?" → `free`
3. Rules lookup — "free + /api/orders = 100 req/min, sliding_window"
4. Redis mein counter check + increment
5. Allow / Deny decision

---

## Common Mistakes to Avoid

| Mistake | Correction |
|---------|-----------|
| `resource: "ip"` in rules | `resource` = API endpoint, not identifier type |
| `client_type` in rules | `client_type` is runtime info — not part of rule config |
| `tokens_limit` | Use generic `limit` — not all algorithms use token concept |
| `retry_after` as timestamp | `retry_after` = seconds, not timestamp. `reset_at` = timestamp |
| Single object in status response | Use array — client has limits on multiple resources |

---

*Next: Chunk 5 — High Level Design (Components + Request Flow)*