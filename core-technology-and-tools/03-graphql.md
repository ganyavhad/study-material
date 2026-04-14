# GraphQL - Study Material

> Based on skills from [ganeshavhad.com](https://ganeshavhad.com) — Software Developer Specialist
> *"Developed and Deployed GraphQL APIs from Scratch, successfully migrating existing Node.js APIs to GraphQL"*

---

## 1. What is GraphQL?

GraphQL is a **query language for APIs** and a runtime for executing those queries. Created by Facebook (2012), open-sourced in 2015.

### GraphQL vs REST

| Feature | REST | GraphQL |
|---------|------|---------|
| Endpoints | Multiple (`/users`, `/posts`) | Single (`/graphql`) |
| Data Fetching | Over/under-fetching common | Client requests exactly what it needs |
| Versioning | URL-based (`/v1/`, `/v2/`) | Schema evolution, no versioning |
| Type System | None built-in | Strongly typed schema |
| Real-time | Polling / SSE | Subscriptions (WebSocket) |
| Caching | HTTP caching (easy) | Requires client-side caching |
| Error Handling | HTTP status codes | Always 200, errors in response body |

---

## 2. Schema Definition Language (SDL)

```graphql
# Type definitions
type User {
  id: ID!
  name: String!
  email: String!
  age: Int
  posts: [Post!]!
  role: Role!
  createdAt: DateTime!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  comments: [Comment!]!
  published: Boolean!
}

type Comment {
  id: ID!
  text: String!
  author: User!
  post: Post!
}

enum Role {
  ADMIN
  USER
  MODERATOR
}

# Input types for mutations
input CreateUserInput {
  name: String!
  email: String!
  age: Int
  role: Role = USER
}

input UpdateUserInput {
  name: String
  email: String
  age: Int
}

# Query & Mutation definitions
type Query {
  users(limit: Int, offset: Int): [User!]!
  user(id: ID!): User
  posts(published: Boolean): [Post!]!
  searchPosts(query: String!): [Post!]!
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
  createPost(title: String!, content: String!, authorId: ID!): Post!
}

type Subscription {
  postCreated: Post!
  commentAdded(postId: ID!): Comment!
}

scalar DateTime
```

---

## 3. Building a GraphQL Server (Node.js + Apollo)

### Setup
```bash
npm install @apollo/server graphql
npm install @apollo/subgraph  # for federation
```

### Server Implementation

```typescript
import { ApolloServer } from '@apollo/server';
import { startStandaloneServer } from '@apollo/server/standalone';
import { readFileSync } from 'fs';

// Type definitions
const typeDefs = readFileSync('./schema.graphql', 'utf-8');

// Resolvers
const resolvers = {
  Query: {
    users: async (_, { limit = 10, offset = 0 }, { dataSources }) => {
      return dataSources.userAPI.getUsers(limit, offset);
    },

    user: async (_, { id }, { dataSources }) => {
      return dataSources.userAPI.getUserById(id);
    },

    posts: async (_, { published }, { dataSources }) => {
      return dataSources.postAPI.getPosts({ published });
    }
  },

  Mutation: {
    createUser: async (_, { input }, { dataSources }) => {
      return dataSources.userAPI.createUser(input);
    },

    updateUser: async (_, { id, input }, { dataSources }) => {
      return dataSources.userAPI.updateUser(id, input);
    },

    deleteUser: async (_, { id }, { dataSources }) => {
      return dataSources.userAPI.deleteUser(id);
    }
  },

  // Field-level resolvers (relationships)
  User: {
    posts: async (parent, _, { dataSources }) => {
      return dataSources.postAPI.getPostsByAuthor(parent.id);
    }
  },

  Post: {
    author: async (parent, _, { dataSources }) => {
      return dataSources.userAPI.getUserById(parent.authorId);
    },
    comments: async (parent, _, { dataSources }) => {
      return dataSources.commentAPI.getCommentsByPost(parent.id);
    }
  }
};

// Server
const server = new ApolloServer({ typeDefs, resolvers });

const { url } = await startStandaloneServer(server, {
  listen: { port: 4000 },
  context: async ({ req }) => ({
    dataSources: {
      userAPI: new UserAPI(),
      postAPI: new PostAPI(),
      commentAPI: new CommentAPI()
    },
    user: await authenticateUser(req)
  })
});

console.log(`🚀 GraphQL server ready at ${url}`);
```

---

## 4. Data Sources (Database Integration)

```typescript
// data-sources/user-api.ts
import { Pool } from 'pg';

export class UserAPI {
  private pool: Pool;

  constructor() {
    this.pool = new Pool({ connectionString: process.env.DATABASE_URL });
  }

  async getUsers(limit: number, offset: number) {
    const result = await this.pool.query(
      'SELECT * FROM users ORDER BY created_at DESC LIMIT $1 OFFSET $2',
      [limit, offset]
    );
    return result.rows;
  }

  async getUserById(id: string) {
    const result = await this.pool.query(
      'SELECT * FROM users WHERE id = $1',
      [id]
    );
    return result.rows[0] || null;
  }

  async createUser(input: { name: string; email: string; age?: number }) {
    const result = await this.pool.query(
      'INSERT INTO users (name, email, age) VALUES ($1, $2, $3) RETURNING *',
      [input.name, input.email, input.age]
    );
    return result.rows[0];
  }

  async updateUser(id: string, input: Partial<{ name: string; email: string; age: number }>) {
    const fields = Object.entries(input)
      .filter(([, v]) => v !== undefined)
      .map(([k], i) => `${k} = $${i + 2}`);

    const values = Object.values(input).filter(v => v !== undefined);

    const result = await this.pool.query(
      `UPDATE users SET ${fields.join(', ')} WHERE id = $1 RETURNING *`,
      [id, ...values]
    );
    return result.rows[0];
  }
}
```

---

## 5. Solving the N+1 Problem with DataLoader

```typescript
import DataLoader from 'dataloader';

// Batch function
const batchUsers = async (userIds: readonly string[]) => {
  const users = await db.query(
    'SELECT * FROM users WHERE id = ANY($1)',
    [userIds]
  );
  // Return in same order as input IDs
  const userMap = new Map(users.rows.map(u => [u.id, u]));
  return userIds.map(id => userMap.get(id) || null);
};

// Create loader per request (in context)
const context = ({ req }) => ({
  loaders: {
    userLoader: new DataLoader(batchUsers),
    postLoader: new DataLoader(batchPosts)
  }
});

// Use in resolvers
const resolvers = {
  Post: {
    author: (parent, _, { loaders }) => {
      return loaders.userLoader.load(parent.authorId);
      // Batches all author lookups into ONE query
    }
  }
};
```

---

## 6. Authentication & Authorization

```typescript
// Auth middleware
async function authenticateUser(req) {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token) return null;

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    return await UserAPI.getUserById(decoded.userId);
  } catch {
    return null;
  }
}

// Auth directive / resolver-level auth
const resolvers = {
  Mutation: {
    deleteUser: async (_, { id }, { user }) => {
      if (!user) throw new GraphQLError('Not authenticated', {
        extensions: { code: 'UNAUTHENTICATED' }
      });
      if (user.role !== 'ADMIN') throw new GraphQLError('Not authorized', {
        extensions: { code: 'FORBIDDEN' }
      });
      return dataSources.userAPI.deleteUser(id);
    }
  }
};
```

---

## 7. Subscriptions (Real-time)

```typescript
import { PubSub } from 'graphql-subscriptions';
const pubsub = new PubSub();

const resolvers = {
  Mutation: {
    createPost: async (_, args, { dataSources }) => {
      const post = await dataSources.postAPI.create(args);
      pubsub.publish('POST_CREATED', { postCreated: post });
      return post;
    }
  },

  Subscription: {
    postCreated: {
      subscribe: () => pubsub.asyncIterator(['POST_CREATED'])
    },
    commentAdded: {
      subscribe: (_, { postId }) => {
        return pubsub.asyncIterator([`COMMENT_ADDED_${postId}`]);
      }
    }
  }
};
```

---

## 8. Migrating REST to GraphQL

> Strategy used: **Incremental Migration (Strangler Fig Pattern)**

### Step-by-Step Approach:
1. **Wrap REST endpoints** as GraphQL data sources
2. **Create schema** matching REST response shapes
3. **Add GraphQL endpoint** alongside REST
4. **Migrate clients** one query at a time
5. **Deprecate REST endpoints** gradually

```typescript
// Wrapping existing REST API
import { RESTDataSource } from '@apollo/datasource-rest';

class LegacyUserAPI extends RESTDataSource {
  override baseURL = 'http://localhost:3000/api/';

  async getUsers() {
    return this.get('users');
  }

  async getUserById(id: string) {
    return this.get(`users/${encodeURIComponent(id)}`);
  }

  async createUser(input: CreateUserInput) {
    return this.post('users', { body: input });
  }
}
```

---

## 9. Best Practices

### Query Complexity & Depth Limiting
```typescript
import depthLimit from 'graphql-depth-limit';
import { createComplexityLimitRule } from 'graphql-validation-complexity';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    depthLimit(7),
    createComplexityLimitRule(1000)
  ]
});
```

### Pagination
```graphql
type Query {
  # Cursor-based (recommended)
  users(first: Int!, after: String): UserConnection!
}

type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type UserEdge {
  cursor: String!
  node: User!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

---

## 10. Interview Questions

1. **What problem does GraphQL solve?** — Over-fetching and under-fetching of data. Client specifies exact data shape needed.
2. **What is a resolver?** — Function that populates data for a field in the schema. Takes `(parent, args, context, info)`.
3. **How to handle N+1 queries?** — Use DataLoader to batch and cache database lookups per request.
4. **Explain schema stitching vs federation** — Stitching: merge schemas at gateway. Federation: each service owns its part of the schema, composed at gateway.
5. **How do Subscriptions work?** — WebSocket-based. Server pushes updates via PubSub when events occur.
6. **What is introspection?** — Ability to query the schema itself. Should be disabled in production.
7. **How to version a GraphQL API?** — You don't. Use `@deprecated` directive and evolve the schema additively.
8. **What are custom scalars?** — User-defined types (DateTime, JSON, URL) with custom serialization/parsing logic.

---

## 11. Practice Exercises

1. Build a GraphQL API with queries, mutations, and subscriptions
2. Implement DataLoader for batching database queries
3. Migrate an existing REST API to GraphQL using the strangler pattern
4. Add cursor-based pagination to a GraphQL API
5. Implement field-level authorization
