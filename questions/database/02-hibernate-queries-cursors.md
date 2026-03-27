# 🗄️ Hibernate, Queries & SQL Cursors (Q3–Q5, Q9–Q10)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q3"></a>
## Q3. What are commonly used Hibernate annotations?

### 📝 One-Liner
Hibernate annotations map Java classes to database tables — `@Entity`, `@Table`, `@Id`, `@GeneratedValue` for entity basics; `@Column`, `@Enumerated`, `@Temporal` for field mapping; `@OneToMany`, `@ManyToOne`, `@ManyToMany` for relationships; `@Transient`, `@Lob`, `@CreationTimestamp` for special cases.

### 🔑 Quick Answer
**Entity annotations**: `@Entity` (marks class as DB entity), `@Table(name = "orders")` (custom table name), `@Id` (primary key), `@GeneratedValue(strategy = IDENTITY)` (auto-increment). **Column annotations**: `@Column(name, nullable, unique, length)` (customize column), `@Enumerated(STRING)` (enum as string in DB), `@Temporal(DATE/TIME/TIMESTAMP)` (date types). **Relationship annotations**: `@OneToMany`, `@ManyToOne`, `@OneToOne`, `@ManyToMany` with `@JoinColumn` or `@JoinTable`. **Special**: `@Transient` (skip DB mapping), `@Lob` (large objects), `@CreationTimestamp/@UpdateTimestamp` (auto timestamps), `@Version` (optimistic locking), `@NamedQuery` (pre-defined queries). *(Hibernate annotations = Java class ko DB table se map karo — bina SQL likhe)*

### 📖 How It Works
```
Java Class → Hibernate Annotations → Database Table

@Entity                          ┌──────────────────────┐
@Table(name = "orders")          │     orders (table)    │
class Order {                    ├──────────────────────┤
  @Id @GeneratedValue            │ id        BIGINT PK   │
  Long id;                       │ order_no  VARCHAR(50)  │
  @Column(unique=true)           │ amount    DECIMAL      │
  String orderNo;                │ status    VARCHAR(20)  │
  BigDecimal amount;             │ customer_id BIGINT FK  │
  @Enumerated(STRING)            │ created_at TIMESTAMP   │
  Status status;                 │ updated_at TIMESTAMP   │
  @ManyToOne                     │ version   INT          │
  Customer customer;             └──────────────────────┘
  @CreationTimestamp
  LocalDateTime createdAt;
  @UpdateTimestamp
  LocalDateTime updatedAt;
  @Version
  int version;
}

Annotation Categories:
  1. Entity & Table:  @Entity, @Table, @Id, @GeneratedValue
  2. Column mapping:  @Column, @Enumerated, @Temporal, @Lob
  3. Relationships:   @OneToMany, @ManyToOne, @OneToOne, @ManyToMany
  4. Join config:     @JoinColumn, @JoinTable, @MappedBy
  5. Lifecycle:       @CreationTimestamp, @UpdateTimestamp, @PrePersist
  6. Special:         @Transient, @Version, @Formula, @Where
  7. Query:           @NamedQuery, @NamedNativeQuery, @Query (Spring Data)
```

### 🗣️ How to Say in Interview
"Hibernate annotations map Java objects to database tables. At the entity level, @Entity marks a class as a JPA entity and @Table customizes the table name. For the primary key, I use @Id with @GeneratedValue using IDENTITY strategy for auto-increment. @Column lets me customize column name, nullability, uniqueness, and length. For relationships, @ManyToOne is the most common — it creates a foreign key column; @OneToMany with mappedBy defines the inverse side. I always use @Enumerated with STRING rather than ORDINAL to prevent bugs when enum values are reordered. @CreationTimestamp and @UpdateTimestamp auto-manage audit dates. @Version enables optimistic locking. And @Transient marks fields that shouldn't be persisted to the database."

### 💻 Code
```java
// Complete entity with commonly used annotations
@Entity
@Table(name = "products", indexes = {
    @Index(name = "idx_product_sku", columnList = "sku", unique = true),
    @Index(name = "idx_product_category", columnList = "category_id")
})
@Where(clause = "deleted = false")   // soft delete filter
@NamedQuery(name = "Product.findActive",
    query = "SELECT p FROM Product p WHERE p.active = true")
public class Product {
    
    // Primary key
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // auto-increment
    private Long id;
    
    // Column customization
    @Column(name = "sku", nullable = false, unique = true, length = 50)
    private String sku;
    
    @Column(nullable = false)
    private String name;
    
    @Column(precision = 10, scale = 2)
    private BigDecimal price;
    
    // Enum mapping — ALWAYS use STRING, not ORDINAL
    @Enumerated(EnumType.STRING)
    @Column(length = 20)
    private ProductStatus status;  // "ACTIVE", "DISCONTINUED" in DB
    
    // Large text/binary
    @Lob
    private String description;   // TEXT/CLOB in DB
    
    // Relationships
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id", nullable = false)
    private Category category;   // FK → categories.id
    
    @OneToMany(mappedBy = "product", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<ProductImage> images = new ArrayList<>();
    
    @ManyToMany
    @JoinTable(name = "product_tags",
        joinColumns = @JoinColumn(name = "product_id"),
        inverseJoinColumns = @JoinColumn(name = "tag_id"))
    private Set<Tag> tags = new HashSet<>();
    
    // Audit timestamps (auto-managed by Hibernate)
    @CreationTimestamp
    @Column(updatable = false)
    private LocalDateTime createdAt;
    
    @UpdateTimestamp
    private LocalDateTime updatedAt;
    
    // Optimistic locking
    @Version
    private int version;
    
    // NOT persisted to DB — exists only in Java
    @Transient
    private double discountedPrice;
    
    // Calculated column (read-only, SQL formula)
    @Formula("price * 0.9")
    private BigDecimal discountPrice;
    
    // Soft delete flag
    private boolean deleted = false;
    
    // Lifecycle callbacks
    @PrePersist
    public void prePersist() {
        if (sku == null) sku = UUID.randomUUID().toString().substring(0, 8);
    }
}

// Relationship annotations summary
@ManyToOne   // FK in this table → Product has category_id column
@OneToMany   // inverse side, uses mappedBy → Category has List<Product>
@OneToOne    // unique FK → UserProfile has user_id (unique)
@ManyToMany  // join table → product_tags (product_id, tag_id)

// GenerationType strategies
@GeneratedValue(strategy = GenerationType.IDENTITY)   // AUTO_INCREMENT (MySQL)
@GeneratedValue(strategy = GenerationType.SEQUENCE)    // SEQUENCE (PostgreSQL) ⭐
@GeneratedValue(strategy = GenerationType.TABLE)       // separate table (portable but slow)
@GeneratedValue(strategy = GenerationType.AUTO)        // provider decides
```

### ⚠️ Pitfalls / Gotchas
- **@Enumerated(ORDINAL)** stores enum as 0, 1, 2 — reordering enums breaks existing DB data! Always use STRING *(ORDINAL mat use karo — enum reorder kiya toh pura data galat ho jaayega)*
- **@Transient** (annotation) ≠ `transient` (keyword): annotation = skip JPA mapping, keyword = skip Java serialization
- **CascadeType.ALL on @ManyToMany** can accidentally delete shared entities
- **orphanRemoval = true** deletes child when removed from parent's collection — dangerous if child has other references
- **@Formula** is read-only and uses native SQL — not portable across databases
- **FetchType.EAGER on @OneToMany** → Cartesian product if multiple eager collections

### 🆚 vs. Comparison
| Annotation | Purpose | Common Attributes |
|-----------|---------|-------------------|
| `@Entity` | Mark as JPA entity | — |
| `@Table` | Customize table | name, indexes, uniqueConstraints |
| `@Id` + `@GeneratedValue` | Primary key | strategy (IDENTITY, SEQUENCE) |
| `@Column` | Customize column | name, nullable, unique, length |
| `@ManyToOne` | FK relationship | fetch, optional, cascade |
| `@OneToMany` | Inverse collection | mappedBy, cascade, orphanRemoval |
| `@Enumerated` | Map enum | STRING ⭐ (never ORDINAL) |
| `@Version` | Optimistic lock | — |
| `@Transient` | Skip DB mapping | — |
| `@CreationTimestamp` | Auto create date | — |

### ⚡ Remember
- **Entity basics**: @Entity + @Table + @Id + @GeneratedValue
- **Columns**: @Column (nullable, unique, length) + @Enumerated(STRING) *(ORDINAL kabhi nahi)*
- **Relationships**: @ManyToOne (FK side) + @OneToMany(mappedBy) (inverse)
- **Audit**: @CreationTimestamp + @UpdateTimestamp (auto-managed)
- **Special**: @Version (optimistic lock), @Transient (skip DB), @Lob (large data)
- @Transient ≠ transient keyword

### 🔗 Follow-ups
- [Q4 → @Entity startup behavior](#q4)
- [Q5 → Native vs Named Query](#q5)
- [Q2 → transient keyword vs @Transient](#q2)

---

<a id="q4"></a>
## Q4. What is the purpose of `@Entity` and how does it behave during application startup?

### 📝 One-Liner
`@Entity` marks a Java class as a JPA-managed entity mapped to a database table — at startup, Hibernate scans for @Entity classes, validates/creates tables (based on ddl-auto), builds metadata, and prepares the persistence context.

### 🔑 Quick Answer
**Purpose**: `@Entity` tells JPA/Hibernate "this class represents a database table." Every @Entity class MUST have a no-arg constructor and an `@Id` field. **At startup**: **(1)** Spring Boot auto-scans for @Entity classes in the base package and sub-packages. **(2)** Hibernate's `MetadataBuilder` builds an internal mapping model — reads all annotations (@Table, @Column, relationships) and creates `EntityPersister` objects. **(3)** Based on `spring.jpa.hibernate.ddl-auto`, it compares the entity model with actual DB schema: `validate` (check match, fail if mismatch), `update` (alter tables), `create` (drop + create), `create-drop` (create, then drop on shutdown). **(4)** `SessionFactory` (EntityManagerFactory) is built and cached — this is expensive and done only once. *(App start hote hi Hibernate @Entity scan karta hai, table match karta hai, aur SessionFactory build karta hai — ye ek hi baar hota hai)*

### 📖 How It Works
```
Application Startup Timeline:

1. Spring Boot starts → Component scan
   └── Finds @Entity classes in base package

2. Hibernate bootstraps (SessionFactory creation):
   ├── a) Scan: Find all @Entity annotated classes
   │       Product.class, Order.class, User.class ...
   ├── b) Parse: Read annotations from each entity
   │       @Table(name="products")
   │       @Column(name="sku", nullable=false)
   │       @ManyToOne → Category.class
   │       Build EntityPersister for each entity
   ├── c) Schema management (ddl-auto):
   │       validate → Compare entity ↔ DB schema → FAIL if mismatch
   │       update   → ALTER TABLE to match entity (add columns, etc.)
   │       create   → DROP + CREATE all tables
   │       none     → Do nothing (production ⭐)
   ├── d) Build SessionFactory (EntityManagerFactory)
   │       → Caches all metadata, SQL templates, type mappings
   │       → This is the EXPENSIVE step (~2-5 seconds)
   └── e) Done! Ready to handle queries.

3. Runtime:
   EntityManager em = entityManagerFactory.createEntityManager();
   em.find(Product.class, 1L);
   → Hibernate already KNOWS Product → products table mapping
   → Generates: SELECT id, sku, name, price FROM products WHERE id = 1
   → Maps ResultSet back to Product object

@Entity Requirements:
  ✅ Must have @Entity annotation
  ✅ Must have @Id field (primary key)
  ✅ Must have no-arg constructor (public or protected)
  ✅ Must NOT be final (CGLIB proxying for lazy loading)
  ❌ Inner classes, interfaces, enums cannot be @Entity
```

### 🗣️ How to Say in Interview
"@Entity marks a class as a JPA entity representing a database table. At application startup, Hibernate scans for all @Entity classes, reads their annotations to build an internal metadata model including column mappings, relationships, and SQL templates. It then runs the ddl-auto strategy — in development I use 'update' to auto-alter tables, but in production I use 'validate' or 'none' with Flyway managing migrations. The SessionFactory, which caches all this metadata, is built once and is expensive — typically taking 2-5 seconds. After that, at runtime, Hibernate already knows how to generate SQL for any entity operation. The entity class must have a no-arg constructor, an @Id field, and cannot be final because Hibernate creates CGLIB subclass proxies for lazy loading."

### 💻 Code
```java
// Minimal valid @Entity
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;  // maps to column "name" by default
    
    // No-arg constructor REQUIRED (can be protected)
    protected User() {}
    
    public User(String name) {
        this.name = name;
    }
    // getters and setters
}

// ddl-auto settings (application.yml)
// spring:
//   jpa:
//     hibernate:
//       ddl-auto: validate    # PRODUCTION → fail if schema mismatch
//       ddl-auto: update      # DEV → auto ALTER tables
//       ddl-auto: create-drop # TESTS → fresh DB each run
//       ddl-auto: none        # PRODUCTION ⭐ → Flyway/Liquibase manages schema

// Hibernate startup logs (what you see in console):
// [INFO] HHH000412: Hibernate Core {6.4.1.Final}
// [INFO] HHH000206: hibernate.properties not found
// [INFO] HHH000490: Using JtaPlatform: SpringJtaPlatform
// [INFO] HHH000400: Using dialect: PostgreSQLDialect
// [INFO] Initialized JPA EntityManagerFactory for PU 'default'  ← SessionFactory ready!

// Entity scan configuration (if entities are outside base package)
@SpringBootApplication
@EntityScan("com.app.domain")   // custom entity scan path
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

// Startup failure examples:
// ddl-auto=validate + missing column:
// SchemaManagementException: Schema-validation: missing column [email] in table [users]

// Missing @Id:
// AnnotationException: No identifier specified for entity: com.app.User

// Missing no-arg constructor:
// InstantiationException: No default constructor for entity: com.app.User
```

### ⚠️ Pitfalls / Gotchas
- **ddl-auto=update in production** → dangerous! It can add columns but never drops them, and can cause data loss with type changes. Use Flyway/Liquibase *(Production mein ddl-auto=update mat karo — Flyway use karo)*
- **Entity class cannot be final** — Hibernate creates CGLIB proxy subclass for lazy loading; final prevents this
- **No-arg constructor** must exist (public or protected) — Hibernate uses it to instantiate entities via reflection
- **Missing @EntityScan** — if entities are in a different package than @SpringBootApplication, they won't be found
- **SessionFactory build is slow** (~2-5s) — it's done once. If you see it rebuilding, there's a config issue
- **Field access vs property access** — Hibernate defaults to field access (annotations on fields) or property access (annotations on getters). Don't mix them in one entity

### 🆚 vs. Comparison
| ddl-auto | Startup Behavior | Use in |
|----------|-----------------|--------|
| `none` | No schema action | Production ⭐ (Flyway manages) |
| `validate` | Fails if mismatch | Production (safety check) |
| `update` | ALTER to match entity | Development |
| `create` | DROP + CREATE | Testing |
| `create-drop` | CREATE, drop on shutdown | Unit tests |

### ⚡ Remember
- `@Entity` = "this class = DB table" — requires @Id + no-arg constructor *(Entity = table, @Id = primary key, constructor chahiye)*
- **Startup**: scan @Entity → build metadata → ddl-auto action → SessionFactory (once)
- **Production**: ddl-auto = `none` or `validate` + Flyway
- Entity cannot be **final** (proxy won't work)
- SessionFactory build = **expensive, single-time** at startup

### 🔗 Follow-ups
- [Q3 → Hibernate annotations (complete list)](#q3)
- [Q5 → Native vs Named Queries](#q5)
- [Q9 → Lazy vs eager loading (proxy mechanism from @Entity)](#q9)

---

<a id="q5"></a>
## Q5. What is the difference between Native Query and Named Query?

### 📝 One-Liner
**Native Query** = raw SQL sent directly to database (database-specific); **Named Query** = pre-defined JPQL/SQL stored on the entity class with a name, validated at startup, cached, and reusable.

### 🔑 Quick Answer
**Native Query** (`@Query(nativeQuery=true)` or `em.createNativeQuery()`): writes raw SQL — `SELECT * FROM users WHERE email = ?`. Database-specific (uses MySQL/PostgreSQL syntax), can use DB-specific features (window functions, CTEs), NOT validated at compile/startup time. **Named Query** (`@NamedQuery` on entity): pre-defined JPQL query with a name — `@NamedQuery(name = "User.findByEmail", query = "SELECT u FROM User u WHERE u.email = :email")`. **Validated at startup** (syntax errors caught early), cached by Hibernate (parsed once), JPQL is database-agnostic. Named queries can also be native via @NamedNativeQuery. Spring Data `@Query` (without nativeQuery) uses JPQL — behaves similarly to named queries. *(Native = seedha SQL, database-specific; Named = pehle se defined JPQL, startup pe validate hota hai)*

### 📖 How It Works
```
Native Query (raw SQL):
  em.createNativeQuery("SELECT * FROM users WHERE email = ?1", User.class)
    .setParameter(1, "user@mail.com")
    .getSingleResult();
  
  → Sent directly to DB engine (MySQL, PostgreSQL, etc.)
  → Database-specific syntax allowed
  → NOT validated at startup — error only at runtime!
  → Useful for: DB-specific features, complex joins, performance tuning

Named Query (JPQL, pre-defined):
  // Defined on entity class
  @Entity
  @NamedQuery(name = "User.findByEmail",
      query = "SELECT u FROM User u WHERE u.email = :email")
  class User { ... }
  
  // Used at runtime
  em.createNamedQuery("User.findByEmail", User.class)
    .setParameter("email", "user@mail.com")
    .getSingleResult();
  
  → JPQL parsed and validated at SessionFactory build time
  → Syntax error → application FAILS to start (early detection ⭐)
  → Parsed once, cached → slightly faster repeated execution
  → Database-agnostic (works on MySQL, PostgreSQL, Oracle)

Spring Data @Query (JPQL vs Native):
  // JPQL (like named query behavior)
  @Query("SELECT u FROM User u WHERE u.email = :email")
  Optional<User> findByEmail(@Param("email") String email);
  
  // Native SQL
  @Query(value = "SELECT * FROM users WHERE email = :email", nativeQuery = true)
  Optional<User> findByEmailNative(@Param("email") String email);

Validation Timeline:
  Named Query / JPQL:  validated at STARTUP → fail fast ⭐
  Native Query:        validated at EXECUTION → runtime error
  Spring @Query JPQL:  validated at STARTUP (Spring Data parses it)
```

### 🗣️ How to Say in Interview
"Native queries use raw SQL sent directly to the database, so you can leverage database-specific features like PostgreSQL window functions or MySQL-specific syntax. However, they're not validated at startup — syntax errors only show up at runtime. Named queries are pre-defined JPQL queries annotated on the entity class or in XML. They're parsed and validated when the SessionFactory builds at startup, so a typo in the query is caught immediately rather than in production. Named queries are also cached by Hibernate, so repeated execution is slightly faster since the query plan doesn't need re-parsing. In practice with Spring Data, I use @Query with JPQL for most repository methods — it's validated at startup like named queries — and switch to nativeQuery only when I need database-specific features like CTEs or complex analytics."

### 💻 Code
```java
// 1. NAMED QUERY — defined on entity, validated at startup
@Entity
@NamedQueries({
    @NamedQuery(
        name = "User.findByEmail",
        query = "SELECT u FROM User u WHERE u.email = :email"),
    @NamedQuery(
        name = "User.findActiveByRole",
        query = "SELECT u FROM User u WHERE u.active = true AND u.role = :role")
})
public class User {
    @Id @GeneratedValue
    private Long id;
    private String email;
    private String role;
    private boolean active;
}

// Using named query
List<User> admins = em.createNamedQuery("User.findActiveByRole", User.class)
        .setParameter("role", "ADMIN")
        .getResultList();

// 2. NATIVE QUERY — raw SQL, database-specific
// Simple native query
List<User> users = em.createNativeQuery(
        "SELECT * FROM users WHERE email ILIKE :pattern", User.class)
        .setParameter("pattern", "%gmail.com")
        .getResultList();

// Native with DB-specific features (PostgreSQL CTE + window function)
@Query(value = """
    WITH ranked AS (
        SELECT *, ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC) as rn
        FROM employees
    )
    SELECT * FROM ranked WHERE rn <= 3
    """, nativeQuery = true)
List<Object[]> findTop3SalariesPerDepartment();

// 3. NAMED NATIVE QUERY — named + native combined
@Entity
@NamedNativeQuery(
    name = "User.findByEmailNative",
    query = "SELECT * FROM users WHERE email = :email",
    resultClass = User.class)
public class User { ... }

// 4. SPRING DATA @Query — most common approach
public interface UserRepository extends JpaRepository<User, Long> {
    
    // JPQL (validated at startup, database-agnostic)
    @Query("SELECT u FROM User u WHERE u.email = :email AND u.active = true")
    Optional<User> findActiveByEmail(@Param("email") String email);
    
    // Native (validated at runtime only)
    @Query(value = "SELECT * FROM users WHERE created_at > NOW() - INTERVAL '7 days'",
           nativeQuery = true)
    List<User> findRecentUsers();
    
    // JPQL with update
    @Modifying
    @Query("UPDATE User u SET u.active = false WHERE u.lastLogin < :cutoff")
    int deactivateInactiveUsers(@Param("cutoff") LocalDateTime cutoff);
    
    // Native with pagination (countQuery required!)
    @Query(value = "SELECT * FROM users WHERE role = :role",
           countQuery = "SELECT COUNT(*) FROM users WHERE role = :role",
           nativeQuery = true)
    Page<User> findByRoleNative(@Param("role") String role, Pageable pageable);
}
```

### ⚠️ Pitfalls / Gotchas
- **Native Query + Pageable** → MUST provide `countQuery` separately (Spring can't auto-generate count for native SQL) *(Native query mein pagination ke liye countQuery alag se dena padta hai)*
- **Native Query + entity mapping** → column names must match entity fields (or use @SqlResultSetMapping)
- **Named Query typo** in name → discovered at startup ONLY if the named query is actually used in a `@Query` reference
- **JPQL uses entity/field names** (not table/column names): `SELECT u FROM User u` not `SELECT * FROM users`
- **@Modifying queries** need `@Transactional` and clear the persistence context (`clearAutomatically = true`) to avoid stale cache
- **Native queries are not portable** — switching from MySQL to PostgreSQL may break native queries

### 🆚 vs. Comparison
| Aspect | Named Query (JPQL) | Native Query (SQL) | Spring @Query |
|--------|-------------------|-------------------|--------------|
| Language | JPQL (entity-based) | Raw SQL (table-based) | Both |
| Validation | Startup ⭐ | Runtime | JPQL: startup, Native: runtime |
| DB-agnostic | ✅ Yes | ❌ No | Depends on mode |
| Caching | Parsed once ⭐ | May or may not cache | Depends |
| DB features | Limited (JPQL subset) | Full (CTEs, window fn) ⭐ | Full if native |
| Defined in | @NamedQuery on entity | Inline or @NamedNativeQuery | Repository method |
| Modern usage | Less common now | When DB-specific needed | Most common ⭐ |

### ⚡ Remember
- **Native** = raw SQL, DB-specific, runtime validation, full DB power
- **Named** = JPQL, pre-defined, **startup validation** ⭐, cached *(Named = startup pe error dikhega — safe hai)*
- Spring Data `@Query` = JPQL by default (startup-validated), `nativeQuery=true` for SQL
- JPQL uses **entity names** (User, email), SQL uses **table names** (users, email)
- Always provide `countQuery` for native + Pageable
- Use native only when JPQL can't express the query (CTEs, window functions)

### 🔗 Follow-ups
- [Q3 → Hibernate annotations](#q3)
- [Q4 → @Entity startup behavior (named queries validated here)](#q4)
- [Q10 → SQL Cursors (native query territory)](#q10)

---

<a id="q9"></a>
## Q9. What is the difference between Eager Loading and Lazy Loading?

### 📝 One-Liner
Eager loading fetches related entities immediately with the parent (JOIN); lazy loading defers fetching until the related entity is actually accessed (proxy) — lazy is default for collections, eager for single associations.

### 🔑 Quick Answer
**Eager Loading** (`FetchType.EAGER`): when the parent entity is loaded, ALL related entities are fetched immediately in the same query (or N additional queries). Default for `@ManyToOne` and `@OneToOne`. **Lazy Loading** (`FetchType.LAZY`): related entities are NOT loaded — Hibernate returns a **proxy** (empty wrapper). The actual DB query fires ONLY when you first access the field (e.g., call `getItems().size()`). Default for `@OneToMany` and `@ManyToMany`. **Lazy is preferred** because you often don't need all related data. The risk is **N+1 problem** — fix with JOIN FETCH or @EntityGraph. *(Eager = sab ek saath laao, Lazy = zaroorat padne pe laao — Lazy best practice hai lekin N+1 ka dhyaan rakho)*

### 📖 How It Works
```
EAGER — loads everything immediately:
  @ManyToOne(fetch = FetchType.EAGER)  // default for @ManyToOne
  private Category category;
  
  Product p = productRepo.findById(1L);
  // SQL: SELECT p.*, c.* FROM products p
  //      LEFT JOIN categories c ON p.category_id = c.id
  //      WHERE p.id = 1
  // category already loaded ✅ — no additional query

LAZY — loads on demand (proxy):
  @OneToMany(mappedBy = "product", fetch = FetchType.LAZY)  // default
  private List<Review> reviews;
  
  Product p = productRepo.findById(1L);
  // SQL: SELECT * FROM products WHERE id = 1
  // p.reviews = HibernateProxy (NOT loaded yet)
  
  p.getReviews().size();  // ← first access triggers query!
  // SQL: SELECT * FROM reviews WHERE product_id = 1
  // NOW reviews are loaded

N+1 PROBLEM (lazy loading trap):
  List<Product> products = productRepo.findAll();           // 1 query
  for (Product p : products) {
      System.out.println(p.getCategory().getName());        // N queries!
  }
  // Total: 1 + 100 = 101 queries for 100 products!

FIX — JOIN FETCH:
  @Query("SELECT p FROM Product p JOIN FETCH p.category")
  List<Product> findAllWithCategory();
  // 1 query! Joins category in same SQL

FIX — @EntityGraph:
  @EntityGraph(attributePaths = {"category", "reviews"})
  List<Product> findAll();
  // 1 query! Loads specified relationships eagerly for THIS query only
```

### 🗣️ How to Say in Interview
"Lazy loading defers loading related entities until they're first accessed — Hibernate returns a proxy object and fires the SQL query only on demand. Eager loading fetches everything upfront. I always keep the default lazy for collections and make @ManyToOne lazy too by explicitly setting fetch = LAZY. This avoids loading unnecessary data. When I know I need related entities for a specific use case — like an order with its items — I use JOIN FETCH in the query or @EntityGraph on the repository method to load them in a single query. This gives me the best of both worlds: lazy by default for efficiency, and selective eager loading per query to prevent the N+1 problem."

### 💻 Code
```java
// Entity with explicit fetch types
@Entity
public class Product {
    @Id @GeneratedValue
    private Long id;
    private String name;
    
    @ManyToOne(fetch = FetchType.LAZY)  // override default EAGER → LAZY ⭐
    @JoinColumn(name = "category_id")
    private Category category;
    
    @OneToMany(mappedBy = "product", fetch = FetchType.LAZY)  // default LAZY
    private List<Review> reviews;
    
    @ManyToMany(fetch = FetchType.LAZY)  // default LAZY
    private Set<Tag> tags;
}

// FIX N+1: JOIN FETCH
public interface ProductRepository extends JpaRepository<Product, Long> {
    
    @Query("SELECT DISTINCT p FROM Product p JOIN FETCH p.category")
    List<Product> findAllWithCategory();
    
    @Query("SELECT p FROM Product p JOIN FETCH p.category JOIN FETCH p.reviews WHERE p.id = :id")
    Optional<Product> findByIdWithDetails(@Param("id") Long id);
    
    // @EntityGraph — declarative approach
    @EntityGraph(attributePaths = {"category", "tags"})
    List<Product> findByNameContaining(String name);
}

// DTO Projection — best for read-only (no lazy/eager issue)
@Query("SELECT new com.app.dto.ProductSummary(p.id, p.name, p.price, c.name) " +
       "FROM Product p JOIN p.category c")
List<ProductSummary> findProductSummaries();

// Hibernate.initialize() — force loading within transaction
@Transactional(readOnly = true)
public ProductDTO getProductWithReviews(Long id) {
    Product p = productRepo.findById(id).orElseThrow();
    Hibernate.initialize(p.getReviews());  // force load within transaction
    return toDTO(p);
}
```

### ⚠️ Pitfalls / Gotchas
- **LazyInitializationException** — accessing lazy collection after session/transaction is closed *(Transaction ke bahar lazy field access kiya toh exception aayega)*
- **Multiple eager collections** → `MultipleBagFetchException` (Hibernate can't fetch two Lists eagerly). Use `Set` instead of `List`, or fetch one at a time
- **spring.jpa.open-in-view=true** (default!) keeps session open during HTTP request — hides N+1 issues until production load
- **Eager on @OneToMany** = almost always wrong — loads ALL children even when you don't need them
- **JOIN FETCH + Pageable** → Hibernate fetches ALL data in memory, then paginates in Java (not SQL!). Use @BatchSize for paginated lazy loading

### 🆚 vs. Comparison
| Aspect | Lazy Loading | Eager Loading |
|--------|-------------|---------------|
| When loaded | On first access (proxy) | Immediately with parent |
| Default for | @OneToMany, @ManyToMany | @ManyToOne, @OneToOne |
| Performance | Better (loads less data) ⭐ | Worse (loads everything) |
| N+1 risk | Yes (fix with JOIN FETCH) | No (but Cartesian risk) |
| Session required | Yes (proxy needs session) | No (already loaded) |
| Best practice | Use as default ⭐ | Override to LAZY |
| Fix when needed | JOIN FETCH, @EntityGraph | Already loaded |

### ⚡ Remember
- **Lazy** = proxy (load on access) → default for collections ⭐
- **Eager** = JOIN immediately → default for @ManyToOne (override to LAZY)
- N+1 fix: **JOIN FETCH** or **@EntityGraph** *(N+1 hoga toh JOIN FETCH lagao)*
- Best practice: **all LAZY** + selective JOIN FETCH per query
- DTO projection = best for read-only (no lazy/eager issues at all)
- `open-in-view=false` in production

### 🔗 Follow-ups
- [Q3 → Hibernate annotations (FetchType)](#q3)
- [Q4 → @Entity (proxy creation at startup)](#q4)
- Q11 → Lazy vs Eager deep-dive (database/01)

---

<a id="q10"></a>
## Q10. What is a Cursor in SQL and when should it be used?

### 📝 One-Liner
A cursor is a database pointer that lets you process query results **row by row** instead of all at once — use it for row-level processing that can't be done in set-based SQL (rare), large batch migrations, or complex per-row logic.

### 🔑 Quick Answer
**Cursor** = a control structure that retrieves one row at a time from a result set. Declared → Opened (executes query) → Fetched row by row in a loop → Closed. Normal SQL operates on **sets** (all rows at once). Cursors operate on **individual rows**. **When to use**: (1) Complex per-row logic that can't be expressed in SQL. (2) Large data migration with per-row transformations. (3) Sending notifications per row. (4) Calling an API for each row. **When NOT to use (99% of cases)**: any logic that can be expressed as `UPDATE ... SET ... WHERE` — set-based SQL is 10-100× faster than cursors. Cursors hold locks, consume memory, and are extremely slow for large datasets. *(Cursor = ek ek row pe kaam karna — lekin bahut slow hai; jab tak zaroori na ho, set-based SQL use karo)*

### 📖 How It Works
```
Normal SQL (set-based — FAST ⭐):
  UPDATE employees SET salary = salary * 1.10 WHERE department = 'ENGINEERING';
  → ALL matching rows updated in ONE operation
  → DB engine optimizes internally (index scan, parallel execution)
  → Milliseconds for millions of rows

Cursor (row-by-row — SLOW):
  DECLARE emp_cursor CURSOR FOR
    SELECT id, salary FROM employees WHERE department = 'ENGINEERING';
  OPEN emp_cursor;
    FETCH NEXT → row 1: {id: 101, salary: 50000} → process → UPDATE
    FETCH NEXT → row 2: {id: 102, salary: 60000} → process → UPDATE
    FETCH NEXT → row 3: {id: 103, salary: 55000} → process → UPDATE
    ... repeat 10,000 times ...
  CLOSE emp_cursor;
  → Each FETCH = disk seek + lock + context switch
  → 10-100× slower than set-based!

Cursor Lifecycle:
  DECLARE → OPEN → FETCH (loop) → CLOSE → DEALLOCATE
  
  ┌─────── Result Set (in DB memory) ───────┐
  │ Row 1:  {id: 1, name: "A", salary: 50K} │ ← cursor pointer
  │ Row 2:  {id: 2, name: "B", salary: 60K} │
  │ Row 3:  {id: 3, name: "C", salary: 55K} │
  │ ...                                       │
  └───────────────────────────────────────────┘
  FETCH NEXT moves pointer down one row
```

### 🗣️ How to Say in Interview
"A cursor is a database mechanism for row-by-row processing of query results — you declare it with a SELECT statement, open it, fetch rows one at a time in a loop, and then close it. In practice, I almost never use cursors because set-based SQL is vastly more efficient — a single UPDATE statement modifying millions of rows will be 10 to 100 times faster than iterating with a cursor. The only cases where cursors are justified are when you have complex per-row logic that truly can't be expressed in SQL — like calling an external API for each row, sending row-specific notifications, or doing complex calculations that depend on the previous row's result. Even then, I prefer doing the iteration in application code with pagination rather than using a DB cursor, because it avoids holding database locks. If I must use a cursor, I use FAST_FORWARD and READ_ONLY options for best performance."

### 💻 Code
```sql
-- SQL Server / PostgreSQL Cursor example
-- Scenario: Apply variable raise based on complex rules per employee

-- CURSOR APPROACH (when set-based isn't feasible)
DECLARE @emp_id INT, @salary DECIMAL, @department VARCHAR(50);
DECLARE @raise DECIMAL;

DECLARE emp_cursor CURSOR FAST_FORWARD READ_ONLY FOR  -- optimized cursor
    SELECT id, salary, department FROM employees WHERE active = true;

OPEN emp_cursor;

FETCH NEXT FROM emp_cursor INTO @emp_id, @salary, @department;

WHILE @@FETCH_STATUS = 0
BEGIN
    -- Complex per-row logic (hard to express in single SQL)
    SET @raise = CASE
        WHEN @department = 'ENGINEERING' AND @salary < 80000 THEN 0.15
        WHEN @department = 'ENGINEERING' AND @salary >= 80000 THEN 0.08
        WHEN @department = 'SALES' THEN (SELECT commission_rate FROM sales_targets WHERE emp_id = @emp_id)
        ELSE 0.05
    END;
    
    UPDATE employees SET salary = salary * (1 + @raise) WHERE id = @emp_id;
    
    -- Log each change (per-row side effect)
    INSERT INTO salary_audit (employee_id, old_salary, new_salary, change_date)
    VALUES (@emp_id, @salary, @salary * (1 + @raise), GETDATE());
    
    FETCH NEXT FROM emp_cursor INTO @emp_id, @salary, @department;
END;

CLOSE emp_cursor;
DEALLOCATE emp_cursor;

-- BETTER: Set-based equivalent (when possible)
UPDATE e
SET salary = salary * (1 + CASE
    WHEN department = 'ENGINEERING' AND salary < 80000 THEN 0.15
    WHEN department = 'ENGINEERING' AND salary >= 80000 THEN 0.08
    ELSE 0.05
END)
FROM employees e
WHERE active = true AND department != 'SALES';
-- 100× faster! No cursor needed.
```

```java
// In Java/Spring — prefer application-level pagination over DB cursors

// OPTION 1: Spring Data pagination (stream large results)
@Transactional(readOnly = true)
public void processAllEmployees() {
    int page = 0;
    Page<Employee> batch;
    do {
        batch = employeeRepo.findByActiveTrue(PageRequest.of(page, 500));
        batch.getContent().forEach(this::applyComplexLogic);
        page++;
    } while (batch.hasNext());
}

// OPTION 2: Spring Data Stream (uses DB cursor internally)
@Transactional(readOnly = true)
public void processAllWithStream() {
    try (Stream<Employee> stream = employeeRepo.streamByActiveTrue()) {
        stream.forEach(this::applyComplexLogic);
    }  // cursor auto-closed
}

// Repository method returning Stream (uses DB cursor)
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    @QueryHints(@QueryHint(name = HINT_FETCH_SIZE, value = "500"))
    Stream<Employee> streamByActiveTrue();  // DB cursor under the hood
}

// OPTION 3: JdbcTemplate with RowCallbackHandler (streaming, low memory)
jdbcTemplate.query(
    "SELECT id, salary, department FROM employees WHERE active = true",
    (ResultSet rs) -> {
        while (rs.next()) {
            long id = rs.getLong("id");
            double salary = rs.getDouble("salary");
            // process row by row — low memory footprint
        }
    }
);
```

### ⚠️ Pitfalls / Gotchas
- **Cursors hold locks** — other transactions may be blocked while cursor is open *(Cursor khula hai toh row pe lock laga rehta hai — dusre transactions wait karenge)*
- **Memory consumption** — cursor result set stored in DB memory; large result → tempdb spills
- **Performance** — 10-100× slower than set-based SQL for the same operation
- **Cursor left open** → resource leak → connection pool exhaustion. Always CLOSE + DEALLOCATE in a try-finally
- **In Java, prefer pagination** over raw DB cursors — same row-by-row processing without holding DB locks for the entire operation
- **FAST_FORWARD + READ_ONLY** options = fastest cursor type (forward-only, no updates through cursor)

### 🆚 vs. Comparison
| Aspect | Set-Based SQL | Cursor (Row-by-Row) | App-Level Pagination |
|--------|--------------|--------------------|--------------------|
| Speed | Fastest ⭐ | Slowest | Fast |
| Memory | DB optimized | Holds result set | Processes batches |
| Locking | Short lock | Long lock ❌ | Short per-batch ⭐ |
| Complexity | Simple SQL | Complex DECLARE/FETCH | Simple Java loop |
| Use case | CRUD, aggregations | Per-row side effects | Process with API calls |
| Recommendation | 99% of cases ⭐ | Last resort | When Java logic needed ⭐ |

### 🎯 Tricky Interview Qs

**Q: Name a real use case where a cursor is truly needed.**
Generating sequential invoice numbers with gap-free numbering: each row's number depends on the previous one. Or: sending a unique notification per row with external API calls — can't be done in pure SQL. *(Ek row ka result pichle row pe depend karta hai — tab cursor zaroor lagega)*

**Q: What's the difference between STATIC, DYNAMIC, FORWARD_ONLY, and KEYSET cursors?**
STATIC = snapshot of data at OPEN time (doesn't see concurrent changes). DYNAMIC = sees all changes (expensive). FORWARD_ONLY = can only FETCH NEXT (fastest). KEYSET = sees value changes but not new rows. Use **FAST_FORWARD (FORWARD_ONLY + READ_ONLY)** for best performance.

### ⚡ Remember
- **Cursor** = row-by-row processing of query results
- **Set-based SQL** = 10-100× faster → use 99% of the time *(Jab tak bilkul zaroor na ho, cursor mat use karo)*
- Cursor lifecycle: DECLARE → OPEN → FETCH loop → CLOSE → DEALLOCATE
- Use cursors ONLY for: per-row side effects, row-dependent calculations, migration scripts
- In Java: prefer **pagination** or **Stream<Entity>** over raw DB cursors
- FAST_FORWARD + READ_ONLY = fastest cursor option

### 🔗 Follow-ups
- [Q5 → Native vs Named Query (how queries execute)](#q5)
- Q14 → SQL query optimization (database/01)
- [Q9 → Lazy loading (similar row-by-row access concept)](#q9)
