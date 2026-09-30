# Scenario / System-Design Questions
Answer structure: **1) clarify requirements → 2) high-level flow → 3) data model → 4) scale & failure → 5) trade-offs.**

## Design a URL shortener
```
POST /shorten {url}
   ▼
Generate unique id (auto-increment / Snowflake ID) → Base62 encode → "aZ3k9"
   ▼
Save (code → longUrl) in DB;  cache hot ones in Redis
GET /aZ3k9 → cache lookup → miss → DB → HTTP 302 redirect
```
Read-heavy (100:1) → cache + read replicas; shard by code hash; collision-free via id encoding.

## Rate limiter (token bucket)
```
Bucket holds N tokens, refills r tokens/sec.
Request arrives → token available? ──Yes──► take one, allow
                                   └─No──► 429 Too Many Requests
```
Distributed: keep counters in **Redis** (`INCR` + `EXPIRE`, or Lua script for atomicity), keyed by userId/IP.

## Prevent double payment / duplicate messages
Idempotency key + unique DB constraint (see `REST_API_Design.md`); consumer idempotency for Kafka/RabbitMQ; **outbox pattern** so DB write + event publish are consistent (in your RabbitMQ notes).

## Cache strategies (Redis)
- **Cache-aside (most common):** read cache → miss → read DB → put in cache. Write: update DB, then **delete** cache.
- Problems: **Cache penetration** (queries for non-existent keys → cache null / Bloom filter), **cache breakdown** (hot key expires, stampede → mutex/lock or refresh early), **cache avalanche** (many keys expire together → random TTL jitter).
- Consistency: cache is eventually consistent; keep TTL short.

## Microservice resilience checklist
Timeout on every remote call → retry with exponential backoff + jitter (only idempotent calls) → **circuit breaker** (Resilience4j) → bulkhead (isolate thread pools) → fallback → rate limit. Link: your Circuit Breaker + Saga notes.

## "API is slow in production – what do you do?"
```
Measure first (APM / metrics / traces) ─► which layer?
  ├─ DB: EXPLAIN, index, N+1, pagination, connection pool
  ├─ Downstream call: timeout, circuit breaker, parallelize with CompletableFuture, cache
  ├─ JVM: GC pauses, thread dump (blocking), CPU profile
  └─ Infra: CPU/memory limits, network, too few pods
Fix biggest bottleneck → load-test again.
```

## Monolith → microservices migration
Strangler-fig: put a gateway in front, extract one bounded context at a time (own DB), route traffic gradually; use events/Saga for cross-service consistency.
