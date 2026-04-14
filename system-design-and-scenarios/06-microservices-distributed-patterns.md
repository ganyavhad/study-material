# 06 — Microservices & Distributed Patterns

> Patterns for building resilient, loosely coupled services at scale.

---

## Table of Contents

1. [Monolith vs Microservices](#1-monolith-vs-microservices)
2. [Service Communication](#2-service-communication)
3. [API Gateway Pattern](#3-api-gateway-pattern)
4. [Saga Pattern (Distributed Transactions)](#4-saga-pattern-distributed-transactions)
5. [Circuit Breaker](#5-circuit-breaker)
6. [Service Discovery & Registry](#6-service-discovery--registry)
7. [Distributed Tracing & Observability](#7-distributed-tracing--observability)
8. [Event Sourcing](#8-event-sourcing)
9. [Interview Questions & Answers](#9-interview-questions--answers)

---

## 1. Monolith vs Microservices

```
Monolith:                           Microservices:
┌─────────────────────┐            ┌──────┐  ┌──────┐  ┌──────┐
│  User Module        │            │ User │  │Order │  │ Pay  │
│  Order Module       │            │ Svc  │  │ Svc  │  │ Svc  │
│  Payment Module     │            └──┬───┘  └──┬───┘  └──┬───┘
│  Notification Module│               │         │         │
│  ─────────────────  │            ┌──▼───┐  ┌──▼───┐  ┌──▼───┐
│  Shared Database    │            │UserDB│  │OrdDB │  │PayDB │
└─────────────────────┘            └──────┘  └──────┘  └──────┘

When to choose:
• Monolith: Small team (<5), early stage, simple domain
• Microservices: Large team, independent deployments needed,
  different scaling requirements per service
```

### Migration Strategy (Strangler Fig Pattern)

```
Phase 1: Identify bounded contexts
Phase 2: Extract one service at a time (start with least coupled)
Phase 3: Route traffic: old path → new service
Phase 4: Remove old code once stable

   ┌───────────────────────────────┐
   │         Monolith              │
   │  ┌──────┐                    │
   │  │ User │ ← Extract first    │
   │  │Module│────────────────┐   │
   │  └──────┘                │   │
   │  ┌──────┐ ┌──────┐      │   │
   │  │Order │ │ Pay  │      │   │
   │  └──────┘ └──────┘      │   │
   └──────────────────────┬───┘   │
                          │       │
                    ┌─────▼─────┐ │
                    │ User Svc  │ │ ← New microservice
                    │ (NestJS)  │ │
                    └───────────┘ │
```

---

## 2. Service Communication

```
┌──────────────────┬──────────────────────────────────────────────┐
│ Pattern          │ Use Case                                     │
├──────────────────┼──────────────────────────────────────────────┤
│ Sync HTTP/REST   │ Simple request-response, real-time needed   │
│ Sync gRPC        │ Low latency, strong typing, streaming       │
│ Async Events     │ Fire-and-forget, eventual consistency       │
│ Async Request-   │ Long processing, need response later        │
│   Reply          │                                              │
└──────────────────┴──────────────────────────────────────────────┘
```

```typescript
// Sync: HTTP call between services (NestJS HttpModule)
@Injectable()
export class OrderService {
  constructor(private httpService: HttpService) {}

  async createOrder(data: CreateOrderDto) {
    // Call user service to validate user
    const { data: user } = await firstValueFrom(
      this.httpService.get(`${USER_SERVICE_URL}/users/${data.userId}`),
    );

    // Call payment service
    const { data: payment } = await firstValueFrom(
      this.httpService.post(`${PAYMENT_SERVICE_URL}/charges`, {
        amount: data.total,
        userId: data.userId,
      }),
    );

    return this.orderRepo.save({ ...data, paymentId: payment.id });
  }
}

// Async: Event-driven communication via Kafka
@Injectable()
export class OrderService {
  constructor(@Inject('KAFKA_CLIENT') private kafka: ClientKafka) {}

  async createOrder(data: CreateOrderDto) {
    const order = await this.orderRepo.save(data);

    // Emit event — don't wait for downstream processing
    this.kafka.emit('order.created', {
      key: order.id,
      value: { orderId: order.id, userId: data.userId, items: data.items },
    });

    return order;
  }
}
```

---

## 3. API Gateway Pattern

```
                         ┌─────────────────┐
  Mobile App ───────────►│                 │
  Web App   ───────────►│   API Gateway   │
  3rd Party ───────────►│   (Kong/AWS)    │
                         │                 │
                         │ • Auth/JWT      │
                         │ • Rate Limiting │
                         │ • SSL Term.     │
                         │ • Request Route │
                         │ • Response Agg. │
                         └────────┬────────┘
                    ┌─────────────┼─────────────┐
              ┌─────▼───┐  ┌─────▼───┐  ┌──────▼──┐
              │User Svc │  │Order Svc│  │Pay Svc  │
              └─────────┘  └─────────┘  └─────────┘

BFF (Backend for Frontend) variant:
• Mobile BFF: Optimized payload, fewer fields
• Web BFF: Rich data, full responses
• Each BFF aggregates from same microservices
```

```typescript
// AWS API Gateway + Lambda (serverless API gateway)
// serverless.yml
// functions:
//   getUser:
//     handler: src/handlers/user.get
//     events:
//       - http:
//           path: /users/{id}
//           method: GET
//           authorizer: jwtAuthorizer
//           throttle:
//             burstLimit: 100
//             rateLimit: 50

// NestJS as API Gateway (GraphQL Federation)
@Module({
  imports: [
    GraphQLModule.forRoot<ApolloGatewayDriverConfig>({
      driver: ApolloGatewayDriver,
      gateway: {
        supergraphSdl: new IntrospectAndCompose({
          subgraphs: [
            { name: 'users', url: 'http://user-svc:3001/graphql' },
            { name: 'orders', url: 'http://order-svc:3002/graphql' },
            { name: 'products', url: 'http://product-svc:3003/graphql' },
          ],
        }),
      },
    }),
  ],
})
export class GatewayModule {}
```

---

## 4. Saga Pattern (Distributed Transactions)

```
Problem: Distributed transaction across multiple services
Solution: Saga — sequence of local transactions with compensating actions

Choreography Saga (event-driven):
  Order ──event──► Payment ──event──► Inventory ──event──► Notification
  Svc              Svc                Svc                  Svc
    │                │                  │
    │ ◄──fail──     │ ◄──compensate── │
    │ (cancel       │  (refund)       │ (restock)
    │  order)       │                 │

Orchestration Saga (central coordinator):
                  ┌──────────────┐
                  │    Saga      │
                  │ Orchestrator │
                  └──────┬───────┘
              ┌──────────┼──────────┐
              │          │          │
        ┌─────▼──┐ ┌─────▼──┐ ┌────▼───┐
        │ Order  │ │Payment │ │Inventory│
        │  Svc   │ │  Svc   │ │  Svc   │
        └────────┘ └────────┘ └────────┘
```

```typescript
// Orchestration Saga — Order Creation
@Injectable()
export class CreateOrderSaga {
  async execute(data: CreateOrderDto): Promise<Order> {
    const sagaLog: SagaStep[] = [];

    try {
      // Step 1: Reserve inventory
      const reservation = await this.inventoryService.reserve(data.items);
      sagaLog.push({ service: 'inventory', action: 'reserve', data: reservation });

      // Step 2: Process payment
      const payment = await this.paymentService.charge({
        amount: data.total,
        userId: data.userId,
        idempotencyKey: data.orderId,
      });
      sagaLog.push({ service: 'payment', action: 'charge', data: payment });

      // Step 3: Create order
      const order = await this.orderService.create({
        ...data,
        paymentId: payment.id,
        status: 'confirmed',
      });
      sagaLog.push({ service: 'order', action: 'create', data: order });

      return order;

    } catch (error) {
      // Compensate in reverse order
      await this.compensate(sagaLog);
      throw new OrderCreationFailedException(error);
    }
  }

  private async compensate(sagaLog: SagaStep[]) {
    for (const step of sagaLog.reverse()) {
      try {
        switch (step.service) {
          case 'payment':
            await this.paymentService.refund(step.data.id);
            break;
          case 'inventory':
            await this.inventoryService.release(step.data.reservationId);
            break;
          case 'order':
            await this.orderService.cancel(step.data.id);
            break;
        }
      } catch (compensateError) {
        // Log for manual intervention — compensating actions should be retried
        this.logger.error(`Compensation failed for ${step.service}`, compensateError);
      }
    }
  }
}
```

---

## 5. Circuit Breaker

```
States:
  ┌────────┐   failures > threshold   ┌────────┐
  │ CLOSED │ ─────────────────────── │  OPEN  │
  │(normal)│                          │(reject)│
  └───┬────┘ ◄──── success ────────── └───┬────┘
      │                                    │
      │                              timeout expires
      │                                    │
      │              ┌──────────┐          │
      │              │HALF-OPEN │◄─────────┘
      │              │(test one)│
      │              └────┬─────┘
      │                   │
      ◄── success ────────┘  (if fails → back to OPEN)
```

```typescript
// Using 'opossum' circuit breaker in Node.js
import CircuitBreaker from 'opossum';

const paymentBreaker = new CircuitBreaker(
  async (data: PaymentDto) => {
    const response = await axios.post(`${PAYMENT_URL}/charge`, data);
    return response.data;
  },
  {
    timeout: 5000,           // If function takes > 5s, trigger failure
    errorThresholdPercentage: 50, // Open circuit if 50% of requests fail
    resetTimeout: 30000,     // Try again after 30 seconds
    volumeThreshold: 10,     // Minimum 10 requests before tripping
  }
);

paymentBreaker.on('open', () => logger.warn('Payment circuit OPEN'));
paymentBreaker.on('halfOpen', () => logger.info('Payment circuit HALF-OPEN'));
paymentBreaker.on('close', () => logger.info('Payment circuit CLOSED'));

// Fallback when circuit is open
paymentBreaker.fallback(() => {
  return { status: 'queued', message: 'Payment will be processed shortly' };
});

// Usage
const result = await paymentBreaker.fire(paymentData);
```

---

## 6. Service Discovery & Registry

```
Client-Side Discovery:              Server-Side Discovery:
┌────────┐  2.call  ┌────────┐     ┌────────┐ 1.call ┌──────┐ 2.route ┌────────┐
│ Client │────────►│Svc Inst│     │ Client │──────►│  LB  │───────►│Svc Inst│
└───┬────┘         └────────┘     └────────┘       └──┬───┘        └────────┘
    │ 1.lookup                                        │ lookup
    │         ┌──────────┐                     ┌──────▼─────┐
    └────────►│ Registry │                     │  Registry  │
              │(Consul)  │                     │ (built-in) │
              └──────────┘                     └────────────┘

AWS:
• ECS Service Discovery (AWS Cloud Map)
• ALB Target Groups (automatic registration)
• Lambda (no discovery needed — API Gateway routes)
```

---

## 7. Distributed Tracing & Observability

```
Three Pillars of Observability:

1. Logs   — What happened (structured JSON logs)
2. Metrics — How much (request count, latency, error rate)
3. Traces  — Where (request flow across services)

Distributed Trace:
┌─────────────────────────────────────────────────────────────┐
│ Trace ID: abc-123                                           │
│                                                             │
│ API Gateway  ├──── 200ms ────────────────────────────────┤  │
│ User Svc       ├── 50ms ──┤                              │  │
│ Order Svc         ├──── 100ms ────┤                      │  │
│ Payment Svc          ├── 80ms ──┤                        │  │
│ Redis Cache       ├ 2ms ┤                                │  │
│ PostgreSQL           ├── 30ms ─┤                         │  │
└─────────────────────────────────────────────────────────────┘
```

```typescript
// Correlation ID middleware — trace requests across services
@Injectable()
export class CorrelationIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const correlationId = req.headers['x-correlation-id'] as string
      || crypto.randomUUID();

    req.headers['x-correlation-id'] = correlationId;
    res.setHeader('x-correlation-id', correlationId);

    // Attach to async local storage for access anywhere
    AsyncLocalStorage.run({ correlationId }, () => next());
  }
}

// Structured logging with correlation ID
@Injectable()
export class AppLogger {
  log(message: string, context?: object) {
    const store = asyncLocalStorage.getStore();
    console.log(JSON.stringify({
      timestamp: new Date().toISOString(),
      level: 'info',
      correlationId: store?.correlationId,
      message,
      ...context,
    }));
  }
}

// Forward correlation ID to downstream services
async callDownstreamService(url: string, data: any) {
  const store = asyncLocalStorage.getStore();
  return axios.post(url, data, {
    headers: { 'x-correlation-id': store?.correlationId },
  });
}
```

---

## 8. Event Sourcing

```
Traditional: Store current state
  users table: { id: 1, name: "Ganesh", email: "g@example.com" }

Event Sourcing: Store all events (append-only log)
  Event 1: UserCreated { name: "Ganesh", email: "g@a.com" }
  Event 2: EmailChanged { email: "g@example.com" }
  Event 3: NameUpdated { name: "Ganesh Avhad" }

  Current state = replay all events

Benefits:
• Complete audit trail
• Replay/rebuild state
• Temporal queries ("what was the state on Jan 1?")
• Natural fit with Kafka

Drawbacks:
• Complex queries → need CQRS read model
• Event schema evolution
• Storage growth
```

---

## 9. Interview Questions & Answers

### Q1: How do you handle a distributed transaction across microservices?

**Answer:**
- **Avoid if possible** — redesign boundaries so transactions are local.
- **Saga pattern:** Sequence of local transactions with compensating actions for rollback.
- **Choreography:** Services emit events, others react. Simple but hard to debug.
- **Orchestration:** Central coordinator manages the flow. More control, easier to trace.
- At Accenture, we used orchestrated Saga for order creation: reserve inventory → charge payment → confirm order, with compensation (refund, restock) on failure.

### Q2: When would you use a circuit breaker?

**Answer:**
- When calling an **external/downstream service** that may be slow or unavailable.
- Prevents cascade failures — if payment service is down, don't keep sending requests (which would exhaust connection pool and make your service slow too).
- Settings: trip after 50% failure rate in 10 requests, wait 30s before retrying.
- Always pair with a **fallback** (queue for later processing, cached response, degraded mode).

### Q3: How do you debug a request that spans 5 microservices?

**Answer:**
- **Distributed tracing:** Each request gets a unique `correlationId` propagated via headers.
- All services log with the same `correlationId` → query CloudWatch/ELK with that ID to see the full journey.
- Tools: AWS X-Ray, Jaeger, Datadog APM.
- Key metrics per span: latency, status code, which service, which operation.

### Q4: Choreography vs Orchestration Saga?

**Answer:**
- **Choreography:** Services are more decoupled, no central point of failure, but flow is implicit (hard to understand from code). Good for simple flows (2-3 services).
- **Orchestration:** Single place defines the flow, easier to debug and modify, but orchestrator is a potential bottleneck. Good for complex flows (4+ services).
- We use choreography for simple events (user created → send welcome email) and orchestration for complex business transactions (order creation saga).

---

**Previous:** [05 — Message Queues & Async ←](05-message-queues-async.md) | **Next:** [07 — API Design & Security →](07-api-design-security.md)
