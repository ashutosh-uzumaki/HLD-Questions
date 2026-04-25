# HLD: Rate Limiter — Chunk 1: What & Why

---

## What is a Rate Limiter?

A rate limiter controls how many requests a client can make to a service within a given time window.

**Core idea:** Track request count per client → reject once threshold is crossed → return HTTP 429 Too Many Requests.

---

## Why Do We Need It?

### 1. Cascading Failures
One overloaded service can bring down downstream dependencies — domino effect across the system.

### 2. Noisy Neighbor Problem
One misbehaving client degrades experience for ALL legitimate clients sharing the same infrastructure.

### 3. Cost Spike
Auto-scaling triggers unnecessarily in response to abuse traffic — burning money that should've been blocked at the gate.

### 4. Security
Without rate limiting, brute force attacks (OTP flooding, password guessing), credential stuffing, and web scraping are unchecked.

---

## What It Does

On every incoming request, the rate limiter answers one question:

> *"Has this client exceeded their allowed quota in the current time window?"*

- **Yes** → Reject with `HTTP 429 Too Many Requests`
- **No** → Allow through to backend

---

## Key Insight

Rate limiting must happen **before** the request reaches backend services. It sits in the **critical path** of every API call — so it must be extremely fast (p99 < 5ms).

---

*Next: Chunk 2 — Clarifying Questions & Requirements*