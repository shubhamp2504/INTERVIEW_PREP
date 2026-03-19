# 🏗️ System Design — PhonePe & Fintech Interview Problems

> System design questions commonly asked at PhonePe, Paytm, and fintech companies.

---

## Q1. Design a UPI-Style Payment Service

### 📝 One-Liner
Design a payment system like UPI that handles peer-to-peer transfers, merchant payments, and bank integrations.

### 🔑 Quick Answer
VPA-based addressing, two-step collect/pay flow, idempotent transaction processing, bank adapter layer, NPCI-like switch for routing. *(VPA se identify karo, 2-step mein pay karo, idempotency se duplicate rok do)*

### 📖 How It Works
1. **User Registration**: Link bank account → create VPA (user@bank)
2. **Pay Flow**: Payer initiates → debit payer's bank → credit payee's bank
3. **Collect Flow**: Payee requests → payer approves → same debit/credit
4. **Transaction Service**: Idempotency key per txn, state machine (INITIATED → PROCESSING → SUCCESS/FAILED)
5. **Bank Adapter**: Abstract different bank APIs behind unified interface
6. **Settlement**: End-of-day batch settlement between banks
7. **Fraud Detection**: Real-time scoring on each transaction

### 🗣️ How to Say in Interview
"The key challenge is ensuring exactly-once payment processing in a distributed system. I'd use an idempotency key per transaction, a state machine for transaction lifecycle, and a bank adapter layer to abstract different bank integrations. For reliability, I'd use Kafka with transactional producers and idempotent consumers."

### 💻 Code
```java
// Transaction State Machine
public enum TxnState {
    INITIATED, DEBIT_PENDING, DEBIT_SUCCESS, CREDIT_PENDING, 
    CREDIT_SUCCESS, FAILED, REFUND_PENDING, REFUNDED
}

// Idempotent transaction processing
@Transactional
public TxnResponse processPayment(PaymentRequest req) {
    // Check idempotency
    Optional<Transaction> existing = txnRepo.findByIdempotencyKey(req.getIdempotencyKey());
    if (existing.isPresent()) return existing.get().toResponse();
    
    Transaction txn = Transaction.create(req);
    txn.setState(TxnState.DEBIT_PENDING);
    txnRepo.save(txn);
    
    // Async debit via bank adapter
    bankAdapter.debit(txn);
    return txn.toResponse();
}
```

### ⚠️ Pitfalls / Gotchas
- Double debit: idempotency key is CRITICAL *(double payment sabse bada risk)*
- Bank timeout: need reconciliation job to check final status
- Scaling: partition by user_id, not txn_id
- Compliance: PCI DSS, RBI guidelines

### ⚡ Remember
- Idempotency > everything in payments
- State machine for transaction lifecycle
- Saga pattern for distributed transactions
- Reconciliation: daily batch to catch discrepancies

---

## Q2. Design a Transaction Feed Service (TStore)

### 📝 One-Liner
A service that provides users their transaction history with filters, search, and real-time updates.

### 🔑 Quick Answer
Event-sourced transaction store, read-optimized projections per user, cursor-based pagination, real-time updates via WebSocket. *(transaction hota hai → event store mein jaata hai → user ki feed update hoti hai)*

### 📖 How It Works
1. **Ingestion**: Transaction events from payment service → Kafka → consumer
2. **Storage**: Write to event store (immutable log) + materialized view per user
3. **Read Model**: Per-user feed in Cassandra (partition by user_id, sorted by timestamp)
4. **Filters**: By date range, amount, category, merchant
5. **Search**: Elasticsearch for full-text search on merchant names
6. **Real-time**: WebSocket push for new transactions

### 🗣️ How to Say in Interview
"I'd use an event-sourced architecture where every transaction is an immutable event. A consumer builds per-user materialized views in Cassandra for fast reads. The API supports cursor-based pagination with filters. New transactions push real-time updates via WebSocket."

### ⚡ Remember
- CQRS: separate write (event store) and read (materialized views) models
- Cassandra: excellent for time-series per-user queries
- Cursor pagination: `WHERE user_id = ? AND timestamp < ? LIMIT 20`
- Eventually consistent: transaction appears in feed within seconds

---

## Q3. Design In-App Inbox/Alerts Pub-Sub (Bullhorn)

### 📝 One-Liner
A notification inbox system where users see alerts, promotions, and transactional messages in-app.

### 🔑 Quick Answer
Pub-sub with per-user inbox, priority-based ordering, read/unread tracking, expiry management. *(har user ki inbox, pub-sub se populate hoti hai)*

### 📖 How It Works
1. **Publishers**: Various services publish notifications to topics
2. **Router**: Consumes from Kafka, determines recipients, applies rules
3. **Inbox Store**: Per-user inbox in Redis (recent) + Cassandra (archive)
4. **Read Status**: Track read/unread per message per user
5. **Priority**: Transactional > Alerts > Promotions
6. **Expiry**: TTL per message type, background cleanup

### 🗣️ How to Say in Interview
"I'd design a fan-out notification system. Publishers send events to Kafka topics. A routing service determines the target users and writes to their inboxes. The inbox has a hot layer in Redis for recent messages and cold storage in Cassandra. The API supports pagination, filtering, and mark-as-read."

### ⚡ Remember
- Fan-out: system writes to each user's inbox (push model)
- Unread count: atomic counter in Redis per user
- Batch processing: bulk notifications via async workers
- Rate limiting: don't spam users

---

## Q4. Design a Job Scheduler at Scale (Clockwork)

### 📝 One-Liner
A distributed job scheduler that executes millions of scheduled tasks reliably and on time.

### 🔑 Quick Answer
Partitioned time-wheel + distributed locking + idempotent workers. Jobs stored in DB, scheduled via sorted set, executed by worker fleet. *(time wheel se schedule, lock se ek baar execute, worker fleet se scale)*

### 📖 How It Works
1. **Job Registration**: API to create/update/delete scheduled jobs (cron or one-time)
2. **Time Wheel**: Redis sorted set with `next_fire_time` as score
3. **Scheduler**: Multiple scheduler instances, leader-elected, scan for due jobs
4. **Dispatch**: Due jobs → Kafka topic → worker pool consumes
5. **Execution**: Worker acquires lock → execute → update status
6. **Retry**: Failed jobs → retry with backoff → DLQ after max retries
7. **Observability**: Latency, success rate, queue depth metrics

### 🗣️ How to Say in Interview
"I'd partition the job space and use Redis sorted sets for the schedule queue, scored by next-fire-time. A leader-elected scheduler polls for due jobs and dispatches to Kafka. Workers consume, acquire a distributed lock, and execute idempotently. This gives us exactly-once execution at scale."

### ⚡ Remember
- Sorted set: O(log n) insert, O(1) pop min *(efficient time-wheel)*
- Leader election: ZooKeeper or Redis Redlock
- Idempotent execution: even if dispatched twice, safe
- Millions of jobs: partition by job_type or tenant

---

## Q5. Design a Metrics Platform for SLOs

### 📝 One-Liner
A platform to define, track, and alert on Service Level Objectives across all services.

### 🔑 Quick Answer
Collect metrics (latency, errors, availability) → aggregate → compute SLI → compare against SLO → alert on budget burn. *(metrics collect karo, SLI calculate karo, SLO se compare karo, alert do)*

### 📖 How It Works
1. **Metrics Collection**: Prometheus scrapes or push-based (StatsD/OpenTelemetry)
2. **SLI Computation**: e.g., % requests < 200ms (latency SLI), error rate < 0.1% 
3. **SLO Definition**: 99.9% availability over 30-day rolling window
4. **Error Budget**: 100% - SLO = budget. Track remaining budget
5. **Alerting**: Multi-window burn rate alerts (fast burn = page, slow burn = ticket)
6. **Dashboard**: Budget remaining, historical SLI trends, incident correlation

### 🗣️ How to Say in Interview
"I'd build on OpenTelemetry for metrics collection, computing SLIs from raw metrics. SLOs are defined as targets over rolling windows. The key innovation is error budget tracking — when the budget burns too fast, we alert. This shifts focus from individual incidents to overall service health."

### ⚡ Remember
- SLI = measured indicator, SLO = target, SLA = agreement with consequences
- Error budget = 100% - SLO target
- Multi-window burn rate: catch both fast and slow degradations
- Google SRE book: canonical reference for SLO-based alerting

---
