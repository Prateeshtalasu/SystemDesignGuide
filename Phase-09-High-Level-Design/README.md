# 🌍 PHASE 9: HIGH-LEVEL DESIGN (Weeks 11-14)

**Goal**: Design production systems end-to-end

## HLD Systems (6 files each):
Each system gets:
- `01-problem-requirements.md`
- `02-capacity-estimation.md`
- `03-api-schema-design.md`
- `04-architecture-diagrams.md`
- `05-tech-deep-dives.md` (Redis, Kafka, DB specifics)
- `06-interview-grilling.md` (Q&A)

## Systems List:

1. **URL Shortener (TinyURL)**
   - Short URL generation (base62 encoding)
   - Redirection
   - Analytics
   - Custom URLs

2. **Typeahead / Autocomplete**
   - Trie-based search
   - Ranking algorithm
   - Caching strategy
   - Real-time updates

3. **Search Engine**
   - Web crawler
   - Indexing (inverted index)
   - Ranking (PageRank simplified)
   - Query processing

4. **News Feed (Facebook/Twitter)**
   - Fan-out on write vs fan-out on read
   - Timeline generation
   - Ranking algorithm
   - Hot vs cold data

5. **Recommendation System**
   - Collaborative filtering
   - Content-based filtering
   - Real-time personalization
   - A/B testing infrastructure

6. **Video Streaming (YouTube/Netflix)**
   - Video upload pipeline
   - Transcoding
   - CDN distribution
   - Adaptive bitrate streaming (HLS/DASH)
   - View count aggregation

7. **Google Maps / Location Services**
   - Geospatial indexing (QuadTree, Geohash)
   - Routing algorithms (Dijkstra, A*)
   - ETA calculation
   - Real-time traffic

8. **Ride Hailing (Uber/Lyft)**
   - Driver-rider matching
   - Real-time location tracking (WebSocket)
   - Surge pricing
   - ETA calculation
   - Payment processing

9. **Food Delivery (DoorDash/UberEats)**
   - Restaurant discovery
   - Order placement
   - Delivery assignment
   - Real-time tracking
   - Inventory management

10. **File Storage (Google Drive/Dropbox)**
    - File upload (chunking)
    - Deduplication
    - Sync across devices
    - Sharing & permissions
    - Version control

11. **Notification System**
    - Multi-channel (push, email, SMS)
    - Priority queue
    - Rate limiting per user
    - Template management
    - Delivery guarantees

12. **Distributed Message Queue (Kafka-like)**
    - Topic partitioning
    - Replication
    - Consumer groups
    - Offset management
    - Exactly-once semantics

13. **Payment System (Stripe-like)**
    - Payment processing
    - Idempotency
    - Reconciliation
    - Fraud detection
    - Ledger system (double-entry bookkeeping)

14. **Distributed Rate Limiter**
    - Token bucket (distributed)
    - Sliding window (Redis)
    - Per-user, per-IP, per-API limits
    - Synchronization across servers

15. **Distributed Cache**
    - Consistent hashing
    - Replication
    - Cache invalidation
    - Eviction policies
    - Cache stampede prevention

16. **Chat System (WhatsApp/Slack)**
    - One-on-one messaging
    - Group chat
    - Online status
    - Read receipts
    - Message sync across devices
    - WebSocket vs polling

17. **E-commerce Checkout**
    - Cart management
    - Inventory reservation
    - Payment processing
    - Order fulfillment
    - Saga pattern for distributed transactions

18. **Distributed Job Scheduler (Cron at scale)**
    - Job submission
    - Scheduling algorithms
    - Execution (worker pools)
    - Retry logic
    - Job dependencies (DAG)

19. **Distributed Tracing System**
    - Trace collection
    - Span storage
    - Trace visualization
    - Sampling strategies
    - Performance overhead

20. **Monitoring & Alerting System**
    - Metrics collection
    - Time-series storage
    - Alerting rules
    - Dashboard creation
    - Anomaly detection

21. **Logging System (Centralized)**
    - Log aggregation
    - Log storage (time-series)
    - Log search and querying
    - Log retention policies
    - Real-time log streaming

22. **Analytics System**
    - Event tracking
    - Data pipeline (ETL)
    - Data warehouse design
    - Real-time vs batch analytics
    - A/B testing infrastructure

23. **Distributed Configuration Management**
    - Configuration storage
    - Dynamic updates
    - Versioning
    - Rollback mechanisms
    - Feature flags

## Deliverable
Can design any system in 45-60 minute interview

