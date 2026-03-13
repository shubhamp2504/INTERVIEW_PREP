# 🌱 Spring Boot / Spring Batch Multithreading (Q77–Q86)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q77"></a>

## Q77. How does Spring support multithreading?

### 🔑 Quick Answer

> Spring provides `TaskExecutor` abstraction (wrapping Java's Executor), `@Async` for declarative async execution, `@Scheduled` for timed tasks, `ThreadPoolTaskExecutor` for production thread pools, and Spring Batch's parallel processing features. Spring manages thread lifecycles through the container.

### 📖 Step-by-Step Explanation

```
Spring Multithreading Stack:

┌─────────────────────────────────────────────────┐
│ @Async / @Scheduled          ← Declarative      │
├─────────────────────────────────────────────────┤
│ TaskExecutor                 ← Abstraction       │
│  ├─ ThreadPoolTaskExecutor   ← Production ⭐     │
│  ├─ SimpleAsyncTaskExecutor  ← Unbounded ⚠️      │
│  └─ ConcurrentTaskExecutor   ← Wrapper           │
├─────────────────────────────────────────────────┤
│ Spring Batch:                                    │
│  ├─ Multi-threaded Step      ← Parallel chunks   │
│  ├─ Partitioned Step         ← Data partitioning │
│  └─ Parallel Flows           ← Independent steps │
├─────────────────────────────────────────────────┤
│ Java ExecutorService / ThreadPoolExecutor        │
└─────────────────────────────────────────────────┘
```

### 🗣️ How to Explain in Interview

> *"Spring provides multithreading at several levels. At the simplest level, @Async lets me run any method asynchronously by just adding an annotation. Under the hood, Spring uses its TaskExecutor abstraction — ThreadPoolTaskExecutor is the production-standard implementation. For scheduled tasks, @Scheduled handles cron-based or fixed-rate execution. For batch processing, Spring Batch offers multi-threaded steps for parallel chunk processing and partitioned steps for splitting data across threads. All these are managed by the Spring container, so thread pools are properly initialized and shut down with the application."*

### ⚡ Key Points to Remember

1. **@Async** = declarative async execution
2. **ThreadPoolTaskExecutor** = production thread pool ⭐
3. **@Scheduled** = cron and fixed-rate tasks
4. **Spring Batch** = multi-threaded steps + partitioning
5. Spring manages **thread lifecycle** (init + shutdown)

---

<a id="q78"></a>

## Q78. What is @Async in Spring?

### 🔑 Quick Answer

> `@Async` makes a method run in a **separate thread** from a configured thread pool. The caller returns immediately — either with `void` or a `Future`/`CompletableFuture`. Requires `@EnableAsync` on a configuration class.

### 📖 Step-by-Step Explanation

```
Without @Async (sequential):
  Controller:  ─── sendEmail() ─── 2000ms ─── response (slow!)

With @Async (parallel):
  Controller:  ─── sendEmail() → returns immediately → response (fast!)
  Thread pool:     └──── sendEmail runs here ──── 2000ms ────┘
```

**How it works internally:**

```
1. @EnableAsync activates Spring's async proxy mechanism
2. Spring wraps @Async methods with a PROXY
3. Proxy intercepts the call → submits to TaskExecutor
4. Original thread continues immediately
5. Async method runs on thread pool thread

⚠️ IMPORTANT: @Async does NOT work when calling from the SAME class!
   (Proxy is bypassed for internal calls)
```

### 💻 Code Example

```java
@Configuration
@EnableAsync
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
    
    @Async("emailExecutor")  // Specify which pool to use
    public CompletableFuture<String> sendEmail(String to, String body) {
        // Runs on email-worker thread
        emailClient.send(to, body);
        return CompletableFuture.completedFuture("Sent to " + to);
    }
    
    @Async("emailExecutor")
    public void sendSms(String phone, String message) {
        // Fire-and-forget — void return
        smsClient.send(phone, message);
    }
}

// Caller — returns immediately
@RestController
public class OrderController {
    @Autowired
    private NotificationService notificationService;
    
    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(@RequestBody Order order) {
        orderService.save(order);
        notificationService.sendEmail(order.getEmail(), "Order created");  // non-blocking
        return ResponseEntity.ok(order);  // Returns immediately!
    }
}
```

### 🗣️ How to Explain in Interview

> *"@Async runs a method on a separate thread from a configured pool. I always specify a named executor — @Async('emailExecutor') — so I know which pool handles what. The return type is either void for fire-and-forget, or CompletableFuture for getting results later. Two critical things: First, @EnableAsync must be present. Second, @Async doesn't work for internal calls within the same class — it relies on Spring's proxy mechanism, so the call must come from another bean. I always configure a bounded ThreadPoolTaskExecutor and set thread name prefixes for easier debugging."*

### ⚡ Key Points to Remember

1. **@EnableAsync** required on config class
2. **Named executor** — always specify which pool
3. **Self-invocation doesn't work** (proxy bypass) ⭐
4. Return `CompletableFuture` for results, `void` for fire-and-forget
5. Always use **bounded** ThreadPoolTaskExecutor

---

<a id="q79"></a>

## Q79. How does Spring Batch support parallel processing?

### 🔑 Quick Answer

> Spring Batch provides four parallel processing approaches: **Multi-threaded Step** (multiple threads process chunks), **Partitioned Step** (data split across threads), **Parallel Flows** (independent steps run simultaneously), and **Remote Chunking/Partitioning** (distributed across JVMs).

### 📖 Step-by-Step Explanation

```
Spring Batch Parallel Processing Options:

1. MULTI-THREADED STEP:
   Step → ThreadPoolTaskExecutor
   Thread-1: [Chunk 1: read→process→write]
   Thread-2: [Chunk 2: read→process→write]
   Thread-3: [Chunk 3: read→process→write]
   ⚠️ Reader must be thread-safe!

2. PARTITIONED STEP:
   Manager → splits data into partitions
   Worker-1: [Partition: IDs 1-10000]
   Worker-2: [Partition: IDs 10001-20000]
   Worker-3: [Partition: IDs 20001-30000]
   ✅ Each worker has own reader → no thread-safety issue!

3. PARALLEL FLOWS:
   Flow-1: [Step-A → Step-B]  ──╮
   Flow-2: [Step-C → Step-D]  ──┼──→ All run simultaneously
   Flow-3: [Step-E]           ──╯

4. REMOTE PARTITIONING:
   Manager JVM → sends partitions → Worker JVM-1, JVM-2, JVM-3
   → Scales across multiple machines
```

### 🗣️ How to Explain in Interview

> *"Spring Batch provides four parallel processing strategies. Multi-threaded step is the simplest — assign a TaskExecutor to a step and multiple threads process different chunks simultaneously. The catch is the reader must be thread-safe. Partitioned step is more powerful — a Partitioner splits the data into ranges, and each worker thread gets its own reader instance for its partition — no thread-safety concern. Parallel flows run independent steps simultaneously — like processing orders and generating reports at the same time. For really large scale, remote partitioning distributes work across multiple JVMs."*

### ⚡ Key Points to Remember

1. **Multi-threaded step** = simplest, reader must be thread-safe
2. **Partitioned step** = each worker has own reader ⭐ (preferred)
3. **Parallel flows** = independent steps run concurrently
4. **Remote partitioning** = distributed across JVMs
5. Most common in production: **partitioned step**

---

<a id="q80"></a>

## Q80. What is a multi-threaded step in Spring Batch?

### 🔑 Quick Answer

> A step that uses a **TaskExecutor** to process multiple chunks **in parallel threads**. Each thread reads a chunk, processes it, and writes it. The reader must be **thread-safe** (e.g., `SynchronizedItemStreamReader` or `JdbcPagingItemReader`).

### 📖 Step-by-Step Explanation

```
Normal Step (single-threaded):
  main-thread: [Chunk1] → [Chunk2] → [Chunk3] → [Chunk4]
  Time: ═══════════════════════════════════════════════════

Multi-threaded Step:
  thread-1: [Chunk1] [Chunk5]
  thread-2: [Chunk2] [Chunk6]
  thread-3: [Chunk3] [Chunk7]
  thread-4: [Chunk4] [Chunk8]
  Time: ══════════════════════  (4x faster with 4 threads)
```

### 💻 Code Example

```java
@Bean
public Step multiThreadedStep() {
    return stepBuilderFactory.get("multiThreadedStep")
        .<InputRecord, OutputRecord>chunk(100)    // 100 records per chunk
        .reader(synchronizedReader())              // MUST be thread-safe!
        .processor(processor())
        .writer(writer())
        .taskExecutor(taskExecutor())              // Enable multi-threading
        .throttleLimit(10)                         // Max 10 concurrent threads
        .build();
}

@Bean
public SynchronizedItemStreamReader<InputRecord> synchronizedReader() {
    SynchronizedItemStreamReader<InputRecord> reader = new SynchronizedItemStreamReader<>();
    reader.setDelegate(flatFileItemReader());  // Wrap non-thread-safe reader
    return reader;
}

@Bean
public ThreadPoolTaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(10);
    executor.setMaxPoolSize(10);
    executor.setQueueCapacity(50);
    executor.setThreadNamePrefix("batch-worker-");
    return executor;
}
```

### 🗣️ How to Explain in Interview

> *"A multi-threaded step assigns a TaskExecutor to the step configuration. Instead of one thread processing chunks sequentially, multiple threads pick up chunks and process them in parallel. The key consideration is thread safety of the reader — FlatFileItemReader is NOT thread-safe, so I wrap it with SynchronizedItemStreamReader. JdbcPagingItemReader IS thread-safe. I set throttleLimit to control the maximum concurrent threads — usually matching the pool size. The processor and writer also need to be stateless or thread-safe."*

### ⚡ Key Points to Remember

1. Add **taskExecutor()** to step builder
2. Reader **must be thread-safe** (SynchronizedItemStreamReader)
3. **throttleLimit** = max concurrent chunk processing threads
4. Processor + writer must be **stateless** or thread-safe
5. Chunk order is **NOT guaranteed** in multi-threaded step

---

<a id="q81"></a>

## Q81. What is partitioning in Spring Batch?

### 🔑 Quick Answer

> Partitioning splits the **input data into independent ranges** (partitions), and each partition is processed by a **separate worker with its own reader/processor/writer**. No thread-safety concerns for the reader since each worker has its own instance.

### 📖 Step-by-Step Explanation

```
Partitioning architecture:

  Manager Step (Partitioner):
    "Total: 100,000 records"
    → Partition 1: IDs 1 – 25,000
    → Partition 2: IDs 25,001 – 50,000
    → Partition 3: IDs 50,001 – 75,000
    → Partition 4: IDs 75,001 – 100,000

  Worker-1: [Reader₁ → Processor₁ → Writer₁]  (IDs 1-25K)
  Worker-2: [Reader₂ → Processor₂ → Writer₂]  (IDs 25K-50K)
  Worker-3: [Reader₃ → Processor₃ → Writer₃]  (IDs 50K-75K)
  Worker-4: [Reader₄ → Processor₄ → Writer₄]  (IDs 75K-100K)

  Each worker has its OWN reader → no thread-safety issue! ✅
  Each partition is independent → true parallel processing
```

### 💻 Code Example

```java
@Bean
public Step managerStep() {
    return stepBuilderFactory.get("managerStep")
        .partitioner("workerStep", partitioner())   // Define partitioner
        .step(workerStep())                          // Worker step template
        .gridSize(10)                                // 10 partitions
        .taskExecutor(taskExecutor())                // Thread pool
        .build();
}

@Bean
public Partitioner partitioner() {
    return gridSize -> {
        Map<String, ExecutionContext> partitions = new HashMap<>();
        int totalRecords = 100000;
        int range = totalRecords / gridSize;
        
        for (int i = 0; i < gridSize; i++) {
            ExecutionContext context = new ExecutionContext();
            context.putInt("minId", i * range + 1);
            context.putInt("maxId", (i + 1) * range);
            partitions.put("partition" + i, context);
        }
        return partitions;
    };
}

@Bean
@StepScope  // New instance per partition!
public JdbcPagingItemReader<Record> reader(
        @Value("#{stepExecutionContext['minId']}") int minId,
        @Value("#{stepExecutionContext['maxId']}") int maxId) {
    
    JdbcPagingItemReader<Record> reader = new JdbcPagingItemReader<>();
    reader.setDataSource(dataSource);
    reader.setPageSize(1000);
    // Each reader queries only its partition range
    Map<String, Object> params = new HashMap<>();
    params.put("minId", minId);
    params.put("maxId", maxId);
    reader.setParameterValues(params);
    return reader;
}
```

### 🗣️ How to Explain in Interview

> *"Partitioning is my preferred Spring Batch parallelism approach. A Partitioner splits the data into independent ranges — typically by ID or date range. Each partition gets its own worker with a separate reader, processor, and writer. Since each worker has its own reader instance, there are no thread-safety concerns — unlike multi-threaded steps. The reader is @StepScope so Spring creates a new instance for each partition, injecting the min and max IDs from the ExecutionContext. I typically use ColumnRangePartitioner for database-driven partitioning."*

### ⚡ Key Points to Remember

1. **Partitioner** splits data into independent ranges
2. Each worker has **own reader** → no thread-safety issue ⭐
3. Use **@StepScope** for partition-aware beans
4. **gridSize** = number of partitions
5. Preferred over multi-threaded step in production

---

<a id="q82"></a>

## Q82. What is remote partitioning?

### 🔑 Quick Answer

> Remote partitioning distributes partitions across **multiple JVMs/machines** instead of threads within one JVM. The **manager** sends partition metadata via messaging (e.g., RabbitMQ, Kafka), and **worker JVMs** process their partitions independently. Used for **massive-scale** batch processing.

### 📖 Step-by-Step Explanation

```
Local Partitioning:
  Single JVM:
    Manager → Thread-1(Partition-1), Thread-2(Partition-2), ...
  Limited by single machine resources

Remote Partitioning:
  Manager JVM → [RabbitMQ/Kafka] → Worker JVM-1 (Partition-1)
                                  → Worker JVM-2 (Partition-2)
                                  → Worker JVM-3 (Partition-3)
  Scales across multiple machines!
  
  Data flow:
    Manager: Creates partitions → sends to messaging middleware
    Workers: Receive partition → read data → process → write
    Manager: Monitors completion via messaging
```

### 🗣️ How to Explain in Interview

> *"Remote partitioning is partitioning across multiple JVMs. The manager step creates partitions as usual, but instead of processing locally, it sends partition metadata — like ID ranges — to a message broker like RabbitMQ. Worker JVMs pick up the messages, read their partition of data, process, and write. The workers handle the full read-process-write pipeline. Only metadata flows through the message broker, not the actual data — workers read directly from the data source. This lets you scale horizontally by adding more worker machines."*

### ⚡ Key Points to Remember

1. **Manager JVM** creates and distributes partitions
2. **Worker JVMs** handle full read-process-write
3. **Only metadata** flows through messaging (not data)
4. Used for **massive scale** (millions+ records)
5. Messaging: **RabbitMQ, Kafka, JMS**

---

<a id="q83"></a>

## Q83. What is remote chunking?

### 🔑 Quick Answer

> Remote chunking sends **actual data items** to worker JVMs for processing and writing. The **manager reads** the data and sends chunks via messaging. **Workers process and write**. Used when processing is the bottleneck, not reading.

### 📖 Step-by-Step Explanation

```
Remote Partitioning vs Remote Chunking:

Remote PARTITIONING:
  Manager: "Process IDs 1-10000"  →  Worker reads + processes + writes
  (sends metadata only)              (worker does everything)

Remote CHUNKING:
  Manager: reads data → sends actual items → Worker processes + writes
  (manager reads, sends data)                (worker processes + writes)

When to use which:
  - Reading is fast, processing is slow → Remote CHUNKING ✅
  - Reading is slow, or data is distributed → Remote PARTITIONING ✅
```

### 🗣️ How to Explain in Interview

> *"Remote chunking differs from remote partitioning in what flows through the message broker. In remote partitioning, only metadata like ID ranges is sent — workers read their own data. In remote chunking, the manager reads the data and sends the actual items to workers through messaging — workers only process and write. Remote chunking is useful when processing is the bottleneck but reading is fast. However, it puts more load on the message broker since actual data flows through it. In most production scenarios, I prefer remote partitioning because it distributes the I/O load."*

### ⚡ Key Points to Remember

1. Manager **reads** data, workers **process + write**
2. **Actual data** flows through messaging ⚠️
3. Use when **processing is the bottleneck**
4. Higher messaging load than remote partitioning
5. Remote partitioning preferred in most cases

---

<a id="q84"></a>

## Q84. How to configure TaskExecutor in Spring?

### 🔑 Quick Answer

> Define a `ThreadPoolTaskExecutor` bean with **core pool size**, **max pool size**, **queue capacity**, **thread name prefix**, and **rejection policy**. Inject by name for `@Async` or as a dependency for Spring Batch steps.

### 💻 Code Example

```java
@Configuration
public class ExecutorConfig {
    
    // General-purpose pool
    @Bean("generalExecutor")
    public ThreadPoolTaskExecutor generalExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);                 // Always running
        executor.setMaxPoolSize(25);                  // Burst capacity
        executor.setQueueCapacity(500);               // Bounded! ⭐
        executor.setKeepAliveSeconds(60);             // Idle thread timeout
        executor.setThreadNamePrefix("general-");     // For debugging ⭐
        executor.setRejectedExecutionHandler(
            new ThreadPoolExecutor.CallerRunsPolicy() // Back-pressure
        );
        executor.setWaitForTasksToCompleteOnShutdown(true);  // Graceful
        executor.setAwaitTerminationSeconds(30);
        return executor;
    }
    
    // Dedicated batch pool
    @Bean("batchExecutor")
    public ThreadPoolTaskExecutor batchExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(20);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("batch-");
        return executor;
    }
}
```

### 🗣️ How to Explain in Interview

> *"I configure ThreadPoolTaskExecutor as a named Spring bean. Core pool size is the number of always-running threads — I base this on the workload type and available cores. Max pool size handles burst traffic. Queue capacity must always be bounded — an unbounded queue risks OutOfMemoryError. Thread name prefix is critical for debugging — when I see 'batch-3' in a thread dump, I know exactly which pool it belongs to. I always set CallerRunsPolicy for rejection — it provides natural back-pressure, slowing down the producer. For graceful shutdown, I enable waitForTasksToComplete."*

### ⚡ Key Points to Remember

1. **Bounded queue** always ⭐
2. **Thread name prefix** for debugging ⭐
3. **CallerRunsPolicy** for back-pressure
4. **Graceful shutdown** — waitForTasksToComplete
5. Separate pools for **different workloads**

---

<a id="q85"></a>

## Q85. What is SimpleAsyncTaskExecutor?

### 🔑 Quick Answer

> SimpleAsyncTaskExecutor creates a **new thread for every task** — no pooling, no reuse, **no upper limit**. It's the **default executor** for `@Async` if none is configured. **Never use in production** — it can create thousands of threads and crash the JVM.

### 📖 Step-by-Step Explanation

```
SimpleAsyncTaskExecutor:
  Task 1 → new Thread()  → runs → thread dies
  Task 2 → new Thread()  → runs → thread dies
  Task 3 → new Thread()  → runs → thread dies
  ...
  Task 10000 → new Thread() → OutOfMemoryError! 💀

ThreadPoolTaskExecutor:
  Task 1 → Thread-1 (reused) → runs → returns to pool
  Task 2 → Thread-2 (reused) → runs → returns to pool
  Task 3 → Thread-1 (reused) → runs → returns to pool  ← same thread!
  ...
  Task 10000 → waits in queue → processed when thread available ✅
```

### 🗣️ How to Explain in Interview

> *"SimpleAsyncTaskExecutor is Spring's default async executor — and it's dangerous in production. It creates a brand new thread for every task with no pooling and no upper limit. Under load, this can create thousands of threads, consuming memory and crashing the JVM with OutOfMemoryError. The first thing I do when setting up @Async is configure a proper ThreadPoolTaskExecutor with bounded pool size and queue capacity. SimpleAsyncTaskExecutor is only acceptable for testing or one-off tasks."*

### ⚡ Key Points to Remember

1. Creates **new thread per task** — no pooling
2. **No upper limit** — can exhaust memory
3. **Default** for @Async if none configured ⚠️
4. **Never use in production** ⭐
5. Always replace with **ThreadPoolTaskExecutor**

---

<a id="q86"></a>

## Q86. What is ThreadPoolTaskExecutor?

### 🔑 Quick Answer

> Spring's production-standard thread pool that wraps Java's `ThreadPoolExecutor`. Provides **thread reuse**, **bounded queues**, **configurable pool sizes**, **thread name prefixes**, and **graceful shutdown**. The **only executor you should use in production**.

### 📖 Step-by-Step Explanation

```
ThreadPoolTaskExecutor behavior:

  New task arrives:
  
  1. Active threads < corePoolSize?
     → Create new thread ✅
  
  2. Active threads >= corePoolSize?
     → Put task in queue ✅
  
  3. Queue full AND active threads < maxPoolSize?
     → Create new thread up to maxPoolSize ✅
  
  4. Queue full AND active threads >= maxPoolSize?
     → Rejection policy kicks in ⚠️

  Example: core=5, max=10, queue=100
  
  Tasks 1-5:     → 5 threads created (core)
  Tasks 6-105:   → queued (100 slots)
  Tasks 106-110: → 5 more threads (up to max=10)
  Task 111:      → REJECTED (CallerRunsPolicy → caller thread runs it)
```

### 💻 Code Example

```java
@Bean
public ThreadPoolTaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    
    // Pool sizing
    executor.setCorePoolSize(10);     // Always ready
    executor.setMaxPoolSize(25);      // Burst handling
    executor.setQueueCapacity(500);   // Buffer
    executor.setKeepAliveSeconds(60); // Idle cleanup
    
    // Debugging & monitoring
    executor.setThreadNamePrefix("app-worker-");
    
    // Rejection handling
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    
    // Graceful shutdown
    executor.setWaitForTasksToCompleteOnShutdown(true);
    executor.setAwaitTerminationSeconds(60);
    
    executor.initialize();
    return executor;
}
```

### 🗣️ How to Explain in Interview

> *"ThreadPoolTaskExecutor is the production standard in Spring. It wraps Java's ThreadPoolExecutor with Spring lifecycle management. Core pool size determines always-alive threads. When core threads are busy, tasks go to the bounded queue. If the queue fills up, it creates threads up to max pool size. If everything is maxed out, the rejection policy handles overflow — I use CallerRunsPolicy for natural back-pressure. Thread name prefix is essential for debugging — you can immediately identify which pool a thread belongs to in logs and thread dumps. Graceful shutdown ensures in-flight tasks complete before the application stops."*

### ⚡ Key Points to Remember

1. **Production standard** for Spring applications ⭐
2. Core → Queue → Max → Reject (task handling flow)
3. **Bounded queue** prevents OOM
4. **Thread name prefix** for debugging
5. **Graceful shutdown** for clean application stop

---

> **🎯 Navigation:** [← Performance (Q70-76)](08-performance.md) | [Next → Production Scenarios (Q87-96)](10-production-scenarios.md) | [📋 All Sections](README.md)
