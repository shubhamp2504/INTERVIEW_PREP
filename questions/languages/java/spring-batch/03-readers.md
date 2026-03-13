<![CDATA[<div align="center">

# 🟡 Spring Batch — Readers Questions (31-40)

[![Questions](https://img.shields.io/badge/Questions-10-blue.svg)](#)
[![Difficulty](https://img.shields.io/badge/Level-Medium-yellow.svg)](#)

</div>

---

<a id="q1"></a>
## Q31. ❓ What types of ItemReader implementations are available?

🔖 **Tags:** `#spring-batch` `#reader` `#must-know`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

| Reader | Source | Use Case |
|--------|--------|----------|
| `FlatFileItemReader` | CSV/TXT/Fixed-width files | File-based ETL |
| `JdbcCursorItemReader` | RDBMS (cursor) | Small-medium datasets |
| `JdbcPagingItemReader` | RDBMS (paging) | Large datasets |
| `JpaPagingItemReader` | RDBMS (JPA) | When using JPA entities |
| `HibernatePagingItemReader` | RDBMS (Hibernate) | Hibernate-based apps |
| `StoredProcedureItemReader` | DB stored procedures | Legacy DB operations |
| `JsonItemReader` | JSON files | JSON file processing |
| `StaxEventItemReader` | XML files | XML file processing |
| `MongoItemReader` | MongoDB | NoSQL data processing |
| `KafkaItemReader` | Kafka topics | Event stream processing |
| `JmsItemReader` | JMS queues | Message queue processing |
| `AmqpItemReader` | RabbitMQ | Message processing |
| `RepositoryItemReader` | Spring Data Repository | Spring Data apps |
| `MultiResourceItemReader` | Multiple files | Processing multiple files |
| `SynchronizedItemStreamReader` | Thread-safe wrapper | Multi-threaded steps |

---

<a id="q2"></a>
## Q32. ❓ What is FlatFileItemReader?

🔖 **Tags:** `#spring-batch` `#reader` `#file`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

`FlatFileItemReader` reads data from **flat files** (CSV, TXT, fixed-width) line by line.

### Key Components:

```
FlatFileItemReader
├── Resource          → Which file to read
├── LineMapper         → How to map a line to an object
│   ├── LineTokenizer  → How to split the line (delimited/fixed-width)
│   └── FieldSetMapper → How to convert fields to object properties
├── LinesToSkip        → Skip header lines
└── Encoding           → File encoding (UTF-8, etc.)
```

```java
@Bean
public FlatFileItemReader<Employee> csvReader() {
    return new FlatFileItemReaderBuilder<Employee>()
            .name("employeeCsvReader")
            .resource(new ClassPathResource("employees.csv"))
            .linesToSkip(1)                    // skip header row
            .delimited()                        // comma-delimited
            .delimiter(",")                     // default is comma
            .names("id", "name", "salary", "dept")  // column names
            .targetType(Employee.class)         // auto-map to POJO
            .build();
}
```

### CSV File Example:
```csv
id,name,salary,dept
1,John,50000,Engineering
2,Jane,60000,Marketing
3,Bob,55000,Engineering
```

### Fixed-Width Example:
```java
@Bean
public FlatFileItemReader<Employee> fixedWidthReader() {
    return new FlatFileItemReaderBuilder<Employee>()
            .name("fixedWidthReader")
            .resource(new ClassPathResource("employees.dat"))
            .fixedLength()
            .columns(new Range(1,5), new Range(6,25), new Range(26,35))
            .names("id", "name", "salary")
            .targetType(Employee.class)
            .build();
}
```

---

<a id="q3"></a>
## Q33. ❓ What is JdbcCursorItemReader?

🔖 **Tags:** `#spring-batch` `#reader` `#jdbc` `#cursor`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

`JdbcCursorItemReader` opens a **database cursor** and reads rows one at a time using JDBC `ResultSet`.

```
┌─────────────────────────────────┐
│        Database                  │
│  ┌───────────────────────┐      │
│  │    SELECT * FROM emp   │      │
│  │    ↓ cursor             │      │
│  │    row1 → read()       │      │
│  │    row2 → read()       │      │
│  │    row3 → read()       │      │
│  │    ...                  │      │
│  │    null → done          │      │
│  └───────────────────────┘      │
│  Connection stays OPEN          │
│  throughout entire Step!        │
└─────────────────────────────────┘
```

```java
@Bean
public JdbcCursorItemReader<Employee> cursorReader(DataSource ds) {
    return new JdbcCursorItemReaderBuilder<Employee>()
            .name("employeeCursorReader")
            .dataSource(ds)
            .sql("SELECT id, name, salary, dept FROM employees WHERE active = true")
            .rowMapper(new BeanPropertyRowMapper<>(Employee.class))
            .fetchSize(1000)        // JDBC fetch size hint
            .build();
}
```

### 🎯 Key Points:
- Opens **one connection**, holds it for entire step
- Very efficient for **sequential reading**
- NOT thread-safe by default
- Connection held for long time → beware of DB timeouts

---

<a id="q4"></a>
## Q34. ❓ What is JdbcPagingItemReader?

🔖 **Tags:** `#spring-batch` `#reader` `#jdbc` `#paging`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

`JdbcPagingItemReader` reads data using **SQL pagination** — fetches one page at a time with separate queries.

```
┌─────────────────────────────────┐
│        Database                  │
│                                  │
│  Page 1: SELECT ... LIMIT 500    │ → read 500 rows
│  Connection closed               │
│                                  │
│  Page 2: SELECT ... LIMIT 500    │ → read 500 rows  
│  OFFSET 500                      │
│  Connection closed               │
│                                  │
│  Page 3: SELECT ... LIMIT 500    │ → read 500 rows
│  Connection closed               │
│                                  │
└─────────────────────────────────┘
```

```java
@Bean
public JdbcPagingItemReader<Employee> pagingReader(DataSource ds) {
    Map<String, Order> sortKeys = new HashMap<>();
    sortKeys.put("id", Order.ASCENDING);  // REQUIRED: sorting key

    return new JdbcPagingItemReaderBuilder<Employee>()
            .name("employeePagingReader")
            .dataSource(ds)
            .selectClause("SELECT id, name, salary, dept")
            .fromClause("FROM employees")
            .whereClause("WHERE active = true")
            .sortKeys(sortKeys)             // MUST have sort keys
            .pageSize(500)                  // rows per page
            .rowMapper(new BeanPropertyRowMapper<>(Employee.class))
            .build();
}
```

### 🎯 Key Points:
- **Requires sort keys** (to maintain consistent ordering across pages)
- Each page = separate DB query + connection
- **Thread-safe** — suitable for multi-threaded steps
- Better for large datasets
- Supports **restartability** natively

---

<a id="q5"></a>
## Q35. ❓ What is JpaPagingItemReader?

🔖 **Tags:** `#spring-batch` `#reader` `#jpa`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

`JpaPagingItemReader` reads data using **JPA/JPQL queries** with pagination. Works with JPA entities.

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

// With parameters:
@Bean
public JpaPagingItemReader<Employee> jpaReaderWithParams(EntityManagerFactory emf) {
    Map<String, Object> params = new HashMap<>();
    params.put("dept", "Engineering");
    
    return new JpaPagingItemReaderBuilder<Employee>()
            .name("deptEmployeeReader")
            .entityManagerFactory(emf)
            .queryString("SELECT e FROM Employee e WHERE e.dept = :dept ORDER BY e.id")
            .parameterValues(params)
            .pageSize(500)
            .build();
}
```

### Comparison:

| Feature | JpaPagingItemReader | JdbcPagingItemReader |
|---------|-------------------|---------------------|
| Query language | JPQL | SQL |
| Entity mapping | Automatic (JPA) | Manual (RowMapper) |
| Performance | Slightly slower (JPA overhead) | Faster (direct JDBC) |
| Cache | Uses JPA cache | No ORM cache |
| Use when | Already using JPA | Performance critical |

---

<a id="q6"></a>
## Q36. ❓ What is the difference between Cursor and Paging reader?

🔖 **Tags:** `#spring-batch` `#cursor-vs-paging` `#must-know` `#frequently-asked`  
📊 **Difficulty:** 🟡 Medium  
🔥 **Frequency:** ⭐⭐⭐⭐⭐ (Top Interview Question)

### ✅ Answer

| Feature | Cursor Reader | Paging Reader |
|---------|--------------|---------------|
| **How it reads** | Opens cursor, reads row-by-row | Executes page queries (LIMIT/OFFSET) |
| **DB Connection** | Single connection, held entire step | New connection per page |
| **Thread-safe** | ❌ No | ✅ Yes |
| **Memory** | Buffered by JDBC fetch size | Loads one page at a time |
| **Restartability** | Via ExecutionContext | Native (page tracking) |
| **Connection timeout** | Risk of timeout (long-held) | No risk (short connections) |
| **Performance** | Faster for small-medium data | Better for large data |
| **Multi-threaded** | ❌ Not supported | ✅ Supported |
| **Requires sort keys** | No | Yes (mandatory) |

### When to Use:

```
Use CURSOR when:
├── Single-threaded step
├── Small to medium dataset (< 1M rows)
├── Simple sequential read
└── No connection timeout issues

Use PAGING when:
├── Multi-threaded step needed
├── Large dataset (millions of rows)
├── Long-running jobs (avoid timeout)
├── Need restartability
└── Horizontal scaling (partitioning)
```

### 📌 Key Takeaway
> 💡 **Paging = safe, scalable, thread-safe.** Use it for production. Cursor = simpler, faster for small jobs.

---

<a id="q7"></a>
## Q37. ❓ Which reader is best for large datasets?

🔖 **Tags:** `#spring-batch` `#reader` `#performance` `#large-data`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

| Dataset Size | Recommended Reader | Why |
|-------------|-------------------|-----|
| < 100K rows | `JdbcCursorItemReader` | Simple, fast |
| 100K - 10M | `JdbcPagingItemReader` | Thread-safe, scalable |
| 10M - 100M | `JdbcPagingItemReader` + **Partitioning** | Parallel processing |
| 100M+ | **Partitioning** + Multiple `JdbcPagingItemReader` | Distributed |
| Files (any size) | `FlatFileItemReader` | Streaming, low memory |
| Multiple files | `MultiResourceItemReader` + `FlatFileItemReader` | Each file = partition |

### For Very Large Datasets:
```java
// Partitioned approach: Split by ID range
@Bean
public Partitioner rangePartitioner(DataSource ds) {
    return gridSize -> {
        Map<String, ExecutionContext> partitions = new HashMap<>();
        long min = 1, max = 100_000_000;
        long range = (max - min) / gridSize + 1;
        
        for (int i = 0; i < gridSize; i++) {
            ExecutionContext ctx = new ExecutionContext();
            ctx.putLong("minId", min + (i * range));
            ctx.putLong("maxId", min + ((i + 1) * range) - 1);
            partitions.put("partition" + i, ctx);
        }
        return partitions;
    };
}
```

---

<a id="q8"></a>
## Q38. ❓ How does JdbcCursorItemReader work internally?

🔖 **Tags:** `#spring-batch` `#cursor` `#internals`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

```
1. Step starts
   │
2. open() called
   ├── Get DB connection from DataSource
   ├── Create PreparedStatement with SQL
   ├── Set fetchSize (JDBC hint for batching)
   ├── Execute query → ResultSet created
   └── Cursor positioned BEFORE first row
   
3. read() called (repeated)
   ├── ResultSet.next() → move cursor to next row
   ├── If row exists → RowMapper maps to object → return
   └── If no more rows → return null (signals end)

4. close() called
   ├── Close ResultSet
   ├── Close PreparedStatement
   └── Close/Release Connection
```

### Internal Pseudocode:
```java
// Simplified
class JdbcCursorItemReader {
    Connection connection;
    ResultSet resultSet;
    
    void open() {
        connection = dataSource.getConnection();
        PreparedStatement ps = connection.prepareStatement(sql,
            ResultSet.TYPE_FORWARD_ONLY,
            ResultSet.CONCUR_READ_ONLY);
        ps.setFetchSize(fetchSize);
        resultSet = ps.executeQuery();
    }
    
    T read() {
        if (resultSet.next()) {
            return rowMapper.mapRow(resultSet, currentRow++);
        }
        return null;  // no more data
    }
    
    void close() {
        resultSet.close();
        connection.close();
    }
}
```

---

<a id="q9"></a>
## Q39. ❓ What are the limitations of cursor readers?

🔖 **Tags:** `#spring-batch` `#cursor` `#limitations`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

| # | Limitation | Impact |
|---|-----------|--------|
| 1 | **Not thread-safe** | Cannot use in multi-threaded steps |
| 2 | **Long-held connection** | Risk of DB connection timeout |
| 3 | **Memory with large result sets** | JDBC driver may buffer rows |
| 4 | **No native pagination** | Relies on forward-only cursor |
| 5 | **Driver-dependent fetch behavior** | Different DBs handle fetchSize differently |
| 6 | **Not suitable for partitioning** | Hard to split cursor across threads |
| 7 | **Restart complexity** | Needs ExecutionContext to track position |

### Workarounds:

| Limitation | Solution |
|-----------|---------|
| Not thread-safe | Use `SynchronizedItemStreamReader` wrapper or switch to paging |
| Connection timeout | Increase timeout or switch to paging reader |
| Memory issues | Set appropriate `fetchSize` |
| Need partitioning | Switch to `JdbcPagingItemReader` |

```java
// Thread-safe wrapper (not recommended for high concurrency)
@Bean
public SynchronizedItemStreamReader<Employee> threadSafeReader() {
    SynchronizedItemStreamReader<Employee> reader = new SynchronizedItemStreamReader<>();
    reader.setDelegate(cursorReader());
    return reader;
}
```

---

<a id="q10"></a>
## Q40. ❓ How do you read data from multiple files?

🔖 **Tags:** `#spring-batch` `#reader` `#multi-file`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

Use `MultiResourceItemReader` — it wraps a delegate reader and iterates over multiple files.

```java
@Bean
public MultiResourceItemReader<Employee> multiFileReader() {
    MultiResourceItemReader<Employee> reader = new MultiResourceItemReader<>();
    
    reader.setResources(new Resource[]{
        new FileSystemResource("data/employees-jan.csv"),
        new FileSystemResource("data/employees-feb.csv"),
        new FileSystemResource("data/employees-mar.csv")
    });
    
    reader.setDelegate(singleFileReader());  // delegate for each file
    return reader;
}

// Or dynamically find files by pattern:
@Value("file:data/employees-*.csv")
private Resource[] inputFiles;

@Bean
public MultiResourceItemReader<Employee> multiFileReader() {
    MultiResourceItemReader<Employee> reader = new MultiResourceItemReader<>();
    reader.setResources(inputFiles);
    reader.setDelegate(singleFileReader());
    reader.setComparator(Comparator.comparing(Resource::getFilename));
    return reader;
}

@Bean
public FlatFileItemReader<Employee> singleFileReader() {
    return new FlatFileItemReaderBuilder<Employee>()
            .name("employeeReader")
            .delimited()
            .names("id", "name", "salary")
            .targetType(Employee.class)
            .build();
}
```

### 📌 Key Takeaway
> 💡 `MultiResourceItemReader` handles restartability too — it tracks which file and which line it was at!

---

[← Back to Spring Batch Index](./README.md) | [← Prev: Chunk Processing](./02-chunk-processing.md) | [Next: Writers →](./04-writers.md)
]]>