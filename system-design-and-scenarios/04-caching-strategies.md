# 04 — Caching Strategies

> Reduce latency from 100ms to 1ms — the single biggest performance lever in system design.

---

## Table of Contents

1. [Caching Layers](#1-caching-layers)
2. [Cache Strategies (Read)](#2-cache-strategies-read)
3. [Cache Strategies (Write)](#3-cache-strategies-write)
4. [Redis Patterns](#4-redis-patterns)
5. [Cache Invalidation](#5-cache-invalidation)
6. [Distributed Caching](#6-distributed-caching)
7. [CDN Caching](#7-cdn-caching)
8. [Interview Questions & Answers](#8-interview-questions--answers)

---

## 1. Caching Layers

```
Client ──► CDN ──► API Gateway ──► Application ──► Database
  │         │          │               │               │
  │ Browser │ Edge     │ Response      │ In-Memory     │ Query
  │ Cache   │ Cache    │ Cache         │ Cache (Redis) │ Cache
  │ (local) │ (CF)     │ (API GW)      │               │ (pg)

Layer:      L1         L2         L3            L4           L5
Latency:   <1ms       5ms       10ms          1-5ms       50-100ms

Cache everything you can, as close to the user as possible.
```

```
┌──────────────────────┬────────────────────────────────────────┐
│ Cache Layer          │ Technology                             │
├──────────────────────┼────────────────────────────────────────┤
│ Browser/Client       │ Cache-Control headers, Service Worker  │
│ CDN Edge             │ CloudFront, Cloudflare                 │
│ API Gateway          │ AWS API Gateway cache                  │
│ Application (local)  │ node-cache, lru-cache, Map            │
│ Distributed          │ Redis, Memcached, ElastiCache          │
│ Database             │ PostgreSQL buffer cache, query cache   │
└──────────────────────┴────────────────────────────────────────┘
```

---

## 2. Cache Strategies (Read)

### Cache-Aside (Lazy Loading) — Most Common

```
App checks cache → miss → reads DB → writes to cache → returns

  ┌─────┐  1. GET   ┌─────┐
  │ App │──────────►│Cache│──► HIT → return cached
  │     │           └──┬──┘
  │     │  2. MISS    │
  │     │◄────────────┘
  │     │  3. Query  ┌──┐
  │     │──────────►│DB│
  │     │◄──────────└──┘
  │     │  4. SET   ┌─────┐
  │     │──────────►│Cache│
  └─────┘           └─────┘
```

```typescript
// Cache-Aside with Redis in NestJS
@Injectable()
export class UserService {
  constructor(
    private redis: Redis,
    private userRepo: Repository<User>,
  ) {}

  async getUser(id: string): Promise<User> {
    const cacheKey = `user:${id}`;

    // 1. Check cache
    const cached = await this.redis.get(cacheKey);
    if (cached) return JSON.parse(cached);

    // 2. Cache miss — query DB
    const user = await this.userRepo.findOneBy({ id });
    if (!user) throw new NotFoundException();

    // 3. Populate cache with TTL
    await this.redis.set(cacheKey, JSON.stringify(user), 'EX', 3600);

    return user;
  }
}
```

### Read-Through

```
App reads from cache only → cache auto-fetches from DB on miss

  ┌─────┐  GET    ┌─────────┐  MISS  ┌──┐
  │ App │────────►│  Cache   │───────►│DB│
  │     │◄────────│(manages  │◄───────└──┘
  └─────┘         │ loading) │
                  └──────────┘

Cache handles DB lookup transparently.
Used with frameworks that support cache loaders (e.g., Caffeine, Guava).
```

---

## 3. Cache Strategies (Write)

### Write-Through

```
Write to cache AND database simultaneously.

  ┌─────┐  WRITE  ┌─────┐  WRITE  ┌──┐
  │ App │────────►│Cache│────────►│DB│
  └─────┘         └─────┘         └──┘

✅ Cache always consistent with DB
❌ Higher write latency (two writes)
❌ Cache filled with data that may never be read
```

### Write-Behind (Write-Back)

```
Write to cache immediately → async batch write to DB later.

  ┌─────┐  WRITE  ┌─────┐  async batch  ┌──┐
  │ App │────────►│Cache│──────────────►│DB│
  └─────┘         └─────┘    (later)     └──┘

✅ Very fast writes
❌ Risk of data loss if cache crashes before DB write
Use for: analytics, logging, non-critical counters
```

### Write-Around

```
Write directly to DB → cache only populated on read.

  ┌─────┐  WRITE  ┌──┐
  │ App │────────►│DB│
  └─────┘         └──┘
       │  READ (miss)
       └────────► Cache loads from DB

✅ No cache pollution with writes
❌ First read always a miss
Use for: infrequently read data
```

---

## 4. Redis Patterns

```typescript
// String — simple key-value with TTL
await redis.set('session:abc123', JSON.stringify(sessionData), 'EX', 1800);
const session = JSON.parse(await redis.get('session:abc123') || 'null');

// Hash — structured object fields
await redis.hset('user:123', { name: 'Ganesh', role: 'admin', loginCount: '5' });
await redis.hincrby('user:123', 'loginCount', 1);
const user = await redis.hgetall('user:123');

// Sorted Set — leaderboard / ranking
await redis.zadd('leaderboard', 1500, 'player1', 2000, 'player2', 1800, 'player3');
const topPlayers = await redis.zrevrange('leaderboard', 0, 9, 'WITHSCORES');

// List — queue / recent items
await redis.lpush('notifications:user1', JSON.stringify(notification));
const recent = await redis.lrange('notifications:user1', 0, 9);

// Set — unique values (tags, online users)
await redis.sadd('online_users', 'user1', 'user2', 'user3');
const isOnline = await redis.sismember('online_users', 'user1');
const count = await redis.scard('online_users');

// Pub/Sub — real-time notifications across instances
const subscriber = redis.duplicate();
subscriber.subscribe('order_updates');
subscriber.on('message', (channel, message) => {
  io.to(message.userId).emit('orderUpdate', JSON.parse(message));
});
await redis.publish('order_updates', JSON.stringify({ userId: '123', status: 'shipped' }));
```

### Redis Pipeline & Transactions

```typescript
// Pipeline — batch commands (reduces round trips)
const pipeline = redis.pipeline();
pipeline.get('user:1');
pipeline.get('user:2');
pipeline.get('user:3');
const results = await pipeline.exec();

// Transaction — atomic multi-command
const multi = redis.multi();
multi.decrby('inventory:product1', 1);
multi.incrby('sales:product1', 1);
await multi.exec(); // All or nothing
```

---

## 5. Cache Invalidation

> "There are only two hard things in Computer Science: cache invalidation and naming things."

```
┌──────────────────────┬──────────────────────────────────────────┐
│ Strategy             │ When to Use                              │
├──────────────────────┼──────────────────────────────────────────┤
│ TTL (Time-to-Live)   │ Default. Set expiration on every key.    │
│                      │ Simple, handles most cases.              │
├──────────────────────┼──────────────────────────────────────────┤
│ Event-driven         │ Invalidate on write events (pub/sub).    │
│                      │ More accurate, more complex.             │
├──────────────────────┼──────────────────────────────────────────┤
│ Version-based        │ Include version in cache key.            │
│                      │ user:123:v5 → new version = new key.     │
├──────────────────────┼──────────────────────────────────────────┤
│ Cache tag / group    │ Invalidate all keys with a tag.          │
│                      │ E.g., invalidate all "product" caches.   │
└──────────────────────┴──────────────────────────────────────────┘
```

```typescript
// Event-driven invalidation pattern
@Injectable()
export class UserService {
  async updateUser(id: string, data: UpdateUserDto): Promise<User> {
    const user = await this.userRepo.save({ id, ...data });

    // Invalidate related caches
    await this.redis.del(`user:${id}`);
    await this.redis.del(`user:email:${user.email}`);
    // Publish event for other services
    await this.redis.publish('cache:invalidate', JSON.stringify({
      entity: 'user',
      id,
    }));

    return user;
  }
}

// Stale-While-Revalidate pattern
async function getWithSWR(key: string, fetchFn: () => Promise<any>, ttl: number) {
  const cached = await redis.get(key);
  const meta = await redis.get(`${key}:meta`);

  if (cached) {
    // Check if stale (past soft TTL but not hard TTL)
    if (meta && JSON.parse(meta).softExpiry < Date.now()) {
      // Return stale data but refresh in background
      fetchFn().then(async (fresh) => {
        await redis.set(key, JSON.stringify(fresh), 'EX', ttl * 2);
        await redis.set(`${key}:meta`, JSON.stringify({ softExpiry: Date.now() + ttl * 1000 }), 'EX', ttl * 2);
      });
    }
    return JSON.parse(cached);
  }

  // Cache miss
  const data = await fetchFn();
  await redis.set(key, JSON.stringify(data), 'EX', ttl * 2);
  await redis.set(`${key}:meta`, JSON.stringify({ softExpiry: Date.now() + ttl * 1000 }), 'EX', ttl * 2);
  return data;
}
```

### Cache Stampede Prevention

```typescript
// Problem: Many requests hit expired cache simultaneously → all query DB
// Solution: Distributed lock

async function getWithLock(key: string, fetchFn: () => Promise<any>, ttlSec: number) {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const lockKey = `lock:${key}`;
  const lockAcquired = await redis.set(lockKey, '1', 'EX', 10, 'NX');

  if (lockAcquired) {
    try {
      const data = await fetchFn();
      await redis.set(key, JSON.stringify(data), 'EX', ttlSec);
      return data;
    } finally {
      await redis.del(lockKey);
    }
  } else {
    // Another process is fetching — wait briefly and retry
    await new Promise(r => setTimeout(r, 100));
    return getWithLock(key, fetchFn, ttlSec);
  }
}
```

---

## 6. Distributed Caching

```
Redis Cluster:
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Shard 0  │  │ Shard 1  │  │ Shard 2  │
  │ Slots    │  │ Slots    │  │ Slots    │
  │ 0-5460   │  │5461-10922│  │10923-16383│
  │ Primary  │  │ Primary  │  │ Primary  │
  │ + Replica│  │ + Replica│  │ + Replica│
  └──────────┘  └──────────┘  └──────────┘

Key → CRC16(key) % 16384 → slot → shard

AWS ElastiCache:
• Managed Redis with automatic failover
• Cluster mode for horizontal scaling
• Multi-AZ for high availability
• Encryption at rest and in-transit
```

---

## 7. CDN Caching

```typescript
// Setting proper cache headers in NestJS
@Controller('api')
export class ApiController {
  // Static data — cache aggressively
  @Get('config')
  @Header('Cache-Control', 'public, max-age=86400, s-maxage=604800')
  getConfig() {
    return this.configService.getPublicConfig();
  }

  // Dynamic but cacheable — short TTL
  @Get('products')
  @Header('Cache-Control', 'public, max-age=60, stale-while-revalidate=300')
  getProducts() {
    return this.productService.findAll();
  }

  // User-specific — never cache in shared caches
  @Get('profile')
  @Header('Cache-Control', 'private, no-cache')
  @UseGuards(AuthGuard)
  getProfile(@CurrentUser() user: User) {
    return user;
  }
}
```

---

## 8. Interview Questions & Answers

### Q1: How do you decide what to cache?

**Answer:**
Cache data that is: **read frequently, changes infrequently, expensive to compute**.
- User profiles: read 100x more than written → cache with TTL + event invalidation.
- Product catalog: changes daily → cache with 5min TTL.
- Session data: read on every request → Redis with 30min TTL.
- Real-time inventory: changes constantly → don't cache (or very short TTL).

### Q2: Cache-Aside vs Read-Through — which do you prefer?

**Answer:**
- **Cache-Aside:** Application manages cache explicitly. More control, works with any cache. This is what we use with Redis + NestJS.
- **Read-Through:** Cache layer handles DB reads transparently. Simpler app code but tightly coupled to cache framework.
- In practice, Cache-Aside is most common in Node.js because it's explicit and easy to debug.

### Q3: How do you handle cache invalidation in a microservices architecture?

**Answer:**
- **Event-driven:** Service publishes event on data change (via Kafka/SQS) → consumers invalidate their caches.
- **TTL-based:** Set reasonable TTLs (5-60 min) as a safety net.
- **Pub/Sub:** Redis pub/sub for immediate cross-instance invalidation.
- We combine all three: short TTL + event-driven invalidation + pub/sub for real-time.

### Q4: What is a cache stampede and how do you prevent it?

**Answer:**
- **Problem:** Popular cache key expires → hundreds of concurrent requests all miss → all hit DB simultaneously → DB overwhelmed.
- **Solutions:**
  1. **Distributed lock:** Only one process refreshes, others wait.
  2. **Stale-while-revalidate:** Serve stale data while refreshing in background.
  3. **TTL jitter:** Add random variance to TTL to avoid simultaneous expiration.
  4. **Pre-warming:** Refresh cache before expiration using background job.

---

**Previous:** [03 — Database Design & Scaling ←](03-database-design-scaling.md) | **Next:** [05 — Message Queues & Async Processing →](05-message-queues-async.md)
