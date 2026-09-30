# Production Skills: OOM, High CPU, Logging, Docker/CI Basics

## OutOfMemoryError debugging flow
```
Alert: app slow / crashed with OutOfMemoryError
   ▼
Which OOM?
   ├─ "Java heap space"        → too many objects / leak
   ├─ "GC overhead limit"      → GC using 98% time, freeing little (leak)
   ├─ "Metaspace"              → too many classes loaded (classloader leak, dynamic proxies)
   ├─ "unable to create native thread" → too many threads
   └─ "Direct buffer memory"   → NIO/Netty off-heap
   ▼
Get evidence:  -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps
              jmap -dump:live,format=b,file=heap.hprof <pid>
   ▼
Open in Eclipse MAT / VisualVM → "Dominator tree" / "Leak suspects"
   ▼
Find object retaining the memory (static Map that only grows, unclosed connections,
ThreadLocal not removed, listeners never unregistered, huge cache w/o eviction)
   ▼
Fix, and set limits (bounded cache, -Xmx)
```

## Common memory-leak sources in Java
Static collections, unbounded caches (use Caffeine/Guava with `maximumSize` + TTL), unclosed resources (use try-with-resources), `ThreadLocal` in pools, mutable keys in HashMap, inner classes holding outer reference.

## High CPU / hang flow
```
top -H -p <pid>          → find hottest thread id (decimal)
printf "%x" <tid>        → convert to hex
jstack <pid> | grep -A20 <hex>   → see what that thread is doing
```
Take 3 thread dumps 5 s apart; threads always in the same method = culprit; many `BLOCKED` = lock contention/deadlock.

## Useful JVM flags & tools
`-Xms -Xmx` (heap), `-XX:+UseG1GC`, `-Xlog:gc*`; tools: `jps`, `jstat -gc`, `jstack`, `jmap`, `jcmd`, VisualVM, JFR.
G1 (default 9+): region-based, predictable pause. ZGC: sub-millisecond pauses, huge heaps. (Link with your GC notes.)

## Logging
- SLF4J (API) + Logback (implementation). Levels: TRACE < DEBUG < INFO < WARN < ERROR.
- Use placeholders: `log.info("User {} logged in", id)` (not string concat).
- **Never log passwords/tokens/PII.**
- **Correlation/Trace ID** in **MDC** so one request can be followed across microservices (Sleuth/Micrometer Tracing).
- Log exception once with stack trace: `log.error("msg", ex)` – don't log and rethrow repeatedly.

## Docker + CI/CD basics
```dockerfile
FROM eclipse-temurin:21-jre
COPY target/app.jar app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```
```
Developer push → CI (build, unit tests, Sonar, security scan)
   → build Docker image → push to registry
   → deploy to staging (integration tests)
   → deploy to prod (rolling / blue-green / canary)
   → monitor (health, metrics, logs)  → rollback if bad
```
- **Image vs container**: image = recipe/template; container = running instance.
- **Kubernetes**: Pod (smallest unit), Deployment (replicas, rolling update), Service (stable network address), ConfigMap/Secret, HPA (autoscale), liveness (restart if dead) vs readiness (send traffic only when ready).
- **Blue-green** (two envs, switch traffic) vs **canary** (5% traffic first).
