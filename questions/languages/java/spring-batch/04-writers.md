# 📤 ItemWriters — Q41 to Q48

---

## Q41. What types of ItemWriter implementations are available in Spring Batch?

### 📝 One-Liner
Spring Batch provides writers for databases (JDBC batch, JPA, Hibernate), files (CSV, JSON, XML), messaging (Kafka, JMS), NoSQL (MongoDB), and composite writers for multiple destinations.

### 🔑 Quick Answer
Built-in writers cover every common destination: **Database**: JdbcBatchItemWriter (fastest, uses JDBC batch), JpaItemWriter, HibernateItemWriter. **Files**: FlatFileItemWriter (CSV/TSV), JsonFileItemWriter, StaxEventItemWriter (XML). **Messaging**: KafkaItemWriter, JmsItemWriter, AmqpItemWriter. **NoSQL**: MongoItemWriter. **Composite**: CompositeItemWriter (same data → multiple destinations), ClassifierCompositeItemWriter (route items conditionally). *(Database ke liye JdbcBatchItemWriter sabse fast hai — JDBC batch operations use karta hai)*

### 📖 How It Works
```
Writer Decision Tree:

Destination?
├── Database
│   ├── Performance critical     → JdbcBatchItemWriter ⭐ (fastest)
│   ├── JPA entities needed      → JpaItemWriter
│   └── Hibernate session        → HibernateItemWriter
├── File
│   ├── CSV / TSV / Fixed-width  → FlatFileItemWriter
│   ├── JSON                     → JsonFileItemWriter
│   └── XML                      → StaxEventItemWriter
├── Messaging
│   ├── Kafka                    → KafkaItemWriter
│   └── JMS / RabbitMQ          → JmsItemWriter / AmqpItemWriter
├── MongoDB                      → MongoItemWriter
├── Same data → multiple places  → CompositeItemWriter
└── Route items by condition     → ClassifierCompositeItemWriter
```

Key difference from readers: writers receive a **List<T>** (entire chunk at once), not one item at a time. This enables batch operations.

### 🗣️ Answering Approach
"Spring Batch provides writers for databases, files, messaging, and NoSQL. For database writes, JdbcBatchItemWriter is the fastest because it uses JDBC batch operations — buffering SQL statements locally and sending them in one network call. JpaItemWriter is convenient when the project uses JPA entities but has ORM overhead. For multi-destination writes, CompositeItemWriter sends the same data to multiple writers, while ClassifierCompositeItemWriter routes items to different writers based on conditions. In my project, we used JdbcBatchItemWriter for the main database insert and CompositeItemWriter to also write an audit log to a separate table."

### 💻 Code
```java
// Database writer (fastest for batch inserts)
@Bean
public JdbcBatchItemWriter<Order> jdbcWriter(DataSource dataSource) {
    return new JdbcBatchItemWriterBuilder<Order>()
            .sql("INSERT INTO processed_orders (id, amount, status) VALUES (:id, :amount, :status)")
            .dataSource(dataSource)
            .beanMapped()  // maps Order fields by name
            .build();
}

// File writer (CSV)
@Bean
public FlatFileItemWriter<Order> csvWriter() {
    return new FlatFileItemWriterBuilder<Order>()
            .name("csvWriter")
            .resource(new FileSystemResource("/output/orders.csv"))
            .delimited().delimiter(",")
            .names("id", "amount", "status")
            .build();
}

// Composite: write to DB + audit file
@Bean
public CompositeItemWriter<Order> compositeWriter() {
    CompositeItemWriter<Order> writer = new CompositeItemWriter<>();
    writer.setDelegates(List.of(jdbcWriter(null), csvWriter()));
    return writer;
}
```

### ⚠️ Pitfalls / Gotchas
- Writers receive a LIST (chunk), not single items — that's what enables batch operations *(writer ko list milti hai, ek item nahi)*
- JdbcBatchItemWriter is 10-50x faster than JpaItemWriter for simple inserts
- CompositeItemWriter delegates run in same transaction — if any fails, all roll back
- Custom writer must implement `ItemWriter<T>` with `write(Chunk<? extends T> items)`

### ⚡ Remember
- JdbcBatchItemWriter = fastest for DB writes ⭐
- Writer receives List (not single item) *(writer ko poora chunk milta hai)*
- Composite = same data → multiple places
- Classifier = route items → different writers by condition
- All delegates in CompositeItemWriter share same transaction

### 🔗 Follow-ups
- [Q42 → JdbcBatchItemWriter deep dive](#q42)
- [Q43 → JpaItemWriter](#q43)
- [Q45 → CompositeItemWriter](#q45)

---

## Q42. What is JdbcBatchItemWriter? How does it work?

### 📝 One-Liner
JdbcBatchItemWriter uses JDBC batch operations (`addBatch()` + `executeBatch()`) to write an entire chunk in one network call — 10-50x faster than individual inserts.

### 🔑 Quick Answer
JdbcBatchItemWriter receives the chunk as a list and prepares SQL statements using `PreparedStatement.addBatch()` for each item (buffered locally, no network). Then it calls `executeBatch()` which sends ALL statements in ONE network call to the database. This is why it's dramatically faster than individual inserts. Two mapping options: `beanMapped()` for automatic field-name mapping or `itemPreparedStatementSetter()` for manual mapping. `assertUpdates(true)` verifies all rows were affected. *(addBatch se buffer karta hai, executeBatch se ek baar mein bhejta hai — 50x fast)*

### 📖 How It Works
```
Individual INSERT (slow):              JDBC Batch (fast):
┌─────────────────────┐               ┌──────────────────────┐
│ INSERT item1 → DB   │ network call  │ addBatch(item1)      │ local buffer
│ INSERT item2 → DB   │ network call  │ addBatch(item2)      │ local buffer
│ INSERT item3 → DB   │ network call  │ addBatch(item3)      │ local buffer
│ ...                  │               │ ...                  │
│ INSERT item500 → DB │ network call  │ addBatch(item500)    │ local buffer
│                      │               │ executeBatch() → DB  │ 1 network call!
│ 500 network calls ❌ │               │ 1 network call ✅    │
└─────────────────────┘               └──────────────────────┘
```

Internal flow:
1. Receive chunk (List of items)
2. For each item: set parameters on PreparedStatement → `addBatch()`
3. After all items: `executeBatch()` → sends all to DB in one call
4. Check affected rows if `assertUpdates(true)`

### 🗣️ Answering Approach
"JdbcBatchItemWriter uses JDBC's batch API — PreparedStatement.addBatch() and executeBatch(). For each item in the chunk, it sets the parameters and calls addBatch, which just buffers the statement locally without any network call. Once all items are batched, executeBatch sends everything to the database in a single network round-trip. In my project, this gave us 50x throughput improvement compared to individual inserts — processing 500K records dropped from 45 minutes to under a minute. We used beanMapped() for automatic field mapping and assertUpdates(true) to verify all rows were inserted."

### 💻 Code
```java
// Using beanMapped (automatic field mapping)
@Bean
public JdbcBatchItemWriter<Order> beanMappedWriter(DataSource dataSource) {
    return new JdbcBatchItemWriterBuilder<Order>()
            .sql("INSERT INTO orders (id, customer_name, amount, status) " +
                 "VALUES (:id, :customerName, :amount, :status)")
            .dataSource(dataSource)
            .beanMapped()              // maps :fieldName to object fields
            .assertUpdates(true)       // verify each INSERT affected 1 row
            .build();
}

// Using manual PreparedStatementSetter
@Bean
public JdbcBatchItemWriter<Order> manualWriter(DataSource dataSource) {
    return new JdbcBatchItemWriterBuilder<Order>()
            .sql("INSERT INTO orders (id, customer_name, amount, status) " +
                 "VALUES (?, ?, ?, ?)")
            .dataSource(dataSource)
            .itemPreparedStatementSetter((order, ps) -> {
                ps.setLong(1, order.getId());
                ps.setString(2, order.getCustomerName());
                ps.setBigDecimal(3, order.getAmount());
                ps.setString(4, order.getStatus());
            })
            .build();
}

// UPSERT (INSERT or UPDATE)
@Bean
public JdbcBatchItemWriter<Order> upsertWriter(DataSource dataSource) {
    return new JdbcBatchItemWriterBuilder<Order>()
            .sql("INSERT INTO orders (id, amount, status) VALUES (:id, :amount, :status) " +
                 "ON DUPLICATE KEY UPDATE amount = :amount, status = :status")
            .dataSource(dataSource)
            .beanMapped()
            .assertUpdates(false)  // UPSERT may affect 1 or 2 rows → disable assertion
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- `assertUpdates(true)` throws exception if any INSERT affects 0 rows — set `false` for UPSERT *(UPSERT ke liye assertUpdates false karo)*
- MySQL: add `?rewriteBatchedStatements=true` to JDBC URL for real multi-row INSERT rewriting
- `beanMapped()` uses named parameters (`:fieldName`), manual uses positional (`?`)
- Don't use with JPA entities — use JpaItemWriter instead (or entity context won't sync)
- Chunk write is atomic — if one item fails in batch, entire chunk rolls back

### 🆚 vs. Comparison
| Aspect | JdbcBatchItemWriter | JpaItemWriter |
|--------|--------------------|---------------|
| Speed | ⚡ 10-50x faster | Slower (ORM overhead) |
| Mapping | Manual (SQL + beanMapped) | Automatic (entity mapping) |
| Relationships | Manual JOINs | Auto cascading |
| Memory | Low | Higher (entity cache) |
| Best for | High-throughput batch | JPA-centric projects |

### 🎯 Tricky Interview Qs

**Q: How does MySQL optimize batch inserts further?**
With `rewriteBatchedStatements=true`, MySQL JDBC driver rewrites multiple `INSERT INTO t VALUES (a)` statements into a single `INSERT INTO t VALUES (a),(b),(c)` — even faster than standard batch.

**Q: What's the internal difference between addBatch and executeBatch?**
`addBatch()` only buffers the SQL locally in the JDBC driver — zero network call. `executeBatch()` sends all buffered statements to the database in one network round-trip.

### ⚡ Remember
- `addBatch()` = local buffer (no network), `executeBatch()` = send all once *(addBatch sirf local, executeBatch ek baar mein bhejta hai)*
- 10-50x faster than individual inserts
- `beanMapped()` for auto mapping, `itemPreparedStatementSetter` for manual
- `assertUpdates(false)` for UPSERT
- MySQL: `rewriteBatchedStatements=true` for extra boost

### 🔗 Follow-ups
- [Q43 → JpaItemWriter comparison](#q43)
- [Q46 → Batch insert internals](#q46)
- [Q48 → Write failure handling](#q48)

---

## Q43. What is JpaItemWriter?

### 📝 One-Liner
JpaItemWriter uses `EntityManager.merge()` to persist JPA entities — convenient for JPA projects but slower than JdbcBatchItemWriter due to ORM overhead.

### 🔑 Quick Answer
JpaItemWriter receives a chunk of JPA entities and calls `EntityManager.merge()` for each entity. It handles INSERT (new entities) and UPDATE (existing entities) automatically. It manages entity lifecycle including cascading relationships. The downside: it's significantly slower than JdbcBatchItemWriter because of ORM overhead — dirty checking, proxy creation, first-level cache, and flush operations. Use it when your project already uses JPA and you need entity-level features. *(JPA entities ke liye convenient hai but JDBC batch se slow)*

### 📖 How It Works
```
JpaItemWriter Internal Flow:

Chunk (List of entities)
  ↓
For each entity:
  EntityManager.merge(entity)
  ├── New entity (no ID) → INSERT
  └── Existing entity (has ID) → UPDATE (if dirty)
  ↓
EntityManager.flush()  ← sends SQL to DB
  ↓
Transaction COMMIT

ORM Overhead per entity:
├── Dirty checking (compare against snapshot)
├── Proxy object creation
├── First-level cache management
├── Cascade operations (relationships)
└── Flush SQL generation
```

### 🗣️ Answering Approach
"JpaItemWriter persists entities using EntityManager.merge. It automatically handles inserts for new entities and updates for existing ones, including cascading relationships. In my project, we used it when processing Order entities with OrderItems — the cascading persist was very convenient. However, when we benchmarked it against JdbcBatchItemWriter for a high-throughput migration job, JPA was 30% slower due to dirty checking and cache management. So we used JpaItemWriter for complex domain operations and JdbcBatchItemWriter for high-throughput simple inserts."

### 💻 Code
```java
@Bean
public JpaItemWriter<Employee> jpaWriter(EntityManagerFactory emf) {
    JpaItemWriter<Employee> writer = new JpaItemWriter<>();
    writer.setEntityManagerFactory(emf);
    return writer;
}

// With cascading relationships
@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;
    private BigDecimal amount;

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items;  // auto-persisted with order
}

// JpaItemWriter will cascade-insert Order + all OrderItems

// Clearing cache to prevent memory issues
@Component
public class JpaClearListener implements ChunkListener {
    @PersistenceContext
    private EntityManager em;

    @Override
    public void afterChunk(ChunkContext context) {
        em.clear();  // free first-level cache after each chunk
    }
}
```

### ⚠️ Pitfalls / Gotchas
- MUST use `JpaTransactionManager` — `DataSourceTransactionManager` won't flush entities *(JpaTransactionManager zaruri hai — DataSource wala kaam nahi karega)*
- First-level cache grows with each chunk → use ChunkListener to `clear()` EntityManager
- `merge()` always does a SELECT first (to check if entity exists) — extra query per entity
- For bulk inserts with no relationships, JdbcBatchItemWriter is dramatically faster
- `usePersist(true)` skips the SELECT check — use when you KNOW entities are new

### 🎯 Tricky Interview Qs

**Q: What's the difference between persist() and merge() in JpaItemWriter?**
`merge()` (default) does a SELECT first to check if entity exists → INSERT or UPDATE. `persist()` skips the check and always does INSERT. Use `usePersist(true)` when you know all entities are new — avoids the extra SELECT.

**Q: How to improve JpaItemWriter performance?**
(1) Set `usePersist(true)` for new entities, (2) clear EntityManager cache in ChunkListener, (3) batch JDBC statements with `hibernate.jdbc.batch_size=500`, (4) disable dirty checking where possible.

### ⚡ Remember
- Uses `EntityManager.merge()` per entity
- Auto handles INSERT/UPDATE + cascade *(naya entity toh INSERT, purana toh UPDATE — automatic)*
- Slower than JdbcBatchItemWriter (ORM overhead)
- MUST use JpaTransactionManager
- Clear EntityManager cache in ChunkListener to avoid OOM

### 🔗 Follow-ups
- [Q42 → JdbcBatchItemWriter (faster)](#q42)
- [Q35 → JpaPagingItemReader (pair together)](#q35)
- [Q27 → Memory issues with JPA cache](#q27)

---

## Q44. What is FlatFileItemWriter?

### 📝 One-Liner
FlatFileItemWriter writes data to text files (CSV, TSV, fixed-width) with support for headers, footers, and append mode.

### 🔑 Quick Answer
FlatFileItemWriter is the counterpart of FlatFileItemReader. It writes items to a text file line by line. It supports **delimited** (CSV) and **formatted** (fixed-width) output. Features include: `headerCallback` for file headers, `footerCallback` for footers, `append(true)` to add to existing file, `shouldDeleteIfEmpty(true)` to remove empty output files. Each item is converted to a line using field names and a delimiter. *(CSV ya text file mein likhna ho toh FlatFileItemWriter use karo)*

### 📖 How It Works
```
FlatFileItemWriter Pipeline:

Java Object    →    LineAggregator    →    Text File
┌──────────┐     ┌────────────────┐     ┌──────────────────┐
│ Employee │  →  │ "1,John,50000" │  →  │ id,name,salary   │ ← header
│ id=1     │     │ DelimitedLine  │     │ 1,John,50000     │
│ name=John│     │ or Formatted   │     │ 2,Jane,60000     │
└──────────┘     └────────────────┘     │ Total: 2 records │ ← footer
                                        └──────────────────┘
```

### 🗣️ Answering Approach
"FlatFileItemWriter writes items to text files. It's the writing counterpart of FlatFileItemReader. I configure it with a delimiter for CSV output, field names to extract from the Java object, and optional header and footer callbacks. In my project, we used it to generate daily reconciliation CSV files — with a header row of column names, the data rows, and a footer with record count and checksum. We also used shouldDeleteIfEmpty(true) so that if the step had no data, no empty file was created."

### 💻 Code
```java
@Bean
public FlatFileItemWriter<Employee> csvWriter() {
    return new FlatFileItemWriterBuilder<Employee>()
            .name("empCsvWriter")
            .resource(new FileSystemResource("/output/employees.csv"))
            .delimited()
            .delimiter(",")
            .names("id", "name", "salary", "department")
            .headerCallback(writer -> writer.write("id,name,salary,department"))
            .footerCallback(writer -> writer.write("END OF FILE"))
            .shouldDeleteIfEmpty(true)   // no data → delete output file
            .build();
}

// Append mode (add to existing file)
@Bean
public FlatFileItemWriter<Order> appendWriter() {
    return new FlatFileItemWriterBuilder<Order>()
            .name("orderAppendWriter")
            .resource(new FileSystemResource("/output/orders_audit.csv"))
            .append(true)               // append, don't overwrite
            .delimited().delimiter("|")
            .names("id", "amount", "processedAt")
            .build();
}

// Fixed-width (formatted) output
@Bean
public FlatFileItemWriter<Employee> fixedWidthWriter() {
    return new FlatFileItemWriterBuilder<Employee>()
            .name("fixedWriter")
            .resource(new FileSystemResource("/output/employees.dat"))
            .formatted()
            .format("%-5d%-20s%10.2f")  // id(5), name(20), salary(10)
            .names("id", "name", "salary")
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Default mode OVERWRITES file — use `.append(true)` if you want to add to existing file *(default mein purani file overwrite ho jaati hai)*
- `headerCallback` and `footerCallback` are NOT called if zero items written AND `shouldDeleteIfEmpty(true)`
- File is created at step start, not first write — empty file exists even with no data (unless shouldDeleteIfEmpty)
- NOT thread-safe — don't use directly in multi-threaded steps
- Encoding: set `.encoding("UTF-8")` explicitly (same as reader)

### 🎯 Tricky Interview Qs

**Q: How do you write record count in footer when you don't know it upfront?**
Use `StepExecution` in the footer callback — inject it via `@BeforeStep` or use the step execution listener to access write count:
```java
.footerCallback(writer -> 
    writer.write("Total records: " + stepExecution.getWriteCount()))
```

### ⚡ Remember
- Counterpart of FlatFileItemReader
- Supports delimited (CSV) and formatted (fixed-width) *(CSV aur fixed-width dono support karta hai)*
- Header/Footer callbacks for custom headers/footers
- `append(true)` for adding to existing file
- `shouldDeleteIfEmpty(true)` removes empty output files

### 🔗 Follow-ups
- [Q32 → FlatFileItemReader (reading counterpart)](#q32)
- [Q45 → CompositeItemWriter for multi-destination](#q45)
- [Q41 → All writer types](#q41)

---

## Q45. What is CompositeItemWriter? What about ClassifierCompositeItemWriter?

### 📝 One-Liner
CompositeItemWriter sends the same chunk to multiple writers; ClassifierCompositeItemWriter routes each item to a different writer based on a condition.

### 🔑 Quick Answer
**CompositeItemWriter**: wraps multiple writers as delegates — every item goes to ALL writers. Use case: write to DB AND audit file. **ClassifierCompositeItemWriter**: uses a classifier (condition function) to decide WHICH writer handles each item. Use case: domestic orders → table A, export orders → table B. Both execute within the same chunk transaction. *(Composite = sab jagah bhejo, Classifier = condition ke hisaab se alag alag jagah bhejo)*

### 📖 How It Works
```
CompositeItemWriter (same data → multiple destinations):
┌──────────┐
│ Chunk    │──→ Writer 1 (DB insert)     ← ALL items go to ALL writers
│ [A,B,C]  │──→ Writer 2 (audit file)
│          │──→ Writer 3 (Kafka event)
└──────────┘

ClassifierCompositeItemWriter (route by condition):
┌──────────┐     ┌──────────────────────┐
│ Chunk    │──→  │ Classifier:          │
│ [A,B,C]  │     │ A → type=DOMESTIC    │──→ Writer 1 (domestic_orders table)
│          │     │ B → type=EXPORT      │──→ Writer 2 (export_orders table)
│          │     │ C → type=DOMESTIC    │──→ Writer 1 (domestic_orders table)
└──────────┘     └──────────────────────┘
```

### 🗣️ Answering Approach
"CompositeItemWriter sends the same data to multiple destinations by delegating to multiple writers — useful when you need to write to a database and simultaneously create an audit file. ClassifierCompositeItemWriter uses a classification function to route each item to a different writer based on a condition. In my project, we used CompositeItemWriter to write processed payments to the main table and also to an audit log table for compliance. In another job, we used ClassifierCompositeItemWriter to route transactions — domestic payments went to one table and international payments to a separate table with different schemas."

### 💻 Code
```java
// CompositeItemWriter: same data → multiple destinations
@Bean
public CompositeItemWriter<Order> compositeWriter() {
    CompositeItemWriter<Order> writer = new CompositeItemWriter<>();
    writer.setDelegates(List.of(
            dbWriter(),        // write to main DB table
            auditFileWriter(), // write to audit CSV
            kafkaWriter()      // publish to Kafka topic
    ));
    return writer;
}

// ClassifierCompositeItemWriter: route by condition
@Bean
public ClassifierCompositeItemWriter<Order> classifierWriter() {
    ClassifierCompositeItemWriter<Order> writer = new ClassifierCompositeItemWriter<>();
    writer.setClassifier(new Classifier<Order, ItemWriter<? super Order>>() {
        @Override
        public ItemWriter<? super Order> classify(Order order) {
            if ("DOMESTIC".equals(order.getType())) {
                return domesticWriter();
            } else {
                return exportWriter();
            }
        }
    });
    return writer;
}

// Lambda version (cleaner)
@Bean
public ClassifierCompositeItemWriter<Order> classifierWriterLambda() {
    ClassifierCompositeItemWriter<Order> writer = new ClassifierCompositeItemWriter<>();
    writer.setClassifier(order ->
        "DOMESTIC".equals(order.getType()) ? domesticWriter() : exportWriter()
    );
    return writer;
}
```

### ⚠️ Pitfalls / Gotchas
- All delegates in CompositeItemWriter share same transaction — if writer 3 fails, writers 1 & 2 also roll back *(ek writer fail hua toh sab rollback — same transaction hai)*
- ClassifierCompositeItemWriter delegates must be registered with `setClassifier` and also as streams if they need lifecycle management (open/close)
- Order matters in CompositeItemWriter — delegates execute in list order
- Kafka/JMS writes are NOT transactional by default — won't roll back with DB

### 🆚 vs. Comparison
| Aspect | CompositeItemWriter | ClassifierCompositeItemWriter |
|--------|--------------------|-----------------------------|
| Routing | ALL items → ALL writers | Each item → ONE writer |
| Use case | DB + audit + event | Route by type/condition |
| Transaction | Shared (all or nothing) | Shared (all or nothing) |
| Config | List of delegates | Classifier function |
| Example | Write to DB + CSV + Kafka | Domestic → table A, Export → table B |

### 🎯 Tricky Interview Qs

**Q: What if you need some items to go to multiple writers AND some to be routed?**
Combine both: use CompositeItemWriter with ClassifierCompositeItemWriter as one of its delegates. Items go to the common writers AND are routed by the classifier.

**Q: Are they transactional?**
Yes — all delegates share the same chunk transaction. But external systems (Kafka, REST) don't participate in JDBC transactions, so they won't rollback.

### ⚡ Remember
- **Composite** = ALL writers get ALL items *(sab jagah sab data jaata hai)*
- **Classifier** = each item routed to ONE writer by condition
- Both share same chunk transaction
- External writes (Kafka, REST) not rollback-safe
- Can nest: Composite containing Classifier

### 🔗 Follow-ups
- [Q47 → Writing to multiple destinations](#q47)
- [Q41 → All writer types overview](#q41)
- [Q42 → JdbcBatchItemWriter as delegate](#q42)

---

## Q46. How does batch insert work internally in JdbcBatchItemWriter?

### 📝 One-Liner
`addBatch()` buffers SQL statements locally in the JDBC driver; `executeBatch()` sends all of them to the database in a single network round-trip.

### 🔑 Quick Answer
Internally, JdbcBatchItemWriter creates a `PreparedStatement`, then for each item in the chunk: sets parameters and calls `ps.addBatch()`. `addBatch()` only buffers the statement in the JDBC driver's memory — zero network calls. After all items are batched, `ps.executeBatch()` sends the entire batch to the database in ONE network call. The database executes all statements and returns affected row counts. With MySQL's `rewriteBatchedStatements=true`, the driver further optimizes by rewriting individual INSERTs into a multi-row INSERT. *(addBatch sirf local memory mein rakhta hai — network pe kuch nahi jaata. executeBatch ek baar mein sab bhejta hai)*

### 📖 How It Works
```
Step-by-Step Internal Flow:

1. PreparedStatement ps = connection.prepareStatement(SQL);

2. For each item in chunk:
   ps.setLong(1, item.getId());        // set params
   ps.setString(2, item.getName());
   ps.addBatch();                      // ← LOCAL BUFFER ONLY, no network!

3. int[] results = ps.executeBatch();  // ← ONE network call, all statements sent

4. Verify: check each results[i] == 1  // assertUpdates

Without rewriteBatchedStatements:
  → Driver sends: INSERT...; INSERT...; INSERT...  (individual statements)

With rewriteBatchedStatements=true (MySQL):
  → Driver rewrites to: INSERT INTO t VALUES (a,b), (c,d), (e,f)  (multi-row)
  → Even fewer DB-side parses
```

### 🗣️ Answering Approach
"JdbcBatchItemWriter leverages the JDBC batch API. For each item, it sets parameters on a PreparedStatement and calls addBatch, which only buffers the statement locally in the driver — no network communication. Once all items are batched, executeBatch sends everything in a single network round-trip. In my project, this reduced 500 individual INSERT network calls to just 1, cutting our write time by 98%. We also enabled rewriteBatchedStatements=true in MySQL, which further optimized by rewriting the batch into a multi-row INSERT statement."

### 💻 Code
```java
// What happens internally (pseudo-code):
public void write(Chunk<? extends Order> items) {
    PreparedStatement ps = connection.prepareStatement(
        "INSERT INTO orders (id, name, amount) VALUES (?, ?, ?)");

    for (Order order : items) {
        ps.setLong(1, order.getId());
        ps.setString(2, order.getName());
        ps.setBigDecimal(3, order.getAmount());
        ps.addBatch();   // buffered locally — NO network call
    }

    int[] results = ps.executeBatch();  // ONE network call for all 500 items

    // assertUpdates: verify each row was affected
    for (int result : results) {
        if (result == 0) throw new EmptyResultDataAccessException(1);
    }
}

// MySQL optimization in application.properties:
// spring.datasource.url=jdbc:mysql://localhost/batch?rewriteBatchedStatements=true
```

### ⚠️ Pitfalls / Gotchas
- `rewriteBatchedStatements=true` is MySQL-specific — PostgreSQL has different optimizations *(ye sirf MySQL ke liye hai)*
- Without `rewriteBatchedStatements`, MySQL still sends individual statements (just in one call)
- Some drivers have a max batch size — very large batches (10K+) may get split
- `executeBatch()` returns `int[]` of affected rows — `-2` means success but count unknown (JDBC `SUCCESS_NO_INFO`)

### ⚡ Remember
- `addBatch()` = buffer locally (zero network) *(addBatch = sirf local buffer)*
- `executeBatch()` = send all in one call (one network)
- 10-50x faster than individual inserts
- MySQL: `rewriteBatchedStatements=true` for multi-row INSERT
- assertUpdates checks the returned row counts

### 🔗 Follow-ups
- [Q42 → JdbcBatchItemWriter config](#q42)
- [Q48 → What if write fails](#q48)
- [Q92 → Performance tuning](#q92)

---

## Q47. How do you write to multiple destinations?

### 📝 One-Liner
Use CompositeItemWriter for same data to multiple places, ClassifierCompositeItemWriter for routing items to different places, or separate steps for independent pipelines.

### 🔑 Quick Answer
Three approaches: **(1) CompositeItemWriter** — same chunk goes to ALL delegate writers (e.g., DB + file + Kafka). **(2) ClassifierCompositeItemWriter** — each item is routed to ONE writer based on a condition (e.g., type=A → writer1, type=B → writer2). **(3) Multiple Steps** — completely independent pipelines with separate readers, processors, writers. Composite/Classifier share same transaction; Steps are independent. *(Same data sab jagah = Composite, alag data alag jagah = Classifier, bilkul alag pipeline = Steps)*

### 📖 How It Works
```
Approach Selection:

Same data → multiple places?     → CompositeItemWriter
  (DB + audit file + event)

Different items → different places? → ClassifierCompositeItemWriter
  (route by type/condition)

Completely independent?            → Separate Steps in Job
  (different readers, no shared data)

Decision Matrix:
| Same Data? | Same Transaction? | Approach |
|------------|-------------------|----------|
| Yes        | Yes               | CompositeItemWriter |
| No (routed)| Yes               | ClassifierCompositeItemWriter |
| Independent| No                | Separate Steps |
```

### 🗣️ Answering Approach
"There are three approaches depending on the requirement. CompositeItemWriter sends the same data to multiple writers — I used this when we needed to write orders to the database and simultaneously to an audit CSV file. ClassifierCompositeItemWriter routes each item to a specific writer based on a condition — we used this to route transactions by type. For completely independent data flows, I use separate steps in the job. The key difference is that composite writers share the same chunk transaction for atomicity, while separate steps are independent."

### 💻 Code
```java
// Approach 1: CompositeItemWriter (same data → multiple places)
@Bean
public CompositeItemWriter<Order> multiDestWriter() {
    CompositeItemWriter<Order> writer = new CompositeItemWriter<>();
    writer.setDelegates(List.of(dbWriter(), auditCsvWriter(), kafkaWriter()));
    return writer;
}

// Approach 2: ClassifierCompositeItemWriter (route by condition)
@Bean
public ClassifierCompositeItemWriter<Order> routingWriter() {
    ClassifierCompositeItemWriter<Order> writer = new ClassifierCompositeItemWriter<>();
    writer.setClassifier(order -> switch (order.getRegion()) {
        case "US" -> usWriter();
        case "EU" -> euWriter();
        default -> globalWriter();
    });
    return writer;
}

// Approach 3: Separate Steps (independent pipelines)
@Bean
public Job multiPipelineJob(JobRepository repo, Step orderStep, Step invoiceStep) {
    return new JobBuilder("multiPipelineJob", repo)
            .start(orderStep)     // completely independent pipeline
            .next(invoiceStep)    // different reader, processor, writer
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Composite writers share transaction — external writes (REST, Kafka) WON'T rollback with DB *(Kafka/REST rollback nahi hoga DB ke saath)*
- Separate steps run sequentially — use parallel steps for concurrent execution
- ClassifierCompositeItemWriter delegates must be registered as ItemStream if they need open/close lifecycle
- Don't mix transactional (DB) and non-transactional (file, Kafka) in composite if atomicity is critical

### ⚡ Remember
- **Same data everywhere** → CompositeItemWriter
- **Route by condition** → ClassifierCompositeItemWriter
- **Independent flows** → Separate Steps *(alag pipeline = alag step)*
- Composite/Classifier = same transaction
- Separate Steps = independent transactions

### 🔗 Follow-ups
- [Q45 → CompositeItemWriter details](#q45)
- [Q42 → JdbcBatchItemWriter as delegate](#q42)
- [Q84 → Parallel step execution](#q84)

---

## Q48. What happens if writing fails during chunk processing?

### 📝 One-Liner
Without skip, the entire chunk rolls back and the job fails; with skip enabled, Spring Batch enters "scan mode" — re-writes items one-by-one to identify and skip the bad item.

### 🔑 Quick Answer
When a write fails: **(1) Without fault tolerance**: entire chunk transaction rolls back, step fails, job fails. **(2) With skip**: Spring Batch enters **scan mode** — it rolls back the chunk, then re-processes and re-writes each item INDIVIDUALLY to find which specific item caused the failure. The bad item is skipped, good items are saved. **(3) With retry**: the write is retried N times for transient errors (deadlocks, timeouts). Scan mode is unique to write failures because the writer receives the entire chunk as a list. *(Write fail hone par scan mode — ek ek item likhta hai pata lagane ke liye kaunsa kharab hai)*

### 📖 How It Works
```
Write Failure WITHOUT skip:
  Chunk [A, B, C, D, E] → writer.write() → EXCEPTION
  → ROLLBACK entire chunk → STEP FAILED → JOB FAILED

Write Failure WITH skip (SCAN MODE):
  Chunk [A, B, C, D, E] → writer.write() → EXCEPTION
  → ROLLBACK chunk
  → Enter SCAN MODE:
    ├── write([A]) → ✅ COMMIT (individual transaction)
    ├── write([B]) → ✅ COMMIT
    ├── write([C]) → ❌ EXCEPTION → SKIP C → log to SkipListener
    ├── write([D]) → ✅ COMMIT
    └── write([E]) → ✅ COMMIT
  → C skipped, A,B,D,E saved
  → Continue with next chunk
```

### 🗣️ Answering Approach
"When a write fails, behavior depends on fault tolerance configuration. Without skip, the entire chunk rolls back and the job fails. With skip enabled, Spring Batch enters scan mode — since the writer received the entire chunk as a list, it doesn't know which specific item caused the failure. So it rolls back, then re-processes and re-writes each item individually in its own transaction. The item that fails again is skipped and logged via SkipListener, while all good items are saved. In my project, we had this for payment processing — one invalid payment in a chunk of 500 shouldn't fail the other 499. We configured skip with a SkipListener that wrote bad records to an error file for manual review."

### 💻 Code
```java
@Bean
public Step resilientStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("resilientStep", repo)
            .<Order, ProcessedOrder>chunk(500, tx)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .faultTolerant()
            .skip(DataIntegrityViolationException.class)    // skip constraint violations
            .skip(DuplicateKeyException.class)               // skip duplicates
            .skipLimit(100)                                   // max 100 skips
            .retry(DeadlockLoserDataAccessException.class)   // retry deadlocks
            .retry(TransientDataAccessException.class)       // retry transient DB errors
            .retryLimit(3)
            .listener(new SkipListener<Order, ProcessedOrder>() {
                @Override
                public void onSkipInWrite(ProcessedOrder item, Throwable t) {
                    log.error("SKIPPED in write: id={}, error={}", 
                        item.getId(), t.getMessage());
                    errorFileWriter.write(item);  // write to error file
                }
            })
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Scan mode is SLOW — chunk of 500 becomes 500 individual writes *(scan mode bahut slow hai — 500 items ek ek karke likhta hai)*
- Large chunks + frequent write failures = very slow scan mode → reduce chunk size
- Skip limit is TOTAL across read + process + write skips
- Scan mode re-runs processor too (not just writer) — processor must be idempotent
- `noSkip()` excludes specific exception types from being skippable

### 🎯 Tricky Interview Qs

**Q: Why does Spring Batch need scan mode for write failures but not read/process failures?**
Because reads and processing happen one item at a time — Spring Batch knows exactly which item failed. But writes get the entire chunk as a list, so it can't identify the failing item without testing each one individually.

**Q: Does scan mode affect performance?**
Yes, dramatically. A chunk of 500 items normally does 1 batch write. In scan mode, it does 500 individual writes. If write failures are frequent, scan mode kills performance. Solution: reduce chunk size or fix the root cause of write failures.

### ⚡ Remember
- No skip → one failure kills job *(bina skip ke ek error = poora job fail)*
- Skip → scan mode: re-write items one-by-one to find bad one
- Scan mode is SLOW (500 items = 500 individual writes)
- Always add SkipListener for audit/logging
- Retry for transient errors, Skip for data errors

### 🔗 Follow-ups
- [Q24 → Chunk failure general behavior](#q24)
- [Q42 → JdbcBatchItemWriter internals](#q42)
- [Q70 → Error handling strategies](#q70)
