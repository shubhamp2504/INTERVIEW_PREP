# 🟡 Spring Batch — ItemReader Deep Dive (Q31–Q40)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q31"></a>

## Q31. What types of ItemReader implementations are available?

### 🔑 Quick Answer

> Spring provides **15+ built-in readers** — for flat files (CSV/TXT), databases (JDBC cursor/paging, JPA, Hibernate), JSON/XML files, messaging (Kafka, JMS), and NoSQL (MongoDB). You rarely need to write a custom one.

### 📖 Step-by-Step Explanation

**Step 1 — Readers organized by data source:**

| Category | Reader | Reads From |
|----------|--------|-----------|
| **File** | `FlatFileItemReader` | CSV, TXT, fixed-width files |
| **File** | `JsonItemReader` | JSON files |
| **File** | `StaxEventItemReader` | XML files |
| **File** | `MultiResourceItemReader` | Multiple files (glob pattern) |
| **Database** | `JdbcCursorItemReader` | SQL via open cursor |
| **Database** | `JdbcPagingItemReader` | SQL via page queries |
| **Database** | `JpaPagingItemReader` | JPQL via JPA pagination |
| **Database** | `HibernateCursorItemReader` | HQL via Hibernate cursor |
| **Database** | `StoredProcedureItemReader` | DB stored procedures |
| **Database** | `RepositoryItemReader` | Spring Data Repository |
| **Messaging** | `KafkaItemReader` | Kafka topics |
| **Messaging** | `JmsItemReader` | JMS queues |
| **Messaging** | `AmqpItemReader` | RabbitMQ queues |
| **NoSQL** | `MongoItemReader` | MongoDB collections |

**Step 2 — How to choose the right reader:**

```
Reading from FILES?
  ├── CSV/TXT → FlatFileItemReader
  ├── JSON → JsonItemReader
  ├── XML → StaxEventItemReader
  └── Multiple files → MultiResourceItemReader

Reading from DATABASE?
  ├── Simple query, sequential → JdbcCursorItemReader
  ├── Large data, need thread safety → JdbcPagingItemReader ⭐
  ├── Using JPA entities → JpaPagingItemReader
  └── Stored procedure → StoredProcedureItemReader

Reading from MESSAGE QUEUE?
  ├── Kafka → KafkaItemReader
  └── RabbitMQ → AmqpItemReader
```

### 🗣️ How to Explain in Interview

> *"Spring Batch provides built-in readers for almost every data source. For files — FlatFileItemReader for CSV, JsonItemReader for JSON, StaxEventItemReader for XML. For databases — JdbcCursorItemReader for simple sequential reads and JdbcPagingItemReader for thread-safe, large-dataset reads. For messaging — KafkaItemReader and AmqpItemReader. The most important decision is choosing between cursor and paging readers for database sources, because it impacts thread safety, restartability, and performance."*

### ⚡ Key Points to Remember

1. **FlatFileItemReader** = most common for file processing
2. **JdbcPagingItemReader** = safest choice for databases (thread-safe)
3. **MultiResourceItemReader** = when you have multiple input files
4. Rarely need custom readers — built-in ones cover 95% of cases

---

<a id="q32"></a>

## Q32. What is FlatFileItemReader? How does it work?

### 🔑 Quick Answer

> FlatFileItemReader reads **text files line by line** (CSV, TSV, fixed-width). It uses a `LineTokenizer` to split each line into fields, and a `FieldSetMapper` to map those fields to a Java object.

### 📖 Step-by-Step Explanation

**Step 1 — How it processes a CSV file:**

```
employees.csv:
  id,name,salary,department       ← Header (can be skipped)
  1,Amit,50000,IT                 ← Line 1
  2,Priya,60000,HR                ← Line 2
  3,Rahul,55000,IT                ← Line 3

Processing flow:
  Line "1,Amit,50000,IT"
    │
    ├── LineTokenizer splits by comma → ["1", "Amit", "50000", "IT"]
    │
    └── FieldSetMapper converts → Employee{id=1, name="Amit", salary=50000, dept="IT"}
```

**Step 2 — Two types of file formats:**

| Format | Example | Tokenizer |
|--------|---------|-----------|
| **Delimited** (CSV) | `1,Amit,50000,IT` | `DelimitedLineTokenizer` (default: comma) |
| **Fixed Width** | `001Amit    50000IT  ` | `FixedLengthTokenizer` (column ranges) |

**Step 3 — Key features:**

- **Skip header lines**: `linesToSkip(1)` — ignores the first N lines
- **Line filtering**: Use `LineCallbackHandler` to skip comment lines
- **Encoding**: Supports UTF-8, ISO-8859-1, etc.
- **Restartable**: Saves current line number in ExecutionContext

### 🗣️ How to Explain in Interview

> *"FlatFileItemReader reads text files line by line. For a CSV file, it splits each line by the delimiter — comma by default — into fields, then maps those fields to a Java object using either a BeanWrapperFieldSetMapper for automatic mapping or a custom mapper. You configure the column names to match your bean properties. It also supports fixed-width formats where you specify column ranges. It handles header skipping, encoding, and is restartable — it saves its current line position in the ExecutionContext so if the job restarts, it picks up from the right line."*

### 💻 Code Example

```java
// CSV file reader
@Bean
public FlatFileItemReader<Employee> csvReader() {
    return new FlatFileItemReaderBuilder<Employee>()
            .name("employeeCsvReader")
            .resource(new ClassPathResource("employees.csv"))
            .linesToSkip(1)                        // Skip header row
            .delimited()                           // CSV format
            .delimiter(",")                        // Split by comma (default)
            .names("id", "name", "salary", "department")  // Column names
            .targetType(Employee.class)            // Map to Employee bean
            .build();
}

// Fixed-width file reader
@Bean
public FlatFileItemReader<Employee> fixedWidthReader() {
    return new FlatFileItemReaderBuilder<Employee>()
            .name("employeeFixedReader")
            .resource(new ClassPathResource("employees.dat"))
            .fixedLength()
            .columns(new Range(1, 3), new Range(4, 13),  // id: 1-3, name: 4-13
                     new Range(14, 19), new Range(20, 21)) // salary: 14-19, dept: 20-21
            .names("id", "name", "salary", "department")
            .targetType(Employee.class)
            .build();
}
```

### ⚡ Key Points to Remember

1. Reads **line by line** — memory efficient (one line at a time)
2. Supports **delimited** (CSV) and **fixed-width** formats
3. `linesToSkip(1)` → skip header row
4. **Restartable** — saves line position in ExecutionContext
5. **NOT thread-safe** — don't use with multi-threaded steps

---

<a id="q33"></a>

## Q33. What is JdbcCursorItemReader?

### 🔑 Quick Answer

> JdbcCursorItemReader opens a **single database cursor** and reads rows **one at a time** by moving the cursor forward. It holds the database connection for the **entire step duration**.

### 📖 Step-by-Step Explanation

**Step 1 — How a cursor works:**

```
Database table: employees (10,000 rows)

JdbcCursorItemReader:
  1. Opens connection
  2. Executes: SELECT * FROM employees WHERE active = true
  3. Gets ResultSet (cursor pointing before first row)
  
  read() call 1: cursor.next() → row 1 → maps to Employee{id=1}
  read() call 2: cursor.next() → row 2 → maps to Employee{id=2}
  read() call 3: cursor.next() → row 3 → maps to Employee{id=3}
  ...
  read() call 10000: cursor.next() → row 10000 → maps to Employee{id=10000}
  read() call 10001: cursor.next() → false → returns null (end of data)
  
  4. Closes ResultSet, Statement, Connection
```

**Step 2 — Key characteristics:**

| Property | Value |
|----------|-------|
| Connection held | Entire step duration (could be hours!) |
| Memory | Very low (one row at a time) |
| Thread safe | ❌ NO — not safe for multi-threaded steps |
| Restartable | ✅ Yes (saves row count in ExecutionContext) |
| Speed | Fast for sequential reads |
| Risk | Connection timeout for large datasets |

**Step 3 — When to use and when NOT to:**

```
✅ USE when:
  - Small to medium data (< 1M rows)
  - Single-threaded step
  - Simple sequential read
  - Connection pool timeout is long enough

❌ DON'T USE when:
  - Multi-threaded step (not thread-safe)
  - Very large dataset (connection held too long)
  - Short connection timeout
  - Need to partition data
```

### 🗣️ How to Explain in Interview

> *"JdbcCursorItemReader executes a SQL query and opens a database cursor. It reads one row at a time by moving the cursor forward — very memory efficient. The main thing to know is that it holds the database connection for the entire step. So if you have 10 million records and each takes 1ms to process, that's 10,000 seconds with one connection held open. This is fine for smaller datasets, but for large ones, I prefer JdbcPagingItemReader. Also, it's not thread-safe, so it can't be used with multi-threaded steps."*

### 💻 Code Example

```java
@Bean
public JdbcCursorItemReader<Employee> cursorReader(DataSource dataSource) {
    return new JdbcCursorItemReaderBuilder<Employee>()
            .name("employeeCursorReader")
            .dataSource(dataSource)
            .sql("SELECT id, name, salary, department FROM employees WHERE active = true")
            .rowMapper((rs, rowNum) -> {
                Employee emp = new Employee();
                emp.setId(rs.getLong("id"));
                emp.setName(rs.getString("name"));
                emp.setSalary(rs.getBigDecimal("salary"));
                emp.setDepartment(rs.getString("department"));
                return emp;
            })
            .build();
}
```

**What happens step by step:**
1. Step starts → reader opens connection, executes SQL
2. Each `read()` call → `ResultSet.next()` → maps row to Employee
3. When `ResultSet.next()` returns false → `read()` returns null
4. Step ends → connection released

### ⚡ Key Points to Remember

1. **One cursor**, **one connection** held for entire step
2. **Memory efficient** — one row at a time
3. **NOT thread-safe** — single-threaded only
4. **Risk**: connection timeout on large datasets
5. Best for **< 1M records** in single-threaded steps

---

<a id="q34"></a>

## Q34. What is JdbcPagingItemReader?

### 🔑 Quick Answer

> JdbcPagingItemReader reads data **page by page** — each page is a separate SQL query with LIMIT/OFFSET. It doesn't hold a connection between pages, making it **thread-safe and suitable for large datasets**.

### 📖 Step-by-Step Explanation

**Step 1 — How paging works:**

```
Database: employees (100,000 rows), page size = 1000

Page 1: SELECT * FROM employees ORDER BY id LIMIT 1000 OFFSET 0
  → Returns rows 1-1000 → Connection released ✅

Page 2: SELECT * FROM employees ORDER BY id LIMIT 1000 OFFSET 1000
  → Returns rows 1001-2000 → Connection released ✅

Page 3: SELECT * FROM employees ORDER BY id LIMIT 1000 OFFSET 2000
  → Returns rows 2001-3000 → Connection released ✅

... continues for 100 pages
```

**Step 2 — Why it's better than cursor for large data:**

| Feature | Cursor Reader | Paging Reader |
|---------|--------------|---------------|
| Connection held | Entire step ❌ | Per page only ✅ |
| Thread-safe | ❌ No | ✅ Yes |
| Memory | 1 row at a time | 1 page in memory |
| Requires sort key | ❌ No | ✅ Yes (must have!) |
| Speed | Slightly faster | Slightly slower (many queries) |
| Partitioning | ❌ Can't partition | ✅ Can partition |

**Step 3 — IMPORTANT: Sort key is mandatory:**

```
❌ WITHOUT sort key — DISASTER:
  Page 1: SELECT * FROM emp LIMIT 1000 OFFSET 0    → returns rows 1-1000
  -- Someone inserts a new row --
  Page 2: SELECT * FROM emp LIMIT 1000 OFFSET 1000  → skips or duplicates rows!

✅ WITH sort key:
  Page 1: SELECT * FROM emp WHERE id > 0 ORDER BY id LIMIT 1000
  Page 2: SELECT * FROM emp WHERE id > 1000 ORDER BY id LIMIT 1000
  → Consistent results regardless of inserts!
```

### 🗣️ How to Explain in Interview

> *"JdbcPagingItemReader fetches data page by page using separate SQL queries for each page. Unlike the cursor reader that holds a connection for the entire step, the paging reader releases the connection after each page, which is much better for large datasets. It's also thread-safe, so you can use it with multi-threaded steps. The catch is you must specify a sort key — without it, pagination is unreliable because rows can shift between pages if the data changes. I always use the primary key as the sort key."*

### 💻 Code Example

```java
@Bean
public JdbcPagingItemReader<Employee> pagingReader(DataSource dataSource) {
    
    Map<String, Order> sortKeys = new HashMap<>();
    sortKeys.put("id", Order.ASCENDING);  // Sort key is MANDATORY
    
    return new JdbcPagingItemReaderBuilder<Employee>()
            .name("employeePagingReader")
            .dataSource(dataSource)
            .selectClause("SELECT id, name, salary, department")
            .fromClause("FROM employees")
            .whereClause("WHERE active = true")
            .sortKeys(sortKeys)                    // Must have sort keys!
            .pageSize(1000)                        // Rows per page (per query)
            .rowMapper((rs, rowNum) -> {
                Employee emp = new Employee();
                emp.setId(rs.getLong("id"));
                emp.setName(rs.getString("name"));
                emp.setSalary(rs.getBigDecimal("salary"));
                emp.setDepartment(rs.getString("department"));
                return emp;
            })
            .build();
}
```

### ⚡ Key Points to Remember

1. **Page by page** — separate SQL per page
2. **Thread-safe** — works with multi-threaded steps
3. **Sort key mandatory** — use primary key
4. **No long-held connections** — releases after each page
5. **Recommended for production** — especially > 100K records

---

<a id="q35"></a>

## Q35. What is JpaPagingItemReader?

### 🔑 Quick Answer

> Same as JdbcPagingItemReader but uses **JPA (JPQL)** instead of raw SQL. It works with JPA entities and supports parameterized JPQL queries with pagination.

### 📖 Step-by-Step Explanation

**Step 1 — When to use JPA vs JDBC reader:**

| Scenario | Use |
|----------|-----|
| Already using JPA in project, want entity mapping | `JpaPagingItemReader` |
| Need maximum performance | `JdbcPagingItemReader` (faster, no ORM overhead) |
| Complex queries with joins | `JdbcPagingItemReader` (more control) |
| Simple entity reads | `JpaPagingItemReader` (cleaner code) |

**Step 2 — Key difference: JPA adds overhead but convenience:**

```
JdbcPagingItemReader:
  SQL → ResultSet → Manual RowMapper → Object
  Speed: ⭐⭐⭐⭐⭐

JpaPagingItemReader:
  JPQL → EntityManager → Automatic mapping → Entity
  Speed: ⭐⭐⭐ (ORM overhead: lazy loading, cache, proxies)
```

### 🗣️ How to Explain in Interview

> *"JpaPagingItemReader is similar to the JDBC paging reader but works with JPA entities and JPQL queries. It's convenient when your project already uses JPA because you get automatic entity mapping. But it's slower than JdbcPagingItemReader due to ORM overhead — EntityManager, first-level cache, proxy objects. For batch processing with millions of records, I usually prefer JdbcPagingItemReader for performance. But for smaller datasets where the entity relationships are needed, JPA reader is more productive."*

### 💻 Code Example

```java
@Bean
public JpaPagingItemReader<Employee> jpaReader(EntityManagerFactory emf) {
    return new JpaPagingItemReaderBuilder<Employee>()
            .name("employeeJpaReader")
            .entityManagerFactory(emf)
            .queryString("SELECT e FROM Employee e WHERE e.active = true ORDER BY e.id")
            .pageSize(500)
            .build();
}

// With parameters
@Bean
@StepScope
public JpaPagingItemReader<Employee> jpaReaderWithParams(
        EntityManagerFactory emf,
        @Value("#{jobParameters['department']}") String dept) {
    
    Map<String, Object> params = new HashMap<>();
    params.put("dept", dept);
    
    return new JpaPagingItemReaderBuilder<Employee>()
            .name("deptEmployeeReader")
            .entityManagerFactory(emf)
            .queryString("SELECT e FROM Employee e WHERE e.department = :dept ORDER BY e.id")
            .parameterValues(params)
            .pageSize(500)
            .build();
}
```

### ⚡ Key Points to Remember

1. Uses **JPQL** (not raw SQL)
2. Works with **JPA entities** (automatic mapping)
3. **Slower** than JdbcPagingItemReader (ORM overhead)
4. **Thread-safe** (like all paging readers)
5. For batch performance → prefer `JdbcPagingItemReader`

---

<a id="q36"></a>

## Q36. What is the difference between Cursor and Paging reader?

### 🔑 Quick Answer

> **Cursor** opens one connection, holds it for the entire step, reads row by row. **Paging** makes separate queries per page, releases connection between pages. Paging is **thread-safe and production-recommended**; Cursor is simpler but limited.

### 📖 Step-by-Step Explanation

**Step 1 — Side-by-side comparison:**

| Feature | Cursor Reader | Paging Reader |
|---------|--------------|---------------|
| **SQL execution** | 1 query, holds ResultSet | N queries (one per page) |
| **Connection** | Held entire step ❌ | Released per page ✅ |
| **Thread-safe** | ❌ No | ✅ Yes |
| **Restartable** | ✅ Yes | ✅ Yes |
| **Sort key needed** | ❌ No | ✅ Yes (mandatory) |
| **Best for** | < 1M rows, single-thread | Any size, multi-thread |
| **Connection pool impact** | 1 connection blocked for hours | Normal pool usage |
| **Risk** | Connection timeout | Slightly more DB queries |

**Step 2 — Visualize the difference:**

```
CURSOR READER:
  ┌─ Connection OPEN ────────────────────────────────────┐
  │ read() → row1                                         │
  │ read() → row2                                         │
  │ read() → row3                                         │
  │ ... (10,000 reads, same connection, same ResultSet)   │
  │ read() → null                                         │
  └─ Connection CLOSED ──────────────────────────────────┘
  
  Total: 1 SQL query, 1 connection held for full step duration

PAGING READER:
  [Connection OPEN] → Page 1 query → get 1000 rows → [Connection CLOSED]
  read() → row1, row2, ..., row1000 (from memory)
  
  [Connection OPEN] → Page 2 query → get 1000 rows → [Connection CLOSED]
  read() → row1001, row1002, ..., row2000 (from memory)
  
  Total: 10 SQL queries, 10 short connections
```

**Step 3 — Decision guide:**

```
Small data (< 100K), single-thread, simple?  → Cursor is fine
Large data (> 100K)?                          → Paging ⭐
Need multi-threading?                         → Paging (only option)
Need partitioning?                            → Paging (only option)
Short connection timeout (e.g., cloud DB)?    → Paging (must)
Team prefers simplicity?                      → Cursor (no sort key needed)
```

### 🗣️ How to Explain in Interview

> *"The key difference is connection management. The cursor reader opens one database connection, executes the query, and holds the ResultSet open for the entire step — which could be hours for large datasets. That's risky because the connection pool has one less connection, and you risk timeouts. The paging reader makes a separate query for each page, gets 1000 rows, releases the connection, processes them, then fetches the next page. This is much safer for production. Paging is also thread-safe — multiple threads can fetch different pages simultaneously — while the cursor reader is not. The tradeoff is that you must provide a sort key for paging, and it makes slightly more database queries."*

### ⚡ Key Points to Remember

1. **Cursor** = 1 connection, entire step; **Paging** = 1 connection per page
2. **Thread safety**: Cursor ❌, Paging ✅
3. **Production recommendation: always use Paging** for large data
4. Paging needs **sort key** (primary key works best)
5. Cursor is fine for **small data, simple jobs**

---

<a id="q37"></a>

## Q37. Which reader is best for large datasets?

### 🔑 Quick Answer

> **JdbcPagingItemReader + Partitioning** for databases. **FlatFileItemReader + MultiResource Partitioning** for files. As data grows, add multi-threading and partitioning.

### 📖 Step-by-Step Explanation

**Step 1 — Recommendations by data size:**

| Data Size | Recommended Reader | Strategy |
|-----------|-------------------|----------|
| < 100K | `JdbcCursorItemReader` | Simple single-threaded |
| 100K – 1M | `JdbcPagingItemReader` | Single-threaded with good chunk size |
| 1M – 10M | `JdbcPagingItemReader` + multi-threaded step | TaskExecutor with N threads |
| 10M – 100M | `JdbcPagingItemReader` + **Partitioning** | Split by ID range, parallel execution |
| 100M+ | `JdbcPagingItemReader` + **Distributed Partitioning** | Multiple machines |
| Large files | `FlatFileItemReader` | Streams line-by-line (memory efficient) |
| Multiple files | `MultiResourceItemReader` + Partitioning | One file per partition |

**Step 2 — Why partitioning is the answer for massive data:**

```
100 million records, 20 partitions:

Partition 1:  WHERE id BETWEEN 1 AND 5,000,000
Partition 2:  WHERE id BETWEEN 5,000,001 AND 10,000,000
Partition 3:  WHERE id BETWEEN 10,000,001 AND 15,000,000
...
Partition 20: WHERE id BETWEEN 95,000,001 AND 100,000,000

Each partition: Own reader, own thread, own connection
Total time: ~1/20th of single-threaded time
```

### 🗣️ How to Explain in Interview

> *"For data under a million rows, a simple JdbcPagingItemReader with a good chunk size works fine. Beyond that, I add multi-threading — a TaskExecutor with maybe 8 threads processing chunks in parallel. But for tens of millions, the real strategy is partitioning — you divide the data into ranges based on ID, and each partition runs in its own thread with its own reader. For 100 million records split into 20 partitions, each partition handles 5 million, and they all run in parallel. For files, FlatFileItemReader is inherently memory-efficient since it streams line by line."*

### ⚡ Key Points to Remember

1. **< 1M**: Single-threaded Paging reader
2. **1M-10M**: Multi-threaded step
3. **10M+**: **Partitioning** (split by ID range)
4. **Files**: FlatFileItemReader streams (always memory efficient)
5. **Always use Paging** for databases (not Cursor)

---

<a id="q38"></a>

## Q38. How does JdbcCursorItemReader work internally?

### 🔑 Quick Answer

> It opens a JDBC connection, creates a PreparedStatement, executes the SQL to get a ResultSet, and on each `read()` call it moves the cursor forward with `ResultSet.next()`, mapping each row via a RowMapper.

### 📖 Step-by-Step Explanation

**Step 1 — Internal lifecycle:**

```
OPEN phase (called once when step starts):
  1. connection = dataSource.getConnection()
  2. preparedStatement = connection.prepareStatement(sql)
  3. resultSet = preparedStatement.executeQuery()
  
READ phase (called once per item):
  4. if resultSet.next() == true:
       return rowMapper.mapRow(resultSet, rowNum++)
     else:
       return null  // No more data

CLOSE phase (called once when step ends):
  5. resultSet.close()
  6. preparedStatement.close()
  7. connection.close()
```

**Step 2 — Key implementation details:**

| Detail | Value |
|--------|-------|
| Fetch size | Controls how many rows JDBC driver buffers (default varies by driver) |
| Verify cursor position | Throws exception if cursor moved unexpectedly |
| Save state | Saves `read.count` in ExecutionContext for restart |
| Driver support | Some drivers load entire result into memory (MySQL default!) |

**Step 3 — MySQL gotcha:**

```java
// MySQL loads ENTIRE result set into memory by default!
// Fix: set fetchSize = Integer.MIN_VALUE for streaming
@Bean
public JdbcCursorItemReader<Employee> mysqlReader(DataSource ds) {
    JdbcCursorItemReader<Employee> reader = new JdbcCursorItemReader<>();
    reader.setDataSource(ds);
    reader.setSql("SELECT * FROM employees");
    reader.setFetchSize(Integer.MIN_VALUE);  // MySQL streaming mode!
    reader.setRowMapper(new EmployeeRowMapper());
    return reader;
}
```

### 🗣️ How to Explain in Interview

> *"Internally, when the step opens, the cursor reader gets a connection from the pool, creates a PreparedStatement, and executes the SQL to get a ResultSet. On each read() call, it calls ResultSet.next() to move the cursor forward and uses the RowMapper to convert the row to a Java object. When next() returns false, it returns null to signal end of data. On step close, it closes the ResultSet, Statement, and Connection. One important gotcha — with MySQL, the default JDBC behavior loads the entire result set into memory. You need to set fetchSize to Integer.MIN_VALUE to enable streaming mode."*

### ⚡ Key Points to Remember

1. `open()` → get connection + execute SQL
2. `read()` → `ResultSet.next()` + RowMapper
3. `close()` → close all JDBC resources
4. **MySQL gotcha**: set `fetchSize=Integer.MIN_VALUE` for streaming
5. Saves **read count** in ExecutionContext on each chunk commit

---

<a id="q39"></a>

## Q39. What are the limitations of cursor readers?

### 🔑 Quick Answer

> Not thread-safe, holds connection for entire step, risk of connection timeout, all rows must fit in driver buffer (or driver streams), not suitable for partitioning, and cannot be used across multiple machines.

### 📖 Step-by-Step Explanation

| Limitation | Impact | Workaround |
|-----------|--------|------------|
| **Not thread-safe** | Can't use with multi-threaded step | Use `SynchronizedItemStreamReader` wrapper or switch to Paging |
| **Connection held entire step** | Pool starved, timeout risk | Increase timeout or use Paging |
| **MySQL loads all to memory** | OutOfMemoryError | Set fetchSize = MIN_VALUE |
| **Not partitionable** | Can't split across threads | Use Paging reader |
| **Long-running** | Connection idle detection kills it | Increase server timeout |
| **Driver-dependent** | Different behavior per DB | Test with your specific driver |

### 🗣️ How to Explain in Interview

> *"The main limitations are: first, it's not thread-safe — you can't use it in a multi-threaded step without wrapping it. Second, it holds the connection for the entire step, which can starve the connection pool and risk timeouts. Third, some drivers like MySQL load the entire result set into memory unless you explicitly enable streaming. Fourth, you can't use it with partitioning because each partition needs its own reader, and the cursor reader wasn't designed for that. For production with large datasets, I almost always switch to JdbcPagingItemReader to avoid these limitations."*

### ⚡ Key Points to Remember

1. **Not thread-safe** — biggest limitation
2. **Connection held entire step** — timeout risk
3. **Cannot partition** — no parallel processing
4. **Driver-dependent** — test with your specific DB
5. **Production rule**: use **Paging** for anything > 100K rows

---

<a id="q40"></a>

## Q40. How do you read from multiple files?

### 🔑 Quick Answer

> Use **MultiResourceItemReader** — it wraps a delegate reader (like FlatFileItemReader) and iterates over multiple files matching a pattern. Each file is processed completely before moving to the next.

### 📖 Step-by-Step Explanation

**Step 1 — The setup:**

```
Input directory:
  /data/uploads/
    ├── orders_2024_01.csv
    ├── orders_2024_02.csv
    ├── orders_2024_03.csv
    └── orders_2024_04.csv

MultiResourceItemReader:
  → Files to process: /data/uploads/orders_*.csv
  → Delegate: FlatFileItemReader (handles one file at a time)
  
  Processing order:
    1. Open orders_2024_01.csv → read all lines → delegate returns null → close
    2. Open orders_2024_02.csv → read all lines → delegate returns null → close
    3. Open orders_2024_03.csv → read all lines → delegate returns null → close
    4. Open orders_2024_04.csv → read all lines → delegate returns null → close
    5. MultiResourceItemReader returns null → step done
```

**Step 2 — Restartability:**

The reader saves **which file** and **which line** it was at. On restart, it opens the right file and skips to the right line.

### 🗣️ How to Explain in Interview

> *"MultiResourceItemReader wraps a delegate reader and processes multiple files one after another. You provide file resources matching a pattern — like all CSV files in a directory — and a delegate FlatFileItemReader that knows how to read one file. The multi-resource reader opens the first file, delegates reading until that file is done, then opens the next file. It's restartable — it tracks both the current file index and the delegate's position within that file. In production, I combine this with partitioning so each file is processed by a different thread."*

### 💻 Code Example

```java
@Bean
public MultiResourceItemReader<Order> multiFileReader(
        @Value("file:/data/uploads/orders_*.csv") Resource[] resources) {
    
    MultiResourceItemReader<Order> reader = new MultiResourceItemReader<>();
    reader.setResources(resources);       // All matching files
    reader.setDelegate(singleFileReader()); // How to read each file
    reader.setComparator(Comparator.comparing(Resource::getFilename)); // Process order
    return reader;
}

@Bean
public FlatFileItemReader<Order> singleFileReader() {
    return new FlatFileItemReaderBuilder<Order>()
            .name("orderFileReader")
            .delimited()
            .names("orderId", "product", "amount", "date")
            .targetType(Order.class)
            .build();
    // Note: DO NOT set resource here — MultiResourceItemReader injects it!
}
```

**What happens:**
1. Spring resolves `orders_*.csv` → 4 files found
2. Sorted by filename (comparator)
3. First file injected into delegate → reads all orders
4. Delegate returns null → next file injected → reads all orders
5. All files done → MultiResourceItemReader returns null → step ends

### ⚡ Key Points to Remember

1. **MultiResourceItemReader** wraps a **delegate** reader
2. Processes files **one after another** (or partition for parallel!)
3. **Restartable** — saves file index + line position
4. Don't set `resource` on the delegate — it's injected automatically
5. Combine with **partitioning** for parallel file processing

---

> **🎯 Navigation:** [← Chunk Processing (Q21-30)](02-chunk-processing.md) | [Next → Writers (Q41-48)](04-writers.md) | [📋 All Sections](README.md)
