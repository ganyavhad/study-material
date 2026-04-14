# Section 3 — System Design & Scenario-Based Questions

> Architect scalable, reliable systems — mapped to real-world Node.js/AWS/microservices experience.
> **Ganesh Avhad** — Software Developer Specialist | 8+ Years Experience

---

## Why This Section?

System design interviews test your ability to:
- Design **large-scale distributed systems** end-to-end
- Make **trade-off decisions** (consistency vs availability, SQL vs NoSQL)
- Handle **scenario-based** questions that probe depth of experience
- Communicate architecture decisions clearly with diagrams and reasoning

---

## Study Modules

| # | Topic | File | Key Concepts |
|---|-------|------|--------------|
| 01 | **System Design Fundamentals** | [01-system-design-fundamentals.md](01-system-design-fundamentals.md) | CAP Theorem, ACID/BASE, Latency, Throughput, SLAs, Design Framework |
| 02 | **Scalability & Load Balancing** | [02-scalability-load-balancing.md](02-scalability-load-balancing.md) | Horizontal/Vertical Scaling, ALB/NLB, Auto Scaling, CDN, Rate Limiting |
| 03 | **Database Design & Scaling** | [03-database-design-scaling.md](03-database-design-scaling.md) | SQL vs NoSQL, Sharding, Replication, Partitioning, Read Replicas, CQRS |
| 04 | **Caching Strategies** | [04-caching-strategies.md](04-caching-strategies.md) | Redis, Cache-Aside, Write-Through, TTL, Cache Invalidation, CDN Caching |
| 05 | **Message Queues & Async Processing** | [05-message-queues-async.md](05-message-queues-async.md) | SQS, Kafka, BullMQ, Event-Driven, Pub/Sub, DLQ, Idempotency |
| 06 | **Microservices & Distributed Patterns** | [06-microservices-distributed-patterns.md](06-microservices-distributed-patterns.md) | Saga, Circuit Breaker, Service Discovery, API Gateway, Distributed Tracing |
| 07 | **API Design & Security** | [07-api-design-security.md](07-api-design-security.md) | REST vs GraphQL, Versioning, Auth (JWT/OAuth), Rate Limiting, CORS |
| 08 | **Real-World System Design Problems** | [08-real-world-system-designs.md](08-real-world-system-designs.md) | URL Shortener, Chat System, Notification Service, Payment System, LMS |
| 09 | **Scenario-Based Interview Questions** | [09-scenario-based-questions.md](09-scenario-based-questions.md) | Production Debugging, Performance, Migration, Outage Handling, Trade-offs |

---

## System Design Interview Framework (RESHADE)

```
R — Requirements (Functional + Non-Functional)
E — Estimation (Traffic, Storage, Bandwidth)
S — System API Design
H — High-Level Architecture (Diagram)
A — Detailed Component Design
D — Data Model & Storage
E — Evaluate Trade-offs & Bottlenecks
```

---

## Recommended Study Path

```
Week 1-2: Fundamentals → Scalability → Database Design
Week 3-4: Caching → Message Queues → Microservices Patterns
Week 5-6: API Design → Real-World System Designs (practice 2-3)
Week 7-8: Scenario-Based Questions → Mock Interviews
```

---

## How to Use

- Each module contains **concepts → diagrams → code patterns → interview answers**
- System designs include **ASCII architecture diagrams** you can redraw on whiteboards
- Scenario questions have structured **STAR-format** answers drawn from real experience
- All patterns are grounded in your **Node.js / NestJS / AWS / PostgreSQL / MongoDB** stack

---
