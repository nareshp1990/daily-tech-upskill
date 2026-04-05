# Day 28 — CDN, Edge Caching & Global Distribution
**Date:** 2026-04-04
**Phase:** 2 — System Design
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**CDN, Edge Caching & Global Distribution — Serving Content at the Speed of Light**

Network latency has a physical lower bound: light travels at ~200,000 km/s through fibre. A round trip from Tokyo to London is at minimum ~150ms — before any processing. A CDN eliminates this by caching content at edge nodes close to users. Today you learn what CDNs cache, how cache invalidation works at global scale, and how to design content delivery for a backend engineer's perspective.

---

## 2. Core Concepts

### 2.1 The Latency Problem

```
Without CDN:
  User in Tokyo → request → Origin server in US → response
  Distance: ~10,000 km
  Min latency: ~100ms (round trip, speed of light)
  + Processing time + TLS handshake = 200-500ms

With CDN:
  User in Tokyo → request → Edge node in Tokyo → response
  Distance: ~50 km
  Latency: ~5ms ✅
  Origin server only hit on cache miss (~5-10% of requests)
```

### 2.2 CDN Architecture

```
                        ┌─────────────────┐
                        │  Origin Server  │
                        │  (your backend) │
                        └────────┬────────┘
                                 │ Cache miss: fetch and cache
                    ┌────────────┼────────────┐
                    ▼            ▼            ▼
            ┌──────────┐ ┌──────────┐ ┌──────────┐
            │ Edge PoP │ │ Edge PoP │ │ Edge PoP │
            │ New York │ │  London  │ │  Tokyo   │
            └─────┬────┘ └─────┬────┘ └─────┬────┘
                  │             │             │
              US users      EU users     APAC users
               ~5ms           ~5ms          ~5ms
```

### 2.3 What to Cache on a CDN

| Content Type | Cache? | TTL | Reasoning |
|-------------|--------|-----|-----------|
| Static assets (JS/CSS/images) | ✅ | 1 year | Content-hash URL = immutable |
| API responses (public, read-only) | ✅ | 1-60 min | User profile, product catalog |
| Personalised responses | ⚠️ | 0 (no-cache) | Per-user data must bypass CDN |
| Auth-required responses | ❌ | Never | Never cache authenticated content |
| POST/PUT/DELETE responses | ❌ | Never | Non-idempotent requests |

---

## 3. Cache Control Headers — The Language CDNs Speak

```http
# Immutable static asset (JS with content hash in filename)
Cache-Control: public, max-age=31536000, immutable

# Public API response — cache at CDN for 5 minutes
Cache-Control: public, max-age=300, s-maxage=600
# max-age=300 → browser TTL, s-maxage=600 → CDN TTL

# Private (user-specific) — CDN must not cache
Cache-Control: private, max-age=0, no-store

# Revalidate on every request (use ETag for efficient 304)
Cache-Control: no-cache
ETag: "abc123"
```

### 3.1 Spring Boot Cache Headers

```java
@RestController
@RequestMapping("/v1")
public class ProductController {

    @GetMapping("/products/{id}")
    public ResponseEntity<Product> getProduct(@PathVariable Long id) {
        Product product = productService.findById(id);

        return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(5, TimeUnit.MINUTES)
                .cachePublic()
                .sMaxAge(10, TimeUnit.MINUTES))
            .eTag(String.valueOf(product.getVersion()))   // ETag for conditional GET
            .body(product);
    }

    @GetMapping("/users/{id}/profile")
    public ResponseEntity<UserProfile> getUserProfile(@PathVariable Long id) {
        UserProfile profile = userService.findProfile(id);

        return ResponseEntity.ok()
            .cacheControl(CacheControl.noStore())        // Never cache user data
            .body(profile);
    }
}
```

### 3.2 Conditional Requests (ETags and Last-Modified)

```
Browser: GET /products/42
Server:  200 OK + ETag: "v5" + body (2KB)

Browser: GET /products/42 + If-None-Match: "v5"
Server (not modified): 304 Not Modified + no body (saves bandwidth)
Server (modified):     200 OK + ETag: "v6" + new body
```

---

## 4. Cache Invalidation at the CDN Level

### 4.1 The Three Cache Invalidation Strategies

#### Purge by URL
```bash
# CloudFlare API
curl -X DELETE \
  "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
  -H "Authorization: Bearer {token}" \
  -d '{"files":["https://example.com/v1/products/42"]}'
```

```java
// Trigger CDN purge after product update
@Service
public class ProductService {

    @Transactional
    public Product updateProduct(Long id, UpdateProductRequest req) {
        Product updated = productRepository.save(...);
        cdnPurgeService.purge("/v1/products/" + id);    // Async purge
        return updated;
    }
}
```

#### Cache Versioning (Immutable URLs)
```
Old: /api/products/42          → TTL 5 min → stale risk
New: /api/v1/products/42?v=1619 → TTL 1 year → version in URL, never stale
     or /api/products/42 + version in path param controlled by server
```

#### Surrogate Keys / Cache Tags
```
Store product:42 with tag "product-42" + "category-electronics"

Purge all electronics at once:
  DELETE tag: "category-electronics"
  → Invalidates all product pages in that category
  → CloudFlare, Fastly, and Varnish support this pattern
```

---

## 5. Edge Computing — Logic at the Edge

Modern CDNs (Cloudflare Workers, AWS Lambda@Edge, Fastly Compute) allow running JavaScript/Wasm at edge nodes:

```javascript
// Cloudflare Worker — runs at edge, 0ms cold start
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  const url = new URL(request.url)

  // A/B test at edge (no origin hit needed)
  const userId = request.headers.get('X-User-Id')
  const variant = parseInt(userId) % 2 === 0 ? 'A' : 'B'

  // Auth check at edge (reject before hitting origin)
  const token = request.headers.get('Authorization')
  if (!token) return new Response('Unauthorized', { status: 401 })

  // Add headers before forwarding to origin
  const modifiedRequest = new Request(request, {
    headers: { ...Object.fromEntries(request.headers), 'X-Variant': variant }
  })

  return fetch(modifiedRequest)
}
```

**Use cases for edge logic:**
- Auth token validation (reject before hitting origin)
- A/B testing routing
- Geolocation-based redirects
- Rate limiting at the edge
- Request transformation/normalisation

---

## 6. Global Distribution Patterns

### 6.1 Active-Active Multi-Region

```
Region: US-East         Region: EU-West         Region: APAC
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│ App + DB     │◄─────►│ App + DB     │◄─────►│ App + DB     │
│ (primary for │       │ (primary for │       │ (primary for │
│  US users)   │       │  EU users)   │       │ APAC users)  │
└──────────────┘       └──────────────┘       └──────────────┘

DNS routes users to nearest region (GeoDNS / Route53 Latency Routing)
Cross-region sync for shared data (eventual consistency acceptable)
```

### 6.2 Active-Passive Failover

```
Primary: US-East (all traffic)
Passive: EU-West (warm standby)

Failover trigger: health check fails on primary
DNS TTL: 60s (low TTL for fast failover)
RPO: seconds (replication lag)
RTO: 60-120 seconds (DNS propagation)
```

---

## 7. Hands-On Exercise

1. Set up a Spring Boot endpoint returning `Cache-Control: public, max-age=60`.
2. Run Varnish locally (`docker run -p 8080:80 varnish`) and point it at your service.
3. Hit the endpoint twice — verify the second is served from Varnish cache (X-Cache: HIT header).
4. Add ETag support using entity version — verify 304 Not Modified on second identical request.
5. Implement a CDN purge notification: when a `@TransactionalEventListener` fires on entity update, log the purge URL (simulate the CDN purge call).

---

## 8. Key Takeaways

1. **CDNs reduce latency by serving from nearby edge nodes** — physical proximity beats fast code.
2. **Cache-Control headers are the contract between your server, CDNs, and browsers** — get them wrong and you serve stale data or miss caching entirely.
3. **Never cache authenticated/personalised responses on the CDN** — a misconfigured cache returning one user's data to another is a serious security incident.
4. **Cache invalidation is hard** — prefer content-hash URLs for static assets (never expire) and surrogate keys for dynamic content.
5. **Edge computing moves logic closer to users** — auth, routing, and A/B logic at the edge reduces origin load.

---

## 9. Connection to Previous Days

Day 18 (Caching Strategies) covered Redis application-level caching. Today you added CDN-level (HTTP) caching — the layer in front of your application. These two work together: CDN caches at the edge, Redis caches inside the service. Database replication (Day 27) reduces origin database load; CDN reduces origin server load.

---

## 10. Resources

- **[MDN HTTP Caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)** — Authoritative reference for Cache-Control.
- **[Cloudflare Cache documentation](https://developers.cloudflare.com/cache/)** — Production CDN caching guide.
- **[High Performance Browser Networking by Ilya Grigorik](https://hpbn.co/)** — Network fundamentals every backend engineer should know.

---

*Day 28. The fastest request is the one that never reaches your server — serve it from the edge.*
