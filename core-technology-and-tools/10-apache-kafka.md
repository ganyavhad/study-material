# Apache Kafka - Study Material

> Based on skills from [ganeshavhad.com](https://ganeshavhad.com) — Software Developer Specialist
> Kafka expertise for event-driven microservices, queuing, and real-time data streaming

---

## 1. What is Apache Kafka?

Apache Kafka is a **distributed event streaming platform** used for high-throughput, fault-tolerant, real-time data pipelines and streaming applications.

### Kafka vs Traditional Message Queues

| Feature | Kafka | SQS | RabbitMQ | Bull (Redis) |
|---------|-------|-----|----------|--------------|
| Throughput | Millions msg/sec | Thousands/sec | Thousands/sec | Thousands/sec |
| Retention | Configurable (days/forever) | 14 days max | Until consumed | Until consumed |
| Replay | Yes (offset-based) | No | No | No |
| Ordering | Per-partition | FIFO queues only | Per-queue | Per-queue |
| Consumer Groups | Native | No | Competing consumers | No |
| Persistence | Disk (append-only log) | AWS managed | Memory/Disk | Redis |
| Use Case | Event streaming, logs | Task queues, decoupling | RPC, task routing | Job processing |
| Scaling | Horizontal (partitions) | Auto (AWS) | Vertical + clustering | Redis cluster |

---

## 2. Core Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     KAFKA CLUSTER                            │
│                                                              │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │ Broker 1 │    │ Broker 2 │    │ Broker 3 │              │
│  │          │    │          │    │          │              │
│  │ Topic A  │    │ Topic A  │    │ Topic A  │              │
│  │ Part 0   │    │ Part 1   │    │ Part 2   │              │
│  │ (Leader) │    │ (Leader) │    │ (Leader) │              │
│  │          │    │          │    │          │              │
│  │ Topic A  │    │ Topic A  │    │ Topic A  │              │
│  │ Part 1   │    │ Part 0   │    │ Part 0   │              │
│  │ (Replica)│    │ (Replica)│    │ (Replica)│              │
│  └──────────┘    └──────────┘    └──────────┘              │
│                                                              │
│  ┌────────────────────────────────────────────┐             │
│  │           ZooKeeper / KRaft                │             │
│  │  (Metadata, Leader Election, Config)       │             │
│  └────────────────────────────────────────────┘             │
└─────────────────────────────────────────────────────────────┘

Producers ──────►  Brokers  ──────► Consumers
(write)            (store)          (read)
```

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Topic** | Named feed/category of messages (like a table) |
| **Partition** | Ordered, immutable sequence of records within a topic |
| **Offset** | Unique sequential ID of a record within a partition |
| **Broker** | Kafka server that stores data and serves clients |
| **Producer** | Publishes messages to topics |
| **Consumer** | Reads messages from topics |
| **Consumer Group** | Set of consumers sharing the work of reading a topic |
| **Replica** | Copy of a partition on another broker (fault tolerance) |
| **Leader** | The partition replica that handles all reads/writes |
| **ISR** | In-Sync Replicas — replicas caught up with the leader |

### How Partitions & Consumer Groups Work

```
Topic: "orders" (3 partitions)

Partition 0: [msg0, msg3, msg6, msg9 ...]   ──► Consumer A (Group 1)
Partition 1: [msg1, msg4, msg7, msg10 ...]   ──► Consumer B (Group 1)
Partition 2: [msg2, msg5, msg8, msg11 ...]   ──► Consumer C (Group 1)

                                              ──► Consumer X (Group 2) reads ALL
                                                  (single consumer in group)

Key insight: Each partition is consumed by exactly ONE consumer in a group.
Max parallelism = number of partitions.
```

---

## 3. Kafka with Node.js (KafkaJS)

### Setup

```bash
npm install kafkajs
```

### Connection & Configuration

```typescript
// kafka.config.ts
import { Kafka, logLevel } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'user-service',
  brokers: (process.env.KAFKA_BROKERS || 'localhost:9092').split(','),
  ssl: process.env.NODE_ENV === 'production',
  sasl: process.env.NODE_ENV === 'production' ? {
    mechanism: 'scram-sha-256',
    username: process.env.KAFKA_USERNAME!,
    password: process.env.KAFKA_PASSWORD!
  } : undefined,
  logLevel: logLevel.WARN,
  retry: {
    initialRetryTime: 300,
    retries: 10
  }
});

export const producer = kafka.producer({
  allowAutoTopicCreation: false,
  idempotent: true // Exactly-once semantics
});

export const consumer = kafka.consumer({
  groupId: 'user-service-group',
  sessionTimeout: 30000,
  heartbeatInterval: 3000
});

export default kafka;
```

### Producer

```typescript
// producer.ts
import { producer } from './kafka.config';
import { CompressionTypes } from 'kafkajs';

// Initialize producer
await producer.connect();

// Send a single message
async function publishUserCreated(user: { id: string; name: string; email: string }) {
  await producer.send({
    topic: 'user-events',
    messages: [{
      key: user.id,                            // Ensures ordering per user
      value: JSON.stringify({
        eventType: 'USER_CREATED',
        data: user,
        timestamp: new Date().toISOString(),
        version: 1
      }),
      headers: {
        'event-type': 'USER_CREATED',
        'source': 'user-service',
        'correlation-id': crypto.randomUUID()
      }
    }]
  });
}

// Batch send (high throughput)
async function publishBatch(events: Array<{ key: string; value: object }>) {
  await producer.send({
    topic: 'analytics-events',
    compression: CompressionTypes.GZIP,
    messages: events.map(e => ({
      key: e.key,
      value: JSON.stringify(e.value),
      timestamp: Date.now().toString()
    }))
  });
}

// Send to specific partition
async function publishToPartition(topic: string, partition: number, message: object) {
  await producer.send({
    topic,
    messages: [{
      partition,
      key: null,
      value: JSON.stringify(message)
    }]
  });
}

// Graceful shutdown
process.on('SIGTERM', async () => {
  await producer.disconnect();
  process.exit(0);
});
```

### Consumer

```typescript
// consumer.ts
import { consumer } from './kafka.config';

// Initialize consumer
await consumer.connect();

// Subscribe to topics
await consumer.subscribe({
  topics: ['user-events', 'order-events'],
  fromBeginning: false  // Start from latest (true = replay all)
});

// Process messages
await consumer.run({
  partitionsConsumedConcurrently: 3,  // Process partitions in parallel

  eachMessage: async ({ topic, partition, message, heartbeat }) => {
    const event = JSON.parse(message.value!.toString());
    const eventType = message.headers?.['event-type']?.toString();

    console.log({
      topic,
      partition,
      offset: message.offset,
      key: message.key?.toString(),
      eventType,
      timestamp: message.timestamp
    });

    try {
      switch (eventType) {
        case 'USER_CREATED':
          await handleUserCreated(event.data);
          break;
        case 'ORDER_PLACED':
          await handleOrderPlaced(event.data);
          break;
        default:
          console.warn(`Unknown event type: ${eventType}`);
      }

      // Call heartbeat for long-running processing
      await heartbeat();

    } catch (error) {
      console.error(`Failed to process message:`, {
        topic,
        partition,
        offset: message.offset,
        error: error.message
      });

      // Dead Letter Queue — send failed messages
      await sendToDeadLetterQueue(topic, message, error);
    }
  }
});

// Handle rebalancing
consumer.on('consumer.rebalancing', () => {
  console.log('Consumer group rebalancing...');
});

consumer.on('consumer.group_join', ({ payload }) => {
  console.log('Joined consumer group:', payload.groupId);
});

// Graceful shutdown
process.on('SIGTERM', async () => {
  await consumer.disconnect();
  process.exit(0);
});
```

---

## 4. Dead Letter Queue (DLQ) Pattern

```typescript
// dead-letter-queue.ts
import { producer } from './kafka.config';

const DLQ_TOPIC = 'dead-letter-queue';
const MAX_RETRIES = 3;

async function sendToDeadLetterQueue(
  originalTopic: string,
  message: KafkaMessage,
  error: Error
) {
  const retryCount = parseInt(
    message.headers?.['retry-count']?.toString() || '0',
    10
  );

  if (retryCount < MAX_RETRIES) {
    // Retry: send back to original topic with incremented retry count
    await producer.send({
      topic: originalTopic,
      messages: [{
        key: message.key,
        value: message.value,
        headers: {
          ...message.headers,
          'retry-count': (retryCount + 1).toString(),
          'last-error': error.message
        }
      }]
    });
  } else {
    // Max retries exceeded — send to DLQ
    await producer.send({
      topic: DLQ_TOPIC,
      messages: [{
        key: message.key,
        value: message.value,
        headers: {
          ...message.headers,
          'original-topic': originalTopic,
          'error-message': error.message,
          'failed-at': new Date().toISOString(),
          'total-attempts': (retryCount + 1).toString()
        }
      }]
    });
    console.error(`Message sent to DLQ after ${MAX_RETRIES} retries`);
  }
}
```

---

## 5. Kafka with NestJS

### Module Setup

```typescript
// kafka.module.ts
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';

@Module({
  imports: [
    ClientsModule.register([{
      name: 'KAFKA_SERVICE',
      transport: Transport.KAFKA,
      options: {
        client: {
          clientId: 'notification-service',
          brokers: ['localhost:9092']
        },
        consumer: {
          groupId: 'notification-group'
        },
        producer: {
          allowAutoTopicCreation: false
        }
      }
    }])
  ],
  exports: [ClientsModule]
})
export class KafkaModule {}
```

### Producer (NestJS)

```typescript
// order.service.ts
import { Inject, Injectable } from '@nestjs/common';
import { ClientKafka } from '@nestjs/microservices';

@Injectable()
export class OrderService {
  constructor(@Inject('KAFKA_SERVICE') private kafkaClient: ClientKafka) {}

  async onModuleInit() {
    await this.kafkaClient.connect();
  }

  async createOrder(orderDto: CreateOrderDto) {
    const order = await this.orderRepository.save(orderDto);

    // Publish event to Kafka
    this.kafkaClient.emit('order-events', {
      key: order.id,
      value: {
        eventType: 'ORDER_CREATED',
        data: order,
        timestamp: new Date().toISOString()
      }
    });

    return order;
  }
}
```

### Consumer (NestJS)

```typescript
// notification.controller.ts
import { Controller } from '@nestjs/common';
import { EventPattern, Payload, Ctx, KafkaContext } from '@nestjs/microservices';

@Controller()
export class NotificationController {
  @EventPattern('order-events')
  async handleOrderEvent(
    @Payload() event: { eventType: string; data: any },
    @Ctx() context: KafkaContext
  ) {
    const { offset, partition, topic } = context.getMessage();

    console.log(`Processing: ${topic}[${partition}] @ offset ${offset}`);

    switch (event.eventType) {
      case 'ORDER_CREATED':
        await this.sendOrderConfirmation(event.data);
        break;
      case 'ORDER_SHIPPED':
        await this.sendShippingNotification(event.data);
        break;
    }
  }

  @EventPattern('user-events')
  async handleUserEvent(@Payload() event: any) {
    if (event.eventType === 'USER_CREATED') {
      await this.sendWelcomeEmail(event.data);
    }
  }
}
```

### NestJS Hybrid Application (HTTP + Kafka)

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Connect Kafka microservice
  app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.KAFKA,
    options: {
      client: {
        brokers: ['localhost:9092']
      },
      consumer: {
        groupId: 'order-service-group'
      }
    }
  });

  await app.startAllMicroservices();
  await app.listen(3000);
}
bootstrap();
```

---

## 6. Event-Driven Architecture Patterns

### Event Sourcing with Kafka

```
Events (immutable log):
┌──────────────────────────────────────────────────────┐
│ offset:0        offset:1          offset:2           │
│ OrderCreated → ItemAdded →      PaymentProcessed →   │
│ {id:1,          {id:1,            {id:1,             │
│  customer:"A"}   item:"Widget"}    amount:29.99}     │
└──────────────────────────────────────────────────────┘

Current State = replay(events[0..N])
Order #1 = { id:1, customer:"A", items:["Widget"], paid:true, amount:29.99 }
```

```typescript
// Event Store Pattern
interface DomainEvent {
  eventId: string;
  aggregateId: string;
  eventType: string;
  data: unknown;
  timestamp: string;
  version: number;
}

// Publish domain events
async function publishDomainEvents(aggregateId: string, events: DomainEvent[]) {
  await producer.send({
    topic: 'domain-events',
    messages: events.map(event => ({
      key: aggregateId,
      value: JSON.stringify(event),
      headers: {
        'event-type': event.eventType,
        'aggregate-id': aggregateId,
        'version': event.version.toString()
      }
    }))
  });
}

// Rebuild state from events
async function rebuildAggregate(aggregateId: string): Promise<OrderState> {
  const events = await getEventsForAggregate(aggregateId);
  return events.reduce((state, event) => applyEvent(state, event), initialState);
}
```

### CQRS with Kafka

```
                    ┌─────────────┐
Commands ──────────►│  Write DB   │
 (POST/PUT/DELETE)  │ (PostgreSQL)│
                    └──────┬──────┘
                           │ Publish events
                    ┌──────▼──────┐
                    │    KAFKA    │
                    │  (Events)   │
                    └──────┬──────┘
                           │ Consume events
                    ┌──────▼──────┐
Queries ◄──────────│  Read DB    │
 (GET)              │ (MongoDB /  │
                    │  ElasticSearch)
                    └─────────────┘
```

### Saga Pattern with Kafka

```typescript
// Order saga — compensating transactions via Kafka

// Step 1: Order Service creates order
// Topic: order-events → { type: ORDER_CREATED, orderId: 1 }

// Step 2: Inventory Service reserves stock
// Listens: order-events (ORDER_CREATED)
// Topic: inventory-events → { type: STOCK_RESERVED, orderId: 1 }
//   OR: inventory-events → { type: STOCK_FAILED, orderId: 1 }

// Step 3: Payment Service processes payment
// Listens: inventory-events (STOCK_RESERVED)
// Topic: payment-events → { type: PAYMENT_COMPLETED, orderId: 1 }
//   OR: payment-events → { type: PAYMENT_FAILED, orderId: 1 }

// Compensations:
// If PAYMENT_FAILED → Inventory releases stock
// If STOCK_FAILED → Order marked as failed

@EventPattern('payment-events')
async handlePaymentEvent(@Payload() event: any) {
  if (event.type === 'PAYMENT_FAILED') {
    // Compensate: release inventory
    this.kafkaClient.emit('inventory-events', {
      key: event.orderId,
      value: { type: 'RELEASE_STOCK', orderId: event.orderId }
    });
    // Update order status
    await this.orderService.updateStatus(event.orderId, 'FAILED');
  }
}
```

---

## 7. Kafka Streams & Data Pipeline

### Log Aggregation Pipeline

```
App Servers                  Kafka                    Consumers
┌─────────┐              ┌──────────┐            ┌─────────────┐
│ Service A│──logs───────►│          │           │ ElasticSearch│
│ Service B│──logs───────►│  "logs"  │──────────►│ (search/viz) │
│ Service C│──logs───────►│  topic   │           └─────────────┘
└─────────┘              │          │           ┌─────────────┐
                         │          │──────────►│ S3 / HDFS   │
                         └──────────┘           │ (archival)   │
                                                └─────────────┘
```

### Real-Time Metrics

```typescript
// Windowed aggregation consumer
import { consumer } from './kafka.config';

const metrics = new Map<string, { count: number; sum: number }>();

await consumer.subscribe({ topics: ['api-requests'] });

await consumer.run({
  eachMessage: async ({ message }) => {
    const request = JSON.parse(message.value!.toString());
    const key = `${request.endpoint}-${getMinuteBucket()}`;

    const current = metrics.get(key) || { count: 0, sum: 0 };
    metrics.set(key, {
      count: current.count + 1,
      sum: current.sum + request.responseTime
    });
  }
});

// Flush metrics every minute
setInterval(async () => {
  for (const [key, value] of metrics) {
    await producer.send({
      topic: 'aggregated-metrics',
      messages: [{
        key,
        value: JSON.stringify({
          endpoint: key.split('-')[0],
          avgResponseTime: value.sum / value.count,
          requestCount: value.count,
          window: key.split('-')[1]
        })
      }]
    });
  }
  metrics.clear();
}, 60000);
```

---

## 8. Kafka Administration

### Topic Management

```bash
# Create topic
kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic user-events \
  --partitions 6 \
  --replication-factor 3 \
  --config retention.ms=604800000      # 7 days
  --config cleanup.policy=delete

# List topics
kafka-topics.sh --list --bootstrap-server localhost:9092

# Describe topic
kafka-topics.sh --describe --topic user-events --bootstrap-server localhost:9092

# Alter partitions (can only increase)
kafka-topics.sh --alter --topic user-events --partitions 12 \
  --bootstrap-server localhost:9092

# Consumer group status
kafka-consumer-groups.sh --describe --group user-service-group \
  --bootstrap-server localhost:9092

# Reset consumer offset
kafka-consumer-groups.sh --reset-offsets \
  --group user-service-group \
  --topic user-events \
  --to-earliest \
  --execute \
  --bootstrap-server localhost:9092
```

### Docker Compose (Development)

```yaml
# docker-compose.yml
version: '3.8'
services:
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    hostname: kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_LOG_RETENTION_HOURS: 168  # 7 days
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
    depends_on:
      - kafka
```

---

## 9. AWS MSK (Managed Streaming for Kafka)

```hcl
# Terraform — AWS MSK Cluster
resource "aws_msk_cluster" "main" {
  cluster_name           = "event-streaming-${var.environment}"
  kafka_version          = "3.5.1"
  number_of_broker_nodes = 3

  broker_node_group_info {
    instance_type   = "kafka.m5.large"
    client_subnets  = var.private_subnet_ids
    security_groups = [aws_security_group.msk.id]

    storage_info {
      ebs_storage_info {
        volume_size = 100  # GB
      }
    }
  }

  encryption_info {
    encryption_in_transit {
      client_broker = "TLS"
      in_cluster    = true
    }
  }

  logging_info {
    broker_logs {
      cloudwatch_logs {
        enabled   = true
        log_group = aws_cloudwatch_log_group.msk.name
      }
    }
  }

  configuration_info {
    arn      = aws_msk_configuration.main.arn
    revision = aws_msk_configuration.main.latest_revision
  }

  tags = {
    Environment = var.environment
  }
}

resource "aws_msk_configuration" "main" {
  name              = "event-streaming-config"
  kafka_versions    = ["3.5.1"]
  server_properties = <<PROPERTIES
auto.create.topics.enable=false
default.replication.factor=3
min.insync.replicas=2
num.partitions=6
log.retention.hours=168
PROPERTIES
}
```

---

## 10. Performance & Best Practices

### Producer Best Practices

| Setting | Recommendation | Why |
|---------|---------------|-----|
| `acks` | `all` (-1) | Ensures message is replicated before ack |
| `idempotent` | `true` | Exactly-once delivery |
| `compression` | `gzip` or `snappy` | Reduces network & disk usage |
| `batch.size` | 32768+ | Batch messages for throughput |
| `linger.ms` | 5-50ms | Wait to fill batches |
| `retries` | `MAX_INT` | Let producer retry indefinitely |
| `max.in.flight.requests` | 5 (with idempotent) | Parallelism without ordering issues |

### Consumer Best Practices

| Setting | Recommendation | Why |
|---------|---------------|-----|
| `enable.auto.commit` | `false` | Manual commit after processing |
| `auto.offset.reset` | `earliest` or `latest` | Depends on use case |
| `max.poll.records` | 500 | Balance throughput and processing time |
| `session.timeout.ms` | 30000 | Detect dead consumers |
| `heartbeat.interval.ms` | 3000 | 1/3 of session timeout |

### Topic Design Guidelines

```
✅ Good:
- user-events (domain-based)
- order.created, order.shipped (event-type based)
- logs.service-name (source-based)

❌ Bad:
- events (too generic)
- user-events-v2 (version in topic name)
- temp-topic (no purpose)

Partitions: Start with 6-12 per topic.
  Rule of thumb: #partitions >= max #consumers in a group
  More partitions = more parallelism but more resources
```

### Exactly-Once Semantics

```typescript
// Transactional producer
const producer = kafka.producer({
  idempotent: true,
  transactionalId: 'order-processor-tx'
});

await producer.connect();

// Atomic produce across topics
const transaction = await producer.transaction();
try {
  await transaction.send({
    topic: 'order-events',
    messages: [{ key: orderId, value: JSON.stringify(orderEvent) }]
  });
  await transaction.send({
    topic: 'inventory-events',
    messages: [{ key: orderId, value: JSON.stringify(inventoryEvent) }]
  });
  await transaction.sendOffsets({
    consumerGroupId: 'order-processor-group',
    topics: [{ topic: 'incoming-orders', partitions: [{ partition: 0, offset: '42' }] }]
  });
  await transaction.commit();
} catch (error) {
  await transaction.abort();
  throw error;
}
```

---

## 11. Monitoring & Observability

### Key Metrics to Watch

| Metric | Alert Threshold | Description |
|--------|----------------|-------------|
| Consumer Lag | > 10,000 | Messages behind. Consumers too slow |
| Under-replicated Partitions | > 0 | Replicas not in sync. Data at risk |
| Request Rate | Varies | Throughput per broker |
| Active Controller Count | != 1 | Exactly one controller should be active |
| ISR Shrink Rate | > 0 | Replicas falling behind |
| Log Flush Latency | > 100ms | Disk bottleneck |

```typescript
// Health check endpoint
app.get('/health/kafka', async (req, res) => {
  try {
    const admin = kafka.admin();
    await admin.connect();
    const topics = await admin.listTopics();
    const groups = await admin.listGroups();
    await admin.disconnect();

    res.json({
      status: 'healthy',
      topicCount: topics.length,
      consumerGroups: groups.groups.length
    });
  } catch (error) {
    res.status(503).json({ status: 'unhealthy', error: error.message });
  }
});
```

---

## 12. Interview Questions

1. **What is Apache Kafka?** — Distributed event streaming platform. Acts as a durable, fault-tolerant commit log. Not just a queue — supports replay, multiple consumers, and stream processing.

2. **Kafka vs RabbitMQ vs SQS?** — Kafka: event streaming, high throughput, replay. RabbitMQ: flexible routing, RPC. SQS: fully managed, simple task queues. Choose based on replay needs, throughput, and managed vs self-hosted.

3. **What is a Consumer Group?** — Consumers sharing the same group ID split partitions among themselves. Each partition is read by exactly one consumer in the group, enabling parallel processing.

4. **How does Kafka ensure ordering?** — Ordering is guaranteed within a single partition. Use message key to route related messages to the same partition (e.g., user ID as key).

5. **What is the ISR?** — In-Sync Replicas. Replicas that are fully caught up with the leader. `min.insync.replicas` ensures durability — a write only succeeds if enough replicas acknowledge it.

6. **How to handle consumer failures?** — Dead Letter Queue pattern. After N retries, send to DLQ topic. Monitor DLQ for manual intervention. Use idempotent consumers to handle reprocessing.

7. **Explain exactly-once semantics** — Achieved via idempotent producers (deduplication) + transactional producers (atomic writes across topics) + `read_committed` isolation on consumers.

8. **What is consumer lag?** — Difference between the latest offset in a partition and the consumer's committed offset. High lag = consumer can't keep up. Fix: add consumers (up to partition count), optimize processing, or increase partitions.

9. **Topic compaction vs deletion?** — Deletion: remove messages after retention period. Compaction: keep only the latest value per key (like a changelog). Use compaction for state/snapshot topics.

10. **KRaft vs ZooKeeper?** — KRaft (Kafka Raft) replaces ZooKeeper for metadata management (Kafka 3.3+). Simpler architecture, fewer dependencies, faster controller failover.

---

## 13. Practice Exercises

1. Set up Kafka locally with Docker and create/consume messages with KafkaJS
2. Build an event-driven microservice architecture (Order → Inventory → Payment) using Kafka
3. Implement the Dead Letter Queue pattern with retry logic
4. Create a NestJS hybrid app (HTTP + Kafka consumer)
5. Build a real-time dashboard consuming Kafka events via WebSocket
6. Implement the Saga pattern with compensating transactions
7. Set up consumer group scaling — test adding/removing consumers
8. Build a log aggregation pipeline: services → Kafka → ElasticSearch
