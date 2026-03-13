# 🟡 Spring Batch — ItemWriter Deep Dive (Q41–Q48)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q41"></a>

## Q41. What types of ItemWriter implementations are available?

### 🔑 Quick Answer

> Spring provides built-in writers for **databases** (JdbcBatch, JPA, Hibernate), **files** (CSV, JSON, XML), **messaging** (Kafka, JMS, AMQP), **NoSQL** (MongoDB), and **composite** writers for writing to multiple destinations.

### 📖 Step-by-Step Explanation

| Category | Writer | Write To | Performance |
|----------|--------|----------|-------------|
| **Database** | `JdbcBatchItemWriter` | Any RDBMS via JDBC | ⭐⭐⭐⭐⭐ Fastest |
| **Database** | `JpaItemWriter` | Any RDBMS via JPA | ⭐⭐⭐ ORM overhead |
| **Database** | `HibernateItemWriter` | Via Hibernate | ⭐⭐⭐ |
| **File** | `FlatFileItemWriter` | CSV, TXT files | ⭐⭐⭐⭐ |
| **File** | `JsonFileItemWriter` | JSON files | ⭐⭐⭐⭐ |
| **File** | `StaxEventItemWriter` | XML files | ⭐⭐⭐ |
| **Message** | `KafkaItemWriter` | Kafka topics | ⭐⭐⭐⭐ |
| **Message** | `JmsItemWriter` | JMS queues | ⭐⭐⭐ |
| **NoSQL** | `MongoItemWriter` | MongoDB | ⭐⭐⭐⭐ |
| **Multi** | `CompositeItemWriter` | Multiple destinations | Depends on delegates |
| **Multi** | `ClassifierCompositeItemWriter` | Conditional routing | Depends on delegates |

**How to choose:**

```
Writing to RDBMS?
  ├── Need maximum speed → JdbcBatchItemWriter ⭐
  └── Already using JPA entities → JpaItemWriter (slower)

Writing to files?
  ├── CSV/TXT → FlatFileItemWriter
  ├── JSON → JsonFileItemWriter
  └── XML → StaxEventItemWriter

Writing to multiple places?
  ├── Same data to DB + file → CompositeItemWriter
  └── Route items conditionally → ClassifierCompositeItemWriter
```

### 🗣️ How to Explain in Interview

> *"Spring provides writers for every common destination. For databases, JdbcBatchItemWriter is the fastest — it uses JDBC batch operations. JpaItemWriter is convenient if you're already using JPA but has ORM overhead. For files, there's FlatFileItemWriter for CSV and JsonFileItemWriter for JSON. What I find really useful is CompositeItemWriter — when you need to write the same data to both a database and a file, or ClassifierCompositeItemWriter when different items should go to different destinations based on a condition."*

### ⚡ Key Points to Remember

1. **JdbcBatchItemWriter** = fastest for databases
2. **FlatFileItemWriter** = for CSV/TXT output
3. **CompositeItemWriter** = write to multiple destinations
4. **ClassifierCompositeItemWriter** = route items conditionally
5. All writers receive the **entire chunk** at once (not one by one)

---

<a id="q42"></a>

## Q42. What is JdbcBatchItemWriter? How does it work?

### 🔑 Quick Answer

> It writes data to a database using **JDBC batch operations** — all items in a chunk are added to a batch and sent to the database in **one network round trip**, making it the fastest writer for RDBMS.

### 📖 Step-by-Step Explanation

**Step 1 — How batch insert differs from single insert:**

```
❌ Single insert (500 items = 500 DB calls):
  INSERT INTO orders VALUES (1, 'laptop', 999)     → network call 1
  INSERT INTO orders VALUES (2, 'phone', 599)      → network call 2
  INSERT INTO orders VALUES (3, 'tablet', 399)     → network call 3
  ...
  INSERT INTO orders VALUES (500, 'mouse', 29)     → network call 500
  Time: ~5 seconds (500 network round trips)

✅ Batch insert (500 items = 1 DB call):
  PreparedStatement.addBatch(1, 'laptop', 999)     → buffer locally
  PreparedStatement.addBatch(2, 'phone', 599)      → buffer locally
  PreparedStatement.addBatch(3, 'tablet', 399)     → buffer locally
  ...
  PreparedStatement.addBatch(500, 'mouse', 29)     → buffer locally
  PreparedStatement.executeBatch()                  → ONE network call!
  Time: ~0.1 seconds (1 network round trip)
```

**Step 2 — Internal flow:**

```
JdbcBatchItemWriter.write(chunk) is called:
│
├── 1. Get connection from pool
├── 2. Create PreparedStatement with SQL
├── 3. FOR EACH item in chunk:
│       Set parameter values on PreparedStatement
│       preparedStatement.addBatch()
├── 4. preparedStatement.executeBatch()  ← ONE database call
├── 5. Verify row counts (optional)
└── Transaction commits (managed by Spring)
```

**Step 3 — Two ways to map parameters:**

```java
// Way 1: Bean Property Mapping (named parameters)
.sql("INSERT INTO orders (id, product, amount) VALUES (:orderId, :product, :amount)")
.beanMapped()    // Maps :orderId to order.getOrderId(), etc.

// Way 2: Positional Parameters
.sql("INSERT INTO orders (id, product, amount) VALUES (?, ?, ?)")
.itemPreparedStatementSetter((order, ps) -> {
    ps.setLong(1, order.getOrderId());
    ps.setString(2, order.getProduct());
    ps.setBigDecimal(3, order.getAmount());
})
```

### 🗣️ How to Explain in Interview

> *"JdbcBatchItemWriter is the fastest database writer in Spring Batch. When the chunk is ready — say 500 items — it doesn't execute 500 separate INSERT statements. Instead, it uses JDBC batch operations: it calls addBatch() for each item to buffer them locally, then executeBatch() to send all 500 in one network round trip. That's the key performance gain — one network call instead of 500. You can map parameters two ways: beanMapped() for automatic named parameter mapping, or a custom ItemPreparedStatementSetter for manual mapping."*

### 💻 Code Example

```java
@Bean
public JdbcBatchItemWriter<Order> orderWriter(DataSource dataSource) {
    return new JdbcBatchItemWriterBuilder<Order>()
            .sql("INSERT INTO orders (order_id, product, amount, tax, status) " +
                 "VALUES (:orderId, :product, :amount, :tax, :status)")
            .dataSource(dataSource)
            .beanMapped()           // Auto-map Order properties to :namedParams
            .assertUpdates(true)    // Verify all rows were inserted (default: true)
            .build();
}
```

**What happens when writer.write(chunk) is called:**
1. Opens PreparedStatement with the INSERT SQL
2. Loops through 500 Order objects
3. For each: maps `orderId`, `product`, `amount`, `tax`, `status` → calls `addBatch()`
4. Calls `executeBatch()` → sends ALL 500 to database
5. If `assertUpdates=true`, verifies 500 rows affected

### ⚡ Key Points to Remember

1. Uses **JDBC batch** — `addBatch()` + `executeBatch()` = 1 network call
2. **Fastest writer** for RDBMS (50x faster than single inserts)
3. `beanMapped()` = automatic, `itemPreparedStatementSetter` = manual
4. `assertUpdates(true)` = verify all rows inserted (default)
5. Set `assertUpdates(false)` for UPSERT operations (affected count varies)

---

<a id="q43"></a>

## Q43. What is JpaItemWriter?

### 🔑 Quick Answer

> JpaItemWriter uses JPA's `EntityManager.merge()` to persist entities. It's **convenient** when working with JPA entities but **slower** than JdbcBatchItemWriter due to ORM overhead.

### 📖 Step-by-Step Explanation

**Step 1 — How it works:**

```
JpaItemWriter.write(chunk) is called:
│
├── FOR EACH entity in chunk:
│   └── entityManager.merge(entity)   ← JPA handles insert/update
│
└── EntityManager.flush()              ← Push to database
    EntityManager.clear()              ← Clear first-level cache
```

**Step 2 — JPA vs JDBC writer performance:**

| Aspect | JdbcBatchItemWriter | JpaItemWriter |
|--------|-------------------|---------------|
| **Speed** | ⭐⭐⭐⭐⭐ (raw JDBC batch) | ⭐⭐⭐ (ORM overhead) |
| **Mapping** | Manual (SQL + params) | Automatic (entity annotations) |
| **Overhead** | None | Dirty checking, proxy, cache, flush |
| **Best for** | High-volume inserts | Entity-centric applications |
| **Relationships** | DIY (manual SQL) | Automatic (cascading) |

**Step 3 — When to use JPA writer:**

```
✅ USE JpaItemWriter when:
  - Project already uses JPA entities everywhere
  - You need cascade operations (save parent + children)
  - Data volume < 1M records
  - Developer productivity > raw performance

❌ USE JdbcBatchItemWriter when:
  - Processing millions of records
  - Maximum performance needed
  - Simple flat inserts (no relationships)
  - Complex SQL (UPSERT, multi-table)
```

### 🗣️ How to Explain in Interview

> *"JpaItemWriter persists entities using EntityManager.merge(). The advantage is you work with JPA entities and annotations — no manual SQL mapping. It handles cascading relationships automatically. The disadvantage is performance — JPA has overhead from dirty checking, proxy objects, and first-level cache management. For batch processing with millions of records, JdbcBatchItemWriter is significantly faster because it uses raw JDBC batch operations. I typically use JPA writer in smaller batches where convenience matters more than raw speed."*

### 💻 Code Example

```java
@Bean
public JpaItemWriter<Employee> jpaWriter(EntityManagerFactory emf) {
    JpaItemWriter<Employee> writer = new JpaItemWriter<>();
    writer.setEntityManagerFactory(emf);
    return writer;
}
```

### ⚡ Key Points to Remember

1. Uses `EntityManager.merge()` per entity
2. **Slower** than JdbcBatch (ORM overhead)
3. **Convenient** — works with entity annotations
4. Auto-handles **cascading** relationships
5. For high volume → **switch to JdbcBatchItemWriter**

---

<a id="q44"></a>

## Q44. What is FlatFileItemWriter?

### 🔑 Quick Answer

> FlatFileItemWriter writes data to **text files** (CSV, TSV, fixed-width). It supports headers, footers, custom delimiters, append mode, and is the counterpart of FlatFileItemReader.

### 📖 Step-by-Step Explanation

**Step 1 — What it produces:**

```
Output: report.csv

id,name,salary,department          ← Header (configured via callback)
1,Amit,50000,IT                    ← Data rows
2,Priya,60000,HR
3,Rahul,55000,IT
---                                ← Footer (optional)
Total Records: 3
Generated: 2024-01-15
```

**Step 2 — Key configuration options:**

| Option | What it does |
|--------|-------------|
| `delimited()` | CSV output (configurable delimiter) |
| `formatted()` | Fixed-width output (format string) |
| `headerCallback` | Write header line(s) before data |
| `footerCallback` | Write footer line(s) after data |
| `append(true)` | Add to existing file (don't overwrite) |
| `shouldDeleteIfEmpty(true)` | Delete file if no data written |

### 🗣️ How to Explain in Interview

> *"FlatFileItemWriter is the counterpart of FlatFileItemReader — it writes Java objects to text files. You configure it with the output file path, column names that map to bean properties, and a delimiter. It also supports header and footer callbacks — so I can write column headers at the top and a summary at the bottom. By default, it overwrites the file, but you can set append mode. It writes the entire chunk at once — so if chunk size is 500, it buffers 500 lines and writes them together."*

### 💻 Code Example

```java
@Bean
public FlatFileItemWriter<Employee> csvWriter() {
    return new FlatFileItemWriterBuilder<Employee>()
            .name("employeeCsvWriter")
            .resource(new FileSystemResource("output/employees.csv"))
            .delimited()
            .delimiter(",")
            .names("id", "name", "salary", "department")  // Bean properties
            .headerCallback(writer -> writer.write("id,name,salary,department"))
            .footerCallback(writer -> writer.write("--- End of Report ---"))
            .build();
}
```

### ⚡ Key Points to Remember

1. **Counterpart** of FlatFileItemReader (reader → writer)
2. Supports **header/footer** callbacks
3. `append(true)` for adding to existing file
4. Writes **entire chunk at once** (buffered)
5. Choose between `delimited()` (CSV) or `formatted()` (fixed-width)

---

<a id="q45"></a>

## Q45. What is CompositeItemWriter? What about ClassifierCompositeItemWriter?

### 🔑 Quick Answer

> **CompositeItemWriter** sends the same data to **multiple writers** (e.g., write to DB AND file). **ClassifierCompositeItemWriter** routes each item to a **different writer based on a condition** (e.g., valid items → DB, invalid → error file).

### 📖 Step-by-Step Explanation

**Step 1 — CompositeItemWriter: same data → multiple destinations:**

```
Every order goes to BOTH:

                    ┌── Writer 1: JdbcBatchItemWriter (to database)
Order ──────────────┤
                    └── Writer 2: FlatFileItemWriter (to CSV file)

Both receive the SAME chunk of items.
```

**Step 2 — ClassifierCompositeItemWriter: conditional routing:**

```
Each order goes to ONE writer based on condition:

Order{type=DOMESTIC}  ──→ Writer 1: domesticOrderWriter (domestic_orders table)
Order{type=EXPORT}    ──→ Writer 2: exportOrderWriter (export_orders table)
Order{type=INVALID}   ──→ Writer 3: errorFileWriter (error.csv)
```

### 🗣️ How to Explain in Interview

> *"CompositeItemWriter is used when you need to write the same data to multiple destinations — like inserting into a database and also writing to a CSV file for audit. All delegates receive the same chunk. ClassifierCompositeItemWriter is different — it routes each item to a specific writer based on a condition. For example, I used it in a project where domestic orders went to one table, export orders to another, and invalid orders to an error file. The classifier is a lambda or class that inspects each item and returns the appropriate writer."*

### 💻 Code Example

```java
// COMPOSITE: Same data to DB + File
@Bean
public CompositeItemWriter<Order> compositeWriter() {
    CompositeItemWriter<Order> writer = new CompositeItemWriter<>();
    writer.setDelegates(List.of(
            dbWriter(),      // Write to database
            csvWriter()      // Also write to CSV file
    ));
    return writer;
}

// CLASSIFIER: Route items conditionally
@Bean
public ClassifierCompositeItemWriter<Order> classifierWriter() {
    ClassifierCompositeItemWriter<Order> writer = new ClassifierCompositeItemWriter<>();
    writer.setClassifier(order -> {
        switch (order.getType()) {
            case "DOMESTIC": return domesticWriter();
            case "EXPORT":   return exportWriter();
            default:         return errorFileWriter();
        }
    });
    return writer;
}
```

### ⚡ Key Points to Remember

1. **Composite** = ALL writers get ALL items (parallel destinations)
2. **Classifier** = each item → ONE writer (conditional routing)
3. Composite writers execute **in order** (first delegate, then second)
4. Both are in the **same transaction** as the chunk

---

<a id="q46"></a>

## Q46. How does batch insert work in JdbcBatchItemWriter?

### 🔑 Quick Answer

> It uses JDBC's `PreparedStatement.addBatch()` to buffer all items locally, then `executeBatch()` to send ALL of them to the database in **one network call**. This is 10-50x faster than individual inserts.

### 📖 Step-by-Step Explanation

**Step 1 — The JDBC batch mechanism:**

```java
// What JdbcBatchItemWriter does INTERNALLY:

PreparedStatement ps = connection.prepareStatement(
    "INSERT INTO orders (id, product, amount) VALUES (?, ?, ?)");

// Step 1: Buffer ALL items locally (NO network calls yet)
for (Order order : chunk) {          // e.g., 500 items
    ps.setLong(1, order.getId());
    ps.setString(2, order.getProduct());
    ps.setBigDecimal(3, order.getAmount());
    ps.addBatch();                   // Just adds to local buffer
}

// Step 2: Send ALL 500 at once (ONE network call)
int[] results = ps.executeBatch();   // 1 network round trip for 500 rows!
```

**Step 2 — Why this is so much faster:**

```
Without batch (500 individual INSERTs):
  App ──INSERT──> DB      (network: 1ms)
  App ──INSERT──> DB      (network: 1ms)
  × 500 times
  Total network latency: 500ms + DB overhead

With batch (1 executeBatch call):
  App ──BATCH of 500──> DB  (network: 1ms, DB processes 500 efficiently)
  Total network latency: 1ms + DB overhead

Speedup: ~50x just from reducing network round trips!
```

**Step 3 — MySQL optimization (rewriteBatchedStatements):**

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:mysql://localhost/mydb?rewriteBatchedStatements=true
```

Without this flag, MySQL driver still sends individual statements. With it, MySQL actually rewrites the batch into a multi-row INSERT: `INSERT INTO orders VALUES (1,'a',10), (2,'b',20), (3,'c',30)...`

### 🗣️ How to Explain in Interview

> *"JdbcBatchItemWriter uses JDBC's batch API internally. For each item in the chunk, it calls PreparedStatement.addBatch() — which just buffers the data locally, no network call yet. When all items are added, it calls executeBatch() which sends everything in one network round trip. This is dramatically faster — instead of 500 individual INSERT statements with 500 network calls, you get 1 call with 500 rows. For MySQL, there's an extra optimization: adding rewriteBatchedStatements=true to the JDBC URL makes MySQL rewrite the batch into a single multi-row INSERT statement, which is even faster."*

### ⚡ Key Points to Remember

1. `addBatch()` = **buffer locally** (no network)
2. `executeBatch()` = **send ALL at once** (one network call)
3. **10-50x faster** than individual inserts
4. MySQL: add `rewriteBatchedStatements=true` to URL
5. Chunk size = batch size (they are directly related)

---

<a id="q47"></a>

## Q47. How do you write to multiple destinations?

### 🔑 Quick Answer

> Three approaches: **CompositeItemWriter** (same data to multiple destinations), **ClassifierCompositeItemWriter** (route items conditionally), or **multiple steps** (separate processing pipelines).

### 📖 Step-by-Step Explanation

**Step 1 — Choose your approach:**

| Need | Approach | Example |
|------|----------|---------|
| Same data → DB + File | `CompositeItemWriter` | Audit trail: insert to DB AND write to CSV |
| Different items → different destinations | `ClassifierCompositeItemWriter` | Domestic → table A, Export → table B |
| Independent processing pipelines | Multiple Steps | Step 1: process + write to DB; Step 2: read DB + write report |

**Step 2 — When to use each:**

```
Same data goes everywhere?
  → CompositeItemWriter
  → Simple, same transaction, delegates execute in order

Different items go to different places?
  → ClassifierCompositeItemWriter
  → Classifier function decides per item

Complete separate processing needed?
  → Multiple Steps
  → Each step has its own reader/processor/writer
  → More code, but fully independent
```

### 🗣️ How to Explain in Interview

> *"There are three approaches. CompositeItemWriter for sending the same chunk to multiple writers — like database and a backup file. ClassifierCompositeItemWriter for routing items conditionally — like valid records to the main table and invalid to an error file. And sometimes, separate steps are the cleanest design — Step 1 processes and writes to database, Step 2 reads from database and generates a report. The advantage of separate steps is complete independence, but the overhead is double the read-write operations."*

### ⚡ Key Points to Remember

1. **Same data → multiple places** = CompositeItemWriter
2. **Different items → different places** = ClassifierCompositeItemWriter
3. **Independent pipelines** = Multiple Steps
4. Composite writers share the **same chunk transaction**
5. Multiple steps = more operations but **cleaner separation**

---

<a id="q48"></a>

## Q48. What happens if writing fails during chunk processing?

### 🔑 Quick Answer

> The **entire chunk rolls back** — none of the items are written. Without skip, the job fails. With skip configured, Spring Batch enters **scan mode**: it re-writes items **one by one** to find and skip the bad item while saving the good ones.

### 📖 Step-by-Step Explanation

**Step 1 — Default (no skip) — the job fails:**

```
Chunk of 500 items:
  Read 500 ✅
  Process 500 ✅
  Write ALL 500 → 💥 DataIntegrityViolationException

  ROLLBACK (all 500 lost)
  Step: FAILED
  Job: FAILED
```

**Step 2 — With skip configured — scan mode:**

```
Chunk of 500 items:
  Read 500 ✅
  Process 500 ✅
  Write ALL 500 → 💥 DataIntegrityViolationException
  ROLLBACK

  → Enter SCAN MODE (one-by-one retry):
     Write [item 1] → ✅ COMMIT
     Write [item 2] → ✅ COMMIT
     ...
     Write [item 247] → 💥 FAIL → SKIP (this is the bad one!)
     ...
     Write [item 500] → ✅ COMMIT

  Result: 499 written ✅, 1 skipped ❌
  Continue to next chunk!
```

**Step 3 — Why scan mode is necessary:**

The writer receives ALL items at once. When `executeBatch()` fails, Spring Batch doesn't know which specific item caused the error. It has to try them individually to find the culprit.

**Step 4 — Performance impact:**

```
Normal chunk: 500 items → 1 write call → fast
Scan mode:    500 items → 500 write calls → SLOW (but only for this one chunk)

In production: scan mode is rare (only when errors occur)
If scan mode happens too often → fix the data/validation, don't rely on skip
```

### 🗣️ How to Explain in Interview

> *"When writing fails, the entire chunk rolls back. Without skip configuration, the job fails immediately. But with skip enabled, Spring Batch enters scan mode — it knows one of the 500 items is bad but doesn't know which one, so it rolls back the batch write and re-tries each item individually. It writes item 1 alone, commits that, writes item 2 alone, commits that, and so on. When it finds the problematic item, it skips it. All the other 499 items are still written successfully. Scan mode is slow — 500 individual writes instead of 1 batch — but it only happens when an error occurs. In production, I always combine skip with a SkipListener to log the failed records."*

### ⚡ Key Points to Remember

1. Writer failure → **entire chunk rolls back**
2. No skip → **job FAILS**
3. With skip → **scan mode** (one-by-one) to find bad item
4. Scan mode is **slow but precise** — identifies exact bad record
5. Always use **SkipListener** to log what was skipped

---

> **🎯 Navigation:** [← Readers (Q31-40)](03-readers.md) | [Next → Processors (Q49-54)](05-processors.md) | [📋 All Sections](README.md)
