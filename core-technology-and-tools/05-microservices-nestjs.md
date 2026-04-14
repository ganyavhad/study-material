# Microservices Architecture & NestJS - Study Material

> Based on skills from [ganeshavhad.com](https://ganeshavhad.com) — Software Developer Specialist
> *"Migrated Java Spring Boot services to Node.js, leveraging NestJS and TypeORM"*

---

## 1. Microservices Architecture

### Monolith vs Microservices

```
MONOLITH                          MICROSERVICES
┌──────────────────┐              ┌──────┐ ┌──────┐ ┌──────┐
│   ┌────────────┐ │              │ User │ │Order │ │Notif │
│   │    UI      │ │              │ Svc  │ │ Svc  │ │ Svc  │
│   ├────────────┤ │              └──┬───┘ └──┬───┘ └──┬───┘
│   │ Business   │ │                 │        │        │
│   │   Logic    │ │              ┌──┴───┐ ┌──┴───┐ ┌──┴───┐
│   ├────────────┤ │              │  DB  │ │  DB  │ │  DB  │
│   │  Database  │ │              └──────┘ └──────┘ └──────┘
│   └────────────┘ │
└──────────────────┘              Each service: own DB, own deploy
```

### Key Principles
| Principle | Description |
|-----------|-------------|
| **Single Responsibility** | Each service does one thing well |
| **Autonomous** | Independent deployment and scaling |
| **Decentralized Data** | Each service owns its database |
| **Fault Isolation** | Failure in one service doesn't cascade |
| **API-First** | Services communicate via well-defined APIs |
| **Observable** | Centralized logging, metrics, tracing |

### Communication Patterns

| Pattern | Type | Use Case | Example |
|---------|------|----------|---------|
| REST/HTTP | Synchronous | Simple request-response | GET /users/123 |
| gRPC | Synchronous | High-performance, internal | Proto-based calls |
| Message Queue | Asynchronous | Fire-and-forget, decoupling | SQS, RabbitMQ |
| Event Bus | Asynchronous | Event-driven architecture | EventBridge, Kafka |
| GraphQL | Synchronous | Client-driven data needs | Apollo Federation |

---

## 2. NestJS Framework

NestJS is a **progressive Node.js framework** inspired by Angular. Built with TypeScript, uses decorators, dependency injection, and modular architecture.

### Project Structure
```
src/
├── app.module.ts              # Root module
├── main.ts                    # Entry point
├── common/
│   ├── filters/               # Exception filters
│   ├── guards/                # Auth guards
│   ├── interceptors/          # Request/response interceptors
│   ├── pipes/                 # Validation pipes
│   └── decorators/            # Custom decorators
├── users/
│   ├── users.module.ts        # Feature module
│   ├── users.controller.ts    # HTTP endpoints
│   ├── users.service.ts       # Business logic
│   ├── users.repository.ts    # Data access
│   ├── dto/
│   │   ├── create-user.dto.ts
│   │   └── update-user.dto.ts
│   └── entities/
│       └── user.entity.ts     # TypeORM entity
├── orders/
│   ├── orders.module.ts
│   ├── orders.controller.ts
│   └── orders.service.ts
└── config/
    ├── database.config.ts
    └── app.config.ts
```

### Module

```typescript
// users.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { User } from './entities/user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService] // Available to other modules
})
export class UsersModule {}
```

### Controller

```typescript
// users.controller.ts
import {
  Controller, Get, Post, Put, Delete,
  Body, Param, Query, HttpCode, HttpStatus,
  UseGuards, ParseUUIDPipe
} from '@nestjs/common';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';
import { JwtAuthGuard } from '../common/guards/jwt-auth.guard';

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  async findAll(
    @Query('page') page = 1,
    @Query('limit') limit = 10
  ) {
    return this.usersService.findAll(page, limit);
  }

  @Get(':id')
  async findOne(@Param('id', ParseUUIDPipe) id: string) {
    return this.usersService.findOne(id);
  }

  @Post()
  @HttpCode(HttpStatus.CREATED)
  async create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }

  @Put(':id')
  @UseGuards(JwtAuthGuard)
  async update(
    @Param('id', ParseUUIDPipe) id: string,
    @Body() updateUserDto: UpdateUserDto
  ) {
    return this.usersService.update(id, updateUserDto);
  }

  @Delete(':id')
  @UseGuards(JwtAuthGuard)
  @HttpCode(HttpStatus.NO_CONTENT)
  async remove(@Param('id', ParseUUIDPipe) id: string) {
    return this.usersService.remove(id);
  }
}
```

### Service

```typescript
// users.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './entities/user.entity';
import { CreateUserDto } from './dto/create-user.dto';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>
  ) {}

  async findAll(page: number, limit: number) {
    const [users, total] = await this.userRepository.findAndCount({
      skip: (page - 1) * limit,
      take: limit,
      order: { createdAt: 'DESC' }
    });
    return { data: users, total, page, lastPage: Math.ceil(total / limit) };
  }

  async findOne(id: string): Promise<User> {
    const user = await this.userRepository.findOne({ where: { id } });
    if (!user) {
      throw new NotFoundException(`User with ID ${id} not found`);
    }
    return user;
  }

  async create(dto: CreateUserDto): Promise<User> {
    const user = this.userRepository.create(dto);
    return this.userRepository.save(user);
  }

  async update(id: string, dto: Partial<CreateUserDto>): Promise<User> {
    await this.findOne(id); // Throws if not found
    await this.userRepository.update(id, dto);
    return this.findOne(id);
  }

  async remove(id: string): Promise<void> {
    const result = await this.userRepository.delete(id);
    if (result.affected === 0) {
      throw new NotFoundException(`User with ID ${id} not found`);
    }
  }
}
```

---

## 3. DTOs & Validation

```typescript
// dto/create-user.dto.ts
import { IsEmail, IsString, IsOptional, IsInt, Min, Max, MinLength } from 'class-validator';
import { Transform } from 'class-transformer';

export class CreateUserDto {
  @IsString()
  @MinLength(2)
  @Transform(({ value }) => value?.trim())
  name: string;

  @IsEmail()
  @Transform(({ value }) => value?.toLowerCase().trim())
  email: string;

  @IsOptional()
  @IsInt()
  @Min(18)
  @Max(120)
  age?: number;
}

// dto/update-user.dto.ts
import { PartialType } from '@nestjs/mapped-types';
import { CreateUserDto } from './create-user.dto';

export class UpdateUserDto extends PartialType(CreateUserDto) {}
```

### Global Validation Pipe
```typescript
// main.ts
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,        // Strip unknown properties
    forbidNonWhitelisted: true,  // Throw on unknown properties
    transform: true,        // Auto-transform types
    transformOptions: { enableImplicitConversion: true }
  }));

  await app.listen(3000);
}
```

---

## 4. TypeORM Entities & Relationships

```typescript
// entities/user.entity.ts
import {
  Entity, PrimaryGeneratedColumn, Column,
  CreateDateColumn, UpdateDateColumn, OneToMany
} from 'typeorm';
import { Post } from '../../posts/entities/post.entity';

@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ length: 100 })
  name: string;

  @Column({ unique: true })
  email: string;

  @Column({ nullable: true })
  age: number;

  @Column({ default: true })
  isActive: boolean;

  @OneToMany(() => Post, (post) => post.author)
  posts: Post[];

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}

// entities/post.entity.ts
@Entity('posts')
export class Post {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  title: string;

  @Column('text')
  content: string;

  @ManyToOne(() => User, (user) => user.posts)
  @JoinColumn({ name: 'author_id' })
  author: User;

  @Column()
  author_id: string;
}
```

### TypeORM Database Config
```typescript
// config/database.config.ts
import { TypeOrmModuleOptions } from '@nestjs/typeorm';

export const databaseConfig: TypeOrmModuleOptions = {
  type: 'postgres',
  host: process.env.DB_HOST || 'localhost',
  port: parseInt(process.env.DB_PORT || '5432', 10),
  username: process.env.DB_USERNAME,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  entities: [__dirname + '/../**/*.entity{.ts,.js}'],
  migrations: [__dirname + '/../migrations/*{.ts,.js}'],
  synchronize: false, // NEVER true in production
  logging: process.env.NODE_ENV === 'development'
};
```

---

## 5. Guards, Interceptors & Filters

### JWT Auth Guard
```typescript
@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const token = request.headers.authorization?.replace('Bearer ', '');

    if (!token) throw new UnauthorizedException('Missing token');

    try {
      const payload = this.jwtService.verify(token);
      request.user = payload;
      return true;
    } catch {
      throw new UnauthorizedException('Invalid token');
    }
  }
}
```

### Logging Interceptor
```typescript
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  private readonly logger = new Logger(LoggingInterceptor.name);

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const { method, url } = request;
    const start = Date.now();

    return next.handle().pipe(
      tap(() => {
        const duration = Date.now() - start;
        this.logger.log(`${method} ${url} - ${duration}ms`);
      })
    );
  }
}
```

### Global Exception Filter
```typescript
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger(AllExceptionsFilter.name);

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();

    const status = exception instanceof HttpException
      ? exception.getStatus()
      : HttpStatus.INTERNAL_SERVER_ERROR;

    const message = exception instanceof HttpException
      ? exception.getResponse()
      : 'Internal server error';

    this.logger.error(`${request.method} ${request.url}`, exception);

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message
    });
  }
}
```

---

## 6. Microservices with NestJS

### TCP Transport
```typescript
// Microservice (server)
async function bootstrap() {
  const app = await NestFactory.createMicroservice(AppModule, {
    transport: Transport.TCP,
    options: { host: '0.0.0.0', port: 3001 }
  });
  await app.listen();
}

// Message patterns
@Controller()
export class UsersMicroController {
  @MessagePattern({ cmd: 'get_user' })
  async getUser(@Payload() data: { id: string }) {
    return this.usersService.findOne(data.id);
  }

  @EventPattern('user_created')
  async handleUserCreated(@Payload() data: User) {
    await this.notificationService.sendWelcome(data);
  }
}

// Client (API Gateway calling microservice)
@Module({
  imports: [
    ClientsModule.register([{
      name: 'USER_SERVICE',
      transport: Transport.TCP,
      options: { host: 'user-service', port: 3001 }
    }])
  ]
})
export class AppModule {}

@Controller('users')
export class UsersController {
  constructor(@Inject('USER_SERVICE') private client: ClientProxy) {}

  @Get(':id')
  async getUser(@Param('id') id: string) {
    return this.client.send({ cmd: 'get_user' }, { id });
  }
}
```

---

## 7. Design Patterns

### Circuit Breaker
```typescript
class CircuitBreaker {
  private failures = 0;
  private lastFailureTime: number;
  private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED';

  constructor(
    private threshold = 5,
    private timeout = 30000
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime > this.timeout) {
        this.state = 'HALF_OPEN';
      } else {
        throw new Error('Circuit breaker is OPEN');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess() {
    this.failures = 0;
    this.state = 'CLOSED';
  }

  private onFailure() {
    this.failures++;
    this.lastFailureTime = Date.now();
    if (this.failures >= this.threshold) {
      this.state = 'OPEN';
    }
  }
}
```

### Saga Pattern (for distributed transactions)
```
Order Saga:
1. Create Order (PENDING)
2. Reserve Inventory → Success? Continue : Compensate(Cancel Order)
3. Process Payment  → Success? Continue : Compensate(Release Inventory, Cancel Order)
4. Ship Order       → Success? Complete  : Compensate(Refund, Release Inventory, Cancel Order)
```

### CQRS (Command Query Responsibility Segregation)
```
Commands (Write)              Queries (Read)
    │                             │
    ▼                             ▼
┌─────────┐   Events    ┌──────────────┐
│ Write DB │──────────►  │  Read Model  │
│ (Source  │             │ (Optimized   │
│ of Truth)│             │  for queries)│
└─────────┘              └──────────────┘
```

---

## 8. Spring Boot to NestJS Migration

> Direct mapping of concepts:

| Spring Boot | NestJS |
|-------------|--------|
| `@RestController` | `@Controller` |
| `@Service` | `@Injectable` |
| `@Autowired` | Constructor injection |
| `@Entity` | `@Entity` (TypeORM) |
| `@Repository` | `@InjectRepository` |
| `application.properties` | `ConfigModule` / `.env` |
| `@Valid @RequestBody` | `ValidationPipe` + DTOs |
| `@ExceptionHandler` | `@Catch` exception filters |
| Spring Security | Guards + Passport |
| Spring Data JPA | TypeORM |

---

## 9. Interview Questions

1. **What is NestJS?** — Progressive Node.js framework with TypeScript, inspired by Angular. Uses decorators, DI, and modular architecture.
2. **Explain the request lifecycle** — Middleware → Guards → Interceptors (before) → Pipes → Controller → Service → Interceptors (after) → Exception Filters
3. **How does DI work in NestJS?** — Token-based IoC container. `@Injectable()` marks providers, `@Inject()` resolves by token.
4. **What is the Saga pattern?** — Manages distributed transactions via compensating actions when a step fails.
5. **Circuit Breaker vs Retry?** — Retry: try again N times. Circuit Breaker: stop trying after threshold, wait before retrying.
6. **What is event sourcing?** — Store all state changes as events. Current state = replay of events.
7. **How to handle distributed tracing?** — Propagate correlation IDs. Use tools like Jaeger, Zipkin, or AWS X-Ray.
8. **Service discovery approaches?** — Client-side (Consul), Server-side (AWS ALB), DNS-based (Kubernetes services).

---

## 10. Practice Exercises

1. Build a NestJS CRUD API with TypeORM + PostgreSQL
2. Implement JWT authentication with Guards
3. Create two microservices communicating via TCP/RabbitMQ
4. Migrate a simple Spring Boot REST API to NestJS
5. Implement the Circuit Breaker pattern
6. Add Swagger documentation with `@nestjs/swagger`
