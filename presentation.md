# Deploying Node.js Microservices

**From localhost to production-grade distributed systems**

Tags: `Node.js` | `Microservices` | `DevOps`

Pipeline: Design --> Build --> **Containerise** --> Deploy --> Monitor

*22 slides -- architecture -- containers -- orchestration -- observability*

---

## Table of Contents

1. [Topics We'll Cover](#slide-2-topics-well-cover)
2. [Monolith vs Microservices](#slide-3-monolith-vs-microservices)
3. [Anatomy of a Node.js Microservice](#slide-4-anatomy-of-a-nodejs-microservice)
4. [Structuring a Node.js Project for Microservices](#slide-5-structuring-a-nodejs-project-for-microservices)
5. [Health Checks & Readiness Probes](#slide-6-health-checks--readiness-probes)
6. [Graceful Shutdown](#slide-7-graceful-shutdown)
7. [Environment Configuration](#slide-8-environment-configuration)
8. [Containerising a Node.js Service](#slide-9-containerising-a-nodejs-service)
9. [Docker Compose for Local Development](#slide-10-docker-compose-for-local-development)
10. [Inter-Service Communication](#slide-11-inter-service-communication)
11. [Message Queues & Event-Driven Architecture](#slide-12-message-queues--event-driven-architecture)
12. [API Gateway Pattern](#slide-13-api-gateway-pattern)
13. [Service Discovery & Load Balancing](#slide-14-service-discovery--load-balancing)
14. [Authentication Between Services](#slide-15-authentication-between-services)
15. [Logging & Observability](#slide-16-logging--observability)
16. [Deploying to Kubernetes](#slide-17-deploying-to-kubernetes)
17. [Deploying to AWS (ECS/Fargate & Lambda)](#slide-18-deploying-to-aws-ecsfargate--lambda)
18. [CI/CD Pipeline for Microservices](#slide-19-cicd-pipeline-for-microservices)
19. [Database per Service](#slide-20-database-per-service)
20. [Common Pitfalls & Production Lessons](#slide-21-common-pitfalls--production-lessons)
21. [Summary & Further Reading](#slide-22-summary--further-reading)

---

## Slide 2: Topics We'll Cover

### Part 1 -- Architecture & Code

- Monolith vs Microservices
- Anatomy of a Node.js microservice
- Project structure (mono/polyrepo)
- Health checks & readiness probes
- Graceful shutdown patterns
- Environment configuration

### Part 2 -- Containers & Networking

- Production Dockerfiles (multi-stage)
- Docker Compose for local dev
- Inter-service communication
- Message queues & event-driven arch
- API Gateway pattern
- Service discovery & load balancing

### Part 3 -- Security & Observability

- Service-to-service authentication
- Logging & correlation IDs
- Database-per-service pattern

### Part 4 -- Deployment & Operations

- Deploying to Kubernetes
- Deploying to AWS (ECS/Fargate/Lambda)
- CI/CD pipelines for microservices
- Common pitfalls & production lessons

---

## Slide 3: Monolith vs Microservices

### When to Stay Monolithic

- Small team (< 5 developers)
- Unclear domain boundaries
- Rapid prototyping / MVP phase
- Low traffic, simple scaling needs

**Rule:** Start monolithic, extract when pain points emerge

### When to Split

- Independent deploy cadences needed
- Different scaling profiles per domain
- Multiple teams owning distinct features
- Technology diversity requirements

**Goal:** Organisational autonomy + independent deployability

### Architecture Diagram

```
MONOLITH                          MICROSERVICES
+-------------------+             +----------------+   +----------------+
| Users Module      |             | user-service   |   | Message Bus    |
| Orders Module     |   ---->     |   PostgreSQL   |   | RabbitMQ/NATS  |
| Payments Module   |             +----------------+   +----------------+
| Shared Database   |             | order-service  |   +----------------+
+-------------------+             |   MongoDB      |   | API Gateway    |
                                  +----------------+   +----------------+
                                  | payment-service|
                                  |   PostgreSQL   |
                                  +----------------+
```

---

## Slide 4: Anatomy of a Node.js Microservice

```typescript
// src/server.ts — Entry point
import Fastify from 'fastify';
import { healthRoutes } from './routes/health.js';
import { userRoutes } from './routes/users.js';
import { gracefulShutdown } from './lifecycle.js';
import { config } from './config.js';

const app = Fastify({
  logger: {
    level: config.LOG_LEVEL,
    transport: config.NODE_ENV === 'development'
      ? { target: 'pino-pretty' }
      : undefined,
  },
});

// Register routes
app.register(healthRoutes, { prefix: '/' });
app.register(userRoutes,   { prefix: '/api/users' });

// Start
const start = async () => {
  await app.listen({
    port: config.PORT,
    host: '0.0.0.0',    // bind all interfaces in Docker
  });
  app.log.info(`Service running on :${config.PORT}`);
};

start();
gracefulShutdown(app);
```

### Key Principles

- **Single responsibility** -- one bounded context per service
- **Bind to 0.0.0.0** -- required for container networking
- **Structured logging** -- JSON via pino (built into Fastify)
- **Health endpoints** -- K8s liveness & readiness probes
- **Graceful shutdown** -- drain in-flight requests on SIGTERM

### Fastify vs Express

- Fastify: **~77k req/s**, schema validation, built-in logging
- Express: **~15k req/s**, massive ecosystem, simpler mental model
- Both work -- Fastify is preferred for high-throughput microservices

---

## Slide 5: Structuring a Node.js Project for Microservices

### Monorepo (Recommended for Most Teams)

```bash
platform/
├── packages/
│   ├── shared-types/        # Shared TS interfaces
│   ├── logger/              # Shared pino config
│   └── auth-middleware/     # JWT validation
├── services/
│   ├── user-service/
│   │   ├── src/
│   │   │   ├── server.ts
│   │   │   ├── config.ts
│   │   │   ├── routes/
│   │   │   ├── services/
│   │   │   └── repositories/
│   │   ├── Dockerfile
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── order-service/
│   └── payment-service/
├── docker-compose.yml
├── turbo.json               # Turborepo config
└── package.json             # Workspace root
```

### Monorepo Pros

- Atomic cross-service changes in one PR
- Shared packages without npm publishing
- Unified CI/CD and tooling config
- Easier refactoring across boundaries

### Polyrepo Pros

- Complete team autonomy
- Independent versioning & release cycles
- Smaller clone / CI scope per service
- Better for large orgs with distinct ownership

### Tooling

- **Turborepo** -- fast, caches build artifacts
- **Nx** -- more features, steeper learning curve
- **pnpm workspaces** -- lightweight, fast installs

---

## Slide 6: Health Checks & Readiness Probes

```typescript
// src/routes/health.ts
import { FastifyInstance } from 'fastify';
import { pool } from '../db.js';
import { redis } from '../redis.js';

export async function healthRoutes(app: FastifyInstance) {
  // Liveness — is the process alive?
  app.get('/health', async (_req, reply) => {
    return reply.send({ status: 'ok', uptime: process.uptime() });
  });

  // Readiness — can it serve traffic?
  app.get('/ready', async (_req, reply) => {
    const checks: Record<string, string> = {};

    try {
      await pool.query('SELECT 1');
      checks.postgres = 'ok';
    } catch {
      checks.postgres = 'fail';
    }

    try {
      await redis.ping();
      checks.redis = 'ok';
    } catch {
      checks.redis = 'fail';
    }

    const allOk = Object.values(checks)
      .every(v => v === 'ok');

    return reply
      .status(allOk ? 200 : 503)
      .send({ status: allOk ? 'ready' : 'degraded', checks });
  });
}
```

### /health (Liveness)

- Returns 200 if process is alive
- K8s restarts container if this fails
- Keep it **fast** -- no DB calls

### /ready (Readiness)

- Checks **all dependencies** (DB, Redis, disk)
- Returns 503 if any check fails
- K8s removes pod from load balancer
- Traffic resumes when check passes again

### What to Check

| Dependency  | Check           |
|-------------|-----------------|
| PostgreSQL  | `SELECT 1`      |
| Redis       | `PING`          |
| RabbitMQ    | Channel open    |
| Disk        | Write temp file |
| External API| HEAD request    |

---

## Slide 7: Graceful Shutdown

```typescript
// src/lifecycle.ts
import { FastifyInstance } from 'fastify';
import { pool } from './db.js';
import { redis } from './redis.js';
import { channel } from './mq.js';

const SHUTDOWN_TIMEOUT = 15_000; // 15 seconds

export function gracefulShutdown(app: FastifyInstance) {
  let shuttingDown = false;

  const shutdown = async (signal: string) => {
    if (shuttingDown) return;
    shuttingDown = true;

    app.log.info({ signal }, 'Shutdown signal received');

    // 1. Stop accepting new connections
    //    Fastify .close() drains in-flight requests
    const timer = setTimeout(() => {
      app.log.error('Shutdown timed out, forcing exit');
      process.exit(1);
    }, SHUTDOWN_TIMEOUT);

    try {
      // 2. Close HTTP server (drain in-flight)
      await app.close();
      app.log.info('HTTP server closed');

      // 3. Close external connections
      await channel?.close();
      app.log.info('RabbitMQ channel closed');

      await redis.quit();
      app.log.info('Redis connection closed');

      await pool.end();
      app.log.info('PostgreSQL pool closed');

      clearTimeout(timer);
      app.log.info('Clean shutdown complete');
      process.exit(0);
    } catch (err) {
      app.log.error(err, 'Error during shutdown');
      process.exit(1);
    }
  };

  process.on('SIGTERM', () => shutdown('SIGTERM'));
  process.on('SIGINT',  () => shutdown('SIGINT'));
}
```

### Why It Matters

- K8s sends **SIGTERM** before killing a pod
- Default grace period: 30 seconds
- Without handling: dropped requests, corrupt data

### Shutdown Order

1. Stop accepting **new connections**
2. Drain **in-flight requests**
3. Close message queue channels
4. Close cache connections (Redis)
5. Close database pools
6. Exit process

### Common Mistakes

- No timeout -- shutdown hangs forever
- Closing DB before draining HTTP requests
- Forgetting `SIGINT` (Ctrl+C in dev)
- Using `process.exit()` without cleanup

---

## Slide 8: Environment Configuration

```typescript
// src/config.ts — Typed, validated configuration
import { z } from 'zod';
import 'dotenv/config';

const envSchema = z.object({
  NODE_ENV:     z.enum(['development', 'production', 'test'])
                 .default('development'),
  PORT:         z.coerce.number().int().default(3000),
  LOG_LEVEL:    z.enum(['fatal','error','warn','info','debug','trace'])
                 .default('info'),

  // Database
  DATABASE_URL: z.string().url()
    .describe('PostgreSQL connection string'),
  DB_POOL_MIN:  z.coerce.number().int().default(2),
  DB_POOL_MAX:  z.coerce.number().int().default(10),

  // Redis
  REDIS_URL:    z.string().url().default('redis://localhost:6379'),

  // Auth
  JWT_SECRET:   z.string().min(32),
  JWT_EXPIRES:  z.string().default('15m'),

  // External services
  ORDER_SERVICE_URL: z.string().url()
    .default('http://order-service:3001'),
});

export type Config = z.infer<typeof envSchema>;

const parsed = envSchema.safeParse(process.env);

if (!parsed.success) {
  console.error('Invalid environment:', parsed.error.format());
  process.exit(1);
}

export const config = parsed.data;
```

### 12-Factor App Config

- Store config in **environment variables**
- Never commit secrets to source control
- Same artefact deployed to all environments
- Only env vars change between dev/staging/prod

### Validation Benefits

- Fail **fast at startup** -- not at 3 AM
- TypeScript types inferred from schema
- Default values for optional config
- Descriptive errors for missing values

### Terminal Output on Failure

```bash
$ node dist/server.js
Invalid environment:
{
  JWT_SECRET: {
    _errors: ["String must contain at
    least 32 character(s)"]
  },
  DATABASE_URL: {
    _errors: ["Required"]
  }
}
# Process exits with code 1
```

---

## Slide 9: Containerising a Node.js Service

```dockerfile
# Dockerfile — Multi-stage production build
# Stage 1: Install dependencies
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile

# Stage 2: Build TypeScript
FROM deps AS build
COPY tsconfig.json ./
COPY src/ ./src/
RUN pnpm build

# Stage 3: Production image
FROM node:22-alpine AS runtime
WORKDIR /app

# Security: non-root user
RUN addgroup -g 1001 appgroup && \
    adduser -u 1001 -G appgroup -s /bin/sh -D appuser

ENV NODE_ENV=production

# Copy only production artifacts
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY package.json ./

# Drop privileges
USER appuser

EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "dist/server.js"]
```

```bash
# .dockerignore
node_modules
dist
.env*
.git
*.md
coverage
.turbo
```

### Multi-Stage Benefits

- Final image has **no devDependencies**
- No TypeScript compiler in production
- Image size: ~150 MB (Alpine) vs ~900 MB (full)

### Security Checklist

- **Non-root user** -- never run as root
- **NODE_ENV=production** -- disables dev features
- No .env files in image
- Pin exact base image versions

### Base Image Comparison

| Base Image                      | Size     | Use Case                    |
|---------------------------------|----------|-----------------------------|
| `node:22-alpine`                | ~55 MB   | Best default choice         |
| `node:22-slim`                  | ~80 MB   | Need glibc (native addons)  |
| `gcr.io/distroless/nodejs`      | ~45 MB   | Minimal attack surface      |
| `node:22`                       | ~350 MB  | Avoid in production         |

---

## Slide 10: Docker Compose for Local Development

```yaml
# docker-compose.yml
services:
  user-service:
    build: ./services/user-service
    ports: ["3000:3000"]
    environment:
      DATABASE_URL: postgres://app:secret@postgres:5432/users
      REDIS_URL: redis://redis:6379
      RABBITMQ_URL: amqp://rabbit:5672
      JWT_SECRET: local-dev-secret-min-32-chars-long!!
    depends_on:
      postgres: { condition: service_healthy }
      redis:    { condition: service_healthy }
      rabbit:   { condition: service_healthy }
    networks: [backend]

  order-service:
    build: ./services/order-service
    ports: ["3001:3001"]
    environment:
      DATABASE_URL: postgres://app:secret@postgres:5432/orders
      REDIS_URL: redis://redis:6379
      RABBITMQ_URL: amqp://rabbit:5672
      USER_SERVICE_URL: http://user-service:3000
      JWT_SECRET: local-dev-secret-min-32-chars-long!!
    depends_on: [postgres, redis, rabbit]
    networks: [backend]

  payment-service:
    build: ./services/payment-service
    ports: ["3002:3002"]
    environment:
      RABBITMQ_URL: amqp://rabbit:5672
      STRIPE_KEY: sk_test_fake_key_for_dev
    depends_on: [rabbit]
    networks: [backend]

  postgres:
    image: postgres:16-alpine
    environment: { POSTGRES_USER: app, POSTGRES_PASSWORD: secret }
    volumes: [pg_data:/var/lib/postgresql/data, ./init-db.sql:/docker-entrypoint-initdb.d/init.sql]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
    networks: [backend]

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
    networks: [backend]

  rabbit:
    image: rabbitmq:3.13-management-alpine
    ports: ["5672:5672", "15672:15672"]
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_running"]
      interval: 10s
    networks: [backend]

volumes:
  pg_data:

networks:
  backend:
    driver: bridge
```

---

## Slide 11: Inter-Service Communication

### Synchronous (Request/Response)

- **HTTP/REST** -- simple, widely understood
- **gRPC** -- binary protocol, codegen, streaming
- Client expects a response immediately
- Creates temporal coupling between services

```typescript
// Calling another service via HTTP
const user = await fetch(
  `${config.USER_SERVICE_URL}/api/users/${id}`,
  { headers: { Authorization: req.headers.authorization } }
).then(r => r.json());
```

### Asynchronous (Event-Driven)

- **Message queues** -- RabbitMQ, SQS, Redis Streams
- **Event bus** -- Kafka, NATS, EventBridge
- Fire-and-forget, eventual consistency
- Services are decoupled in time

```typescript
// Publishing an event
await channel.publish('events', 'order.created',
  Buffer.from(JSON.stringify({
    orderId, userId, total, createdAt: new Date()
  }))
);
```

### Comparison Table

| Criteria    | HTTP/REST       | gRPC                     | Message Queue  | Event Bus     |
|-------------|-----------------|--------------------------|----------------|---------------|
| Latency     | Medium          | Low                      | Variable       | Variable      |
| Coupling    | High            | High                     | Low            | Very Low      |
| Complexity  | Low             | Medium                   | Medium         | High          |
| Reliability | Retry needed    | Retry needed             | Built-in       | Built-in      |
| Best For    | CRUD, queries   | High-throughput internal  | Task processing| Domain events |

---

## Slide 12: Message Queues & Event-Driven Architecture

### Publisher (order-service)

```typescript
// src/mq.ts — RabbitMQ with amqplib
import amqp from 'amqplib';

let connection: amqp.Connection;
let channel: amqp.Channel;

export async function connectMQ(url: string) {
  connection = await amqp.connect(url);
  channel = await connection.createChannel();

  // Ensure exchange exists
  await channel.assertExchange(
    'events', 'topic', { durable: true }
  );
  return { connection, channel };
}

// Publish a domain event
export async function publishEvent(
  routingKey: string,
  payload: Record<string, unknown>
) {
  const msg = JSON.stringify({
    id: crypto.randomUUID(),
    type: routingKey,
    timestamp: new Date().toISOString(),
    data: payload,
  });

  channel.publish(
    'events',
    routingKey,
    Buffer.from(msg),
    { persistent: true, contentType: 'application/json' }
  );
}
```

### Consumer (payment-service)

```typescript
// src/consumers/order-created.ts
import { channel } from '../mq.js';

export async function startConsumer() {
  const q = await channel.assertQueue(
    'payment.order-created',
    { durable: true }
  );

  await channel.bindQueue(
    q.queue, 'events', 'order.created'
  );

  // Prefetch 1 = process one at a time
  channel.prefetch(1);

  channel.consume(q.queue, async (msg) => {
    if (!msg) return;
    try {
      const event = JSON.parse(
        msg.content.toString()
      );
      await processPayment(event.data);
      channel.ack(msg);     // success
    } catch (err) {
      channel.nack(msg, false, true); // requeue
    }
  });
}
```

### Broker Comparison

| Broker        | Strength                      |
|---------------|-------------------------------|
| RabbitMQ      | Routing flexibility, mature   |
| NATS          | Ultra low latency, cloud-native|
| Redis Streams | If you already have Redis     |
| AWS SQS       | Zero ops, serverless          |

---

## Slide 13: API Gateway Pattern

### Architecture

```
Clients                  API Gateway                   Services
+----------+         +-------------------+         +--------------+
| Web App  | ------> |  Auth / JWT       | ------> | user-svc     |
| Mobile   | ------> |  Rate Limiting    | ------> | order-svc    |
| 3rd Party| ------> |  Routing          | ------> | payment-svc  |
+----------+         |  Request Transform|         +--------------+
                      |  Logging          |
                      +-------------------+
```

### Gateway Options

| Tool       | Type          | Notes                  |
|------------|---------------|------------------------|
| Kong       | Full-featured | Plugins, Lua-based     |
| Nginx      | Reverse proxy | High perf, config-driven|
| AWS API GW | Managed       | Zero ops, pay-per-call |
| Express    | Custom        | Full control, more work|

### Simple Express Gateway

```typescript
// gateway/src/server.ts
import express from 'express';
import { createProxyMiddleware } from
  'http-proxy-middleware';
import rateLimit from 'express-rate-limit';
import { verifyJWT } from './auth.js';

const app = express();

// Global rate limiting
app.use(rateLimit({
  windowMs: 60_000,
  max: 100,
  standardHeaders: true,
}));

// Auth middleware (skip for public routes)
app.use('/api', verifyJWT);

// Route to services
app.use('/api/users',
  createProxyMiddleware({
    target: 'http://user-service:3000',
    pathRewrite: { '^/api/users': '/api/users' },
    changeOrigin: true,
  })
);

app.use('/api/orders',
  createProxyMiddleware({
    target: 'http://order-service:3001',
    pathRewrite: { '^/api/orders': '/api/orders' },
    changeOrigin: true,
  })
);

app.use('/api/payments',
  createProxyMiddleware({
    target: 'http://payment-service:3002',
    pathRewrite: {
      '^/api/payments': '/api/payments'
    },
    changeOrigin: true,
  })
);

app.listen(8080, () =>
  console.log('Gateway on :8080')
);
```

---

## Slide 14: Service Discovery & Load Balancing

### DNS-Based (Simplest)

- Docker Compose: service name = hostname
- Kubernetes: `service-name.namespace.svc.cluster.local`
- No code changes needed
- DNS caching can cause stale lookups

```typescript
// In Docker Compose, service names
// resolve to container IPs:
const url = 'http://user-service:3000';
// K8s:
const url =
  'http://user-svc.default.svc.cluster.local';
```

### Client-Side Discovery

- Service registers itself in a registry
- Client queries registry, picks an instance
- Tools: Consul, etcd, Eureka
- More control, more complexity

```typescript
// Using Consul for discovery
import Consul from 'consul';
const consul = new Consul();

const services = await consul.health
  .service({ service: 'user-service', passing: true });

const instance = services[
  Math.floor(Math.random() * services.length)
];
const url =
  `http://${instance.Service.Address}:${instance.Service.Port}`;
```

### Service Mesh

- Sidecar proxy intercepts all traffic
- Zero application code changes
- mTLS, retries, circuit breaking built in
- Tools: Istio, Linkerd, Consul Connect

```yaml
# Istio VirtualService
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: user-service
spec:
  hosts: [user-service]
  http:
  - route:
    - destination:
        host: user-service
      weight: 90
    - destination:
        host: user-service-canary
      weight: 10
```

### Complexity Progression

DNS (Compose/K8s) --> Client Discovery (Consul) --> Service Mesh (Istio) *(complexity & power -->)*

---

## Slide 15: Authentication Between Services

### JWT Middleware

```typescript
// src/middleware/auth.ts
import jwt from 'jsonwebtoken';
import { config } from '../config.js';
import { FastifyRequest, FastifyReply } from 'fastify';

interface JWTPayload {
  sub: string;
  email: string;
  roles: string[];
  iat: number;
  exp: number;
}

export async function verifyJWT(
  req: FastifyRequest,
  reply: FastifyReply
) {
  const header = req.headers.authorization;
  if (!header?.startsWith('Bearer ')) {
    return reply.status(401).send({
      error: 'Missing bearer token'
    });
  }

  try {
    const token = header.slice(7);
    const payload = jwt.verify(
      token, config.JWT_SECRET
    ) as JWTPayload;

    // Attach to request for downstream use
    req.user = payload;
  } catch (err) {
    return reply.status(401).send({
      error: 'Invalid or expired token'
    });
  }
}

// Propagate JWT to downstream services
export function forwardAuth(req: FastifyRequest) {
  return {
    Authorization: req.headers.authorization,
    'X-Request-Id': req.headers['x-request-id'],
  };
}
```

### JWT Propagation (Most Common)

- Gateway validates user JWT once
- JWT forwarded to internal services via `Authorization` header
- Each service can inspect claims (roles, sub)
- No extra network calls for auth

### Service-to-Service Auth

- **API Keys** -- simple, rotated via env vars
- **mTLS** -- mutual TLS, handled by service mesh
- **Service JWTs** -- short-lived, machine-to-machine

```typescript
// Service-to-service with API key
const res = await fetch(url, {
  headers: {
    'X-Service-Key': config.INTERNAL_API_KEY,
    'X-Service-Name': 'order-service',
  }
});
```

### Security Pitfalls

- Never expose internal services to the internet
- Rotate keys/secrets via K8s Secrets or Vault
- Set short JWT expiry (15m) + refresh tokens
- Log auth failures -- detect brute force

---

## Slide 16: Logging & Observability

### Structured Logging with Pino

```typescript
// src/logger.ts
import pino from 'pino';
import { config } from './config.js';

export const logger = pino({
  level: config.LOG_LEVEL,
  base: {
    service: 'order-service',
    version: process.env.npm_package_version,
    env: config.NODE_ENV,
  },
  timestamp: pino.stdTimeFunctions.isoTime,
  formatters: {
    level: (label) => ({ level: label }),
  },
  redact: ['req.headers.authorization',
           'req.headers.cookie'],
});
```

### Correlation IDs Across Services

```typescript
// src/middleware/correlation.ts
import { FastifyInstance } from 'fastify';
import crypto from 'node:crypto';

export function correlationPlugin(app: FastifyInstance) {
  app.addHook('onRequest', (req, _reply, done) => {
    // Use incoming ID or generate new one
    const correlationId =
      (req.headers['x-request-id'] as string)
      ?? crypto.randomUUID();

    req.correlationId = correlationId;

    // Bind to logger for this request
    req.log = req.log.child({ correlationId });
    done();
  });

  app.addHook('onSend', (req, reply, _payload, done) => {
    reply.header('X-Request-Id', req.correlationId);
    done();
  });
}
```

### JSON Log Output

```json
{
  "level": "info",
  "time": "2026-03-24T10:23:45.123Z",
  "service": "order-service",
  "correlationId": "a1b2c3d4-e5f6-...",
  "msg": "Order created",
  "orderId": "ord_98765",
  "userId": "usr_12345",
  "total": 49.99,
  "responseTime": 23
}
```

### Observability Stack

| Pillar   | Tool                             |
|----------|----------------------------------|
| Logs     | ELK Stack / Loki + Grafana      |
| Metrics  | Prometheus + Grafana             |
| Traces   | Jaeger / Tempo (OpenTelemetry)   |
| Alerting | Grafana Alerting / PagerDuty     |

### Key Rule

Every log line must include a **correlationId**. Without it, debugging across 5+ services is impossible.

---

## Slide 17: Deploying to Kubernetes

### Deployment Manifest

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  labels:
    app: user-service
spec:
  replicas: 3
  selector:
    matchLabels: { app: user-service }
  template:
    metadata:
      labels: { app: user-service }
    spec:
      containers:
      - name: user-service
        image: ghcr.io/myorg/user-service:1.4.2
        ports:
        - containerPort: 3000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: user-db-secret
              key: url
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet: { path: /health, port: 3000 }
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet: { path: /ready, port: 3000 }
          initialDelaySeconds: 5
          periodSeconds: 5
      terminationGracePeriodSeconds: 30
```

### Service & Ingress

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector: { app: user-service }
  ports:
  - port: 80
    targetPort: 3000
  type: ClusterIP
---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /users(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: user-service
            port: { number: 80 }
```

### Horizontal Pod Autoscaler

```yaml
# k8s/hpa.yaml — Autoscaling
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

## Slide 18: Deploying to AWS (ECS/Fargate & Lambda)

### ECS Task Definition

```json
{
  "family": "user-service",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [{
    "name": "user-service",
    "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/user-service:1.4.2",
    "portMappings": [{ "containerPort": 3000 }],
    "environment": [
      { "name": "NODE_ENV", "value": "production" }
    ],
    "secrets": [{
      "name": "DATABASE_URL",
      "valueFrom": "arn:aws:ssm:us-east-1:123456789:parameter/prod/user-service/db-url"
    }],
    "healthCheck": {
      "command": ["CMD-SHELL",
        "wget -qO- http://localhost:3000/health || exit 1"],
      "interval": 30,
      "timeout": 5,
      "retries": 3
    },
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/user-service",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "ecs"
      }
    }
  }]
}
```

### ECS/Fargate vs Lambda Comparison

| Criteria      | ECS/Fargate            | Lambda                   |
|---------------|------------------------|--------------------------|
| Startup       | ~30s (container)       | ~200ms (cold start)      |
| Long-running  | Yes, always on         | Max 15 min               |
| Scaling       | Min/max tasks          | Automatic, per-request   |
| WebSockets    | Supported              | Via API GW WS            |
| Cost model    | Per hour (vCPU+mem)    | Per invocation           |
| Best for      | Stateful / high traffic| Bursty / event-driven    |

### Fargate vs EC2 Launch Type

- **Fargate:** zero server management, pay-per-task, best default
- **EC2:** cheaper at scale, GPU workloads, more control

### Lambda as Microservice

```typescript
// handler.ts — Lambda with API GW
import { APIGatewayProxyHandlerV2 } from 'aws-lambda';

export const handler: APIGatewayProxyHandlerV2 =
  async (event) => {
    const { pathParameters, body } = event;
    // Same business logic, different wrapper
    const user = await userService.getById(
      pathParameters?.id!
    );
    return {
      statusCode: 200,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(user),
    };
  };
```

---

## Slide 19: CI/CD Pipeline for Microservices

```yaml
# .github/workflows/deploy-service.yml
name: Build & Deploy Service
on:
  push:
    branches: [main]
    paths: ['services/user-service/**']   # Only trigger for this service

env:
  SERVICE: user-service
  REGISTRY: ghcr.io/${{ github.repository_owner }}

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
    - uses: actions/checkout@v4

    - uses: actions/setup-node@v4
      with: { node-version: 22, cache: 'pnpm' }

    - name: Install & Test
      run: |
        corepack enable
        pnpm install --frozen-lockfile
        pnpm --filter ${{ env.SERVICE }} run lint
        pnpm --filter ${{ env.SERVICE }} run test
        pnpm --filter ${{ env.SERVICE }} run build

    - name: Set image tag
      id: tag
      run: echo "tag=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

    - name: Build & Push Docker Image
      uses: docker/build-push-action@v5
      with:
        context: services/${{ env.SERVICE }}
        push: true
        tags: |
          ${{ env.REGISTRY }}/${{ env.SERVICE }}:${{ steps.tag.outputs.tag }}
          ${{ env.REGISTRY }}/${{ env.SERVICE }}:latest
        cache-from: type=gha
        cache-to: type=gha,mode=max

    - name: Deploy to Kubernetes
      uses: azure/k8s-deploy@v5
      with:
        manifests: services/${{ env.SERVICE }}/k8s/
        images: ${{ env.REGISTRY }}/${{ env.SERVICE }}:${{ steps.tag.outputs.tag }}
        namespace: production
        strategy: canary
        percentage: 20
```

### Pipeline Flow

Push to main --> Lint + Test --> Build Image --> Canary 20% --> Full Rollout

---

## Slide 20: Database per Service

### Architecture

```
+----------------+    +----------------+    +----------------+
| user-service   |    | order-service  |    | payment-svc    |
+-------+--------+    +-------+--------+    +-------+--------+
        |                     |                     |
        v                     v                     v
+----------------+    +----------------+    +----------------+
| PostgreSQL     |    | MongoDB        |    | PostgreSQL     |
| users, profiles|    | orders, items  |    | transactions   |
+----------------+    +----------------+    +----------------+
        \                     |                    /
         \                    |                   /
    +---------------------------------------------+
    | Event Bus (domain events for data sync)      |
    +---------------------------------------------+

Rule: services NEVER access another service's database
Communicate via APIs or events only
```

### Why Own Your Data

- Independent schema evolution
- Choose the right DB per workload
- No shared-DB coupling between teams
- Services can be deployed independently

### The Saga Pattern

Distributed transactions across services using a sequence of local transactions + compensating actions:

1. **order-service**: create order (PENDING)
2. **payment-service**: charge card
3. **inventory-service**: reserve stock
4. If any step fails --> run **compensations**

### Data Consistency Strategies

| Pattern                | Consistency    |
|------------------------|----------------|
| Saga (choreography)    | Eventual       |
| Saga (orchestration)   | Eventual       |
| Event Sourcing         | Eventual       |
| Outbox Pattern         | At-least-once  |

---

## Slide 21: Common Pitfalls & Production Lessons

### Distributed System Fallacies

- The network is **not** reliable
- Latency is **not** zero
- Bandwidth is **not** infinite
- The topology **does** change

Every inter-service call can fail. Design for it.

### Circuit Breaker (opossum)

```typescript
import CircuitBreaker from 'opossum';

const breaker = new CircuitBreaker(
  async (userId: string) => {
    const res = await fetch(
      `${config.USER_SERVICE_URL}/api/users/${userId}`
    );
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return res.json();
  },
  {
    timeout: 3000,           // 3s per call
    errorThresholdPercentage: 50,
    resetTimeout: 10000,     // try again after 10s
    volumeThreshold: 5,      // min calls before tripping
  }
);

breaker.on('open',    () => log.warn('Circuit OPEN'));
breaker.on('halfOpen',() => log.info('Circuit HALF-OPEN'));
breaker.on('close',   () => log.info('Circuit CLOSED'));

// Usage
breaker.fallback(() => ({ id: userId, name: 'Unknown' }));
const user = await breaker.fire(userId);
```

### Retry with Exponential Backoff

```typescript
// src/utils/retry.ts
export async function retry<T>(
  fn: () => Promise<T>,
  opts = { maxRetries: 3, baseDelay: 200 }
): Promise<T> {
  let lastError: Error | undefined;

  for (let attempt = 0; attempt <= opts.maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      lastError = err as Error;
      if (attempt < opts.maxRetries) {
        const delay = opts.baseDelay * 2 ** attempt
          + Math.random() * 100; // jitter
        await new Promise(r => setTimeout(r, delay));
      }
    }
  }

  throw lastError;
}

// Usage
const data = await retry(
  () => fetchFromService(url),
  { maxRetries: 3, baseDelay: 200 }
);
```

### Timeout Management

- Set timeouts on **every** outbound call
- Use `AbortController` with `fetch`
- Cascade: gateway 30s > service 10s > DB 5s
- Without timeouts: one slow service kills all

```typescript
const controller = new AbortController();
setTimeout(() => controller.abort(), 5000);

const res = await fetch(url, {
  signal: controller.signal,
});
```

---

## Slide 22: Summary & Further Reading

### Key Takeaways

1. **Start monolithic** -- extract services when team/scaling pain is real
2. Each service: health checks, graceful shutdown, structured logging
3. Validate config at startup -- fail fast, not at 3 AM
4. Multi-stage Dockerfiles, non-root user, Alpine base
5. Prefer **async messaging** over sync HTTP between services
6. Database per service + saga pattern for consistency
7. Circuit breakers, retries, timeouts on every call
8. Correlation IDs in every log line across every service

### Learning Path

Node.js basics --> REST APIs --> **Docker** --> K8s --> Service Mesh

### Recommended Books

- **Building Microservices** -- Sam Newman (2nd ed.) -- the definitive guide to microservice architecture
- **Node.js Design Patterns** -- Mario Casciaro & Luciano Mammino -- advanced patterns for production Node
- **Designing Data-Intensive Applications** -- Martin Kleppmann -- distributed data fundamentals
- **Release It!** -- Michael Nygard -- production stability patterns

### Tools & Resources

| Category      | Tools                            |
|---------------|----------------------------------|
| Framework     | Fastify, NestJS, Express         |
| Queue         | RabbitMQ, BullMQ, NATS           |
| Containers    | Docker, Podman, Buildah          |
| Orchestration | Kubernetes, ECS, Nomad           |
| CI/CD         | GitHub Actions, GitLab CI        |
| Monitoring    | Prometheus, Grafana, Datadog     |
| Tracing       | OpenTelemetry, Jaeger            |
