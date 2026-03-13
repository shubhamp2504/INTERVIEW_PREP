# 🟡 Spring Batch — ItemProcessor Deep Dive (Q49–Q54)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q49"></a>

## Q49. What is ItemProcessor used for?

### 🔑 Quick Answer

> ItemProcessor sits between Reader and Writer to **transform**, **validate**, **filter**, or **enrich** each item. It takes one type as input and can return a different type as output. Returning `null` filters the item out.

### 📖 Step-by-Step Explanation

**Step 1 — Four common purposes:**

```
1. TRANSFORM  — Change data format
   CsvRow{name="amit", salary="50000"}  →  Employee{name="Amit", salary=50000.00}

2. VALIDATE   — Check business rules
   Employee{salary=-5000}  →  null (filtered!) or throw ValidationException

3. FILTER     — Include/exclude based on criteria
   Employee{active=false}  →  null (excluded from output)

4. ENRICH     — Add data from external source
   Order{customerId=123}  →  Order{customerId=123, customerName="Amit", address="Mumbai"}
```

**Step 2 — The interface:**

```java
public interface ItemProcessor<I, O> {
    O process(I item) throws Exception;
}

// I = Input type (what Reader returns)
// O = Output type (what Writer receives)
// Can be same type or different types!
```

**Step 3 — It's OPTIONAL:**

```java
// Without processor — Reader output goes directly to Writer
.<Employee, Employee>chunk(500, tx)
    .reader(reader())
    .writer(writer())      // No .processor() — totally valid!
    .build();
```

### 🗣️ How to Explain in Interview

> *"ItemProcessor is the business logic layer between reading and writing. It serves four purposes: transforming data — like converting a CSV row into an entity; validating — checking business rules before writing; filtering — returning null to exclude items; and enriching — looking up additional data from other services. The key things to know are: it's optional, it runs once per item, input and output types can be different, and returning null is the standard way to filter items without throwing an error."*

### 💻 Code Example

```java
// Real-world processor: all four purposes in one
@Bean
public ItemProcessor<CsvOrderRow, Order> orderProcessor() {
    return csvRow -> {
        
        // 1. VALIDATE — reject invalid data
        if (csvRow.getAmount() == null || csvRow.getAmount().isEmpty()) {
            return null;  // Filter out rows with no amount
        }
        
        // 2. TRANSFORM — convert CSV strings to proper types
        Order order = new Order();
        order.setOrderId(Long.parseLong(csvRow.getOrderId()));
        order.setProduct(csvRow.getProduct().trim());
        order.setAmount(new BigDecimal(csvRow.getAmount()));
        
        // 3. ENRICH — calculate tax
        order.setTax(order.getAmount().multiply(new BigDecimal("0.18")));
        order.setTotal(order.getAmount().add(order.getTax()));
        
        // 4. ENRICH — add creation timestamp
        order.setCreatedAt(LocalDateTime.now());
        order.setStatus("PENDING");
        
        return order;  // Goes to Writer
    };
}
```

### ⚡ Key Points to Remember

1. **Transform, Validate, Filter, Enrich** — four main uses
2. **Optional** — skip if not needed
3. **Return null** = filter item (not an error, tracked as filterCount)
4. **Input ≠ Output type** is fine (CsvRow → Entity)
5. Runs **once per item**, not once per chunk

---

<a id="q50"></a>

## Q50. Can ItemProcessor change/modify the data?

### 🔑 Quick Answer

> Yes! The processor can **change field values**, **convert types**, **add computed fields**, **merge data from external sources**, and even **completely change the object type**. This is its primary purpose.

### 📖 Step-by-Step Explanation

**Step 1 — Type conversion (most common):**

```java
// Input: Raw CSV data (all Strings)
CsvRow { orderId="1001", amount="599.99", date="2024-01-15" }

// Output: Proper Java types
Order { orderId=1001L, amount=599.99, date=LocalDate(2024,1,15), tax=108.00 }
```

**Step 2 — Field modification:**

```java
@Bean
public ItemProcessor<Employee, Employee> salaryAdjuster() {
    return emp -> {
        // Modify existing fields
        emp.setName(emp.getName().toUpperCase());          // Normalize
        emp.setSalary(emp.getSalary().multiply(1.10));     // 10% raise
        emp.setUpdatedDate(LocalDate.now());               // Set timestamp
        emp.setStatus("PROCESSED");                        // Update status
        return emp;
    };
}
```

**Step 3 — Data enrichment from external service:**

```java
@Bean
public ItemProcessor<Order, EnrichedOrder> enrichmentProcessor(
        CustomerService customerService) {
    return order -> {
        // Look up customer details from another service/DB
        Customer customer = customerService.findById(order.getCustomerId());
        
        EnrichedOrder enriched = new EnrichedOrder();
        enriched.setOrderId(order.getOrderId());
        enriched.setAmount(order.getAmount());
        enriched.setCustomerName(customer.getName());     // Enriched!
        enriched.setCustomerEmail(customer.getEmail());   // Enriched!
        enriched.setShippingAddress(customer.getAddress()); // Enriched!
        return enriched;
    };
}
```

### 🗣️ How to Explain in Interview

> *"Yes, modification is the primary purpose of the processor. In a typical ETL job, the reader gives you raw data — strings from a CSV or basic fields from a query — and the processor converts them to proper types, normalizes values, calculates derived fields, and enriches with data from other sources. For example, in our billing job, the processor took raw order data, calculated tax and total, looked up customer details from a service, and produced a fully enriched BillingRecord that the writer then inserted into the database."*

### ⚡ Key Points to Remember

1. Can **change field values** (normalize, compute)
2. Can **change the type entirely** (CsvRow → Entity)
3. Can **enrich** with external data (DB lookups, API calls)
4. Returns a **new object or modified object** — both are fine
5. Don't do heavy I/O in processor if possible (slows down chunk processing)

---

<a id="q51"></a>

## Q51. Can we have multiple processors? How?

### 🔑 Quick Answer

> Yes! Use **CompositeItemProcessor** to chain multiple processors in sequence. The output of processor 1 becomes the input of processor 2, and so on. If any processor returns null, the item is filtered and none of the remaining processors run.

### 📖 Step-by-Step Explanation

**Step 1 — How chaining works:**

```
Reader → Processor 1 → Processor 2 → Processor 3 → Writer

Item flow:
  CsvRow ──→ [Validator] ──→ [Transformer] ──→ [Enricher] ──→ Writer
              returns          returns            returns
              CsvRow           Employee            EnrichedEmployee
              (or null         (different type!)    (final type for writer)
               to filter)
```

**Step 2 — Type chain:**

```
CompositeItemProcessor<CsvRow, EnrichedEmployee>
  ├── Processor 1: ItemProcessor<CsvRow, CsvRow>           (validate)
  ├── Processor 2: ItemProcessor<CsvRow, Employee>          (transform)
  └── Processor 3: ItemProcessor<Employee, EnrichedEmployee> (enrich)

Note: Output of previous = Input of next
If P1 returns null → P2 and P3 are SKIPPED → item filtered
```

### 🗣️ How to Explain in Interview

> *"Yes, using CompositeItemProcessor. You chain multiple processors where the output of one becomes the input of the next. This keeps each processor focused on one responsibility — say, processor 1 validates, processor 2 transforms, processor 3 enriches. The types can change through the chain — processor 1 might take a CsvRow and return a CsvRow, processor 2 takes that CsvRow and returns an Employee, processor 3 takes an Employee and returns an EnrichedEmployee. If any processor returns null, the remaining processors are skipped and the item is filtered."*

### 💻 Code Example

```java
@Bean
public CompositeItemProcessor<CsvRow, EnrichedEmployee> compositeProcessor() {
    CompositeItemProcessor<CsvRow, EnrichedEmployee> composite = 
            new CompositeItemProcessor<>();
    
    composite.setDelegates(List.of(
            validationProcessor(),    // Step 1: Validate
            transformProcessor(),     // Step 2: CsvRow → Employee
            enrichmentProcessor()     // Step 3: Employee → EnrichedEmployee
    ));
    
    return composite;
}

// Processor 1: Validate (filter bad records)
@Bean
public ItemProcessor<CsvRow, CsvRow> validationProcessor() {
    return row -> {
        if (row.getName() == null || row.getName().isBlank()) return null;
        if (row.getSalary() == null) return null;
        return row;  // Valid — pass to next processor
    };
}

// Processor 2: Transform type
@Bean
public ItemProcessor<CsvRow, Employee> transformProcessor() {
    return row -> {
        Employee emp = new Employee();
        emp.setName(row.getName().trim());
        emp.setSalary(new BigDecimal(row.getSalary()));
        return emp;
    };
}

// Processor 3: Enrich with computed data
@Bean
public ItemProcessor<Employee, EnrichedEmployee> enrichmentProcessor() {
    return emp -> {
        EnrichedEmployee enriched = new EnrichedEmployee(emp);
        enriched.setTax(emp.getSalary().multiply(new BigDecimal("0.30")));
        enriched.setNetSalary(emp.getSalary().subtract(enriched.getTax()));
        return enriched;
    };
}
```

### ⚡ Key Points to Remember

1. **CompositeItemProcessor** chains multiple processors
2. Output of one = Input of next (**type chain**)
3. **Null from any processor** → remaining processors skipped → item filtered
4. Keeps each processor **single responsibility**
5. Step config: `.<CsvRow, EnrichedEmployee>chunk(500, tx)` — first=Reader output, last=Writer input

---

<a id="q52"></a>

## Q52. What is CompositeItemProcessor?

### 🔑 Quick Answer

> CompositeItemProcessor is a **chain of processors** that execute in sequence. Each processor can transform the type, and the final output goes to the writer. It implements the **Decorator pattern** — wrapping multiple processors as one.

### 📖 Step-by-Step Explanation

This is the same concept as Q51 — CompositeItemProcessor IS the mechanism for chaining processors. The key additional detail:

**Internal execution:**

```java
// What CompositeItemProcessor does internally:
public O process(I item) throws Exception {
    Object result = item;
    
    for (ItemProcessor processor : delegates) {
        result = processor.process(result);
        
        if (result == null) {
            return null;  // Short-circuit! Skip remaining processors
        }
    }
    
    return (O) result;
}
```

**Important design points:**

1. **Order matters** — processors execute in the order you add them
2. **Early termination** — null from any processor stops the chain
3. **Type safety** — compile-time checking only on first input and last output

### 🗣️ How to Explain in Interview

> *"CompositeItemProcessor chains processors using the Decorator pattern. Internally, it loops through each delegate processor and passes the result to the next one. If any processor returns null, it immediately returns null — short-circuiting the rest. This is useful because you can put your validation first. If an item fails validation, you don't waste time enriching it. The composite only type-checks the first input and last output — the intermediate types are handled at runtime."*

### ⚡ Key Points to Remember

1. **Decorator pattern** — wraps N processors as 1
2. **Execution order** = order of delegates list
3. **Null = short-circuit** — stops immediately
4. Spring config: `.<FirstType, LastType>chunk(...)` — only first and last types matter
5. Put **validation first** to avoid unnecessary processing

---

<a id="q53"></a>

## Q53. What happens if ItemProcessor throws an exception?

### 🔑 Quick Answer

> Without fault tolerance → the **chunk fails and rolls back**, the job fails. With skip configured → the **item is skipped** (processSkipCount incremented). With retry configured → the item is **retried N times** before skipping or failing.

### 📖 Step-by-Step Explanation

**Step 1 — Default behavior (no fault tolerance):**

```
Chunk: read 500 items
Processing item #247 → 💥 ValidationException!

Result:
  - ALL 500 items in this chunk are LOST (rollback)
  - Step: FAILED
  - Job: FAILED
```

**Step 2 — With skip:**

```java
.faultTolerant()
.skip(ValidationException.class)
.skipLimit(100)
```

```
Processing item #247 → 💥 ValidationException!

Result:
  - Item #247 is SKIPPED
  - Other 499 items continue processing
  - processSkipCount: 1
  - Processing continues to next item
```

**Step 3 — With retry:**

```java
.faultTolerant()
.retry(ServiceTimeoutException.class)
.retryLimit(3)
```

```
Processing item #247 → 💥 ServiceTimeoutException!
  Retry 1: process(#247) → 💥 ServiceTimeoutException!
  Retry 2: process(#247) → ✅ Success! Continue normally.

If all 3 retries fail → skip (if configured) or fail the chunk
```

**Step 4 — Skip + Retry together (typical production config):**

```java
.faultTolerant()
.retry(ServiceTimeoutException.class)    // Retry transient errors
.retryLimit(3)                           // Up to 3 times
.skip(ValidationException.class)         // Skip validation errors
.skip(ServiceTimeoutException.class)     // Skip after retries exhausted
.skipLimit(50)                           // Max 50 total skips
```

### 🗣️ How to Explain in Interview

> *"Without fault tolerance, a processor exception fails the entire chunk and the job. In production, I always configure skip and retry. For transient errors like a service timeout, I configure retry with 3 attempts — often the second try succeeds. For permanent errors like validation failures, I configure skip so the bad item is excluded and processing continues. I always combine both: retry transient errors first, and if retries are exhausted, skip that item. With a SkipListener, I log every skipped item for later investigation."*

### ⚡ Key Points to Remember

1. **No fault tolerance** → chunk fails, job fails
2. **Skip** → bad item excluded, rest continue
3. **Retry** → try again (for transient errors like timeouts)
4. **Skip + Retry together** = best production setup
5. Always log skipped items with **SkipListener**

---

<a id="q54"></a>

## Q54. Can we filter records using ItemProcessor?

### 🔑 Quick Answer

> Yes! **Return null** from the processor to filter an item. It won't be sent to the writer. This is tracked as `filterCount` in StepExecution — it's NOT treated as an error or skip.

### 📖 Step-by-Step Explanation

**Step 1 — Filter is intentional, Skip is error handling:**

```
FILTER (processor returns null):
  - Business decision: "I don't want this item"
  - Tracked as: filterCount
  - NOT an error
  - Doesn't count toward skipLimit

SKIP (exception + skip configured):
  - Error handling: "This item is broken"
  - Tracked as: processSkipCount / readSkipCount / writeSkipCount
  - IS an error
  - Counts toward skipLimit
```

**Step 2 — Filter example:**

```java
@Bean
public ItemProcessor<Employee, Employee> activeEmployeeFilter() {
    return emp -> {
        // Business rule: only process active employees in IT dept
        if (!emp.isActive()) return null;
        if (!"IT".equals(emp.getDepartment())) return null;
        
        return emp;  // This employee passes the filter
    };
}
```

**Step 3 — Resulting counts:**

```
Total employees in CSV: 10,000
  - Active IT: 3,000             → writeCount: 3,000
  - Active non-IT: 4,000         → filterCount: 4,000 (returned null)
  - Inactive: 3,000              → filterCount: 3,000 (returned null)
  
StepExecution:
  readCount: 10,000
  filterCount: 7,000     ← 4000 + 3000 (processor returned null)
  writeCount: 3,000      ← Only these went to writer
  skipCount: 0           ← No errors!
```

### 🗣️ How to Explain in Interview

> *"Yes, returning null from the processor is the standard filtering mechanism. If I'm processing 10,000 employees but only want active IT employees, the processor returns null for everyone else. Spring Batch tracks these as filterCount, not as errors. This is different from skip — filtering is a business decision, skipping is error handling. readCount minus filterCount minus skipCount equals writeCount. In monitoring, I check if filterCount is unexpectedly high or low, because that might indicate a data quality issue."*

### ⚡ Key Points to Remember

1. **Return null** = filtered out
2. Tracked as **filterCount** (not error, not skip)
3. `readCount - filterCount - skipCount = writeCount`
4. **Filter ≠ Skip**: filter is intentional, skip is error handling
5. Use filtering for **business rules**, skip for **error handling**

---

> **🎯 Navigation:** [← Writers (Q41-48)](04-writers.md) | [Next → Job Execution (Q55-62)](06-job-execution.md) | [📋 All Sections](README.md)
