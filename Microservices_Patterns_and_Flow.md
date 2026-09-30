# Microservices: Patterns, Resilience, Saga, Outbox, Observability (Spring Boot 3 / Spring Cloud 2023+ / Resilience4j)

> Related notes: `Kafka.md` (topics, partitions, consumer groups, exactly-once), `Spring_Transactional.md` (local ACID, propagation, rollback rules), `RabbitMQ.md` (broker theory). Kafka and `@Transactional` internals are NOT repeated here; this note is about how services fit together and fail together.
> Versions: examples target Spring Boot 3.2-3.4, Spring Cloud 2023.0.x / 2024.0.x, Resilience4j 2.x (`resilience4j-spring-boot3`), Java 17/21. Spring Cloud 2025.0 renamed the Gateway artifacts/properties (see Part 2.6). Where I am not 100% sure of an exact key, the text says "conceptually".

## Contents
1. 60-second mental model + analogy
2. Deep explanation (2.1 when to use - 2.20 e-commerce worked design)
3. Code (Resilience4j, Gateway, outbox, inbox, saga orchestrator, idempotency filter) + real simulation output
4. Production war stories
5. Interview questions (60) with follow-ups and wrong answers
6. One-page cheat sheet

---

# PART 1 - 60-second mental model

**Microservices = you trade in-process method calls (fast, atomic, reliable) for network calls (slow, partial-failure, unordered) in exchange for independent deployability and team autonomy.** Everything else in this note is damage control for that trade.

```
Monolith call:   orderService.pay(order)      -> nanoseconds, same transaction, throws or returns
Network call:    POST /payments               -> can be: refused, slow, timed-out-but-succeeded,
                                                 executed twice, executed once but reply lost,
                                                 executed while the caller already gave up
```

Analogy - a restaurant chain vs one kitchen:
- Monolith = one kitchen. Chef shouts "table 4 needs fries", it happens. If the kitchen floods, everyone goes home.
- Microservices = separate stalls (grill, fryer, payments desk) each with its own cash box (database). To sell a combo meal you must coordinate stalls by passing tickets (messages). A stall may be closed, slow, or lose a ticket. So you need: a timeout on how long you wait (timeout), a rule about asking again (retry + idempotency), a sign "fryer closed, stop asking" (circuit breaker), a limit on how many customers queue at one stall (bulkhead), and a refund procedure if the combo cannot be completed (saga compensation). You cannot lock all stalls at once (no distributed ACID).

The five ideas that answer 80% of interviews:
1. **Own your data** (database per service) - therefore no joins, no cross-service ACID.
2. **Every remote call needs a timeout**, and every retry needs idempotency.
3. **Fail fast and isolate** (circuit breaker, bulkhead) so one slow dependency cannot exhaust all your threads.
4. **Consistency is eventual**: Saga for multi-service workflows, **Transactional Outbox** to publish events atomically with the DB write.
5. **You cannot debug what you cannot see**: correlation/trace IDs, metrics (RED), SLOs.

---

# PART 2 - Deep explanation

## 2.1 When microservices are (and are not) the right choice

**Conway's law:** "Organizations design systems that mirror their communication structure." If three teams must coordinate to change one deploy unit, you get a slow monolith. Inverse Conway maneuver: shape teams first (Team Topologies: stream-aligned teams owning a business capability end-to-end, with a platform team providing paved-road CI/CD, K8s, observability), and let service boundaries follow.

**What you actually buy:**
- Independent deploy and release cadence per team (the real prize).
- Independent scaling of hot paths (search vs. admin).
- Fault isolation (if bulkheaded properly), technology heterogeneity (a cost as much as a benefit).

**What you pay:**
- Network: latency, partial failure, serialization, versioning.
- Data: no joins, no ACID across services, eventual consistency.
- Ops: N pipelines, N dashboards, N on-call surfaces, service discovery, config, secrets, tracing, mesh.
- Cognitive: end-to-end debugging spans repos and teams. Local dev needs docker-compose of 12 things.

**Decision heuristics**
| Situation | Recommendation |
|---|---|
| Startup, 1-2 teams (< ~15 devs), domain still changing | Modular monolith |
| Many teams blocked by one release train, clear domain seams | Extract services along seams |
| One hot component needs 50x scale or different runtime (e.g. video encoding) | Extract just that one |
| No CI/CD, no monitoring, no on-call culture | Do NOT start microservices |
| Strong cross-entity transactional invariants everywhere (core ledger) | Keep together; consistency boundary = service boundary |

**Modular monolith first:** one deployable, but strict module boundaries (Java modules / Spring Modulith / separate Gradle modules, package-private internals, module-owned tables, communication via published interfaces or in-process events). If module boundaries survive a year without cross-module joins, extraction later is mostly mechanical. If you cannot keep boundaries in one process, network calls will not fix it - you will get a *distributed monolith*.

**The 8 fallacies of distributed computing** (Deutsch/Gosling): the network is reliable; latency is zero; bandwidth is infinite; the network is secure; topology does not change; there is one administrator; transport cost is zero; the network is homogeneous. Each pattern below targets one: reliable -> timeouts/retries/idempotency; latency -> latency budgets/async; topology changes -> discovery; secure -> mTLS/JWT.

**Cost-of-distribution rule of thumb:** an in-process call ~ 10^-7 s; same-AZ HTTP call ~ 1-5 ms plus tail latency; cross-region ~ 50-150 ms. A page that fans out to 8 services in sequence at p99 100 ms each is 800 ms before any work.

## 2.2 Decomposition

### Bounded contexts (DDD)
Same word, different model: "Product" in Catalog (description, images), Inventory (SKU, quantity), Pricing (price rules), Shipping (weight, dimensions). Each bounded context has its **own model and its own ubiquitous language**; forcing one giant `Product` class is the coupling. Start from **business capabilities** and **event storming** (orange stickies = domain events like `OrderPlaced`, `PaymentAuthorized`); clusters of events + commands + the aggregates that handle them suggest contexts.

### Aggregates
An **aggregate** = a cluster of entities changed together under one **consistency boundary**, with one **aggregate root**. Rules: reference other aggregates *by ID only*; one transaction modifies one aggregate; use eventual consistency between aggregates. `Order` (+ `OrderLine`s) is an aggregate; `Customer` is another - Order stores `customerId`, not a `Customer` object. Service boundary >= aggregate boundary (a service holds one or more aggregates, never half of one).

### Context mapping (how contexts relate)
- **Customer/Supplier** - upstream plans with downstream input.
- **Conformist** - downstream accepts upstream model as is.
- **Anti-Corruption Layer (ACL)** - translation layer protecting your model from a legacy/external model (crucial in migrations).
- **Open Host Service + Published Language** - upstream offers a stable documented API/schema for many consumers.
- **Shared Kernel** - two contexts share a small model; dangerous, use rarely.
- **Separate Ways** - no integration at all (often the right answer).

### Database per service; joins and reporting
Rule: **only the owning service touches its tables**; others use its API or events. Shared DB = distributed monolith (schema change breaks N teams, no independent deploy).

How to answer "but I need a join":
1. **API composition:** an aggregator (BFF/gateway/composite service) calls Order and Customer, joins in memory. Fine for small result sets; bad for large pagination/sorting across services.
2. **CQRS read model / materialized view:** consume events from both services and keep a denormalized read table (e.g. `order_view` with customer name) in a query service. Eventually consistent, fast.
3. **Data replication by events:** Order service keeps a small local copy of the customer fields it needs (`customer_snapshot`) updated from `CustomerChanged` events. Denormalize on purpose.
4. **Reporting:** do not query OLTP databases across services. Stream (CDC/Kafka) into a warehouse/lake (BigQuery, Snowflake, ClickHouse) and report there.

### Shared-library coupling
A "common" jar containing DTOs/entities used by all services means one change forces N redeploys and version lockstep - a hidden monolith. Acceptable shared code: technical, stable, non-domain (logging/tracing config starter, security starter, error-format). Not acceptable: shared domain entities, shared repositories, shared "model" jar. Prefer **each service defines its own DTO** for what it consumes (tolerant reader), published schemas (OpenAPI/Avro/Protobuf) instead of shared classes.

### Distributed monolith - smells
- Deploying service A always requires deploying B and C (lockstep releases).
- Shared database or shared "common-entities" jar.
- Long synchronous call chains A -> B -> C -> D for one user request.
- One service down = whole product down (no fallback/degradation).
- Chatty services: N+1 remote calls per page.
- Cannot run/test a service without starting 10 others.
- Circular dependencies between services.
- Same team owns and must change all of them together (boundaries wrong).

## 2.3 Communication styles

| | REST/HTTP+JSON | gRPC (HTTP/2 + Protobuf) | Messaging (Kafka/RabbitMQ) |
|---|---|---|---|
| Coupling | Temporal (both up) | Temporal + stricter schema | Loose temporal; coupled by message schema |
| Latency | Medium | Low (binary, multiplexing) | Async - not request/response |
| Contract | OpenAPI (optional) | `.proto` (mandatory, codegen) | Event schema (Avro/JSON schema/registry) |
| Streaming | SSE/WebSocket workaround | Native bi-di streaming | Native (log/queue) |
| Browser-friendly | Yes | Needs grpc-web | No |
| Best for | Public/external APIs, simple CRUD | Internal low-latency service-to-service | Workflows, fan-out, decoupling, buffering spikes |
| Failure mode | Timeouts, retries needed | Deadlines propagate (nice), retries need care | Duplicates, ordering, poison messages, lag |

Rules of thumb: **commands that need an immediate answer** -> sync; **facts that happened** ("OrderPlaced") -> events; **long-running workflows** -> events + saga. Prefer async between services whenever the user does not need the result in the same HTTP response.

### Sync chains: availability math
If a request needs services A -> B -> C -> D sequentially and each has availability 0.999, the chain has **0.999^4 ~ 99.6%** (availabilities multiply). Real output from the simulation in Part 3:
```
 1 services @99.9% each -> 99.9000%  (downtime/yr ~ 8.8 h)
 3 services @99.9% each -> 99.7003%  (downtime/yr ~ 26.3 h)
 5 services @99.9% each -> 99.5010%  (downtime/yr ~ 43.7 h)
10 services @99.9% each -> 99.0045%  (downtime/yr ~ 87.2 h)
 5 services @99.5% each -> 97.525%
```
So to keep a 99.9% SLO you cannot call five 99.9% services in a required sync chain. Fixes: cut hops, make some calls async, make non-essential ones optional with fallback (recommendations down -> still show the product), cache, and use redundancy inside each service.

### Latency budget and timeout ordering
Total SLA for the user request, e.g. 2 s at the gateway. **Timeouts must strictly decrease down the call chain**, otherwise the caller gives up (and retries!) while the callee is still working - wasted work and duplicated effects.
```
client           : 3000 ms
gateway          : 2500 ms   (leaves room for response & network)
order-service    : 2000 ms   (its own budget)
  -> payment     : 1200 ms   (per attempt; with 2 attempts ~ 2400 > 2000: the retry budget must fit too!)
     -> bank-api :  800 ms
```
The retry rule: `attempts x per-attempt-timeout + backoff waits <= caller's budget`. Propagate the *remaining* deadline (gRPC does; in HTTP use a header like `X-Request-Deadline` by convention) so deep services can refuse work that is already hopeless.

## 2.4 API design between services

**Backward-compatible evolution (Postel / tolerant reader):**
- Adding an optional response field: safe. Removing/renaming/retyping: breaking.
- Consumers must ignore unknown fields (`spring.jackson.deserialization.fail-on-unknown-properties=false` is the Boot default; do not turn it on).
- Adding a required request field: breaking - make it optional with a default first.
- Never change the meaning of an existing field or enum value; adding a new enum value can break strict consumers - document "unknown value" handling.
- Use **expand/contract** for APIs too: add new alongside old, migrate consumers, then remove old.
- Deprecation: `Deprecation`/`Sunset` headers, usage metrics per consumer (you can only remove what nobody calls).

**Versioning:** URI (`/v2/orders`), header, or media type. Best practice is to *avoid* new major versions by evolving compatibly; when you must, run v1 and v2 side by side and track which clients use v1. Versioning events: put `eventType` + `schemaVersion` in the envelope, evolve with schema registry compatibility rules (BACKWARD/FORWARD/FULL, see `Kafka.md`).

**Consumer-driven contract testing (CDC):** instead of a slow shared end-to-end environment, each *consumer* records what it needs from the provider (a "contract"), and the *provider's* build verifies it can still satisfy every consumer's contract.
- **Pact:** consumer test generates a pact JSON (request -> expected response, with matchers) and publishes it to a Pact Broker; provider verifies pacts in its CI; `can-i-deploy` gates release. Works polyglot; also supports message pacts (events).
- **Spring Cloud Contract:** contracts (Groovy/YAML DSL) live with the provider; provider build generates tests and a **stubs jar**; consumers run against WireMock stubs via `@AutoConfigureStubRunner`. Good for Spring-only estates.
- Result: provider can refactor freely as long as contracts pass; breaking a consumer fails the provider build *before* deploy.

## 2.6 API Gateway (Spring Cloud Gateway)

```
Client -> [ API Gateway ] -> order-service
             | auth (JWT validate) | rate limit | routing | TLS termination | CORS | request logging/trace start
```

**Spring Cloud Gateway (WebFlux/Netty)** concepts:
- **Route** = id + destination URI + predicates + filters.
- **Predicate** = match condition (`Path=/orders/**`, `Method=GET`, `Header=X-Tenant`, `Host=`, `Weight=group,80`).
- **Filter** = pre/post processing (`StripPrefix=1`, `AddRequestHeader`, `RequestRateLimiter`, `CircuitBreaker`, `Retry`, `TokenRelay`, `RewritePath`).
- **Global filters** apply to all routes (auth, correlation ID).
- `lb://order-service` uses Spring Cloud LoadBalancer + discovery.
- Since Spring Cloud 2023.0 there is also **Gateway Server MVC** (servlet-based) for teams not on WebFlux. In Spring Cloud **2025.0** artifacts were renamed (`spring-cloud-starter-gateway-server-webflux`, properties under `spring.cloud.gateway.server.webflux.*`; the old names are deprecated). Check your release train.

**Responsibilities that belong in the gateway:** TLS termination, JWT signature/expiry validation (auth offload), coarse authorization (scopes), rate limiting, routing/versioning, CORS, request size limits, correlation ID injection, sometimes response caching.
**Do NOT put in the gateway:** business logic, fine-grained domain authorization, orchestration of multi-service workflows (it becomes an "ESB in disguise" and a change bottleneck).

**Rate limiting with Redis token bucket** (`RequestRateLimiter` + `RedisRateLimiter`): each key (user/API key/IP from a `KeyResolver` bean) has a bucket with capacity `burstCapacity`, refilled at `replenishRate` tokens/sec; each request costs `requestedTokens` (default 1). Empty bucket -> HTTP 429. State lives in Redis (a Lua script makes check-and-decrement atomic) so all gateway instances share limits. Token bucket permits bursts up to capacity but bounds the long-run rate. If Redis is down the limiter *fails open* by default (allows requests) - a deliberate default; know it.

**BFF (Backend for Frontend):** one gateway/aggregator per client type (web, mobile, partner). It shapes payloads (mobile gets slim JSON), aggregates calls, and is owned by the frontend team. Avoids a single "generic API" that fits nobody.

**Gateway vs service mesh:** gateway = **north-south** (external client to cluster), edge concerns (public auth, rate limits, API products). Mesh = **east-west** (service to service) concerns: mTLS, retries, traffic splitting, telemetry, done by sidecars. They complement each other; do not implement business retries in both layers (retry amplification, see 2.8).

## 2.7 Service discovery and configuration

**Problem:** instances are ephemeral (autoscaling, redeploys) so IP:port change constantly.
- **Client-side discovery:** the caller queries a registry (Eureka/Consul), gets the instance list, and load-balances itself (Spring Cloud LoadBalancer via `lb://name` or `@LoadBalanced` `RestClient.Builder`/`WebClient.Builder`). Pros: smart client-side balancing; cons: every client embeds the logic, registry becomes critical, stale entries.
- **Server-side discovery:** the caller calls a stable virtual address; a load balancer/router resolves (AWS ALB, **Kubernetes Service**). Pros: language agnostic, no client logic.

**Eureka** (`spring-cloud-starter-netflix-eureka-server`, `@EnableEurekaServer`): services register with heartbeats (default every 30 s; eviction after 90 s missing heartbeats); clients cache the registry and refresh every 30 s. Eureka is AP: it prefers availability and can serve stale entries, and has "self-preservation mode" (stops evicting when too many heartbeats are missing, assuming a network problem). Hence stale instances are normal and **clients still need timeouts + retries + circuit breakers**.

**On Kubernetes prefer native:** a `Service` gives a stable DNS name (`http://payment-service.shop.svc.cluster.local` or just `http://payment-service` in-namespace); kube-proxy/endpoints handle instance sets; readiness probes remove unhealthy pods. Running Eureka on K8s duplicates what the platform does (and Eureka registers pod IPs that may go stale). Use Eureka/Consul when you are on VMs or multi-platform. Spring Cloud Kubernetes exists (discovery via the API server, ConfigMap/Secret property sources) but plain DNS + `RestClient` is the simplest.
Caveat: K8s Services load-balance per **connection** (L4). Long-lived HTTP/2 or gRPC connections stick to one pod -> use a headless service + client-side LB, or a mesh/L7 proxy.

**Configuration**
- **Spring Cloud Config Server** (`@EnableConfigServer`, Git-backed): clients use `spring.config.import=optional:configserver:http://config:8888` (Boot 2.4+ style; the old bootstrap.yml mechanism is legacy). Supports profiles/labels, encryption of values (`{cipher}...`). Runtime refresh: `@RefreshScope` beans + `POST /actuator/refresh`, or fan-out via Spring Cloud Bus (needs a broker).
- **Kubernetes ConfigMap / Secret:** mounted as env vars or files; simplest and platform-native. Env vars require pod restart; mounted files update eventually but Spring does not rebind by itself unless you use Spring Cloud Kubernetes reload or roll the deployment. Secrets are only base64 by default: enable encryption at rest / use an external secrets manager (Vault, AWS Secrets Manager via External Secrets Operator).
- **Guidance:** on K8s, ConfigMap/Secret + GitOps (Helm/Kustomize/Argo CD) usually replaces Config Server. Keep config immutable per release where possible (restart to change) - easier to reason about. Use feature flags (not config refresh) for runtime toggles.

## 2.8 Resilience in depth

### Timeouts (the most important one)
No timeout = infinite wait = thread stuck forever. Two kinds:
- **Connect timeout:** time to establish TCP (+TLS). Keep short (100-500 ms in-DC): if you cannot connect quickly, the host is down/overloaded.
- **Read (socket) timeout:** time waiting for data after the request was sent. Set from the callee's p99/p99.9 + margin, within the latency budget.
Many HTTP clients default to infinite or 30-60 s; *always set explicitly*. Also a **connection-pool acquire timeout**, and DB query timeouts. Note: a timeout does NOT cancel the work on the server; the request may still complete (see 2.14).

Spring Boot 3 examples (details in Part 3): `RestClient` with `JdkClientHttpRequestFactory`/`HttpComponentsClientHttpRequestFactory` timeouts (Boot 3.4+ also has `spring.http.client.connect-timeout` / `read-timeout`), OpenFeign `spring.cloud.openfeign.client.config.default.connectTimeout/readTimeout`, WebClient via Reactor Netty `responseTimeout`.

### Retries
Retry only **transient** failures (timeouts, 503, connection reset), not 4xx/business errors, and only if the operation is **idempotent** (GET/PUT/DELETE by nature; POST only with an idempotency key, see 2.14).
- **Exponential backoff:** delay = base x 2^(n-1), capped.
- **Jitter:** randomize so clients do not retry in lockstep. Full jitter = random(0, exp); equal jitter = exp/2 + random(0, exp/2).
- **Retry storm / amplification:** if each of 3 layers makes 3 attempts, the deepest service sees up to 3^3 = 27 calls per user request; when it is *already overloaded* retries multiply load and keep it down (metastable failure). Countermeasures: retry at **one layer only** (usually the outermost that owns the idempotency key), **retry budget** (retries <= ~10% of requests), circuit breaker outside/around retries, honor `Retry-After`, jitter, and deadlines.

Real simulation output (Part 3.9.2):
```
Backoff schedule (base=100ms, multiplier=2, cap=5000ms):
attempt  no-jitter  full-jitter[0,exp]     equal-jitter[exp/2,exp]
1        100        72                     84
2        200        61                     127
3        400        266                    380
4        800        295                    510
5        1600       741                    1426
...
fixed 200ms, no jitter: arrivals per 100ms bucket = [0, 0, 1000, 0, 1000, 0, 1000, 0, ...]  peak/bucket=1000
exp backoff, full jitter: arrivals = [556, 693, 293, 333, 305, 180, 137, 134, ...]        peak/bucket=693
```
(1000 clients that all failed at t=0. Without jitter the whole herd re-arrives in the same 100 ms bucket three times; with jitter the load is spread and decays.)
Amplification: 1 layer 3x, 2 layers 9x, 3 layers 27x, 4 layers 81x.

### Circuit breaker
Idea: stop calling a dependency that is failing so you (a) fail fast instead of tying up threads for the timeout duration, and (b) give it room to recover.

```
                failure rate >= threshold  OR  slow-call rate >= threshold
   +--------+   (over the sliding window, and >= minimumNumberOfCalls)   +--------+
   | CLOSED | ---------------------------------------------------------> |  OPEN  |
   |        |                                                            | calls  |
   | calls  |                                                            | rejected|
   | pass,  |                                                            | with   |
   | recorded|                                                           | CallNotPermittedException
   +---^----+                                                            +---+----+
       |                                                                     | waitDurationInOpenState elapsed
       | failure rate of the permittedNumberOfCallsInHalfOpenState           | (auto, if automaticTransitionFromOpenToHalfOpenEnabled;
       | trial calls  < threshold                                            |  otherwise on the next call)
       |                         +-------------+                             |
       +-------------------------| HALF_OPEN   |<----------------------------+
                                 | N trial calls|
                                 +------+------+
                                        | failure rate of trial calls >= threshold
                                        v
                                      OPEN again (wait restarts)
```
Also two special states: `DISABLED` (always allow) and `FORCED_OPEN` (always reject), used for ops/testing.

**Resilience4j configuration explained**
| Key | Meaning | Default |
|---|---|---|
| `slidingWindowType` | `COUNT_BASED` (last N calls) or `TIME_BASED` (last N seconds) | COUNT_BASED |
| `slidingWindowSize` | N calls or N seconds | 100 |
| `minimumNumberOfCalls` | Do not compute a rate before this many calls (prevents tripping on 1 failure of 1 call) | 100 |
| `failureRateThreshold` | % failures that opens the circuit | 50 |
| `slowCallDurationThreshold` | A call longer than this counts as *slow* | 60 s |
| `slowCallRateThreshold` | % slow calls that opens the circuit | 100 (i.e. off unless lowered) |
| `waitDurationInOpenState` | how long to stay OPEN before trying HALF_OPEN | 60 s |
| `permittedNumberOfCallsInHalfOpenState` | trial calls allowed in HALF_OPEN | 10 |
| `automaticTransitionFromOpenToHalfOpenEnabled` | move to HALF_OPEN by a timer instead of waiting for next call | false |
| `maxWaitDurationInHalfOpenState` | cap on time in HALF_OPEN (0 = wait forever) | 0 |
| `recordExceptions` / `ignoreExceptions` | which exceptions count as failures / are ignored (neither success nor failure) | all exceptions count |
| `recordFailurePredicate` | custom predicate, e.g. count only 5xx | - |

Tuning notes: count-based windows are simple; time-based windows suit variable traffic. Pick `minimumNumberOfCalls` so low-traffic services do not flap. **Business exceptions (404, validation 400) must not count as failures** - use `ignoreExceptions`/`recordExceptions`. Slow-call threshold slightly above your normal p99 and below your read timeout, so the breaker trips on *slowness before timeouts pile up*. Per-dependency (and sometimes per-endpoint) breakers - not one global breaker.

The Java simulation in Part 3.9.1 is a small count-based breaker with a fake clock; the real output there matches this diagram.

### Bulkhead
Named after ship compartments: limit resources per dependency so one slow dependency cannot drain a shared pool.
- **SemaphoreBulkhead:** max N concurrent calls executed on the *caller's* thread (`maxConcurrentCalls`, `maxWaitDuration` to wait for a permit; else `BulkheadFullException`). Cheap; works with blocking and reactive; no thread hand-off.
- **ThreadPoolBulkhead:** dedicated thread pool + bounded queue per dependency (`coreThreadPoolSize`, `maxThreadPoolSize`, `queueCapacity`, `keepAliveDuration`); the call runs on another thread, so the *caller* thread is free. Annotated methods must return `CompletableFuture`. Costs context switching, and thread-locals (MDC, security context) do not propagate unless you handle it. With **Java 21 virtual threads**, the semaphore bulkhead is generally preferred (virtual threads are cheap, but the *downstream* still needs a concurrency cap).

### Rate limiter (Resilience4j)
Client-side/self-protection: `limitForPeriod` permits per `limitRefreshPeriod`; callers wait up to `timeoutDuration` for a permit, else `RequestNotPermitted`. Use it to respect a third-party quota (e.g. bank API 50 rps) and to protect a fragile downstream. Different from the gateway's per-client server-side limiter.

### Time limiter
`timeoutDuration` and `cancelRunningFuture`. Works with `CompletableFuture`/reactive types (a plain blocking method cannot be interrupted by an annotation; set client read timeout instead - that is the real fix for blocking clients). Do not rely on `@TimeLimiter` alone on synchronous `RestClient` code.

### Aspect order (Resilience4j Spring Boot default)
The default nesting is (outermost first):
```
Retry ( CircuitBreaker ( RateLimiter ( TimeLimiter ( Bulkhead ( yourFunction ) ) ) ) )
```
Consequences:
- Retry is outermost: **each retry attempt goes through the circuit breaker**, and every attempt (including failures) is recorded by it. When the breaker opens, `CallNotPermittedException` is thrown *inside* the retry; make sure Retry ignores it (`ignoreExceptions`) or it will futilely retry against an open circuit (still cheap, but wasteful with waits).
- Bulkhead is innermost: it caps concurrency of the actual call.
- TimeLimiter inside the breaker: a time-limiter timeout counts as a failure for the breaker.
The order is configurable via properties (e.g. `resilience4j.circuitbreaker.circuitBreakerAspectOrder`, `resilience4j.retry.retryAspectOrder`; a *lower* number means higher precedence, i.e. more outer) - if you need something else, verify against the docs for your version. Decide deliberately; e.g. put the breaker *outside* Retry if you want a whole retry sequence to count as one call.

### Fallback design
A fallback must be **cheaper and more reliable than the primary**, and must not call the same failing dependency.
1. **Stale cache:** serve last-known-good (product price from 5 minutes ago). Mark it stale.
2. **Default value:** empty recommendations list, "unknown" shipping estimate.
3. **Graceful degradation:** hide the feature (no reviews section); checkout still works.
4. **Queue for later:** accept the order as PENDING, process asynchronously, notify the user.
5. **Fail with a clear error** (503 + `Retry-After`) when no honest fallback exists. **Never fake success for payments/writes.**
Fallback method signature must match + an exception parameter; Resilience4j picks the fallback with the most specific exception type.

### Load shedding and backpressure
- **Load shedding:** when saturated, reject early (429/503) rather than queue unboundedly - prioritize critical traffic (checkout > browse). Bounded queues, concurrency limits (bulkhead), gateway limits, `server.tomcat.threads.max` and `accept-count` tuned, Tomcat/Undertow thread exhaustion is the classic failure.
- **Backpressure:** slow producers when consumers cannot keep up: Reactor's `onBackpressure*`, bounded channels, Kafka `max.poll.records` + consumer lag monitoring, RabbitMQ `prefetch`. Unbounded queues just move the failure to out-of-memory later and add latency.

### Cascading failure - the story and where each pattern stops it
```
t0  inventory-service DB gets slow (a bad query)
t1  inventory calls now take 20 s instead of 50 ms
t2  order-service threads block waiting on inventory (no/long timeout)
t3  order-service Tomcat pool (200 threads) fully blocked -> even /health and unrelated endpoints time out
t4  gateway retries -> 3x load on order-service; clients retry too
t5  order-service marked unready by K8s -> traffic shifts to remaining pods -> they die faster
t6  everything upstream (cart, web) is down; inventory recovers but is instantly re-flooded by retries -> cannot recover
```
Which pattern breaks the chain: **timeout** (t2: thread freed after 500 ms) -> **bulkhead** (t3: only N threads can be stuck on inventory; others remain) -> **circuit breaker** (t2-t3: stop calling; fail fast) -> **fallback** (degrade: "stock unknown, allow order as pending") -> **retry with jitter/budget + single layer** (t4) -> **load shedding** (protect order-service itself) -> **circuit half-open** with few trial calls (t6: gentle recovery) -> **autoscaling with limits** and **readiness that does not depend on downstreams** (t5).

## 2.9 Data consistency: no distributed ACID

Each service has its own DB. To update Order DB and Payment DB atomically you would need **2PC / XA**: a coordinator asks all participants to *prepare* (durably promise), then *commit*. Downsides: blocking (participants hold locks while waiting for the coordinator; coordinator crash leaves in-doubt transactions), latency (multiple round trips, slowest participant), availability (the product of all participants' availability again), not supported by many modern stores (Kafka, MongoDB partly, REST APIs, SaaS), and tightly couples services. So: **relax to eventual consistency using Saga + Outbox.**

### Saga - definition
A saga is a sequence of **local ACID transactions**, one per service; after each, the next is triggered. If a step fails *business-wise*, **compensating transactions** semantically undo the earlier steps (refund, release stock, cancel order). It is **ACD without I**: no isolation - other transactions see intermediate states, so you must design for that (semantic locks, see below).

### Choreography vs orchestration
**Choreography (event-driven, no coordinator):** each service reacts to events and emits new ones.
```
OrderService   --OrderCreated-->   PaymentService --PaymentSucceeded--> InventoryService --StockReserved--> OrderService(confirm)
                                          | PaymentFailed                     | StockRejected
                                          v                                   v
                                    OrderService(cancel)     PaymentService(refund) --PaymentRefunded--> OrderService(cancel)
```
Pros: no central component, loose coupling, simple for 2-4 steps. Cons: the flow exists only implicitly across services (hard to see, hard to change), cyclic dependencies creep in, hard to answer "where is order 123 now?", compensation logic scattered.

**Orchestration (a coordinator owns the flow):** an `OrderSagaOrchestrator` (in Order service or its own) sends *commands* and reacts to *replies*, persisting saga state.
Pros: explicit flow in one place, easy monitoring/timeouts/retries, participants know nothing about each other. Cons: extra component (must be reliable and idempotent), risk of logic-heavy "god orchestrator", coordinator coupling.
Rule: choreography for short, stable flows; orchestration when > ~3-4 steps, branching, timeouts, or when the business wants visibility.
Orchestration engines: **Camunda 8 (Zeebe)/Camunda 7**, **Temporal** (durable code-as-workflow with retries/timers built in), **Axon Framework** (sagas + event sourcing), **AWS Step Functions**, Netflix Conductor, Spring State Machine (in-process state modelling). A hand-rolled orchestrator (Part 3.4) is fine for one or two sagas; adopt an engine when timers/retries/versioning multiply.

### Full example: order -> payment -> inventory (orchestrated)

Steps (transactions) and compensations:
| # | Step (service) | Type | Compensation |
|---|---|---|---|
| T1 | Create order `PENDING` (Order) | compensatable | Cancel order (`CANCELLED`) |
| T2 | Reserve stock (Inventory) | compensatable | Release stock |
| T3 | Authorize/charge payment (Payment) | **pivot** (point of no return, see below) | Refund (only if it already captured) |
| T4 | Confirm order `CONFIRMED` (Order) | **retryable** (must eventually succeed) | none - retry forever |

Design choice: order the steps so that the **most likely to fail / cheapest to undo go first**; put the **pivot** in the middle and retryable steps after it. Here reserving stock (cheap to undo) before charging money (costly, external) avoids refund noise.

**Pivot transaction:** the step after which the saga can no longer be rolled back - either it is the last compensatable step's successor whose success means "we go forward no matter what". Steps *before* the pivot are compensatable; steps *after* are **retryable** (guaranteed to complete eventually through retries, idempotent). Some texts define the pivot as the transaction that neither has a compensation nor is retryable-only; state your definition when asked: "the go/no-go step - if it succeeds, the saga is committed to completion; if it fails, we compensate the previous steps".

**Happy path**
```
Client    Order(Orchestrator)        Inventory              Payment
  | POST /orders  |                       |                     |
  |-------------->| 1. tx: order=PENDING, saga=STARTED, outbox(ReserveStock)   [one local tx]
  |<--202 + id----|                       |                     |
  |               |--ReserveStockCmd----->| 2. tx: reserve; outbox(StockReserved)
  |               |<--StockReserved-------|                     |
  |               | 3. tx: saga=STOCK_RESERVED, outbox(ChargePayment)
  |               |--ChargePaymentCmd------------------------->| 4. tx: charge (idempotent by orderId); outbox(PaymentSucceeded)
  |               |<--PaymentSucceeded-------------------------|
  |               | 5. tx: order=CONFIRMED, saga=COMPLETED, outbox(OrderConfirmed)
  |--GET /orders/id (poll) or push--> CONFIRMED
```

**Failure branch A - stock unavailable (fail at T2)**
```
Order --ReserveStockCmd--> Inventory: not enough stock
Order <--StockRejected------
Order: tx: order=REJECTED, saga=COMPENSATED   (nothing to undo: only T1 done; T1's compensation = cancel)
```

**Failure branch B - payment declined (fail at T3, business failure)**
```
Order --ReserveStockCmd--> Inventory (stock reserved)
Order --ChargePaymentCmd--> Payment: card declined
Order <--PaymentFailed-----
Order: saga=COMPENSATING ; outbox(ReleaseStockCmd)
Order --ReleaseStockCmd--> Inventory: release (idempotent)
Order <--StockReleased----
Order: order=CANCELLED, saga=COMPENSATED
```

**Failure branch C - payment timeout, outcome UNKNOWN (the nasty one)**
```
Order --ChargePaymentCmd--> Payment ... no reply within timeout
Order: DO NOT assume failure. saga=PAYMENT_UNKNOWN.
   1) retry same command with same idempotency key (orderId) -> Payment returns the original result (idempotent)
   2) or query: GET /payments?orderId=... (reconciliation)
   3) if Payment says CAPTURED -> continue forward; if NOT_FOUND after N tries and timeout -> send CancelPaymentCmd (idempotent void) then compensate stock
```

**Failure branch D - compensation itself fails**
Compensations must be **retryable and idempotent** and are retried until success (with backoff); after a bounded number of attempts, park the saga in `NEEDS_MANUAL_ATTENTION` and alert. Compensations never "fail permanently" by design - if "refund" can be rejected, the business needs a manual process.

**Failure branch E - orchestrator crashes mid-saga**
Because every state transition + outgoing command is written in **one local transaction** (state row + outbox row), after restart it reloads `saga` rows in non-terminal states and continues (or a timeout sweeper re-sends). That is why saga state must be persisted.

### Saga design topics
- **Semantic locks:** since there is no isolation, mark records with a state that other transactions respect: order `PENDING` (not shippable), account "hold" instead of debit, `reservedQty` separate from `availableQty`. Other flows either wait, fail, or treat pending as not-yet-real.
- **Other countermeasures for lack of isolation** (from Richardson): *commutative updates*, *pessimistic view* (reorder steps to reduce risk), *reread value* (optimistic check before update), *version file*, *by value* (high-risk requests use distributed transaction/locking).
- **Timeouts:** every waiting state has a deadline (`saga.deadline_at`). A sweeper (ShedLock-guarded `@Scheduled`) picks up expired sagas: retry the step, query status, or compensate. Without this, lost messages create stuck sagas forever.
- **Idempotent handlers:** each participant dedups by `messageId` (inbox table) *and* makes business operations naturally idempotent (charge keyed by `orderId`).
- **Out-of-order / duplicate events:** e.g. `PaymentFailed` arrives after `PaymentSucceeded` (retry raced). Use saga **state machine guards**: an event is valid only in specific states; otherwise log and ignore (or raise an alert). Use version/sequence numbers per aggregate; consumer ignores lower versions.
- **Compensation is semantic, not a rollback:** refund != un-charge (the customer saw the charge on the statement; a fee may apply). Emails already sent cannot be unsent - place such non-compensatable steps last (they become the retryable steps).
- **Saga state persistence:** table `saga_instance(id, type, correlation_id/order_id, state, current_step, payload_json, deadline_at, version, created_at, updated_at)`, plus optional `saga_step_log` for audit. Optimistic locking (`@Version`) prevents two events advancing the same saga concurrently.

## 2.10 The dual-write problem and the Transactional Outbox

**Dual write:** one operation writes to the DB *and* publishes to a broker. Two different systems, no shared transaction.
```
1. save(order)            OK
2. kafka.send(OrderCreated)  -> crash/broker down/timeout
=> DB has the order, no event: downstream never learns (state divergence, silent)
Swap order (send first, then save): event published, DB save fails => phantom event for a non-existent order.
Doing it inside @Transactional does not help: the broker send is not part of the DB transaction (commit can fail AFTER the send).
```
**Transactional Outbox:** write the event to an `outbox` table **in the same local DB transaction** as the business change. A separate *relay* reads the table and publishes to the broker. The atomicity problem is reduced to "DB commit", which is ACID.
```
             one local DB transaction
   +--------------------------------------------+
   | INSERT INTO orders ...                     |
   | INSERT INTO outbox(id, aggregate_id, type, payload, created_at, published_at=NULL) |
   +--------------------------------------------+ COMMIT
                        |
        (A) Polling publisher                 (B) CDC (Debezium)
   @Scheduled: SELECT unpublished        reads DB transaction log (WAL/binlog)
   ORDER BY id LIMIT n FOR UPDATE        of the outbox table, emits change events
   SKIP LOCKED -> send to Kafka ->       to Kafka (EventRouter SMT routes by
   mark published                        aggregate type/id) - no polling, low latency
                        |
                    Kafka topic (key = aggregate_id)  -->  consumer (inbox dedup) -> business effect
```
**Polling publisher vs CDC:**
| | Polling | Debezium CDC |
|---|---|---|
| Complexity | Low: just Spring code | Needs Kafka Connect + Debezium + DB log config |
| Latency | Poll interval (100 ms - 1 s) | Near real time |
| DB load | Repeated queries, index on `published_at` | Reads the log, minimal |
| Ordering | Keep `ORDER BY id`, single publisher per shard or key-partitioned | Log order preserved |
| Cleanup | Delete/archive published rows | Can delete right after insert (Debezium sees the insert in the log) |

**Delivery guarantee = at-least-once.** The relay may crash after `send` but before marking published -> event sent twice. So **consumers must be idempotent**: an **inbox / processed_messages table** with primary key `message_id`; the consumer inserts the id and does the business change in the *same local transaction*; a duplicate insert violates the PK -> skip.
**Ordering:** the outbox `id` is monotonic per DB; publish with **Kafka key = aggregateId** so all events of one aggregate land in one partition in order (see `Kafka.md`). With multiple polling publishers, use `FOR UPDATE SKIP LOCKED` carefully - rows can be published out of order across instances; per-aggregate ordering then needs partitioned claiming (e.g. `hash(aggregate_id) % N`) or a single publisher. Beware auto-increment gaps: a transaction with a lower id can commit *after* one with a higher id, so a poller that tracks "last seen id" may skip it; use an `published_at IS NULL` flag instead of a cursor (or CDC).
Publish the **event, not the entity**: contain enough data (`eventId`, `aggregateId`, `type`, `version`, `occurredAt`, payload) so consumers need not call back.

## 2.11 CQRS and event sourcing

**CQRS:** separate the **write model** (commands, enforcing invariants, normalized) from the **read model(s)** (queries, denormalized, tailored). They can share a DB (simple CQRS) or use different stores updated by events (Elasticsearch for search, Redis for hot views).
Example: e-commerce `order` write DB (Postgres, normalized: orders, order_lines). The "My orders" screen needs order + product name + shipment status: an `order_view` projector consumes `OrderPlaced`, `ProductRenamed`, `ShipmentDispatched` and writes one denormalized row per order into a read store. The page is one indexed read.
Pros: fast, scalable reads; independent optimization; answers the "joins across services" question. Cons: eventual consistency (read-your-writes gap), more moving parts, projections must be rebuildable and idempotent. Do not use CQRS for simple CRUD.

**Event sourcing (ES):** store the *sequence of events* as the source of truth (`AccountOpened`, `MoneyDeposited(50)`, `MoneyWithdrawn(20)`); current state = fold of events; snapshots speed up loading.
Pros: complete audit log, time travel/debugging, natural events for integration (no dual write: the event store *is* the outbox), easy new projections by replay. Cons: steep learning curve, event schema evolution forever (upcasting), querying needs projections (so ES nearly always comes with CQRS), GDPR deletion is hard (crypto-shredding), replay time, eventual consistency. Frameworks: Axon (+ Axon Server), EventStoreDB, or events in a Postgres table.
ES != messaging: Kafka topics are integration logs; ES is *aggregate persistence*. Use ES where audit/history is the domain (ledger, orders lifecycle in regulated flows), not by default.

## 2.12 Eventual consistency and UX

Users notice when consistency lags. Patterns:
- **202 Accepted + status resource:** `POST /orders` returns `202` with `Location: /orders/{id}`; UI polls or receives SSE/WebSocket push for status (`PENDING` -> `CONFIRMED`).
- **Optimistic UI:** show the result immediately with a "processing" badge; reconcile on event.
- **Read-your-writes:** after a write, read from the primary/write model, or include a version/timestamp token and have the read side wait until it caught up; or route that user's reads to the write side for a few seconds (sticky).
- **Explicit states in the domain:** `PENDING`, `PROCESSING` visible to users and support.
- **Notifications** (email/push) when the async workflow ends, including failures with a clear next action.
- **Reconciliation jobs** that compare systems (payments vs orders) and fix/flag drift; a safety net for lost events.

## 2.14 Idempotency keys - design

Why: timeouts, retries, gateway retries, user double-clicks, message redelivery all cause duplicates. **Idempotent = applying the operation N times has the same effect as once.**

Design (Stripe-style `Idempotency-Key` header on non-idempotent POSTs):
1. Client generates a unique key (UUID) **per logical operation** (reused on retries of that operation, new for a new operation).
2. Server stores `(key, request_fingerprint, status, response_status, response_body, created_at)` with **PRIMARY KEY / UNIQUE on key** (scope by client/tenant: unique(`client_id`, `key`)).
3. First request: `INSERT ... status=IN_PROGRESS`. **Concurrent duplicates** race on the unique constraint: exactly one insert wins; the loser gets a duplicate-key error.
   - Loser sees `IN_PROGRESS` -> reply `409 Conflict` (or wait briefly) with `Retry-After`.
   - Loser sees `COMPLETED` -> **replay the stored response** (same status and body).
4. Same key, *different* request payload (hash mismatch) -> `422`/`400`: client bug, key reuse.
5. After processing, update the row to `COMPLETED` with the response. Ideally the business change and the idempotency row update commit in **one transaction**, so a crash cannot leave "charged but not marked". If the resource lives in another system (bank), pass the key downstream too (their idempotency).
6. **Expiry:** keep keys 24 h-7 days (Stripe: 24 h); a TTL cleanup job. `IN_PROGRESS` rows older than a lease (e.g. 60 s) are treated as abandoned/recoverable.
7. Store where the business data lives (same DB = transactional); Redis `SET key NX EX` is faster but not atomic with the business write (a crash between the write and the key set allows a duplicate).
Natural idempotency alternatives: PUT with client-chosen ID; `INSERT ... ON CONFLICT DO NOTHING`; upsert; state-machine guards (`UPDATE orders SET status='PAID' WHERE id=? AND status='PENDING'` -> 0 rows = already done).
Consumer side: inbox table (2.10). Kafka "exactly-once" (`Kafka.md`) covers Kafka-to-Kafka; a DB side effect still needs idempotency.

## 2.15 Distributed locking, scheduled jobs, distributed IDs

**Do you need a distributed lock?** Often no - prefer optimistic locking (`@Version`), unique constraints, atomic updates, or single-writer partitioning (Kafka key). Use locks only for mutual exclusion of side-effecting work.
- **DB row lock:** `SELECT ... FOR UPDATE` (pessimistic) or an advisory lock (`pg_advisory_lock`). Simple, correct if everything uses the same DB; holds a connection.
- **Redis lock:** `SET lock:key <uuid> NX PX 30000`, release with a Lua script that checks the value (never plain `DEL`, or you delete someone else's lock after your expiry). **Problem:** if the holder pauses (GC, network) beyond TTL, two holders exist. **Fencing token:** a monotonically increasing number issued with the lock; the protected resource rejects writes with an older token. **Redlock controversy:** Redis's multi-node Redlock algorithm (antirez) vs Martin Kleppmann's critique - Redlock relies on timing assumptions (bounded clock drift and pauses) and has no fencing tokens, so it is unsafe for correctness-critical mutual exclusion; fine as an *efficiency* lock (avoid duplicate work occasionally). For correctness use consensus stores (ZooKeeper/etcd with fencing) or DB constraints. Redisson provides `RLock` and, for Redlock, `RedissonRedLock`.
- **ShedLock** for `@Scheduled` in multiple instances: prevents the same scheduled task running concurrently on several pods by taking a lock in a shared store (JDBC/Redis/Mongo). `@EnableSchedulerLock(defaultLockAtMostFor = "10m")` + `@SchedulerLock(name="...", lockAtMostFor="...", lockAtLeastFor="...")` (`lockAtMostFor` protects against a crashed node holding the lock; `lockAtLeastFor` prevents rapid re-run by clock-skewed nodes). It is **not** a distributed lock library for arbitrary code, and it does not make a long job that outlives `lockAtMostFor` safe. Alternative: Kubernetes `CronJob`, Quartz clustered mode.

**Distributed ID generation**
| Approach | Pros | Cons |
|---|---|---|
| Auto-increment / sequence per service | Simple, compact, index-friendly | Single-DB bottleneck, guessable, leaks volume, not globally unique across services |
| UUIDv4 | Zero coordination | 128-bit random -> index page splits/poor locality, not sortable |
| **UUIDv7** (RFC 9562) | 48-bit ms timestamp + randomness: time-sortable, good B-tree locality, no coordination | No JDK built-in in 21 (use a library like java-uuid-generator or uuid-creator); leaks creation time |
| **Snowflake** (Twitter) | 64-bit: timestamp + worker id + sequence; sortable, compact | Need unique worker ids (config/coordination), clock going backwards must be handled |
| **DB sequence segment/hi-lo** | Fetch a block of ids (e.g. 1000) per node; few DB round trips; JPA `@SequenceGenerator(allocationSize=50)` is hi-lo-like | Gaps, ids not strictly time ordered across nodes |
| ULID/KSUID | Sortable strings | Text size |
Never derive ordering guarantees from wall-clock IDs across nodes (skew). Public IDs vs internal: expose UUIDs/opaque IDs externally to avoid enumeration.

## 2.16 Caching across services
- **Levels:** client/CDN -> gateway -> service local (Caffeine) -> shared distributed (Redis) -> DB. Local cache: fastest, but per-instance inconsistency; Redis: shared, extra hop.
- **Patterns:** cache-aside (default: read cache, on miss load + populate; write DB then evict), read-through, write-through, write-behind (risky).
- **Invalidation across services:** service A caches B's data. Options: short TTL (simple, bounded staleness), **event-driven eviction** (B publishes `ProductChanged`, A evicts), versioned keys, ETag/conditional GET (`If-None-Match` -> 304).
- **Pitfalls:** cache stampede/dogpile (many misses at once when a hot key expires; fix: request coalescing/single-flight, jittered TTL, early refresh); cache penetration (queries for non-existent keys; cache negative results, bloom filter); hot key; stale data that violates a rule (never cache authorization decisions or balances for long).
- **Caching as resilience:** the "stale cache" fallback (2.8) - keep serving last-known-good when the dependency is down, ideally with `stale-while-revalidate` semantics.
See `Spring_Caching` notes in `JavaFullstack.md` for `@Cacheable` mechanics.

## 2.17 Security between services
- **Authentication at the edge, propagation inside:** gateway validates the user's JWT (signature via JWKS, `exp`, `iss`, `aud`). Downstream services are also **resource servers** (`spring-boot-starter-oauth2-resource-server`, `http.oauth2ResourceServer(o -> o.jwt(...))`) - **never trust "it came from the gateway"** (zero trust).
- **JWT propagation (token relay):** forward the user's token so downstream authorizes per user. In Spring Cloud Gateway the `TokenRelay=` filter (with `spring-boot-starter-oauth2-client`) forwards the access token; for `RestClient`/`WebClient` add an interceptor/exchange filter that copies the `Authorization` header (or `ServerOAuth2AuthorizedClientExchangeFilterFunction`). Risks: token replay to services that do not need it, broad audience, token lifetime. Use narrow `aud`/scopes.
- **Token exchange (RFC 8693):** service swaps the incoming user token for a new token with a narrower audience/scope for the downstream (on-behalf-of); supported by Keycloak and others - reduces blast radius vs relaying the same token everywhere.
- **Service accounts / client credentials:** for calls without a user context (batch jobs, event handlers) the service authenticates as itself using OAuth2 `client_credentials` (Spring Security `OAuth2AuthorizedClientManager`), scoped minimally. Do not pass user identity via unauthenticated headers (`X-User-Id`) unless the hop is cryptographically trusted.
- **mTLS:** both sides present certificates: encrypts traffic and proves *service identity* (workload identity like SPIFFE IDs). Usually provided by a service mesh (automatic cert rotation) rather than coded in each app.
- **Zero trust:** assume the network is hostile; authenticate and authorize every call; least privilege (Kubernetes NetworkPolicies, per-service DB credentials, scoped secrets); encrypt in transit; secrets in a vault, rotate; audit logs.
- **Also:** validate input at every service, do not leak internals in errors, rate-limit at the edge, scan images/dependencies, propagate identity in async messages as a claim in the event (not a raw user token).

## 2.18 Observability

**Three pillars:** logs (what happened), metrics (aggregated numbers), traces (path and timing of one request). Add events/profiles as needed. Observability = ability to ask new questions of a running system without deploying code.

### Structured logs, correlation/trace IDs, MDC
- Log as JSON (Boot 3.4+ has built-in structured logging: `logging.structured.format.console=ecs|logstash|gelf`; earlier use logstash-logback-encoder) so fields are searchable.
- **Correlation ID:** in Boot 3 with Micrometer Tracing, `traceId`/`spanId` are put in the SLF4J **MDC** automatically and Boot's default log pattern includes them (correlation pattern, Boot 3.2+; `logging.pattern.correlation`). A gateway/servlet filter that reads/creates `X-Correlation-Id` is still handy for non-trace contexts (e.g. support ticket references) and must copy to MDC and outgoing headers.
- **MDC and threads:** MDC is thread-local; it is lost on `@Async`, executors, reactive chains. Use `ContextSnapshot`/`Context Propagation` library (`Hooks.enableAutomaticContextPropagation()` in Reactor), or decorate executors (`ContextExecutorService`, `TaskDecorator`).
- Never log secrets/PII/full card numbers; log at boundaries: request id, user id (hashed), outcome, duration.

### Distributed tracing: Micrometer Tracing + OpenTelemetry
- Spring Boot 3 replaced Spring Cloud Sleuth with **Micrometer Tracing** (facade) + a **bridge** (`micrometer-tracing-bridge-otel` or `-brave`) + an exporter (`opentelemetry-exporter-zipkin` / `opentelemetry-exporter-otlp`, or `zipkin-reporter-brave`). Backends: Zipkin, Jaeger (accepts OTLP), Tempo, vendor APMs.
- **Concepts:** trace = whole request (traceId); span = one operation (spanId, parentSpanId, start, duration, tags); sampling decides which traces are kept (`management.tracing.sampling.probability=0.1`; default is 0.1; 1.0 in dev).
- **Context propagation:** the caller injects headers; the callee extracts them and continues the trace. **W3C Trace Context**: `traceparent: 00-<32-hex traceId>-<16-hex parentSpanId>-<2-hex flags>` (+ optional `tracestate`). Boot's default propagation type is W3C (B3 optional via `management.tracing.propagation.type`).
- Auto-instrumented if you build clients from Boot-provided builders (`RestClient.Builder`, `RestTemplateBuilder`, `WebClient.Builder`) - do not `new RestTemplate()` or you lose propagation. Server side is automatic with actuator + observation.
- **Across Kafka:** the trace context is carried in **record headers** (`traceparent`). Enable observation on the template and listener containers: `spring.kafka.template.observation-enabled=true` and `spring.kafka.listener.observation-enabled=true`; then produce/consume spans link. Same idea for RabbitMQ (`spring.rabbitmq.template.observation-enabled`, `spring.rabbitmq.listener.simple.observation-enabled`). For outbox: **store the traceparent in the outbox row** (column `trace_context`) and set it as a header when the relay publishes, otherwise the trace breaks at the relay (the poller runs in a different thread/time).
- Exemplars link metrics to traces; useful for jumping from a latency spike to a trace.

### Metrics: RED and USE; SLI/SLO/error budget
- **RED** (request-driven services): **R**ate, **E**rrors, **D**uration (histograms with percentiles p50/p95/p99, never averages). `http.server.requests` in Micrometer gives all three; `resilience4j_circuitbreaker_state`, `_calls`, `_failure_rate` (Micrometer registry binds automatically with actuator+prometheus).
- **USE** (resources: CPU, memory, pools, queues): **U**tilization, **S**aturation, **E**rrors - e.g. Tomcat busy threads, HikariCP pending connections, Kafka consumer lag, JVM GC pause.
- **SLI:** measured indicator (fraction of requests < 300 ms and non-5xx). **SLO:** target (99.9% over 30 days). **SLA:** contract with penalties. **Error budget** = 1 - SLO (99.9% over 30 days = 43.2 min of "badness"); spent budget -> freeze features, prioritize reliability.
- **Alert design:** alert on *symptoms users feel* (SLO burn rate, error rate, latency) not causes (CPU 80%). **Multi-window multi-burn-rate** (e.g. burn 14.4x over 1 h and 5 min windows pages someone; slower burn opens a ticket). Every page must be actionable with a runbook; alert fatigue kills on-call. Dashboards per service: RED + dependency breaker states + saturation.

### Health checks (Actuator)
- **Liveness** (`/actuator/health/liveness`): "is the process alive or wedged?" Failure -> K8s **restarts** the container. Must NOT check downstream dependencies (a DB outage would restart every pod - a cascading failure you caused).
- **Readiness** (`/actuator/health/readiness`): "can I take traffic right now?" Failure -> removed from Service endpoints (no restart). May include critical local deps, warm-up, draining; be careful with hard downstream dependencies (if all pods go unready, you have zero capacity; prefer degrade over unready).
- **Startup probe:** protects slow-starting apps: liveness/readiness are disabled until it succeeds (avoids killing during long startup/migrations).
- Boot: `management.endpoint.health.probes.enabled=true` (auto-enabled when running on K8s), `management.endpoint.health.group.readiness.include=readinessState,db`. Also handle graceful shutdown: `server.shutdown=graceful`, `spring.lifecycle.timeout-per-shutdown-phase=30s`, and a `preStop` sleep so endpoints deregister before SIGTERM stops accepting.

## 2.19 Deployment, testing, and platform topics

### Containers and Kubernetes (link)
Build layered/OCI images (`spring-boot:build-image` buildpacks or Jib), non-root user, JVM container-aware memory (`-XX:MaxRAMPercentage=75`), resource requests/limits, one process per container. K8s primitives (Deployment, Service, Ingress, HPA, ConfigMap/Secret, probes): see the K8s notes; only the microservices-relevant bits are here.

### Deployment strategies
- **Rolling:** replace pods gradually (`maxUnavailable`, `maxSurge`); default; **both versions run at once** -> everything must be backward/forward compatible.
- **Blue-green:** full parallel environment, switch traffic at once; instant rollback, double cost, DB must serve both.
- **Canary:** send 1% -> 5% -> 50% to the new version, watch SLIs, auto-rollback (Argo Rollouts/Flagger/mesh traffic splitting).
- **Feature flags:** decouple *deploy* from *release*; dark launch, gradual rollout by user %, kill switch (Unleash, LaunchDarkly, OpenFeature, Togglz). Remove stale flags; flags are also tech debt.

### Database migrations under rolling deploy: expand/contract (parallel change)
Old and new app versions overlap, so schema changes must be compatible with both:
```
Goal: rename column  orders.total -> orders.grand_total
1. EXPAND:   add column grand_total (nullable). Deploy vN+1 that WRITES both, READS old (or new with fallback). Backfill in batches.
2. MIGRATE:  deploy vN+2 that reads grand_total only; keep writing both for one more release (rollback safety).
3. CONTRACT: after all old versions are gone and rollback window passed, drop `total`.
```
Rules: additive changes first; never drop/rename in the same release as the code that stops using it; new columns nullable or defaulted; long backfills in batches outside the deploy; use Flyway/Liquibase; index creation `CONCURRENTLY` (Postgres); run migrations as a separate job (init container / Helm hook), not by N pods racing.

### Backward/forward compatibility of messages
Consumers of a topic are deployed independently, and messages persist in Kafka for days.
- **Backward compatible:** new schema can read *old* data (add optional fields with defaults, remove fields consumers ignore). Consumers upgraded first.
- **Forward compatible:** old schema can read *new* data (old consumers ignore unknown fields). Producers upgraded first.
- **Full:** both - the safe default for long-lived topics; enforce with schema registry compatibility mode. Rename = add new + deprecate old (expand/contract for events). Never reuse a field name with a new meaning. Keep old consumers able to handle events during rollout and replay of history.

### Service mesh (Istio, Linkerd)
Sidecar proxy (Envoy in Istio; Linkerd's own Rust micro-proxy) next to each pod handles traffic; a control plane pushes config. (Istio also has an ambient, sidecar-less mode.)
Gives: **automatic mTLS** and workload identity, **retries/timeouts/circuit-breaking (outlier detection)** outside the app, **traffic splitting** for canary (90/10), fault injection, uniform metrics and tracing without code, authz policies.
Trade-offs: extra latency hop and CPU/memory per pod, operational complexity and a steep learning curve, debugging is harder (two proxies in path), version upgrades of the mesh, and **double retries** if the app also retries (amplification!). Pick: small estate -> libraries (Resilience4j) + K8s; large polyglot estate/security mandate (mTLS everywhere) -> mesh. Business-aware fallbacks (stale cache, degraded response) still belong in the app; the mesh can only retry/time out/eject.

### Testing strategy
```
      /\      few E2E (critical journeys only; slow, flaky, expensive)
     /  \     contract tests (Pact / Spring Cloud Contract) - replace most E2E integration checks
    /----\    integration tests (Testcontainers: real Postgres/Kafka/Redis; @SpringBootTest slices; WireMock for HTTP deps)
   /------\   many unit tests (domain logic, no Spring)
```
- **Testcontainers** with `@ServiceConnection` (Boot 3.1+) gives real DB/Kafka in tests, no H2 lies. Use `@DynamicPropertySource` otherwise.
- Test resilience explicitly: WireMock/Toxiproxy to inject delay/500s and assert fallback, timeouts, breaker opening; test idempotent consumers by delivering a message twice; test saga compensations per failure branch.
- **Chaos testing:** deliberately inject failure in a controlled way (kill pods, add latency, drop network) - Chaos Monkey for Spring Boot, Chaos Mesh, Litmus, AWS FIS. Start in staging with a hypothesis ("if payment is down, checkout still accepts orders as pending"), then production with small blast radius (game days).
- Keep E2E to a few smoke journeys run after deploy (synthetic monitoring in prod is often better).

## 2.20 CAP, PACELC, multi-region

**CAP:** during a network **P**artition you must choose **C** (reject/err some requests to stay consistent: CP) or **A** (serve possibly stale: AP). It says nothing about normal operation. Consistency here = linearizability, not ACID's "C".
**PACELC:** **if P**artition -> A or C; **E**lse (normal) -> **L**atency or **C**onsistency. E.g. Cassandra (tunable) PA/EL; a single-primary Postgres with sync replicas PC/EC; DynamoDB default eventually consistent reads = PA/EL.
In service terms: a service that needs strong consistency (inventory decrement, payment ledger) picks CP for that data and accepts unavailability during partitions; product catalog/search/recommendations pick AP with caches and eventual consistency. Different services, different choices - that is a microservices advantage. Sagas = choosing availability + eventual consistency at the workflow level.

**Multi-region overview:**
- **Active-passive:** one region serves; standby has replicated data; failover (RTO minutes, RPO = replication lag). Simplest.
- **Active-active:** all regions serve users (geo-routing via DNS/global LB). Data is the hard part: single-writer per record (home region by user), or multi-master with conflict resolution (last-write-wins, CRDTs), or a globally consistent DB (Spanner/CockroachDB - higher write latency).
- Considerations: data residency (GDPR), replication lag and read-your-writes, cross-region latency 50-150 ms (avoid sync cross-region calls in the request path), Kafka MirrorMaker/cluster linking, idempotency everywhere (failover replays), failover drills, and cell-based architecture to limit blast radius.

## 2.21 Migrating from a monolith

- **Strangler Fig:** put a facade/router (gateway or proxy) in front of the monolith; carve out one capability at a time behind it; route those URLs to the new service; the monolith shrinks until removable. Start with a low-risk, loosely coupled, high-value seam (notifications, search), not the core.
```
Phase 1:  Client -> Gateway --------> Monolith (all routes)
Phase 2:  Client -> Gateway --/notify--> NotificationService   (new)
                            \--rest----> Monolith
Phase 3:  more routes move; monolith data still shared? -> replicate/CDC, then own the data
```
- **Branch by abstraction:** inside the monolith, introduce an interface around the capability, make all callers use it, build a new implementation (calling the new service) behind the interface, switch by flag, delete old. Lets you refactor incrementally on trunk without long branches.
- **Anti-Corruption Layer:** translator between monolith's model and the new service's model so legacy concepts do not leak in (adapter that calls monolith API/DB view and maps to clean DTOs).
- **Data migration:** move the *code* first with a shared/replicated DB, then split data: dual-read/dual-write with verification is risky; prefer CDC (Debezium) to sync then cut over per-slice; keep a rollback path; compare results in shadow mode ("parallel run").
- Extract by **business capability with the fewest inbound dependencies first**; avoid extracting where the transaction boundary would have to be split (that needs a saga).

## 2.22 Common anti-patterns
- Distributed monolith (2.2); shared database; shared domain library.
- **Nano-services:** a service per entity/CRUD table; more network than logic.
- Sync chains and chatty N+1 calls; "microservice as function".
- **Missing timeouts** / default infinite timeouts; retries without idempotency; retries at every layer.
- Distributed transactions via 2PC or "fire and forget" dual writes.
- Ignoring versioning/compat; breaking contracts silently.
- Gateway with business logic / god orchestrator / ESB reborn.
- No observability before splitting; no per-service ownership/on-call.
- Using Kafka topics as a database of record without retention/compaction thinking; events that are CRUD notifications ("OrderUpdated") with no semantics.
- Config Server or discovery as a single point of failure without local fallback/caching.
- Liveness probe checking the database.
- Entity services / "database over HTTP"; anemic services calling each other for every field.
- Big-bang rewrite instead of strangler.

## 2.23 Worked design: e-commerce order flow with failure analysis per hop

Requirements: place order, pay by card, reserve stock, notify. Targets: checkout p99 < 2 s (to acceptance), no double charge, no oversell, no lost order, tolerate any single service outage.

```
 Browser -> [Gateway] -> Order Service (Postgres + outbox) ==Kafka==> Inventory (own DB)
                              ^  |                                     ==Kafka==> Payment (own DB) -> Bank API (external)
                              |  +-- saga orchestrator inside Order ==Kafka==> Notification (email)
                              +----- results (events) --------------------------+
```
Flow (async saga, orchestrated by Order service):
1. Client `POST /orders` with `Idempotency-Key`. Gateway: validates JWT, rate limits, adds `traceparent`, forwards (no retry on POST!).
2. Order service (one local tx): idempotency row, `orders(PENDING)`, `saga_instance(STARTED, deadline)`, outbox `ReserveStockCommand`. Returns `202` + `Location`.
3. Outbox relay publishes `ReserveStockCommand` (key = orderId). Inventory consumes (inbox dedup), tx: decrement available/increment reserved, outbox `StockReserved`/`StockRejected`.
4. Orchestrator consumes reply (inbox dedup), tx: advance saga -> outbox `ChargePaymentCommand{orderId, amount, idempotencyKey=orderId}`.
5. Payment consumes, calls Bank API with idempotency key = orderId (timeouts, breaker, bounded retry), tx: record `payments(CAPTURED)`, outbox `PaymentSucceeded`.
6. Orchestrator: order `CONFIRMED`, saga `COMPLETED`, outbox `OrderConfirmed` -> Notification sends email (retryable, non-critical, DLQ on poison).

**Failure analysis at each hop**
| Hop / failure | What happens without care | Mitigation |
|---|---|---|
| Client double-clicks or retries POST | Two orders | `Idempotency-Key` unique constraint; second call replays the first response |
| **Gateway retries a POST** after upstream timeout | Duplicate order/charge | Disable gateway retry for non-idempotent methods (Gateway `Retry` filter defaults to GET); only retry with idempotency key; retry at one layer only |
| Gateway -> Order: Order slow | Gateway threads/connection pile up | Timeouts, bulkhead, load shedding (429/503), breaker on the route |
| Order DB commit OK, response lost to client | Client thinks failed, resubmits | Idempotency key -> same order returned |
| Order tx commit OK but Kafka down | Dual-write loss (no event) | **Outbox**: row stays unpublished, relay publishes when Kafka returns |
| Relay publishes twice | Inventory reserves twice | Inbox dedup by messageId + reservation keyed by orderId (unique) |
| **Inventory event lost / never processed** | Saga waits forever, order stuck PENDING | Kafka durability (acks=all, replication) + outbox makes loss rare; **saga deadline sweeper** re-sends command or queries inventory; alert on sagas > SLA; reconciliation job |
| Inventory rejects (OOS) | - | `StockRejected` -> order `REJECTED`, notify user; no money moved yet (why stock precedes payment) |
| Inventory consumer poison message | Partition blocked | Bounded retries with backoff -> DLQ; alert; replay after fix (see `Kafka.md`) |
| Orchestrator sends ChargePayment, **Payment times out** | Unknown outcome | Never assume failure. Retry with same key -> Payment replays original result; else query by orderId; else `Cancel/void` then compensate stock |
| **Payment timed out but actually succeeded at the bank** | Customer charged, order cancelled | Idempotency key at bank + reconciliation: on timeout we ask "what happened to key X?". If captured -> continue forward. Nightly reconciliation (bank statement vs payments table) catches drift; auto-refund orphan captures |
| Payment declined | - | `PaymentFailed` -> compensate: release stock, order `CANCELLED` |
| Payment succeeded, but `PaymentSucceeded` event delayed/duplicated/out of order | Saga confused | State machine guards, dedup, version numbers; event after terminal state is ignored/alerted |
| Orchestrator crashes mid-saga | In-memory state lost | State + outgoing command in same tx; restart resumes; sweeper |
| Compensation (release stock) fails | Stock leaks | Retry with backoff forever (idempotent), alert after N, manual queue |
| Notification down | Email not sent | Async + retry + DLQ; order unaffected (non-critical, after pivot) |
| Bank API down | All payments fail | Circuit breaker opens quickly; fallback = accept order as `PAYMENT_PENDING` and retry later, or clear 503 - business decision; bulkhead so payment problems do not starve inventory handling |
| Region/AZ outage | - | Multi-AZ pods + DB replicas; idempotent replays on failover; RPO/RTO defined |
| Traceability | "Where is my order?" | traceId in logs/outbox headers; `GET /orders/{id}` exposes saga state; dashboards for saga age |

Consistency statement for the interview: *"Order, stock and payment are each strongly consistent locally; across services the system is eventually consistent, converging via the saga; anomalies (timeouts, lost messages) are caught by deadlines, idempotent retries and reconciliation."*

---

# PART 3 - Code

## 3.1 Dependencies (Maven, versions via Boot + Spring Cloud BOMs)
```xml
<dependencies>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-web</artifactId></dependency>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-actuator</artifactId></dependency>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-aop</artifactId></dependency> <!-- required for R4j annotations -->
  <dependency><groupId>io.github.resilience4j</groupId><artifactId>resilience4j-spring-boot3</artifactId></dependency>
  <dependency><groupId>io.micrometer</groupId><artifactId>micrometer-registry-prometheus</artifactId></dependency>
  <dependency><groupId>io.micrometer</groupId><artifactId>micrometer-tracing-bridge-otel</artifactId></dependency>
  <dependency><groupId>io.opentelemetry</groupId><artifactId>opentelemetry-exporter-otlp</artifactId></dependency>
  <dependency><groupId>org.springframework.kafka</groupId><artifactId>spring-kafka</artifactId></dependency>
</dependencies>
```
(Resilience4j's version is managed by the Spring Cloud BOM `spring-cloud-dependencies` via `spring-cloud-starter-circuitbreaker-resilience4j`; with plain `resilience4j-spring-boot3` declare its version yourself.)

## 3.2 Resilience4j: configuration and annotations
```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        registerHealthIndicator: true
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 20
        minimumNumberOfCalls: 10
        failureRateThreshold: 50
        slowCallDurationThreshold: 800ms
        slowCallRateThreshold: 60
        waitDurationInOpenState: 20s
        permittedNumberOfCallsInHalfOpenState: 5
        automaticTransitionFromOpenToHalfOpenEnabled: true
        ignoreExceptions:
          - com.shop.common.BusinessException      # 4xx-style errors are not dependency failures
    instances:
      payment:
        baseConfig: default
  retry:
    instances:
      payment:
        maxAttempts: 3                    # includes the first call
        waitDuration: 200ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        enableRandomizedWait: true        # jitter
        randomizedWaitFactor: 0.5
        retryExceptions:
          - java.io.IOException
          - org.springframework.web.client.ResourceAccessException
        ignoreExceptions:
          - io.github.resilience4j.circuitbreaker.CallNotPermittedException
          - com.shop.common.BusinessException
  bulkhead:
    instances:
      payment:
        maxConcurrentCalls: 20
        maxWaitDuration: 0ms              # fail immediately when full
  ratelimiter:
    instances:
      payment:
        limitForPeriod: 50
        limitRefreshPeriod: 1s
        timeoutDuration: 0
  timelimiter:
    instances:
      payment:
        timeoutDuration: 1500ms
        cancelRunningFuture: true
```
```java
@Service
class PaymentGateway {
    private final RestClient rest;

    PaymentGateway(RestClient.Builder builder) {                 // Boot-provided builder => tracing headers propagate
        var factory = new JdkClientHttpRequestFactory(
                HttpClient.newBuilder().connectTimeout(Duration.ofMillis(300)).build());
        factory.setReadTimeout(Duration.ofMillis(1200));          // read timeout: real protection for blocking calls
        this.rest = builder.baseUrl("http://payment-service").requestFactory(factory).build();
    }

    // Default nesting: Retry( CircuitBreaker( RateLimiter( TimeLimiter( Bulkhead( this method ))))).
    // @TimeLimiter/@Bulkhead(THREADPOOL) need CompletableFuture returns; for blocking code we rely on
    // the client read timeout + semaphore bulkhead instead.
    @Retry(name = "payment")
    @CircuitBreaker(name = "payment", fallbackMethod = "chargeFallback")
    @Bulkhead(name = "payment")
    public PaymentResult charge(ChargeRequest req) {
        return rest.post().uri("/payments")
                .header("Idempotency-Key", req.orderId().toString())   // same key on every retry
                .body(req).retrieve().body(PaymentResult.class);
    }

    // Fallback: same params + exception. Most specific exception type wins.
    PaymentResult chargeFallback(ChargeRequest req, CallNotPermittedException e) {
        return PaymentResult.pending(req.orderId(), "circuit open");     // honest degradation, NOT fake success
    }
    PaymentResult chargeFallback(ChargeRequest req, Exception e) {
        return PaymentResult.unknown(req.orderId(), e.getClass().getSimpleName()); // saga will reconcile
    }
}
```
Note: with `Retry` outermost, a fallback attached to `@CircuitBreaker` fires per attempt inside the retry - which means Retry may never see the exception (fallback swallowed it). If you want retries to run *before* the fallback, put the fallback on the outermost annotation (`@Retry(fallbackMethod=...)`) and make inner ones throw. Decide consciously.

Programmatic API (clear nesting, testable) - decorators compose in the order you call them:
```java
Supplier<PaymentResult> call = () -> gateway.rawCharge(req);
Supplier<PaymentResult> decorated = Decorators.ofSupplier(call)
        .withBulkhead(bulkhead)
        .withCircuitBreaker(circuitBreaker)
        .withRetry(retry)                    // added last => outermost
        .withFallback(List.of(CallNotPermittedException.class), e -> PaymentResult.pending(req.orderId(), "open"))
        .decorate();
```
(`Decorators` is in `resilience4j-all`/`resilience4j-decorators`; the last `with...` wraps outermost.)

Observing the breaker:
```java
circuitBreakerRegistry.circuitBreaker("payment").getEventPublisher()
    .onStateTransition(e -> log.warn("breaker {} {}", e.getCircuitBreakerName(), e.getStateTransition()))
    .onCallNotPermitted(e -> meterRegistry.counter("payment.rejected").increment());
```
Actuator: `GET /actuator/circuitbreakers`, `/actuator/circuitbreakerevents`, `/actuator/retries`; expose them via `management.endpoints.web.exposure.include`.

## 3.3 Transactional Outbox: entity, service, polling publisher, inbox

**Schema (PostgreSQL)**
```sql
CREATE TABLE outbox (
  id            BIGSERIAL PRIMARY KEY,
  event_id      UUID        NOT NULL UNIQUE,
  aggregate_type VARCHAR(60) NOT NULL,
  aggregate_id  VARCHAR(60) NOT NULL,
  event_type    VARCHAR(80) NOT NULL,
  payload       JSONB       NOT NULL,
  trace_context VARCHAR(80),                   -- W3C traceparent captured at write time
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  published_at  TIMESTAMPTZ
);
CREATE INDEX idx_outbox_unpublished ON outbox (id) WHERE published_at IS NULL;   -- partial index

CREATE TABLE processed_message (               -- inbox / dedup on the consumer side
  message_id   UUID PRIMARY KEY,
  consumer     VARCHAR(60) NOT NULL,
  processed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```
```java
@Entity @Table(name = "outbox")
public class OutboxEvent {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    @Column(name = "event_id", nullable = false, unique = true) private UUID eventId = UUID.randomUUID();
    @Column(name = "aggregate_type") private String aggregateType;
    @Column(name = "aggregate_id")   private String aggregateId;
    @Column(name = "event_type")     private String eventType;
    @JdbcTypeCode(SqlTypes.JSON) @Column(columnDefinition = "jsonb") private String payload;  // Hibernate 6
    @Column(name = "trace_context")  private String traceContext;
    @Column(name = "created_at")     private Instant createdAt = Instant.now();
    @Column(name = "published_at")   private Instant publishedAt;
    protected OutboxEvent() {}
    public OutboxEvent(String type, String aggId, String eventType, String json, String trace) {
        this.aggregateType = type; this.aggregateId = aggId; this.eventType = eventType;
        this.payload = json; this.traceContext = trace;
    }
    // getters omitted
    public void markPublished() { this.publishedAt = Instant.now(); }
}

public interface OutboxRepository extends JpaRepository<OutboxEvent, Long> {
    // FOR UPDATE SKIP LOCKED lets several relay instances claim disjoint batches without blocking each other.
    @Query(value = """
        SELECT * FROM outbox WHERE published_at IS NULL
        ORDER BY id LIMIT :batch FOR UPDATE SKIP LOCKED
        """, nativeQuery = true)
    List<OutboxEvent> lockNextBatch(@Param("batch") int batch);
}
```
**Business write + outbox row in the SAME transaction**
```java
@Service
public class OrderService {
    private final OrderRepository orders; private final OutboxRepository outbox;
    private final ObjectMapper mapper; private final Tracer tracer;   // io.micrometer.tracing.Tracer

    @Transactional
    public Order placeOrder(PlaceOrder cmd) {
        Order order = orders.save(Order.pending(cmd));                // 1. business change
        var evt = new OrderCreated(order.getId(), order.getCustomerId(), order.getTotal());
        outbox.save(new OutboxEvent("Order", order.getId().toString(), "OrderCreated",
                toJson(evt), currentTraceparent()));                   // 2. same tx => atomic
        return order;                                                  // commit persists BOTH or NEITHER
    }
    private String toJson(Object o) { try { return mapper.writeValueAsString(o); } catch (Exception e) { throw new IllegalStateException(e); } }
    private String currentTraceparent() {
        var span = tracer.currentSpan();
        return span == null ? null : "00-" + span.context().traceId() + "-" + span.context().spanId() + "-01";
    }
}
```
**Polling publisher**
```java
@Component
public class OutboxRelay {
    private final OutboxRepository repo; private final KafkaTemplate<String, String> kafka;
    public OutboxRelay(OutboxRepository repo, KafkaTemplate<String, String> kafka) { this.repo = repo; this.kafka = kafka; }

    @Scheduled(fixedDelayString = "${outbox.poll-ms:200}")
    @Transactional                                                   // row locks live for the batch
    public void publishBatch() {
        List<OutboxEvent> batch = repo.lockNextBatch(100);
        for (OutboxEvent e : batch) {
            var record = new ProducerRecord<>("order-events", e.getAggregateId(), e.getPayload());   // key = aggregateId => per-aggregate ordering
            record.headers().add("eventId", e.getEventId().toString().getBytes(StandardCharsets.UTF_8));
            record.headers().add("eventType", e.getEventType().getBytes(StandardCharsets.UTF_8));
            if (e.getTraceContext() != null) record.headers().add("traceparent", e.getTraceContext().getBytes(StandardCharsets.UTF_8));
            try {
                kafka.send(record).get(5, TimeUnit.SECONDS);           // wait for broker ack (acks=all, enable.idempotence=true)
            } catch (Exception ex) {
                throw new IllegalStateException("publish failed, tx rolls back, rows stay unpublished", ex);
            }
            e.markPublished();
        }
    }
    // If the app crashes after send() but before COMMIT, the batch is re-sent => at-least-once. Consumers dedup.
}
// Cleanup: @Scheduled job (ShedLock) deleting rows WHERE published_at < now() - interval '7 days'.
```
Producer config essentials: `spring.kafka.producer.acks=all`, `spring.kafka.producer.properties.enable.idempotence=true`, `max.in.flight.requests.per.connection<=5` (idempotence keeps ordering). Detail in `Kafka.md`.

**Debezium alternative (config sketch):** a Postgres connector with `transforms=outbox`, `transforms.outbox.type=io.debezium.transforms.outbox.EventRouter`, table = `public.outbox`; Debezium maps columns by default names (`id`, `aggregatetype`, `aggregateid`, `type`, `payload`) - configurable via `transforms.outbox.table.field.event.*`. Verify column mapping against your Debezium version. The app then only inserts rows; no relay code.

**Idempotent consumer (inbox)**
```java
@Component
public class InventoryEventConsumer {
    private final ProcessedMessageRepository processed; private final InventoryService inventory;

    @KafkaListener(topics = "order-events", groupId = "inventory")
    @Transactional                                                    // dedup insert + business change commit together
    public void on(ConsumerRecord<String, String> rec) {
        UUID messageId = UUID.fromString(header(rec, "eventId"));
        try {
            processed.saveAndFlush(new ProcessedMessage(messageId, "inventory"));   // PK violation => duplicate
        } catch (DataIntegrityViolationException dup) {
            return;                                                    // already handled: ack and skip
        }
        inventory.handle(header(rec, "eventType"), rec.value());       // business effect in same tx
    }
    private String header(ConsumerRecord<?, ?> r, String k) { return new String(r.headers().lastHeader(k).value(), StandardCharsets.UTF_8); }
}
```
Caveat: with JPA, when a constraint violation happens inside a transaction Hibernate marks the transaction rollback-only (the PK failure kills the tx state); for a cleaner design use `INSERT ... ON CONFLICT DO NOTHING` via `JdbcTemplate.update` and check the affected-row count (0 = duplicate). See `Spring_Transactional.md` for rollback-only semantics.
```java
int inserted = jdbc.update("INSERT INTO processed_message(message_id, consumer) VALUES (?, ?) ON CONFLICT DO NOTHING", id, "inventory");
if (inserted == 0) return;   // duplicate
```

## 3.4 Saga orchestrator skeleton (hand-rolled, persisted state machine)
```java
public enum SagaState { STARTED, STOCK_RESERVED, PAYMENT_UNKNOWN, PAYMENT_DONE, COMPLETED,
                        COMPENSATING_STOCK, COMPENSATED, NEEDS_ATTENTION }

@Entity @Table(name = "saga_instance")
public class OrderSaga {
    @Id private UUID id;                       // = orderId
    @Enumerated(EnumType.STRING) private SagaState state;
    private Instant deadlineAt;
    private int attempts;
    @Version private long version;             // optimistic lock: two events cannot advance the saga concurrently
    // getters/setters ...
}

@Service
public class OrderSagaOrchestrator {
    private final SagaRepository sagas; private final OrderRepository orders; private final OutboxWriter outbox;

    @Transactional
    public void start(PlaceOrder cmd) {
        Order o = orders.save(Order.pending(cmd));
        sagas.save(OrderSaga.started(o.getId(), Instant.now().plusSeconds(60)));
        outbox.command("inventory-commands", o.getId(), "ReserveStock", new ReserveStock(o.getId(), cmd.items()));
    }

    @Transactional                              // called from an idempotent (inbox-guarded) listener
    public void onStockReserved(UUID orderId) {
        OrderSaga s = sagas.findById(orderId).orElseThrow();
        if (s.getState() != SagaState.STARTED) { log.warn("ignored StockReserved in {}", s.getState()); return; }   // guard: duplicate / out-of-order
        s.moveTo(SagaState.STOCK_RESERVED, Instant.now().plusSeconds(30));
        outbox.command("payment-commands", orderId, "ChargePayment", new ChargePayment(orderId, orders.total(orderId)));
    }

    @Transactional
    public void onStockRejected(UUID orderId) {
        OrderSaga s = sagas.findById(orderId).orElseThrow();
        if (s.getState() != SagaState.STARTED) return;
        orders.markRejected(orderId); s.moveTo(SagaState.COMPENSATED, null);
    }

    @Transactional
    public void onPaymentSucceeded(UUID orderId) {
        OrderSaga s = sagas.findById(orderId).orElseThrow();
        if (s.getState() != SagaState.STOCK_RESERVED && s.getState() != SagaState.PAYMENT_UNKNOWN) return;
        orders.markConfirmed(orderId);           // retryable step after the pivot
        s.moveTo(SagaState.COMPLETED, null);
        outbox.event("order-events", orderId, "OrderConfirmed", new OrderConfirmed(orderId));
    }

    @Transactional
    public void onPaymentFailed(UUID orderId) {                       // business failure => compensate
        OrderSaga s = sagas.findById(orderId).orElseThrow();
        if (s.getState() != SagaState.STOCK_RESERVED && s.getState() != SagaState.PAYMENT_UNKNOWN) return;
        s.moveTo(SagaState.COMPENSATING_STOCK, Instant.now().plusSeconds(30));
        outbox.command("inventory-commands", orderId, "ReleaseStock", new ReleaseStock(orderId));
    }

    @Transactional
    public void onStockReleased(UUID orderId) {
        OrderSaga s = sagas.findById(orderId).orElseThrow();
        if (s.getState() != SagaState.COMPENSATING_STOCK) return;
        orders.markCancelled(orderId); s.moveTo(SagaState.COMPENSATED, null);
    }

    /** Timeout sweeper: the safety net for lost messages and unknown outcomes. */
    @Scheduled(fixedDelay = 10_000)
    @SchedulerLock(name = "saga-sweeper", lockAtMostFor = "PT30S", lockAtLeastFor = "PT5S")
    @Transactional
    public void sweep() {
        for (OrderSaga s : sagas.findExpired(Instant.now(), 50)) {
            if (s.incrementAttempts() > 5) { s.moveTo(SagaState.NEEDS_ATTENTION, null); alert(s); continue; }
            switch (s.getState()) {
                case STARTED           -> outbox.command("inventory-commands", s.getId(), "ReserveStock", rebuild(s));   // idempotent re-send
                case STOCK_RESERVED    -> { s.moveTo(SagaState.PAYMENT_UNKNOWN, later(30));
                                            outbox.command("payment-commands", s.getId(), "QueryOrChargePayment", queryCmd(s)); }   // same idempotency key => safe
                case PAYMENT_UNKNOWN   -> outbox.command("payment-commands", s.getId(), "QueryOrChargePayment", queryCmd(s));
                case COMPENSATING_STOCK-> outbox.command("inventory-commands", s.getId(), "ReleaseStock", new ReleaseStock(s.getId()));
                default -> { }
            }
            s.bumpDeadline(backoff(s.getAttempts()));
        }
    }
}
```
Points to say aloud: every handler is (1) idempotent via inbox, (2) state-guarded, (3) writes state + next command in one transaction, (4) the sweeper covers lost messages, (5) `@Version` prevents concurrent double-advance (`OptimisticLockingFailureException` -> retry the message). In Temporal the same logic is ordinary code (`activities` with retry policies; the workflow history is the persisted state); in Camunda it is a BPMN model with service tasks, boundary timer events and compensation events.

## 3.5 Idempotency filter (Spring MVC)
```sql
CREATE TABLE idempotency_key (
  client_id    VARCHAR(80)  NOT NULL,
  idem_key     VARCHAR(100) NOT NULL,
  request_hash CHAR(64)     NOT NULL,
  status       VARCHAR(12)  NOT NULL,           -- IN_PROGRESS | COMPLETED
  resp_status  INT,
  resp_body    TEXT,
  created_at   TIMESTAMPTZ  NOT NULL DEFAULT now(),
  PRIMARY KEY (client_id, idem_key)              -- concurrency gate
);
```
```java
@Component
@Order(Ordered.LOWEST_PRECEDENCE - 10)            // after security filters so we know the principal
public class IdempotencyFilter extends OncePerRequestFilter {
    private final JdbcTemplate jdbc;
    public IdempotencyFilter(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    @Override protected boolean shouldNotFilter(HttpServletRequest req) {
        return !"POST".equals(req.getMethod()) || req.getHeader("Idempotency-Key") == null;
    }

    @Override protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {
        byte[] body = request.getInputStream().readAllBytes();
        var replayable = new ReplayableRequest(request, body);              // small wrapper re-exposing the bytes to downstream filters/controller
        String client = request.getUserPrincipal() != null ? request.getUserPrincipal().getName() : "anonymous";
        String key = request.getHeader("Idempotency-Key");
        String hash = sha256(request.getRequestURI() + "|" + new String(body, StandardCharsets.UTF_8));

        // autocommit insert: the unique key decides the single winner among concurrent duplicates
        int won = jdbc.update("""
            INSERT INTO idempotency_key(client_id, idem_key, request_hash, status)
            VALUES (?,?,?,'IN_PROGRESS') ON CONFLICT DO NOTHING""", client, key, hash);

        if (won == 0) {                                                     // duplicate: replay or reject
            var row = jdbc.queryForMap("SELECT request_hash, status, resp_status, resp_body FROM idempotency_key WHERE client_id=? AND idem_key=?", client, key);
            if (!hash.equals(row.get("request_hash"))) { response.sendError(422, "Idempotency-Key reused with different request"); return; }
            if ("IN_PROGRESS".equals(row.get("status"))) { response.setHeader("Retry-After", "1"); response.sendError(409, "Request in progress"); return; }
            response.setStatus((Integer) row.get("resp_status"));
            response.setContentType("application/json");
            response.getWriter().write((String) row.get("resp_body"));      // replay stored response
            response.setHeader("Idempotent-Replayed", "true");
            return;
        }
        var wrapped = new ContentCachingResponseWrapper(response);
        try {
            chain.doFilter(replayable, wrapped);
            String out = new String(wrapped.getContentAsByteArray(), StandardCharsets.UTF_8);
            jdbc.update("UPDATE idempotency_key SET status='COMPLETED', resp_status=?, resp_body=? WHERE client_id=? AND idem_key=?",
                        wrapped.getStatus(), out, client, key);
        } catch (RuntimeException | ServletException | IOException ex) {
            jdbc.update("DELETE FROM idempotency_key WHERE client_id=? AND idem_key=?", client, key);   // failed before any effect: allow retry
            throw ex;
        } finally {
            wrapped.copyBodyToResponse();                                   // MUST be called or the client receives an empty body
        }
    }
    // sha256(..) helper and ReplayableRequest (HttpServletRequestWrapper over a byte[]) omitted.
}
```
Honest caveats: (1) the filter's update to `COMPLETED` is not atomic with the controller's DB transaction; a crash between them leaves `IN_PROGRESS` (handle with a lease timeout + the business operation being naturally idempotent by e.g. `orderId`), or move this logic into the service layer inside the business transaction for true atomicity. (2) 5xx responses: decide whether to store them (Stripe stores results including 500s for a key; simpler systems delete and allow retry). (3) Do not delete on exception if side effects could have happened. This is a teaching skeleton; the simplest production approach for message consumers is the inbox insert from 3.3.

## 3.6 Spring Cloud Gateway: routes, filters, Redis rate limiter
```yaml
spring:
  cloud:
    gateway:                          # in Spring Cloud 2025.0+: spring.cloud.gateway.server.webflux.*
      default-filters:
        - DedupeResponseHeader=Access-Control-Allow-Origin
      routes:
        - id: orders
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
            - Method=GET,POST
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 20      # tokens/second (steady rate)
                redis-rate-limiter.burstCapacity: 40      # bucket size (burst)
                redis-rate-limiter.requestedTokens: 1
                key-resolver: "#{@userKeyResolver}"
            - name: CircuitBreaker
              args:
                name: ordersCb
                fallbackUri: forward:/fallback/orders
            - name: Retry                                  # GET only by default; never blanket-retry POST
              args:
                retries: 2
                methods: GET
                statuses: BAD_GATEWAY, SERVICE_UNAVAILABLE
                backoff: { firstBackoff: 50ms, maxBackoff: 500ms, factor: 2, basedOnPreviousValue: false }
        - id: catalog
          uri: lb://catalog-service
          predicates: [ "Path=/api/catalog/**", "Weight=catalog, 90" ]      # canary group
          filters: [ "StripPrefix=1", "TokenRelay=" ]
```
```java
@Configuration
class GatewayConfig {
    @Bean KeyResolver userKeyResolver() {                                   // rate-limit per authenticated user (fallback: IP)
        return exchange -> exchange.getPrincipal().map(Principal::getName)
            .switchIfEmpty(Mono.fromSupplier(() -> exchange.getRequest().getRemoteAddress().getAddress().getHostAddress()));
    }
}

@Component
class CorrelationGlobalFilter implements GlobalFilter, Ordered {           // custom global filter
    @Override public Mono<Void> filter(ServerWebExchange ex, GatewayFilterChain chain) {
        String id = Optional.ofNullable(ex.getRequest().getHeaders().getFirst("X-Correlation-Id")).orElse(UUID.randomUUID().toString());
        var mutated = ex.mutate().request(r -> r.header("X-Correlation-Id", id)).build();
        return chain.filter(mutated);
    }
    @Override public int getOrder() { return -1; }
}
```
Gateway security: `spring-boot-starter-oauth2-resource-server` with a reactive `SecurityWebFilterChain` (`http.oauth2ResourceServer(o -> o.jwt(withDefaults()))`); rate-limit keys come after authentication so users are identified.

## 3.7 Timeouts, tracing, health: configuration
```yaml
management:
  tracing:
    sampling:
      probability: 0.1
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces        # or Zipkin: management.zipkin.tracing.endpoint
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,circuitbreakers,retries
  endpoint:
    health:
      probes:
        enabled: true
      group:
        readiness:
          include: readinessState,db
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
  kafka:
    template:
      observation-enabled: true
    listener:
      observation-enabled: true
  cloud:
    openfeign:
      client:
        config:
          default:
            connectTimeout: 300
            readTimeout: 1200
```
```yaml
# Kubernetes probes
livenessProbe:  { httpGet: { path: /actuator/health/liveness,  port: 8080 }, periodSeconds: 10, failureThreshold: 3 }
readinessProbe: { httpGet: { path: /actuator/health/readiness, port: 8080 }, periodSeconds: 5 }
startupProbe:   { httpGet: { path: /actuator/health/liveness,  port: 8080 }, periodSeconds: 5, failureThreshold: 30 }   # up to 150 s to start
```

## 3.8 ShedLock for @Scheduled in multiple instances
```java
@Configuration
@EnableScheduling
@EnableSchedulerLock(defaultLockAtMostFor = "PT10M")
class SchedulingConfig {
    @Bean LockProvider lockProvider(DataSource ds) { return new JdbcTemplateLockProvider(ds); }   // needs table shedlock(name PK, lock_until, locked_at, locked_by)
}
@Component
class Jobs {
    @Scheduled(cron = "0 */5 * * * *")
    @SchedulerLock(name = "reconcile-payments", lockAtMostFor = "PT4M", lockAtLeastFor = "PT30S")
    void reconcile() { /* runs on only one pod at a time */ }
}
```

## 3.9 Real, compiled-and-run simulations (Java 21)

### 3.9.1 Circuit breaker state machine (count-based window, fake clock)
Config in the simulation: window = 10 calls, minimumNumberOfCalls = 5, failureRateThreshold = 50%, waitDurationInOpenState = 1000 ms, permittedNumberOfCallsInHalfOpenState = 3. Core of the class:
```java
enum State { CLOSED, OPEN, HALF_OPEN }
<T> T call(Supplier<T> fn, Supplier<T> fallback) {
    if (state == State.OPEN) {
        if (clock.getAsLong() - openedAt >= waitOpenMs) { state = State.HALF_OPEN; halfOpenAttempts = 0; halfOpenFailures = 0; }
        else return fallback.get();                       // like CallNotPermittedException: downstream NOT touched
    }
    if (state == State.HALF_OPEN && halfOpenAttempts >= halfOpenPermits) return fallback.get();
    ... run fn, record(failed) ...
}
void record(boolean failed) {
    if (state == State.HALF_OPEN) {                      // evaluate once all trial calls are done
        if (failed) halfOpenFailures++;
        if (halfOpenAttempts >= halfOpenPermits)
            if (100.0 * halfOpenFailures / halfOpenPermits >= failureRate) trip(); else { state = CLOSED; window.clear(); }
        return;
    }
    window.addLast(failed); if (window.size() > windowSize) window.removeFirst();
    if (window.size() >= minCalls && failureRatePercent() >= failureRate) trip();
}
```
**Actual output** (`javac MiniBreaker.java && java MiniBreaker`; `downstreamCalls` proves the downstream is protected while OPEN):
```
== phase 1: healthy
 call -> OK state=CLOSED   (x6)
== phase 2: downstream dies
 call -> FALLBACK state=CLOSED downstreamCalls=7
 call -> FALLBACK state=CLOSED downstreamCalls=8
 call -> FALLBACK state=CLOSED downstreamCalls=9
 call -> FALLBACK state=CLOSED downstreamCalls=10
  [t=1100ms] -> OPEN                        <- 5th failure: window (6 ok + 5 fail, last 10) = 50% -> trips
 call -> FALLBACK state=OPEN downstreamCalls=11
 call -> FALLBACK state=OPEN downstreamCalls=11     <- downstream counter FROZEN: fail fast, no calls
 ... (5 more fallbacks, downstreamCalls stays 11)
== phase 3: wait elapsed but downstream STILL down (trial calls fail)
  [t=2610ms] OPEN -> HALF_OPEN
 call -> FALLBACK state=HALF_OPEN downstreamCalls=12
 call -> FALLBACK state=HALF_OPEN downstreamCalls=13
  [t=2630ms] -> OPEN                        <- 3 trial calls failed: 100% >= 50% -> OPEN again
 call -> FALLBACK state=OPEN downstreamCalls=14 ... (no more calls)
== phase 4: downstream recovers, wait elapsed again
  [t=3650ms] OPEN -> HALF_OPEN
 call -> OK state=HALF_OPEN downstreamCalls=15
 call -> OK state=HALF_OPEN downstreamCalls=16
  [t=3670ms] HALF_OPEN -> CLOSED            <- trial calls succeeded
 call -> OK state=CLOSED downstreamCalls=17 ...
```
(The log line prints during the call that caused the transition, so it appears just before that call's own result line.) Total downstream calls while it was dead: 8 wasted calls (7 to 14) instead of ~20; with real 800 ms timeouts, that is the difference between threads stuck for 6 s and instant fallbacks.

### 3.9.2 Retry, backoff, jitter, storm and availability math (`RetrySim.java`)
The full listing is short: delay = `min(cap, base * 2^(n-1))`, full jitter = `random()*exp`, 1000 clients failing at t=0 counted per 100 ms bucket, plus amplification and availability arithmetic. Actual output is in section 2.8 (backoff table, herd histogram) and section 2.3 (availability). Key numbers reproduced: no jitter -> 1000 requests re-arrive in one 100 ms bucket three separate times; full jitter -> spread, first bucket 556 then decaying; 3 layers x 3 attempts -> 27x; 5 x 99.9% -> 99.50%.

---

# PART 4 - Production war stories (symptom -> diagnosis -> fix)

### 4.1 Retry storm
- **Symptom:** a 30-second blip in the pricing service turned into a 25-minute outage of checkout; pricing CPU pegged long after the DB was healthy; request rate to pricing was 8x normal.
- **Diagnosis:** dashboards showed request rate up while success rate down; traces showed each user request spawning attempts at gateway (3), order-service Feign (3), pricing client (3) = up to 27. All fixed 1 s waits (no jitter) so waves hit together. Circuit breaker was inside retry with `slidingWindowSize=100, minimumNumberOfCalls=100` -> took too long to open.
- **Fix:** retry at one layer; exponential backoff + full jitter; retry budget; gateway retries GET only; breaker `minimumNumberOfCalls=10`, `slowCallDurationThreshold` set; load shedding on pricing (concurrency limit + 429 with `Retry-After`); alert on "retries as % of calls".

### 4.2 Cascading failure via thread pool exhaustion
- **Symptom:** whole site down; only one dependency (recommendations) was slow. All services' `/health` timing out; K8s restarting pods in a loop.
- **Diagnosis:** thread dump: 200/200 Tomcat threads WAITING in `SocketInputStream.read` on the recommendations call; the RestTemplate had no read timeout. Liveness probe hit the same exhausted pool -> restarts made it worse.
- **Fix:** connect/read timeouts, semaphore bulkhead per dependency (20), circuit breaker with fallback (empty recommendations), keep management port separate (`management.server.port`) so probes are not starved, liveness independent of downstreams.

### 4.3 Poison message
- **Symptom:** consumer lag on one partition grows to millions, others fine; logs repeat the same stack trace every few seconds; CPU normal.
- **Diagnosis:** one record with a malformed field (schema mismatch after an unannounced producer change) throws in the listener; default error handling retries the same offset forever, blocking the partition.
- **Fix:** `DefaultErrorHandler` with `ExponentialBackOff` (bounded) + `DeadLetterPublishingRecoverer` to `<topic>.DLT`; classify non-retryable exceptions (deserialization/validation) to skip retries; `ErrorHandlingDeserializer`; alert on DLT depth; schema registry compatibility checks in CI (contract tests); replay tool for DLT after fix (`Kafka.md`).

### 4.4 Duplicate payment
- **Symptom:** support tickets: "charged twice for one order"; two `payments` rows, 2 s apart, same order.
- **Diagnosis:** payment call timed out at 1 s (bank took 1.4 s and succeeded); Feign/Resilience4j retry re-POSTed without an idempotency key; also the Kafka consumer processed `ChargePayment` twice after a rebalance (no dedup).
- **Fix:** idempotency key = orderId sent to the bank and stored in a unique-constraint column `payments.order_id`; inbox table on the consumer; timeout treated as *unknown*, resolved by querying by key; nightly reconciliation of bank settlement vs `payments` with auto-refund of orphans; alert on duplicate charge attempts (the unique violations are a metric).

### 4.5 Stuck saga
- **Symptom:** ~0.3% of orders stay `PENDING` for hours; customers were charged in some cases.
- **Diagnosis:** query `saga_instance` for non-terminal states older than the SLA: stuck in `STOCK_RESERVED`. Payment service had consumed the command, but the `PaymentSucceeded` event was written to its outbox and the relay had a bug that dropped events larger than the Kafka `max.request.size` (payload included a large receipt blob) - exception swallowed. No deadline sweeper existed; no metric on saga age.
- **Fix:** relay fails loudly and retries (rows stay unpublished; alarm on oldest unpublished outbox row age); events carry ids not blobs; add saga deadlines + sweeper that queries Payment by orderId; dashboard `saga_age_seconds` by state; reconciliation job; runbook for `NEEDS_ATTENTION`.

### 4.6 Bonus: gateway retried a POST
- **Symptom:** duplicate orders only when the order service was deploying.
- **Diagnosis:** rolling deploy killed pods mid-request; gateway `Retry` filter had `methods: GET,POST` and re-sent to another pod; original request had actually committed.
- **Fix:** retry GET only; graceful shutdown + `preStop`; idempotency keys on POST /orders.

### 4.7 Bonus: Eureka stale instance / liveness restarts
- **Symptom:** 1-in-N calls fail after a redeploy for ~90 s.
- **Diagnosis:** clients hold cached registry with a dead instance (heartbeat eviction window + client cache refresh); calls to its IP time out.
- **Fix:** shorter lease/refresh intervals only helps partly; retry idempotent GETs on the next instance; readiness + graceful shutdown that deregisters first; on K8s use Services instead of Eureka.

---

# PART 5 - Interview questions (60)

Format: **Q (level)** - model answer. *Follow-ups* in italics with short answers. **Wrong answer** = the tempting mistake.

### A. Fundamentals and decomposition

**1. (Easy) What are microservices and why use them?**
Small, independently deployable services organized around business capabilities, each owning its data; buy independent releases, scaling, team autonomy, fault isolation. *Follow-up: what do you pay? -* network failure modes, eventual consistency, ops overhead, debugging. *Follow-up: when not? -* small team, unclear domain, no CI/CD/observability - use a modular monolith.
**Wrong answer:** "They make the application faster." (They add latency; the benefit is organizational/deployment.)

**2. (Easy) Monolith vs microservices?**
See table in the JavaFullstack note; key: deployment unit, data ownership, in-process vs network calls, team structure. *Follow-up: is a monolith bad? -* no; well-modularized monolith is often best until team/scale pressure appears.

**3. (Medium) What is Conway's law and why does it matter?**
System structure mirrors org communication structure. If two teams must change one service together, coupling follows. Design teams and boundaries together (inverse Conway); service ownership = team ownership. *Follow-up: what if boundaries and teams mismatch? -* constant cross-team PRs and coordinated releases -> distributed monolith.

**4. (Medium) How do you decide service boundaries?**
Bounded contexts from DDD/event storming, business capabilities, aggregates as consistency boundaries, team ownership, rate of change, data ownership. Avoid splitting by technical layer or by CRUD entity. *Follow-up: how do you know you got it wrong? -* chatty calls, lockstep deploys, cross-service transactions everywhere, circular deps.
**Wrong answer:** "One microservice per database table."

**5. (Medium) Explain aggregate and why service boundary >= aggregate boundary.**
Cluster changed atomically under one root and one transaction; invariants require single-DB ACID; splitting an aggregate across services forces distributed transactions for invariants. References between aggregates by ID.

**6. (Medium) How do you handle joins/reporting when each service has its own DB?**
API composition for small joins; CQRS read model built from events; local snapshot replication of needed fields; reporting via CDC/streaming to a warehouse; never share tables. *Follow-up: downside of composition? -* N calls, in-memory joins, pagination/sort across services is hard, availability multiplies.

**7. (Medium) Why is a shared "common" library a problem?**
Shared domain classes couple release cycles - a change forces coordinated upgrades. Share only technical starters/stable contracts; each service owns its DTOs; publish schemas rather than classes.

**8. (Hard) List smells of a distributed monolith.**
Lockstep deploys, shared DB, long sync chains, one failure kills all, cannot test in isolation, circular deps, chatty N+1 calls, the same people change all services. Fix by merging services (yes, merging is legitimate), fixing boundaries, async events, ownership.
**Wrong answer:** "It is fine as long as each is in its own repo."

**9. (Hard) State the 8 fallacies and map two to design decisions.**
Reliable network, zero latency, infinite bandwidth, secure, static topology, one admin, zero transport cost, homogeneous. Reliable -> timeouts/retries/idempotency/outbox; topology changes -> discovery/DNS; secure -> mTLS/zero trust.

### B. Communication and APIs

**10. (Easy) Sync vs async communication - when each?**
Sync when the caller needs the answer now (query, validation); async (events/commands via broker) for workflows, fan-out, spike smoothing, decoupling availability. *Follow-up: cost of async? -* eventual consistency, duplicates, ordering, harder debugging.

**11. (Medium) REST vs gRPC vs messaging?**
REST: universal, cacheable, browsers. gRPC: Protobuf + HTTP/2, low latency, strict contracts, streaming, deadlines; internal use; L7 balancing needed. Messaging: temporal decoupling, buffering, needs idempotency. *Follow-up: why gRPC and K8s Service load balancing issue? -* long-lived HTTP/2 connection pins to one pod; use headless service/client-side LB/mesh.

**12. (Medium) A request goes through 4 services each at 99.9% - what is availability?**
Multiply: 0.999^4 ~ 99.6% (~35 h downtime/year vs 8.8 h for one). Improve by fewer hops, async, optional dependencies with fallbacks, caching, redundancy.
**Wrong answer:** "99.9% - the weakest link." (Availability of serial dependencies multiplies; it is worse than the weakest.)

**13. (Hard) How should timeouts be set along a call chain?**
Decreasing downward: client > gateway > service A > service B > external; each per-attempt timeout x attempts plus backoff must fit in the caller's remaining budget; propagate deadlines; otherwise callers abandon (and retry) while callees still work. *Follow-up: what if the callee timeout is longer? -* orphaned work, duplicate effects, resource waste, retry storms.

**14. (Medium) How do you evolve an API without breaking clients?**
Additive changes only, tolerant readers, optional new fields, never repurpose fields, expand/contract, deprecate with metrics, version only when unavoidable, verify with consumer-driven contracts. *Follow-up: is adding an enum value safe? -* Not always - strict consumers may fail; document unknown handling.

**15. (Medium) What is consumer-driven contract testing? Pact vs Spring Cloud Contract?**
Consumers state expectations; provider CI verifies them - catches breaking changes without a shared E2E environment. Pact: consumer-generated JSON pacts, Broker, `can-i-deploy`, polyglot. SCC: provider-side DSL contracts, generated tests + stubs jar for consumer via StubRunner, JVM-centric. *Follow-up: contract test vs integration test? -* contracts check the interface shape/semantics per consumer with stubs, fast and isolated; integration tests check real wiring.

**16. (Easy) API versioning approaches?**
URI, header, query param, media type. Prefer compatible evolution; if needed run two versions with usage tracking and a sunset date.

**17. (Medium) What belongs in an API gateway and what does not?**
Belongs: routing, TLS, JWT validation, coarse authz, rate limiting, CORS, correlation headers, aggregation for BFF. Not: business rules, workflows, per-domain authorization, data ownership.

**18. (Medium) Explain Spring Cloud Gateway route, predicate, filter.**
Route = id + URI + predicates + filters. Predicates match request attributes (Path, Method, Header, Weight); filters modify request/response pre/post (StripPrefix, RequestRateLimiter, CircuitBreaker, Retry, TokenRelay); global filters apply to all; `lb://` resolves through LoadBalancer. WebFlux-based (Netty); MVC variant exists. *Follow-up: why not block in a gateway filter? -* event loop threads stall all requests.

**19. (Hard) How does Redis token-bucket rate limiting work in the gateway?**
Per key bucket: capacity `burstCapacity`, refill `replenishRate`/s, cost `requestedTokens`; a Lua script in Redis atomically refills and deducts; 429 when empty; shared across gateway instances; permits bursts but bounds average. Fail-open if Redis is unavailable by default. *Follow-up: token bucket vs leaky bucket vs fixed window? -* fixed window allows 2x burst at boundaries; sliding window is exact but costlier; leaky bucket smooths output rate; token bucket allows bursts.

**20. (Medium) Gateway vs service mesh vs BFF?**
Gateway: north-south edge concerns. Mesh: east-west via sidecars (mTLS, retries, traffic split, telemetry). BFF: per-client-type aggregation/shaping API owned by the frontend team. They coexist.

### C. Discovery and config

**21. (Easy) Client-side vs server-side discovery?**
Client asks registry and balances itself (Eureka + Spring Cloud LoadBalancer); server-side: fixed address, infrastructure balances (K8s Service, ALB). *Follow-up: on Kubernetes? -* use Service DNS; Eureka is redundant; mind L4 connection-level balancing for long-lived connections.

**22. (Medium) Explain Eureka behavior: heartbeats, self-preservation, stale entries.**
30 s heartbeats, 90 s eviction, client caches refreshed every 30 s; AP design; self-preservation stops eviction when heartbeat loss is massive; so stale instances are possible -> clients need timeouts/retries.

**23. (Medium) Spring Cloud Config vs Kubernetes ConfigMap/Secret?**
Config Server: Git-backed, profiles, encryption, refresh via `/actuator/refresh`/Bus; extra component. ConfigMap/Secret: native, GitOps-friendly; env vars need restart; Secrets need encryption at rest/external vault. On K8s prefer native; use feature flags for runtime toggles. *Follow-up: what does `@RefreshScope` do? -* re-creates annotated beans on refresh to rebind properties; does not affect singletons that copied values.

### D. Resilience

**24. (Easy) Why must every remote call have a timeout? Connect vs read?**
Otherwise threads block indefinitely -> pool exhaustion -> cascading failure. Connect = establish connection (short); read = wait for response bytes (based on callee p99 within budget). Also pool-acquire and DB timeouts. *Follow-up: does a timeout cancel the server work? -* No; the callee may still complete - hence idempotency.

**25. (Medium) Design retries properly.**
Only transient errors; idempotent ops (or idempotency key); exponential backoff with jitter; max attempts; retry budget; single layer; respect `Retry-After`; deadlines; log/metric each retry. *Follow-up: why jitter? -* de-synchronize clients to avoid thundering herd (simulation: 1000 clients re-arrive in one bucket vs spread).
**Wrong answer:** "Retry 5 times immediately on any exception."

**26. (Hard) What is retry amplification and how do you prevent it?**
Nested layers multiply attempts (3x3x3 = 27 calls); overloaded service gets more load and cannot recover (metastable failure). Prevent: retry at one layer, budgets, circuit breaker, backoff+jitter, mesh/gateway/app coordination, load shedding.

**27. (Medium) Explain the circuit breaker states and transitions.**
CLOSED counts outcomes in a sliding window; when failure rate or slow-call rate >= threshold (and >= minimumNumberOfCalls) -> OPEN: rejects with CallNotPermittedException for waitDurationInOpenState; then HALF_OPEN allows permittedNumberOfCallsInHalfOpenState trial calls; if their failure rate < threshold -> CLOSED else OPEN. Plus DISABLED/FORCED_OPEN.

**28. (Hard) Explain the Resilience4j circuit-breaker settings and how you would tune them.**
Sliding window type/size (count vs time), minimumNumberOfCalls (avoid tripping on tiny samples), failureRateThreshold, slowCallDurationThreshold/RateThreshold (trip on slowness before timeouts), waitDurationInOpenState (recovery time of the dependency), permittedNumberOfCallsInHalfOpenState, ignoreExceptions for business errors. Tune from real p99 and traffic volume; per dependency. *Follow-up: low-traffic service? -* time-based window or small minimumNumberOfCalls; beware flapping.

**29. (Hard) What is the default Resilience4j aspect order and why does it matter?**
Retry ( CircuitBreaker ( RateLimiter ( TimeLimiter ( Bulkhead ( function ))))). Retry outermost so each attempt is subject to and recorded by the breaker; the breaker's open state throws CallNotPermittedException into the retry (should be ignored); Bulkhead innermost. Configurable through aspect order properties. *Follow-up: what changes if the breaker wraps retry? -* the whole retry sequence counts as one call; the breaker sees fewer, more-final failures.
**Wrong answer:** stating the reverse (CircuitBreaker outside Retry) without qualification.

**30. (Medium) Semaphore vs thread-pool bulkhead?**
Semaphore: caps concurrency on caller thread; cheap; works with blocking/reactive. Thread-pool: separate pool + queue; caller thread freed; context-switch cost, thread-locals lost, needs CompletableFuture. Virtual threads make semaphore usually preferable, but the downstream still needs a cap.

**31. (Medium) What are fallbacks and what makes a good one?**
Cheaper/independent of the failing dependency: stale cache, default, degraded feature, deferred processing, clear error. Never fabricate success for writes. Distinguish exceptions (CallNotPermitted vs timeout vs business). Monitor fallback rate.

**32. (Medium) TimeLimiter - how does it differ from a client read timeout?**
TimeLimiter bounds a `CompletionStage`/future; annotation cannot forcibly stop a blocking synchronous call; real protection for blocking clients is the client read timeout. Use TimeLimiter for async/reactive.

**33. (Hard) Walk through a cascading failure and how each pattern stops it.**
(Use 2.8 story) slow dependency -> blocked threads -> pool exhaustion -> retries multiply -> readiness flap. Timeouts free threads; bulkhead isolates; breaker fails fast; fallback degrades; jittered budgeted retries prevent storm; load shedding protects self; half-open gentle recovery; liveness independent of dependencies.

**34. (Medium) Load shedding vs backpressure?**
Shedding: reject excess (429/503) to protect latency for admitted work, prioritize critical. Backpressure: signal producers to slow (reactive streams demand, bounded queues, consumer lag/prefetch). Unbounded queues hide overload and blow up latency/memory.

### E. Consistency, sagas, outbox

**35. (Easy) Why no ACID transactions across microservices? Why is 2PC discouraged?**
Separate databases; 2PC blocks with locks held, coordinator SPOF/in-doubt state, latency, availability multiplies, unsupported by many stores/brokers/HTTP. Use saga + eventual consistency.

**36. (Medium) Explain the Saga pattern with an example.**
Sequence of local transactions with compensations: create order -> reserve stock -> charge -> confirm; failure of payment triggers release stock + cancel order. No isolation -> semantic locks. *Follow-up: compensation vs rollback? -* compensation is a new business transaction (refund), visible to others; not a technical undo.

**37. (Medium) Choreography vs orchestration - choose?**
Choreography: no coordinator, loose, good for 2-4 simple steps; hard to see/track, cyclic dependencies. Orchestration: explicit flow, easy timeouts/monitoring/compensation, central component to keep thin and reliable. Choose orchestration when flows are long/branching or visibility matters. Engines: Temporal, Camunda, Axon, Step Functions.

**38. (Hard) Payment call timed out - what does the orchestrator do?**
Treat as unknown, not failed. Retry with the same idempotency key; or query payment by key/orderId; deadline; if confirmed captured proceed; if not found after bounded attempts void and compensate; reconciliation as safety net. *Follow-up: what if the bank does not support idempotency keys? -* use your own payment record keyed by orderId with a state machine and query the bank by merchant reference before re-charging; nightly settlement reconciliation.
**Wrong answer:** "Assume failure and cancel the order" (customer may be charged with no order).

**39. (Hard) What is a pivot transaction and retryable step?**
Pivot = go/no-go step; before it steps are compensatable, after it steps are retryable (must eventually succeed, idempotent) - e.g. confirm order, send email. Order steps: compensatable first, pivot, then retryable.

**40. (Hard) How do you deal with lack of isolation in sagas?**
Semantic locks (PENDING states, reservations), commutative updates, pessimistic view (reorder steps), reread value (optimistic check), versioning, by-value distributed lock for high risk. Example: `reserved` vs `available` stock.

**41. (Hard) What can go wrong with events in a saga (duplicates, out-of-order, lost) and defenses?**
Duplicates -> inbox dedup + natural idempotency; out-of-order -> state-machine guards, version numbers per aggregate; lost -> outbox durability + deadline sweeper + reconciliation; poison -> bounded retries + DLQ.

**42. (Easy) What is the dual-write problem?**
Updating DB and publishing to a broker as two separate operations; one can succeed and the other fail, causing divergence. `@Transactional` does not cover the broker.

**43. (Medium) Explain the Transactional Outbox and its guarantees.**
Event row inserted in the same transaction as the state change; relay (polling with `FOR UPDATE SKIP LOCKED` or Debezium CDC) publishes to the broker; marks published. Guarantee: at-least-once, ordered per aggregate if keyed; consumers need idempotency (inbox table). *Follow-up: polling vs CDC? -* polling simple/higher latency and DB load; CDC near real-time, infra heavy. *Follow-up: what about ordering with multiple pollers? -* SKIP LOCKED can reorder; partition claiming by aggregate hash or single relay; key Kafka by aggregateId. *Follow-up: why not just Kafka transactions? -* they cover Kafka-to-Kafka, not a separate DB commit.
**Wrong answer:** "Outbox gives exactly-once delivery."

**44. (Hard) CQRS and event sourcing - pros/cons, when?**
CQRS: separate write/read models for scale and tailored views; cost eventual consistency, projections. ES: events as source of truth: audit, replay, temporal queries; cost schema evolution, complexity, GDPR. Use where audit/history/complex read scaling matter; not for simple CRUD. ES is not Kafka usage.

**45. (Medium) How do you design idempotency keys for a payment API?**
Client UUID per logical operation; server stores key + request hash + status + response with unique constraint (scoped per client); winner processes, concurrent duplicates get 409/wait, completed replays stored response, different payload -> 422; expire after 24 h+; commit with business change; forward key downstream. *Follow-up: why not Redis SETNX only? -* not atomic with the DB write; crash windows.

**46. (Hard) Redis distributed lock - is it safe? Redlock debate?**
Single-instance `SET NX PX` + token-checked release is fine as an efficiency lock. TTL expiry during GC/network pause can lead to two holders; correctness needs fencing tokens checked by the resource. Kleppmann argued Redlock depends on timing assumptions and lacks fencing; antirez defended it. For correctness use DB constraints/row locks or a consensus system; ShedLock for scheduled jobs (`lockAtMostFor`/`lockAtLeastFor`).

**47. (Medium) Distributed ID options?**
DB sequence with hi-lo segments; UUIDv4 (poor index locality); UUIDv7 (time-ordered, RFC 9562, library needed on JDK 21); Snowflake (needs worker ids, clock issues). Choose for sortability/locality/coordination.

### F. Security, observability, deployment, architecture

**48. (Medium) How do you secure service-to-service calls?**
Edge JWT validation plus every service validates (zero trust); propagate user token or exchange it (RFC 8693) for narrower audience; client-credentials service accounts for system calls; mTLS (mesh) for identity/encryption; least privilege, network policies, secrets vault. *Follow-up: risks of relaying the same JWT everywhere? -* broad audience, replay, confused deputy.

**49. (Medium) Explain distributed tracing in Spring Boot 3.**
Micrometer Tracing + bridge (OTel/Brave) + exporter to Zipkin/Jaeger/OTLP; traceId/spanId in MDC; W3C `traceparent` header propagation via Boot-built clients; Kafka via record headers with observation enabled; sample rate; outbox must persist the trace context. *Follow-up: what replaced Sleuth? -* Micrometer Tracing.

**50. (Medium) Liveness vs readiness vs startup probe?**
Liveness: restart if wedged (no downstream checks); readiness: receive traffic or not; startup: delay other probes for slow starts. Wrong liveness (DB check) causes restart storms.

**51. (Medium) RED vs USE; SLI/SLO/error budget; how to alert?**
RED for services (Rate, Errors, Duration); USE for resources; SLI measurement, SLO target, budget = 1 - SLO; alert on burn rate/symptoms with runbooks, not on CPU.

**52. (Hard) How do you change a DB column in a rolling deployment?**
Expand/contract: add nullable new column, deploy dual-write, backfill, switch reads, later drop old after rollback window. Never rename/drop in one step. Migrations run as a separate job.

**53. (Hard) Backward vs forward compatibility for events?**
Backward: new consumer reads old events; forward: old consumer reads new events; use full compatibility with a schema registry for long-lived topics; additive optional fields; deprecate not rename.

**54. (Medium) Rolling vs blue-green vs canary; where do feature flags fit?**
Rolling: gradual replace, both versions live; blue-green: instant switch, instant rollback, double cost; canary: small % + metrics gate. Flags decouple deploy from release and provide kill switches.

**55. (Hard) Service mesh trade-offs?**
Pros: mTLS, retries/timeouts/outlier detection, traffic splitting, telemetry without code. Cons: latency and resource overhead, complexity, debugging, double retry amplification with app retries. Business fallbacks stay in the app.

**56. (Medium) Testing strategy for microservices?**
Many unit, integration with Testcontainers, contract tests instead of most E2E, few E2E/synthetic, resilience tests with WireMock/Toxiproxy, chaos experiments with hypotheses.

**57. (Hard) CAP and PACELC in service design?**
CAP: in partition choose C or A; PACELC adds latency vs consistency in normal operation. Choose per data: inventory/ledger CP, catalog/search AP; sagas embrace eventual consistency.

**58. (Hard) How do you migrate a monolith?**
Strangler fig with a routing facade, extract low-coupling seams, branch by abstraction, anti-corruption layer, data via CDC then cut, parallel runs, rollback plan; not big-bang.

**59. (Hard, scenario) Gateway retries a POST /orders after timeout - consequences and fixes?**
Duplicate orders. Retry GET only, idempotency keys on POST, retry at one layer, deadlines; return the original response on duplicate key.

**60. (Hard, scenario) An inventory event was lost - how does the system recover?**
Outbox prevents most loss; the saga deadline sweeper re-sends the idempotent command or queries; reconciliation compares orders vs reservations; alert on saga age and oldest unpublished outbox row; DLQ replay.

### Common wrong answers (quick list)
- "Circuit breaker retries the call" (it stops calls; Retry retries).
- "Use `@Transactional` across services / on the Kafka send to make it atomic."
- "Exactly-once delivery is achievable end to end" - achievable is effectively-once via idempotency.
- "Eureka is required for microservices" (not on K8s).
- "Spring Cloud Sleuth for Boot 3" (removed; Micrometer Tracing).
- "Timeout cancels the remote work."
- "Put all cross-service reads in one shared reporting DB fed by direct table access."
- "Liveness probe should check the database."
- "Sagas give rollback" (they give compensation, no isolation).
- "Retry everything including 400s/POSTs."

---

# PART 6 - One-page cheat sheet

```
DECIDE      modular monolith first; split for team autonomy / independent deploy; boundaries = bounded context/aggregate; DB per service
MATH        availability of sync chain = product;  timeouts shrink downward;  attempts x timeout + backoff <= caller budget
TIMEOUT     connect ~100-500ms, read ~ callee p99 (+margin); ALWAYS set; timeout != cancel
RETRY       transient only + idempotent; exp backoff + FULL JITTER; one layer; budget <=10%; GET only at gateway; amplification n^layers
CB          CLOSED -> (fail% or slow% >= thr, calls >= min) -> OPEN -> (wait) -> HALF_OPEN -> (trial ok) CLOSED | (fail) OPEN
            keys: slidingWindowType/Size, minimumNumberOfCalls, failureRateThreshold, slowCallDurationThreshold/RateThreshold,
                  waitDurationInOpenState, permittedNumberOfCallsInHalfOpenState, ignoreExceptions(business errors)
ORDER       Retry ( CircuitBreaker ( RateLimiter ( TimeLimiter ( Bulkhead ( fn )))))   (default; configurable)
BULKHEAD    semaphore (caller thread, cheap)  vs  thread-pool (CompletableFuture, own pool+queue)
FALLBACK    stale cache | default | degrade | pending/queue | honest 503;  never fake success on writes
SHED        bounded queues, 429/503 + Retry-After, priority; backpressure via demand/prefetch/lag
2PC         blocking, coordinator SPOF, availability multiplies -> avoid.  Use SAGA (local tx + compensation, no isolation)
SAGA        choreography (events, simple) | orchestration (state row + commands + timeouts; Temporal/Camunda/Axon/StepFn)
            order: compensatable -> PIVOT -> retryable;  semantic locks; deadlines + sweeper; guards for dup/out-of-order; unknown outcome != failure
OUTBOX      business row + outbox row in ONE tx; relay: polling (FOR UPDATE SKIP LOCKED) or Debezium CDC; at-least-once;
            key = aggregateId for ordering; consumer INBOX (PK message_id) in same tx as effect; store traceparent
IDEMPOTENCY Idempotency-Key + unique(client,key) + request hash + status + stored response; 409 in-progress, 422 mismatch; TTL 24h+
CQRS/ES     separate read/write models (eventual) / events as truth (audit, replay; schema evolution, GDPR pain)
LOCKS       prefer optimistic/unique constraints; Redis lock = efficiency only; fencing tokens for correctness; ShedLock for @Scheduled
IDS         UUIDv7 (sortable) | Snowflake (worker id) | DB sequence hi-lo | avoid UUIDv4 as clustered PK
GATEWAY     routes+predicates+filters; JWT validate, rate limit (Redis token bucket: replenishRate, burstCapacity, requestedTokens), BFF; no business logic
DISCOVERY   K8s Service DNS > Eureka on K8s;  Config: ConfigMap/Secret (+GitOps) > Config Server on K8s
SECURITY    zero trust: validate JWT in every service; token relay vs token exchange (RFC 8693); client_credentials; mTLS via mesh
OBSERVE     logs (JSON+MDC traceId) | metrics RED/USE (percentiles) | traces (Micrometer Tracing + OTel, W3C traceparent, Kafka headers)
            SLI/SLO/error budget; alert on burn rate; liveness (no deps) / readiness / startup
DEPLOY      rolling|blue-green|canary + feature flags; DB expand -> migrate -> contract; events: full compat via schema registry
MESH        sidecar mTLS/retries/traffic split; cost: latency, complexity, double-retries;  gateway = north-south, mesh = east-west
TEST        unit >> integration (Testcontainers) > contract (Pact/SCC) > few E2E; WireMock/Toxiproxy; chaos with hypotheses
CAP/PACELC  partition: C or A; else latency vs consistency; choose per data
MIGRATE     strangler fig + branch by abstraction + ACL; CDC for data; never big-bang
ANTI        distributed monolith, shared DB/lib, nano-services, no timeouts, retry everywhere, dual write, gateway business logic, DB liveness probe
```
