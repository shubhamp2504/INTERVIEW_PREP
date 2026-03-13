
# 🔴 Spring Batch — Performance Optimization Questions (92-98)

[![Questions](https://img.shields.io/badge/Questions-7-blue.svg)](#)
[![Difficulty](https://img.shields.io/badge/Level-Hard-red.svg)](#)


---

<a id="q1"></a>

## Q92. ❓ How do you process millions of records efficiently?

🔖 **Tags:** `#spring-batch` `#performance` `#must-know` `#frequently-asked`  
📊 **Difficulty:** 🔴 Hard  
🔥 **Frequency:** ⭐⭐⭐⭐⭐

### ✅ Answer

| Strategy | Impact | How |
|----------|--------|-----|
| **Partitioning** | ⭐⭐⭐⭐⭐ | Split data into ranges, process in parallel |
| **Optimal chunk size** | ⭐⭐⭐⭐ | Start with 500, benchmark, tune |
| **JDBC batch insert** | ⭐⭐⭐⭐ | `JdbcBatchItemWriter` with batch size = chunk size |
| **Paging reader** | ⭐⭐⭐⭐ | `JdbcPagingItemReader` for memory efficiency |
| **Disable JPA features** | ⭐⭐⭐ | No lazy loading, no dirty checking |
| **Index optimization** | ⭐⭐⭐ | Ensure DB indexes on WHERE/ORDER columns |
| **Connection pooling** | ⭐⭐⭐ | HikariCP with adequate pool size |
| **Reduce logging** | ⭐⭐ | Set batch logging to WARN in production |

### Production Recipe for 10M+ Records:
```java
@Bean
public Step masterStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("masterStep", repo)
            .partitioner("worker", rangePartitioner())
            .step(workerStep(repo, tx))
            .gridSize(16)                        // 16 parallel partitions
            .taskExecutor(taskExecutor())
            .build();
}

@Bean
public Step workerStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("workerStep", repo)
            .<Employee, Employee>chunk(500, tx)   // 500 items per chunk
            .reader(pagingReader(null, null))      // Paging, not cursor
            .writer(batchWriter())                 // JDBC batch insert
            .build();
}

@Bean
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
    exec.setCorePoolSize(16);
    exec.setMaxPoolSize(16);
    exec.setQueueCapacity(0);
    return exec;
}
```

---

<a id="q2"></a>

## Q93. ❓ How do you improve Spring Batch performance?

🔖 **Tags:** `#spring-batch` `#performance` `#optimization`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

### Performance Checklist:

```
Reader Optimization:
  ✅ Use JdbcPagingItemReader (not cursor for large data)
  ✅ Set appropriate pageSize (match chunk size)
  ✅ Add DB indexes on sort/filter columns
  ✅ Use projections (SELECT specific columns, not *)

Processor Optimization:
  ✅ Keep processing lightweight
  ✅ Batch external API calls (don't call per-item)
  ✅ Cache lookup data (load once, use for all items)
  ✅ Remove processor if no transformation needed

Writer Optimization:
  ✅ Use JdbcBatchItemWriter (JDBC batch API)
  ✅ Set hibernate.jdbc.batch_size = chunk size
  ✅ Disable hibernate auto-flush during batch
  ✅ Use COPY command for PostgreSQL bulk load

Infrastructure:
  ✅ Tune chunk size (benchmark 100, 500, 1000)
  ✅ Use connection pooling (HikariCP)
  ✅ Allocate enough JVM memory (-Xmx)
  ✅ Use partitioning for parallelism
  ✅ Reduce logging level in production
```

---

<a id="q3"></a>

## Q94. ❓ How do you reduce database calls in Spring Batch?

🔖 **Tags:** `#spring-batch` `#performance` `#database`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

| Technique | Savings |
|-----------|---------|
| **Batch inserts** | 100 inserts → 1 batch call |
| **Larger chunks** | Fewer commits = fewer DB round trips |
| **Cache reference data** | Load lookup tables once, not per-item |
| **Paging with correct page size** | Fewer SELECT queries |
| **Minimize metadata updates** | Use in-memory JobRepository for non-critical jobs |
| **Disable auto-flush** | Prevent unnecessary Hibernate flushes |

```java
// Cache lookup data in processor
@Component
@StepScope
public class EnrichmentProcessor implements ItemProcessor<Order, Order> {
    
    private Map<Long, Customer> customerCache;
    
    @BeforeStep
    public void loadCache(StepExecution stepExecution) {
        // Load ALL customers once, not per-order
        customerCache = customerRepo.findAll().stream()
            .collect(Collectors.toMap(Customer::getId, Function.identity()));
    }
    
    @Override
    public Order process(Order order) {
        Customer customer = customerCache.get(order.getCustomerId());
        order.setCustomerName(customer.getName());
        return order;
    }
}
```

---

<a id="q4"></a>

## Q95. ❓ What is the best reader for large datasets?

🔖 **Tags:** `#spring-batch` `#reader` `#large-data`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

**`JdbcPagingItemReader`** — for database sources, combined with **partitioning** for very large data.

| Data Size | Reader | Approach |
|-----------|--------|----------|
| < 1M | `JdbcPagingItemReader` | Single step |
| 1M - 10M | `JdbcPagingItemReader` | + multi-threaded step |
| 10M+ | `JdbcPagingItemReader` | + partitioning (8-16 partitions) |
| 100M+ | `JdbcPagingItemReader` | + partitioning (distributed, 50+ workers) |
| Files | `FlatFileItemReader` | Memory efficient (streaming) |

---

<a id="q5"></a>

## Q96. ❓ How do you optimize batch inserts?

🔖 **Tags:** `#spring-batch` `#batch-insert` `#optimization`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 500        # Match your chunk size
        order_inserts: true      # Group inserts by entity
        order_updates: true      # Group updates by entity
  datasource:
    hikari:
      maximum-pool-size: 20
      # Add rewriteBatchedStatements for MySQL:
      jdbc-url: jdbc:mysql://host/db?rewriteBatchedStatements=true
```

```java
// Use JdbcBatchItemWriter (fastest)
@Bean
public JdbcBatchItemWriter<Employee> writer(DataSource ds) {
    return new JdbcBatchItemWriterBuilder<Employee>()
            .sql("INSERT INTO employees(name,salary,dept) VALUES(:name,:salary,:dept)")
            .dataSource(ds)
            .beanMapped()
            .assertUpdates(false)   // Skip row count verification for speed
            .build();
}
```

---

<a id="q6"></a>

## Q97. ❓ How do you tune chunk size?

🔖 **Tags:** `#spring-batch` `#chunk-size` `#tuning`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

### Benchmarking Approach:
```
Test with different chunk sizes and measure:

Chunk Size | Time    | Memory  | Commits
100        | 120 sec | 256 MB  | 1000
500        | 80 sec  | 512 MB  | 200     ← Sweet spot
1000       | 75 sec  | 1 GB    | 100
5000       | 73 sec  | 3 GB    | 20
10000      | OOM     | 💀      | —        ← Too large!
```

### Rules of Thumb:
| Scenario | Recommended Chunk Size |
|----------|----------------------|
| Simple read/write | 500 - 1000 |
| Complex processing | 100 - 300 |
| External API calls in processor | 50 - 100 |
| Very large objects | 50 - 100 |
| Simple INSERTs | 1000 - 5000 |

---

<a id="q7"></a>

## Q98. ❓ How do you avoid memory issues?

🔖 **Tags:** `#spring-batch` `#memory` `#troubleshooting`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

| Issue | Solution |
|-------|---------|
| Large chunk size | Reduce chunk size (500 → 100) |
| Cursor reader buffering | Switch to paging reader |
| JPA caching all entities | Clear EntityManager per chunk |
| Large objects in memory | Use streaming, process smaller batches |
| Accumulating state | Clear caches between chunks |
| Too many concurrent threads | Reduce thread pool size |

```java
// Clear JPA cache per chunk to avoid memory leak
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx, 
                 EntityManagerFactory emf) {
    return new StepBuilder("step", repo)
            .<I, O>chunk(100, tx)
            .reader(reader())
            .writer(writer())
            .listener(new ChunkListener() {
                @Override
                public void afterChunk(ChunkContext context) {
                    emf.createEntityManager().clear();  // Free memory
                }
            })
            .build();
}
```

---

[← Back to Spring Batch Index](./README.md) | [← Prev: Parallel](./10-parallel-processing.md) | [Next: Scheduling →](./12-scheduling.md)
]]>
