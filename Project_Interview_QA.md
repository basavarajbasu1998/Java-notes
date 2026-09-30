# Project Interview Q&A + 8-Week Plan

**"Explain your project architecture."**
> "React SPA served from S3/CloudFront calls our API via an ALB and API Gateway. Spring Boot microservices (order, payment, inventory) run as containers on Kubernetes on AWS. Each service owns its MySQL schema on RDS. Synchronous calls use Feign with Resilience4j; asynchronous work goes through RabbitMQ (emails, inventory) and business events go to Kafka (analytics, fraud). Distributed consistency uses Saga + Outbox. Code flows Git → PR review → CI (tests, Sonar) → Docker image in ECR → Helm deploy to EKS, with CloudWatch/Prometheus monitoring."

**"What happens when you type a URL / click Place Order?"** → DNS → TCP/TLS → CDN/ALB → gateway → filter chain → controller → service → transaction → DB → response → React re-render (use the timeline in `Project_Architecture_Overview.md`).

**"How do you make the system reliable?"** timeouts, retries with backoff (idempotent only), circuit breaker, bulkhead, DLQ, idempotent consumers, outbox, health probes, multi-AZ, autoscaling, backups, alerts.

**"How do you make it fast?"** indexes/EXPLAIN, pagination, caching (Redis cache-aside), async messaging, connection pools, N+1 fixes, CDN, compression, horizontal scale.

**"How do you make it secure?"** HTTPS, JWT/OAuth2, RBAC, input validation, parameterised SQL, secrets manager, least-privilege IAM, image scanning, dependency updates, CORS/CSRF, rate limiting, audit logs.

**"Monolith or microservices?"** Start monolith for small teams; split when teams/scale/deploy-independence need it; pay cost of distribution knowingly.

**"How do you use AI in your work?"** assistant for boilerplate/tests/reviews within a spec-driven flow with human approval gates; never paste secrets; all AI output passes the same tests and review.

**Tool-vs-tool quick table**
| Question | Short answer |
|---|---|
| Docker vs VM | shares host kernel → lighter/faster start |
| Docker vs Kubernetes | package/run one container vs orchestrate many |
| Deployment vs StatefulSet | interchangeable pods vs stable identity/storage |
| RabbitMQ vs Kafka | task queue (deleted after ack) vs replayable log |
| SQS vs RabbitMQ | managed AWS simple queue vs feature-rich broker you run |
| ECS vs EKS | AWS-native simple vs Kubernetes standard |
| EC2 vs Lambda | you manage server vs run per invocation |
| SQL vs NoSQL | ACID relations vs scale/flexibility |
| Scrum vs Kanban | timeboxed sprints vs continuous flow with WIP limits |
| CI vs CD | auto build+test on every push vs auto release to environment |
| var/let/const | function-scope / block / block-constant |
| Flexbox vs Grid | one dimension vs two |
| Props vs state | from parent, read-only vs owned by component |
| merge vs rebase | preserves history vs linear history |
| RAG vs fine-tuning | inject knowledge at query time vs change model weights |

---

## Suggested Learning Order (8 weeks, 1.5 h/day)
| Weeks | Focus | Deliverable |
|---|---|---|
| 1 | Git + SQL + Spring Boot CRUD (`Git.md`, `SpringBoot_Order_Service.md`, `SQL_Database_Performance.md`) | Order API with MySQL |
| 2 | Security + validation + tests (`Spring_Security_JWT_OAuth2.md`, `REST_API_Design.md`, `Testing_JUnit_Mockito.md`) | JWT-secured API with tests |
| 3 | JS + CSS + React (`JavaScript.md`, `CSS.md`, `React.md`) | Order page calling your API |
| 4 | Docker + compose (`Docker.md`) | whole stack with one command |
| 5 | Microservices + RabbitMQ (`Microservices_Patterns_and_Flow.md`, `RabbitMQ.md`) | Order → email service |
| 6 | Kafka + Saga/Outbox (`Kafka.md`) | events to analytics consumer |
| 7 | Kubernetes + AWS (`Kubernetes.md`, `AWS.md`) | deploy to minikube/kind, then EKS free-tier lab |
| 8 | CI/CD + AI + revision (`CI_CD_Pipeline.md`, `AI_LLM_RAG.md`, this file) | pipeline + small RAG chatbot; mock interviews |
