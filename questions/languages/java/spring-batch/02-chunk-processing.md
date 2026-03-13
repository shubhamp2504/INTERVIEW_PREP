
# 🟡 Spring Batch — Chunk Processing Questions (21-30)

[![Questions](https://img.shields.io/badge/Questions-10-blue.svg)](#)
[![Difficulty](https://img.shields.io/badge/Level-Medium-yellow.svg)](#)


---

<a id="q1"></a>

## Q21. ❓ What is chunk processing in Spring Batch?

🔖 **Tags:** `#spring-batch` `#chunk` `#must-know` `#frequently-asked`  
📊 **Difficulty:** 🟡 Medium  
🔥 **Frequency:** ⭐⭐⭐⭐⭐ (Top Interview Question)

### ✅ Answer

**Chunk processing** is Spring Batch's core processing model where data is read, processed, and written in **fixed-size groups (chunks)** within a single transaction.

```
┌─────────────────── One Chunk (size = 3) ──────────────────┐
│                                                            │
│  READ item1 ─→ PROCESS item1 ─┐                          │
│  READ item2 ─→ PROCESS item2 ─┼─→ WRITE [item1,item2,item3] → COMMIT
│  READ item3 ─→ PROCESS item3 ─┘                          │
│                                                            │
├─────────────────── Next Chunk ────────────────────────────┤
│                                                            │
│  READ item4 ─→ PROCESS item4 ─┐                          │
│  READ item5 ─→ PROCESS item5 ─┼─→ WRITE [item4,item5,item6] → COMMIT
│  READ item6 ─→ PROCESS item6 ─┘                          │
│                                                            │
└────────────── Repeats until reader returns null ──────────┘
```

### 🎯 Key Points:
- Items are **read one at a time**, **processed one at a time**, but **written as a batch**
- Each chunk = **1 database transaction**
- If chunk fails → only that chunk rolls back, previous chunks are safe
- Chunk size is configurable

```java
@Bean
public Step processStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("processStep", repo)
            .<Input, Output>chunk(100, tx)  // chunk size = 100
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .build();
}
```

### 📌 Key Takeaway
> 💡 Chunk = Read N items + Process N items + Write N items + Commit. This is the heart of Spring Batch!

---

<a id="q2"></a>

## Q22. ❓ How does chunk processing work internally?

🔖 **Tags:** `#spring-batch` `#chunk` `#internals` `#must-know`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

```
Step starts
│
├── Open transaction
│   │
│   ├── REPEAT (chunk-size times):
│   │   ├── reader.read()  → returns 1 item (or null = done)
│   │   └── processor.process(item) → returns transformed item (or null = filtered)
│   │
│   ├── Collect all processed items into a Chunk<T>
│   │
│   ├── writer.write(chunk)  → writes all items at once
│   │
│   └── COMMIT transaction  ✅
│
├── If exception during chunk:
│   └── ROLLBACK transaction  ❌
│
├── Repeat for next chunk...
│
└── reader.read() returns null → Step complete
```

### Internal Pseudocode:

```java
// Simplified internal logic
List<O> processedItems = new ArrayList<>();

while (true) {
    // Begin Transaction
    transaction.begin();
    
    try {
        processedItems.clear();
        
        for (int i = 0; i < chunkSize; i++) {
            I item = reader.read();  // Read one item
            if (item == null) break;  // No more data
            
            O processed = processor.process(item);  // Process
            if (processed != null) {
                processedItems.add(processed);  // null = filtered
            }
        }
        
        if (!processedItems.isEmpty()) {
            writer.write(new Chunk<>(processedItems));  // Write batch
        }
        
        transaction.commit();  // ✅ Commit chunk
        
    } catch (Exception e) {
        transaction.rollback();  // ❌ Rollback chunk
        // Handle skip/retry logic
    }
    
    if (readerExhausted) break;
}
```

---

<a id="q3"></a>

## Q23. ❓ What is the difference between chunk size and commit interval?

🔖 **Tags:** `#spring-batch` `#chunk-size` `#commit-interval`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

**They are the same thing!**

| Term | Meaning |
|------|---------|
| **Chunk Size** | Number of items processed before writing + committing |
| **Commit Interval** | Number of items processed before a transaction commit |

In Spring Batch, chunk size **IS** the commit interval. When you set `chunk(100)`, it means:
- Read 100 items
- Process 100 items  
- Write 100 items
- **Commit** the transaction

```java
// chunk(100) means: commit after every 100 items
.<Input, Output>chunk(100, transactionManager)
```

> 💡 In older XML-based config, `commit-interval` was the attribute name. In Java config, it's just the chunk size parameter.

---

<a id="q4"></a>

## Q24. ❓ What happens if a failure occurs in the middle of chunk processing?

🔖 **Tags:** `#spring-batch` `#error-handling` `#chunk` `#must-know`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

```
Processing 500 records with chunk size 100:

Chunk 1 (items 1-100)   → COMMITTED ✅ (safe in DB)
Chunk 2 (items 101-200) → COMMITTED ✅ (safe in DB)
Chunk 3 (items 201-300) → FAILED at item 250 ❌
                          → ROLLBACK entire chunk 3
                          → Items 201-300 are NOT written

Chunk 4 & 5 → NEVER executed

Job Status: FAILED
StepExecution: readCount=250, writeCount=200, commitCount=2, rollbackCount=1
```

### What Happens Next?

| Scenario | Result |
|----------|--------|
| **No skip/retry** | Step fails, job fails |
| **Skip configured** | Item 250 is skipped, chunk continues |
| **Retry configured** | Item 250 is retried N times |
| **Job restart** | Resumes from item 201 (chunk 3) |

### 📌 Key Takeaway
> 💡 Previously committed chunks are **SAFE**. Only the current chunk rolls back. This is why chunk processing is reliable!

---

<a id="q5"></a>

## Q25. ❓ How does Spring Batch manage transactions in chunk processing?

🔖 **Tags:** `#spring-batch` `#transactions` `#chunk`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

```
┌──── Chunk 1 ────────────────────────┐
│ BEGIN TRANSACTION                    │
│   Read item1, Process item1         │
│   Read item2, Process item2         │
│   Read item3, Process item3         │
│   Write [item1, item2, item3]       │
│ COMMIT TRANSACTION ✅                │
└─────────────────────────────────────┘

┌──── Chunk 2 ────────────────────────┐
│ BEGIN TRANSACTION                    │
│   Read item4, Process item4         │
│   Read item5 → EXCEPTION ❌         │
│ ROLLBACK TRANSACTION ❌              │  ← Only chunk 2 rolls back
└─────────────────────────────────────┘

Chunk 1 data is safe! ✅
```

### 🎯 Key Points:
- **1 chunk = 1 transaction**
- Transaction is opened **before** reading and committed **after** writing
- If exception occurs → entire chunk rolls back
- Reader reads are **inside** the transaction
- Spring uses `PlatformTransactionManager`

```java
// Custom transaction configuration
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("step", repo)
            .<I, O>chunk(100, tx)
            .reader(reader())
            .writer(writer())
            .transactionAttribute(transactionAttribute())  // custom isolation/propagation
            .build();
}
```

---

<a id="q6"></a>

## Q26. ❓ What is the optimal chunk size?

🔖 **Tags:** `#spring-batch` `#chunk-size` `#performance` `#frequently-asked`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

**There is no universal optimal chunk size.** It depends on your use case.

### General Guidelines:

| Chunk Size | Trade-offs |
|-----------|------------|
| **Too Small (1-10)** | Too many commits → slow, high overhead |
| **Small (10-50)** | Good for complex processing, less memory |
| **Medium (100-500)** | ✅ **Best for most cases** |
| **Large (1000-5000)** | Better throughput but higher memory, longer rollback |
| **Too Large (10000+)** | Memory issues, very long rollback on failure |

### Factors to Consider:

| Factor | Smaller Chunk | Larger Chunk |
|--------|--------------|--------------|
| Memory usage | Lower ✅ | Higher ❌ |
| Throughput | Lower ❌ | Higher ✅ |
| Commit frequency | More ❌ | Less ✅ |
| Rollback impact | Less data lost ✅ | More data lost ❌ |
| Processing complexity | Better for heavy processing | Better for simple processing |

### 📌 Recommendation:
> 💡 Start with **100-500**, benchmark, then tune. If processing is heavy → go smaller. If it's simple reads/writes → go larger.

---

<a id="q7"></a>

## Q27. ❓ How do you handle memory issues when processing large chunks?

🔖 **Tags:** `#spring-batch` `#memory` `#performance`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

| Strategy | How |
|----------|-----|
| **Reduce chunk size** | Smaller chunks = less items in memory at once |
| **Use Paging readers** | `JdbcPagingItemReader` loads page-by-page, not all at once |
| **Avoid cursor readers for huge data** | Cursors hold DB connection open + buffering |
| **Clear processor caches** | Don't accumulate data in processor between chunks |
| **Use streaming** | Process items without loading all into memory |
| **Increase JVM heap** | `-Xmx4g` as last resort |
| **Use partitioning** | Split data into partitions, process independently |

```java
// Good: Paging reader (fetches page-by-page)
@Bean
public JdbcPagingItemReader<Employee> reader(DataSource ds) {
    return new JdbcPagingItemReaderBuilder<Employee>()
            .dataSource(ds)
            .name("employeeReader")
            .selectClause("SELECT *")
            .fromClause("FROM employees")
            .sortKeys(Map.of("id", Order.ASCENDING))
            .pageSize(500)       // fetches 500 at a time
            .rowMapper(new BeanPropertyRowMapper<>(Employee.class))
            .build();
}
```

---

<a id="q8"></a>

## Q28. ❓ What happens if ItemProcessor returns null?

🔖 **Tags:** `#spring-batch` `#processor` `#filtering`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

When `ItemProcessor.process()` returns **null**, that item is **filtered out** — it will NOT be passed to the writer.

```java
@Bean
public ItemProcessor<Order, Order> processor() {
    return order -> {
        if (order.getAmount() <= 0) {
            return null;  // ❌ Filtered — won't be written
        }
        return order;     // ✅ Passed to writer
    };
}
```

```
Chunk (size=5):
  Read order1 (amt=100) → Process → order1 ✅
  Read order2 (amt=0)   → Process → null   ❌ FILTERED
  Read order3 (amt=50)  → Process → order3 ✅
  Read order4 (amt=-10) → Process → null   ❌ FILTERED
  Read order5 (amt=200) → Process → order5 ✅

Writer receives: [order1, order3, order5]  (only 3 items)
StepExecution: readCount=5, filterCount=2, writeCount=3
```

### 📌 Key Takeaway
> 💡 `return null` in processor = **filter/skip** that item. It's tracked in `StepExecution.filterCount`.

---

<a id="q9"></a>

## Q29. ❓ What happens if ItemWriter fails?

🔖 **Tags:** `#spring-batch` `#writer` `#error-handling`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

When `ItemWriter.write()` throws an exception:

```
1. Current chunk's transaction → ROLLBACK ❌
2. ALL items in current chunk are NOT written
3. Previously committed chunks → SAFE ✅
4. Step enters error handling mode:

   If skip configured → Re-process chunk one-by-one to find bad item
   If retry configured → Retry the entire chunk
   If neither → Step FAILS, Job FAILS
```

### Skip Mode (when writer fails):

```
Chunk [item1, item2, item3] → Writer FAILS on item2

Spring Batch now re-processes one at a time:
  Write [item1] → SUCCESS ✅
  Write [item2] → FAILS → SKIP ❌ (logged)
  Write [item3] → SUCCESS ✅

This is called "scan mode" — finds the exact bad item
```

---

<a id="q10"></a>

## Q30. ❓ Can we skip records in chunk processing?

🔖 **Tags:** `#spring-batch` `#skip` `#chunk`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

**Yes!** Spring Batch has built-in skip logic for handling bad records.

```java
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("step", repo)
            .<Input, Output>chunk(100, tx)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .faultTolerant()                              // Enable fault tolerance
            .skip(ValidationException.class)              // Skip this exception
            .skip(FlatFileParseException.class)           // Skip this too
            .skipLimit(50)                                // Max 50 skips allowed
            .noSkip(DatabaseException.class)              // NEVER skip this
            .listener(skipListener())                     // Log skipped items
            .build();
}
```

### Skip Behavior by Phase:

| Phase | What happens on skip |
|-------|---------------------|
| **Reader** | Bad item is skipped, next item read |
| **Processor** | Item is excluded, not passed to writer |
| **Writer** | Chunk re-processed one-by-one to isolate bad item |

### 📌 Key Takeaway
> 💡 Skip = "I expect some bad data, skip it and continue." Always set a `skipLimit` and log skipped items!

---

[← Back to Spring Batch Index](./README.md) | [← Prev: Basics](./01-basics.md) | [Next: Readers →](./03-readers.md)
]]>
