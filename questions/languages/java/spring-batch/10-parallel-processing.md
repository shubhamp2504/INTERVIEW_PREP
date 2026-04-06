# ⚡ Parallel Processing — Q84 to Q91

---

## Q84. What are the parallel processing options in Spring Batch?

### 📝 One-Liner
Four options: multi-threaded step, parallel steps, partitioning (recommended for production), and remote chunking.

### 🔑 Quick Answer
**(1) Multi-threaded step** — multiple threads process chunks concurrently in the same JVM. Simple but loses restartability. **(2) Parallel steps** — independent steps run simultaneously (e.g., process orders AND invoices at the same time). **(3) Partitioning** — data split into ranges, each processed by an independent worker with its own reader/writer. Best for production (restartable, isolated). **(4) Remote chunking** — read on master, send items to remote workers for processing/writing. *(Production ke liye partitioning sabse best — restartable, isolated, monitorable)*

### 📖 How It Works
```
Scaling Options:

1. Multi-threaded Step:
   ┌─────────────────────────┐
   │ Step with TaskExecutor  │
   │ Thread 1: chunk A       │
   │ Thread 2: chunk B       │ Same JVM, same reader
   │ Thread 3: chunk C       │ Reader must be thread-safe
   └─────────────────────────┘

2. Parallel Steps:
   ┌──── Step A (orders) ────┐
   │                         │ Independent steps
   ├──── Step B (invoices) ──┤ run at same time
   │                         │
   └──── Step C (reports) ───┘

3. Partitioning ⭐ (recommended):
   Master Step
   ├── Partition 1: IDs 1-10000      (own reader/writer)
   ├── Partition 2: IDs 10001-20000  (own reader/writer)
   ├── Partition 3: IDs 20001-30000  (own reader/writer)
   └── Each = independent StepExecution (restartable!)

4. Remote Chunking:
   Master (reads) ──items──→ Worker JVM 1 (process+write)
                   ──items──→ Worker JVM 2 (process+write)
```

### 🗣️ Answering Approach
"Spring Batch offers four scaling strategies. Multi-threaded step is simplest — add a TaskExecutor and multiple threads process chunks concurrently. But it loses restartability because threads read in unpredictable order. Parallel steps run independent steps simultaneously — great when steps don't depend on each other. Partitioning is the production standard — it splits data into ranges and each partition gets its own reader, writer, and StepExecution, making it restartable and isolated. Remote chunking distributes actual data to remote workers but adds network overhead. In my project, we used partitioning with 8 threads for our 10-million-record payment job, reducing processing time from 4 hours to 30 minutes."

### 💻 Code
```java
// Quick overview — detailed code in individual questions

// 1. Multi-threaded step
.taskExecutor(new SimpleAsyncTaskExecutor())
.throttleLimit(4)  // max 4 threads

// 2. Parallel steps
new FlowBuilder<SimpleFlow>("parallelFlow")
    .split(new SimpleAsyncTaskExecutor())
    .add(flow1, flow2, flow3)
    .build();

// 3. Partitioning (recommended)
new StepBuilder("masterStep", repo)
    .partitioner("workerStep", rangePartitioner())
    .step(workerStep)
    .gridSize(8)
    .taskExecutor(taskExecutor())
    .build();
```

### 🆚 vs. Comparison
| Option | Restartable | Thread-safe? | Complexity | Best For |
|--------|------------|-------------|------------|---------|
| Multi-threaded | ❌ No | Reader must be | Low | Quick wins |
| Parallel steps | ✅ Yes | N/A (separate) | Low | Independent steps |
| Partitioning ⭐ | ✅ Yes | No concern | Medium | Production scaling |
| Remote chunking | ❌ Limited | N/A (separate) | High | CPU-intensive work |

### ⚡ Remember
- Multi-threaded: simple but loses restart *(asaan hai lekin restart nahi milta)*
- Parallel steps: for independent steps only
- **Partitioning**: production standard (restartable + isolated) ⭐
- Remote chunking: for distributed processing across machines
- Default to partitioning for production jobs

### 🔗 Follow-ups
- [Q85 → Multi-threaded step details](#q85)
- [Q86 → Partitioning details](#q86)
- [Q90 → Partitioning vs Multi-threading comparison](#q90)

---

## Q85. What is multi-threaded step?

### 📝 One-Liner
Multi-threaded step adds a TaskExecutor to a chunk step so multiple threads process different chunks concurrently — simple but loses restartability.

### 🔑 Quick Answer
By default, Spring Batch processes chunks sequentially in one thread. Adding a `TaskExecutor` to the step makes multiple threads process chunks concurrently. Reader MUST be thread-safe (use Paging readers, not Cursor readers). Drawback: loses restartability because threads read items in unpredictable order, so checkpoint positions are unreliable. Simple config for quick performance wins, but partitioning is better for production. *(TaskExecutor lagao toh multiple threads chunks process karenge — lekin restart nahi milega)*

### 📖 How It Works
```
Single-threaded (default):
  Thread-1: chunk1 → chunk2 → chunk3 → chunk4 → ...

Multi-threaded (with TaskExecutor):
  Thread-1: chunk1 → chunk5 → chunk9  → ...
  Thread-2: chunk2 → chunk6 → chunk10 → ...
  Thread-3: chunk3 → chunk7 → chunk11 → ...
  Thread-4: chunk4 → chunk8 → chunk12 → ...

Reader thread-safety:
  JdbcCursorItemReader: ❌ NOT safe → SynchronizedItemStreamReader wrapper
  JdbcPagingItemReader: ✅ Thread-safe
  FlatFileItemReader:   ❌ NOT safe → SynchronizedItemStreamReader wrapper
```

### 🗣️ Answering Approach
"Multi-threaded step adds a TaskExecutor so multiple threads process chunks concurrently. The reader must be thread-safe — I use JdbcPagingItemReader which is inherently thread-safe. The main drawback is loss of restartability because threads read items in unpredictable order, breaking checkpoint accuracy. In my project, we used it for a one-off data migration where restartability wasn't critical — we could easily re-run the entire job. For our daily production jobs, we used partitioning instead because restart support was essential."

### 💻 Code
```java
@Bean
public Step multiThreadedStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("multiThreadedStep", repo)
            .<Order, ProcessedOrder>chunk(500, tx)
            .reader(pagingReader())         // MUST be thread-safe!
            .processor(orderProcessor())
            .writer(dbWriter())
            .taskExecutor(taskExecutor())   // enable multi-threading
            .throttleLimit(4)                // max 4 concurrent threads
            .build();
}

@Bean
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(4);
    executor.setMaxPoolSize(8);
    executor.setQueueCapacity(25);
    executor.setThreadNamePrefix("batch-thread-");
    executor.initialize();
    return executor;
}

// Making non-thread-safe reader thread-safe (wrapper)
@Bean
public SynchronizedItemStreamReader<Order> synchronizedReader() {
    SynchronizedItemStreamReader<Order> syncReader = new SynchronizedItemStreamReader<>();
    syncReader.setDelegate(flatFileReader());  // wraps non-thread-safe reader
    return syncReader;
}
```

### ⚠️ Pitfalls / Gotchas
- Loses restartability — threads read in unpredictable order *(restart kaam nahi karta — thread ka order predict nahi ho sakta)*
- Reader MUST be thread-safe — Cursor readers need SynchronizedItemStreamReader wrapper
- SynchronizedItemStreamReader serializes reads → reduces multi-threading benefit
- Writer is already thread-safe (receives separate chunks per thread)
- `throttleLimit` is deprecated in Batch 5 — use thread pool size instead

### 🆚 vs. Comparison
| Aspect | Multi-threaded Step | Partitioning |
|--------|-------------------|-------------|
| Restartable | ❌ No | ✅ Yes |
| Thread-safety | Reader must be safe | No concern (each partition has own reader) |
|Config | Simple (add TaskExecutor) | Medium (Partitioner + worker step) |
| Failure isolation | Entire step fails | Only failed partition retries |
| Monitoring | One StepExecution | N StepExecutions (1 per partition) |

### ⚡ Remember
- Add TaskExecutor + throttleLimit to enable *(TaskExecutor lagao = multi-threaded)*
- Reader MUST be thread-safe (use Paging readers)
- Loses restartability
- Simple config, good for quick wins
- Production → prefer partitioning

### 🔗 Follow-ups
- [Q86 → Partitioning (production alternative)](#q86)
- [Q90 → Partitioning vs Multi-threading](#q90)
- [Q34 → JdbcPagingItemReader (thread-safe)](#q34)

---

## Q86. What is partitioning? How does it work?

### 📝 One-Liner
Partitioning splits data into non-overlapping ranges (partitions) and processes each range in parallel with independent readers, writers, and StepExecutions — the production standard for scaling.

### 🔑 Quick Answer
Partitioning: a **Partitioner** splits data into N ranges (e.g., IDs 1-10K, 10K-20K, 20K-30K). Each partition gets its own `ExecutionContext` with range parameters. A **master step** submits partitions as independent **worker steps**, each with its own reader (scoped to the range), writer, and StepExecution. Workers run in parallel via TaskExecutor. Key advantages: restartable (only failed partitions retry), isolated (one partition failure doesn't affect others), monitorable (per-partition metrics). *(Data ko ranges mein baanto, har range ko alag thread mein process karo — restart bhi ho sakta hai)*

### 📖 How It Works
```
Partitioning Architecture:

Master Step (orchestrator):
  Partitioner creates:
    ├── Partition 0: {minId: 1,     maxId: 10000}
    ├── Partition 1: {minId: 10001, maxId: 20000}
    ├── Partition 2: {minId: 20001, maxId: 30000}
    └── Partition 3: {minId: 30001, maxId: 40000}

  TaskExecutor runs 4 workers in parallel:
    Worker 0: reader(1-10K)     → process → writer → StepExecution #1 ✅
    Worker 1: reader(10K-20K)   → process → writer → StepExecution #2 ✅ 
    Worker 2: reader(20K-30K)   → process → writer → StepExecution #3 ❌ FAILED
    Worker 3: reader(30K-40K)   → process → writer → StepExecution #4 ✅

  On restart: ONLY Partition 2 re-runs (StepExecution #3)

Benefits:
  ✅ Restartable (per-partition)
  ✅ Isolated (partition failure doesn't affect others)
  ✅ No thread-safety concerns (each has own reader/writer)
  ✅ Per-partition metrics and monitoring
```

### 🗣️ Answering Approach
"Partitioning is the production standard for scaling Spring Batch jobs. A Partitioner splits data into non-overlapping ranges — each partition gets its own ExecutionContext with range parameters. Worker steps run in parallel, each with its own reader scoped to its range. The key advantages over multi-threading are restartability and isolation — if one partition fails, only that partition retries on restart, and other partitions' work is preserved. In my project, we partitioned our 10-million-record payment job by customer ID ranges across 8 threads. Processing time dropped from 4 hours to 30 minutes, and when a partition failed due to a data issue, restart only re-processed that specific range."

### 💻 Code
```java
// Partitioner: splits data into ranges
@Component
public class IdRangePartitioner implements Partitioner {
    @Autowired private JdbcTemplate jdbc;

    @Override
    public Map<String, ExecutionContext> partition(int gridSize) {
        Long min = jdbc.queryForObject("SELECT MIN(id) FROM orders", Long.class);
        Long max = jdbc.queryForObject("SELECT MAX(id) FROM orders", Long.class);
        long range = (max - min) / gridSize + 1;

        Map<String, ExecutionContext> partitions = new HashMap<>();
        for (int i = 0; i < gridSize; i++) {
            ExecutionContext ctx = new ExecutionContext();
            ctx.putLong("minId", min + (i * range));
            ctx.putLong("maxId", min + ((i + 1) * range) - 1);
            partitions.put("partition" + i, ctx);
        }
        return partitions;
    }
}

// Master step
@Bean
public Step masterStep(JobRepository repo, Step workerStep) {
    return new StepBuilder("masterStep", repo)
            .partitioner("workerStep", idRangePartitioner())
            .step(workerStep)
            .gridSize(8)          // 8 partitions
            .taskExecutor(taskExecutor())
            .build();
}

// Worker step (scoped reader uses partition range)
@Bean
public Step workerStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("workerStep", repo)
            .<Order, ProcessedOrder>chunk(500, tx)
            .reader(scopedReader(null, null))
            .processor(processor())
            .writer(writer())
            .build();
}

@Bean
@StepScope
public JdbcPagingItemReader<Order> scopedReader(
        @Value("#{stepExecutionContext['minId']}") Long minId,
        @Value("#{stepExecutionContext['maxId']}") Long maxId) {
    return new JdbcPagingItemReaderBuilder<Order>()
            .name("partitionReader")
            .dataSource(dataSource)
            .selectClause("SELECT *")
            .fromClause("FROM orders")
            .whereClause("WHERE id BETWEEN :minId AND :maxId")
            .sortKeys(Map.of("id", Order.ASCENDING))
            .parameterValues(Map.of("minId", minId, "maxId", maxId))
            .pageSize(500)
            .rowMapper(new BeanPropertyRowMapper<>(Order.class))
            .build();
}

@Bean
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(8);
    executor.setMaxPoolSize(8);
    executor.initialize();
    return executor;
}
```

### ⚠️ Pitfalls / Gotchas
- Reader and writer MUST be `@StepScope` — each partition needs its own instance *(StepScope zaruri hai — har partition ko apna reader chahiye)*
- Partitioner must create non-overlapping ranges (no duplicates)
- gridSize should match available threads/cores
- Data skew (uneven partition sizes) reduces effectiveness — consider even data distribution
- Writer must handle concurrent writes (same table from multiple threads)

### 🎯 Tricky Interview Qs

**Q: How does partitioned step restart work?**
Each partition has its own StepExecution. On restart, only FAILED partitions re-run. COMPLETED partitions are skipped. Each partition's checkpoint is independent.

**Q: What if data is unevenly distributed across partitions?**
Faster partitions finish and idle while the largest one continues. Solutions: use more partitions than threads (dynamic scheduling), or partition by hash instead of range for even distribution.

### ⚡ Remember
- Partitioner splits data → each partition has own reader/writer/StepExecution
- **Restartable**: only failed partitions retry *(sirf fail partition restart hota hai)*
- **Isolated**: one failure doesn't affect others
- Reader/Writer must be @StepScope
- gridSize = number of partitions (match to thread count)

### 🔗 Follow-ups
- [Q87 → Partitioning internal flow](#q87)
- [Q88 → Remote partitioning](#q88)
- [Q90 → Partitioning vs Multi-threading](#q90)

---

## Q87. How does partitioning work internally?

### 📝 One-Liner
Partitioner creates a Map of ExecutionContexts (one per partition), master submits each as a separate worker StepExecution via TaskExecutor, and waits for all to complete.

### 🔑 Quick Answer
Internally: **(1)** `Partitioner.partition(gridSize)` creates a `Map<String, ExecutionContext>` — each entry defines one partition's range. **(2)** The master step handler (`TaskExecutorPartitionHandler`) creates a `StepExecution` for each partition. **(3)** Each StepExecution is submitted to the `TaskExecutor` as a concurrent task. **(4)** Workers run in parallel, each executing the worker step with its partition's ExecutionContext. **(5)** Master waits for all workers to complete. If all succeed → master COMPLETED. If any fail → master FAILED. *(Partitioner map banata hai → master threads ko submit karta hai → sab parallel chalte hain → master wait karta hai)*

### 📖 How It Works
```
Internal Flow:

Step 1: Partitioner creates Map
  partition("partition0") → ExecutionContext{minId: 1, maxId: 10000}
  partition("partition1") → ExecutionContext{minId: 10001, maxId: 20000}
  partition("partition2") → ExecutionContext{minId: 20001, maxId: 30000}

Step 2: PartitionHandler creates StepExecutions
  StepExecution#1 (partition0) → context: {minId: 1, maxId: 10000}
  StepExecution#2 (partition1) → context: {minId: 10001, maxId: 20000}
  StepExecution#3 (partition2) → context: {minId: 20001, maxId: 30000}

Step 3: Submit to TaskExecutor
  Thread-1 → execute workerStep with StepExecution#1
  Thread-2 → execute workerStep with StepExecution#2
  Thread-3 → execute workerStep with StepExecution#3

Step 4: Workers run with @StepScope reader
  Each reader gets minId/maxId from its StepExecution's context
  → reads only its range of data

Step 5: Master aggregates results
  All COMPLETED → master COMPLETED ✅
  Any FAILED → master FAILED ❌
```

### 🗣️ Answering Approach
"Internally, the Partitioner creates a Map of ExecutionContexts, each defining a data range for one partition. The PartitionHandler creates a StepExecution for each partition and submits them to the TaskExecutor for parallel execution. Each worker step initializes its StepScope reader with the partition's range from the ExecutionContext. Workers run completely independently — each with its own reader, processor, writer, and StepExecution. The master waits for all workers to finish and aggregates the results. This independence is what makes partitioning restartable — each partition's StepExecution tracks its own progress independently."

### 💻 Code
```java
// Internal classes involved (you don't write these — but knowing helps)

// 1. Partitioner creates Map<String, ExecutionContext>
Map<String, ExecutionContext> partitions = partitioner.partition(gridSize);
// returns: {"partition0": {minId:1, maxId:10K}, "partition1": {minId:10K, maxId:20K}, ...}

// 2. TaskExecutorPartitionHandler manages the flow
@Bean
public TaskExecutorPartitionHandler partitionHandler() {
    TaskExecutorPartitionHandler handler = new TaskExecutorPartitionHandler();
    handler.setStep(workerStep());
    handler.setTaskExecutor(taskExecutor());
    handler.setGridSize(8);
    return handler;
}

// 3. Master step uses the handler
@Bean
public Step masterStep(JobRepository repo) {
    return new StepBuilder("masterStep", repo)
            .partitioner("workerStep", partitioner())
            .partitionHandler(partitionHandler())
            .build();
}

// 4. Worker step with @StepScope beans
@Bean
public Step workerStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("workerStep", repo)
            .<Order, Order>chunk(500, tx)
            .reader(partitionedReader(null, null))  // @StepScope injects range
            .writer(writer())
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- gridSize determines number of partitions, NOT number of threads — thread pool size controls concurrency *(gridSize = partition count, thread pool = actual parallel threads)*
- If gridSize > thread pool size, some partitions wait in queue
- Master step doesn't process data — only orchestrates
- All partition metadata stored in DB — visible in BATCH_STEP_EXECUTION

### ⚡ Remember
- Partitioner → Map of ExecutionContexts
- PartitionHandler → creates StepExecutions → submits to TaskExecutor *(handler threads ko manage karta hai)*
- Workers run independently with @StepScope readers
- Master waits for all → aggregates results
- Each partition = independent StepExecution

### 🔗 Follow-ups
- [Q86 → Partitioning overview](#q86)
- [Q88 → Remote partitioning](#q88)
- [Q89 → Remote chunking](#q89)

---

## Q88. What is remote partitioning?

### 📝 One-Liner
Remote partitioning distributes partition execution across multiple JVMs/machines — only metadata (partition definitions) travels over the network, not actual data.

### 🔑 Quick Answer
Remote partitioning extends local partitioning across machines. The master creates partitions and sends partition metadata (not data) to remote workers via messaging (Kafka, RabbitMQ). Each remote worker reads data directly from the source, processes, and writes. Data never travels over the network — only small partition definitions do. Useful for scaling beyond a single JVM when you need more memory and CPU. *(Data network pe nahi jaata — sirf partition definition jaata hai, worker khud data padhta hai)*

### 📖 How It Works
```
Remote Partitioning:

Master JVM:
  Partitioner → {part0: minId=1..10K, part1: minId=10K..20K, ...}
  → Send partition METADATA via Kafka/RabbitMQ

                     ↓ metadata only (small)

Worker JVM 1: receives part0 →  reads data (1..10K) from source DB → process → write
Worker JVM 2: receives part1 →  reads data (10K..20K) from source DB → process → write
Worker JVM 3: receives part2 →  reads data (20K..30K) from source DB → process → write

Workers read directly from source → NO data over network
Master monitors via polling or messaging reply
```

### 🗣️ Answering Approach
"Remote partitioning distributes partition execution across multiple JVMs. The master creates partition definitions and sends them to remote workers via messaging middleware like Kafka or RabbitMQ. The critical point is that only metadata travels over the network — partition boundaries like min/max IDs, not actual data. Each worker reads data directly from the source database, processes it, and writes the results. This is efficient for I/O-bound workloads because you're scaling database connections and processing power without serializing data over the network."

### 💻 Code
```java
// Master side: send partition definitions
@Bean
public Step masterStep(JobRepository repo) {
    return new StepBuilder("masterStep", repo)
            .partitioner("workerStep", rangePartitioner())
            .partitionHandler(messagePartitionHandler())  // sends over messaging
            .build();
}

@Bean
public MessageChannelPartitionHandler messagePartitionHandler() {
    MessageChannelPartitionHandler handler = new MessageChannelPartitionHandler();
    handler.setStepName("workerStep");
    handler.setGridSize(4);
    handler.setMessagingOperations(messagingTemplate);  // Kafka/RabbitMQ
    return handler;
}

// Worker side: receives partition and processes
// Workers run as separate Spring Boot applications
// Connected to same JobRepository database
```

### ⚠️ Pitfalls / Gotchas
- Requires shared JobRepository database (all workers + master must access same metadata DB) *(sab workers ko same metadata database access chahiye)*
- Messaging infrastructure needed (Kafka, RabbitMQ, ActiveMQ)
- More complex setup than local partitioning
- Worker JVMs must have same Spring Batch job configuration

### 🆚 vs. Comparison
| Aspect | Local Partitioning | Remote Partitioning |
|--------|-------------------|-------------------|
| Workers | Same JVM (threads) | Different JVMs/machines |
| Network | No network | Only metadata over network |
| Data | Local reads | Workers read from source |
| Infra | Just thread pool | Messaging + shared DB |
| Scale | Single machine | Multiple machines |

### ⚡ Remember
- Only METADATA travels over network (not data) *(data nahi, sirf metadata jaata hai)*
- Workers read data directly from source
- Requires shared JobRepository + messaging infrastructure
- Use when single JVM not enough for performance
- Extension of local partitioning pattern

### 🔗 Follow-ups
- [Q86 → Local partitioning basics](#q86)
- [Q89 → Remote chunking (sends actual data)](#q89)
- [Q90 → Partitioning vs Multi-threading](#q90)

---

## Q89. What is remote chunking?

### 📝 One-Liner
Remote chunking sends actual data items from a master reader to remote worker JVMs for processing and writing — heavy network load but offloads CPU-intensive work.

### 🔑 Quick Answer
In remote chunking, the master JVM reads data and sends the actual items to remote workers via messaging. Workers process and write the items. Unlike remote partitioning (where only metadata travels), remote chunking sends FULL item data over the network. Use when reading is cheap but processing is expensive (CPU-intensive: image processing, ML inference, encryption). Downside: heavy network load from serializing and transferring data. *(Master padhta hai, actual data bhejta hai remote workers ko — network pe zyada load)*

### 📖 How It Works
```
Remote Chunking:

Master JVM:
  Reader reads items → serializes → sends ACTUAL DATA via messaging
                                        ↓ heavy network traffic!
Worker JVM 1: receives items → process() → write() → result back
Worker JVM 2: receives items → process() → write() → result back

Remote Chunking vs Remote Partitioning:
  Remote Chunking:     Master reads → sends ITEMS    → Workers process+write
  Remote Partitioning: Master sends METADATA only → Workers read+process+write
```

### 🗣️ Answering Approach
"Remote chunking distributes the processing and writing work to remote JVMs while the master handles reading. The master reads items and sends the actual data over messaging to workers for processing and writing. Unlike remote partitioning where only partition metadata travels, remote chunking sends full item data over the network. It's ideal when reading is fast but processing is CPU-intensive — like image processing or ML inference. In practice I prefer remote partitioning because it avoids the network overhead, but remote chunking makes sense when the data source can only be read from one location."

### 💻 Code
```java
// Master side: reads and sends items to workers
@Bean
public Step masterChunkStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("masterStep", repo)
            .<Order, Order>chunk(500, tx)
            .reader(orderReader())                // master reads
            .writer(chunkMessageWriter())          // sends items to workers
            .build();
}

@Bean
public ChunkMessageChannelItemWriter<Order> chunkMessageWriter() {
    ChunkMessageChannelItemWriter<Order> writer = new ChunkMessageChannelItemWriter<>();
    writer.setMessagingOperations(messagingTemplate);
    writer.setReplyChannel(replies);
    return writer;
}

// Worker side: receives items, processes and writes
@Bean
public IntegrationFlow workerFlow() {
    return IntegrationFlow.from(requestChannel)
            .handle(chunkProcessorChunkHandler())
            .get();
}
```

### ⚠️ Pitfalls / Gotchas
- Heavy network load — all items serialized and sent over wire *(actual data jaata hai — network pe zyada load)*
- Items must be Serializable
- Master is single reader → can be bottleneck
- Remote partitioning is preferred in most cases
- Complex setup with Spring Integration

### 🆚 vs. Comparison
| Aspect | Remote Chunking | Remote Partitioning |
|--------|----------------|-------------------|
| Network data | FULL items (heavy) | Only metadata (light) |
| Reader location | Master only | Each worker reads |
| Use case | CPU-intensive processing | I/O-intensive processing |
| Bottleneck | Master reader + network | Database access |
| Complexity | High | Medium |

### ⚡ Remember
- Master reads → sends ACTUAL DATA to workers *(actual items network pe jaate hain)*
- Workers process + write
- Heavy network traffic (vs metadata-only in remote partitioning)
- Best for: CPU-intensive work (images, ML, encryption)
- Prefer remote partitioning in most cases

### 🔗 Follow-ups
- [Q88 → Remote partitioning (preferred)](#q88)
- [Q86 → Local partitioning](#q86)
- [Q84 → All parallel options](#q84)

---

## Q90. Partitioning vs Multi-threading — which is better?

### 📝 One-Liner
Partitioning is better for production — restartable, isolated, monitorable; multi-threading is simpler but loses restartability.

### 🔑 Quick Answer
**Partitioning wins for production**: (1) Restartable — only failed partitions retry. (2) Isolated — one partition's failure doesn't affect others. (3) No thread-safety concerns — each partition has its own reader/writer. (4) Monitorable — per-partition StepExecution with metrics. **Multi-threading wins for simplicity**: just add TaskExecutor — but loses restart and the reader must be thread-safe. *(Production mein hamesha partitioning use karo — restart, isolation, monitoring sab milta hai)*

### 📖 How It Works
```
Failure Handling Comparison:

Multi-threaded Step:
  Thread-1: chunks 1,5,9    ✅
  Thread-2: chunks 2,6,10   ❌ FAILED at chunk 6
  Thread-3: chunks 3,7,11   ✅
  Thread-4: chunks 4,8,12   ✅
  → ENTIRE step FAILED → must re-process ALL data on restart

Partitioned Step:
  Partition-1: IDs 1-10K    ✅ COMPLETED (safe)
  Partition-2: IDs 10K-20K  ❌ FAILED   (only this retries)
  Partition-3: IDs 20K-30K  ✅ COMPLETED (safe)
  Partition-4: IDs 30K-40K  ✅ COMPLETED (safe)
  → Only Partition-2 re-runs on restart
```

### 🗣️ Answering Approach
"For production, partitioning is the clear winner. Multi-threading is simpler to configure — just add a TaskExecutor — but it loses restartability because threads read items in unpredictable order. If the step fails, you have to re-process everything. With partitioning, each partition has its own StepExecution, so on restart only failed partitions re-run. There's also no thread-safety concern because each partition has its own reader and writer. In my project, we switched from multi-threaded to partitioned processing and gained reliable restart capability — a failed partition affecting 12K records didn't require re-processing the other 988K records."

### 💻 Code
```java
// Multi-threaded: simple but no restart
@Bean
public Step multiThreadedStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("mtStep", repo)
            .<Order, Order>chunk(500, tx)
            .reader(pagingReader()) // must be thread-safe!
            .writer(writer())
            .taskExecutor(new SimpleAsyncTaskExecutor())
            .build();
    // ❌ No reliable restart
}

// Partitioned: production standard
@Bean
public Step partitionedStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("masterStep", repo)
            .partitioner("worker", rangePartitioner())
            .step(workerStep(repo, tx))
            .gridSize(8)
            .taskExecutor(taskExecutor())
            .build();
    // ✅ Restartable, isolated, monitorable
}
```

### 🆚 vs. Comparison
| Aspect | Multi-threaded | Partitioning ⭐ |
|--------|---------------|----------------|
| Config | Simple (TaskExecutor) | Medium (Partitioner + worker) |
| Restartable | ❌ No | ✅ Yes (per partition) |
| Thread-safety | Reader must be safe | No concern |
| Failure | Entire step fails | Only failed partition retries |
| Monitoring | One StepExecution | N StepExecutions |
| Production use | Quick wins only | Standard ⭐ |

### ⚡ Remember
- **Multi-threaded**: simple, no restart, reader must be thread-safe
- **Partitioning**: production standard, restartable, isolated *(partitioning = production ka standard — hamesha prefer karo)*
- Failed multi-threaded → re-process ALL; Failed partition → re-process ONLY that partition
- Default to partitioning for production jobs

### 🔗 Follow-ups
- [Q85 → Multi-threaded step details](#q85)
- [Q86 → Partitioning details](#q86)
- [Q84 → All parallel options](#q84)

---

## Q91. How do you configure parallel step execution?

### 📝 One-Liner
Use `FlowBuilder.split()` with a TaskExecutor to run independent steps simultaneously in parallel.

### 🔑 Quick Answer
Parallel steps execute independent steps at the same time using `split()`. Each step runs in a separate thread. Steps must be truly independent — no data dependency between them. After all parallel steps complete, the job continues to the next step. Use `SimpleAsyncTaskExecutor` or a thread pool. Only put steps in parallel if they don't depend on each other's output. *(Independent steps ko parallel mein chalao — split() use karo TaskExecutor ke saath)*

### 📖 How It Works
```
Sequential (default):
  Step A → Step B → Step C → Step D
  Time: A + B + C + D

Parallel:
  ┌── Step A ──┐
  │            │
  ├── Step B ──┤ ← run at same time
  │            │
  └── Step C ──┘
       ↓
     Step D      ← runs after all parallel steps complete

  Time: max(A, B, C) + D  ← much faster!
```

### 🗣️ Answering Approach
"For parallel step execution, I use FlowBuilder with split() and a TaskExecutor. This runs independent steps simultaneously — for example, processing orders and invoices at the same time since they don't depend on each other. The job continues to the next step only after all parallel steps complete. In my project, we had three independent data loads — customers, products, and inventory — that we parallelized, reducing total job time from 3 hours to 1 hour since each load took roughly an hour."

### 💻 Code
```java
@Bean
public Job parallelJob(JobRepository repo, Step setupStep,
                       Step ordersStep, Step invoicesStep, Step reportsStep,
                       Step summaryStep) {
    // Define flows for parallel execution
    Flow ordersFlow = new FlowBuilder<SimpleFlow>("ordersFlow")
            .start(ordersStep).build();
    Flow invoicesFlow = new FlowBuilder<SimpleFlow>("invoicesFlow")
            .start(invoicesStep).build();
    Flow reportsFlow = new FlowBuilder<SimpleFlow>("reportsFlow")
            .start(reportsStep).build();

    // Split: run 3 flows in parallel
    Flow parallelFlow = new FlowBuilder<SimpleFlow>("parallelFlow")
            .split(new SimpleAsyncTaskExecutor())  // parallel execution
            .add(ordersFlow, invoicesFlow, reportsFlow)
            .build();

    return new JobBuilder("parallelJob", repo)
            .start(setupStep)       // runs first (sequential)
            .next(parallelFlow)     // 3 steps run in parallel
            .next(summaryStep)      // runs after all parallel complete
            .build();
}

// With thread pool (better control)
@Bean
public Job parallelJobWithPool(JobRepository repo, 
                                Step ordersStep, Step invoicesStep) {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(3);
    executor.setMaxPoolSize(3);
    executor.initialize();

    Flow flow1 = new FlowBuilder<SimpleFlow>("f1").start(ordersStep).build();
    Flow flow2 = new FlowBuilder<SimpleFlow>("f2").start(invoicesStep).build();

    Flow parallel = new FlowBuilder<SimpleFlow>("parallel")
            .split(executor).add(flow1, flow2).build();

    return new JobBuilder("parallelJob", repo)
            .start(parallel).end().build();
}
```

### ⚠️ Pitfalls / Gotchas
- Steps MUST be independent — no data dependency between parallel steps *(parallel steps mein koi dependency nahi honi chahiye)*
- If any parallel step fails, the entire split flow is marked FAILED
- Steps share the same JobExecution (but have separate StepExecutions)
- Resource contention: parallel steps competing for same DB connections
- Job continues only after ALL parallel steps complete

### 🎯 Tricky Interview Qs

**Q: What happens if one parallel step fails?**
All other parallel steps continue to completion. After all finish, the split flow is marked FAILED because one step failed. The failed step can be restarted.

**Q: Can you have dependencies within parallel steps?**
No. If Step B depends on Step A's output, they can't be parallel. Put dependent steps in the same sequential flow.

### ⚡ Remember
- `FlowBuilder.split(taskExecutor).add(flow1, flow2, flow3)` *(split = parallel mein chalao)*
- Steps must be independent (no data dependency)
- Job continues after ALL parallel steps complete
- Failed parallel step → entire flow FAILED
- Total time = max(parallel steps) instead of sum

### 🔗 Follow-ups
- [Q84 → All parallel options](#q84)
- [Q86 → Partitioning (for data parallelism)](#q86)
- [Q85 → Multi-threaded (for chunk parallelism)](#q85)
