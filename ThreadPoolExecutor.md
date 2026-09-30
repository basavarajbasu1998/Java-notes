# ThreadPoolExecutor, ForkJoin, CompletableFuture and Virtual Threads - Deep Interview Notes

Scope: JDK 17 / 21 source-level behaviour of `java.util.concurrent` executors, plus Spring integration, sizing, monitoring and production failure modes. Memory model, locks, AQS internals and `ConcurrentHashMap` live in other notes; here AQS appears only where `Worker` uses it.

Everything marked "Observed output" was produced by compiling and running the code shown, on JDK 21.0.4 (12 logical cores). Your thread numbers will differ; the shapes will not.

---

## Table of contents
1. 60-second mental model
2. Deep internals (ctl, execute, addWorker, Worker, runWorker, getTask, exit, shutdown)
3. Queue choice, rejection, ThreadFactory, Executors pitfalls
4. submit vs execute, FutureTask, ScheduledThreadPoolExecutor
5. Sizing pools rigorously, bulkheads, monitoring
6. Spring: ThreadPoolTaskExecutor and @Async traps
7. ForkJoinPool and parallel streams
8. CompletableFuture in depth
9. Virtual threads (Java 21)
10. Runnable demos with observed output
11. Production war stories
12. Interview questions (45) with follow-ups and wrong answers
13. One-page cheat sheet

---

# 1. 60-second mental model

A thread pool is **N long-lived worker threads pulling from a queue**. `ThreadPoolExecutor` (TPE) decides, per submitted task, one of four things, in this order:

```
1. fewer than corePoolSize workers?   -> start a new worker that runs THIS task first
2. else can the queue accept it?      -> enqueue (an existing worker will take it)
3. else fewer than maximumPoolSize?   -> start a non-core worker that runs THIS task first
4. else                               -> reject (handler)
```

**Analogy - a restaurant.** Core = permanent waiters. Queue = the waiting line at the door. Max = the total staff you are allowed to call in on a rush night. You only call temp staff **when the line is full**, not when it starts to form. If line and staff are both full, the host turns people away (rejection) or, with CallerRuns, makes the customer serve themselves.

Three sentences that separate seniors from juniors:
1. "Threads beyond core are created only after the queue is **full**; with an unbounded queue, `maximumPoolSize` is dead configuration."
2. "`submit()` swallows the task's exception into the `Future`; `execute()` lets it kill the worker (and the pool replaces it)."
3. "A pool is a **capacity limit and a failure-isolation boundary**; size it from Little's law, not by guessing, and give every slow dependency its own."

---

# 2. Deep internals

## 2.1 The `ctl` word: state and count in one atomic int

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;      // 29
private static final int COUNT_MASK = (1 << COUNT_BITS) - 1; // 0x1FFFFFFF  (~536 million workers max)

private static final int RUNNING    = -1 << COUNT_BITS;  // 111 + 29 zeros
private static final int SHUTDOWN   =  0 << COUNT_BITS;  // 000
private static final int STOP       =  1 << COUNT_BITS;  // 001
private static final int TIDYING    =  2 << COUNT_BITS;  // 010
private static final int TERMINATED =  3 << COUNT_BITS;  // 011
```

The **top 3 bits are runState**, the **low 29 bits are workerCount**. Packing both means "is the pool still accepting AND how many workers are there" can be read and CAS-updated as **one atomic operation** - no lock needed on the hot path.

```
 bit31 30 29 | 28 ......................................... 0
 [ runState ]| [           workerCount (29 bits)            ]

 Example: RUNNING with 2 workers
   RUNNING = 0xE0000000 (-536870912);  ctl = -536870910
   runStateOf(c)   = c & ~COUNT_MASK   -> 0xE0000000
   workerCountOf(c)= c &  COUNT_MASK   -> 2
```

Because RUNNING is negative and the states are numerically ordered, comparisons like `runStateAtLeast(c, STOP)` and `c < SHUTDOWN` ("is running") are single integer compares.

### State machine

```
RUNNING --shutdown()--------> SHUTDOWN --(queue empty AND workers==0)--+
   |                                                                     v
   +------shutdownNow()-----> STOP ----(workers==0)----------------> TIDYING --terminated() hook--> TERMINATED
                               ^
                 SHUTDOWN --shutdownNow()
```

| State | Accepts new tasks | Runs queued tasks | Interrupts running tasks |
|---|---|---|---|
| RUNNING | yes | yes | no |
| SHUTDOWN | no | **yes (drains queue)** | only idle workers are interrupted (to wake them) |
| STOP | no | **no** (queue drained into a returned list) | **yes** |
| TIDYING | - | - | all workers gone, `terminated()` about to run |
| TERMINATED | - | - | `terminated()` finished; `awaitTermination` returns |

## 2.2 `execute()` - exact decision flow

```java
public void execute(Runnable command) {
    if (command == null) throw new NullPointerException();
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {                 // step 1
        if (addWorker(command, true)) return;
        c = ctl.get();                                     // lost a race, re-read
    }
    if (isRunning(c) && workQueue.offer(command)) {        // step 2
        int recheck = ctl.get();
        if (!isRunning(recheck) && remove(command))        //   pool shut down meanwhile
            reject(command);
        else if (workerCountOf(recheck) == 0)              //   all workers died meanwhile
            addWorker(null, false);                        //   start a worker with no first task
    }
    else if (!addWorker(command, false))                   // step 3
        reject(command);                                   // step 4
}
```

ASCII flowchart:

```
execute(cmd)
   |
   v
workerCount < core? --yes--> addWorker(cmd, core=true) --ok--> return
   | no (or CAS lost)                         | fail
   v                                          v
isRunning && queue.offer(cmd)? ---- no ---> addWorker(cmd, core=false) --ok--> return
   | yes                                                     | fail
   v                                                         v
recheck ctl                                              reject(cmd)
   |-- not running AND remove(cmd) --> reject(cmd)
   |-- workerCount == 0 -------------> addWorker(null,false)   (safety net)
   '-- else done (a worker will poll it)
```

**Why the re-check after enqueue?** Between the `isRunning` check and `offer`, two things can happen: (a) `shutdown()` runs - the task would sit in a queue nobody drains, so we `remove` and reject; (b) all workers time out or die - the task would be stranded, so we start a worker that has no first task and will simply `getTask()` from the queue. That `addWorker(null, false)` is also how a pool with `corePoolSize = 0` (e.g. cached pool with a non-synchronous queue) ever starts its first thread.

Three consequences interviewers love:
- **Task 5 can overtake tasks 3 and 4.** Once the queue is full, a new non-core worker gets the *new* task as its `firstTask`, bypassing queued ones. Queue order is not global order (see demo 10.1).
- **Core threads are created lazily**, one per submission, even if existing core threads are idle. That is why `getPoolSize()` climbs 1, 2, ... on the first `core` tasks. `prestartAllCoreThreads()` starts them eagerly.
- With `LinkedBlockingQueue` (unbounded), step 2 always succeeds, so step 3 is unreachable.

## 2.3 `addWorker(firstTask, core)`

Two phases: (1) reserve a slot with CAS; (2) create and start the thread.

```java
private boolean addWorker(Runnable firstTask, boolean core) {
  retry:
  for (int c = ctl.get();;) {
    // Refuse if pool is beyond SHUTDOWN, or SHUTDOWN but this is a new client task
    // (SHUTDOWN still allows a task-less worker to drain a non-empty queue)
    if (runStateAtLeast(c, SHUTDOWN)
        && (runStateAtLeast(c, STOP) || firstTask != null || workQueue.isEmpty()))
        return false;
    for (;;) {
        if (workerCountOf(c) >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))
            return false;                         // at limit
        if (compareAndIncrementWorkerCount(c)) break retry;  // slot reserved
        c = ctl.get();
        if (runStateAtLeast(c, SHUTDOWN)) continue retry;
    }
  }
  Worker w = null; boolean workerStarted = false, workerAdded = false;
  try {
    w = new Worker(firstTask);                    // calls threadFactory.newThread(w)
    final Thread t = w.thread;
    if (t != null) {
      mainLock.lock();
      try {
        int c = ctl.get();
        if (isRunning(c) || (runStateLessThan(c, STOP) && firstTask == null)) {
            if (t.getState() != Thread.State.NEW) throw new IllegalThreadStateException();
            workers.add(w);                       // HashSet<Worker>, guarded by mainLock
            workerAdded = true;
            largestPoolSize = Math.max(largestPoolSize, workers.size());
        }
      } finally { mainLock.unlock(); }
      if (workerAdded) { t.start(); workerStarted = true; }
    }
  } finally {
    if (!workerStarted) addWorkerFailed(w);       // decrement count, remove, tryTerminate
  }
  return workerStarted;
}
```

Points to state in an interview:
- The CAS reserves the **count** first; the thread object comes later. If `ThreadFactory` returns null or `t.start()` throws (e.g. `OutOfMemoryError: unable to create native thread`), `addWorkerFailed` rolls the count back.
- The `workers` set is a plain `HashSet` under `mainLock`. It is **not** touched on the task hot path, only on worker birth/death and shutdown.
- `firstTask == null` means "start a worker just to poll the queue" (used by the recheck, by `processWorkerExit`, by `prestart*`).

## 2.4 `Worker` - why it extends AQS

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    final Thread thread;
    Runnable firstTask;
    volatile long completedTasks;

    Worker(Runnable firstTask) {
        setState(-1);                          // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }
    public void run() { runWorker(this); }

    protected boolean tryAcquire(int unused) { // NON-reentrant: 0 -> 1 only
        if (compareAndSetState(0, 1)) { setExclusiveOwnerThread(Thread.currentThread()); return true; }
        return false;
    }
    protected boolean tryRelease(int unused) { setExclusiveOwnerThread(null); setState(0); return true; }
    public void lock() { acquire(1); }  public boolean tryLock() { return tryAcquire(1); }
    public void unlock() { release(1); }
    void interruptIfStarted() {            // used by shutdownNow
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted())
            try { t.interrupt(); } catch (SecurityException ignore) {}
    }
}
```

Worker uses AQS as a **cheap, non-reentrant "I am running a task" flag**:

| Purpose | How |
|---|---|
| Tell idle from busy | The worker holds its own lock **while executing a task**. `interruptIdleWorkers()` (called by `shutdown()`, `setCorePoolSize`, etc.) does `w.tryLock()` first: success => worker is idle (blocked in `getTask`) => safe to interrupt. Failure => it is running a task, leave it alone. |
| Don't interrupt a task by accident | `shutdown()` must interrupt only workers waiting on the queue, never a worker mid-task. Holding the lock during `task.run()` guarantees that. |
| Non-reentrant on purpose | If it were a `ReentrantLock`, a task calling a pool-control method such as `setCorePoolSize()` (which calls `interruptIdleWorkers`) would re-acquire its own lock and look "idle", getting itself interrupted. |
| `state = -1` initially | `interruptIfStarted` ignores state < 0, so a worker cannot be interrupted before `runWorker` unlocks it (state -> 0). |

## 2.5 `runWorker` loop

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask; w.firstTask = null;
    w.unlock();                                    // state -1 -> 0: interrupts now allowed
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();                              // "busy" flag
            // If pool is STOP+, ensure thread interrupted; else clear any stale interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) && !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                try {
                    task.run();
                    afterExecute(task, null);
                } catch (Throwable ex) {
                    afterExecute(task, ex);
                    throw ex;                      // rethrown: worker DIES
                }
            } finally {
                task = null; w.completedTasks++; w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

Numbered trace of one worker's life:
1. Thread starts, `run()` -> `runWorker(w)`. Takes `firstTask`, unlocks (interruptible).
2. Loop condition: run `firstTask`, otherwise call `getTask()` (blocks on the queue).
3. `w.lock()` - marks busy.
4. Interrupt hygiene: if pool is STOP or beyond, make sure the thread carries the interrupt flag (so the task can observe it). Otherwise **clear a stale interrupt** left over from an earlier `shutdown()` wake-up or a previous task, so it does not poison this task.
5. `beforeExecute(thread, task)` hook. If it throws, the task is **not** run and the worker dies.
6. `task.run()`; then `afterExecute(task, null)` or `afterExecute(task, ex)` and **rethrow**.
7. `finally`: null the task, bump `completedTasks`, unlock.
8. Rethrow escapes the `while`, `completedAbruptly` stays true, `processWorkerExit` runs, the thread terminates and the JVM calls its `UncaughtExceptionHandler`.

Hooks: `beforeExecute`/`afterExecute`/`terminated` are `protected` empty methods for subclasses - use them for timing, MDC set/clear, and metrics. For a `submit()`-ed task, `afterExecute`'s `Throwable` is **null** because `FutureTask` caught it; to see it, check `r instanceof Future<?>` and call `get()` (that is the documented idiom).

## 2.6 `getTask()` - keep-alive, timed poll vs take

```java
private Runnable getTask() {
    boolean timedOut = false;
    for (;;) {
        int c = ctl.get();
        // Pool shutting down and (STOP or queue empty): this worker should exit
        if (runStateAtLeast(c, SHUTDOWN) && (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
            decrementWorkerCount(); return null;
        }
        int wc = workerCountOf(c);
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;   // can THIS worker time out?
        if ((wc > maximumPoolSize || (timed && timedOut)) && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c)) return null;        // retire
            continue;
        }
        try {
            Runnable r = timed ? workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS)
                               : workQueue.take();
            if (r != null) return r;
            timedOut = true;                                           // poll timed out
        } catch (InterruptedException retry) { timedOut = false; }
    }
}
```

Key facts:
- **Workers are not "core" or "non-core" labelled.** Any worker is a candidate for retirement when `wc > corePoolSize`. Whoever's poll times out first while the count is above core exits. There is no permanent identity for core threads.
- Under-core workers use `take()` (block forever). Above-core workers use `poll(keepAlive)`. Setting `allowCoreThreadTimeOut(true)` (requires `keepAliveTime > 0`) makes **all** workers time out, so an idle pool shrinks to zero threads.
- The extra condition `(wc > 1 || workQueue.isEmpty())` prevents the last worker from retiring while tasks remain.
- `InterruptedException` here is how `shutdown()` wakes idle workers; the loop re-checks the state and exits.
- **SynchronousQueue + keepAlive** is why the cached pool's threads die after 60 s idle: `wc > core (0)` so every worker polls with timeout.

## 2.7 `processWorkerExit` - replacement after exceptions

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    if (completedAbruptly) decrementWorkerCount();      // normal exit already decremented in getTask
    mainLock.lock();
    try { completedTaskCount += w.completedTasks; workers.remove(w); } finally { mainLock.unlock(); }
    tryTerminate();
    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && !workQueue.isEmpty()) min = 1;
            if (workerCountOf(c) >= min) return;        // enough workers, don't replace
        }
        addWorker(null, false);                         // REPLACE the dead worker
    }
}
```

Trace when `execute(() -> { throw new X(); })`:
1. `task.run()` throws -> `afterExecute(task, X)` -> rethrown.
2. `completedAbruptly == true` -> count decremented, worker removed from set.
3. Not STOP -> `addWorker(null, false)`: a **brand-new thread** (new name, empty ThreadLocals) replaces it.
4. The old thread dies and the default handler prints the stack trace on `System.err` (or your `UncaughtExceptionHandler` runs). Demo 10.2 shows `pool-worker-1` dying and `pool-worker-2` taking over.

Costs: thread churn (creation, lost ThreadLocal caches) if tasks throw constantly via `execute()`. Benefit: the pool is self-healing - one bad task never permanently shrinks it.

## 2.8 `shutdown` vs `shutdownNow` vs `awaitTermination` vs `close`

| Call | State change | Queued tasks | Running tasks | Blocks? |
|---|---|---|---|---|
| `shutdown()` | RUNNING -> SHUTDOWN | **still executed** | keep running; idle workers interrupted so they notice | no |
| `shutdownNow()` | -> STOP | drained and **returned** as `List<Runnable>` (never run) | `interrupt()` sent to every started worker; tasks that ignore interrupts keep going | no |
| `awaitTermination(t,u)` | none | - | - | yes, until TERMINATED or timeout; returns boolean |
| `close()` (Java 19+, `ExecutorService` is `AutoCloseable`) | `shutdown()` then loops `awaitTermination(1 day)`; if the calling thread is interrupted it calls `shutdownNow()` | drained | - | yes |

```java
// canonical graceful shutdown (this is basically what the Javadoc example does)
pool.shutdown();
try {
    if (!pool.awaitTermination(30, TimeUnit.SECONDS)) {
        List<Runnable> dropped = pool.shutdownNow();
        log.warn("dropped {} queued tasks", dropped.size());
        if (!pool.awaitTermination(10, TimeUnit.SECONDS)) log.error("pool did not terminate");
    }
} catch (InterruptedException e) {
    pool.shutdownNow();
    Thread.currentThread().interrupt();       // preserve the flag
}
```

Gotchas:
- `shutdown()` does not wait. Without `awaitTermination` your `main` may proceed while tasks still run.
- `shutdownNow()` is **cooperative**: a task in `while(true){compute();}` never sees the interrupt and the pool never terminates. Tasks must check `Thread.interrupted()` or call blocking methods that throw `InterruptedException`.
- Never calling `shutdown()` on a pool of non-daemon threads keeps the JVM alive forever. (Cached pool threads eventually die after 60 s; fixed pool threads never do.)
- `tryTerminate()` moves SHUTDOWN -> TIDYING only when queue empty and workerCount 0, then runs `terminated()`, then TERMINATED and signals `awaitTermination` waiters.

---

# 3. Queue choice, rejection, ThreadFactory, Executors pitfalls

## 3.1 Queue choice determines behaviour

| Queue | Bounded | TPE behaviour | Use / danger |
|---|---|---|---|
| `SynchronousQueue` | capacity 0 (hand-off) | `offer` succeeds only if a worker is **already waiting in `poll`/`take`**. Otherwise step 3 makes a new thread up to max. | Cached pool. With max unbounded => thread explosion under load. |
| `LinkedBlockingQueue()` | unbounded (Integer.MAX_VALUE) | Step 2 always wins; **max is never used**; only core threads exist | Fixed pool. Backlog grows silently -> OOM and unbounded latency. |
| `LinkedBlockingQueue(n)` | bounded | Standard TPE behaviour; two locks (put/take) => good throughput | General-purpose |
| `ArrayBlockingQueue(n)` | bounded, array, one lock | Standard; optional fairness flag; less allocation per task | Predictable memory |
| `PriorityBlockingQueue` | unbounded, heap | Tasks ordered by comparator. **Unbounded => max unused.** Tasks submitted via `submit()` are wrapped in `FutureTask`, which is **not Comparable** -> `ClassCastException`; use `execute()` with your own `Comparable` Runnable or override `newTaskFor`. Starvation of low priority. | Priority jobs |
| `DelayedWorkQueue` | unbounded (internal) | Used by `ScheduledThreadPoolExecutor` | Timers |
| `LinkedTransferQueue` | unbounded | Rarely used with TPE | - |

Behaviour cheat: **"a pool with a queue only grows past core when the queue rejects"**. A common trick to prefer threads before queueing (Tomcat's `TaskQueue`) is a queue subclass whose `offer` returns false when `poolSize < max`, so TPE creates threads first. It works by subclassing `offer`; mention it if asked "how do I make a pool scale before queueing".

## 3.2 Rejection policies and real use

| Policy | Behaviour | When to use |
|---|---|---|
| `AbortPolicy` (default) | throws `RejectedExecutionException` to the caller of `execute/submit` | Fail fast; callers must handle it (HTTP 503) |
| `CallerRunsPolicy` | caller thread runs the task itself (unless pool shut down, then discards) | **Back-pressure**: the producer is slowed because it is busy doing the work |
| `DiscardPolicy` | silently drops | Only for truly optional work (metrics sampling). A silent-loss risk |
| `DiscardOldestPolicy` | polls (drops) the head of the queue and retries `execute` | "Latest wins" (e.g. UI refresh). With PriorityBlockingQueue it discards the **highest** priority - a trap |
| custom | log + metric + persist to DLQ | Production default |

CallerRuns real use: a Kafka/consumer thread or a scheduler thread feeds a pool; when full, that thread executes work itself and therefore stops polling. **Danger**: if the caller is a Tomcat request thread, that request now does work in-line (ok, backpressure); if the caller is an event-loop thread (Netty), you have just blocked the loop. Also CallerRuns runs a task while the caller might hold locks.

```java
// custom handler: count, log, then apply backpressure
RejectedExecutionHandler h = (r, ex) -> {
    rejectedCounter.increment();
    log.warn("pool {} saturated: active={} queue={}", name, ex.getActiveCount(), ex.getQueue().size());
    if (!ex.isShutdown()) r.run();                 // same as CallerRuns but observable
};
```

## 3.3 ThreadFactory: naming and uncaught exceptions

```java
static ThreadFactory named(String prefix) {
    AtomicInteger n = new AtomicInteger();
    return r -> {
        Thread t = new Thread(r, prefix + "-" + n.incrementAndGet());
        t.setDaemon(false);                                    // decide consciously
        t.setUncaughtExceptionHandler((th, e) -> log.error("uncaught in {}", th.getName(), e));
        return t;
    };
}
```

- The default factory names threads `pool-N-thread-M`. In a thread dump of 20 pools that tells you **nothing**. Name every pool after its dependency (`payments-http-3`). Guava's `ThreadFactoryBuilder` or Spring's `CustomizableThreadFactory` do this.
- `UncaughtExceptionHandler` fires only for exceptions escaping `execute()`-ed tasks (worker death). It **never fires for `submit()`**.
- `setDaemon(true)` for background pools you do not want to block JVM exit, `false` for pools whose tasks must finish (drain with `shutdown`).

## 3.4 `Executors` factory pitfalls (source-level)

| Factory | Actual construction | Trap |
|---|---|---|
| `newFixedThreadPool(n)` | `TPE(n, n, 0L, MILLISECONDS, new LinkedBlockingQueue<>())` | unbounded queue; OOM; latency grows without bound; no rejection ever |
| `newSingleThreadExecutor()` | same with n=1, wrapped so it cannot be cast/reconfigured | same; a task exception replaces the thread, order preserved |
| `newCachedThreadPool()` | `TPE(0, Integer.MAX_VALUE, 60s, new SynchronousQueue<>())` | unbounded threads; thread explosion when downstream slows -> `unable to create native thread`, memory exhaustion |
| `newScheduledThreadPool(n)` | `ScheduledThreadPoolExecutor(n)` | unbounded `DelayedWorkQueue`; max threads irrelevant |
| `newWorkStealingPool()` | `ForkJoinPool` with parallelism = cores | async mode, daemon threads; not for blocking |
| `newVirtualThreadPerTaskExecutor()` | new virtual thread per task, no pool | no queue, no limit: add a `Semaphore` if the downstream needs protection |

Alibaba's guidelines and every static analyser forbid the first three. Rule: construct `ThreadPoolExecutor` directly with a **bounded queue, a named ThreadFactory, and an explicit handler**.

---

# 4. submit vs execute, FutureTask, scheduling

## 4.1 submit() vs execute()

```java
public Future<?> submit(Runnable task) {
    RunnableFuture<Void> ftask = newTaskFor(task, null);   // wraps in FutureTask
    execute(ftask);
    return ftask;
}
```

`FutureTask.run()` catches `Throwable` and stores it via `setException`. So:

| | `execute(Runnable)` | `submit(Runnable/Callable)` |
|---|---|---|
| Exception in task | propagates out of `run` -> worker dies -> UEH prints -> worker replaced | captured in `FutureTask`, **nothing logged**, worker survives |
| Where you see it | stderr / UEH | only when you call `future.get()` (as `ExecutionException`) |
| `afterExecute` Throwable | non-null | null (unwrap the Future) |
| Result | none | `Future` |

The classic bug: `pool.submit(() -> doWork()); // fire and forget` - failures vanish. Fixes: (a) use `execute` for fire-and-forget; (b) wrap the body in try/catch that logs; (c) always consume the Future; (d) for `CompletableFuture`, add `.exceptionally/whenComplete` logging. Demo 10.2 proves it.

## 4.2 FutureTask states

```
NEW -> COMPLETING -> NORMAL          (result set)
NEW -> COMPLETING -> EXCEPTIONAL     (task threw)
NEW -> CANCELLED                     (cancel(false))
NEW -> INTERRUPTING -> INTERRUPTED   (cancel(true))
```

- State is a `volatile int`, transitions by CAS; waiters form a Treiber stack and are woken by `finishCompletion()`.
- `get()` on NORMAL returns; EXCEPTIONAL -> `ExecutionException(cause)`; CANCELLED/INTERRUPTED -> `CancellationException`; timed `get` -> `TimeoutException` (**the task keeps running**; a timeout does not cancel).
- `cancel(true)` interrupts the runner thread - which is your pool worker - and only stops the task if it cooperates. `cancel(false)` on a running task marks it cancelled but lets it finish.
- Java 19+ adds `Future.resultNow()`, `exceptionNow()`, `state()` - non-blocking inspection, throwing `IllegalStateException` if not in the right state.
- A cancelled task still sitting in the queue stays there (holding memory) until dequeued, unless the queue is purged (`purge()`); for scheduled tasks use `setRemoveOnCancelPolicy(true)`.

## 4.3 ScheduledThreadPoolExecutor

Extends TPE, always uses an unbounded `DelayedWorkQueue` (a heap ordered by trigger time, then sequence number). It behaves like a fixed pool of **core** threads (max is ignored: the queue never rejects, so step 3 is never reached).

```java
ses.schedule(task, 5, SECONDS);                       // once
ses.scheduleAtFixedRate(task, 0, 10, SECONDS);        // next = previous SCHEDULED start + period
ses.scheduleWithFixedDelay(task, 0, 10, SECONDS);     // next = previous END + delay
```

| | scheduleAtFixedRate | scheduleWithFixedDelay |
|---|---|---|
| Anchor | start-to-start | end-to-start |
| Task takes longer than period | next run starts **immediately after** the previous ends (never concurrently), so runs are late and can "catch up" back to back | gap always equals delay |
| Use for | clock-aligned, must-keep-rate jobs (heartbeat) | polling / cleanup where spacing matters |

**Critical rule:** if a periodic task throws (any Throwable), **all subsequent executions are silently suppressed** - the `ScheduledFuture` completes exceptionally and nobody is told. Always wrap the body in `try { ... } catch (Throwable t) { log }`. Also a single scheduler thread doing slow work delays every other job: give the scheduler enough threads or hand work off to another pool.

`@Scheduled` in Spring uses a `ThreadPoolTaskScheduler` with **pool size 1 by default** (Boot sets `spring.task.scheduling.pool.size=1`): two long jobs serialise. Configure the pool size. Spring's scheduled wrappers do log and continue for `@Scheduled` (the error handler logs and lets the trigger continue), unlike raw `scheduleAtFixedRate`.

---

# 5. Sizing pools rigorously; bulkheads; monitoring

## 5.1 Little's law

```
L = lambda * W
L      = average number of tasks in the system (in flight)
lambda = arrival rate (tasks per second)
W      = average time a task spends in the system (service time + waiting)
```

Applied to a pool: the number of **concurrently busy threads needed** = arrival rate x service time.

Worked example: an endpoint calls a payment service. Traffic 300 req/s, payment latency mean 80 ms.
- L = 300 x 0.08 = **24 concurrent calls**. A pool of 24 threads is exactly at 100% utilisation: any burst queues, and queues at 100% utilisation grow without bound.
- Target utilisation 60-70% for variance headroom: threads = 24 / 0.65 ~ **37**.
- If the p99 is 400 ms (slow-down scenario), demand becomes 300 x 0.4 = 120 concurrent: with 37 threads throughput collapses to 37 / 0.4 = **92 req/s**. The queue absorbs the rest, and waiting time explodes. This is the entire mechanism of "slow downstream took the service down".
- **Queue length** should be derived from a latency budget: if you can afford 200 ms of waiting and the pool drains at 37/0.08 = 460 tasks/s, a queue of ~90 covers 200 ms. Anything bigger just stores requests that the client has already timed out on.

## 5.2 Formulas

- **CPU-bound** (pure computation, no waiting): threads = cores (or cores + 1 as a cushion for page faults). More threads only add context switching and cache misses.
- **Mixed / IO-bound** (Brian Goetz, *Java Concurrency in Practice*): 

```
threads = cores * targetUtilisation * (1 + W / C)
W = wait time (blocked on IO), C = compute time per task

8 cores, U = 0.8, W = 90 ms, C = 10 ms  ->  8 * 0.8 * (1 + 9) = 64 threads
```

- W and C come from measurement (tracing spans, profiler), **not guesses**. Also cap by the downstream's limit: 64 threads calling a database whose connection pool is 20 gives 44 threads waiting for connections. **The smallest limit in the chain sets the real concurrency** (thread pool -> HTTP client pool -> DB pool -> DB CPU).
- Total memory: each platform thread reserves stack (default 1 MB virtual on Linux x64, `-Xss`), committed lazily; thousands of threads cost real RSS plus kernel scheduling overhead.
- Validate under load; monitor queue depth and saturation; resize.

## 5.3 Bulkheads: one pool per dependency

A ship has watertight compartments so one hole does not sink it. If `payments`, `inventory`, and `email` all share one pool of 50 and `email` starts taking 30 s per call, within seconds all 50 threads are blocked in `email`, and payment calls queue behind them: **a slow non-critical dependency takes down the critical path**.

```
Shared pool:   [ pay pay email email email email ... ]   -> email hogs everything
Bulkheaded:    payments(20) | inventory(10) | email(5, bounded queue, reject -> skip)
```

Rules: separate pools (or `Semaphore` bulkheads / Resilience4j `Bulkhead`) per downstream; small bounded queue; fail fast on rejection; timeouts on every call; critical work never shares with best-effort work. The same logic is why the **ForkJoin common pool must never host blocking work** (section 7).

## 5.4 Monitoring

TPE getters (approximate, taken under `mainLock` or from atomic counters - snapshots, not transactions):

| Method | Meaning | Alert on |
|---|---|---|
| `getPoolSize()` | current workers | pinned at max |
| `getActiveCount()` | workers currently running a task (approx.) | == max for long |
| `getQueue().size()` | tasks waiting | sustained growth, > 70% of capacity |
| `getCompletedTaskCount()` | finished tasks (approx.) | rate drops to 0 while queue > 0 |
| `getTaskCount()` | scheduled ever (approx.) | - |
| `getLargestPoolSize()` | high-water mark | shows if max ever reached |
| `getQueue().remainingCapacity()` | space left | near 0 |

Micrometer:

```java
ExecutorService monitored = ExecutorServiceMetrics.monitor(registry, tpe, "payments", Tags.of("tier", "critical"));
```

Publishes (names as of Micrometer 1.x): `executor.pool.size`, `executor.active`, `executor.queued`, `executor.queue.remaining`, `executor.completed`, `executor.pool.core`, `executor.pool.max`, and timers `executor.seconds` (execution) and `executor.idle` (time in queue before start). Spring Boot auto-binds executor beans it finds (e.g. `applicationTaskExecutor`) when Micrometer is present. Alert on **queue wait time** (`executor.idle`) - it is the user-visible symptom - rather than only on thread count.

---

# 6. Spring: ThreadPoolTaskExecutor and @Async traps

```java
@Configuration @EnableAsync
class AsyncConfig implements AsyncConfigurer {
    @Bean("mailExecutor")
    ThreadPoolTaskExecutor mailExecutor() {
        var e = new ThreadPoolTaskExecutor();
        e.setCorePoolSize(4); e.setMaxPoolSize(8); e.setQueueCapacity(50);
        e.setThreadNamePrefix("mail-");
        e.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        e.setTaskDecorator(new ContextCopyingDecorator());
        e.setWaitForTasksToCompleteOnShutdown(true);
        e.setAwaitTerminationSeconds(30);
        return e;                       // initialize() called by the container (afterPropertiesSet)
    }
    @Override public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> log.error("async {} failed", method, ex);
    }
}
```

`ThreadPoolTaskExecutor` is a thin wrapper around a real TPE with the same queue-then-max semantics. **`queueCapacity` defaults to `Integer.MAX_VALUE`**, so if you set `maxPoolSize` but forget `queueCapacity`, max is never used - the same unbounded-queue trap.

## 6.1 Which executor does @Async use?

| Setup | Executor |
|---|---|
| Plain Spring Framework, `@EnableAsync`, no `Executor` bean, no `AsyncConfigurer` | `SimpleAsyncTaskExecutor`: **a new thread per task, no reuse, concurrency unbounded** by default (`setConcurrencyLimit` opt-in). Trap: load spike -> thousands of threads |
| Spring Boot (auto-config) | Bean `applicationTaskExecutor` (`ThreadPoolTaskExecutor`), defaults `core=8`, `max=Integer.MAX_VALUE`, `queue-capacity=Integer.MAX_VALUE`, `keep-alive=60s`, prefix `task-`, core threads allowed to time out. Effective behaviour: **8 threads and an unbounded queue** |
| Boot 3.2+ with `spring.threads.virtual.enabled=true` | `applicationTaskExecutor` becomes a `SimpleAsyncTaskExecutor` running **virtual threads** (`@Async`, `@Scheduled`, Tomcat/Jetty also switch) |
| Boot with your own `Executor` bean | Boot's auto-config backs off; if several `Executor` beans exist, `@Async` needs a qualifier or a bean named `taskExecutor` |

Boot properties: `spring.task.execution.pool.core-size`, `max-size`, `queue-capacity`, `keep-alive`, `allow-core-thread-timeout`, `spring.task.execution.thread-name-prefix`, `spring.task.execution.shutdown.await-termination` and `await-termination-period`.

## 6.2 @Async traps

1. **Self-invocation** bypasses the proxy: `this.asyncMethod()` runs synchronously. Call it from another bean. (Same for `@Transactional`.)
2. **`void` async methods lose exceptions** unless `AsyncUncaughtExceptionHandler` is set; return `CompletableFuture<T>` instead so callers can handle failures.
3. **Transactions do not propagate**: the transaction is bound to the caller's thread via `ThreadLocal`. The async method starts its own (or none). Never pass a lazily loaded entity to an `@Async` method: the Hibernate session is gone (`LazyInitializationException`). Pass IDs. Conversely, `@Async` called inside a transaction may run **before the caller commits** and not see the data: use `@TransactionalEventListener(phase = AFTER_COMMIT)` or `TransactionSynchronization.afterCommit`.
4. **SecurityContext is a ThreadLocal** (default strategy `MODE_THREADLOCAL`): in a pool thread `SecurityContextHolder.getContext()` is empty. Wrap the executor with `DelegatingSecurityContextAsyncTaskExecutor` / `DelegatingSecurityContextExecutorService`, or a `TaskDecorator` that copies the context. `MODE_INHERITABLETHREADLOCAL` is wrong for pools (copies only at thread creation, then serves stale identities to later tasks).
5. **MDC (logging correlation ids)** is also a ThreadLocal: without propagation, async logs lose `traceId`.
6. **Request scope** beans/`RequestContextHolder` are not available in the pool thread.

One decorator solves MDC and security together:

```java
class ContextCopyingDecorator implements TaskDecorator {
    public Runnable decorate(Runnable task) {
        Map<String,String> mdc = MDC.getCopyOfContextMap();            // captured on the CALLER thread
        SecurityContext sc = SecurityContextHolder.getContext();
        return () -> {
            Map<String,String> prev = MDC.getCopyOfContextMap();
            SecurityContext prevSc = SecurityContextHolder.getContext();
            try {
                if (mdc != null) MDC.setContextMap(mdc); else MDC.clear();
                SecurityContextHolder.setContext(sc);
                task.run();
            } finally {                                                // ALWAYS restore: pool threads are reused
                if (prev != null) MDC.setContextMap(prev); else MDC.clear();
                SecurityContextHolder.setContext(prevSc);
            }
        };
    }
}
```

The `finally` restoring/clearing is mandatory: otherwise the next unrelated task on that worker sees the previous user's identity (a security incident). Micrometer's `context-propagation` library and Spring Boot's `ContextPropagatingTaskDecorator` (newer Boot versions) automate this.

## 6.3 Graceful shutdown in Spring / Kubernetes

Order of events at pod termination: K8s sends SIGTERM -> readiness starts failing/endpoint removal (asynchronous!) -> Spring begins shutdown -> SIGKILL after `terminationGracePeriodSeconds` (default 30).

- Web: `server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase=25s` (stop accepting new requests, finish in-flight).
- Pools: `setWaitForTasksToCompleteOnShutdown(true)` + `setAwaitTerminationSeconds(n)` (or Boot's `spring.task.execution.shutdown.await-termination=true`). Otherwise Spring calls `shutdownNow`-style behaviour and in-flight `@Async` work is interrupted.
- Budget: `terminationGracePeriodSeconds` > web timeout + pool await time + margin. Add a short `preStop` sleep (5-10 s) so the load balancer stops sending traffic before shutdown starts.
- Queued, un-started tasks are lost on SIGKILL: anything that must not be lost belongs in a durable queue (Kafka, DB outbox), not in an in-memory pool queue.

---

# 7. ForkJoinPool and parallel streams

## 7.1 Work stealing

A `ForkJoinPool` is a TPE-like pool tuned for **many small, CPU-bound, recursively split tasks**. Each worker has its own **deque**:

```
worker A deque:  base -> [t1][t2][t3][t4] <- top
   owner pushes and pops at the TOP  (LIFO: hot cache, small sub-tasks first)
   thieves steal from the BASE       (FIFO: the oldest = largest sub-task, so a steal gets a lot of work)
```

- No single shared queue -> low contention. Idle workers scan others' deques and steal.
- `fork()` = push to my deque; `join()` = if the child is not done, **help**: run it myself or steal other work rather than blocking (this is why fork/join can run recursive splits with few threads).
- Contrast with TPE: TPE has one queue with a lock; FJP suits fine-grained tasks, TPE suits coarse independent tasks.

```java
class Sum extends RecursiveTask<Long> {
    protected Long compute() {
        if (hi - lo <= THRESHOLD) return sequentialSum();
        Sum left = new Sum(a, lo, mid), right = new Sum(a, mid, hi);
        left.fork();                   // schedule left asynchronously
        return right.compute() + left.join();   // compute right in THIS thread, then join left
    }
}
```

Fork one, compute the other yourself (do not fork both then join both - wastes the current thread). Choose a threshold so the leaf costs 10-100 microseconds or more; below that, task overhead dominates. `RecursiveAction` = no result. Tasks must be side-effect-free and non-blocking.

## 7.2 The common pool

`ForkJoinPool.commonPool()` is a JVM-wide shared pool used by parallel streams, `CompletableFuture` `*Async` methods without an executor, and `ForkJoinTask.fork()` outside any pool.
- Parallelism defaults to **`availableProcessors() - 1`** (the submitting thread is expected to participate when it joins). Demo output: 12 cores -> parallelism 11. Override with `-Djava.util.concurrent.ForkJoinPool.common.parallelism=N`.
- If parallelism computes to 1 or less, `CompletableFuture` **does not use the common pool** and creates a **new thread per async task** (`ThreadPerTaskExecutor`). Small containers (1 CPU limit) hit this: surprising thread-per-task behaviour.
- Container caveat: the JVM derives cores from cgroup CPU limits (JDK 10+), so a pod limited to 2 CPUs on a 64-core node gets parallelism 1.
- Common-pool threads are **daemon**: a `main` that exits while `supplyAsync` tasks run just loses them.

## 7.3 Why blocking in the common pool starves everything

11 workers for the whole JVM. If 11 tasks each block 2 s on `HttpClient.send`, then **every parallel stream and every default-executor `CompletableFuture` in the application waits** - including unrelated endpoints and libraries (the common pool is shared by libraries you did not write). Symptom: unrelated features become slow whenever one call is slow; thread dump shows all `ForkJoinPool.commonPool-worker-N` parked in socket read.

Escape hatches:
- Give blocking work its own executor (`supplyAsync(fn, ioPool)`).
- Use `ForkJoinPool.managedBlock(ManagedBlocker)`: tells the pool a worker is about to block, so the pool may add a compensation thread (`CompletableFuture.get/join` inside FJP and `Phaser` use this). It is for library authors; it lets parallelism temporarily exceed the target, not an excuse to block in the common pool.
- Or virtual threads for blocking IO.

## 7.4 Parallel stream pitfalls

1. **Shared common pool** - see above; a slow stage in one request slows all.
2. **Only worth it for**: large N, CPU-heavy per-element work, cheap-to-split source (`ArrayList`, arrays, `IntStream.range`), stateless non-blocking functions. For 1 000 elements of trivial work, parallel is **slower** (split + merge + thread wake).
3. **Poor splitters**: `LinkedList`, `Stream.iterate`, `BufferedReader.lines()` split badly and mostly run sequentially or with large overhead.
4. **Ordering**: `forEach` on a parallel stream has no defined order (`forEachOrdered` restores it at the cost of parallelism). `findFirst`/`limit` on an ordered stream are more expensive in parallel; `findAny` and `unordered()` are cheaper.
5. **Stateful / side-effecting lambdas are bugs**: `list.parallelStream().forEach(x -> result.add(x))` on an `ArrayList` loses elements or throws. Use `collect(Collectors.toList())` (safe) or `toConcurrentMap`.
6. **reduce** requires an associative accumulator and a true identity; `reduce(0, (a,b) -> a - b)` gives garbage in parallel.
7. **Exceptions** from a parallel stage surface from the terminal operation, possibly one of several.
8. Running the stream inside your own `ForkJoinPool.submit(() -> stream.parallel()...)` makes it use that pool, but this is an implementation detail, not a specified contract.
9. `ThreadLocal`/MDC/tx context does not follow into common-pool workers.

---

# 8. CompletableFuture in depth

`CompletableFuture<T>` = a `Future` + a `CompletionStage` (a dependency graph of callbacks) + manual completion (`complete`, `completeExceptionally`). It is a **push** model: callbacks run when the result arrives; no thread is blocked *if you never call `join/get`*.

## 8.1 Which thread runs my callback?

| Method form | Executor |
|---|---|
| `supplyAsync(fn)` / `runAsync(fn)` | `ForkJoinPool.commonPool()` (or thread-per-task if parallelism <= 1). Java 9+: overridable in subclass through `defaultExecutor()` |
| `supplyAsync(fn, executor)` | your executor |
| `thenApply / thenAccept / thenRun / thenCompose / thenCombine / whenComplete / handle / exceptionally` (non-async) | **the thread that completes the source stage**, or **the caller's thread if the source is already complete when you attach the callback** |
| `thenApplyAsync(fn)` etc. | common pool (default async executor) |
| `thenApplyAsync(fn, ex)` | `ex` |

Observed (demo 10.3): callback attached to a still-running stage ran on `io-thread` (the completing thread); attached to an already-completed stage it ran on `main`; `*Async` without executor ran on `ForkJoinPool.commonPool-worker-1`.

Consequences:
- A non-async callback that does heavy work **hijacks the IO thread that completed the source** (e.g. an HTTP client's selector thread), stalling all other completions. Use `*Async(fn, cpuPool)` for non-trivial callbacks.
- Nondeterminism: whether a callback runs on your thread depends on a race - so a `ThreadLocal` may or may not be present. Never rely on it.

## 8.2 Composition operators

| Operator | Purpose | Shape |
|---|---|---|
| `thenApply(f)` | sync map | `T -> U` (like `Stream.map`) |
| `thenCompose(f)` | dependent async step (flatMap) | `T -> CompletableFuture<U>`; avoids `CF<CF<U>>` |
| `thenCombine(other, bf)` | join two independent results | both complete -> `bf(a,b)` |
| `thenAccept / thenRun` | terminal side effect | no result |
| `allOf(cfs...)` | wait for all | returns `CF<Void>`; you read each future after; completes **exceptionally only after ALL have finished** if any failed |
| `anyOf(cfs...)` | first to complete | `CF<Object>`; the first completion wins **even if it is an exception** |
| `applyToEither / acceptEither` | first of two | - |

## 8.3 Exception propagation

- If a stage fails, dependents complete exceptionally with a **`CompletionException` wrapping the original**. `join()` throws `CompletionException`; `get()` throws `ExecutionException`. Both have the real cause in `getCause()`.
- Demo: `exceptionally(t -> ...)` on the `supplyAsync` stage itself received a `CompletionException` (the async runner wraps a thrown exception) - so **always unwrap** (`t instanceof CompletionException ? t.getCause() : t`) before `instanceof` checks. If you call `completeExceptionally(ex)` directly on a future, `exceptionally` **on that same future** sees raw `ex`, but **downstream** stages see it wrapped.

| Method | Called when | Can change result? | Sees |
|---|---|---|---|
| `exceptionally(fn)` | only on failure | yes: supplies a fallback value | `Throwable` |
| `handle((r, t) -> x)` | success **or** failure | yes: returns new value | both; `t` null on success |
| `whenComplete((r, t) -> {})` | success or failure | **no**: passes the original outcome (or original exception) along; if the callback itself throws, that exception is used only when the source succeeded | both |
| `exceptionallyCompose` (12+) | failure | async fallback | - |

Guideline: `whenComplete` for logging/metrics; `handle` for recover-or-transform; `exceptionally` for a simple fallback. Placement matters: an `exceptionally` **before** later stages recovers and lets the chain continue; an `exceptionally` **at the end** only sees whatever reaches it.

## 8.4 Timeouts (Java 9+)

```java
cf.orTimeout(2, SECONDS);                       // fails with TimeoutException if not complete in time
cf.completeOnTimeout(defaultValue, 2, SECONDS); // completes with fallback instead
CompletableFuture.delayedExecutor(1, SECONDS);  // schedule a delayed async stage
```

Implemented with a single shared daemon `Delayer` scheduler thread; the timeout only completes the **future**. The underlying work (the HTTP call) **keeps running** and consuming a thread until it finishes on its own - you must also set a client-side timeout. Java 8 has neither: you must build it with a scheduler.

## 8.5 Why `join()`/`get()` inside callbacks deadlocks the pool

```java
// bounded pool with 2 threads
CompletableFuture.supplyAsync(() -> {
    return CompletableFuture.supplyAsync(() -> "inner", pool).join();   // blocks a worker waiting for another worker
}, pool);
```

With all workers occupied by "outer" stages that wait for "inner" stages queued behind them, nothing can progress: **starvation deadlock** (no lock cycle, just resource exhaustion; demo 10.4 with plain `Future`). Fix: **chain, do not block**: `thenCompose(x -> supplyAsync(inner, pool))`. Blocking `join()` is only for the outermost caller (request thread / `main`).

## 8.6 Cancellation caveat

`cf.cancel(true)` completes the future with `CancellationException`, but:
- it **does not interrupt** the running supplier (the `mayInterruptIfRunning` flag has no effect on CompletableFuture);
- it is **not propagated upstream**: cancelling a dependent does not cancel the source; cancelling the source makes dependents fail with `CompletionException(CancellationException)`.

So for real cancellation, run the work on a `Future` you control (`executor.submit`) and link them, or use structured concurrency (section 9.8).

## 8.7 Fan-out / fan-in with timeouts, without blocking pool threads

```java
ExecutorService io = /* bounded, named pool */;

CompletableFuture<User>    u = supplyAsync(() -> userSvc.get(id),     io).orTimeout(800, MILLISECONDS);
CompletableFuture<Orders>  o = supplyAsync(() -> orderSvc.list(id),   io).orTimeout(800, MILLISECONDS)
                                   .exceptionally(t -> Orders.EMPTY);          // degrade gracefully
CompletableFuture<Recs>    r = supplyAsync(() -> recSvc.top(id),      io).completeOnTimeout(Recs.EMPTY, 300, MILLISECONDS);

Page page = u.thenCombine(o, Page::withOrders)
             .thenCombine(r, Page::withRecs)
             .orTimeout(1, SECONDS)                     // overall SLA
             .join();                                   // only the outermost caller blocks
```

Generic list version:

```java
List<CompletableFuture<Item>> fs = ids.stream()
    .map(id -> supplyAsync(() -> fetch(id), io).orTimeout(1, SECONDS).exceptionally(t -> null))
    .toList();
CompletableFuture.allOf(fs.toArray(CompletableFuture[]::new)).join();
List<Item> ok = fs.stream().map(CompletableFuture::join).filter(Objects::nonNull).toList();
```

Rules: per-branch timeout **and** overall timeout; decide per branch whether failure degrades or fails the whole; bounded pool with bulkhead; do not `join` inside pool tasks.

---

# 9. Virtual threads (Java 21)

## 9.1 How they work

A **virtual thread** (`java.lang.VirtualThread`, JEP 444, final in Java 21) is a `Thread` scheduled by the JVM, not the OS. Its stack lives on the heap as a growable chain of **stack chunk objects**.

```
   virtual threads (millions)        carrier = platform threads in a dedicated ForkJoinPool
   V1 V2 V3 V4 V5 V6 ...             (parallelism = cores by default;
        |   |                          -Djdk.virtualThreadScheduler.parallelism / maxPoolSize, default max 256)
   mount|   |mount
        v   v
      [carrier-1] [carrier-2] ... [carrier-N]  -> OS threads -> CPU cores
```

1. `Thread.startVirtualThread` / executor submit: the continuation is scheduled on the carrier pool.
2. A carrier **mounts** it (copies frames onto the carrier's stack lazily) and runs it.
3. When it does blocking IO, `Thread.sleep`, `LockSupport.park`, or blocks in a `java.util.concurrent` lock/queue, the JDK **parks** the virtual thread: its frames are copied back to the heap (**unmount**) and the carrier is free to run another virtual thread.
4. When the IO is ready (`epoll` via the JDK's poller threads) or `unpark` occurs, the continuation is re-scheduled and mounted on **any** carrier (may be a different one).

Result: thread-per-request code style (blocking, simple, debuggable stack traces) with near-async scalability. Demo 10.6: 10 000 tasks each sleeping 1 s finish in ~1.5-2.4 s using 12 carriers; a 200-thread platform pool needs ~50 s.

Properties: always **daemon**; priority fixed; cheap to create (hundreds of bytes to KBs initially); `Thread.isVirtual()`; visible in `jcmd <pid> Thread.dump_to_file -format=json out.json` (a normal `jstack` shows only carriers/platform threads).

## 9.2 Pinning

A virtual thread is **pinned** (cannot unmount; blocks its carrier) when it blocks while:
- inside a **`synchronized` block/method** (JDK 21-23: the monitor is tied to the carrier stack), or
- a **native method / foreign function** frame is on its stack.

If it blocks (IO, `sleep`, `Lock.lock()` contention on a monitor) while pinned, the carrier thread is blocked too. Few pinned threads are harmless; many pinned + blocked exhaust the carriers (parallelism ~ cores) and the application freezes.

- Detect on JDK 21: `-Djdk.tracePinnedThreads=full` (or `short`) prints a stack when a thread blocks while pinned; JFR event `jdk.VirtualThreadPinned`.
- Mitigate: replace `synchronized` around **blocking** operations with `ReentrantLock` (which parks properly); keep synchronized sections short and non-blocking; upgrade libraries (older JDBC drivers, some HTTP clients had synchronized-with-IO).
- **JDK 24 (JEP 491) removes most synchronized-related pinning**: virtual threads can unmount while blocked inside `synchronized`. Native/foreign frames still pin, and `Object.wait()` behaviour differs by version - so on JDK 21 (the LTS you probably run) you must still treat `synchronized` + blocking as a pinning risk.

## 9.3 Why NOT to pool virtual threads

Pools exist to (a) bound scarce, expensive threads and (b) reuse creation cost. Virtual threads are neither scarce nor expensive: creating one is roughly the cost of an object. **One virtual thread per task**; use `Executors.newVirtualThreadPerTaskExecutor()` (not a pool - just a factory plus lifecycle, it is `AutoCloseable`). Pooling them also breaks the mental model (ThreadLocals accumulate across tasks). The way to **limit concurrency** is to limit the *resource*, with a `Semaphore`:

```java
Semaphore dbPermits = new Semaphore(50);          // downstream can take 50 concurrent calls
try (var ex = Executors.newVirtualThreadPerTaskExecutor()) {
    for (Request r : requests) ex.submit(() -> {
        dbPermits.acquire();                      // virtual thread parks cheaply while waiting
        try { return db.query(r); } finally { dbPermits.release(); }
    });
}
```

The semaphore is your bulkhead now. Without it, 100k virtual threads = 100k simultaneous queries against your database.

## 9.4 Details and caveats

- **ThreadLocal works** (each virtual thread has its own), but with millions of threads a heavy ThreadLocal (a 1 MB buffer, `SimpleDateFormat`) multiplies. Prefer passing values explicitly; `ScopedValue` (below) is the intended replacement. Do not use ThreadLocal for **pooling** objects across tasks.
- **CPU-bound work gains nothing**, and virtual threads are **not time-sliced**: a CPU-bound virtual thread never yields and holds its carrier until it blocks or ends, so a few CPU hogs starve the others. Use a bounded platform pool / ForkJoinPool for CPU-bound tasks.
- Blocking `synchronized`, file IO (regular file reads are not yet unmounting on many platforms; they compensate by temporarily adding carrier capacity), and native calls remain "carrier-blocking".
- Memory: each parked virtual thread keeps its stack chunk on heap: 100k threads with deep stacks = real heap use.
- Debug/observability: thread dumps get large; names default empty - set names (`Thread.ofVirtual().name("req-", 0)`).
- Do not rely on thread identity for capacity signals: `Thread.activeCount`, thread-count based autoscaling no longer represents load.

## 9.5 Spring Boot 3.2+

`spring.threads.virtual.enabled=true` (needs Java 21): Tomcat/Jetty request handling, `applicationTaskExecutor` (`@Async`), `@Scheduled` (`SimpleAsyncTaskScheduler`) and other Boot-managed executors use virtual threads. Controllers stay plain blocking code. Watch: DB connection pool (Hikari 10 by default) becomes the new bottleneck - virtual threads simply queue on `getConnection`; keep timeouts. Watch: synchronized in libraries; ThreadLocal-heavy frameworks.

## 9.6 When platform threads / other models still win

| Scenario | Prefer |
|---|---|
| CPU-bound parallelism | bounded platform pool or ForkJoinPool (threads = cores) |
| Heavy `synchronized` + blocking on JDK 21 | platform threads or refactor to `ReentrantLock` |
| Need streaming backpressure, operators, million-connection push (websocket fan-out, gateways) | reactive (Reactor / WebFlux) |
| Existing blocking code, request/response services, IO fan-out | **virtual threads** |
| Need strict priority/affinity | platform threads |

**Reactive vs virtual:** reactive gives backpressure, composition operators, and a tiny thread footprint but at the cost of a different programming model, hard stack traces, and viral non-blocking requirements. Virtual threads give the scalability of blocking IO with ordinary code and stack traces, but no built-in backpressure (use semaphores/bounded queues) and no operator library. Mixed workloads: keep reactive where streaming semantics matter.

## 9.7 Structured concurrency and scoped values - status

- **`StructuredTaskScope`** (JEP 453): **preview in Java 21** (needs `--enable-preview`); API has changed between previews - do not put it in production code yet on 21. Idea: forked subtasks live and die within a scope; `ShutdownOnFailure` / `ShutdownOnSuccess` policies; cancellation and error propagation automatic.
- **`ScopedValue`** (JEP 446): also **preview in 21**. Immutable, bounded-lifetime, inheritable by structured child threads - intended to replace ThreadLocal for context passing. (Later JDKs moved it further along; check your target JDK's status before relying on it.)
- Virtual threads themselves and `Executors.newVirtualThreadPerTaskExecutor()` are final in 21.

```java
// preview API on 21: javac/java --enable-preview --release 21
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var u = scope.fork(() -> userSvc.get(id));
    var o = scope.fork(() -> orderSvc.list(id));
    scope.joinUntil(Instant.now().plusSeconds(1));   // overall timeout
    scope.throwIfFailed();                           // first failure cancels siblings
    return new Page(u.get(), o.get());
}
```

This is what section 8.7 wants to be: real cancellation of siblings and no leaked tasks.

---

# 10. Runnable demos (compiled and run on JDK 21.0.4)

All sources are plain single-file Java; compile with `javac X.java && java X`.

## 10.1 Core / queue / max / reject transitions - `PoolTrace.java`

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class PoolTrace {
    public static void main(String[] a) throws Exception {
        AtomicInteger n = new AtomicInteger();
        ThreadPoolExecutor p = new ThreadPoolExecutor(2, 4, 1, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(2),
            r -> new Thread(r, "w-" + n.incrementAndGet()),
            new ThreadPoolExecutor.AbortPolicy());
        CountDownLatch gate = new CountDownLatch(1);
        for (int i = 1; i <= 8; i++) {
            String outcome;
            try {
                p.execute(() -> { try { gate.await(); } catch (InterruptedException e) {} });
                outcome = "accepted";
            } catch (RejectedExecutionException e) { outcome = "REJECTED"; }
            System.out.printf("task %d %-9s poolSize=%d active=%d queue=%d%n",
                i, outcome, p.getPoolSize(), p.getActiveCount(), p.getQueue().size());
        }
        gate.countDown();
        p.shutdown();
        p.awaitTermination(5, TimeUnit.SECONDS);
        System.out.println("completed=" + p.getCompletedTaskCount() + " largest=" + p.getLargestPoolSize());
    }
}
```

Observed output:

```
task 1 accepted  poolSize=1 active=1 queue=0
task 2 accepted  poolSize=2 active=2 queue=0
task 3 accepted  poolSize=2 active=2 queue=1
task 4 accepted  poolSize=2 active=2 queue=2
task 5 accepted  poolSize=3 active=3 queue=2
task 6 accepted  poolSize=4 active=4 queue=2
task 7 REJECTED  poolSize=4 active=4 queue=2
task 8 REJECTED  poolSize=4 active=4 queue=2
completed=6 largest=4
```

Reading it: tasks 1-2 create core workers (step 1). 3-4 fill the queue (step 2). Tasks 5-6 hit the full queue and create non-core workers (step 3) - and those two tasks run **before** queued tasks 3 and 4 (they are `firstTask`). 7-8 are rejected (step 4). Only 6 tasks ever completed; the 2 rejected were lost unless the caller handles the exception.

## 10.2 submit vs execute exception handling - `SubmitVsExecute.java`

```java
import java.util.concurrent.*;

public class SubmitVsExecute {
    public static void main(String[] a) throws Exception {
        ThreadPoolExecutor p = new ThreadPoolExecutor(1, 1, 0, TimeUnit.SECONDS, new LinkedBlockingQueue<>(),
            new ThreadFactory() {
                int n = 0;
                public Thread newThread(Runnable r) {
                    Thread t = new Thread(r, "pool-worker-" + (++n));
                    t.setUncaughtExceptionHandler((th, e) -> System.out.println("  [UEH] " + th.getName() + " died: " + e));
                    return t;
                }
            });
        System.out.println("--- execute(): exception escapes to UncaughtExceptionHandler, worker replaced");
        p.execute(() -> { System.out.println("  running on " + Thread.currentThread().getName()); throw new IllegalStateException("boom-execute"); });
        Thread.sleep(300);
        p.execute(() -> System.out.println("  next task on " + Thread.currentThread().getName()));
        Thread.sleep(300);
        System.out.println("--- submit(): exception captured in Future, nothing printed");
        Future<?> f = p.submit(() -> { throw new IllegalStateException("boom-submit"); });
        Thread.sleep(300);
        System.out.println("  isDone=" + f.isDone() + " (no [UEH] line above = swallowed)");
        try { f.get(); } catch (ExecutionException e) { System.out.println("  get() -> ExecutionException cause=" + e.getCause()); }
        p.shutdown();
    }
}
```

Observed output:

```
--- execute(): exception escapes to UncaughtExceptionHandler, worker replaced
  running on pool-worker-1
  [UEH] pool-worker-1 died: java.lang.IllegalStateException: boom-execute
  next task on pool-worker-2
--- submit(): exception captured in Future, nothing printed
  isDone=true (no [UEH] line above = swallowed)
  get() -> ExecutionException cause=java.lang.IllegalStateException: boom-submit
```

`pool-worker-1` died and `pool-worker-2` (a new thread) ran the next task: `processWorkerExit(completedAbruptly=true)` -> `addWorker(null,false)`. The submit case: same worker survived, nothing logged.

## 10.3 CompletableFuture thread names - `CfThreads.java`

```java
import java.util.concurrent.*;

public class CfThreads {
    static void log(String s) { System.out.println(String.format("%-30s on %s", s, Thread.currentThread().getName())); }
    static void sleep(long ms) { try { Thread.sleep(ms); } catch (InterruptedException e) {} }

    public static void main(String[] a) throws Exception {
        ExecutorService io = Executors.newFixedThreadPool(2, r -> new Thread(r, "io-thread"));
        System.out.println("cores=" + Runtime.getRuntime().availableProcessors()
            + " commonPoolParallelism=" + ForkJoinPool.commonPool().getParallelism());
        System.out.println("== A: callback registered while source still running -> runs on completing thread");
        CompletableFuture<String> slow = CompletableFuture.supplyAsync(() -> { sleep(200); log("supplyAsync(slow, io)"); return "x"; }, io);
        slow.thenApply(v -> { log("thenApply (registered early)"); return v; }).join();
        System.out.println("== B: source already complete -> callback runs on the CALLER");
        CompletableFuture<String> done = CompletableFuture.completedFuture("y");
        done.thenApply(v -> { log("thenApply (already done)"); return v; }).join();
        System.out.println("== C: thenApplyAsync, no executor -> common pool");
        done.thenApplyAsync(v -> { log("thenApplyAsync (default)"); return v; }).join();
        System.out.println("== D: thenApplyAsync with executor");
        done.thenApplyAsync(v -> { log("thenApplyAsync(io)"); return v; }, io).join();
        System.out.println("== E: supplyAsync, no executor");
        CompletableFuture.supplyAsync(() -> { log("supplyAsync (default)"); return 1; }).join();
        System.out.println("== F: exception propagation");
        CompletableFuture<Integer> bad = CompletableFuture.supplyAsync(() -> { if (true) throw new IllegalArgumentException("bad"); return 1; });
        bad.exceptionally(t -> { System.out.println("  exceptionally on the SOURCE stage sees: " + t.getClass().getSimpleName() + " cause=" + t.getCause()); return -1; }).join();
        CompletableFuture<Integer> bad2 = CompletableFuture.supplyAsync(() -> { if (true) throw new IllegalArgumentException("bad"); return 1; });
        bad2.thenApply(x -> x + 1).exceptionally(t -> { System.out.println("  exceptionally DOWNSTREAM sees: " + t.getClass().getSimpleName() + " cause=" + t.getCause()); return -1; }).join();
        try { bad2.join(); } catch (CompletionException e) { System.out.println("  join() throws CompletionException cause=" + e.getCause()); }
        try { bad2.get(); } catch (ExecutionException e) { System.out.println("  get() throws ExecutionException cause=" + e.getCause()); }
        System.out.println("== G: timeouts (Java 9+)");
        System.out.println("  completeOnTimeout -> " + new CompletableFuture<String>().completeOnTimeout("fallback", 100, TimeUnit.MILLISECONDS).join());
        try { new CompletableFuture<String>().orTimeout(100, TimeUnit.MILLISECONDS).join(); }
        catch (CompletionException e) { System.out.println("  orTimeout -> " + e.getCause().getClass().getSimpleName()); }
        io.shutdown();
    }
}
```

Observed output:

```
cores=12 commonPoolParallelism=11
== A: callback registered while source still running -> runs on completing thread
supplyAsync(slow, io)          on io-thread
thenApply (registered early)   on io-thread
== B: source already complete -> callback runs on the CALLER
thenApply (already done)       on main
== C: thenApplyAsync, no executor -> common pool
thenApplyAsync (default)       on ForkJoinPool.commonPool-worker-1
== D: thenApplyAsync with executor
thenApplyAsync(io)             on io-thread
== E: supplyAsync, no executor
supplyAsync (default)          on ForkJoinPool.commonPool-worker-1
== F: exception propagation
  exceptionally on the SOURCE stage sees: CompletionException cause=java.lang.IllegalArgumentException: bad
  exceptionally DOWNSTREAM sees: CompletionException cause=java.lang.IllegalArgumentException: bad
  join() throws CompletionException cause=java.lang.IllegalArgumentException: bad
  get() throws ExecutionException cause=java.lang.IllegalArgumentException: bad
== G: timeouts (Java 9+)
  completeOnTimeout -> fallback
  orTimeout -> TimeoutException
```

Note that in F the source-stage handler *also* got a `CompletionException` (the async runner wraps what the supplier threw), which is why unwrapping is always needed.

## 10.4 Starvation deadlock in a bounded pool - `Starve.java`

```java
import java.util.concurrent.*;

public class Starve {
    public static void main(String[] a) throws Exception {
        ExecutorService pool = Executors.newFixedThreadPool(2);
        Callable<String> parent = () -> {
            Future<String> child = pool.submit(() -> "child-result"); // queued behind the parents
            return "parent got " + child.get();                       // blocks a worker forever
        };
        Future<String> p1 = pool.submit(parent);
        Future<String> p2 = pool.submit(parent);
        try { System.out.println(p1.get(2, TimeUnit.SECONDS)); }
        catch (TimeoutException e) { System.out.println("TIMEOUT after 2s: starvation deadlock"); }
        for (Thread t : Thread.getAllStackTraces().keySet()) {
            if (t.getName().startsWith("pool-1-thread")) {
                StackTraceElement[] st = t.getStackTrace();
                System.out.println(t.getName() + " state=" + t.getState() + " frame[0..1]=" + st[0] + " | " + st[1]);
            }
        }
        System.out.println("queued children waiting: " + ((ThreadPoolExecutor) pool).getQueue().size());
        ((ThreadPoolExecutor) pool).shutdownNow();
    }
}
```

Observed output:

```
TIMEOUT after 2s: starvation deadlock
pool-1-thread-1 state=WAITING frame[0..1]=java.base/jdk.internal.misc.Unsafe.park(Native Method) | java.base/java.util.concurrent.locks.LockSupport.park(LockSupport.java:221)
pool-1-thread-2 state=WAITING frame[0..1]=java.base/jdk.internal.misc.Unsafe.park(Native Method) | java.base/java.util.concurrent.locks.LockSupport.park(LockSupport.java:221)
queued children waiting: 2
```

Both workers are parked in `FutureTask.get()`, the two children sit in the queue. No `synchronized` cycle, so `jstack` will **not** report "Found one Java-level deadlock" - it is invisible to deadlock detectors; you diagnose it from `WAITING` in `FutureTask.awaitDone` and a non-empty queue with zero progress.

Fixes: separate pool for child tasks; `CompletableFuture.thenCompose` instead of blocking; make the parent and child the same task (run child inline); or `SynchronousQueue`/cached pool (which grows). Never make a task wait on work submitted to its own bounded pool.

## 10.5 RecursiveTask (fork/join) - `FJ.java`

```java
import java.util.concurrent.*;

public class FJ {
    static class Sum extends RecursiveTask<Long> {
        final long[] a; final int lo, hi;
        Sum(long[] a, int lo, int hi) { this.a = a; this.lo = lo; this.hi = hi; }
        protected Long compute() {
            if (hi - lo <= 100_000) { long s = 0; for (int i = lo; i < hi; i++) s += a[i]; return s; }
            int mid = (lo + hi) >>> 1;
            Sum left = new Sum(a, lo, mid), right = new Sum(a, mid, hi);
            left.fork();                 // push onto my own deque
            long r = right.compute();    // do one half myself
            return r + left.join();      // then join the other
        }
    }
    public static void main(String[] x) {
        long[] a = new long[10_000_000];
        for (int i = 0; i < a.length; i++) a[i] = i;
        ForkJoinPool pool = new ForkJoinPool(4);
        long sum = pool.invoke(new Sum(a, 0, a.length));
        System.out.println("sum=" + sum + " expected=" + (9_999_999L * 10_000_000L / 2));
        System.out.println("stealCount>0 = " + (pool.getStealCount() > 0) + " parallelism=" + pool.getParallelism());
        pool.shutdown();
    }
}
```

Observed output:

```
sum=49999995000000 expected=49999995000000
stealCount>0 = true parallelism=4
```

`stealCount > 0` shows that idle workers stole sub-tasks from others' deques.

## 10.6 Virtual threads, 10 000 tasks - `VT.java`

```java
import java.time.Duration;
import java.util.Set;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class VT {
    public static void main(String[] a) throws Exception {
        Set<String> carriers = ConcurrentHashMap.newKeySet();
        AtomicInteger done = new AtomicInteger();
        long t = System.nanoTime();
        try (var ex = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 10_000; i++) {
                ex.submit(() -> {
                    Thread.sleep(Duration.ofSeconds(1));  // parks the virtual thread, frees the carrier
                    String s = Thread.currentThread().toString(); // VirtualThread[#22]/runnable@ForkJoinPool-1-worker-3
                    int at = s.indexOf('@');
                    if (at > 0) carriers.add(s.substring(at + 1));
                    done.incrementAndGet();
                    return null;
                });
            }
        } // close() waits for all tasks
        System.out.printf("virtual: 10000 tasks x 1s sleep finished in %d ms, done=%d, distinct carriers=%d%n",
            (System.nanoTime() - t) / 1_000_000, done.get(), carriers.size());
        System.out.println("cores=" + Runtime.getRuntime().availableProcessors());
        t = System.nanoTime();
        try (var ex = Executors.newFixedThreadPool(200)) {
            for (int i = 0; i < 10_000; i++) ex.submit(() -> { Thread.sleep(1000); return null; });
        }
        System.out.printf("platform pool(200): same work took %d ms (expect ~50 waves x 1s)%n", (System.nanoTime() - t) / 1_000_000);
        Thread v = Thread.ofVirtual().name("v1").unstarted(() -> {});
        System.out.println("isVirtual=" + v.isVirtual() + " isDaemon=" + v.isDaemon());
    }
}
```

Observed output (second run; the first run of the virtual part took 1527 ms):

```
virtual: 10000 tasks x 1s sleep finished in 2440 ms, done=10000, distinct carriers=12
cores=12
platform pool(200): same work took 50467 ms (expect ~50 waves x 1s)
isVirtual=true isDaemon=true
```

10 000 concurrent one-second sleeps complete in roughly 1.5-2.5 s on 12 carrier threads; the 200-thread pool needs 50 waves = ~50 s. Also shows `try-with-resources` on `ExecutorService` (Java 19+ `close()` = shutdown + await). The variation (1.5 s vs 2.4 s) is JIT warm-up and startup cost of creating 10 000 tasks, not a property of the scheduler.

---

# 11. Production war stories

**1. Unbounded queue OOM (fixed pool)**
- Symptom: latency climbs for 20 minutes, then `OutOfMemoryError: Java heap space`, pod restarts, cycle repeats.
- Evidence: heap dump - dominator is `LinkedBlockingQueue$Node` -> `FutureTask` -> request payloads (millions). Thread dump shows 8 workers all in the same downstream call. Metrics: `executor.queued` monotonic.
- Fix: bounded `ArrayBlockingQueue(size from latency budget)`, `CallerRunsPolicy` or 503 on rejection, timeouts on the downstream, alert on queue depth and `executor.idle`. Lesson: an unbounded queue converts a *throughput* problem into a *memory* problem and hides it until it is fatal.

**2. All threads blocked on a slow downstream**
- Symptom: whole service returns 504 although CPU is 5%; health endpoint (served by the same Tomcat pool) times out and K8s kills pods -> cascading restarts.
- Evidence: 200 of 200 `http-nio-exec-*` threads `TIMED_WAITING/RUNNABLE` in `SocketInputStream.socketRead` to the same host; downstream latency graph up.
- Fix: connect/read timeouts (few seconds max), bulkhead the dependency (own pool or semaphore), circuit breaker, separate management port/pool for health endpoints, degrade via fallback. Little's law: latency x rate exceeded thread count.

**3. Tasks rejected silently**
- Symptom: "some emails never sent", no errors.
- Evidence: `DiscardPolicy` (someone copied a snippet) or `submit()` result ignored so the `RejectedExecutionException`/exception went unnoticed; rejected count metric was not exported.
- Fix: custom handler that logs + counts + persists; alert on rejection > 0; treat lossy work explicitly.

**4. Pool per request leak**
- Symptom: threads grow to thousands; `unable to create native thread`; RSS grows.
- Evidence: thread dump with thousands of `pool-1234-thread-1`; the code has `Executors.newFixedThreadPool(4)` inside a controller method with no `shutdown()`. Each unreferenced pool's core threads never die (non-daemon, blocked on `take()`), so the pool is never garbage collected.
- Fix: create pools once (bean/static), inject; if per-request parallelism is needed use a shared pool or `try (var ex = Executors.newVirtualThreadPerTaskExecutor())`.

**5. ThreadLocal leak / context bleed in pools**
- Symptom: user A occasionally sees user B's data; or Metaspace/heap grows after redeploys (classloader leak).
- Evidence: `ThreadLocal` set in a filter and never removed; pool threads reused; heap dump shows `Thread.threadLocals` -> app classloader.
- Fix: always `try { set } finally { remove }`; use `TaskDecorator` that restores; in app servers never leave ThreadLocals holding classes from the webapp classloader; clear in `afterExecute`.

**6. Starvation deadlock from nested submit**
- Symptom: after a traffic spike, batch endpoint hangs forever, CPU idle, no deadlock reported by `jstack`.
- Evidence: all N pool threads `WAITING` at `FutureTask.awaitDone`/`CompletableFuture.join`; queue non-empty; `getCompletedTaskCount` frozen (demo 10.4).
- Fix: parent and child on **different** pools; chain with `thenCompose`; or eliminate nested submit. Add `orTimeout`/`get(timeout)` so it degrades rather than hangs.

**7. Common-pool starvation from blocking parallel stream**
- Symptom: unrelated endpoints slow whenever the report endpoint runs.
- Evidence: all `ForkJoinPool.commonPool-worker-*` in JDBC/HTTP reads.
- Fix: never do IO in `parallelStream`/default `supplyAsync`; own pool or virtual threads; cap by `-D...common.parallelism` only as a stopgap.

**8. Scheduled job silently stopped**
- Symptom: nightly cache refresh stops after one NPE; no logs.
- Evidence: `scheduleAtFixedRate` raw; future completed exceptionally, never inspected.
- Fix: try/catch Throwable inside the task; monitor "last successful run" timestamp.

**9. `@Async` exhausting memory after "we set max-size to 50"**
- Symptom: max-size ignored; queue grows; OOM.
- Evidence: `queue-capacity` unset = `Integer.MAX_VALUE`; only 8 core threads ever exist.
- Fix: set `queue-capacity` to a bounded number; then max applies; add rejection handler.

**10. Virtual threads freeze under JDBC**
- Symptom: after enabling `spring.threads.virtual.enabled`, throughput plateaus, some requests hang.
- Evidence: `-Djdk.tracePinnedThreads=full` or JFR `jdk.VirtualThreadPinned` show a driver's `synchronized` block doing IO; Hikari pool of 10 saturated so 10k virtual threads queue on `getConnection`.
- Fix: upgrade driver/JDK; a semaphore + realistic connection-acquire timeout; size DB pool deliberately.

### Reading a thread dump for an exhausted pool
1. `jcmd <pid> Thread.print` (or `jstack -l`), take 3 dumps 5-10 s apart.
2. Group by thread-name prefix (this is why naming matters). Count states per pool.
3. All workers in the **same top application frame** = the common blocker (downstream, lock, DB).
4. `WAITING` at `LockSupport.park` under `FutureTask.awaitDone` or `CompletableFuture$Signaller` = tasks waiting on other tasks (starvation).
5. `BLOCKED` on a monitor: find the owner thread in the same dump (`locked <0x...>`).
6. Workers idle (`WAITING` at `LinkedBlockingQueue.take`) but requests fail = the problem is upstream of the pool (rejection, wrong pool, deadlocked producers).
7. Cross-check with metrics: queue depth, active count, rejected count.

---

# 12. Interview questions (45)

Format: **Q (level)** - model answer; **Follow-up** chain; **Wrong answer** where common.

### Fundamentals (Easy)

**Q1 (Easy). Why use a thread pool instead of `new Thread()` per task?**
Reuse avoids creation cost (stack allocation, kernel thread), bounds concurrency to protect the CPU and downstreams, provides a queue and lifecycle (shutdown), and gives one place for naming, metrics, and rejection. Unbounded thread-per-task collapses under load.
Follow-up: *What does a pool NOT do?* It does not make tasks faster; it limits and reuses.
Follow-up: *Is that still true with virtual threads?* No - you do not pool virtual threads.
Wrong answer: "Pools make code run faster / in parallel." Parallelism comes from cores, not pools.

**Q2 (Easy). Name the ThreadPoolExecutor constructor parameters.**
`corePoolSize, maximumPoolSize, keepAliveTime + unit, workQueue, threadFactory, handler`. Core = kept threads, max = ceiling, keepAlive = idle time for threads above core, queue holds waiting tasks, factory names threads, handler decides on rejection.
Follow-up: *Which are commonly forgotten?* threadFactory (unnamed threads) and handler (default Abort).

**Q3 (Easy). In what order does a pool use core threads, the queue, and max threads?**
Core threads first, then queue, then non-core up to max, then reject.
Wrong answer: "It grows to max first, then queues." Opposite.
Follow-up: *Then why does my max never get used?* Unbounded queue.

**Q4 (Easy). What happens when the pool and queue are full?**
`RejectedExecutionHandler` is invoked; default `AbortPolicy` throws `RejectedExecutionException` to the submitter.
Follow-up: *Which policy gives backpressure?* CallerRuns.

**Q5 (Easy). `execute` vs `submit`?**
`execute(Runnable)` no result, exceptions kill the worker (UEH). `submit` returns a `Future`, wraps the task in `FutureTask`, exceptions captured and rethrown only on `get()` as `ExecutionException`.
Follow-up: *Which one would you use for fire-and-forget and why?* `execute`, or `submit` with try/catch inside - otherwise failures are silent.

**Q6 (Easy). Why is `Executors.newFixedThreadPool` discouraged?**
It uses an unbounded `LinkedBlockingQueue`: no rejection, unbounded memory and latency. `newCachedThreadPool` allows `Integer.MAX_VALUE` threads. Build TPE directly.
Follow-up: *Is `newSingleThreadExecutor` safe?* Same unbounded queue; useful only for ordering.

**Q7 (Easy). `shutdown()` vs `shutdownNow()`?**
`shutdown` stops intake, runs queued tasks. `shutdownNow` drains the queue (returns it) and interrupts workers. Neither blocks; use `awaitTermination`.
Follow-up: *Does `shutdownNow` stop a task in an infinite loop?* No, interruption is cooperative.

**Q8 (Easy). What are daemon threads and why does it matter for pools?**
Daemon threads do not keep the JVM alive. A pool of non-daemon workers that is never shut down prevents JVM exit. Common-pool and virtual threads are daemon: tasks may be lost at exit.

**Q9 (Easy). Runnable vs Callable vs Future?**
`Runnable.run()` no result, no checked exceptions. `Callable.call()` returns a value and may throw. `Future` is the handle to a pending result (`get`, `cancel`, `isDone`).

### Core internals (Medium)

**Q10 (Medium). What is `ctl`?**
A single `AtomicInteger` packing `runState` (top 3 bits) and `workerCount` (low 29 bits). Lets the pool read and atomically update state+count together without a lock. States: RUNNING(-1<<29), SHUTDOWN(0), STOP(1<<29), TIDYING(2<<29), TERMINATED(3<<29).
Follow-up: *Max workers?* 2^29-1.
Follow-up: *Why order states numerically?* So checks like `c < SHUTDOWN` (running) and `runStateAtLeast` are one comparison.

**Q11 (Medium). Walk through `execute()` exactly.**
(1) workerCount < core -> `addWorker(cmd, true)`. (2) else if running and `queue.offer(cmd)` -> re-check ctl: if not running and `remove(cmd)` -> reject; else if workerCount == 0 -> `addWorker(null,false)`. (3) else `addWorker(cmd,false)`; if false -> reject.
Follow-up: *Why the re-check after offer?* Pool may have been shut down or lost all workers between the check and the enqueue: a stranded task otherwise.
Follow-up: *When does `addWorker(null,false)` matter most?* `corePoolSize=0` pools, and after the last worker died.

**Q12 (Medium). Can a later task run before an earlier queued one?**
Yes. Once the queue is full a new non-core worker takes the *new* task directly as `firstTask`, overtaking queued tasks. Also a priority queue reorders. Pool gives no FIFO guarantee except a single-thread executor with a FIFO queue.

**Q13 (Medium). Why does `Worker` extend AQS?**
To use a non-reentrant exclusive lock as a "busy" marker. Held during task execution; `interruptIdleWorkers` uses `tryLock` to interrupt only idle workers (blocked in `getTask`). Non-reentrant so a task calling `setCorePoolSize` cannot re-acquire its own lock and appear idle. State -1 blocks interrupts until `runWorker` starts.
Follow-up: *Why not `ReentrantLock`?* See reentrancy above; also lighter (no separate lock object).
Wrong answer: "For thread safety of the queue." The queue is thread-safe by itself.

**Q14 (Medium). How do idle non-core threads die?**
`getTask()` computes `timed = allowCoreThreadTimeOut || wc > core`. Timed workers use `poll(keepAlive)`; on timeout with count still above core they CAS-decrement and return null, so `runWorker` exits and `processWorkerExit` removes them. There is no permanent "core thread" identity: any worker may retire when count exceeds core.
Follow-up: *How to let core threads time out?* `allowCoreThreadTimeOut(true)` with keepAlive > 0.

**Q15 (Medium). What happens when a task throws in `execute()`?**
Exception propagates out of `runWorker` (after `afterExecute(task, ex)`); `completedAbruptly = true`; `processWorkerExit` decrements count and calls `addWorker(null,false)` for a replacement thread; the dying thread's `UncaughtExceptionHandler` runs (default: stack trace to stderr).
Follow-up: *And with `submit`?* Worker survives; exception in the Future.
Follow-up: *Cost?* Thread churn and lost ThreadLocals.

**Q16 (Medium). How do you observe exceptions from `submit()`ed tasks in `afterExecute`?**
`Throwable` argument is null; check `if (r instanceof Future<?> f) { try { f.get(); } catch (ExecutionException e) { t = e.getCause(); } catch (CancellationException ce) {...} }` - only when `f.isDone()`.

**Q17 (Medium). Consequences of each queue type.**
`SynchronousQueue`: direct hand-off; grows threads to max (cached pool). Unbounded `LinkedBlockingQueue`: max never used, OOM risk. `ArrayBlockingQueue`: bounded, one lock, predictable memory. `PriorityBlockingQueue`: unbounded, ordering; `submit` wraps in non-Comparable `FutureTask` -> `ClassCastException`.
Follow-up: *How to make a pool spawn threads before queueing?* Custom queue whose `offer` returns false while pool size < max.

**Q18 (Medium). Rejection policies and when you'd choose each.**
Abort (fail fast, propagate 503), CallerRuns (backpressure for producer threads that can afford to work), Discard (only optional work), DiscardOldest (latest-wins, never with priority queues), custom (log+metric+DLQ).
Follow-up: *Danger of CallerRuns?* Runs on the submitter: bad on event-loop threads, may hold locks, can mask overload by slowing everything.

**Q19 (Medium). `scheduleAtFixedRate` vs `scheduleWithFixedDelay`; what if the task throws?**
Fixed rate: start-to-start period; if task overruns, next runs immediately after (never concurrent), can bunch up. Fixed delay: end-to-start gap. If the task throws, **all subsequent runs are suppressed**, silently. Wrap in try/catch(Throwable).
Follow-up: *Why does the scheduler pool ignore max size?* Unbounded DelayedWorkQueue: step 3 never reached.

**Q20 (Medium). What are FutureTask states and what does `cancel(true)` do?**
NEW, COMPLETING, NORMAL, EXCEPTIONAL, CANCELLED, INTERRUPTING, INTERRUPTED. `cancel(true)` marks cancelled and interrupts the running thread; task must cooperate. `get` afterwards throws `CancellationException`. A `get(timeout)` timeout does not cancel.

**Q21 (Medium). Give the formulas to size a pool.**
CPU-bound: ~cores (+1). IO-bound: `cores x U x (1 + W/C)`. Then cap by the smallest downstream limit (connection pools), verify with Little's law `L = lambda x W`, and confirm under load.
Follow-up: *Example?* 300 req/s x 80 ms = 24 concurrent -> ~37 threads at 65% utilisation.
Wrong answer: "2 x cores always" or "as many threads as possible".

**Q22 (Medium). What does the default `@Async` executor do?**
Plain Spring: `SimpleAsyncTaskExecutor`, new thread per task (unbounded). Boot: `applicationTaskExecutor`, `ThreadPoolTaskExecutor` core 8, max/queue `Integer.MAX_VALUE` -> effectively 8 threads with an unbounded queue. Boot 3.2 with virtual threads enabled: virtual thread per task.
Follow-up: *How do you configure safely?* Named bean, bounded queue, rejection policy, TaskDecorator, graceful shutdown settings.

**Q23 (Medium). Why can `@Async` lose the security context / MDC / transaction?**
They are ThreadLocals bound to the caller. Copy them with a `TaskDecorator` (capture on caller, set + restore in a `finally` on worker) or `DelegatingSecurityContext*` wrappers. Transactions cannot be propagated to another thread; pass IDs, and use after-commit hooks.
Follow-up: *Why not inheritable ThreadLocal?* Copies only at thread creation; pool threads are reused, leaking stale identity.

**Q24 (Medium). `thenApply` vs `thenApplyAsync` - which thread runs?**
`thenApply`: the thread that completes the source, or the caller if already complete. `thenApplyAsync`: default async executor (common pool) or the given executor.
Follow-up: *Why care?* Heavy sync callback hijacks the completing (often IO) thread; nondeterministic ThreadLocal visibility.

**Q25 (Medium). `thenApply` vs `thenCompose` vs `thenCombine`.**
map vs flatMap vs join-two-independent. `thenApply(x -> asyncCall(x))` gives `CF<CF<U>>` - wrong.

**Q26 (Medium). `exceptionally` vs `handle` vs `whenComplete`.**
`exceptionally`: failure only, gives fallback. `handle`: both paths, returns new value. `whenComplete`: observe only; result/exception passes through unchanged. Failures surface wrapped in `CompletionException`.
Follow-up: *What does `join` throw vs `get`?* `CompletionException` (unchecked) vs `ExecutionException` (checked).

**Q27 (Medium). How do you time out a `CompletableFuture`?**
Java 9+: `orTimeout` (fail with `TimeoutException`), `completeOnTimeout` (fallback value). Only the future completes; underlying work continues - add client timeouts. In Java 8 you need a `ScheduledExecutorService`.

**Q28 (Medium). What is the ForkJoin common pool and its default size?**
JVM-wide shared pool used by parallel streams and default `CompletableFuture` async; parallelism = `availableProcessors() - 1`; daemon threads; container CPU limits affect it. If parallelism <= 1, CF falls back to a new thread per task.

**Q29 (Medium). Why is blocking inside `parallelStream` dangerous?**
Only ~cores-1 workers shared by the entire JVM; blocking tasks occupy them and starve all other parallel streams and default async stages, including library code. Use a dedicated executor.

### Advanced (Hard)

**Q30 (Hard). Explain work stealing.**
Per-worker deques; owner pushes/pops the top (LIFO), thieves take from the base (FIFO = oldest, biggest tasks). Reduces contention versus a single queue; `join` helps by running/stealing tasks instead of blocking. Best for many small recursive CPU tasks.
Follow-up: *Why take from the opposite end?* Minimises contention between owner and thief and steals large chunks.
Follow-up: *Why fork one child and compute the other directly?* Avoid idle current thread and extra task overhead.

**Q31 (Hard). What is `ManagedBlocker`?**
A protocol (`ForkJoinPool.managedBlock`) letting a worker declare a blocking operation so the pool can add a spare thread and keep parallelism up. Used by `CompletableFuture.join/get`, `Phaser`, some collections; not a licence to block in the common pool at scale.

**Q32 (Hard). Explain a starvation deadlock.**
A bounded pool's workers all block waiting on tasks that are queued in the same pool. No lock cycle so `jstack` reports no deadlock; threads are `WAITING` in `FutureTask.get`/`CompletableFuture.join`, queue non-empty, no progress. Fixes: separate pools, non-blocking composition, timeouts, or unbounded threads.
Follow-up: *Would it happen with CachedThreadPool?* Not typically (SynchronousQueue spawns threads) - but risk becomes unbounded threads.
Follow-up: *With virtual threads?* Not from pool exhaustion, because there is no small pool; but you can still deadlock on locks or on a semaphore.

**Q33 (Hard). Why can `join()` inside a CompletableFuture callback deadlock?**
A callback running on a pool thread blocks waiting for another stage that needs a pool thread. Under load all threads are blocked waiters: starvation. Chain with `thenCompose` instead.

**Q34 (Hard). How does cancellation work in CompletableFuture?**
`cancel` completes with `CancellationException` but does not interrupt the running task, and is not propagated to upstream sources; dependents get a `CompletionException` wrapping it. True cancellation needs a `Future`/interrupt link or structured concurrency.

**Q35 (Hard). Give a non-blocking fan-out/fan-in with timeouts and partial failure.**
`supplyAsync(..., ioPool).orTimeout(...).exceptionally(fallback)` per branch, combine with `thenCombine` / `allOf`, overall `orTimeout`, `join` only at the outermost. (Code in 8.7.) `allOf` completes only after all branches finish, so per-branch timeouts are essential; `anyOf` returns the first outcome even if it is a failure.

**Q36 (Hard). Describe how virtual threads work.**
Continuations whose stack frames are heap objects; run mounted on carrier threads (a ForkJoinPool, parallelism = cores). On blocking IO/park, frames are copied to heap (unmount), carrier freed; on readiness, rescheduled onto any carrier. Scheduling is cooperative (yield only at blocking points): no time slicing.
Follow-up: *Who wakes it on socket readiness?* JDK poller threads using epoll/kqueue.

**Q37 (Hard). What is pinning and how do you diagnose it?**
A virtual thread that blocks while holding a monitor (`synchronized`) or with a native frame on its stack cannot unmount and holds its carrier. Diagnose with `-Djdk.tracePinnedThreads=full` (JDK 21) or JFR `jdk.VirtualThreadPinned`. Fix: `ReentrantLock`, shorter synchronized sections, upgrade libs. JDK 24 (JEP 491) largely removes the `synchronized` case; native frames still pin.
Wrong answer: "Pinning only matters for CPU-bound code." It matters for blocking under a monitor.

**Q38 (Hard). Why should you not pool virtual threads, and how do you limit concurrency instead?**
They are cheap and not scarce; pooling adds queueing and ThreadLocal accumulation with no benefit. Use a virtual thread per task and limit the scarce *resource* with a `Semaphore` (or the connection pool). Blocked virtual threads on a semaphore cost almost nothing.

**Q39 (Hard). When would you choose platform threads / reactive over virtual threads?**
CPU-bound work (bounded pool sized to cores), code with heavy pinned `synchronized` blocking on JDK 21, need for built-in backpressure and stream operators (reactive), extremely high fan-out streaming. Virtual threads win for blocking request/response IO with legacy code.
Follow-up: *Does virtual threads remove need for bulkheads?* No - it makes them more necessary (no thread cap protecting your downstream).

**Q40 (Hard). Is StructuredTaskScope/ScopedValue usable in Java 21?**
Preview in 21 (JEP 453 and JEP 446): needs `--enable-preview`, API may change. Virtual threads themselves are final. Do not ship preview APIs in libraries.

**Q41 (Hard). Derive pool size for 500 req/s, each request makes 2 sequential 50 ms DB calls and 5 ms CPU. 16 cores. DB pool = 40.**
Per request time in system W ~ 105 ms -> L = 500 x 0.105 = 52.5 concurrent requests; but DB concurrency = 500 x 0.1 = 50 connections needed **> 40** - DB pool is the bottleneck (max throughput = 40/0.1 = 400 req/s). Threads beyond 40 only wait for connections; tune DB pool/queries first. CPU: 500 x 0.005 = 2.5 cores busy - not the constraint. Answer: the smallest limit sets throughput; scaling threads alone is pointless.

**Q42 (Hard). How do you gracefully shut down a Spring Boot app with async work on Kubernetes?**
Readiness off + preStop sleep for LB drain; `server.shutdown=graceful` with a phase timeout; pools with `waitForTasksToCompleteOnShutdown` + await seconds; `terminationGracePeriodSeconds` larger than the sum; durable queues for work that must not be lost; interrupt-friendly tasks.

**Q43 (Hard). A service shows CPU 5%, 504s, all workers busy. Walk through diagnosis.**
Thread dump x3, group by pool name; see identical top frames (socket read/JDBC); check downstream latency and connection pool waiters; look at `executor.queued/idle`; confirm via Little's law that latency x rate > threads; mitigate: timeouts, circuit breaker, bulkhead, shed load, scale downstream; long-term: separate pools, health endpoint isolation.

**Q44 (Hard). Why can a TPE with `corePoolSize=0` and a `LinkedBlockingQueue` still run tasks?**
`execute`: count (0) not < core (0); queue offer succeeds; recheck sees workerCount 0 -> `addWorker(null,false)` starts one worker (limited by max) which polls the queue. Without that safety net the task would be stranded. It runs with one thread (queue never rejects, so max never used).

**Q45 (Hard). `parallelStream().forEach(list::add)` - what's wrong and how to fix?**
Non-thread-safe accumulation from multiple common-pool threads -> lost updates/`ArrayIndexOutOfBounds`; also unordered. Use `collect(toList())`; for order-sensitive side effects `forEachOrdered`; avoid shared mutable state; also consider whether the source splits well and whether the work is CPU-bound enough.

### Common wrong answers (quick list)
| Wrong | Correct |
|---|---|
| "Pool grows to max before queueing" | Queue first; max only when queue full |
| "`submit` throws my task's exception" | It is stored; thrown at `get()` |
| "`shutdownNow` kills threads" | Interrupts; cooperative |
| "`orTimeout` cancels the work" | It only completes the future |
| "`cancel(true)` interrupts a CompletableFuture task" | It does not |
| "CompletableFuture is always async" | Non-async callbacks run on caller/completing thread |
| "Virtual threads are faster" | Higher concurrency for blocking IO, not faster CPU |
| "Pool virtual threads to limit them" | Semaphore the resource |
| "Starvation deadlock will be shown by jstack" | It will not; no lock cycle |
| "`parallelStream` is free speed" | Shared pool, overhead, needs big CPU-bound N |
| "Core threads are special threads that never die" | Any worker may retire if count > core; core is just a count |

---

# 13. One-page cheat sheet

```
DECISION ORDER:  workers<core -> new worker | queue.offer ok -> enqueue | workers<max -> new worker | reject
ctl = runState(3 bits) | workerCount(29 bits)   RUNNING(-1<<29) SHUTDOWN(0) STOP(1<<29) TIDYING(2<<29) TERMINATED(3<<29)
execute recheck after offer: !running -> remove+reject ; workers==0 -> addWorker(null,false)
Worker = AQS non-reentrant lock held while running a task -> tryLock() == "idle"
getTask: timed = allowCoreThreadTimeOut || wc>core ; timed ? poll(keepAlive) : take()
task throws via execute -> worker dies, UEH, replaced ; via submit -> stored in FutureTask, silent
shutdown = drain queue ; shutdownNow = STOP + interrupt + return queue ; then awaitTermination ; close() (19+) = both
```

| Queue | Effect |
|---|---|
| SynchronousQueue | hand-off; threads up to max (cached pool) |
| LinkedBlockingQueue() | unbounded -> max unused, OOM |
| ArrayBlockingQueue(n) | bounded, predictable |
| PriorityBlockingQueue | unbounded, needs Comparable (not with submit) |

| Rejection | Use |
|---|---|
| Abort | fail fast |
| CallerRuns | backpressure |
| Discard / DiscardOldest | lossy, rare |

**Sizing:** `L = lambda x W`. CPU-bound: cores. IO-bound: `cores x U x (1 + W/C)`. Capped by the smallest downstream limit. Queue = latency budget x drain rate. One pool per dependency (bulkhead).

**Monitor:** `poolSize, activeCount, queue.size, remainingCapacity, completedTaskCount, largestPoolSize`; Micrometer `ExecutorServiceMetrics` (`executor.queued`, `executor.active`, `executor.idle`). Alert on queue growth, rejections, queue wait.

**Spring:** plain Spring `@Async` default = SimpleAsyncTaskExecutor (thread per task). Boot = `applicationTaskExecutor` core 8, unbounded queue. Set `queue-capacity`. Propagate MDC/security via `TaskDecorator` (restore in finally). Return `CompletableFuture` or set `AsyncUncaughtExceptionHandler`. Graceful: await termination + K8s grace period.

**ForkJoin:** per-worker deques, owner LIFO, thief FIFO; common pool parallelism = cores-1 (daemon); never block in it; fork one, compute the other; `managedBlock` for necessary blocking.

**CompletableFuture:** default pool = common pool; non-async callback runs on completing/caller thread; `*Async(fn, ex)` for control. Failures wrapped in `CompletionException`. `exceptionally` (fallback), `handle` (both), `whenComplete` (observe). `orTimeout`/`completeOnTimeout` (9+) do not stop the work. Never `join` inside pool tasks; `thenCompose` instead. `cancel` does not interrupt or propagate. `allOf` waits for all; `anyOf` first outcome.

**Virtual threads (21):** thread per task, never pool, `Semaphore` to limit; mounted on ForkJoin carriers, unmount on blocking; pinned by `synchronized`+blocking and native frames (21-23; JEP 491 in 24 relaxes synchronized); detect with `-Djdk.tracePinnedThreads=full` / JFR; CPU-bound stays on platform pool; `spring.threads.virtual.enabled=true` (Boot 3.2+); StructuredTaskScope and ScopedValue are preview in 21.

**Incident quick map:** heap growing + queue growing = unbounded queue; all workers same frame = slow dependency; workers `WAITING` in `FutureTask.get` + queue > 0 = starvation; unrelated slowness = common pool blocked; job stopped = exception in periodic task; silent loss = Discard policy or ignored Future; thread count rising = pool per request.
