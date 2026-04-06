# 📥 ItemReaders — Q31 to Q40

---

## Q31. What types of ItemReader implementations are available in Spring Batch?

### 📝 One-Liner
Spring Batch provides 15+ built-in readers for files, databases (cursor & paging), messaging queues, and NoSQL stores.

### 🔑 Quick Answer
Built-in readers cover every common data source: **Files** (FlatFileItemReader for CSV/TSV, JsonItemReader, StaxEventItemReader for XML), **Database** (JdbcCursorItemReader, JdbcPagingItemReader, JpaPagingItemReader, HibernateCursorItemReader, StoredProcedureItemReader, RepositoryItemReader), **Messaging** (KafkaItemReader, JmsItemReader, AmqpItemReader), **NoSQL** (MongoItemReader). For databases, the most important decision is **cursor vs paging**. *(Database ke liye sabse important faisla — cursor reader ya paging reader)*

### 📖 How It Works
```
Reader Decision Tree:

Data Source?
├── Flat File (CSV/TSV/Fixed) → FlatFileItemReader
├── JSON File                  → JsonItemReader
├── XML File                   → StaxEventItemReader
├── Database
│   ├── Simple query, < 1M rows, single thread → JdbcCursorItemReader
│   ├── Large data, multi-thread, production   → JdbcPagingItemReader ⭐
│   ├── JPA entities needed                    → JpaPagingItemReader
│   └── Stored procedure                       → StoredProcedureItemReader
├── Kafka                      → KafkaItemReader
├── JMS / RabbitMQ             → JmsItemReader / AmqpItemReader
├── MongoDB                    → MongoItemReader
└── Custom source              → Implement ItemReader<T> interface
```

All readers follow the same contract:
- `read()` returns one item at a time
- Returns `null` when no more data (signals end)
- Spring Batch calls `read()` repeatedly until null

### 🗣️ Answering Approach
"Spring Batch provides built-in readers for files, databases, messaging, and NoSQL. For files, FlatFileItemReader handles CSV and fixed-width formats. For databases, the key decision is between cursor and paging readers — cursor holds one DB connection for the entire step and is simpler but not thread-safe, while paging reader fetches data page by page, releases connections between pages, and is thread-safe. In my project, we used JdbcPagingItemReader for production because we needed multi-threaded steps for performance, and JpaPagingItemReader when we needed entity mapping for complex domain objects."

### 💻 Code
```java
// File reader
@Bean
public FlatFileItemReader<Order> csvReader() {
    return new FlatFileItemReaderBuilder<Order>()
            .name("csvReader")
            .resource(new FileSystemResource("orders.csv"))
            .delimited().delimiter(",")
            .names("id", "amount", "date")
            .targetType(Order.class)
            .linesToSkip(1)  // skip header
            .build();
}

// Database reader (paging — recommended for production)
@Bean
public JdbcPagingItemReader<Order> dbReader() {
    return new JdbcPagingItemReaderBuilder<Order>()
            .name("dbReader")
            .dataSource(dataSource)
            .selectClause("SELECT id, amount, status")
            .fromClause("FROM orders")
            .whereClause("WHERE status = 'PENDING'")
            .sortKeys(Map.of("id", Order.ASCENDING))
            .pageSize(500)
            .rowMapper(new BeanPropertyRowMapper<>(Order.class))
            .build();
}

// Custom reader — implement the interface
@Component
public class ApiReader implements ItemReader<ApiRecord> {
    @Override
    public ApiRecord read() {
        // Return next item, or null when done
        return hasNext() ? fetchNext() : null;
    }
}
```

### ⚠️ Pitfalls / Gotchas
- Custom reader MUST return null to signal end of data — otherwise infinite loop *(null return nahi kiya toh infinite loop chalega)*
- FlatFileItemReader is NOT thread-safe — don't use in multi-threaded step without synchronization
- Cursor readers hold DB connection for entire step — risk of timeout on large datasets
- Reader `read()` is called inside the chunk transaction

### ⚡ Remember
- 15+ built-in readers covering files, DB, messaging, NoSQL
- Database: Cursor (simple, not thread-safe) vs Paging (production, thread-safe) *(cursor simple hai, paging production ke liye)*
- All readers: return item or null (null = done)
- Custom reader: implement `ItemReader<T>` interface
- Most used: FlatFileItemReader, JdbcPagingItemReader

### 🔗 Follow-ups
- [Q32 → FlatFileItemReader details](#q32)
- [Q33 → JdbcCursorItemReader](#q33)
- [Q34 → JdbcPagingItemReader](#q34)
- [Q36 → Cursor vs Paging comparison](#q36)

---

## Q32. What is FlatFileItemReader? How does it work?

### 📝 One-Liner
FlatFileItemReader reads text files (CSV, TSV, fixed-width) line by line, tokenizes each line into fields, and maps them to Java objects.

### 🔑 Quick Answer
FlatFileItemReader reads one line at a time from a text file. It uses a **LineTokenizer** to split the line into fields (delimited by comma, tab, or fixed column positions) and a **FieldSetMapper** to map those fields to a Java object. You can skip header lines with `linesToSkip()`, set encoding, and handle multi-line records. It supports both delimited (CSV) and fixed-width formats. *(Ek line padhta hai, fields mein todta hai, Java object mein map karta hai)*

### 📖 How It Works
```
FlatFileItemReader Pipeline:

CSV File                LineTokenizer           FieldSetMapper        Java Object
┌──────────────┐     ┌──────────────────┐    ┌────────────────┐    ┌──────────┐
│ 1,John,50000 │  →  │ [1] [John] [50000]│ →  │ new Employee() │ →  │ Employee │
│ 2,Jane,60000 │     │ DelimitedLineToken│    │ .setId(1)      │    │ id=1     │
│ ...          │     │ or FixedLength    │    │ .setName(John) │    │ name=John│
└──────────────┘     └──────────────────┘    └────────────────┘    └──────────┘
      ↑                                              
  linesToSkip(1) — skips header row
```

Two tokenizer types:
1. **DelimitedLineTokenizer** — splits by delimiter (comma, tab, pipe)
2. **FixedLengthTokenizer** — splits by column positions (columns 0-5, 6-20, 21-30)

### 🗣️ Answering Approach
"FlatFileItemReader reads text files line by line. Internally, it uses a LineTokenizer to split each line into fields — DelimitedLineTokenizer for CSV or FixedLengthTokenizer for fixed-width files — and a FieldSetMapper to map those fields to Java objects. In my project, we used it to read daily payment CSV files from an SFTP server. We configured linesToSkip(1) to skip the header, set the encoding to UTF-8, and used a custom FieldSetMapper for date parsing since the date format in the CSV was non-standard."

### 💻 Code
```java
// Delimited (CSV) reader
@Bean
public FlatFileItemReader<Employee> csvReader() {
    return new FlatFileItemReaderBuilder<Employee>()
            .name("empCsvReader")
            .resource(new FileSystemResource("/data/employees.csv"))
            .encoding("UTF-8")
            .linesToSkip(1)   // skip header row
            .delimited()
            .delimiter(",")   // comma-separated (default)
            .names("id", "name", "salary", "department")  // column names
            .targetType(Employee.class)  // auto-maps by field name
            .build();
}

// Fixed-width reader
@Bean
public FlatFileItemReader<Employee> fixedWidthReader() {
    return new FlatFileItemReaderBuilder<Employee>()
            .name("empFixedReader")
            .resource(new FileSystemResource("/data/employees.dat"))
            .fixedLength()
            .columns(new Range(1, 5), new Range(6, 25), new Range(26, 35))
            .names("id", "name", "salary")
            .targetType(Employee.class)
            .build();
}

// Custom FieldSetMapper for complex mapping
@Bean
public FlatFileItemReader<Payment> customReader() {
    return new FlatFileItemReaderBuilder<Payment>()
            .name("paymentReader")
            .resource(new FileSystemResource("/data/payments.csv"))
            .delimited().delimiter("|")    // pipe-delimited
            .names("txnId", "amount", "date", "status")
            .fieldSetMapper(fieldSet -> {
                Payment p = new Payment();
                p.setTxnId(fieldSet.readString("txnId"));
                p.setAmount(fieldSet.readBigDecimal("amount"));
                p.setDate(LocalDate.parse(fieldSet.readString("date"),
                    DateTimeFormatter.ofPattern("dd-MM-yyyy")));
                p.setStatus(PaymentStatus.valueOf(fieldSet.readString("status")));
                return p;
            })
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- NOT thread-safe — don't use in multi-threaded step directly *(thread-safe nahi hai — multi-threaded step mein mat use karo)*
- Column names in `.names()` must match field names in target class for auto-mapping
- `linesToSkip(1)` skips first N lines — use `skippedLinesCallback` to capture the skipped header
- File encoding mismatch causes garbled characters — always set encoding explicitly
- Large files with millions of lines → fine, reads one line at a time (memory efficient)

### 🆚 vs. Comparison
| Aspect | FlatFileItemReader | JdbcPagingItemReader |
|--------|-------------------|---------------------|
| Source | Text files (CSV, TSV) | Database tables |
| Thread-safe | No | Yes |
| Restartable | Yes (line count checkpoint) | Yes (page-based checkpoint) |
| Performance | I/O bound (disk speed) | DB query speed |
| Use case | File processing | Database migration |

### 🎯 Tricky Interview Qs

**Q: How does FlatFileItemReader handle restart?**
It stores the current line number in ExecutionContext. On restart, it skips ahead to that line number. This is why `name()` is required — it acts as the key for storing state.

**Q: Can FlatFileItemReader read multi-line records?**
Not directly. Each `read()` call returns one line. For multi-line records, use a custom `RecordSeparatorPolicy` or aggregate lines in a custom reader/processor.

### ⚡ Remember
- Reads one line at a time → memory efficient
- Two tokenizers: Delimited (CSV) and FixedLength *(CSV ke liye delimited, fixed-width ke liye fixedLength)*
- NOT thread-safe
- `linesToSkip(1)` for header
- `name()` is required for restart support (checkpoint key)

### 🔗 Follow-ups
- [Q33 → JdbcCursorItemReader for DB](#q33)
- [Q38 → MultiResourceItemReader for multiple files](#q38)
- [Q40 → Skip header/footer lines](#q40)

---

## Q33. What is JdbcCursorItemReader?

### 📝 One-Liner
JdbcCursorItemReader opens a single database cursor and reads rows one at a time by moving the cursor forward — simple but holds the connection for the entire step.

### 🔑 Quick Answer
JdbcCursorItemReader executes ONE SQL query that opens a database cursor. Each `read()` call moves the cursor forward and returns the next row mapped to a Java object via `RowMapper`. The connection stays open for the ENTIRE step duration — could be minutes or hours. It's simple and memory-efficient (one row at a time) but NOT thread-safe and risks connection timeout on large datasets. Best for simple single-threaded steps with moderate data (< 1M rows). *(Ek SQL query, ek cursor — ek ek row padho, lekin connection poore step tak open rehta hai)*

### 📖 How It Works
```
JdbcCursorItemReader Lifecycle:

Step START:
  └── Open DB Connection ──────────────────────────┐
      └── Execute SQL (returns ResultSet/Cursor)    │
                                                    │ Connection HELD
Chunk 1: read() read() read() ... (500 times)      │ for ENTIRE
Chunk 2: read() read() read() ... (500 times)      │ step duration
Chunk 3: read() read() read() ... (500 times)      │ (could be hours!)
...                                                 │
Step END:                                           │
  └── Close Cursor + Close Connection ─────────────┘

Each read():
  cursor.next() → rowMapper.mapRow() → return Java object
  returns null when cursor reaches end
```

### 🗣️ Answering Approach
"JdbcCursorItemReader executes a single SQL query and opens a database cursor. Each read call moves the cursor forward and maps the row to a Java object using a RowMapper. The key characteristic is that it holds the database connection for the entire step duration, which makes it simple and memory-efficient but unsuitable for long-running steps or multi-threaded processing. In my project, we used it for small daily batch runs processing under 100K records in a single-threaded step, where the simplicity was worth the connection hold time. For our larger jobs, we switched to JdbcPagingItemReader."

### 💻 Code
```java
@Bean
public JdbcCursorItemReader<Order> cursorReader(DataSource dataSource) {
    return new JdbcCursorItemReaderBuilder<Order>()
            .name("orderCursorReader")
            .dataSource(dataSource)
            .sql("SELECT id, customer_name, amount, status FROM orders " +
                 "WHERE status = 'PENDING' ORDER BY id")
            .rowMapper((rs, rowNum) -> {
                Order order = new Order();
                order.setId(rs.getLong("id"));
                order.setCustomerName(rs.getString("customer_name"));
                order.setAmount(rs.getBigDecimal("amount"));
                order.setStatus(rs.getString("status"));
                return order;
            })
            .fetchSize(500)         // hint to JDBC driver: fetch 500 rows at a time
            .maxRows(1_000_000)     // optional: safety limit
            .queryTimeout(3600)     // 1 hour timeout
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Holds DB connection for ENTIRE step — can be hours for large data *(poora step chalte tak connection pakda rehta hai)*
- NOT thread-safe — don't use in multi-threaded steps
- Connection timeout risk — set `queryTimeout` and increase DB `wait_timeout`
- `fetchSize` is a hint to driver (MySQL often ignores it unless `useCursorFetch=true`)
- MySQL: add `?useCursorFetch=true` to JDBC URL for real cursor behavior

### 🆚 vs. Comparison
| Aspect | JdbcCursorItemReader | JdbcPagingItemReader |
|--------|---------------------|---------------------|
| SQL Queries | 1 (single cursor) | N (one per page) |
| Connection | Held entire step | Released between pages |
| Thread-safe | ❌ No | ✅ Yes |
| Memory | Very low (1 row) | Low (page-size rows) |
| Sort key required | No (ORDER BY optional) | Yes (mandatory) |
| Best for | < 1M rows, single thread | Any size, production |

### 🎯 Tricky Interview Qs

**Q: Why might JdbcCursorItemReader timeout on large datasets?**
Because it holds one DB connection for the entire step. If the step takes 2 hours but DB `wait_timeout` is 30 minutes, the connection drops mid-step and the job fails.

**Q: Is fetchSize the same as pageSize?**
No. `fetchSize` is a JDBC driver hint for how many rows to buffer locally from the cursor. It's still one query, one cursor. `pageSize` in PagingReader triggers separate SQL queries.

### ⚡ Remember
- One SQL, one cursor, one connection (held entire step) *(ek query, ek connection, poora step tak)*
- NOT thread-safe → single-threaded steps only
- Memory efficient (one row at a time)
- Risk: connection timeout on large data
- Best for < 1M rows, simple single-threaded steps

### 🔗 Follow-ups
- [Q34 → JdbcPagingItemReader (production alternative)](#q34)
- [Q36 → Cursor vs Paging comparison](#q36)
- [Q31 → All reader types](#q31)

---

## Q34. What is JdbcPagingItemReader?

### 📝 One-Liner
JdbcPagingItemReader fetches data page by page with separate SQL queries, releasing the DB connection between pages — thread-safe and production-recommended.

### 🔑 Quick Answer
JdbcPagingItemReader executes a new SQL query for each page (with LIMIT/OFFSET or equivalent). It reads one page of rows into memory, serves them one at a time via `read()`, and fetches the next page when current page is exhausted. Connection is released between pages, making it thread-safe and safe for long-running steps. A **sort key is mandatory** to ensure consistent ordering across pages. *(Har page ke liye alag SQL query — connection chhod deta hai beech mein, thread-safe hai)*

### 📖 How It Works
```
JdbcPagingItemReader Lifecycle:

Page 1: SQL "...ORDER BY id LIMIT 500 OFFSET 0"     → 500 rows in memory
  └── read() × 500 (serve from memory)
  └── release connection ✅

Page 2: SQL "...ORDER BY id LIMIT 500 OFFSET 500"    → 500 rows in memory
  └── read() × 500 (serve from memory)
  └── release connection ✅

Page 3: SQL "...WHERE id > 1000 LIMIT 500"           → 500 rows in memory
  └── read() × 500 (serve from memory)
  └── release connection ✅

...
Last page returns < pageSize rows → signals end of data
```

Spring Batch auto-generates pagination SQL based on your database:
- MySQL: `LIMIT ? OFFSET ?`
- Oracle: `ROWNUM`
- SQL Server: `OFFSET FETCH`
- PostgreSQL: `LIMIT ? OFFSET ?`

### 🗣️ Answering Approach
"JdbcPagingItemReader fetches data page by page using separate SQL queries for each page. The key advantage over cursor reader is that it releases the database connection between pages, making it thread-safe and suitable for long-running jobs. A sort key is mandatory to ensure consistent row ordering across pages. In my project, we used JdbcPagingItemReader as our standard for all production batch jobs. We set page size equal to chunk size at 500, used the primary key as sort key, and ran multi-threaded steps with a thread pool of 4 for our high-volume payment processing job."

### 💻 Code
```java
@Bean
public JdbcPagingItemReader<Order> pagingReader(DataSource dataSource) {
    Map<String, Order> sortKeys = new HashMap<>();
    sortKeys.put("id", Order.ASCENDING);  // sort key is MANDATORY

    return new JdbcPagingItemReaderBuilder<Order>()
            .name("orderPagingReader")
            .dataSource(dataSource)
            .selectClause("SELECT id, customer_name, amount, status")
            .fromClause("FROM orders")
            .whereClause("WHERE status = 'PENDING'")
            .sortKeys(sortKeys)           // MUST have sort key
            .pageSize(500)                // 500 rows per SQL query
            .rowMapper(new BeanPropertyRowMapper<>(Order.class))
            .build();
}

// With parameterized query
@Bean
public JdbcPagingItemReader<Order> paramReader(DataSource dataSource) {
    Map<String, Object> params = new HashMap<>();
    params.put("status", "PENDING");
    params.put("minAmount", 1000);

    return new JdbcPagingItemReaderBuilder<Order>()
            .name("paramOrderReader")
            .dataSource(dataSource)
            .selectClause("SELECT *")
            .fromClause("FROM orders")
            .whereClause("WHERE status = :status AND amount >= :minAmount")
            .sortKeys(Map.of("id", Order.ASCENDING))
            .parameterValues(params)
            .pageSize(500)
            .rowMapper(new BeanPropertyRowMapper<>(Order.class))
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- **Sort key is MANDATORY** — without it, pages may have duplicates or missing rows *(sort key nahi diya toh rows chhoot sakti hain ya double aa sakti hain)*
- Sort key should be UNIQUE (use primary key) — non-unique key + OFFSET can skip rows
- `pageSize` is NOT chunk size — page size = rows per SQL query, chunk size = items per transaction
- If data changes between pages (inserts/deletes), OFFSET-based paging can miss or duplicate rows → use keyset paging (sort by unique key + WHERE id > lastId)
- Set page size ≤ chunk size for predictable behavior

### 🆚 vs. Comparison
| Aspect | JdbcPagingItemReader | JpaPagingItemReader |
|--------|---------------------|---------------------|
| Query Language | Raw SQL | JPQL |
| Performance | ⚡ Faster (no ORM overhead) | Slower (ORM overhead) |
| Entity Mapping | Manual (RowMapper) | Automatic (JPA) |
| Thread-safe | ✅ Yes | ✅ Yes |
| Best for | Performance-critical batch | JPA-centric projects |

### 🎯 Tricky Interview Qs

**Q: Why is sort key mandatory?**
Without consistent ordering, page 1 might return rows A,B,C and page 2 might return B,C,D — causing duplicates and missing rows. Sort key ensures deterministic ordering.

**Q: Page size 1000 with chunk size 500 — what happens?**
Reader fetches 1000 rows but only 500 are consumed per chunk. The remaining 500 stay buffered in the reader and are served in the next chunk without another SQL query.

### ⚡ Remember
- Page by page = release connection between pages → thread-safe *(page per page = connection chhod deta hai)*
- **Sort key MANDATORY** (use primary key)
- Page size ≤ chunk size (best practice)
- Auto-generates DB-specific pagination SQL
- Production standard for database reading

### 🔗 Follow-ups
- [Q33 → JdbcCursorItemReader comparison](#q33)
- [Q36 → Cursor vs Paging detailed comparison](#q36)
- [Q23 → Page size vs chunk size](#q23)

---

## Q35. What is JpaPagingItemReader?

### 📝 One-Liner
JpaPagingItemReader is like JdbcPagingItemReader but uses JPQL queries and returns JPA entities instead of raw JDBC rows.

### 🔑 Quick Answer
JpaPagingItemReader fetches data page by page using JPQL (JPA Query Language) instead of raw SQL. It returns fully-mapped JPA entities with automatic relationship handling. It's thread-safe and restartable like JdbcPagingItemReader, but slower due to ORM overhead (entity lifecycle, dirty checking, first-level cache). Use it when your project already uses JPA entities and you want consistent domain mapping. *(JPA entities chahiye toh ye use karo, lekin JDBC paging se slow hai ORM overhead ki wajah se)*

### 📖 How It Works
```
JpaPagingItemReader vs JdbcPagingItemReader:

JdbcPagingItemReader:
  SQL → ResultSet → RowMapper → POJO (manual mapping, fast)

JpaPagingItemReader:
  JPQL → EntityManager.createQuery() → Entity (auto mapping, slow)
       → First-level cache → Dirty checking → Proxy objects
       → ORM overhead (10-30% slower)
```

### 🗣️ Answering Approach
"JpaPagingItemReader works similarly to JdbcPagingItemReader but uses JPQL queries and returns JPA entities. The advantage is automatic entity mapping, relationship handling, and consistency with the rest of the JPA-based application. The downside is performance — it's 10-30% slower than JdbcPagingItemReader due to ORM overhead like dirty checking and first-level cache management. In my project, we used JpaPagingItemReader for steps that needed complex entity relationships like Order with OrderItems, and JdbcPagingItemReader for high-throughput steps where we only needed flat data."

### 💻 Code
```java
@Bean
public JpaPagingItemReader<Employee> jpaReader(EntityManagerFactory emf) {
    return new JpaPagingItemReaderBuilder<Employee>()
            .name("empJpaReader")
            .entityManagerFactory(emf)
            .queryString("SELECT e FROM Employee e WHERE e.status = :status ORDER BY e.id")
            .parameterValues(Map.of("status", "ACTIVE"))
            .pageSize(500)
            .build();
}

// With named query
@Bean
public JpaPagingItemReader<Order> namedQueryReader(EntityManagerFactory emf) {
    return new JpaPagingItemReaderBuilder<Order>()
            .name("orderJpaReader")
            .entityManagerFactory(emf)
            .queryString("SELECT o FROM Order o JOIN FETCH o.items WHERE o.status = 'PENDING' ORDER BY o.id")
            .pageSize(100)   // smaller with JOIN FETCH (more data per entity)
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Slower than JdbcPagingItemReader — don't use for high-throughput if you don't need JPA features *(performance chahiye toh JDBC paging use karo)*
- First-level cache accumulates entities → clear in ChunkListener to avoid OOM
- `JOIN FETCH` with paging can cause cartesian product issues → use smaller page size
- Must use `JpaTransactionManager` (not DataSourceTransactionManager)
- JPQL `ORDER BY` is required for consistent paging

### 🎯 Tricky Interview Qs

**Q: When would you choose JpaPagingItemReader over JdbcPagingItemReader?**
When the project uses JPA entities extensively and you need automatic entity mapping, relationship handling, or when the step's writer is also JPA-based (JpaItemWriter). If performance is the priority and you just need flat data, use JdbcPagingItemReader.

### ⚡ Remember
- JPQL + JPA entities (auto mapping, relationships)
- Slower than JDBC paging (ORM overhead) *(JDBC se slow, par JPA entities mil jaate hain)*
- Must use JpaTransactionManager
- Clear EntityManager cache in ChunkListener
- Use for JPA projects; use JDBC paging for performance

### 🔗 Follow-ups
- [Q34 → JdbcPagingItemReader (faster alternative)](#q34)
- [Q43 → JpaItemWriter (pair with JPA reader)](#q43)
- [Q27 → Memory issues with JPA cache](#q27)

---

## Q36. What is the difference between Cursor and Paging reader?

### 📝 One-Liner
Cursor reader holds one connection with a scrolling cursor (simple, not thread-safe); Paging reader makes separate SQL queries per page (thread-safe, production-ready).

### 🔑 Quick Answer
**Cursor reader**: executes 1 SQL query, opens 1 cursor, holds 1 DB connection for the ENTIRE step. Reads one row at a time by moving cursor forward. Simple but not thread-safe and risky for large data (connection timeout). **Paging reader**: executes N SQL queries (one per page), releases connection between pages. Thread-safe, no timeout risk, but requires a sort key. For production, always prefer paging reader. *(Cursor = ek connection pura step, Paging = har page pe naya query — production mein paging use karo)*

### 📖 How It Works
```
CURSOR READER:
┌─────────────────────────────────────────────────┐
│ Connection OPEN ─────────────────────────────── │
│  SQL#1 → Cursor                                 │
│  read() → next row    ┐                        │
│  read() → next row    │ All from same cursor    │
│  read() → next row    │ Same connection         │
│  ...                   │ NOT thread-safe         │
│  read() → null (EOF)  ┘                        │
│ Connection CLOSE ────────────────────────────── │
└─────────────────────────────────────────────────┘

PAGING READER:
┌─────────────────────────────────┐
│ Page 1: SQL "...LIMIT 500 OFFSET 0"     │ Connect → Query → Read → Release
│ Page 2: SQL "...LIMIT 500 OFFSET 500"   │ Connect → Query → Read → Release  
│ Page 3: SQL "...LIMIT 500 OFFSET 1000"  │ Connect → Query → Read → Release
│ ...separate connection per page          │ Thread-safe ✅
└─────────────────────────────────┘
```

### 🗣️ Answering Approach
"The key difference is connection management. Cursor reader opens one database connection and one cursor for the entire step — it streams rows one at a time, which is memory efficient but holds the connection for hours on large datasets and is not thread-safe. Paging reader makes a separate SQL query for each page, releasing the connection between pages, making it thread-safe and safe from connection timeouts. In my project, we standardized on JdbcPagingItemReader for all production jobs because we needed multi-threaded steps and couldn't risk connection timeouts. We only used cursor reader in dev for quick testing with small datasets."

### 💻 Code
```java
// CURSOR — simple, holds connection
@Bean
public JdbcCursorItemReader<Order> cursorReader() {
    return new JdbcCursorItemReaderBuilder<Order>()
            .name("cursorReader")
            .dataSource(dataSource)
            .sql("SELECT * FROM orders WHERE status = 'PENDING' ORDER BY id")
            .rowMapper(new BeanPropertyRowMapper<>(Order.class))
            .build();
    // ⚠️ Connection held for ENTIRE step, NOT thread-safe
}

// PAGING — production recommended
@Bean
public JdbcPagingItemReader<Order> pagingReader() {
    return new JdbcPagingItemReaderBuilder<Order>()
            .name("pagingReader")
            .dataSource(dataSource)
            .selectClause("SELECT *")
            .fromClause("FROM orders")
            .whereClause("WHERE status = 'PENDING'")
            .sortKeys(Map.of("id", Order.ASCENDING))  // MANDATORY
            .pageSize(500)
            .rowMapper(new BeanPropertyRowMapper<>(Order.class))
            .build();
    // ✅ Connection released between pages, thread-safe
}
```

### ⚠️ Pitfalls / Gotchas
- Cursor on MySQL needs `?useCursorFetch=true` in JDBC URL for real cursor behavior *(MySQL mein cursor ke liye special URL parameter chahiye)*
- Paging with non-unique sort key → OFFSET can skip/duplicate rows
- Cursor `fetchSize` ≠ paging `pageSize` — fetchSize is a driver hint, pageSize triggers new queries
- If data is modified during step (INSERT/DELETE), paging with OFFSET can have inconsistencies

### 🆚 vs. Comparison
| Aspect | Cursor Reader | Paging Reader |
|--------|--------------|---------------|
| SQL Queries | 1 (single cursor) | N (one per page) |
| DB Connection | Held entire step | Released between pages |
| Thread-safe | ❌ No | ✅ Yes |
| Sort key | Optional | **Mandatory** |
| Memory | Very low (1 row) | Low (1 page) |
| Timeout risk | ⚠️ High (long connection) | ✅ Low |
| Multi-threaded | ❌ Cannot | ✅ Can |
| Best for | Dev/test, < 100K rows | Production, any size |
| Complexity | Simple to configure | Needs sort keys, clauses |

### 🎯 Tricky Interview Qs

**Q: Can you make cursor reader thread-safe?**
Technically yes with `SynchronizedItemStreamReader` wrapper — but it serializes reads, defeating the purpose of multi-threading. Just use paging reader.

**Q: Is paging reader slower than cursor because of multiple queries?**
The overhead of N queries vs 1 is negligible. The paging reader's advantages (thread-safety, no timeout) far outweigh the minor query overhead.

### ⚡ Remember
- **Cursor**: 1 query, 1 connection (held entire step), NOT thread-safe
- **Paging**: N queries, release connection, thread-safe ✅ *(paging = safe, thread-safe, production ke liye)*
- Paging needs sort key (use primary key)
- Production → always paging reader
- Cursor → only for small datasets, single-threaded, dev/test

### 🔗 Follow-ups
- [Q33 → JdbcCursorItemReader details](#q33)
- [Q34 → JdbcPagingItemReader details](#q34)
- [Q84 → Multi-threaded step with paging reader](#q84)

---

## Q37. What is StaxEventItemReader?

### 📝 One-Liner
StaxEventItemReader reads XML files by streaming XML events (SAX-like) and mapping XML fragments to Java objects using JAXB or XStream.

### 🔑 Quick Answer
StaxEventItemReader reads XML files using StAX (Streaming API for XML). It doesn't load the entire file into memory — it streams XML events and extracts fragments matching a configured root element name. Each fragment is unmarshalled to a Java object using JAXB, XStream, or a custom Unmarshaller. Memory efficient for large XML files. *(Poori XML file memory mein nahi laadta — stream karta hai aur ek ek fragment padh ke Java object banata hai)*

### 📖 How It Works
```
XML File:
<orders>
  <order>                    ← fragment root = "order"
    <id>1</id>
    <amount>500</amount>
  </order>                   ← StaxEventItemReader extracts this fragment
  <order>                    ← and unmarshals to Order.class
    <id>2</id>
    <amount>600</amount>
  </order>
</orders>

Flow: XML stream → StAX parser → fragment extraction → JAXB unmarshal → Order object
```

### 🗣️ Answering Approach
"StaxEventItemReader reads XML files using the StAX streaming API. Instead of loading the entire XML into memory like DOM, it streams events and extracts fragments matching a configured root element name. Each fragment is unmarshalled to a Java object using JAXB. In my project, we received daily transaction reports as XML files from a partner system, and we used StaxEventItemReader with JAXB to read and process them. It handled 500MB XML files without memory issues because it streams rather than loads."

### 💻 Code
```java
@Bean
public StaxEventItemReader<Order> xmlReader() {
    Jaxb2Marshaller marshaller = new Jaxb2Marshaller();
    marshaller.setClassesToBeBound(Order.class);

    return new StaxEventItemReaderBuilder<Order>()
            .name("orderXmlReader")
            .resource(new FileSystemResource("/data/orders.xml"))
            .addFragmentRootElements("order")  // extract <order>...</order>
            .unmarshaller(marshaller)           // JAXB unmarshalling
            .build();
}

// JAXB-annotated class
@XmlRootElement(name = "order")
@XmlAccessorType(XmlAccessType.FIELD)
public class Order {
    private Long id;
    private BigDecimal amount;
    private String status;
    // getters, setters
}
```

### ⚠️ Pitfalls / Gotchas
- Fragment root element name must match exactly (case-sensitive)
- JAXB annotations must match XML structure precisely
- NOT thread-safe (like FlatFileItemReader)
- Namespace-aware: if XML has namespaces, configure them on the marshaller

### ⚡ Remember
- StAX = streaming (memory efficient for large XML) *(badi XML ke liye best — stream karta hai)*
- Configure fragment root element + JAXB unmarshaller
- NOT thread-safe
- Counterpart writer: StaxEventItemWriter

### 🔗 Follow-ups
- [Q31 → All reader types](#q31)
- [Q32 → FlatFileItemReader for CSV](#q32)
- [Q44 → FlatFileItemWriter (writing files)](#q44)

---

## Q38. What is MultiResourceItemReader?

### 📝 One-Liner
MultiResourceItemReader reads from multiple files sequentially by wrapping a delegate reader and switching to the next file when the current one is exhausted.

### 🔑 Quick Answer
MultiResourceItemReader wraps another reader (like FlatFileItemReader) and feeds it multiple files one after another. When the delegate reader returns null (file exhausted), it automatically opens the next file. Useful when you receive daily files like `orders_01.csv`, `orders_02.csv`, etc. and need to process them all in one step. *(Ek reader ke andar multiple files — ek file khatam toh agla file automatically open hota hai)*

### 📖 How It Works
```
MultiResourceItemReader:
┌──────────────────────────────────────────────┐
│ Resources: [file1.csv, file2.csv, file3.csv] │
│                                              │
│ Delegate: FlatFileItemReader                 │
│                                              │
│ file1.csv → read, read, read... → null (EOF) │
│ file2.csv → read, read, read... → null (EOF) │
│ file3.csv → read, read, read... → null (EOF) │
│ → return null (all files done)               │
└──────────────────────────────────────────────┘
```

### 🗣️ Answering Approach
"MultiResourceItemReader wraps a delegate reader and processes multiple files sequentially. When one file is exhausted, it automatically switches to the next. In my project, we received hourly transaction files and used MultiResourceItemReader to process all files in a directory in one batch step. We combined it with a file pattern to pick up only files matching our naming convention, and used a ResourceSuffixCreator to track which file each record came from for error reporting."

### 💻 Code
```java
@Bean
public MultiResourceItemReader<Order> multiFileReader() {
    Resource[] resources = new PathMatchingResourcePatternResolver()
            .getResources("file:/data/input/orders_*.csv");  // glob pattern

    MultiResourceItemReader<Order> reader = new MultiResourceItemReader<>();
    reader.setName("multiFileReader");
    reader.setResources(resources);
    reader.setDelegate(singleFileReader());  // reused for each file
    return reader;
}

@Bean
public FlatFileItemReader<Order> singleFileReader() {
    return new FlatFileItemReaderBuilder<Order>()
            .name("singleFileReader")
            .delimited().delimiter(",")
            .names("id", "amount", "date")
            .targetType(Order.class)
            .linesToSkip(1)
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- File order depends on OS directory listing — sort resources explicitly if order matters *(file ka order OS pe depend karta hai — sort karo agar order zaroori hai)*
- Restart saves which file and line — but adding/removing files between restart can cause issues
- Delegate reader must be a new instance or properly resettable
- NOT thread-safe

### ⚡ Remember
- Wraps delegate reader + multiple resources
- Auto-switches to next file when current exhausts
- Sort resources if order matters *(order chahiye toh sort karo)*
- Restart-aware (tracks file + line position)

### 🔗 Follow-ups
- [Q32 → FlatFileItemReader as delegate](#q32)
- [Q37 → StaxEventItemReader for XML files](#q37)
- [Q40 → Skip header/footer in files](#q40)

---

## Q39. How do you handle encoding issues in file readers?

### 📝 One-Liner
Set the encoding explicitly on FlatFileItemReader using `.encoding("UTF-8")` — never rely on system default.

### 🔑 Quick Answer
Encoding issues cause garbled characters (mojibake) when the file encoding doesn't match what the reader expects. Always set encoding explicitly: `.encoding("UTF-8")` for FlatFileItemReader. Common issues: Windows files in `CP1252` or `ISO-8859-1` read as UTF-8 show garbled special characters. For BOM (Byte Order Mark) in UTF-8 files, use `UnicodeBOMInputStream` wrapper or a skip callback. *(Encoding explicitly set karo — system default pe bharosa mat karo)*

### 📖 How It Works
```
Encoding Pipeline:

File on disk (bytes) → Reader decodes with charset → String → Java Object
                         ↑
                         Must match actual file encoding!

Common encodings:
├── UTF-8        → Most common, supports all characters
├── ISO-8859-1   → Western European (legacy)
├── CP1252       → Windows default (legacy)
├── UTF-16       → Some enterprise systems
└── Shift_JIS    → Japanese systems
```

### 🗣️ Answering Approach
"Encoding issues occur when the reader's charset doesn't match the file's actual encoding. I always set the encoding explicitly — never rely on system default. In my project, we received files from a legacy Windows system in CP1252 encoding. Without explicit encoding config, UTF-8 was assumed and special characters were garbled. We fixed it by setting `.encoding(\"CP1252\")` on the FlatFileItemReader. We also had UTF-8 files with BOM that caused parse errors on the first record, which we handled by adding a linesToSkip callback to strip the BOM."

### 💻 Code
```java
@Bean
public FlatFileItemReader<Employee> encodedReader() {
    return new FlatFileItemReaderBuilder<Employee>()
            .name("encodedReader")
            .resource(new FileSystemResource("/data/employees.csv"))
            .encoding("UTF-8")     // ALWAYS set explicitly
            .delimited().delimiter(",")
            .names("id", "name", "department")
            .targetType(Employee.class)
            .linesToSkip(1)
            .build();
}

// For legacy Windows files
@Bean
public FlatFileItemReader<Employee> windowsFileReader() {
    return new FlatFileItemReaderBuilder<Employee>()
            .name("windowsReader")
            .resource(new FileSystemResource("/data/legacy_export.csv"))
            .encoding("CP1252")    // Windows encoding
            .delimited().delimiter(";")  // European CSVs often use semicolon
            .names("id", "name", "salary")
            .targetType(Employee.class)
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- System default encoding varies by OS (Windows: CP1252, Linux: UTF-8) *(Windows aur Linux ka default encoding alag hai)*
- JVM `-Dfile.encoding=UTF-8` is NOT reliable — always set on reader
- BOM in UTF-8 files can cause first field to be unreadable
- Binary files (Excel, PDF) can't be read with FlatFileItemReader

### ⚡ Remember
- ALWAYS set `.encoding()` explicitly
- Never rely on system/JVM default *(system default pe depend mat karo)*
- UTF-8 is default/standard; set CP1252 for legacy Windows files
- BOM can corrupt first record — handle explicitly

### 🔗 Follow-ups
- [Q32 → FlatFileItemReader configuration](#q32)
- [Q40 → Header/footer handling](#q40)

---

## Q40. How do you skip header and footer lines in file readers?

### 📝 One-Liner
Use `linesToSkip(N)` for headers and a custom `RecordSeparatorPolicy` or post-processing for footers.

### 🔑 Quick Answer
For **headers**: use `linesToSkip(1)` (or N for multi-line headers). You can capture the skipped header with `skippedLinesCallback()` to validate column names. For **footers**: there's no built-in support. Options: (1) filter footer in processor (return null), (2) custom `RecordSeparatorPolicy`, or (3) pre-process the file. *(Header ke liye linesToSkip, footer ke liye processor mein filter karo)*

### 📖 How It Works
```
CSV File:
Line 1: id,name,amount       ← linesToSkip(1) skips this
Line 2: 1,John,500           ← first record read
Line 3: 2,Jane,600           ← second record read
...
Line N: TRAILER|RECORDS:998  ← footer — filter in processor

With skippedLinesCallback:
  linesToSkip(1) → skippedLinesCallback(line -> validate(line))
  → Can validate: "id,name,amount" matches expected columns
```

### 🗣️ Answering Approach
"For headers, I use linesToSkip to skip the header rows. I also use skippedLinesCallback to capture and validate the header — ensuring column names match the expected format. For footers, there's no direct support, so in my project I handled it in the processor — when the line matched the trailer pattern, I returned null to filter it out. For complex files with multi-line headers and footers, we pre-processed the file in a tasklet step that stripped headers and footers before the main chunk step."

### 💻 Code
```java
@Bean
public FlatFileItemReader<Order> readerWithHeaderFooter() {
    return new FlatFileItemReaderBuilder<Order>()
            .name("orderReader")
            .resource(new FileSystemResource("/data/orders.csv"))
            .linesToSkip(2)   // skip 2-line header
            .skippedLinesCallback(line -> {
                // Validate header columns
                if (line.startsWith("id,") || line.startsWith("ORDER_FILE")) {
                    log.info("Skipped header: {}", line);
                }
            })
            .delimited().delimiter(",")
            .names("id", "amount", "status")
            .targetType(Order.class)
            .build();
}

// Handle footer in processor
@Bean
public ItemProcessor<Order, Order> footerFilter() {
    return item -> {
        // Footer lines typically have a marker
        if (item.getId() == null || "TRAILER".equals(item.getStatus())) {
            return null;  // filter out footer → tracked as filterCount
        }
        return item;
    };
}
```

### ⚠️ Pitfalls / Gotchas
- `linesToSkip` counts from the beginning — can't skip lines in the middle *(linesToSkip sirf shuru se skip karta hai)*
- Footer filtering in processor counts as `filterCount`, not `skipCount`
- `skippedLinesCallback` gets the raw line string, not parsed fields
- If header count is wrong (skip too many), you'll miss data silently

### ⚡ Remember
- Header: `linesToSkip(N)` + `skippedLinesCallback()` for validation
- Footer: filter in processor (return null) *(footer ke liye processor mein null return karo)*
- No built-in footer support
- Validate headers with callback to catch file format changes early

### 🔗 Follow-ups
- [Q32 → FlatFileItemReader details](#q32)
- [Q54 → Filtering with processor](#q54)
- [Q38 → MultiResourceItemReader](#q38)
