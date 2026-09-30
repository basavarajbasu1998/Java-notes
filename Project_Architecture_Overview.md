# Project Architecture Overview (One Feature, Every Technology)

```
 ┌──────────────────────────── USER'S BROWSER ────────────────────────────┐
 │  React app (JS + CSS)  → click "Place Order" → fetch POST /api/orders   │
 └───────────────────────────────────┬─────────────────────────────────────┘
                                     │ HTTPS
                          ┌──────────▼──────────┐
                          │ AWS: Route53 → CloudFront/ALB (load balancer) │
                          └──────────┬──────────┘
                                     │
                 ┌───────────────────▼────────────────────┐
                 │  Kubernetes cluster (EKS)               │
                 │  ┌──────────────┐                       │
                 │  │ API Gateway  │ (auth JWT, rate limit)│
                 │  └──────┬───────┘                       │
                 │   ┌─────▼──────┐  REST/Feign  ┌────────────┐
                 │   │Order Service│────────────►│Payment Svc │
                 │   │(Spring Boot)│             └────────────┘
                 │   └──┬──────┬───┘
                 │      │      │ publish "OrderPlaced"
                 │      │      ▼
                 │      │  ┌─────────┐   ┌────────────────────────────┐
                 │      │  │RabbitMQ │──►│ Email Svc / Inventory Svc  │
                 │      │  │ (tasks) │   └────────────────────────────┘
                 │      │  └─────────┘
                 │      │  ┌─────────┐   ┌────────────────────────────┐
                 │      └─►│ Kafka   │──►│ Analytics / Fraud / Search │
                 │         │(events) │   └────────────────────────────┘
                 └──────────────┬──────────────────────────┘
                                ▼
                     ┌────────────────────┐
                     │ RDS MySQL/Postgres │ (SQL)     Redis (cache)
                     └────────────────────┘
 Code path:  Git → CI pipeline → Docker image → registry (ECR) → Kubernetes deploy
 Work path:  Jira story → SDLC/Agile sprint → (AI assists at each phase) → release
```

**Request timeline (memorise this story):**
```
1  User clicks button                          React
2  fetch() sends JSON with JWT                 JavaScript
3  Load balancer picks a healthy pod           AWS ALB / K8s Service
4  Gateway checks JWT, routes to Order Service Spring Cloud Gateway
5  Controller → validates → Service            Spring Boot
6  @Transactional: save order in DB            SQL / JPA
7  Call Payment service (with circuit breaker) Microservices
8  Publish "send email" job                    RabbitMQ
9  Publish "OrderPlaced" event                 Kafka
10 Return 201 + order id → React updates UI    React state
```
