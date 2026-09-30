# Deep JVM and GC Internals (HotSpot, JDK 17/21)

> Audience: 5-year Java developer preparing for senior/lead interviews and for production incidents.
> Builds on (does not repeat) the JVM Memory / Garbage Collection / Strings sections of `JavaFullstack.md` and the OOM/high-CPU flow in `Production_Debugging.md`.
> All demos below were compiled and run on **Oracle JDK 21.0.4, Windows 11, 12 CPUs, 15 GB RAM**. Every number labelled "observed" is machine-specific; the shape of the output is what matters, not the digits.
> Flags: every flag in this note was checked against `java -XX:+PrintFlagsFinal` on JDK 21 (exceptions are called out: `UseContainerSupport` is Linux-only so it does not show on Windows; Shenandoah is not shipped in Oracle JDK builds, only in OpenJDK builds such as Temurin/Red Hat).

## Table of contents
1. 60-second mental model + analogy
2. Deep internals
   2.1 JVM architecture and runtime data areas
   2.2 Class loading in depth
   2.3 Object layout and compressed oops
   2.4 Stack frames, StackOverflowError, -Xss
   2.5 Escape analysis, scalar replacement, TLAB
   2.6 String pool, dedup, compact strings
   2.7 JIT: interpreter, C1, C2, inlining, deopt, code cache
   2.8 GC fundamentals (roots, algorithms, barriers, safepoints)
   2.9 The collectors: Serial, Parallel, G1, ZGC, Shenandoah
   2.10 Choosing a collector, containers, ergonomics
   2.11 Reading GC logs
   2.12 Heap sizing and tuning methodology
   2.13 Memory-leak patterns, diagnosis toolbox, OOM variants
   2.14 Native memory, NMT, Kubernetes OOMKilled
   2.15 References, Cleaner, finalization
   2.16 Startup: CDS, AppCDS, native image
   2.17 High CPU flow and thread-dump reading
3. Runnable demos with real output
4. Production war stories
5. Interview questions (50) with follow-ups and wrong answers
6. One-page cheat sheet

---

# 1. 60-second mental model

**The JVM is a small operating system for your bytecode.** It has:

| JVM part | OS analogy | One-liner |
|---|---|---|
| Class loader subsystem | Program loader / dynamic linker | Finds `.class` bytes, verifies them, prepares statics, runs `<clinit>` lazily |
| Heap | Shared warehouse | Every object lives here; the GC is the warehouse cleaner |
| Thread stacks | Each worker's desk | Frames with locals and operand stack; auto-cleaned on return |
| Metaspace | The catalogue of blueprints | Class metadata in **native** memory |
| JIT + code cache | Photocopier that learns which pages you re-read | Hot bytecode becomes machine code, stored in the code cache |
| GC | Janitor | Finds what is unreachable from roots and reclaims/compacts |

**Analogy: a restaurant.** Orders (requests) create plates (objects). Most plates are used once and are dirty in seconds (young generation, "most objects die young"). A few plates are the fine china that lives for years (old generation). The dishwasher (GC) can pause the kitchen (stop-the-world) or work beside the cooks (concurrent). Serial = one dishwasher who closes the kitchen. Parallel = many dishwashers, kitchen closed, done fast. G1 = the kitchen divided into small stations, clean the dirtiest station first within a time budget. ZGC = dishwashers work while cooks cook, the cooks are told "if you grab a plate that was moved, fix your pointer" (load barrier).

**Five sentences you should be able to say in an interview**

1. The heap holds objects, the stack holds frames, Metaspace holds class metadata; only the heap is GC-managed in the classical sense (Metaspace is freed when class loaders die).
2. An object is garbage when it is **unreachable from GC roots** (thread stacks, statics, JNI, etc.), not when it is "not used".
3. Generational GC works because of the weak generational hypothesis; copying young objects is cheap when most are dead.
4. Java OOM and Kubernetes OOMKilled are different things: the first is the JVM refusing an allocation, the second is the kernel killing the process because **RSS (heap + metaspace + threads + code cache + direct + native) exceeded the cgroup limit**.
5. Tune by measuring: pause time (p99), throughput %, allocation rate, promotion rate. Change one thing at a time.

**Decision cheat in one glance**

```
Tiny container (<2 CPU or <~1.8 GB)      -> Serial (JVM picks it automatically)
Batch/ETL, throughput matters, pauses OK -> Parallel
General web services, heap 1-30 GB       -> G1 (default) with defaults first
Latency-critical, big heap, p99 < 10 ms  -> ZGC (generational on 21: -XX:+ZGenerational)
```

---

# 2. Deep internals

## 2.1 JVM architecture and runtime data areas

```
                         +---------------------------------------------------------+
   .class / .jar  --->   |  CLASS LOADER SUBSYSTEM  (load -> link -> initialize)   |
                         +----------------------------+----------------------------+
                                                      |
   +--------------------------------------------------v--------------------------------------------------+
   |                                   RUNTIME DATA AREAS                                                 |
   |                                                                                                      |
   |  SHARED by all threads                          PER THREAD                                           |
   |  +-----------------------------------+          +----------------------------------------------+     |
   |  | HEAP (GC managed)                 |          | Java stack: [frame][frame][frame]...         |     |
   |  |  young: Eden | S0 | S1            |          |   each frame: locals[] + operand stack       |     |
   |  |  old / tenured                    |          |            + frame data (constant pool ref)  |     |
   |  |  (G1/ZGC: region based)           |          | PC register: address of current bytecode     |     |
   |  +-----------------------------------+          | Native method stack (JNI frames)             |     |
   |  +-----------------------------------+          | TLAB (thread-local allocation buffer, in heap)|    |
   |  | METASPACE (native memory)         |          +----------------------------------------------+     |
   |  |  class metadata, method bytecode, |                                                               |
   |  |  constant pools, vtables          |          OUTSIDE the Java heap but inside the process:        |
   |  |  + Compressed Class Space         |          - Code cache (JIT machine code)                      |
   |  +-----------------------------------+          - Direct ByteBuffers / Unsafe.allocateMemory         |
   |                                                 - Thread stacks (OS memory), GC data structures       |
   |                                                 - Symbol tables, NMT overhead, malloc arenas          |
   +------------------------------------------------------------------------------------------------------+
                          EXECUTION ENGINE:  Interpreter  ->  C1 (client)  ->  C2 (server)   +   GC
```

Area-by-area facts that interviewers probe:

| Area | Holds | Sizing flag | Failure |
|---|---|---|---|
| Heap | Objects, arrays, (String pool entries are heap objects) | `-Xms -Xmx`, `-XX:MaxRAMPercentage` | `OutOfMemoryError: Java heap space` |
| Metaspace | Class metadata (not the `Class` mirror object, which is in heap) | `-XX:MetaspaceSize` (first GC trigger threshold, NOT a reservation), `-XX:MaxMetaspaceSize` (default unlimited) | `OutOfMemoryError: Metaspace` |
| Compressed class space | Klass structures when compressed class pointers are on; **1 GB reserved by default** (seen in the real log below: `reserved size: 1073741824`) | `-XX:CompressedClassSpaceSize` | `OutOfMemoryError: Compressed class space` |
| Thread stack | Frames | `-Xss` / `-XX:ThreadStackSize` (Linux x64 default 1 MB) | `StackOverflowError`; too many threads gives `unable to create native thread` |
| PC register | Next bytecode index (undefined for native methods) | none | none |
| Native method stack | C frames for JNI | OS | crash |
| Code cache | JIT-compiled code, adapters, interpreter stubs | `-XX:ReservedCodeCacheSize` (default 240 MB with tiered) | `CodeCache is full. Compiler has been disabled.` warning (no exception, but performance falls off a cliff) |
| Direct memory | `ByteBuffer.allocateDirect` | `-XX:MaxDirectMemorySize` (defaults to max heap) | `OutOfMemoryError: Direct buffer memory` |

Points people get wrong:
- **PermGen was removed in JDK 8**, not "replaced by heap". Metaspace lives in native memory. The **String pool was moved to the heap in JDK 7**, and static fields/`Class` mirrors also moved to the heap.
- `-XX:MetaspaceSize=22M` (observed default `22020096`) does **not** reserve anything; it is the high-water mark that triggers the first metadata-driven GC. `-XX:MaxMetaspaceSize` is the cap (unlimited by default: a leak grows until the OS/container kills you unless you set it).
- Each JVM thread stack is **virtual** memory reserved up front and committed as touched. 500 threads x 1 MB = 500 MB virtual, but RSS grows only with actual depth.
- JDK 21 virtual threads keep their stack as heap objects (stack chunks) when unmounted, so they add to heap pressure, not native stack memory.

## 2.2 Class loading in depth

### Three phases

```
LOADING        find bytes (file/jar/network), define Class, create Klass in Metaspace
   |
LINKING
   |-- Verification   bytecode verifier: type safety, stack map frames, no illegal jumps
   |-- Preparation    allocate static fields, set DEFAULT values (0/null; not your initializers)
   |-- Resolution     symbolic refs in constant pool -> direct refs (may be lazy)
   |
INITIALIZATION run <clinit>: static initializers and static blocks, in textual order,
               parent class first; once per class per loader, guarded by an init lock
```

**Initialization is lazy**. Triggers: `new`, invoking a static method, reading/writing a static field that is **not** a compile-time constant (`static final int X = 5` is inlined and does not trigger), `Class.forName(name)` (initializes; `ClassLoader.loadClass` does not), initializing a subclass (superclass first), and the main class. Accessing a static field via a subclass reference only initializes the class that declares it.

Static init order for one class: static fields and static blocks in **source order**; then, per instance: instance initializer blocks and field initializers in source order, then the constructor body (after `super(...)`).

### Class loader hierarchy (JDK 9+)

```
Bootstrap loader (native, null in Java)  : java.base and core modules
   ^
Platform loader (was "Extension" before 9): java.sql, java.xml.* etc.
   ^
Application (system) loader              : classpath / module path
   ^
Custom loaders                           : Tomcat WebappClassLoader, OSGi, Spring Boot LaunchedClassLoader, plugins
```

### Parent delegation (parent-first)

`loadClass(name)`: 1) `findLoadedClass` cache, 2) ask parent, 3) only if parent fails call `findClass` on self.
Why: (a) **security**, so you cannot supply your own `java.lang.String`; (b) **consistency**, a class has one identity `(name, defining loader)`. Same bytes loaded by two loaders are two different, incompatible classes: the classic `ClassCastException: com.Foo cannot be cast to com.Foo`.

### Why servlet containers break delegation

Tomcat `WebappClassLoader` is **child-first** for webapp classes (it looks in `WEB-INF/classes` and `WEB-INF/lib` before asking its parent), except for `java.*` and a few container packages. Reason: every webapp must be able to ship **its own version** of a library (Jackson 2.9 in app A, 2.15 in app B) and be **redeployed** by discarding the loader. Consequences:
- Duplicate copies of a class loaded per webapp: more Metaspace.
- **Metaspace leak on redeploy**: if anything outside the webapp (a JVM-wide thread, a `ThreadLocal` in a container thread, a JDBC driver registered in `DriverManager`, a shutdown hook, a static in a shared-loader class) holds a reference to one class or instance from the old loader, the **entire old loader and every class it loaded stay reachable**. Each redeploy adds another copy: Metaspace grows until `OutOfMemoryError: Metaspace`.
- The same pattern appears in Spring Boot devtools restart, plugin systems, Groovy/script engines and dynamically generated proxies/`Lambda` classes.

### Class unloading

A class is unloaded only when its **defining class loader** becomes unreachable (all its classes, all instances, all `Class` objects unreferenced). Bootstrap/platform/app-loader classes are effectively never unloaded. Unloading happens as part of GC (G1: at the end of concurrent marking/remark, `-XX:+ClassUnloadingWithConcurrentMark` default true; ZGC concurrent; Parallel/Serial: Full GC). Demo 3.6 shows a leaking loader hitting `Metaspace` OOM while a non-leaking one runs 20,000 loads without issue.

### The three "class problems" people confuse

| Error | Type | When | Typical cause |
|---|---|---|---|
| `ClassNotFoundException` | checked `Exception` | Explicit dynamic load: `Class.forName`, `ClassLoader.loadClass`, reflection, JDBC driver name string | Jar missing, wrong name, wrong loader |
| `NoClassDefFoundError` | `Error` (LinkageError) | Class existed at compile time, **cannot be found at runtime** (jar not packaged, `provided` scope), **or** an earlier static initializer failed ("Could not initialize class X") | Missing dependency at runtime; previous `ExceptionInInitializerError` |
| `ExceptionInInitializerError` | `Error` | **First** access: a `static {}` or static field initializer threw | Config read, `1/0`, bad resource in static init |

Real output from demo 3.7: the first touch of a class with a failing static init gives `ExceptionInInitializerError`; every **later** touch gives `NoClassDefFoundError: Could not initialize class ...` whose cause is the original error, and the class is dead until its loader is discarded. Look at the **first** occurrence in the logs, not the NCDFE spam.

### Class-initialization deadlock

`<clinit>` runs under a per-class init lock. If class A's static init needs B and B's static init needs A, and two threads start from opposite ends, they deadlock. **`jstack`/`Thread.print` does NOT report it as a deadlock**; the threads appear `RUNNABLE` (see real dump in demo 3.7: `- waiting on the Class initialization monitor for Init$B`). Recognise it by frames sitting in `<clinit>`. Fix: no cross-references between static initializers, no threads started from static blocks that touch other classes, lazy holder idiom.

## 2.3 Object layout and compressed oops

### 64-bit HotSpot object (JDK 21, default flags)

```
Instance object:
 offset  0  +--------------------------------+
            | mark word (8 bytes)            |  identity hashcode, GC age (4 bits, max 15),
            |                                |  lock state (unlocked/thin/inflated), GC forwarding info
 offset  8  +--------------------------------+
            | klass pointer (4 bytes)        |  compressed class pointer -> Klass in Metaspace
 offset 12  +--------------------------------+   (header = 12 bytes)
            | instance fields...             |  JVM reorders: longs/doubles, ints, shorts, bytes, refs;
            | (gaps are back-filled)         |  fills the 4-byte gap at offset 12 with an int/ref when possible
            +--------------------------------+
            | padding to multiple of 8       |  ObjectAlignmentInBytes = 8 by default
            +--------------------------------+

Array object: header 12 + length (4 bytes) = 16 bytes, then elements, then padding to 8.
```

Facts:
- JDK 21 header = **12 bytes** (mark 8 + compressed klass 4). Compressed class pointers are independent from compressed oops since JDK 15, so `-XX:-UseCompressedOops` alone still leaves a 12-byte header (verified in demo 3.5: `Object` stays 16 bytes). JDK 24+ has compact object headers (8-byte header, JEP 450, experimental initially); not in 21.
- Biased locking was disabled in JDK 15 and removed in 18. Lock state now lives in the mark word as thin (stack) lock or inflated to an `ObjectMonitor`.
- An identity `hashCode()` is stored in the mark word once computed; that is why an object that has had `hashCode()` called on it cannot use biased locking (historical) and why `System.identityHashCode` is not free.

### Compressed oops (ordinary object pointers)

References are 4 bytes instead of 8 by storing `(address - base) >> 3`. Because objects are 8-byte aligned, 32 bits address 2^32 x 8 = **32 GB**. So:
- Heap below ~32 GB (the exact threshold is slightly under, depending on heap base/zero-based mode; in practice stay **at or under ~31 GB**) uses 4-byte refs: less memory, better cache use.
- Cross the threshold and every reference doubles to 8 bytes: a 32 GB heap can hold **less** than a 31 GB heap. Classic advice: "do not set Xmx between 32 and ~48 GB".
- `-XX:ObjectAlignmentInBytes=16` extends the limit to 64 GB at the cost of more padding.
- ZGC does not use compressed oops (colored pointers need 64-bit refs) so footprint is larger with ZGC.

### Computing object size (worked example)

```java
class Order { long id; int qty; Object customer; boolean paid; }
```
```
offset  0  mark word                 8
offset  8  klass ptr                 4
offset 12  qty (int)                 4     <- back-fills the gap before the 8-byte-aligned long
offset 16  id (long)                 8
offset 24  customer (compressed ref) 4
offset 28  paid (boolean)            1
offset 29  padding                   3     -> total 32 bytes
```

Strings: `"hello"` = `String` object (12 header + `byte[] value` ref 4 + `int hash` 4 + `byte coder` 1 + `boolean hashIsZero` 1 = 22, padded to **24**) + `byte[5]` (16 header/length + 5 = 21, padded to **24**) = **48 bytes** for 5 characters. That is why huge `Map<String,X>` keys hurt.

`Integer` = 16 bytes; an `int[]` of 3 = 16 + 12 = 28 -> 32 (measured `int[3]` = 32 in demo 3.5). `List<Integer>` with 1M entries: 4 MB array of refs + 16 MB of `Integer` objects (beyond the -128..127 cache) = about 20 MB, versus 4 MB for `int[]`.

Tools: **JOL** (`org.openjdk.jol:jol-core`, `ClassLayout.parseClass(X.class).toPrintable()`) prints real offsets; `jcmd <pid> GC.class_histogram` gives per-class totals; `ThreadMXBean.getThreadAllocatedBytes` (used in demo 3.5) measures allocation directly.

## 2.4 Stack frames, StackOverflowError and -Xss

Each method call pushes a **frame**:
```
+------------------------------+
| local variable array         |  slot 0 = this (instance methods), then params, then locals (long/double take 2 slots)
| operand stack                |  scratch stack for bytecode: iload, iload, iadd, istore
| frame data                   |  runtime constant pool ref, return address, exception table lookup
+------------------------------+
```
`a + b` compiles to `iload_1; iload_2; iadd; istore_3`. The JIT maps locals and operand stack onto registers and machine stack, so compiled frames are usually **smaller** than interpreted frames: depth reached varies with warm-up (deep recursion may work after the JIT compiles it and fail before).

`StackOverflowError` causes: unbounded/very deep recursion (most common: missing base case, cyclic `toString`/`hashCode`/`equals`, bidirectional JPA entities serialized by Jackson, cyclic Spring bean proxies calling themselves); legit deep recursion (parsers, tree walking) on a small stack; huge frames (big local arrays are not on stack, but many locals/params are); frameworks with deep proxy/AOP chains.

Tuning `-Xss`: applies to **new** threads (main thread on Linux is governed by the launcher/ulimit). Bigger `-Xss` x thread count = virtual memory; 1 MB x 1000 threads = 1 GB reserved. Prefer fixing recursion (iteration or explicit stack) over raising it. Measured depths in demo 3.4 (trivial one-int frame): 256k gives ~2.6k frames, 1m ~27k, 4m ~218k (machine-specific; interpreted frames are larger than compiled ones). Real code with 5-10 locals reaches maybe a third of that.

The stack trace of an SOE is truncated at `-XX:MaxJavaStackTraceDepth` (default 1024): look at the repeating pattern in the top lines.

## 2.5 Escape analysis, scalar replacement and TLAB (conceptual)

**TLAB**: each thread gets a private chunk of Eden. Allocation = bump a pointer inside your own chunk (about 10 instructions, no lock, no CAS). When the TLAB is full the thread asks for a new one (a CAS on the shared Eden top). Large arrays that do not fit go outside the TLAB (slow path). This is why `new` in Java is cheaper than `malloc`. `-XX:+UseTLAB` is default; `-XX:TLABSize` rarely needs touching.

**Escape analysis (C2 only, `-XX:+DoEscapeAnalysis` default)**: if the JIT can prove an object never escapes the method/thread (not stored in a field, not passed to unknown code, not returned), it can:
- **Scalar replacement**: do not allocate at all, keep fields in registers (`new Point(x,y)` used only for `p.x + p.y` costs nothing).
- **Lock elision**: remove `synchronized` on non-escaping objects (a local `StringBuffer`).
It depends on inlining succeeding. Implications: microbenchmarks that discard results measure nothing (dead-code elimination); "Java allocates everything on the heap" is true by specification, but the JIT may not allocate at all. Java does **not** stack-allocate objects.

## 2.6 String pool, dedup and compact strings

- The **string table** is a native hash table (`-XX:StringTableSize`, default 65536 buckets, resizes dynamically) whose entries point to `String` objects **in the heap**. Literals are added at first use (lazily resolved from the constant pool by `ldc`).
- `intern()` returns the pooled instance, adding one if missing. It is a global hash lookup: cost, and adds GC roots pressure for the table. Use for a bounded set of high-duplication values (country codes); never for unbounded data (IDs, user input).
- `new String("a")` makes a distinct heap object; `"a" == new String("a")` is `false`; compile-time constants (`"a" + "b"`) are folded and pooled; runtime concatenation is not.
- **Compact strings (JDK 9+, `-XX:+CompactStrings` default)**: `String` holds `byte[] value` plus `coder` (0 = LATIN1, 1 = UTF16). ASCII/Latin-1 text costs 1 byte/char instead of 2, roughly halving String memory for typical English data. A single non-Latin-1 char makes that whole string UTF16.
- **G1 String Deduplication (`-XX:+UseStringDeduplication`, off by default)**: G1 (and since JDK 18 also Serial/Parallel/ZGC/Shenandoah support it) finds `String` objects with equal content and makes them share **one `byte[]`**. It does **not** merge the String objects, does not change identity (`==` semantics unchanged) and only considers strings that survived a few GCs (age threshold) - so it targets long-lived duplicates such as cache contents. It costs background CPU; measure with `-Xlog:stringdedup*`.
- Pool vs dedup: interning shares the **String object** (explicit, you call it); dedup shares only the **backing array** (automatic, GC-driven).

## 2.7 JIT: interpreter, C1, C2, inlining, deoptimization

### Tiered compilation (default since JDK 8)

```
Level 0  Interpreter            profiles invocation & backedge counters
Level 1  C1, no profiling       trivial methods
Level 2  C1, limited profiling  (when C2 queue is long)
Level 3  C1, full profiling     ~30% slower than level 1, collects branch/type data
Level 4  C2, optimized          aggressive, uses the profile
Typical path: 0 -> 3 -> 4
```
- **Counters**: per-method invocation counter + loop back-edge counter. Defaults observed: `Tier3InvocationThreshold=200`, `Tier4InvocationThreshold=5000` (thresholds scale with compile-queue length). Hot loop in a single call is compiled via **OSR** (on-stack replacement), shown as `%` in `-XX:+PrintCompilation` (real trace in demo 3.8: `61 % 3 Warm::work @ 4`, then `% 4`, then `made not entrant` for the level-3 version).
- Compilation happens on background threads (`C1 CompilerThread`, `C2 CompilerThread`; `-XX:CICompilerCount`). Under CPU quota (containers with 1 CPU) compile threads compete with your app, so warm-up is slow and latency spikes early.
- **Inlining** is the master optimization (enables escape analysis, constant folding). Limits: `MaxInlineSize=35` bytecodes for cold call sites, `FreqInlineSize=325` for hot ones; virtual calls inline via **class hierarchy analysis** and **inline caches** (monomorphic/bimorphic guarded inlining; megamorphic calls do not inline). Huge methods (>8000 bytecodes, `HugeMethodLimit`) are never compiled.
- **Deoptimization**: compiled code rests on speculative assumptions (this branch never taken, this call site sees only `ArrayList`, this class has no subclass). When an assumption breaks (a new class is loaded that overrides the method, a null appears where profile said none), the code is marked **not entrant** and execution falls back to the interpreter, then recompiles. Frequent deopts (`-Xlog:deoptimization=debug`) produce periodic latency blips.
- **Code cache**: default 240 MB (segmented: non-nmethods, profiled, non-profiled). When full: "CodeCache is full. Compiler has been disabled" and the app falls back to interpreted speed (10x slower, see demo 3.8: `-Xint` 220 ms vs 24 ms per round). Look with `jcmd <pid> Compiler.codecache`. Causes: heavy dynamic class generation, huge apps, tiny `ReservedCodeCacheSize`. `-XX:+UseCodeCacheFlushing` (default) helps but is not magic.

### Warm-up and why microbenchmarks lie

- First N iterations run in the interpreter/C1: measured times drop, then plateau.
- Dead-code elimination removes a loop whose result is unused; constant folding evaluates an input that is a constant; loop unrolling and OSR make a single-`main` benchmark differ from real call patterns; GC and JIT threads add noise; profile pollution (running case A then B makes call sites megamorphic).
- Use **JMH** (`@Benchmark`, `@Warmup`, `@Measurement`, `@Fork`, `Blackhole.consume`), which forks a fresh JVM, warms up, controls dead-code elimination. Report distribution not one number.
- Observation in demo 3.8: interpreter-only is ~10x slower than JIT; but in this trivial demo even round 0 was fast because OSR/C1/C2 kicked in within ~50 ms. Real warm-up (Spring app, many classes) takes seconds to minutes: relevant to autoscaling, canaries and load tests (discard the warm-up window).

## 2.8 GC fundamentals

### Reachability and GC roots

An object is live iff reachable from a **GC root** by following references:
- Local variables and operand stack entries in all thread stacks (only live slots per compiled-code oop maps)
- Static fields of loaded classes (via the `Class` mirror)
- JNI global/local handles
- Active `Thread` objects, monitors held, the system class loader and other VM-internal handles
- Anything reachable through the string table/`ClassLoader` structures of live loaders

`Reference` objects add nuance (2.15). A **cycle without a root is garbage** (no reference counting).

### Algorithms

| Algorithm | How | Pros | Cons |
|---|---|---|---|
| Mark-sweep | Mark live, free dead in place | Simple | Fragmentation, slow allocation (free lists) |
| Mark-compact | Mark, then slide live objects together | No fragmentation, bump alloc | Touches all live data, long pause |
| Copying (scavenge) | Copy live from *from-space* to *to-space*, then reset the old space | Cost proportional to **live** data, bump alloc, compacts for free | Needs spare space |

Young gen uses copying (few survivors); old gen uses mark-compact (Serial/Parallel) or region evacuation (G1) or concurrent relocation (ZGC/Shenandoah).

### Generational hypothesis

Weak: most objects die young. Strong: the older an object is, the less likely it dies. So: collect the small young area often (cost proportional to survivors: a few ms), promote survivors after **tenuring threshold** (`MaxTenuringThreshold` observed default 15, but adaptively lowered when the survivor space overflows), collect old rarely. Objects that die in old gen cost you a big collection, so **premature promotion** (survivor space too small, or a bursty allocation that fills survivors) is a major cause of Full GCs.

### Card table, remembered sets, write barriers

Young GC must not scan the entire old gen to find old-to-young pointers. HotSpot divides the heap into **512-byte cards** (log line `CardTable entry size: 512`). Every reference store `obj.f = ref` runs a **post-write barrier**:
```
obj.f = ref            // the store
card[obj >> 9] = DIRTY // Parallel/Serial: unconditional dirty; G1: filters same-region / null, then enqueues the card
```
Young GC scans only dirty cards as extra roots. G1 keeps per-region **remembered sets** (which other regions point into me) fed by concurrent refinement threads (`G1ConcRefinementThreads` observed 10). G1 also has a **pre-write barrier (SATB)** that logs the *old* value being overwritten during concurrent marking, so marking sees a snapshot as of the cycle start. ZGC/Shenandoah instead use **load barriers** (ZGC: on every reference *load* from the heap check color bits and heal the pointer; generational ZGC also has store barriers/remembered set).

Barrier cost is why G1 has ~5-15% lower raw throughput than Parallel, and ZGC a bit more.

### Safepoints

A **safepoint** is a moment when all Java threads are paused at known points where the VM can inspect their stacks. Needed for: GC pauses, deoptimization, biased-lock revocation (historical), class redefinition, thread dumps/heap dumps, code-cache sweeps.
```
VM thread: "safepoint now" -> sets a poll flag
Java threads: keep running until their next SAFEPOINT POLL (method return, loop back-edge, etc.)
   ... time-to-safepoint (TTSP): waiting for the slowest thread
All at safepoint -> VM operation runs (GC, ...)  -> resume
```
Pause seen by users = **TTSP + VM operation time**. GC logs show only the GC part; a long TTSP (one thread in a long counted loop, or slow I/O in a JNI call) makes p99 latency worse than the GC log suggests. Detect with `-Xlog:safepoint` ("Safepoint ... Total: X ns, Reaching safepoint: Y ns"). Frequent tiny safepoints (thread dumps by a monitoring agent, `System.gc()`) also hurt.

## 2.9 The collectors

### Heap layouts

```
SERIAL / PARALLEL   (contiguous generations)
+-----------------------------------------------+---------------------------------------+
|                   YOUNG                       |              OLD (tenured)            |
| +----------------------+-----------+--------+ |                                       |
| |         EDEN         |   S0(from)|S1(to)  | |   promoted objects, large arrays      |
| +----------------------+-----------+--------+ |                                       |
+-----------------------------------------------+---------------------------------------+
 NewRatio=2 (old = 2x young); SurvivorRatio=8 (eden = 8x each survivor)

G1  (equal-sized regions, roles change over time; region size 1-32 MB power of 2, ~heap/2048)
+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
| E | E | S | O | O | E | H | H | H | O | F | E | O | S | F | E |
+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
E=Eden  S=Survivor  O=Old  H=Humongous (start+continues)  F=Free
Humongous object: size >= 1/2 region; gets contiguous H regions, not copied.

ZGC  (regions of dynamic size classes: small 2 MB, medium 32 MB, large = multiple of 2 MB)
+-----------+-----------+---------+------------------+
| small (2M)| small (2M)| medium  |   large object   |  ... generational ZGC (JDK 21) keeps young+old region sets
+-----------+-----------+---------+------------------+
```

### Serial
Single-threaded, stop-the-world young (copy) and full (mark-sweep-compact). No barriers beyond card marking, lowest footprint and overhead. Chosen automatically when the JVM sees < 2 CPUs or < ~1792 MB memory ("not server class"). Good for small heaps (< a few hundred MB), CLI tools, tiny sidecars.

### Parallel (throughput collector)
Same layout, multi-threaded young and full GCs (`ParallelGCThreads`). Maximizes throughput; pauses grow with heap size and live set (full GC of a 20 GB heap can be many seconds). Adaptive sizing (`UseAdaptiveSizePolicy`) resizes eden/survivors to hit `MaxGCPauseMillis`/`GCTimeRatio`. `GCTimeRatio` (Parallel default 99 = GC target about 1% of time; G1 default 12 = about 8%; formula 1/(1+ratio)) steers the sizing. (The value 12 in a `PrintFlagsFinal` on a default JVM is G1's.) "GC overhead limit exceeded" (`UseGCOverheadLimit`, `GCTimeLimit=98`) is implemented by Parallel: >98% of time in GC recovering <2% heap. Best for batch/analytics where a 1-3 s pause is acceptable.

### G1 (default since JDK 9)

Goals: predictable pauses (`-XX:MaxGCPauseMillis`, default 200, a **target**, not a guarantee) on multi-GB heaps, no long Full GCs.

**Young collection** (STW, parallel): all Eden + Survivor regions form the collection set; live objects are *evacuated* (copied) into new survivor regions or promoted into old regions. Eden count adapts to hit the pause goal using pause-time prediction.

**Concurrent marking cycle** (starts when old occupancy exceeds IHOP):
```
1. Concurrent Start   (STW, piggybacked on a young pause)  mark roots ("initial mark")
2. Root region scan   (concurrent)  scan survivor regions for pointers into old
3. Concurrent Mark    (concurrent)  trace the object graph, SATB barrier records overwritten refs
4. Remark             (STW, short)  finish SATB buffers, weak refs, class unloading
5. Cleanup            (STW, short)  count live bytes per region, reclaim fully empty regions
6. Concurrent Cleanup/Rebuild remembered sets
```
Then **mixed collections**: young regions **plus** the old regions with the most garbage ("garbage first"), spread over up to `G1MixedGCCountTarget=8` pauses, only regions whose garbage exceeds ~`G1HeapWastePercent=5` overall. The cycle ends when reclaimable old space falls below that.

**IHOP** (`InitiatingHeapOccupancyPercent`, default 45, and `G1UseAdaptiveIHOP` default true): start marking early enough that it finishes before the heap fills. Adaptive IHOP learns allocation and marking speed and starts at (roughly) the occupancy that leaves the reserve. `G1ReservePercent=10` keeps free space for evacuation.

**Humongous objects**: size >= half a region (region = 1..32 MB; heap 4 GB -> 2 MB region -> objects >= 1 MB). Allocated directly in contiguous old-gen-like regions, never copied, reclaimed at cleanup or eagerly at young GC if the object is a primitive array with no incoming refs. Many big short-lived `byte[]` (JSON buffers, image processing) cause "Pause Young (Concurrent Start) (G1 Humongous Allocation)" **premature marking cycles** and fragmentation (need *contiguous* free regions even when total free space is enough). Real example in demo 3.1: `GC(3) Pause Young (Concurrent Start) (G1 Humongous Allocation) 55M->29M(64M)`. Fix: `-XX:G1HeapRegionSize=` larger (16m/32m) so objects stop being humongous, or chunk the buffers.

**Evacuation failure ("To-space exhausted")**: G1 could not find free regions to copy survivors into. Result: objects are left in place (pinned), the pause is long, and it is often followed by a Full GC. Causes: heap too small for live set + allocation burst, IHOP too high, reserve too low, humongous churn. Mitigation: bigger heap, lower `InitiatingHeapOccupancyPercent` (or leave adaptive), raise `G1ReservePercent`, fix allocation spikes.

**Full GC** in G1 (`Pause Full (G1 Compaction Pause)` or "(Allocation Failure)") is a parallel STW mark-compact of the entire heap (multi-threaded since JDK 10). Seeing any Full GC in G1 is a **design signal**: the concurrent cycle lost the race. Real example in demo 3.3 with a leak.

### ZGC (`-XX:+UseZGC`; generational: `-XX:+UseZGC -XX:+ZGenerational`, JDK 21)

- **Colored pointers**: 64-bit refs carry metadata bits (marked0/marked1/remapped/finalizable) in the pointer itself. Consequently no compressed oops, max heap 16 TB.
- **Load barrier**: after loading a reference field from the heap, compiled code tests the color bits against a "bad mask"; if bad (object might have moved or is unmarked) a slow path fixes the pointer ("self-healing"). Cost: about 4 extra instructions on the fast path.
- **Concurrent phases**: mark start (tiny STW), concurrent mark, mark end (tiny STW), concurrent select relocation set, relocation start (tiny STW), **concurrent relocate** (compaction happens while the app runs; stale pointers are healed lazily by load barriers). All STW pauses are typically **< 1 ms and independent of heap size or live set** (they only scan roots).
- **Generational ZGC (JDK 21, opt-in)** adds young/old generations to ZGC: most garbage dies young and is collected cheaply, cutting CPU and the risk of **allocation stalls**. In JDK 23 generational became the default, JDK 24 removed non-generational.
- Trade-offs: 5-20% throughput cost vs Parallel; needs headroom (heap should be larger than live set, roughly 2x live set is comfortable); if allocation outpaces concurrent collection the app hits **Allocation Stall** (threads block until memory is freed): that is the ZGC "OOM before OOM" and shows in the log as `Allocation Stall (main) 168.9ms` (real, demo 3.1, on a deliberately tiny 64 MB heap).
- Uncommits unused memory (`ZUncommit` default true, delay `ZUncommitDelay`), `-XX:SoftMaxHeapSize` softens the target.

### Shenandoah (OpenJDK builds; not in Oracle JDK)
Concurrent **evacuation** too, using a load-reference barrier (older versions: Brooks forwarding pointer) plus SATB marking; keeps compressed oops. Pause ~1-10 ms, largely independent of heap size. Non-generational in 21 (generational arrived experimental in 24). Choose it on Red Hat/Temurin-based stacks when you want ZGC-like latency with compressed oops; Red Hat's docs favour it for heaps < ~30 GB. Enable: `-XX:+UseShenandoahGC`.

### Removed/deprecated worth naming
CMS: deprecated JDK 9, removed JDK 14. Concurrent-mode-failure was its notorious Full GC. Epsilon (`-XX:+UseEpsilonGC`, needs `UnlockExperimentalVMOptions`) is a no-op GC for benchmarks.

## 2.10 Choosing a collector, containers and ergonomics

| Workload | Priority | Pick | Why |
|---|---|---|---|
| Batch, ETL, report jobs | Throughput | Parallel | Least barrier overhead, fastest total run time |
| Typical REST microservice, 512 MB-8 GB | Balance | G1 (default) | Good pauses, adaptive; tune only if data says so |
| Trading, ad serving, real-time APIs, heaps 8 GB-multi-TB, p99 < 10 ms | Latency | ZGC (generational on 21) or Shenandoah | Pauses independent of heap |
| Sidecar, CLI, Lambda-like, < 1 CPU, < 512 MB | Footprint/startup | Serial | Least memory and threads |
| Huge caches in heap (Cassandra/ES-like) | Latency + capacity | ZGC | Old-gen pauses don't scale with live set |

**Container awareness**: `-XX:+UseContainerSupport` (default true on Linux since JDK 10, backported to 8u191) makes the JVM read cgroup limits (v1 and v2) instead of host resources.
- **Memory**: max heap = `MaxRAMPercentage` (default **25%**, observed `25.000000`) of the container limit. A 2 GB pod gives a 512 MB heap by default: a very common surprise. `InitialRAMPercentage` default 1.5625%; `MinRAMPercentage` (default 50) applies when the limit is tiny (< ~250 MB). Setting explicit `-Xmx` overrides all percentages.
- **CPU**: `availableProcessors()` = ceil(cpu limit) (limit/period, e.g. `limits.cpu: 1500m` -> 2). CPU *requests* (cgroup shares) are ignored by recent JDKs. `-XX:ActiveProcessorCount=N` overrides. This drives: GC thread counts (`ParallelGCThreads`, `ConcGCThreads`), `ForkJoinPool.commonPool` size, and **collector selection**: fewer than 2 CPUs or < ~1792 MB -> Serial is chosen automatically instead of G1. So `limits.cpu: 1` silently gives you SerialGC (verify with `-Xlog:gc*` first line `Using Serial`, or `-XX:+PrintFlagsFinal | grep Use.*GC`).
- CPU throttling (CFS quota) is not visible as a pause in GC logs but stretches wall-clock time: GC threads and JIT threads burn the pod's quota; p99 rises. Watch `container_cpu_cfs_throttled_seconds_total`.

## 2.11 Reading GC logs

Enable (JDK 9+ unified logging): `-Xlog:gc*:file=gc.log:time,uptime,level,tags:filecount=5,filesize=20m`. Minimal: `-Xlog:gc`. Details: `-Xlog:gc*,gc+heap=debug,safepoint`. (JDK 8's `-XX:+PrintGCDetails` no longer exists.)

### Real G1 log, annotated (demo 3.1, 64 MB heap, `-Xlog:gc*`)

```
[0.020s][info][gc,init] CardTable entry size: 512                 <- card = 512 bytes (write barrier granularity)
[0.023s][info][gc,init] CPUs: 12 total, 12 available               <- what ergonomics saw; in a pod check this line first
[0.023s][info][gc,init] Memory: 15182M
[0.023s][info][gc,init] Compressed Oops: Enabled (32-bit)          <- heap < 32 GB
[0.023s][info][gc,init] Heap Region Size: 1M                       <- 64M heap -> 1 MB regions (humongous >= 512 KB)
[0.023s][info][gc,init] Parallel Workers: 10                       <- STW worker threads
[0.023s][info][gc,init] Concurrent Workers: 3                      <- marking threads
[0.032s][info][gc,metaspace] Compressed class space mapped at ..., reserved size: 1073741824   <- 1 GB reservation
[0.081s][info][gc,start    ] GC(0) Pause Young (Normal) (G1 Evacuation Pause)
                                                                    ^ young pause #0, cause = eden filled up
[0.081s][info][gc,task     ] GC(0) Using 2 workers of 10 for evacuation      <- G1 scales workers to the work
[0.083s][info][gc,phases   ] GC(0)   Pre Evacuate Collection Set: 0.1ms
[0.083s][info][gc,phases   ] GC(0)   Merge Heap Roots: 0.0ms                 <- remembered set/dirty cards merged
[0.083s][info][gc,phases   ] GC(0)   Evacuate Collection Set: 0.7ms          <- the copying; dominates normally
[0.083s][info][gc,phases   ] GC(0)   Post Evacuate Collection Set: 0.2ms
[0.083s][info][gc,phases   ] GC(0)   Other: 0.3ms
[0.083s][info][gc,heap     ] GC(0) Eden regions: 23->0(35)          <- 23 used before, 0 after, next eden target 35
[0.083s][info][gc,heap     ] GC(0) Survivor regions: 0->1(3)
[0.083s][info][gc,heap     ] GC(0) Old regions: 0->0
[0.083s][info][gc,heap     ] GC(0) Humongous regions: 1->1          <- one big array stays (my 512 KB survivor array)
[0.083s][info][gc,metaspace] GC(0) Metaspace: 133K(320K)->133K(320K) ...
[0.083s][info][gc          ] GC(0) Pause Young (Normal) (G1 Evacuation Pause) 24M->1M(64M) 2.079ms
                                                                    ^ heap before -> after (capacity) pause time
[0.083s][info][gc,cpu      ] GC(0) User=0.00s Sys=0.00s Real=0.00s   <- user >> real = parallel; sys high = page faults/swap; real > user+sys = waiting (throttling/IO)
```

### Real one-line logs (`-Xlog:gc`), annotated

Serial (observed):
```
[0.185s][info][gc] GC(0) Pause Young (Allocation Failure) 17M->0M(61M) 1.566ms
[0.233s][info][gc] GC(7) Pause Young (Allocation Failure) 18M->15M(61M) 5.768ms
```
"Allocation Failure" only means "the young space was full", **not** an error. The second line has 15M left: survivors are being copied/promoted (pause grew from 1 ms to 5.7 ms).

Parallel (observed): `GC(7) Pause Young (Allocation Failure) 19M->11M(62M) 2.693ms` (same shape, parallel threads).

G1 with humongous (observed):
```
GC(3) Pause Young (Concurrent Start) (G1 Humongous Allocation) 55M->29M(64M) 1.273ms
GC(4) Concurrent Mark Cycle
GC(4) Concurrent Mark Cycle 19.454ms
```
Concurrent Start triggered by a **humongous allocation** (heap 55/64 used). Concurrent phases run alongside the app; only "Pause" lines stop threads.

G1 in a leak, near death (observed, demo 3.3):
```
GC(34) Pause Young (Concurrent Start) (G1 Humongous Allocation) 62M->62M(64M) 0.543ms   <- nothing reclaimed: live set = heap
GC(36) Pause Young (Normal) (G1 Humongous Allocation) 62M->62M(64M) 0.625ms
GC(37) Pause Full (G1 Compaction Pause) 62M->62M(64M) 4.437ms                          <- Full GC that frees nothing
GC(38) Pause Full (G1 Compaction Pause) 62M->62M(64M) 2.282ms                          <- then OOM
```
**The leak fingerprint**: after-GC occupancy keeps ratcheting upward and Full GCs reclaim ~0.

### Cheat: cause strings

| Log text | Meaning |
|---|---|
| `Pause Young (Allocation Failure)` (Serial/Parallel) | Eden full: normal |
| `Pause Young (Normal) (G1 Evacuation Pause)` | Normal G1 young |
| `Pause Young (Concurrent Start) (G1 Humongous Allocation)` | Big arrays trigger marking: region size/humongous problem |
| `Pause Young (Mixed) (G1 Evacuation Pause)` | Mixed collection reclaiming old regions |
| `... To-space exhausted` / `Evacuation Failure` | No room to copy survivors: increase heap/reserve, lower IHOP |
| `Pause Full (Allocation Failure)` / `Pause Full (G1 Compaction Pause)` | Concurrent cycle lost the race, or leak |
| `Pause Full (System.gc())` | Someone called `System.gc()` (RMI does hourly; `-XX:+DisableExplicitGC` or `ExplicitGCInvokesConcurrent`) |
| `Pause Full (Metadata GC Threshold)` | Metaspace hit its high-water mark: class churn/leak, raise `MetaspaceSize` |
| `Promotion failed` (Parallel/CMS era) / old-gen full during young | Old full while promoting: premature promotion or leak |
| ZGC `Allocation Stall` | Threads blocked: GC cannot keep up; bigger heap or more `ConcGCThreads` |
| ZGC `Minor Collection (Allocation Rate)` / `Major Collection (High Usage/Warmup)` | Trigger reasons (real lines in demo 3.1) |
| `Safepoint ... Reaching safepoint: 800 ns/ms` (in `-Xlog:safepoint`) | Large "Reaching" = TTSP problem, not GC |

Key ratios to compute from a log: **allocation rate** = (after-previous-GC to before-this-GC delta) / time between GCs; **promotion rate** = growth of old gen per GC; **GC time %** = sum(pause)/wall time; **max/p99 pause**. Tools: GCViewer, GCeasy, `jfr summary`, JMC's GC page.

## 2.12 Heap sizing and tuning methodology

Sizing rules of thumb:
- **`-Xms` = `-Xmx`** in servers: no resize pauses, predictable RSS, avoids the OS committing lazily under load. (For dense pods with idle apps, a lower Xms is acceptable.) `-XX:+AlwaysPreTouch` commits at start (slower start, no page-fault stalls).
- Size heap as **live set after Full GC x 3-4** as a first estimate (ZGC: x2+ headroom minimum). Live set = the "after" number of a Full GC, or old gen after mixed cycles.
- Young size: leave to ergonomics under G1 (`-Xmn`/`NewRatio` disable adaptive young sizing and your pause goal). Under Parallel `-Xmn` or `NewRatio` matter.
- Metaspace: set `-XX:MaxMetaspaceSize` (e.g. 256m) in containers so a class-loader leak fails fast with an OOM and heap dump instead of an OOMKilled with no diagnostics. Do not set `MetaspaceSize` unless you see `Metadata GC Threshold` Full GCs at startup.
- Direct memory: `-XX:MaxDirectMemorySize` (defaults to `Xmx`); Netty pooled buffers make this real memory.
- Thread stacks: threads x `-Xss`.

**Tuning methodology** (memorise):
```
1. Define target (SLO): p99 pause < 50 ms? throughput > 95% of time in mutator? 
2. Measure baseline under realistic load: GC log + JFR, capture 30+ min, discard warm-up.
3. Compute: allocation rate (MB/s), promotion rate (MB/s), live set, GC %, p99/max pause, Full GC count.
4. Hypothesise from evidence (not folklore):
      high alloc rate -> reduce garbage (fix code) or larger young
      high promotion -> survivors too small / caching / leak
      humongous -> region size or chunking
      Full GCs -> heap too small, IHOP, leak, System.gc()
5. Change ONE variable, re-run same load, compare.
6. Keep defaults where they meet the SLO. Remove old flags on each JDK upgrade.
```
Most effective "tuning" is **allocating less**: avoid boxing, big temp strings/JSON buffers, unbounded `SELECT *` result lists, `toArray` copies, regex recompiles.

## 2.13 Memory leaks, diagnosis toolbox and OOM variants

### Common leak patterns (Java "leak" = reachable but unused)
Static collections/caches without eviction; `ThreadLocal` values in pooled threads never `remove()`d (also the classloader-leak source); listeners/callbacks never deregistered; `HashMap` keys with mutable `hashCode`; unbounded queues between producers and consumers (back-pressure missing); `ClassLoader` leaks; inner/anonymous classes and lambdas capturing `this`; open `Statement`/`ResultSet`/streams; `String.intern` on unbounded input; `substring` retention (fixed in JDK 7u6); sessions never invalidated; per-request objects put into Spring singleton fields; Hibernate first-level cache in a huge transaction/batch import (fix `flush()+clear()`); Micrometer tags with unbounded cardinality (a metrics leak).

### Diagnosis workflow

```
1. Confirm it is a leak: post-GC "old used" trending up over hours (jstat -gcutil O column, GC log after-values, Prometheus jvm_memory_used after GC)
2. Cheap look:   jcmd <pid> GC.class_histogram | head -20      (top classes by bytes; take 2 samples 10 min apart, diff)
3. Heap summary: jcmd <pid> GC.heap_info
4. Capture:      jcmd <pid> GC.heap_dump /dumps/h.hprof         (STW, size ~ used heap; use disk with room)
                 or -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps
5. Analyse:      Eclipse MAT -> Leak Suspects report -> Dominator Tree (retained heap) -> "Path to GC Roots (exclude weak/soft)"
6. Fix + verify: same load test, sawtooth returns to flat baseline.
Continuous:      JFR (jcmd <pid> JFR.start duration=10m filename=r.jfr settings=profile) -> JDK Mission Control: Memory, Object Allocation in new TLAB, Old Object Sample (leak candidates with stack traces at allocation), GC pauses.
Allocation prof: async-profiler  asprof -e alloc -d 30 -f alloc.html <pid>   (finds allocation hot spots)
CPU profile:     asprof -e cpu -d 30 -f cpu.html <pid>
```
`jmap -dump:live` forces a Full GC first (`live` option), `jmap -histo:live` too. `jcmd GC.heap_dump` also defaults to live only (option `-all` to include unreachable).

MAT vocabulary: **shallow heap** = the object's own bytes; **retained heap** = memory freed if that object were collected (its dominator subtree); **dominator tree** = who single-handedly keeps what alive; **histogram**; **OQL**. A 64 MB `ArrayList` of 1 MB arrays has shallow heap of a few dozen bytes and retained heap ~64 MB.

`jstat -gcutil <pid> 1000` columns: `S0 S1 E O M CCS` (% used), `YGC YGCT` (young count/time), `FGC FGCT` (Full), `CGC CGCT` (concurrent cycles, G1), `GCT` total. Real output in demo 3.3.

### OOM variants

| Message | Real meaning | Diagnosis | Fix |
|---|---|---|---|
| `Java heap space` | Live set (or one huge allocation) exceeds max heap | heap dump, histogram | fix leak / raise Xmx / stream instead of loading |
| `GC overhead limit exceeded` | Parallel GC spent > 98% time recovering < 2% heap | same as above (early symptom of leak) | same; do not just disable `UseGCOverheadLimit` |
| `Requested array size exceeds VM limit` | Array > ~2^31-2 elements | stack trace | chunk |
| `Metaspace` / `Compressed class space` | Too many classes, loader leak, endless proxies/generated classes (Groovy, reflection inflation, CGLIB per call) | `jcmd VM.classloader_stats`, `GC.class_histogram`, `-Xlog:class+load`/`class+unload` | fix leak; cap with MaxMetaspaceSize; cache proxies |
| `Direct buffer memory` | NIO direct buffers > `MaxDirectMemorySize` (freed only when the owning `ByteBuffer` object is GC'd/Cleaned) | NMT `Internal`/`Other`; Netty leak detector `-Dio.netty.leakDetection.level=advanced` | release buffers, bound pools, raise limit |
| `unable to create native thread` | OS refused `pthread_create`: thread limit (`ulimit -u`, cgroup `pids.max`), or no address space/memory for stacks | `jcmd Thread.print | grep -c "^\""`, `ps -L`, `ulimit -a` | bound thread pools, lower `-Xss`, raise limits, use virtual threads |
| `Out of swap space?` / `mmap failed` | Native malloc failed (OS out of memory) | NMT, `hs_err` file | lower heap, add memory |
| `Java heap space` with `HeapDumpOnOutOfMemoryError` and pod restart | see 2.14 | | |

Important: `OutOfMemoryError` is thrown in **whichever thread** hit the wall, often a victim, not the leaker. The dump shows the leaker.

## 2.14 Native memory, NMT and Kubernetes OOMKilled

### Java OOM vs container OOMKilled

```
Java OOM (OutOfMemoryError)                 Container OOMKilled (exit code 137, reason OOMKilled)
- JVM decides: a JVM memory pool is full    - Kernel cgroup OOM killer: process RSS + page cache charged > limit
- Exception, stack trace, heap dump         - SIGKILL, no stack trace, no heap dump, no JVM shutdown hooks
- Logged by the app                         - Visible in `kubectl describe pod` (Last State: OOMKilled), dmesg
- Cause: heap/metaspace/direct limit hit    - Cause: TOTAL process memory > limit, even though heap < Xmx
```

### What makes up the process (RSS)

```
RSS  =  Java heap (committed, touched)
      + Metaspace + compressed class space
      + Thread stacks (threads x touched stack)
      + Code cache (JIT)
      + GC structures (remembered sets, mark bitmaps: G1 ~10-20% of heap in bad cases; ZGC multi-mapping)
      + Direct / mapped buffers (Netty, NIO, Kafka clients)
      + Symbol/string tables, JNI/native libs, malloc arenas & fragmentation (glibc arenas)
      + JVM internal (NMT "Internal", "Other"), agents, JFR buffers
```
Rule of thumb: non-heap overhead is **150-500 MB** for a typical Spring Boot app; more with Netty or many threads. So `-Xmx` = 100% of the limit is a guaranteed OOMKill.

### Sizing in Kubernetes

```yaml
resources:
  requests: { memory: "1Gi", cpu: "500m" }   # scheduling; requests==limits for memory = Guaranteed-ish, no surprise eviction
  limits:   { memory: "1Gi", cpu: "1" }
env: [{ name: JAVA_TOOL_OPTIONS,
        value: "-XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps -XX:MaxMetaspaceSize=256m" }]
```
- `MaxRAMPercentage=75` is a reasonable starting point for large pods (>= 2 GB): 25% left for non-heap. For small pods (512 MB) use ~50-60%; for heavy Netty/direct use, less. There is no universal number: **measure RSS** (`kubectl top`, `container_memory_working_set_bytes`) and NMT.
- `-XX:+ExitOnOutOfMemoryError` lets Kubernetes restart the pod cleanly instead of a half-dead JVM (an OOM'd thread may die while the process stays "healthy" for the probe). `-XX:+CrashOnOutOfMemoryError` produces `hs_err`/core.
- Memory **requests** drive scheduling and eviction order; **limits** drive OOMKilled. CPU limit drives throttling and ergonomics.
- Heap dump on OOM needs an `emptyDir`/PVC path with space >= heap; otherwise the dump fails or fills the disk and gets the pod evicted.

### NMT

`-XX:NativeMemoryTracking=summary` (or `detail`; ~5-10% overhead; must be set at startup).
```
jcmd <pid> VM.native_memory summary
jcmd <pid> VM.native_memory baseline
   ... run load ...
jcmd <pid> VM.native_memory summary.diff       <- categories that grew: Thread, Class, Code, GC, Internal, Other, Arena Chunk, Symbol...
```
Read "committed" per category: `Java Heap`, `Class`, `Thread` (stacks), `Code`, `GC`, `Internal`, `Symbol`, `Native Memory Tracking`, `Arena Chunk`. NMT does **not** cover memory malloc'd by third-party native libraries (JNI, some compression libs) or glibc fragmentation. If NMT total is far below RSS, suspect glibc malloc arenas (`MALLOC_ARENA_MAX=2`, or use jemalloc/tcmalloc) or native libs.

## 2.15 References, Cleaner and finalization

| Type | Cleared when | Use |
|---|---|---|
| Strong | never while reachable | normal |
| `SoftReference` | when GC needs memory, before OOM (policy uses recent-use time, `SoftRefLRUPolicyMSPerMB`) | memory-sensitive caches (prefer Caffeine with size bounds: soft refs give unpredictable GC behaviour) |
| `WeakReference` | next GC when only weakly reachable | `WeakHashMap` (keys), canonicalizing maps, listeners |
| `PhantomReference` | `get()` is always null; enqueued after the object is finalized and unreachable | post-mortem cleanup; foundation of `Cleaner` |
| `java.lang.ref.Cleaner` (JDK 9+) | Runs a registered action when the object becomes phantom-reachable | Free native resources safely (the action must **not** reference the object itself) |

**Finalization**: `Object.finalize()` deprecated in JDK 9, **deprecated for removal in JDK 18**; can be disabled with `--finalization=disabled`. Problems: unpredictable timing, run by a single low-priority `Finalizer` thread, resurrects objects, objects with finalizers need an extra GC cycle to be freed (a big slowdown and a source of OOM when finalizer thread lags), exceptions swallowed. Use `try-with-resources` + `AutoCloseable`; for safety-net use `Cleaner`.

## 2.16 Startup: CDS, AppCDS, native image

- **CDS (Class Data Sharing)**: JDK ships a pre-parsed archive (`classes.jsa`) of core classes memory-mapped at startup (visible in the real log: `CDS archive(s) mapped at: ...`, saves ~20-30% startup and shares memory across JVMs).
- **AppCDS**: archive **your app's** classes too. `java -XX:ArchiveClassesAtExit=app.jsa -jar app.jar` (dynamic archive at exit), then run with `-XX:SharedArchiveFile=app.jsa`. JDK 19+: `-XX:+AutoCreateSharedArchive -XX:SharedArchiveFile=app.jsa` creates/refreshes it automatically. Spring Boot 3.3+ supports it. Typical: 30-50% faster startup for big apps and a smaller footprint for many pods. Build the archive in the image build so pods start fast.
- **GraalVM native image**: AOT compiles to a native executable: startup in milliseconds, RSS tens of MB. Trade-offs: closed-world (reflection/proxies/resources need configuration; Spring AOT helps), no JIT so lower peak throughput (mitigate with PGO), slower builds, different GC (Serial/G1 in enterprise), harder debugging (no JVMTI/agents the same way). JVM+CDS wins on throughput and tooling; native wins on cold start and scale-to-zero. Project Leyden and CRaC (checkpoint/restore) aim at the middle.
- Other startup knobs: `-XX:TieredStopAtLevel=1` (C1 only: fast start, lower peak, often used for short-lived jobs), `-Xshare:auto` (default), `-XX:+UseSerialGC` for tiny pods, lazy init in Spring.

## 2.17 High CPU flow and thread-dump reading

```
1. top -H -p <pid>                     -> hottest thread TID in decimal
2. printf "%x\n" <tid>                 -> hex (nid in the dump is hex)
3. jcmd <pid> Thread.print > t1.txt    (or jstack -l <pid>); take 3 dumps, 5-10 s apart
4. grep -A25 "nid=0x<hex>" t1.txt      -> stack of the hot thread
5. Classify:
   - App thread in your code/regex/JSON/loop        -> profile with async-profiler (asprof -e cpu)
   - "GC Thread#n", "G1 Conc#n", "ZWorker"           -> GC is the CPU: read GC log; allocation rate, leak, heap too small
   - "C2 CompilerThread"                              -> JIT storm (warm-up, deopt loop, code cache pressure)
   - many threads RUNNABLE in same monitor/spin       -> contention / busy-wait
   - "VM Thread"                                      -> safepoint operations
```

Thread states in `Thread.print`:
| State | Meaning | Look for |
|---|---|---|
| `RUNNABLE` | Running or ready, **also blocked in native I/O (socket read)** | Do not equate with CPU; check top -H |
| `BLOCKED` | Waiting to enter `synchronized` (monitor) | `- waiting to lock <0x..>` and find `- locked <0x..>` owner |
| `WAITING` | `Object.wait()`, `LockSupport.park`, `join` (no timeout) | idle pool workers are normal |
| `TIMED_WAITING` | `sleep`, timed wait/park | normal for schedulers |

**Deadlock**: `Thread.print` ends with `Found one Java-level deadlock:` listing each thread "waiting to lock monitor 0x.. held by Thread-N" (also for `ReentrantLock`/`synchronizer` with `-l`). Cycle A holds X wants Y, B holds Y wants X. Fixes: consistent lock ordering, `tryLock` with timeout, less lock scope. Class-init deadlocks are **not** reported (2.2). Lock convoy / thread-pool starvation: all pool threads `WAITING` on a `Future` submitted to the same pool.

Programmatic detection: `ThreadMXBean.findDeadlockedThreads()`.

Also useful: `jcmd <pid> VM.uptime`, `VM.flags`, `VM.system_properties`, `VM.version`, `Thread.dump_to_file -format=json` (JDK 21, includes virtual threads).

---

# 3. Runnable demos with real output

Setup used: JDK 21.0.4 (`javac X.java` then `java ...`). Machine: 12 CPUs, ~15 GB RAM, Windows. Numbers (ms, depths) are machine-specific; the patterns are portable. `jcmd`/`jstat` live in `$JAVA_HOME/bin`.

## 3.1 Same allocation workload under four collectors (`Alloc.java`)

```java
public class Alloc {
    static byte[][] keep = new byte[40][];
    public static void main(String[] a) {
        long t0 = System.nanoTime();
        int k = 0;
        for (int i = 0; i < 3_000_000; i++) {
            byte[] b = new byte[1024];                       // 1 KB short-lived garbage (3 GB total)
            if (i % 75_000 == 0) keep[k++ % keep.length] = new byte[512 * 1024]; // occasional survivors
        }
        System.out.println("done ms=" + (System.nanoTime() - t0) / 1_000_000);
    }
}
```
Run: `java -Xms64m -Xmx64m -XX:+Use<X>GC -Xlog:gc:file=<X>.log Alloc`

**Serial** (observed):
```
[0.078s][info][gc] Using Serial
[0.185s][info][gc] GC(0) Pause Young (Allocation Failure) 17M->0M(61M) 1.566ms
[0.190s][info][gc] GC(1) Pause Young (Allocation Failure) 18M->0M(61M) 1.350ms
   ... GC(2)-GC(6) each ~1 ms, 18M->0M or 1M ...
[0.233s][info][gc] GC(7) Pause Young (Allocation Failure) 18M->15M(61M) 5.768ms
```
Annotation: capacity 61M = 64M minus one survivor space (only one survivor is usable at a time). Eden is about 17 MB, so 3 GB of garbage costs ~170 young GCs. Each pause is ~1 ms because almost nothing survives (copy cost ~ live data). GC(7): the 512 KB survivor arrays were live, 15M survive, and the pause is 5x longer: **pause ~ survivors, not garbage**.

**Parallel** (observed): same shape: `GC(0) ... 16M->1M(61M) 1.597ms` ... `GC(7) ... 19M->11M(62M) 2.693ms`. On a tiny heap extra threads barely help.

**G1** (observed):
```
[0.028s][info][gc] Using G1
[0.111s][info][gc] GC(0) Pause Young (Normal) (G1 Evacuation Pause) 24M->1M(64M) 3.032ms
[0.123s][info][gc] GC(1) Pause Young (Normal) (G1 Evacuation Pause) 36M->1M(64M) 1.423ms
[0.137s][info][gc] GC(2) Pause Young (Normal) (G1 Evacuation Pause) 39M->2M(64M) 2.680ms
[0.155s][info][gc] GC(3) Pause Young (Concurrent Start) (G1 Humongous Allocation) 55M->29M(64M) 1.273ms
[0.155s][info][gc] GC(4) Concurrent Mark Cycle
[0.175s][info][gc] GC(4) Concurrent Mark Cycle 19.454ms
```
Annotation: eden grows adaptively (24M, 36M, 39M) because G1 sizes young to fit the pause goal. `byte[512K]` with 1 MB regions is exactly half a region, so it is **humongous**; allocating it triggered an early concurrent marking cycle (GC(3): "Concurrent Start" is the STW initial-mark piggybacked on a young pause). GC(4) is not a pause: it ran concurrently for 19 ms.

**ZGC generational** on the same 64 MB heap (observed): it **fails**.
```
Exception: java.lang.OutOfMemoryError thrown from the UncaughtExceptionHandler in thread "main"
[0.130s][info][gc] GC(0) Major Collection (Warmup)
[0.346s][info][gc] GC(1) Minor Collection (High Usage)
[0.352s][info][gc] Allocation Stall (main) 168.916ms
[0.353s][info][gc] GC(1) Minor Collection (High Usage) 64M(100%)->10M(16%) 0.007s
[0.399s][info][gc] Allocation Stall (main) 16.527ms
[0.601s][info][gc] Allocation Stall (main) 193.081ms
```
Lesson worth quoting: ZGC is concurrent, so it **needs headroom**. With allocation faster than concurrent reclamation on a tiny heap the app stalls (168 ms) and finally OOMs. Rerun with `-Xms256m -Xmx256m -XX:+UseZGC -XX:+ZGenerational` (observed): completes; `GC(0) Major Collection (Warmup) 26M(10%)->136M(53%) 0.108s`. That `0.108s` is the whole concurrent cycle, not an application pause; ZGC's real pauses are separate `Pause Mark Start/End` lines visible with `-Xlog:gc*`.

## 3.2 G1 detail log (`-Xlog:gc*`)
The full annotated first pause is in 2.11. What to look for: `Using 2 workers of 10 for evacuation` (adaptive), phases `Evacuate Collection Set` (main cost) and `Merge Heap Roots` (remembered-set cost), the `Eden/Survivor/Old/Humongous regions: before->after(target)` lines, and the `User/Sys/Real` line.

## 3.3 Memory leak to OOM with heap dump (`Leak.java`)

```java
import java.util.*;
public class Leak {
    static final List<byte[]> CACHE = new ArrayList<>();      // the leak: static, never evicted
    public static void main(String[] a) throws Exception {
        System.out.println("pid=" + ProcessHandle.current().pid());
        while (true) { CACHE.add(new byte[1024 * 1024]); Thread.sleep(150); }
    }
}
```
Run: `java -Xmx64m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=leak.hprof Leak`

While it runs, about 2 s in (observed):
```
$ jstat -gcutil 21468
  S0     S1     E      O      M     CCS    YGC     YGCT     FGC    FGCT     CGC    CGCT       GCT
     -  70.34   0.00  80.00  69.71  19.90      4     0.007     0     0.000     6     0.001     0.009

$ jcmd 21468 GC.heap_info
 garbage-first heap   total 65536K, used 45744K [0x00000000fc000000, 0x0000000100000000)
  region size 1024K, 1 young (1024K), 1 survivors (1024K)
 Metaspace       used 401K, committed 576K, reserved 1114112K
  class space    used 25K, committed 128K, reserved 1048576K

$ jcmd 21468 GC.class_histogram | head
 num     #instances         #bytes  class name (module)
   1:          2658       26322776  [B (java.base@21.0.4)          <- byte[] dominates
   2:           770          96192  java.lang.Class (java.base@21.0.4)
   3:          1082          70680  [Ljava.lang.Object; (java.base@21.0.4)
```
Reading: `O` (old) 80% and rising, `FGC=0` so far, but `CGC=6` concurrent cycles already happened (each triggered by humongous allocations) and reclaimed nothing. The histogram says WHAT (`byte[]`); the dump says WHO holds it. Outcome:
```
Dumping heap to leak.hprof ...
Heap dump file created [35510306 bytes in 0.038 secs]
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at Leak.main(Leak.java:6)                 <- the thread that hit the wall, not necessarily the culprit
```
and in `-Xlog:gc` the last lines (observed, see 2.11): `Pause Full (G1 Compaction Pause) 62M->62M(64M)` twice, then OOM. In MAT: dominator tree top = `java.util.ArrayList` with retained ~60 MB, "Path to GC Roots" = `class Leak -> CACHE -> elementData -> byte[]`.

## 3.4 StackOverflowError depth vs -Xss (`Soe.java`)

```java
public class Soe {
    static int depth = 0;
    static void r() { depth++; r(); }
    public static void main(String[] a) throws Exception {
        Thread t = new Thread(null, () -> {
            try { r(); } catch (StackOverflowError e) { System.out.println("depth=" + depth); }
        }, "t", 0);                      // stackSize 0 = use -Xss default
        t.start(); t.join();
    }
}
```
Observed (Windows, JDK 21):
```
-Xss256k depth=2596     -Xss512k depth=9595     -Xss1m depth=27501     -Xss4m depth=217843
no flag (default 1 MB)  depth=22817
```
Notes: depth is not linear in `-Xss`, and the same nominal 1 MB gave 22,817 (default) and 27,501 (explicit flag): interpreted frames are larger than C1/C2 frames, and compilation kicks in mid-recursion. There is no fixed answer to "how deep can I recurse"; use iteration when depth depends on data.

## 3.5 Object size measurement (`Size.java`), compressed oops on and off

```java
import java.lang.management.*;
public class Size {
    static class P1 { int a; }  static class P2 { int a; long b; }
    static class P3 { byte a; byte b; byte c; }  static class P4 { Object ref; int x; }
    static Object sink;
    static long measure(java.util.function.Supplier<Object> s) {
        var bean = (com.sun.management.ThreadMXBean) ManagementFactory.getThreadMXBean();
        long id = Thread.currentThread().threadId();
        for (int i = 0; i < 200_000; i++) sink = s.get();          // warm-up so the JIT compiles the allocation
        long before = bean.getThreadAllocatedBytes(id);
        int n = 100_000;
        for (int i = 0; i < n; i++) sink = s.get();                // static write => object escapes, not scalar-replaced
        return (bean.getThreadAllocatedBytes(id) - before) / n;
    }
    // main prints measure(Object::new), P1::new, P2::new, P3::new, P4::new, new byte[0/1/10], new int[3], new Object[2]
}
```
Observed bytes per allocation:

| Allocation | Default (compressed oops) | `-XX:-UseCompressedOops` | Explanation |
|---|---|---|---|
| `new Object()` | 16 | 16 | 12 header + 4 padding |
| class with `int a` | 16 | 16 | 12 + 4 |
| `int + long` | 24 | 24 | 12 + int back-fills gap 12..16 + long at 16..24 |
| 3 x `byte` | 16 | 16 | 12 + 3 + 1 padding |
| `Object ref + int` | 24 | 24 | 12+4+4 = 20 -> 24; with 8-byte ref: 12 + 4 (int) + 8 (ref) = 24 |
| `byte[0]` / `[1]` / `[10]` | 16 / 24 / 32 | same | 16 header+length; 17 -> 24; 26 -> 32 |
| `int[3]` | 32 | 32 | 16 + 12 = 28 -> 32 |
| `Object[2]` | **24** | **32** | 16 + 2x4 vs 16 + 2x8: the only row that changes |

This proves: the header stays 12 bytes even with `-UseCompressedOops` (compressed class pointers are separate) and references cost 4 vs 8 bytes.

## 3.6 Metaspace exhaustion via class-loader leak (`Meta.java`)

Idea: 20,000 times create a fresh `URLClassLoader` (parent = null), load a tiny class `Plugin` from `cls/`, instantiate it. In "leak" mode keep the loaders in a static list.
```java
URL u = Path.of("cls").toUri().toURL();
for (int i = 0; i < 20000; i++) {
    URLClassLoader cl = new URLClassLoader(new URL[]{u}, null);
    Class<?> c = cl.loadClass("Plugin");
    c.getDeclaredConstructor().newInstance();
    if (leak) LEAKED.add(cl);            // THE LEAK: loader (and all its classes) stay reachable
}
```
Observed with `-XX:MaxMetaspaceSize=16m`:
```
$ java -XX:MaxMetaspaceSize=16m Meta leak
i=0
i=2000
Exception in thread "main" java.lang.OutOfMemoryError: Metaspace
	at java.base/java.lang.ClassLoader.defineClass1(Native Method)
	at java.base/java.lang.ClassLoader.defineClass(ClassLoader.java:1027)

$ java -XX:MaxMetaspaceSize=16m Meta          (no leak)
i=18000
finished
```
Same code, same cap: the non-leaking loop completes because GC unloads unreachable loaders and their classes. This is Tomcat redeploy, Groovy scripts and plugin reload leaking. The OOM was thrown in `defineClass1` (native): the victim, not the retainer. In production use `jcmd <pid> VM.classloader_stats` (loader counts) and `-Xlog:class+unload=info`.

## 3.7 Class initialization: failure and deadlock (`Init.java`)

```java
static class A { static { sleep(100); new B(); } }
static class B { static { sleep(100); new A(); } }
static class Bad { static int v = 1 / Integer.parseInt("0"); }
// main: touch Bad twice; Class.forName("Nope"); start T-A (new A()) and T-B (new B()); after 1.5 s print states + Thread.print
```
Observed:
```
1st: java.lang.ExceptionInInitializerError
2nd: java.lang.NoClassDefFoundError: Could not initialize class Init$Bad cause=java.lang.ExceptionInInitializerError: Exception java.lang.ArithmeticException: / by zero [in thread "main"]
java.lang.ClassNotFoundException: Nope
B init start
A init start
states: RUNNABLE RUNNABLE                      <- "A init end" never printed: both are stuck
"T-A" ... java.lang.Thread.State: RUNNABLE
	at Init$A.<clinit>(Init.java:2)
	- waiting on the Class initialization monitor for Init$B
"T-B" ... java.lang.Thread.State: RUNNABLE
	at Init$B.<clinit>(Init.java:3)
	- waiting on the Class initialization monitor for Init$A
```
All three error kinds in one run, plus a deadlock that reports `RUNNABLE` with no "Found one Java-level deadlock" section. Senior answer: look for `<clinit>` frames and `Class initialization monitor`.

## 3.8 JIT warm-up (`Warm.java`) and compilation trace

```java
static int work(int x) { int s = 0; for (int i = 0; i < 1000; i++) s += (x ^ i) % 7; return s; }
// 8 rounds x 20,000 calls; print ms per round
```
Observed default: `round 0: 27.29 ms, round 1: 23.06, ... round 7: 24.94` (flat: JIT reached level 4 within milliseconds). With `-Xint`: `round 0: 239 ms, 226, 215` (**~10x slower**, pure interpreter).
`java -XX:+PrintCompilation Warm | grep Warm::work`:
```
     51   61 %     3       Warm::work @ 4 (28 bytes)          <- OSR (%) C1 with profiling: the loop went hot
     52   62       3       Warm::work (28 bytes)              <- normal C1 level 3
     52   63 %     4       Warm::work @ 4 (28 bytes)          <- OSR C2
     52   61 %     3       Warm::work @ 4 (28 bytes)   made not entrant   <- superseded
     53   64       4       Warm::work (28 bytes)              <- full C2 method
     55   62       3       Warm::work (28 bytes)   made not entrant
```
Columns: timestamp ms, compile id, flags (`%` OSR, `s` synchronized, `!` has exception handler, `n` native wrapper), tier, method, bytecode size. Honest caveat: with this trivial loop warm-up was invisible after round 0; real services (thousands of methods, class loading, inline caches, pools) need seconds to minutes.

---

# 4. Production war stories (symptom -> evidence -> root cause -> fix)

> These are composite, illustrative scenarios built from common real patterns; the tools and log shapes are real, the numbers are typical.

### Story 1: "Heap is only 60% used but the pod keeps restarting" (OOMKilled)
- **Symptom**: `kubectl get pods` shows restarts; `Last State: Terminated, Reason: OOMKilled, Exit Code: 137`. No stack trace, no heap dump. Dashboards show heap at 60% of `-Xmx`.
- **Evidence**: container limit 1 GiB, JVM `-Xmx1g`. `container_memory_working_set_bytes` climbs to 1 GiB. `jcmd <pid> VM.native_memory summary` (NMT enabled in staging): Heap 1024 MB reserved, Thread 180 MB (300 threads), Class 90 MB, Code 60 MB, Internal 150 MB.
- **Root cause**: `-Xmx` equal to the pod limit; non-heap memory (400+ MB) pushed RSS over the cgroup limit; the kernel sent SIGKILL. Not a Java OOM.
- **Fix**: `-XX:MaxRAMPercentage=70`, `-XX:MaxMetaspaceSize=256m`, bounded thread pools, `-XX:+ExitOnOutOfMemoryError`; alert on working-set/limit ratio, not heap alone.

### Story 2: p99 spikes every couple of minutes, GC log looks clean
- **Symptom**: API p99 jumps from 40 ms to 800 ms periodically; GC log says pauses < 20 ms.
- **Evidence**: `-Xlog:safepoint` showed safepoints where "Reaching safepoint" (time to safepoint) was hundreds of ms. A monitoring agent dumped thread stacks of thousands of threads on a schedule, and one thread ran a long counted loop with no safepoint poll.
- **Root cause**: time-to-safepoint plus VM operation; GC logs only show the GC portion.
- **Fix**: stop/limit the scraper, break the long loop (chunks, `long` counter or a call that has a poll), keep `-Xlog:safepoint` enabled. Lesson: correlate latency with safepoint logs, not just GC logs.

### Story 3: G1 Full GCs on a JSON service (humongous allocations)
- **Symptom**: 2-4 s Full GCs a few times an hour, heap 8 GB, average occupancy 40%.
- **Evidence**: log full of `Pause Young (Concurrent Start) (G1 Humongous Allocation)`, then `To-space exhausted`, then `Pause Full (G1 Compaction Pause)`. Region size 4 MB, so any array >= 2 MB is humongous; an endpoint serialized 5-10 MB payloads into a `ByteArrayOutputStream`.
- **Root cause**: humongous objects need contiguous regions; frequent large buffers fragment the heap, trigger early marking, and evacuation ran out of space.
- **Fix**: stream instead of buffering, chunk/pool buffers; stop-gap `-XX:G1HeapRegionSize=16m`. Verified by `Humongous regions:` staying low and Full GCs gone.

### Story 4: Metaspace OOM after the 30th redeploy (Tomcat)
- **Symptom**: `OutOfMemoryError: Metaspace` every ~10 hot redeploys of a WAR; a Tomcat restart "fixes" it.
- **Evidence**: `jcmd <pid> VM.classloader_stats` lists dozens of `WebappClassLoader` instances; heap dump path-to-GC-roots shows each retained through a `ThreadLocalMap` of a container pool thread; Tomcat logs "created a ThreadLocal with key of type ... but failed to remove it".
- **Root cause**: a webapp `ThreadLocal` value (class from the webapp) sat in a container thread that outlives the app: thread -> ThreadLocalMap -> value -> class -> loader -> all classes.
- **Fix**: `ThreadLocal.remove()` in `finally`, deregister JDBC drivers and stop app-started threads on `contextDestroyed`; in production avoid hot redeploy, use rolling pod replacement.

### Story 5: High CPU at 3 AM with no traffic
- **Symptom**: CPU 400% on a 4-core pod; throughput near zero.
- **Evidence**: `top -H` showed 4 hot threads; hex TID in `Thread.print` = `GC Thread#0..3`. GC log: repeated `Pause Full` with `62M->61M`-style results.
- **Root cause**: an unbounded in-memory queue (consumer stuck on a dead broker) filled the heap; GC threads burned CPU trying to reclaim nothing.
- **Fix**: bounded queue with back-pressure, consumer timeouts, alert on old-gen-after-GC trend and GC time rate. Lesson: hot GC threads = a memory problem.

### Story 6: Slow for two minutes after each deploy
- **Symptom**: new pods get traffic and p99 is 1.5 s for ~2 min, then 60 ms; HPA scale-out worsens it.
- **Evidence**: compile queue long, C2 threads busy; `limits.cpu: 1` so JIT threads competed with request threads (high CFS throttling metric).
- **Root cause**: cold JIT + CPU quota + traffic sent immediately.
- **Fix**: readiness that waits for a warm-up routine, more CPU during startup (raise limit / rely on requests), AppCDS for faster class loading, slow-ramp canary; `-XX:TieredStopAtLevel=1` only for short-lived jobs.

### Story 7: `NoClassDefFoundError` flood
- **Symptom**: thousands of `NoClassDefFoundError: Could not initialize class com.x.Config` per minute.
- **Evidence**: searching the logs backwards found one `ExceptionInInitializerError` at startup (env var missing in a `static final` initializer).
- **Root cause**: a failed `<clinit>` poisons the class for the life of its loader.
- **Fix**: fix config and restart; no fallible I/O in static initializers; lazy init with retry.

---

# 5. Interview questions (50), graded, with follow-up chains

Legend: **E** easy, **M** medium, **H** hard. "Wrong answer" lines list what candidates commonly say and why it fails.

## Architecture and class loading

**Q1 (E). Name the JVM runtime data areas and which are per-thread.**
Per-thread: Java stack, PC register, native method stack (and TLAB in heap). Shared: heap, Metaspace, code cache (plus direct memory outside the model). 
*Follow-up*: Which one has no `OutOfMemoryError`? -> PC register. *Follow-up*: Where do static variables live? -> In the heap (in the `Class` mirror object); metadata in Metaspace.
Wrong answer: "Metaspace is part of the heap" (native memory) / "PermGen still exists" (removed in 8).

**Q2 (E). What replaced PermGen and how does it differ?**
Metaspace (JDK 8), native memory, grows dynamically up to `MaxMetaspaceSize` (unlimited by default). Class metadata only; strings and statics moved to heap (7). 
*Follow-up*: Does `MetaspaceSize` reserve memory? No, it's the threshold that first triggers a metadata GC. *Follow-up*: Why set `MaxMetaspaceSize` in containers? Leak fails fast with an OOM/heap dump instead of an OOMKilled.

**Q3 (E). Explain class loading phases.**
Loading, linking (verify, prepare, resolve), initialization (`<clinit>`). Prepare sets defaults; initialization runs initializers in source order. 
*Follow-up*: When is a class initialized? `new`, static method/field (non-constant), `Class.forName`, subclass init, main. `static final int X=5` does not trigger init.

**Q4 (M). What is parent delegation and why does it exist?**
Loader asks parent first; guarantees core classes can't be spoofed and each class has a single identity per name in the hierarchy. 
*Follow-up*: How do you break it and why would you? Override `loadClass` (child-first): app servers for per-webapp library versions, OSGi, plugin systems. *Follow-up*: Consequence? Same class name from two loaders -> `ClassCastException` between "identical" classes, extra Metaspace.
Wrong answer: "delegation is for speed" (it is for security/consistency).

**Q5 (M). `ClassNotFoundException` vs `NoClassDefFoundError`.**
CNFE: checked exception from explicit dynamic loading when the class can't be found. NCDFE: an `Error`; class was present at compile time but is missing at runtime **or** its static init previously failed. 
*Follow-up*: How do you distinguish the second NCDFE cause? Message "Could not initialize class X"; search logs for the earlier `ExceptionInInitializerError` (demo 3.7).

**Q6 (M). What is `ExceptionInInitializerError` and what happens on the second access?**
Wraps an unchecked exception from a static initializer; class becomes erroneous; every later use throws `NoClassDefFoundError` until the loader is discarded.
*Follow-up*: How would you make static init safer? Lazy holder idiom, no I/O, catch and log with context.

**Q7 (H). Describe how a Metaspace leak happens on redeploy and how you would prove it.**
Old `WebappClassLoader` kept reachable by a long-lived thread/ThreadLocal/static/JDBC driver/shutdown hook; all its classes stay. Prove: `VM.classloader_stats` shows loader count growing; heap dump: path-to-GC-roots for old `ClassLoader` instances excluding weak refs; `-Xlog:class+unload`. 
*Follow-up*: Why is the OOM thread not the culprit? The thread that needs new metadata dies; retainer is elsewhere. *Follow-up*: Fix? Remove ThreadLocals, deregister drivers, stop threads, or avoid hot redeploy.

**Q8 (H). Can two threads deadlock during class initialization? Will jstack show it?**
Yes (A's `<clinit>` needs B, B's needs A on different threads). `Thread.print` shows both `RUNNABLE`, "waiting on the Class initialization monitor" and does not print "Found one Java-level deadlock". 
*Follow-up*: Prevention? Avoid cycles between static initializers; no thread start in `<clinit>`.

**Q9 (M). When are classes unloaded?**
Only when their defining loader is unreachable; happens during GC (G1 at remark with `ClassUnloadingWithConcurrentMark`, Full GC, ZGC concurrently). Never for bootstrap/app loader. 
*Follow-up*: Do lambdas/proxies create classes? Yes (hidden classes/proxy classes), unloadable with their loader; endless dynamic generation without caching is a Metaspace leak.

## Object layout

**Q10 (E). What is in an object header on a 64-bit HotSpot?**
Mark word (8 bytes: hash, age, lock bits) + compressed klass pointer (4) = 12 bytes; arrays add a 4-byte length. Objects are 8-byte aligned.
Wrong answer: "16 bytes always" (it is 12 with compressed class pointers; total minimum object is 16 due to alignment). JDK 24 compact headers make it 8.

**Q11 (M). What are compressed oops and what is the 32 GB rule?**
4-byte references shifted by 3 (objects 8-byte aligned) address 32 GB. Above ~32 GB refs become 8 bytes: usable capacity can shrink. Stay <= ~31 GB, or use `ObjectAlignmentInBytes=16` (64 GB) or go ZGC (no compressed oops anyway). 
*Follow-up*: Do compressed oops affect the header? Klass pointers are compressed independently (JDK 15+); demo 3.5.

**Q12 (M). How many bytes is `new Order()` with `long id; int qty; Object customer; boolean paid`?**
12 header + qty 4 (back-fills) + id 8 + customer 4 + paid 1 = 29 -> padded to 32. 
*Follow-up*: `String "hello"`? String 24 + byte[5] 24 = 48. *Follow-up*: How verify? JOL `ClassLayout` or `getThreadAllocatedBytes`.

**Q13 (H). Why is `List<Integer>` with 1M elements ~5x bigger than `int[]`?**
Array of 4-byte refs (4 MB) + a 16-byte `Integer` per value (16 MB) = ~20 MB vs 4 MB; plus GC has 1M more objects to mark/copy and poorer cache locality. Fix: primitive collections (fastutil, Eclipse Collections) or `int[]`.

## Stack, JIT, strings

**Q14 (E). What causes `StackOverflowError` and can you fix it with `-Xss`?**
Unbounded/too-deep recursion. `-Xss` can postpone it but costs memory per thread; fix root cause (base case, iteration, cycle in `toString`/JSON). 
*Follow-up*: Does StackOverflow leave the JVM broken? Usually recoverable per thread; but state may be inconsistent (locks/finally blocks).

**Q15 (M). What is in a stack frame? Why can recursion depth vary between runs?**
Local variable array, operand stack, frame data. Interpreter frames are larger than JIT-compiled ones, so depth changes as methods get compiled (demo 3.4: 22,817 vs 27,501 at nominally 1 MB).

**Q16 (M). Explain TLAB and why `new` is fast.**
Each thread bump-allocates in its own Eden chunk without synchronization; refill via CAS when exhausted. Big arrays may bypass TLAB. 
*Follow-up*: Are objects ever stack-allocated? No; C2 escape analysis can scalar-replace non-escaping objects (no allocation at all).

**Q17 (H). What is scalar replacement and when does it not happen?**
C2 replaces a non-escaping object with its fields in registers. Not when the object escapes (stored to field/static, passed to non-inlined call, returned), in interpreter/C1, when inlining fails (big methods), or with merge points of different objects.
*Follow-up*: Why does it matter for benchmarks? Result discarded => the whole allocation vanishes; use JMH `Blackhole`.

**Q18 (M). Where is the String pool and what does `intern()` do?**
String table (native hash table) referencing heap objects. `intern()` returns the canonical instance, adding it if absent. Use for bounded sets; not for unbounded input. `"a"+"b"` constant is pooled at compile time; runtime concatenation isn't.
*Follow-up*: `new String("x") == "x"`? false. `new String("x").intern() == "x"`? true.

**Q19 (M). What are Compact Strings and String Deduplication?**
Compact: `byte[]` + coder, Latin-1 strings use 1 byte/char (JDK 9, on by default). Dedup: (`-XX:+UseStringDeduplication`, off by default) GC merges the backing `byte[]` of equal, older strings; does not merge String objects or change `==`.
*Follow-up*: Difference between dedup and intern? Intern shares the String object, explicit and synchronous; dedup shares only the array, automatic and concurrent.

**Q20 (M). Explain tiered compilation.**
Interpreter (level 0) profiles; hot methods go to C1 (level 3 with full profiling), then C2 (level 4) using profile. Thresholds ~200 invocations for C1, ~5000 for C2, loops use back-edge counters and OSR. 
*Follow-up*: Why are first requests slow? Interpreter/C1 code and cold caches; compile threads compete for CPU.

**Q21 (H). What is deoptimization? Give an example.**
Speculative optimizations (monomorphic inline, class-hierarchy assumptions, unreached branches) are guarded; when a guard fails (new subclass loaded, unexpected type) compiled code is invalidated ("made not entrant") and execution continues in the interpreter, then recompiles. 
*Follow-up*: Symptoms? Periodic latency blips, `-Xlog:deoptimization=debug`, `PrintCompilation` "made not entrant" storms.

**Q22 (M). Why do microbenchmarks lie? What do you do?**
Warm-up, dead-code elimination, constant folding, OSR effects, GC noise, profile pollution. Use JMH with forks and warm-up iterations, blackholes, look at percentiles. 
Wrong answer: "just loop a million times and time it with `System.currentTimeMillis`".

**Q23 (M). What if the code cache fills up?**
JIT stops compiling ("CodeCache is full. Compiler has been disabled"); app continues at interpreter speed (~10x slower for hot paths, demo 3.8). Check `jcmd Compiler.codecache`; raise `ReservedCodeCacheSize`, fix dynamic class generation.

## GC fundamentals

**Q24 (E). When is an object eligible for GC?**
When unreachable from GC roots (thread stacks, statics, JNI, etc.), even in a cycle. Not "when set to null" necessarily, and not when unused-but-reachable (that is a leak).
*Follow-up*: Is `System.gc()` guaranteed? No; it is a hint (a full GC unless `-XX:+DisableExplicitGC`/`ExplicitGCInvokesConcurrent`).

**Q25 (M). Explain the generational hypothesis and object promotion.**
Most objects die young, so collect young often (copying, cost ~ survivors), promote after surviving enough young GCs (tenuring threshold, adaptive; max 15). Premature promotion (survivor too small, bursts) fills old gen with short-lived garbage and causes Full GCs.

**Q26 (M). Mark-sweep vs mark-compact vs copying.**
Sweep: in-place, fragmentation. Compact: slides live objects, no fragmentation, touches all live data. Copying: cost ~ live data, needs to-space, natural compaction; used for young gen.

**Q27 (H). How does young GC find old-to-young references without scanning old gen?**
Card table (512-byte cards) dirtied by a post-write barrier on reference stores; young GC scans only dirty cards. G1 additionally keeps per-region remembered sets built by concurrent refinement. 
*Follow-up*: Cost? Barrier on every reference store; why G1/ZGC throughput trails Parallel.

**Q28 (H). What is a safepoint and time-to-safepoint? Why can pauses exceed GC log times?**
Point where threads can be stopped to run VM operations (GC, deopt, dumps). Threads stop only at polls; TTSP is the wait for the slowest thread. User-visible pause = TTSP + operation; GC logs show only the operation. Use `-Xlog:safepoint`.

**Q29 (H). What is SATB in G1?**
Snapshot-at-the-beginning: a pre-write barrier records the overwritten reference during concurrent marking so anything alive at cycle start is marked, even if the mutator drops it; guarantees correctness of concurrent marking at the cost of floating garbage until next cycle.

## Collectors

**Q30 (E). Which collector is default in JDK 8 / 9+ / in a 1-CPU container?**
Parallel in 8; G1 from 9 on server-class machines; **Serial** when the JVM sees < 2 CPUs or < ~1792 MB memory. 
*Follow-up*: Trap? `limits.cpu: 1` in Kubernetes makes ergonomics choose Serial: check the first GC log line.

**Q31 (M). Explain how G1 works.**
Regions (1-32 MB), young collections evacuate eden+survivor within a pause goal (200 ms default); concurrent marking starts at IHOP (45%, adaptive); then mixed collections reclaim the most garbage-rich old regions; Full GC only as a fallback. Humongous objects (>= half region) get contiguous regions.
*Follow-up*: Is `MaxGCPauseMillis` a guarantee? No, a target; too low means smaller young gen, more frequent GCs, lower throughput.

**Q32 (H). Walk through G1's concurrent marking cycle phases and where the pauses are.**
Concurrent Start (STW, with a young GC), Root Region Scan (conc.), Concurrent Mark (conc., SATB), Remark (STW), Cleanup (STW, reclaims empty regions), Concurrent Cleanup / rebuild remembered sets; then Mixed GCs. Log lines `Pause Remark/Cleanup` vs `Concurrent Mark Cycle`.

**Q33 (H). What is "To-space exhausted" / evacuation failure and how do you fix it?**
No free regions to copy survivors into; objects are left in place, pause is long, often followed by Full GC. Causes: heap too small, marking too late, humongous churn, huge promotion burst. Fixes: larger heap, lower/adaptive IHOP, larger `G1ReservePercent`, reduce allocation/humongous use.

**Q34 (H). What are humongous objects and why are they a problem?**
Objects >= 1/2 region are allocated in dedicated contiguous regions outside the normal young/old flow; need contiguous space (fragmentation), trigger early marking, aren't copied. Diagnose with `Humongous Allocation` cause; fix via larger region size or avoid big arrays (demo 3.1, story 3).

**Q35 (M). When would you choose Parallel over G1?**
Batch/throughput jobs where pause length is irrelevant, no latency SLO, cores available; smaller barrier overhead. Also when the app is tiny and you want simplicity. 
Wrong answer: "Parallel is obsolete". It's supported and default for many batch setups.

**Q36 (H). How does ZGC achieve sub-millisecond pauses?**
Colored pointers + load barriers let it relocate objects concurrently: mark and relocate phases run alongside the app; pointers are healed lazily on load. STW phases only scan roots (independent of heap size). Generational ZGC (`-XX:+ZGenerational` in 21) collects young separately, improving throughput and reducing stalls.
*Follow-up*: Downsides? Extra CPU/throughput cost, no compressed oops (bigger footprint), need headroom, allocation stalls if outpaced (demo 3.1).

**Q37 (M). ZGC vs Shenandoah?**
Both concurrent compacting. ZGC: colored pointers, load barrier, no compressed oops, Oracle/OpenJDK. Shenandoah: load-reference barrier, supports compressed oops, in OpenJDK builds (Red Hat), not Oracle JDK. Both target pauses ~1-10 ms regardless of heap size.

**Q38 (M). Which flags do you set first for a Spring Boot service on JDK 21 in K8s?**
`-XX:MaxRAMPercentage=70..75` (or `-Xmx`), `-XX:+ExitOnOutOfMemoryError`, `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=...`, `-XX:MaxMetaspaceSize`, `-Xlog:gc*:file=...` rotated, keep G1 default. Add ZGC only with latency data.

## Logs, tuning, diagnosis

**Q39 (M). Read this line: `GC(37) Pause Full (G1 Compaction Pause) 62M->62M(64M) 4.437ms`.**
Full GC in G1; heap before 62M, after 62M, capacity 64M: **nothing reclaimed** -> live set ~= heap -> leak or heap too small; the OOM is imminent. 
*Follow-up*: What if it were `Pause Full (System.gc())`? Explicit call (RMI, library); disable or make concurrent.

**Q40 (M). What does `Pause Young (Allocation Failure)` mean?**
Young space full (Serial/Parallel). It is a normal cause, not an error.

**Q41 (M). Metrics you look at for GC health?**
p99/max pause, GC time % (throughput), allocation rate, promotion rate, live set after Full GC/mixed, Full GC count, and (ZGC) allocation stalls. Compute from `-Xlog:gc*`, JFR, or `jvm_gc_pause_seconds`.

**Q42 (H). Give your tuning methodology.**
Set SLO, measure under realistic load with warm-up discarded, compute the metrics above, form a hypothesis from evidence (e.g. high promotion -> survivor sizing/caching), change one variable, re-measure. Prefer reducing allocation to flag tuning; remove stale flags on upgrades.
Wrong answer: "copy the flags from a blog" / "always set Xms=Xmx and NewRatio".

**Q43 (M). `-Xms` equals `-Xmx`: why?**
Avoid heap resize pauses and lazy commit surprises; predictable RSS. Downside: reserves memory even if idle (dense clusters); use `AlwaysPreTouch` only if page-fault stalls matter.

**Q44 (H). Diagnose a memory leak from scratch.**
Confirm trend of post-GC old gen (`jstat -gcutil`, GC log). Two `GC.class_histogram` samples to see growing classes. Capture a dump (`GC.heap_dump` or on OOM). MAT: leak suspects and dominator tree, path-to-GC-roots (exclude weak/soft). Fix and verify sawtooth flattening. JFR Old Object Sample for allocation stacks. 
*Follow-up*: Dump is 10 GB and the pod is tiny? Use JFR `OldObjectSample`, `class_histogram` diff, or dump to a volume then analyse elsewhere; `-XX:+HeapDumpOnOutOfMemoryError` with `emptyDir` sized appropriately.

**Q45 (H). Java OOM vs Kubernetes OOMKilled: explain and diagnose.**
Java OOM: JVM-level pool exhausted, exception, dump possible. OOMKilled: kernel kills the container as RSS exceeds the limit: SIGKILL, exit 137, no dump. RSS = heap + metaspace + thread stacks + code cache + GC structures + direct + native/malloc. Diagnose: `kubectl describe pod`, working-set metrics, NMT summary/diff, thread count, direct buffer metrics, glibc arenas. Fix: `MaxRAMPercentage` < 100 (typically 50-75), cap metaspace/direct, bound threads, `MALLOC_ARENA_MAX`.
*Follow-up*: Why does heap 60% but pod dies? Committed vs used, and non-heap. *Follow-up*: Why is page cache a factor? Working set includes file cache that cannot be reclaimed fast enough in some cases.

**Q46 (H). What are the OOM variants and their fixes?**
See table in 2.13: heap, GC overhead limit (Parallel), Metaspace, direct buffer, unable to create native thread, array size limits. Key: identify which pool from the message first; the wrong tool wastes hours.

**Q47 (M). Soft, weak, phantom references and `Cleaner`?**
Soft: cleared under memory pressure (caches, but prefer bounded caches). Weak: cleared at next GC (`WeakHashMap`). Phantom: `get()` null, enqueued after finalization; basis of `Cleaner` for native resource cleanup. `finalize()` deprecated for removal (JDK 18); slow, unpredictable, needs an extra GC cycle.

**Q48 (M). CDS/AppCDS and native image trade-offs?**
CDS maps pre-parsed core classes; AppCDS adds application classes (`-XX:ArchiveClassesAtExit`, `-XX:SharedArchiveFile`, JDK 19+ `-XX:+AutoCreateSharedArchive`): 30-50% faster startup. Native image: ms startup and low RSS, but closed world/reflection config, lower peak throughput without PGO, no JIT/dynamic tooling.

**Q49 (H). High CPU: walk through your steps.**
`top -H -p pid` -> hex TID -> `jcmd Thread.print` x3 -> match `nid=0x..` -> classify: app code (profile with async-profiler), GC threads (memory problem), compiler threads (JIT storm), spin/contention (BLOCKED/lock owner). 
*Follow-up*: Thread is RUNNABLE but CPU 0? Blocked in native I/O; RUNNABLE != on-CPU.

**Q50 (H). How do you read a deadlock in a thread dump and prevent it?**
`Found one Java-level deadlock`: each thread `waiting to lock` a monitor held by another (cycle). Prevent: global lock ordering, `tryLock(timeout)`, narrower critical sections, fewer locks, concurrent structures. Also detect via `ThreadMXBean.findDeadlockedThreads()`; note class-init deadlocks are invisible to it.

## Common wrong answers (quick list)
- "Objects are allocated on the stack when they don't escape." (No: scalar replaced, or heap.)
- "Metaspace is unlimited so leaks are harmless." (Unlimited until the OS/container kills it.)
- "Set `-Xmx` to the container memory." (OOMKilled.)
- "Major GC = Full GC = old only" (terminology is inconsistent; G1 has mixed collections, Full is a fallback.)
- "Full GC is the same as `System.gc()`" (it is one possible cause.)
- "`finalize()` is a destructor." (Deprecated for removal; not deterministic.)
- "`jmap -dump` is free" (STW, forces Full GC with `live`.)
- "More GC threads always faster" (contention, container CPU limit).
- "G1's `MaxGCPauseMillis=50` guarantees 50 ms." (Target only.)
- "ZGC has no pauses." (Sub-ms STW phases plus possible allocation stalls.)
- "`WeakReference` is good for a cache." (Cleared at next GC: use bounded Caffeine.)

---

# 6. One-page cheat sheet

```
MEMORY MAP     heap(objects) | metaspace(class meta, native) | stacks(per thread) | code cache | direct | native
RSS (pod)      = heap + metaspace + threads*stack + code cache + GC data + direct + malloc/native   -> Xmx < limit!
OBJECT         header 12 (mark 8 + klass 4), arrays +4 length, align 8, compressed oops <= ~31 GB (4-byte refs)
STRING         48 bytes for "hello" (24 + 24); compact strings Latin-1; intern = shared object; dedup = shared byte[]
CLASS LOADING  load -> link(verify, prepare, resolve) -> init(<clinit>, once, lazy); parent-first; Tomcat child-first
ERRORS         CNFE=dynamic load failed | NCDFE=missing at runtime or earlier init failed | EIIE=static init threw
JIT            interp -> C1(L3 profile) -> C2(L4); ~200 / ~5000 invocations; OSR for loops; deopt = "made not entrant"
GC ROOTS       stacks, statics, JNI, active threads, class loaders
COLLECTORS     Serial(1 thread, tiny) | Parallel(throughput) | G1(default, regions, pause goal 200 ms)
               ZGC(-XX:+UseZGC [-XX:+ZGenerational in 21], <1 ms pauses, colored ptrs, load barrier, needs headroom)
               Shenandoah(OpenJDK builds, concurrent evac)   CMS removed in 14
G1             IHOP 45% adaptive | humongous >= 1/2 region | mixed GCs | evac failure = To-space exhausted | Full GC = red flag
ERGONOMICS     <2 CPU or <~1.8 GB -> Serial | MaxRAMPercentage default 25 | cpu limit -> availableProcessors
ALERT SIGNALS  post-GC old-gen rising | Full GC not reclaiming | Allocation Stall | Metadata GC Threshold | long safepoint
```

**Flags cheat sheet (all verified on JDK 21)**

| Purpose | Flag |
|---|---|
| Heap size | `-Xms -Xmx`, `-XX:MaxRAMPercentage=75`, `-XX:InitialRAMPercentage`, `-XX:MinRAMPercentage` |
| Collector | `-XX:+UseSerialGC`, `-XX:+UseParallelGC`, `-XX:+UseG1GC`, `-XX:+UseZGC -XX:+ZGenerational`, `-XX:+UseShenandoahGC` (OpenJDK builds) |
| G1 | `-XX:MaxGCPauseMillis=200`, `-XX:InitiatingHeapOccupancyPercent=45`, `-XX:G1HeapRegionSize=16m`, `-XX:G1ReservePercent=10`, `-XX:G1MixedGCCountTarget=8`, `-XX:G1HeapWastePercent=5` |
| Young sizing (Serial/Parallel) | `-Xmn`, `-XX:NewRatio=2`, `-XX:SurvivorRatio=8`, `-XX:MaxTenuringThreshold=15` |
| ZGC | `-XX:SoftMaxHeapSize`, `-XX:ZCollectionInterval`, `-XX:+ZUncommit` (default) |
| Metaspace/class | `-XX:MetaspaceSize`, `-XX:MaxMetaspaceSize=256m`, `-XX:CompressedClassSpaceSize` |
| Stack | `-Xss1m` (`-XX:ThreadStackSize`) |
| Direct memory | `-XX:MaxDirectMemorySize` |
| Code cache/JIT | `-XX:ReservedCodeCacheSize`, `-XX:TieredStopAtLevel=1`, `-XX:+PrintCompilation`, `-XX:CICompilerCount` |
| OOM handling | `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps`, `-XX:+ExitOnOutOfMemoryError`, `-XX:+CrashOnOutOfMemoryError` |
| Logging | `-Xlog:gc*:file=gc.log:time,uptime,level,tags:filecount=5,filesize=20m`, `-Xlog:safepoint`, `-Xlog:class+unload`, `-Xlog:stringdedup*` |
| Native memory | `-XX:NativeMemoryTracking=summary` (or `detail`) |
| Containers | `-XX:+UseContainerSupport` (Linux, default on), `-XX:ActiveProcessorCount=N` |
| Misc | `-XX:+AlwaysPreTouch`, `-XX:+DisableExplicitGC`, `-XX:+ExplicitGCInvokesConcurrent`, `-XX:+UseStringDeduplication`, `-XX:-UseCompressedOops`, `-XX:ObjectAlignmentInBytes=16` |
| Startup | `-XX:ArchiveClassesAtExit=app.jsa`, `-XX:SharedArchiveFile=app.jsa`, `-XX:+AutoCreateSharedArchive` |
| Diagnostics | `-XX:+PrintFlagsFinal` (run `java -XX:+PrintFlagsFinal -version` and grep for the flag) |

**Tool cheat sheet**

```
jcmd -l | jps -lv                           list JVMs
jcmd <pid> GC.heap_info                     heap summary per collector
jcmd <pid> GC.class_histogram               instances/bytes per class (diff two samples)
jcmd <pid> GC.heap_dump /d/h.hprof          heap dump (live objects only; -all for everything)
jcmd <pid> Thread.print [-l]                thread dump (jstack equivalent); Thread.dump_to_file -format=json for virtual threads
jcmd <pid> VM.flags | VM.system_properties | VM.uptime | VM.info
jcmd <pid> VM.native_memory summary|baseline|summary.diff       (needs -XX:NativeMemoryTracking)
jcmd <pid> VM.classloader_stats             loaders and their class counts (Metaspace leak)
jcmd <pid> Compiler.codecache               JIT code cache usage
jcmd <pid> JFR.start duration=60s filename=r.jfr settings=profile     then open in JDK Mission Control
jstat -gcutil <pid> 1000                    S0 S1 E O M CCS YGC YGCT FGC FGCT CGC CGCT GCT (each second)
jmap -histo:live <pid> | jmap -dump:live,format=b,file=h.hprof <pid>   (live forces a Full GC)
asprof -e cpu|alloc -d 30 -f out.html <pid> async-profiler flame graphs
top -H -p <pid>; printf "%x" <tid>; grep "nid=0x.." dump     hot thread -> stack
Eclipse MAT: Leak Suspects, Dominator Tree, Path to GC Roots (exclude weak/soft), OQL
```

**Triage table**

| Symptom | First command | Likely cause |
|---|---|---|
| Pod restarts, exit 137, no stack trace | `kubectl describe pod`, NMT | RSS > limit (non-heap) |
| `OutOfMemoryError: Java heap space` | heap dump on OOM, MAT | leak or undersized heap |
| `OutOfMemoryError: Metaspace` | `VM.classloader_stats` | loader leak/dynamic classes |
| p99 spikes, GC log clean | `-Xlog:safepoint` | TTSP, safepoint ops |
| CPU high in `GC Thread` | GC log after-values | leak / heap too small |
| Slow after deploy for minutes | PrintCompilation, CPU throttling | JIT warm-up + CPU quota |
| Repeated `Humongous Allocation` | GC log region size | big arrays: region size / chunking |
| `NoClassDefFoundError: Could not initialize` | logs backwards | earlier `ExceptionInInitializerError` |
| Threads stuck `RUNNABLE` in `<clinit>` | `Thread.print` | class-init deadlock |
| App fine on laptop, slow in pod | first line of GC log | Serial chosen (1 CPU), or 25% default heap |

**Last-minute soundbites**
- "Java OOM is the JVM saying no; OOMKilled is the kernel saying no."
- "Pause time scales with what survives, not with what dies."
- "G1 Full GC is a design failure signal, not a normal event."
- "ZGC trades a little throughput and headroom for heap-size-independent pauses."
- "Measure first: p99 pause, GC %, allocation rate, promotion rate; change one thing."
