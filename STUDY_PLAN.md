# 📚 STUDY PLAN — Phase-Based Interview Preparation

[![Last Updated](https://img.shields.io/badge/Updated-April%202026-orange.svg)](#)
[![Total Questions](https://img.shields.io/badge/Total%20Questions-1136-blue.svg)](#)
[![Phases](https://img.shields.io/badge/Phases-7-green.svg)](#)

> **How to use this file:** Start from Phase 1. Complete each phase before moving to the next.
> When new batches are added, they slot into their phase automatically — see [Phase Assignment Rules](#-phase-assignment-rules).

---

## 🎯 My Targets

> Fill this section in. Everything below depends on these numbers.

| Field | Your Value |
|-------|------------|
| 📅 Study Start Date | _(e.g. April 14, 2026)_ |
| 🎯 Target Interview Date | _(e.g. June 30, 2026)_ |
| 🏢 Target Company / Role | _(e.g. Citi Bank — Senior Java Developer)_ |
| ⏳ Available Hours / Day | _(e.g. 3 hrs on weekdays, 5 hrs on weekends)_ |
| 📍 Track | Full (14 wk) / Express 8-wk / Emergency 2-wk |

---

## ⚡ Express Tracks — When You Have Less Time

### ⚡ Emergency Track — 2 Weeks (Interview in < 2 weeks)
> Do only these, in this order:
1. Phase 1 Day 2 → `core/10` (basics) + `core/12` (exceptions) + `core/15` (collections)
2. Phase 3 Day 1–2 → `mt/01` + `mt/02` (threading fundamentals + locks)
3. Phase 4 Day 1–3 → `spring/01,08,11,12` (Spring core + scenarios)
4. Phase 5 → `sd/04` (Product company designs)
5. Phase 7 → Your **target company's file** from company-specific/
6. Always: `hr-behavioral/01` + `hr-behavioral/02`

### ⚡ Fast Track — 4 Weeks
> Complete Phase 1, 2, 4 fully + cherry-pick from Phase 3, 5, 6:
- Week 1: Phase 1 (full)
- Week 2: Phase 2 (full)
- Week 3: Phase 4 Spring (spring/01–12) + Phase 3 Days 1–4
- Week 4: Phase 5 System Design (sd/01–08) + Phase 7 target company Qs + HR

### 📅 Standard Track — 8 Weeks
> Complete Phases 1–5 fully + top files from Phase 6 + Phase 7 target company:
- Weeks 1–2: Phase 1 + 2
- Weeks 3–4: Phase 3 + 4
- Weeks 5–6: Phase 5 + prod-debugging/06–07
- Weeks 7–8: Phase 7 (target company + HR + AI/ML)

### 📅 Full Track — 14+ Weeks
> Follow the complete phase-by-phase schedule below.

---

## 🗺️ Overview — 7 Phases

| Phase | Name | Weeks | Questions | Difficulty | Goal |
|-------|------|-------|-----------|------------|------|
| 🟢 **1** | Java Foundation | Week 1–2 | ~158 | Beginner | Answer every basic Java Q confidently |
| 🟢 **2** | Java Deep Dive | Week 3–4 | ~99 | Intermediate | Master Java 8+, JVM, Streams |
| 🟡 **3** | Multithreading & Concurrency | Week 5–6 | 112 | Intermediate → Hard | Write thread-safe production code |
| 🟡 **4** | Spring Boot & Database | Week 7–8 | ~110 | Intermediate → Hard | Build production-grade Spring apps |
| 🔴 **5** | Architecture & System Design | Week 9–10 | ~90 | Hard | Design scalable distributed systems |
| 🔴 **6** | Production, Spring Batch & DevOps | Week 11–12 | ~196 | Hard → Expert | Think like a senior engineer |
| ⭐ **7** | Interview Simulation | Week 13–16 | ~433 | Mixed (Interview Mode) | Speed + confidence + pattern recognition |

> **Total: ~1136 questions across 14 weeks**
> Daily target: ~10–15 questions on study days + 5 revision Qs from previous phase.

---

## 📅 Daily Study Routine

```
🌅 Morning   (45 min) — Revise yesterday's ⚡ Remember sections
📖 Afternoon (90 min) — Study today's file(s) using 10-section format
🌙 Evening   (30 min) — Self-quiz: cover answers, attempt 5 Qs from today
📝 Saturday  (2 hr)   — Do matching company-specific Qs from Phase 7
```
### ⏰ Only Have 30 Minutes Today?

| Time Available | What to Study |
|---------------|---------------|
| 30 min | Open current phase → read only the ⚡ **Remember** + 🔑 **Quick Answer** sections of 5 Qs |
| 45 min | Add 📝 **One-Liner** scan for 10 more Qs from same file |
| 60 min | Full 10-section study for 5 Qs as normal |
| Weekend 2 hr | One complete file end-to-end |

---

## 🔁 Spaced Repetition Schedule
> Forgetting curve fix — revisit older phases while progressing forward.

| When You Are In... | Also Revise... |
|--------------------|----------------|
| Phase 3 (Week 5–6) | Phase 1 — scan ⚡ Remember sections |
| Phase 4 (Week 7–8) | Phase 2 — scan ⚡ Remember sections |
| Phase 5 (Week 9–10) | Phase 3 — attempt 5 Qs self-quiz |
| Phase 6 (Week 11–12) | Phase 4 — attempt 5 Qs self-quiz |
| Phase 7 (Week 13–16) | Phase 1+2 quick scan + Phase 5 system design Qs |

> Rule: **Spend Saturday morning (30 min) on revision** from 2 phases back, before the 2-hr company-specific session.
---

## 🟢 Phase 1 — Java Foundation
**Weeks 1–2 | ~158 Questions | Goal: Never fumble basics**

Study these files in order:

| Day | Topic | File | Questions |
|-----|-------|------|-----------|
| 1 | OOP Principles (Encapsulation, Inheritance, Polymorphism, Abstraction) | [oops-patterns/01](./questions/oops-patterns/01-oop-principles.md) | 3 |
| 1 | OOP Real Project Examples | [oops-patterns/02](./questions/oops-patterns/02-oop-real-project-examples.md) | 5 |
| 2 | Java Basics — Static, Constructors, JIT, String | [core/10](./questions/languages/java/core/10-java-basics-fundamentals.md) | 25 |
| 3 | Access Modifiers, Packages, Keywords | [core/11](./questions/languages/java/core/11-access-modifiers-packages.md) | 26 |
| 4 | Exception Handling (Checked, Unchecked, try-catch, finally) | [core/12](./questions/languages/java/core/12-exception-handling.md) | 27 |
| 5 | Inner Classes & Nested Classes | [core/13](./questions/languages/java/core/13-inner-classes-nested.md) | 12 |
| 6–7 | Misc Java Fundamentals (Wrappers, Enums, Varargs, String pool) | [core/14](./questions/languages/java/core/14-misc-java-fundamentals.md) | 35 |
| 8–9 | Collections Framework (List, Set, Map — how they work) | [core/15](./questions/languages/java/core/15-collections-framework.md) | 25 |

**Phase 1 done when:** You can explain OOP, collections, exceptions, string vs StringBuilder, and access modifiers without hesitation.

---

## 🟢 Phase 2 — Java Deep Dive
**Weeks 3–4 | ~99 Questions | Goal: Java 8+, JVM internals, Streams mastery**

| Day | Topic | File | Questions |
|-----|-------|------|-----------|
| 1 | Java Fundamentals — Immutability, Comparable, String internals | [core/04](./questions/languages/java/core/04-java-fundamentals-epam.md) | 5 |
| 1 | Java 8 Functional — Streams, Lambda, Functional Interfaces | [core/05](./questions/languages/java/core/05-java8-functional-epam.md) | 5 |
| 2 | Collections Internals — TreeMap, Fail-fast iterators | [core/06](./questions/languages/java/core/06-collections-internals.md) | 4 |
| 2 | Collections, JVM & Threading (cross-topic) | [core/01](./questions/languages/java/core/01-collections-jvm-threading.md) | 6 |
| 3 | Streams Coding — Patterns (findFirst, duplicates, max) | [core/07](./questions/languages/java/core/07-streams-coding-patterns.md) | 6 |
| 4 | Streams Coding — Basic Operations | [core/08](./questions/languages/java/core/08-streams-coding-basic.md) | 16 |
| 5 | Streams Coding — Grouping & Collectors | [core/09](./questions/languages/java/core/09-streams-coding-collectors.md) | 12 |
| 6 | Java 8 Coding Part 2 — Multi-sort, Grouping, Real Interviews | [core/20](./questions/languages/java/core/20-java8-coding-interviews.md) | 13 |
| 7 | Serialization | [core/16](./questions/languages/java/core/16-serialization.md) | 8 |
| 7 | Streams & Serialization (combined) | [core/02](./questions/languages/java/core/02-streams-serialization.md) | 2 |
| 8 | Java Version Features (7, 8, 11, 17, 21) | [core/17](./questions/languages/java/core/17-java-versions-features.md) | 2 |
| 8 | JVM Performance Tuning & GC | [core/03](./questions/languages/java/core/03-jvm-performance-tuning.md) | 2 |
| 9 | Exception Handling — Advanced (2+ YOE patterns) | [core/19](./questions/languages/java/core/19-exception-handling-advanced.md) | 15 |
| 9 | Design Patterns, Sorting & Singleton | [core/18](./questions/languages/java/core/18-sorting-singleton-patterns.md) | 3 |

**Phase 2 done when:** You can write any stream pipeline from memory and explain JVM memory areas, GC, and Java 8+ features fluently.

---

## 🟡 Phase 3 — Multithreading & Concurrency
**Weeks 5–6 | 112 Questions | Goal: Thread-safe production code**

| Day | Topic | File | Questions |
|-----|-------|------|-----------|
| 1–2 | Core MT Basics — Thread lifecycle, start vs run, daemon | [mt/01](./questions/languages/java/multithreading/01-basics.md) | 23 |
| 3 | Synchronization & Locks — synchronized, ReentrantLock, volatile | [mt/02](./questions/languages/java/multithreading/02-synchronization.md) | 13 |
| 4 | Thread Communication — wait/notify, CountDownLatch, CyclicBarrier | [mt/03](./questions/languages/java/multithreading/03-thread-communication.md) | 16 |
| 5 | Concurrency Utilities — ExecutorService, Future, CompletableFuture | [mt/04](./questions/languages/java/multithreading/04-concurrency-utilities.md) | 14 |
| 6 | Concurrent Collections — ConcurrentHashMap, CopyOnWriteArrayList | [mt/05](./questions/languages/java/multithreading/05-concurrent-collections.md) | 6 |
| 6 | Memory Model & Visibility — happens-before, volatile, Atomic* | [mt/06](./questions/languages/java/multithreading/06-memory-model.md) | 6 |
| 7 | Deadlock & Concurrency Problems — detection, prevention | [mt/07](./questions/languages/java/multithreading/07-deadlock-problems.md) | 7 |
| 8 | Performance & Optimization | [mt/08](./questions/languages/java/multithreading/08-performance.md) | 7 |
| 9 | Spring Boot / Spring Batch MT | [mt/09](./questions/languages/java/multithreading/09-spring-multithreading.md) | 10 |
| 10 | Real Production MT Scenarios | [mt/10](./questions/languages/java/multithreading/10-production-scenarios.md) | 10 |

**Phase 3 done when:** You can explain volatile, synchronized, ReentrantLock differences, write a thread-safe singleton, and debug a deadlock.

---

## 🟡 Phase 4 — Spring Boot & Database
**Weeks 7–8 | ~110 Questions | Goal: Production-grade Spring apps**

**Spring Boot (do in order):**

| Day | Topic | File | Questions |
|-----|-------|------|-----------|
| 1 | Spring Framework Internals — Bean lifecycle, DI, @Transactional | [spring/01](./questions/languages/java/spring/01-spring-framework-internals.md) | 7 |
| 1 | AOP — Aspect, Pointcut, Advice, JoinPoint | [spring/02](./questions/languages/java/spring/02-aop.md) | 1 |
| 1 | Enterprise Practices — REST security, SQL injection prevention | [spring/03](./questions/languages/java/spring/03-enterprise-practices.md) | 4 |
| 2 | Spring Boot Internals (auto-config, starters, Actuator) | [spring/04](./questions/languages/java/spring/04-springboot-internals-epam.md) | 4 |
| 2 | MVC, Beans & Configuration | [spring/05](./questions/languages/java/spring/05-mvc-beans-config.md) | 4 |
| 2 | API Design Decisions | [spring/06](./questions/languages/java/spring/06-api-design-decisions.md) | 4 |
| 3 | Project Infrastructure Decisions | [spring/07](./questions/languages/java/spring/07-project-infrastructure-decisions.md) | 4 |
| 3 | Spring Boot Fundamentals | [spring/08](./questions/languages/java/spring/08-springboot-fundamentals.md) | 5 |
| 4 | Advanced Config & Security (Logging, Filters, Performance) | [spring/09](./questions/languages/java/spring/09-advanced-config-security.md) | 7 |
| 5 | Spring Boot Internals Advanced (Fat JAR, Cloud-Native, Properties) | [spring/10](./questions/languages/java/spring/10-springboot-internals-advanced.md) | 12 |
| 6 | Spring Boot Scenario Interviews (OCP, Parallel APIs, @Transactional pitfalls) | [spring/11](./questions/languages/java/spring/11-springboot-scenario-interviews.md) | 12 |
| 7–8 | Spring Boot REST + JPA Advanced (Book API, Redis, N+1, dirty checking) | [spring/12](./questions/languages/java/spring/12-springboot-rest-jpa-advanced.md) | 28 |

**Database (do alongside Spring):**

| Day | Topic | File |
|-----|-------|------|
| 4 | JPA, SQL & Transactions | [db/01](./questions/database/01-jpa-sql-transactions.md) |
| 5 | Hibernate Queries & Cursors | [db/02](./questions/database/02-hibernate-queries-cursors.md) |
| 6 | JPA Persistence Operations | [db/03](./questions/database/03-jpa-persistence-ops.md) |
| 7 | Hibernate Cache & Entity States | [db/04](./questions/database/04-hibernate-cache-states.md) |
| 8 | Hibernate Production Mistakes | [db/05](./questions/database/05-hibernate-production-mistakes.md) |

**Phase 4 done when:** You can explain Spring Bean scopes, auto-configuration, @Transactional rollback rules, and JPA N+1 problem with fix.

---

## 🔴 Phase 5 — Architecture & System Design
**Weeks 9–10 | ~90 Questions | Goal: Design scalable systems confidently**

| Day | Topic | File |
|-----|-------|------|
| 1 | Web Networking & Protocols (HTTP, HTTPS, DNS, TCP/UDP) | [web-dev/01](./questions/web-dev/01-networking-protocols-security.md) |
| 1 | REST, SOAP & HTTP — Methods, Status Codes, Idempotency | [web-dev/03](./questions/web-dev/03-rest-soap-http.md) |
| 2 | API Microservices Architecture | [java/arch/01](./questions/languages/java/architecture/01-api-design-microservices.md) |
| 2 | Caching Architecture Patterns | [java/arch/02](./questions/languages/java/architecture/02-caching-architecture-patterns.md) |
| 3 | System Design Distributed (Consistent Hashing, CAP, SAGA) | [java/arch/03](./questions/languages/java/architecture/03-system-design-distributed.md) |
| 3 | In-Memory Grids & Akka | [java/arch/04](./questions/languages/java/architecture/04-inmemory-grids-akka.md) |
| 4 | Service Communication (Sync vs Async, gRPC, MQ) | [arch/05](./questions/architecture/05-service-communication.md) |
| 4 | Technical BA API & Integration Interviews | [arch/06](./questions/architecture/06-technical-ba-api-interviews.md) |
| 5 | Distributed Systems Fundamentals | [sd/01](./questions/system-design/01-distributed-systems-fundamentals.md) |
| 5 | Coordination, Failover & Event Sourcing | [sd/02](./questions/system-design/02-coordination-failover-eventsourcing.md) |
| 6 | E-Commerce & Payment Systems Design | [sd/03](./questions/system-design/03-ecommerce-payment-systems.md) |
| 6–7 | Product Company Designs (Netflix, Uber, CDN, Chat, Cache) | [sd/04](./questions/system-design/04-product-company-designs.md) |
| 8 | PhonePe & Fintech Designs (UPI, Feed, SLOs) | [sd/05](./questions/system-design/05-phonepe-fintech-designs.md) |
| 8 | JPMorgan & Banking Designs | [sd/06](./questions/system-design/06-jpmorgan-banking-designs.md) |
| 9 | Backend Scenario Debugging & Design | [sd/08](./questions/system-design/08-backend-scenario-debugging.md) |
| 9 | System Design Roadmap 2026 (50 Topics Reference) | [sd/07](./questions/system-design/07-design-roadmap-2026.md) |

**Phase 5 done when:** You can design a URL shortener, rate limiter, notification system, and payment gateway end-to-end in 40 minutes.

---

## 🔴 Phase 6 — Production, Spring Batch & DevOps
**Weeks 11–12 | ~196 Questions | Goal: Senior engineer mindset**

**Production Debugging:**

| Day | Topic | File | Questions |
|-----|-------|------|-----------|
| 1 | JVM Memory & Performance Issues | [prod/01](./questions/languages/java/production-debugging/01-jvm-memory-performance.md) | 5 |
| 1 | Concurrency & Threading Issues in Prod | [prod/02](./questions/languages/java/production-debugging/02-concurrency-threading.md) | 6 |
| 2 | API, State & Design Issues | [prod/03](./questions/languages/java/production-debugging/03-api-state-design.md) | 5 |
| 2 | Services, Ops & Infrastructure | [prod/04](./questions/languages/java/production-debugging/04-services-ops-infra.md) | 4 |
| 2 | Backend Scenarios (Staging vs Prod, Slow DB, Kafka Dupes) | [prod/05](./questions/languages/java/production-debugging/05-backend-scenarios.md) | 4 |
| 3 | Java Runtime & JVM Scenarios (Memory Leaks, GC, Deadlock, OOM) | [prod/06](./questions/languages/java/production-debugging/06-java-runtime-scenarios.md) | 15 |
| 4 | Spring Boot Production Scenarios (Config, Pool, Docker, @Async) | [prod/07](./questions/languages/java/production-debugging/07-springboot-production-scenarios.md) | 15 |

**Spring Batch (complete in sequence):**

| Day | Topic | File | Questions |
|-----|-------|------|-----------|
| 5 | Spring Batch Basics | [sb/01](./questions/languages/java/spring-batch/01-basics.md) | 20 |
| 5 | Chunk Processing | [sb/02](./questions/languages/java/spring-batch/02-chunk-processing.md) | 10 |
| 6 | Readers | [sb/03](./questions/languages/java/spring-batch/03-readers.md) | 10 |
| 6 | Writers | [sb/04](./questions/languages/java/spring-batch/04-writers.md) | 8 |
| 6 | Processors | [sb/05](./questions/languages/java/spring-batch/05-processors.md) | 6 |
| 7 | Job Execution | [sb/06](./questions/languages/java/spring-batch/06-job-execution.md) | 8 |
| 7 | Transactions & Restart | [sb/07](./questions/languages/java/spring-batch/07-transactions-restart.md) | 7 |
| 7 | Error Handling | [sb/08](./questions/languages/java/spring-batch/08-error-handling.md) | 9 |
| 8 | Tasklet | [sb/09](./questions/languages/java/spring-batch/09-tasklet.md) | 5 |
| 8 | Parallel Processing | [sb/10](./questions/languages/java/spring-batch/10-parallel-processing.md) | 8 |
| 9 | Performance Optimization | [sb/11](./questions/languages/java/spring-batch/11-performance.md) | 7 |
| 9 | Scheduling | [sb/12](./questions/languages/java/spring-batch/12-scheduling.md) | — |
| 9 | Monitoring | [sb/13](./questions/languages/java/spring-batch/13-monitoring.md) | — |
| 10 | Database Metadata | [sb/14](./questions/languages/java/spring-batch/14-database-metadata.md) | — |
| 10 | Production Scenarios | [sb/15](./questions/languages/java/spring-batch/15-production-scenarios.md) | — |

**Cloud, DevOps & Kafka:**

| Day | Topic | File | Questions |
|-----|-------|------|-----------|
| 11 | Observability & Alerting | [cloud/01](./questions/cloud-devops/01-observability-alerting.md) | — |
| 11 | Cloud Infra & Processing | [cloud/02](./questions/cloud-devops/02-cloud-infra-processing.md) | — |
| 12 | DevOps, Kafka Advanced (Ordering, Streams, Dedup, K8s) | [arch/07](./questions/architecture/07-devops-kafka-advanced.md) | 10 |

**Phase 6 done when:** You can describe a production OOM incident resolution, explain Kafka at-least-once vs exactly-once, and design a Spring Batch pipeline with retry.

---

## ⭐ Phase 7 — Interview Simulation
**Weeks 13–16 | ~433 Questions | Goal: Speed + confidence + company patterns**

> 📌 433 Qs across 4 weeks = ~15–20 per day. Pair with HR every Saturday.

**HR & Behavioral (do first — always asked):**

| Topic | File |
|-------|------|
| HR & Behavioral — Project, STAR stories, salary | [hr/01](./questions/hr-behavioral/01-project-behavioral.md) |
| Techno-Managerial Round — Design decisions, prioritization, mentoring | [hr/02](./questions/hr-behavioral/02-techno-managerial-round.md) |

**Company-Specific (match to your target level):**

| Company | Level | File |
|---------|-------|------|
| Capgemini L1 Java | Junior | [cs/16](./questions/company-specific/16-capgemini-l1-java.md) |
| Infosys React | Junior | [cs/12](./questions/company-specific/12-infosys-react-interview.md) |
| Accenture Multi-Round | Mid | [cs/05](./questions/company-specific/05-accenture-interview.md) |
| Accenture Multi-Role | Mid | [cs/15](./questions/company-specific/15-accenture-multi-role-interviews.md) |
| Accenture R1 Java Spring Boot | Mid | [cs/19](./questions/company-specific/19-accenture-java-springboot-r1.md) |
| TCS & Capgemini Java Backend | Mid | [cs/20](./questions/company-specific/20-tcs-capgemini-java-backend.md) |
| HCLTech Java Spring Boot | Mid | [cs/03](./questions/company-specific/03-hcltech-interview.md) |
| EY Java Spring Boot | Mid | [cs/06](./questions/company-specific/06-ey-interview.md) |
| Citi Bank Senior Java | Senior | [cs/17](./questions/company-specific/17-citibank-senior-java.md) |
| Product Company 3 YOE | Senior | [cs/18](./questions/company-specific/18-product-company-java-3yoe.md) |
| Goldman Sachs Java Backend | Senior | [cs/04](./questions/company-specific/04-goldmansachs-interview.md) |
| Bloomberg Senior SWE | Senior | [cs/02](./questions/company-specific/02-bloomberg-interview.md) |
| Amazon SDE-1 | FAANG | [cs/08](./questions/company-specific/08-amazon-sde1-interview.md) |
| Atlassian Senior SWE | FAANG | [cs/01](./questions/company-specific/01-atlassian-interview.md) |
| AmEx Java Backend | Finance | [cs/07](./questions/company-specific/07-amex-interview.md) |
| Bajaj Allianz Full-Stack | Finance | [cs/10](./questions/company-specific/10-bajaj-allianz-interview.md) |
| Deloitte Middleware | Consulting | [cs/13](./questions/company-specific/13-deloitte-interview.md) |
| Capgemini QA Automation | QA | [cs/14](./questions/company-specific/14-capgemini-qa-interview.md) |
| EY React Frontend | Frontend | [cs/09](./questions/company-specific/09-ey-react-interview.md) |
| CGI Frontend React | Frontend | [cs/11](./questions/company-specific/11-cgi-frontend-interview.md) |

**Supporting Areas (do based on role):**

| Topic | File |
|-------|------|
| DSA Coding Problems | [dsa/arrays-strings](./questions/dsa/arrays-strings.md), [dsa/02](./questions/dsa/02-classic-interview-problems.md) |
| Automation Testing / Selenium | [testing/01](./questions/testing/01-automation-testing-selenium.md) |
| AI/ML Fundamentals | [ai-ml/01](./questions/ai-ml/01-ai-ml-fundamentals.md) |
| Angular Scenario-Based | [web-dev/02](./questions/web-dev/02-angular-scenario-based.md) |
| Angular Components & Services | [web-dev/04](./questions/web-dev/04-angular-components-services.md) |
| React & JS Fundamentals | [web-dev/05](./questions/web-dev/05-react-js-fundamentals.md) |
| React Scenario-Based | [web-dev/06](./questions/web-dev/06-react-scenario-based.md) |

---

## 🔄 Phase Assignment Rules
### _When a new file/batch is added, it auto-slots here_

| New content is about... | ➜ Goes into Phase |
|-------------------------|------------------|
| Java basics, OOP, exceptions, collections, access modifiers | **Phase 1** |
| Java 8, Streams, JVM internals, Design Patterns, Serialization | **Phase 2** |
| Multithreading, concurrency, locks, memory model | **Phase 3** |
| Spring Boot, Spring Framework, JPA, Hibernate, Database | **Phase 4** |
| System Design, Microservices, Caching, APIs, Architecture | **Phase 5** |
| Production debugging, Spring Batch, DevOps, Kafka, Cloud | **Phase 6** |
| Company interviews, HR, Testing, AI/ML, Frontend (React/Angular) | **Phase 7** |

> 📌 When new file added → add row to the correct phase table above + update the Day column.

---

## ⚡ Post-Batch Sync Checklist
### _Run this every time a new batch is added — takes < 5 minutes_

```
After adding a new batch, do these 5 steps IN ORDER:

STEP 1 — Update folder README
  ✅ Add new file row to the table (name, questions, link)
  ✅ Update total question count in README heading/badge

STEP 2 — Update parent README(s)
  ✅ java/README.md  → update subtotal for the affected topic
  ✅ languages/README.md → update Java total if Java folder changed

STEP 3 — Update root README.md
  ✅ Update the matching row count in Table of Contents
  ✅ Update Grand Total (Progress Tracker table)
  ✅ Add entry to 🔄 Update Log (Date | Batch | Summary)

STEP 4 — Update daily-updates/README.md
  ✅ Add row under correct month section (March / April / etc.)
  ✅ If new month: add "## 📅 Month YYYY" heading first

STEP 5 — Update STUDY_PLAN.md
  ✅ Add the new file(s) to the correct Phase table above
  ✅ Update memory note: new total question count
```

---

## 🔖 Quick Navigation

| I want to... | Go to |
|---|---|
| Start from the beginning | [Phase 1 — Java Foundation](#-phase-1--java-foundation) |
| Find a specific topic | [`Ctrl+Shift+F`](.) in VS Code to search all files |
| Add new questions | [PROMPT_TEMPLATE.md](./PROMPT_TEMPLATE.md) |
| See all question counts | [README.md](./README.md) |
| See what was added when | [daily-updates/README.md](./daily-updates/README.md) |
| Review the 10-section format | [multithreading/README.md](./questions/languages/java/multithreading/README.md) |

---

[← Back to Home](./README.md)
