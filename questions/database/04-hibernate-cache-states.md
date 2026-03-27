# 🗄️ Hibernate/JPA — Caching Levels & Entity States (Q1–Q2)

> **Source**: Capgemini Java Developer Interview (4+ years)  
> **Coverage**: First/Second Level Cache, JPA entity lifecycle states

---

<a id="q1"></a>
## Q1. What is First Level Cache and Second Level Cache in Hibernate?

### 📝 One-Liner
**First Level Cache** (L1) = per-`EntityManager`/Session, always ON, not shared; **Second Level Cache** (L2) = per-`SessionFactory`, shared across sessions, optional (Ehcache/Hazelcast/Infinispan).

### 🔑 Quick Answer
**L1 Cache** — built into every Hibernate Session (= JPA EntityManager). When you load entity with `findById(1)`, it's stored in the Session. Subsequent `findById(1)` in SAME session → returns cached object, NO SQL. Cleared when session closes (end of `@Transactional` method). **Cannot be disabled**. **L2 Cache** — shared across sessions/transactions. Stores entity data at the `SessionFactory` level. When Session A loads User#1, it goes to L2 cache. Session B requesting User#1 → cache hit, no DB query. Requires configuration + cache provider (Ehcache, Hazelcast, Infinispan). **Query Cache** — caches query results (separate from entity cache). *(L1 = session ke andar cache — automatic; L2 = sessions ke beech shared cache — configure karna padta hai)*

### 📖 How It Works (Detailed Explanation)

```
L1 Cache (per Session/EntityManager):
┌─────────────────────────────────┐
│  @Transactional method          │
│  Session / EntityManager        │
│  ┌─────────────────────────┐    │
│  │ L1 Cache (Persistence   │    │
│  │ Context)                │    │
│  │ User#1 → {Alice, ...}  │    │
│  │ Order#5 → {O-123, ...} │    │
│  └─────────────────────────┘    │
│                                 │
│  findById(1) → L1 hit! No SQL  │
│  Session closed → L1 cleared   │
└─────────────────────────────────┘

L2 Cache (per SessionFactory, shared):
┌─────────────────────────────────┐
│  SessionFactory                  │
│  ┌─────────────────────────┐    │
│  │ L2 Cache (Ehcache/      │    │
│  │ Hazelcast/Infinispan)   │    │
│  │ User#1 → {Alice, ...}  │    │
│  │ Product#42 → {Laptop}  │    │
│  └─────────────────────────┘    │
│                                 │
│  Session A: findById(1) → DB   │──→ stored in L2
│  Session B: findById(1) → L2!  │──→ no DB query
└─────────────────────────────────┘

Lookup order:
  findById(1) → L1 → L2 → Database
                 ↓      ↓       ↓
              fastest  fast    slow
```

**L1 Cache = Persistence Context**: same as the managed entities in the JPA persistence context. Dirty checking happens here — Hibernate compares snapshot at flush time. **L2 Cache stores dehydrated data** (not full objects) — entity field values, not the entity object itself. When read from L2, Hibernate creates a new entity instance and populates it. **Cache regions**: each entity class has its own cache region. Configure TTL, max entries, eviction per region. **Query Cache**: stores query → result-ID mappings. When the query runs again with same params → returns cached entity IDs → fetches entities from L2 (not DB). Useful for read-heavy, rarely-changing data.

### 🗣️ Answering Approach
"Hibernate has two cache levels. The first-level cache is the persistence context within each EntityManager session — it's always active and can't be disabled. When I load an entity, it's stored in the L1 cache, and any subsequent load within the same transaction returns the cached instance without a database hit. This cache is cleared when the session closes. The second-level cache is shared across sessions at the SessionFactory level. When one session loads an entity, subsequent sessions can find it in the L2 cache without hitting the database. I configure it with providers like Ehcache or Hazelcast, enabling it selectively on entities annotated with @Cacheable. It's great for read-heavy, relatively static data like product catalogs or configuration lookups. I also use the query cache for frequently executed queries with the same parameters."

### 💻 Code Example

```java
// ✅ L1 Cache — automatic, within a session
@Transactional
public void demonstrateL1Cache() {
    User user1 = userRepo.findById(1L).orElseThrow();  // SQL: SELECT * FROM users WHERE id=1
    User user2 = userRepo.findById(1L).orElseThrow();  // ⭐ NO SQL — L1 cache hit!
    System.out.println(user1 == user2);                  // true — SAME object reference
}

// ✅ L2 Cache — configuration
// 1. Add dependency
// <dependency>
//     <groupId>org.hibernate.orm</groupId>
//     <artifactId>hibernate-jcache</artifactId>
// </dependency>
// <dependency>
//     <groupId>org.ehcache</groupId>
//     <artifactId>ehcache</artifactId>
// </dependency>

// 2. Enable in application.properties
// spring.jpa.properties.hibernate.cache.use_second_level_cache=true
// spring.jpa.properties.hibernate.cache.region.factory_class=jcache
// spring.jpa.properties.javax.cache.provider=org.ehcache.jsr107.EhcacheCachingProvider
// spring.jpa.properties.hibernate.cache.use_query_cache=true

// 3. Annotate entity
@Entity
@Cacheable                          // JPA standard
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)  // Hibernate-specific
public class Product {
    @Id
    private Long id;
    private String name;
    private BigDecimal price;

    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE)  // cache collection too
    @OneToMany(mappedBy = "product")
    private List<Review> reviews;
}

// 4. Usage — L2 cache in action
@Transactional
public Product loadProduct(Long id) {
    return productRepo.findById(id).orElseThrow();  // Session A: DB → L2 cache
}

@Transactional
public Product loadProductAgain(Long id) {
    return productRepo.findById(id).orElseThrow();  // Session B: L2 hit! No DB!
}

// ✅ Query Cache
@Query("SELECT p FROM Product p WHERE p.category = :cat")
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
List<Product> findByCategory(@Param("cat") String category);

// ✅ Cache concurrency strategies
// READ_ONLY        — never modified (reference data, enums)
// READ_WRITE       — read/write with soft locks (most common)
// NONSTRICT_READ_WRITE — eventual consistency (no locks, rare writes)
// TRANSACTIONAL    — JTA required, full ACID in cache
```

### ⚠️ Common Pitfalls
- **L2 cache with frequent writes** — write-heavy entities thrash the cache → only cache read-heavy data
- **Stale data** — L2 cache may serve outdated data if direct SQL bypasses Hibernate → use appropriate TTL
- **Query cache invalidation** — ANY write to the entity table invalidates ALL cached queries for that entity → only useful for rarely-changing data
- **Not caching collections** — entity is cached but `@OneToMany` collections aren't by default → annotate with `@Cache` separately
- **Memory** — L2 cache consumes heap (or off-heap with Ehcache) → configure max entries and eviction policies

### 🆚 Comparison Table

| Aspect | L1 (First Level) | L2 (Second Level) | Query Cache |
|--------|------------------|-------------------|-------------|
| Scope | Per Session/EntityManager | Per SessionFactory (shared) | Per SessionFactory |
| Enabled | **Always** (can't disable) | Optional (manual config) | Optional |
| Shared | ❌ Not shared | ✅ Across sessions | ✅ Across sessions |
| Stores | Entity objects | Dehydrated entity data | Query → entity ID mapping |
| Cleared | Session close | TTL / eviction / manual | Entity table write |
| Provider | Built-in | Ehcache, Hazelcast, Infinispan | Built-in (with L2) |
| Use case | Automatic optimization | Read-heavy, rarely-changing | Repeated same queries |

### ⚡ Remember (Quick Recall)
- **L1** = persistence context, always ON, per-session, same object reference
- **L2** = shared across sessions, optional, needs provider (Ehcache/Hazelcast)
- **Lookup order**: L1 → L2 → Database
- `@Cacheable` + `@Cache(usage = READ_WRITE)` on entity
- **Query Cache** = query + params → entity IDs → L2 cache
- Only cache **read-heavy, rarely-modified** entities

### 🔗 Follow-up Topics
- [Q11 in database/01 → Lazy vs Eager loading](01-jpa-sql-transactions.md#q11)
- [IMDG in architecture/04 → Infinispan as Hibernate L2 Cache](../languages/java/architecture/04-inmemory-grids-akka.md)
- Ehcache configuration (TTL, max entries, off-heap)

---

<a id="q2"></a>
## Q2. What are JPA entity states? Explain the entity lifecycle.

### 📝 One-Liner
JPA entities have 4 states: **Transient** (new, not managed) → **Managed** (tracked by persistence context) → **Detached** (was managed, session closed) → **Removed** (marked for deletion).

### 🔑 Quick Answer
**(1) Transient/New** — `new User()` — just a Java object, no DB row, not tracked by EntityManager. **(2) Managed/Persistent** — after `persist()` or `find()` — tracked by the persistence context. Changes are auto-detected (dirty checking) and flushed to DB. **(3) Detached** — after session closes or `detach()` — entity has an ID (was in DB) but is no longer tracked. Changes are NOT auto-saved. Use `merge()` to re-attach. **(4) Removed** — after `remove()` — marked for deletion, SQL DELETE fires at flush. *(Transient = naya object; Managed = EntityManager track kar raha hai; Detached = session band, changes track nahi honge; Removed = delete hoga)*

### 📖 How It Works (Detailed Explanation)

```
JPA Entity Lifecycle:
                   new User()
                       │
                       ▼
              ┌─────────────────┐
              │   TRANSIENT     │  ← No ID, not in DB, not tracked
              │   (New)         │
              └────────┬────────┘
                       │ persist()
                       ▼
              ┌─────────────────┐
    find() →  │   MANAGED       │  ← Tracked by EntityManager
   merge() →  │   (Persistent)  │  ← Dirty checking active
              │                 │  ← Changes auto-flushed to DB
              └───┬────────┬───┘
                  │        │
         detach() │        │ remove()
         session  │        │
         close    ▼        ▼
    ┌──────────────┐   ┌─────────────┐
    │  DETACHED    │   │  REMOVED    │
    │  (has ID,    │   │  (DELETE    │
    │  not tracked)│   │  on flush)  │
    └──────────────┘   └─────────────┘
         │
         │ merge()
         ▼
    Back to MANAGED
```

**Dirty checking**: for managed entities, Hibernate takes a snapshot at load time. At flush time (before commit or before JPQL query), it compares current state with snapshot — changed fields generate UPDATE SQL automatically. **Detach scenarios**: session closes (end of `@Transactional`), explicit `entityManager.detach(entity)`, `entityManager.clear()` (detaches all). **merge()**: copies detached entity's state into a NEW managed instance → returns the managed copy. The original detached object remains detached.

### 🗣️ Answering Approach
"JPA entities go through four states. Transient is a plain Java object — just instantiated, not associated with any persistence context or database row. When I call persist or the entity is loaded via findById, it becomes Managed — the persistence context tracks it, takes a snapshot of its state, and at flush time compares the current state to detect changes via dirty checking. Any modifications to managed entities are automatically persisted without calling save explicitly. When the session closes — typically at the end of a @Transactional method — the entity becomes Detached. It still has its ID and data, but changes aren't tracked anymore. If I need to re-attach it, I use merge, which returns a new managed instance with the detached entity's state copied over. Removed is the state after calling remove — the entity is marked for deletion and the DELETE SQL fires at flush."

### 💻 Code Example

```java
// ✅ TRANSIENT → new object, not in DB
User user = new User("Alice", "alice@example.com");
// user.getId() == null; not tracked by EntityManager

// ✅ TRANSIENT → MANAGED (persist)
@Transactional
public User createUser(String name, String email) {
    User user = new User(name, email);     // TRANSIENT
    entityManager.persist(user);            // → MANAGED (INSERT queued)
    user.setEmail("new@example.com");       // ⭐ auto-detected! UPDATE queued
    return user;                            // flush at commit → INSERT + UPDATE
}

// ✅ MANAGED via find
@Transactional
public void updateUser(Long id) {
    User user = entityManager.find(User.class, id);  // MANAGED (loaded from DB)
    user.setName("Bob");                              // ⭐ dirty checking detects change
    // No save/persist needed! Hibernate auto-flushes the UPDATE
}

// ✅ MANAGED → DETACHED (session closes)
@Transactional
public User loadUser(Long id) {
    User user = userRepo.findById(id).orElseThrow();  // MANAGED inside @Transactional
    return user;
}
// After method returns → session closes → user is now DETACHED

public void modifyDetachedUser(User user) {
    user.setName("Charlie");   // ❌ NOT tracked — change is LOST!
    // Must re-attach to save changes
}

// ✅ DETACHED → MANAGED (merge)
@Transactional
public User reattachUser(User detachedUser) {
    User managedUser = entityManager.merge(detachedUser);  // copies state → new MANAGED instance
    // ⚠️ detachedUser is STILL detached! Use managedUser.
    managedUser.setName("Updated");  // ✅ tracked
    return managedUser;
}

// ✅ MANAGED → REMOVED
@Transactional
public void deleteUser(Long id) {
    User user = entityManager.find(User.class, id);  // MANAGED
    entityManager.remove(user);                        // → REMOVED (DELETE queued)
    // DELETE fires at flush/commit
}

// ✅ Spring Data JPA equivalents
@Transactional
public void springDataExample() {
    User user = new User("Alice", "a@x.com");
    userRepo.save(user);           // persist (transient → managed) or merge (detached → managed)
    user.setEmail("new@x.com");    // dirty checking if still within @Transactional

    userRepo.delete(user);         // remove (managed → removed)
}

// ✅ Check entity state
boolean isManaged = entityManager.contains(user);  // true if MANAGED
```

### ⚠️ Common Pitfalls
- **Modifying detached entity** — changes are silently lost (not tracked) → always work within `@Transactional`
- **merge() returns a NEW instance** — the original detached entity stays detached → use the return value
- **persist() on detached entity** — throws `EntityExistsException` → use `merge()` for detached entities
- **LazyInitializationException** — accessing lazy collection on DETACHED entity → session already closed → use `JOIN FETCH` or `@Transactional`
- **save() in Spring Data** does persist OR merge based on entity's `isNew()` check (null ID = persist, non-null = merge)

### 🆚 Comparison Table

| State | In DB | In Persistence Context | Changes Tracked | Transition |
|-------|-------|----------------------|----------------|------------|
| **Transient** | ❌ | ❌ | ❌ | `persist()` → Managed |
| **Managed** | ✅ | ✅ | ✅ (dirty checking) | `detach()`/close → Detached |
| **Detached** | ✅ | ❌ | ❌ | `merge()` → Managed |
| **Removed** | ✅ (until flush) | ✅ | N/A | `persist()` → Managed |

### 🎯 Tricky Follow-up Questions
- **Q**: Does `save()` in Spring Data JPA always do `persist()`?  
  **A**: No — if the entity has a null ID (or `isNew()` returns true), it calls `persist()`. If the entity already has an ID, it calls `merge()` instead.
- **Q**: Can you modify a detached entity and then call `merge()`?  
  **A**: Yes — `merge()` copies the detached entity's state into a new managed instance. But the original object stays detached.

### ⚡ Remember (Quick Recall)
- **Transient** → `persist()` → **Managed** → session close → **Detached** → `merge()` → **Managed**
- **Managed** → `remove()` → **Removed**
- Dirty checking = automatic UPDATE for managed entities
- `merge()` returns a NEW managed copy (original stays detached)
- `save()` in Spring Data = persist (new) or merge (existing)
- LazyInitializationException = accessing lazy field on detached entity

### 🔗 Follow-up Topics
- [Q1 → First/Second Level Cache (L1 = managed entities)](#q1)
- [Q1 in database/03 → save() vs saveAndFlush()](03-jpa-persistence-ops.md#q1)
- [Q11 in database/01 → Lazy vs Eager loading](01-jpa-sql-transactions.md#q11)
