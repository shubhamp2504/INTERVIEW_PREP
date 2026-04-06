# 🌱 Spring Boot / Spring Batch Multithreading (Q77–Q86)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q77"></a>
## Q77. How does Spring support multithreading?

### 📝 One-Liner
> @Async for async methods, ThreadPoolTaskExecutor for pools, @Scheduled for timers, Spring Batch for parallel chunk/partition processing.

### 🔑 Quick Answer
> Spring provides: **@Async** (run method on separate thread), **ThreadPoolTaskExecutor** (production thread pool), **@Scheduled** (cron/fixed-rate tasks), and **Spring Batch** (multi-threaded steps + partitioning). Spring manages the thread lifecycle through the application context. *(Spring mein thread khud manage karo ki jaroorat nahi — annotation lagao, Spring sambhal lega)*

### 📖 How It Works
```
Spring Multithreading Stack:
┌─────────────────────────────────────────────┐
│ @Async / @Scheduled         ← Declarative   │
├─────────────────────────────────────────────┤
│ TaskExecutor                ← Abstraction    │
│  ├─ ThreadPoolTaskExecutor  ← Production ⭐  │
│  ├─ SimpleAsyncTaskExecutor ← Unbounded ⚠️   │
│  └─ ConcurrentTaskExecutor  ← Wrapper        │
├─────────────────────────────────────────────┤
│ Spring Batch:                                │
│  ├─ Multi-threaded Step     ← Parallel chunks│
│  ├─ Partitioned Step        ← Data splitting │
│  └─ Parallel Flows          ← Independent    │
├─────────────────────────────────────────────┤
│ Java ExecutorService / ThreadPoolExecutor    │
└─────────────────────────────────────────────┘
```

### 🗣️ Answering Approach
> *"Spring provides multithreading at several levels. @Async runs any method asynchronously — just add the annotation. ThreadPoolTaskExecutor is the production thread pool. @Scheduled handles cron-based or fixed-rate execution. For batch processing, Spring Batch offers multi-threaded steps and partitioned steps. All are managed by the Spring container — proper initialization and graceful shutdown."*

### ⚡ Remember
1. **@Async** = declarative async execution
2. **ThreadPoolTaskExecutor** = production pool ⭐
3. **@Scheduled** = cron and fixed-rate
4. **Spring Batch** = multi-threaded + partitioning
5. Spring manages **lifecycle** (init + shutdown)

### 🔗 Follow-ups
→ [Q78. @Async](#q78) → [Q79. Spring Batch parallel](#q79)

---

<a id="q78"></a>
## Q78. What is @Async in Spring?

### 📝 One-Liner
> Annotation that makes a method run on a separate thread from a pool — caller returns immediately.

### 🔑 Quick Answer
> `@Async` = method runs in a **separate thread**, caller returns **immediately**. Requires `@EnableAsync`. Uses proxy — does **NOT work** for self-invocation (calling from same class). Return `void` (fire-and-forget) or `CompletableFuture` (get result later). *(Method alag thread pe chalega — main thread ruka nahi)*

### 📖 How It Works
```
Without @Async (sequential):
  Controller: ─── sendEmail() ─── 2000ms ─── response (slow!)

With @Async (parallel):
  Controller: ─── sendEmail() → returns instantly → response (fast!)
  Thread pool:    └──── sendEmail runs here ──── 2000ms ────┘
  *(Email alag thread pe bhej do — user ko wait nahi karana)*

How it works internally:
  1. @EnableAsync activates proxy
  2. Spring wraps @Async method with PROXY
  3. Proxy intercepts → submits to TaskExecutor
  4. Caller continues immediately
  ⚠️ Self-invocation BYPASSES proxy → @Async ignored!
```

### 💻 Code
```java
@Configuration
@EnableAsync  // Required!
public class AsyncConfig {
    @Bean("emailExecutor")
    public ThreadPoolTaskExecutor emailExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("email-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        return executor;
    }
}

@Service
public class NotificationService {
    @Async("emailExecutor")  // specify which pool
    public CompletableFuture<String> sendEmail(String to) {
        emailClient.send(to);
        return CompletableFuture.completedFuture("Sent to " + to);
    }

    @Async("emailExecutor")
    public void sendSms(String phone) {  // fire-and-forget
        smsClient.send(phone);
    }
}
```

### ⚠️ Pitfalls / Gotchas
- **Self-invocation doesn't work** — calling @Async from same class bypasses proxy *(ek hi class ke andar call kiya toh @Async kaam nahi karega — proxy bypass)*
- Without named executor → uses **SimpleAsyncTaskExecutor** (creates new thread per task!) *(executor specify nahi kiya toh har baar naya thread — OOM risk)*
- **@EnableAsync** must be present *(bhul gaye toh annotation ignore hoga)*

### 🎯 Tricky Interview Qs
**Q: Why doesn't @Async work when called from the same class?**
> Spring AOP uses proxies — when you call a method within the same class, you bypass the proxy and call the real method directly. The proxy interceptor never fires. Fix: extract to a separate service bean. *(Same class = proxy nahi lagta — doosri class mein daalo)*

### 🗣️ Answering Approach
> *"@Async runs a method on a separate thread from a configured pool. I always specify a named executor to control which pool handles what. Two critical things: @EnableAsync must be present, and self-invocation doesn't work because Spring AOP proxies are bypassed for internal calls. I always configure a bounded ThreadPoolTaskExecutor with thread name prefixes for debugging."*

### ⚡ Remember
1. **@EnableAsync** required *(warna ignore hoga)*
2. **Named executor** — always specify pool
3. **Self-invocation** = proxy bypass = doesn't work ⭐
4. Return `CompletableFuture` or `void`
5. Always **bounded** ThreadPoolTaskExecutor

### 🔗 Follow-ups
→ [Q84. Configure TaskExecutor](#q84)

---

<a id="q79"></a>
## Q79. How does Spring Batch support parallel processing?

### 📝 One-Liner
> Multi-threaded step (parallel chunks), partitioned step (data split), parallel flows (independent steps), remote chunking/partitioning (multi-JVM).

### 🔑 Quick Answer
> Four approaches: **Multi-threaded step** (threads share reader ⚠️), **Partitioned step** (each worker has own reader ✅ — preferred), **Parallel flows** (independent steps simultaneously), **Remote partitioning** (across JVMs). *(Partitioned step sabse best — har worker apna data padhe)*

### 🆚 vs. Comparison
| Approach | Reader Thread-Safe? | Complexity | Production Use |
|----------|-------------------|------------|---------------|
| Multi-threaded step | ⚠️ Required | Low | Simple cases |
| **Partitioned step** | ✅ Own reader | Medium | **Most common** ⭐ |
| Parallel flows | N/A (different steps) | Low | Independent steps |
| Remote partitioning | ✅ Own reader | High | Massive scale |

### 📖 How It Works
```
1. MULTI-THREADED STEP:
   Thread-1: [Chunk 1: read→process→write]
   Thread-2: [Chunk 2: read→process→write]
   ⚠️ Reader shared! Must be thread-safe

2. PARTITIONED STEP (preferred ⭐):
   Partitioner: "Split by ID range"
   Worker-1: [IDs 1-10K]    own reader ✅
   Worker-2: [IDs 10K-20K]  own reader ✅
   Worker-3: [IDs 20K-30K]  own reader ✅
   *(Har worker apna hissa padhe — koi sharing nahi)*

3. PARALLEL FLOWS:
   Flow-1: [Step-A → Step-B]  ──╮
   Flow-2: [Step-C]           ──┼──→ simultaneously
```

### 🗣️ Answering Approach
> *"Spring Batch provides four parallel processing strategies. My preferred is partitioned step — a Partitioner splits data into ranges, each worker gets its own reader instance, no thread-safety concerns. Multi-threaded step is simpler but the reader must be thread-safe. Parallel flows run independent steps concurrently. For massive scale, remote partitioning distributes across JVMs."*

### ⚡ Remember
1. **Partitioned step** = preferred (own reader per worker) ⭐
2. **Multi-threaded step** = simpler but reader must be thread-safe
3. **Parallel flows** = independent steps concurrently
4. **Remote partitioning** = distributed across JVMs
5. Most production use: **partitioned step**

### 🔗 Follow-ups
→ [Q80. Multi-threaded step](#q80) → [Q81. Partitioning](#q81)

---

<a id="q80"></a>
## Q80. What is a multi-threaded step in Spring Batch?

### 📝 One-Liner
> Add TaskExecutor to a step — multiple threads process chunks in parallel, but reader MUST be thread-safe.

### 🔑 Quick Answer
> Assign a `TaskExecutor` to the step — instead of one thread, **multiple threads pick up chunks** and process them in parallel. The reader must be **thread-safe** (wrap with `SynchronizedItemStreamReader`). Chunk order is **not guaranteed**. *(Ek step mein bahut threads ek saath chunk process karein — par reader thread-safe hona chahiye)*

### 💻 Code
```java
@Bean
public Step multiThreadedStep() {
    return stepBuilderFactory.get("multiThreadedStep")
        .<Input, Output>chunk(100)
        .reader(synchronizedReader())       // MUST be thread-safe!
        .processor(processor())
        .writer(writer())
        .taskExecutor(taskExecutor())       // enable multi-threading
        .throttleLimit(10)                  // max 10 concurrent
        .build();
}

@Bean
public SynchronizedItemStreamReader<Input> synchronizedReader() {
    SynchronizedItemStreamReader<Input> reader = new SynchronizedItemStreamReader<>();
    reader.setDelegate(flatFileItemReader());  // wrap non-thread-safe reader
    return reader;
}
```

### ⚠️ Pitfalls / Gotchas
- **FlatFileItemReader is NOT thread-safe** — must wrap with SynchronizedItemStreamReader *(FlatFile reader thread-safe nahi hai — wrap karo)*
- **Chunk order not guaranteed** — data may be written out of order *(order guarantee nahi hai)*
- Processor and writer must be **stateless** or thread-safe

### 🗣️ Answering Approach
> *"A multi-threaded step assigns a TaskExecutor to the step — multiple threads pick up chunks in parallel. The key consideration is reader thread safety — I wrap FlatFileItemReader with SynchronizedItemStreamReader. throttleLimit controls max concurrent threads. For most production cases, I prefer partitioned steps where each worker has its own reader."*

### ⚡ Remember
1. Add **taskExecutor()** to step builder
2. Reader **must be thread-safe** *(warna data corrupt)*
3. **throttleLimit** = max concurrent threads
4. Chunk order **not guaranteed**
5. Prefer **partitioned step** for production ⭐

### 🔗 Follow-ups
→ [Q81. Partitioning](#q81)

---

<a id="q81"></a>
## Q81. What is partitioning in Spring Batch?

### 📝 One-Liner
> Split data into independent ranges — each worker gets its own reader/processor/writer, no thread-safety concerns.

### 🔑 Quick Answer
> A `Partitioner` splits data into **independent ranges** (by ID, date, etc.). Each partition processed by a **separate worker** with its **own reader** (via `@StepScope`). No thread-safety issues — each worker is independent. **Production preferred** approach. *(Data ko hisson mein baato — har worker apna hissa padhe, koi sharing nahi)*

### 💻 Code
```java
@Bean
public Step managerStep() {
    return stepBuilderFactory.get("managerStep")
        .partitioner("workerStep", partitioner())
        .step(workerStep())
        .gridSize(10)                  // 10 partitions
        .taskExecutor(taskExecutor())  // thread pool
        .build();
}

@Bean
public Partitioner partitioner() {
    return gridSize -> {
        Map<String, ExecutionContext> partitions = new HashMap<>();
        int range = 100000 / gridSize;
        for (int i = 0; i < gridSize; i++) {
            ExecutionContext ctx = new ExecutionContext();
            ctx.putInt("minId", i * range + 1);
            ctx.putInt("maxId", (i + 1) * range);
            partitions.put("partition" + i, ctx);
        }
        return partitions;
    };
}

@Bean @StepScope  // NEW instance per partition! ⭐
public JdbcPagingItemReader<Record> reader(
        @Value("#{stepExecutionContext['minId']}") int minId,
        @Value("#{stepExecutionContext['maxId']}") int maxId) {
    // Each reader queries ONLY its range
    JdbcPagingItemReader<Record> reader = new JdbcPagingItemReader<>();
    reader.setDataSource(dataSource);
    reader.setPageSize(1000);
    return reader;
}
```

### 🗣️ Answering Approach
> *"Partitioning is my preferred Spring Batch parallelism approach. A Partitioner splits data into independent ranges — typically by ID or date. Each partition gets its own worker with a separate reader, processor, and writer. Since each worker has its own reader instance via @StepScope, there are no thread-safety concerns. I use ColumnRangePartitioner for database-driven partitioning."*

### 🎯 Tricky Interview Qs
**Q: Why is @StepScope important in partitioning?**
> @StepScope creates a **new bean instance per step execution**. Each partition is a separate step execution, so each gets its own reader with its own min/max ID range. Without @StepScope, all partitions would share one reader — broken! *(StepScope = har partition ko nayi bean — bina iske sab ek hi reader share karenge)*

### ⚡ Remember
1. **Partitioner** splits data into ranges
2. Each worker has **own reader** → no thread-safety issue ⭐
3. **@StepScope** = new instance per partition
4. **gridSize** = number of partitions
5. **Preferred** over multi-threaded step in production

### 🔗 Follow-ups
→ [Q82. Remote partitioning](#q82)

---

<a id="q82"></a>
## Q82. What is remote partitioning?

### 📝 One-Liner
> Distributes partitions across multiple JVMs via messaging — manager sends metadata, worker JVMs do full read-process-write.

### 🔑 Quick Answer
> Manager JVM creates partitions → sends **metadata** (not data) via messaging (RabbitMQ/Kafka) → **worker JVMs** independently read their partition, process, and write. Used for **massive-scale** batch where one JVM isn't enough. *(Ek machine kaafi nahi — bahut machines pe kaam baato)*

### 📖 How It Works
```
Local Partitioning:
  Single JVM: Manager → Thread-1, Thread-2, ...

Remote Partitioning:
  Manager JVM → [RabbitMQ] → Worker JVM-1 (reads+processes IDs 1-1M)
                            → Worker JVM-2 (reads+processes IDs 1M-2M)
                            → Worker JVM-3 (reads+processes IDs 2M-3M)
  *(Bahut machines — Linear scaling)*
  
  Only METADATA flows through broker (ID ranges, not actual data)
```

### 🗣️ Answering Approach
> *"Remote partitioning distributes across multiple JVMs. The manager creates partitions and sends metadata — like ID ranges — to a message broker. Worker JVMs pick up the ranges, read their data directly from the source, process, and write. Only metadata flows through the broker, not actual data. This scales horizontally — add more worker machines for more throughput."*

### ⚡ Remember
1. **Manager JVM** creates partitions
2. **Worker JVMs** do full read-process-write
3. **Only metadata** through messaging (not data)
4. **Massive scale** (millions+ records)
5. Messaging: **RabbitMQ, Kafka, JMS**

### 🔗 Follow-ups
→ [Q83. Remote chunking](#q83)

---

<a id="q83"></a>
## Q83. What is remote chunking?

### 📝 One-Liner
> Manager reads data and sends actual items to workers for processing/writing — use when processing is the bottleneck.

### 🔑 Quick Answer
> Manager **reads** data → sends **actual items** via messaging → workers **process + write**. Unlike remote partitioning where only metadata flows, here **actual data** goes through the broker. Use when **processing is the bottleneck**, not reading. *(Manager padhe, workers process karein — jab process karna slow ho)*

### 🆚 vs. Comparison
| | Remote Partitioning | Remote Chunking |
|-|-------------------|----------------|
| Broker carries | Metadata (IDs) only | **Actual data** |
| Worker does | Read + process + write | Process + write only |
| Use when | Reading is bottleneck | **Processing** is bottleneck |
| Broker load | Low ✅ | High ⚠️ |
| **Preferred** | ⭐ Most cases | Special cases |

### 🗣️ Answering Approach
> *"Remote chunking differs from partitioning in what flows through the broker. In partitioning, only metadata like ID ranges is sent — workers read their own data. In chunking, the manager reads data and sends actual items. Chunking is useful when processing is the bottleneck but reading is fast. I prefer remote partitioning in most cases because it distributes the I/O load."*

### ⚡ Remember
1. Manager **reads**, workers **process + write**
2. **Actual data** through messaging ⚠️
3. Use when **processing** is the bottleneck
4. Higher broker load than partitioning
5. **Remote partitioning preferred** in most cases

### 🔗 Follow-ups
→ [Q84. TaskExecutor config](#q84)

---

<a id="q84"></a>
## Q84. How to configure TaskExecutor in Spring?

### 📝 One-Liner
> ThreadPoolTaskExecutor bean with core/max pool size, bounded queue, thread name prefix, CallerRunsPolicy, graceful shutdown.

### 🔑 Quick Answer
> Define `ThreadPoolTaskExecutor` bean: **core** (always alive), **max** (burst), **queue** (bounded!), **prefix** (debugging), **rejection** (CallerRunsPolicy). Use names like `"emailExecutor"`, `"batchExecutor"` — separate pools for separate concerns. *(Har kaam ka alag pool banao — bounded raho, naam do)*

### 💻 Code
```java
@Configuration
public class ExecutorConfig {
    @Bean("generalExecutor")
    public ThreadPoolTaskExecutor generalExecutor() {
        ThreadPoolTaskExecutor e = new ThreadPoolTaskExecutor();
        e.setCorePoolSize(10);
        e.setMaxPoolSize(25);
        e.setQueueCapacity(500);          // bounded! ⭐
        e.setKeepAliveSeconds(60);
        e.setThreadNamePrefix("general-");  // debugging ⭐
        e.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        e.setWaitForTasksToCompleteOnShutdown(true);  // graceful
        e.setAwaitTerminationSeconds(30);
        return e;
    }

    @Bean("batchExecutor")  // separate pool for batch
    public ThreadPoolTaskExecutor batchExecutor() {
        ThreadPoolTaskExecutor e = new ThreadPoolTaskExecutor();
        e.setCorePoolSize(20);
        e.setMaxPoolSize(20);
        e.setQueueCapacity(100);
        e.setThreadNamePrefix("batch-");
        return e;
    }
}
```

### 🗣️ Answering Approach
> *"I configure ThreadPoolTaskExecutor as named Spring beans. Bounded queue capacity is critical — unbounded risks OOM. Thread name prefix is essential for debugging — 'batch-3' in a thread dump tells me exactly which pool. CallerRunsPolicy for rejection provides natural backpressure. Graceful shutdown with waitForTasksToComplete ensures in-flight tasks finish. I create separate pools for different workloads to prevent interference."*

### ⚡ Remember
1. **Bounded queue** always ⭐
2. **Thread name prefix** for debugging ⭐
3. **CallerRunsPolicy** for backpressure
4. **Graceful shutdown** — waitForTasksToComplete
5. Separate pools for **different workloads**

### 🔗 Follow-ups
→ [Q86. ThreadPoolTaskExecutor details](#q86)

---

<a id="q85"></a>
## Q85. What is SimpleAsyncTaskExecutor?

### 📝 One-Liner
> Creates a NEW thread for every task — no pooling, no limit — default for @Async if none configured. NEVER use in production.

### 🔑 Quick Answer
> Creates a **brand new thread per task** — no reuse, no upper limit. It's the **default** executor for `@Async` when none is configured. Under load → thousands of threads → **OOM crash**. **Always replace** with ThreadPoolTaskExecutor. *(Har kaam ke liye naya thread — koi limit nahi — production mein mat use karo)*

### 📖 How It Works
```
SimpleAsyncTaskExecutor:
  Task 1 → new Thread() → runs → dies
  Task 2 → new Thread() → runs → dies
  Task 10000 → new Thread() → OutOfMemoryError! 💀
  *(Koi pool nahi — har baar naya thread)*

ThreadPoolTaskExecutor:
  Task 1 → Thread-1 (reused) → returns to pool
  Task 2 → Thread-2 (reused) → returns to pool
  Task 10000 → waits in queue → processed when available ✅
```

### ⚠️ Pitfalls / Gotchas
- **Default** for @Async if no executor configured *(bhulo mat — warna ye lag jaayega)*
- **No upper limit** = OOM guaranteed under load *(production mein crash hoga)*
- First thing after @EnableAsync: **configure ThreadPoolTaskExecutor**

### 🗣️ Answering Approach
> *"SimpleAsyncTaskExecutor is Spring's default and it's dangerous — creates a new thread per task with no pooling, no limit. Under load, thousands of threads = OOM crash. The first thing I do after @EnableAsync is configure a proper ThreadPoolTaskExecutor with bounded pool and queue."*

### ⚡ Remember
1. **New thread per task** — no pooling *(har baar naya)*
2. **No upper limit** — OOM risk 💀
3. **Default** for @Async ⚠️
4. **Never** in production
5. Always replace with **ThreadPoolTaskExecutor** ⭐

### 🔗 Follow-ups
→ [Q86. ThreadPoolTaskExecutor](#q86)

---

<a id="q86"></a>
## Q86. What is ThreadPoolTaskExecutor?

### 📝 One-Liner
> Spring's production thread pool — wraps Java's ThreadPoolExecutor with Spring lifecycle, bounded pool/queue, graceful shutdown.

### 🔑 Quick Answer
> The **only executor you should use in production**. Wraps `ThreadPoolExecutor` with Spring lifecycle management. Flow: **core threads → queue → max threads → rejection**. Provides thread name prefix, graceful shutdown, and bounded resources. *(Production ka standard — hamesha ye use karo)*

### 📖 How It Works
```
Task arrival flow (core=5, max=10, queue=100):

  Tasks 1-5:      → 5 core threads created ✅
  Tasks 6-105:    → queued (100 slots) ✅
  Tasks 106-110:  → 5 more threads (up to max=10) ✅
  Task 111:       → REJECTED → CallerRunsPolicy ⚠️
  *(Pehle core, phir queue, phir max, phir reject)*
```

### 💻 Code
```java
@Bean
public ThreadPoolTaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor e = new ThreadPoolTaskExecutor();
    e.setCorePoolSize(10);      // always ready
    e.setMaxPoolSize(25);       // burst handling
    e.setQueueCapacity(500);    // buffer (bounded!)
    e.setKeepAliveSeconds(60);  // idle cleanup
    e.setThreadNamePrefix("app-worker-");   // debugging ⭐
    e.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    e.setWaitForTasksToCompleteOnShutdown(true);
    e.setAwaitTerminationSeconds(60);
    e.initialize();
    return e;
}
```

### 🗣️ Answering Approach
> *"ThreadPoolTaskExecutor is the production standard. Core threads are always alive, tasks queue when core is busy, extra threads created up to max under load, and rejection policy handles overflow. I always set bounded queue to prevent OOM, thread name prefix for debugging, CallerRunsPolicy for backpressure, and graceful shutdown settings."*

### 🎯 Tricky Interview Qs
**Q: When are max threads created?**
> Only when the **queue is full** AND active < maxPoolSize. Common misunderstanding: people think max threads are created when core threads are busy — but tasks go to queue first. *(Queue bhar jaaye TABHI max threads bante hain — pehle nahi)*

### ⚡ Remember
1. **Production standard** ⭐
2. Core → Queue → Max → Reject (flow)
3. **Bounded queue** prevents OOM
4. **Thread name prefix** for debugging
5. **Graceful shutdown** for clean app stop

### 🔗 Follow-ups
→ [Q91. Pool exhaustion](10-production-scenarios.md#q91)

---

> **🎯 Navigation:** [← Performance (Q70-76)](08-performance.md) | [Next → Production Scenarios (Q87-96)](10-production-scenarios.md) | [📋 All Sections](README.md)
