<![CDATA[<div align="center">

# 🔴 Spring Batch — Parallel Processing Questions (84-91)

[![Questions](https://img.shields.io/badge/Questions-8-blue.svg)](#)
[![Difficulty](https://img.shields.io/badge/Level-Hard-red.svg)](#)

</div>

---

<a id="q1"></a>
## Q84. ❓ What is parallel processing in Spring Batch?

🔖 **Tags:** `#spring-batch` `#parallel` `#must-know`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

Spring Batch offers **4 approaches** to parallel processing:

```
┌──────────────────────────────────────────────────────┐
│              Parallel Processing Options               │
├──────────────┬────────────┬──────────────┬────────────┤
│ Multi-Thread │ Parallel   │ Partitioning │ Remote     │
│ Step         │ Steps      │              │ Chunking   │
├──────────────┼────────────┼──────────────┼────────────┤
│ Multiple     │ Run steps  │ Split data   │ Distribute │
│ threads in   │ in parallel│ into ranges, │ chunks to  │
│ single step  │ (independent│ each range  │ remote     │
│              │ steps)     │ processed by │ workers    │
│              │            │ own thread   │            │
├──────────────┼────────────┼──────────────┼────────────┤
│ Simple       │ Simple     │ Most common  │ Most       │
│              │            │ in production│ complex    │
└──────────────┴────────────┴──────────────┴────────────┘
```

---

<a id="q2"></a>
## Q85. ❓ What is a multi-threaded step?

🔖 **Tags:** `#spring-batch` `#multi-threaded` `#parallel`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

A **multi-threaded step** processes multiple chunks concurrently using a `TaskExecutor`.

```java
@Bean
public Step multiThreadedStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("multiThreadedStep", repo)
            .<Employee, Employee>chunk(100, tx)
            .reader(pagingReader())              // Must be thread-safe!
            .processor(processor())
            .writer(writer())
            .taskExecutor(taskExecutor())         // Enable multi-threading
            .throttleLimit(8)                     // Max 8 concurrent threads
            .build();
}

@Bean
public TaskExecutor taskExecutor() {
    SimpleAsyncTaskExecutor executor = new SimpleAsyncTaskExecutor();
    executor.setConcurrencyLimit(8);
    return executor;
}
```

```
Thread-1: Read chunk → Process → Write [items 1-100]
Thread-2: Read chunk → Process → Write [items 101-200]
Thread-3: Read chunk → Process → Write [items 201-300]
... all running concurrently!
```

### ⚠️ Important:
- Reader MUST be **thread-safe** (use `JdbcPagingItemReader`, NOT cursor)
- **Restartability is LOST** (order of processing is non-deterministic)
- Writer should be thread-safe (JDBC batch writer is safe)

---

<a id="q3"></a>
## Q86. ❓ What is partitioning?

🔖 **Tags:** `#spring-batch` `#partitioning` `#must-know` `#frequently-asked`  
📊 **Difficulty:** 🔴 Hard  
🔥 **Frequency:** ⭐⭐⭐⭐⭐

### ✅ Answer

**Partitioning** splits data into independent **partitions** and processes each partition with its own step execution (separate reader/writer per partition).

```
Master Step (Partitioner)
│
├── Partition 0: IDs    1 - 25000  → Slave Step (own reader, processor, writer)
├── Partition 1: IDs 25001 - 50000 → Slave Step (own reader, processor, writer)
├── Partition 2: IDs 50001 - 75000 → Slave Step (own reader, processor, writer)
└── Partition 3: IDs 75001 - 100000→ Slave Step (own reader, processor, writer)

Each partition runs in its own thread!
Each has its own ExecutionContext → RESTARTABLE!
```

```java
@Bean
public Step masterStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("masterStep", repo)
            .partitioner("slaveStep", rangePartitioner())  // Split data
            .step(slaveStep(repo, tx))                     // Worker step
            .gridSize(8)                                    // 8 partitions
            .taskExecutor(taskExecutor())                   // Parallel execution
            .build();
}

@Bean
public Partitioner rangePartitioner() {
    return gridSize -> {
        Map<String, ExecutionContext> partitions = new HashMap<>();
        long totalRecords = 100000;
        long range = totalRecords / gridSize;
        
        for (int i = 0; i < gridSize; i++) {
            ExecutionContext ctx = new ExecutionContext();
            ctx.putLong("minId", i * range + 1);
            ctx.putLong("maxId", (i + 1) * range);
            partitions.put("partition" + i, ctx);
        }
        return partitions;
    };
}

@Bean
@StepScope
public JdbcPagingItemReader<Employee> reader(
        @Value("#{stepExecutionContext['minId']}") Long minId,
        @Value("#{stepExecutionContext['maxId']}") Long maxId) {
    // Each partition reads its own range!
    return new JdbcPagingItemReaderBuilder<Employee>()
            .selectClause("SELECT *")
            .fromClause("FROM employees")
            .whereClause("WHERE id >= :minId AND id <= :maxId")
            .build();
}
```

### Partitioning vs Multi-Threading:

| Feature | Partitioning | Multi-Threading |
|---------|-------------|----------------|
| **Restartable** | ✅ Yes (each partition tracked) | ❌ No |
| **Reader thread-safety** | Not required (each partition has own) | Required |
| **Data isolation** | Full (separate ranges) | None (shared reader) |
| **Complexity** | More setup | Simple |
| **Recommended for** | Production systems | Simple parallel needs |

---

<a id="q4"></a>
## Q87. ❓ How does partitioning work?

🔖 **Tags:** `#spring-batch` `#partitioning` `#internals`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

```
1. Master Step starts
   │
2. Partitioner.partition(gridSize) called
   ├── Returns Map<String, ExecutionContext>
   │   partition0: {minId:1, maxId:25000}
   │   partition1: {minId:25001, maxId:50000}
   │   partition2: {minId:50001, maxId:75000}
   │   partition3: {minId:75001, maxId:100000}
   │
3. For each partition:
   ├── Create separate StepExecution
   ├── Inject ExecutionContext (minId, maxId)
   ├── Create reader scoped to this partition (@StepScope)
   └── Submit to TaskExecutor
   
4. All partitions run in parallel
   
5. Master waits for all partitions to complete
   
6. If ALL succeed → Master step COMPLETED
   If ANY fails → Master step FAILED (can restart failed partitions only)
```

---

<a id="q5"></a>
## Q88. ❓ What is remote partitioning?

🔖 **Tags:** `#spring-batch` `#remote-partitioning` `#distributed`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

**Remote partitioning** distributes partitions across **multiple JVMs/machines** using messaging middleware.

```
Machine A (Master):
  Partitioner → Creates partition metadata
  → Sends partition info via Messaging (Kafka/RabbitMQ/JMS)

Machine B (Worker 1): Receives partition0 → full read/process/write
Machine C (Worker 2): Receives partition1 → full read/process/write
Machine D (Worker 3): Receives partition2 → full read/process/write

Each worker does its own reading + processing + writing.
Data doesn't travel over the network — only partition metadata does!
```

### When to Use:
- Data is **too large for one machine**
- Need **horizontal scaling**
- Workers can access the same database/file system

---

<a id="q6"></a>
## Q89. ❓ What is remote chunking?

🔖 **Tags:** `#spring-batch` `#remote-chunking` `#distributed`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

**Remote chunking** distributes the **processing and writing** to remote workers while the master does the reading.

```
Master (reads data):
  Reader → reads items
  → Sends items via Messaging to workers

Worker 1: Process chunk + Write
Worker 2: Process chunk + Write  
Worker 3: Process chunk + Write

Difference from Remote Partitioning:
- Remote Partition: Workers read+process+write (only metadata sent)
- Remote Chunking: Master reads, Workers process+write (actual DATA sent)
```

### Remote Partitioning vs Remote Chunking:

| Feature | Remote Partitioning | Remote Chunking |
|---------|-------------------|-----------------|
| **Who reads** | Each worker | Master only |
| **Data over network** | Only metadata | Actual items |
| **Network load** | Low | High |
| **I/O bottleneck** | None | Reader on master |
| **Best for** | Large DB reads | CPU-heavy processing |

---

<a id="q7"></a>
## Q90. ❓ What is the difference between partitioning and multi-threading?

🔖 **Tags:** `#spring-batch` `#comparison` `#must-know`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

| Feature | Multi-Threading | Partitioning |
|---------|----------------|-------------|
| **Approach** | Multiple threads share ONE reader | Each partition has OWN reader |
| **Thread safety** | Reader must be thread-safe | Not required |
| **Restartability** | ❌ Lost | ✅ Each partition restartable |
| **Data isolation** | Shared | Completely isolated |
| **Configuration** | Simple (add TaskExecutor) | More complex (Partitioner + StepScope) |
| **Scalability** | Single JVM | Single JVM or distributed |
| **Production** | Simple jobs | ✅ Recommended for production |

### 📌 Key Takeaway
> 💡 **Partitioning > Multi-threading** for production. It's restartable, scalable, and each partition is isolated.

---

<a id="q8"></a>
## Q91. ❓ How do you configure parallel execution?

🔖 **Tags:** `#spring-batch` `#parallel` `#configuration`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

### 1️⃣ Parallel Steps (Independent steps running simultaneously):
```java
@Bean
public Job job(JobRepository repo, Step step1, Step step2, Step step3, Step step4) {
    Flow flow1 = new FlowBuilder<SimpleFlow>("flow1").start(step1).build();
    Flow flow2 = new FlowBuilder<SimpleFlow>("flow2").start(step2).build();
    
    return new JobBuilder("parallelJob", repo)
            .start(new FlowBuilder<SimpleFlow>("splitFlow")
                .split(new SimpleAsyncTaskExecutor())   // Run in parallel
                .add(flow1, flow2)                      // step1 & step2 parallel
                .build())
            .next(step3)                                // step3 after both complete
            .end()
            .build();
}
```

### 2️⃣ Multi-Threaded Step:
```java
.taskExecutor(new SimpleAsyncTaskExecutor())
.throttleLimit(8)
```

### 3️⃣ Partitioned Step:
```java
.partitioner("slave", partitioner())
.step(slaveStep())
.gridSize(8)
.taskExecutor(taskExecutor())
```

---

[← Back to Spring Batch Index](./README.md) | [← Prev: Tasklet](./09-tasklet.md) | [Next: Performance →](./11-performance.md)
]]>