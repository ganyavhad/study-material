# 07 — API Design & Security

> Design robust APIs and protect them with industry-standard security patterns.

---

## Table of Contents

1. [REST API Design Principles](#1-rest-api-design-principles)
2. [GraphQL Design Patterns](#2-graphql-design-patterns)
3. [REST vs GraphQL Decision](#3-rest-vs-graphql-decision)
4. [API Versioning](#4-api-versioning)
5. [Authentication & Authorization](#5-authentication--authorization)
6. [Rate Limiting & Throttling](#6-rate-limiting--throttling)
7. [CORS & Security Headers](#7-cors--security-headers)
8. [API Pagination](#8-api-pagination)
9. [Idempotency in APIs](#9-idempotency-in-apis)
10. [Interview Questions & Answers](#10-interview-questions--answers)

---

## 1. REST API Design Principles

```
Resource Naming:
  ✅ GET    /users              — list users
  ✅ GET    /users/:id          — get one user
  ✅ POST   /users              — create user
  ✅ PATCH  /users/:id          — partial update
  ✅ PUT    /users/:id          — full replace
  ✅ DELETE /users/:id          — delete user

  ✅ GET    /users/:id/orders   — nested resource
  ✅ POST   /users/:id/orders   — create order for user

  ❌ GET    /getUsers           — verb in URL
  ❌ POST   /createUser         — RPC style
  ❌ GET    /user               — singular for collection

HTTP Status Codes:
  200 OK           — success, data returned
  201 Created      — resource created (POST)
  204 No Content   — success, no body (DELETE)
  400 Bad Request  — validation error (client's fault)
  401 Unauthorized — missing/invalid auth token
  403 Forbidden    — valid token, insufficient permission
  404 Not Found    — resource doesn't exist
  409 Conflict     — duplicate, optimistic lock conflict
  422 Unprocessable — semantically invalid (valid JSON, bad logic)
  429 Too Many     — rate limited
  500 Server Error — unexpected failure
```

```typescript
// NestJS — well-structured REST controller
@Controller('users')
@UseGuards(JwtAuthGuard)
export class UsersController {
  @Get()
  @HttpCode(200)
  async findAll(@Query() query: PaginationDto) {
    return this.usersService.findAll(query);
  }

  @Get(':id')
  async findOne(@Param('id', ParseUUIDPipe) id: string) {
    const user = await this.usersService.findOne(id);
    if (!user) throw new NotFoundException(`User ${id} not found`);
    return user;
  }

  @Post()
  @HttpCode(201)
  async create(@Body(ValidationPipe) dto: CreateUserDto) {
    return this.usersService.create(dto);
  }

  @Patch(':id')
  async update(
    @Param('id', ParseUUIDPipe) id: string,
    @Body(ValidationPipe) dto: UpdateUserDto,
  ) {
    return this.usersService.update(id, dto);
  }

  @Delete(':id')
  @HttpCode(204)
  async remove(@Param('id', ParseUUIDPipe) id: string) {
    await this.usersService.remove(id);
  }
}

// Consistent error response format
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": ["email must be a valid email", "name should not be empty"],
  "timestamp": "2024-01-15T10:30:00.000Z",
  "path": "/api/users"
}
```

---

## 2. GraphQL Design Patterns

```typescript
// Schema-first approach — types define the contract
@ObjectType()
export class Course {
  @Field(() => ID)
  id: string;

  @Field()
  title: string;

  @Field(() => [Module])
  modules: Module[];

  @Field(() => Int)
  enrollmentCount: number;
}

// Resolver with DataLoader to prevent N+1
@Resolver(() => Course)
export class CourseResolver {
  @Query(() => [Course])
  async courses(
    @Args('filter', { nullable: true }) filter?: CourseFilterInput,
    @Args('pagination', { nullable: true }) pagination?: PaginationInput,
  ) {
    return this.courseService.findAll(filter, pagination);
  }

  // Field resolver — only called if client requests 'modules'
  @ResolveField()
  async modules(
    @Parent() course: Course,
    @Context('modulesLoader') loader: DataLoader<string, Module[]>,
  ) {
    return loader.load(course.id); // Batched query, no N+1
  }
}

// DataLoader setup — batches individual loads into one query
@Injectable()
export class ModulesLoader {
  createLoader(): DataLoader<string, Module[]> {
    return new DataLoader(async (courseIds: string[]) => {
      const modules = await this.moduleRepo.find({
        where: { courseId: In(courseIds as string[]) },
      });
      // Map modules back to courseIds maintaining order
      return courseIds.map(id => modules.filter(m => m.courseId === id));
    });
  }
}
```

### Query Complexity & Depth Limiting

```typescript
// Prevent abusive queries
GraphQLModule.forRoot({
  validationRules: [
    depthLimit(7),                    // Max 7 levels deep
    createComplexityRule({
      maximumComplexity: 1000,        // Max complexity score
      estimators: [
        fieldExtensionsEstimator(),
        simpleEstimator({ defaultComplexity: 1 }),
      ],
    }),
  ],
});
```

---

## 3. REST vs GraphQL Decision

```
┌──────────────────┬──────────────────────┬──────────────────────┐
│ Factor           │ REST                 │ GraphQL              │
├──────────────────┼──────────────────────┼──────────────────────┤
│ Data shape       │ Server-defined       │ Client-defined       │
│ Over-fetching    │ Common               │ Eliminated           │
│ Under-fetching   │ Multiple round trips │ Single query         │
│ Caching          │ HTTP caching easy    │ Needs client cache   │
│ File uploads     │ Native support       │ Needs multipart spec │
│ Real-time        │ Polling / SSE        │ Subscriptions        │
│ Learning curve   │ Low                  │ Medium               │
│ Versioning       │ URL / Header         │ Schema evolution     │
│ Tooling          │ Swagger/OpenAPI      │ Playground/Codegen   │
│ Best for         │ CRUD, public APIs    │ Complex UIs, BFF     │
└──────────────────┴──────────────────────┴──────────────────────┘

Example: GraphQL for LMS frontend (course → modules → content
→ user progress — one query instead of 5 REST calls).
REST for webhooks, file upload, simple CRUD admin APIs.
```

---

## 4. API Versioning

```
Strategy 1: URL Path (most common)
  /api/v1/users
  /api/v2/users

Strategy 2: Header
  Accept: application/vnd.myapi.v2+json

Strategy 3: Query Parameter
  /api/users?version=2

GraphQL: No versioning — evolve schema
  @deprecated(reason: "Use fullName instead")
```

```typescript
// NestJS versioning
app.enableVersioning({ type: VersioningType.URI });

@Controller({ path: 'users', version: '1' })
export class UsersV1Controller {
  @Get()
  findAll() { /* v1 response shape */ }
}

@Controller({ path: 'users', version: '2' })
export class UsersV2Controller {
  @Get()
  findAll() { /* v2 response shape with breaking changes */ }
}
```

---

## 5. Authentication & Authorization

```
Authentication Flow (JWT):
  Client              API Gateway        Auth Service         Resource API
    │                     │                   │                    │
    │──── POST /login ──►│──── validate ───►│                    │
    │                     │                   │── verify creds ──│
    │                     │   ◄── JWT token ──│                    │
    │◄── JWT token ───────│                   │                    │
    │                     │                   │                    │
    │── GET /orders ─────►│                   │                    │
    │   (Bearer token)    │── verify JWT ────►│                    │
    │                     │◄─ valid ──────────│                    │
    │                     │─────────────── forward ──────────────►│
    │◄── orders ──────────│◄──────────────────────── response ────│

JWT Token Structure:
  Header:  { "alg": "RS256", "typ": "JWT" }
  Payload: { "sub": "user-123", "roles": ["admin"], "iat": ..., "exp": ... }
  Signature: RS256(header + payload, privateKey)
```

```typescript
// JWT Auth Guard (NestJS Passport)
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(private configService: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: configService.get('JWT_PUBLIC_KEY'),
      algorithms: ['RS256'],  // Use asymmetric — API only needs public key
    });
  }

  async validate(payload: JwtPayload) {
    return { userId: payload.sub, roles: payload.roles };
  }
}

// Role-based authorization guard
@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<string[]>('roles', context.getHandler());
    if (!requiredRoles) return true;

    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return requiredRoles.some(role => user.roles.includes(role));
  }
}

// Usage with decorator
@Post()
@Roles('admin', 'manager')
@UseGuards(JwtAuthGuard, RolesGuard)
async createCourse(@Body() dto: CreateCourseDto) { ... }
```

### OAuth 2.0 / OIDC Flow

```
Authorization Code Flow (for web apps):
  1. User clicks "Login with Google"
  2. Redirect to Google: /authorize?response_type=code&client_id=...
  3. User consents → Google redirects to callback with ?code=xyz
  4. Server exchanges code for access_token + id_token (server-to-server)
  5. Validate id_token, create session / issue own JWT

PKCE (for SPAs / mobile):
  Same flow but adds code_verifier / code_challenge
  to prevent authorization code interception
```

---

## 6. Rate Limiting & Throttling

```typescript
// NestJS built-in throttler
@Module({
  imports: [
    ThrottlerModule.forRoot({
      throttlers: [
        { name: 'short', ttl: 1000, limit: 3 },    // 3 per second
        { name: 'medium', ttl: 10000, limit: 20 },  // 20 per 10 seconds
        { name: 'long', ttl: 60000, limit: 100 },   // 100 per minute
      ],
    }),
  ],
})
export class AppModule {}

// Endpoint-specific override
@Throttle({ short: { ttl: 1000, limit: 1 } })  // 1 per second for login
@Post('login')
async login(@Body() dto: LoginDto) { ... }

// Skip throttle for health checks
@SkipThrottle()
@Get('health')
healthCheck() { return { status: 'ok' }; }

// Redis-based distributed rate limiting (for multi-instance)
// Uses sliding window counter in Redis
@Injectable()
export class RedisThrottlerStorage implements ThrottlerStorage {
  async increment(key: string, ttl: number): Promise<ThrottlerStorageRecord> {
    const multi = this.redis.multi();
    multi.incr(key);
    multi.pttl(key);
    const [count, currentTtl] = await multi.exec();
    if (currentTtl === -1) await this.redis.pexpire(key, ttl);
    return { totalHits: count as number, timeToExpire: ttl };
  }
}
```

---

## 7. CORS & Security Headers

```typescript
// NestJS CORS configuration
app.enableCors({
  origin: ['https://app.example.com', 'https://admin.example.com'],
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Correlation-Id'],
  credentials: true,
  maxAge: 86400,  // Preflight cache 24h
});

// Security headers with Helmet
import helmet from 'helmet';
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
    },
  },
  hsts: { maxAge: 31536000, includeSubDomains: true },
}));

// Input validation — ALWAYS validate at system boundary
export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(2)
  @MaxLength(50)
  @Matches(/^[a-zA-Z\s]+$/)   // Prevent injection via name field
  name: string;

  @IsEnum(UserRole)
  role: UserRole;
}
```

---

## 8. API Pagination

```typescript
// Offset-based pagination (simple, common)
// GET /users?page=2&limit=20
@Get()
async findAll(@Query() query: PaginationDto) {
  const [data, total] = await this.userRepo.findAndCount({
    skip: (query.page - 1) * query.limit,
    take: query.limit,
    order: { createdAt: 'DESC' },
  });

  return {
    data,
    meta: {
      page: query.page,
      limit: query.limit,
      total,
      totalPages: Math.ceil(total / query.limit),
    },
  };
}

// Cursor-based pagination (better for large datasets / real-time feeds)
// GET /messages?cursor=abc123&limit=20
@Get()
async findAll(@Query() query: CursorPaginationDto) {
  const queryBuilder = this.messageRepo.createQueryBuilder('m');

  if (query.cursor) {
    const cursorDate = decodeCursor(query.cursor); // base64 → timestamp
    queryBuilder.where('m.createdAt < :cursor', { cursor: cursorDate });
  }

  const data = await queryBuilder
    .orderBy('m.createdAt', 'DESC')
    .take(query.limit + 1) // Fetch one extra to check hasMore
    .getMany();

  const hasMore = data.length > query.limit;
  if (hasMore) data.pop();

  return {
    data,
    meta: {
      hasMore,
      nextCursor: hasMore ? encodeCursor(data[data.length - 1].createdAt) : null,
    },
  };
}

// Offset vs Cursor:
// Offset: Easy, supports "jump to page 5", but slow on large tables (OFFSET scans rows)
// Cursor: Consistent performance, no skipped/duplicate items on insert, but no page jumping
```

---

## 9. Idempotency in APIs

```typescript
// Idempotency key middleware — prevent duplicate operations
@Injectable()
export class IdempotencyMiddleware implements NestMiddleware {
  constructor(private redis: Redis) {}

  async use(req: Request, res: Response, next: NextFunction) {
    if (req.method !== 'POST' && req.method !== 'PATCH') return next();

    const idempotencyKey = req.headers['idempotency-key'] as string;
    if (!idempotencyKey) return next();

    const cached = await this.redis.get(`idempotency:${idempotencyKey}`);
    if (cached) {
      const response = JSON.parse(cached);
      return res.status(response.status).json(response.body);
    }

    // Intercept response to cache it
    const originalJson = res.json.bind(res);
    res.json = (body: any) => {
      this.redis.setex(
        `idempotency:${idempotencyKey}`,
        86400, // 24h TTL
        JSON.stringify({ status: res.statusCode, body }),
      );
      return originalJson(body);
    };

    next();
  }
}
```

---

## 10. Interview Questions & Answers

### Q1: REST vs GraphQL — when do you use each?

**Answer:**
- **REST** for simple CRUD, public APIs (easy caching, well-understood), webhook integrations, file uploads.
- **GraphQL** when the frontend needs flexible queries (dashboard with multiple widgets), needs to reduce over-fetching (mobile), or when you have deeply nested related data.
- Use GraphQL for the LMS frontend (course → modules → lessons → progress in one query) and REST for admin CRUD and third-party integrations.

### Q2: How do you secure an API?

**Answer:**
- **Auth:** JWT with RS256 (asymmetric), short-lived access tokens + refresh tokens.
- **Input validation:** class-validator on every DTO at the controller boundary.
- **Rate limiting:** ThrottlerModule + Redis for distributed environments.
- **Headers:** Helmet for CSP, HSTS, X-Frame-Options.
- **CORS:** Whitelist specific origins, never `*` in production with credentials.
- **Parameterized queries:** TypeORM/Knex handle this by default.
- **Secrets:** AWS Secrets Manager / SSM, never in env files or code.

### Q3: How do you handle API versioning without breaking existing clients?

**Answer:**
- **Additive changes** (new fields, new endpoints) don't need a new version.
- **Breaking changes** (removed fields, changed types) → new version.
- Strategy: URL path versioning (`/v1/`, `/v2/`), maintain both for a deprecation period.
- In GraphQL: use `@deprecated` directive, no versions needed — clients request only what they need.

### Q4: Offset vs cursor pagination?

**Answer:**
- **Offset:** Simple, supports "go to page 5", but suffers from performance issues on large tables (DB still scans skipped rows) and data shifts (new inserts cause duplicates/skips).
- **Cursor:** Consistent O(1) performance using indexed column, no data shifts, but no random page access.
- Use cursor for feeds, chat messages, real-time data. Use offset for admin tables, search results with page navigation.

---

**Previous:** [06 — Microservices & Distributed Patterns ←](06-microservices-distributed-patterns.md) | **Next:** [08 — Real-World System Designs →](08-real-world-system-designs.md)
