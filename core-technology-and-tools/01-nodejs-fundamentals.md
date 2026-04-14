# Node.js Fundamentals - Study Material

> Based on skills from [ganeshavhad.com](https://ganeshavhad.com) — Software Developer Specialist

---

## 1. What is Node.js?

Node.js is a **JavaScript runtime** built on Chrome's V8 engine. It enables server-side JavaScript execution with an **event-driven, non-blocking I/O model**.

### Key Characteristics
- **Single-threaded** with event loop
- **Non-blocking I/O** — handles thousands of concurrent connections
- **NPM** — world's largest package ecosystem
- **Cross-platform** — runs on Windows, Linux, macOS

---

## 2. Core Concepts

### 2.1 Event Loop

```
   ┌───────────────────────────┐
┌─>│           timers          │  (setTimeout, setInterval)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  (I/O callbacks)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  (retrieve new I/O events)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  (setImmediate)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  (socket.on('close'))
   └───────────────────────────┘
```

**Phases Explained:**
1. **Timers** — executes callbacks from `setTimeout()` and `setInterval()`
2. **Pending Callbacks** — executes I/O callbacks deferred to the next loop
3. **Poll** — retrieves new I/O events; executes I/O-related callbacks
4. **Check** — `setImmediate()` callbacks
5. **Close** — close event callbacks (e.g., `socket.on('close')`)

### 2.2 Modules System

```javascript
// CommonJS (traditional)
const express = require('express');
module.exports = { myFunction };

// ES Modules (modern)
import express from 'express';
export const myFunction = () => {};
```

### 2.3 Streams

```javascript
const fs = require('fs');
const readStream = fs.createReadStream('large-file.txt');
const writeStream = fs.createWriteStream('output.txt');

readStream.pipe(writeStream);

readStream.on('data', (chunk) => {
  console.log(`Received ${chunk.length} bytes`);
});

readStream.on('end', () => {
  console.log('Finished reading');
});
```

**Stream Types:**
| Type | Description | Example |
|------|-------------|---------|
| Readable | Data source | `fs.createReadStream()` |
| Writable | Data destination | `fs.createWriteStream()` |
| Duplex | Both read & write | TCP Socket |
| Transform | Modify data in transit | `zlib.createGzip()` |

---

## 3. Async Patterns

### 3.1 Callbacks (Legacy)
```javascript
fs.readFile('file.txt', (err, data) => {
  if (err) throw err;
  console.log(data);
});
```

### 3.2 Promises
```javascript
const readFile = (path) => {
  return new Promise((resolve, reject) => {
    fs.readFile(path, (err, data) => {
      if (err) reject(err);
      else resolve(data);
    });
  });
};

readFile('file.txt').then(data => console.log(data)).catch(console.error);
```

### 3.3 Async/Await (Modern)
```javascript
const { readFile } = require('fs/promises');

async function processFile() {
  try {
    const data = await readFile('file.txt', 'utf-8');
    console.log(data);
  } catch (err) {
    console.error('Error reading file:', err.message);
  }
}
```

---

## 4. Building REST APIs with Express

```javascript
const express = require('express');
const app = express();

// Middleware
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Routes
app.get('/api/users', async (req, res) => {
  try {
    const users = await User.find();
    res.json(users);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.post('/api/users', async (req, res) => {
  try {
    const user = await User.create(req.body);
    res.status(201).json(user);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// Error handling middleware
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: 'Something went wrong!' });
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

---

## 5. NPM Package Development

> *"Designed and developed reusable NPM packages, streamlining integration processes"*

### Creating a Reusable NPM Package

```bash
mkdir my-package && cd my-package
npm init -y
```

```json
// package.json
{
  "name": "@myorg/utils",
  "version": "1.0.0",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "test": "jest",
    "prepublishOnly": "npm run build && npm test"
  },
  "files": ["dist"]
}
```

```typescript
// src/index.ts
export function validateEmail(email: string): boolean {
  const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return regex.test(email);
}

export function formatCurrency(amount: number, currency = 'USD'): string {
  return new Intl.NumberFormat('en-US', { style: 'currency', currency }).format(amount);
}
```

---

## 6. Server-Side Queuing Mechanism

> *"Engineered a Robust Server-Side Queuing Mechanism from Scratch"*

### Using Bull Queue with Redis

```javascript
const Queue = require('bull');

// Create queue
const emailQueue = new Queue('email-processing', {
  redis: { host: '127.0.0.1', port: 6379 }
});

// Producer: Add jobs
async function sendEmail(to, subject, body) {
  await emailQueue.add({ to, subject, body }, {
    attempts: 3,
    backoff: { type: 'exponential', delay: 2000 },
    removeOnComplete: true
  });
}

// Consumer: Process jobs
emailQueue.process(async (job) => {
  const { to, subject, body } = job.data;
  await mailer.send({ to, subject, body });
  return { delivered: true };
});

// Event listeners
emailQueue.on('completed', (job, result) => {
  console.log(`Job ${job.id} completed:`, result);
});

emailQueue.on('failed', (job, err) => {
  console.error(`Job ${job.id} failed:`, err.message);
});
```

---

## 7. Performance Optimization

### Clustering
```javascript
const cluster = require('cluster');
const os = require('os');

if (cluster.isPrimary) {
  const numCPUs = os.cpus().length;
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }
  cluster.on('exit', (worker) => {
    console.log(`Worker ${worker.process.pid} died. Restarting...`);
    cluster.fork();
  });
} else {
  require('./server'); // Your Express app
}
```

### Caching with Redis
```javascript
const Redis = require('ioredis');
const redis = new Redis();

async function getCachedData(key, fetchFn, ttl = 3600) {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const data = await fetchFn();
  await redis.setex(key, ttl, JSON.stringify(data));
  return data;
}
```

---

## 8. Security Best Practices

```javascript
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');
const hpp = require('hpp');

const app = express();

// Security headers
app.use(helmet());

// Rate limiting
app.use(rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // limit each IP to 100 requests per window
}));

// Prevent HTTP parameter pollution
app.use(hpp());

// Input validation with Joi
const Joi = require('joi');
const userSchema = Joi.object({
  name: Joi.string().min(2).max(50).required(),
  email: Joi.string().email().required(),
  age: Joi.number().integer().min(18).max(120)
});
```

---

## 9. Interview Questions

1. **What is the event loop?** — Single-threaded mechanism that handles async operations via phases (timers, poll, check, etc.)
2. **Difference between `process.nextTick()` and `setImmediate()`?** — `nextTick` executes before the next event loop phase; `setImmediate` executes in the check phase
3. **How does Node.js handle child processes?** — Via `child_process` module (`spawn`, `exec`, `fork`)
4. **What are Worker Threads?** — True multi-threading for CPU-intensive tasks (Node.js 10.5+)
5. **Explain middleware in Express** — Functions that access `req`, `res`, and `next()` in the request-response cycle
6. **How to prevent memory leaks?** — Avoid global variables, close streams, use WeakMap/WeakRef, monitor with `--inspect`
7. **What is `libuv`?** — C library that provides the event loop and async I/O to Node.js
8. **Explain the difference between `require` and `import`** — `require` is CommonJS (sync), `import` is ESM (async, tree-shakeable)

---

## 10. Practice Exercises

1. Build a REST API with CRUD operations using Express + MongoDB
2. Create a custom NPM package with TypeScript
3. Implement a job queue using Bull/Redis
4. Build a real-time chat using WebSockets (Socket.io)
5. Create a CLI tool using Commander.js
6. Implement clustering for a CPU-intensive API
