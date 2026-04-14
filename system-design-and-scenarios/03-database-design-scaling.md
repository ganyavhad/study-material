# 03 — Database Design & Scaling

> Choosing the right database, designing schemas, and scaling data layers for millions of users.

---

## Table of Contents

1. [SQL vs NoSQL Decision Matrix](#1-sql-vs-nosql-decision-matrix)
2. [Schema Design Patterns](#2-schema-design-patterns)
3. [Indexing Strategies](#3-indexing-strategies)
4. [Replication](#4-replication)
5. [Sharding & Partitioning](#5-sharding--partitioning)
6. [CQRS Pattern](#6-cqrs-pattern)
7. [Database Per Service (Microservices)](#7-database-per-service-microservices)
8. [Interview Questions & Answers](#8-interview-questions--answers)

---

## 1. SQL vs NoSQL Decision Matrix

```
┌────────────────────┬────────────────────┬──────────────────────┐
│ Factor             │ SQL (PostgreSQL)   │ NoSQL (MongoDB)      │
├────────────────────┼────────────────────┼──────────────────────┤
│ Data Structure     │ Structured, fixed  │ Flexible, evolving   │
│ Relationships      │ Complex JOINs      │ Embed or reference   │
│ Transactions       │ Full ACID          │ Document-level ACID  │
│ Schema             │ Enforced           │ Schema-less          │
│ Scale              │ Vertical → RR      │ Horizontal (shards)  │
│ Query Language     │ SQL                │ MongoDB Query/Agg    │
│ Use Cases          │ Finance, ERP, CRM  │ CMS, IoT, catalogs   │
│ Consistency        │ Strong             │ Tunable              │
│ Read Performance   │ With indexes       │ Excellent (embedded) │
│ Write Performance  │ Good               │ Excellent            │
└────────────────────┴────────────────────┴──────────────────────┘

When BOTH: Use PostgreSQL for transactional data + MongoDB for flexible content
Example: orders in PostgreSQL, course content in MongoDB
```

### Multi-Database Architecture

```
            ┌──────────────────────────────────┐
            │         Application Layer        │
            │         (NestJS / Node.js)        │
            └──────┬───────┬────────┬──────────┘
                   │       │        │
          ┌────────▼──┐ ┌──▼────┐ ┌─▼──────────┐
          │PostgreSQL │ │MongoDB│ │ElasticSearch│
          │           │ │       │ │             │
          │• Users    │ │• CMS  │ │• Full-text  │
          │• Orders   │ │• Logs │ │  search     │
          │• Payments │ │• Docs │ │• Analytics  │
          └───────────┘ └───────┘ └─────────────┘
```

---

## 2. Schema Design Patterns

### Relational (PostgreSQL)

```sql
-- E-commerce: Normalized schema with indexes
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  name VARCHAR(100) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),
  total_amount DECIMAL(10,2) NOT NULL,
  status VARCHAR(20) DEFAULT 'pending',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE order_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL REFERENCES orders(id),
  product_id UUID NOT NULL REFERENCES products(id),
  quantity INT NOT NULL CHECK (quantity > 0),
  unit_price DECIMAL(10,2) NOT NULL
);

-- Indexes for common queries
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
```

### Document (MongoDB)

```typescript
// MongoDB: Embed frequently accessed data, reference for large/changing data

// Embedded pattern (1:few, read-heavy)
const courseSchema = {
  _id: ObjectId,
  title: "Node.js Fundamentals",
  instructor: {               // Embedded — always read together
    name: "Ganesh Avhad",
    avatar: "url",
  },
  modules: [                   // Embedded — bounded array
    {
      title: "Event Loop",
      duration: 30,
      lessons: [
        { title: "Lesson 1", videoUrl: "..." },
      ],
    },
  ],
  enrollmentCount: 1500,
};

// Referenced pattern (1:many, write-heavy)
const userSchema = {
  _id: ObjectId,
  email: "user@example.com",
  name: "John",
};

const reviewSchema = {
  _id: ObjectId,
  courseId: ObjectId,          // Reference to course
  userId: ObjectId,           // Reference to user
  rating: 5,
  comment: "Great course!",
  createdAt: new Date(),
};
// reviews collection can grow unbounded — never embed
```

---

## 3. Indexing Strategies

```
Index Types (PostgreSQL):
┌──────────────────┬──────────────────────────────────────────┐
│ Type             │ Use Case                                 │
├──────────────────┼──────────────────────────────────────────┤
│ B-Tree (default) │ Equality, range, sorting                 │
│ Hash             │ Equality only (faster than B-Tree)       │
│ GIN              │ Full-text search, JSONB, arrays          │
│ GiST             │ Geometric, range types, PostGIS          │
│ BRIN             │ Large tables with naturally ordered data  │
│ Partial          │ Index subset of rows (WHERE clause)      │
│ Covering         │ Include non-key columns (index-only scan)│
│ Composite        │ Multi-column queries                     │
└──────────────────┴──────────────────────────────────────────┘
```

```sql
-- Composite index: order matters! (leftmost prefix rule)
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
-- ✅ WHERE user_id = X                  (uses index)
-- ✅ WHERE user_id = X AND status = Y   (uses index)
-- ❌ WHERE status = Y                   (can't use — needs leftmost column)

-- Partial index: only index what you need
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'pending';
-- Smaller index, faster for: SELECT * FROM orders WHERE status = 'pending'

-- Covering index: avoid table lookup
CREATE INDEX idx_orders_cover ON orders(user_id) INCLUDE (total_amount, status);
-- Index-only scan for: SELECT total_amount, status FROM orders WHERE user_id = X

-- EXPLAIN ANALYZE — always verify index usage
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 'abc' AND status = 'pending';
```

### MongoDB Indexing

```typescript
// Compound index (follows ESR rule: Equality → Sort → Range)
db.orders.createIndex({ userId: 1, status: 1, createdAt: -1 });

// Text index for search
db.courses.createIndex({ title: "text", description: "text" });

// TTL index for auto-expiration
db.sessions.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 });

// Wildcard index for flexible schemas
db.events.createIndex({ "metadata.$**": 1 });
```

---

## 4. Replication

```
Primary-Replica (PostgreSQL / MongoDB):

  Write ──► ┌─────────┐ ───Replication──► ┌─────────┐
            │ Primary │                    │ Replica1│ ◄── Read
            └─────────┘ ───Replication──► ┌─────────┐
                                          │ Replica2│ ◄── Read
                                          └─────────┘

Types:
• Synchronous: Primary waits for replica ACK (strong consistency, higher latency)
• Asynchronous: Primary doesn't wait (eventual consistency, lower latency)
• Semi-synchronous: Wait for at least 1 replica (balance)

AWS RDS Multi-AZ:
• Synchronous standby in different AZ
• Automatic failover (60-120 seconds)
• Read replicas can be in different regions
```

```typescript
// TypeORM: Route reads to replica, writes to primary
const dataSource = new DataSource({
  type: 'postgres',
  replication: {
    master: {
      host: 'primary.rds.amazonaws.com',
      port: 5432,
      username: 'admin',
      password: process.env.DB_PASSWORD,
      database: 'app',
    },
    slaves: [
      { host: 'replica1.rds.amazonaws.com', port: 5432, username: 'reader', password: process.env.DB_READ_PASSWORD, database: 'app' },
      { host: 'replica2.rds.amazonaws.com', port: 5432, username: 'reader', password: process.env.DB_READ_PASSWORD, database: 'app' },
    ],
  },
});

// Writes automatically go to master
await userRepository.save(user);

// Reads automatically distributed to slaves
await userRepository.find({ where: { status: 'active' } });
```

---

## 5. Sharding & Partitioning

```
Partitioning (single database):
┌───────────────────────────────────────┐
│           orders table                │
├───────────────┬───────────────────────┤
│ 2024 data     │ Partition: orders_2024│
│ 2025 data     │ Partition: orders_2025│
│ 2026 data     │ Partition: orders_2026│
└───────────────┴───────────────────────┘

Sharding (multiple databases):
              ┌──── hash(user_id) % 3 ────┐
              │                            │
    ┌─────────▼──┐  ┌──────────┐  ┌───────▼────┐
    │  Shard 0   │  │ Shard 1  │  │  Shard 2   │
    │ users A-I  │  │users J-R │  │ users S-Z  │
    └────────────┘  └──────────┘  └────────────┘
```

### Sharding Strategies

```
┌──────────────────┬──────────────────────────────────────────┐
│ Strategy         │ Description                              │
├──────────────────┼──────────────────────────────────────────┤
│ Hash-based       │ hash(key) % N → shard                   │
│                  │ Even distribution, hard to range query   │
├──────────────────┼──────────────────────────────────────────┤
│ Range-based      │ A-M → shard1, N-Z → shard2              │
│                  │ Easy range queries, potential hot spots  │
├──────────────────┼──────────────────────────────────────────┤
│ Directory-based  │ Lookup table maps key → shard            │
│                  │ Flexible, adds lookup overhead           │
├──────────────────┼──────────────────────────────────────────┤
│ Geographic       │ US → shard-us, EU → shard-eu             │
│                  │ Data locality, GDPR compliance           │
└──────────────────┴──────────────────────────────────────────┘

Choosing a shard key:
• High cardinality (many unique values)
• Even distribution (no hot partition)
• Query isolation (most queries hit single shard)
• Example: user_id (good), country (bad — skewed), created_date (bad — hot recent shard)
```

### PostgreSQL Table Partitioning

```sql
-- Range partitioning by date
CREATE TABLE orders (
  id UUID NOT NULL,
  user_id UUID NOT NULL,
  total DECIMAL(10,2),
  created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2025 PARTITION OF orders
  FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
CREATE TABLE orders_2026 PARTITION OF orders
  FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');

-- Queries automatically pruned to relevant partitions
SELECT * FROM orders WHERE created_at >= '2026-01-01'; -- Only scans orders_2026
```

---

## 6. CQRS Pattern

```
Command Query Responsibility Segregation:
Separate read and write models for different optimization.

  Commands (Write)              Queries (Read)
  ┌─────────────┐              ┌──────────────┐
  │ POST /order │              │ GET /orders  │
  │ PUT /order  │              │ GET /reports │
  │ DELETE      │              │ GET /search  │
  └──────┬──────┘              └──────┬───────┘
         │                            │
  ┌──────▼──────┐              ┌──────▼───────┐
  │ Write Model │  ──Event──►  │ Read Model   │
  │ PostgreSQL  │   Stream     │ ElasticSearch │
  │ (normalized)│   (Kafka)    │ (denormalized)│
  └─────────────┘              └──────────────┘
```

```typescript
// CQRS in NestJS — separate handlers for commands and queries
// Command: Create order (writes to PostgreSQL)
@CommandHandler(CreateOrderCommand)
export class CreateOrderHandler implements ICommandHandler<CreateOrderCommand> {
  async execute(command: CreateOrderCommand) {
    const order = await this.orderRepo.save(command.data);
    // Emit event for read model sync
    this.eventBus.publish(new OrderCreatedEvent(order));
    return order;
  }
}

// Query: Get orders (reads from optimized read store)
@QueryHandler(GetOrdersQuery)
export class GetOrdersHandler implements IQueryHandler<GetOrdersQuery> {
  async execute(query: GetOrdersQuery) {
    // Read from denormalized view / cache / Elasticsearch
    return this.readStore.findOrders(query.filters);
  }
}
```

---

## 7. Database Per Service (Microservices)

```
            ┌────────────┐     ┌────────────┐     ┌────────────┐
            │ User Svc   │     │ Order Svc  │     │ Payment Svc│
            └─────┬──────┘     └─────┬──────┘     └─────┬──────┘
                  │                  │                   │
            ┌─────▼──────┐     ┌─────▼──────┐     ┌─────▼──────┐
            │ Users DB   │     │ Orders DB  │     │ Payments DB│
            │ PostgreSQL │     │ PostgreSQL │     │ PostgreSQL │
            └────────────┘     └────────────┘     └────────────┘

Rules:
• Each service owns its data — NO direct DB access from other services
• Cross-service data: API calls or events (not JOINs)
• Eventual consistency via events (Kafka / SQS)
• Saga pattern for distributed transactions
```

---

## 8. Interview Questions & Answers

### Q1: When would you choose PostgreSQL over MongoDB?

**Answer:**
- **PostgreSQL:** Complex relationships (JOINs), ACID transactions (payments, inventory), structured data that rarely changes schema, advanced queries (window functions, CTEs).
- **MongoDB:** Rapidly evolving schema, document-oriented data (CMS, catalogs), horizontal scaling needed early, geo-spatial queries, write-heavy workloads.
- Use PostgreSQL for user/subscription data (financial) and MongoDB for course content (flexible schema).

### Q2: How do you handle a slow database query in production?

**Answer:**
1. **Identify:** Check slow query logs (`pg_stat_statements` / MongoDB profiler).
2. **Analyze:** `EXPLAIN ANALYZE` to check query plan — sequential scan vs index scan.
3. **Fix:** Add missing index, rewrite query (avoid `SELECT *`, reduce JOINs), denormalize if needed.
4. **Cache:** Add Redis caching for frequently read, rarely changing data.
5. **Long-term:** Read replicas for read-heavy queries, partitioning for large tables.

### Q3: Explain database sharding challenges.

**Answer:**
- **Cross-shard queries:** JOINs across shards are expensive — design to avoid them.
- **Resharding:** Adding shards requires data migration — use consistent hashing.
- **Hotspots:** Poor shard key leads to uneven load.
- **Transactions:** No cross-shard ACID — use Saga pattern.
- **Operational complexity:** Backup, monitoring, schema changes on all shards.
- Recommendation: Delay sharding as long as possible — optimize, cache, read replicas first.

### Q4: What is connection pooling and why does it matter?

**Answer:**
- Creating a new DB connection takes 50-100ms (TCP handshake, auth, SSL).
- Connection pool maintains a set of pre-established connections.
- Node.js: Use `pg-pool` (PostgreSQL) or Mongoose connection pool (MongoDB).
- AWS Lambda: Use RDS Proxy to share connections across cold starts.
- Settings: `min: 2, max: 20, idleTimeoutMillis: 30000` — tune based on load.

---

**Previous:** [02 — Scalability & Load Balancing ←](02-scalability-load-balancing.md) | **Next:** [04 — Caching Strategies →](04-caching-strategies.md)
