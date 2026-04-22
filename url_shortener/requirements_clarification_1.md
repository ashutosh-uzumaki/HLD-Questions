# URL Shortener — Step 1: Requirements Clarification

**Problem:** Design a URL shortener like bit.ly / tinyurl
**Target:** Razorpay / PhonePe / Flipkart SDE-2 level
**Step:** 1 of 8 (Requirements)

---

## 1. The Framework — FR vs NFR vs Constraints

Know the difference cold. Mixing these up signals junior-level thinking.

| Bucket | Definition | Example |
|--------|------------|---------|
| **FR** | What the system does (verbs, features) | "User can shorten a URL" |
| **NFR** | How well it does it (scale, latency, availability) | "p99 latency < 100ms" |
| **Constraints** | Hard limits imposed on the system | "Short URL ≤ 7 chars" |

**Rule of thumb:** If it answers *what* → FR. If it answers *how well* → NFR. If it answers *what's the limit* → Constraint.

---

## 2. Functional Requirements (Locked)

1. User can shorten a long URL
2. User is redirected (302) from short → long URL
3. User can optionally provide a custom alias
4. URLs can optionally have expiry (1 day to 1 year)
5. Track redirection count per short URL
6. Same long URL shortened twice = different short URLs (no dedup)

### Key Decisions & Reasoning

**302 (Temporary Redirect) vs 301 (Permanent Redirect):**

Use **302**. Here's why:

- **301** is cached by browsers aggressively — subsequent clicks skip our server
- If we use 301, we **break click tracking** (counter never increments after first visit)
- **302** forces the browser to hit our server every time → enables analytics
- 99% of real URL shorteners (bit.ly, tinyurl) use 302 for this reason

**Interview soundbite:**
> *"I'm choosing 302 over 301 because we need click tracking. 301 gets cached by browsers, which means the shortener service never sees repeat clicks, breaking our counter."*

**Dedup behavior — same long URL shortened twice:**

Chose **Option B: New short URL every time** (no dedup).

| Option A: Dedup | Option B: Unique per request (CHOSEN) |
|-----------------|---------------------------------------|
| Same long URL → same short URL | Each shorten request → new short URL |
| Saves storage | More storage |
| Breaks per-user analytics | Each user owns their link |
| Simpler | Matches real-world UX (bit.ly, tinyurl) |

---

## 3. Non-Functional Requirements (Locked)

| NFR | Value | Reasoning |
|-----|-------|-----------|
| **Availability** | 99.99% on redirect path | Broken links = reputation damage. 99.9% = 8.76 hrs/year downtime across millions of links = unacceptable |
| **Redirect latency** | p99 < 100ms | User-facing. Redirect delay = bad product feel |
| **Shorten latency** | p99 < 500ms | Background action, one-time setup — acceptable to be slower |
| **Durability** | Zero data loss on existing URLs | Losing a URL breaks live links on emails/ads/business cards |
| **Read:Write ratio** | ~100:1 (read-heavy) | Drives cache-first architecture |
| **Scalability** | Horizontal, auto-scale | Handle 10x spike in <2 min (viral links) |

### Why Every NFR Needs a Number

Each number changes the architecture. Examples:

- **99.9% vs 99.99%** → single region vs multi-region (massive cost difference)
- **100ms vs 500ms latency** → need cache vs can hit DB directly
- **1000:1 vs 10:1 read:write** → cache-first vs write-optimized design

**Weak answer in interview:**
> "System should be highly available and scalable"

**Strong answer:**
> "99.99% availability on redirects, p99 < 100ms, 100:1 read:write ratio"

### Percentile Thinking — Why p99 Matters

Average latency hides outliers. At scale, p99 is what users actually experience.

**Example:** 99 requests at 10ms + 1 request at 10s
- Average ≈ 100ms (looks fine)
- p99 = 10s (the truth)
- At 10M requests/day, 1% = 100K bad experiences/day

**Standard percentiles:**
- **p50 (median)** — typical experience
- **p95** — most users
- **p99** — tail latency (the bad cases)
- **p99.9** — worst cases (matters at massive scale)

**Interview rule:** Never say "average latency." Always say "p99 latency."

### Availability vs Durability — Don't Conflate

| Term | Meaning | URL Shortener Requirement |
|------|---------|----------------------------|
| **Availability** | System is up and responding | 99.99% uptime |
| **Durability** | Data is not lost | 100% — can't lose existing short URLs |

A system can be 99.99% available but have 1% data loss (bad durability). Separate them explicitly.

---

## 4. Constraints (Locked)

| Constraint | Value | Reasoning |
|------------|-------|-----------|
| Long URL max length | 2048 chars | Browser/HTTP standard cap |
| Custom alias length | 4-20 chars | Min 4 avoids collision with auto-gen, max 20 keeps it "short" |
| Auto-generated short URL | 7 chars | 62^7 ≈ 3.5 trillion URLs — justified in Step 2 |
| Character set | Base62 `[a-zA-Z0-9]` | URL-safe, no encoding issues |
| Retention after expiry | 30d soft delete, then hard delete | Recovery window for users |

### Why Base62, Not Base64?

Classic interview follow-up. Lock this in:

> *"Base64 uses `+`, `/`, and `=` — all of which have special meaning in URLs. `+` becomes space when URL-decoded, `/` is a path separator, `=` is used in query params. URL-encoding them defeats the purpose of a 'short' URL. Base62 uses only alphanumerics — URL-safe, no encoding needed."*

---

## 5. Out of Scope (Explicit)

Always state what you're NOT building. Signals scope discipline.

- Authentication / user accounts
- Full analytics (geo, device, referrer, time-of-day)
- Malicious URL detection / phishing checks
- URL preview pages
- Link management dashboard

**Interview framing:**
> *"For v1, I'm keeping auth and full analytics out of scope — we're just a URL redirection service. We can layer these on later."*

---

## 6. The Clarification Framework — How to Drive Step 1

### What to Do

1. **Bucket questions into FR / NFR / Constraints** — show structured thinking
2. **Propose NFR numbers yourself** — don't wait for interviewer to feed them
3. **Ask about dedup behavior** ("same URL shortened twice — what happens?")
4. **Ask about redirect type** (301 vs 302) — signals analytics awareness
5. **Explicitly state out-of-scope** — scope discipline
6. **Defend every decision with one-line reasoning**

### What NOT to Do

- Don't list generic buzzwords ("highly available, scalable, fault tolerant") without numbers
- Don't confuse FRs with Constraints (custom alias is an FR, not a constraint)
- Don't forget percentiles (p99, not "around 100ms")
- Don't skip the "same URL twice" question — it's a major design decision
- Don't ask the interviewer for every number — propose and let them correct

### The Tier-2 Magic Phrase

> *"I'll assume [X]. Is that aligned with what you have in mind?"*

Does 3 things:
1. Shows you can reason independently
2. Commits to a number → forces design decisions
3. Gives interviewer chance to adjust

### Assume vs Ask Decision Tree

**ASSUME + commit to a number when:**
- Standard product patterns (read-heavy for URL shortener)
- You can reason from product context
- There's a reasonable default

**ASK the interviewer when:**
- Business-specific constraints you can't know (global vs India-only)
- Tech stack preferences (cloud provider)
- Scope decisions (include analytics?)

---

## 7. Interview Soundbites to Memorize

**On 302 vs 301:**
> *"302 preserves click tracking — 301 gets cached by browsers, breaking our counter."*

**On p99 vs average:**
> *"Average hides outliers. At 10M requests/day, 1% bad experiences = 100K angry users daily. p99 is what users actually experience."*

**On availability vs durability:**
> *"Availability is about uptime, durability is about data loss. A system can be up but lose data — these are separate concerns."*

**On read:write ratio:**
> *"100:1 makes this read-heavy — drives cache-first architecture, read replicas, and aggressive TTLs on redirect lookups."*

---

## 8. Final Requirements Snapshot (for Revision)

```
FUNCTIONAL
----------
1. Shorten long URL
2. Redirect (302) short → long
3. Optional custom alias (4-20 chars)
4. Optional expiry (1d - 1y)
5. Track redirection count
6. No dedup — same URL twice = different short URLs

NON-FUNCTIONAL
--------------
- 99.99% availability (redirect)
- Redirect p99 < 100ms
- Shorten p99 < 500ms
- Zero data loss
- 100:1 read:write
- Horizontal scale, 10x spike handling

CONSTRAINTS
-----------
- Long URL: ≤ 2048 chars
- Custom alias: 4-20 chars
- Auto-gen: 7 chars, base62
- 30d soft delete post-expiry

OUT OF SCOPE
------------
- Auth
- Full analytics
- Malicious URL detection
- URL preview
- Dashboard
```

---

## 9. Self-Assessment from This Step

**What I did well:**
- Asked probing FR questions (didn't dump assumptions)
- Asked about 302 vs 301 (Tier-2 signal)
- Separated durability from availability in NFRs (after correction)
- Asked right constraint questions (length, character set)
- Skipped auth with justification (scope discipline)

**What to improve:**
- First NFR attempt was vague buzzwords — need to go straight to numbers
- Missed "same URL shortened twice" question (conceptual gap)
- Missed retention as a constraint (minor)
- Mixed up FRs and Constraints initially (custom alias, expiry were FRs)

**Step 1 grade: 8/10**

---

## 10. Next Step Preview — Estimation

Need to compute:
1. DAU + traffic (shorten requests/day, redirect requests/day)
2. QPS (from DAU) for both operations
3. Storage (per URL × retention period)
4. Bandwidth (ingress for shorten, egress for redirects)
5. Cache memory (hot URLs, 80/20 rule)

Starting assumption: DAU + derive from there. Show math.