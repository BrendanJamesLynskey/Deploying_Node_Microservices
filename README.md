# ⬢ Deploying Node.js Microservices

An interactive Reveal.js presentation on deploying Node.js microservices — service design, containerisation, inter-service communication, Kubernetes, AWS, and production observability.

## ▶ [Open the Presentation](https://brendanjameslynskey.github.io/Deploying_Node_Microservices/)

## 📄 [Markdown Version](presentation.md)

---

## Contents

| # | Topic | Description |
|---|-------|-------------|
| 01 | Topics We'll Cover | Architecture, containers, security, deployment |
| 02 | Monolith vs Microservices | When to stay monolithic, when to split |
| 03 | Anatomy of a Node.js Microservice | Fastify entry point and key principles |
| 04 | Structuring a Project | Monorepo vs polyrepo, Turborepo, pnpm |
| 05 | Health Checks & Readiness Probes | Liveness vs readiness endpoints |
| 06 | Graceful Shutdown | SIGTERM handling and shutdown ordering |
| 07 | Environment Configuration | Typed, validated config with Zod |
| 08 | Containerising a Node.js Service | Multi-stage Dockerfile, non-root user |
| 09 | Docker Compose for Local Dev | Full stack with Postgres, Redis, RabbitMQ |
| 10 | Inter-Service Communication | HTTP, gRPC, message queues, event buses |
| 11 | Message Queues & Event-Driven | RabbitMQ publishers and consumers |
| 12 | API Gateway Pattern | Auth, rate limits, routing in one place |
| 13 | Service Discovery & Load Balancing | DNS, Consul, service mesh progression |
| 14 | Authentication Between Services | JWT propagation, API keys, mTLS |
| 15 | Logging & Observability | Structured logs, correlation IDs, Pino |
| 16 | Deploying to Kubernetes | Deployments, services, HPA autoscaling |
| 17 | Deploying to AWS | ECS/Fargate task definitions and Lambda |
| 18 | CI/CD Pipeline for Microservices | GitHub Actions per-service builds and canaries |
| 19 | Database per Service | Owned data, saga pattern, outbox |
| 20 | Common Pitfalls & Production Lessons | Circuit breakers, retries, timeouts |
| 21 | Summary & Further Reading | Key takeaways and recommended books |

---

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Export to PDF | Append `?print-pdf` to URL, then print |

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) · Playfair Display + DM Sans + JetBrains Mono

Single self-contained `index.html` — no build step, no npm, no dependencies to install.

## References

[Kubernetes Documentation](https://kubernetes.io/docs/) · [AWS ECS Documentation](https://docs.aws.amazon.com/ecs/) · [Fastify Documentation](https://fastify.dev/docs/latest/) · *Building Microservices* by Sam Newman · *Release It!* by Michael Nygard

## License

Educational use. Code examples provided as-is.
