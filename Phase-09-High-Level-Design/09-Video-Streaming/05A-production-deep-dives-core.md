# Video Streaming - Production Deep Dives (Core)

## Overview

This document covers the technical implementation details for key components: video transcoding, adaptive bitrate streaming, CDN optimization, view count aggregation, and chunked uploads.

---

## 1. Video Transcoding Deep Dive

### Transcoding Service

```java
@Service
public class TranscodingService {

    private final S3Client s3Client;
    private final KafkaTemplate<String, TranscodeJob> kafkaTemplate;
    private final TranscodeJobRepository jobRepository;

    private static final List<Resolution> RESOLUTIONS = List.of(
        new Resolution("240p", 426, 240, 400_000),
        new Resolution("360p", 640, 360, 800_000),
        new Resolution("480p", 854, 480, 1_500_000),
        new Resolution("720p", 1280, 720, 3_000_000),
        new Resolution("1080p", 1920, 1080, 6_000_000),
        new Resolution("1440p", 2560, 1440, 10_000_000),
        new Resolution("4k", 3840, 2160, 20_000_000)
    );

    public void processVideo(String videoId, String inputPath) {
        // 1. Analyze input video
        VideoMetadata metadata = analyzeVideo(inputPath);

        // 2. Determine output resolutions (don't upscale)
        List<Resolution> outputResolutions = RESOLUTIONS.stream()
            .filter(r -> r.getHeight() <= metadata.getHeight())
            .collect(Collectors.toList());

        // 3. Create jobs for each resolution
        for (Resolution resolution : outputResolutions) {
            TranscodeJob job = TranscodeJob.builder()
                .videoId(videoId)
                .inputPath(inputPath)
                .resolution(resolution.getName())
                .targetBitrate(resolution.getBitrate())
                .status(JobStatus.PENDING)
                .build();

            jobRepository.save(job);
            kafkaTemplate.send("transcode-jobs", videoId, job);
        }

        // 4. Create thumbnail job
        createThumbnailJob(videoId, inputPath, metadata.getDuration());
    }

    private VideoMetadata analyzeVideo(String inputPath) {
        // Use FFprobe to get video metadata
        ProcessBuilder pb = new ProcessBuilder(
            "ffprobe", "-v", "quiet", "-print_format", "json",
            "-show_format", "-show_streams", inputPath
        );

        Process process = pb.start();
        String output = new String(process.getInputStream().readAllBytes());
        return parseFFprobeOutput(output);
    }
}
```

### Transcoding Worker

```java
@Component
public class TranscodeWorker {

    private final S3Client s3Client;
    private final TranscodeJobRepository jobRepository;

    @KafkaListener(topics = "transcode-jobs")
    public void processJob(TranscodeJob job) {
        job.setStatus(JobStatus.PROCESSING);
        job.setStartedAt(Instant.now());
        jobRepository.save(job);

        try {
            // 1. Download input file
            String localInput = downloadFromS3(job.getInputPath());

            // 2. Transcode
            String localOutput = transcode(localInput, job);

            // 3. Segment for HLS
            List<String> segments = segmentVideo(localOutput, job);

            // 4. Upload segments to S3
            uploadSegments(job.getVideoId(), job.getResolution(), segments);

            // 5. Create playlist
            createPlaylist(job.getVideoId(), job.getResolution(), segments);

            job.setStatus(JobStatus.COMPLETED);
            job.setCompletedAt(Instant.now());

        } catch (Exception e) {
            job.setStatus(JobStatus.FAILED);
            job.setErrorMessage(e.getMessage());
            log.error("Transcoding failed for {}", job.getVideoId(), e);
        }

        jobRepository.save(job);

        // Check if all resolutions complete
        checkAllJobsComplete(job.getVideoId());
    }

    private String transcode(String input, TranscodeJob job) {
        String output = "/tmp/" + job.getVideoId() + "_" + job.getResolution() + ".mp4";

        // FFmpeg command for transcoding
        List<String> command = List.of(
            "ffmpeg", "-i", input,
            "-c:v", "libx264",
            "-preset", "medium",
            "-b:v", String.valueOf(job.getTargetBitrate()),
            "-maxrate", String.valueOf((int)(job.getTargetBitrate() * 1.5)),
            "-bufsize", String.valueOf(job.getTargetBitrate() * 2),
            "-vf", "scale=-2:" + getHeight(job.getResolution()),
            "-c:a", "aac",
            "-b:a", "128k",
            "-movflags", "+faststart",
            output
        );

        executeCommand(command);
        return output;
    }

    private List<String> segmentVideo(String input, TranscodeJob job) {
        String outputPattern = "/tmp/" + job.getVideoId() + "_" +
                              job.getResolution() + "_segment_%04d.ts";

        // Segment into 10-second chunks
        List<String> command = List.of(
            "ffmpeg", "-i", input,
            "-c", "copy",
            "-f", "segment",
            "-segment_time", "10",
            "-segment_format", "mpegts",
            "-segment_list", "/tmp/playlist.m3u8",
            outputPattern
        );

        executeCommand(command);

        // Return list of segment files
        return listSegmentFiles(job.getVideoId(), job.getResolution());
    }
}
```

---

## 2. Adaptive Bitrate Streaming

### HLS Manifest Generation

```java
@Service
public class ManifestService {

    public String generateMasterPlaylist(String videoId, List<String> resolutions) {
        StringBuilder playlist = new StringBuilder();
        playlist.append("#EXTM3U\n");
        playlist.append("#EXT-X-VERSION:3\n");

        for (String resolution : resolutions) {
            ResolutionConfig config = getConfig(resolution);

            playlist.append("#EXT-X-STREAM-INF:");
            playlist.append("BANDWIDTH=").append(config.getBitrate());
            playlist.append(",RESOLUTION=").append(config.getWidth())
                    .append("x").append(config.getHeight());
            playlist.append(",CODECS=\"avc1.64001f,mp4a.40.2\"\n");
            playlist.append(resolution).append("/playlist.m3u8\n");
        }

        return playlist.toString();
    }

    public String generateMediaPlaylist(String videoId, String resolution,
                                         List<Segment> segments) {
        StringBuilder playlist = new StringBuilder();
        playlist.append("#EXTM3U\n");
        playlist.append("#EXT-X-VERSION:3\n");
        playlist.append("#EXT-X-TARGETDURATION:10\n");
        playlist.append("#EXT-X-MEDIA-SEQUENCE:0\n");

        for (Segment segment : segments) {
            playlist.append("#EXTINF:").append(segment.getDuration()).append(",\n");
            playlist.append(segment.getFilename()).append("\n");
        }

        playlist.append("#EXT-X-ENDLIST\n");

        return playlist.toString();
    }
}
```

### Client-Side ABR Algorithm

```javascript
class AdaptiveBitrateController {
  constructor(player) {
    this.player = player;
    this.bandwidthHistory = [];
    this.currentQuality = 0;
    this.bufferTarget = 30; // seconds
  }

  onSegmentDownloaded(segment) {
    // Calculate bandwidth from download
    const bandwidth = (segment.size * 8) / segment.downloadTime;
    this.bandwidthHistory.push(bandwidth);

    // Keep last 5 samples
    if (this.bandwidthHistory.length > 5) {
      this.bandwidthHistory.shift();
    }

    // Estimate available bandwidth (conservative)
    const estimatedBandwidth = this.getEstimatedBandwidth();

    // Get current buffer level
    const bufferLevel = this.player.getBufferLevel();

    // Select quality
    const newQuality = this.selectQuality(estimatedBandwidth, bufferLevel);

    if (newQuality !== this.currentQuality) {
      this.switchQuality(newQuality);
    }
  }

  getEstimatedBandwidth() {
    if (this.bandwidthHistory.length === 0) return 0;

    // Use harmonic mean (conservative estimate)
    const sum = this.bandwidthHistory.reduce((a, b) => a + 1 / b, 0);
    return this.bandwidthHistory.length / sum;
  }

  selectQuality(bandwidth, bufferLevel) {
    const qualities = this.player.getQualities();

    // If buffer is low, be more conservative
    let safetyFactor = 0.8;
    if (bufferLevel < 10) {
      safetyFactor = 0.5; // More conservative when buffer is low
    }

    const safeBandwidth = bandwidth * safetyFactor;

    // Find highest quality that fits
    for (let i = qualities.length - 1; i >= 0; i--) {
      if (qualities[i].bitrate <= safeBandwidth) {
        return i;
      }
    }

    return 0; // Lowest quality
  }

  switchQuality(newQuality) {
    console.log(`Switching from ${this.currentQuality} to ${newQuality}`);
    this.currentQuality = newQuality;
    this.player.setQuality(newQuality);
  }
}
```

### Quality Levels

```
┌───────────────────────────────────────────────────────────────────────────────────┐
│  QUALITY LEVELS                                                                    │
│                                                                                    │
│  Resolution │ Bitrate  │ Min Bandwidth │ Use Case                                 │
│  ───────────┼──────────┼───────────────┼──────────────────────────────────────    │
│  240p       │ 400 Kbps │ 500 Kbps      │ Very slow connections                    │
│  360p       │ 800 Kbps │ 1 Mbps        │ Mobile data saving                       │
│  480p       │ 1.5 Mbps │ 2 Mbps        │ Standard mobile                          │
│  720p       │ 3 Mbps   │ 4 Mbps        │ HD on mobile/tablet                      │
│  1080p      │ 6 Mbps   │ 8 Mbps        │ Full HD on desktop                       │
│  1440p      │ 10 Mbps  │ 12 Mbps       │ 2K displays                              │
│  4K         │ 20 Mbps  │ 25 Mbps       │ 4K TVs                                   │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. CDN Optimization

### Cache Key Strategy

```java
@Service
public class CacheKeyService {

    /**
     * Generate cache key for video segment
     *
     * Key format: /v/{video_id}/{resolution}/segment_{n}.ts
     *
     * Why this format:
     * - Video ID first: Easy to purge all versions of a video
     * - Resolution next: Different cache entries per quality
     * - Segment number: Individual segment caching
     */
    public String generateCacheKey(String videoId, String resolution, int segmentNumber) {
        return String.format("/v/%s/%s/segment_%04d.ts",
            videoId, resolution, segmentNumber);
    }

    /**
     * Generate cache key with signed URL for access control
     */
    public String generateSignedUrl(String videoId, String resolution,
                                     int segmentNumber, Duration ttl) {
        String cacheKey = generateCacheKey(videoId, resolution, segmentNumber);
        long expiry = Instant.now().plus(ttl).getEpochSecond();

        String signature = generateSignature(cacheKey, expiry);

        return String.format("https://cdn.example.com%s?expires=%d&sig=%s",
            cacheKey, expiry, signature);
    }
}
```

### Cache Warming

```java
@Service
public class CacheWarmingService {

    private final CdnClient cdnClient;
    private final VideoRepository videoRepository;

    /**
     * Warm cache for newly processed video
     */
    public void warmCacheForVideo(String videoId) {
        // Get all resolutions
        List<String> resolutions = videoRepository.getResolutions(videoId);

        // Warm first 30 seconds (3 segments) at each edge
        for (String resolution : resolutions) {
            for (int i = 0; i < 3; i++) {
                String url = generateUrl(videoId, resolution, i);
                cdnClient.prefetch(url);
            }
        }

        // Warm manifest files
        cdnClient.prefetch(generateMasterManifestUrl(videoId));
        for (String resolution : resolutions) {
            cdnClient.prefetch(generateMediaManifestUrl(videoId, resolution));
        }
    }

    /**
     * Predictive cache warming for trending videos
     */
    @Scheduled(fixedRate = 300000) // Every 5 minutes
    public void warmTrendingVideos() {
        List<String> trendingVideos = getTrendingVideos(100);

        for (String videoId : trendingVideos) {
            // Warm popular resolutions (720p, 1080p)
            warmResolution(videoId, "720p");
            warmResolution(videoId, "1080p");
        }
    }

    private void warmResolution(String videoId, String resolution) {
        int totalSegments = getSegmentCount(videoId, resolution);

        // Warm first 50% of video (most viewers don't finish)
        int segmentsToWarm = totalSegments / 2;

        for (int i = 0; i < segmentsToWarm; i++) {
            String url = generateUrl(videoId, resolution, i);
            cdnClient.prefetch(url);
        }
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: CDN Technology-Level

**Normal Flow: Video Playback via CDN**

```
Request: GET /v/video_abc123/720p/segment_0001.ts
    ↓
Step 1: Client Request
┌─────────────────────────────────────────────────────────────┐
│ Client: Requests video segment from CDN                    │
│ URL: https://cdn.videoplatform.com/v/video_abc123/720p/   │
│      segment_0001.ts                                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: CDN Edge Check
┌─────────────────────────────────────────────────────────────┐
│ CDN Edge (NYC): Check cache for segment                    │
│ Cache Key: /v/video_abc123/720p/segment_0001.ts           │
│ Result: HIT (segment cached from previous request)         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Return Cached Content
┌─────────────────────────────────────────────────────────────┐
│ CDN Edge: Return segment from cache                         │
│ Latency: ~10ms (edge to client)                            │
│ Bandwidth: 3 Mbps (720p segment)                           │
│ Total time: ~2 seconds (for 6MB segment)                   │
└─────────────────────────────────────────────────────────────┘
```

**Cache MISS Flow: CDN to Origin**

```
Request: GET /v/video_xyz789/1080p/segment_0005.ts
    ↓
Step 1: CDN Edge Cache MISS
┌─────────────────────────────────────────────────────────────┐
│ CDN Edge (SF): Check cache for segment                      │
│ Cache Key: /v/video_xyz789/1080p/segment_0005.ts           │
│ Result: MISS (segment not cached)                          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Origin Shield Check
┌─────────────────────────────────────────────────────────────┐
│ Origin Shield (Regional): Check cache                       │
│ Result: MISS (segment not in regional cache)                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Fetch from Origin Storage
┌─────────────────────────────────────────────────────────────┐
│ Origin Shield: Fetch from S3                                │
│ S3: GetObject video_xyz789/1080p/segment_0005.ts           │
│ Latency: ~50ms (S3 to origin shield)                       │
│ Download: 6MB segment (1080p)                              │
│ Total: ~200ms                                               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Cache at Multiple Levels
┌─────────────────────────────────────────────────────────────┐
│ Origin Shield: Cache segment (TTL: 24 hours)               │
│ CDN Edge: Cache segment (TTL: 24 hours)                   │
│ Return: Segment to client                                   │
│ Latency: ~250ms total (first request)                      │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: CDN Edge Failure**

```
Request: GET /v/video_abc123/720p/segment_0001.ts
    ↓
Step 1: Primary Edge Failure
┌─────────────────────────────────────────────────────────────┐
│ CDN Edge (NYC): Health check fails                          │
│ Status: Edge location down                                  │
│ Detection: < 30 seconds                                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Automatic Failover
┌─────────────────────────────────────────────────────────────┐
│ Anycast DNS: Routes to next closest edge                    │
│ NYC → DC edge (automatic)                                   │
│ Failover time: < 30 seconds                                 │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: DC Edge Serves Request
┌─────────────────────────────────────────────────────────────┐
│ CDN Edge (DC): Check cache                                  │
│ Result: HIT (segment cached)                                │
│ Return: Segment to client                                   │
│ Latency: +20ms (NYC to DC routing)                         │
│ Impact: Minimal, automatic recovery                         │
└─────────────────────────────────────────────────────────────┘
```

**Traffic Spike Handling: Viral Video**

```
Scenario: Viral video, 1M concurrent viewers
    ↓
Step 1: Initial Requests
┌─────────────────────────────────────────────────────────────┐
│ T+0s: First 1000 requests hit CDN edge                     │
│       - Cache MISS (new video)                             │
│       - All requests forward to origin shield               │
│       - Origin shield fetches from S3                       │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Cache Population
┌─────────────────────────────────────────────────────────────┐
│ T+5s: Origin shield caches segments                         │
│ T+10s: CDN edges cache segments (from origin shield)        │
│ T+15s: 95% of requests served from edge cache              │
│        - Edge hit rate: 95%                                 │
│        - Origin load: 5% of requests                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Scale Handling
┌─────────────────────────────────────────────────────────────┐
│ T+30s: 1M concurrent viewers                                │
│        - Each edge: ~10K requests/second                    │
│        - 100 edges globally: 1M requests/second             │
│        - All served from edge cache (no origin load)       │
│        - No S3 throttling                                   │
└─────────────────────────────────────────────────────────────┘
```

**Recovery Behavior: CDN Origin Failure**

```
T+0s:    Origin storage (S3) fails
    ↓
T+10s:   CDN detects origin errors
    ↓
T+20s:   CDN switches to secondary origin (cross-region S3)
    ↓
T+30s:   Requests resume, served from secondary origin
    ↓
T+60s:   Primary origin recovers
    ↓
T+90s:   CDN switches back to primary origin
    ↓
RTO: < 1 minute
RPO: 0 (content in secondary region)
```

---

## 3.5. Kafka for Transcoding Jobs (Async Messaging)

### A) CONCEPT: What is Kafka?

Apache Kafka is a distributed event streaming platform designed for high-throughput, fault-tolerant data pipelines. For video transcoding, Kafka decouples the upload service from the transcoding workers, allowing horizontal scaling and fault tolerance.

**What problems does Kafka solve here?**

1. **Decoupling**: Upload service doesn't wait for transcoding to complete
2. **Buffering**: Handles traffic spikes (viral video uploads)
3. **Reliability**: Transcoding jobs persist even if workers crash
4. **Scalability**: Multiple workers can process jobs in parallel
5. **Ordering**: Jobs for same video processed in order

### B) OUR USAGE: How We Use Kafka Here

**Topic Design:**

```
Topic: transcode-jobs
Partitions: 32
Replication Factor: 3
Retention: 7 days
Cleanup Policy: delete
```

**Partition Key Choice:**

We partition by `video_id` hash. This ensures:

- All jobs for the same video go to the same partition
- Ordering is preserved per video (important for dependency management)
- Even distribution across partitions

**Consumer Group Design:**

```
Consumer Group: transcoding-workers
Consumers: 20 instances (each handles 1-2 partitions)
Processing: One job at a time per worker (CPU-intensive)
```

### C) REAL STEP-BY-STEP SIMULATION

**Normal Flow: Video Upload and Transcoding**

```
Step 1: Video Upload Complete
┌─────────────────────────────────────────────────────────────┐
│ POST /v1/videos/{video_id}/upload/complete                   │
│ Video ID: "vid_abc123"                                      │
│ Original file: s3://originals/vid_abc123/original.mp4       │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Transcoding Service Creates Jobs
┌─────────────────────────────────────────────────────────────┐
│ 1. Analyze video: 1080p source, 10 minutes                 │
│ 2. Create 7 transcoding jobs (one per resolution):          │
│    - Job 1: 240p                                             │
│    - Job 2: 360p                                             │
│    - Job 3: 480p                                             │
│    - Job 4: 720p                                             │
│    - Job 5: 1080p                                            │
│    - Job 6: Thumbnail                                        │
│ 3. Store jobs in PostgreSQL (status: PENDING)                 │
│ 4. Publish to Kafka:                                         │
│    Topic: transcode-jobs                                     │
│    Partition: hash(vid_abc123) % 32 = partition 15          │
│    Message: {                                                │
│      "job_id": "job_1",                                      │
│      "video_id": "vid_abc123",                               │
│      "resolution": "240p",                                    │
│      "input_path": "s3://originals/vid_abc123/original.mp4", │
│      "status": "PENDING"                                     │
│    }                                                          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Transcoding Worker Consumes Job
┌─────────────────────────────────────────────────────────────┐
│ Worker 15 (assigned to partition 15) polls:                 │
│ - Receives batch of 1 job (job_1 for 240p)                  │
│ - Updates job status: PROCESSING                            │
│ - Downloads original from S3: ~30 seconds                   │
│ - Transcodes: FFmpeg processing ~2 minutes                  │
│ - Segments video: ~10 seconds                               │
│ - Uploads segments to S3: ~20 seconds                       │
│ - Updates job status: COMPLETED                             │
│ - Commits Kafka offset                                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: All Jobs Complete
┌─────────────────────────────────────────────────────────────┐
│ When last job completes:                                    │
│ - Check all 7 jobs are COMPLETED                            │
│ - Generate HLS playlists                                     │
│ - Update video status: READY                                 │
│ - Publish event: video-ready (for CDN warming)              │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: Worker Crash During Transcoding**

```
Scenario: Worker crashes mid-transcoding (after 1 minute of processing)

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Worker 15 starts transcoding job_1                    │
│ T+30s: Download from S3 complete                           │
│ T+60s: FFmpeg processing 50% complete                      │
│ T+61s: Worker crashes (OOM, hardware failure)              │
│ T+61s: Kafka offset NOT committed (job still processing)   │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+65s: Kafka consumer group rebalance                       │
│ - Coordinator detects worker 15 left group                  │
│ - Reassigns partition 15 to worker 16                       │
│ - Worker 16 starts from last committed offset              │
│ - Receives job_1 again (offset not committed)             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Worker 16 Processing:                                       │
│ - Checks job status in DB: PROCESSING (stale)              │
│ - Resets job status: PENDING                                │
│ - Downloads original from S3 again (idempotent)           │
│ - Transcodes from scratch                                   │
│ - Completes successfully                                    │
│ - Commits offset                                            │
└─────────────────────────────────────────────────────────────┘
```

**Idempotency Handling:**

```
Problem: Same job processed twice (worker crash + retry)

Solution: Idempotent operations
1. S3 uploads: Same key overwrites (idempotent)
2. Database updates: UPSERT with job_id (unique constraint)
3. Segment generation: Delete existing segments before creating new ones

Example:
- Job processed twice → Same S3 keys → Overwrites (no duplicates)
- Job status: PENDING → PROCESSING → COMPLETED (idempotent state machine)
```

**Traffic Spike Handling:**

```
Scenario: Viral video upload, 1000 videos uploaded in 1 hour

┌─────────────────────────────────────────────────────────────┐
│ Normal: 10 videos/hour = 70 jobs/hour (7 jobs per video)    │
│ Spike: 1000 videos/hour = 7,000 jobs/hour                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Kafka Buffering:                                            │
│ - 7,000 messages buffered in Kafka                         │
│ - 7-day retention = 1.2M messages capacity                 │
│ - No message loss                                           │
│                                                              │
│ Worker Scaling:                                             │
│ - Auto-scale workers: 20 → 100 instances                    │
│ - Each worker: 1 job at a time (CPU-bound)                 │
│ - Processing time: 2-5 minutes per job                      │
│ - Throughput: 100 workers × 12 jobs/hour = 1,200 jobs/hour│
│                                                              │
│ Result:                                                      │
│ - Jobs queued in Kafka (no loss)                            │
│ - Processing delayed but continues                           │
│ - All videos eventually processed                           │
└─────────────────────────────────────────────────────────────┘
```

**Hot Partition Mitigation:**

```
Problem: Celebrity uploads video, all jobs go to one partition

Scenario: Partition 15 receives 7 jobs for viral video

Mitigation:
1. Partition by video_id (distributes across videos)
2. Multiple workers per partition (if needed)
3. Priority queue for viral videos (separate topic)

If hot partition occurs:
- Monitor partition lag
- Add more workers to that partition
- Consider priority topic for high-profile videos
```

**Recovery Behavior:**

```
Auto-healing:
- Worker crash → Kafka rebalance → Job reassigned → Retry
- S3 upload failure → Retry with exponential backoff
- Transcoding failure → Mark job FAILED, alert ops

Human intervention:
- Job stuck in PROCESSING > 1 hour → Manual investigation
- Repeated failures → Check video file integrity
- Worker cluster issues → Scale up or replace workers
```

---

## 3.6. Redis Video Metadata Cache (Detailed Simulation)

### A) CONCEPT: What is Video Metadata Caching?

Video metadata (title, description, thumbnail, duration, etc.) is frequently accessed but rarely changes. Caching this data in Redis dramatically reduces database load and improves response times.

**What we cache:**

- Video metadata (title, description, creator, duration)
- Video availability (which resolutions are ready)
- Thumbnail URLs
- Creator information

### B) OUR USAGE: How We Use Redis Here

**Cache Keys:**

```
Key: video:meta:{video_id}
Value: JSON {title, description, creator_id, duration, resolutions, thumbnail_url}
TTL: 1 hour

Key: creator:{creator_id}
Value: JSON {name, avatar_url, verified}
TTL: 24 hours
```

**Cache-Aside Pattern:**

- Application checks Redis first
- If miss: Query database, cache result, return
- If hit: Return from cache

### C) REAL STEP-BY-STEP SIMULATION

**Cache Hit Path:**

```
Step 1: User Requests Video Metadata
┌─────────────────────────────────────────────────────────────┐
│ Request: GET /v1/videos/vid_123                            │
│ User wants to display video info on page                    │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Check Redis Cache
┌─────────────────────────────────────────────────────────────┐
│ Redis: GET video:meta:vid_123                              │
│ Result: HIT                                                 │
│ Value: {                                                    │
│   "title": "Amazing Video",                                 │
│   "description": "...",                                    │
│   "creator_id": "user_456",                                │
│   "duration": 300,                                          │
│   "resolutions": ["720p", "1080p"],                        │
│   "thumbnail_url": "https://..."                            │
│ }                                                           │
│ Latency: ~1ms                                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Return Cached Data
┌─────────────────────────────────────────────────────────────┐
│ Response: 200 OK                                            │
│ Body: Cached video metadata                                │
│ Total latency: ~2ms (cache hit)                            │
└─────────────────────────────────────────────────────────────┘
```

**Cache Miss Path:**

```
Step 1: User Requests Video Metadata
┌─────────────────────────────────────────────────────────────┐
│ Request: GET /v1/videos/vid_789                            │
│ (New video, not in cache yet)                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Check Redis Cache
┌─────────────────────────────────────────────────────────────┐
│ Redis: GET video:meta:vid_789                              │
│ Result: MISS                                                │
│ Latency: ~1ms                                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Query Database
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: SELECT * FROM videos WHERE id = 'vid_789'      │
│ JOIN creators ON videos.creator_id = creators.id            │
│ Latency: ~10ms                                             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Cache Results
┌─────────────────────────────────────────────────────────────┐
│ Redis: SET video:meta:vid_789 {metadata} EX 3600          │
│ (Cache for 1 hour)                                         │
│ Latency: ~1ms                                              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Return Results
┌─────────────────────────────────────────────────────────────┐
│ Response: 200 OK                                            │
│ Body: Video metadata                                        │
│ Total latency: ~15ms (cache miss)                          │
└─────────────────────────────────────────────────────────────┘
```

**What if Redis is down?**

- Circuit breaker opens after 50% failure rate
- Fallback to direct database queries
- Latency increases from ~2ms to ~10ms
- System continues operating (graceful degradation)

**Cache Invalidation:**

```
Step 1: Creator Updates Video Title
┌─────────────────────────────────────────────────────────────┐
│ Request: PUT /v1/videos/vid_123                             │
│ Body: {"title": "Updated Title"}                           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Update Database
┌─────────────────────────────────────────────────────────────┐
│ PostgreSQL: UPDATE videos SET title = 'Updated Title'      │
│              WHERE id = 'vid_123'                          │
│ Transaction committed                                       │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Invalidate Cache
┌─────────────────────────────────────────────────────────────┐
│ Redis: DEL video:meta:vid_123                              │
│ (Remove stale cache entry)                                 │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Next Request
┌─────────────────────────────────────────────────────────────┐
│ Next GET /v1/videos/vid_123 → Cache MISS                   │
│ → Query database → Get updated title                       │
│ → Cache new value                                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. View Count Aggregation

### Real-time View Counter

```java
@Service
public class ViewCountService {

    private final RedisTemplate<String, Long> redis;
    private final KafkaTemplate<String, ViewEvent> kafka;

    /**
     * Record a view event
     *
     * A view is counted when:
     * - User watches at least 30 seconds, OR
     * - User watches 50% of video (for short videos)
     */
    public void recordView(ViewEvent event) {
        // 1. Validate view (prevent fraud)
        if (!isValidView(event)) {
            return;
        }

        // 2. Increment real-time counter in Redis
        String key = "views:" + event.getVideoId();
        redis.opsForValue().increment(key);

        // 3. Track unique viewers (HyperLogLog)
        String hlKey = "unique_viewers:" + event.getVideoId();
        redis.opsForHyperLogLog().add(hlKey, event.getUserId());

        // 4. Publish to Kafka for analytics
        kafka.send("view-events", event.getVideoId(), event);
    }

    private boolean isValidView(ViewEvent event) {
        // Check watch duration
        if (event.getWatchDuration() < 30 &&
            event.getWatchDuration() < event.getVideoDuration() * 0.5) {
            return false;
        }

        // Check for duplicate view (same user, same video, within 30 minutes)
        String dedupKey = "viewed:" + event.getUserId() + ":" + event.getVideoId();
        Boolean isNew = redis.opsForValue().setIfAbsent(dedupKey, 1L,
            Duration.ofMinutes(30));

        return Boolean.TRUE.equals(isNew);
    }

    public long getViewCount(String videoId) {
        // Try Redis first (real-time)
        Long redisCount = redis.opsForValue().get("views:" + videoId);
        if (redisCount != null) {
            return redisCount;
        }

        // Fallback to database
        return videoRepository.getViewCount(videoId);
    }
}
```

### Cache Stampede Prevention (Redis View Count Cache)

**What is Cache Stampede?**

Cache stampede occurs when cached view count data expires and many requests simultaneously try to regenerate it, overwhelming the database.

**Scenario:**

```
T+0s:   Popular video's view count cache expires
T+0s:   10,000 requests arrive simultaneously
T+0s:   All 10,000 requests see cache MISS
T+0s:   All 10,000 requests hit database simultaneously
T+1s:   Database overwhelmed, latency spikes
```

**Prevention Strategy: Distributed Lock + Double-Check Pattern**

For view count cache misses, this system uses distributed locking:

1. **Distributed locking**: Only one thread fetches from database on cache miss
2. **Double-check pattern**: Re-check cache after acquiring lock
3. **TTL jitter**: Add random jitter to TTL to prevent simultaneous expiration

**Implementation:**

The view count cache uses Redis INCR operations which are atomic, but for video metadata cache (if implemented), use distributed locks:

```java
@Service
public class VideoMetadataCacheService {

    private final RedisTemplate<String, VideoMetadata> redis;
    private final DistributedLock lockService;
    private final VideoRepository videoRepository;

    public VideoMetadata getCachedMetadata(String videoId) {
        String key = "video:meta:" + videoId;

        // 1. Try cache first
        VideoMetadata cached = redis.opsForValue().get(key);
        if (cached != null) {
            return cached;
        }

        // 2. Cache miss: Try to acquire lock
        String lockKey = "lock:video:meta:" + videoId;
        boolean acquired = lockService.tryLock(lockKey, Duration.ofSeconds(5));

        if (!acquired) {
            // Another thread is fetching, wait briefly
            try {
                Thread.sleep(50);
                cached = redis.opsForValue().get(key);
                if (cached != null) {
                    return cached;
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            // Fall back to database (acceptable degradation)
        }

        try {
            // 3. Double-check cache (might have been populated while acquiring lock)
            cached = redis.opsForValue().get(key);
            if (cached != null) {
                return cached;
            }

            // 4. Fetch from database (only one thread does this)
            VideoMetadata metadata = videoRepository.findById(videoId);

            if (metadata != null) {
                // 5. Populate cache with TTL jitter (add ±10% random jitter)
                Duration baseTtl = Duration.ofHours(1);
                long jitter = (long)(baseTtl.getSeconds() * 0.1 * (Math.random() * 2 - 1));
                Duration ttl = baseTtl.plusSeconds(jitter);

                redis.opsForValue().set(key, metadata, ttl);
            }

            return metadata;

        } finally {
            if (acquired) {
                lockService.unlock(lockKey);
            }
        }
    }
}
```

**TTL Jitter Benefits:**

- Prevents simultaneous expiration of related cache entries
- Reduces cache stampede probability
- Smooths out cache refresh load over time

### Batch Aggregation

```java
@Service
public class ViewAggregationService {

    private final JdbcTemplate jdbcTemplate;
    private final RedisTemplate<String, Long> redis;

    /**
     * Flush Redis view counts to PostgreSQL
     * Runs every hour
     */
    @Scheduled(cron = "0 0 * * * *")
    public void flushViewCounts() {
        // Get all view count keys
        Set<String> keys = redis.keys("views:*");

        for (String key : keys) {
            String videoId = key.replace("views:", "");
            Long count = redis.opsForValue().getAndDelete(key);

            if (count != null && count > 0) {
                // Update PostgreSQL
                jdbcTemplate.update(
                    "UPDATE videos SET view_count = view_count + ? WHERE video_id = ?",
                    count, videoId
                );

                // Update hourly aggregates
                jdbcTemplate.update(
                    "INSERT INTO view_aggregates (video_id, hour_bucket, view_count) " +
                    "VALUES (?, date_trunc('hour', NOW()), ?) " +
                    "ON CONFLICT (video_id, hour_bucket) " +
                    "DO UPDATE SET view_count = view_aggregates.view_count + ?",
                    videoId, count, count
                );
            }
        }
    }

    /**
     * Get unique viewer count (approximate)
     */
    public long getUniqueViewers(String videoId) {
        String key = "unique_viewers:" + videoId;
        return redis.opsForHyperLogLog().size(key);
    }
}
```

### View Count Aggregation Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          VIEW COUNT AGGREGATION                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘

View Event                  Kafka                  Aggregator              Database
   │                          │                       │                       │
   │ 1. Video played 30s      │                       │                       │
   │ ────────────────────────>│                       │                       │
   │                          │                       │                       │
   │                          │ 2. Batch events       │                       │
   │                          │ ──────────────────────>                       │
   │                          │                       │                       │
   │                          │                       │ 3. Aggregate per video│
   │                          │                       │ (1-minute windows)    │
   │                          │                       │                       │
   │                          │                       │ 4. Update Redis       │
   │                          │                       │ ──────────────────────>
   │                          │                       │    INCRBY views:vid   │
   │                          │                       │                       │
   │                          │                       │ 5. Hourly: Flush to PG│
   │                          │                       │ ──────────────────────>
   │                          │                       │                       │

┌───────────────────────────────────────────────────────────────────────────────────┐
│  APPROXIMATE COUNTING (for very popular videos)                                    │
│                                                                                    │
│  Problem: Video goes viral, millions of views per minute                          │
│  Solution: HyperLogLog for unique viewers, sampling for total views               │
│                                                                                    │
│  // Count every view (approximate)                                                │
│  if (random() < 0.01) {  // 1% sampling                                          │
│      incrementViewCount(videoId, 100);  // Scale up                              │
│  }                                                                                │
│                                                                                    │
│  // Unique viewers (HyperLogLog)                                                  │
│  PFADD unique_viewers:{video_id} {user_id}                                       │
│  PFCOUNT unique_viewers:{video_id}  // Returns approximate count                 │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Chunked Upload with Resume

### Upload Service

```java
@Service
public class ChunkedUploadService {

    private final S3Client s3Client;
    private final RedisTemplate<String, String> redis;

    private static final int CHUNK_SIZE = 10 * 1024 * 1024; // 10MB

    public UploadSession initiateUpload(String videoId, long fileSize) {
        // Calculate number of chunks
        int totalChunks = (int) Math.ceil((double) fileSize / CHUNK_SIZE);

        // Create multipart upload in S3
        CreateMultipartUploadRequest request = CreateMultipartUploadRequest.builder()
            .bucket("video-uploads")
            .key("originals/" + videoId + "/original.mp4")
            .build();

        CreateMultipartUploadResponse response = s3Client.createMultipartUpload(request);

        // Store upload session in Redis
        UploadSession session = UploadSession.builder()
            .videoId(videoId)
            .uploadId(response.uploadId())
            .totalChunks(totalChunks)
            .uploadedChunks(new HashSet<>())
            .build();

        redis.opsForValue().set(
            "upload:" + videoId,
            serialize(session),
            Duration.ofHours(24)
        );

        return session;
    }

    public ChunkUploadResult uploadChunk(String videoId, int chunkNumber,
                                          byte[] data) {
        UploadSession session = getSession(videoId);

        // Upload part to S3
        UploadPartRequest request = UploadPartRequest.builder()
            .bucket("video-uploads")
            .key("originals/" + videoId + "/original.mp4")
            .uploadId(session.getUploadId())
            .partNumber(chunkNumber + 1) // S3 uses 1-based indexing
            .build();

        UploadPartResponse response = s3Client.uploadPart(request,
            RequestBody.fromBytes(data));

        // Track uploaded chunk
        session.getUploadedChunks().add(chunkNumber);
        session.getPartETags().put(chunkNumber, response.eTag());
        updateSession(session);

        // Check if complete
        if (session.getUploadedChunks().size() == session.getTotalChunks()) {
            completeUpload(session);
            return ChunkUploadResult.complete();
        }

        return ChunkUploadResult.inProgress(
            session.getUploadedChunks().size(),
            session.getTotalChunks()
        );
    }

    public UploadSession getResumeInfo(String videoId) {
        // Return which chunks are already uploaded
        return getSession(videoId);
    }

    private void completeUpload(UploadSession session) {
        // Complete multipart upload
        List<CompletedPart> parts = session.getPartETags().entrySet().stream()
            .sorted(Map.Entry.comparingByKey())
            .map(e -> CompletedPart.builder()
                .partNumber(e.getKey() + 1)
                .eTag(e.getValue())
                .build())
            .collect(Collectors.toList());

        CompleteMultipartUploadRequest request = CompleteMultipartUploadRequest.builder()
            .bucket("video-uploads")
            .key("originals/" + session.getVideoId() + "/original.mp4")
            .uploadId(session.getUploadId())
            .multipartUpload(CompletedMultipartUpload.builder().parts(parts).build())
            .build();

        s3Client.completeMultipartUpload(request);

        // Trigger processing
        triggerProcessing(session.getVideoId());

        // Clean up session
        redis.delete("upload:" + session.getVideoId());
    }
}
```

---

## Summary

| Component     | Technology/Algorithm          | Key Configuration            |
| ------------- | ----------------------------- | ---------------------------- |
| Transcoding   | FFmpeg with GPU acceleration  | H.264, 7 resolutions         |
| ABR           | Client-side, bandwidth-based  | Harmonic mean, buffer-aware  |
| CDN           | Multi-tier with origin shield | 85% edge hit rate target     |
| View counting | Redis + HyperLogLog + Kafka   | Hourly batch to PostgreSQL   |
| Upload        | S3 multipart, resumable       | 10MB chunks, 24h session TTL |
