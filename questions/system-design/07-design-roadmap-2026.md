# 🗺️ System Design Roadmap 2026 — 50 Topics (Simple → Complex)

> Perfect learning roadmap for system design interviews. Topics organized from foundational to planet-scale, with cross-links to existing detailed Q&As where available.

---

## 📊 Coverage Status

| Status | Count | Meaning |
|--------|-------|---------|
| ✅ Covered | 21 | Full Q&A exists in this repo |
| 🟡 Partial | 5 | Related content exists |
| 📝 New | 24 | Topic listed for future deep-dive |

---

## 🟢 Beginner — Building Blocks (1–10)

| # | Topic | Status | Cross-Link |
|---|-------|--------|------------|
| 1 | Design a Rate Limiter | ✅ | [Atlassian Q8](../company-specific/01-atlassian-interview.md#q8), [Goldman Sachs Q8](../company-specific/04-goldmansachs-interview.md#q8) |
| 2 | Design a URL Shortener | ✅ | [Product Designs Q6](./04-product-company-designs.md#q6) |
| 3 | Design Pastebin | 📝 | Similar to URL Shortener + blob storage |
| 4 | Design a Unique ID Generator | 🟡 | Related: [Distributed Systems Q5](./01-distributed-systems-fundamentals.md#q5) (sharding/consistent hashing) |
| 5 | Design Consistent Hashing | ✅ | [Distributed Systems Q5](./01-distributed-systems-fundamentals.md#q5) |
| 6 | Design a Load Balancer | 🟡 | Covered in multiple architecture discussions |
| 7 | Design an API Gateway | ✅ | [Product Designs Q16](./04-product-company-designs.md#q16), [HCLTech Q28](../company-specific/03-hcltech-interview.md#q28) |
| 8 | Design a Basic Key-Value Store | 📝 | Foundation for distributed databases |
| 9 | Design a Caching System (LRU Cache) | 🟡 | Related: [Architecture Q2](../languages/java/architecture/02-caching-architecture-patterns.md) |
| 10 | Design a Notification System | ✅ | [Product Designs Q8](./04-product-company-designs.md#q8) |

---

## 🟡 Intermediate — Core Systems (11–20)

| # | Topic | Status | Cross-Link |
|---|-------|--------|------------|
| 11 | Design a Typeahead/Autocomplete | ✅ | [Product Designs Q13](./04-product-company-designs.md#q13) |
| 12 | Design a Web Crawler | ✅ | [Atlassian Q9](../company-specific/01-atlassian-interview.md#q9) |
| 13 | Design a Message Queue | 📝 | Kafka, RabbitMQ, SQS comparison |
| 14 | Design a 1:1 Chat System | ✅ | [Product Designs Q5](./04-product-company-designs.md#q5) |
| 15 | Design a Group Chat System | 🟡 | Extension of [Product Designs Q5](./04-product-company-designs.md#q5) |
| 16 | Design a News Feed System | 📝 | Push vs Pull model, fan-out |
| 17 | Design a Proximity Service (nearby friends) | 📝 | Geohash, QuadTree, S2 cells |
| 18 | Design Instagram (photo/video + feed) | 📝 | CDN + feed + media processing |
| 19 | Design Twitter/X (timeline + posts) | 📝 | Fan-out on write vs read |
| 20 | Design WhatsApp (real-time messaging) | 📝 | WebSocket, end-to-end encryption |

---

## 🔴 Advanced — Product Systems (21–30)

| # | Topic | Status | Cross-Link |
|---|-------|--------|------------|
| 21 | Design Dropbox (file storage & sync) | 📝 | Block-level sync, deduplication |
| 22 | Design a Ticket Booking System | 📝 | Concurrency, double booking prevention |
| 23 | Design an E-commerce Platform | ✅ | [E-Commerce Designs](./03-ecommerce-payment-systems.md) |
| 24 | Design a Recommendation System | 📝 | Collaborative + content-based filtering |
| 25 | Design a Distributed Cache | ✅ | [Product Designs Q7](./04-product-company-designs.md#q7) |
| 26 | Design Uber (ride-sharing + matching) | ✅ | [Product Designs Q2](./04-product-company-designs.md#q2) |
| 27 | Design Netflix (video streaming) | ✅ | [Product Designs Q1](./04-product-company-designs.md#q1) |
| 28 | Design YouTube (video upload + streaming) | 📝 | Transcoding pipeline, CDN |
| 29 | Design TikTok (short-video platform) | 📝 | Recommendation + video processing |
| 30 | Design Facebook News Feed | 📝 | Similar to #16 but at Facebook scale |

---

## ⚫ Expert — Infrastructure & Platform (31–40)

| # | Topic | Status | Cross-Link |
|---|-------|--------|------------|
| 31 | Design Google Docs (real-time collab) | 📝 | OT / CRDT algorithms |
| 32 | Design a CDN | ✅ | [Product Designs Q3](./04-product-company-designs.md#q3) |
| 33 | Design a Search Engine | 📝 | Inverted index, ranking, crawling |
| 34 | Design Google Maps (routing + location) | 📝 | Graph algorithms, tile serving |
| 35 | Design a Distributed Database | ✅ | [Distributed Systems Q6](./01-distributed-systems-fundamentals.md#q6) |
| 36 | Design a Real-time Analytics System | ✅ | [Product Designs Q17](./04-product-company-designs.md#q17) |
| 37 | Design an Ad Serving & Tracking System | 📝 | RTB, click tracking, attribution |
| 38 | Design a Fraud Detection System | 📝 | ML pipeline + rules engine |
| 39 | Design a Stock Trading/Exchange System | 🟡 | Related: [JPMorgan Designs](./06-jpmorgan-banking-designs.md) |
| 40 | Design a Distributed Job Scheduler | ✅ | [Product Designs Q15](./04-product-company-designs.md#q15) |

---

## 🌟 Master — Planet-Scale (41–50)

| # | Topic | Status | Cross-Link |
|---|-------|--------|------------|
| 41 | Design Event Sourcing + CQRS | ✅ | [Coordination & Event Sourcing](./02-coordination-failover-eventsourcing.md) |
| 42 | Design a Multi-tenant SaaS Platform | ✅ | [Product Designs Q11](./04-product-company-designs.md#q11) |
| 43 | Design Live Video Streaming at Scale | 📝 | HLS/DASH, adaptive bitrate, latency |
| 44 | Design a Highly Scalable NoSQL DB | 📝 | LSM tree, compaction, replication |
| 45 | Design a Multiplayer Game Backend | 📝 | Game loop, state sync, lag compensation |
| 46 | Design ML Model Serving Infra | 📝 | Feature store, A/B testing, canary |
| 47 | Design a Geo-distributed Low-Latency System | 📝 | Multi-region, edge computing |
| 48 | Design a Strongly Consistent Global DB | 📝 | Spanner-like, TrueTime, Paxos |
| 49 | Design a High-Frequency Trading Platform | 📝 | Ultra-low latency, kernel bypass |
| 50 | Design a Planet-Scale Distributed System | 📝 | Billions of users, multi-region HA |

---

## 🎯 Study Strategy

```
Week 1-2:  Topics 1-10  (Building blocks — nail these first)
Week 3-4:  Topics 11-20 (Core systems — most commonly asked)
Week 5-6:  Topics 21-30 (Product systems — company favorites)
Week 7-8:  Topics 31-40 (Infrastructure — for senior roles)
Week 9-10: Topics 41-50 (Planet-scale — for staff+ roles)
```

### Key Patterns That Repeat

| Pattern | Used In |
|---------|---------|
| **Consistent Hashing** | #5, #8, #25, #35, #44, #48 |
| **Fan-out (Push vs Pull)** | #16, #19, #30 |
| **CDN + Edge Caching** | #27, #28, #29, #32, #43 |
| **Message Queue / Event Bus** | #10, #13, #22, #38, #41 |
| **Rate Limiting / Throttling** | #1, #7, #37, #49 |
| **Geospatial Indexing** | #17, #26, #34, #47 |
| **Leader Election / Consensus** | #35, #40, #48 |
| **CRDT / OT** | #14, #15, #20, #31 |
| **Inverted Index** | #11, #33 |
| **Write-Ahead Log** | #35, #41, #44, #48 |

---

## 📚 Recommended Resources

- **System Design Interview** — Alex Xu (Vol 1 & 2)
- **Designing Data-Intensive Applications** — Martin Kleppmann
- **Building Microservices** — Sam Newman
- **System Design Primer** — github.com/donnemartin/system-design-primer

---

[← Back to System Design](./README.md) | [← Back to Home](../../README.md)
