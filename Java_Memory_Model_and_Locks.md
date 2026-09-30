# Java Memory Model, `volatile`, `synchronized`, Locks, CAS

## Why threads see stale data
Each CPU core has its **own cache**. Thread A on core 1 changes `flag = true` in its cache; Thread B on core 2 may keep reading old `false`.

```
   Thread A (core 1)            Thread B (core 2)
   [cache: flag=true]           [cache: flag=false]   ← stale!
            \                      /
             ───── Main Memory ────
```

Three concurrency problems:
| Problem | Meaning | Solved by |
|---|---|---|
| **Visibility** | one thread's write not seen by others | `volatile`, `synchronized` |
| **Atomicity** | `count++` = read + add + write (3 steps) can interleave | `synchronized`, `AtomicInteger`, locks |
| **Ordering** | CPU/compiler reorder instructions | `volatile`, `synchronized`, happens-before |

## `volatile`
- Every read comes from main memory, every write goes to main memory → **visibility**.
- Prevents instruction reordering around it.
- **NOT atomic**: `volatile int c; c++` is still unsafe.

```java
class Worker {
    private volatile boolean running = true;     // stop flag – perfect use of volatile
    void run()  { while (running) { doWork(); } }
    void stop() { running = false; }             // other thread sees it immediately
}
```
Without `volatile`, `run()` may loop forever.

**Double-checked locking Singleton needs `volatile`:**
```java
class Singleton {
    private static volatile Singleton instance;      // volatile is REQUIRED
    static Singleton get() {
        if (instance == null) {                      // 1st check (no lock, fast)
            synchronized (Singleton.class) {
                if (instance == null)                // 2nd check
                    instance = new Singleton();      // = allocate, construct, assign (can be reordered!)
            }
        }
        return instance;
    }
}
```
Without volatile another thread could see a non-null but half-constructed object.

## Happens-before (simple version)
"If action A *happens-before* B, then A's results are guaranteed visible to B."
Key rules:
- Unlock of a monitor → later lock of the same monitor.
- Write to `volatile` → later read of that variable.
- `thread.start()` → everything inside that thread.
- Everything in thread → `thread.join()` return.

## `synchronized` vs `ReentrantLock`

| | `synchronized` | `ReentrantLock` |
|---|---|---|
| Release | automatic | **manual** in `finally` |
| Try without waiting | ✗ | `tryLock()`, `tryLock(2, SECONDS)` |
| Interruptible wait | ✗ | `lockInterruptibly()` |
| Fairness (FIFO) | ✗ | `new ReentrantLock(true)` |
| Multiple conditions | 1 (`wait/notify`) | many (`newCondition()`) |
| Speed | similar in modern JVM | similar |

```java
private final ReentrantLock lock = new ReentrantLock();
void transfer() {
    if (lock.tryLock()) {            // don't wait forever → avoids deadlock
        try { /* critical section */ }
        finally { lock.unlock(); }   // ALWAYS in finally
    }
}
```
**Reentrant** = same thread can lock again without blocking itself (holds a counter).

**`ReadWriteLock`:** many readers OR one writer. Good for read-heavy cache.
**`StampedLock`:** optimistic reads, even faster for read-heavy.

## What happens with `synchronized`? (flow)
```
Thread arrives at synchronized(obj)
   │
   ▼
Is obj's monitor free? ──Yes──► take it, run code ──► release on exit / exception
   │No
   ▼
BLOCKED state (waits in entry set) ─► woken when owner releases
```

## `AtomicInteger` & CAS (no locks)
CAS = Compare-And-Swap: "set value to 6 **only if** it is still 5; otherwise retry".
```
loop:
   old = get()
   new = old + 1
   if CAS(old → new) succeeded → done
   else → someone changed it, go to loop
```
Faster than locks under low contention. Weakness: **ABA problem** (value A→B→A looks unchanged) → `AtomicStampedReference`. `LongAdder` is better than `AtomicLong` for heavy counters.

## Other must-know items
- **`ThreadLocal`**: each thread has its own copy (user context, DateFormat). **Always `remove()` in thread pools** or you leak memory / leak data to the next request.
- **`ConcurrentHashMap` (Java 8):** array of buckets; empty bucket insert via **CAS**, non-empty via `synchronized` on the **first node of the bucket** (per-bucket lock). Reads are lock-free. Null key/value not allowed.
- **Deadlock 4 conditions:** mutual exclusion, hold-and-wait, no preemption, circular wait → break any one. Detect with `jstack <pid>` ("Found one Java-level deadlock").
- **`wait()` vs `sleep()`:** wait releases the lock and needs `notify`; sleep keeps lock.
- **Always call `wait()` in a `while` loop** (spurious wakeups).
- **`start()` vs `run()`:** `run()` executes in the current thread, no new thread.
