# TinyURL — Production System Design
### Visual + Technically Precise Study Guide for L5/L6 Interviews

---

## Step 0 — Frame the Hard Problems

```
╔══════════════════════════════════════════════════════════════════════════════╗
║              WHAT MAKES TINYURL GENUINELY HARD                             ║
╠═══════════════════════════════════════╦════════════════════════════════════╣
║  HARD PROBLEM #1                      ║  HARD PROBLEM #2                   ║
║  Globally Unique ID Generation        ║  Sub-10ms Redirects at             ║
║  Without Coordination Bottlenecks     ║  Pathological Read Skew            ║
║                                       ║                                    ║
║  6,000 writes/sec. Each needs a       ║  1 viral URL = millions of         ║
║  globally unique 7-char code.         ║  req/sec to a SINGLE key.          ║
║  Any centralized lock = SPOF +        ║  p99 < 50ms means you cannot       ║
║  throughput ceiling.                  ║  touch a DB on redirect.           ║
║                                       ║                                    ║
║  Naive fail: auto-increment counter   ║  Naive fail: SELECT long_url       ║
║  → Redis INCR hits ~100K/sec limit    ║  FROM db WHERE short_code=X        ║
║  → Single Redis node = SPOF           ║  → DB melts at 10x traffic         ║
║  → Sequential codes = enumerable      ║  → p99 > 500ms for viral URLs      ║
╚═══════════════════════════════════════╩════════════════════════════════════╝

Every design decision below anchors to one of these two problems.
```

---

## 1. Clarifying Questions

```
┌─────────────────────────────────────────────────────────────────────────┐
│  ASK BEFORE DRAWING ANYTHING  (first 5 minutes)                        │
├──────────────────────────────────┬──────────────────────────────────────┤
│  FUNCTIONAL                      │  NON-FUNCTIONAL                      │
│                                  │                                      │
│  □ Custom aliases allowed?       │  □ Redirect latency target?          │
│  □ URL expiry / TTL per URL?      │  □ Availability: 99.9% or 99.99%?   │
│  □ Click analytics needed?        │  □ Consistency: strong / eventual?  │
│  □ Update/delete after creation? │  □ Expected DAU / QPS?               │
│  □ User accounts or anonymous?   │  □ Single-region or global?          │
│  □ URL safety scanning?          │  □ GDPR / data residency needed?     │
│  □ Custom domain mapping?        │  □ Team size / ops budget?           │
│                                  │                                      │
│  THE ONE QUESTION THAT SIGNALS   │  301 vs 302? ← This alone           │
│  DEPTH:                          │  separates L4 from L6                │
└──────────────────────────────────┴──────────────────────────────────────┘
```

### 301 vs 302 — The Technical Trade-off

```
GET /xK3p9Qa HTTP/1.1

── 301 Permanent Redirect ──────────────────────────────────────────────
HTTP/1.1 301 Moved Permanently
Location: https://destination.com/path
Cache-Control: max-age=31536000, immutable
                        ↑
                Browser caches FOREVER. Next visit = zero network request.

  PRO:  Zero server load on repeat visits
  CON:  Destination cannot be changed after first visit (browser cache)
  CON:  Analytics blind to repeat visitors from same browser
  CON:  Delete/expiry is unenforceable (browser ignores it)

── 302 Found / Temporary Redirect ──────────────────────────────────────
HTTP/1.1 302 Found
Location: https://destination.com/path
Cache-Control: max-age=3600, s-maxage=3600
                        ↑             ↑
                Browser doesn't    CDN caches 1hr,
                cache 302s         handles scale

  PRO:  Every click tracked (analytics accurate)
  PRO:  Destination updatable, expiry enforceable
  PRO:  CDN absorbs scale (s-maxage); browser doesn't cache
  CON:  Browser always makes a network request

DECISION: 302 default + s-maxage=3600 for CDN.
          Offer 301 as explicit opt-in for power users.
```

### Scope

| In Scope (V1) | Out of Scope → V2 |
|---|---|
| Shorten URL → 7-char code | Custom domains (`brand.com/abc`) |
| Redirect short → long URL | URL preview / screenshot |
| Optional custom alias | QR code generation |
| Optional TTL per URL | Real-time analytics dashboard |
| Async click analytics | ML spam detection (V1: rules only) |
| User accounts (JWT auth) | Bulk URL import API |
| Delete / soft-deactivate | |

### Non-Functional Targets

```
┌─────────────────────────────────────────────────────────┐
│  Redirect latency    p50 < 5ms     p99 < 50ms           │
│  Availability        99.99%  (< 52 min downtime/year)   │
│  Durability          No URL mapping lost after creation  │
│  Consistency         Eventual OK  (new URL visible ~1s)  │
│  Write QPS           ~6,000/sec peak                     │
│  Read QPS            ~1,160,000/sec peak (viral burst)   │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Back-of-Envelope Estimation

### Request Volume

```
ASSUMPTIONS: 100M new URLs/day  •  Read:write = 100:1  •  5yr retention

WRITES:
  100,000,000 / 86,400 sec = 1,157/sec  avg
                           ×5 peak      = 5,787/sec  ≈ 6,000/sec peak

READS (redirects):
  100M × 100 = 10B redirects/day
  10B / 86,400 = 115,741/sec  avg
              ×10 viral burst = 1,157,407/sec  ≈ 1.16M/sec peak
                                              ↑
                              THIS is the number that drives all caching decisions
```

### Storage Per Record

```
┌──────────────────────────────────┬───────────┐
│  Field                           │  Size     │
├──────────────────────────────────┼───────────┤
│  short_code      CHAR(7)         │    7 B    │
│  long_url        TEXT avg 200    │  200 B    │
│  user_id         BIGINT          │    8 B    │
│  timestamps (×2) TIMESTAMPTZ     │   16 B    │
│  is_deleted      BOOLEAN         │    1 B    │
│  metadata        JSONB ~100B     │  100 B    │
│  B-tree + MVCC overhead          │  168 B    │
├──────────────────────────────────┼───────────┤
│  TOTAL                           │ ~500 B    │
└──────────────────────────────────┴───────────┘

5-year storage:
  100M/day × 365 × 5 × 500B = ~91 TB raw + ~10 TB index = ~100 TB total
  → Far exceeds single-node PostgreSQL. Sharding is mandatory.
```

### Hot Set & Cache Sizing

```
Power-law (Zipf) distribution of URL traffic:
  Top 1%  of 100M URLs = 1M URLs  → receive ~80% of traffic
  Top 10% of 100M URLs = 10M URLs → receive ~95% of traffic

Hot set memory:
  1M "hot" URLs × 500B = 500 MB ← fits in single Redis node (64 GB typical)
  Cache hit rate target: >95%
  → DB sees only: 1.16M/sec × (1-0.95) × (1-0.15 L2) × ... ≈ 5,800 req/sec net
```

### Numbers That Drive Design Decisions

```
┌─────────────────────────────────────────────────────────────────────┐
│  NUMBER              VALUE         DESIGN DECISION DRIVEN           │
├─────────────────────────────────────────────────────────────────────┤
│  Peak redirects     1.16M/sec     → CDN + 4-layer cache mandatory  │
│  Hot set size       500 MB        → Single Redis node fits hot set  │
│  DB reads (net)     5,800/sec     → Read replicas handle easily     │
│  5yr storage        100 TB        → Hash-shard DB (not single node) │
│  Key space 62^7     3.52 trillion → Never exhaust for 100yr+        │
│  Write peak         6,000/sec     → Key pool approach (no coord.)   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. High-Level Architecture

### MVP → Production Evolution

```
STAGE 1 — MVP (0–100K users)
  Browser ──► Nginx ──► Node.js App ──► PostgreSQL (single node)
  BREAKS AT: ~5,000 redirects/sec (DB overwhelmed)

STAGE 2 — Add Read Replica + Redis
  Browser ──► Nginx ──► App ──► Redis ──► PG Primary (writes)
                                      └──► PG Replica (cache miss reads)
  BREAKS AT: Single Redis SPOF; ~50K redirects/sec

STAGE 3 — CDN + Service Split
  Browser ──► CDN ──► LB ──► Redirect Svc ──► Redis ──► DB Replica
                         └──► Write Svc    ──► DB Primary
  BREAKS AT: Single region; storage nearing ~5TB

STAGE 4 — Production (multi-region, sharded DB)  ← see diagram below
```

### Full Production Architecture

```
                    ┌─────────────────────────────────────────┐
                    │   GeoDNS (Cloudflare / AWS Route53)     │
                    │   Latency-based routing to nearest PoP  │
                    └───────────────┬─────────────────────────┘
                                    │
          ┌─────────────────────────┼──────────────────────────┐
          │ US-EAST-1               │                          │ EU-WEST-1
          ▼                         │                          ▼
  ┌───────────────┐                 │               ┌───────────────┐
  │ Cloudflare    │                 │               │ Cloudflare    │
  │ CDN Edge PoP  │                 │               │ CDN Edge PoP  │
  └───────┬───────┘                 │               └───────┬───────┘
          │ cache miss              │                       │ cache miss
          ▼                         │                       ▼
  ┌───────────────────────────────────────────────────────────────────┐
  │                       US-EAST-1 REGION                            │
  │                                                                   │
  │  ┌────────────────────────────────────────────────────────────┐  │
  │  │  API Gateway (Kong / AWS API GW)                           │  │
  │  │  • JWT/API-key auth        • Rate limiting (token bucket)  │  │
  │  │  • Request validation      • TLS termination               │  │
  │  │  • Routing (path-based)    • WAF rules                     │  │
  │  └────────────────────┬───────────────────────────────────────┘  │
  │                       │                                           │
  │          ┌────────────┴─────────────┐                           │
  │          │                          │                           │
  │  ┌───────▼────────┐        ┌────────▼───────┐                  │
  │  │  Redirect Svc  │        │   Write Svc    │                  │
  │  │  (x20 pods)    │        │   (x5 pods)    │                  │
  │  │  Stateless     │        │  Stateless     │                  │
  │  │  L2 LRU:10K   │        │  Key pool:     │                  │
  │  │  entries       │        │  1000 pre-fetched               │
  │  └───────┬────────┘        └───────┬────────┘                  │
  │          │                         │                            │
  │  ┌───────▼────────────────┐        │                            │
  │  │  Redis Cluster (L3)    │        │                            │
  │  │  6 shards, 2 replicas  │        │                            │
  │  │  each, spread 3 AZs    │        │                            │
  │  └───────┬────────────────┘        │                            │
  │          │ cache miss              │                            │
  │          └─────────────┬───────────┘                            │
  │                        │                                         │
  │  ┌─────────────────────▼──────────────────────────────────┐    │
  │  │  CockroachDB Cluster                                   │    │
  │  │  • 9 nodes: 3 per AZ × 3 AZs                          │    │◄── async
  │  │  • Sharded by hash(short_code)                         │    │    cross-region
  │  │  • REGIONAL BY ROW replication                         │    │    replication
  │  └────────────────────────────────────────────────────────┘    │
  │                        │                                         │
  │  ┌─────────────────────▼──────────────────────────────────┐    │
  │  │  Kafka Cluster (9 brokers, RF=3, min.ISR=2)            │    │
  │  │  Topics: url.created, url.clicks, url.deleted          │    │
  │  └─────────────────────┬──────────────────────────────────┘    │
  │                        │                                         │
  │  ┌─────────────────────▼──────────────────────────────────┐    │
  │  │  ClickHouse Cluster (Analytics OLAP)                   │    │
  │  │  Columnar, partitioned by month, MergeTree             │    │
  │  └────────────────────────────────────────────────────────┘    │
  └───────────────────────────────────────────────────────────────────┘
```

### Redirect: Sequence Diagram

```
Browser  CDN Edge  LB   Redirect Svc    Redis    DB Replica   Kafka
  │         │       │        │              │          │          │
  ├─GET /xK─►       │        │              │          │          │
  │  ┌──────┤       │        │              │          │          │
  │  │CDN   │       │        │              │          │          │
  │  │HIT   │       │        │              │          │          │
  │  │~75%  │       │        │              │          │          │
  ◄──┴302───┤(DONE) │        │              │          │          │
  │  ┌──────┤       │        │              │          │          │
  │  │CDN   ├──────►│        │              │          │          │
  │  │MISS  │       ├───────►│              │          │          │
  │  │~25%  │       │  ┌─────┤              │          │          │
  │  │      │       │  │L2   │              │          │          │
  │  │      │       │  │HIT  │              │          │          │
  │  │      │◄──────┼──┴302──┤(~2ms total)  │          │          │
  │  │      │       │  ┌─────┤──GET key────►│          │          │
  │  │      │       │  │Redis│◄─HIT (~1ms)──┤(DONE)    │          │
  │  │      │◄──────┼──┴302──┤              │          │          │
  │  │      │       │  ┌─────┤──SELECT──────────────►  │          │
  │  │      │       │  │DB   │◄─row (~10ms)────────────┤(DONE)    │
  │  │      │◄──────┼──┴302──┤──SET Redis ─►           │          │
  ◄──┴302───┤       │        │                          │          │
  │         │       │        ├─publish click (fire+forget)────────►
```

---

## 4. API Design

### Protocol Choice

```
REST ✓  vs  gRPC  vs  GraphQL
─────────────────────────────────────────────────────────────────
WHY REST:
  • Redirect MUST use HTTP 301/302 with Location header (native REST)
  • CDN and browser caching work natively with HTTP semantics
  • gRPC over HTTP/2 cannot return redirect codes directly
  • GraphQL has no concept of HTTP redirect response codes
```

### Core Endpoints

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
POST /api/v1/urls                    ← CREATE SHORT URL
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Headers:
  Authorization:    Bearer <jwt_access_token>
  Idempotency-Key:  <client-uuid-v4>   ← REQUIRED for safe retries
  Content-Type:     application/json

Body:
  {
    "long_url":      "https://example.com/very/long/path?q=1",
    "custom_code":   "my-brand",         // optional, 3-20 chars
    "ttl_seconds":   2592000,            // optional, null = permanent
    "redirect_type": 302,               // optional, default 302
    "tags":          ["campaign-q2"]    // optional, analytics grouping
  }

Response 201:
  {
    "short_code":    "xK3p9Qa",
    "short_url":     "https://tiny.url/xK3p9Qa",
    "redirect_type": 302,
    "created_at":    "2026-05-02T10:00:00Z",
    "expires_at":    "2026-06-01T10:00:00Z"
  }

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GET /{short_code}                    ← REDIRECT (THE HOT PATH)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
No auth. Public. Rate-limited by IP at CDN level.

Response 302:   Location: https://destination.com/path
                Cache-Control: max-age=3600, s-maxage=3600
Response 301:   Location: https://destination.com/path
                Cache-Control: max-age=31536000, immutable
Response 404:   { "error": "NOT_FOUND",  "retry_safe": false }
Response 410:   { "error": "EXPIRED",    "retry_safe": false }

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GET    /api/v1/urls/{short_code}     ← GET METADATA + ANALYTICS
DELETE /api/v1/urls/{short_code}     ← SOFT DELETE
GET    /api/v1/urls?cursor=X&limit=N ← PAGINATED LIST
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Idempotency Key: Exact Implementation

```
Step 1 — Redis SET NX (only sets if key doesn't exist)
─────────────────────────────────────────────────────────────
  result = redis.set(
    key   = "idem:{uuid}",
    value = "PROCESSING",   // sentinel
    nx    = true,
    px    = 86400000        // 24hr TTL
  )
  if result == nil:         // key already existed
    existing = redis.get("idem:{uuid}")
    if existing == "PROCESSING": return 409  // concurrent duplicate
    return existing                          // cached 201 response

Step 2 — DB transaction (atomic: URL insert + idem log)
─────────────────────────────────────────────────────────────
  BEGIN TRANSACTION;
    INSERT INTO url_mappings (...) VALUES (...);
    INSERT INTO idempotency_log (idem_key, short_code, expires_at)
      VALUES ($1, $2, NOW() + INTERVAL '24 hours');
  COMMIT;

Step 3 — Overwrite Redis sentinel with actual response
─────────────────────────────────────────────────────────────
  redis.set("idem:{uuid}", json_encode(response_201), px=86400000)

CRASH RECOVERY:
  If crash between COMMIT and redis.set:
    Next retry sees "PROCESSING" in Redis
    Check idempotency_log in DB → reconstruct response → return 201
    (DB unique constraint prevents double-insert)
```

### Error Model

```
┌─────────┬──────────────────────────┬─────────────┐
│  Code   │  Error                   │  Retry safe │
├─────────┼──────────────────────────┼─────────────┤
│  201    │  —                       │  N/A        │
│  400    │  INVALID_URL             │  No         │
│  401    │  UNAUTHORIZED            │  No         │
│  404    │  NOT_FOUND               │  No         │
│  409    │  IDEM_IN_FLIGHT          │  Wait 1s    │
│  409    │  CUSTOM_CODE_TAKEN       │  No         │
│  410    │  EXPIRED                 │  No         │
│  422    │  MALFORMED_REQUEST       │  No         │
│  429    │  RATE_LIMIT_EXCEEDED     │  Yes ✓      │
│  500    │  INTERNAL_ERROR          │  Yes ✓      │
│  503    │  SERVICE_UNAVAILABLE     │  Yes ✓      │
└─────────┴──────────────────────────┴─────────────┘
```

### Pagination: Cursor-Based (Never Offset)

```sql
-- Cursor = Base64({"created_at": "2026-05-02T10:00:00Z", "short_code": "xY3z9Qa"})
SELECT short_code, long_url, created_at
FROM url_mappings
WHERE user_id = $1
  AND is_deleted = FALSE
  AND (created_at, short_code) < ($cursor_ts, $cursor_code)
ORDER BY created_at DESC, short_code DESC
LIMIT 51;  -- fetch 51; if 51 returned → has_more=true

-- WHY NOT OFFSET: OFFSET n scans and discards n rows → O(n)
-- At offset=10000: scan 10K rows per page → unusable at scale
-- Cursor-based: always O(1) index seek regardless of page depth
```

---

## 6. Data Modeling

### Schema

```sql
-- PRIMARY TABLE — sharded by hash(short_code)
CREATE TABLE url_mappings (
  short_code    CHAR(7)      NOT NULL,
  long_url      TEXT         NOT NULL CHECK (length(long_url) <= 2048),
  user_id       BIGINT,                    -- NULL for anonymous
  created_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  expires_at    TIMESTAMPTZ,               -- NULL = permanent
  is_deleted    BOOLEAN      NOT NULL DEFAULT FALSE,
  redirect_type SMALLINT     NOT NULL DEFAULT 302 CHECK (redirect_type IN (301,302)),
  metadata      JSONB,
  -- CockroachDB: pin row to creation region for low-latency local reads
  crdb_region   crdb_internal_region NOT NULL DEFAULT gateway_region(),

  PRIMARY KEY (short_code)
) LOCALITY REGIONAL BY ROW;

-- User URL listing: paginated by recency
CREATE INDEX idx_url_mappings_user_created
  ON url_mappings (user_id, created_at DESC, short_code DESC)
  WHERE is_deleted = FALSE;

-- TTL expiry cleanup job
CREATE INDEX idx_url_mappings_expires
  ON url_mappings (expires_at ASC)
  WHERE expires_at IS NOT NULL AND is_deleted = FALSE;

-- Idempotency log: durable across Redis restarts
CREATE TABLE idempotency_log (
  idem_key    UUID         NOT NULL,
  short_code  CHAR(7)      NOT NULL,
  user_id     BIGINT,
  created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  expires_at  TIMESTAMPTZ  NOT NULL,
  PRIMARY KEY (idem_key),
  INDEX idx_idem_expires (expires_at ASC)
);

-- Key pool: pre-generated available short codes
CREATE TABLE key_pool (
  short_code CHAR(7) NOT NULL,
  PRIMARY KEY (short_code)
);
```

```sql
-- ANALYTICS (ClickHouse — separate OLAP cluster)
CREATE TABLE url_clicks
(
  short_code      String,
  clicked_at      DateTime,
  ip_country      LowCardinality(String),
  referrer_domain String,
  user_agent_hash UInt64,    -- hashed, not raw (privacy)
  request_id      UUID       -- dedup key for at-least-once delivery
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/url_clicks', '{replica}')
PARTITION BY toYYYYMM(clicked_at)
ORDER BY (short_code, clicked_at)
SETTINGS index_granularity = 8192;

-- Materialized view: pre-aggregate hourly click counts
CREATE MATERIALIZED VIEW url_click_counts_hourly
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(hour)
ORDER BY (short_code, hour)
AS SELECT
  short_code,
  toStartOfHour(clicked_at) AS hour,
  count()                    AS clicks
FROM url_clicks
GROUP BY short_code, hour;
```

### Database Selection

```
url_mappings (OLTP):
  Need: strong uniqueness constraint, PK lookup at 100K+ RPS, geo-dist.
  → CockroachDB (PostgreSQL-compatible, Raft-based, native geo-dist.)
     OR DynamoDB (if AWS-only, simpler ops, weaker SQL)
     OR Vitess/MySQL (if team has MySQL expertise)

click analytics (OLAP):
  Need: GROUP BY + time range aggs, 116K appends/sec, no point lookups
  → ClickHouse (columnar, 100x faster aggregations than PostgreSQL)

hot URL cache:
  Need: sub-ms reads, key-value, TTL per key, 500MB hot set
  → Redis Cluster
  NOT Memcached: no TTL per key in cluster mode, no persistence
```

### CockroachDB: REGIONAL BY ROW vs GLOBAL

```
GLOBAL TABLE (sync replication to all regions)
  Write: commit must ack from EU + APAC → ~150ms latency
  Read:  always fresh from any region
  USE FOR: reserved codes blocklist (small, write-once)

REGIONAL BY ROW (home region per row, async cross-region)
  Write: commit in home region only → ~5ms latency
  Read:  home region = fresh; other region = stale by ~500ms
  USE FOR: url_mappings  ← CHOSEN
  
  Trade-off: new EU URL may be invisible in US for ~500ms.
             Accepted: 404 window is brief, self-heals, rare in practice.
```

### Sharding: Hot-Spot Analysis

```
Shard key: hash(short_code) ← CHOSEN
  Redirect path = primary key lookup (O(log n), single shard)
  User listing  = scatter-gather (acceptable: low-frequency operation)

  Hot-spot math: top URL gets 1% of traffic = 1.16M × 0.01 = 11.6K req/sec
  BUT: hot URL is served from cache → shard NEVER sees this traffic
  Cache hit rate eliminates hot-shard problem entirely.

Shard key: user_id ← REJECTED
  User listing = single shard (good)
  Redirect = must lookup (short_code→user_id→shard) = 2 hops (bad)
  Kills p99 on the 1.16M/sec hot path.
```

---

## 6b. Short Code Generation — Technical Deep Dive

### Collision Math (Birthday Paradox)

```
HASH-BASED APPROACH: Base62(first 7 chars of MD5(long_url))
  Key space K = 62^7 = 3,521,614,606,208 (~3.52 trillion)

  P(collision after n insertions) ≈ 1 - e^(-n²/2K)

  n = 100K:  P ≈ 1 - e^(-0.00142)  ≈ 0.14%   (1 in ~700)
  n = 1M:    P ≈ 1 - e^(-0.142)    ≈ 13.2%
  n = 100M:  n²/2K = 1,420         → P ≈ 1.0 (certain)

  At 6,000 writes/sec, even 0.14% collision rate = 8 retries/sec.
  At 100M URLs, collisions are mathematically certain.
  → Hash approach requires retry logic + race condition handling.
  → Key pool eliminates this entirely.
```

### Key Pool: Exact Implementation

```sql
-- OFFLINE: batch generate keys (background job, runs continuously)
INSERT INTO key_pool (short_code)
SELECT DISTINCT
  array_to_string(
    ARRAY(SELECT substring(
      'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'
      FROM (floor(random() * 62) + 1)::int FOR 1)
    FROM generate_series(1, 7)
  ), '')
FROM generate_series(1, 10000000)  -- generate 10M at a time
ON CONFLICT (short_code) DO NOTHING;

-- ONLINE: claim one key atomically (per URL creation)
DELETE FROM key_pool
WHERE short_code = (
  SELECT short_code
  FROM   key_pool
  LIMIT  1
  FOR UPDATE SKIP LOCKED   -- ← no waiting, no contention between pods
)
RETURNING short_code;

-- PRE-FETCH: each Write Service pod fetches 1000 keys at startup
-- and refills async when local buffer drops below 100
-- → zero DB calls per request on the write hot path
```

```go
// Pre-fetch optimization (Go)
type WriteService struct {
    localKeyPool []string
    mu           sync.Mutex
    refillChan   chan struct{}
}

func (ws *WriteService) claimKey() string {
    ws.mu.Lock()
    defer ws.mu.Unlock()
    key := ws.localKeyPool[0]
    ws.localKeyPool = ws.localKeyPool[1:]
    if len(ws.localKeyPool) < 100 {
        select {
        case ws.refillChan <- struct{}{}: // non-blocking trigger
        default:
        }
    }
    return key
}
```

**Monitoring thresholds:**

| Pool size | Status | Action |
|---|---|---|
| > 10M | Healthy | — |
| 1M – 10M | Watch | Accelerate generation |
| < 1M | Alert P2 | Page oncall |
| < 100K | Alert P1 | Incident — creation may fail soon |
| 0 | Critical | URL creation returns 503 |

---

## 7. Write Path / Ingestion

### Kafka Configuration

```
TOPIC: url.clicks (analytics — fire and forget)
  Partitions:          256  (scales consumer parallelism)
  Partition key:       short_code (ordered per URL)
  Replication factor:  3
  min.insync.replicas: 2
  Producer acks:       0  ← fire-and-forget; NEVER block redirect for Kafka ack
  compression:         lz4
  linger.ms:           5  (batch for 5ms to improve throughput)
  max.block.ms:        0  (if buffer full → DROP, not block)

TOPIC: url.created (cross-region cache warming)
  Partitions:          64
  acks:                1  (durability matters for cache warm UX)
  Consumer action:     SET "url:{code}" in Redis of each region

TOPIC: url.deleted (cache invalidation — GDPR critical)
  Partitions:          16
  acks:                all  (deletion must be durable)
  Consumer action:     Redis DEL + CDN purge + ClickHouse delete job
```

### At-Least-Once Dedup in ClickHouse

```sql
-- Each click event carries request_id (UUID).
-- ReplacingMergeTree deduplicates asynchronously during merges.
CREATE TABLE url_clicks (...) ENGINE = ReplicatedMergeTree(...)
ORDER BY (short_code, request_id);

-- Queries use FINAL to force synchronous dedup:
SELECT short_code, count() AS clicks
FROM url_clicks FINAL
WHERE short_code = 'xK3p9Qa'
GROUP BY short_code;
-- FINAL is expensive on large tables; use for analytics queries only.
-- For dashboards: query the pre-aggregated materialized view instead.
```

---

## 8. Read Path / Query Serving

### Four-Layer Cache Architecture

```
LAYER 0: Browser Cache
  Applies to: 301 only  •  TTL: max-age=31536000 (1yr)
  Hit rate: ~20-30% of repeat visitors to 301 URLs
  302 responses: Cache-Control: no-store → never browser-cached

        │ miss (all 302s; cold 301s)
        ▼

LAYER 1: CDN Edge (Cloudflare, 200+ PoPs globally)
  TTL:      s-maxage=3600 (1hr, CDN-shared, not browser)
  Hit rate: ~72-80% of all origin-bound requests
  Cache key: {method}:{short_code}  (strip query params)
  Eviction: TTL expiry + explicit Cloudflare Cache-Tag purge API
  Capacity: unlimited (Cloudflare's distributed network)

        │ miss (~20-28% of requests)
        ▼

LAYER 2: In-Process LRU (per Redirect Service pod)
  Size:     10,000 entries max, ~5 MB per pod
  TTL:      60 seconds (hard expiry)
  Hit rate: ~15% of CDN misses
  Eviction: LRU eviction when full + TTL expiry
  Scope:    single pod (NOT shared; 20 pods = 20 independent L2s)
  Key insight: ultra-hot URL = all 20 pods have it → Redis sees <20 req/sec

        │ miss (~85% of CDN misses)
        ▼

LAYER 3: Redis Cluster
  Size:     6 shards × 64 GB; 500 MB hot set fits easily
  TTL:      3600s per key; reset on each access (GETEX key EX 3600)
  Hit rate: >95% of L2 misses
  Eviction: allkeys-lru (evict LRU when memory full)
  Key:      "url:{short_code}"
  Value:    {"long_url":"...","redirect_type":302,"expires_at":1748865600}
  Redis TTL: min(3600, max(0, expires_at - now()))

        │ miss (~5% of L3 = ~1% of all = ~5,800 req/sec)
        ▼

LAYER 4: CockroachDB Read Replica
  Query:    SELECT long_url, redirect_type, expires_at, is_deleted
            FROM url_mappings WHERE short_code = $1
  Index:    PRIMARY KEY B-tree → O(log n)
  Latency:  ~5-20ms
  Load:     ~5,800 req/sec (well within read replica limits)
  Post-hit: populate Redis + L2; CDN populates itself on next response
```

### Cache Stampede: PER + Redis Mutex

```go
// Probabilistic Early Revalidation (PER / XFetch algorithm)
// Refreshes cache before expiry with probability growing as TTL shrinks
func getWithPER(shortCode string, beta float64) *URLEntry {
    entry, remainingTTL := redis.getWithTTL("url:" + shortCode)
    if entry == nil {
        return fetchFromDB(shortCode)  // hard miss
    }

    delta := 0.010 // seconds to compute fresh value (measure empirically)

    // Revalidate if: -delta * beta * ln(rand()) >= remainingTTL
    if -delta*beta*math.Log(rand.Float64()) >= float64(remainingTTL) {
        go func() {  // async background revalidation
            fresh := fetchFromDB(shortCode)
            redis.setex("url:"+shortCode, 3600, fresh)
        }()
    }
    return entry  // serve (possibly slightly stale) value immediately
}
// At remainingTTL=60s: P(revalidate) ≈ 0.01% per request (low)
// At remainingTTL=1s:  P(revalidate) ≈ 1% per request (high)
// Naturally distributes revalidation load as expiry approaches.


// Redis Mutex: for hard misses (all goroutines miss simultaneously)
func getRedirectURL(shortCode string) *URLEntry {
    entry := redis.get("url:" + shortCode)
    if entry != nil {
        return entry
    }

    lockKey := "lock:" + shortCode
    acquired := redis.set(lockKey, "1", nx=true, px=500) // 500ms lock

    if acquired {
        defer redis.del(lockKey)
        fresh := fetchFromDB(shortCode)
        if fresh != nil {
            redis.setex("url:"+shortCode, 3600, fresh)
        }
        return fresh
    }
    // Another goroutine holds the lock: wait briefly, retry cache
    time.Sleep(10 * time.Millisecond)
    entry = redis.get("url:" + shortCode)
    if entry != nil {
        return entry
    }
    return fetchFromDB(shortCode) // lock holder found nothing (404)
}
```

### Cache Invalidation Race Condition on Delete

```
PROBLEM: ~499ms window where CDN still serves a "deleted" URL

Timeline:
  T=0ms    DB: SET is_deleted=TRUE, long_url=NULL  (committed)
  T=1ms    Return 204 to user
  T=50ms   Kafka "url.deleted" consumed → Redis DEL (immediate)
  T=100ms  CDN purge API call in-flight
  T=499ms  CDN purge fully propagated to all PoPs
  
  T=250ms  User clicks the "deleted" link
           → CDN edge: CACHE HIT (purge not yet propagated!)
           → Returns 302 → user lands on "deleted" destination

ACCEPTABLE for most cases (soft delete, eventually consistent).

FOR HIGH-SECURITY DELETION (e.g., credential leak link):
  Option A: Synchronous CDN purge before 204 → adds ~100-500ms latency
  Option B: Cloudflare Worker checks Redis "deleted:{code}" flag at edge
            → 1 Redis lookup per CDN edge miss (not per redirect)
  Option C: SLA: "deletions propagate within 30 seconds" (document it)

L2 in-process cache window: up to 60s
FIX: Publish to Redis pubsub "url.invalidate" channel on delete.
     Each pod subscribes and evicts from L2 immediately (sub-1ms).
```

---

## 9. Inter-Service Communication

### Timeout Budget Per Hop

```
REDIRECT REQUEST — Total budget: 100ms
─────────────────────────────────────────────────────────────────────
  Operation                       Budget    Notes
  ─────────────────────────────────────────────────────────────────
  DNS + network (client→CDN)       5ms      GeoDNS nearest PoP
  CDN processing                   2ms      Edge cache lookup
  Network (CDN→LB→Redirect Svc)   3ms      Internal network, same region
  L2 in-process cache              0.1ms   Concurrent map lookup
  Redis GET "url:xK3p9Qa"          5ms      95th pct, single shard
  DB replica SELECT (on miss)     20ms      B-tree + index read
  Application logic                2ms      JSON decode, expiry check
  Response serialization           1ms      HTTP 302 construction
  Buffer                          61.9ms
  ─────────────────────────────────────────────────────────────────
  TOTAL (Redis HIT path)          ~18ms    p99 target: < 50ms ✓
  TOTAL (DB HIT path)             ~40ms    p99 target: < 50ms ✓

ENFORCEMENT:
  Redis call: context.WithTimeout(ctx, 20*time.Millisecond)
  DB call:    context.WithTimeout(ctx, 30*time.Millisecond)
  If DB timeout: return 503 immediately. Never remove timeouts.
```

### Circuit Breaker States

```
         error_rate > 50%           2/3 probes succeed
CLOSED ──────────────────► OPEN ──────────────────────► CLOSED
  ▲                          │                              │
  │                          │ 30s timeout                  │
  │                          ▼                              │
  └─────────────────── HALF-OPEN ───────────────────────────┘
                             │
                             │ any probe fails
                             └──────────────► OPEN (reset 30s)

Thresholds:
  Sliding window:      10 seconds
  Volume threshold:    20 requests minimum (avoid cold-start trips)
  Error rate to open:  50% (errors = timeout, conn refused, 500, 503)
  Ignore:              404 (correct "not found" behavior)
  Half-open probes:    3 requests; need 2/3 success to close

Fallback on Redis circuit open:  serve L2 → fall through to DB
Fallback on DB circuit open:     return 503 Retry-After: 5
Both open:                        L2 serves what's available → 503
```

### API Gateway vs Service Mesh

```
┌─────────────────────┬────────────────────────┬──────────────────────┐
│  Concern            │  API Gateway (Kong)    │  Service Mesh (Istio)│
├─────────────────────┼────────────────────────┼──────────────────────┤
│  Auth (JWT/API key) │  ✓                     │  ✗                   │
│  Rate limiting      │  ✓ per-user/IP         │  ✗                   │
│  External TLS term  │  ✓                     │  ✓ (Ingress)         │
│  Internal mTLS      │  ✗                     │  ✓ (sidecar-auto)    │
│  Circuit breaking   │  Limited               │  ✓ Envoy proxy       │
│  Distributed trace  │  ✓ inject trace ID     │  ✓ auto-propagate    │
│  Service discovery  │  ✗                     │  ✓ DNS + registry    │
└─────────────────────┴────────────────────────┴──────────────────────┘
Both needed: API Gateway for north-south; Service Mesh for east-west.
```

---

## 10. Failure Handling

### Component Failure Matrix

```
┌──────────────────┬──────────────────────────────────┬────────────────────────────────┐
│  Component       │  What Fails                      │  Graceful Degradation          │
├──────────────────┼──────────────────────────────────┼────────────────────────────────┤
│ Redis (2/6       │ ~33% of codes miss Redis cache   │ Fall through to DB replicas    │
│ shards down)     │ DB sees ~38K extra req/sec        │ Auto-failover in <30s          │
├──────────────────┼──────────────────────────────────┼────────────────────────────────┤
│ Redis (all down) │ All redirects miss cache          │ L2 serves ~15%; DB handles     │
│                  │ DB sees ~10x normal load          │ rest; scale up replicas        │
├──────────────────┼──────────────────────────────────┼────────────────────────────────┤
│ DB Primary       │ URL creation fails (writes)       │ Return 503 + Retry-After: 10  │
│                  │ Reads: unaffected (replicas)      │ Raft election <5s              │
├──────────────────┼──────────────────────────────────┼────────────────────────────────┤
│ Kafka            │ Analytics events dropped           │ Redirect unaffected (async)   │
│                  │ (url.clicks: acks=0)              │ Under-count by ~5-10%          │
│                  │ url.deleted: retry with backoff   │ Cache invalidation delayed     │
├──────────────────┼──────────────────────────────────┼────────────────────────────────┤
│ CDN PoP          │ Users rerouted to next PoP        │ ~10-30ms extra latency         │
│                  │ Origin sees ~10-30x spike         │ Auto-scale origin fleet        │
├──────────────────┼──────────────────────────────────┼────────────────────────────────┤
│ Entire Region    │ All US-EAST-1 offline             │ GeoDNS failover to EU-WEST-1  │
│                  │                                  │ <60s; data stale by ~500ms     │
└──────────────────┴──────────────────────────────────┴────────────────────────────────┘
```

### Cascading Failure Prevention

```
WITHOUT protection:                 WITH protection:
──────────────────                  ─────────────────────────────
DB slows                            DB slows
    ↓                                   ↓
Connections pile up             Circuit breaker opens
    ↓                                   ↓
Thread pool exhausted           503 returned fast (fail fast)
    ↓                                   ↓
Redirect Svc crashes            DB pressure drops
    ↓                                   ↓
CDN gets 0 responses            DB recovers
    ↓                                   ↓
Total outage                    CB half-opens → closes
                                Partial outage, not cascade

Bulkheads: Redirect Svc and Write Svc are separate K8s Deployments
with separate connection pools. Write Svc crash ≠ Redirect Svc crash.
```

### Raft Quorum Math

```
Quorum = floor(N/2) + 1

N=2: quorum=2 → 1 node failure = unavailable (no quorum)
N=3: quorum=2 → 1 node failure = still have quorum ✓
N=5: quorum=3 → 2 simultaneous failures = still have quorum ✓

CockroachDB: 9 nodes, 3 per AZ × 3 AZs; replication factor = 3
  AZ-1 fails (3 nodes down):
    6 remaining nodes; quorum of 2/3 per range achievable ✓
  AZ-1 + AZ-2 fail (6 nodes down):
    3 nodes in AZ-3; quorum of 2/3 still achievable ✓
    BUT: those 3 nodes may not have latest writes (async replication)
    → This is why multi-region GeoDNS failover exists:
      AZ-1+AZ-2 failure → GeoDNS fails over to EU-WEST-1
```

---

## 11. Scalability

### What Scales Linearly vs What Doesn't

```
SCALES LINEARLY (add nodes = add capacity)
  ✓ Redirect Service   — stateless, add pods behind LB
  ✓ Write Service      — stateless (with key pool approach)
  ✓ CDN                — Cloudflare handles globally
  ✓ Redis Cluster      — add shards (online resharding, operational cost)
  ✓ Kafka              — add partitions + brokers
  ✓ ClickHouse         — add nodes to cluster

DOES NOT SCALE LINEARLY
  ✗ DB primary         — single-leader writes; must shard (Vitess / CockroachDB)
  ✗ Key pool           — needs offline replenishment job + monitoring
  ✗ Cache invalidation — CDN purge API has rate limits (Cloudflare: 1000 purges/s)
  ✗ Cross-region sync  — network latency is physics
```

### Viral URL: Layer-by-Layer at 500K req/sec

```
Scenario: Celebrity tweet → 0 to 500K req/sec for "xK3p9Qa" in 30s

┌─────────────────────┬───────────────────────────────────────────────┐
│  Layer              │  What Happens                                 │
├─────────────────────┼───────────────────────────────────────────────┤
│  CDN (~75%)         │  ~375K req/sec served at edge, sub-5ms        │
│  L2 (~15% of miss)  │  ~19K req/sec served in-process, sub-1ms      │
│  Redis (>95% miss)  │  ~100K req/sec (single key, Redis handles)    │
│  DB (5% of Redis)   │  ~5K req/sec (manageable on read replicas)    │
└─────────────────────┴───────────────────────────────────────────────┘

Redis hot-key risk: "url:xK3p9Qa" at 100K GET/sec on 1 shard
  → Redis shard CPU: ~30% at 100K ops/sec → manageable
  → If > 500K req/sec to single key (Super Bowl):
      Replicate key across ALL shards; hash to ANY shard (6x load spread)
      OR: L2 in-process cache in all 20 pods handles it first

Detection: redis OBJECT FREQ (LFU policy); alert if key > 50K ops/sec/shard
```

---

## 12. Durability & Availability

### Multi-AZ Layout

```
                AZ-1               AZ-2               AZ-3
                ──────────────     ──────────────     ──────────────
Redirect Svc:   ████ pods           ████ pods           ████ pods
                active              active              active
                ← active-active across all 3 AZs →

Write Svc:      ██ pods             ██ pods             ██ pods

Redis Cluster:  Shard1 Primary      Shard2 Primary      Shard3 Primary
                Shard4 Replica      Shard5 Replica      Shard6 Replica
                Shard5 Primary      Shard6 Primary      Shard4 Primary
                (each primary's replica is in a different AZ)

CockroachDB:    Node1, Node4,7      Node2, Node5,8      Node3, Node6,9
                (Raft ranges: 1 replica per AZ; quorum survives 1 AZ loss)

Kafka:          Brokers 1,4,7       Brokers 2,5,8       Brokers 3,6,9
                (partition leaders spread; followers cross-AZ)
```

### RPO / RTO Per Component

```
┌──────────────────┬─────────────────┬──────────┬─────────────────────────────┐
│  Component       │  RPO            │  RTO     │  Mechanism                  │
├──────────────────┼─────────────────┼──────────┼─────────────────────────────┤
│  CockroachDB     │  0 (sync Raft)  │  < 5s    │  Raft auto leader election   │
│  (single region) │                 │          │  (single AZ failure)         │
├──────────────────┼─────────────────┼──────────┼─────────────────────────────┤
│  CockroachDB     │  ~500ms         │  < 60s   │  GeoDNS + cross-region       │
│  (region fail)   │  (async repl)   │          │  follower reads              │
├──────────────────┼─────────────────┼──────────┼─────────────────────────────┤
│  Redis Cluster   │  < 1s           │  < 30s   │  Async repl; Sentinel/       │
│                  │  (cache ok)     │          │  Cluster auto-failover       │
├──────────────────┼─────────────────┼──────────┼─────────────────────────────┤
│  Kafka           │  0 (sync ISR=2) │  < 60s   │  ISR list; broker recovery   │
├──────────────────┼─────────────────┼──────────┼─────────────────────────────┤
│  ClickHouse      │  < 1hr (Kafka)  │  < 10min │  Replay from Kafka offset    │
├──────────────────┼─────────────────┼──────────┼─────────────────────────────┤
│  Entire Region   │  ~500ms         │  < 60s   │  GeoDNS health-check         │
└──────────────────┴─────────────────┴──────────┴─────────────────────────────┘
```

---

## 13. Multi-Region & Global Deployment

### Active-Active Architecture

```
                    GLOBAL DNS (GeoDNS / Cloudflare)
                                │
           ┌────────────────────┼────────────────────┐
           │                    │                    │
      US-EAST                EU-WEST               APAC
          │                      │                    │
      ┌───▼───┐              ┌───▼───┐           ┌───▼───┐
      │  CDN  │              │  CDN  │           │  CDN  │
      └───┬───┘              └───┬───┘           └───┬───┘
          │                      │                    │
      ┌───▼───┐              ┌───▼───┐           ┌───▼───┐
      │ Redis │              │ Redis │           │ Redis │  Regional caches
      └───┬───┘              └───┬───┘           └───┬───┘
          │                      │                    │
      ┌───▼───┐              ┌───▼───┐           ┌───▼───┐
      │  DB   │◄────────────►│  DB   │◄─────────►│  DB   │  CockroachDB
      │Cluster│  async repl  │Cluster│ async repl│Cluster│  ~500ms lag
      └───────┘              └───────┘           └───────┘

WRITE FLOW (EU user creates URL):
  EU user → EU Write Svc → EU DB (commit, ~5ms) → 201 returned
  Async: Kafka "url.created" → US + APAC cache warm (~500ms)

READ FLOW EDGE CASE (US user clicks EU link within 500ms of creation):
  US Redis MISS → US DB MISS (replication not yet complete)
  → Options:
    A) Return 404 (self-heals in <1s) — accepted
    B) Cross-region read fallback to EU DB (~100ms extra) — for lower error rate
  → We choose A for simplicity; document "new URLs visible globally within 1s"
```

### Consistency vs Latency Trade-off

```
STRONG CONSISTENCY                    EVENTUAL CONSISTENCY (CHOSEN)
──────────────────────────            ────────────────────────────────
Sync cross-region write ack           Async after home-region commit
Write latency: ~150ms (RTT)           Write latency: ~5ms local
Reads: always fresh                   Reads: stale by up to ~500ms
Good for: banking, inventory          Good for: URL shortener ✓

Trade-off accepted:
  • 150ms vs 5ms write latency is a massive UX difference
  • A new URL not globally visible for <1s is acceptable
  • The only downside: rare "404 for brand-new URL" that self-heals
```

---

## 14. Observability & Operations

### Golden Signals Dashboard

```
REDIRECT SERVICE — ONCALL DASHBOARD
─────────────────────────────────────────────────────────────────────
  LATENCY    p50:  ████░░░░  5ms   ← target
             p99:  ████████░ 45ms  ← watch closely
             p999: ██████████ 200ms ← ALERT if > 500ms

  TRAFFIC    Current:  850K req/sec
             CDN hit:  76%          ← ALERT if < 60%
             5xx rate: 0.01%        ← ALERT if > 0.1%

  SATURATION Redis CPU: 45%         ← OK
             Redis mem: 62%         ← WATCH if > 80%
             DB conns:  34%         ← OK

WRITE SERVICE — ONCALL DASHBOARD
─────────────────────────────────────────────────────────────────────
  p99 creation latency: 85ms        ← ALERT if > 500ms
  Key pool remaining:   12.4M       ← ALERT if < 1M
  Kafka consumer lag:   1,200 msgs  ← ALERT if > 100,000
```

### Alert Priority Matrix

```
┌──────────────────────────────────────────┬──────────┬────────┬────────────────┐
│  Alert                                   │ Severity │ Window │ Action         │
├──────────────────────────────────────────┼──────────┼────────┼────────────────┤
│  p99 redirect > 200ms                    │  P1-PAGE │  5min  │  Wake oncall   │
│  5xx error rate > 0.5%                   │  P1-PAGE │  5min  │  Wake oncall   │
│  DB replication lag > 30s               │  P1-PAGE │  2min  │  Wake oncall   │
│  Redis shard unreachable                │  P1-PAGE │  1min  │  Wake oncall   │
├──────────────────────────────────────────┼──────────┼────────┼────────────────┤
│  CDN cache hit rate < 60%               │  P2      │  15min │  Slack alert   │
│  Key pool remaining < 1M               │  P2      │  5min  │  Slack alert   │
│  Kafka consumer lag > 100K             │  P2      │  10min │  Slack alert   │
├──────────────────────────────────────────┼──────────┼────────┼────────────────┤
│  DLQ depth > 10,000                     │  P3      │  60min │  Ticket        │
│  DB slow queries > 10/min              │  P3      │  30min │  Ticket        │
└──────────────────────────────────────────┴──────────┴────────┴────────────────┘
Alert rule: minimum 5-min window + require 2 consecutive data points.
No single-point alerts (reduces false alarms by ~80%).
```

### SLO Error Budget

```
SLO: 99.99% availability = 52.6 min/year downtime budget
  At 1% burn rate:   52.6 min / year  (nominal)
  At 5x burn rate:   5 min/hour       → page oncall
  At 10x burn rate:  2.5 min/hour     → incident response

SLI: % of redirect requests returning valid response within 500ms
Measured: over 30-day rolling window
Tool: Prometheus + recording rules + Grafana dashboards
```

---

## 15. Security & Abuse Prevention

### Rate Limiting: Token Bucket Algorithm

```go
// Token bucket implemented with Redis Lua (atomic read-modify-write)
const rateLimitScript = `
  local key       = KEYS[1]
  local capacity  = tonumber(ARGV[1])  -- 10 (burst)
  local refill    = tonumber(ARGV[2])  -- 10/60 per second
  local now       = tonumber(ARGV[3])  -- current unix time (ms)

  local data   = redis.call('HMGET', key, 'tokens', 'last_refill')
  local tokens = tonumber(data[1]) or capacity
  local last   = tonumber(data[2]) or now

  local elapsed = (now - last) / 1000
  tokens = math.min(capacity, tokens + elapsed * refill)

  if tokens >= 1 then
    tokens = tokens - 1
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    redis.call('EXPIRE', key, 3600)
    return {1, tokens}  -- allowed, remaining tokens
  else
    return {0, 0}       -- denied
  end
`

// Limits:
//   Unauthenticated: 10 creates/hr per IP
//   Authenticated:   1000 creates/hr per user, 10/sec burst
//   Redirect:        1000/min per IP (enumeration prevention)
```

### Abuse Detection Signals

```
┌──────────────────────────────────────┬─────────────────┬──────────────────────┐
│  Signal                              │  Threshold      │  Action              │
├──────────────────────────────────────┼─────────────────┼──────────────────────┤
│  Creates/min from one IP             │  > 100          │  Temp block IP (1hr) │
│  Creates/min from one user           │  > 500          │  Suspend account     │
│  URLs → same destination domain      │  > 1000/day     │  Flag for review     │
│  Sequential short_code scanning      │  3+ sequential  │  Block IP + alert    │
│  Google Safe Browsing match          │  Any            │  Block (return 451)  │
│  Redirect loop detected at creation  │  tiny.url → X   │  Reject (400)        │
└──────────────────────────────────────┴─────────────────┴──────────────────────┘

Safe Browsing check:
  On creation: async POST to GSB API (don't block 201 on result)
  If flagged: UPDATE url_mappings SET is_blocked=TRUE → serve 451 on redirect
```

### Encryption

```
In transit:  TLS 1.3 everywhere. Internal services: Istio mTLS.
At rest:     AES-256 via cloud KMS. long_url field: field-level encryption
             (URL query params may contain OAuth tokens, emails, API keys).
Secrets:     HashiCorp Vault / AWS Secrets Manager. Rotated every 90 days.
             No hardcoded credentials anywhere. Audit log all secret access.
```

---

## 16. Data Privacy & Compliance

### GDPR Right-to-Erasure Cascade

```
User requests deletion of their account / specific URL
                │
                ▼
        Write Service
                │
  1. DB: UPDATE url_mappings
         SET is_deleted=TRUE, long_url=NULL   ← NULL erases PII
         WHERE short_code=$1
         -- keep short_code (prevent reuse) and created_at (audit)
         -- physical DELETE after 30 days via scheduled job
                │
  2. Kafka: publish "url.deleted" {short_code, user_id}
                │
  3. Return 204 to user
                │
  ─────────────────────────────────────────────────────
  Async Erasure Worker (consumes "url.deleted"):
                │
  4. Redis: DEL "url:{short_code}"                    immediate
  5. CDN:   Cloudflare Cache-Tag purge API            ~100ms propagation
  6. L2:    Redis pubsub "url.invalidate" → pods evict ~1ms
  7. ClickHouse: DELETE FROM url_clicks
                 WHERE short_code=$1                  < 30 days (async job)
                 (no long_url stored there, only short_code + metadata)
  8. Backups: PII remains until backup rotation (document in privacy policy;
              backups used only for disaster recovery, not analytics)
  ─────────────────────────────────────────────────────
  Verification: erasure_audit table records per-layer completion timestamps
  SLA: full erasure verified within 30 days (GDPR requirement)
```

### PII Fields

```
┌────────────────┬──────────────┬───────────────────────────────────────────┐
│  Field         │  PII?        │  Handling                                 │
├────────────────┼──────────────┼───────────────────────────────────────────┤
│  long_url      │  YES         │  Field-level AES-256 encryption at rest   │
│                │              │  (may contain tokens, emails in params)   │
│  user_id       │  YES         │  BIGINT, foreign key to users table       │
│  ip address    │  YES         │  NOT stored in url_clicks; only           │
│                │              │  ip_country (anonymized via MaxMind)      │
│  user_agent    │  Indirect    │  Stored as hash (UInt64), not raw string  │
│  short_code    │  No          │  Random, no PII                           │
│  click count   │  No          │  Aggregate, no user linkage               │
└────────────────┴──────────────┴───────────────────────────────────────────┘
```

---

## 18. Deployment & Rollout

### Zero-Downtime Schema Migration: Expand/Contract

```sql
-- SCENARIO: Adding ip_country column to url_mappings
-- NEVER: ALTER TABLE url_mappings ADD COLUMN ip_country VARCHAR(2) NOT NULL;
--        ← locks 100TB table for hours. Production outage guaranteed.

-- ═══════════════════════════════════════
-- WEEK 1: EXPAND (backward-compatible add)
-- ═══════════════════════════════════════
ALTER TABLE url_mappings
  ADD COLUMN ip_country VARCHAR(2) NULL;   -- nullable: old code ignores it
                                           -- new code writes it on CREATE
-- Deploy new code: writes ip_country on new records.
-- Old pods running simultaneously: fine (column optional).

-- ═══════════════════════════════════════
-- WEEK 2: BACKFILL (small batches, no locks)
-- ═══════════════════════════════════════
-- Run as background job, repeat until all rows filled:
UPDATE url_mappings
SET    ip_country = lookup_country(created_from_ip)
WHERE  ip_country IS NULL
LIMIT  10000;   -- small batches prevent lock contention and replication lag

-- ═══════════════════════════════════════
-- WEEK 3: CONTRACT (after backfill complete, old code fully retired)
-- ═══════════════════════════════════════
-- Verify: SELECT COUNT(*) FROM url_mappings WHERE ip_country IS NULL; → 0
ALTER TABLE url_mappings
  ALTER COLUMN ip_country SET NOT NULL;   -- safe now, all rows populated

-- Remove fallback code path from application. Clean up.
```

### Canary Deploy for Redirect Service

```
REDIRECT SERVICE CANARY ROLLOUT (99.99% SLA — never rush)
──────────────────────────────────────────────────────────────────────
  Step 1: Deploy new version to 1 pod (of 20) = 5% traffic
          Monitor 30 minutes.
          Rollback trigger: p99 > 100ms OR 5xx > 0.5%

  Step 2: Promote to 4 pods = 20% traffic
          Monitor 30 minutes.
          Rollback trigger: same thresholds

  Step 3: Promote to 10 pods = 50% traffic
          Monitor 30 minutes.

  Step 4: Promote to 20 pods = 100% traffic

  AUTO-ROLLBACK IMPLEMENTATION:
    Argo Rollouts (Kubernetes) + Prometheus metrics integration
    analysisTemplate:
      metrics:
        - name: p99-latency
          successCondition: result < 100
          failureLimit: 3
          provider:
            prometheus:
              query: |
                histogram_quantile(0.99,
                  sum(rate(redirect_duration_seconds_bucket{version="canary"}[5m]))
                  by (le))
        - name: error-rate
          successCondition: result < 0.005
          failureLimit: 3

  Schema changes deploy BEFORE code changes.
  Both old + new code must work with the schema during overlap period.
  Minimum 24hr gap between schema deploy and code deploy.
```

---

## 19. Trade-offs and Alternatives

```
┌──────────────────────────────┬────────────────────┬───────────────────────────────────────────┐
│  Decision                    │  Chosen            │  Rejected + Why                           │
├──────────────────────────────┼────────────────────┼───────────────────────────────────────────┤
│  Short code generation       │  Key pool          │  Hash: ~13% collision rate at 1M URLs     │
│                              │                    │  Counter: global Redis SPOF bottleneck    │
│                              │                    │  UUID truncated: still needs collision    │
│                              │                    │  handling logic                           │
├──────────────────────────────┼────────────────────┼───────────────────────────────────────────┤
│  Redirect type               │  302 default       │  301 default: analytics blind to repeat   │
│                              │  (301 opt-in)      │  visitors; deletion unenforceable         │
├──────────────────────────────┼────────────────────┼───────────────────────────────────────────┤
│  Cache                       │  Redis Cluster     │  Memcached: no TTL per-key in cluster,    │
│                              │                    │  no persistence, simpler but insufficient  │
├──────────────────────────────┼────────────────────┼───────────────────────────────────────────┤
│  Analytics writes            │  Async Kafka       │  Sync DB write: blocks redirect path at   │
│                              │                    │  116K/sec; DB can't sustain it            │
├──────────────────────────────┼────────────────────┼───────────────────────────────────────────┤
│  Analytics store             │  ClickHouse        │  PostgreSQL: row-oriented; GROUP BY at    │
│                              │                    │  116K events/sec is prohibitive           │
├──────────────────────────────┼────────────────────┼───────────────────────────────────────────┤
│  Consistency                 │  Eventual          │  Strong: 150ms write latency vs 5ms; URL  │
│                              │                    │  shortener doesn't need strong consistency│
├──────────────────────────────┼────────────────────┼───────────────────────────────────────────┤
│  Primary DB                  │  CockroachDB       │  DynamoDB: valid alternative; less SQL    │
│                              │                    │  power, AWS vendor lock-in                │
│                              │                    │  Cassandra: eventual consistency makes    │
│                              │                    │  uniqueness enforcement harder            │
├──────────────────────────────┼────────────────────┼───────────────────────────────────────────┤
│  Shard key                   │  short_code hash   │  user_id hash: user listing = 1 shard,    │
│                              │                    │  but redirect = 2-hop lookup on hot path  │
└──────────────────────────────┴────────────────────┴───────────────────────────────────────────┘

IF READ:WRITE RATIO DROPPED TO 10:1:
  → Cache less valuable (low URL reuse)
  → Invest in DB horizontal scale, compression, tiered storage
  → CDN hit rate drops; origin fleet must scale proportionally

IF LATENCY TIGHTENED TO p99 < 5ms:
  → Move redirect logic to CDN edge (Cloudflare Workers)
  → Store hot URLs in Cloudflare KV (edge key-value store)
  → Eliminate origin hop entirely for cached entries
  → DB lookups: must use in-memory (Redis as primary store, not cache)
  → Multi-region active-active mandatory from day 1
```

---

## 20. Follow-ups and Advanced Topics

### 5 Hard Interviewer Questions

**Q1: "A URL goes viral — 0 to 500K req/sec in 30 seconds. Walk me through each layer."**

Strong answer: T+0s: first hit per CDN PoP (~200 global PoPs × 1 miss = 200 origin requests, negligible). T+1s: all CDN PoPs cached → 75% served at edge. L2 in-process warms per pod. Redis handles remainder (single key, 100K GET/sec manageable). T+5s: DB sees 0 queries for this URL. Real risk: CDN miss storm lasts only ~1s. Mitigation: detect click velocity > 10K/min via Kafka analytics → pre-warm CDN via purge API before viral peak. Redis hot-key mitigation: if shard CPU > 70%, replicate key across all 6 shards.

**Q2: "How does your system handle per-click URL expiry (expire after 1000 clicks)?"**

Strong answer: Naive approach — increment counter in DB on every redirect → 1M writes/sec on one row → hot row bottleneck. Better: store `max_clicks` + atomic counter in Redis. `INCR "clicks:{code}"` → if result > max_clicks → return 410. Periodically flush Redis counter to DB (every 1000 increments or every 60s). Race condition: a few clicks beyond the limit may slip through during flush window. If exact enforcement required: Redis INCR + Lua script (atomic check + increment), but this serializes all clicks through Redis for that key — accepts the hot-key risk for accuracy.

**Q3: "Your Redis cluster has a partial failure — 2 of 6 shards unreachable. Walk through exactly what happens."**

Strong answer: Redis Cluster: 16384 hash slots / 6 shards = ~2730 slots per shard. 2 shards down = ~5460 slots affected = ~33% of short codes miss cache. DB replicas see ~38K extra req/sec (33% × 116K avg). Circuit breaker does NOT trip (DB is healthy). L2 in-process cache absorbs repeat requests within 60s window. Redis Sentinel/Cluster automatically promotes replicas in <30s. Net user impact: ~30-60ms p99 spike for ~30 seconds, then self-heals. No data loss. No redirect failures.

**Q4: "How do you implement GDPR erasure across CDN, Redis, backups, and ClickHouse?"**

Strong answer: DB: soft delete (NULL long_url), physical delete in 30 days. Redis: DEL via Kafka consumer (immediate). CDN: Cloudflare Cache-Tag purge API (~500ms propagation). L2 in-process: Redis pubsub "url.invalidate" → pod eviction in <1ms. ClickHouse: async DELETE job within 30 days (no long_url stored; only short_code + hashed metadata). Backups: PII remains until rotation — document in privacy policy, backups are DR-only. Audit table records per-layer completion timestamps. 30-day SLA enforced with scheduled verification job.

**Q5: "How do you prevent someone from enumerating all short URLs?"**

Strong answer: 7-char Base62 = 3.52T possible values. At 1000 redirects/min per IP rate limit: 3.52T / 1000 × 1 IP-minute = 3.52B IP-minutes to exhaust. With 1M IPs: 3.52B / 1M = 3520 minutes ≈ 2.4 days. Impractical at scale. For higher security: extend to 12 chars (62^12 ≈ 3.2 × 10^21). Add anomaly detection: 3+ sequential short_code scans from same IP → block. For sensitive links: optional password protection layer (POST /{code} with password body). Monitor access pattern entropy via Kafka analytics.

### L4 vs L6 Answer Comparison

```
┌──────────────────┬─────────────────────────────┬───────────────────────────────────────┐
│  Dimension       │  L4 Answer                  │  L6 Answer                            │
├──────────────────┼─────────────────────────────┼───────────────────────────────────────┤
│  ID generation   │  "Use auto-increment"        │  Key pool: birthday paradox math,     │
│                  │                             │  SKIP LOCKED SQL, pre-fetch opt.      │
├──────────────────┼─────────────────────────────┼───────────────────────────────────────┤
│  301 vs 302      │  Picks one, no reason        │  Cache-Control headers spelled out,   │
│                  │                             │  analytics/mutability trade-off       │
├──────────────────┼─────────────────────────────┼───────────────────────────────────────┤
│  Caching         │  "Add Redis"                 │  4 layers, PER algorithm, stampede    │
│                  │                             │  prevention, delete race condition    │
├──────────────────┼─────────────────────────────┼───────────────────────────────────────┤
│  Scaling         │  "Add more servers"          │  Specific bottleneck at each 10x step │
│                  │                             │  with ordered fix sequence            │
├──────────────────┼─────────────────────────────┼───────────────────────────────────────┤
│  Analytics       │  "Increment click count in DB│  Async Kafka (acks=0), ClickHouse,   │
│                  │                             │  never block redirect path            │
├──────────────────┼─────────────────────────────┼───────────────────────────────────────┤
│  Failures        │  "We have retries"           │  Circuit breaker thresholds, Raft     │
│                  │                             │  quorum math, cascade prevention      │
├──────────────────┼─────────────────────────────┼───────────────────────────────────────┤
│  GDPR            │  Not mentioned               │  Full cascade: DB → Redis → CDN →     │
│                  │                             │  ClickHouse → backups, per-layer SLA  │
├──────────────────┼─────────────────────────────┼───────────────────────────────────────┤
│  Numbers         │  Vague estimates             │  Every choice anchored to specific    │
│                  │                             │  QPS/bytes numbers derived upfront    │
└──────────────────┴─────────────────────────────┴───────────────────────────────────────┘
```

---

## 21. Interview Pacing Guide

```
╔════════════════════════════════════════════════════════════════════════════╗
║                    45-MINUTE INTERVIEW MAP                                ║
╠══════════════════╦═════════════════════════════════════════════════════════╣
║  0–5 min         ║  REQUIREMENTS                                          ║
║                  ║  □ Confirm: shorten URL, redirect, analytics           ║
║                  ║  □ Ask: 301 vs 302 preference?                         ║
║                  ║  □ Ask: custom aliases? TTLs? user accounts?           ║
║                  ║  □ State scale assumptions; confirm with interviewer   ║
╠══════════════════╬═════════════════════════════════════════════════════════╣
║  5–10 min        ║  ESTIMATION + HARD PROBLEMS                            ║
║                  ║  □ Derive: 1.16M peak redirect/sec                     ║
║                  ║  □ Derive: 500 MB hot set → fits in Redis              ║
║                  ║  □ NAME the two hard problems (ID gen + latency)        ║
║                  ║  □ State: "CDN + Redis are non-negotiable"              ║
╠══════════════════╬═════════════════════════════════════════════════════════╣
║  10–20 min       ║  HIGH-LEVEL ARCHITECTURE + API                         ║
║                  ║  □ Draw: CDN → LB → Redirect Svc → Redis → DB          ║
║                  ║  □ Separate Redirect Svc from Write Svc (bulkhead)      ║
║                  ║  □ Mention: Kafka for async analytics (never blocks)    ║
║                  ║  □ Define: POST /api/v1/urls + GET /{code}              ║
║                  ║  □ Explain: 301 vs 302 + Cache-Control headers ← depth ║
╠══════════════════╬═════════════════════════════════════════════════════════╣
║  20–35 min       ║  DEEP DIVE (spend time here — this is where L6 shines) ║
║                  ║  □ Key pool: birthday paradox math (why hash fails),    ║
║                  ║    SKIP LOCKED SQL, pre-fetch, monitoring thresholds   ║
║                  ║  □ Cache: all 4 layers with exact hit rates + TTLs,     ║
║                  ║    PER stampede prevention, delete race condition       ║
║                  ║  □ Idempotency: Redis SET NX + DB log + crash recovery  ║
║                  ║  □ Kafka: acks=0 for analytics (why), acks=all delete   ║
╠══════════════════╬═════════════════════════════════════════════════════════╣
║  35–45 min       ║  FAILURES + TRADE-OFFS + FOLLOW-UPS                    ║
║                  ║  □ Redis partial failure → Raft quorum math             ║
║                  ║  □ DB failure → reads OK (replicas), writes 503 + retry ║
║                  ║  □ Viral URL → CDN absorbs, explain all 4 cache layers  ║
║                  ║  □ State key trade-off: "302 for analytics + CDN cache" ║
║                  ║  □ GDPR erasure cascade (signals compliance awareness)  ║
║                  ║  □ Invite follow-up: "I can go deeper on X"             ║
╠══════════════════╬═════════════════════════════════════════════════════════╣
║  IF SHORT        ║  DEFER (mention but don't detail):                     ║
║  ON TIME         ║  • Multi-region consistency trade-off                  ║
║                  ║  • Full GDPR cascade                                   ║
║                  ║  • Canary deploy thresholds                            ║
║                  ║  • Observability / alerting specifics                  ║
╚══════════════════╩═════════════════════════════════════════════════════════╝

THE 5 SIGNALS THAT SEPARATE L6 FROM L4 ON THIS PROBLEM:
────────────────────────────────────────────────────────────────────────────
  1. Frame the hard problems BEFORE drawing (ID gen + read skew)
  2. 301 vs 302 with Cache-Control headers spelled out
  3. Key pool: birthday paradox math + SKIP LOCKED + pre-fetch opt.
  4. 4-layer cache with PER stampede algorithm + delete race condition
  5. Numbers drive every decision (500MB hot set, 5800 DB req/sec, etc.)
```
