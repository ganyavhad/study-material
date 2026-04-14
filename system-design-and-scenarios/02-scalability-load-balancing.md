# 02 — Scalability & Load Balancing

> Designing systems that handle growth from 100 users to 100 million.

---

## Table of Contents

1. [Scaling Strategies](#1-scaling-strategies)
2. [Load Balancing](#2-load-balancing)
3. [Auto Scaling](#3-auto-scaling)
4. [CDN & Edge Caching](#4-cdn--edge-caching)
5. [Rate Limiting & Throttling](#5-rate-limiting--throttling)
6. [Stateless vs Stateful Services](#6-stateless-vs-stateful-services)
7. [Node.js Cluster & PM2](#7-nodejs-cluster--pm2)
8. [Interview Questions & Answers](#8-interview-questions--answers)

---

## 1. Scaling Strategies

```
Vertical Scaling (Scale Up)          Horizontal Scaling (Scale Out)
─────────────────────────           ──────────────────────────────
┌─────────────┐                     ┌───────┐ ┌───────┐ ┌───────┐
│             │                     │ App 1 │ │ App 2 │ │ App 3 │
│  BIG SERVER │                     └───┬───┘ └───┬───┘ └───┬───┘
│  64 CPU     │                         └─────────┼─────────┘
│  256GB RAM  │                              ┌────▼────┐
│             │                              │   LB    │
└─────────────┘                              └─────────┘

Pros: Simple, no code changes      Pros: No machine limits, fault
Cons: Hardware limits, SPOF              tolerant, cost-effective
                                   Cons: Complexity, state mgmt
```

### Scaling Decision Matrix

```
┌──────────────────────┬─────────────────┬──────────────────┐
│ Component            │ Scale Strategy  │ Why              │
├──────────────────────┼─────────────────┼──────────────────┤
│ Node.js API servers  │ Horizontal      │ Stateless, cheap │
│ PostgreSQL (write)   │ Vertical first  │ ACID, complex    │
│ PostgreSQL (read)    │ Horizontal (RR) │ Read replicas    │
│ MongoDB              │ Horizontal      │ Built-in sharding│
│ Redis                │ Horizontal      │ Redis Cluster    │
│ Lambda functions     │ Auto-horizontal │ AWS manages      │
│ WebSocket servers    │ Horizontal+sticky│ Connection state│
└──────────────────────┴─────────────────┴──────────────────┘
```

---

## 2. Load Balancing

### Load Balancer Types

```
┌──────────────────┬────────────────────────────────────────────────┐
│ Type             │ Description                                    │
├──────────────────┼────────────────────────────────────────────────┤
│ L4 (Transport)   │ TCP/UDP level, fastest, no content inspection  │
│                  │ AWS: NLB (Network Load Balancer)               │
├──────────────────┼────────────────────────────────────────────────┤
│ L7 (Application) │ HTTP level, path/header routing, SSL termination│
│                  │ AWS: ALB (Application Load Balancer)           │
├──────────────────┼────────────────────────────────────────────────┤
│ DNS-based        │ Geographic routing, multi-region               │
│                  │ AWS: Route 53 with latency-based routing       │
└──────────────────┴────────────────────────────────────────────────┘
```

### Load Balancing Algorithms

```
┌─────────────────────────┬────────────────────────────────────────┐
│ Algorithm               │ How It Works                           │
├─────────────────────────┼────────────────────────────────────────┤
│ Round Robin             │ Rotate sequentially (default ALB)      │
│ Weighted Round Robin    │ More traffic to stronger servers       │
│ Least Connections       │ Route to server with fewest active     │
│ IP Hash                 │ Same client IP → same server           │
│ Random                  │ Random selection (simple, effective)   │
│ Least Response Time     │ Route to fastest responding server     │
└─────────────────────────┴────────────────────────────────────────┘
```

### AWS ALB Configuration Pattern

```
                         Route 53 (DNS)
                              │
                         ┌────▼────┐
                         │   ALB   │
                         └────┬────┘
                    ┌─────────┼─────────┐
                    │         │         │
              ┌─────▼──┐ ┌───▼───┐ ┌───▼────┐
              │ /api/* │ │ /ws/* │ │ /admin│
              │ Target │ │ Target│ │ Target │
              │ Group1 │ │ Group2│ │ Group3 │
              └────────┘ └───────┘ └────────┘

Path-based routing:
• /api/*    → API servers (NestJS)
• /ws/*     → WebSocket servers (Socket.IO)
• /admin/*  → Admin dashboard
• /*        → Frontend (S3 + CloudFront)
```

### Health Checks

```typescript
// NestJS health check endpoint for ALB
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
    private redis: RedisHealthIndicator,
  ) {}

  @Get()
  async check() {
    return this.health.check([
      () => this.db.pingCheck('database'),
      () => this.redis.pingCheck('redis'),
    ]);
  }
}

// ALB health check config:
// Path: /health
// Interval: 30 seconds
// Healthy threshold: 3
// Unhealthy threshold: 2
// Timeout: 5 seconds
```

---

## 3. Auto Scaling

```
Auto Scaling Policies:

┌───────────────────┬──────────────────────────────────────┐
│ Policy Type       │ When to Use                          │
├───────────────────┼──────────────────────────────────────┤
│ Target Tracking   │ Maintain CPU at 70%, request count   │
│ Step Scaling      │ Add 2 at 80% CPU, 4 at 90% CPU      │
│ Scheduled         │ Scale up before known peak hours     │
│ Predictive        │ ML-based, learns traffic patterns    │
└───────────────────┴──────────────────────────────────────┘

Scaling Metrics:
• CPU Utilization > 70%
• Memory Utilization > 80%
• Request Count per target > 1000/min
• Custom metric: Queue depth > 100
```

```
Auto Scaling Timeline:

Traffic ▲
       │        ┌──── Scale Out (add instances)
       │    ╱╲  │
       │   ╱  ╲ │  ╱╲
       │  ╱    ╲│ ╱  ╲
       │ ╱      ╲╱    ╲──── Scale In (remove instances)
       │╱                ╲
       └──────────────────────► Time
       
     Min: 2 instances │ Max: 20 instances │ Cooldown: 300s
```

### Lambda Auto-Scaling (Serverless)

```
Lambda scales automatically:
• Concurrent executions: 1000 default (can request increase)
• Burst: 500-3000 instant (region dependent)
• Provisioned concurrency: Pre-warm for predictable latency

             Request spike
                  │
    ┌─────────────▼─────────────┐
    │ API Gateway               │
    │ (throttle: 10,000 req/s)  │
    └─────────────┬─────────────┘
                  │
    ┌─────────────▼─────────────┐
    │ Lambda (auto-scales)      │
    │ 1 request = 1 invocation  │
    │ Concurrent limit: 1000    │
    └───────────────────────────┘
```

---

## 4. CDN & Edge Caching

```
Without CDN:                        With CDN (CloudFront):
User(India)──────────► US Server    User(India)──► Mumbai Edge──► US Origin
       150ms RTT                           10ms        (cached)
                                    
Content Delivery Network:
• Caches static assets at edge locations closest to users
• Reduces latency for global users
• Offloads origin server traffic
```

### CloudFront Configuration

```
┌───────────────────┬──────────────────────────────────────┐
│ Content Type      │ Cache Strategy                       │
├───────────────────┼──────────────────────────────────────┤
│ Static (JS, CSS)  │ TTL: 1 year, versioned filenames     │
│ Images            │ TTL: 30 days, S3 origin              │
│ API responses     │ TTL: 0 (pass through) or short TTL   │
│ HTML pages        │ TTL: 5 min, stale-while-revalidate   │
│ User-specific     │ No cache (Cookie/Auth header varies) │
└───────────────────┴──────────────────────────────────────┘

Cache Invalidation:
• Versioned URLs: /app.abc123.js (best — no invalidation needed)
• Cache-Control headers: max-age, s-maxage, stale-while-revalidate
• CloudFront Invalidation API: /* (emergency, takes 10-15 min)
```

---

## 5. Rate Limiting & Throttling

### Algorithms

```
┌──────────────────────┬──────────────────────────────────────────┐
│ Algorithm            │ How It Works                             │
├──────────────────────┼──────────────────────────────────────────┤
│ Fixed Window         │ Count in time window (e.g., 100/min)     │
│                      │ ⚠️ Burst at window boundary              │
├──────────────────────┼──────────────────────────────────────────┤
│ Sliding Window Log   │ Track each request timestamp             │
│                      │ Accurate but memory heavy                │
├──────────────────────┼──────────────────────────────────────────┤
│ Sliding Window       │ Weighted count across windows            │
│ Counter              │ Good balance of accuracy + memory        │
├──────────────────────┼──────────────────────────────────────────┤
│ Token Bucket         │ Tokens added at fixed rate               │
│                      │ Allows bursts, smooth rate limiting      │
├──────────────────────┼──────────────────────────────────────────┤
│ Leaky Bucket         │ Requests processed at fixed rate         │
│                      │ Queues excess, smooth output             │
└──────────────────────┴──────────────────────────────────────────┘
```

### Token Bucket Implementation (Redis)

```typescript
// Distributed rate limiter using Redis — Token Bucket
import Redis from 'ioredis';

class TokenBucketRateLimiter {
  constructor(
    private redis: Redis,
    private maxTokens: number,     // Bucket capacity
    private refillRate: number,    // Tokens per second
  ) {}

  async isAllowed(clientId: string): Promise<boolean> {
    const key = `rate_limit:${clientId}`;
    const now = Date.now();

    const result = await this.redis.eval(`
      local key = KEYS[1]
      local maxTokens = tonumber(ARGV[1])
      local refillRate = tonumber(ARGV[2])
      local now = tonumber(ARGV[3])

      local data = redis.call('HMGET', key, 'tokens', 'lastRefill')
      local tokens = tonumber(data[1]) or maxTokens
      local lastRefill = tonumber(data[2]) or now

      -- Refill tokens
      local elapsed = (now - lastRefill) / 1000
      tokens = math.min(maxTokens, tokens + elapsed * refillRate)

      if tokens >= 1 then
        tokens = tokens - 1
        redis.call('HMSET', key, 'tokens', tokens, 'lastRefill', now)
        redis.call('EXPIRE', key, 60)
        return 1
      else
        redis.call('HMSET', key, 'tokens', tokens, 'lastRefill', now)
        redis.call('EXPIRE', key, 60)
        return 0
      end
    `, 1, key, this.maxTokens, this.refillRate, now) as number;

    return result === 1;
  }
}

// Usage in NestJS Guard
@Injectable()
export class RateLimitGuard implements CanActivate {
  constructor(private limiter: TokenBucketRateLimiter) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const clientId = request.ip;

    if (!(await this.limiter.isAllowed(clientId))) {
      throw new HttpException('Too Many Requests', 429);
    }
    return true;
  }
}
```

---

## 6. Stateless vs Stateful Services

```
Stateless (preferred for scaling):
┌───────┐    ┌───────┐    ┌───────┐
│ App 1 │    │ App 2 │    │ App 3 │   ← Any instance handles any request
└───┬───┘    └───┬───┘    └───┬───┘
    └────────────┼────────────┘
            ┌────▼────┐
            │  Redis  │  ← Session/state stored externally
            └─────────┘

Stateful (harder to scale):
┌───────┐    ┌───────┐    ┌───────┐
│ App 1 │    │ App 2 │    │ App 3 │   ← Session stuck to specific instance
│ sess1 │    │ sess2 │    │ sess3 │
└───────┘    └───────┘    └───────┘
  ↑ Sticky sessions required (limits flexibility)
```

```typescript
// ❌ Stateful — session stored in-process
const sessions: Record<string, any> = {}; // Lost on restart or scale

// ✅ Stateless — session in Redis
import session from 'express-session';
import RedisStore from 'connect-redis';

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
}));
// Any server instance can serve any user
```

---

## 7. Node.js Cluster & PM2

```typescript
// Node.js Cluster Mode — utilize all CPU cores
import cluster from 'cluster';
import os from 'os';

if (cluster.isPrimary) {
  const numCPUs = os.cpus().length;
  console.log(`Primary ${process.pid} starting ${numCPUs} workers`);

  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker) => {
    console.log(`Worker ${worker.process.pid} died, restarting...`);
    cluster.fork(); // Auto-restart
  });
} else {
  // Worker process — runs the app
  app.listen(3000);
}

// PM2 ecosystem.config.js — production process manager
module.exports = {
  apps: [{
    name: 'api',
    script: 'dist/main.js',
    instances: 'max',        // Use all CPUs
    exec_mode: 'cluster',    // Cluster mode
    max_memory_restart: '500M',
    env_production: {
      NODE_ENV: 'production',
      PORT: 3000,
    },
  }],
};
```

---

## 8. Interview Questions & Answers

### Q1: How would you scale a Node.js application from 1K to 1M users?

**Answer:**
```
Stage 1 (1K users): Single server, PM2 cluster mode, PostgreSQL
Stage 2 (10K):      Add Redis cache, CDN for static assets
Stage 3 (100K):     ALB + multiple instances, read replicas, SQS for async
Stage 4 (1M):       Multi-AZ, auto-scaling, DB sharding, Kafka for events
                    Microservices split, ElastiCache cluster
```

### Q2: What's the difference between ALB and NLB?

**Answer:**
- **ALB (L7):** HTTP/HTTPS, path-based routing, WebSocket support, SSL termination. Use for web APIs.
- **NLB (L4):** TCP/UDP, ultra-low latency, static IP, millions of QPS. Use for high-throughput, non-HTTP protocols.
- We use ALB for our NestJS APIs (path routing to microservices) and NLB for Kafka consumers.

### Q3: How do you handle sticky sessions with load balancing?

**Answer:**
- **Avoid if possible** — use external session store (Redis).
- If needed: ALB supports cookie-based sticky sessions.
- WebSocket connections inherently need stickiness — use Redis adapter for Socket.IO to share state across instances.
- Use Redis-backed sessions so any instance can handle any request.

### Q4: Explain rate limiting at different layers.

**Answer:**
- **API Gateway (L7):** AWS API Gateway throttling (10K req/s default).
- **Application (L7):** Token bucket with Redis (per-user, per-API key).
- **Network (L4):** AWS WAF, Shield for DDoS protection.
- **Database:** Connection pooling limits concurrent queries.
- Defense in depth — rate limit at every layer.

---

**Previous:** [01 — System Design Fundamentals ←](01-system-design-fundamentals.md) | **Next:** [03 — Database Design & Scaling →](03-database-design-scaling.md)
