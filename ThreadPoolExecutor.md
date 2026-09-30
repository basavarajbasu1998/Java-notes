# ThreadPoolExecutor Internals, Coordination Utilities, `@Async`

`Executors.newFixedThreadPool()` hides the real parameters. Real one:

```java
new ThreadPoolExecutor(
    corePoolSize,      // threads always kept alive          e.g. 4
    maximumPoolSize,   // upper limit when queue is full     e.g. 8
    keepAliveTime, unit, // idle time before extra threads die
    workQueue,         // holds waiting tasks                e.g. ArrayBlockingQueue(100)
    threadFactory,     // name threads ("order-worker-1")
    rejectedHandler    // when everything is full
);
```

**Analogy:** Restaurant. 4 permanent waiters (core). Queue = waiting line at the door (100 seats). If line is full, call 4 temporary waiters (up to max 8). If even that is full → turn customers away (rejection).

## Task submission flow (MOST ASKED)

```
submit(task)
    │
    ▼
Running threads < corePoolSize ? ──Yes──► create NEW thread, run task
    │No
    ▼
Queue has space? ──Yes──► put task in queue (a core thread picks it later)
    │No
    ▼
Running threads < maximumPoolSize ? ──Yes──► create NEW extra thread, run task
    │No
    ▼
REJECT  (RejectedExecutionHandler)
```
⚠ Extra threads (beyond core) are created only **after the queue is full** – people often think the opposite.

With an **unbounded** queue (`LinkedBlockingQueue` default), queue never fills → max is never used, and memory can blow up.

## Rejection policies
| Policy | Behaviour |
|---|---|
| `AbortPolicy` (default) | throws `RejectedExecutionException` |
| `CallerRunsPolicy` | calling thread runs the task → natural back-pressure |
| `DiscardPolicy` | silently drops |
| `DiscardOldestPolicy` | drops oldest queued task, retries |

## Why `Executors.newFixedThreadPool` is discouraged
- `newFixedThreadPool` / `newSingleThreadExecutor` → **unbounded queue** → OOM.
- `newCachedThreadPool` → max = `Integer.MAX_VALUE` threads → OOM.
Always create `ThreadPoolExecutor` manually with a **bounded queue**.

## Sizing the pool
- CPU-bound (calculation): threads ≈ `cores + 1`.
- IO-bound (DB, HTTP): threads ≈ `cores × (1 + waitTime/computeTime)` – often 2×–10× cores.

## Coordination utilities
| Tool | Analogy | Use |
|---|---|---|
| `CountDownLatch(n)` | Race start: wait until n runners are ready. One-time. | Main waits for n workers to finish |
| `CyclicBarrier(n)` | Tour group: everyone waits at the gate, then all go together. Reusable. | Phased parallel computation |
| `Semaphore(n)` | Parking lot with n slots | Limit concurrent access (rate-limit DB calls) |
| `Exchanger` | Two people swap items | rare |

```java
CountDownLatch latch = new CountDownLatch(3);
for (int i = 0; i < 3; i++)
    pool.submit(() -> { callService(); latch.countDown(); });
latch.await();     // main blocks until all 3 finish
```

## `@Async` in Spring
```java
@EnableAsync
@Bean Executor taskExecutor() { ...ThreadPoolTaskExecutor with core/max/queue... }

@Async("taskExecutor") public CompletableFuture<String> send() {...}
```
Traps: same self-invocation proxy trap as `@Transactional`; default executor may be unbounded; exceptions in `void` async methods are lost (need `AsyncUncaughtExceptionHandler`).

## Graceful shutdown
`shutdown()` = stop accepting, finish queued. `shutdownNow()` = interrupt running, return queued list. Then `awaitTermination()`.
