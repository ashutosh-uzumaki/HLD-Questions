# URL Shortener — ID Generation Deep Dive

## Why This Matters
Every URL shortener write needs a unique, short, collision-free 7-char string.
At peak: **6500 writes/sec** in a distributed system with multiple app servers.

---

## Base62 Math (Lock This In)

```
Base62 = [a-z] + [A-Z] + [0-9] = 62 symbols
62^7   = 3,521,614,606,208 ≈ 3.5 trillion unique IDs
3.5T / 200M per year = ~17,500 years of capacity
```

**Why not Base64?**
Base64 adds `+` and `/` — URL-unsafe characters. Base62 stays clean.

---

## The 4 Approaches (Worst → Best)

---

### ❌ Approach 1 — Hash-Based (Worst)

**How it works:**
```
MD5("https://amazon.com/long/url") → a9f3c2b1e8d4... (128-bit)
→ take first 7 chars → Base62 encode → "aB3kR9z"
```

**Problems:**
1. **Collisions guaranteed** — you're truncating 78% of entropy. Two different URLs can produce the same first 7 chars.
2. **Collision resolution is ugly** — append salt, rehash, re-check DB. At 6500 writes/sec this becomes a latency nightmare.
3. **Determinism kills flexibility** — same URL always gets same short code. Can't support A/B tracking, per-user links, or differential expiry.

**Verdict:** Collision-prone, operationally messy, inflexible. Never use for production URL shortener.

---

### ❌ Approach 2 — Random Generation

**How it works:**
```
Generate random 7-char Base62 string → "xK9mP2q"
→ check DB: exists? regenerate. No? save it.
```

**Problems:**
1. **Birthday problem at scale:**
   ```
   Total space = 3.5 trillion
   At 10B records: collision probability ≈ n²/2m ≈ 14%
   ```
2. **Mandatory read-before-write** — 6500 extra reads/sec just for uniqueness checks. Doubles DB load.
3. **Retry storms** — collisions cluster under write spikes. Hard to bound worst-case latency.

**One thing it does well:** Unpredictable IDs — no enumeration attack possible.

**Verdict:** Works at small scale. Degrades badly at high scale.

---

### ⚠️ Approach 3 — Counter-Based

**How it works:**
```
Global counter: 1, 2, 3...
→ encode integer to Base62
→ "1" → "1", "62" → "Z", "12345" → "dnh"
```

**What's good:**
- Guaranteed unique — no collision checks
- Simple to implement
- Predictable capacity planning

**Problems:**

| Problem | Detail |
|---|---|
| SPOF | Single counter service (Redis/ZooKeeper). Goes down → writes stop |
| Sequential IDs | Attacker enumerates all URLs by incrementing |
| Network hop | Every write hits central counter at 6500/sec |

**Partial fix — Range Allocation:**
```
App Server 1 → gets range: 1–10,000 (burns locally)
App Server 2 → gets range: 10,001–20,000
```
Reduces coordination calls. But: server crashes → gaps in IDs. Range size needs tuning.

**Redis INCR specifically:**
- Redis INCR is atomic — no duplicate IDs
- Fast (~100k ops/sec single node)
- But: still sequential (security risk) + Redis primary is ceiling

**Verdict:** Valid for medium scale (Bitly used this early on). Sequential IDs are a real security concern for public URL shorteners.

---

### ✅ Approach 4 — Snowflake-Style (Best)

**How it works:**
Generate a structured **64-bit integer** locally on each app server — no coordination needed.

```
| 41 bits timestamp | 10 bits machine ID | 12 bits sequence |
  ms since epoch      1024 unique nodes    4096 IDs/ms/node
```

**Throughput:**
```
1024 machines × 4096 IDs/ms = ~4 billion IDs/second
You need: 6500/sec. Massively overprovisioned. ✅
```

**The Base62 connection:**
```
Snowflake ID (64-bit int) → Base62 encode → "aB3kR9z"
```

**How it solves every problem:**

| Problem | Hash | Random | Counter | Snowflake |
|---|---|---|---|---|
| Collision-free | ❌ | ❌ at scale | ✅ | ✅ |
| No DB check needed | ❌ | ❌ | ✅ | ✅ |
| Decentralized | ✅ | ✅ | ❌ | ✅ |
| Not enumerable | ✅ | ✅ | ❌ | ✅ |
| No SPOF | ✅ | ✅ | ❌ | ✅ |
| Time-sortable | ❌ | ❌ | ✅ | ✅ |

**The one real problem — Clock Skew:**
```
Machine clock drifts backward:
time = 1000ms → ID: 1000|001|0001
time = 998ms  → ID: 998|001|0001  ← DUPLICATE
```

**Fix:** If `current_time < last_timestamp`, wait until clock catches up. NTP keeps drift small (<10ms typically), so wait is negligible.

**For URL Shortener — 7-char constraint:**
Full 64-bit Snowflake → Base62 → up to 11 chars. Three options:

| Option | Approach | Risk |
|---|---|---|
| A | First 7 chars of encoded Snowflake | Minor collision risk from truncation |
| B | Full Snowflake as seed, store full ID, return first 7 | Extremely rare collisions |
| C | Shrink Snowflake to 42-bit | Fits in 7 Base62 chars, less capacity |

**Recommendation:** Option B for production. Collisions are so rare they're a non-issue.

---

## Machine ID Assignment — ZooKeeper

**Why you need it:**
```
Two servers boot simultaneously
Both hit Redis counter → both read ID=47
Both generate Snowflake IDs with machine_id=47
→ DUPLICATE IDs
```

**How ZooKeeper solves it:**
ZooKeeper uses **ZAB protocol** (ZooKeeper Atomic Broadcast) — every write is linearizable. Only one machine can read-and-commit a given ID.

```
Machine boots → connects to ZooKeeper
ZK cluster (3 nodes) → quorum write: ID=47 assigned
Majority (2/3) acknowledge → committed
Next machine → gets ID=48
```

**ZooKeeper quorum = no SPOF:**
```
3 ZK nodes
Write succeeds if 2/3 acknowledge
One node dies → cluster keeps working
```

**ZooKeeper vs Redis for machine ID assignment:**

| | ZooKeeper | Redis |
|---|---|---|
| Purpose-built for coordination | ✅ | ❌ |
| Quorum-based (no SPOF) | ✅ | Needs Redlock |
| Operationally complex | ❌ Heavy | ✅ Simpler |
| Used in practice | Twitter, Kafka | Smaller systems |

**Practical shortcut:** For <1024 machines with stable deployments, just use **manual config assignment**:
```yaml
machine_id: 47  # set once at deployment
```
ZooKeeper only needed when machines spin up/down dynamically (auto-scaling).

**What else ZooKeeper is used for:**
- Leader election (Kafka broker controller)
- Distributed locks (prevent two servers processing same thing)
- Service discovery (track which nodes are alive)
- Configuration management (push config changes to all nodes instantly)

---

## Custom Alias Collision Handling

When user provides `short.ly/mycoolbrand`:

```
Check DB: SELECT WHERE short_url = "mycoolbrand"
→ Exists? Return 409 Conflict + suggest alternatives
→ Doesn't exist? INSERT and return 200
```

**Alternatives suggestion strategy:**
```
"mycoolbrand" taken
→ suggest: mycoolbrand1, mycoolbrand2, mycoolbrand-2024
```

**Race condition on custom alias:**
Two users submit same custom alias simultaneously:
```
Both read → not exists
Both write → one succeeds, one gets unique constraint violation
Loser → retry with 409 response
```
Cassandra handles this with **lightweight transactions (LWT):**
```
INSERT IF NOT EXISTS → atomic check-and-set
Only one writer wins
```
LWT is slower (Paxos under the hood) but acceptable since custom aliases are rare writes.

---

## Interview Soundbite

> "We use a Snowflake-inspired ID generator — timestamp + machine ID + per-ms sequence, encoded to Base62. This gives us decentralized generation with no DB uniqueness checks, time-sortable IDs, and effectively zero collision probability at our scale. The only failure mode is clock skew, which we handle by waiting for the clock to catch up before issuing the next ID. Machine IDs are assigned either via ZooKeeper for dynamic environments or manual config for stable deployments."

---

## Pattern Transfer → Problem 3 (Unique ID Generator)

Problem 3 **is** Snowflake. You've already covered ~60% of it:

| Topic | Covered Here |
|---|---|
| 41/10/12 bit breakdown | ✅ |
| Clock skew handling | ✅ |
| Machine ID assignment strategies | ✅ |
| ZooKeeper internals | ✅ |
| Base62 encoding math | ✅ |

**What Problem 3 adds:**
- Custom epoch choice (why not Unix epoch?)
- Sequence overflow handling (what if you exhaust 4096/ms?)
- Multi-datacenter ID generation
- Comparison with UUID, ULID, NanoID