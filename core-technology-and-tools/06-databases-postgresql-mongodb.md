# Databases: PostgreSQL & MongoDB - Study Material

> Based on skills from [ganeshavhad.com](https://ganeshavhad.com) — Software Developer Specialist

---

## Part A: PostgreSQL (Relational)

### 1. Core Concepts

PostgreSQL is an advanced **open-source relational database** with ACID compliance, JSON support, and extensibility.

| Feature | Description |
|---------|-------------|
| ACID | Atomicity, Consistency, Isolation, Durability |
| MVCC | Multi-Version Concurrency Control (no read locks) |
| JSON/JSONB | Store and query semi-structured data |
| Full-Text Search | Built-in text search with ranking |
| Extensions | PostGIS, pg_trgm, uuid-ossp, etc. |
| Partitioning | Range, List, Hash partitioning |

### 2. Table Design & Relationships

```sql
-- Users table
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  role VARCHAR(20) DEFAULT 'user' CHECK (role IN ('admin', 'user', 'moderator')),
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Posts table (One-to-Many with Users)
CREATE TABLE posts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title VARCHAR(255) NOT NULL,
  content TEXT NOT NULL,
  published BOOLEAN DEFAULT false,
  author_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Tags (Many-to-Many)
CREATE TABLE tags (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50) UNIQUE NOT NULL
);

CREATE TABLE post_tags (
  post_id UUID REFERENCES posts(id) ON DELETE CASCADE,
  tag_id INT REFERENCES tags(id) ON DELETE CASCADE,
  PRIMARY KEY (post_id, tag_id)
);

-- Auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

### 3. Indexing Strategies

```sql
-- B-tree (default, general purpose)
CREATE INDEX idx_users_email ON users(email);

-- Partial index (only active users)
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;

-- Composite index
CREATE INDEX idx_posts_author_date ON posts(author_id, created_at DESC);

-- GIN index for JSONB
CREATE INDEX idx_user_metadata ON users USING GIN (metadata);

-- GiST for full-text search
CREATE INDEX idx_posts_search ON posts USING GIN (to_tsvector('english', title || ' ' || content));

-- Expression index
CREATE INDEX idx_users_lower_email ON users (LOWER(email));
```

### 4. Advanced Queries

```sql
-- Window functions
SELECT
  name,
  department,
  salary,
  RANK() OVER (PARTITION BY department ORDER BY salary DESC) as dept_rank,
  salary - LAG(salary) OVER (ORDER BY salary) as diff_from_prev
FROM employees;

-- CTE (Common Table Expressions)
WITH monthly_stats AS (
  SELECT
    DATE_TRUNC('month', created_at) as month,
    COUNT(*) as total_orders,
    SUM(amount) as revenue
  FROM orders
  GROUP BY DATE_TRUNC('month', created_at)
)
SELECT
  month,
  total_orders,
  revenue,
  revenue - LAG(revenue) OVER (ORDER BY month) as revenue_growth
FROM monthly_stats;

-- JSONB queries
SELECT * FROM users
WHERE metadata @> '{"preferences": {"theme": "dark"}}';

SELECT metadata->>'city' as city, COUNT(*)
FROM users
WHERE metadata ? 'city'
GROUP BY metadata->>'city';

-- Full-text search
SELECT title, ts_rank(search_vector, query) as rank
FROM posts, plainto_tsquery('english', 'graphql api') as query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

### 5. Migrations (with TypeORM)

```typescript
import { MigrationInterface, QueryRunner, Table } from 'typeorm';

export class CreateUsersTable1700000000000 implements MigrationInterface {
  async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.createTable(new Table({
      name: 'users',
      columns: [
        { name: 'id', type: 'uuid', isPrimary: true, default: 'gen_random_uuid()' },
        { name: 'name', type: 'varchar', length: '100' },
        { name: 'email', type: 'varchar', isUnique: true },
        { name: 'created_at', type: 'timestamptz', default: 'NOW()' }
      ]
    }));
  }

  async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropTable('users');
  }
}
```

### 6. Performance Tuning

```sql
-- EXPLAIN ANALYZE to understand query plans
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';

-- Connection pooling config (pgBouncer)
-- pool_mode = transaction
-- max_client_conn = 1000
-- default_pool_size = 20

-- Vacuum and analyze
VACUUM ANALYZE users;

-- Monitor slow queries
ALTER SYSTEM SET log_min_duration_statement = 200; -- Log queries > 200ms
SELECT pg_reload_conf();
```

---

## Part B: MongoDB (Document Database)

### 1. Core Concepts

MongoDB is a **NoSQL document database** storing data as flexible JSON-like documents (BSON).

| Feature | Description |
|---------|-------------|
| Documents | JSON-like (BSON) flexible schema |
| Collections | Groups of documents (like tables) |
| Replica Sets | High availability with automatic failover |
| Sharding | Horizontal scaling across servers |
| Aggregation | Powerful data processing pipeline |
| Change Streams | Real-time data change notifications |

### 2. Schema Design Patterns

#### Embedding (1:Few)
```javascript
// User with embedded addresses (good for 1:few)
{
  _id: ObjectId("..."),
  name: "Ganesh Avhad",
  email: "miganeshavhad@gmail.com",
  addresses: [
    { type: "home", city: "Thane", state: "Maharashtra" },
    { type: "work", city: "Mumbai", state: "Maharashtra" }
  ]
}
```

#### Referencing (1:Many / Many:Many)
```javascript
// Post referencing author (good for 1:many)
{
  _id: ObjectId("..."),
  title: "GraphQL Best Practices",
  content: "...",
  author_id: ObjectId("user_id_here"),  // Reference
  tags: ["graphql", "api", "nodejs"]
}
```

#### Bucket Pattern (Time-Series)
```javascript
// Group measurements into buckets
{
  sensor_id: "sensor_001",
  start_date: ISODate("2024-01-01"),
  end_date: ISODate("2024-01-01T23:59:59"),
  measurements: [
    { timestamp: ISODate("2024-01-01T00:00:00"), temp: 22.5, humidity: 45 },
    { timestamp: ISODate("2024-01-01T00:05:00"), temp: 22.7, humidity: 44 }
    // ... up to 200 per bucket
  ],
  count: 288,
  sum_temp: 6480
}
```

### 3. CRUD Operations

```javascript
const { MongoClient } = require('mongodb');

const client = new MongoClient(process.env.MONGODB_URI);
const db = client.db('myapp');
const users = db.collection('users');

// CREATE
const result = await users.insertOne({
  name: 'Ganesh',
  email: 'miganeshavhad@gmail.com',
  skills: ['nodejs', 'angular', 'graphql'],
  createdAt: new Date()
});

// READ
const user = await users.findOne({ email: 'miganeshavhad@gmail.com' });
const activeUsers = await users.find({ isActive: true })
  .sort({ createdAt: -1 })
  .limit(10)
  .toArray();

// UPDATE
await users.updateOne(
  { _id: user._id },
  {
    $set: { name: 'Ganesh Avhad' },
    $push: { skills: 'terraform' },
    $inc: { loginCount: 1 }
  }
);

// DELETE
await users.deleteOne({ _id: user._id });
```

### 4. Aggregation Pipeline

```javascript
// RFM (Recency, Frequency, Monetary) Analysis
// "Developed and Integrated an RFM Scorecard for Product Administrators"
const rfmAnalysis = await orders.aggregate([
  {
    $group: {
      _id: '$customer_id',
      lastPurchase: { $max: '$createdAt' },        // Recency
      totalOrders: { $sum: 1 },                     // Frequency
      totalSpent: { $sum: '$amount' }               // Monetary
    }
  },
  {
    $addFields: {
      recencyDays: {
        $dateDiff: {
          startDate: '$lastPurchase',
          endDate: new Date(),
          unit: 'day'
        }
      }
    }
  },
  {
    $addFields: {
      recencyScore: {
        $switch: {
          branches: [
            { case: { $lte: ['$recencyDays', 30] }, then: 5 },
            { case: { $lte: ['$recencyDays', 90] }, then: 4 },
            { case: { $lte: ['$recencyDays', 180] }, then: 3 },
            { case: { $lte: ['$recencyDays', 365] }, then: 2 }
          ],
          default: 1
        }
      },
      frequencyScore: {
        $switch: {
          branches: [
            { case: { $gte: ['$totalOrders', 20] }, then: 5 },
            { case: { $gte: ['$totalOrders', 10] }, then: 4 },
            { case: { $gte: ['$totalOrders', 5] }, then: 3 },
            { case: { $gte: ['$totalOrders', 2] }, then: 2 }
          ],
          default: 1
        }
      }
    }
  },
  { $sort: { totalSpent: -1 } }
]).toArray();

// Report generation (CSV/Excel export)
// "Automated Report Generation Processes"
const monthlyReport = await orders.aggregate([
  {
    $match: {
      createdAt: {
        $gte: new Date('2024-01-01'),
        $lt: new Date('2024-02-01')
      }
    }
  },
  {
    $group: {
      _id: { $dateToString: { format: '%Y-%m-%d', date: '$createdAt' } },
      orderCount: { $sum: 1 },
      revenue: { $sum: '$amount' },
      avgOrderValue: { $avg: '$amount' }
    }
  },
  { $sort: { _id: 1 } }
]).toArray();
```

### 5. Indexing

```javascript
// Single field index
await users.createIndex({ email: 1 }, { unique: true });

// Compound index
await posts.createIndex({ author_id: 1, createdAt: -1 });

// Text index for search
await posts.createIndex({ title: 'text', content: 'text' });

// TTL index (auto-delete after 30 days)
await sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 2592000 });

// Partial index
await orders.createIndex(
  { status: 1 },
  { partialFilterExpression: { status: 'pending' } }
);
```

### 6. Mongoose ODM

```typescript
import mongoose, { Schema, Document } from 'mongoose';

interface IUser extends Document {
  name: string;
  email: string;
  password: string;
  role: 'admin' | 'user';
  posts: mongoose.Types.ObjectId[];
}

const userSchema = new Schema<IUser>({
  name: { type: String, required: true, trim: true },
  email: { type: String, required: true, unique: true, lowercase: true },
  password: { type: String, required: true, select: false },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
  posts: [{ type: Schema.Types.ObjectId, ref: 'Post' }]
}, {
  timestamps: true,
  toJSON: { virtuals: true }
});

// Virtual
userSchema.virtual('postCount').get(function() {
  return this.posts?.length || 0;
});

// Pre-save middleware
userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 12);
  }
  next();
});

// Static method
userSchema.statics.findByEmail = function(email: string) {
  return this.findOne({ email: email.toLowerCase() });
};

export const User = mongoose.model<IUser>('User', userSchema);
```

---

## Part C: PostgreSQL vs MongoDB Decision Guide

| Criteria | PostgreSQL | MongoDB |
|----------|------------|---------|
| Data Structure | Fixed schema, relational | Flexible documents |
| Relationships | JOINs with foreign keys | Embedding or referencing |
| Transactions | Full ACID | Multi-document ACID (4.0+) |
| Scaling | Vertical (read replicas) | Horizontal (sharding) |
| Best For | Complex queries, reporting | Rapid iteration, varied data |
| Schema Changes | ALTER TABLE (migration needed) | Schema-less (flexible) |
| Use When | Financial data, strict consistency | Content management, IoT, catalogs |

---

## Interview Questions

1. **What is MVCC in PostgreSQL?** — Each transaction sees a snapshot. Writers don't block readers. Old versions are cleaned by VACUUM.
2. **Explain MongoDB sharding** — Distributes data across shards using a shard key. Config servers store metadata. Mongos routes queries.
3. **When to embed vs reference in MongoDB?** — Embed: data accessed together, 1:few. Reference: 1:many, many:many, data changes independently.
4. **What are PostgreSQL window functions?** — Perform calculations across rows related to current row without grouping (RANK, ROW_NUMBER, LAG, LEAD).
5. **MongoDB aggregation pipeline vs SQL?** — Pipeline: sequential stages ($match → $group → $sort). Each stage transforms the data stream.
6. **What is connection pooling?** — Reuse DB connections instead of creating new ones. PgBouncer for PostgreSQL, built-in for MongoDB driver.
7. **How to handle schema migrations?** — PostgreSQL: migration files (TypeORM, Flyway). MongoDB: gradual migration with code handling old/new schemas.
8. **What is a materialized view?** — PostgreSQL: cached query result. Refreshed on demand. Great for complex reporting queries.
