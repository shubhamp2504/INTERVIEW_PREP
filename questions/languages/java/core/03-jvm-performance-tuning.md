# ☕ Java Core — JVM Performance Tuning (Q4)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Interview Script → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q4"></a>
## Q4. What strategies do you use to optimize JVM performance, including garbage collection tuning and memory management?

### 📝 One-Liner
Profile first (don't guess) → right-size heap (-Xmx) → choose GC algorithm (G1 for balanced, ZGC for low-latency) → tune GC flags → minimize object allocation → monitor with GC logs + JFR.

### 🔑 Quick Answer
**Step 1: Profile** — use JFR (Java Flight Recorder) + GC logs to identify bottlenecks. **Step 2: Right-size heap** — `-Xms` = `-Xmx` (avoid resize overhead). Container-aware: `-XX:MaxRAMPercentage=75.0`. **Step 3: Choose GC** — **G1GC** (default, balanced throughput + pause time, best for most apps), **ZGC** (sub-millisecond pauses for latency-critical apps), **Shenandoah** (similar to ZGC, Red Hat), **Parallel GC** (max throughput for batch/offline). **Step 4: Tune** — G1: `-XX:MaxGCPauseMillis=200` (target pause), `-XX:G1HeapRegionSize`. ZGC: `-XX:+UseZGC` (minimal tuning needed). **Step 5: Optimize code** — reduce object churn (reuse, pools, primitive streams), avoid finalizers, use off-heap for large data (ByteBuffer). **Monitor**: GC logs (`-Xlog:gc*`), JVisualVM, Grafana+Prometheus (Micrometer JVM metrics). *(Pehle profile karo — phir heap size, GC algorithm, aur tuning flags adjust karo)*

### 📖 How It Works
```
JVM Memory Layout:
  ┌──────────────────────────────────────────────────┐
  │                    JVM Process                    │
  ├──────────────────────────────────────────────────┤
  │ HEAP (-Xmx)                                      │
  │ ┌──────────────────────┬───────────────────────┐ │
  │ │    Young Generation   │   Old Generation      │ │
  │ │ ┌──────┬─────┬─────┐ │                       │ │
  │ │ │ Eden │ S0  │ S1  │ │  Long-lived objects   │ │
  │ │ │      │     │     │ │  (survived many GCs)  │ │
  │ │ └──────┴─────┴─────┘ │                       │ │
  │ │  New objects created  │                       │ │
  │ │  here (short-lived)   │                       │ │
  │ └──────────────────────┴───────────────────────┘ │
  ├──────────────────────────────────────────────────┤
  │ NON-HEAP                                         │
  │ • Metaspace (class metadata, unlimited by default)│
  │ • Code Cache (JIT compiled code)                 │
  │ • Thread Stacks (1MB per thread by default)      │
  ├──────────────────────────────────────────────────┤
  │ OFF-HEAP / Direct Memory                          │
  │ • NIO Buffers, memory-mapped files               │
  └──────────────────────────────────────────────────┘

GC Algorithms Comparison:

  Parallel GC:        [====== STW ======]  → high throughput, long pauses
  G1GC:               [== STW ==] concurrent [= STW =]  → balanced ⭐
  ZGC:                [.] concurrent...........[.]  → sub-ms pauses ⭐
  
  STW = Stop-The-World (application threads paused)
  Concurrent = GC runs alongside application

Tuning Decision Tree:
  Is it a batch/offline job? → Parallel GC (max throughput)
  Is it a web app (balanced)? → G1GC (default, tune MaxGCPauseMillis)
  Is p99 latency critical? → ZGC (sub-ms pauses)
  Heap > 16GB? → ZGC or G1 (Parallel GC struggles)
```

### 🗣️ How to Say in Interview
"I start by profiling rather than guessing — using Java Flight Recorder and GC logs to identify whether the issue is GC pauses, memory leaks, or thread contention. For heap sizing, I set Xms equal to Xmx to avoid resize overhead, and in containers I use MaxRAMPercentage at 75%. For GC selection, I use G1GC for most web applications — it provides a good balance of throughput and pause time with configurable MaxGCPauseMillis. For latency-sensitive services where p99 matters, I switch to ZGC which gives sub-millisecond pauses regardless of heap size. At the code level, I reduce object allocation: using primitive streams instead of boxed types, StringBuilder for string concatenation in loops, and object pooling for expensive-to-create objects like database connections. I monitor with Micrometer exporting JVM metrics to Prometheus — GC pause time, heap utilization, and thread count are on our Grafana dashboards. In my project, switching from Parallel GC to G1GC and tuning MaxGCPauseMillis from default to 100ms reduced our p99 latency from 800ms to 120ms."

### 💻 Code
```bash
# JVM FLAGS — Production configuration

# Heap sizing (container-aware)
-XX:MaxRAMPercentage=75.0      # use 75% of container memory for heap
-XX:InitialRAMPercentage=75.0  # start with same size (no resize)
# OR fixed sizing:
-Xms4g -Xmx4g                 # 4GB heap, same min/max (no resize)

# G1GC Configuration (most web apps)
-XX:+UseG1GC                   # default since Java 9
-XX:MaxGCPauseMillis=200       # target max pause time (200ms)
-XX:G1HeapRegionSize=8m        # region size (auto-tuned usually)
-XX:G1ReservePercent=15        # reserve for promotion failures
-XX:ConcGCThreads=4            # concurrent GC threads

# ZGC Configuration (latency-critical, Java 17+)
-XX:+UseZGC                    # sub-millisecond pauses
-XX:+ZGenerational             # generational ZGC (Java 21+, better throughput)
# ZGC needs minimal tuning — just enable and set heap size

# GC Logging (essential for troubleshooting)
-Xlog:gc*:file=/var/log/gc.log:time,uptime,level,tags:filecount=5,filesize=20m

# Memory safety
-XX:+ExitOnOutOfMemoryError               # kill JVM on OOM (K8s restarts)
-XX:+HeapDumpOnOutOfMemoryError            # heap dump for analysis
-XX:HeapDumpPath=/var/log/heapdump.hprof

# Metaspace
-XX:MaxMetaspaceSize=512m     # cap metaspace (prevent unbounded growth)

# JFR (Java Flight Recorder) — low-overhead profiling
-XX:StartFlightRecording=duration=60s,filename=/tmp/recording.jfr
```

```java
// CODE-LEVEL OPTIMIZATIONS

// 1. Avoid unnecessary boxing — use primitive streams
// BAD: boxing int → Integer for every element
int sum = list.stream().map(x -> x * 2).reduce(0, Integer::sum);

// GOOD: IntStream avoids boxing
int sum = list.stream().mapToInt(x -> x * 2).sum();

// 2. StringBuilder for loops (not + concatenation)
// BAD: creates new String object every iteration
String result = "";
for (String s : items) result += s + ",";  // O(n²) allocations!

// GOOD: single StringBuilder
StringBuilder sb = new StringBuilder(items.size() * 20);
for (String s : items) sb.append(s).append(',');

// 3. Object pooling for expensive objects
// HikariCP pools DB connections (don't create/close per query)
// Apache Commons Pool for custom object pools

// 4. Avoid finalizers and cleaners (GC overhead)
// BAD: finalizer delays GC, adds overhead
@Override protected void finalize() { /* don't do this */ }
// GOOD: use try-with-resources
try (Connection conn = dataSource.getConnection()) { /* auto-closed */ }

// 5. Monitor JVM metrics with Micrometer
@Configuration
public class JvmMetricsConfig {
    @Bean
    MeterRegistryCustomizer<MeterRegistry> metricsConfig() {
        return registry -> {
            new JvmMemoryMetrics().bindTo(registry);      // heap/non-heap
            new JvmGcMetrics().bindTo(registry);           // GC pauses
            new JvmThreadMetrics().bindTo(registry);       // thread count
            new ProcessorMetrics().bindTo(registry);        // CPU
        };
    }
}
// Grafana dashboards: jvm_memory_used_bytes, jvm_gc_pause_seconds_max,
//                     jvm_threads_live_threads, hikaricp_connections_active

// 6. Large datasets — use streaming (don't load all into heap)
@Transactional(readOnly = true)
public void processLargeDataset() {
    try (Stream<Order> stream = orderRepo.streamByStatusPending()) {
        stream.forEach(order -> {
            process(order);
            entityManager.detach(order);  // release from persistence context
        });
    }
}
```

### ⚠️ Pitfalls / Gotchas
- **Premature optimization** — profile first, THEN tune. "GC tuning" without data is guessing *(Pehle profile karo, bina data ke tune mat karo)*
- **-Xms ≠ -Xmx** → JVM resizes heap (GC pressure + latency spikes during resize)
- **MaxRAMPercentage=100** → OOM killed by K8s (OS + JVM non-heap also need memory). Use 75%
- **Too many GC threads** → context switching overhead. Default is usually fine
- **Metaspace leak** from classloader leaks (common in app servers with hot redeploy)
- **Forcing `System.gc()`** — never call it; it stops the world and JVM may ignore it anyway
- **ZGC on Java 11-16** was experimental — use Java 21+ for production ZGC with generational mode

### 🆚 vs. Comparison
| GC Algorithm | Pause Time | Throughput | Heap Size | Best For |
|-------------|-----------|------------|-----------|----------|
| Parallel GC | Long (seconds) | Highest ⭐ | Any | Batch jobs, offline |
| G1GC | Medium (~200ms) | High | 4GB-64GB | Web apps ⭐ (default) |
| ZGC | Sub-millisecond ⭐ | Good | Any (up to TB) | Latency-critical |
| Shenandoah | Sub-millisecond | Good | Any | Alternative to ZGC (RH) |
| Serial GC | Longest | Low | <100MB | Embedded/tiny apps |

### 🎯 Tricky Interview Qs

**Q: How do you diagnose a memory leak in production?**
(1) Enable `-XX:+HeapDumpOnOutOfMemoryError`. (2) Analyze heap dump with Eclipse MAT — find objects consuming most memory. (3) Look for growing collections, unclosed resources, or static field accumulation. (4) Common leaks: unbounded caches, event listeners not deregistered, ThreadLocal not cleared after request. *(Heap dump lo → Eclipse MAT se analyze karo → konsa object sabse zyada memory le raha hai woh dhundho)*

**Q: What is the difference between Young GC (Minor) and Old GC (Major/Full)?**
Minor GC: collects only Young Generation (Eden + Survivors). Fast (~10-50ms). Objects that survive get promoted to Old Gen. Major/Full GC: collects entire heap including Old Gen. Much slower (~100ms-seconds). Triggered when Old Gen is full. Full GC with STW = application pause = latency spike.

### ⚡ Remember
- **Profile first** → don't guess (JFR, GC logs, Micrometer)
- **-Xms = -Xmx** → avoid resize overhead
- **G1GC** = default, balanced (most apps) / **ZGC** = sub-ms pauses (latency-critical) *(G1 = normal apps, ZGC = jahan latency matter karti hai)*
- **MaxRAMPercentage=75** in containers (leave room for OS + non-heap)
- Reduce object allocation: primitive streams, StringBuilder, object pooling
- Monitor: GC pause time + heap usage + thread count

### 🔗 Follow-ups
- Q3 → JVM memory architecture (core/01)
- Q4 → OOM troubleshooting (core/01)
- Q6 → Performance bottleneck identification (this batch architecture/03)
