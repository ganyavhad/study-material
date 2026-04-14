# 05 — Message Queues & Async Processing

> Decouple services, handle spikes, and guarantee delivery — the backbone of resilient architectures.

---

## Table of Contents

1. [Why Async Processing](#1-why-async-processing)
2. [Message Queue Patterns](#2-message-queue-patterns)
3. [AWS SQS](#3-aws-sqs)
4. [Apache Kafka](#4-apache-kafka)
5. [BullMQ (Redis-based)](#5-bullmq-redis-based)
6. [Dead Letter Queues & Retry](#6-dead-letter-queues--retry)
7. [Idempotency](#7-idempotency)
8. [Interview Questions & Answers](#8-interview-questions--answers)

---

## 1. Why Async Processing

```
Synchronous (tight coupling):
User → API → Process Payment → Send Email → Update Inventory → Response
         └─────────── 2-5 seconds total ────────────────────┘

Asynchronous (decoupled):
User → API → Process Payment → Response (200ms)
               │
               └──► Queue ──► Email Worker
                    Queue ──► Inventory Worker

Benefits:
• Faster response times (return immediately)
• Resilience (if email service is down, retries later)
• Scale independently (add more workers for heavy load)
• Smooth traffic spikes (queue absorbs burst)
```

---

## 2. Message Queue Patterns

```
┌──────────────────────┬──────────────────────────────────────────┐
│ Pattern              │ Description                              │
├──────────────────────┼──────────────────────────────────────────┤
│ Point-to-Point       │ One producer, one consumer               │
│ (Queue)              │ Each message processed once               │
│                      │ SQS, BullMQ                               │
├──────────────────────┼──────────────────────────────────────────┤
│ Pub/Sub              │ One producer, many consumers              │
│ (Topic)              │ Each consumer gets a copy                 │
│                      │ SNS, Kafka topics, Redis Pub/Sub          │
├──────────────────────┼──────────────────────────────────────────┤
│ Fan-Out              │ One message triggers multiple actions     │
│                      │ SNS → multiple SQS queues                 │
├──────────────────────┼──────────────────────────────────────────┤
│ Request-Reply        │ Producer sends request, waits for reply   │
│                      │ With correlation ID                       │
└──────────────────────┴──────────────────────────────────────────┘
```

### Choosing the Right Queue

```
┌─────────────────┬──────────────┬──────────────┬──────────────┐
│ Feature         │ SQS          │ Kafka        │ BullMQ       │
├─────────────────┼──────────────┼──────────────┼──────────────┤
│ Type            │ Managed      │ Distributed  │ Redis-based  │
│ Ordering        │ FIFO option  │ Per partition │ Per queue    │
│ Throughput      │ ~3K msg/sec  │ millions/sec │ ~10K/sec     │
│ Retention       │ 14 days max  │ Configurable │ Configurable │
│ Replay          │ No           │ Yes          │ No           │
│ Consumer Groups │ No           │ Yes          │ No           │
│ Delay/Schedule  │ Yes (15min)  │ No           │ Yes          │
│ Priority        │ No           │ No           │ Yes          │
│ Best For        │ AWS tasks    │ Event stream │ Job queues   │
└─────────────────┴──────────────┴──────────────┴──────────────┘
```

---

## 3. AWS SQS

```typescript
// SQS Producer — Send message
import { SQSClient, SendMessageCommand } from '@aws-sdk/client-sqs';

const sqs = new SQSClient({ region: 'us-east-1' });

async function sendToQueue(queueUrl: string, payload: object) {
  await sqs.send(new SendMessageCommand({
    QueueUrl: queueUrl,
    MessageBody: JSON.stringify(payload),
    MessageAttributes: {
      eventType: { DataType: 'String', StringValue: 'ORDER_CREATED' },
    },
    // FIFO queue: prevent duplicates
    // MessageGroupId: 'order-group',
    // MessageDeduplicationId: orderId,
  }));
}

// SQS Consumer — Lambda handler
export const handler = async (event: SQSEvent) => {
  for (const record of event.Records) {
    try {
      const payload = JSON.parse(record.body);
      await processOrder(payload);
    } catch (error) {
      console.error('Processing failed:', error);
      throw error; // Let Lambda retry / send to DLQ
    }
  }
};

// SQS with NestJS consumer (polling)
@Injectable()
export class OrderQueueConsumer implements OnModuleInit {
  async onModuleInit() {
    this.pollMessages();
  }

  private async pollMessages() {
    while (true) {
      const response = await this.sqs.send(new ReceiveMessageCommand({
        QueueUrl: this.queueUrl,
        MaxNumberOfMessages: 10,
        WaitTimeSeconds: 20,     // Long polling (cheaper, fewer empty responses)
        VisibilityTimeout: 60,   // Time to process before re-delivery
      }));

      for (const msg of response.Messages || []) {
        try {
          await this.processMessage(JSON.parse(msg.Body!));
          await this.sqs.send(new DeleteMessageCommand({
            QueueUrl: this.queueUrl,
            ReceiptHandle: msg.ReceiptHandle!,
          }));
        } catch (error) {
          // Message becomes visible again after VisibilityTimeout
          console.error('Failed to process:', error);
        }
      }
    }
  }
}
```

### SNS + SQS Fan-Out

```
              ┌──────────┐
   Order ────►│   SNS    │
  Created     │  Topic   │
              └────┬─────┘
          ┌────────┼────────┐
    ┌─────▼──┐ ┌───▼───┐ ┌─▼────────┐
    │ SQS    │ │ SQS   │ │ SQS      │
    │ Email  │ │ Inven. │ │ Analytics│
    │ Queue  │ │ Queue  │ │ Queue    │
    └────────┘ └───────┘ └──────────┘
```

---

## 4. Apache Kafka

```
Kafka Architecture:
┌─────────────────────────────────────────────────┐
│                  Kafka Cluster                   │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐         │
│  │Broker 1 │  │Broker 2 │  │Broker 3 │         │
│  │Partition0│  │Partition1│  │Partition2│        │
│  │(Leader)  │  │(Leader)  │  │(Leader)  │        │
│  │Partition1│  │Partition2│  │Partition0│        │
│  │(Replica) │  │(Replica) │  │(Replica) │        │
│  └─────────┘  └─────────┘  └─────────┘         │
└─────────────────────────────────────────────────┘

Key Concepts:
• Topic: Named category of messages (e.g., "orders")
• Partition: Ordered, append-only log within a topic
• Offset: Position of message in partition
• Consumer Group: Set of consumers that share partitions
• Replication Factor: Copies across brokers (typically 3)
```

```typescript
// Kafka Producer with KafkaJS
import { Kafka } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'order-service',
  brokers: ['kafka1:9092', 'kafka2:9092', 'kafka3:9092'],
});

const producer = kafka.producer();

async function publishOrderEvent(order: Order) {
  await producer.send({
    topic: 'order-events',
    messages: [{
      key: order.userId,           // Same user → same partition → ordering
      value: JSON.stringify({
        eventType: 'ORDER_CREATED',
        data: order,
        timestamp: Date.now(),
      }),
      headers: {
        correlationId: order.id,
      },
    }],
  });
}

// Kafka Consumer Group
const consumer = kafka.consumer({ groupId: 'notification-service' });

async function startConsumer() {
  await consumer.connect();
  await consumer.subscribe({ topic: 'order-events', fromBeginning: false });

  await consumer.run({
    eachMessage: async ({ topic, partition, message }) => {
      const event = JSON.parse(message.value!.toString());

      switch (event.eventType) {
        case 'ORDER_CREATED':
          await sendOrderConfirmationEmail(event.data);
          break;
        case 'ORDER_SHIPPED':
          await sendShippingNotification(event.data);
          break;
      }
    },
  });
}

// NestJS + Kafka Microservice
@Controller()
export class OrderConsumer {
  @EventPattern('order-events')
  handleOrderEvent(@Payload() data: OrderEvent, @Ctx() context: KafkaContext) {
    const { offset } = context.getMessage();
    console.log(`Processing offset ${offset}:`, data);
    return this.orderService.processEvent(data);
  }
}
```

---

## 5. BullMQ (Redis-based)

```typescript
// BullMQ — Feature-rich job queue for Node.js
import { Queue, Worker } from 'bullmq';
import Redis from 'ioredis';

const connection = new Redis({ host: 'localhost', port: 6379, maxRetriesPerRequest: null });

// Define queue
const emailQueue = new Queue('email', { connection });

// Add jobs with options
await emailQueue.add('welcome-email', {
  to: 'user@example.com',
  template: 'welcome',
  data: { name: 'Ganesh' },
}, {
  attempts: 3,                    // Retry 3 times
  backoff: { type: 'exponential', delay: 1000 }, // 1s, 2s, 4s
  delay: 5000,                    // Delay 5 seconds before processing
  removeOnComplete: { count: 1000 }, // Keep last 1000 completed
  removeOnFail: { count: 5000 },
  priority: 1,                    // 1 = highest priority
});

// Scheduled/Recurring jobs
await emailQueue.add('daily-digest', { type: 'digest' }, {
  repeat: { pattern: '0 9 * * *' }, // Cron: every day at 9 AM
});

// Worker — processes jobs
const worker = new Worker('email', async (job) => {
  console.log(`Processing job ${job.id}: ${job.name}`);

  switch (job.name) {
    case 'welcome-email':
      await sendEmail(job.data.to, job.data.template, job.data.data);
      break;
    case 'daily-digest':
      await generateAndSendDigest();
      break;
  }

  return { sent: true }; // Return value stored in job result
}, {
  connection,
  concurrency: 5,        // Process 5 jobs simultaneously
  limiter: {
    max: 100,             // Max 100 jobs per duration
    duration: 60000,      // Per minute (rate limiting)
  },
});

worker.on('completed', (job) => console.log(`Job ${job.id} completed`));
worker.on('failed', (job, err) => console.error(`Job ${job?.id} failed:`, err));
```

### BullMQ in NestJS

```typescript
// Module setup
@Module({
  imports: [
    BullModule.registerQueue({ name: 'email' }),
  ],
})
export class EmailModule {}

// Producer
@Injectable()
export class EmailService {
  constructor(@InjectQueue('email') private emailQueue: Queue) {}

  async sendWelcomeEmail(userId: string) {
    await this.emailQueue.add('welcome', { userId }, { attempts: 3 });
  }
}

// Consumer
@Processor('email')
export class EmailProcessor {
  @Process('welcome')
  async handleWelcome(job: Job<{ userId: string }>) {
    const user = await this.userService.findById(job.data.userId);
    await this.mailer.send(user.email, 'Welcome!', 'welcome-template');
  }

  @OnQueueFailed()
  onFailed(job: Job, error: Error) {
    this.logger.error(`Email job ${job.id} failed: ${error.message}`);
  }
}
```

---

## 6. Dead Letter Queues & Retry

```
Retry Flow with DLQ:

  ┌──────┐    ┌──────────┐    ┌────────┐
  │ Main │───►│ Consumer │───►│ Success │
  │Queue │    │          │    └────────┘
  └──┬───┘    └────┬─────┘
     │             │ Fail (retry 1, 2, 3)
     │             │
     │        ┌────▼─────┐
     │        │   DLQ    │  ← Messages that failed all retries
     │        │(Dead     │
     │        │ Letter)  │
     │        └──────────┘
     │             │
     │        ┌────▼──────────────┐
     │        │ Manual Review /   │
     │        │ Alert / Dashboard │
     │        └───────────────────┘

Retry Strategies:
• Fixed delay:       1s, 1s, 1s
• Linear backoff:    1s, 2s, 3s
• Exponential:       1s, 2s, 4s, 8s, 16s
• Exponential+jitter: 1s±0.5s, 2s±1s, 4s±2s (prevents thundering herd)
```

```typescript
// SQS DLQ Configuration (via Terraform)
// Main Queue → maxReceiveCount: 3 → DLQ

// BullMQ retry with exponential backoff
await queue.add('process-payment', paymentData, {
  attempts: 5,
  backoff: {
    type: 'exponential',
    delay: 2000, // 2s, 4s, 8s, 16s, 32s
  },
});

// Custom retry strategy
await queue.add('critical-job', data, {
  attempts: 5,
  backoff: {
    type: 'custom',
  },
});

// In worker:
const worker = new Worker('queue', handler, {
  settings: {
    backoffStrategy: (attemptsMade: number) => {
      // Exponential with jitter
      const delay = Math.pow(2, attemptsMade) * 1000;
      const jitter = Math.random() * 1000;
      return delay + jitter;
    },
  },
});
```

---

## 7. Idempotency

```
Problem: Network failure → client retries → duplicate processing

  Client ──► API ──► Process Payment ──► DB
       │         │
       │ timeout │ (payment actually succeeded!)
       │         │
       └── Retry ──► Process Payment ──► DUPLICATE CHARGE!

Solution: Idempotency Key

  Client ──► API (idempotency_key: "abc123")
       │
       │ ┌─ Check: key "abc123" seen before? ──► Yes → return cached result
       │ └─ No → process → store result with key → return
       │
       └── Retry (same key "abc123") ──► return cached result (no duplicate)
```

```typescript
// Idempotency middleware for NestJS
@Injectable()
export class IdempotencyMiddleware implements NestMiddleware {
  constructor(private redis: Redis) {}

  async use(req: Request, res: Response, next: NextFunction) {
    if (req.method !== 'POST' && req.method !== 'PUT') return next();

    const idempotencyKey = req.headers['idempotency-key'] as string;
    if (!idempotencyKey) return next();

    const cacheKey = `idempotency:${idempotencyKey}`;
    const cached = await this.redis.get(cacheKey);

    if (cached) {
      const { statusCode, body } = JSON.parse(cached);
      return res.status(statusCode).json(body);
    }

    // Acquire lock to prevent concurrent processing of same key
    const lockKey = `lock:${cacheKey}`;
    const lockAcquired = await this.redis.set(lockKey, '1', 'EX', 30, 'NX');
    if (!lockAcquired) {
      return res.status(409).json({ error: 'Request already in progress' });
    }

    // Capture response
    const originalJson = res.json.bind(res);
    res.json = (body: any) => {
      // Store result for 24 hours
      this.redis.set(cacheKey, JSON.stringify({
        statusCode: res.statusCode,
        body,
      }), 'EX', 86400);
      this.redis.del(lockKey);
      return originalJson(body);
    };

    next();
  }
}

// Database-level idempotency (Stripe pattern)
async function processPayment(idempotencyKey: string, paymentData: PaymentDto) {
  return await dataSource.transaction(async (manager) => {
    // Check if already processed
    const existing = await manager.findOne(IdempotencyRecord, {
      where: { key: idempotencyKey },
    });

    if (existing) return existing.response;

    // Process payment
    const result = await stripeClient.charges.create(paymentData, {
      idempotencyKey,  // Stripe natively supports idempotency keys
    });

    // Store result
    await manager.save(IdempotencyRecord, {
      key: idempotencyKey,
      response: result,
      expiresAt: new Date(Date.now() + 24 * 60 * 60 * 1000),
    });

    return result;
  });
}
```

---

## 8. Interview Questions & Answers

### Q1: When would you use SQS vs Kafka vs BullMQ?

**Answer:**
- **SQS:** Simple task queues on AWS, Lambda integration, low maintenance. We use it for async email/notification delivery.
- **Kafka:** High-throughput event streaming, event sourcing, replay capability, multiple consumer groups. Use for order events consumed by 5+ services.
- **BullMQ:** Feature-rich job queue (priorities, scheduling, rate limiting). Use for background jobs in Node.js — PDF generation, data imports.

### Q2: How do you guarantee exactly-once processing?

**Answer:**
- True exactly-once is impossible in distributed systems. We achieve **effectively once** through:
  1. **At-least-once delivery** (SQS/Kafka guaranteed)
  2. **Idempotent consumers** (deduplicate using idempotency key + DB/Redis)
  3. **Transactional outbox** (write event + business data in same DB transaction)
- Stripe uses idempotency keys — same key = same result, no duplicate charges.

### Q3: What happens if a consumer crashes mid-processing?

**Answer:**
- **SQS:** Message becomes visible again after `VisibilityTimeout` expires → another consumer picks it up.
- **Kafka:** Offset not committed → consumer group reprocesses from last committed offset.
- **BullMQ:** Job returns to queue after `lockDuration` expires → retried by another worker.
- Key: ensure processing is idempotent so retries don't cause issues.

### Q4: Explain the outbox pattern.

**Answer:**
- **Problem:** How to atomically update DB AND publish event? If DB succeeds but publish fails (or vice versa), data is inconsistent.
- **Solution:** Write the event to an "outbox" table in the SAME transaction as the business data. A separate process polls the outbox and publishes events.
- Guarantees: if the transaction commits, the event will eventually be published.

---

**Previous:** [04 — Caching Strategies ←](04-caching-strategies.md) | **Next:** [06 — Microservices & Distributed Patterns →](06-microservices-distributed-patterns.md)
