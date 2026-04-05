# Day 35 — System Design Case Study: Video Streaming Platform
**Date:** 2026-04-11
**Phase:** 2 — System Design
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**System Design Case Study: Video Streaming Platform — Scale, Encoding, and Adaptive Bitrate**

Netflix has 260M subscribers. YouTube processes 500 hours of video every minute. Video streaming is among the most data-intensive challenges in backend engineering. The core problems: ingesting raw video, transcoding to multiple quality levels, delivering efficiently to users on varying network conditions, and keeping the CDN cost manageable. Today you design a video-on-demand (VOD) platform like Netflix.

---

## 2. Requirements

**Functional:**
- Upload videos (content creators / admins)
- Stream videos (users)
- Support multiple resolutions: 360p, 480p, 720p, 1080p, 4K
- Adaptive bitrate streaming: auto-adjust quality based on network speed
- Resume from where the user left off
- Search videos (from Day 34 — Elasticsearch)

**Non-Functional:**
- 10M daily active users, each watches ~1 hour/day
- 100K video uploads/day
- Data transfer: 10M users × 1h × ~2GB/h = 20 PB/day → **CDN handles 95%+ of this**
- Upload latency: video available within 5 minutes of upload
- Stream start latency: < 2 seconds
- p99 buffering: < 0.5% of playback time

---

## 3. Video Transcoding Pipeline

### 3.1 Why Transcoding?

```
Raw camera footage: 4K ProRes → 50GB for 1 hour, requires specialised decoder
User's phone: can't play ProRes, limited bandwidth, 720p screen

Transcoding output:
  - 360p @ 500 Kbps  (for 3G/slow connections)
  - 480p @ 1 Mbps    (for 4G)
  - 720p @ 3 Mbps    (for broadband)
  - 1080p @ 8 Mbps   (for fast broadband/TV)
  - 4K @ 20 Mbps     (for 4K TV + 100Mbps+ connections)

Format: H.264/H.265 in .mp4 or HLS segments (.ts files + .m3u8 manifest)
```

### 3.2 Upload & Transcoding Architecture

```
Creator uploads video
      │
      ▼ (multipart upload, resumable)
Upload Service → S3 (raw storage)
      │
      ▼ (publish event)
Kafka: video.uploaded
      │
      ▼ (consumed by)
Transcoding Orchestrator
      │
      ├──► Transcode Worker 1: 360p
      ├──► Transcode Worker 2: 480p
      ├──► Transcode Worker 3: 720p
      ├──► Transcode Worker 4: 1080p
      └──► Transcode Worker 5: 4K (only for premium content)
                │
                ▼ (each worker: ffmpeg → S3 output bucket)
         S3: transcoded-videos/{videoId}/360p/
                                        /480p/
                                        /720p/
                                        /1080p/
                                        ...
                │
                ▼ (all renditions complete)
Kafka: video.transcoded
      │
      ▼
Video Metadata Service: mark video as AVAILABLE
CDN: pre-warm edge caches for popular videos
```

### 3.3 FFmpeg Transcoding (Java Process Wrapper)

```java
@Service
public class TranscodingService {

    public void transcode(String inputS3Key, String outputPrefix,
                          Resolution resolution) throws Exception {
        // Download from S3 to local temp
        Path inputPath = downloadFromS3(inputS3Key);
        Path outputDir = Files.createTempDirectory("transcode-output");

        // Run ffmpeg with HLS output
        ProcessBuilder pb = new ProcessBuilder(
            "ffmpeg",
            "-i", inputPath.toString(),
            "-vf", "scale=" + resolution.getWidth() + ":-2",
            "-c:v", "libx264",
            "-crf", "23",
            "-preset", "fast",
            "-c:a", "aac",
            "-b:a", "128k",
            "-hls_time", "6",           // 6-second segments
            "-hls_list_size", "0",      // Keep all segments in playlist
            "-hls_segment_filename", outputDir + "/seg%d.ts",
            outputDir + "/playlist.m3u8"
        );

        Process process = pb.redirectErrorStream(true).start();
        int exitCode = process.waitFor();
        if (exitCode != 0) throw new TranscodingException("ffmpeg failed");

        // Upload segments to S3
        uploadDirectoryToS3(outputDir, outputPrefix + "/" + resolution.name() + "/");
    }
}
```

---

## 4. Adaptive Bitrate Streaming (ABR)

### 4.1 HLS (HTTP Live Streaming) — Apple Standard

```
Master playlist (videoId/master.m3u8):
  #EXTM3U
  #EXT-X-STREAM-INF:BANDWIDTH=500000,RESOLUTION=640x360
  360p/playlist.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=1000000,RESOLUTION=854x480
  480p/playlist.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=3000000,RESOLUTION=1280x720
  720p/playlist.m3u8

Segment playlist (videoId/720p/playlist.m3u8):
  #EXTM3U
  #EXT-X-VERSION:3
  #EXT-X-TARGETDURATION:6
  seg0.ts
  seg1.ts
  seg2.ts
  ...

Client player:
  1. Downloads master.m3u8
  2. Picks appropriate quality based on measured bandwidth
  3. Downloads 720p/playlist.m3u8
  4. Downloads segments sequentially
  5. If buffer starts draining → switches to 480p/playlist.m3u8
```

### 4.2 DASH (Dynamic Adaptive Streaming over HTTP) — YouTube/Netflix Standard

Similar to HLS but uses XML manifest (MPD file). Netflix uses MPEG-DASH.

---

## 5. Content Delivery Architecture

```
User plays video
      │
      ▼
Player downloads: https://cdn.example.com/{videoId}/master.m3u8
      │
      ▼ CDN Edge Node (Tokyo for APAC users)
      │  Cache hit: return cached m3u8 + segments
      │  Cache miss: fetch from S3 origin, cache
      │
      ▼ S3 (only on cache miss, ~5% of requests)

CDN setup:
  - CloudFront / Akamai in front of S3
  - Segment TTL: 1 year (segments are immutable)
  - Playlist TTL: 5 minutes (playlist could be updated)
  - Pre-warm: push popular video segments to all edge nodes
    when a video goes viral

Bandwidth math:
  10M users × 1h × 3 Mbps avg = ~1.35 PB/hour CDN throughput
  CDN cost: ~$0.01/GB → $13.5M/hour ← why video platforms need massive scale to be profitable
```

---

## 6. Resuming Playback (Watch History)

```java
@RestController
@RequiredArgsConstructor
public class WatchHistoryController {

    private final WatchHistoryService watchHistoryService;

    // Player calls this every 30 seconds during playback
    @PostMapping("/v1/videos/{videoId}/progress")
    public void updateProgress(@PathVariable Long videoId,
                               @RequestBody ProgressUpdate update,
                               @AuthenticationPrincipal Long userId) {
        watchHistoryService.updatePosition(userId, videoId, update.getPositionSeconds());
    }

    // Player calls on load — resume from last position
    @GetMapping("/v1/videos/{videoId}/progress")
    public ProgressResponse getProgress(@PathVariable Long videoId,
                                        @AuthenticationPrincipal Long userId) {
        return watchHistoryService.getPosition(userId, videoId);
    }
}

@Service
public class WatchHistoryService {
    // Write to Redis (fast) + async to MySQL (durable)
    public void updatePosition(Long userId, Long videoId, int positionSeconds) {
        redis.opsForValue().set("progress:" + userId + ":" + videoId,
            positionSeconds, Duration.ofDays(90));
        kafkaTemplate.send("watch.progress", new WatchProgressEvent(userId, videoId, positionSeconds));
    }
}
```

---

## 7. Database Schema

```sql
CREATE TABLE videos (
    id              BIGINT PRIMARY KEY,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    creator_id      BIGINT NOT NULL,
    status          ENUM('UPLOADING', 'PROCESSING', 'AVAILABLE', 'DELETED'),
    duration_seconds INT,
    thumbnail_url   VARCHAR(512),
    s3_key_prefix   VARCHAR(512),    -- S3 path prefix for HLS files
    view_count      BIGINT DEFAULT 0,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_creator (creator_id),
    INDEX idx_status (status)
);

CREATE TABLE watch_history (
    user_id         BIGINT NOT NULL,
    video_id        BIGINT NOT NULL,
    position_seconds INT DEFAULT 0,
    completed       BOOLEAN DEFAULT FALSE,
    last_watched_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, video_id)
);
```

---

## 8. Key Takeaways

1. **Transcoding is a CPU-intensive async pipeline** — never block uploads; use Kafka + worker pools.
2. **HLS/DASH adaptive streaming lets the player choose quality** — no server-side logic needed for ABR.
3. **CDN absorbs 95%+ of video bandwidth** — without it, S3 costs would be unsustainable.
4. **Segments are immutable, playlists are mutable** — different TTLs for efficient caching.
5. **Watch position in Redis, async persisted to MySQL** — speed on hot path, durability on cold path.

---

## 9. Resources

- **[Netflix Tech Blog — Video Encoding](https://netflixtechblog.com/high-quality-video-encoding-at-scale-d159db052746)** — How Netflix does per-title encoding optimisation.
- **[HLS Specification (Apple)](https://developer.apple.com/streaming/)** — The authoritative reference.
- **[DASH Industry Forum](https://dashif.org/)** — MPEG-DASH standard.

---

*Day 35. Video streaming is the most bandwidth-intensive problem in consumer technology — the CDN is the product.*
