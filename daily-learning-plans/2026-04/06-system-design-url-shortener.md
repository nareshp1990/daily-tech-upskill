# Day 30 — System Design Case Study: URL Shortener
**Date:** 2026-04-06
**Phase:** 2 — System Design
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**System Design Case Study: URL Shortener — A Classic Interview Problem with Real Depth**

URL shorteners (bit.ly, tinyurl.com) are deceptively simple: turn `https://www.example.com/very/long/url` into `https://bit.ly/abc123`. But at scale — billions of URLs, trillions of redirects — every design decision matters. This case study exercises everything from Phase 2: hashing, caching, database design, rate limiting, and observability.

---

## 2. Requirements Clarification

Always start here in interviews:

**Functional Requirements:**
- Create a short URL from a long URL
- Redirect short URL to the original long URL
- Optional: Custom alias (e.g., `bit.ly/my-brand`)
- Optional: URL expiry (TTL)
- Optional: Analytics (click counts, geo, device)

**Non-Functional Requirements:**
- Scale: 100M URLs created/day = ~1,200 writes/second
- Reads (redirects): 10:1 read:write ratio = 12,000 reads/second
- Availability: 99.99% (redirects are critical path)
- Latency: redirect p99 < 10ms
- Storage: 100M URLs/day × 365 days × 5 years = 182 billion URLs
  - Each URL ~500 bytes → ~91 TB storage over 5 years

---

## 3. Core Design

### 3.1 Key Generation — How to Create Short Codes

#### Option 1: MD5 Hash (Bad)
```
shortCode = MD5(longUrl).substring(0, 7)
Problem: collisions (two different URLs → same short code)
Problem: not truly unique without collision detection loop
```

#### Option 2: Base62 Encoding of Auto-Increment ID (Recommended)
```
characters: [0-9][a-z][A-Z] = 62 chars
7 characters: 62^7 = 3.5 trillion unique codes (enough for 5 years)

Algorithm:
  id = auto_increment_id (from DB sequence or distributed ID generator)
  code = toBase62(id)

id=1       → "0000001"
id=12345   → "0003lX"
id=3.5T    → "zzzzzzz"

Benefits:
  + No collisions (unique IDs)
  + No DB lookup needed to check collision
  + Predictable length
  - Exposes sequential IDs (add shuffling or random offset to obscure)
```

```java
public class Base62Encoder {
    private static final String CHARSET = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";

    public static String encode(long id) {
        if (id == 0) return String.valueOf(CHARSET.charAt(0));
        StringBuilder sb = new StringBuilder();
        while (id > 0) {
            sb.append(CHARSET.charAt((int)(id % 62)));
            id /= 62;
        }
        return sb.reverse().toString();
    }

    public static long decode(String code) {
        long id = 0;
        for (char c : code.toCharArray()) {
            id = id * 62 + CHARSET.indexOf(c);
        }
        return id;
    }
}
```

#### Option 3: Distributed Key Generation (for multi-region)
```
Use Snowflake ID (Day 27) per region → encode to Base62
Avoids distributed counter coordination
```

### 3.2 Database Schema

```sql
CREATE TABLE urls (
    id          BIGINT         NOT NULL AUTO_INCREMENT,
    short_code  VARCHAR(10)    NOT NULL,
    long_url    TEXT           NOT NULL,
    user_id     BIGINT,
    created_at  TIMESTAMP      DEFAULT CURRENT_TIMESTAMP,
    expires_at  TIMESTAMP,
    click_count BIGINT         DEFAULT 0,
    PRIMARY KEY (id),
    UNIQUE KEY idx_short_code (short_code),
    KEY idx_user_id (user_id),
    KEY idx_expires_at (expires_at)   -- For TTL cleanup job
);
```

---

## 4. Architecture

```
               User
                │
    ┌───────────▼──────────────┐
    │      CDN (CloudFlare)    │  ← Cache redirects at edge
    │  302 responses cached    │    99% cache hit rate
    └───────────┬──────────────┘
                │ cache miss (~1%)
    ┌───────────▼──────────────┐
    │      API Gateway         │  ← Rate limiting: 10 creates/min per user
    │      (Kong)              │    Auth for create, anonymous for redirect
    └───┬───────────────────┬──┘
        │                   │
   POST /shorten       GET /:shortCode
        │                   │
   ┌────▼────┐          ┌────▼────┐
   │ Write   │          │  Read   │  ← Separate read/write services
   │ Service │          │ Service │    Redirect SLA: <10ms
   └────┬────┘          └────┬────┘
        │                   │
        │               ┌───▼───┐
        │               │ Redis │  ← Cache: shortCode → longUrl
        │               │ Cache │    TTL: 24h, hit rate >99%
        │               └───┬───┘
        │               cache miss
    ┌───▼───────────────────▼───┐
    │        MySQL              │
    │  Primary  │  Replicas     │  ← Reads → replicas, Writes → primary
    └───────────────────────────┘
```

### 4.1 Write Flow
```
1. POST /shorten { longUrl, customAlias?, ttl? }
2. Validate URL (max length, allowed domains)
3. Check if longUrl already shortened (dedup) → return existing code
4. Generate ID from DB sequence (or Snowflake)
5. Encode ID → Base62 short code
6. INSERT into MySQL primary
7. Write to Redis cache
8. Return { shortUrl: "https://bit.ly/abc123" }
```

### 4.2 Read Flow (the hot path)
```
1. GET /abc123
2. Check Redis → HIT → 301/302 redirect (< 1ms)
3. MISS → Query MySQL replica
4. Write to Redis, return 301/302
5. Increment click_count asynchronously (Kafka event → analytics service)
```

### 4.3 301 vs 302 Redirect

| Status | Cache Behaviour | Analytics | Use Case |
|--------|----------------|-----------|---------|
| 301 Moved Permanently | Browser caches forever | Miss subsequent clicks | Reduce server load |
| 302 Found | Browser re-requests each time | Every click counted | Analytics tracking |

**Best practice:** Use 302 for analytics; optionally upgrade to 301 after 30 days for static URLs.

---

## 5. Scaling Decisions

### 5.1 Caching Strategy
```
Redis setup: consistent hashing ring (Day 23) across 5 Redis nodes
  Key: "url:{shortCode}" → longUrl
  TTL: 24h (refresh on each hit)
  Expected hit rate: 99%+ (80/20 rule: 20% of URLs = 80% of traffic)

Cache size calculation:
  Popular URLs: ~20M (top 20% of 100M created/day)
  Per URL: 10 bytes (shortCode) + 2KB (longUrl) = ~2KB
  Cache size: 20M × 2KB = 40GB → fits in Redis
```

### 5.2 Database Scaling
```
Writes: 1,200/s → single MySQL primary handles this easily
Reads:  12,000/s → 3-5 MySQL replicas + Redis cache absorbs this

Future: Shard by shortCode hash if needed (unlikely for this scale)
```

### 5.3 Analytics (Async)
```
Click event → Kafka topic → Analytics Consumer → ClickHouse (OLAP)

Never do analytics on the hot redirect path — it adds latency!
Count increments: eventually consistent, updated async every minute
```

---

## 6. Edge Cases & Gotchas

1. **Custom alias conflicts** — check availability before creating
2. **URL deduplication** — same long URL, multiple short codes? Allow or detect?
3. **Malicious URLs** — validate against Google Safe Browsing API before shortening
4. **Expired URLs** — return 410 Gone (not 404) with message
5. **URL encoding** — handle special characters in long URLs
6. **Hot URLs** — if a URL goes viral (1M requests/second), Redis single node becomes bottleneck → use local in-memory cache (Caffeine) in front of Redis

---

## 7. Self-Assessment Questions

1. How would you handle 100x traffic spike on a specific short URL?
2. Where would you add rate limiting, and at what limits?
3. How do you prevent someone from enumerating all shortened URLs?
4. Design the analytics service — how do you handle 1B daily click events?

---

## 8. Key Takeaways

1. **Read:write ratio drives architecture** — 10:1 means aggressively cache reads.
2. **Separate read and write paths** — different scaling requirements, different SLAs.
3. **Never block the hot path with sync operations** — analytics, notifications → async via Kafka.
4. **CDN is the first cache** — redirect responses are ideal CDN candidates.
5. **Base62 encoding of sequential IDs is the simplest correct solution** — no collisions, no DB loops.

---

## 9. Resources

- **[Designing Data-Intensive Applications, Ch. 5-6](https://dataintensive.net/)** — Replication and partitioning patterns.
- **[bit.ly engineering blog](https://word.bitly.com/)** — Real production insights.
- **[System Design Primer — URL Shortener](https://github.com/donnemartin/system-design-primer/blob/master/solutions/system_design/pastebin/README.md)** — Reference solution.

---

*Day 30. The humble URL shortener hides a distributed system lesson in every redirect.*
