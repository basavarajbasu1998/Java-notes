# Java Memory Model, `volatile`, `synchronized`, Locks, CAS, ConcurrentHashMap Internals

> Target: JDK 17 / 21. Level: 5-year developer preparing for senior interviews.
> Scope note: thread pools, `CompletableFuture` and virtual threads are covered in other notes; collections in general are covered elsewhere. This note covers the memory model, locks, atomics, and `ConcurrentHashMap`'s *concurrency* internals.
> Where a JVM-internal detail varies by version I describe it conceptually and say so. All demo outputs in section 3 were actually produced on JDK 21.0.4 (Windows, 12 cores).

## Contents
1. 60-second mental model and analogy
2. Deep internals
   - 2.1 Why concurrency bugs exist (caches, store buffers, reordering, MESI)
   - 2.2 The Java Memory Model (JMM): data race, happens-before (all rules)
   - 2.3 Safe publication
   - 2.4 What `volatile` really guarantees + barriers
   - 2.5 Double-checked locking: the reordering trace
   - 2.6 `synchronized` internals (monitor, mark word, lock states, JIT optimisations)
   - 2.7 `wait` / `notify` and spurious wakeups
   - 2.8 Thread states and reading `jstack`
   - 2.9 AbstractQueuedSynchronizer (AQS)
   - 2.10 `ReentrantLock` flow, fair vs nonfair, `Condition`
   - 2.11 `ReentrantReadWriteLock`, `StampedLock`
   - 2.12 `Semaphore`, `CountDownLatch`, `CyclicBarrier`, `Phaser`, `LockSupport`
   - 2.13 Interruption
   - 2.14 CAS, `Unsafe`, `VarHandle`, atomics, ABA, `LongAdder`
   - 2.15 `ConcurrentHashMap` concurrency internals
   - 2.16 Concurrent queues overview
   - 2.17 `ThreadLocal` internals
   - 2.18 Deadlock, livelock, starvation
   - 2.19 False sharing, immutability, confinement, thread-safety levels
   - 2.20 Classic bug catalogue
3. Runnable demos (compiled and run on JDK 21)
4. Production war stories
5. Interview questions (50) with follow-ups and wrong answers
6. One-page cheat sheet

---

# 1. The 60-second mental model

Three independent things go wrong when threads share mutable state. Every tool in this note fixes one or more of them:

| Problem | What it means | Fixed by |
|---|---|---|
| **Atomicity** | A compound action (`count++` = read, add, write) is interleaved with another thread's actions | `synchronized`, `Lock`, atomics (CAS) |
| **Visibility** | A write by thread A is not seen (or seen late) by thread B | `volatile`, `synchronized`, `Lock`, `final` (with safe construction), `Thread.start/join` |
| **Ordering** | Compiler/CPU execute or make visible operations in an order different from source order | `volatile`, `synchronized`, `VarHandle` fences, happens-before |

**Analogy: a shared whiteboard in an office with private notebooks.**
- Every engineer (thread) keeps a *private notebook* (CPU registers, store buffer, L1/L2 cache) and copies numbers from the *whiteboard* (main memory) when convenient.
- Without rules, engineer B may keep reading his old notebook copy forever (**visibility**).
- Two engineers who both read "5", add 1, and write "6" lose one increment (**atomicity**).
- A clever assistant (compiler/CPU) may rearrange your to-do list to be faster; one engineer may see "door unlocked" written before "safe emptied" even though you did it in the other order (**ordering**).
- `synchronized` = a single key to the meeting room: only one person inside, and when you leave you *must* flush your notebook to the whiteboard; whoever enters *must* re-read the whiteboard.
- `volatile` = a variable that lives on the whiteboard only: each read looks at the board, each write goes on the board, and nothing may be reordered across it. But "read-then-write" is still two steps.
- CAS = "write 6 only if the board still says 5, otherwise tell me and I retry".

**The single sentence to remember:** *Correct multi-threaded code is code in which every pair of conflicting accesses to shared data is ordered by a happens-before edge.* Everything below is either how to create such an edge, or how JVM/CPU machinery implements it.

---

# 2. Deep internals

## 2.1 Why concurrency bugs exist

### The hardware picture

```
   Core 0                          Core 1
 +---------+                     +---------+
 | regs    |                     | regs    |
 | store   |                     | store   |
 | buffer  |                     | buffer  |
 | L1  L2  |                     | L1  L2  |
 +----+----+                     +----+----+
      |        shared L3 cache        |
      +--------------+----------------+
                     |
                Main memory (DRAM)
```

1. **Registers / compiler**: the JIT may keep a field in a register for the whole loop (hoisting the load). It is *allowed* to, because in the absence of synchronization the single-threaded meaning is unchanged. This is the classic `while (!stop)` infinite loop (demo 3.1).
2. **Store buffer**: a core's write does not go to cache immediately. It sits in a small per-core queue so the core does not stall. Other cores cannot see it yet. The core *can* read its own buffered value (store forwarding).
3. **Cache coherence (MESI at conceptual level)**: caches keep lines (typically 64 bytes) in one of four states.
   - **M**odified: only this cache has the line, and it is dirty.
   - **E**xclusive: only this cache has it, clean.
   - **S**hared: several caches may hold it, all clean.
   - **I**nvalid: not usable.
   To write a line a core must first make it M/E by sending an *invalidate* to the others. Invalidates are also queued (invalidate queue), so a remote core may read a stale value for a short while. Coherence gives you "one line converges to one value", **not** "all cores see all writes in program order".
4. **Out-of-order execution and reordering**: CPUs and compilers reorder independent instructions. The classic StoreLoad reordering, allowed even on x86 (the strongest common hardware model):

```
 initially x = 0, y = 0
 Thread 1          Thread 2
 x = 1;            y = 1;
 r1 = y;           r2 = x;
 Possible outcome: r1 == 0 && r2 == 0   (each store sat in the store buffer while the load ran)
```
   That is impossible under sequential consistency (SC) but legal in JMM without `volatile` (and even *with* `volatile` fields it is forbidden only because the JVM inserts a StoreLoad fence).

Architectures differ: x86-64 is "TSO" (only StoreLoad reordering visible). ARM/POWER (AArch64 servers, Apple silicon, Graviton) allow almost any reordering. **A bug that never shows on your x86 laptop can appear in production on ARM.** This is why DCL demo 3.3 below prints 0 anomalies yet the code is still wrong.

### Why the JMM exists

Java promises "write once, run anywhere". The JMM is the contract between *you*, the *JIT compiler*, and the *CPU*: it states exactly which optimisations are legal, so the JVM can be fast on every architecture while you get well-defined behaviour if you follow the rules. The JVM inserts the right fences for the platform; you reason only in terms of **happens-before**.

**Key theorem (JLS 17.4.5):** if a program is **data-race-free** (no two conflicting accesses unordered by happens-before), then all executions appear **sequentially consistent**: as if threads' actions were interleaved in one global order, respecting each thread's program order. Your job is to be data-race-free; then you can forget caches and reordering.

## 2.2 The JMM: data race and happens-before

**Conflicting accesses:** two accesses to the same variable (field, array element) where at least one is a write.
**Data race:** two conflicting accesses **not ordered by happens-before**. Note: a data race is about *ordering*, not about "at the same time". Even if one thread writes at 10:00 and another reads at 10:05, it is still a race if no hb edge connects them.

**Happens-before (hb):** a partial order. If `A hb B` then A's effects (all writes) are visible to B and A is ordered before B. *It does not mean A executes earlier in wall-clock time* in every sense (only that the result is as if so for B), and absence of hb does not mean "reordered", only "permitted to be".

### All happens-before rules (JLS 17.4.5 plus java.util.concurrent guarantees)

| # | Rule | Practical use |
|---|---|---|
| 1 | **Program order**: each action in a thread hb every later action *in that same thread*. | Single thread always sees its own writes in order. |
| 2 | **Monitor lock**: `unlock` of monitor M hb every *subsequent* `lock` of M. | `synchronized` visibility. |
| 3 | **Volatile variable**: write to volatile `v` hb every *subsequent* read of `v`. | Flags, safe publication. |
| 4 | **Thread start**: `t.start()` hb any action in `t`. | Arguments prepared before `start()` are visible in the thread. |
| 5 | **Thread termination**: all actions in `t` hb another thread's detection that `t` ended (`t.join()` returns, or `t.isAlive()` returns false, or `getState()==TERMINATED`). | Read results after `join`. |
| 6 | **Interruption**: `t.interrupt()` hb the interrupted thread detecting it (throws `InterruptedException` or `isInterrupted()/interrupted()` returns true). | Data prepared before `interrupt()` is visible after the check. |
| 7 | **Default values**: the write of default values (0/null/false) to every variable hb the first action of every thread. | No out-of-thin-air (mostly). |
| 8 | **Finalizer**: end of an object's constructor hb the start of its finalizer. | Rarely relevant. |
| 9 | **Transitivity**: `A hb B` and `B hb C` implies `A hb C`. | Lets edges chain: write plain field, then write volatile flag; reader sees flag, then sees plain field ("piggybacking"). |
| 10 | **`final` field freeze** (JLS 17.5, formally a separate mechanism): reads of a final field (and objects reachable *only* through it) via a properly-constructed reference see values at least as up to date as at end of constructor. | Immutable objects publishable through a data race. |

**java.util.concurrent additions (documented in the package "Memory Consistency Properties"):**
- Actions before submitting a `Runnable/Callable` to an `Executor` hb its execution begins.
- Actions in the task hb `Future.get()` returning.
- `BlockingQueue`: actions before `put`/`offer` hb actions after the matching `take`/`poll`.
- `CountDownLatch.countDown()` hb return from `await()`.
- `Semaphore.release()` hb subsequent `acquire()`; `Lock.unlock()` hb subsequent `lock()` of the same lock.
- `CyclicBarrier.await`, `Phaser.arrive`, `Exchanger.exchange` similar.
- Concurrent collections: putting an object hb another thread's access/removal of that element.
- Atomics / `VarHandle` volatile/acquire-release modes carry the same volatile semantics.

### Worked example: chaining edges (rule 9)

```java
int data;                       // plain
volatile boolean ready;         // volatile
// Thread A                     // Thread B
data = 42;      // (1)          if (ready) {          // (3) reads true
ready = true;   // (2)              print(data);      // (4) prints 42, guaranteed
                                }
// (1) hb (2) by program order; (2) hb (3) by volatile rule; (3) hb (4) by program order
// => (1) hb (4) by transitivity.  If ready were NOT volatile, (4) could print 0.
```

### What the JMM does NOT promise
- No hb between two threads that do nothing except call unrelated methods. `Thread.sleep()` / `Thread.yield()` create **no** hb edge. "I slept 1 second so it must be visible" is a bug.
- `System.out.println` may incidentally synchronize (its `PrintStream` uses a lock), which is why adding a print "fixes" visibility bugs (Heisenbugs). Never rely on it.
- Word tearing: for non-volatile `long`/`double` the spec *permits* two 32-bit writes (tearing). 64-bit HotSpot in practice writes atomically; declare `volatile` if you need the guarantee by spec.
- Out-of-thin-air values are prohibited for ordinary variables; the JMM is careful about it (causality rules), but you never need to reason about those details in interviews.

## 2.3 Safe publication

**Publication** = making a reference to an object visible to other threads. **Safe publication** = the reference *and* the object's state become visible together, and completely.

**Unsafe publication:** `public static Holder h; ... h = new Holder(42);` and another thread does `h.n`: no hb edge, so the reader may see `h != null` but `h.n == 0` (the constructor's writes may be reordered after the reference write, or be sitting in a store buffer).

### The safe-publication idioms (any one creates an hb edge)
1. **Static initializer**: `static final Holder H = new Holder();` Class initialization is guarded by the class-init lock (JLS 12.4.2); the initialisation hb every use of the class.
2. **`volatile` field** (or `AtomicReference`) holding the reference.
3. **`final` field** of a properly-constructed object (freeze semantics).
4. **Guarded by a lock**: write and read the reference inside `synchronized` (same monitor), or `Lock`.
5. **Thread-safe containers**: put into `ConcurrentHashMap`, `BlockingQueue`, `ConcurrentLinkedQueue`, `CopyOnWriteArrayList`, `Collections.synchronizedX`; other thread retrieves it.
6. **Thread handoff**: create object, then `start()` the thread (rule 4), or `Executor.submit`, or `Future.get`.

### `final` field semantics (JLS 17.5) in detail
At the **end of the constructor** the JVM performs a "freeze" of final fields (conceptually a StoreStore barrier before the constructor returns and the reference can be published). A thread that obtains the reference by *any* means, even a data race, sees:
- correct values of `final` fields,
- correct state of objects reachable *through* those final fields as of the freeze (e.g. the contents of a `final int[]` or a `final ArrayList` filled in the constructor).
It does **not** protect: non-final fields, or changes made to a final-referenced object *after* the constructor. And it is void if **`this` escapes** in the constructor (registering a listener, starting a thread, storing `this` in a static): another thread could see the half-built object with default values in the final fields.

```java
class Point { final int x, y; int z;                 // z is NOT frozen
  Point(int x, int y){ this.x=x; this.y=y; this.z=x+y;
     // BAD: EVENTS.register(this);   // this escapes before construction ends
  } }
```

### Lazy-holder (initialization-on-demand) idiom
```java
class Registry {
    private Registry() {}
    private static class Holder { static final Registry INSTANCE = new Registry(); }
    static Registry get() { return Holder.INSTANCE; }   // lazy, thread-safe, no volatile, no lock on the fast path
}
```
`Holder` is not initialised until `get()` first touches it; the JVM guarantees class init runs once under a lock; later reads are ordinary field reads (the JIT removes the check after initialisation). Prefer this or an `enum` singleton over DCL.

## 2.4 What `volatile` really guarantees

For a `volatile` variable `v` the JMM guarantees:
1. **Visibility**: a read sees the last write in the synchronization order (it is a total order over all volatile accesses and synchronization actions).
2. **Ordering (acquire/release)**: ordinary accesses before a volatile write cannot be moved *after* it; ordinary accesses after a volatile read cannot be moved *before* it. And volatile accesses are not reordered with each other (sequentially consistent among themselves).
3. **Atomicity of the single read or single write** (even for `long`/`double`). **Not** atomicity of `v++`, `v = v + 1`, or check-then-act.

It does **not** mean "reads go to main memory and skip caches" in a literal sense. Caches are coherent; what actually matters is preventing compiler register-caching/hoisting and CPU reordering.

### Barrier model (conceptual "cookbook", JSR-133)

```
 ordinary stores/loads ...
 [StoreStore]           <- before a volatile write: earlier writes are flushed first
 volatile WRITE  v = x
 [StoreLoad]            <- after a volatile write: expensive full fence
 ...
 volatile READ   r = v
 [LoadLoad]             <- later loads cannot move above the volatile read
 [LoadStore]            <- later stores cannot move above the volatile read
 ordinary stores/loads ...
```

- **LoadLoad**: loads before the barrier complete before loads after it.
- **StoreStore**: stores before the barrier become visible before stores after it.
- **LoadStore**: loads before complete before stores after.
- **StoreLoad**: stores before are visible before loads after. The most expensive and the only one x86 needs an actual instruction for: HotSpot emits a `lock`-prefixed instruction (e.g. `lock addl $0,(rsp)`) after the volatile store on x86. On ARM the JVM uses `stlr`/`ldar` (release/acquire) or `dmb` fences.

Implication: a volatile *read* on x86 is nearly free (a plain load, but the JIT cannot hoist it or cache it), a volatile *write* costs on the order of tens of cycles. Volatile writes in hot loops hurt; volatile reads mostly don't.

### Correct and incorrect uses
| Use | Verdict |
|---|---|
| Stop flag `volatile boolean running` | Correct. One writer, simple state. |
| Publishing an immutable snapshot: `volatile Config cfg;` replaced wholesale | Correct (the "immutable holder" pattern). |
| `volatile int counter; counter++` | **Wrong**: read + write are separate. Use `AtomicInteger` / `LongAdder`. |
| `volatile` array reference and `arr[i] = x` | The volatile covers only the *reference*, not elements. Use `AtomicIntegerArray` / `VarHandle` array access. |
| `if (v == null) v = create();` | Check-then-act race. |
| `volatile` to protect two related variables (`lo <= hi` invariant) | Wrong. Needs a lock or a single immutable pair object. |

## 2.5 Double-checked locking (DCL): why `volatile` is required

```java
class Singleton {
    private static volatile Singleton instance;     // <- required
    private final int a; private final String b;    // (even final fields: see note below)
    private Singleton() { a = 1; b = "x"; }
    static Singleton get() {
        Singleton r = instance;                     // one volatile read on the fast path
        if (r == null) {
            synchronized (Singleton.class) {
                r = instance;
                if (r == null) instance = r = new Singleton();
            }
        }
        return r;
    }
}
```

`instance = new Singleton()` is **not one action**. At bytecode/JIT level it is:

```
 1. memory = allocate();         // raw zeroed memory
 2. ctorInit(memory);            // run constructor: a=1, b="x"
 3. instance = memory;           // publish reference
```
Steps 2 and 3 have no data dependence for a *single* thread, so JIT/CPU may execute **1, 3, 2**.

### Interleaving trace (without `volatile`)

| Step | Thread T1 | Thread T2 | State |
|---|---|---|---|
| 1 | `get()`; `instance == null` (unlocked check) | | instance = null |
| 2 | enters `synchronized`, second check null | | |
| 3 | allocates memory (step 1 above) | | memory zeroed |
| 4 | **writes reference `instance = memory`** (reordered before ctor) | | instance != null, fields = 0/null |
| 5 | (still about to run constructor) | `get()`: first check `instance != null` -> **skips the lock** | |
| 6 | | returns `instance`, uses `a`: reads **0** (or `b == null` -> NPE) | **BUG** |
| 7 | runs constructor, `a=1` | | too late |

With `volatile`: the volatile write in step 4 cannot be reordered before the constructor's writes (StoreStore barrier before it), and T2's volatile read of a non-null reference sees everything before that write via the volatile hb rule. In the safe version above, T2 that reads `null` goes to the `synchronized` block, which gives it the hb edge from T1's unlock.

Notes:
- Since **JDK 5** (JSR-133) `volatile` DCL is correct. Pre-1.5 it was broken.
- `final` fields inside `Singleton` would be safe even without `volatile` (freeze semantics), but that is fragile; one non-final field breaks it. Use `volatile` or the holder idiom.
- The local `r` copy avoids a second volatile read on the fast path.
- On x86 you may **never** be able to reproduce the bug (demo 3.3 shows zero anomalies). Still wrong. ARM/POWER and aggressive JIT can expose it.

## 2.6 `synchronized` internals

### Semantics
- **Mutual exclusion** on the object's **monitor** (intrinsic lock). `synchronized` instance method locks `this`; static method locks `Foo.class`; block locks the given object.
- **Reentrant**: same thread can re-enter; JVM keeps a recursion count.
- **Visibility**: entering the monitor = acquire (re-read shared data); exiting = release (flush). Rule 2 above.
- Released automatically on normal exit *and* on exception.
- Bytecode: `monitorenter` / `monitorexit` (the compiler emits an exception-table entry so `monitorexit` also runs on exceptions). Synchronized *methods* use the `ACC_SYNCHRONIZED` flag instead. Check with `javap -c -v`.

### Monitor model

```
                  +----------------------------------------+
   arriving       |            MONITOR of object            |
   threads -----> |   Owner: T3 (recursions: 1)             |
                  |                                        |
   Entry set  --> |   [T7] [T9] [T12]   (BLOCKED)           |   waiting to acquire
                  |                                        |
   Wait set   --> |   [T4] [T5]         (WAITING)           |   called wait(), released monitor
                  +----------------------------------------+
   notify(): moves one thread wait set -> entry set (it still has to win the monitor)
   exit: owner leaves; one entry-set thread is woken to compete (no FIFO guarantee)
```

### Object header and the mark word (HotSpot, 64-bit, conceptual)

Every object has a header: **mark word** (8 bytes) + **klass pointer** (4 bytes compressed). The mark word is overloaded depending on state; the lowest 2 bits are the lock state:

| Lock bits | State | Mark word contents (conceptual) |
|---|---|---|
| `01` | **Unlocked** (normal) | identity hashCode (once computed), GC age |
| `00` | **Lightweight (thin) locked** | pointer to a *lock record* on the owner thread's stack (the original mark word is "displaced" into it) |
| `10` | **Heavyweight (inflated)** | pointer to an `ObjectMonitor` (C++ structure with owner, recursions, entry list, wait set) |
| `11` | GC marked | used by GC |

### Lock states and inflation (what is true in JDK 17/21)

1. **Biased locking (historical)**: JDK 6 introduced biasing the lock to the first thread so re-locking cost nothing (no CAS). It made revocation complicated (safepoint pauses), and modern hardware CAS is cheap. **Deprecated in JDK 15 and disabled by default (JEP 374); the code was removed in JDK 18.** So on JDK 17 the flags exist but biased locking is off by default; on JDK 21 it is gone. Old blog posts describing "biased -> thin -> fat" are describing JDK 6-14.
2. **Thin / lightweight locking (stack locking)**: uncontended case. Thread copies the mark word into a *lock record* in its stack frame and does one CAS to replace the mark word with a pointer to that record. Success = lock acquired, cost = one CAS. Recursive entry: another lock record with zero displaced word. (JDK 21 still uses this classic scheme; later JDK releases re-implemented lightweight locking with a per-thread "lock stack", which is an internal detail you do not need for interviews.)
3. **Inflation to a heavyweight monitor**: when a second thread contends (CAS fails and spinning does not help), or `wait()`/`notify()` is called, or `hashCode` interplay requires it, the JVM allocates an `ObjectMonitor` and points the mark word at it. Contending threads then **spin briefly (adaptive spinning)** and if still unsuccessful **park** via the OS (futex on Linux; expensive: syscall + context switch). Inflated monitors are deflated later by the JVM housekeeping (async monitor deflation since JDK 15).

```
 unlocked (01) --CAS ok--> thin locked (00) --contention / wait()--> inflated (10, ObjectMonitor)
       ^                                                                   |
       +---------------------- deflation (async, by JVM) -------------------+
```

Practical points:
- `wait()`/`notify()` always use the inflated monitor.
- Calling `System.identityHashCode()` / default `hashCode()` stores the hash in the mark word.
- **Virtual threads (JDK 21):** a virtual thread blocked inside `synchronized` *pins* its carrier thread (JDK 24 fixed this via JEP 491). In JDK 21 prefer `ReentrantLock` on paths that block for long inside virtual threads.

### JIT optimisations on locks (conceptual)
- **Lock elision (escape analysis)**: if the object never escapes the thread (e.g. a local `StringBuffer`), the JIT removes the locking entirely.
- **Lock coarsening**: consecutive lock/unlock on the same object in a loop or straight-line code (e.g. repeated `sb.append`) are merged into one larger critical section.
- **Adaptive spinning**: before parking, spin for a duration learned from past outcomes.
- **Consequence:** micro-benchmarks of "uncontended synchronized" are misleading; the JVM may remove the lock. Use JMH.

### `synchronized` vs `ReentrantLock`

| | `synchronized` | `ReentrantLock` |
|---|---|---|
| Release | automatic | manual, `finally { unlock(); }` |
| Non-blocking try / timeout | no | `tryLock()`, `tryLock(t, unit)` |
| Interruptible acquisition | no | `lockInterruptibly()` |
| Fairness option | no | `new ReentrantLock(true)` |
| Multiple wait-sets | one per monitor | many `Condition`s |
| Diagnostics | thread dump shows monitors | `isLocked()`, `getQueueLength()`, `getOwner()` (protected) |
| Cost today | comparable; sometimes faster uncontended (elision, thin lock) | comparable |
| Virtual threads (JDK 21) | pins carrier | does not pin |
| Scope | block/method structure only (lock and unlock in same block) | hand-over-hand locking possible |

Rule of thumb: use `synchronized` by default (simplest, cannot forget to unlock); use `ReentrantLock` when you need `tryLock`, interruptibility, fairness or multiple conditions.

## 2.7 `wait` / `notify` semantics

```java
synchronized (lock) {                    // MUST hold the monitor, else IllegalMonitorStateException
    while (!conditionHolds()) {          // WHILE, never IF
        lock.wait();                     // releases monitor (all recursion levels), joins wait set
    }
    // condition true and we hold the monitor again
}
// producer:
synchronized (lock) { changeState(); lock.notifyAll(); }   // waiters can't proceed until we exit the block
```

Precise semantics:
- `wait()`: atomically **releases the monitor** and adds the thread to the wait set; thread state `WAITING` (or `TIMED_WAITING` for `wait(ms)`). It returns only after: `notify`/`notifyAll` selecting it, interruption (throws `InterruptedException`, flag cleared), timeout, or a **spurious wakeup**; and then it must **re-acquire the monitor** (in the meantime it is `BLOCKED` in the entry set).
- `notify()` wakes **one arbitrary** waiter; `notifyAll()` wakes all (they then compete for the monitor one at a time). Neither releases the monitor: the notified thread proceeds only when the notifier exits `synchronized`.
- **Why `while`**: (1) *spurious wakeups* are permitted by the JLS (underlying OS primitives can return without a signal); (2) *stolen wakeup*: between the notify and the waiter reacquiring the monitor, another thread may have consumed the state; (3) `notifyAll` wakes threads whose condition is not yet true. Always re-check the predicate.
- **Lost notification**: `notify()` before the waiter called `wait()` is lost (no memory). Hence state (the predicate) must live in a variable guarded by the same lock, and the waiter checks it *before* waiting.
- **`notify` vs `notifyAll`**: use `notifyAll` unless all waiters are equivalent and exactly one can proceed. Mixed conditions plus `notify` can wake the wrong thread, leaving the right one asleep forever (a hang).
- `sleep()` keeps monitors it holds; `wait()` releases only the monitor of the object it waits on (other locks stay held: nested-monitor lockout risk).
- `Thread.join()` is implemented with `wait()` on the Thread object (which is why you should never use a `Thread` object as a lock).

## 2.8 Thread states and reading `jstack`

`Thread.State` has 6 values: `NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED`.

```
                         start()
   NEW ───────────────────────────────► RUNNABLE ◄───────────────┐
                                        │  ▲  (running or ready) │
             ┌──────────────────────────┘  │                     │
             │ waiting to ENTER a          │ monitor acquired    │ notify/notifyAll/unpark/
             │ synchronized block          │                     │ join target ends/interrupt
             ▼                             │                     │
          BLOCKED ─────────────────────────┘                     │
                                                                 │
   RUNNABLE ── Object.wait() | join() | LockSupport.park() ──► WAITING ─┘
   RUNNABLE ── sleep(ms) | wait(ms) | join(ms) | parkNanos ──► TIMED_WAITING ─► (timeout/unpark) ─► RUNNABLE
   RUNNABLE ── run() returns / uncaught exception ──► TERMINATED
   Note: after notify(), a wait()ing thread goes WAITING -> BLOCKED (waiting to re-acquire the monitor) -> RUNNABLE
```

Transition table:

| From | To | Trigger |
|---|---|---|
| NEW | RUNNABLE | `start()` (calling twice throws `IllegalThreadStateException`) |
| RUNNABLE | BLOCKED | tries to enter `synchronized` held by another thread |
| BLOCKED | RUNNABLE | monitor acquired |
| RUNNABLE | WAITING | `Object.wait()`, `Thread.join()`, `LockSupport.park()` (this includes `ReentrantLock.lock()` waiting, `Condition.await()`, `CountDownLatch.await()`, `Future.get()`) |
| RUNNABLE | TIMED_WAITING | `sleep`, timed `wait/join/parkNanos/tryLock(timeout)` |
| WAITING/TIMED_WAITING | RUNNABLE | notify/unpark/interrupt/timeout (via BLOCKED for monitor-based waits) |
| RUNNABLE | TERMINATED | `run()` completes or throws |

**Traps that interviewers love:**
- Threads blocked in `ReentrantLock.lock()` are **WAITING (parking)**, *not* BLOCKED. `BLOCKED` is exclusively about monitor entry.
- A thread blocked in **socket/file I/O** or in a native call is shown as **RUNNABLE** although it consumes no CPU. High "RUNNABLE" count in a dump does not mean CPU saturation; check `top -H`.
- There is no "RUNNING" state, and `BLOCKED` is not "waiting on I/O".

### Reading a dump (real output, demo 3.4 variant on JDK 21)

Take a dump with `jstack <pid>` (or `jcmd <pid> Thread.print`, or `kill -3 <pid>` on Linux which prints to stdout; for JSON with virtual threads `jcmd <pid> Thread.dump_to_file -format=json file`).

```
"worker-1" #21 [6228] prio=5 os_prio=0 cpu=0.00ms elapsed=4.47s tid=0x... nid=6228 waiting for monitor entry
   java.lang.Thread.State: BLOCKED (on object monitor)
	at DL.lambda$main$0(DL.java:5)
	- waiting to lock <0x00000007219e6220> (a java.lang.Object)      <- what I want
	- locked <0x00000007219e6210> (a java.lang.Object)               <- what I hold
...
Found one Java-level deadlock:
=============================
"worker-1":
  waiting to lock monitor 0x000001ae9c7d4960 (object 0x00000007219e6220, a java.lang.Object),
  which is held by "worker-2"
"worker-2":
  waiting to lock monitor 0x000001aeff17ec00 (object 0x00000007219e6210, a java.lang.Object),
  which is held by "worker-1"
```

Field guide to lines in a dump:

| Line | Meaning |
|---|---|
| `waiting for monitor entry` / `BLOCKED (on object monitor)` + `waiting to lock <addr>` | contending for a `synchronized`. Find the thread with `locked <addr>` (same address): that's the holder. |
| `in Object.wait()` + `waiting on <addr>` + `WAITING (on object monitor)` | in a wait set. |
| `parking to wait for <addr> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)` + `WAITING (parking)` | blocked on a j.u.c primitive; the object type (`Sync`, `CountDownLatch$Sync`, `FutureTask`, `ThreadPoolExecutor$Worker`...) tells you which. For `ReentrantLock` the *owner* is printed in the `Locked ownable synchronizers` section of the holding thread (`- <0x...> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)`). |
| `TIMED_WAITING (sleeping)` | `Thread.sleep`. |
| `RUNNABLE` at `java.net.SocketInputStream.socketRead0` / `sun.nio.ch...EPoll.wait` | blocked on network I/O, not CPU. |
| `cpu=..ms elapsed=..s` (JDK 17/21 dumps) | CPU time consumed vs age: compare to find hot threads. |
| `nid=0x..` | native thread id in hex. Map to `top -H -p <pid>` (decimal thread id) to find the CPU-hot thread. |

**Workflow:** take 3 dumps 5-10 s apart; threads in the same frame in all three are stuck. Deadlocks of *monitors and ownable synchronizers* are auto-detected by `jstack` ("Found one Java-level deadlock"); deadlocks involving only `wait/notify`, latches, or missing signals are **not** flagged; you must read the stacks.

## 2.9 AbstractQueuedSynchronizer (AQS)

AQS (`java.util.concurrent.locks.AbstractQueuedSynchronizer`) is the framework behind `ReentrantLock`, `ReentrantReadWriteLock`, `Semaphore`, `CountDownLatch` (and `FutureTask`, `ThreadPoolExecutor.Worker` in older/newer versions). You implement a small `tryAcquire`-style protocol over an `int state`; AQS supplies queueing, blocking, waking, cancellation, timeouts, interruption.

### Three ingredients
1. **`volatile int state`** with `getState()`, `setState()`, `compareAndSetState()`. Meaning is defined by the subclass:
   - `ReentrantLock`: 0 = free, n = hold count (reentrancy).
   - `Semaphore`: available permits.
   - `CountDownLatch`: remaining count (open when 0).
   - `ReentrantReadWriteLock`: high 16 bits = read holds (total), low 16 bits = write holds.
2. **A FIFO wait queue**, a variant of the **CLH** (Craig-Landin-Hagersten) lock queue: a doubly-linked list of `Node`s, each holding a thread and a wait status; `head` is a dummy/current-holder node, `tail` is where new waiters are CAS-appended. Threads spin briefly and then **park** (`LockSupport.park`); the predecessor **unparks** its successor on release.
3. **Template methods** the subclass overrides: exclusive mode `tryAcquire/tryRelease`; shared mode `tryAcquireShared/tryReleaseShared`; and `isHeldExclusively` (for `Condition`). The final methods `acquire/release/acquireShared/releaseShared` (+ interruptible/timed variants) implement the algorithm.

```
                      state (volatile int)
                            |
   head (dummy) <-> [Node T1] <-> [Node T2] <-> [Node T3] <- tail
       ^ current holder / just released     each waiting node: thread + status, parked
```

### Exclusive acquire flow (conceptual; the code was refactored in JDK 17 but the logic is the same)

```
acquire(arg):
   1. if tryAcquire(arg) succeeds              -> return (fast path: usually 1 CAS on state)
   2. else create Node(currentThread) and CAS-append to tail  (init head/tail lazily)
   3. loop:
        if node's predecessor is head:          // I'm first in line
             if tryAcquire(arg) succeeds -> make node the new head, drop old head, return
        else / if failed:
             mark predecessor's status = SIGNAL ("wake me when you release")
             LockSupport.park(this)             // block until unparked or interrupted
        after wake: re-loop (park may return spuriously; interruption remembered, flag re-set on exit)
release(arg):
   1. if tryRelease(arg) returns true (state fully released):
   2.     unpark the first non-cancelled successor of head
```

Details worth knowing:
- **Barging**: step 1 lets a newly arriving thread try to acquire *before* queuing, even if others are queued: that's why the default is "nonfair". The woken successor must still *compete* with barging threads.
- **Interruption/cancellation**: `acquireInterruptibly` throws on interrupt; a cancelled node is skipped/unlinked by neighbours.
- **`Condition`** waits use a *separate* queue (see 2.10). A node moves from the condition queue to the sync queue on `signal`.

### Exclusive vs shared mode
- **Exclusive**: at most one holder (`ReentrantLock`, write lock).
- **Shared**: many holders can succeed at once. `tryAcquireShared` returns `< 0` fail, `0` success but no further sharing, `> 0` success and later shared waiters may also succeed. On success the woken node **propagates**: it unparks the *next* node too if it is also shared (so all queued readers wake as a chain, or all `CountDownLatch.await` threads are released when the count reaches 0). `Semaphore` (permits), `CountDownLatch`, read lock use shared mode.

### Custom AQS example (a 1-permit non-reentrant mutex)
```java
final class Mutex {
    private static class Sync extends AbstractQueuedSynchronizer {
        protected boolean tryAcquire(int ignore) { return compareAndSetState(0, 1) ? setOwner() : false; }
        private boolean setOwner() { setExclusiveOwnerThread(Thread.currentThread()); return true; }
        protected boolean tryRelease(int ignore) {
            if (getState() == 0) throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null); setState(0); return true;   // volatile write publishes
        }
        protected boolean isHeldExclusively() { return getState() == 1; }
    }
    private final Sync sync = new Sync();
    void lock() { sync.acquire(1); }  void unlock() { sync.release(1); }
}
```
That is all you write; queueing and parking are inherited. (Mirrors the class-Javadoc example of AQS.)

## 2.10 `ReentrantLock`: source-level flow, fairness, `Condition`

`ReentrantLock` holds a `Sync` (extends AQS) with two subclasses `NonfairSync` (default) and `FairSync`.

### Nonfair `tryAcquire` (paraphrased from the JDK source, `Sync.nonfairTryAcquire`)
```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {                                          // lock is free
        if (compareAndSetState(0, acquires)) {             // BARGE: try immediately, ignoring queue
            setExclusiveOwnerThread(current);
            return true;
        }
    } else if (current == getExclusiveOwnerThread()) {     // reentrant
        int nextc = c + acquires;
        if (nextc < 0) throw new Error("Maximum lock count exceeded");
        setState(nextc);                                   // plain volatile write: only owner touches it
        return true;
    }
    return false;                                          // AQS will enqueue and park
}
```

### Fair `tryAcquire`
Same, but at `c == 0`:
```java
if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) { ... }
```
`hasQueuedPredecessors()` = "is there any thread queued ahead of me?" If yes, do not barge; queue behind them. Result: strict FIFO among *queued* threads.

### `tryRelease`
```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread()) throw new IllegalMonitorStateException();
    boolean free = (c == 0);
    if (free) setExclusiveOwnerThread(null);
    setState(c);                          // volatile write = the release edge (hb rule for Lock.unlock)
    return free;                          // only when fully released does AQS unpark a successor
}
```

### Fair vs nonfair

| | Nonfair (default) | Fair |
|---|---|---|
| Throughput | Higher: a running thread can grab the lock right after release without a context switch to the waiter | Lower (typically 10x or more under contention): each handoff wakes a parked thread |
| Starvation | Possible in theory | Prevented for queued threads |
| `tryLock()` (no-arg) | Barges even on a fair lock (!) | Use `tryLock(0, SECONDS)` to honour fairness |
| Use when | almost always | when order truly matters and lock hold time is significant |

Fairness of the lock does not guarantee fairness of *thread scheduling*.

### `lock()` vs `lockInterruptibly()` vs `tryLock()`
- `lock()`: not interruptible; if interrupted while waiting it keeps waiting and re-sets the interrupt flag on return.
- `lockInterruptibly()`: throws `InterruptedException` if interrupted (before or while waiting).
- `tryLock()`: single barging attempt, returns immediately. `tryLock(time, unit)`: waits up to timeout, interruptible.
- **Canonical usage**:
```java
lock.lock();
try { /* critical section */ } finally { lock.unlock(); }   // lock() OUTSIDE try, so a failed lock() isn't "unlocked"
```
- `unlock()` by a non-owner throws `IllegalMonitorStateException`. Unlock count must match lock count.

### `Condition` (await/signal) mechanics
```java
private final ReentrantLock lock = new ReentrantLock();
private final Condition notFull = lock.newCondition(), notEmpty = lock.newCondition();
void put(E e) throws InterruptedException {
    lock.lock();
    try {
        while (count == items.length) notFull.await();     // must hold lock
        enqueue(e); notEmpty.signal();
    } finally { lock.unlock(); }
}
```
Internally (`AbstractQueuedSynchronizer.ConditionObject`):
1. `await()`: (a) add a node to the *condition queue* of this Condition, (b) **fully release** the lock, saving the hold count (so reentrant locks are fully released), (c) park while the node is not on the sync queue, (d) after signal it is on the sync queue: re-acquire the lock with the **saved hold count**, (e) handle interrupt policy (throw `InterruptedException` if interrupted before signal, else re-assert the flag).
2. `signal()`: requires holding the lock (else `IllegalMonitorStateException`); moves the first waiting node from the condition queue to the sync queue (and unparks it if needed). It will actually run only after the signaller releases the lock.
3. `signalAll()`: moves all.
4. Spurious wakeups are allowed here too: always `while`.
5. Advantage over `wait/notify`: multiple conditions per lock (`notFull` and `notEmpty`) so `signal()` can target the right class of waiters instead of `notifyAll` thundering herd. `ArrayBlockingQueue` is exactly this design.

```
 await():   holder --release fully--> [condition queue]  ---signal--->  [sync queue] --acquire--> continues
```

## 2.11 `ReentrantReadWriteLock` and `StampedLock`

### `ReentrantReadWriteLock`
- One AQS `state` int **split**: `state >>> 16` = number of read holds (shared count), `state & 0xFFFF` = write hold count (reentrancy). Each limited to 65535. Per-thread read hold counts are tracked in a `ThreadLocal` (plus fast-path cache for first reader) to enforce correct unlocking.
- Rules: many readers OR one writer. A writer waits until read count is 0. Readers are blocked while a writer holds.
- **Reentrancy**: a thread holding write lock can acquire read or write again. A reader can re-acquire read.
- **Downgrading is allowed**: hold write -> acquire read -> release write. You keep a consistent view and let other readers in (demo 3.6).
- **Upgrading is NOT supported**: holding read and calling `writeLock().lock()` **deadlocks itself** (write waits for read count 0, but it holds a read). `tryLock()` returns false (demo 3.6). Fix: release read, acquire write, **re-check** state.
- **Fair mode** (`new ReentrantReadWriteLock(true)`): approximate arrival order; nonfair mode (default) uses a heuristic: a reader will *not* barge if the first queued thread is a writer (`apparentlyFirstQueuedIsExclusive`), which reduces (not eliminates) writer starvation. Under steady read load in nonfair mode writers can still wait long; a continuous stream of overlapping readers keeps count > 0.
- `ReadLock.newCondition()` throws `UnsupportedOperationException`; only the write lock has conditions.
- Costs: read lock still does CAS on shared state; for *very* short critical sections a plain lock can beat RRWL. RRWL wins when reads are long and much more frequent than writes.
- A read-write lock does **not** make a "read-mostly" cache correct if reads populate it (lazy load = write).

### `StampedLock` (JDK 8+)
Not built on AQS's `Lock` interface; has its own state and queue. Modes: write (exclusive), read (shared), **optimistic read** (no lock at all, returns a stamp).

```java
class Point {
    private double x, y; private final StampedLock sl = new StampedLock();
    double distanceFromOrigin() {
        long stamp = sl.tryOptimisticRead();          // 0 if write-locked, otherwise a version stamp
        double cx = x, cy = y;                        // read into locals (may be inconsistent!)
        if (!sl.validate(stamp)) {                    // did any writer run since the stamp?
            stamp = sl.readLock();                    // fall back to a real read lock
            try { cx = x; cy = y; } finally { sl.unlockRead(stamp); }
        }
        return Math.sqrt(cx*cx + cy*cy);              // compute only on validated locals
    }
    void move(double dx, double dy) {
        long stamp = sl.writeLock();
        try { x += dx; y += dy; } finally { sl.unlockWrite(stamp); }
    }
}
```
Rules:
- **Optimistic read pattern**: read fields into locals, `validate(stamp)`, and only then use them. Between read and validate you may see a torn, inconsistent state, so do nothing with side effects or invariants (e.g. don't index arrays with those values before validating) until validated.
- **Not reentrant**: locking twice from the same thread self-deadlocks. Stamps are not owned by threads; any thread may unlock with the stamp.
- **No `Condition`s**, no fairness policy, no interruption of `readLock()`/`writeLock()` (use `readLockInterruptibly`).
- Conversions: `tryConvertToWriteLock(stamp)` (may fail, returns 0), `tryConvertToReadLock`, `tryConvertToOptimisticRead`.
- Best for read-heavy, short critical sections where writers are rare.

## 2.12 `Semaphore`, `CountDownLatch`, `CyclicBarrier`, `Phaser`, `LockSupport`

| Class | Built on | Semantics | Notes |
|---|---|---|---|
| `Semaphore(n)` | AQS shared, `state` = permits | `acquire` decrements (blocks at 0), `release` increments; permits are not tied to threads (any thread can release) | fair option; `tryAcquire`; limits concurrency to n (resource pools, rate-limit connections); `Semaphore(1)` is a non-reentrant mutex that another thread may release |
| `CountDownLatch(n)` | AQS shared, `state` = count | `countDown()` decrements; `await()` blocks until 0; then open **forever** | one-shot; cannot reset; `countDown` hb `await` return; if a worker dies before `countDown`, waiters hang (use `await(timeout)` and `finally { countDown(); }`) |
| `CyclicBarrier(n[, action])` | `ReentrantLock` + `Condition` (not AQS directly) | `await()` blocks until n parties arrive; then the barrier action runs (by the last arriver) and everyone released; **reusable** (generations) | If a party is interrupted or times out, barrier becomes **broken** (`BrokenBarrierException` for everyone else); `reset()` |
| `Phaser` | own implementation: a packed `long` state (phase, parties, unarrived) + Treiber-stack queues, tiering to reduce contention | dynamic parties (`register`, `arriveAndDeregister`), multiple phases, `onAdvance` hook; can terminate | flexible generalisation of latch + barrier |
| `Exchanger<V>` | CAS-based slots | two threads swap objects at a rendezvous | rare |

Latch vs barrier: latch = "wait for events" (parties differ from waiters, one-shot); barrier = "wait for each other" (all parties both arrive and wait, reusable).

### `LockSupport.park/unpark` (the primitive under everything)
- Each thread has a **permit** (0 or 1, not cumulative). `unpark(t)` makes the permit available (if already available, no effect). `park()` consumes the permit and returns immediately if available; otherwise blocks.
- `park()` may **return spuriously** and returns on interrupt (without throwing; the flag stays set) — so it must be used in a loop that re-checks a condition.
- Unlike `wait/notify`: no monitor needed; `unpark` before `park` is **not lost** (permit persists, max 1).
- `parkNanos`, `parkUntil` for timed. `park(Object blocker)` records the blocker shown in thread dumps ("parking to wait for <..>").
- HotSpot implementation: a per-thread `Parker` using a mutex + condition variable (pthread) on POSIX (details are platform-specific).

## 2.13 Interruption semantics

- Interruption is **cooperative**: `t.interrupt()` sets the thread's **interrupt flag**; it doesn't stop anything by itself. The target must poll the flag or be in an interruptible blocking call.
- Blocking methods that throw `InterruptedException` (`sleep`, `wait`, `join`, `BlockingQueue.put/take`, `Condition.await`, `lockInterruptibly`, `Future.get`, `Semaphore.acquire`, `CountDownLatch.await`...): if the flag is set when called, or it is set while blocked, they **clear the flag** and throw.
- `Thread.interrupted()` (static): returns flag **and clears** it. `t.isInterrupted()`: returns flag, doesn't clear.
- Not interruptible: blocking `InputStream.read` on classic sockets/files, `synchronized` entry, `Lock.lock()`. (Close the socket, or use NIO `InterruptibleChannel`, or `lockInterruptibly`.)
- Interrupting a thread that is not blocked simply leaves the flag set until checked.

### Rules for handling `InterruptedException` (never swallow)
```java
// WRONG: swallowed, the request to stop is lost forever
try { queue.take(); } catch (InterruptedException e) { }

// WRONG: only logs
try { Thread.sleep(100); } catch (InterruptedException e) { e.printStackTrace(); }

// RIGHT 1: propagate (declare throws), the caller decides
void work() throws InterruptedException { queue.take(); }

// RIGHT 2: cannot throw (e.g. Runnable, override): restore the flag and stop
try { queue.take(); }
catch (InterruptedException e) { Thread.currentThread().interrupt(); return; }   // re-assert

// RIGHT 3 (loop): exit the loop
while (!Thread.currentThread().isInterrupted()) {
    try { process(queue.take()); }
    catch (InterruptedException e) { Thread.currentThread().interrupt(); break; }
}
```
Reason: the flag is cleared when the exception is thrown; if you neither propagate nor restore, upper layers (executor shutdown via `shutdownNow`, request timeouts) cannot cancel the work. Exception: you own the thread's whole lifecycle and are about to exit anyway.
- `ExecutorService.shutdownNow()` works by `interrupt()`; tasks that swallow interrupts never stop.
- `Future.cancel(true)` interrupts the running thread.

## 2.14 CAS, `Unsafe`, `VarHandle`, atomics, ABA, `LongAdder`

### CAS (compare-and-swap)
A single atomic CPU instruction: "if memory location == expected then set to new, return success; else return failure". x86: `lock cmpxchg`; ARM: `ldxr/stxr` loop or `cas` (LSE). It is the basis of all lock-free algorithms and of AQS itself.

```java
// what AtomicInteger.incrementAndGet() effectively does (getAndAddInt loop)
int prev;
do { prev = get(); } while (!compareAndSet(prev, prev + 1));
return prev + 1;
```
```
 Thread A: read 5 -----------------> CAS(5->6) succeeds
 Thread B: read 5 --> CAS(5->6) FAILS (value is 6 now) --> retry: read 6, CAS(6->7) succeeds
```
Lock-free = the system as a whole always progresses (some thread's CAS succeeds), but an individual thread may retry indefinitely (not wait-free). No blocking, no context switch, no deadlock, no priority inversion. Under **high contention** many CAS failures waste CPU and the cache line bounces between cores.

### `Unsafe` and `VarHandle`
- `sun.misc.Unsafe` / `jdk.internal.misc.Unsafe`: intrinsic access to memory offsets and CAS (`compareAndSetInt` on object field offsets). Used inside the JDK. Application code should not use it (internal, removal underway for memory-access methods).
- **`java.lang.invoke.VarHandle` (Java 9+)** is the supported replacement: typed handle to a field/array element/ByteBuffer with access modes:

| Mode family | Methods | Semantics |
|---|---|---|
| Plain | `get`, `set` | like ordinary access |
| Opaque | `getOpaque`, `setOpaque` | atomic, coherent per variable, no ordering with others |
| Acquire/Release | `getAcquire`, `setRelease` | one-way ordering (cheaper than volatile) |
| Volatile | `getVolatile`, `setVolatile` | full volatile semantics |
| CAS / RMW | `compareAndSet`, `weakCompareAndSet*`, `getAndAdd`, `getAndSet`, `getAndBitwiseOr`... | atomic |

```java
class Counter {
    private volatile int value;
    private static final VarHandle VALUE;
    static { try { VALUE = MethodHandles.lookup().findVarHandle(Counter.class, "value", int.class); }
             catch (ReflectiveOperationException e) { throw new ExceptionInInitializerError(e); } }
    int inc() { return (int) VALUE.getAndAdd(this, 1) + 1; }
}
```
Fence-only API: `VarHandle.fullFence()`, `acquireFence()`, `releaseFence()`, `loadLoadFence()`, `storeStoreFence()`. Most application code should stick to atomics and locks.

### The atomic classes (`java.util.concurrent.atomic`)
| Class | Purpose |
|---|---|
| `AtomicInteger/Long/Boolean` | counters, flags. `incrementAndGet`, `getAndAdd`, `compareAndSet`, `updateAndGet(fn)`, `accumulateAndGet(x, fn)` (fn must be side-effect free: may be retried) |
| `AtomicReference<V>` | atomic swap of immutable snapshots; lock-free stacks (Treiber stack) |
| `AtomicIntegerArray/LongArray/ReferenceArray` | element-wise atomics |
| `AtomicIntegerFieldUpdater`... | atomic ops on a `volatile` field of another class (saves an object per instance) |
| `AtomicStampedReference<V>` | reference + int stamp (defeats ABA) |
| `AtomicMarkableReference<V>` | reference + boolean mark |
| `LongAdder`, `DoubleAdder`, `LongAccumulator` | high-contention accumulation |

Internals: `AtomicInteger` stores `private volatile int value` and uses `Unsafe`/`VarHandle` intrinsics for CAS (implementation detail depends on class and JDK version).

Pitfalls:
- `compareAndSet` on `AtomicReference` compares by **reference identity** (`==`), not `equals`. With boxed `Integer` outside the cache range (-128..127), `ref.compareAndSet(Integer.valueOf(1000), x)` fails since it's a different object.
- Two independent atomics do **not** make a compound invariant atomic (`AtomicInteger lo, hi` with `lo<=hi`): put both into one immutable object in one `AtomicReference`, or use a lock.
- `AtomicInteger.get()` then `set()` is a check-then-act race; use `updateAndGet`/`compareAndSet`.

### ABA problem
```
 Lock-free stack: top -> A -> B -> C
 T1: reads top=A, next=B, prepares CAS(top, A, B) ... gets preempted
 T2: pop A, pop B, push A back  =>  top -> A -> C      (B is gone, freed/reused)
 T1 resumes: CAS(top, A, B) succeeds (top is still "A")  =>  top -> B  (B is garbage!) CORRUPTED
```
Value looked unchanged but history changed. With garbage-collected Java it is rarer than in C (nodes aren't recycled behind your back) but still real when you pool/reuse nodes or when "value A" carries meaning across states (e.g. balance 100 -> 50 -> 100).
Fix: **`AtomicStampedReference<V>`**: CAS on (reference, stamp) pair, increment the stamp on every change.
```java
AtomicStampedReference<Node> top = new AtomicStampedReference<>(null, 0);
int[] h = new int[1];
Node cur = top.get(h);                       // reads ref and stamp together
top.compareAndSet(cur, next, h[0], h[0] + 1);// fails if either changed
```
(`AtomicMarkableReference` if you only need "logically deleted" flag.)

### `LongAdder` vs `AtomicLong` under contention
- `AtomicLong`: one memory location. N threads CAS the same cache line: cache-line ping-pong (MESI invalidations), CAS failures, retries. Throughput *falls* as threads increase.
- `LongAdder` (extends `Striped64`): a `volatile long base` plus a lazily created **`Cell[]`** table (power-of-2 size, grows up to about the number of CPUs). `add(x)`:
  1. Try CAS on `base` (no contention -> behaves like AtomicLong).
  2. If it fails, pick a cell via the thread's **probe** (per-thread hash), CAS that cell's `value`. Each `Cell` is annotated `@Contended` (padded to its own cache line, avoids false sharing).
  3. If the cell CAS also fails, rehash the thread's probe; if cell missing, create; if collisions persist, **double the table** (up to the CPU-count bound).
- `sum()` = `base` + all cells, **not an atomic snapshot** (concurrent adds may or may not be counted). `sumThenReset()` likewise. Do not use `LongAdder` for values you compare-and-decide on (sequence numbers, "if count == limit"): use `AtomicLong` there.
- Costs: more memory, `sum()` is O(cells). Use for metrics, hit counters, statistics.
- `ConcurrentHashMap.size()` uses the same idea (`baseCount` + `CounterCell[]`).

Observed in demo 3.5 (12 cores, JDK 21): 8 threads x 5M increments: AtomicLong ~560-610 ms vs LongAdder ~20 ms; single thread they are equal.

---

## 2.15 `ConcurrentHashMap` (Java 8+) concurrency internals

Java 7 used `Segment[]` (each a `ReentrantLock` protecting a mini hash table; default concurrency level 16). **Java 8 removed segments**: one `Node<K,V>[] table`; concurrency control per **bin (bucket)**, using CAS for empty bins and `synchronized` on the bin's first node otherwise. Locking granularity therefore scales with the table size.

### Structure and key fields
```java
transient volatile Node<K,V>[] table;      // the buckets (power of 2 length)
private transient volatile Node<K,V>[] nextTable;   // only non-null while resizing
private transient volatile long baseCount;          // element count when uncontended
private transient volatile CounterCell[] counterCells; // striped counts under contention
private transient volatile int sizeCtl;              // control word (see below)
static class Node<K,V> { final int hash; final K key; volatile V val; volatile Node<K,V> next; }
```
- `Node.val` and `next` are **volatile**, so `get()` needs **no lock**: it reads the table slot with a volatile read (`tabAt` = `Unsafe.getReferenceAcquire`) and walks the chain, seeing safely-published nodes.
- Special nodes: `ForwardingNode` (hash = `MOVED` = -1, marks a bin already migrated during resize), `TreeBin` (hash = -2, root holder of red-black tree bins with its own read/write lock), `ReservationNode` (hash = -3, placeholder used by `computeIfAbsent`/`compute` on an empty bin).
- **Spread**: `spread(h) = (h ^ (h >>> 16)) & 0x7fffffff` mixes high bits into low ones because the index uses `(n - 1) & hash`; the sign bit is cleared so normal hashes are non-negative and never collide with negative special hash values.

### `sizeCtl`
| Value | Meaning |
|---|---|
| `0` (default) | table not yet allocated; use default capacity 16 |
| `> 0` (before init) | initial capacity requested via constructor |
| `> 0` (after init) | **next resize threshold** (0.75 * capacity) |
| `-1` | some thread is **initialising** the table |
| `< -1` | **resizing** in progress: high bits hold a *resize stamp* (derived from table length), low 16 bits hold (number of resizing threads + 1) |

### `putVal` flow

```
putVal(key, value):
  if key == null || value == null -> NullPointerException
  hash = spread(key.hashCode())
  loop forever:
   tab = table
   1. if tab == null or empty        -> initTable()
        (initTable: CAS sizeCtl 0/positive -> -1; only the winner allocates; others Thread.yield() and re-loop)
   2. i = (n-1) & hash; f = tabAt(tab, i)
      if f == null                     -> casTabAt(tab, i, null, new Node(...))   // NO LOCK
                                          success -> break;  failure -> another thread inserted; re-loop
   3. else if f.hash == MOVED (-1)     -> tab = helpTransfer(tab, f)              // join the resize, then re-loop
   4. else:                              // bin has nodes
        synchronized (f) {               // lock ONLY this bin (head node as the monitor)
            if (tabAt(tab, i) == f) {    // re-verify head hasn't changed (resize/removal)
                 if f.hash >= 0:  walk the list: replace value if key found (unless onlyIfAbsent), else append at tail; binCount++
                 else if f is TreeBin: tree insert
            }
        }
        if binCount >= TREEIFY_THRESHOLD (8): treeifyBin(tab, i)   // (converts to tree only if table length >= 64, else resizes)
        if replaced an existing key: return old value
        break
  addCount(1, binCount)     // update size, maybe trigger resize
  return null
```
Why it is correct:
- **CAS on empty bin**: two threads inserting into the same empty bin: exactly one CAS wins; the loser loops and now sees a non-null head, so it takes the synchronized path. No lock ever needed for the common case (empty bin).
- **Synchronized on head node**: threads in *different bins* never block each other; threads in the same bin serialize. The check `tabAt(tab,i) == f` after entering handles the case where the head was replaced or moved while waiting for the monitor.
- **Reads never lock.** A `get` racing with a `put` sees either the old or new state of the bin, never garbage, thanks to volatile fields and the fact that inserts append at the tail with a volatile `next` write (the node is fully built before being linked).

### Size counting: `baseCount` + `CounterCell[]`
`addCount(x, check)`:
1. If `counterCells == null`, try to CAS `baseCount += x`. If it succeeds, done (no contention).
2. Otherwise pick a `CounterCell` by the thread's probe and CAS it; on repeated failure call `fullAddCount` (creates/expands cells; same idea as `LongAdder`).
3. If `check >= 0` (i.e., insertion path), compute `s = sumCount()`; **while `s >= sizeCtl`** trigger/join a resize (`transfer`).

`size()` = `sumCount()` = `baseCount + Σ cells[i].value`, clamped to `int` range. It is an **estimate under concurrent modification**, never a linearizable snapshot. Use `mappingCount()` (returns `long`) for huge maps. Never use `size()` for control flow like `if (map.size() < MAX) map.put(...)` (check-then-act).

### Resize: `transfer`, `ForwardingNode`, `helpTransfer`
Resizing is done **cooperatively and incrementally** (no stop-the-world lock):

```
 old table (n)                              nextTable (2n)
 [0][1][2][3][4][5][6][7]                   [0][1]...[15]
  ^  ^                                       
  |  |  each finished old bin is replaced by a ForwardingNode(nextTable), hash = MOVED
  thread A claims stride [4..7], thread B claims [0..3] (transferIndex decreases atomically)
```
1. First thread that detects `size >= sizeCtl`: CAS `sizeCtl` to a negative value `(resizeStamp << 16) + 2`, allocates `nextTable` (2x), sets `transferIndex = n`.
2. Each participating thread claims a **stride** (minimum 16 bins, or n / (8 * NCPU)) by CAS-decrementing `transferIndex`, then migrates its bins from high index to low.
3. For each bin: `synchronized (head)` on the old bin, split it into a **low list** (stays at index i) and a **high list** (goes to i + n) based on the bit `hash & n`; the code reuses a trailing run of nodes that all land in the same half (`lastRun` optimisation) and clones the others; tree bins are split into two trees (or untreeified when small, <= 6). Then `setTabAt(tab, i, forwardingNode)`.
4. **Operations arriving during resize:**
   - `get` hitting a `ForwardingNode` **follows it** into `nextTable` and searches there (still lock-free).
   - `put` hitting a `ForwardingNode` calls **`helpTransfer`**: joins the resize (increments the thread count in `sizeCtl`), migrates some bins, then retries. So writers *help* rather than wait: resize gets faster with more writers.
5. When all strides are done (thread count in `sizeCtl` drops back), the last thread publishes `table = nextTable`, sets `sizeCtl = 0.75 * 2n`.
Because migration is per-bin and a bin is locked only while that bin is being copied, other bins stay fully usable.

### Why no `null` keys or values
For a plain `HashMap`, `get(k) == null` is ambiguous (absent, or mapped to null) but you can disambiguate with `containsKey(k)`. In a concurrent map `if (map.containsKey(k)) return map.get(k)` is a check-then-act: between the two calls another thread may remove the key, so the ambiguity can't be resolved safely. Doug Lea's decision: forbid `null` so `get()==null` unambiguously means "absent". (Also `putIfAbsent`/`computeIfAbsent` semantics rely on null meaning "no mapping".) Hence `NullPointerException` from `put(null, v)`, `put(k, null)`, even `get(null)`/`containsKey(null)`.

### Compound atomic operations
| Method | Atomic? | Notes |
|---|---|---|
| `putIfAbsent(k, v)`, `remove(k, v)`, `replace(k, old, new)` | yes | single-bin atomic |
| `computeIfAbsent(k, fn)` | yes: `fn` runs **at most once, while holding the bin lock** (or reservation on an empty bin) | `fn` must be short, side-effect free, **must not modify the map** (a recursive update of the same map can `IllegalStateException("Recursive update")` in Java 9+; in Java 8 it could spin forever / livelock in some cases), and must not block on I/O (blocks all writers to that bin) |
| `compute`, `computeIfPresent`, `merge(k, v, fn)` | yes | classic counter: `map.merge(word, 1, Integer::sum)` |
| `get` + `put` combination | **no** | lost update, e.g. `map.put(k, map.get(k)+1)`: use `merge` |
| `size()`, `isEmpty()` combined with other ops | no | |
| `forEach/reduce/search(parallelismThreshold, ...)` | bulk ops are weakly consistent | use a threshold of `Long.MAX_VALUE` for sequential |

```java
// WRONG: check-then-act, two threads both see null and both create
if (!cache.containsKey(k)) cache.put(k, load(k));
// RIGHT
cache.computeIfAbsent(k, this::load);        // load runs once per key
// WRONG: lost update
map.put(k, map.getOrDefault(k, 0) + 1);
// RIGHT
map.merge(k, 1, Integer::sum);              // or ConcurrentHashMap<K, LongAdder> + computeIfAbsent(k, x->new LongAdder()).increment()
```

### Iterators: weakly consistent
Iterators (and `keySet()`, `values()`, `entrySet()`) **never throw `ConcurrentModificationException`**, they traverse the table as it was when created and *may* (not must) reflect updates made afterwards. They never return the same element twice and never fail. They are not a snapshot and not fail-fast. They are safe to use while other threads mutate the map (and while resizing: they follow forwarding nodes via a `TableStack`).
Contrast: `CopyOnWriteArrayList` iterator = true snapshot (immutable array copy); `Collections.synchronizedMap` iterator must be manually locked and is fail-fast.

### CHM vs alternatives
| | Hashtable / `synchronizedMap` | `ConcurrentHashMap` |
|---|---|---|
| Lock | one lock on entire map | per bin + CAS |
| Reads | lock | lock-free |
| Null | Hashtable no; synchronizedMap yes | no |
| Iteration | fail-fast, needs external lock | weakly consistent |
| Compound ops | external lock | `compute*`, `merge` |
CHM's guarantees: `put` hb subsequent `get` of that key (rule "concurrent collections"). Its **`hashCode` quality** still matters: a bad hash yields long chains, treeification at 8 (with table >= 64) keeps worst case O(log n) for `Comparable` keys.

---

## 2.16 Concurrent queues and other concurrent collections (overview)

| Class | Bound | Technique | Typical use |
|---|---|---|---|
| `ConcurrentLinkedQueue` / `ConcurrentLinkedDeque` | unbounded | **lock-free**, CAS on head/tail (Michael-Scott queue); `size()` is O(n) and weakly consistent | many producers, polling consumers, no blocking needed |
| `ArrayBlockingQueue` | bounded | **one `ReentrantLock`**, two `Condition`s (`notEmpty`, `notFull`), circular array | fixed capacity backpressure; optional fairness |
| `LinkedBlockingQueue` | optional (default `Integer.MAX_VALUE`, dangerous: unbounded memory) | **two locks** (`putLock`, `takeLock`) + count `AtomicInteger` -> producers and consumers don't contend | default in `Executors.newFixedThreadPool` |
| `LinkedBlockingDeque` | optional | one lock | work stealing/deque semantics |
| `PriorityBlockingQueue` | unbounded | one lock, binary heap | priority scheduling (no fairness among equal priorities) |
| `DelayQueue<E extends Delayed>` | unbounded | lock + priority queue; `take()` blocks until head's delay expires | scheduled tasks, expiring caches |
| `SynchronousQueue` | 0 capacity | direct hand-off (lock-free dual stack/queue) | `newCachedThreadPool`; each `put` waits for a `take` |
| `LinkedTransferQueue` | unbounded | lock-free dual queue; `transfer()` waits for consumer | high-throughput hand-off |
| `CopyOnWriteArrayList/Set` | n/a | every mutation copies the array under a lock; readers lock-free on an immutable snapshot | rarely-changing listener lists; O(n) writes |
| `ConcurrentSkipListMap/Set` | n/a | lock-free skip list, sorted, O(log n) | concurrent sorted map (CHM is not sorted) |

Producer/consumer methods: `put/take` (block), `offer/poll` (non-blocking or timed), `add/remove` (throw). All give hb: `put` hb the matching `take`.

---

## 2.17 `ThreadLocal` internals

```
 Thread object                      ThreadLocalMap (per thread, in Thread.threadLocals)
 +-----------------+               +--------------------------------------------+
 | threadLocals ---|-------------> | Entry[] table  (open addressing, linear probing)
 +-----------------+               |   Entry extends WeakReference<ThreadLocal<?>>
                                   |     referent (key) = the ThreadLocal   (WEAK)
                                   |     value                              (STRONG)
                                   +--------------------------------------------+
 tl.get(): map = Thread.currentThread().threadLocals; e = map.getEntry(tl); return e.value (or initialValue())
```
- The map is owned by the *thread*, not the ThreadLocal. No locking is needed since only its own thread touches it. Hash = `threadLocalHashCode` (an incrementing golden-ratio constant per ThreadLocal instance).
- The **key** is weakly referenced: if your `static ThreadLocal` variable is gone (e.g. the owner class was unloaded/ThreadLocal instance became unreachable), the key becomes `null` in the entry: a **stale entry**. The **value is strongly referenced by the entry**, and the entry by the map, and the map by the thread. So the value stays reachable as long as the thread lives and the stale entry isn't cleaned up.
- **Cleanup is opportunistic:** `get/set/remove` on that thread probe and expunge stale entries encountered (`expungeStaleEntry`, `cleanSomeSlots`), but if the thread never touches any ThreadLocal again, stale values linger.
- **Leak in thread pools**: pool threads live forever. A request handler does `tl.set(hugeObject)` without `remove()`: the object stays until the same thread overwrites it (wasted memory; and in app servers, the value's classloader prevents redeploy garbage collection = **classloader leak / PermGen-Metaspace growth**). Worse, *data leaks to the next request* on that thread (security bug: user A's context seen by user B).
- Pattern: `try { ctx.set(x); handle(); } finally { ctx.remove(); }`. Use `static final ThreadLocal`, and `ThreadLocal.withInitial(...)`.
- `InheritableThreadLocal`: child thread's initial values are **copied from the parent at thread creation time** (`new Thread(...)`, not at `start`). Later changes are not propagated. With thread pools, the thread was created by whoever first submitted work, so inherited values are wrong for later tasks. Alternatives: pass context explicitly, or libraries like TransmittableThreadLocal, or Spring's `TaskDecorator` / Micrometer context propagation.
- **Virtual threads** support `ThreadLocal` but with millions of threads, per-thread caches (e.g. pooled `SimpleDateFormat`, `ByteBuffer`) get expensive.
- **Scoped Values** (`ScopedValue`, JEP 446): **preview in JDK 21** (needs `--enable-preview`). Immutable, bounded to a dynamic scope `ScopedValue.where(KEY, v).run(() -> ...)`, automatically cleaned up (no `remove()` to forget), inherited cheaply by `StructuredTaskScope` children. Intended long-term replacement for many ThreadLocal uses. Do not present it as production-final in 21.
- Common legit uses: request/user/trace context (MDC in logging), non-thread-safe helpers (`Random` before `ThreadLocalRandom`, formatters), transaction context (Spring `TransactionSynchronizationManager`).

---

## 2.18 Deadlock, livelock, starvation

### Deadlock: 4 necessary (Coffman) conditions
All four must hold simultaneously; break any one to prevent it:
1. **Mutual exclusion**: resources held exclusively.
2. **Hold and wait**: a thread holds one resource while waiting for another.
3. **No preemption**: resources can't be forcibly taken away (a `synchronized` lock cannot be revoked).
4. **Circular wait**: a cycle of threads each waiting for the next's resource.

```
   T1 holds A --wants--> B held by T2
   T2 holds B --wants--> A held by T1       (cycle)
```

| Prevention | Which condition it breaks | How |
|---|---|---|
| **Global lock ordering** | circular wait | always acquire locks in the same order (e.g. by account id, or `System.identityHashCode` with a tie-breaker lock) |
| **`tryLock` with timeout + back off / release all** | hold-and-wait / no-preemption | if the second lock cannot be obtained in time, release the first and retry (with random back-off to avoid livelock) |
| Acquire all needed locks at once (single coarse lock) | hold-and-wait | reduces concurrency |
| Avoid calling alien/overridable methods while holding a lock | circular wait through callbacks | "open calls" |
| Use lock-free / immutable / confined data | mutual exclusion | |

Lock-ordering example (transfer):
```java
void transfer(Account a, Account b, long amt) {
    Account first = a.id < b.id ? a : b, second = first == a ? b : a;   // total order by id
    synchronized (first) { synchronized (second) { a.debit(amt); b.credit(amt); } }
}
```

### Detection
- `jstack <pid>` / `jcmd <pid> Thread.print`: prints **"Found one Java-level deadlock"** for monitors and ownable synchronizers (real output in 2.8).
- Programmatically: `ThreadMXBean.findDeadlockedThreads()` (monitors **and** `java.util.concurrent` locks; `findMonitorDeadlockedThreads()` only monitors) — see demo 3.4. Can be scheduled in a health check thread to alert.
- JMX tools: VisualVM, JConsole "Detect Deadlock", JFR (`jdk.JavaMonitorEnter` events show long waits).
- **Not** detected: a thread waiting forever on a latch/condition/queue that nobody will signal ("lost wake-up hang"), or a thread pool deadlock (all pool threads wait for tasks queued behind them).

### Thread-pool starvation deadlock (extremely common)
A task running in a fixed pool submits a child task to *the same pool* and blocks on `future.get()`. When all workers are parents, no worker is free for children: deadlock that `jstack` won't flag (workers are `WAITING (parking)` in `FutureTask.get`). Fix: separate pools, non-blocking composition, or larger/unbounded pool for the blocking layer.

### Livelock
Threads are active but make no progress because they keep reacting to each other (two people in a corridor sidestepping, or two threads that `tryLock`, fail, release, retry in lockstep). Dumps show `RUNNABLE` threads with high CPU. Fix: randomised back-off, ordering, or a coordinator.

### Starvation
A thread never gets the resource: low priority vs busy CPU, nonfair lock barging, a writer behind endless readers, a long-running task hogging a pool with an unbounded queue. Fixes: fair locks (cost throughput), bounded work, separate pools, `ReentrantReadWriteLock` fair mode, timeouts.

---

## 2.19 False sharing, immutability, confinement, thread-safety levels

### False sharing
Cache coherence works on **cache lines (64 bytes)**. If two independent hot variables written by different threads share one line, each write invalidates the other core's copy, even though the logically-independent variables never conflict.
```
 line: [ counterA (thread1 writes) | counterB (thread2 writes) ....]  -> ping-pong invalidations, 5-20x slowdown
```
Mitigation: pad fields to separate lines, or use `@jdk.internal.vm.annotation.Contended` (JDK-internal: used by `Thread` fields, `Striped64.Cell`, `ConcurrentHashMap.CounterCell`; in user code it only takes effect with `-XX:-RestrictContended` and, on JDK 9+, needs the package exported for compilation; it is not a public API). Ready-made solutions: `LongAdder`, or manual padding (`long p1..p7`), or Disruptor-style padded sequences. Diagnose with `perf c2c`/hardware counters, not with jstack.

### Immutability (the best thread safety)
Immutable object: all fields `final`, no state change after construction, class `final` (or private constructor), no `this` escape, defensive copies of mutable inputs/outputs, mutable components not exposed. Safely publishable via *any* means thanks to final-field freeze. Examples: `String`, boxed types, `BigDecimal`, `LocalDate`, records (shallowly immutable: components that are mutable objects are not protected), `List.of(...)` (unmodifiable, and truly immutable when contents are).
Note: `Collections.unmodifiableList(list)` is only a *view*: the backing list can still change.

### Thread confinement
1. **Stack confinement**: local variables/objects that never escape the method.
2. **Thread-local confinement**: `ThreadLocal`.
3. **Ad-hoc confinement**: by convention (e.g. only the UI or event-loop thread touches the object; Netty's `EventLoop`, single-writer designs, actor model).
4. **Executor confinement**: `Executors.newSingleThreadExecutor()` serializes all access to one structure.

### Levels of thread safety (Bloch, *Effective Java* item 82)
1. **Immutable**: `String`, `Long`, `BigInteger`. No external sync ever.
2. **Unconditionally thread-safe**: mutable but internal sync: `AtomicLong`, `ConcurrentHashMap`, `LongAdder`.
3. **Conditionally thread-safe**: some methods need external synchronization for sequences of calls: `Collections.synchronizedList` (iteration needs `synchronized(list)`), `Hashtable` iterators.
4. **Not thread-safe**: needs full external synchronization: `ArrayList`, `HashMap`, `SimpleDateFormat`, `StringBuilder`.
5. **Thread-hostile**: unsafe even with external synchronization (e.g. static mutable global state, methods that change system-wide state).
Document which one your class is (`@ThreadSafe`, `@NotThreadSafe`, `@Immutable` from JCIP).

---

## 2.20 Classic bug catalogue

| Bug | Why | Fix |
|---|---|---|
| **Check-then-act** (`if (!map.containsKey(k)) map.put(k, v)`, lazy init `if (x == null) x = new X()`) | condition can be invalidated between check and act | one lock for both, or `putIfAbsent` / `computeIfAbsent`, or holder idiom |
| **Read-modify-write on volatile** (`volatile int c; c++`) | volatile gives visibility, not atomicity | `AtomicInteger` / `LongAdder` / lock |
| **Publishing `this` in constructor** (registering listener, `new Thread(this).start()`, storing in static) | another thread sees partially constructed object; breaks final-field guarantee | factory method: construct fully, then register; `private` ctor + `static create()` |
| **Broken DCL** (no `volatile`) | reference published before constructor writes | `volatile`, holder idiom, or enum |
| **`SimpleDateFormat` shared static** | holds a mutable `Calendar` inside; concurrent `format/parse` corrupts it (wrong dates, `NumberFormatException`, `ArrayIndexOutOfBounds`) | `DateTimeFormatter` (immutable, thread-safe), or `ThreadLocal<SimpleDateFormat>` |
| **Java 7 `HashMap` infinite loop** | concurrent `resize` transfers nodes by *head insertion, reversing bucket order*; two threads can create a cycle in a chain (`A -> B -> A`); later `get()` on that bucket loops forever at 100% CPU. Symptom: threads RUNNABLE in `HashMap.get` / `transfer`. Java 8 changed transfer to preserve order (no cycle from that path) but `HashMap` is *still* unsafe: lost updates, missing entries, stale reads, possible tree-bin corruption. | `ConcurrentHashMap` |
| **`synchronized` on a non-final / changing lock object**, or on `String` literal/boxed `Integer` (interned/shared) | different threads lock different objects, or unrelated code shares your lock | `private final Object lock = new Object();` |
| **Locking on `this` in a public class** | outsiders can lock it too (denial of service, deadlock) | private lock object |
| **Different locks for read and write of the same data** | no mutual exclusion | same lock for all accesses |
| **Iterating a synchronized collection without lock** | `ConcurrentModificationException` or missed elements | lock during iteration, or use concurrent collection |
| **Swallowing `InterruptedException`** | unstoppable tasks | see 2.13 |
| **`lock()` inside `try`** | if `lock()` throws, `finally { unlock }` unlocks a lock never held -> `IllegalMonitorStateException` masks original | `lock(); try {...} finally { unlock(); }` |
| **Forgetting `unlock` on some path** | lock leaked, everything hangs | `finally` |
| **`Thread.sleep` as synchronization** | no hb | latch/condition |
| **Calling `Thread.run()` instead of `start()`** | no new thread | `start()` |
| **Unbounded queues** in producer-consumer | OOM | bounded queue + backpressure policy |
| **Double increments via `AtomicInteger.get()+set()`** | two-step | `incrementAndGet` |
| **Non-atomic `long`/`double`** on 32-bit VMs | tearing | `volatile` / atomics |
| **Using `parallelStream()` with shared mutable state** | data race | collectors / `reduce` |

---

# 3. Runnable demos (compiled and run with `javac` / `java` 21.0.4)

Every program is a single file: `javac X.java && java X`. Outputs below are **real observed output**. Where output is nondeterministic I say so; do not expect identical numbers on your machine.

## 3.1 Visibility bug (no `volatile`)

```java
public class VisibilityBug {
    static boolean stop = false;            // try: static volatile boolean stop
    public static void main(String[] a) throws Exception {
        Thread w = new Thread(() -> {
            long n = 0;
            while (!stop) { n++; }          // JIT may hoist the read of 'stop' out of the loop
            System.out.println("worker saw stop after " + n + " iterations");
        });
        w.setDaemon(true);
        w.start();
        Thread.sleep(1000);
        stop = true;
        System.out.println("main set stop=true");
        w.join(2000);
        System.out.println(w.isAlive() ? "RESULT: worker STILL RUNNING (visibility bug)" : "RESULT: worker exited");
    }
}
```

Observed output (run once, default JIT flags):
```
main set stop=true
RESULT: worker STILL RUNNING (visibility bug)
```
Explanation: once the loop is JIT-compiled (C2), the read of `stop` can be hoisted out of the loop (conceptually `if (!stop) while (true) n++;`). Nondeterminism: with `-Xint` (interpreter only) or a short-lived loop the worker often *does* see the write and exits, and adding a `System.out.println` inside the loop often "fixes" it (incidental locking inside `PrintStream`). Declare the field `static volatile boolean stop` and the output becomes `worker saw stop after <N> iterations` / `RESULT: worker exited`. The daemon flag only lets the JVM exit despite the stuck thread.

## 3.2 Lost update: plain vs volatile vs synchronized vs atomic

```java
import java.util.concurrent.atomic.AtomicInteger;
public class LostUpdate {
    static int plain; static volatile int vol; static int sync; static final AtomicInteger atomic = new AtomicInteger();
    static final Object L = new Object();
    public static void main(String[] a) throws Exception {
        int T = 4, N = 1_000_000;
        Thread[] ts = new Thread[T];
        for (int i = 0; i < T; i++) { ts[i] = new Thread(() -> { for (int j = 0; j < N; j++) {
            plain++; vol++; synchronized (L) { sync++; } atomic.incrementAndGet(); } }); ts[i].start(); }
        for (Thread t : ts) t.join();
        System.out.println("expected  = " + (T * N));
        System.out.println("plain     = " + plain);
        System.out.println("volatile  = " + vol);
        System.out.println("synchron. = " + sync);
        System.out.println("atomic    = " + atomic.get());
    }
}
```

Observed output (4 threads x 1,000,000 increments; the two wrong numbers vary run to run, the correct ones never do):
```
expected  = 4000000
plain     = 3967589
volatile  = 3957072
synchron. = 4000000
atomic    = 4000000
```
Lesson: `volatile` does not help `++`. About 1% of updates were lost in this run (nondeterministic).

## 3.3 Double-checked locking (broken vs safe)

```java
public class DclDemo {
    static class Holder { int a, b; Holder() { a = 1; b = 2; } }
    static Holder broken;                      // no volatile
    static volatile Holder safe;               // correct DCL
    static Holder getBroken() { if (broken == null) synchronized (DclDemo.class) { if (broken == null) broken = new Holder(); } return broken; }
    static Holder getSafe()   { Holder h = safe; if (h == null) synchronized (DclDemo.class) { h = safe; if (h == null) safe = h = new Holder(); } return h; }
    public static void main(String[] x) throws Exception {
        int bad = 0, runs = 20000;
        for (int r = 0; r < runs; r++) {
            broken = null;
            Thread[] ts = new Thread[4]; int[] seen = new int[4];
            for (int i = 0; i < 4; i++) { final int k = i; ts[i] = new Thread(() -> { Holder h = getBroken(); if (h.a != 1 || h.b != 2) seen[k] = 1; }); }
            for (Thread t : ts) t.start(); for (Thread t : ts) t.join();
            for (int s : seen) bad += s;
        }
        System.out.println("broken DCL: half-constructed observations = " + bad + " / " + runs + " runs");
        System.out.println("safe DCL sample: " + getSafe().a + "," + getSafe().b);
    }
}
```

Observed output:
```
broken DCL: half-constructed observations = 0 / 20000 runs
safe DCL sample: 1,2
```
**Honest reading:** zero anomalies does **not** prove the broken version is correct. On x86-64 (TSO: only StoreLoad reordering is visible in hardware) and given how HotSpot compiles simple constructors, this bug is extremely hard to reproduce; it is the JLS that forbids relying on it, and weak-memory hardware or purpose-built tests can expose it. The demo shows the *shape* of a stress test and the correct pattern. For real evidence use OpenJDK **jcstress**.

## 3.4 Deadlock detection with `ThreadMXBean`

```java
import java.lang.management.*;
public class DeadlockDetect {
    static final Object A = new Object(), B = new Object();
    public static void main(String[] x) throws Exception {
        Thread t1 = new Thread(() -> { synchronized (A) { sleep(200); synchronized (B) { System.out.println("t1 done"); } } }, "worker-1");
        Thread t2 = new Thread(() -> { synchronized (B) { sleep(200); synchronized (A) { System.out.println("t2 done"); } } }, "worker-2");
        t1.setDaemon(true); t2.setDaemon(true); t1.start(); t2.start();
        Thread.sleep(800);
        ThreadMXBean mx = ManagementFactory.getThreadMXBean();
        long[] ids = mx.findDeadlockedThreads();
        System.out.println("deadlocked thread count = " + (ids == null ? 0 : ids.length));
        if (ids != null) for (ThreadInfo ti : mx.getThreadInfo(ids, true, true)) {
            System.out.println(ti.getThreadName() + " state=" + ti.getThreadState() + " waitingOn=" + ti.getLockName() + " ownedBy=" + ti.getLockOwnerName());
        }
    }
    static void sleep(long ms) { try { Thread.sleep(ms); } catch (InterruptedException e) { Thread.currentThread().interrupt(); } }
}
```

Observed output (object hash suffixes differ each run):
```
deadlocked thread count = 2
worker-1 state=BLOCKED waitingOn=java.lang.Object@4d405ef7 ownedBy=worker-2
worker-2 state=BLOCKED waitingOn=java.lang.Object@76fb509a ownedBy=worker-1
```
The same program with non-daemon threads, dumped with `jstack <pid>` while hung, printed the `Found one Java-level deadlock` block shown in section 2.8 (real output). The 200 ms sleep forces both first locks to be taken before either second lock is attempted, so it is deterministic; without it the program would deadlock only sometimes.

## 3.5 `LongAdder` vs `AtomicLong` under contention

```java
import java.util.concurrent.atomic.*;
public class AdderVsAtomic {
    static long run(int threads, Runnable inc, int n) throws Exception {
        Thread[] ts = new Thread[threads];
        long t0 = System.nanoTime();
        for (int i = 0; i < threads; i++) { ts[i] = new Thread(() -> { for (int j = 0; j < n; j++) inc.run(); }); ts[i].start(); }
        for (Thread t : ts) t.join();
        return (System.nanoTime() - t0) / 1_000_000;
    }
    public static void main(String[] a) throws Exception {
        int n = 5_000_000;
        for (int round = 0; round < 2; round++) {
            System.out.println("-- round " + round + (round == 0 ? " (warm-up)" : ""));
            for (int th : new int[]{1, 4, 8}) {
                AtomicLong al = new AtomicLong(); LongAdder ad = new LongAdder();
                long m1 = run(th, al::incrementAndGet, n), m2 = run(th, ad::increment, n);
                System.out.printf("threads=%d  AtomicLong=%d ms  LongAdder=%d ms  (sums %d / %d)%n", th, m1, m2, al.get(), ad.sum());
            }
        }
        System.out.println("cores=" + Runtime.getRuntime().availableProcessors());
    }
}
```

Observed output (12 cores; timings are machine and run dependent, the *shape* is stable):
```
-- round 0 (warm-up)
threads=1  AtomicLong=18 ms  LongAdder=19 ms  (sums 5000000 / 5000000)
threads=4  AtomicLong=254 ms  LongAdder=54 ms  (sums 20000000 / 20000000)
threads=8  AtomicLong=559 ms  LongAdder=21 ms  (sums 40000000 / 40000000)
-- round 1
threads=1  AtomicLong=22 ms  LongAdder=16 ms  (sums 5000000 / 5000000)
threads=4  AtomicLong=279 ms  LongAdder=37 ms  (sums 20000000 / 20000000)
threads=8  AtomicLong=613 ms  LongAdder=20 ms  (sums 40000000 / 40000000)
cores=12
```
Reading: at 1 thread both are equal (LongAdder just CASes `base`). At 4 and 8 threads AtomicLong is roughly 5-30x slower because all threads hammer one cache line; LongAdder spreads across `Cell`s. This is a naive micro-benchmark (no JMH, thread creation included, only one warm-up round), so treat the ratio as indicative; the sums are correct in both.

## 3.6 `ReentrantReadWriteLock`: downgrade works, upgrade does not

```java
import java.util.concurrent.locks.*;
public class RwDowngrade {
    public static void main(String[] x) throws Exception {
        ReentrantReadWriteLock rw = new ReentrantReadWriteLock();
        // 1. downgrade: write -> read -> release write
        rw.writeLock().lock();
        rw.readLock().lock();
        rw.writeLock().unlock();
        System.out.println("downgrade ok: readLocks=" + rw.getReadLockCount() + " writeLocked=" + rw.isWriteLocked());
        // 2. upgrade attempt: holding read, try write (would deadlock with lock())
        boolean got = rw.writeLock().tryLock();
        System.out.println("upgrade tryLock while holding read -> " + got + "  (lock() here would hang forever)");
        rw.readLock().unlock();
        // 3. now write lock is free
        System.out.println("after read released, tryLock write -> " + rw.writeLock().tryLock());
        rw.writeLock().unlock();
        // 4. reentrant write while holding write
        rw.writeLock().lock(); rw.writeLock().lock();
        System.out.println("write hold count = " + rw.getWriteHoldCount());
        rw.writeLock().unlock(); rw.writeLock().unlock();
    }
}
```

Observed output (deterministic):
```
downgrade ok: readLocks=1 writeLocked=false
upgrade tryLock while holding read -> false  (lock() here would hang forever)
after read released, tryLock write -> true
write hold count = 2
```
Calling `rw.writeLock().lock()` instead of `tryLock()` at that point would block the thread forever (self-deadlock: the writer waits for the read count to reach 0 while the same thread holds a read). I did not run that variant; expect the thread to sit in `WAITING (parking)` inside the write lock acquisition, so read dumps for that shape rather than expecting a "Found one Java-level deadlock" banner.

---

# 4. Production war stories (symptom -> evidence -> root cause -> fix)

### War story 1: Service "hangs" every few days, CPU 0%, requests time out
- **Symptom:** health check passes for a while, then all HTTP requests time out; CPU ~0%; restart fixes it.
- **Evidence:** three `jstack` dumps 10 s apart show the same picture: 200 of 200 Tomcat threads `BLOCKED (on object monitor)` at `OrderService.process` with `waiting to lock <0x...a1f0>`; one thread holds `locked <0x...a1f0>` and is itself `WAITING (parking)` in `CompletableFuture.get` -> `RestClient.call`... i.e. holding a monitor while doing a remote call that has no timeout.
- **Root cause:** `synchronized` method wrapping a network call; downstream slowdown -> monitor held for minutes -> every request queues on that monitor.
- **Fix:** shrink the critical section (copy data under the lock, call outside), set client timeouts, replace shared lock with per-key locking (`ConcurrentHashMap.compute` or striped locks). Alert on `jdk.JavaMonitorEnter` JFR events > 1 s.
- **Lesson:** never hold a lock across I/O.

### War story 2: Classic lock-order deadlock in money transfer
- **Symptom:** a handful of threads stuck; a batch job never finishes.
- **Evidence:** `jstack` prints `Found one Java-level deadlock` with `"pool-3-thread-4" waiting to lock monitor ... held by "pool-3-thread-9"` and vice-versa, both in `AccountService.transfer(Account,Account,long)`.
- **Root cause:** `synchronized(from){synchronized(to){...}}`: transfer(A,B) and transfer(B,A) run concurrently.
- **Fix:** order the locks by immutable account id (section 2.18) or use `tryLock(timeout)` with retry and jitter. Add a unit stress test with two threads doing opposite transfers in a loop.

### War story 3: Thread-pool starvation (the deadlock jstack does not flag)
- **Symptom:** throughput falls to zero after a traffic burst; no exceptions; queue length rising.
- **Evidence:** all 16 threads of `pool-1` are `WAITING (parking)` at `FutureTask.awaitDone` -> `Future.get`, launched from `ReportTask.run`; no thread executes `ChildTask`; `ThreadPoolExecutor` queue holds hundreds of `ChildTask`s. No "Found one Java-level deadlock".
- **Root cause:** parent tasks block on children submitted to the *same* fixed pool.
- **Fix:** separate pools per layer, or compose asynchronously (`CompletableFuture.thenCombine`) without blocking, or `ForkJoinPool` for recursive decomposition. Size pools by workload; expose queue-depth metrics.

### War story 4: 100% CPU on one core; threads stuck in `HashMap.get` (Java 7 era, still seen in legacy code)
- **Symptom:** after a deploy, CPU pegged, some requests never return; `top -H` shows 2-3 threads at 100%.
- **Evidence:** convert `top -H` thread ids to hex, match `nid=0x...` in `jstack`: `RUNNABLE` at `java.util.HashMap.get(HashMap.java:...)` in the same frame across all dumps; sometimes `HashMap.transfer` (resize).
- **Root cause:** static `HashMap` used as cache without synchronization; concurrent resize created a cycle in a bucket chain (Java 7 head-insertion transfer); `get` spins forever. (On Java 8+ you see lost entries or `ClassCastException` on tree bins instead.)
- **Fix:** `ConcurrentHashMap` (+ `computeIfAbsent`), or immutable snapshot swapped via `volatile`. Code-review rule: any `static` mutable collection needs an owner and a thread-safety statement.

### War story 5: Wrong dates in reports under load (`SimpleDateFormat`)
- **Symptom:** random invalid dates (`0002-...`), occasional `NumberFormatException: multiple points`, `ArrayIndexOutOfBoundsException` in `sun.util.calendar`; only under load; cannot reproduce in QA.
- **Evidence:** stack traces in `java.text.DigitList` / `SimpleDateFormat.parse`; code has `private static final SimpleDateFormat FMT`.
- **Root cause:** `SimpleDateFormat` holds an internal mutable `Calendar` and `DigitList`; concurrent use corrupts them.
- **Fix:** `DateTimeFormatter` (immutable) with `java.time`. If stuck on `SimpleDateFormat`, `ThreadLocal<SimpleDateFormat>` (with `remove()` discipline).

### War story 6: Memory climbs, user A sees user B's data (`ThreadLocal` in a pool)
- **Symptom:** rare "wrong customer" incident; old-gen grows; after redeploy Metaspace does not shrink (Tomcat "created a ThreadLocal with key of type X but failed to remove it" warning).
- **Evidence:** heap dump: `Thread.threadLocals -> ThreadLocalMap$Entry[]` on `http-nio-8080-exec-*` threads retaining large `RequestContext` objects and the old `WebappClassLoader`.
- **Root cause:** `THREAD_CTX.set(ctx)` in an interceptor; `remove()` only on the happy path; on exceptions the value persisted into the next request on that pooled thread.
- **Fix:** `try { set; proceed } finally { remove(); }` in a filter; use `ScopedValue` (preview in 21) or explicit parameters for new code; test that the context is null at the start of each request.

### War story 7: "Impossible" NPE in a lazily-initialised singleton
- **Symptom:** an NPE on `Config.get().getUrl()` in a constructor-initialised field, ~1 per million requests, only on the ARM (Graviton) fleet, never on x86 staging.
- **Evidence:** no stack anomaly; the singleton was written as DCL with `private static Config instance;` (no `volatile`), fields non-final.
- **Root cause:** unsafe publication (2.5) exposed on a weakly-ordered CPU.
- **Fix:** `volatile`, or holder idiom/enum; add jcstress test in CI. **Lesson:** "works on my machine (x86)" is not evidence.

### War story 8: Lost increments in metrics; `AtomicLong` shows as hot in a profiler
- **Symptom:** (a) dashboard totals lower than log counts; (b) after fixing with an `AtomicLong`, request latency rises 5% under 64-thread load.
- **Evidence:** (a) code was `volatile long hits; hits++`. (b) async-profiler shows `AtomicLong.incrementAndGet` / `Unsafe.getAndAddLong` loops near the top, CPU at cache-miss stalls.
- **Fix:** `LongAdder` for write-hot counters read rarely (2.14). Keep `AtomicLong` for sequence numbers.

### War story 9: `ConcurrentHashMap.computeIfAbsent` freezes a service
- **Symptom:** all threads hitting one cache stall; some threads `RUNNABLE` in `ConcurrentHashMap.computeIfAbsent`/`transfer` at 100% CPU (Java 8), or `IllegalStateException: Recursive update` (Java 9+).
- **Root cause:** the mapping function itself called `cache.computeIfAbsent(otherKey, ...)` on the same map (memoised recursion, e.g. Fibonacci), or did a slow DB call while holding the bin lock.
- **Fix:** compute outside then `putIfAbsent`; or store `Future`/`CompletableFuture` values (`computeIfAbsent(k, x -> new FutureTask(...))` then run and get outside the lock); never mutate the same map inside the function.

### War story 10: Graceful shutdown never finishes (swallowed interrupt)
- **Symptom:** `executor.shutdownNow()` then `awaitTermination` times out; pod killed with SIGKILL after 30 s.
- **Evidence:** worker threads loop in `while(true){ try{ queue.take(); }catch(InterruptedException e){ log.warn(..) } }`; jstack shows them `WAITING` in `LinkedBlockingQueue.take` again and again.
- **Fix:** restore the flag and exit (2.13); test shutdown under load.

### War story 11: Lock convoy / fair lock disaster
- **Symptom:** after "making the lock fair to prevent starvation", throughput dropped 10x with no CPU increase.
- **Evidence:** thread dumps show dozens of `WAITING (parking)` on `ReentrantLock$FairSync`; short critical sections.
- **Root cause:** fair lock forces a park/unpark handoff on every release (no barging), turning a fast lock into a context-switch chain.
- **Fix:** nonfair lock, reduce hold time, shard the lock (striping), or use `ConcurrentHashMap`/`LongAdder`.

---

# 5. Interview questions (50)

Format per question: **Q** (difficulty), **A** model answer, **Follow-ups**, **Wrong answers** (what candidates commonly say).

## Foundations

**Q1 (Easy). What are the three problems in multithreading and what solves each?**
A: Atomicity (compound actions interleave) -> locks/atomics; visibility (writes not seen) -> `volatile`/locks/final/start/join; ordering (reordering by compiler/CPU) -> `volatile`/locks/happens-before.
Follow-ups: Which does `volatile` solve? (visibility + ordering, not compound atomicity.) Which does `synchronized` solve? (all three, for the guarded state.) Give a bug for each.
Wrong: "volatile makes the variable thread-safe" (only for single reads/writes).

**Q2 (Easy). `synchronized` vs `volatile`?**
A: `synchronized` = mutual exclusion + visibility + ordering on a block; can guard compound actions and multiple variables; may block. `volatile` = visibility + ordering for one variable, no mutual exclusion, never blocks, no compound atomicity.
Follow-ups: When is `volatile` enough? (one writer or independent writes of whole values, no invariant across variables.) Can a `volatile` replace `synchronized` for a counter? (No.)

**Q3 (Easy). What does `Thread.start()` do vs `run()`? What if `start()` is called twice?**
A: `start()` creates a new OS thread (platform thread) that executes `run()`; calling `run()` directly executes on the caller's thread. Second `start()` throws `IllegalThreadStateException`.
Follow-up: hb relationship? `start()` hb everything in the new thread.

**Q4 (Easy). Why does `while (!flag) {}` sometimes never end?**
A: Without volatile/sync, JIT may hoist the read of `flag` out of the loop (allowed; single-threaded semantics unchanged), or the write may not become visible. Fix: `volatile`, or synchronization, or `Thread.onSpinWait()` *with* volatile.
Follow-ups: Would `Thread.sleep` inside the loop fix it? (Often appears to but the JMM doesn't guarantee it; no hb edge.) Why does `println` in the loop fix it? (Incidental synchronization.)

**Q5 (Medium). Explain the Java Memory Model in two minutes.**
A: A specification of which values a read may return and which reorderings are legal. Built on *happens-before*: if two conflicting accesses are hb-ordered there is no race and the read sees the write; unordered conflicting accesses form a data race with weak guarantees. If the program has no data races, it behaves sequentially consistently. hb edges: program order, monitor unlock->lock, volatile write->read, thread start/join, interrupt, final-field freeze, plus j.u.c. actions (queue put->take, executor submit, future get, latch).
Follow-ups: Is hb the same as "happens before in time"? (No.) Define data race. What is the SC-for-DRF guarantee?
Wrong: "JMM = heap/stack/metaspace layout" (that's the runtime memory areas; confusion is the #1 wrong answer).

**Q6 (Medium). Name all happens-before rules.**
A: Program order, monitor lock, volatile, thread start, thread termination (join/isAlive), interruption, default-value initialisation, finalizer, transitivity; plus final-field semantics and the j.u.c. guarantees (see 2.2 table).
Follow-up: does `Thread.sleep` create an hb? (No.) Does `ExecutorService.submit` ? (Yes: submit hb task execution; task hb `Future.get` return.)

**Q7 (Medium). What is a data race? Is it the same as a race condition?**
A: Data race: conflicting accesses unordered by hb (a memory-model notion). Race condition: outcome depends on timing (a logic notion). A program can have a race condition with no data race (all accesses synchronized, but check-then-act across two separate synchronized calls) and can have a data race without visible race condition (benign racy `String.hashCode` caching, deliberate).
Follow-up: Example of benign data race in the JDK? (`String.hash` lazily cached; works because `int` writes are atomic and the value is idempotent.)

**Q8 (Medium). What does `volatile` guarantee exactly? Is it "read from main memory"?**
A: Visibility of last write in synchronization order, no reordering of surrounding accesses across it (release on write, acquire on read), atomic single read/write for long/double. Caches are coherent so "main memory" is a simplification; the real effect is preventing register caching/hoisting and inserting fences (StoreStore+StoreLoad after write, LoadLoad+LoadStore after read).
Follow-ups: Which barrier is expensive on x86? (StoreLoad, via a `lock`-prefixed instruction.) Are volatile reads cheap? (On x86 nearly free.)
Wrong: "volatile makes `i++` thread-safe"; "volatile disables the CPU cache".

**Q9 (Medium). Why does double-checked locking need `volatile`? Walk through it.**
A: `instance = new X()` is allocate -> construct -> assign; construct/assign can be reordered, so another thread passing the unsynchronized first check may read a non-null reference to an unconstructed object. `volatile` forbids that reordering and makes the read see the constructor's writes (trace in 2.5).
Follow-ups: Would `final` fields fix it? (Freeze semantics make final fields safe but not other fields; fragile.) Better alternatives? (holder idiom, enum singleton.) Is it broken before Java 5? (Yes.)
Wrong: "volatile is needed so the second thread sees the latest value of instance" (true but doesn't explain the half-constructed object).

**Q10 (Medium). What is safe publication? Give idioms.**
A: Making a reference visible with all its state. Static initializer, volatile/atomic field, final field, lock-guarded field, concurrent collection, thread handoff (`start`, executor submit).
Follow-ups: Why are immutable objects safe to publish unsafely? (final freeze.) Does that hold if `this` escapes? (No.)

**Q11 (Hard). How do `final` fields work in the JMM?**
A: At the end of a constructor a freeze occurs: any thread that sees the reference sees final fields' values (and what is reachable through them) as of construction end, even via a data race, provided `this` did not escape. Compiler/JVM emits a StoreStore barrier before the constructor returns (conceptually). Doesn't protect non-final fields or later modifications.
Follow-up: Reflection or `Unsafe` modifying finals? (Outside the guarantees.) Why are records/`String` safely published? (fields final.)

**Q12 (Hard). Why can reordering happen at all, and how would you reproduce a reordering bug?**
A: Compiler (instruction scheduling, hoisting, register allocation), CPU (out-of-order, store buffer, invalidate queues) both optimise assuming single-thread semantics. Reproduce with a *litmus test* using jcstress (e.g. the store-buffering test `x=1;r1=y / y=1;r2=x` with outcome 0,0), preferably on ARM; ad-hoc loops rarely show it.
Follow-up: Which reordering does x86 permit? (StoreLoad.) Why isn't hand-written stress a proof? (Absence of evidence.)

## Volatile / atomics / CAS

**Q13 (Easy). What does `AtomicInteger` use internally?**
A: A `volatile int` and CAS via `Unsafe`/`VarHandle` intrinsics compiled to `lock cmpxchg` (x86); `incrementAndGet` loops on CAS (or uses an atomic add instruction where available).
Follow-up: Difference from `synchronized` counter? (No blocking, no context switch; may spin under contention.)

**Q14 (Medium). Explain CAS and its limits.**
A: Atomic "compare expected, set new". Limits: ABA, spin waste/livelock-ish behaviour under high contention (still lock-free overall), only single-variable atomicity, cache-line bouncing.
Follow-ups: How to update two fields atomically? (Immutable pair in `AtomicReference`, or lock.) Difference between `compareAndSet` and `weakCompareAndSet`? (Weak may fail spuriously, no ordering guarantees by default in older docs; `weakCompareAndSetPlain/Volatile` in VarHandle.)

**Q15 (Medium). What is the ABA problem, how to solve?**
A: Value changes A->B->A so CAS succeeds though state changed in between (lock-free stack example, 2.14). `AtomicStampedReference` (version stamp) or `AtomicMarkableReference`; or avoid node reuse; hazard pointers in non-GC languages.
Follow-up: Does GC remove ABA? (Removes memory-reuse ABA for freshly allocated nodes, not logical ABA like balance 100->50->100 or pooled nodes.)

**Q16 (Medium). `LongAdder` vs `AtomicLong`?**
A: Striped counter (`base` + `Cell[]`, @Contended cells, probe-based hashing, table grows) reduces contention; `sum()` not atomic snapshot; more memory; ideal for statistics; `AtomicLong` for exact compare-and-set semantics (IDs, limits).
Follow-ups: Why is `LongAdder` not suitable for "increment if below limit"? (No atomic read+conditional.) Where else same idea? (`ConcurrentHashMap.size` counter cells.)

**Q17 (Medium). Is `volatile int count; count++` thread-safe? What happens in 4 threads x 1M increments?**
A: No. `count++` = load, add, store; two threads can load same value; updates lost (demo 3.2 lost ~1%). Use `AtomicInteger`/`LongAdder`/lock.
Wrong: "volatile ensures atomic increments since it writes to main memory".

**Q18 (Hard). `VarHandle` vs `Unsafe` vs atomics; what are acquire/release modes for?**
A: `VarHandle` (Java 9) is the supported API for atomic/ordered field access with modes plain/opaque/acquire-release/volatile. Acquire/release gives one-way ordering cheaper than volatile (e.g. on ARM `ldar/stlr` without a full fence): used for single-producer/single-consumer queues, publish flags.
Follow-up: When would you use it as an app developer? (Almost never; prefer atomics/locks; used in libraries.)

## synchronized / monitors / wait-notify

**Q19 (Easy). What does `synchronized` lock on for instance method, static method, block?**
A: `this`; the `Class` object; the given object. Static and instance synchronized methods use different locks and therefore do not exclude each other.
Follow-up: Is it reentrant? (Yes.) What happens on exception? (Released.)

**Q20 (Medium). How is `synchronized` implemented? Explain biased/thin/fat locks.**
A: `monitorenter/monitorexit` on the object's monitor; state in the mark word. Uncontended: lightweight (stack) lock by one CAS of a pointer to a lock record; contended or `wait`: inflated `ObjectMonitor` with entry list and wait set; threads spin adaptively then park. Biased locking (JDK 6) was deprecated in JDK 15 and disabled by default, removed in JDK 18, so it's not in 21. JIT does lock elision and coarsening.
Follow-ups: What does the mark word contain? (hash, age, lock bits, or pointer to lock record/monitor.) Does inflation get undone? (Yes, async deflation.) Why was biased locking removed? (complexity + safepoint revocation cost, modern CAS cheap.)
Wrong: "synchronized is always slow/heavyweight OS mutex" (only when contended/inflated).

**Q21 (Medium). `wait()` vs `sleep()`?**
A: `wait` is `Object` method, needs monitor, releases it, woken by notify/timeout/interrupt; `sleep` is `Thread` static, no monitor needed, keeps locks. Different states: both `WAITING/TIMED_WAITING`; `wait` returns via BLOCKED re-entry.
Follow-up: Why `IllegalMonitorStateException`? (Called without owning monitor.)

**Q22 (Medium). Why must `wait()` be in a `while` loop?**
A: Spurious wakeups, stolen wakeups (someone consumed condition first), `notifyAll` waking threads whose condition isn't true; the predicate must be rechecked after each return.
Follow-up: Why must state be guarded by the same lock? (Prevents lost notification: check and wait atomic w.r.t. state change.) `notify` vs `notifyAll`? (Prefer notifyAll unless single-condition, uniform waiters.)
Wrong: "`if` is fine because notify only wakes when condition is true".

**Q23 (Medium). Does `notify()` release the lock?**
A: No. The lock is released when the notifier leaves the synchronized block; the woken thread then competes for the monitor (state BLOCKED until then).
Follow-up: Which thread does `notify` wake? (Unspecified; HotSpot typically FIFO-ish but don't rely.)

**Q24 (Medium). Draw thread state transitions. Which state is a thread waiting on `ReentrantLock.lock()` in?**
A: See diagram 2.8. `WAITING` (parking), *not* BLOCKED; BLOCKED only for monitor entry. Thread in blocking socket read is RUNNABLE.
Follow-up: Difference `BLOCKED` vs `WAITING`? (BLOCKED: waiting to acquire monitor; WAITING: waiting for another thread's action: notify/unpark/join.) Wrong: "BLOCKED means waiting for I/O".

**Q25 (Medium). Can `synchronized` be interrupted? Can you time out on it?**
A: No to both; entering a monitor is not interruptible and has no timeout. Use `ReentrantLock.lockInterruptibly()` / `tryLock(timeout)`.

**Q26 (Hard). Explain lock elision and coarsening. Does this affect benchmarks?**
A: Elision: escape analysis proves the lock object thread-local, drop it. Coarsening: merge adjacent lock regions on the same object. Effect: naive micro-benchmarks of uncontended synchronized measure nothing; use JMH with escaping objects.
Follow-up: Why is `StringBuffer` "synchronized" not always slow? (elision on non-escaping local buffers.)

**Q27 (Hard). A virtual thread blocks inside `synchronized` in JDK 21. What happens?**
A: It pins its carrier thread (can't unmount while holding a monitor and blocking), potentially exhausting carriers; use `ReentrantLock` on such paths; addressed in later JDK (24). Diagnose with `-Djdk.tracePinnedThreads`.

## java.util.concurrent locks

**Q28 (Medium). `synchronized` vs `ReentrantLock`; when choose the latter?**
A: Table 2.6: tryLock, timeout, interruptible, fairness, multiple conditions, non-block-structured locking, no carrier pinning in JDK 21. Otherwise `synchronized` is simpler and safe against forgotten unlock.
Follow-up: Correct idiom? (lock before try; unlock in finally.) What if you `unlock` without holding? (`IllegalMonitorStateException`.) 

**Q29 (Hard). How does AQS work?**
A: `volatile int state` + CLH-style FIFO queue of waiting threads + template methods (`tryAcquire/tryRelease/tryAcquireShared/tryReleaseShared`). `acquire`: `tryAcquire` (CAS on state); on failure enqueue node (CAS tail), then loop: if predecessor is head try again, else set predecessor's status to SIGNAL and `park`. `release`: `tryRelease`, then unpark successor. Shared mode propagates wakeups through consecutive shared nodes. Conditions have their own queues.
Follow-ups: What does `state` mean in ReentrantLock/Semaphore/CountDownLatch/RRWL? What does CLH stand for/why a queue? (Fairness and scalable spinning/parking on predecessor's flag rather than a shared one.) How does cancellation work?

**Q30 (Hard). ReentrantLock fair vs nonfair implementation difference?**
A: Both `CAS(0,1)`; fair adds `!hasQueuedPredecessors()`. Nonfair allows barging on both first attempt and after wakeup, increasing throughput (avoids context-switch handoff) but risks starvation. `tryLock()` always barges.
Follow-up: When is fair good? (Strict ordering, long hold times, starvation observed.) What's the throughput cost? (Often order-of-magnitude under contention; war story 11.)

**Q31 (Medium). How does reentrancy work in `ReentrantLock`?**
A: `state` holds the hold count and `exclusiveOwnerThread` the owner; if the owner acquires again, increment `state` (no CAS needed); `unlock` decrements; when it hits 0 the owner is cleared and a successor is unparked. Overflow throws `Error("Maximum lock count exceeded")`.

**Q32 (Medium). How do `Condition.await/signal` work; how differ from `wait/notify`?**
A: Each Condition has a queue; `await` adds a node, fully releases the lock (saving holds), parks; `signal` moves a node to the AQS sync queue; it then reacquires the lock with the saved count. Multiple conditions per lock (targeted wake-up); must hold the lock; spurious wakeups possible. Implementation of `ArrayBlockingQueue` (`notFull`, `notEmpty`).

**Q33 (Medium). ReadWriteLock: rules? Can you upgrade? downgrade?**
A: Many readers or one writer; writer excludes; reentrant; state split high 16/low 16 bits; downgrade (write -> read -> release write) OK; upgrade (read -> write) self-deadlocks; fair vs nonfair; read lock has no conditions; writer starvation possible in nonfair mode though readers yield to a queued writer at the head.
Follow-up: When is RRWL slower than a plain lock? (short critical sections, write-heavy.) Why can you downgrade but not upgrade? (Two readers each trying to upgrade would deadlock each other; upgrading isn't atomic w.r.t. other writers.)

**Q34 (Hard). What's `StampedLock` optimistic reading and its pitfalls?**
A: `tryOptimisticRead` gives a stamp with no lock; read fields into locals; `validate(stamp)`; on failure fall back to `readLock`. Pitfalls: non-reentrant, no conditions, stamp not tied to thread, must not act on unvalidated data, easy to misuse; great for tiny read-mostly structures.
Follow-up: Can validation succeed yet data be stale? (Validate ensures no write since the stamp; combined with reads before it, the snapshot is consistent.)

**Q35 (Medium). CountDownLatch vs CyclicBarrier vs Semaphore vs Phaser?**
A: Latch: one-shot, waiters wait for count events (AQS shared). Barrier: N parties wait for each other, reusable, optional barrier action, breaks on interruption/timeouts. Semaphore: N permits limiting concurrency, permits not owned. Phaser: dynamic parties + multiple phases.
Follow-up: What if a worker throws before `countDown`? (Waiters hang: use `finally`.) Can a latch be reset? (No.)

**Q36 (Medium). `LockSupport.park/unpark` vs `wait/notify`?**
A: park/unpark works on threads with a binary permit; no monitor; unpark-before-park not lost; may return spuriously, and upon interrupt without throwing. Basis of AQS; use in loops.

**Q37 (Medium). How should you handle `InterruptedException`?**
A: Never swallow: propagate, or restore the flag (`Thread.currentThread().interrupt()`) and exit/clean up. Blocking methods clear the flag when they throw. `interrupted()` clears, `isInterrupted()` doesn't. Cooperative cancellation; `shutdownNow`/`cancel(true)` rely on it.
Wrong: "just log it" / "ignore, it never happens".

## ConcurrentHashMap and collections

**Q38 (Medium). How does `ConcurrentHashMap` achieve thread safety in Java 8 vs 7?**
A: 7: `Segment[]` (16 default) each a `ReentrantLock` over a small table. 8: single table, empty bin insert by CAS, non-empty bin locked via `synchronized` on the head node; volatile `val/next` for lock-free reads; treeification; cooperative resize; `baseCount + CounterCell[]` for size.
Follow-ups: What is locked during `put`? (One bin.) Are reads locked? (No.) Why synchronized instead of ReentrantLock in Java 8? (No per-node lock object overhead; JVM optimises synchronized; memory.)

**Q39 (Hard). Walk through `putVal`.**
A: null check; spread hash; loop: initTable if empty (CAS `sizeCtl`); if bin empty -> CAS new node; if head hash is MOVED -> `helpTransfer`; else `synchronized(head)`: recheck head, list or tree insert; treeify >= 8 (if table >= 64); `addCount` updates size and may resize. (2.15.)
Follow-ups: What if CAS fails? (loop and retry into synchronized path.) What is `sizeCtl`? What does `MOVED` mean?

**Q40 (Hard). How does CHM resize concurrently?**
A: `transfer` with `nextTable` 2x; threads claim strides (min 16 bins) via `transferIndex`; each bin locked while split into low/high lists; replaced by `ForwardingNode`; `get` follows forwarding nodes to `nextTable`; `put` on forwarded bin calls `helpTransfer`; last thread swaps `table`, sets `sizeCtl` = 0.75*new capacity.
Follow-up: Does resizing block readers? (No.) Why split by `hash & n`? (Bit determining new index i or i+n.)

**Q41 (Medium). Why doesn't `ConcurrentHashMap` allow null keys/values?**
A: `get==null` must unambiguously mean absent; `containsKey`+`get` is racy so ambiguity can't be resolved.
Wrong: "because it uses null to mark empty bucket" (secondary at best).

**Q42 (Medium). Is `if (!map.containsKey(k)) map.put(k,v)` safe on CHM? How to fix?**
A: No (check-then-act). `putIfAbsent`/`computeIfAbsent`/`merge`. `computeIfAbsent` function runs at most once, under the bin lock; must be short and not touch the map.
Follow-up: What is the danger of a long mapping function? (blocks that bin's writers; recursive update issue.)

**Q43 (Medium). Is `ConcurrentHashMap.size()` accurate? Iterator behaviour?**
A: Estimate under concurrency (`baseCount` + cells); `mappingCount()` long. Iterators are weakly consistent: no CME, may or may not reflect concurrent updates, no duplicates.
Follow-up: What does `CopyOnWriteArrayList`'s iterator do? (snapshot.)

**Q44 (Medium). `ConcurrentLinkedQueue` vs `LinkedBlockingQueue` vs `ArrayBlockingQueue`?**
A: CLQ: lock-free, unbounded, non-blocking, O(n) size. LBQ: two locks, optionally bounded, blocking. ABQ: single lock, bounded, array, optional fairness. Use bounded blocking queues for backpressure; default unbounded LBQ risks OOM.

## ThreadLocal, deadlock, misc

**Q45 (Medium). How does `ThreadLocal` work and why can it leak?**
A: Each `Thread` has a `ThreadLocalMap` (open addressing) with entries `WeakReference<ThreadLocal>` -> strong value. Stale-key entries stay until probing cleans them; in thread pools threads never die, so values (and classloaders) stay; also stale context leaks to the next task. Always `remove()` in `finally`.
Follow-ups: Why weak key but strong value? (Key weakness lets ThreadLocal be collected; value can't be weak or it might vanish while ThreadLocal is alive.) `InheritableThreadLocal` semantics? (copy at thread creation) Scoped values? (preview in 21.)

**Q46 (Medium). Explain deadlock: 4 conditions, prevention, detection.**
A: Mutual exclusion, hold-and-wait, no preemption, circular wait. Prevent via global lock ordering, `tryLock` with timeout/back-off, coarse lock, avoid alien calls under lock. Detect: `jstack`/`jcmd Thread.print`, `ThreadMXBean.findDeadlockedThreads`, JMX/VisualVM. Not detected: lost signals and pool starvation.
Follow-ups: Deadlock vs livelock vs starvation? Write a deadlock in 10 lines (demo 3.4). How do you fix `transfer(a,b)`? (order by id.)

## Extra scenario questions (rapid-fire, graded)

**Q47 (Hard). Design a thread-safe lazy cache with per-key loading and no duplicate loads.**
A: `ConcurrentHashMap<K, CompletableFuture<V>>` (or `FutureTask`): `computeIfAbsent(k, kk -> new CompletableFuture<>())` returns the future; the thread that created it computes *outside* the map lock and completes it; others `join()`. On failure remove the entry so retries are possible. Bound size/expiry by Caffeine in real life.
Follow-up: Why not do the DB call inside `computeIfAbsent`? (holds the bin lock; blocks other keys in that bin; recursion hazards.)

**Q48 (Hard). Two threads, `x=0,y=0`: T1 `x=1; r1=y`, T2 `y=1; r2=x`. Which outcomes are possible in Java with plain fields? With volatile?**
A: Plain: (0,0), (0,1), (1,0), (1,1) all possible; (0,0) via store buffering/reordering. Volatile: (0,0) impossible (volatile accesses are sequentially consistent among themselves).

**Q49 (Hard). Why `Collections.synchronizedList` iteration still needs a lock?**
A: Each method is atomic, but iteration is many calls; another thread can modify between them -> CME or inconsistent view; must `synchronized(list){ for... }`. It is "conditionally thread-safe".

**Q50 (Medium). How to diagnose a hung production app?**
A: 3 thread dumps ~10 s apart (`jcmd <pid> Thread.print`), look for `Found one Java-level deadlock`; group threads by top frame; find `locked <addr>` owner of `waiting to lock <addr>`; `parking to wait for` types; check pool queues; `top -H` -> hex nid for CPU-hot threads; JFR for lock contention; heap dump for ThreadLocal/queue growth.

## Common wrong answers cheat-list (say the opposite)
| Wrong statement | Truth |
|---|---|
| "volatile reads/writes go to main memory bypassing cache" | Caches are coherent; volatile prevents reordering/register caching and adds fences |
| "volatile makes count++ atomic" | No |
| "synchronized is always heavyweight/slow" | Thin lock is one CAS; uncontended is cheap; JIT elides/coarsens |
| "Biased locking is default in modern Java" | Deprecated 15, disabled; removed 18 |
| "BLOCKED means waiting for lock (any lock)" | Only monitor entry; j.u.c. locks show WAITING (parking) |
| "ConcurrentHashMap locks the whole map on put / uses segments" | Segments removed in 8; CAS + per-bin synchronized |
| "ConcurrentHashMap.size() is exact" | Estimate under concurrency |
| "Iterator of CHM throws CME" | Weakly consistent, never CME |
| "`notify` releases the lock" | It doesn't |
| "if instead of while for wait is fine" | Spurious/stolen wakeups |
| "Thread.sleep guarantees visibility" | No hb edge |
| "Fair lock = faster/better" | Fair is slower; only prevents starvation |
| "ReadWriteLock can be upgraded" | Self-deadlock |
| "ThreadLocal has no leak as key is weak" | Value strong; pooled threads |
| "JMM = heap/stack layout" | JMM = visibility/ordering rules |
| "Final fields are always safe" | Only if `this` doesn't escape |

---

# 6. One-page cheat sheet

```
PROBLEMS      atomicity | visibility | ordering
HB EDGES      program order | unlock->lock | volatile write->read | start() | join()/isAlive | interrupt
              | transitivity | final freeze | executor submit | Future.get | queue put->take | latch countDown->await
DATA RACE     conflicting accesses (>=1 write) NOT ordered by hb.  DRF program => sequentially consistent
SAFE PUBLISH  static init | volatile/atomic | final | lock | concurrent collection | start()/submit
VOLATILE      visibility + ordering + atomic single read/write.  NOT count++ / check-then-act / two-var invariants
              write: StoreStore before, StoreLoad after.  read: LoadLoad+LoadStore after
DCL           volatile instance; local var copy; or holder idiom / enum.  reorder: alloc, publish, construct
SYNCHRONIZED  monitorenter/exit; reentrant; entry set (BLOCKED) / wait set (WAITING)
              mark word: 01 unlocked | 00 thin (CAS lock record) | 10 inflated ObjectMonitor
              biased: deprecated 15, disabled, removed 18. JIT: elision, coarsening, adaptive spin
              JDK21: pins virtual thread carrier
WAIT/NOTIFY   hold monitor; while(!cond) wait(); notifyAll(); spurious wakeups; notify doesn't release lock
STATES        NEW RUNNABLE BLOCKED(monitor entry only) WAITING(wait/join/park) TIMED_WAITING TERMINATED
              I/O wait shows RUNNABLE.  j.u.c lock wait = WAITING (parking)
AQS           volatile int state + CLH FIFO queue + park/unpark; tryAcquire/tryRelease (excl), *Shared (shared)
              RL state=holds | Semaphore=permits | Latch=count | RRWL hi16 read/lo16 write
REENTRANTLOCK nonfair: CAS(0,1) barge | fair: !hasQueuedPredecessors() && CAS | lock(); try{}finally{unlock()}
CONDITION     own queue; await = release fully + park; signal = move to sync queue; while-loop
RRWL          readers OR writer; downgrade OK; upgrade deadlocks; no Condition on read lock
STAMPED       tryOptimisticRead -> read to locals -> validate -> fallback readLock; NOT reentrant
SYNCHRONIZERS Latch one-shot | Barrier reusable (Lock+Condition) | Semaphore permits | Phaser dynamic
PARK          permit 0/1, spurious return, unpark-before-park OK
INTERRUPT     flag; blocking calls clear it + throw; interrupted() clears, isInterrupted() no; never swallow
CAS           expected/new; ABA -> AtomicStampedReference; refs compared by ==
LONGADDER     base + @Contended Cell[]; sum() not atomic snapshot; for stats not for decisions
CHM (8+)      empty bin: CAS | else synchronized(head) | get lock-free (volatile val/next)
              sizeCtl: -1 init, <-1 resizing, >0 threshold | MOVED=-1 ForwardingNode -> helpTransfer
              size = baseCount + CounterCells | no nulls (get==null unambiguous) | weakly-consistent iterators
              compute/merge atomic, function short + must not touch map | treeify at 8 (table>=64)
QUEUES        CLQ lock-free | ABQ 1 lock bounded | LBQ 2 locks | SynchronousQueue handoff | CoW snapshot
THREADLOCAL   Thread.threadLocals -> Entry(weak key, strong value); remove() in finally; ITL copies at creation
              ScopedValue = preview in 21
DEADLOCK      mutual excl + hold&wait + no preemption + circular wait | order locks, tryLock, jstack, ThreadMXBean
BUGS          check-then-act | volatile++ | this escape | broken DCL | SimpleDateFormat | Java7 HashMap loop
              | lock in try | swallowed interrupt | sync on String/Integer | pool starvation deadlock
JSTACK        3 dumps 10s apart; waiting to lock <a> vs locked <a>; parking to wait for <type>; nid hex = top -H
```

Decision guide:
- Single flag/reference, one writer -> `volatile`.
- Counter -> `AtomicLong` (exact) / `LongAdder` (hot metrics).
- Compound invariant -> `synchronized` (or `ReentrantLock` if you need try/timeout/interrupt/conditions).
- Read-mostly with longer reads -> `ReentrantReadWriteLock`; tiny reads -> `StampedLock` optimistic.
- Shared map -> `ConcurrentHashMap` + `merge/compute*`.
- Hand-off between threads -> bounded `BlockingQueue`.
- Prefer immutability and confinement over locking whenever possible.
