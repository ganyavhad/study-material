# 01 — System Design Fundamentals

> Core building blocks and vocabulary for designing large-scale distributed systems.

---

## Table of Contents

1. [The RESHADE Framework](#1-the-reshade-framework)
2. [CAP Theorem](#2-cap-theorem)
3. [ACID vs BASE](#3-acid-vs-base)
4. [Latency, Throughput & Availability](#4-latency-throughput--availability)
5. [Consistency Models](#5-consistency-models)
6. [Back-of-the-Envelope Estimation](#6-back-of-the-envelope-estimation)
7. [Key Building Blocks](#7-key-building-blocks)
8. [Interview Questions & Answers](#8-interview-questions--answers)

---

## 1. The RESHADE Framework

Use this structure in every system design interview:

```
┌──────────────────────────────────────────────────────────────────┐
│  R — REQUIREMENTS (5 min)                                        │
│  • Functional: What does the system DO?                          │
│  • Non-Functional: Scale, latency, availability, consistency     │
│  • Constraints: Budget, team size, timeline                      │
├──────────────────────────────────────────────────────────────────┤
│  E — ESTIMATION (3 min)                                          │
│  • DAU, QPS (queries/sec), storage, bandwidth                    │
│  • Read:Write ratio, peak vs average traffic                     │
├──────────────────────────────────────────────────────────────────┤
│  S — SYSTEM API (3 min)                                          │
│  • Key endpoints (REST/GraphQL)                                  │
│  • Request/Response shapes                                       │
├──────────────────────────────────────────────────────────────────┤
│  H — HIGH-LEVEL ARCHITECTURE (10 min)                            │
│  • Draw boxes: clients, LB, servers, DB, cache, queue            │
│  • Show data flow arrows                                         │
├──────────────────────────────────────────────────────────────────┤
│  A — DETAILED COMPONENT DESIGN (10 min)                          │
│  • Deep dive into 2-3 critical components                        │
│  • Data structures, algorithms, protocols                        │
├──────────────────────────────────────────────────────────────────┤
│  D — DATA MODEL & STORAGE (5 min)                                │
│  • Schema design (SQL/NoSQL)                                     │
│  • Indexing strategy, partitioning                               │
├──────────────────────────────────────────────────────────────────┤
│  E — EVALUATE TRADE-OFFS (5 min)                                 │
│  • Bottlenecks, failure modes, scaling limits                    │
│  • What would you change at 10x / 100x scale?                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. CAP Theorem

```
In a distributed system, you can guarantee at most TWO of three:

        Consistency
           /\
          /  \
         /    \
        / CP   \ CA
       /        \
      /____AP____\
  Partition      Availability
  Tolerance

C — Consistency: Every read receives the most recent write
A — Availability: Every request receives a response (may be stale)
P — Partition Tolerance: System continues despite network failures

In distributed systems, partitions WILL happen → you must choose CP or AP.
```

### Real-World Choices

```
┌─────────────────────┬────────┬────────────────────────────────────┐
│ System              │ Choice │ Reasoning                          │
├─────────────────────┼────────┼────────────────────────────────────┤
│ Banking / Payments  │ CP     │ Can't show wrong balance           │
│ Social Media Feed   │ AP     │ Stale post OK, must always load    │
│ E-commerce Cart     │ AP     │ Eventually consistent, always up   │
│ Inventory Count     │ CP     │ Overselling is worse than downtime │
│ DNS                 │ AP     │ Stale records OK, must resolve     │
│ User Auth (JWT)     │ AP     │ Token validation works offline     │
│ MongoDB (default)   │ CP     │ Primary handles writes             │
│ DynamoDB            │ AP     │ Eventually consistent reads        │
│ PostgreSQL          │ CP     │ ACID transactions                  │
└─────────────────────┴────────┴────────────────────────────────────┘
```

---

## 3. ACID vs BASE

```
ACID (SQL — PostgreSQL)          BASE (NoSQL — MongoDB, DynamoDB)
─────────────────────            ────────────────────────────────
A — Atomicity                    BA — Basically Available
    All or nothing                    System guarantees availability
C — Consistency                  S  — Soft State
    Valid state transitions           State may change over time
I — Isolation                    E  — Eventual Consistency
    Concurrent txns independent       Will converge eventually
D — Durability
    Committed = persisted

When to use which:
• ACID: Financial transactions, inventory, order management
• BASE: Social feeds, analytics, logging, session storage
```

```typescript
// ACID example — PostgreSQL transaction (NestJS + TypeORM)
async function transferFunds(fromId: string, toId: string, amount: number) {
  const queryRunner = dataSource.createQueryRunner();
  await queryRunner.startTransaction();

  try {
    await queryRunner.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1',
      [amount, fromId]
    );
    await queryRunner.query(
      'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
      [amount, toId]
    );
    await queryRunner.commitTransaction();
  } catch (err) {
    await queryRunner.rollbackTransaction(); // Atomicity
    throw err;
  } finally {
    await queryRunner.release();
  }
}

// BASE example — MongoDB eventual consistency
// Write to primary, read from secondary (may be stale)
const result = await collection.find({ userId })
  .readPreference('secondaryPreferred') // AP: available but may be stale
  .toArray();
```

---

## 4. Latency, Throughput & Availability

### Latency Numbers Every Developer Should Know

```
Operation                         │ Time
──────────────────────────────────┼──────────────
L1 cache reference                │ 0.5 ns
L2 cache reference                │ 7 ns
Main memory reference             │ 100 ns
SSD random read                   │ 16 μs
HDD random read                  │ 2 ms
Round trip same datacenter        │ 0.5 ms
Send 1MB over 1Gbps network      │ 10 ms
Round trip US East → West         │ 40 ms
Round trip US → Europe            │ 80 ms
Round trip US → India             │ 150 ms
```

### Availability & SLA

```
Availability │ Downtime/Year  │ Downtime/Month │ Downtime/Week
─────────────┼────────────────┼────────────────┼──────────────
99%          │ 3.65 days      │ 7.3 hours      │ 1.68 hours
99.9%        │ 8.77 hours     │ 43.8 min       │ 10.1 min
99.99%       │ 52.6 min       │ 4.4 min        │ 1.01 min
99.999%      │ 5.26 min       │ 26.3 sec       │ 6.05 sec

Achieving high availability:
• Redundancy (multi-AZ, multi-region)
• Health checks + auto-failover
• Graceful degradation
• Circuit breakers
```

### Throughput Calculation

```typescript
// Example: API server capacity planning
// Given:
//   - Each request takes 50ms average
//   - Node.js single thread (but non-blocking I/O)
//   - Cluster mode with 4 workers

// Theoretical max QPS per worker:
// If 50ms per request, purely sequential: 1000/50 = 20 QPS
// But Node.js is async → can handle ~200 concurrent I/O-bound requests
// Per worker: ~200 * (1000/50) = ~4000 QPS (I/O bound)
// 4 workers: ~16,000 QPS

// Real world with overhead: expect ~5,000-10,000 QPS per instance
```

---

## 5. Consistency Models

```
Strong ─────────────────────────────────────────── Eventual
  │                                                    │
  ├─ Linearizable     (single leader, sync replication)
  ├─ Sequential       (all see same order)
  ├─ Causal           (cause before effect)
  ├─ Read-your-writes (see your own writes)
  ├─ Monotonic reads  (never see older data)
  └─ Eventual         (converges... eventually)

Node.js examples:
• PostgreSQL: Strong consistency (SERIALIZABLE isolation)
• MongoDB w/ readConcern "majority": Read-your-writes
• DynamoDB: Eventual by default, strong on request
• Redis Cluster: Eventual (async replication)
```

---

## 6. Back-of-the-Envelope Estimation

### Quick Reference

```
1 Million requests/day  ≈ 12 QPS
10 Million requests/day ≈ 120 QPS
100 Million requests/day ≈ 1,200 QPS
1 Billion requests/day  ≈ 12,000 QPS

Storage:
1 char = 1 byte (ASCII) / 2-4 bytes (UTF-8 with special chars)
1 KB ≈ short text (tweet, chat message)
1 MB ≈ small image, long document
1 GB = 1,000 MB
1 TB = 1,000 GB

Peak traffic ≈ 2-3x average (sometimes 10x for flash sales)
```

### Example: Design a URL Shortener

```
Requirements:
• 100M URLs created/month
• 10:1 read:write ratio
• URLs stored for 5 years

Estimation:
• Writes: 100M / (30 * 24 * 3600) ≈ 40 writes/sec
• Reads:  400 reads/sec (10x writes)
• Peak:   ~1,200 reads/sec (3x average)

Storage:
• Each URL record: ~500 bytes (short URL + long URL + metadata)
• 5 years: 100M * 12 * 5 = 6 Billion records
• Total: 6B * 500B = 3TB

Bandwidth:
• Write: 40 * 500B = 20 KB/sec (negligible)
• Read:  400 * 500B = 200 KB/sec (negligible)
```

---

## 7. Key Building Blocks

```
┌──────────────────────┬──────────────────────────────────────────┐
│ Building Block       │ Purpose                                  │
├──────────────────────┼──────────────────────────────────────────┤
│ Load Balancer        │ Distribute traffic across servers         │
│ CDN                  │ Serve static content close to users       │
│ API Gateway          │ Auth, rate limiting, request routing      │
│ Application Server   │ Business logic (Node.js / NestJS)        │
│ Relational DB        │ Structured data, ACID (PostgreSQL)       │
│ NoSQL DB             │ Flexible schema, scale (MongoDB/Dynamo)  │
│ Cache                │ Fast reads, reduce DB load (Redis)       │
│ Message Queue        │ Async processing (SQS, Kafka, BullMQ)   │
│ Object Storage       │ Files, images, backups (S3)              │
│ Search Engine        │ Full-text search (Elasticsearch)         │
│ Monitoring           │ Metrics, alerts (CloudWatch, Datadog)    │
│ DNS                  │ Domain resolution (Route 53)             │
└──────────────────────┴──────────────────────────────────────────┘
```

### High-Level Architecture Template

```
                    ┌─────────┐
                    │   DNS   │
                    └────┬────┘
                         │
                    ┌────▼────┐
        ┌──────────│   CDN   │──────────┐
        │          └────┬────┘          │
        │ (static)      │ (dynamic)     │
        │          ┌────▼────┐          │
        │          │   ALB   │          │
        │          └────┬────┘          │
        │      ┌────────┼────────┐      │
        │  ┌───▼──┐ ┌───▼──┐ ┌──▼───┐  │
        │  │ App1 │ │ App2 │ │ App3 │  │
        │  └──┬───┘ └──┬───┘ └──┬───┘  │
        │     └────────┼────────┘      │
        │         ┌────▼────┐          │
        │         │  Redis  │          │
        │         └────┬────┘          │
        │    ┌─────────┼─────────┐     │
        │ ┌──▼───┐  ┌──▼───┐ ┌──▼──┐  │
        │ │Pg-RW │  │Pg-RO │ │Mongo│  │
        │ └──────┘  └──────┘ └─────┘  │
        │              │               │
        │         ┌────▼────┐          │
        │         │  SQS /  │          │
        │         │  Kafka  │          │
        │         └────┬────┘          │
        │         ┌────▼────┐          │
        └─────────│   S3    │──────────┘
                  └─────────┘
```

---

## 8. Interview Questions & Answers

### Q1: How do you approach a system design problem?

**Answer:**
Use the **RESHADE** framework:
1. Clarify **requirements** — ask 3-5 questions (What's the scale? Read-heavy or write-heavy? Latency requirements?).
2. **Estimate** traffic, storage, bandwidth.
3. Define **System APIs**.
4. Draw **high-level architecture** on the whiteboard.
5. **Deep dive** into 2-3 critical components.
6. Define **data model** and database choices.
7. **Evaluate** trade-offs, bottlenecks, and future scaling.

### Q2: Explain CAP theorem with a real example.

**Answer:**
In a payment system (Stripe integration), we choose **CP** — consistency over availability. If a network partition occurs between our Node.js service and PostgreSQL, we'd rather return an error than process a duplicate charge. We use database transactions (`SERIALIZABLE` isolation) and idempotency keys to ensure exactly-once processing. The system may be briefly unavailable during a partition, but it never shows incorrect balances.

### Q3: How do you calculate QPS from daily active users?

**Answer:**
- **DAU → Daily requests:** DAU × actions_per_user (e.g., 10M × 5 = 50M requests/day)
- **Average QPS:** 50M / 86,400 ≈ 580 QPS
- **Peak QPS:** 580 × 3 ≈ 1,740 QPS (assume 3x peak)
- **Rule of thumb:** 1M requests/day ≈ 12 QPS average

### Q4: What's the difference between horizontal and vertical scaling?

**Answer:**
- **Vertical (scale up):** Bigger machine (more CPU/RAM). Simple but has limits. Good for databases.
- **Horizontal (scale out):** More machines behind a load balancer. No single-machine limit. Requires stateless services.
- Node.js is ideal for horizontal scaling — stateless, event-driven, low per-instance memory.
- We horizontally scale our NestJS services behind an ALB, with state in Redis/PostgreSQL.

### Q5: How do you ensure high availability?

**Answer:**
- **No single point of failure:** Multi-AZ deployment, database replicas, redundant load balancers.
- **Health checks:** ALB health checks remove unhealthy instances.
- **Auto-scaling:** Scale based on CPU/memory/request count.
- **Circuit breakers:** Prevent cascade failures (using libraries like `opossum` in Node.js).
- **Graceful degradation:** If recommendation service is down, show popular items instead.
- At Accenture, we achieved 99.9% uptime using multi-AZ Lambda + RDS with auto-failover.

---

**Next:** [02 — Scalability & Load Balancing →](02-scalability-load-balancing.md)
