
# 🟡 Spring Batch — Writers Questions (41-48)

[![Questions](https://img.shields.io/badge/Questions-8-blue.svg)](#)
[![Difficulty](https://img.shields.io/badge/Level-Medium-yellow.svg)](#)


---

<a id="q1"></a>

## Q41. ❓ What types of ItemWriter implementations exist?

🔖 **Tags:** `#spring-batch` `#writer` `#must-know`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

| Writer | Destination | Use Case |
|--------|------------|----------|
| `JdbcBatchItemWriter` | RDBMS via JDBC | High performance DB writes |
| `JpaItemWriter` | RDBMS via JPA | When using JPA entities |
| `HibernateItemWriter` | RDBMS via Hibernate | Hibernate apps |
| `FlatFileItemWriter` | CSV/TXT files | File output |
| `JsonFileItemWriter` | JSON files | JSON output |
| `StaxEventItemWriter` | XML files | XML output |
| `KafkaItemWriter` | Kafka topics | Event publishing |
| `MongoItemWriter` | MongoDB | NoSQL writes |
| `CompositeItemWriter` | Multiple destinations | Write to DB + file |
| `ClassifierCompositeItemWriter` | Conditional routing | Different writers per item type |
| `JmsItemWriter` | JMS queues | Message publishing |

---

<a id="q2"></a>

## Q42. ❓ What is JdbcBatchItemWriter?

🔖 **Tags:** `#spring-batch` `#writer` `#jdbc` `#must-know`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

`JdbcBatchItemWriter` writes data to a database using **JDBC batch operations** — the most performant DB writer.

```java
@Bean
public JdbcBatchItemWriter<Employee> writer(DataSource ds) {
    return new JdbcBatchItemWriterBuilder<Employee>()
            .dataSource(ds)
            .sql("INSERT INTO employees (name, salary, dept) VALUES (:name, :salary, :department)")
            .beanMapped()           // maps bean properties to :name parameters
            .build();
}

// Or using positional parameters:
@Bean
public JdbcBatchItemWriter<Employee> writer(DataSource ds) {
    return new JdbcBatchItemWriterBuilder<Employee>()
            .dataSource(ds)
            .sql("INSERT INTO employees (name, salary, dept) VALUES (?, ?, ?)")
            .itemPreparedStatementSetter((item, ps) -> {
                ps.setString(1, item.getName());
                ps.setDouble(2, item.getSalary());
                ps.setString(3, item.getDepartment());
            })
            .build();
}
```

### 🎯 How Batch Insert Works:
```
Chunk of 100 items arrives at writer:
  1. PreparedStatement created
  2. For each item → addBatch()     ← adds to batch buffer
  3. executeBatch()                  ← sends ALL 100 in single round-trip
  4. Database executes 100 INSERTs in one go

Result: 100 INSERTs in 1 DB round-trip instead of 100!
```

### Performance Tip:
> 💡 `JdbcBatchItemWriter` uses JDBC batch API internally. For max performance, set `spring.jpa.properties.hibernate.jdbc.batch_size=100` matching your chunk size.

---

<a id="q3"></a>

## Q43. ❓ What is JpaItemWriter?

🔖 **Tags:** `#spring-batch` `#writer` `#jpa`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

`JpaItemWriter` persists items using JPA's `EntityManager.merge()`.

```java
@Bean
public JpaItemWriter<Employee> jpaWriter(EntityManagerFactory emf) {
    JpaItemWriter<Employee> writer = new JpaItemWriter<>();
    writer.setEntityManagerFactory(emf);
    return writer;
}
```

### JPA vs JDBC Writer:

| Feature | JpaItemWriter | JdbcBatchItemWriter |
|---------|-------------|-------------------|
| **Performance** | Slower (JPA overhead) | Faster (direct JDBC) |
| **Mapping** | Automatic (entity annotations) | Manual (SQL + mapper) |
| **Batch insert** | Depends on Hibernate config | Native JDBC batch |
| **Cascade** | Supports JPA cascades | Manual |
| **Cache** | Uses L1/L2 cache | No ORM cache |
| **Use when** | Already using JPA | Performance critical |

---

<a id="q4"></a>

## Q44. ❓ What is FlatFileItemWriter?

🔖 **Tags:** `#spring-batch` `#writer` `#file`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

```java
@Bean
public FlatFileItemWriter<Employee> csvWriter() {
    return new FlatFileItemWriterBuilder<Employee>()
            .name("employeeCsvWriter")
            .resource(new FileSystemResource("output/employees-report.csv"))
            .headerCallback(writer -> writer.write("ID,Name,Salary,Department"))
            .delimited()
            .delimiter(",")
            .names("id", "name", "salary", "department")
            .footerCallback(writer -> writer.write("--- End of Report ---"))
            .append(false)          // overwrite file (true = append)
            .build();
}
```

Output:
```csv
ID,Name,Salary,Department
1,John,55000,Engineering
2,Jane,66000,Marketing
--- End of Report ---
```

---

<a id="q5"></a>

## Q45. ❓ What is CompositeItemWriter?

🔖 **Tags:** `#spring-batch` `#writer` `#composite` `#must-know`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

`CompositeItemWriter` delegates writing to **multiple writers** — write the same chunk to multiple destinations.

```java
@Bean
public CompositeItemWriter<Employee> compositeWriter() {
    CompositeItemWriter<Employee> writer = new CompositeItemWriter<>();
    writer.setDelegates(List.of(
        dbWriter(),      // Write to database
        csvWriter(),     // Also write to CSV file
        kafkaWriter()    // Also publish to Kafka
    ));
    return writer;
}
```

```
Chunk [emp1, emp2, emp3]
  │
  ├──→ JdbcBatchItemWriter → Database ✅
  ├──→ FlatFileItemWriter  → CSV file ✅
  └──→ KafkaItemWriter     → Kafka topic ✅
```

### ClassifierCompositeItemWriter (Conditional Routing):

```java
@Bean
public ClassifierCompositeItemWriter<Employee> classifiedWriter() {
    ClassifierCompositeItemWriter<Employee> writer = new ClassifierCompositeItemWriter<>();
    writer.setClassifier(employee -> {
        if ("Engineering".equals(employee.getDept())) {
            return engineeringWriter();    // Different file/table
        } else {
            return generalWriter();        // Different destination
        }
    });
    return writer;
}
```

---

<a id="q6"></a>

## Q46. ❓ How does batch insert work in JdbcBatchItemWriter?

🔖 **Tags:** `#spring-batch` `#batch-insert` `#performance`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

```
Without Batch Insert (100 items):
  INSERT INTO emp VALUES (1, 'John')   → 1 DB round trip
  INSERT INTO emp VALUES (2, 'Jane')   → 1 DB round trip
  INSERT INTO emp VALUES (3, 'Bob')    → 1 DB round trip
  ...
  Total: 100 DB round trips ❌ SLOW

With JDBC Batch Insert (100 items):
  addBatch(INSERT VALUES (1, 'John'))  → added to buffer
  addBatch(INSERT VALUES (2, 'Jane'))  → added to buffer
  addBatch(INSERT VALUES (3, 'Bob'))   → added to buffer
  ...
  executeBatch()                        → 1 DB round trip
  Total: 1 DB round trip ✅ FAST
```

### Internal Flow:
```java
// What JdbcBatchItemWriter does internally
PreparedStatement ps = connection.prepareStatement(sql);

for (Employee emp : chunk.getItems()) {
    ps.setString(1, emp.getName());
    ps.setDouble(2, emp.getSalary());
    ps.addBatch();                      // Add to batch buffer
}

ps.executeBatch();                      // Execute ALL at once
```

---

<a id="q7"></a>

## Q47. ❓ How do you write data to multiple destinations?

🔖 **Tags:** `#spring-batch` `#writer` `#multi-destination`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

**3 approaches:**

| Approach | Use Case |
|----------|----------|
| `CompositeItemWriter` | Same data to multiple destinations |
| `ClassifierCompositeItemWriter` | Different data to different destinations |
| Multiple Steps | Independent processing per destination |

Covered in Q45 above with code examples.

---

<a id="q8"></a>

## Q48. ❓ What happens if writing fails during chunk processing?

🔖 **Tags:** `#spring-batch` `#writer` `#error-handling`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

```
Chunk [item1, item2, item3, item4, item5]
  │
  └──→ writer.write(chunk) → EXCEPTION on item3! ❌
       │
       └──→ ROLLBACK entire chunk transaction
            ALL 5 items are NOT written

If skip configured:
  Spring Batch enters "scan mode":
  Write [item1] → ✅
  Write [item2] → ✅  
  Write [item3] → ❌ SKIP (logged)
  Write [item4] → ✅
  Write [item5] → ✅
  
  Result: 4 items written, 1 skipped
```

### 📌 Key Takeaway
> 💡 Writer failure = chunk rollback. With skip, it retries one-by-one to find the bad item. Without skip, job fails.

---

[← Back to Spring Batch Index](./README.md) | [← Prev: Readers](./03-readers.md) | [Next: Processors →](./05-processors.md)
]]>
