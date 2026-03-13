# 🔴 Spring Batch — Parallel Processing (Q84–Q91)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q84"></a>

## Q84. What are the parallel processing options in Spring Batch?

### 🔑 Quick Answer

> Four approaches: (1) **Multi-threaded Step** — multiple threads process chunks concurrently, (2) **Parallel Steps** — independent steps run simultaneously, (3) **Partitioning** — data split into ranges processed in parallel, (4) **Remote Chunking** — distribute processing across machines.

### 📖 Step-by-Step Explanation

**Step 1 — The four approaches at a glance:**

```
1. MULTI-THREADED STEP (simplest)
   One step, N threads, each thread processes a different chunk
   ┌─────────────────────────────────┐
   │ Step: processOrders             │
   │  Thread-1: chunk 1  ─────→     │
   │  Thread-2: chunk 2  ─────→     │
   │  Thread-3: chunk 3  ─────→     │
   │  Thread-4: chunk 4  ─────→     │
   └─────────────────────────────────┘

2. PARALLEL STEPS (independent steps)
   Multiple independent steps run at the same time
   ┌──── Flow 1 ────┐  ┌──── Flow 2 ────┐
   │ processOrders   │  │ processReturns  │
   └────────────────┘  └─────────────────┘
   Both run simultaneously, then join

3. PARTITIONING (best for large data) ⭐
   Data split into ranges, each partition = own reader/writer
   ┌── Partition 1: ID 1-25K ──┐
   ├── Partition 2: ID 25K-50K ─┤ All run in parallel
   ├── Partition 3: ID 50K-75K ─┤ Each has OWN reader/writer
   └── Partition 4: ID 75K-100K┘

4. REMOTE CHUNKING (distributed)
   Master reads data, sends to remote workers for processing
   ┌── Master ──┐      ┌── Worker 1 (Machine A) ──┐
   │  Reader    │ ───→ │  Processor + Writer       │
   └────────────┘      └───────────────────────────┘
                       ┌── Worker 2 (Machine B) ──┐
                 ───→  │  Processor + Writer       │
                       └───────────────────────────┘
```

**Step 2 — Comparison:**

| Approach | Setup | Restartable? | Thread-safe Reader? | Data Movement |
|----------|-------|-------------|-------------------|--------------|
| Multi-threaded | Easy | ❌ No | ✅ Required | None |
| Parallel Steps | Easy | ✅ Yes | N/A | None |
| Partitioning | Medium | ✅ Yes | Not needed | None (each has own reader) |
| Remote Chunking | Complex | ❌ No | N/A | Data over network |

### 🗣️ How to Explain in Interview

> *"Spring Batch offers four parallel processing approaches. Multi-threaded step is the simplest — add a TaskExecutor and multiple threads process different chunks simultaneously. But the reader must be thread-safe and you lose restartability. Parallel steps are for running independent steps simultaneously — like processing orders and returns at the same time. Partitioning is the most powerful — you divide data into ranges and each partition runs independently with its own reader and writer, so there's no thread-safety concern and it's restartable. Remote chunking distributes work across machines but is the most complex. For production, I recommend partitioning for large datasets."*

### ⚡ Key Points to Remember

1. **Multi-threaded** = simplest but loses restartability
2. **Parallel steps** = independent steps run simultaneously
3. **Partitioning** = best for production (restartable, isolated) ⭐
4. **Remote chunking** = distribute across machines (complex)
5. Choose based on: data size, restartability needs, infrastructure

---

<a id="q85"></a>

## Q85. What is multi-threaded step?

### 🔑 Quick Answer

> Add a `TaskExecutor` to a step, and Spring Batch uses multiple threads to process different chunks concurrently. The reader must be **thread-safe** and you **lose restartability**.

### 📖 Step-by-Step Explanation

**Step 1 — How it works:**

```
Normal (single-threaded):
  Thread-1: Chunk1 → Chunk2 → Chunk3 → Chunk4 → ... → Done
  Total: 200 seconds

Multi-threaded (4 threads):
  Thread-1: Chunk1 → Chunk5 → Chunk9  → ...
  Thread-2: Chunk2 → Chunk6 → Chunk10 → ...
  Thread-3: Chunk3 → Chunk7 → Chunk11 → ...
  Thread-4: Chunk4 → Chunk8 → Chunk12 → ...
  Total: ~50 seconds (4x faster)
```

**Step 2 — Critical requirements:**

```
⚠️ Reader MUST be thread-safe:
  ✅ JdbcPagingItemReader — thread-safe
  ✅ SynchronizedItemStreamReader wrapper — makes any reader thread-safe
  ❌ FlatFileItemReader — NOT thread-safe
  ❌ JdbcCursorItemReader — NOT thread-safe

⚠️ YOU LOSE RESTARTABILITY:
  Multiple threads read out of order → ExecutionContext can't track position
  Job can't resume from exact failure point
```

**Step 3 — Configuration:**

```java
@Bean
public Step multiThreadedStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("processOrders", repo)
            .<Order, Order>chunk(500, tx)
            .reader(pagingReader())          // Must be thread-safe!
            .processor(processor())
            .writer(writer())
            .taskExecutor(taskExecutor())     // Add this for multi-threading
            .throttleLimit(8)                 // Max 8 concurrent threads
            .build();
}

@Bean
public TaskExecutor taskExecutor() {
    SimpleAsyncTaskExecutor executor = new SimpleAsyncTaskExecutor();
    executor.setConcurrencyLimit(8);  // Same as throttleLimit
    return executor;
}
```

### 🗣️ How to Explain in Interview

> *"Multi-threaded step is the simplest parallelism. You add a TaskExecutor to the step and set a throttle limit — say 8 threads. Spring Batch assigns chunks to different threads. The key constraints are: the reader must be thread-safe — I use JdbcPagingItemReader because it's inherently thread-safe — and you lose restartability because multiple threads read items out of order. For most production jobs where restartability is important, I prefer partitioning instead. Multi-threaded step is good for quick wins where restart isn't critical."*

### ⚡ Key Points to Remember

1. Add **TaskExecutor** + **throttleLimit** to enable
2. Reader **must be thread-safe** (use Paging readers)
3. **Loses restartability** (can't track position with multiple threads)
4. Good for **quick performance wins**
5. For production → prefer **partitioning** (restartable)

---

<a id="q86"></a>

## Q86. What is partitioning? How does it work?

### 🔑 Quick Answer

> Partitioning splits data into **ranges** (partitions), and each partition runs as an **independent step execution** with its own reader, processor, and writer. It's **restartable** and the **recommended approach for large datasets**.

### 📖 Step-by-Step Explanation

**Step 1 — The concept:**

```
100,000 records, 4 partitions:

Master Step (Partitioner):
  → Partition 1: ID 1-25,000       → Worker Step with own reader
  → Partition 2: ID 25,001-50,000  → Worker Step with own reader
  → Partition 3: ID 50,001-75,000  → Worker Step with own reader
  → Partition 4: ID 75,001-100,000 → Worker Step with own reader

Each worker is INDEPENDENT:
  - Own database connection
  - Own reader (reads only its range)
  - Own writer
  - Own StepExecution (tracked separately)
  - Own transaction
```

**Step 2 — Why it's better than multi-threaded step:**

| Feature | Multi-threaded | Partitioning |
|---------|---------------|-------------|
| **Restartable** | ❌ No | ✅ Yes (each partition tracked separately) |
| **Thread-safe reader needed** | ✅ Yes | ❌ No (each partition has own reader) |
| **Data isolation** | ❌ Shared reader | ✅ Each partition reads its own range |
| **Monitoring** | 1 StepExecution | N StepExecutions (per partition) |
| **Setup complexity** | Simple | Medium (need Partitioner) |

**Step 3 — How the Partitioner divides data:**

```java
// Partitioner creates ExecutionContext for each partition:
{
  "partition0": ExecutionContext { minId=1, maxId=25000 },
  "partition1": ExecutionContext { minId=25001, maxId=50000 },
  "partition2": ExecutionContext { minId=50001, maxId=75000 },
  "partition3": ExecutionContext { minId=75001, maxId=100000 }
}

// Each worker step reads its range via @StepScope:
@Bean
@StepScope
public JdbcPagingItemReader<Employee> reader(
    @Value("#{stepExecutionContext['minId']}") Long minId,
    @Value("#{stepExecutionContext['maxId']}") Long maxId) {
    
    // This reader only reads records WHERE id BETWEEN minId AND maxId
}
```

### 🗣️ How to Explain in Interview

> *"Partitioning divides data into ranges and processes each range independently. A Partitioner decides the ranges — for example, by ID: partition 1 gets IDs 1-25000, partition 2 gets 25001-50000, and so on. Each partition runs as a separate StepExecution with its own reader, processor, and writer. This is superior to multi-threaded steps because each partition is independently restartable, the readers don't need to be thread-safe since there's no sharing, and you get per-partition monitoring. If partition 3 fails, only partition 3 needs to restart — the others are already committed."*

### 💻 Code Example

```java
// 1. Partitioner — divides data into ranges
@Bean
public Partitioner idRangePartitioner(DataSource ds) {
    return gridSize -> {
        JdbcTemplate jdbc = new JdbcTemplate(ds);
        Long min = jdbc.queryForObject("SELECT MIN(id) FROM employees", Long.class);
        Long max = jdbc.queryForObject("SELECT MAX(id) FROM employees", Long.class);
        long range = (max - min) / gridSize + 1;
        
        Map<String, ExecutionContext> partitions = new HashMap<>();
        long start = min;
        for (int i = 0; i < gridSize; i++) {
            ExecutionContext ctx = new ExecutionContext();
            ctx.putLong("minId", start);
            ctx.putLong("maxId", Math.min(start + range - 1, max));
            partitions.put("partition" + i, ctx);
            start += range;
        }
        return partitions;
    };
}

// 2. Worker step — processes one partition
@Bean
public Step workerStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("worker", repo)
            .<Employee, Employee>chunk(500, tx)
            .reader(partitionReader(null, null))  // Injected at runtime
            .processor(processor())
            .writer(writer())
            .build();
}

// 3. Reader with @StepScope — reads only its partition's range
@Bean
@StepScope
public JdbcPagingItemReader<Employee> partitionReader(
        @Value("#{stepExecutionContext['minId']}") Long minId,
        @Value("#{stepExecutionContext['maxId']}") Long maxId) {
    
    Map<String, Order> sortKeys = Map.of("id", Order.ASCENDING);
    
    return new JdbcPagingItemReaderBuilder<Employee>()
            .name("partitionReader")
            .dataSource(dataSource)
            .selectClause("SELECT *")
            .fromClause("FROM employees")
            .whereClause("WHERE id >= :minId AND id <= :maxId")
            .sortKeys(sortKeys)
            .pageSize(500)
            .parameterValues(Map.of("minId", minId, "maxId", maxId))
            .rowMapper(new EmployeeRowMapper())
            .build();
}

// 4. Master step — orchestrates partitions
@Bean
public Step masterStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("master", repo)
            .partitioner("worker", idRangePartitioner(dataSource))
            .step(workerStep(repo, tx))
            .gridSize(10)                    // 10 partitions
            .taskExecutor(taskExecutor())     // Run in parallel
            .build();
}
```

### ⚡ Key Points to Remember

1. **Partitioner** divides data into ranges
2. Each partition = **independent StepExecution**
3. **Restartable** — failed partition can retry independently
4. **No thread-safety concern** — each partition has own reader
5. **gridSize** = number of partitions (roughly = number of threads)

---

<a id="q87"></a>

## Q87. How does partitioning work internally?

### 🔑 Quick Answer

> The **Partitioner** creates a Map of ExecutionContexts (one per partition). The **Master Step** submits each partition as a separate worker StepExecution to a TaskExecutor. Workers run in parallel, and the Master waits for all to complete.

### 📖 Step-by-Step Explanation

**Step 1 — Internal execution flow:**

```
1. Master Step starts
│
2. Partitioner.partition(gridSize=10) is called
│  Returns: Map<String, ExecutionContext>
│  {
│    "partition0": { minId: 1, maxId: 10000 },
│    "partition1": { minId: 10001, maxId: 20000 },
│    ...
│    "partition9": { minId: 90001, maxId: 100000 }
│  }
│
3. For each partition:
│  ├── Create a StepExecution (with the ExecutionContext)
│  └── Submit to TaskExecutor
│       └── Worker Step executes:
│           ├── reader.open() reads minId/maxId from ExecutionContext
│           ├── Chunk loop: read → process → write → commit
│           └── StepExecution: COMPLETED or FAILED
│
4. Master waits for ALL partitions to finish
│
5. If ALL completed → Master COMPLETED
   If ANY failed → Master FAILED
```

**Step 2 — What each worker sees:**

```
Worker "partition3":
  StepExecution: { name: "worker:partition3" }
  ExecutionContext: { minId: 30001, maxId: 40000 }
  
  Reader: SELECT * FROM employees WHERE id >= 30001 AND id <= 40000
  → processes 10,000 records in chunks of 500
  → 20 chunk commits
  → StepExecution: readCount=10000, writeCount=9800, commitCount=20
```

### 🗣️ How to Explain in Interview

> *"Internally, the Master Step calls the Partitioner which returns a Map of ExecutionContexts — one per partition, each containing the data range for that partition. The Master then submits each partition as a worker StepExecution to a TaskExecutor. Each worker runs independently — creates its own reader with the range from ExecutionContext, processes its data in chunks, and records its own StepExecution stats. The Master waits for all workers to finish. If all succeed, the Master completes. If any fail, the Master fails, but the completed partitions don't need to re-run on restart."*

### ⚡ Key Points to Remember

1. **Partitioner** creates ExecutionContext per partition
2. **Master** submits workers to TaskExecutor
3. Each worker = **independent StepExecution**
4. Workers run **in parallel** (TaskExecutor manages threads)
5. Master **waits** for all workers, then reports overall status

---

<a id="q88"></a>

## Q88. What is remote partitioning?

### 🔑 Quick Answer

> Remote partitioning distributes partitions across **multiple JVMs/machines**. The Master sends only **partition metadata** (not data) over the network. Workers on remote machines read, process, and write independently.

### 📖 Step-by-Step Explanation

```
Machine A (Master):
  Partitioner → creates 20 partitions
  Sends metadata to queue: { partition0: {minId:1, maxId:50K}, ... }

Machine B (Worker 1): Picks up partition 1-5
  Own DB connection → reads records → processes → writes

Machine C (Worker 2): Picks up partition 6-10
  Own DB connection → reads records → processes → writes

Machine D (Worker 3): Picks up partition 11-15
  ...

KEY POINT: Only METADATA moves over network (not actual data)
Workers read data directly from the database
```

**When to use:**

| Local Partitioning | Remote Partitioning |
|-------------------|-------------------|
| 1 machine with 8-16 cores | Multiple machines needed |
| Data fits in one DB connection pool | Need distributed processing |
| < 100M records | 100M+ records |
| Simpler setup | Needs messaging (Kafka/RabbitMQ) |

### 🗣️ How to Explain in Interview

> *"Remote partitioning is like local partitioning but distributed across multiple machines. The Master on one machine creates partitions and sends the metadata — just the ID ranges — over a messaging system like Kafka or RabbitMQ. Workers on other machines pick up these assignments, connect to the database directly, and process their partition independently. The key advantage over remote chunking is that no actual data travels over the network — only small metadata messages. Workers read data directly from the source. This scales horizontally — add more worker machines for more throughput."*

### ⚡ Key Points to Remember

1. Partitions distributed across **multiple machines**
2. Only **metadata** travels over network (not data)
3. Workers read data **directly from source**
4. Needs **messaging infrastructure** (Kafka/RabbitMQ)
5. Use for **100M+ records** that exceed single-machine capacity

---

<a id="q89"></a>

## Q89. What is remote chunking?

### 🔑 Quick Answer

> In remote chunking, the Master **reads data and sends actual items** over the network to remote workers that **process and write** them. Use when processing is CPU-heavy but reading is simple.

### 📖 Step-by-Step Explanation

```
REMOTE CHUNKING:
  Master: READ data
  Send ACTUAL DATA over network → Workers: PROCESS + WRITE

REMOTE PARTITIONING (comparison):
  Master: Create metadata
  Send METADATA over network → Workers: READ + PROCESS + WRITE
```

| Feature | Remote Chunking | Remote Partitioning |
|---------|----------------|-------------------| 
| What travels over network | **Actual data items** | **Only metadata** (ID ranges) |
| Network load | Heavy | Light |
| Master reads data | ✅ Yes | ❌ No |
| Workers read data | ❌ No | ✅ Yes |
| Best for | CPU-heavy processing | I/O-heavy reading |

### 🗣️ How to Explain in Interview

> *"Remote chunking is different from partitioning. In remote chunking, the Master reads the data and sends the actual items over the network to worker machines that process and write them. This is useful when reading is simple but processing is CPU-intensive — like image processing or complex calculations. The downside is network overhead — you're transmitting actual data. In most cases, remote partitioning is preferred because only metadata travels over the network and workers read data directly from the source."*

### ⚡ Key Points to Remember

1. Master **reads**, workers **process + write**
2. **Actual data** travels over network (heavy)
3. Good for **CPU-intensive processing** (image, ML, encryption)
4. Remote partitioning is **usually preferred** (less network load)
5. Needs messaging like **Kafka/RabbitMQ/JMS** for communication

---

<a id="q90"></a>

## Q90. Partitioning vs Multi-threading — which is better?

### 🔑 Quick Answer

> **Partitioning is better for production** — it's restartable, data is isolated per partition, no thread-safety concerns. Multi-threading is simpler to set up but loses restartability.

### 📖 Step-by-Step Explanation

| Feature | Multi-threading | Partitioning |
|---------|----------------|-------------|
| **Setup** | 2 lines of code | Need Partitioner + @StepScope |
| **Restartable** | ❌ No | ✅ Yes |
| **Thread-safe reader** | ✅ Required | ❌ Not needed |
| **Data isolation** | ❌ Shared reader | ✅ Isolated per partition |
| **Monitoring** | 1 StepExecution | N StepExecutions |
| **Failure impact** | Entire step fails | Only failed partition retries |
| **Production ready** | ⚠️ Limited | ✅ Production recommended |

**Decision guide:**

```
Quick job, restart not needed, thread-safe reader available?
  → Multi-threading (simpler)

Production job, restart important, large data, need monitoring?
  → Partitioning (always recommended) ⭐

Not sure?
  → Default to Partitioning
```

### 🗣️ How to Explain in Interview

> *"For production, I always recommend partitioning over multi-threading. Partitioning gives you restartability — if partition 5 out of 10 fails, only partition 5 restarts. With multi-threading, the entire step restarts from the beginning. Partitioning gives data isolation — each partition has its own reader, so there's no thread-safety concern. And you get per-partition monitoring with separate StepExecutions. The only advantage of multi-threading is simplicity — it's two lines of config. But the benefits of partitioning far outweigh the setup effort."*

### ⚡ Key Points to Remember

1. **Partitioning** = production standard ⭐
2. **Multi-threading** = quick wins, non-critical jobs
3. Partitioning: **restartable**, **isolated**, **monitorable**
4. Multi-threading: **simpler** but loses restartability
5. When in doubt → **choose partitioning**

---

<a id="q91"></a>

## Q91. How do you configure parallel step execution?

### 🔑 Quick Answer

> Use `FlowBuilder` with `.split()` to run independent steps simultaneously. Each flow runs in its own thread, and the job continues after all parallel flows complete.

### 📖 Step-by-Step Explanation

**Step 1 — Parallel steps (independent flows):**

```
Normal (sequential):
  Step1 → Step2 → Step3 → Step4
  Total: 40 min (10+10+10+10)

Parallel (Steps 2 and 3 are independent):
  Step1 → [Step2 ──────] → Step4
           [Step3 ──────]
  Total: 30 min (10 + max(10,10) + 10)
```

### 💻 Code Example

```java
@Bean
public Job parallelJob(JobRepository repo, PlatformTransactionManager tx) {
    
    // Flow 1: Process orders
    Flow orderFlow = new FlowBuilder<SimpleFlow>("orderFlow")
            .start(processOrdersStep(repo, tx))
            .build();
    
    // Flow 2: Process returns (independent of orders)
    Flow returnFlow = new FlowBuilder<SimpleFlow>("returnFlow")
            .start(processReturnsStep(repo, tx))
            .build();
    
    // Run both in parallel, then continue with report
    return new JobBuilder("parallelJob", repo)
            .start(validateStep(repo, tx))           // Step 1: Sequential
            .next(orderFlow)                          // Step 2+3: Start parallel
            .split(new SimpleAsyncTaskExecutor())      // Run simultaneously
            .add(returnFlow)                          // Add second parallel flow
            .end()                                    // Wait for both to finish
            .next(generateReportStep(repo, tx))       // Step 4: Sequential (after both done)
            .build()
            .build();
}
```

**What happens:**
```
1. validateStep runs (single thread)
2. processOrdersStep + processReturnsStep run SIMULTANEOUSLY (2 threads)
3. Job waits for BOTH to complete
4. generateReportStep runs (single thread)
```

### 🗣️ How to Explain in Interview

> *"To run steps in parallel, I use FlowBuilder with split(). Each independent step is wrapped in a Flow, and the split method runs them simultaneously using a TaskExecutor. The job waits for all parallel flows to complete before moving to the next step. This is useful when steps are completely independent — like processing orders and processing returns at the same time. It's different from multi-threaded steps where multiple threads process chunks within one step."*

### ⚡ Key Points to Remember

1. Use `FlowBuilder` + `split()` for parallel steps
2. Steps must be **completely independent** (no shared data)
3. Job waits for **all parallel flows** before continuing
4. Different from multi-threaded step (that's within ONE step)
5. Good for **independent data pipelines**

---

> **🎯 Navigation:** [← Tasklet (Q79-83)](09-tasklet.md) | [Next → Performance (Q92-98)](11-performance.md) | [📋 All Sections](README.md)
