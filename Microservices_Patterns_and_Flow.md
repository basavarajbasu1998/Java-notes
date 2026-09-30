# Microservices: Flow, Resilience, Saga, Patterns

## From monolith to services
```
MONOLITH: one app, one DB                MICROSERVICES: each service owns its data
┌──────────────────────┐                 ┌────────┐ ┌────────┐ ┌─────────┐ ┌────────┐
│ users orders payment │ one deploy      │ User   │ │ Order  │ │ Payment │ │ Notify │
│ inventory email      │ one DB          │ +DB    │ │ +DB    │ │ +DB     │ │        │
└──────────────────────┘                 └────────┘ └────────┘ └─────────┘ └────────┘
```
Split by **business capability (bounded context)**, not by technical layer. **Database per service** (no shared tables).

## Component flow
```
Client → API Gateway (auth, routing, rate limit)
              │ asks Service Registry (Eureka/Consul/K8s DNS) "where is order-service?"
              ▼
         Load balancer → Order Service instance 1/2/3
              │ Feign/WebClient (sync)  or  events (async)
              ▼
         Payment Service …    Config Server (central config)   Tracing (Zipkin/Jaeger)
```
Spring stack: Spring Cloud Gateway, OpenFeign, Resilience4j, Config Server, Micrometer Tracing. On Kubernetes, service discovery and config are provided by K8s itself (Service DNS, ConfigMap).

## Sync call with resilience
```java
@FeignClient(name = "payment-service", url = "${payment.url}")
interface PaymentClient { @PostMapping("/payments") PaymentResponse charge(PaymentRequest r); }

@CircuitBreaker(name = "payment", fallbackMethod = "paymentFallback")
@Retry(name = "payment")
@TimeLimiter(name = "payment")
public PaymentResponse charge(PaymentRequest r) { return paymentClient.charge(r); }
```
Circuit breaker states:
```
CLOSED (normal) ──failures > threshold──► OPEN (fail fast, return fallback)
   ▲                                          │ after wait time
   └── trial calls succeed ◄── HALF-OPEN ◄────┘   (a few test calls; fail → OPEN again)
```

## Distributed transaction → SAGA
No single DB transaction across services. Use a chain of local transactions with **compensations**.
```
Order created ─► Reserve stock ─► Charge payment ─► Confirm order
                      │                 │ FAILS
                      ▼                 ▼
              Release stock ◄──── Cancel order      (compensating actions)
```
- **Choreography:** services react to each other's events (no central brain; simple, hard to track).
- **Orchestration:** a saga coordinator tells each service what to do (clear flow, one more component).

## Patterns checklist
API Gateway · Service Discovery · Config Server · Circuit Breaker · Bulkhead · Retry with backoff · Saga · **Outbox** · CQRS (separate read/write models) · Event Sourcing · Strangler Fig (migrate monolith) · Sidecar/Service mesh (Istio) · BFF (Backend-for-Frontend) · Idempotent consumers · Distributed tracing (trace-id across services) · Health checks.
CAP theorem: in a network partition choose **C**onsistency or **A**vailability; microservices usually pick availability + **eventual consistency**.
Inter-service security: JWT propagation, mTLS, OAuth2 client-credentials.
Disadvantages (say them!): network latency, distributed debugging, data consistency, operational overhead → don't use microservices for small teams/apps.
