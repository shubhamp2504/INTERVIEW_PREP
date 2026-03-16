# ⚙️ ItemProcessors — Q49 to Q54

---

## Q49. What is ItemProcessor used for?

### 📝 One-Liner
ItemProcessor transforms, validates, filters, or enriches each item between reading and writing — it's the business logic layer of chunk processing.

### 🔑 Quick Answer
ItemProcessor sits between reader and writer. It takes one input item and returns one output item (or null to filter). Four main uses: **(1) Transform** — change format, calculate fields, convert types. **(2) Validate** — check business rules, throw exception for invalid data. **(3) Filter** — return null to exclude items. **(4) Enrich** — add data from external sources (DB lookups, API calls). It's OPTIONAL — you can skip it if reader output goes directly to writer. Runs once per item, not per chunk. *(Reader aur Writer ke beech ka business logic — transform, validate, filter, enrich)*

### 📖 How It Works
```
ItemProcessor<I, O>:

  Input (I)          Processor           Output (O)
┌──────────┐    ┌────────────────┐    ┌──────────┐
│ CsvRow   │ →  │ Transform      │ →  │ Employee │  (type change)
│ raw data │    │ Validate       │    │ enriched │  
│          │    │ Filter (null)  │    │ data     │
│          │    │ Enrich         │    │          │
└──────────┘    └────────────────┘    └──────────┘

Returns:
- Object → item continues to writer
- null   → item FILTERED (excluded from write, tracked as filterCount)
- throws → error handling (skip/retry/fail based on config)
```

Key facts:
- Input type (I) and Output type (O) can be SAME or DIFFERENT
- Called once PER ITEM (not per chunk)
- OPTIONAL component — can skip if no processing needed
- Runs INSIDE the chunk transaction

### 🗣️ How to Say in Interview
"ItemProcessor is the business logic layer between reader and writer. It processes one item at a time with four main uses: transforming data format, validating business rules, filtering unwanted records, or enriching with external data. It's optional — you only use it when you need to transform or validate data. In my project, our processor validated payment amounts, converted currency using an exchange rate API, calculated fees, and filtered out duplicate transactions by returning null. The processor changed the type from RawPayment to ProcessedPayment."

### 💻 Code
```java
// Transform + Validate processor
@Component
public class PaymentProcessor implements ItemProcessor<RawPayment, ProcessedPayment> {

    @Override
    public ProcessedPayment process(RawPayment raw) {
        // Validate
        if (raw.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new ValidationException("Invalid amount: " + raw.getAmount());
        }

        // Transform (type change: RawPayment → ProcessedPayment)
        ProcessedPayment processed = new ProcessedPayment();
        processed.setId(raw.getTxnId());
        processed.setAmount(raw.getAmount());
        processed.setFee(raw.getAmount().multiply(new BigDecimal("0.02"))); // 2% fee
        processed.setProcessedAt(LocalDateTime.now());

        // Filter (return null to exclude)
        if ("DUPLICATE".equals(raw.getStatus())) {
            return null;  // filtered → tracked as filterCount
        }

        return processed;  // continues to writer
    }
}

// Usage in step (processor is optional)
@Bean
public Step paymentStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("paymentStep", repo)
            .<RawPayment, ProcessedPayment>chunk(500, tx)
            .reader(reader())
            .processor(paymentProcessor())   // optional — remove to skip processing
            .writer(writer())
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Returning null is NOT an error — it's intentional filtering *(null return karna error nahi, filtering hai)*
- Processor runs per ITEM, not per chunk — expensive operations (API calls) multiply by item count
- If processor throws, behavior depends on fault tolerance config (skip/retry/fail)
- Processor runs INSIDE chunk transaction — long processing extends transaction duration
- Don't do heavy I/O in processor without considering throughput impact

### 🆚 vs. Comparison
| Processor Action | Return Value | Tracked As |
|-----------------|-------------|------------|
| Transform/Validate | New object | writeCount |
| Filter | null | filterCount |
| Error | throw exception | skipCount (if skip enabled) |

### 🎯 Tricky Interview Qs

**Q: Is ItemProcessor mandatory?**
No. It's completely optional. If your reader output can go directly to writer, skip the processor: `.<Order, Order>chunk(500, tx).reader(r).writer(w).build()`.

**Q: Can processor change the type?**
Yes. `ItemProcessor<CsvRow, Employee>` takes CsvRow and returns Employee. The generic types I and O can be different.

### ⚡ Remember
- Four uses: Transform, Validate, Filter, Enrich *(TVFE yaad rakho)*
- Return null = filter (tracked as filterCount) *(null = filter, error nahi)*
- I and O can be same or different types
- OPTIONAL — skip if no processing needed
- Runs per item, inside chunk transaction

### 🔗 Follow-ups
- [Q50 → Can processor modify data?](#q50)
- [Q51 → Multiple processors in chain](#q51)
- [Q54 → Filtering with processor](#q54)

---

## Q50. Can ItemProcessor change/modify the data?

### 📝 One-Liner
Yes — ItemProcessor can change field values, change the object type entirely, or enrich with data from external sources.

### 🔑 Quick Answer
Absolutely. Three types of modifications: **(1) Field modification** — change values (normalize, calculate, format). **(2) Type conversion** — input type differs from output type (CsvRow → JPA Entity). **(3) Data enrichment** — fetch additional data from database, API, or cache and add to the object. The processor creates and returns a new object (or modifies the existing one). *(Field badlo, type badlo, ya bahar se data jod ke enrich karo — sab ho sakta hai)*

### 📖 How It Works
```
Three Types of Modification:

1. Field Modification:
   Employee {salary: 50000} → Employee {salary: 55000}  (10% raise)

2. Type Conversion:
   CsvRow {line: "1,John,50000"} → Employee {id:1, name:"John", salary:50000}

3. Data Enrichment:
   Order {customerId: 123} → EnrichedOrder {customerId:123, customerName:"John", creditScore:750}
                                              ↑ looked up from DB/API
```

### 🗣️ How to Say in Interview
"Yes, ItemProcessor can modify data in three ways. Field modification changes values like calculating a 10% salary increase or normalizing names to uppercase. Type conversion transforms the input type to a completely different output type — like converting a raw CSV row to a JPA entity. Data enrichment fetches additional information from external sources and adds it to the object. In my project, our processor fetched the customer's credit score from an external API and added it to the order object for risk evaluation, effectively enriching the data before writing."

### 💻 Code
```java
// 1. Field modification (same type)
@Bean
public ItemProcessor<Employee, Employee> salaryProcessor() {
    return emp -> {
        emp.setSalary(emp.getSalary().multiply(new BigDecimal("1.10")));  // 10% raise
        emp.setName(emp.getName().toUpperCase());  // normalize
        emp.setUpdatedAt(LocalDateTime.now());
        return emp;
    };
}

// 2. Type conversion (different type)
@Bean
public ItemProcessor<CsvRow, Employee> conversionProcessor() {
    return row -> {
        Employee emp = new Employee();
        emp.setId(Long.parseLong(row.getField(0)));
        emp.setName(row.getField(1));
        emp.setSalary(new BigDecimal(row.getField(2)));
        emp.setDepartment(DepartmentMapper.map(row.getField(3)));
        return emp;
    };
}

// 3. Data enrichment (add external data)
@Component
public class EnrichmentProcessor implements ItemProcessor<Order, EnrichedOrder> {
    
    private final CustomerService customerService;
    
    @Override
    public EnrichedOrder process(Order order) {
        // Lookup customer data from DB/API
        Customer customer = customerService.findById(order.getCustomerId());
        
        EnrichedOrder enriched = new EnrichedOrder();
        enriched.setOrderId(order.getId());
        enriched.setAmount(order.getAmount());
        enriched.setCustomerName(customer.getName());
        enriched.setCreditScore(customer.getCreditScore());
        enriched.setRiskLevel(calculateRisk(order, customer));
        return enriched;
    }
}
```

### ⚠️ Pitfalls / Gotchas
- Enrichment with external API → runs per item → can be slow at scale *(API call per item = scale pe slow hoga)*
- Consider caching for repeated lookups (same customer across many orders)
- Modifying the original input object works but creating a new object is cleaner
- Heavy processing extends chunk transaction duration

### ⚡ Remember
- Field modification: change values (normalize, calculate)
- Type conversion: `Processor<A, B>` converts A to B *(type badal sakta hai)*
- Enrichment: fetch from DB/API and add to object
- Cache external lookups for performance
- All three can be combined in one processor

### 🔗 Follow-ups
- [Q49 → ItemProcessor basics](#q49)
- [Q51 → Chain multiple processors](#q51)
- [Q53 → Exception in processor](#q53)

---

## Q51. Can we have multiple processors? How?

### 📝 One-Liner
Yes — use CompositeItemProcessor to chain multiple processors in sequence where the output of one becomes the input of the next.

### 🔑 Quick Answer
`CompositeItemProcessor` chains N processors in sequence. Output of processor 1 becomes input of processor 2, and so on. Types can change through the chain: `CsvRow → Employee → EnrichedEmployee`. If ANY processor in the chain returns null, the remaining processors are skipped and the item is filtered. Only the first input type and last output type are type-checked by the chunk step. *(Ek ke baad ek processor — pehle wale ka output agle ka input banta hai)*

### 📖 How It Works
```
CompositeItemProcessor Chain:

Input → Processor1 → Processor2 → Processor3 → Output
CsvRow → Validator → Transformer → Enricher → EnrichedEmployee

Data flow:
CsvRow ──→ validate() ──→ transform() ──→ enrich() ──→ EnrichedEmployee
           │                │                │
           │ returns null?  │ returns null?  │ returns null?
           └── SKIP ALL ──→└── SKIP REST ──→└── FILTERED
               remaining        remaining
```

Key rule: **If ANY processor returns null → remaining processors skipped, item filtered.**

### 🗣️ How to Say in Interview
"Yes, you can chain multiple processors using CompositeItemProcessor. Processors execute in the order they're registered, with each processor's output becoming the next processor's input. Types can change through the chain — for example, validator takes CsvRow, transformer converts to Employee, enricher adds customer data and returns EnrichedEmployee. If any processor in the chain returns null, the remaining processors are skipped and the item is filtered. In my project, we had four processors in chain: validation, deduplication, currency conversion, and risk scoring — each focused on a single responsibility."

### 💻 Code
```java
@Bean
public CompositeItemProcessor<CsvRow, EnrichedEmployee> compositeProcessor() {
    CompositeItemProcessor<CsvRow, EnrichedEmployee> composite = new CompositeItemProcessor<>();
    composite.setDelegates(List.of(
            validationProcessor(),    // CsvRow → CsvRow (validates, returns null if invalid)  
            transformProcessor(),     // CsvRow → Employee (type change)
            enrichProcessor()         // Employee → EnrichedEmployee (adds external data)
    ));
    return composite;
}

@Bean
public ItemProcessor<CsvRow, CsvRow> validationProcessor() {
    return row -> {
        if (row.getField("name").isBlank()) return null;   // filter invalid
        if (row.getField("salary").equals("0")) return null; // filter zero salary
        return row;  // valid → continue chain
    };
}

@Bean
public ItemProcessor<CsvRow, Employee> transformProcessor() {
    return row -> {
        Employee emp = new Employee();
        emp.setName(row.getField("name").toUpperCase());
        emp.setSalary(new BigDecimal(row.getField("salary")));
        return emp;
    };
}

@Bean
public ItemProcessor<Employee, EnrichedEmployee> enrichProcessor() {
    return emp -> {
        EnrichedEmployee enriched = new EnrichedEmployee(emp);
        enriched.setDepartmentHead(departmentService.getHead(emp.getDepartment()));
        return enriched;
    };
}

// Step only sees first input and last output type
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("step", repo)
            .<CsvRow, EnrichedEmployee>chunk(500, tx)  // CsvRow in, EnrichedEmployee out
            .reader(csvReader())
            .processor(compositeProcessor())
            .writer(enrichedWriter())
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Delegate ORDER matters — executes in list order *(delegates ka order matter karta hai)*
- Put validation FIRST to avoid unnecessary processing on invalid items
- Only first input type and last output type need to match step generics
- Middle types are not type-checked at compile time — runtime ClassCastException possible
- Null from any processor = skip ALL remaining processors

### ⚡ Remember
- CompositeItemProcessor chains N processors in sequence
- Output of P1 = Input of P2 *(ek ka output agle ka input)*
- Types can change through chain
- Null from any = rest skipped, item filtered
- Put validation first (fail fast)

### 🔗 Follow-ups
- [Q52 → CompositeItemProcessor detail](#q52)
- [Q49 → Single processor basics](#q49)
- [Q54 → Filtering with null return](#q54)

---

## Q52. What is CompositeItemProcessor?

### 📝 One-Liner
CompositeItemProcessor implements the Decorator pattern — wrapping N processors as one, executing them sequentially with null-short-circuit behavior.

### 🔑 Quick Answer
CompositeItemProcessor is a decorator that wraps multiple `ItemProcessor` instances into a single processor. It iterates through delegates in order, passing each output as the next input. If any delegate returns null (filter), it short-circuits — remaining delegates are skipped and null is returned. This follows the **Chain of Responsibility / Decorator pattern**. Only the first delegate's input type and last delegate's output type matter for the step configuration. *(Decorator pattern — bahut saare processors ko ek mein wrap kar deta hai)*

### 📖 How It Works
```
Internal Pseudocode:

public O process(I item) {
    Object result = item;
    for (ItemProcessor delegate : delegates) {
        result = delegate.process(result);
        if (result == null) {
            return null;  // SHORT-CIRCUIT: skip remaining, filter item
        }
    }
    return (O) result;
}

Example with 3 delegates:
  Validator.process(csvRow) → csvRow (valid)
  Transformer.process(csvRow) → employee (type change)  
  Enricher.process(employee) → enrichedEmployee (enriched)
  → return enrichedEmployee to writer

Short-circuit example:
  Validator.process(csvRow) → null (invalid!)
  → Transformer NEVER called
  → Enricher NEVER called
  → return null → item FILTERED
```

### 🗣️ How to Say in Interview
"CompositeItemProcessor is a decorator that chains multiple processors into one. Internally, it loops through delegates in order, passing each processor's output as the next processor's input. If any processor returns null, it short-circuits — the remaining processors are skipped and the item is filtered. In my project, we designed each processor with a single responsibility — validation, transformation, enrichment, and risk scoring. The composite wrapped them as one processor for the step. We made sure validation was first so invalid records were filtered before expensive enrichment operations."

### 💻 Code
```java
// Creating CompositeItemProcessor
@Bean
public CompositeItemProcessor<RawOrder, ProcessedOrder> orderProcessor() {
    CompositeItemProcessor<RawOrder, ProcessedOrder> composite = new CompositeItemProcessor<>();
    
    // Delegates execute in this order — ORDER MATTERS
    composite.setDelegates(List.of(
            deduplicationProcessor(),   // filter duplicates (null = filter)
            validationProcessor(),       // validate business rules
            transformationProcessor(),   // convert to target type
            enrichmentProcessor()        // add external data
    ));
    return composite;
}

// Builder approach (Spring Batch 5)
@Bean
public CompositeItemProcessor<RawOrder, ProcessedOrder> orderProcessorBuilder() {
    return new CompositeItemProcessorBuilder<RawOrder, ProcessedOrder>()
            .delegates(List.of(
                deduplicationProcessor(),
                validationProcessor(),
                transformationProcessor(),
                enrichmentProcessor()
            ))
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Null from ANY delegate = short-circuit (all remaining skipped) *(ek bhi null return kare toh baaki sab skip)*
- Order matters — put cheap validators first, expensive enrichers last
- Middle delegate types not checked at compile time — ClassCastException at runtime if types don't align
- Each delegate should be focused (single responsibility)

### 🎯 Tricky Interview Qs

**Q: What design pattern does CompositeItemProcessor follow?**
Decorator/Chain of Responsibility. It wraps multiple processors as one, delegating in sequence with short-circuit on null.

**Q: Can you nest CompositeItemProcessors?**
Yes. A CompositeItemProcessor can contain another CompositeItemProcessor as one of its delegates. Though usually there's no need for this level of nesting.

### ⚡ Remember
- Decorator pattern: N processors → 1 processor *(bahut saare ek mein wrap)*
- Null from any = short-circuit (rest skipped, item filtered)
- Order matters: cheap first, expensive last
- Only first input + last output are type-checked
- Builder available in Spring Batch 5

### 🔗 Follow-ups
- [Q51 → Multiple processors chain](#q51)
- [Q45 → CompositeItemWriter (similar pattern for writers)](#q45)
- [Q53 → Exception handling in processor chain](#q53)

---

## Q53. What happens if ItemProcessor throws an exception?

### 📝 One-Liner
Without fault tolerance the chunk fails and job stops; with skip configured the item is skipped; with retry the processing is retried — best practice is skip + retry together.

### 🔑 Quick Answer
Three scenarios: **(1) No fault tolerance**: exception → chunk rolls back → step FAILED → job FAILED. **(2) With skip**: exception → item skipped (tracked as `processSkipCount`) → remaining items continue. **(3) With retry**: exception → retried up to N times (for transient errors like timeouts). Best practice: combine both — retry first for transient errors, then skip after retries are exhausted. Always add SkipListener to log skipped items for audit. *(Retry pehle karo transient error ke liye, phir skip karo agar retry fail ho jaaye)*

### 📖 How It Works
```
Exception Handling Flow:

processor.process(item) → throws Exception
                              ↓
                   faultTolerant configured?
                   ├── NO  → chunk ROLLBACK → step FAILED → job FAILED
                   └── YES
                        ├── retry configured for this exception?
                        │   ├── YES → retry up to retryLimit times
                        │   │         └── all retries fail → check skip
                        │   └── NO → check skip
                        └── skip configured for this exception?
                            ├── YES → skip item (processSkipCount++)
                            │         → continue processing remaining items
                            └── NO → chunk ROLLBACK → step FAILED
```

### 🗣️ How to Say in Interview
"When a processor throws an exception, behavior depends on fault tolerance configuration. Without it, the chunk rolls back and job fails immediately. With skip enabled, that specific item is skipped and processing continues with the next item — tracked as processSkipCount. With retry, the processing is retried up to the configured limit — ideal for transient errors like API timeouts. In my project, we combined both: retry 3 times for TimeoutException from our external API, and skip up to 100 ValidationExceptions for bad data. We always used a SkipListener to log skipped records to an error table for the operations team to review."

### 💻 Code
```java
@Bean
public Step faultTolerantStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("faultTolerantStep", repo)
            .<Order, ProcessedOrder>chunk(500, tx)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .faultTolerant()
            // Retry transient errors first
            .retry(TimeoutException.class)
            .retry(SocketTimeoutException.class)
            .retryLimit(3)
            // Then skip if retry exhausted or data errors
            .skip(ValidationException.class)
            .skip(TimeoutException.class)    // skip after 3 retries fail
            .skipLimit(100)
            // Always log what was skipped
            .listener(new SkipListener<Order, ProcessedOrder>() {
                @Override
                public void onSkipInProcess(Order item, Throwable t) {
                    log.error("SKIPPED in process: id={}, error={}", 
                        item.getId(), t.getMessage());
                    errorRepository.save(new ErrorRecord(item.getId(), 
                        "PROCESS_SKIP", t.getMessage()));
                }
            })
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Skip limit is TOTAL across read + process + write — not per phase *(skip limit total hai — read + process + write sab mein se)*
- Retry is per ITEM, not per chunk — retryLimit(3) means 3 retries for each failing item
- Without SkipListener, skipped items disappear silently — always log them
- `noSkip(FatalException.class)` prevents certain exceptions from being skipped (job must fail)
- Retry in processor means the same item is re-processed — processor must handle this correctly

### 🎯 Tricky Interview Qs

**Q: What's the difference between skip in processor vs skip in writer?**
In processor: item is simply excluded — Spring Batch knows exactly which item caused the error. In writer: enters scan mode — has to re-process and re-write each item individually to find the bad one (more expensive).

**Q: If retryLimit is 3 and the processor fails on the 3rd retry, does the item get skipped?**
Only if skip is also configured for that exception type AND skipLimit is not reached. Otherwise the chunk/step fails.

### ⚡ Remember
- No config → exception kills job
- Skip = exclude bad item, continue rest *(kharab item hatao, baaki chalte raho)*
- Retry = try again N times (for transient errors)
- Best: retry + skip together
- ALWAYS add SkipListener (otherwise silent data loss)

### 🔗 Follow-ups
- [Q24 → Chunk failure general behavior](#q24)
- [Q48 → Write failure and scan mode](#q48)
- [Q70 → Error handling strategies](#q70)

---

## Q54. Can we filter records using ItemProcessor?

### 📝 One-Liner
Yes — return null from the processor to filter (exclude) an item; it will be tracked as `filterCount`, not `skipCount`.

### 🔑 Quick Answer
Returning null from `ItemProcessor.process()` is the standard way to filter records. The item is excluded from the write list and tracked as `filterCount` in StepExecution. This is fundamentally different from skipping: **filtering is a business decision** (intentional exclusion), while **skipping is error handling** (exception-based exclusion). The formula: `readCount - filterCount - skipCount = writeCount`. *(Null return karo toh filter — ye business decision hai, skip se alag hai jo error handling hai)*

### 📖 How It Works
```
Filter vs Skip:

FILTER (business decision):
  processor.process(item) → return null
  → filterCount++
  → No error, no log needed (intentional)
  → Example: skip inactive employees, ignore test data

SKIP (error handling):
  processor.process(item) → throws Exception
  → skipCount++ (if fault tolerance enabled)
  → Should be logged (unexpected)
  → Example: invalid data format, timeout

Record Count Formula:
  readCount - filterCount - processSkipCount - writeSkipCount = writeCount
  
  Example: Read 1000, filtered 50, skipped 5
  → 1000 - 50 - 5 = 945 written
```

### 🗣️ How to Say in Interview
"Yes, returning null from the processor is the standard way to filter records. It's counted as filterCount in StepExecution, which is different from skipCount. Filtering is a business decision — like excluding inactive employees or test records. Skipping is error handling — for items that throw exceptions. In my project, our processor filtered out three types of records: inactive customers, orders below the minimum amount, and duplicate transactions detected by a dedup cache. We monitored filterCount in our post-step listener to alert if the filter rate exceeded 20%, which could indicate a data quality issue."

### 💻 Code
```java
@Bean
public ItemProcessor<Employee, Employee> filterProcessor() {
    return emp -> {
        // Business rule: exclude inactive employees
        if (!emp.isActive()) {
            return null;  // FILTERED → filterCount++
        }

        // Business rule: exclude test data
        if (emp.getDepartment().startsWith("TEST")) {
            return null;  // FILTERED
        }

        // Business rule: minimum salary threshold
        if (emp.getSalary().compareTo(new BigDecimal("1000")) < 0) {
            return null;  // FILTERED
        }

        return emp;  // passes through → will be written
    };
}

// Monitor filter rate in step listener
@Bean
public StepExecutionListener filterMonitor() {
    return new StepExecutionListener() {
        @Override
        public ExitStatus afterStep(StepExecution se) {
            long read = se.getReadCount();
            long filtered = se.getFilterCount();
            long skipped = se.getSkipCount();
            long written = se.getWriteCount();
            
            double filterRate = (double) filtered / read * 100;
            log.info("Read: {}, Filtered: {} ({}%), Skipped: {}, Written: {}",
                read, filtered, String.format("%.1f", filterRate), skipped, written);
            
            // Alert if filter rate is too high
            if (filterRate > 20.0) {
                log.warn("HIGH FILTER RATE: {}% — check data quality!", 
                    String.format("%.1f", filterRate));
            }
            return se.getExitStatus();
        }
    };
}
```

### ⚠️ Pitfalls / Gotchas
- Filter (null return) does NOT require `.faultTolerant()` — it's normal behavior *(filter ke liye faultTolerant config nahi chahiye)*
- Don't confuse filterCount with skipCount — they track different things
- Filtering too aggressively → silently losing data → always monitor filter rate
- In CompositeItemProcessor, null from ANY delegate = immediate filter (short-circuit)
- `readCount - filterCount - skipCount = writeCount` (use this to verify data)

### 🆚 vs. Comparison
| Aspect | Filter (null return) | Skip (exception) |
|--------|---------------------|-------------------|
| Intent | Business decision | Error handling |
| Mechanism | Return null | Throw exception |
| Requires faultTolerant? | No | Yes |
| Tracked as | filterCount | skipCount |
| Should log? | Optional (monitoring) | Always (audit) |
| Example | Exclude inactive, test data | Invalid format, timeout |

### 🎯 Tricky Interview Qs

**Q: Does filtering affect restart?**
No. Filtered items are already "processed successfully" (intentionally excluded). On restart, Spring Batch won't re-process already-processed items. The filter is a normal outcome, not a failure.

**Q: Can you filter in reader instead of processor?**
Technically yes — add WHERE clause in SQL reader or filter lines in file reader. But business-level filtering in processor is cleaner and more testable.

### ⚡ Remember
- Return null = FILTER (business decision, filterCount) *(null = filter, intentional)*
- Throw exception = SKIP (error, skipCount)
- `readCount - filterCount - skipCount = writeCount`
- No faultTolerant needed for filtering
- Monitor filter rate to catch data quality issues early

### 🔗 Follow-ups
- [Q49 → ItemProcessor uses](#q49)
- [Q53 → Exception handling in processor](#q53)
- [Q22 → filterCount in internal chunk flow](#q22)
