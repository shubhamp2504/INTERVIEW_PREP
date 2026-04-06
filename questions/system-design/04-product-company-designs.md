# 🏗️ System Design — Product Company Interview Problems

> Classic system design questions from Stripe, Netflix, Uber, Cloudflare, Reddit, Miro & other US product companies.

---

## Q1. Design a Video Streaming Platform (Netflix)

### 📝 One-Liner
Design a system for uploading, processing, storing, and streaming video content to millions of users.

### 🔑 Quick Answer
Video ingestion → transcoding pipeline → CDN distribution → adaptive bitrate streaming. Key concerns: storage (petabytes), CDN caching, ABR for variable bandwidth. *(video upload hoga, transcode hoga multiple quality mein, CDN se deliver hoga)*

### 📖 How It Works
1. **Upload Service**: Accept video files, store originals in S3/blob storage
2. **Transcoding Pipeline**: Convert to multiple resolutions (240p–4K) + formats (HLS/DASH segments) using worker queues *(ek video ke 10+ versions bante hain)*
3. **Content Storage**: Transcoded segments in cloud storage, metadata in DB
4. **CDN**: Edge servers cache popular content close to users
5. **Streaming**: Client requests manifest file → ABR selects quality based on bandwidth
6. **Recommendation Engine**: ML models for personalized feeds

### 🗣️ Answering Approach
"I'd design three main pipelines: upload/ingest, transcoding, and streaming. Videos are uploaded to blob storage, a message queue triggers the transcoding service which produces multiple quality levels as HLS segments. These are pushed to CDN edge servers. The player uses adaptive bitrate streaming, switching quality based on network conditions."

### ⚠️ Pitfalls / Gotchas
- Transcoding is CPU-intensive — use auto-scaling worker pools
- CDN cache invalidation for updated content
- DRM for content protection *(piracy protection zaroori hai)*
- Cold start problem: new content not yet cached at edges

### ⚡ Remember
- HLS/DASH: adaptive bitrate protocols
- CDN: 90%+ of traffic served from edge
- Transcoding: async pipeline with queue
- Storage: hot/warm/cold tiers for cost

---

## Q2. Design a Ride-Sharing Backend (Uber)

### 📝 One-Liner
Real-time matching of riders with nearby drivers, including pricing, routing, and tracking.

### 🔑 Quick Answer
Geospatial indexing (QuadTree/Geohash) for nearby driver lookup, WebSocket for real-time tracking, surge pricing based on demand/supply ratio. *(location se nearby drivers dhundho, real-time track karo)*

### 📖 How It Works
1. **Location Service**: Drivers push GPS every 5s → stored in geospatial index
2. **Matching**: Rider requests ride → find N nearest drivers → dispatch to closest accepting driver
3. **Pricing**: Base fare + distance + time + surge multiplier (supply/demand)
4. **Trip Service**: Track trip state: REQUESTED → MATCHED → IN_PROGRESS → COMPLETED
5. **Payment**: Post-trip charge via payment service
6. **ETA Calculation**: Graph-based routing with real-time traffic data

### 🗣️ Answering Approach
"The core challenge is real-time geospatial matching. I'd use a geohash-based index to find nearby drivers. When a rider requests a ride, I query the index for drivers in expanding rings. The matching service dispatches to the optimal driver. WebSocket connections provide real-time location updates."

### ⚡ Remember
- QuadTree or Geohash for spatial indexing
- WebSocket for bi-directional real-time updates
- Surge pricing = demand/supply ratio per geo-cell
- Write-heavy: millions of location updates/sec

---

## Q3. Design a CDN (Cloudflare)

### 📝 One-Liner
A globally distributed network of servers that cache and serve content close to users for low latency.

### 🔑 Quick Answer
Edge PoPs worldwide → DNS-based routing to nearest PoP → cache static content → fallback to origin on cache miss. *(duniya bhar mein servers lagao, user ke paas wale se content do)*

### 📖 How It Works
1. **DNS Resolution**: Anycast/GeoDNS routes user to nearest PoP
2. **Edge Cache**: Check if content is cached locally → serve instantly
3. **Cache Miss**: Fetch from origin server → cache at edge → serve
4. **Cache Invalidation**: TTL-based, purge APIs, versioned URLs
5. **DDoS Protection**: Rate limiting, WAF at edge
6. **TLS Termination**: Edge handles SSL for faster handshake

### 🗣️ Answering Approach
"I'd design a multi-tier caching architecture. DNS routes users to the nearest PoP using anycast. The edge server checks its local cache — on a hit, it serves immediately. On a miss, it fetches from a regional cache or origin, then caches locally. Cache invalidation uses a combination of TTL and explicit purge."

### ⚡ Remember
- Anycast: single IP, multiple servers, routed by proximity
- Cache hierarchy: Edge → Regional → Origin
- Hot content: 95%+ cache hit ratio
- Invalidation: TTL + versioned URLs + purge API

---

## Q4. Design a Search System (Algolia)

### 📝 One-Liner
A low-latency, typo-tolerant search-as-you-type system for structured data.

### 🔑 Quick Answer
Inverted index with prefix matching, typo tolerance via edit distance, ranked by relevance score. Data replicated across regions for <50ms response. *(inverted index se fast search, typo support bhi)*

### 📖 How It Works
1. **Indexing**: Build inverted index from documents — tokenize, normalize, stem
2. **Query Processing**: Parse query → spell correction → prefix expansion → search index
3. **Ranking**: TF-IDF or BM25 + custom ranking rules (popularity, recency, geo)
4. **Typo Tolerance**: Levenshtein distance ≤ 2, pre-computed for common words
5. **Search-as-you-type**: Prefix matching on inverted index
6. **Faceting**: Pre-computed aggregations for filters

### ⚡ Remember
- Inverted index: word → [doc_ids] mapping
- BM25 ranking > TF-IDF
- Prefix trie for autocomplete
- Shard by document ID, replicate for read throughput

---

## Q5. Design a Real-Time Chat System

### 📝 One-Liner
A messaging system supporting 1:1 and group chats with real-time delivery, read receipts, and offline support.

### 🔑 Quick Answer
WebSocket for persistent connections, message queue for guaranteed delivery, Cassandra/ScyllaDB for message storage. *(WebSocket se real-time, queue se guarantee, NoSQL mein store)*

### 📖 How It Works
1. **Connection**: Client opens WebSocket to a chat gateway
2. **Send**: Message → gateway → message router → recipient's gateway → deliver
3. **Offline**: If recipient offline, store in persistent queue → deliver on reconnect
4. **Storage**: Messages in Cassandra (partition by chat_id, sorted by timestamp)
5. **Group Chat**: Fan-out to all group members
6. **Read Receipts**: Track last_read_timestamp per user per chat

### 🗣️ Answering Approach
"I'd use WebSocket for real-time bidirectional communication. Messages flow through a chat service to a message router that identifies the recipient's server. For offline users, messages queue until reconnection. I'd store messages in Cassandra partitioned by chat ID for efficient retrieval."

### ⚡ Remember
- WebSocket: persistent, bidirectional, low overhead
- Fan-out: small groups inline, large groups via queue
- Last-seen, typing indicators via lightweight heartbeats
- E2E encryption for security

---

## Q6. Design a URL Shortening Service

### 📝 One-Liner
Convert long URLs to short ones (e.g., bit.ly/abc123) and redirect users to the original URL.

### 🔑 Quick Answer
Generate unique short code (base62 encoding of unique ID), store mapping in DB + cache, 301/302 redirect on access. *(unique ID generate karo, base62 encode karo, redirect karo)*

### 📖 How It Works
1. **Shorten**: Receive long URL → generate unique ID (auto-increment or snowflake) → base62 encode → store mapping
2. **Redirect**: Short URL hit → look up in cache (Redis) → fallback to DB → 301 redirect
3. **Analytics**: Log clicks with metadata (timestamp, IP, user-agent)
4. **Expiry**: TTL-based cleanup for temporary links

### 💻 Code
```java
// Base62 encoding
private static final String CHARS = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
public static String encode(long id) {
    StringBuilder sb = new StringBuilder();
    while (id > 0) {
        sb.append(CHARS.charAt((int)(id % 62)));
        id /= 62;
    }
    return sb.reverse().toString();
}
```

### ⚡ Remember
- 62^7 = 3.5 trillion combinations (enough for years)
- Cache hot URLs in Redis: 99%+ read requests
- 301 (permanent) vs 302 (temporary) redirect — analytics needs 302
- Collision: use unique ID generation, not hash

---

## Q7. Design a Distributed Cache System

### 📝 One-Liner
In-memory key-value store distributed across nodes for sub-millisecond reads.

### 🔑 Quick Answer
Consistent hashing for data distribution, replication for availability, LRU eviction, write-through/write-behind for persistence. *(consistent hashing se distribute, replicate for safety)*

### 📖 How It Works
1. **Partitioning**: Consistent hashing distributes keys across nodes
2. **Replication**: Each key replicated to N next nodes on the ring
3. **Eviction**: LRU/LFU when memory full
4. **Cache Patterns**: Cache-aside, read-through, write-through, write-behind
5. **Client**: Hash key → route to correct node → read/write

### 🆚 vs. Comparison
| Pattern | Description |
|---------|------------|
| Cache-aside | App checks cache, fetches from DB on miss, populates cache |
| Read-through | Cache itself fetches from DB on miss |
| Write-through | Write to cache + DB synchronously |
| Write-behind | Write to cache, async write to DB |

### ⚡ Remember
- Consistent hashing: nodes join/leave with minimal redistribution
- Trade-off: memory cost vs latency improvement
- Cache stampede: use locking or probabilistic early refresh
- Redis Cluster: real-world implementation

---

## Q8. Design a Scalable Notification Service

### 📝 One-Liner
Multi-channel notification system supporting push, SMS, email at scale with delivery guarantees.

### 🔑 Quick Answer
Event-driven with priority queues, channel-specific workers, deduplication, and delivery tracking. *(event aata hai, queue mein jaata hai, channel ke hisaab se deliver hota hai)*

### 📖 How It Works
1. **Ingestion**: API accepts notification requests → validates → publishes to Kafka
2. **Router**: Reads events, routes to channel-specific topics (push/SMS/email)
3. **Workers**: Channel workers consume and deliver via providers (FCM, Twilio, SES)
4. **Dedup**: Redis-based dedup to prevent double-sends
5. **Tracking**: Delivery status tracked (sent, delivered, read, failed)
6. **Rate Limiting**: Per-user rate limits to avoid spam

### ⚡ Remember
- Kafka for durability + ordering
- Separate queues per channel (different latency/priority)
- Exponential backoff for retries
- User preferences: opt-in/opt-out per channel

---

## Q9. Design a Feed System (Reddit)

### 📝 One-Liner
A personalized content feed with ranking, pagination, and near-real-time updates.

### 🔑 Quick Answer
Fan-out on write for small followings, fan-out on read for celebrities. Ranking by score (upvotes, time, engagement). *(chhote users ka write-time fan-out, bade users ka read-time fan-out)*

### 📖 How It Works
1. **Post Creation**: Write post → store in DB → trigger feed updates
2. **Fan-out on Write**: For users with <10K followers, push post to all followers' feeds in cache
3. **Fan-out on Read**: For celebrities, merge their posts at read time
4. **Ranking**: Hot = f(upvotes, time). Wilson score for confidence
5. **Pagination**: Cursor-based (not offset) for stability
6. **Caching**: Pre-computed feeds in Redis sorted sets

### ⚡ Remember
- Hybrid fan-out: write for normal users, read for celebrities
- Cursor pagination: `?after=post_id` (not `?page=2`)
- Redis sorted set: score = ranking function
- Near real-time: eventual consistency acceptable

---

## Q10. Design a Collaborative Whiteboard (Miro)

### 📝 One-Liner
Real-time multi-user drawing/editing on a shared canvas with conflict resolution.

### 🔑 Quick Answer
WebSocket for real-time sync, CRDTs or OT for conflict resolution, canvas partitioned into tiles for efficiency. *(real-time sync WebSocket se, conflict CRDTs se solve)*

### 📖 How It Works
1. **Connection**: Users connect via WebSocket to room-specific servers
2. **Operations**: Each edit = operation (add shape, move, delete) → broadcast to room
3. **Conflict Resolution**: CRDTs (Conflict-free Replicated Data Types) for eventual consistency
4. **State**: Full board state stored, operations log for undo/redo
5. **Rendering**: Canvas divided into tiles, only load visible area
6. **Persistence**: Auto-save operations to DB, periodic snapshots

### ⚡ Remember
- CRDT: converges without coordination *(bina coordinator ke sab same state pe aate hain)*
- OT (Operational Transform): used by Google Docs
- WebSocket rooms: one server per active board
- Infinite canvas: tile-based loading for performance

---

## Q11. Design a Multi-Tenant SaaS Architecture

### 📝 One-Liner
A single application instance serving multiple isolated customer tenants.

### 🔑 Quick Answer
Three models: shared DB, schema-per-tenant, DB-per-tenant. Trade-off between isolation and resource efficiency. *(ek app, bahut saare customers — isolate kaise kare)*

### 📖 How It Works
| Model | Isolation | Cost | Complexity |
|-------|-----------|------|-----------|
| Shared table (tenant_id column) | Low | Low | Low |
| Schema per tenant | Medium | Medium | Medium |
| DB per tenant | High | High | High |

- **Routing**: Identify tenant from subdomain/header/JWT → set tenant context
- **Data isolation**: Row-level security or separate schemas
- **Resource limits**: Per-tenant quotas (API calls, storage)
- **Customization**: Feature flags per tenant

### ⚡ Remember
- Start shared → migrate heavy tenants to dedicated as they grow
- Always include `tenant_id` in WHERE clauses (RLS for safety)
- Shared infra = cost efficient, dedicated = better isolation
- GDPR: tenant data deletion must be clean

---

## Q12. Design a Log Aggregation & Monitoring System

### 📝 One-Liner
Collect, index, and search logs from thousands of services in near-real-time.

### 🔑 Quick Answer
Agents collect logs → Kafka for buffering → index in Elasticsearch → visualize in Grafana/Kibana. *(har server se collect karo, Kafka se flow karo, ES mein index karo)*

### 📖 How It Works
1. **Collection**: Agents (Fluentd/Filebeat) on each server ship logs
2. **Buffering**: Kafka absorbs spike traffic, decouples producers/consumers
3. **Processing**: Stream processing (Flink/Logstash) for enrichment, parsing
4. **Storage**: Elasticsearch for indexed search, S3 for long-term archive
5. **Querying**: Kibana/Grafana dashboards, alert rules
6. **Retention**: Hot (ES, 7 days) → Warm (30 days) → Cold (S3, years)

### ⚡ Remember
- ELK Stack: Elasticsearch + Logstash + Kibana
- Kafka: handles 100K+ events/sec buffering
- Structured logging (JSON) >>> unstructured
- Cost: storage is #1 concern — tiered retention

---

## Q13. Design a Search Autocomplete System

### 📝 One-Liner
Suggest top completions as the user types, with <100ms latency.

### 🔑 Quick Answer
Trie data structure storing popular query prefixes, ranked by frequency/recency. Precomputed top-K per prefix node. *(Trie mein popular queries store karo, har node pe top-K rakh lo)*

### 📖 How It Works
1. **Data Collection**: Log all search queries with frequency
2. **Trie Building**: Offline job builds trie, each node stores top-K completions
3. **Serving**: User types prefix → traverse trie → return pre-computed top-K
4. **Update**: Periodic rebuild (hourly/daily) or real-time with sampling
5. **Personalization**: Blend global top-K with user's recent searches

### ⚡ Remember
- Pre-compute top-K per node → O(1) lookup at serve time
- Trie fits in memory for most use cases (~10GB)
- Filter offensive/sensitive suggestions
- Debounce: don't query on every keystroke

---

## Q14. Design a Feature Flag System

### 📝 One-Liner
Toggle features on/off without code deployment, with targeting rules.

### 🔑 Quick Answer
Central config store with SDK for evaluation. Rules: percentage rollout, user targeting, environment-based. *(bina deploy kiye features on/off karo, percentage se rollout karo)*

### 📖 How It Works
1. **Flag Definition**: name, type (boolean/string/number), default value, targeting rules
2. **Rule Engine**: Evaluate user context against rules (userId %, region, role)
3. **SDK**: Client library caches flags, evaluates locally for speed
4. **Admin UI**: Create/edit flags, set rollout percentage, target users
5. **Audit Log**: Track who changed what, when
6. **Kill Switch**: Instantly disable any feature

### ⚡ Remember
- Local evaluation (cached) for <1ms latency
- Percentage rollout: hash(userId + flagKey) % 100
- Technical debt: clean up old flags!
- A/B testing: variant assignment via flags

---

## Q15. Design a Distributed Job Scheduling System

### 📝 One-Liner
Schedule and execute tasks reliably at specified times across a distributed cluster.

### 🔑 Quick Answer
Central scheduler with DB-backed job store, worker nodes pull jobs, idempotent execution with distributed locking. *(scheduler job assign karta hai, workers execute karte hain, lock se duplicate nahi hote)*

### 📖 How It Works
1. **Job Store**: DB table with job definition, schedule (cron), next_fire_time
2. **Scheduler**: Leader-elected scheduler scans for due jobs → pushes to queue
3. **Workers**: Pull from queue → execute → report status
4. **Locking**: Distributed lock (Redis/ZooKeeper) prevents duplicate execution
5. **Retry**: Failed jobs → exponential backoff retry queue
6. **Monitoring**: Track execution time, failures, SLA breaches

### ⚡ Remember
- Idempotent execution: same job safety across retries
- Leader election: only one scheduler instance active
- Cron parsing: Quartz-style expressions
- Scale: partition jobs by type/priority

---

## Q16. Design an API Gateway for Microservices

### 📝 One-Liner
Single entry point for all client requests that handles routing, auth, rate limiting, and aggregation.

### 🔑 Quick Answer
Reverse proxy that routes requests to backend services, handles cross-cutting concerns (auth, rate limiting, logging, transformation). *(sab requests gateway se jaati hain, waha pe auth, rate limit sab hota hai)*

### 📖 How It Works
1. **Routing**: Path-based routing to backend services (`/users` → user-service)
2. **Authentication**: Validate JWT/OAuth tokens at gateway level
3. **Rate Limiting**: Per-client rate limits (token bucket algorithm)
4. **Load Balancing**: Distribute across service instances
5. **Request Aggregation**: Combine multiple backend calls into one response
6. **Circuit Breaker**: Protect against cascading failures

### ⚡ Remember
- Examples: Kong, Spring Cloud Gateway, Netflix Zuul
- Keep gateway stateless for horizontal scaling
- Don't put business logic in gateway
- Health checks: gateway → backend health monitoring

---

## Q17. Design a Real-Time Analytics Pipeline

### 📝 One-Liner
Process and aggregate millions of events per second for dashboards and alerts.

### 🔑 Quick Answer
Event ingestion → stream processing → time-series aggregation → dashboards. Kafka + Flink/Spark Streaming + ClickHouse/Druid. *(events aate hain, stream mein process hote hain, aggregate hoke dashboard pe dikhte hain)*

### 📖 How It Works
1. **Ingestion**: Client SDKs send events → Kafka topics
2. **Stream Processing**: Flink/Spark Streaming computes windowed aggregations
3. **Storage**: Pre-aggregated metrics in time-series DB (ClickHouse, Druid, TimescaleDB)
4. **Query Layer**: API for dashboards — query pre-aggregated tables
5. **Alerting**: Threshold rules on streaming data → trigger alerts
6. **Late Data**: Watermarks + allowed lateness for eventual consistency

### ⚡ Remember
- Lambda architecture: batch + stream layers
- Kappa architecture: stream-only (simpler)
- Windowing: tumbling, sliding, session windows
- Pre-aggregate at write time → fast reads

---
