
# 🟡 Spring Batch — Processor Questions (49-54)

[![Questions](https://img.shields.io/badge/Questions-6-blue.svg)](#)
[![Difficulty](https://img.shields.io/badge/Level-Medium-yellow.svg)](#)


---

<a id="q1"></a>

## Q49. ❓ What is ItemProcessor used for?

🔖 **Tags:** `#spring-batch` `#processor` `#must-know`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

ItemProcessor sits between Reader and Writer for **3 main purposes**:

| Purpose | Example |
|---------|---------|
| **Transformation** | Convert currency, format names, map DTOs |
| **Validation** | Check required fields, validate business rules |
| **Filtering** | Return `null` to exclude items from writing |
| **Enrichment** | Call external API to add extra data |

```java
@Bean
public ItemProcessor<RawOrder, ProcessedOrder> orderProcessor() {
    return rawOrder -> {
        // 1. Validate
        if (rawOrder.getAmount() <= 0) {
            return null;  // Filter out invalid orders
        }
        
        // 2. Transform
        ProcessedOrder order = new ProcessedOrder();
        order.setOrderId(rawOrder.getId());
        order.setAmountUSD(rawOrder.getAmount() * exchangeRate);
        order.setCustomerName(rawOrder.getName().toUpperCase());
        
        // 3. Enrich
        order.setTaxAmount(calculateTax(order.getAmountUSD()));
        
        return order;
    };
}
```

---

<a id="q2"></a>

## Q50. ❓ Can ItemProcessor modify the data?

🔖 **Tags:** `#spring-batch` `#processor` `#transformation`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

**Yes!** That's its primary purpose. ItemProcessor can:

| Modification | How |
|-------------|-----|
| Change field values | `item.setName(item.getName().toUpperCase())` |
| Convert types | Input: `RawCSVRecord` → Output: `Employee` |
| Add computed fields | `item.setTax(item.getSalary() * 0.3)` |
| Merge data | Combine with data from another source |

```java
// Input type ≠ Output type: full transformation
public interface ItemProcessor<I, O> {
    O process(I item) throws Exception;
}

// Example: CSV row → Database entity
ItemProcessor<String[], Employee> processor = csvRow -> {
    Employee emp = new Employee();
    emp.setName(csvRow[0]);
    emp.setSalary(Double.parseDouble(csvRow[1]));
    emp.setDepartment(csvRow[2]);
    emp.setCreatedAt(LocalDateTime.now());
    return emp;
};
```

---

<a id="q3"></a>

## Q51. ❓ Can we have multiple processors in Spring Batch?

🔖 **Tags:** `#spring-batch` `#processor` `#composite`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

**Yes!** Use `CompositeItemProcessor` to chain multiple processors.

```java
@Bean
public CompositeItemProcessor<Employee, Employee> compositeProcessor() {
    CompositeItemProcessor<Employee, Employee> processor = new CompositeItemProcessor<>();
    processor.setDelegates(List.of(
        validationProcessor(),    // Step 1: Validate
        transformProcessor(),     // Step 2: Transform
        enrichmentProcessor()     // Step 3: Enrich
    ));
    return processor;
}
```

```
Item → Processor1 (validate) → Processor2 (transform) → Processor3 (enrich) → Writer
         │                          │                         │
         └─ return null = FILTER    └─ modify data           └─ add extra data
```

> 💡 If ANY processor returns `null`, the item is filtered — subsequent processors are NOT called.

---

<a id="q4"></a>

## Q52. ❓ What is CompositeItemProcessor?

🔖 **Tags:** `#spring-batch` `#processor` `#composite`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

`CompositeItemProcessor` chains multiple processors in sequence. Output of one becomes input of next.

```java
@Bean
public CompositeItemProcessor<RawData, FinalData> compositeProcessor() {
    return new CompositeItemProcessorBuilder<RawData, FinalData>()
            .delegates(List.of(
                (ItemProcessor<RawData, CleanData>) raw -> {
                    // Clean data
                    CleanData clean = new CleanData();
                    clean.setName(raw.getName().trim());
                    return clean;
                },
                (ItemProcessor<CleanData, FinalData>) clean -> {
                    // Enrich data
                    FinalData finalData = new FinalData();
                    finalData.setName(clean.getName());
                    finalData.setScore(calculateScore(clean));
                    return finalData;
                }
            ))
            .build();
}
```

**Type Chain:** `RawData → CleanData → FinalData`

---

<a id="q5"></a>

## Q53. ❓ What happens if ItemProcessor throws an exception?

🔖 **Tags:** `#spring-batch` `#processor` `#error-handling`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

| Configuration | Behavior |
|--------------|----------|
| **No fault tolerance** | Chunk rolls back, step FAILS, job FAILS |
| **Skip configured** | Item is skipped, chunk continues |
| **Retry configured** | Item is retried N times |

```java
// With skip
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("step", repo)
            .<I, O>chunk(100, tx)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .faultTolerant()
            .skip(ValidationException.class)
            .skipLimit(100)
            .retry(TransientException.class)
            .retryLimit(3)
            .build();
}
```

---

<a id="q6"></a>

## Q54. ❓ Can we filter records using ItemProcessor?

🔖 **Tags:** `#spring-batch` `#processor` `#filtering`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

**Yes!** Return `null` from `process()` to filter/exclude an item.

```java
@Bean
public ItemProcessor<Employee, Employee> filterProcessor() {
    return employee -> {
        // Filter: only keep employees with salary > 30000
        if (employee.getSalary() <= 30000) {
            return null;  // FILTERED — not passed to writer
        }
        
        // Filter: skip inactive
        if (!employee.isActive()) {
            return null;  // FILTERED
        }
        
        return employee;  // PASSED to writer
    };
}
```

Filtered items are tracked in `StepExecution.filterCount` — they are NOT errors.

---

[← Back to Spring Batch Index](./README.md) | [← Prev: Writers](./04-writers.md) | [Next: Job Execution →](./06-job-execution.md)
]]>
