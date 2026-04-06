# 🗄️ JPA, SQL & Transactions (Q11–Q14)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q11"></a>
## Q11. What is lazy vs eager loading in JPA? In which scenarios can it impact performance?

### 📝 One-Liner
Lazy loading fetches related entities only when accessed (default for collections); eager loading fetches everything immediately — lazy causes N+1 problem, eager causes unnecessary data loading.

### 🔑 Quick Answer
**Lazy loading** (default for @OneToMany, @ManyToMany): related entities are NOT loaded from DB until you actually access them. JPA returns a **proxy/wrapper** that triggers a DB query on first access. **Eager loading** (default for @ManyToOne, @OneToOne): related entities are loaded immediately WITH the parent query via JOIN. **Performance impact**: Lazy → **N+1 problem** (1 query for parents + N queries for each parent's children). Eager → **unnecessary data** (loads child data even when you don't need it, plus Cartesian products with multiple eager collections). Fix N+1 with **JOIN FETCH** or **@EntityGraph** for specific queries. *(Lazy = zaroorat padne pe DB call, Eager = pehle se sab load — dono mein performance issue ho sakti hai)*

### 📖 How It Works
```
Lazy Loading:
  Order order = orderRepo.findById(1L);
  // SQL: SELECT * FROM orders WHERE id = 1
  // order.items is a PROXY (not loaded yet)

  order.getItems().size();  // ← ACCESS triggers actual query!
  // SQL: SELECT * FROM order_items WHERE order_id = 1

N+1 Problem:
  List<Order> orders = orderRepo.findAll();
  // SQL 1: SELECT * FROM orders  → returns 100 orders

  for (Order o : orders) {
    o.getItems().size();
    // SQL 2: SELECT * FROM order_items WHERE order_id = 1
    // SQL 3: SELECT * FROM order_items WHERE order_id = 2
    // ... SQL 101: SELECT * FROM order_items WHERE order_id = 100
  }
  // TOTAL: 101 queries! (1 + N where N=100)

Eager Loading:
  Order order = orderRepo.findById(1L);
  // SQL: SELECT o.*, i.* FROM orders o
  //      LEFT JOIN order_items i ON o.id = i.order_id
  //      WHERE o.id = 1
  // Everything loaded in ONE query

  order.getItems().size();  // already in memory, no query

JOIN FETCH (fix N+1):
  @Query("SELECT o FROM Order o JOIN FETCH o.items")
  List<Order> findAllWithItems();
  // SQL: SELECT o.*, i.* FROM orders o
  //      LEFT JOIN order_items i ON o.id = i.order_id
  // ONE query loads everything!
```

### 🗣️ Answering Approach
"Lazy loading defers fetching related entities until they're accessed — JPA returns a proxy that triggers a database query on first touch. Eager loading fetches everything immediately via JOIN. The key performance issue with lazy loading is the N+1 problem: loading 100 orders plus accessing each order's items generates 101 queries instead of one. The fix is using JOIN FETCH in JPQL or @EntityGraph on repository methods, which tells JPA to load the relationship in a single query only when needed. I keep the default lazy loading on entities and selectively use JOIN FETCH in specific queries where I know I need the associated data. In my project, fixing an N+1 with JOIN FETCH reduced a page load query from 230ms to 8ms."

### 💻 Code
```java
// Entity with lazy and eager defaults
@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;
    
    @ManyToOne(fetch = FetchType.EAGER)   // default for @ManyToOne
    private Customer customer;             // loaded immediately
    
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)  // default for @OneToMany
    private List<OrderItem> items;          // loaded on access (proxy)
}

// N+1 PROBLEM — BAD
public List<OrderDTO> getAllOrders() {
    List<Order> orders = orderRepo.findAll();     // 1 query
    return orders.stream()
        .map(o -> new OrderDTO(o.getId(), o.getItems().size()))  // N queries!
        .toList();
}

// FIX 1: JOIN FETCH (JPQL)
@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items")
List<Order> findAllWithItems();  // 1 query!

// FIX 2: @EntityGraph (declarative)
@EntityGraph(attributePaths = {"items", "customer"})
@Query("SELECT o FROM Order o")
List<Order> findAllWithItemsAndCustomer();  // 1 query!

// FIX 3: DTO Projection (best for read-only)
@Query("SELECT new com.app.dto.OrderSummary(o.id, o.amount, c.name) " +
       "FROM Order o JOIN o.customer c")
List<OrderSummary> findOrderSummaries();  // no entity loading, no N+1

// FIX 4: Batch fetching (@BatchSize)
@Entity
public class Order {
    @OneToMany(mappedBy = "order")
    @BatchSize(size = 50)  // loads 50 orders' items at once instead of 1-by-1
    private List<OrderItem> items;
}
// Instead of 100 individual queries → 2 queries (100/50 batches)
```

### ⚠️ Pitfalls / Gotchas
- **LazyInitializationException** — accessing lazy collection outside transaction (session closed) *(Transaction ke bahar lazy collection access kiya toh exception aayega)*
- **Eager on @OneToMany** with multiple collections → Cartesian product explosion (MultipleBagFetchException)
- JOIN FETCH paginates in MEMORY (not SQL) — dangerous for large datasets → use @BatchSize instead
- `open-in-view=true` (default!) keeps session open through HTTP request — hides N+1 issues until production
- DTO projections avoid all these issues — best for read-only queries

### 🆚 vs. Comparison
| Aspect | Lazy | Eager |
|--------|------|-------|
| Default for | @OneToMany, @ManyToMany | @ManyToOne, @OneToOne |
| DB query | On first access | Immediately with parent |
| Risk | N+1 queries | Unnecessary data, Cartesian |
| Fix | JOIN FETCH, @EntityGraph | Don't use for collections |
| Best practice | Keep lazy ⭐ + JOIN FETCH | Only for @ManyToOne |

### 🎯 Tricky Interview Qs

**Q: How do you detect N+1 queries in production?**
Enable Hibernate statistics (`spring.jpa.properties.hibernate.generate_statistics=true`) or use log `org.hibernate.SQL=DEBUG`. Look for repeated similar queries. Tools: Hibernate metrics, p6spy, datasource-proxy. *(Hibernate ka SQL log on karo — ek jaisi queries baar baar dikh rahi hain toh N+1 hai)*

**Q: What is `spring.jpa.open-in-view` and why should it be false?**
It keeps the Hibernate session open for the entire HTTP request. Lazy loading "works" from controllers but it hides N+1 problems and can cause DB connection pool exhaustion. Set to `false` in production.

### ⚡ Remember
- **Lazy = proxy** (query on access), **Eager = JOIN** (query on load)
- N+1 = 1 + N lazy queries → fix with **JOIN FETCH** or **@EntityGraph** *(N+1 = performance killer — JOIN FETCH se thik karo)*
- Keep entities lazy, use JOIN FETCH in specific queries
- Best for read-only: DTO projections (no entity, no N+1, no lazy)
- `open-in-view=false` in production

### 🔗 Follow-ups
- [Q12 → Concurrent database updates](#q12)
- [Q14 → SQL query optimization](#q14)
- [Q8 → @Transactional (session scope)](#q8)

---

<a id="q12"></a>
## Q12. How do you manage concurrent updates to the same database record?

### 📝 One-Liner
Use optimistic locking (@Version field — last writer detected and rejected) for low-contention scenarios, or pessimistic locking (SELECT FOR UPDATE) for high-contention scenarios.

### 🔑 Quick Answer
**Optimistic locking** (most common): add a `@Version` field to your entity. On update, JPA adds `WHERE version = ?` to the UPDATE statement. If another transaction changed the record (version mismatch), it throws `OptimisticLockException` → your code catches it, retries or notifies user. No DB lock held — best for low contention. **Pessimistic locking**: `SELECT ... FOR UPDATE` — acquires a row-level DB lock. Other transactions wait until the lock is released. Use for high-contention scenarios (inventory counters, bank balances). *(Optimistic = version check karo, conflict toh exception; Pessimistic = pehle se lock karo, wait karo)*

### 📖 How It Works
```
Optimistic Locking (@Version):

  User A reads: {id: 1, balance: 1000, version: 5}
  User B reads: {id: 1, balance: 1000, version: 5}

  User A updates: UPDATE accounts SET balance=900, version=6 WHERE id=1 AND version=5
  → Success! (version was 5) ✅ version is now 6

  User B updates: UPDATE accounts SET balance=800, version=6 WHERE id=1 AND version=5
  → FAIL! (version is now 6, not 5) → OptimisticLockException ❌
  → User B must retry: re-read (version=6, balance=900), re-apply logic

Pessimistic Locking (SELECT FOR UPDATE):

  User A: SELECT * FROM accounts WHERE id=1 FOR UPDATE
  → Row LOCKED! 🔒 User A works with the row...

  User B: SELECT * FROM accounts WHERE id=1 FOR UPDATE
  → BLOCKED! ⏳ Waiting for User A to commit/rollback...

  User A: UPDATE accounts SET balance=900 WHERE id=1 → COMMIT
  → Lock released 🔓

  User B: (now gets the row) → balance=900 → proceeds

When to use which:
  Optimistic:  Read-heavy, low contention, web forms → most apps ⭐
  Pessimistic: Write-heavy, high contention, inventory → critical sections
```

### 🗣️ Answering Approach
"I use optimistic locking as the default strategy — adding a @Version field to the entity. JPA automatically adds a version check to every UPDATE statement. If two users read the same record and both try to update, the second one gets an OptimisticLockException because the version no longer matches. The application catches this and either retries automatically or notifies the user. For high-contention scenarios like inventory management where multiple threads decrement stock simultaneously, I use pessimistic locking with @Lock(PESSIMISTIC_WRITE) on the repository method — this acquires a row-level lock preventing concurrent access. In my project, we used optimistic locking for order updates — where conflicts are rare — and pessimistic locking for the stock count decrement during checkout."

### 💻 Code
```java
// OPTIMISTIC LOCKING — @Version field
@Entity
public class Account {
    @Id @GeneratedValue
    private Long id;
    private double balance;
    
    @Version  // ← Optimistic lock! JPA checks this on every update
    private int version;
}

// JPA generates: UPDATE accounts SET balance=?, version=version+1
//                WHERE id=? AND version=?
// If version mismatch → 0 rows updated → OptimisticLockException

// Service with retry logic for optimistic lock
@Service
public class AccountService {
    @Autowired private AccountRepository accountRepo;
    
    @Retryable(value = OptimisticLockException.class, maxAttempts = 3)
    @Transactional
    public void debit(Long accountId, double amount) {
        Account account = accountRepo.findById(accountId).orElseThrow();
        if (account.getBalance() < amount) throw new InsufficientFundsException();
        account.setBalance(account.getBalance() - amount);
        accountRepo.save(account);
        // If OptimisticLockException → @Retryable auto-retries (re-reads, re-debits)
    }
}

// PESSIMISTIC LOCKING — SELECT FOR UPDATE
public interface ProductRepository extends JpaRepository<Product, Long> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)  // SELECT ... FOR UPDATE
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);
}

@Service
public class InventoryService {
    @Transactional
    public void decrementStock(Long productId, int quantity) {
        // Row locked! Other threads wait here until this tx commits
        Product product = productRepo.findByIdForUpdate(productId)
                .orElseThrow();
        if (product.getStock() < quantity) throw new OutOfStockException();
        product.setStock(product.getStock() - quantity);
        productRepo.save(product);
    }  // commit → lock released
}

// ATOMIC SQL UPDATE — no lock needed (best for simple counters)
@Modifying
@Query("UPDATE Product p SET p.stock = p.stock - :qty WHERE p.id = :id AND p.stock >= :qty")
int decrementStock(@Param("id") Long id, @Param("qty") int qty);
// Returns 1 if updated, 0 if insufficient stock → check return value
```

### ⚠️ Pitfalls / Gotchas
- **Optimistic lock + no retry** → user sees error but data is correct. Always add retry or notification *(retry nahi lagaya toh user ko sirf error dikhega)*
- **Pessimistic lock** can cause **deadlocks** if two transactions lock rows in different order
- **Pessimistic lock timeout**: set `@QueryHints(@QueryHint(name="javax.persistence.lock.timeout", value="5000"))` to avoid infinite waits
- **Detached entity update**: if you read-modify-save outside transaction boundaries, version check may not work correctly
- **Atomic SQL UPDATE** is the simplest for counters — no lock, no version, just `WHERE stock >= qty`

### 🆚 vs. Comparison
| Aspect | Optimistic (@Version) | Pessimistic (FOR UPDATE) | Atomic SQL |
|--------|----------------------|-------------------------|-----------|
| Lock type | No DB lock | Row-level lock | No lock |
| Conflict handling | Exception + retry | Wait in queue | Return value check |
| Concurrency | High (no blocking) | Low (blocking) | Highest |
| Contention | Low | High | Any |
| Deadlock risk | None | Possible | None |
| Use case | Web forms, CRUD | Inventory, banking | Simple counters |
| Complexity | Medium | Medium | Low ⭐ |

### 🎯 Tricky Interview Qs

**Q: Can optimistic locking cause starvation?**
In theory, if there's very high contention, the same transaction could keep retrying and always losing to others. In practice, with low-to-moderate contention, 3 retries is sufficient. For high contention → switch to pessimistic.

**Q: What is the lost update problem?**
Two users read balance=1000. User A deducts 100→900. User B deducts 200→800. User B's write overwrites A's → balance=800 instead of 700. @Version prevents this. *(Dono ne 1000 padha, ek ka update dusre ne overwrite kiya — @Version ye prevent karta hai)*

### ⚡ Remember
- **Optimistic** = @Version field, exception on conflict, retry → best default *(Version check = safe update)*
- **Pessimistic** = FOR UPDATE, row lock, wait → high contention only
- **Atomic SQL** = `UPDATE ... SET qty = qty - :n WHERE qty >= :n` → simplest for counters
- Optimistic: no DB lock, good concurrency, retry needed
- Pessimistic: DB lock, blocks others, deadlock possible

### 🔗 Follow-ups
- [Q13 → ACID properties](#q13)
- [Q8 → @Transactional (isolation levels)](#q8)
- [Q14 → SQL optimization](#q14)

---

<a id="q13"></a>
## Q13. Explain ACID properties using a banking transaction example.

### 📝 One-Liner
ACID: Atomicity (all-or-nothing), Consistency (valid state only), Isolation (concurrent txns don't interfere), Durability (committed = permanent) — a bank transfer either completes fully or not at all.

### 🔑 Quick Answer
Bank transfer: ₹5000 from Account A (balance ₹10000) to Account B (balance ₹3000). **Atomicity**: BOTH debit A AND credit B happen, or NEITHER happens — no half-transfer. **Consistency**: total money before (₹13000) = total after (₹13000). A's balance can't go negative if constraint exists. **Isolation**: if another transaction checks A's balance mid-transfer, it sees the old balance (₹10000) — not partially deducted. **Durability**: once committed, even if the server crashes 1ms later, the transfer is permanent — written to disk/WAL. *(ACID = ya toh poora hoga, ya kuch nahi hoga — beech mein nahi rukega)*

### 📖 How It Works
```
Bank Transfer: A(₹10000) → ₹5000 → B(₹3000)

ATOMICITY (all or nothing):
  BEGIN TRANSACTION
    UPDATE accounts SET balance = balance - 5000 WHERE id = 'A';  ✅
    UPDATE accounts SET balance = balance + 5000 WHERE id = 'B';  ❌ FAILS!
  ROLLBACK → A still has ₹10000, B still has ₹3000
  → No money lost or created!

  Without atomicity:
    A debited → ₹5000 ✅
    B credit fails ❌
    → ₹5000 vanished! Money lost!

CONSISTENCY (valid state transitions only):
  Before: A=₹10000, B=₹3000, Total=₹13000
  After:  A=₹5000,  B=₹8000, Total=₹13000 ✅
  
  Constraint: balance >= 0
  Transfer ₹15000 from A? → violates constraint → REJECTED
  → Database moves from one valid state to another

ISOLATION (concurrent txns don't see each other's partial work):
  Tx1: Transfer A→B (₹5000)    Tx2: Check A's balance
  ────────────────────────       ────────────────────
  A -= 5000 (now 5000 in Tx1)
                                  SELECT balance FROM A
                                  → sees ₹10000 (not ₹5000!)
                                  → Tx1 hasn't committed yet
  B += 5000
  COMMIT ✅
                                  SELECT balance FROM A
                                  → now sees ₹5000 ✅

DURABILITY (committed = permanent):
  COMMIT executed ✅
  → Written to WAL (Write-Ahead Log) on disk
  → Server crashes!⚡
  → Server restarts → reads WAL → transfer is intact ✅
```

### 🗣️ Answering Approach
"ACID ensures reliable transactions. In a bank transfer of 5000 rupees from A to B — Atomicity guarantees both the debit and credit happen together or neither happens; if the credit fails, the debit is rolled back. Consistency ensures the total money in the system remains unchanged — database constraints like minimum balance are enforced. Isolation means concurrent transactions don't see each other's uncommitted changes — if someone checks A's balance mid-transfer, they see the original amount until the transfer commits. Durability means once committed, the data survives crashes — databases achieve this through Write-Ahead Logging, flushing to disk before reporting success. In my project, we rely on ACID for payment processing — @Transactional on our transfer method ensures atomicity, and we use READ_COMMITTED isolation to prevent dirty reads."

### 💻 Code
```java
// Bank transfer with ACID via @Transactional
@Service
@RequiredArgsConstructor
public class TransferService {
    private final AccountRepository accountRepo;
    
    @Transactional(isolation = Isolation.READ_COMMITTED)
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        // ATOMICITY: if any step fails, everything rolls back
        Account from = accountRepo.findByIdForUpdate(fromId)  // pessimistic lock
                .orElseThrow(() -> new AccountNotFoundException(fromId));
        Account to = accountRepo.findByIdForUpdate(toId)
                .orElseThrow(() -> new AccountNotFoundException(toId));
        
        // CONSISTENCY: enforce business rules
        if (from.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException("Balance too low");
            // RuntimeException → @Transactional rolls back (ATOMICITY)
        }
        
        from.setBalance(from.getBalance().subtract(amount));
        to.setBalance(to.getBalance().add(amount));
        
        accountRepo.save(from);
        accountRepo.save(to);
        // COMMIT → DURABILITY: data written to WAL → survives crash
        // ISOLATION: other transactions see old balances until this commits
    }
}

// Spring transaction isolation levels
@Transactional(isolation = Isolation.READ_UNCOMMITTED)  // dirty reads allowed
@Transactional(isolation = Isolation.READ_COMMITTED)     // no dirty reads (default)
@Transactional(isolation = Isolation.REPEATABLE_READ)    // no non-repeatable reads
@Transactional(isolation = Isolation.SERIALIZABLE)       // no phantom reads (slowest)
```

### ⚠️ Pitfalls / Gotchas
- **@Transactional only rolls back on RuntimeException** by default! Checked exceptions commit → use `rollbackFor = Exception.class` *(checked exception pe commit ho jaata hai — rollbackFor lagao)*
- Atomicity doesn't cover external systems — if you called an API + DB update, API call can't be rolled back
- Higher isolation = more correctness but slower (more locking)
- Durability depends on DB config — fsync disabled = data loss risk on crash
- Distributed transactions (2 databases) need 2PC or Saga pattern — standard @Transactional won't work

### 🆚 vs. Comparison
| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|----------------|-----------|-------------------|-------------|------------|
| READ_UNCOMMITTED | ✅ Yes | ✅ Yes | ✅ Yes | Fastest |
| READ_COMMITTED | ❌ No | ✅ Yes | ✅ Yes | Fast ⭐ |
| REPEATABLE_READ | ❌ No | ❌ No | ✅ Yes | Medium |
| SERIALIZABLE | ❌ No | ❌ No | ❌ No | Slowest |

### 🎯 Tricky Interview Qs

**Q: What is a dirty read?**
Reading data written by another transaction that hasn't committed yet. If that transaction rolls back, you read "phantom" data that never existed. READ_COMMITTED prevents this.

**Q: How does atomicity work at the database level?**
Databases use Write-Ahead Logging (WAL/redo log). Changes are written to WAL first. On COMMIT, WAL is flushed to disk. On ROLLBACK, undo log reverses changes. On crash, WAL replays committed changes.

### ⚡ Remember
- **A**tomicity = all or nothing *(ya poora, ya kuch nahi)*
- **C**onsistency = valid state to valid state
- **I**solation = concurrent txns don't see partial work
- **D**urability = committed = permanent (WAL)
- Default isolation: READ_COMMITTED (most databases)
- @Transactional provides A, Spring + DB provides C/I/D

### 🔗 Follow-ups
- [Q12 → Concurrent updates (locking)](#q12)
- [Q8 → @Transactional internals](#q8)
- [Q14 → SQL optimization](#q14)

---

<a id="q14"></a>
## Q14. How would you optimize a slow-performing SQL query?

### 📝 One-Liner
Analyze with EXPLAIN/EXPLAIN ANALYZE → add missing indexes → rewrite query (avoid SELECT *, reduce JOINs) → check for N+1 → consider caching/pagination.

### 🔑 Quick Answer
Systematic approach: **(1) EXPLAIN ANALYZE** — see actual execution plan, find Seq Scans (missing index), Nested Loops (bad joins), Sort (missing sort index). **(2) Add indexes** on WHERE, JOIN, ORDER BY columns. Composite indexes for multi-column filters. **(3) Query rewrite** — avoid `SELECT *` (fetch only needed columns), use EXISTS instead of IN for subqueries, avoid functions in WHERE (nullifies index). **(4) N+1 detection** — check for repeated similar queries from ORM. **(5) Pagination** — LIMIT/OFFSET or keyset pagination for large results. **(6) Caching** — Redis/Caffeine for frequently-read, rarely-changed data. *(EXPLAIN se dekho kahaan slow hai — phir index, query fix, cache lagao)*

### 📖 How It Works
```
Step-by-Step Optimization:

Step 1: EXPLAIN ANALYZE (find the bottleneck)
  EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 123 ORDER BY created_at DESC;
  
  → Seq Scan on orders (cost=0..15000, actual time=850ms)  ← NO INDEX!
  → Sort (actual time=200ms)
  
  Problem: Sequential scan of 1M rows + in-memory sort

Step 2: Add index
  CREATE INDEX idx_orders_customer_date ON orders(customer_id, created_at DESC);
  
  → Index Scan using idx_orders_customer_date (actual time=2ms)  ← 425× faster!

Step 3: Query rewrites
  BAD:  SELECT * FROM orders WHERE YEAR(created_at) = 2024
        → function on column kills index usage!
  GOOD: SELECT * FROM orders WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
        → index used ✅

  BAD:  SELECT * FROM orders WHERE status IN (SELECT ...)
  GOOD: SELECT * FROM orders WHERE EXISTS (SELECT 1 FROM ...)
        → EXISTS short-circuits, IN materializes full subquery

  BAD:  SELECT o.*, c.*, i.* FROM orders o JOIN customers c ... JOIN items i ...
  GOOD: SELECT o.id, o.amount, c.name FROM orders o JOIN customers c ...
        → fetch only needed columns

Step 4: Pagination
  BAD:  SELECT * FROM orders ORDER BY id OFFSET 100000 LIMIT 20
        → DB still scans 100,020 rows!
  GOOD: SELECT * FROM orders WHERE id > :lastSeenId ORDER BY id LIMIT 20
        → Keyset pagination, O(1) with index ⭐
```

### 🗣️ Answering Approach
"My systematic approach starts with EXPLAIN ANALYZE to understand the actual execution plan. I look for sequential scans indicating missing indexes, nested loops suggesting bad join order, and in-memory sorts. The biggest wins usually come from adding proper indexes — composite indexes on columns used together in WHERE and ORDER BY. I also check for ORM-generated N+1 queries that hit the DB hundreds of times. For large datasets, I use keyset pagination instead of OFFSET, which stays constant-time at any page depth. In my project, a customer order history query was taking 850ms — adding a composite index on (customer_id, created_at) brought it down to 2ms, a 425× improvement."

### 💻 Code
```java
// JPA: Avoid SELECT * — use DTO projections
// BAD: loads full entity with all columns
List<Order> orders = orderRepo.findByCustomerId(customerId);

// GOOD: load only what you need
@Query("SELECT new com.app.dto.OrderSummary(o.id, o.amount, o.status, o.createdAt) " +
       "FROM Order o WHERE o.customerId = :cid ORDER BY o.createdAt DESC")
List<OrderSummary> findSummariesByCustomerId(@Param("cid") Long cid);

// Keyset pagination (much faster than OFFSET for deep pages)
@Query("SELECT o FROM Order o WHERE o.id > :lastId ORDER BY o.id ASC")
List<Order> findNextPage(@Param("lastId") Long lastId, Pageable pageable);

// Force index hint (if needed, DB-specific)
@Query(value = "SELECT /*+ INDEX(orders idx_orders_customer_date) */ " +
               "id, amount FROM orders WHERE customer_id = :cid", nativeQuery = true)
List<Object[]> findWithIndexHint(@Param("cid") Long cid);

// Caching for frequently-read data
@Cacheable(value = "products", key = "#id")
public Product getProduct(Long id) {
    return productRepo.findById(id).orElseThrow();
}

// application.yml — Hibernate SQL logging for finding slow queries
// spring:
//   jpa:
//     show-sql: true
//     properties:
//       hibernate:
//         format_sql: true
//         generate_statistics: true  # shows query counts and timing
//   datasource:
//     hikari:
//       leak-detection-threshold: 30000  # detect connection leaks
```

```sql
-- Essential Index Types:

-- Single column (equality + range)
CREATE INDEX idx_orders_status ON orders(status);

-- Composite (multi-column WHERE)
CREATE INDEX idx_orders_cust_date ON orders(customer_id, created_at DESC);
-- Rule: equality columns first, range/sort columns last

-- Covering index (index includes all needed columns — no table lookup)
CREATE INDEX idx_orders_covering ON orders(customer_id, created_at DESC)
    INCLUDE (amount, status);

-- Partial index (PostgreSQL — index only relevant rows)
CREATE INDEX idx_orders_pending ON orders(created_at)
    WHERE status = 'PENDING';
```

### ⚠️ Pitfalls / Gotchas
- **Functions in WHERE** kill indexes: `WHERE YEAR(date) = 2024` → Seq Scan! Use range instead *(Function lagaya toh index kaam nahi karega)*
- **Over-indexing** slows writes: every INSERT/UPDATE must update all indexes
- **OFFSET pagination** gets slower with depth: OFFSET 1000000 scans 1M rows before returning 20
- **Statistics stale**: run `ANALYZE table_name` (PostgreSQL) to update query planner stats
- **ORM can generate bad SQL**: always check generated queries with `show-sql=true`
- **Composite index order matters**: (A, B) supports WHERE A=? and WHERE A=? AND B=?, but NOT WHERE B=? alone

### 🎯 Tricky Interview Qs

**Q: Would you add an index on every column used in WHERE?**
No — indexes speed up reads but slow writes. Only index columns with high selectivity that are frequently queried. A boolean column with 50/50 distribution is not a good index candidate.

**Q: When does an index NOT help?**
Low selectivity (gender: M/F), small tables (< few thousand rows — seq scan is faster), columns wrapped in functions, leading wildcards (`LIKE '%abc'`). *(Bahut kam unique values hai toh index ka fayda nahi)*

### ⚡ Remember
- **EXPLAIN ANALYZE** first — don't guess, measure *(pehle EXPLAIN karo — andaze se optimize mat karo)*
- **Indexes** on WHERE + JOIN + ORDER BY columns (composite: equality first, range last)
- **No functions in WHERE** (kills index)
- **Keyset pagination** > OFFSET for deep pages
- **DTO projections** > SELECT * (less data = faster)
- **Cache** for read-heavy, rarely-changed data

### 🔗 Follow-ups
- [Q11 → JPA lazy/eager loading (N+1 optimization)](#q11)
- [Q15 → SQL aggregate queries with date filtering](#q15)
- [Q13 → ACID properties (isolation levels affect query performance)](#q13)

---

<a id="q15"></a>
## Q15. Write a SQL query to find total quantity sold per product in the last 30 days

### 📝 One-Liner
`GROUP BY product` with `SUM(quantity)` and a `WHERE` clause filtering orders from the last 30 days — this tests aggregate functions, date filtering, JOINs, and indexing awareness.

### 🔑 Quick Answer
**Core query**: `SELECT p.name, SUM(oi.quantity) FROM order_items oi JOIN products p ON oi.product_id = p.id JOIN orders o ON oi.order_id = o.id WHERE o.order_date >= CURRENT_DATE - INTERVAL '30' DAY GROUP BY p.name ORDER BY SUM(oi.quantity) DESC`. **Key decisions**: use `>=` not `BETWEEN` (avoids off-by-one with timestamps), index on `order_date` for performance, handle products with zero sales using `LEFT JOIN`, use `COALESCE` for null safety. *(GROUP BY + SUM = basic aggregation; WHERE date filter + index = production-ready)*

### 📖 How It Works
```
Tables:
  products:    id | name
  orders:      id | order_date | customer_id
  order_items: id | order_id | product_id | quantity | price

Execution Order (SQL logical processing):
  1. FROM + JOIN    → combine tables
  2. WHERE          → filter last 30 days
  3. GROUP BY       → group by product
  4. HAVING         → (optional) filter groups
  5. SELECT         → pick columns + SUM()
  6. ORDER BY       → sort results
  7. LIMIT/OFFSET   → paginate
```

### 🗣️ Answering Approach
"I'd write a query that joins products, order_items, and orders — filtering orders where order_date is within the last 30 days. I use SUM(quantity) grouped by product name. For the date filter, I prefer `order_date >= CURRENT_DATE - INTERVAL '30' DAY` over BETWEEN because BETWEEN with timestamps can accidentally include or exclude boundary records. I'd add an index on `orders.order_date` since this is a range scan on a potentially large table. If the interviewer asks about products with zero sales, I'd switch to a LEFT JOIN from products to order_items so those products appear with a 0 total. In a Spring Boot app, this would be a `@Query` annotation on the repository with a DTO projection for performance."

### 💻 Code Example

```sql
-- ✅ Basic: total quantity per product in last 30 days
SELECT p.name AS product_name,
       SUM(oi.quantity) AS total_quantity
FROM   order_items oi
JOIN   orders o    ON oi.order_id = o.id
JOIN   products p  ON oi.product_id = p.id
WHERE  o.order_date >= CURRENT_DATE - INTERVAL '30' DAY
GROUP BY p.name
ORDER BY total_quantity DESC;

-- ✅ Include products with zero sales (LEFT JOIN)
SELECT p.name AS product_name,
       COALESCE(SUM(oi.quantity), 0) AS total_quantity
FROM   products p
LEFT JOIN order_items oi ON p.id = oi.product_id
LEFT JOIN orders o       ON oi.order_id = o.id
                         AND o.order_date >= CURRENT_DATE - INTERVAL '30' DAY
GROUP BY p.name
ORDER BY total_quantity DESC;

-- ✅ With HAVING (only products selling > 100 units)
SELECT p.name, SUM(oi.quantity) AS total_qty
FROM   order_items oi
JOIN   orders o   ON oi.order_id = o.id
JOIN   products p ON oi.product_id = p.id
WHERE  o.order_date >= CURRENT_DATE - INTERVAL '30' DAY
GROUP BY p.name
HAVING SUM(oi.quantity) > 100
ORDER BY total_qty DESC;

-- ✅ Spring Data JPA — DTO projection
@Query("""
    SELECT new com.app.dto.ProductSalesDTO(p.name, SUM(oi.quantity))
    FROM OrderItem oi
    JOIN oi.order o
    JOIN oi.product p
    WHERE o.orderDate >= :since
    GROUP BY p.name
    ORDER BY SUM(oi.quantity) DESC
    """)
List<ProductSalesDTO> findTopSellingProducts(@Param("since") LocalDate since);

-- ✅ Index for performance
CREATE INDEX idx_orders_order_date ON orders(order_date);
CREATE INDEX idx_order_items_product ON order_items(product_id, order_id);
```

### ⚠️ Pitfalls / Gotchas
- **BETWEEN with timestamps**: `BETWEEN '2026-03-01' AND '2026-03-31'` misses records at `2026-03-31 14:00:00` — use `< '2026-04-01'` instead *(BETWEEN timestamp ke saath tricky hai — boundary miss ho sakta hai)*
- **Missing index on date column** — full table scan on large orders table
- **NULL in SUM** — `SUM(NULL)` returns NULL, not 0; wrap with `COALESCE`
- **GROUP BY non-aggregated column** — MySQL allows it (silently picks random value), PostgreSQL rejects it — always GROUP BY what you SELECT

### 🆚 WHERE vs HAVING

| Clause | Filters | Runs | Use For |
|--------|---------|------|---------|
| **WHERE** | Individual rows | Before GROUP BY | `order_date >= ...` |
| **HAVING** | Grouped results | After GROUP BY | `SUM(qty) > 100` |

### 🎯 Tricky Interview Qs

**Q: What if you need daily breakdown, not just total?**
Add `o.order_date::date` (or `DATE(o.order_date)`) to both SELECT and GROUP BY.

**Q: How would you optimize this for millions of rows?**
Partition orders table by date range. Use covering index `(order_date, id)`. Materialize with a daily cron job into a summary table for dashboard queries.

### ⚡ Remember
- **JOIN → WHERE → GROUP BY → HAVING → SELECT → ORDER BY** (logical execution order)
- `COALESCE(SUM(...), 0)` for null safety with LEFT JOINs
- Index the date column + use `>=` over `BETWEEN` for timestamps
