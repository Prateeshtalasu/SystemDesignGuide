# 🌍 PHASE 9: HIGH-LEVEL DESIGN (Weeks 11-14)

**Goal**: Design production systems end-to-end

**Learning Objectives**:
- Design systems that scale to millions of users
- Make and defend architectural trade-offs
- Deep dive into any component when asked
- Complete a design in 45-60 minutes

**Estimated Time**: 60-80 hours

---

## HLD Systems (7 files each):
Each system gets:
- `01-problem-requirements.md` (STEP 1: Requirements & Scope)
- `02-capacity-estimation.md` (STEP 2: Back-of-Envelope Calculations)
- `03-api-schema-design.md` (STEP 3: API Design)
- `04-data-model-architecture.md` (STEP 4-5: Data Model, Storage & Architecture)
- `05A-production-deep-dives-core.md` (STEP 6-8: Async/Messaging, Caching, Search)
- `05B-production-deep-dives-ops.md` (STEP 9-13: Scaling, Monitoring, Security, Simulation, Cost)
- `06-interview-grilling.md` (STEP 14-15: Tradeoffs, Interview Q&A, Failure Scenarios)

---

## Systems List:

### URL & Search Systems

1. **URL Shortener (TinyURL)**
   - Short URL generation (base62 encoding)
   - Redirection
   - Analytics (click tracking)
   - Custom URLs
   - Expiration

2. **Pastebin**
   - Text storage and sharing
   - Expiration policies
   - Access controls (public, private, unlisted)
   - Syntax highlighting
   - Rate limiting

3. **Typeahead / Autocomplete**
   - Trie-based search
   - Ranking algorithm
   - Caching strategy
   - Real-time updates
   - Personalization

4. **Search Engine**
   - Web crawler
   - Indexing (inverted index)
   - Ranking (PageRank simplified)
   - Query processing
   - Spell correction

5. **Web Crawler**
   - URL frontier
   - Politeness (robots.txt)
   - Duplicate detection
   - Distributed crawling
   - Content extraction

### Social Media & Feeds

6. **News Feed (Facebook/Twitter)**
   - Fan-out on write vs fan-out on read
   - Timeline generation
   - Ranking algorithm
   - Hot vs cold data
   - Celebrity problem

7. **Recommendation System**
   - Collaborative filtering
   - Content-based filtering
   - Real-time personalization
   - A/B testing infrastructure
   - Cold start problem

8. **Chat System (WhatsApp/Slack)**
   - One-on-one messaging
   - Group chat
   - Online status
   - Read receipts
   - Message sync across devices
   - WebSocket vs polling
   - End-to-end encryption

### Media & Streaming

9. **Video Streaming (YouTube/Netflix)**
   - Video upload pipeline
   - Transcoding
   - CDN distribution
   - Adaptive bitrate streaming (HLS/DASH)
   - View count aggregation
   - Recommendations

10. **Instagram / Photo Sharing**
    - Photo upload and storage
    - Feed generation
    - Stories feature
    - Image processing
    - CDN for images

### Location & Maps

11. **Google Maps / Location Services**
    - Geospatial indexing (QuadTree, Geohash)
    - Routing algorithms (Dijkstra, A*)
    - ETA calculation
    - Real-time traffic
    - Map tile serving

12. **Ride Hailing (Uber/Lyft)**
    - Driver-rider matching
    - Real-time location tracking (WebSocket)
    - Surge pricing
    - ETA calculation
    - Payment processing
    - Rating system

13. **Food Delivery (DoorDash/UberEats)**
    - Restaurant discovery
    - Order placement
    - Delivery assignment
    - Real-time tracking
    - Inventory management
    - Estimated delivery time

### Storage & Files

14. **File Storage (Google Drive/Dropbox)**
    - File upload (chunking, resumable)
    - Deduplication
    - Sync across devices
    - Sharing & permissions
    - Version control
    - Conflict resolution

15. **Collaborative Document Editor (Google Docs)**
    - Real-time collaboration
    - Operational Transformation (OT)
    - CRDTs for conflict resolution
    - Cursor presence
    - Version history
    - Offline support

### Infrastructure Systems

16. **Notification System**
    - Multi-channel (push, email, SMS)
    - Priority queue
    - Rate limiting per user
    - Template management
    - Delivery guarantees
    - Preference management

17. **Distributed Message Queue (Kafka-like)**
    - Topic partitioning
    - Replication
    - Consumer groups
    - Offset management
    - Exactly-once semantics

18. **Payment System (Stripe-like)**
    - Payment processing
    - Idempotency
    - Reconciliation
    - Fraud detection
    - Ledger system (double-entry bookkeeping)
    - PCI compliance

19. **Distributed Rate Limiter**
    - Token bucket (distributed)
    - Sliding window (Redis)
    - Per-user, per-IP, per-API limits
    - Synchronization across servers
    - Rate limit headers

20. **Distributed Cache**
    - Consistent hashing
    - Replication
    - Cache invalidation
    - Eviction policies
    - Cache stampede prevention

### E-commerce

21. **E-commerce Checkout**
    - Cart management
    - Inventory reservation
    - Payment processing
    - Order fulfillment
    - Saga pattern for distributed transactions

22. **Ticket Booking System (BookMyShow)**
    - Seat selection and locking
    - Concurrent booking handling
    - Payment integration
    - Booking confirmation
    - Waitlist management

### DevOps & Monitoring

23. **Distributed Job Scheduler (Cron at scale)**
    - Job submission
    - Scheduling algorithms
    - Execution (worker pools)
    - Retry logic
    - Job dependencies (DAG)

24. **Distributed Tracing System**
    - Trace collection
    - Span storage
    - Trace visualization
    - Sampling strategies
    - Performance overhead

25. **Monitoring & Alerting System**
    - Metrics collection
    - Time-series storage
    - Alerting rules
    - Dashboard creation
    - Anomaly detection

26. **Logging System (Centralized)**
    - Log aggregation
    - Log storage (time-series)
    - Log search and querying
    - Log retention policies
    - Real-time log streaming

27. **Analytics System**
    - Event tracking
    - Data pipeline (ETL)
    - Data warehouse design
    - Real-time vs batch analytics
    - A/B testing infrastructure

28. **Distributed Configuration Management**
    - Configuration storage
    - Dynamic updates
    - Versioning
    - Rollback mechanisms
    - Feature flags

### Key-Value & Database

29. **Key-Value Store (Redis-like)**
    - Data structures
    - Persistence
    - Replication
    - Clustering
    - Memory management

---

## Design Framework:
For each system, follow this structure:
1. **Requirements** (5 min): Functional + Non-functional
2. **Capacity Estimation** (5 min): Traffic, storage, bandwidth
3. **API Design** (5 min): Core endpoints
4. **High-Level Architecture** (15 min): Components and data flow
5. **Deep Dive** (15 min): Pick 1-2 components
6. **Trade-offs** (5 min): Alternatives and decisions

---

## Practice Tips:
1. **Use a timer**: 45-60 minutes per design
2. **Draw diagrams**: Architecture, sequence, data flow
3. **Explain trade-offs**: Not just "it depends"
4. **Anticipate follow-ups**: "What if this fails?"
5. **Know your numbers**: QPS, storage, latency

---

## Common Interview Follow-ups:
- "How would you scale this 10x?"
- "What happens when X fails?"
- "How do you ensure consistency?"
- "What are the bottlenecks?"
- "How would you monitor this?"

---

## Deliverable
Can design any system in 45-60 minute interview
