# TinyURL — System Design Study Guide

---

## 0. What Is This System?

A URL shortener takes a long web address and gives you a short one. When someone visits the short link, they get silently redirected to the original. The system has two jobs: **create** short links, and **redirect** visitors — and it must do the second one extremely fast, billions of times a day.

---

## 1. What Makes It Hard?

Most systems are hard because of write complexity. TinyURL is hard because of **read scale** and **ID uniqueness** — a rare combination.

### Hard Problem #1 — Giving every URL a unique short code without a bottleneck

In plain English: millions of people are creating short links simultaneously. Every link needs a unique 7-character code. How do you hand out unique codes that fast without every server waiting in line for the same counter?

**Technical consequence:** Any single global counter — whether in a database or Redis — becomes a chokepoint. At 6,000 link creations per second, a centralized counter limits your entire system's throughput to whatever that one node can handle.

**What beginners get wrong:** `SELECT MAX(id) + 1` or `Redis INCR` — both work fine on one server, but a single Redis node maxes out around 100,000 operations/second and is a single point of failure. If it goes down, nobody can create links.

---

### Hard Problem #2 — Redirecting billions of times a day in under 50ms

In plain English: when someone tweets a short link and it goes viral, millions of people click it within minutes. Every click needs to find the original URL and redirect the browser — in under 50ms. You cannot look it up in a database on every single click.

**Technical consequence:** At peak, this system handles over 1 million redirects per second. A typical database can handle maybe 10,000–50,000 reads per second. That's a 20–100x gap. You need multiple layers of caching between the user and the database.

**What beginners get wrong:** `SELECT long_url FROM urls WHERE short_code = 'xK3p9Qa'` on every redirect. This works at 100 users. It collapses at 100,000.

---

## 2. Requirements

### Functional Requirements

| Feature | In Scope | Notes |
|---|---|---|
| Shorten a URL | ✅ | Returns a 7-character short code |
| Redirect short → original URL | ✅ | The hot path — must be very fast |
| Custom alias (e.g. `/my-brand`) | ✅ | User-chosen code instead of random |
| Link expiry (TTL) | ✅ | e.g. "this link expires in 30 days" |
| Delete a link | ✅ | Soft delete — code is retired, not reused |
| Click analytics | ✅ | Count clicks, country, referrer — but async |
| Custom domains (`brand.com/abc`) | ❌ | V2 |
| URL preview / screenshots | ❌ | V2 |
| Real-time analytics dashboard | ❌ | V2 |

### Non-Functional Requirements

| Requirement | Target | What it means |
|---|---|---|
| Redirect latency (p99) | < 50ms | 99% of all redirects complete in under 50ms |
| Availability | 99.99% | Less than 52 minutes of downtime per year |
| Durability | 100% | A created link is never lost |
| Consistency | Eventual | A new link may not be visible globally for ~1 second — acceptable |
| Write throughput | 6,000/sec peak | Peak link creation rate |
| Read throughput | 1,160,000/sec peak | Peak redirect rate during viral events |

> **p99 latency** means: if you took all your requests and sorted them by how long they took, the 99th percentile is the slowest 1%. Keeping p99 < 50ms means even your slowest requests are fast.

---

## 3. Scale Estimation

Work through the numbers step by step:

**Link creation (writes)**
```
100 million new links per day
÷ 86,400 seconds per day
= 1,157 link creations per second (average)
× 5 (peak traffic burst)
= ~6,000 per second at peak

→ This means we need to hand out unique codes at 6,000/sec
  without any single server becoming a bottleneck.
```

**Redirects (reads)**
```
100:1 read-to-write ratio (every link gets clicked ~100 times)
= 10 billion redirects per day
÷ 86,400 seconds
= 115,741 per second (average)
× 10 (viral burst — one link goes viral)
= ~1,160,000 per second at peak

→ This means we cannot touch a database on most redirects.
  We need caching at multiple levels.
```

**Storage**
```
Each link record ≈ 500 bytes (URL + metadata)
100M links/day × 365 days × 5 years = 182 billion records
× 500 bytes = ~91 TB over 5 years

→ This means a single database server isn't enough.
  We need to split (shard) the data across many servers.
```

**Hot cache size**
```
Top 1% of links receive ~80% of all traffic (power law)
1% of 100M total links = 1 million "hot" links
1M × 500 bytes = 500 MB

→ 500 MB fits easily in memory (Redis nodes are typically 64 GB).
  If we cache just the top 1% of links, we serve 80% of traffic
  without ever touching the database.
```

### The 5 Numbers That Drive Every Design Decision

| Number | Value | Design decision forced |
|---|---|---|
| Peak redirects | 1,160,000/sec | CDN + multi-layer cache is mandatory |
| Hot set size | 500 MB | Cache the hot set in Redis; >95% hit rate is achievable |
| DB reads after cache | ~5,800/sec | Database read replicas can easily handle this |
| 5-year storage | ~100 TB | Must shard the database across multiple servers |
| Code space (62^7) | 3.5 trillion | Won't run out of codes for decades |

---

## 4. Architecture

### Start Simple: The MVP

The simplest design that works:

```
User → Web Server → Database
```

1. User POSTs a long URL
2. Server generates a short code, saves it in the DB
3. User GETs `/xK3p9Qa` → server looks up DB → redirects

**This breaks at ~5,000 redirects/second** because the database can't keep up.

---

### Add Caching: First Fix

```
User → Web Server → Redis Cache → Database
                      ↑
                 Stores hot links
                 in fast memory
```

Now 95% of redirects are served from Redis (memory, ~1ms) instead of the database (~10–20ms). Database load drops 20x.

**This breaks at ~100,000 redirects/second** because a single Redis node and single web server are both limits.

---

### Production Architecture

```
                      ┌─────────────────────┐
                      │   GeoDNS            │  Routes users to
                      │   (Cloudflare)      │  nearest datacenter
                      └──────────┬──────────┘
                                 │
              ┌──────────────────┼─────────────────────┐
              │                  │                     │
        US-EAST             EU-WEST                  APAC
              │
              ▼
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  [1] CDN Edge Cache  ──────────────────────────────────── │
│       Cloudflare PoP (200+ worldwide)                      │
│       Serves ~75% of redirects with no server involved     │
│                                                            │
│  [2] Load Balancer                                         │
│       Distributes remaining traffic across app servers     │
│                                                            │
│  ┌──────────────────┐    ┌──────────────────┐            │
│  │ [3] Redirect Svc │    │ [4] Write Svc    │            │
│  │  (20 servers)    │    │  (5 servers)     │            │
│  │  Reads only      │    │  Creates links   │            │
│  └────────┬─────────┘    └────────┬─────────┘            │
│           │                       │                       │
│  [5] Redis Cache                  │                       │
│      Fast in-memory store         │                       │
│      Holds the 500MB hot set      │                       │
│           │ (cache miss)          │ (writes)              │
│           └───────────┬───────────┘                       │
│                       │                                   │
│  [6] Database Cluster (CockroachDB)                       │
│      Sharded across 9 servers                             │
│      Stores all 100TB of link data                        │
│                       │                                   │
│  [7] Kafka Message Queue                                  │
│      Receives click events from Redirect Svc              │
│      Feeds analytics pipeline (never blocks redirects)    │
│                       │                                   │
│  [8] ClickHouse (Analytics DB)                            │
│      Stores click counts, countries, referrers            │
│                                                           │
└────────────────────────────────────────────────────────────┘
```

**Component legend:**

| # | Component | What it does | Why it exists |
|---|---|---|---|
| 1 | CDN Edge | Caches redirects at 200+ global locations | Serves 75% of traffic with <5ms latency, no origin hit |
| 2 | Load Balancer | Spreads traffic across servers | No single server is overwhelmed |
| 3 | Redirect Service | Handles GET `/{code}` requests | Read-only, can scale independently from writes |
| 4 | Write Service | Handles POST `/api/v1/urls` | Write path isolated — its failure doesn't affect redirects |
| 5 | Redis Cache | In-memory key-value store | Serves hot URLs in ~1ms vs ~15ms from database |
| 6 | Database | Permanent storage for all links | Source of truth; only hit on cache misses |
| 7 | Kafka | Message queue for click events | Decouples analytics from the redirect path |
| 8 | ClickHouse | Analytics database | Optimized for counting and aggregating billions of events |

---

### The Redirect Request: Step by Step

When someone clicks `https://tiny.url/xK3p9Qa`:

1. Browser asks DNS: "where is tiny.url?" → **GeoDNS** returns the nearest Cloudflare PoP
2. **CDN checks its cache** — if this code was recently clicked, it returns the redirect immediately (~5ms). Done for ~75% of requests.
3. CDN miss → request reaches the **Load Balancer** → routed to a **Redirect Service** server
4. Redirect Service checks its **local in-memory cache** (top 10,000 hottest links per server). Hit → return redirect (~2ms)
5. Not in local cache → Redirect Service queries **Redis**. Hit → return redirect, populate local cache (~3ms)
6. Not in Redis → Redirect Service queries the **Database** read replica. Always a hit (or 404). Return redirect, populate Redis (~15ms)
7. **In parallel (never blocking the redirect):** a click event is published to Kafka → Analytics Service counts it in ClickHouse

**Key insight: the database is only involved in ~1% of redirects.** The other 99% are served from CDN, local memory, or Redis.

---

## 5. API Design

### Two main endpoints

**Create a short link**
```
POST /api/v1/urls
Authorization: Bearer <token>
Idempotency-Key: <unique-id>    ← explained below

{
  "long_url":    "https://example.com/very/long/path",
  "custom_code": "my-brand",   // optional
  "ttl_seconds": 2592000       // optional — 30 days
}

Response 201:
{
  "short_url":  "https://tiny.url/xK3p9Qa",
  "short_code": "xK3p9Qa",
  "expires_at": "2026-06-01T10:00:00Z"
}
```

**Redirect**
```
GET /xK3p9Qa

Response:
HTTP 302 Found
Location: https://example.com/very/long/path
Cache-Control: max-age=3600, s-maxage=3600
```

### The non-obvious design decision: 301 vs 302

There are two ways to do a redirect:

| | 301 Permanent | 302 Temporary |
|---|---|---|
| Browser behaviour | Caches forever — future clicks skip our server | Re-requests every time |
| Analytics | Misses repeat clicks (browser never calls us again) | Captures every click |
| Can update destination? | No — browser cached it forever | Yes |
| Can delete the link? | No — browser ignores deletion | Yes |

**Decision: use 302.** The CDN (`s-maxage=3600`) absorbs the scale. We keep click analytics and the ability to delete links. Offer 301 as an opt-in for power users who want maximum speed and don't need analytics.

### The non-obvious design decision: Idempotency Key

What if a client sends a "create link" request, the server creates the link, but the network drops before the response arrives? The client retries — and now the link is created twice.

The fix: the client sends a unique `Idempotency-Key` header with every create request. The server stores this key for 24 hours. If it sees the same key again, it returns the original response instead of creating a duplicate.

---

## 6. Data Model

### Main table: `url_mappings`

| Column | Type | Notes |
|---|---|---|
| `short_code` | CHAR(7) | Primary key — the 7-character code |
| `long_url` | TEXT | The original URL (may contain sensitive query params) |
| `user_id` | BIGINT | Who created it (null for anonymous) |
| `created_at` | TIMESTAMP | When it was created |
| `expires_at` | TIMESTAMP | When it expires (null = never) |
| `is_deleted` | BOOLEAN | Soft delete — we keep the row to retire the code |
| `redirect_type` | SMALLINT | 301 or 302 |
| `metadata` | JSON | Tags, campaign name, etc. |

### Database choice: CockroachDB

CockroachDB is PostgreSQL-compatible (familiar SQL) but built for distributed scale. It automatically splits data across many servers and replicates it for reliability. We chose it because:
- We need PostgreSQL-style SQL and strong consistency for the unique short_code constraint
- We need to scale to 100TB across multiple servers without manual sharding
- It handles multiple geographic regions natively

### Sharding key: `short_code`

> **Sharding** means splitting your database rows across multiple servers. You need a rule for which server stores which row.

We shard by `hash(short_code)`. This spreads rows evenly across all servers. Since every redirect is a lookup by `short_code`, each redirect hits exactly one server — fast and predictable.

We do **not** shard by `user_id`, even though it would make "list all my links" faster — because that would make every redirect hit two servers (look up user → look up code), doubling latency on the 1M/sec hot path.

---

## 7. Deep Dives

### Deep Dive 1: How do we generate unique short codes at 6,000/sec?

#### The options

**Option A: Hash the URL**
Take the long URL, run it through MD5, take the first 7 characters.

Problem: math. With 100 million links, the probability of two URLs getting the same 7-character code is essentially 100% (birthday paradox — the same reason two people in a group of 23 share a birthday more often than you'd expect). You'd need complex collision-handling logic on the critical write path.

**Option B: Global counter**
Keep a counter somewhere (database or Redis). Each new link gets the next number, encoded as Base62.

Problem: the counter is a single point of failure and a throughput limit. A single Redis node maxes out at ~100,000 operations per second and if it goes down, link creation stops entirely.

**Option C: Pre-generated key pool ← chosen**

Generate millions of random codes offline (background job, no rush). Store them in a `key_pool` table. When a link is created, claim one code from the pool atomically.

**Why this works:**
- Codes are generated offline with guaranteed uniqueness — no collision risk
- Claiming a code from the pool is a simple database delete — fast and reliable
- No single bottleneck: each Write Service server pre-fetches 1,000 codes from the pool into local memory at startup, eliminating database calls on the hot path

#### The exact mechanism

```sql
-- Claim one code atomically from the pool
-- FOR UPDATE SKIP LOCKED: if two servers run this simultaneously,
-- they each get a DIFFERENT code — no waiting, no contention
DELETE FROM key_pool
WHERE short_code = (
  SELECT short_code FROM key_pool
  LIMIT 1
  FOR UPDATE SKIP LOCKED
)
RETURNING short_code;
```

Each Write Service server runs this once to grab 1,000 codes into memory. It refills asynchronously when the local buffer drops below 100. In normal operation, link creation never touches the key pool database at all.

**Monitoring:** Alert when the pool drops below 1 million codes. The background generator runs continuously and keeps it topped up.

---

### Deep Dive 2: How do we serve 1.16M redirects/second in under 50ms?

The answer is **four layers of cache**, each serving a different slice of traffic. Think of it as a series of filters — each layer handles what it can, and only passes the rest deeper.

#### Layer 0: Browser cache (for 301 links)
If a user has clicked this link before and it's a 301 redirect, their browser cached it. Zero network request. Approximately 20–30% of repeat visitors.

#### Layer 1: CDN edge cache (~75% of remaining traffic)
Cloudflare has 200+ servers worldwide. When the first person in London clicks a link, Cloudflare's London server caches it. Every subsequent click from London is served from that same nearby server — no trip to our origin servers.

- TTL: 1 hour (`s-maxage=3600` in the response header)
- Response time: ~5ms (nearest PoP, not crossing the Atlantic)
- Capacity: effectively unlimited

#### Layer 2: In-process memory cache (~15% of CDN misses)
Each of our 20 Redirect Service servers keeps the 10,000 hottest links in its own local memory. This is the fastest possible lookup — no network, no serialization, just a hash map.

- Size: ~5MB per server
- TTL: 60 seconds
- Why it matters: a viral URL with 1M clicks/sec — if all 20 servers have it locally, Redis only sees ~20 requests/sec for that key

#### Layer 3: Redis (~95% of layer 2 misses)
Redis is an in-memory key-value store — much faster than a database because everything is in RAM, not on disk.

- Size: 500MB hot set fits comfortably (Redis nodes are 64GB)
- TTL: 1 hour per key (resets on each access)
- Response time: ~1ms
- Eviction: when memory is full, kick out the least-recently-used key

#### Layer 4: Database (~5% of Redis misses — ~5,800/sec)
Only reached when a URL is genuinely cold (not in any cache). At a 99%+ combined cache hit rate, the database sees less than 6,000 redirects/second — well within what read replicas can handle.

```
Total traffic:  1,160,000 / sec
After CDN:        290,000 / sec  (75% served at CDN)
After L2 memory:   246,500 / sec  (15% of CDN misses served from memory)
After Redis:        12,325 / sec  (95% of remainder served from Redis)
After DB replicas:   ~5,800 / sec  (source of truth — always the answer)
```

#### The cache stampede problem

Imagine a link's cache entry expires at exactly 12:00:00. In that split second, 10,000 concurrent users all get a cache miss simultaneously. All 10,000 try to query the database at once. The database is suddenly slammed.

**Fix: probabilistic early expiration.** Before a cache entry expires, start refreshing it in the background — with increasing probability as expiry approaches. At 60 seconds left, there's a tiny chance of refresh. At 1 second left, there's a near-certain refresh. This smooths out the expiry cliff.

```
# The closer to expiry, the more likely we refresh early
# This means the entry is almost always refreshed before
# thousands of users simultaneously hit a cold cache
```

**Fix 2: Redis lock.** The first request that gets a cache miss acquires a lock and queries the database. All other simultaneous requests wait briefly and then re-check the cache (which the first request just populated).

#### Cache invalidation on delete

When a user deletes a link, the database is updated immediately. But the CDN still has the old entry cached for up to 1 hour.

The fix: on delete, call the Cloudflare cache purge API to evict the entry from all edge nodes. This takes ~500ms to propagate globally. We also delete the Redis key immediately. There's a short window (~500ms) where a deleted link still redirects — acceptable for most use cases, and documented in the SLA.

---

## 8. Handling Failures

**Redis goes down**
- ~15% of redirects served from each server's local memory cache
- Remaining redirects fall through to the database directly
- Database read replicas see ~3x normal load — manageable
- Redis auto-recovers via replica promotion in ~30 seconds
- No link creation failures (Redis is not in the write path for new links)

**Database primary goes down**
- Reads: unaffected — all reads go to read replicas
- Writes (link creation): fail with 503 "Service Unavailable, retry in 10 seconds"
- CockroachDB automatically elects a new leader in under 5 seconds
- Clients retry with the same Idempotency-Key — no duplicate links created

**Kafka goes down**
- Click analytics events are silently dropped (fire-and-forget — we never wait for Kafka before sending the redirect)
- Redirects are completely unaffected
- Analytics may under-count by a few percent during the outage
- This is acceptable — analytics precision is not a correctness requirement

**One AWS availability zone goes down (e.g. power failure)**
- CockroachDB: 3 copies of each row across 3 zones. Losing 1 zone = still have 2 copies = still working
- Redis: same pattern — each shard's primary and replica are in different zones
- App servers: spread across zones, load balancer routes around the failed zone

---

## 9. Key Trade-offs

| Decision | What we chose | What we rejected | Why |
|---|---|---|---|
| Short code generation | Pre-generated key pool | Global counter (Redis INCR) | Counter = single point of failure; key pool has no bottleneck |
| Short code generation | Pre-generated key pool | Hash of the URL | Birthday paradox: near-certain collisions at 100M links |
| Redirect type | 302 (temporary) by default | 301 (permanent) | 301 breaks analytics and makes deletion impossible |
| Analytics writes | Async via Kafka | Synchronous DB write | Writing analytics inline would add ~10ms to every redirect |
| Cache | Redis | Memcached | Redis supports TTL per key and has better cluster support |
| Consistency | Eventual (new links visible in ~1s globally) | Strong (visible everywhere instantly) | Strong consistency would add ~150ms to every write — unacceptable |
| Shard key | `short_code` | `user_id` | Every redirect is a lookup by `short_code` — must be single-shard lookup |

---

## 10. Interview Playbook

### Minute-by-minute guide

| Time | What to do |
|---|---|
| 0–5 min | Ask clarifying questions. Confirm: are analytics needed? custom aliases? TTL? global users? Lock in the 301 vs 302 question — it signals depth. |
| 5–10 min | Estimate scale. Derive the two key numbers: **1.16M redirects/sec** and **500MB hot set**. State what they force: "these numbers mean CDN and Redis are non-negotiable." |
| 10–20 min | Draw the architecture. Start with MVP (server + DB). Show what breaks. Add CDN, Redis, then service split. Explain what each component does in one sentence. |
| 20–35 min | Go deep on the two hard problems. Key pool generation (why counter fails, how SKIP LOCKED works). Four-layer cache (what % each layer handles, why the ordering matters). |
| 35–45 min | Failure scenarios, trade-offs table, invite follow-up questions. |

### What separates an L6 answer from an L4 answer

1. **L6 frames the hard problems before drawing.** L4 starts drawing boxes immediately. Framing shows you understand what's genuinely difficult vs. what's routine.

2. **L6 explains 301 vs 302 with Cache-Control headers.** L4 picks one without reasoning. This trade-off (analytics accuracy vs. browser caching vs. mutability) is specific to this system.

3. **L6 has numbers that justify every decision.** "500MB hot set fits in Redis" isn't a guess — it's derived from Zipf distribution × record size. Numbers make decisions defensible.

### Three hard follow-up questions

**"A celebrity tweets your link. You go from 1,000 to 500,000 req/sec in 30 seconds. What happens?"**

Good answer: CDN serves 75% from edge cache (<5ms, no origin hit). Local memory cache on each server serves the next ~15%. Redis handles the remaining ~10%. At full viral scale, the database sees near-zero traffic for that specific URL. The only risk is the first 1–2 seconds before the CDN warms up — 200 PoPs × 1 miss each = 200 origin requests, totally manageable.

**"How would you implement a link that expires after 1,000 clicks?"**

Good answer: Don't increment a database counter on every redirect — that's 1M writes/sec on one row. Instead: store a click counter in Redis (`INCR clicks:xK3p9Qa`). Check against max_clicks in the same Redis call (Lua script for atomicity). Flush to the database every 1,000 increments. Accept that a few extra clicks may slip through in the flush window — precision isn't critical here.

**"How do you delete a user's data for GDPR compliance?"**

Good answer: it's not just one delete. Database: null out the `long_url` field (removes PII), keep the row so the code isn't reused. Redis: delete the key immediately. CDN: call Cloudflare's purge API (~500ms to propagate). ClickHouse (analytics): async delete job within 30 days. Backups: document that PII remains in backups until rotation (disaster recovery only). Track completion of each step in an audit table. Full erasure within 30 days as required by GDPR.

### If you're running short on time, skip these

- Multi-region deployment details (mention it exists, don't detail it)
- Observability / alerting specifics
- Deployment strategy (mention canary deploys, skip the rollback thresholds)
- Security details beyond "rate limiting and JWT auth"
