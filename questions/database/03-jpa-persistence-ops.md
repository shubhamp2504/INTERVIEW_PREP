# 🗄️ Hibernate/JPA — Persistence Operations (Q1)

> **Source**: EPAM Systems Java Backend Interview  
> **Coverage**: save() vs saveAndFlush() and JPA persistence context flush mechanics

---

<a id="q1"></a>
## Q1. What is the difference between save() and saveAndFlush() in Spring Data JPA?

### 📝 One-Liner
`save()` stages the entity in the **persistence context** (SQL sent later at flush time); `saveAndFlush()` stages AND immediately **flushes SQL to the database** in the same call.

### 🔑 Quick Answer
Both are from `JpaRepository`. `save()` → calls `EntityManager.persist()` (new) or `merge()` (existing) → entity is **managed** but SQL is NOT sent yet — it's written to DB at flush time (before commit, before queries, or explicit flush). `saveAndFlush()` → same as `save()` + immediately calls `EntityManager.flush()` → SQL is sent to DB right away (but NOT committed — still within the transaction). **Use save()** in 99% of cases — let JPA batch and optimize. **Use saveAndFlush()** when you need the DB-generated ID immediately, need to catch DB constraint violations before further logic, or need data visible to raw SQL/JDBC queries within the same transaction. *(save = entity manage karo, SQL baad mein jayega; saveAndFlush = entity manage karo AUR SQL abhi bhejo)*

### 📖 How It Works (Detailed Explanation)

```
save() — deferred SQL:
┌──────────────────────────────────────────────────────────┐
│ Transaction Start                                         │
│                                                           │
│  save(entity1)  → persist in context   ← NO SQL yet     │
│  save(entity2)  → persist in context   ← NO SQL yet     │
│  save(entity3)  → persist in context   ← NO SQL yet     │
│                                                           │
│  ... more business logic ...                              │
│                                                           │
│  Transaction Commit                                       │
│    └── Flush → INSERT entity1, entity2, entity3  ← SQL! │
│    └── Commit → permanent                                │
└──────────────────────────────────────────────────────────┘

saveAndFlush() — immediate SQL:
┌──────────────────────────────────────────────────────────┐
│ Transaction Start                                         │
│                                                           │
│  saveAndFlush(entity1) → persist + FLUSH                  │
│    └── INSERT entity1       ← SQL sent NOW               │
│    └── entity1.getId() = 42 ← DB-generated ID available │
│                                                           │
│  ... use entity1.getId() ...                              │
│                                                           │
│  Transaction Commit → permanent                           │
└──────────────────────────────────────────────────────────┘
```

**When does JPA flush automatically?** (1) Before transaction commit. (2) Before a JPQL/HQL query that touches the same entity type (to ensure consistency). (3) When you explicitly call `flush()`. **Flush ≠ Commit**: flush sends SQL to the DB but the transaction can still be rolled back. Commit makes it permanent. **Batching**: `save()` allows Hibernate to batch multiple INSERTs into one JDBC batch call (if `spring.jpa.properties.hibernate.jdbc.batch_size=50` is set) — `saveAndFlush()` on each entity breaks batching.

### 🗣️ Answering Approach
"save() and saveAndFlush() both persist the entity, but the timing of SQL execution differs. save() adds the entity to the persistence context — the actual INSERT is deferred until flush time, which is usually right before the transaction commits. This deferral lets Hibernate batch multiple inserts together for better performance. saveAndFlush() forces the SQL to be sent to the database immediately, though it's still within the same transaction boundary — it can still be rolled back. I use saveAndFlush() in specific scenarios: when I need the database-generated ID right away for a subsequent operation, when I want to catch a unique constraint violation at a specific point in my code rather than at commit time, or when I'm mixing JPA with native SQL queries within the same transaction and need the data to be visible. In my projects, I use save() by default and configure Hibernate batch inserts for bulk operations."

### 💻 Code Example

```java
// ✅ save() — default, deferred flush (preferred)
@Transactional
public Order createOrder(OrderRequest request) {
    Order order = new Order(request.getCustomerId(), request.getItems());
    orderRepository.save(order);  // entity managed, SQL deferred

    // ⚠️ order.getId() might be null here if using IDENTITY strategy!
    // (available for SEQUENCE strategy since IDs are pre-fetched)

    auditService.logCreation("Order created");  // more logic
    // SQL INSERT happens at commit time — Hibernate may batch it
    return order;
}

// ✅ saveAndFlush() — when you need ID immediately
@Transactional
public Order createOrderWithDetails(OrderRequest request) {
    Order order = new Order(request.getCustomerId());
    orderRepository.saveAndFlush(order);  // SQL sent NOW

    // ✅ order.getId() is guaranteed available
    for (ItemRequest item : request.getItems()) {
        OrderItem oi = new OrderItem(order.getId(), item.getProductId(), item.getQty());
        orderItemRepository.save(oi);
    }
    return order;
}

// ✅ saveAndFlush() — catch constraint violation at specific point
@Transactional
public User registerUser(UserRequest request) {
    User user = new User(request.getEmail(), request.getName());
    try {
        userRepository.saveAndFlush(user);  // ⭐ constraint check happens HERE
    } catch (DataIntegrityViolationException e) {
        throw new DuplicateEmailException("Email already registered: " + request.getEmail());
    }
    // If we used save(), the exception would be thrown at commit time
    // — harder to map to a clean business exception
    
    sendWelcomeEmail(user);
    return user;
}

// ✅ Batch insert — save() enables batching, saveAndFlush() breaks it
@Transactional
public void bulkImport(List<ProductDTO> products) {
    for (int i = 0; i < products.size(); i++) {
        Product p = new Product(products.get(i));
        productRepository.save(p);  // ⭐ deferred — allows JDBC batching

        if (i % 50 == 0) {
            entityManager.flush();   // flush every 50 — prevent huge persistence context
            entityManager.clear();   // clear first-level cache — prevent OOM
        }
    }
}
```

```properties
# Enable Hibernate batching (only works with save(), not saveAndFlush per entity)
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
```

### ⚠️ Common Pitfalls
- **Using saveAndFlush() everywhere** — breaks Hibernate batching → performance hit for bulk operations
- **Expecting save() to return DB-generated ID immediately** — with `IDENTITY` strategy, the INSERT must execute to get the ID; Hibernate does this under the hood but it disables batch inserts
- **Confusing flush with commit** — `flush()` sends SQL but doesn't commit → transaction can still rollback
- **Large batch without clear()** — `save()` in a loop fills the persistence context (first-level cache) → OOM → flush + clear periodically
- **saveAndFlush() outside @Transactional** — each call is auto-committed → no rollback possible for multi-step operations

### 🆚 Comparison Table

| Aspect | save() | saveAndFlush() |
|--------|--------|----------------|
| SQL execution | Deferred (at flush/commit) | Immediate |
| JDBC batching | ✅ Supported | ❌ Breaks batching |
| DB-generated ID | May not be available immediately (IDENTITY) | ✅ Guaranteed available |
| Constraint check | At commit time | ✅ At call time |
| Performance | Better (batching, fewer round-trips) | Extra DB round-trip |
| Use case | **Default choice (99%)** | Need ID / need early constraint check |
| Transaction | Required for deferred flush | Required for rollback safety |

### 🎯 Tricky Follow-up Questions
- **Q**: What if I call `save()` and then run a JPQL query — is the saved entity visible?  
  **A**: Yes — Hibernate auto-flushes before JPQL queries that touch the same entity type (FlushMode.AUTO). But native SQL queries do NOT trigger auto-flush — use `saveAndFlush()` or manual `flush()`.
- **Q**: Does `saveAndFlush()` commit the transaction?  
  **A**: No — it only sends the SQL. The transaction commits when the `@Transactional` method finishes. You can still rollback after a flush.

### ⚡ Remember (Quick Recall)
- `save()` = deferred, batch-friendly, use by default
- `saveAndFlush()` = immediate SQL, use for ID or constraint check
- **Flush ≠ Commit** — flush sends SQL, commit makes permanent
- Bulk: `save()` + periodic `flush()` + `clear()` → prevent OOM
- `hibernate.jdbc.batch_size=50` + `order_inserts=true` for batch performance
- IDENTITY strategy forces per-entity INSERT (can't batch)

### 🔗 Follow-up Topics
- [Q11 in database/01 → Lazy vs Eager loading](01-jpa-sql-transactions.md#q11)
- [Q9 in database/02 → Hibernate query optimization](02-hibernate-queries-cursors.md#q9)
- JPA ID generation strategies: IDENTITY vs SEQUENCE vs TABLE
