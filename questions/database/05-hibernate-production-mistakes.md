# 🛡️ Hibernate Production Mistakes — Common Pitfalls & Fixes (Q14–Q18)

> Most developers use Hibernate daily but few use it well. These are the most common production-grade mistakes and their fixes.

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Interview Script → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q14"></a>
## Q14. What is the N+1 Query Problem in Hibernate? How do you detect and fix it?

### 📝 One-Liner
N+1 problem: fetching a list of N entities triggers 1 query for the list + N additional queries for each entity's lazy-loaded association — silently kills performance at scale.

### 🔑 Quick Answer
When you load a list of entities with a lazy `@OneToMany`/`@ManyToOne`, Hibernate fires one SELECT for the list and then one SELECT per entity when you access the association. Fix: use `JOIN FETCH` in JPQL, `@EntityGraph`, or `@BatchSize`. Detection: enable SQL logging or use Hibernate statistics. *(N+1 = list fetch ke baad har entity ke liye separate query — pehle detect karo SQL log se, phir JOIN FETCH se fix karo)*

### 📖 How It Works
```
Scenario: 10 departments, each with employees

WITHOUT fix (N+1):
Query 1: SELECT * FROM department                          → 1 query
Query 2: SELECT * FROM employee WHERE dept_id = 1          → +1
Query 3: SELECT * FROM employee WHERE dept_id = 2          → +1
...
Query 11: SELECT * FROM employee WHERE dept_id = 10        → +1
Total: 1 + 10 = 11 queries (N+1 where N=10)

WITH JOIN FETCH (fixed):
Query 1: SELECT d.*, e.* FROM department d
         LEFT JOIN employee e ON d.id = e.dept_id          → 1 query
Total: 1 query
```

### 💻 Code
```java
// PROBLEM: N+1 queries
@Entity
public class Department {
    @Id private Long id;
    @OneToMany(mappedBy = "department", fetch = FetchType.LAZY)
    private List<Employee> employees; // lazy — triggers N+1 when accessed
}

// This triggers N+1:
List<Department> depts = deptRepo.findAll(); // 1 query
for (Department d : depts) {
    d.getEmployees().size(); // N additional queries!
}

// FIX 1: JOIN FETCH in JPQL
@Query("SELECT d FROM Department d LEFT JOIN FETCH d.employees")
List<Department> findAllWithEmployees();

// FIX 2: @EntityGraph
@EntityGraph(attributePaths = {"employees"})
List<Department> findAll();

// FIX 3: @BatchSize (batches lazy loads)
@OneToMany(mappedBy = "department")
@BatchSize(size = 20) // loads 20 associations per query instead of 1
private List<Employee> employees;

// DETECTION: Enable Hibernate statistics
// spring.jpa.properties.hibernate.generate_statistics=true
// Logs: "N queries executed for collection loading"
```

### ⚠️ Pitfalls
- `JOIN FETCH` with pagination (`Pageable`) causes Hibernate to fetch ALL results in memory and paginate in-app — use `@BatchSize` or subquery fetch instead
- Multiple `JOIN FETCH` on collection associations → Cartesian product explosion
- `@EntityGraph` is cleaner than JPQL but creates dynamic query each time

### 🆚 vs.
| Fix | Approach | Best For |
|-----|----------|----------|
| JOIN FETCH | Single query with join | Known associations, no pagination |
| @EntityGraph | Declarative graph | Spring Data repos, flexible |
| @BatchSize | Batch lazy loads | Pagination-safe, multiple collections |
| Subselect fetch | Subquery for collection | Large collections, consistent perf |

### ⚡ Remember
- **Detection**: enable `spring.jpa.show-sql=true` + `hibernate.generate_statistics=true`
- **Rule**: if you iterate over a list and access associations, you likely have N+1
- JOIN FETCH is the most common fix — but watch out for Cartesian products
- Production monitoring: use p6spy or datasource-proxy for SQL logging

### 🔗 Follow-ups
→ See Q15 (FetchType defaults) | Q1 in [04-hibernate-cache-states.md](./04-hibernate-cache-states.md) (caching)

---

<a id="q15"></a>
## Q15. Why is FetchType.EAGER problematic? When should you use LAZY vs EAGER?

### 📝 One-Liner
EAGER loading fetches all associated data whether you need it or not — adds unnecessary DB load and memory usage on high-volume queries.

### 🔑 Quick Answer
`FetchType.EAGER` loads associations immediately with the parent entity. On read-heavy modules, this pulls in entire object graphs even when you only need the parent. Default to `FetchType.LAZY` (load on access) and use `JOIN FETCH` or `@EntityGraph` when you explicitly need associations. JPA defaults: `@OneToMany` = LAZY, `@ManyToOne` = EAGER. *(Eager = sab kuch load kar do chahe zaroorat ho ya na ho — Lazy default rakho, zaroorat pe fetch karo)*

### 📖 How It Works
```
FetchType.EAGER:
  findById(1) → SELECT dept.*, emp.*, project.*, address.*
  Even if you only need dept.name!
  Memory: loads entire object graph

FetchType.LAZY:
  findById(1) → SELECT dept.*           (only department)
  dept.getEmployees() → SELECT emp.*     (on demand)
  Only loads what you actually access
```

### 💻 Code
```java
@Entity
public class Department {
    @Id private Long id;
    private String name;

    // ❌ EAGER — loads employees with every department query
    @OneToMany(mappedBy = "department", fetch = FetchType.EAGER)
    private List<Employee> employees;

    // ✅ LAZY — loads employees only when accessed
    @OneToMany(mappedBy = "department", fetch = FetchType.LAZY)
    private List<Employee> employees;
}

// LAZY with explicit fetch when needed
@Query("SELECT d FROM Department d LEFT JOIN FETCH d.employees WHERE d.id = :id")
Optional<Department> findByIdWithEmployees(@Param("id") Long id);
```

### 🆚 vs.
| Aspect | LAZY | EAGER |
|--------|------|-------|
| When loaded | On access | Immediately |
| Default for @OneToMany | ✅ LAZY | — |
| Default for @ManyToOne | — | ✅ EAGER |
| N+1 risk | Yes (if not JOIN FETCH) | No (but overfetches) |
| Memory | Lower | Higher |
| **Best practice** | ✅ Default | Only for always-needed data |

### ⚡ Remember
- **Rule**: LAZY by default, EAGER by exception
- Override `@ManyToOne` default: `@ManyToOne(fetch = FetchType.LAZY)`
- LAZY outside transaction → `LazyInitializationException` (use Open Session in View or DTO projection)
- EAGER on collections in REST APIs → infinite recursion → use `@JsonIgnore` or DTOs

---

<a id="q16"></a>
## Q16. Why should you NOT use hbm2ddl.auto=update in production? What's the alternative?

### 📝 One-Liner
`hbm2ddl.auto=update` auto-generates DDL from entities — convenient in dev but unpredictable, irreversible, and dangerous in production. Use Flyway or Liquibase instead.

### 🔑 Quick Answer
`hbm2ddl.auto=update` attempts to diff entity model vs DB schema and applies ALTER TABLE. Problems: it **never drops columns**, can't handle renames, doesn't manage data migration, and has no rollback. Production requires **version-controlled migrations** — Flyway (SQL files) or Liquibase (XML/YAML/SQL). *(Dev mein convenience ke liye theek hai, production mein kabhi use mat karo — Flyway ya Liquibase se schema manage karo)*

### 📖 How It Works
```
hbm2ddl.auto options:
├── none       → do nothing (production default)
├── validate   → check schema matches entities, fail if mismatch (safe for prod)
├── update     → apply DDL changes (DANGEROUS in prod)
├── create     → drop all + recreate (DESTROYS data)
└── create-drop→ create on start, drop on stop (testing only)

Why update is dangerous:
- Adds columns ✅ but NEVER removes columns ❌
- Can't rename columns (creates new, old remains)
- No data migration support
- No rollback capability
- Unreviewed DDL executed automatically
- Can lock tables on ALTER in MySQL/PostgreSQL
```

### 💻 Code
```properties
# DEV environment
spring.jpa.hibernate.ddl-auto=update

# PRODUCTION — use validate + Flyway
spring.jpa.hibernate.ddl-auto=validate
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
```
```sql
-- Flyway migration: V1__create_employee_table.sql
CREATE TABLE employee (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    department VARCHAR(100),
    salary DECIMAL(10,2)
);

-- V2__add_email_column.sql
ALTER TABLE employee ADD COLUMN email VARCHAR(255);

-- V3__create_index_on_department.sql
CREATE INDEX idx_employee_dept ON employee(department);
```

### 🆚 vs.
| Aspect | hbm2ddl.auto=update | Flyway/Liquibase |
|--------|--------------------|--------------------|
| Version controlled | ❌ | ✅ Git-tracked SQL files |
| Rollback | ❌ | ✅ (Liquibase) / manual (Flyway) |
| Data migration | ❌ | ✅ |
| Column removal | ❌ Never | ✅ Explicit |
| Rename support | ❌ | ✅ |
| Audit trail | ❌ | ✅ Migration history table |
| **Production safe** | ❌ | ✅ |

### ⚡ Remember
- **Production rule**: `ddl-auto=validate` (or `none`) + Flyway/Liquibase
- Flyway: simpler, SQL-first approach (versioned SQL files)
- Liquibase: more powerful, supports XML/YAML/JSON changeSets, auto-rollback
- Migration naming convention: `V1__description.sql`, `V2__description.sql` (Flyway)

---

<a id="q17"></a>
## Q17. Why is it important to monitor Hibernate-generated SQL? How to enable it?

### 📝 One-Liner
Hibernate abstracts SQL but the generated queries may be inefficient — monitoring reveals N+1 problems, unnecessary joins, missing indexes, and unexpected query patterns.

### 🔑 Quick Answer
Enable `show_sql` and `format_sql` in non-prod. For production monitoring, use datasource-proxy or p6spy for SQL logging with execution times. Hibernate statistics track query counts, cache hit/miss ratios, and slow queries. Without monitoring, you're blind to the actual SQL your mappings produce. *(Hibernate jo SQL generate karta hai wo dekhna zaroori hai — show_sql enable karo dev mein, production mein p6spy use karo)*

### 💻 Code
```properties
# DEV — basic SQL logging
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

# Better: use logging (goes to log file, not stdout)
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE  # show bind params

# Hibernate statistics
spring.jpa.properties.hibernate.generate_statistics=true

# PRODUCTION — datasource-proxy (programmatic, with execution time)
# See code below
```
```java
// datasource-proxy — production-grade SQL monitoring
@Configuration
public class DataSourceProxyConfig {
    @Bean
    public DataSource dataSource(DataSource originalDataSource) {
        return ProxyDataSourceBuilder.create(originalDataSource)
            .logQueryBySlf4j(SLF4JLogLevel.INFO)
            .multiline()
            .countQuery()
            .build();
    }
}

// Log output example:
// Name: MyDS, Connection: 3, Time: 12ms, Success: True
// Type: Prepared, Batch: False, QuerySize: 1, BatchSize: 0
// Query: SELECT e.id, e.name FROM employee e WHERE e.department = ?
// Params: [(IT)]
```

### ⚠️ Pitfalls
- `show_sql=true` prints to stdout (not to logging framework) — use `logging.level.org.hibernate.SQL=DEBUG` instead
- TRACE-level bind parameter logging is expensive — never in production
- Hibernate statistics add overhead — enable only for profiling sessions
- Don't ignore `HHH90000022: Using @ElementCollection` warnings — they indicate potential N+1

### ⚡ Remember
- **Dev**: `show_sql=true` + `format_sql=true` — see what Hibernate generates
- **Test**: Hibernate statistics — count queries per transaction
- **Production**: datasource-proxy or p6spy — log slow queries with timing
- Assert query count in tests: `assertThat(queryCount).isLessThanOrEqualTo(3)`

---

<a id="q18"></a>
## Q18. Explain the Second-Level Cache configuration and when to use it

### 📝 One-Liner
Second-level cache (L2) is a SessionFactory-scoped cache shared across sessions — reduces DB hits for frequently read, rarely changed entities using providers like Ehcache or Hazelcast.

### 🔑 Quick Answer
L1 cache (Session-scoped) is automatic. L2 cache (SessionFactory-scoped) needs explicit setup: add cache provider (Ehcache), annotate entities with `@Cacheable` + `@Cache`, and configure regions. Use for: reference data (countries, currencies), config tables, read-heavy/write-rare entities. Don't use for: frequently updated data, transactional data, user-specific data. *(L2 cache lagao un entities pe jo bahut baar read hoti hain par rarely change hoti hain — jaise country list, config table)*

### 💻 Code
```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jcache</artifactId>
</dependency>
<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```
```properties
# application.properties
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=jcache
spring.jpa.properties.javax.cache.provider=org.ehcache.jsr107.EhcacheCachingProvider
spring.jpa.properties.javax.cache.uri=classpath:ehcache.xml
spring.jpa.properties.hibernate.cache.use_query_cache=true
```
```java
@Entity
@Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Country {
    @Id private Long id;
    private String name;
    private String code;
}

// Query cache (caches query results, not just entities)
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
List<Country> findAll();
```

### 🆚 vs.
| Aspect | L1 Cache | L2 Cache |
|--------|----------|----------|
| Scope | Session (EntityManager) | SessionFactory (shared) |
| Enabled by default | ✅ Always | ❌ Needs config |
| Shared across sessions | ❌ | ✅ |
| Eviction | Session close | TTL / manual |
| Use case | Repeat reads in same TX | Cross-request caching |

| Cache Strategy | Use When |
|---------------|----------|
| READ_ONLY | Immutable reference data |
| READ_WRITE | Most common — read-heavy, occasional writes |
| NONSTRICT_READ_WRITE | Eventual consistency OK |
| TRANSACTIONAL | Full JTA transaction support needed |

### ⚡ Remember
- L2 cache entity: `@Cacheable` (JPA) + `@Cache(usage = ...)` (Hibernate)
- Query cache: caches query → entity ID mapping (still needs entity cache for data)
- Cache invalidation on entity update is automatic (Hibernate manages it)
- Don't cache: frequently updated tables, large collections, user-specific data

### 🔗 Follow-ups
→ See Q14 (N+1 problem) | Q1 in [04-hibernate-cache-states.md](./04-hibernate-cache-states.md) (L1 vs L2 deep dive)
