# Day 31 — System Design Case Study: Social Media Feed
**Date:** 2026-04-07
**Phase:** 2 — System Design
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**System Design Case Study: Social Media Feed — The Fan-Out Problem at Scale**

Building a timeline for 500 million users (think Twitter/Instagram) is one of the canonical hard system design problems. The core challenge: when Alice posts a tweet, how does it appear in the timeline of her 10 million followers? This is the "fan-out" problem — and the right answer depends entirely on whether Alice is a regular user or a celebrity. Today you learn both approaches and when to use each.

---

## 2. Requirements

**Functional:**
- Users can create posts (text, images, videos)
- Users see a personalised feed (posts from people they follow)
- Feed is ranked by recency (or engagement in advanced version)
- Like, comment, repost
- Notification when followed user posts

**Non-Functional:**
- 500M daily active users
- 200M posts/day = ~2,300 writes/second
- Feed reads: 50 × per user × 500M = 25B reads/day = 289,000 reads/second
- Post creation latency: < 500ms
- Feed read latency: p99 < 200ms
- Eventual consistency acceptable (it's OK to see a post 1-2 seconds after it's created)

---

## 3. The Fan-Out Problem

```
Alice has 10M followers. She posts a tweet.
How do her followers see it in their feed?

Option A: Fan-out on WRITE (push model)
  When Alice posts → immediately write to all 10M followers' feed caches
  Pros: Feed reads are O(1) — just read from pre-computed cache
  Cons: 10M writes per Alice post → write amplification
        If Alice posts 10x/day → 100M cache writes/day just for Alice

Option B: Fan-out on READ (pull model)
  Store only Alice's post in her post list
  When follower opens feed → query all followed users' post lists, merge, sort
  Pros: One write per post, no write amplification
  Cons: Feed read is O(N) where N = number of people you follow
        With 1000 followees: 1000 DB queries per feed load → slow

Solution: HYBRID (what Twitter/Instagram actually does)
  Regular users (<= 10K followers): fan-out on WRITE
  Celebrities (> 10K followers): fan-out on READ
```

---

## 4. Architecture

```
                     User creates post
                           │
                    ┌──────▼──────┐
                    │  Post API   │  Validates, stores media
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  Kafka      │  post.created topic
                    └──────┬──────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
   ┌─────────────┐  ┌───────────┐  ┌──────────────┐
   │  Feed       │  │Notif      │  │  Analytics   │
   │  Fanout     │  │Service    │  │  Service     │
   │  Service    │  └───────────┘  └──────────────┘
   └──────┬──────┘
          │
          │ Check: regular user or celebrity?
          │
     ┌────┴────────────────────┐
     │ Regular user             │ Celebrity
     │ (fan-out on write)       │ (skip fan-out)
     ▼                         │
Write post_id to each          │ Posts stored only in
follower's feed cache          │ celebrity's own post list
(Redis sorted set)             │
                               │
                    Feed Read Request
                           │
                    ┌──────▼──────┐
                    │  Feed API   │
                    └──────┬──────┘
                           │
          ┌────────────────┤
          ▼                ▼
   Regular user's     Merge regular feed
   pre-computed       + celebrity posts
   feed from Redis    (fan-out on read
                       for celeb posts)
```

---

## 5. Data Storage Design

### 5.1 Posts Table (MySQL, sharded by user_id)

```sql
CREATE TABLE posts (
    id          BIGINT PRIMARY KEY,    -- Snowflake ID (time-sortable)
    user_id     BIGINT NOT NULL,
    content     TEXT,
    media_url   VARCHAR(512),
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    like_count  INT DEFAULT 0,
    INDEX idx_user_created (user_id, created_at DESC)
);
```

### 5.2 Feed Cache (Redis Sorted Set)

```
Redis key: "feed:{userId}"
Type: Sorted Set
Score: post_id (Snowflake — numerically sortable by time)
Value: post_id

ZADD feed:user123 1617000000123 post456    -- Add post to feed
ZREVRANGE feed:user123 0 49               -- Get latest 50 posts
ZREMRANGEBYRANK feed:user123 0 -501       -- Keep only 500 most recent

Each feed: max 500 post IDs
Storage: 500 posts × 8 bytes × 500M users = 2TB → distribute across Redis cluster
```

### 5.3 Fan-Out Service Implementation (Kafka Consumer)

```java
@Component
@RequiredArgsConstructor
public class FeedFanoutConsumer {

    private final FollowerRepository followerRepository;
    private final RedisTemplate<String, Long> redisTemplate;

    @KafkaListener(topics = "post.created", groupId = "feed-fanout")
    public void onPostCreated(PostCreatedEvent event) {
        // Only fan-out for non-celebrity accounts
        if (event.getFollowerCount() > 10_000) {
            log.debug("Skipping fan-out for celebrity userId={}", event.getUserId());
            return;
        }

        // Batch fetch followers (paginate for large follower counts)
        List<Long> followers = followerRepository.getFollowers(event.getUserId());

        // Bulk write to Redis using pipeline for performance
        redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
            for (Long followerId : followers) {
                String key = "feed:" + followerId;
                connection.zSetCommands().zAdd(
                    key.getBytes(),
                    event.getPostId(),          // score = postId (time-sortable)
                    String.valueOf(event.getPostId()).getBytes()
                );
                // Trim to 500 entries
                connection.zSetCommands().zRemRangeByRank(key.getBytes(), 0, -501);
            }
            return null;
        });
    }
}
```

### 5.4 Feed Read Service

```java
@Service
@RequiredArgsConstructor
public class FeedService {

    private final RedisTemplate<String, Long> redisTemplate;
    private final PostRepository postRepository;
    private final CelebrityFollowRepository celebrityFollows;

    public List<Post> getFeed(Long userId, int page, int pageSize) {
        // 1. Get pre-computed feed from Redis
        String key = "feed:" + userId;
        int start = page * pageSize;
        Set<Long> postIds = redisTemplate.opsForZSet()
            .reverseRange(key, start, start + pageSize - 1);

        // 2. Merge in celebrity posts (fan-out on read)
        List<Long> celebIds = celebrityFollows.getCelebrityFollowees(userId);
        List<Post> celebPosts = postRepository.getRecentPostsByUsers(celebIds, pageSize);

        // 3. Hydrate post IDs → full post objects
        List<Post> feedPosts = postRepository.findByIds(postIds);

        // 4. Merge, deduplicate, sort by postId (which is time-ordered)
        return Stream.concat(feedPosts.stream(), celebPosts.stream())
            .distinct()
            .sorted(Comparator.comparing(Post::getId).reversed())
            .limit(pageSize)
            .collect(Collectors.toList());
    }
}
```

---

## 6. Media Storage

```
Images/Videos → Object Storage (S3 / Azure Blob)
CDN (CloudFront) in front of S3 for delivery

Upload flow:
  Client → POST /media/upload → media service
  Media service → S3 (original)
  Media service → async resize/transcode job
  CDN URL returned to client

Storage: 200M posts/day × 1MB avg = 200TB/day → S3 auto-scales
```

---

## 7. Notification Service

```java
@KafkaListener(topics = "post.created")
public void onPostCreated(PostCreatedEvent event) {
    // Push notification to follower devices
    List<Long> followerIds = followerRepository.getActiveFollowers(event.getUserId());

    // Batch send push notifications (APNs/FCM)
    notificationBatcher.scheduleNotifications(
        followerIds,
        "New post from " + event.getUsername(),
        Map.of("postId", event.getPostId(), "action", "open_post")
    );
}
```

---

## 8. Key Takeaways

1. **The fan-out decision is the core design question** — fan-out on write (push) for regular users, fan-out on read (pull) for celebrities.
2. **Pre-compute feeds for read performance** — 289K reads/second demands O(1) feed retrieval.
3. **Kafka decouples post creation from feed updates** — a post is created fast; feed propagation is async.
4. **Store post IDs in feeds, not full posts** — posts can be updated/deleted; IDs are stable references.
5. **Object storage + CDN for media** — never serve images/videos from your application servers.

---

## 9. Connection to Previous Days

This design uses: Kafka (Day 19), Redis sorted sets (Day 18), consistent hashing for Redis cluster (Day 23), read replicas for MySQL (Day 27), CDN for media delivery (Day 28), circuit breakers on post service calls (Day 25), and distributed tracing to debug feed latency (Day 26).

---

## 10. Resources

- **[Twitter's Timeline Architecture](https://www.infoq.com/presentations/Twitter-Timeline-Scalability/)** — The real-world evolution.
- **[Instagram Engineering Blog — Feed Architecture](https://instagram-engineering.com/)** — Meta's approach.
- **[Designing Data-Intensive Applications, Ch. 11](https://dataintensive.net/)** — Stream processing patterns.

---

*Day 31. A social feed is a distributed systems problem wearing a consumer product's clothing.*
