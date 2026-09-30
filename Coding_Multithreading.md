# Coding Problems: Multithreading

## E1. Print odd/even alternately with two threads ⭐⭐⭐
**Flow:**
```
Thread-Odd:  wait while (number is even) → print → number++ → notify
Thread-Even: wait while (number is odd)  → print → number++ → notify
```
```java
public class OddEven {
    private int n = 1; private final int max = 10;

    synchronized void print(boolean odd) throws InterruptedException {
        while (n <= max) {
            while ((n % 2 == 1) != odd) {
                wait();
                if (n > max) return;
            }
            if (n > max) break;
            System.out.println(Thread.currentThread().getName() + " " + n++);
            notifyAll();
        }
    }
    public static void main(String[] a) {
        OddEven o = new OddEven();
        new Thread(() -> { try { o.print(true);  } catch (InterruptedException e) {} }, "odd").start();
        new Thread(() -> { try { o.print(false); } catch (InterruptedException e) {} }, "even").start();
    }
}
```
Alternative: two `Semaphore`s, ping-pong.

## E2. Producer–Consumer with BlockingQueue ⭐⭐⭐
```java
public class ProducerConsumer {
    public static void main(String[] args) {
        BlockingQueue<Integer> q = new ArrayBlockingQueue<>(5);      // bounded

        Thread producer = new Thread(() -> {
            try { for (int i = 1; i <= 10; i++) { q.put(i); System.out.println("Produced " + i); } q.put(-1); }
            catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        });
        Thread consumer = new Thread(() -> {
            try {
                while (true) {
                    int v = q.take();
                    if (v == -1) break;                   // poison pill = stop signal
                    System.out.println("Consumed " + v);
                }
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        });
        producer.start(); consumer.start();
    }
}
```
`put` blocks when full, `take` blocks when empty – no manual wait/notify. Classic from-scratch version uses `wait()/notify()` with `while(queue.size()==cap) wait();`.

## E3. Deadlock example + fix
```java
Object A = new Object(), B = new Object();
// Deadlock: t1 locks A then B, t2 locks B then A
new Thread(() -> { synchronized (A) { sleep(50); synchronized (B) { } } }).start();
new Thread(() -> { synchronized (B) { sleep(50); synchronized (A) { } } }).start();
// FIX: both threads lock in the SAME order (A then B), or use tryLock with timeout.
```

## E4. Thread-safe Singleton (double-checked locking)
See `Java_Memory_Model_and_Locks.md` (double-checked locking needs `volatile`). Best: enum singleton, or initialization-on-demand holder:
```java
class Holder { private Holder(){}
    private static class H { static final Holder I = new Holder(); }   // class-load is thread-safe & lazy
    static Holder get() { return H.I; } }
```

## E5. Thread-safe counter: 3 ways
```java
synchronized void inc() { count++; }              // 1. synchronized
AtomicInteger c = new AtomicInteger(); c.incrementAndGet();   // 2. CAS
LongAdder adder = new LongAdder(); adder.increment();         // 3. best under heavy contention
```
Demo of the bug: 2 threads × 1000 × `count++` on plain int → result < 2000.

## E6. Run tasks in parallel and combine (CompletableFuture) ⭐⭐
```java
CompletableFuture<String> user   = CompletableFuture.supplyAsync(() -> getUser());
CompletableFuture<String> orders = CompletableFuture.supplyAsync(() -> getOrders());
String result = user.thenCombine(orders, (u, o) -> u + " / " + o)
                    .exceptionally(ex -> "fallback")
                    .get(2, TimeUnit.SECONDS);

CompletableFuture.allOf(f1, f2, f3).join();       // wait for all
CompletableFuture.anyOf(f1, f2).join();           // first to finish
```
`thenApply` (transform), `thenCompose` (chain another future = flatMap), `thenCombine` (merge two independent).

## E7. Print A B C in order with 3 threads
Use a shared `turn` variable + `wait/notifyAll`, or 3 `Semaphore(0/1)` in a ring:
```java
Semaphore sa = new Semaphore(1), sb = new Semaphore(0), sc = new Semaphore(0);
// thread A: sa.acquire(); print("A"); sb.release();
// thread B: sb.acquire(); print("B"); sc.release();
// thread C: sc.acquire(); print("C"); sa.release();   (loop N times)
```

## E8. Custom thread pool (simple) – shows understanding
```java
class SimplePool {
    private final BlockingQueue<Runnable> queue = new LinkedBlockingQueue<>();
    private final List<Thread> workers = new ArrayList<>();
    private volatile boolean stopped;

    SimplePool(int n) {
        for (int i = 0; i < n; i++) {
            Thread t = new Thread(() -> {
                while (!stopped || !queue.isEmpty()) {
                    try { Runnable r = queue.poll(100, TimeUnit.MILLISECONDS); if (r != null) r.run(); }
                    catch (InterruptedException e) { return; }
                }
            });
            workers.add(t); t.start();
        }
    }
    void submit(Runnable r) { if (stopped) throw new IllegalStateException(); queue.offer(r); }
    void shutdown() { stopped = true; }
}
```

## E9. Rate limiter (token bucket, single JVM)
```java
class TokenBucket {
    private final long capacity, refillPerSec; private double tokens; private long last = System.nanoTime();
    TokenBucket(long capacity, long refillPerSec) { this.capacity = capacity; this.refillPerSec = refillPerSec; this.tokens = capacity; }
    synchronized boolean tryAcquire() {
        long now = System.nanoTime();
        tokens = Math.min(capacity, tokens + (now - last) / 1e9 * refillPerSec);
        last = now;
        if (tokens >= 1) { tokens -= 1; return true; }
        return false;
    }
}
```

## E10. Quick output-prediction questions
```java
Thread t = new Thread(() -> System.out.println("run"));
t.run();     // prints "run" on MAIN thread (no new thread)
t.start();   // new thread
t.start();   // IllegalThreadStateException – a thread can't be restarted
```
`ExecutorService.submit()` swallows exceptions inside the Future (call `get()` to see `ExecutionException`); `execute()` prints to uncaught handler.
