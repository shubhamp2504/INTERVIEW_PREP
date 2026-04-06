# 🏢 Atlassian — Interview Experience (Senior SWE, 45 LPA)

> Complete 5-round interview experience at Atlassian. Position: Senior Software Engineer.

---

## Q1. What Are the Potential Issues with Consistent Hashing for Music Streaming Servers?

### 📝 One-Liner
Consistent hashing can suffer from hotspots, uneven distribution, and cascading failures in streaming workloads.

### 🔑 Quick Answer
Key issues: non-uniform distribution (some servers get more traffic), hotspots from popular content, cascading load on failover, and rebalancing complexity. Virtual nodes help but add memory overhead. *(consistent hashing mein load equally distribute nahi hota, popular content hotspot banata hai)*

### 📖 How It Works
**Problems**:
1. **Uneven Distribution**: Without virtual nodes, some servers get larger hash ranges
2. **Hotspots**: Popular songs/playlists concentrate requests on specific nodes
3. **Cascading Failure**: When a node dies, its load shifts to ONE neighbor *(ek node fail hota hai toh saara load next node pe aa jaata hai)*
4. **Rebalancing**: New server addition moves data from only one neighbor
5. **Stateful Connections**: Streaming sessions break on remap

**Solutions**:
- Virtual nodes (vnodes): 100-200 per physical node for better distribution
- Bounded-load consistent hashing (Google): cap max load per node
- Separate caching layer for hot content
- Session stickiness with graceful migration

### 🗣️ Answering Approach
"The main issue is non-uniform load distribution. In music streaming, some songs are orders of magnitude more popular, creating hotspots. When a node fails, its entire load cascades to one neighbor. I'd mitigate this with virtual nodes for better distribution, a bounded-load variant to cap per-node traffic, and a separate CDN layer for popular content."

### ⚡ Remember
- Virtual nodes: solve distribution, add memory overhead
- Bounded-load: Google's 2017 paper — cap at (1+ε) × average
- Hot content: separate CDN/cache layer, not hash ring
- Failover: replicate popular data to multiple nodes

---

## Q2. Pre-Loaded Hints vs Server-Loaded Hints — Pros & Cons

### 📝 One-Liner
Pre-loaded hints are bundled in the app, server-loaded hints are fetched from the backend at runtime.

### 🔑 Quick Answer
Pre-loaded: fast, offline-capable, but stale and increases app size. Server-loaded: always fresh, personalized, but requires network and has latency. *(pre-loaded: fast but purana data, server-loaded: fresh but network chahiye)*

### 📖 How It Works

| Aspect | Pre-Loaded | Server-Loaded |
|--------|-----------|--------------|
| Speed | Instant (local) | Network latency |
| Freshness | Stale until app update | Always current |
| Offline | ✅ Works | ❌ Fails |
| App Size | Larger (data bundled) | Smaller |
| Personalization | Generic | Per-user targeting |
| Update Frequency | App release cycle | Real-time |
| Cost | No server cost | API + compute cost |

**Hybrid Approach** (Best):
- Pre-load defaults → overlay with server-loaded personalized hints
- Cache server hints locally with TTL *(default pre-load karo, server se fresh layover karo)*

### 🗣️ Answering Approach
"I'd use a hybrid approach. Ship default hints in the app for immediate display and offline support. On app launch, fetch personalized hints from the server and overlay them. Cache server hints locally with a TTL. This gives us best-of-both: instant display, always fresh when online."

### ⚡ Remember
- Hybrid = best of both worlds
- Pre-load for critical path (no network dependency)
- Server-load for personalization and A/B testing
- Local cache with TTL bridges the gap

---

## Q3. How Would You Process a File Larger Than Available RAM?

### 📝 One-Liner
Stream the file in chunks instead of loading it entirely into memory.

### 🔑 Quick Answer
Use buffered reading (line-by-line or chunk-by-chunk), external sort for sorting, and MapReduce for aggregation. Never load the full file at once. *(poora file memory mein mat daalo, chhote chhote chunks mein process karo)*

### 📖 How It Works
1. **Streaming Read**: `BufferedReader` reads line-by-line — constant memory
2. **External Sort**: Split file → sort chunks in memory → merge sorted chunks *(chhote parts sort karo, phir merge karo)*
3. **MapReduce**: Map each chunk → reduce/aggregate results
4. **Memory-Mapped Files**: `MappedByteBuffer` — OS manages paging
5. **Parallel Processing**: Split file by byte ranges → process in parallel threads

### 💻 Code
```java
// Line-by-line streaming (constant memory)
try (BufferedReader br = new BufferedReader(new FileReader("huge.csv"))) {
    String line;
    while ((line = br.readLine()) != null) {
        process(line);
    }
}

// Java NIO - chunked reading
try (FileChannel channel = FileChannel.open(path)) {
    ByteBuffer buffer = ByteBuffer.allocate(64 * 1024); // 64KB chunks
    while (channel.read(buffer) > 0) {
        buffer.flip();
        processChunk(buffer);
        buffer.clear();
    }
}

// External sort: split into sorted chunks, then merge
public void externalSort(String file, int chunkSize) {
    List<File> chunks = splitAndSort(file, chunkSize);
    mergeFiles(chunks, "output.txt"); // k-way merge with PriorityQueue
}
```

### ⚡ Remember
- BufferedReader: simplest, line-by-line, constant memory
- External sort: for sorting files larger than RAM
- Memory-mapped: OS handles paging, good for random access
- Split by byte position for parallel processing

---

## Q4. Sports News Classification Service — Resource Estimation

### 📝 One-Liner
Estimate compute, storage, and network resources for a service that downloads articles and applies ML bias detection.

### 🔑 Quick Answer
Key inputs needed: article volume, ML model size, inference latency, storage retention, and SLA. Then calculate: CPU/GPU for inference, storage for articles + models, bandwidth for downloads. *(volume pata karo, model size pata karo, phir resources calculate karo)*

### 📖 How It Works
**Information needed**:
1. **Volume**: Articles/day? (e.g., 100K articles/day)
2. **Article Size**: Avg text size? (e.g., 5KB)
3. **ML Model**: Size? CPU or GPU inference? Latency per article?
4. **SLA**: Processing latency requirement? (real-time or batch?)
5. **Retention**: How long to store results?
6. **Throughput**: Peak vs average load?

**Estimation Example** (100K articles/day):
- Download: 100K × 5KB = 500MB/day bandwidth
- ML Inference: If 100ms/article on GPU → 100K × 100ms = ~2.8 hours → 1 GPU can handle
- Storage: 500MB/day × 365 = 182GB/year
- If real-time (<1s SLA): need parallel GPU workers *(real-time chahiye toh zyada GPUs lagenge)*

### 🗣️ Answering Approach
"First, I'd ask clarifying questions: expected article volume, latency requirements, and the ML model's resource needs. For back-of-envelope: if we process 100K articles/day with 100ms inference time, one GPU handles it. I'd add a queue for buffering, auto-scaling workers for peak loads, and separate storage tiers for raw articles vs processed results."

### ⚡ Remember
- Always ask: volume, latency SLA, model requirements
- Back-of-envelope: articles/sec × inference_time = compute needed
- Separate concerns: download, process, store, serve
- GPU for ML inference, CPU for text preprocessing

---

## Q5. Expanding Application to Multiple Countries — Backend Changes

### 📝 One-Liner
Multi-region deployment requires changes in data residency, localization, compliance, and latency optimization.

### 🔑 Quick Answer
Key changes: data residency (keep data in-country), multi-region deployment, i18n/l10n, currency/timezone handling, compliance (GDPR/local laws), CDN for latency. *(data us desh mein rakho, localize karo, compliance follow karo)*

### 📖 How It Works
1. **Data Residency**: Deploy DB in each country/region (GDPR, local laws) *(user ka data usi country mein rehna chahiye)*
2. **Multi-Region Deployment**: API servers in each region, global load balancer
3. **Localization**: i18n for text, l10n for formats (date, currency, numbers)
4. **Currency**: Multi-currency support, exchange rates, tax calculations
5. **Timezone**: Store in UTC, convert for display
6. **Compliance**: GDPR (EU), CCPA (California), PDPA (India), etc.
7. **Payment**: Local payment methods per country (UPI in India, iDEAL in Netherlands)

### 🗣️ Answering Approach
"I'd break this into infrastructure and application changes. Infra: multi-region deployments with data residency compliance — user data stays in-country. Application: localization framework for text, currency, and date formats. Cross-cutting: timezone handling in UTC, local payment integrations, and compliance with regional regulations."

### ⚡ Remember
- Store timestamps in UTC always
- Data residency: legal requirement in many countries
- Feature flags: enable features per-region
- Local payment methods critical for adoption

---

## Q6. Find Word from Jumbled String Characters (Coding)

### 📝 One-Liner
Given a list of words and a jumbled string, find which word can be formed from the string's characters.

### 🔑 Quick Answer
Count character frequencies of the jumbled string, then check each word — if all chars available in sufficient frequency, it's a match. *(jumbled string ka character count karo, word ke characters available hain toh match)*

### 📖 How It Works
- Build frequency map of the jumbled string
- For each word: check if every character in the word exists in the map with sufficient count
- If all characters satisfied → return the word *(sab characters mil gaye toh word mil gaya)*
- Time: O(W × L) where W = number of words, L = avg word length

### 💻 Code
```java
public static String find(String[] words, String jumbled) {
    int[] available = new int[26];
    for (char c : jumbled.toCharArray()) available[c - 'a']++;
    
    for (String word : words) {
        if (canForm(word, available.clone())) return word;
    }
    return "-";
}

private static boolean canForm(String word, int[] available) {
    for (char c : word.toCharArray()) {
        if (--available[c - 'a'] < 0) return false;
    }
    return true;
}

// Test: find(["baby","cat","dada","dog"], "ctay") → "cat"
// Test: find(["baby","cat","dada","dog"], "dad") → "-"
```

### ⚡ Remember
- Clone the freq array for each word check (don't modify shared state)
- "dad" → "-" because "dada" needs 2 d's and 2 a's, "dad" string only has 1 a
- For large word lists: sort words by length (shorter first for early match)

---

## Q7. File Collection Report — Total Size & Top N

### 📝 One-Liner
Each file has a collectionId — generate report showing total size and top N collections by total file size.

### 🔑 Quick Answer
Group files by collectionId → sum sizes per collection → sort by total descending → take top N. *(collection ke hisaab se group karo, size jodo, sort karo, top N nikalo)*

### 💻 Code
```java
// File: {name, size, collectionId}
record FileInfo(String name, long size, String collectionId) {}

public static void generateReport(List<FileInfo> files, int topN) {
    // Total size
    long totalSize = files.stream().mapToLong(FileInfo::size).sum();
    System.out.println("Total size: " + totalSize);
    
    // Top N collections by total size
    Map<String, Long> collectionSizes = files.stream()
        .collect(Collectors.groupingBy(
            FileInfo::collectionId, 
            Collectors.summingLong(FileInfo::size)));
    
    collectionSizes.entrySet().stream()
        .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
        .limit(topN)
        .forEach(e -> System.out.println(e.getKey() + ": " + e.getValue()));
}
```

### 📖 Follow-up: Multiple Collections Per File
```java
// Modified: file can belong to multiple collections
record FileInfo(String name, long size, Set<String> collectionIds) {}

Map<String, Long> collectionSizes = files.stream()
    .flatMap(f -> f.collectionIds().stream()
        .map(cid -> Map.entry(cid, f.size())))
    .collect(Collectors.groupingBy(
        Map.Entry::getKey, 
        Collectors.summingLong(Map.Entry::getValue)));
```

### 📖 Follow-up: Multithreaded Environment
```java
// Thread-safe with ConcurrentHashMap
ConcurrentMap<String, LongAdder> collectionSizes = new ConcurrentHashMap<>();
files.parallelStream().forEach(f ->
    f.collectionIds().forEach(cid ->
        collectionSizes.computeIfAbsent(cid, k -> new LongAdder()).add(f.size())
    )
);
```

### ⚡ Remember
- `groupingBy` + `summingLong` = most readable approach
- `flatMap` for many-to-many relationships
- `LongAdder` > `AtomicLong` for high contention
- `parallelStream` for CPU-bound operations on large datasets

---

## Q8. Design a Rate Limiter

### 📝 One-Liner
Control the number of requests a client can make in a given time window.

### 🔑 Quick Answer
Common algorithms: Token Bucket (smooth traffic), Sliding Window (precise), Fixed Window (simple). Store counters in Redis for distributed rate limiting. *(requests ko limit karo — time window mein kitne allow)*

### 📖 How It Works

**Token Bucket** (most popular):
- Bucket fills at constant rate (e.g., 10 tokens/sec)
- Each request consumes 1 token
- If empty → reject (429 Too Many Requests)
- Allows bursts up to bucket capacity *(bucket bhar jaaye toh burst allow)*

**Sliding Window Log**:
- Store timestamp of each request
- Count requests in last N seconds
- Precise but memory-heavy

**Sliding Window Counter** (hybrid):
- Weighted count from current + previous window
- Good balance of precision and memory

### 💻 Code
```java
// Token Bucket
public class TokenBucketRateLimiter {
    private final int maxTokens;
    private final double refillRate; // tokens/sec
    private double tokens;
    private long lastRefillTime;
    
    public synchronized boolean tryAcquire() {
        refill();
        if (tokens >= 1) {
            tokens -= 1;
            return true;
        }
        return false;
    }
    
    private void refill() {
        long now = System.nanoTime();
        double elapsed = (now - lastRefillTime) / 1e9;
        tokens = Math.min(maxTokens, tokens + elapsed * refillRate);
        lastRefillTime = now;
    }
}
```

### 📖 Follow-up: Credit-Based Model (Unused Requests Carry Over)
```java
// Modify: unused tokens from window N carry over to N+1
// Simply increase maxTokens to allow accumulation:
// maxTokens = baseRate * maxCarryoverWindows
// e.g., 100 req/min with up to 3 min carryover → maxTokens = 300
```

### 📖 Follow-up: Multithreaded Rate Limiter
```java
// Use AtomicReference + CAS for lock-free implementation
public class LockFreeRateLimiter {
    private final AtomicReference<double[]> state; // [tokens, lastRefillNanos]
    
    public boolean tryAcquire() {
        while (true) {
            double[] current = state.get();
            double[] updated = refill(current);
            if (updated[0] < 1) return false;
            updated[0] -= 1;
            if (state.compareAndSet(current, updated)) return true;
        }
    }
}
```

### ⚡ Remember
- Token bucket: allows bursts, smooth overall rate
- Distributed: Redis INCR + EXPIRE for shared counter
- 429 status code: include `Retry-After` header
- Credit carryover: just increase bucket capacity

---

## Q9. Design a Web Scraper System

### 📝 One-Liner
A system that recursively crawls URLs, extracts nested links and images, with depth limits and fault tolerance.

### 🔑 Quick Answer
BFS crawler with URL frontier queue, worker pool for parallel fetching, seen-URL set for dedup, depth tracking, and retry logic. *(BFS se crawl karo, queue mein URLs daalo, dedup karo, depth limit lagao)*

### 📖 How It Works
1. **URL Frontier**: Priority queue of URLs to crawl (seed URLs initially)
2. **Worker Pool**: Thread pool fetches URLs in parallel
3. **Parser**: Extract nested URLs + image links from HTML
4. **Dedup**: Bloom filter or HashSet for seen URLs *(already crawl ho chuka URL skip karo)*
5. **Depth Limit**: Track depth per URL, stop at max depth
6. **Politeness**: Per-domain rate limiting, respect robots.txt
7. **Fault Tolerance**: Retry with backoff, DLQ for persistent failures
8. **Output**: Map of URL → [image links]

### 💻 Code
```java
public class WebScraper {
    private final Set<String> visited = ConcurrentHashMap.newKeySet();
    private final int maxDepth;
    private final ExecutorService pool;
    
    public Map<String, List<String>> scrape(List<String> seedUrls) {
        ConcurrentMap<String, List<String>> results = new ConcurrentHashMap<>();
        BlockingQueue<UrlTask> queue = new LinkedBlockingQueue<>();
        
        seedUrls.forEach(url -> queue.offer(new UrlTask(url, 0)));
        
        while (!queue.isEmpty()) {
            UrlTask task = queue.poll();
            if (task.depth > maxDepth || !visited.add(task.url)) continue;
            
            pool.submit(() -> {
                try {
                    Document doc = fetchAndParse(task.url);
                    List<String> images = extractImages(doc);
                    results.put(task.url, images);
                    
                    extractLinks(doc).forEach(link ->
                        queue.offer(new UrlTask(link, task.depth + 1)));
                } catch (Exception e) {
                    retryQueue.offer(task); // fault tolerance
                }
            });
        }
        return results;
    }
    
    record UrlTask(String url, int depth) {}
}
```

### ⚡ Remember
- BFS with depth tracking: prevents infinite crawling
- Bloom filter: memory-efficient dedup for millions of URLs
- robots.txt: always respect, legal requirement
- Politeness: rate limit per domain (1 req/sec typical)
- Retry: exponential backoff, max 3 retries

---

## Q10. Handling a Project with Vague Requirements (Managerial)

### 📝 One-Liner
Demonstrate how you scope, clarify, and deliver when requirements are unclear.

### 🔑 Quick Answer
Ask clarifying questions, create an MVP scope, get stakeholder alignment, iterate incrementally. *(requirements unclear hain toh questions pucho, MVP banao, feedback lo, iterate karo)*

### 📖 How It Works
1. **Identify Gaps**: List what's known vs unknown
2. **Stakeholder Meetings**: Ask specific questions to narrow scope
3. **MVP Definition**: Define smallest deliverable that provides value
4. **Prototype**: Build a quick prototype for feedback *(chhota sa prototype banao, dikhao, feedback lo)*
5. **Iterate**: Ship MVP → gather feedback → refine → repeat
6. **Document**: Write decisions and assumptions as you go

### 🗣️ Answering Approach
"In my previous project, we received a requirement to 'improve user engagement' with no specific metrics. I organized a workshop with product and design to define what engagement means — we agreed on DAU and session duration. I proposed a 2-week MVP with push notifications and in-app tips. After the first sprint, we had data to guide the next iteration."

### ⚡ Remember
- Show initiative: don't wait for perfect requirements
- MVP mindset: deliver small, learn fast
- Document assumptions: protects you when scope changes
- Communication: keep stakeholders in the loop

---

## Q11. Helping a Team Member Grow Technically (Managerial)

### 📝 One-Liner
Demonstrate mentorship and leadership by describing how you helped someone develop their skills.

### 🔑 Quick Answer
Identify skill gap, create a learning plan, pair program, give stretch assignments, regular feedback. *(skill gap pehchano, plan banao, saath kaam karo, feedback do)*

### 📖 How It Works
1. **Identify**: One-on-one conversation to understand their goals and gaps
2. **Plan**: Create a structured learning path (books, courses, hands-on tasks)
3. **Pair Programming**: Work together on complex problems
4. **Stretch Assignments**: Assign tasks slightly above their level
5. **Code Reviews**: Detailed reviews with explanations, not just approvals
6. **Feedback Loop**: Regular check-ins on progress

### 🗣️ Answering Approach
"A junior developer on my team was strong in coding but struggled with system design. I created a 6-week plan: first 2 weeks we did book study on Designing Data-Intensive Applications, then I assigned them to lead the design of a cache service with me as a reviewer. I paired with them on the initial architecture, then let them drive. By the end, they presented the design to the team and implemented it. Six months later, they were confidently designing features independently."

### ⚡ Remember
- Be specific: name the skill, the plan, and the outcome
- Show patience and investment of YOUR time
- Quantify impact: "6 months later they could..."
- Framework: Identify → Plan → Support → Stretch → Reflect

---
