# 08 — Real-World System Design Problems

> Full system design walkthroughs using the RESHADE framework.

---

## Table of Contents

1. [URL Shortener (like bit.ly)](#1-url-shortener)
2. [Real-Time Chat System](#2-real-time-chat-system)
3. [Notification Service](#3-notification-service)
4. [Payment Processing System](#4-payment-processing-system)
5. [Learning Management System (LMS)](#5-learning-management-system)
6. [Design Checklist](#6-design-checklist)

---

## RESHADE Framework Recap

```
R — Requirements    : Clarify functional & non-functional
E — Estimation      : DAU, QPS, storage, bandwidth
S — Storage         : Database choices, schema
H — High-level      : Architecture diagram, major components
A — API             : Endpoints / queries design
D — Deep Dive       : Scaling, caching, edge cases
E — Evaluate        : Trade-offs, bottlenecks, improvements
```

---

## 1. URL Shortener

### Requirements
- **Functional:** Create short URL, redirect to original, optional custom alias, TTL/expiry, analytics (click count).
- **Non-functional:** Low latency redirect (<100ms), high availability, 100M URLs/month.

### Estimation
```
Write: 100M/month = ~40 URLs/sec
Read:  10:1 ratio = 400 redirects/sec
Storage: 100M × 500 bytes = 50GB/month → 3TB over 5 years
URL length: 7 chars (base62) = 62^7 = 3.5 trillion combinations
```

### Architecture

```
┌────────┐   POST /shorten   ┌──────────┐        ┌──────────┐
│ Client │──────────────────►│  API     │───────►│PostgreSQL│
└────────┘                   │  Server  │        │(URL map) │
     │                       │ (NestJS) │        └──────────┘
     │   GET /abc1234        └────┬─────┘
     │       │                    │
     │       ▼                    │
     │  ┌──────────┐        ┌────▼─────┐
     │  │  CDN /   │◄───────│  Redis   │
     │  │  Cache   │        │ (cache)  │
     │  └──────────┘        └──────────┘
     │       │ cache miss         │
     │       └───────────────────►│
     │                            │
     └── 301 Redirect ◄──────────┘
```

### Key Design Decisions

```typescript
// Short URL generation — Base62 encoding of auto-increment ID
function encodeBase62(num: number): string {
  const chars = '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz';
  let result = '';
  while (num > 0) {
    result = chars[num % 62] + result;
    num = Math.floor(num / 62);
  }
  return result.padStart(7, '0');
}

// Alternative: Pre-generate IDs with a Key Generation Service (KGS)
// KGS generates batch of unique keys → write service picks from pool
// Avoids collision, works across multiple instances

// Schema
// CREATE TABLE urls (
//   id BIGSERIAL PRIMARY KEY,
//   short_code VARCHAR(10) UNIQUE NOT NULL,
//   original_url TEXT NOT NULL,
//   user_id UUID,
//   created_at TIMESTAMPTZ DEFAULT NOW(),
//   expires_at TIMESTAMPTZ,
//   click_count BIGINT DEFAULT 0
// );
// CREATE INDEX idx_short_code ON urls(short_code);

// Redirect with caching
@Get(':code')
async redirect(@Param('code') code: string, @Res() res: Response) {
  // 1. Check Redis cache
  let url = await this.redis.get(`url:${code}`);

  if (!url) {
    // 2. Cache miss — query DB
    const record = await this.urlRepo.findOne({ where: { shortCode: code } });
    if (!record) throw new NotFoundException();
    url = record.originalUrl;
    await this.redis.setex(`url:${code}`, 3600, url);
  }

  // 3. Async analytics (don't block redirect)
  this.analyticsQueue.add('click', { code, timestamp: Date.now() });

  // 301 = permanent redirect (cacheable), 302 = temporary (for analytics accuracy)
  return res.redirect(302, url);
}
```

---

## 2. Real-Time Chat System

### Requirements
- **Functional:** 1:1 and group chat, online status, read receipts, message history, media sharing.
- **Non-functional:** Real-time delivery (<200ms), offline message delivery, 10M DAU.

### Architecture

```
┌────────┐  WebSocket   ┌──────────┐       ┌──────────┐
│ Client │◄────────────►│  Chat    │──────►│  Redis   │
│ (Web/  │              │  Server  │       │  PubSub  │
│ Mobile)│              │ (NestJS) │       └──────────┘
└────────┘              └───┬──┬───┘
                            │  │         ┌──────────┐
                            │  └────────►│ Cassandra│ (messages)
                            │            └──────────┘
                            │
                       ┌────▼─────┐      ┌──────────┐
                       │  Kafka   │─────►│Notif Svc │
                       │(events)  │      │(push/SMS)│
                       └──────────┘      └──────────┘

Why WebSocket? Persistent connection, server can push messages.
Why Redis PubSub? Route messages when sender/receiver on different servers.
Why Cassandra? Write-heavy, time-series message storage, partition by chatId.
```

```typescript
// WebSocket Gateway (NestJS)
@WebSocketGateway({ cors: true })
export class ChatGateway {
  @WebSocketServer() server: Server;

  // Track which server each user is connected to
  private userSockets = new Map<string, Socket>();

  @SubscribeMessage('sendMessage')
  async handleMessage(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { chatId: string; content: string },
  ) {
    const userId = client.handshake.auth.userId;

    // 1. Persist to Cassandra
    const message = await this.messageService.save({
      chatId: data.chatId,
      senderId: userId,
      content: data.content,
      timestamp: new Date(),
    });

    // 2. Publish to Redis PubSub (for cross-server delivery)
    await this.redis.publish(`chat:${data.chatId}`, JSON.stringify(message));

    // 3. Emit to local clients in this room
    this.server.to(data.chatId).emit('newMessage', message);
  }

  @SubscribeMessage('joinChat')
  handleJoin(@ConnectedSocket() client: Socket, @MessageBody() chatId: string) {
    client.join(chatId);
  }

  handleConnection(client: Socket) {
    const userId = client.handshake.auth.userId;
    this.userSockets.set(userId, client);
    // Broadcast online status
    this.server.emit('userOnline', { userId });
  }
}

// Message storage — Cassandra schema
// CREATE TABLE messages (
//   chat_id UUID,
//   message_id TIMEUUID,
//   sender_id UUID,
//   content TEXT,
//   created_at TIMESTAMP,
//   PRIMARY KEY (chat_id, message_id)
// ) WITH CLUSTERING ORDER BY (message_id DESC);
// Partition by chat_id, ordered by time — efficient for "last 50 messages"
```

---

## 3. Notification Service

### Requirements
- **Functional:** Push notifications, email, SMS, in-app. Template management. User preferences (opt-in/out). Scheduling.
- **Non-functional:** At-least-once delivery, 1M notifications/day, deduplication.

### Architecture

```
┌──────────┐     ┌──────────────┐     ┌──────────┐
│ Services │────►│ Notification │────►│  Kafka   │
│(order,   │     │    API       │     │ (topic   │
│ payment) │     │  (NestJS)    │     │per type) │
└──────────┘     └──────────────┘     └────┬─────┘
                                      ┌────┼────────────┐
                                ┌─────▼─┐ ┌▼───────┐ ┌──▼────┐
                                │ Email │ │  Push  │ │  SMS  │
                                │Worker │ │ Worker │ │Worker │
                                │(SES)  │ │(FCM)  │ │(Twilio│
                                └───────┘ └────────┘ └───────┘

   Kafka Topics: notification.email, notification.push, notification.sms
   Each worker: consume → render template → send → mark delivered
```

```typescript
// Notification service — route to correct channel
@Injectable()
export class NotificationService {
  async send(notification: NotificationDto) {
    // 1. Check user preferences
    const prefs = await this.userPrefsService.get(notification.userId);
    const channels = notification.channels.filter(ch => prefs[ch] !== false);

    // 2. Render template
    const rendered = await this.templateService.render(
      notification.templateId,
      notification.data,
    );

    // 3. Deduplicate (idempotency key)
    const dedupKey = `notif:${notification.idempotencyKey}`;
    const exists = await this.redis.set(dedupKey, '1', 'EX', 86400, 'NX');
    if (!exists) return; // Already sent

    // 4. Publish to each channel topic
    for (const channel of channels) {
      await this.kafka.emit(`notification.${channel}`, {
        userId: notification.userId,
        channel,
        subject: rendered.subject,
        body: rendered.body,
        metadata: notification.metadata,
      });
    }
  }
}

// Email worker — consumes from notification.email topic
@Controller()
export class EmailWorker {
  @EventPattern('notification.email')
  async handleEmail(@Payload() data: EmailNotification) {
    try {
      await this.sesClient.send(new SendEmailCommand({
        Source: 'noreply@example.com',
        Destination: { ToAddresses: [data.email] },
        Message: {
          Subject: { Data: data.subject },
          Body: { Html: { Data: data.body } },
        },
      }));
      await this.markDelivered(data.notificationId);
    } catch (error) {
      // Will be retried by Kafka consumer (at-least-once)
      throw error;
    }
  }
}
```

---

## 4. Payment Processing System

### Requirements
- **Functional:** Process payments (Stripe), refunds, subscription billing, webhook handling, transaction history.
- **Non-functional:** Exactly-once processing (idempotency), PCI compliance (no card data), audit trail.

### Architecture

```
┌────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ Client │──►│ API Svc  │──►│ Payment  │──►│  Stripe  │
│        │   │ (NestJS) │   │  Svc     │   │   API    │
└────────┘   └──────────┘   └────┬─────┘   └────┬─────┘
                                 │               │
                            ┌────▼─────┐         │
                            │PostgreSQL│         │
                            │(txn log) │    ┌────▼─────┐
                            └──────────┘    │ Webhooks │
                                            │(events)  │
                                            └────┬─────┘
                                            ┌────▼─────┐
                                            │ Webhook  │
                                            │ Handler  │
                                            │ (NestJS) │
                                            └──────────┘

Key: Client NEVER sends card data to your server.
Use Stripe Elements/Checkout → tokenized on Stripe's side → you only handle tokens.
```

```typescript
// Payment service with idempotency
@Injectable()
export class PaymentService {
  async createPayment(dto: CreatePaymentDto): Promise<Payment> {
    // 1. Create local record first (pending state)
    const payment = await this.paymentRepo.save({
      userId: dto.userId,
      amount: dto.amount,
      currency: dto.currency,
      status: 'pending',
      idempotencyKey: dto.idempotencyKey,
    });

    try {
      // 2. Call Stripe with idempotency key
      const paymentIntent = await this.stripe.paymentIntents.create(
        {
          amount: dto.amount,
          currency: dto.currency,
          customer: dto.stripeCustomerId,
          payment_method: dto.paymentMethodId,
          confirm: true,
          metadata: { internalPaymentId: payment.id },
        },
        { idempotencyKey: dto.idempotencyKey },
      );

      // 3. Update local record
      payment.stripePaymentIntentId = paymentIntent.id;
      payment.status = paymentIntent.status === 'succeeded' ? 'completed' : 'processing';
      return this.paymentRepo.save(payment);

    } catch (error) {
      payment.status = 'failed';
      payment.failureReason = error.message;
      await this.paymentRepo.save(payment);
      throw error;
    }
  }
}

// Stripe webhook handler — source of truth for payment status
@Controller('webhooks')
export class StripeWebhookController {
  @Post('stripe')
  async handleWebhook(@Req() req: RawBodyRequest<Request>) {
    const sig = req.headers['stripe-signature'];
    const event = this.stripe.webhooks.constructEvent(
      req.rawBody, sig, this.webhookSecret, // Verify signature!
    );

    // Idempotent processing — store event ID
    const processed = await this.redis.set(
      `stripe-event:${event.id}`, '1', 'EX', 86400, 'NX',
    );
    if (!processed) return { received: true }; // Already handled

    switch (event.type) {
      case 'payment_intent.succeeded':
        await this.paymentService.markSucceeded(event.data.object);
        break;
      case 'payment_intent.payment_failed':
        await this.paymentService.markFailed(event.data.object);
        break;
      case 'invoice.payment_succeeded':
        await this.subscriptionService.renewAccess(event.data.object);
        break;
    }

    return { received: true };
  }
}
```

---

## 5. Learning Management System (LMS)

### Requirements
- **Functional:** Course management, enrollment, content delivery (video/text/quizzes), user progress tracking, certificates, multi-tenant.
- **Non-functional:** Content to 50k concurrent users, SCORM compliance, 99.9% availability.

### Architecture

```
┌──────────┐  GraphQL  ┌──────────────┐
│ Angular  │◄────────►│  GraphQL     │
│   SPA    │          │  Gateway     │
└──────────┘          │  (NestJS)    │
                      └──────┬───────┘
              ┌───────────┬──┴───────────────┐
        ┌─────▼───┐ ┌─────▼───┐        ┌─────▼───┐
        │ Course  │ │ User &  │        │Progress │
        │  Svc    │ │ Auth Svc│        │  Svc    │
        └────┬────┘ └────┬────┘        └────┬────┘
             │           │                  │
        ┌────▼────┐ ┌────▼────┐        ┌────▼────┐
        │  PSQL   │ │  PSQL   │        │  Redis  │
        │(courses)│ │ (users) │        │+ Mongo  │
        └─────────┘ └─────────┘        └─────────┘

Content Delivery:
  Video → S3 + CloudFront (CDN) → signed URLs (time-limited)
  Documents → S3 → presigned download URLs
  Quizzes → in-app, stored in PostgreSQL with answers hashed

Multi-Tenant:
  Option A: Shared DB, tenant_id column (row-level security)
  Option B: Schema per tenant (PostgreSQL schemas)
  Option C: DB per tenant (full isolation, expensive)

  Recommended: Schema per tenant — good balance of isolation and cost.
```

```typescript
// Multi-tenant middleware — resolve tenant from subdomain/header
@Injectable()
export class TenantMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const tenantId = req.headers['x-tenant-id'] as string
      || req.hostname.split('.')[0]; // subdomain

    if (!tenantId) throw new BadRequestException('Tenant not identified');

    // Set tenant in async local storage for downstream use
    tenantContext.run({ tenantId }, () => next());
  }
}

// Dynamic schema selection (TypeORM)
@Injectable()
export class TenantConnectionService {
  getConnection(): Connection {
    const { tenantId } = tenantContext.getStore();
    return getConnection().createQueryRunner().connection
      .createQueryBuilder()
      .connection; // Uses schema set per request
  }
}

// Progress tracking — Redis for real-time, MongoDB for persistence
@Injectable()
export class ProgressService {
  async updateProgress(userId: string, lessonId: string, progress: number) {
    // Redis for real-time state (fast reads for UI)
    await this.redis.hset(
      `progress:${userId}`,
      lessonId,
      JSON.stringify({ progress, updatedAt: Date.now() }),
    );

    // Queue persistence to MongoDB (eventual consistency)
    await this.progressQueue.add('persist', { userId, lessonId, progress });
  }

  async getCourseProgress(userId: string, courseId: string) {
    const lessons = await this.lessonService.findByCourse(courseId);
    const progressMap = await this.redis.hgetall(`progress:${userId}`);

    return lessons.map(lesson => ({
      lessonId: lesson.id,
      title: lesson.title,
      progress: progressMap[lesson.id]
        ? JSON.parse(progressMap[lesson.id]).progress
        : 0,
    }));
  }
}
```

---

## 6. Design Checklist

Use this for any system design interview:

```
□ Clarified requirements (functional + non-functional)
□ Back-of-envelope estimation (QPS, storage, bandwidth)
□ Identified core entities and relationships
□ Chose databases (SQL vs NoSQL vs both)
□ Drew high-level architecture diagram
□ Defined API contracts
□ Addressed scaling (horizontal, caching, CDN, sharding)
□ Data partitioning strategy
□ Handled failure modes (circuit breaker, retry, DLQ)
□ Security (auth, rate limiting, input validation)
□ Monitoring & alerting
□ Discussed trade-offs made
```

---

**Previous:** [07 — API Design & Security ←](07-api-design-security.md) | **Next:** [09 — Scenario-Based Questions →](09-scenario-based-questions.md)
