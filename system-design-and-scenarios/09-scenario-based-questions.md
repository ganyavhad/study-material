# 09 — Scenario-Based Interview Questions

> Real production scenarios drawn from experience. Use STAR format (Situation → Task → Action → Result).

---

## Table of Contents

1. [Production Debugging](#1-production-debugging)
2. [Performance Optimization](#2-performance-optimization)
3. [Database Migration & Schema Changes](#3-database-migration--schema-changes)
4. [Outage Handling & Incident Response](#4-outage-handling--incident-response)
5. [Architecture Trade-off Decisions](#5-architecture-trade-off-decisions)
6. [Scaling Challenges](#6-scaling-challenges)
7. [Security Incidents](#7-security-incidents)
8. [Team & Process Scenarios](#8-team--process-scenarios)

---

## 1. Production Debugging

### Q: "Your API response times jump from 200ms to 5 seconds. How do you investigate?"

**Answer (STAR):**

**Situation:** Production latency spike detected via CloudWatch alarms.

**Task:** Identify root cause and restore normal performance.

**Action:**
```
Step 1: Check metrics dashboard
  • Is it all endpoints or specific ones?
  • CPU / Memory / Disk I/O on EC2/ECS instances?
  • Database connection pool utilization?

Step 2: Check external dependencies
  • Redis latency? → redis-cli --latency
  • Database slow query log → pg_stat_activity for long-running queries
  • Third-party API (Stripe, SES) latency spikes?

Step 3: Check application logs
  • Filter by correlation ID for slow requests
  • Look for error patterns, retry storms, connection timeouts

Step 4: Check infrastructure
  • Did an auto-scaling event fail?
  • DNS resolution issues?
  • Recent deployment? → rollback if correlated

Common causes in Node.js:
  • Event loop blocked (CPU-intensive sync operation)
  • Memory leak → GC pauses → use --max-old-space-size, check heap
  • Connection pool exhaustion (PostgreSQL max connections)
  • N+1 query (missing DataLoader / eager loading)
  • Missing database index on a query that grew with data volume
```

**Result:** "At Accenture, we had a latency spike caused by a missing index on a query that was fine with 10K rows but degraded at 500K. Added a composite index, latency dropped from 4s to 50ms. Added slow query monitoring to prevent recurrence."

---

### Q: "Your Node.js service is consuming 2GB memory and growing. How do you debug?"

**Answer:**
```
Step 1: Confirm the leak
  • Monitor over time — is it steadily growing or plateauing?
  • Check with process.memoryUsage() (heapUsed, rss, external)

Step 2: Generate heap snapshot
  • In production: --inspect flag + Chrome DevTools (careful — pauses process)
  • Or use clinic.js: npx clinic heap -- node server.js
  • Compare two snapshots taken minutes apart

Step 3: Common Node.js memory leaks
  • Unbounded arrays/maps used as cache (no eviction)
  • Event listeners not removed (emitter.removeListener)
  • Closures holding references to large objects
  • Streams not properly consumed or destroyed
  • Global variables accumulating data

Step 4: Fix example
```

```typescript
// LEAK: Unbounded in-memory cache
const cache = new Map(); // Grows forever!
function getData(key: string) {
  if (cache.has(key)) return cache.get(key);
  const data = fetchFromDB(key);
  cache.set(key, data); // Never evicted
  return data;
}

// FIX: Use LRU cache with max size
import { LRUCache } from 'lru-cache';
const cache = new LRUCache({ max: 1000, ttl: 60000 }); // Max 1000 entries, 60s TTL

// FIX: Use Redis instead of in-memory cache (recommended for production)
async function getData(key: string) {
  const cached = await redis.get(`cache:${key}`);
  if (cached) return JSON.parse(cached);
  const data = await fetchFromDB(key);
  await redis.setex(`cache:${key}`, 60, JSON.stringify(data));
  return data;
}
```

---

## 2. Performance Optimization

### Q: "A report endpoint takes 30 seconds to respond. How do you optimize it?"

**Answer:**
```
Analyze first:
  • EXPLAIN ANALYZE the query → is it doing sequential scan or index scan?
  • How much data? Is it aggregating millions of rows on every request?

Optimization strategies (in order of impact):
  1. Add appropriate indexes (composite, partial, covering)
  2. Materialized views for complex aggregations (refresh hourly)
  3. Pre-compute reports async with BullMQ → store result → serve from cache
  4. Pagination — don't return 100K rows, paginate with cursor
  5. Read replica — route heavy queries to a read replica

For long-running reports:
  • Don't make the user wait — async pattern:
    POST /reports → 202 Accepted, returns reportId
    Worker generates report → stores in S3
    GET /reports/:id → returns status or download URL
    Optional: WebSocket/SSE for real-time progress
```

```typescript
// Async report generation pattern
@Post('reports')
@HttpCode(202)
async createReport(@Body() dto: CreateReportDto, @User() user: AuthUser) {
  const report = await this.reportRepo.save({
    userId: user.id,
    type: dto.type,
    status: 'processing',
    filters: dto.filters,
  });

  await this.reportQueue.add('generate', {
    reportId: report.id,
    type: dto.type,
    filters: dto.filters,
  });

  return { reportId: report.id, status: 'processing' };
}

// Worker
@Processor('reports')
export class ReportProcessor {
  @Process('generate')
  async generate(job: Job<ReportJobData>) {
    const data = await this.queryService.runReport(job.data.filters);
    const csv = this.csvService.generate(data);
    const s3Key = `reports/${job.data.reportId}.csv`;
    await this.s3.upload(s3Key, csv);

    await this.reportRepo.update(job.data.reportId, {
      status: 'completed',
      fileUrl: s3Key,
    });
  }
}
```

---

### Q: "Your GraphQL endpoint has N+1 query problem. How do you fix it?"

**Answer:**
```
Problem:
  query { courses { modules { lessons { ... } } } }
  → 1 query for courses
  → N queries for modules (one per course)
  → N×M queries for lessons (one per module)

Solution: DataLoader (batching + caching per request)
```

```typescript
// Without DataLoader: N+1
@ResolveField()
async modules(@Parent() course: Course) {
  return this.moduleRepo.find({ where: { courseId: course.id } }); // Called N times!
}

// With DataLoader: 1 batched query per field
@ResolveField()
async modules(@Parent() course: Course, @Context() ctx: GqlContext) {
  return ctx.loaders.modules.load(course.id); // Batched into 1 query
}

// DataLoader groups all load(courseId) calls in one tick → single query:
// SELECT * FROM modules WHERE course_id IN ($1, $2, $3, ...)
```

---

## 3. Database Migration & Schema Changes

### Q: "You need to rename a column in a table with 50M rows and zero downtime. How?"

**Answer:**
```
NEVER: ALTER TABLE RENAME COLUMN in production → breaks running app code instantly

Strategy: Expand-Contract Migration (4 phases)

Phase 1: Expand — Add new column
  ALTER TABLE users ADD COLUMN full_name VARCHAR(100);
  -- Backfill in batches:
  UPDATE users SET full_name = name WHERE id BETWEEN 1 AND 10000;
  -- Repeat for batches to avoid locking entire table

Phase 2: Dual-Write — App writes to BOTH columns
  await userRepo.save({ name: dto.name, fullName: dto.name });

Phase 3: Switch reads — App reads from new column
  // Change resolvers/controllers to read fullName
  // Verify with canary deployment

Phase 4: Contract — Drop old column
  ALTER TABLE users DROP COLUMN name;
  -- Only after all code uses fullName

Timeline: Each phase is a separate deployment, days apart.
```

---

### Q: "You need to migrate from MongoDB to PostgreSQL for a critical service."

**Answer:**
```
1. Dual-Write Phase:
   • New code writes to BOTH MongoDB and PostgreSQL
   • Existing reads still from MongoDB

2. Backfill:
   • Migrate historical data in batches (ETL script)
   • Verify row counts and checksums

3. Shadow Read:
   • Read from both, compare results, log discrepancies
   • Fix data inconsistencies

4. Switch Read:
   • Point reads to PostgreSQL
   • Monitor for any issues

5. Stop MongoDB Writes:
   • Remove dual-write code
   • Keep MongoDB as read-only backup for 30 days

6. Decommission:
   • Remove MongoDB references from code
```

---

## 4. Outage Handling & Incident Response

### Q: "Production is down. Walk me through your incident response."

**Answer:**
```
Minute 0-5: Detect & Acknowledge
  • Alert fires (CloudWatch, PagerDuty)
  • Acknowledge, check status page
  • Quick assessment: partial or full outage?

Minute 5-15: Triage
  • Check: recent deployment? → rollback immediately
  • Check: external dependency down? (Stripe, AWS region)
  • Check: infrastructure? (ECS tasks crashing, RDS CPU 100%)
  • Assemble incident team if needed

Minute 15-30: Mitigate
  • Priority: restore service, NOT find root cause
  • Rollback deployment if suspected cause
  • Scale up if resource exhaustion
  • Enable circuit breaker / fallback mode
  • Communicate: update status page, notify stakeholders

Post-Incident:
  • Root Cause Analysis (5 Whys)
  • Blameless post-mortem document
  • Action items with owners and deadlines
  • Update runbooks and monitoring
```

---

### Q: "A downstream service (Stripe) is down. How does your system handle it?"

**Answer:**
```
Real-time handling:
  1. Circuit breaker opens after threshold failures
  2. Fallback: queue payment for later processing
  3. Return 202 Accepted: "Payment is being processed"
  4. User sees "Processing..." instead of error

Recovery:
  1. Circuit breaker enters half-open state (30s timer)
  2. Test one request → if succeeds, close circuit
  3. Process queued payments from DLQ/queue
  4. Idempotency keys prevent double charges
```

```typescript
// Graceful degradation when Stripe is down
async processPayment(dto: PaymentDto) {
  try {
    return await this.paymentBreaker.fire(dto); // Circuit breaker
  } catch (error) {
    if (error.type === 'circuit-open') {
      // Queue for later — don't lose the payment
      await this.paymentQueue.add('retry-payment', dto, {
        attempts: 5,
        backoff: { type: 'exponential', delay: 60000 },
      });
      return { status: 'queued', message: 'Payment will be processed shortly' };
    }
    throw error;
  }
}
```

---

## 5. Architecture Trade-off Decisions

### Q: "Your team wants to adopt microservices. What questions do you ask?"

**Answer:**
```
Questions before migrating:
  1. Is the team big enough? Microservices need ≥3 teams, each owning a service.
  2. Do we have DevOps maturity? CI/CD, monitoring, container orchestration.
  3. Are the domain boundaries clear? Bad boundaries = distributed monolith.
  4. Is the current monolith actually causing problems? Don't fix what isn't broken.
  5. Can we start with a modular monolith? (NestJS modules with clear interfaces)

Red flags — DON'T do microservices if:
  • Small team (<5 developers)
  • Simple domain
  • No DevOps capability
  • "Everyone is doing it" is the only reason
```

---

### Q: "SQL or NoSQL for a new feature?"

**Answer:**
```
Use SQL (PostgreSQL) when:
  • Data has clear relationships (users → orders → items)
  • Need ACID transactions (payments, inventory)
  • Complex queries with JOINs, aggregations
  • Schema is well-defined and stable

Use NoSQL (MongoDB) when:
  • Schema varies per document (product catalog with different attributes)
  • High write throughput needed
  • Denormalized reads (embed related data)
  • Rapid prototyping, schema evolving

Use Both when:
  • PostgreSQL for transactional core (users, orders, payments)
  • MongoDB for flexible content (course content, form responses)
  • Redis for caching and real-time state

At Accenture: PostgreSQL for tenant config, billing, users.
MongoDB for compliance training content (varied structure per client).
```

---

## 6. Scaling Challenges

### Q: "Your service handles 1K requests/sec. It needs to handle 100K. What do you do?"

**Answer:**
```
Layer-by-layer scaling:

1. CDN: Static assets, API responses for public data
   → Eliminates 50%+ of requests hitting your servers

2. Application Layer:
   → Horizontal scaling: ECS auto-scaling (CPU/memory based)
   → Stateless services: sessions in Redis, not in memory
   → Node.js cluster mode or PM2 (use all CPU cores)

3. Caching Layer:
   → Redis for hot data (user sessions, frequently accessed configs)
   → Application-level caching (SWR pattern)
   → 90% cache hit ratio = 90K requests never hit DB

4. Database Layer:
   → Read replicas for read-heavy traffic
   → Connection pooling (PgBouncer)
   → Sharding if single DB can't handle writes
   → Partitioning large tables (time-based partitions)

5. Async Processing:
   → Move non-critical work to queues (email, analytics, logging)
   → Reduces response time and server load

6. Infrastructure:
   → Multi-AZ deployment for availability
   → Lambda for spiky workloads (auto-scales to 0)
```

---

## 7. Security Incidents

### Q: "You discover a JWT secret has been committed to Git. What do you do?"

**Answer:**
```
Immediate (< 1 hour):
  1. Rotate the secret immediately in AWS Secrets Manager / SSM
  2. Deploy config change → all services pick up new secret
  3. All existing JWTs signed with old secret are now invalid
  4. Users will need to re-authenticate (acceptable for security)

Containment:
  5. Revoke the Git history: git filter-branch or BFG Repo Cleaner
  6. Force-push cleaned history (coordinate with team)
  7. Check if secret was in any public repo or CI/CD logs

Investigation:
  8. Audit access logs — was the secret used by unauthorized parties?
  9. Check for unusual API activity during exposure window

Prevention:
  10. Add pre-commit hook with tools like gitleaks or trufflehog
  11. .gitignore all .env files
  12. Use AWS Secrets Manager / SSM Parameter Store, never env files
  13. Enable GitHub secret scanning alerts
```

---

### Q: "A penetration test reveals SQL injection in one endpoint. How do you respond?"

**Answer:**
```
1. Fix immediately: Ensure parameterized queries everywhere
   ❌ query(`SELECT * FROM users WHERE id = '${id}'`)
   ✅ query('SELECT * FROM users WHERE id = $1', [id])
   ✅ TypeORM/Prisma (parameterized by default)

2. Audit: Search codebase for any raw SQL without parameters
   → grep for template literals in .query() calls

3. Validate: Add input validation at controller layer
   → @Param('id', ParseUUIDPipe) — only accepts valid UUIDs

4. Test: Add SQL injection test cases to security test suite
   → Input: "'; DROP TABLE users; --" should return 400, not 500

5. WAF: Enable AWS WAF SQL injection rule as defense-in-depth
```

---

## 8. Team & Process Scenarios

### Q: "A junior developer's PR introduces a security vulnerability. How do you handle it?"

**Answer:**
- **Never blame publicly.** Handle in private code review comments.
- Point out the specific vulnerability and explain the risk.
- Show the secure alternative with a code example.
- Add it to the team's security checklist / PR template.
- Consider: Is this a gap in our linting rules? Can we automate detection?
- Schedule a team knowledge-sharing session on the topic.

### Q: "You disagree with the tech lead's architecture decision. What do you do?"

**Answer:**
- Write down my concerns with **specific** technical reasons and trade-offs.
- Request a 1:1 discussion (not public Slack debate).
- Present alternatives with pros/cons.
- If the decision stands, **commit fully** — don't undermine it.
- Document the decision and rationale (ADR — Architecture Decision Record) so future team members understand why.

### Q: "You inherited a legacy codebase with no tests. Where do you start?"

**Answer:**
```
1. Don't rewrite — incrementally improve
2. Add tests to the most critical/fragile paths first:
   → Payment processing
   → Authentication
   → Core business logic
3. Write integration tests for API endpoints (highest ROI)
4. Add unit tests when modifying existing code (boy scout rule)
5. Set up CI/CD to run tests on every PR
6. Track coverage trend (not absolute %) — should always go up
```

---

**Previous:** [08 — Real-World System Designs ←](08-real-world-system-designs.md) | **Back to:** [README](README.md)
