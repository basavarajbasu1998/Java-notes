# Java 8 — Lambda & Stream API Interview Q&A (5–6 Years Experience)

---

## 1. Lambda Expressions

A lambda is an **anonymous function** — a block of code with parameters that can be passed around like a value. It implements a **functional interface** (an interface with exactly one abstract method).

```java
// Before Java 8
Runnable r1 = new Runnable() {
    public void run() { System.out.println("Hello"); }
};

// With lambda
Runnable r2 = () -> System.out.println("Hello");
```

Syntax forms:
```java
() -> expression
(x) -> expression
(x, y) -> { statement; return value; }
```

**Interview line:** "A lambda is syntactic sugar over an anonymous class implementing a functional interface — but unlike anonymous classes, it doesn't create a new `.class` file per lambda; the JVM uses `invokedynamic` and `LambdaMetafactory` under the hood, which is more efficient and doesn't bloat the classpath."

---

## 2. Functional Interfaces

An interface with **exactly one abstract method** (SAM — Single Abstract Method). Can have any number of `default` and `static` methods. Marked (optionally, but recommended) with `@FunctionalInterface` — this annotation makes the compiler enforce the single-abstract-method rule.

```java
@FunctionalInterface
interface Calculator {
    int calculate(int a, int b);
    default void log() { System.out.println("Calculating..."); } // allowed
}

Calculator add = (a, b) -> a + b;
```

Built-in examples: `Runnable`, `Callable`, `Comparator`, `Predicate`, `Function`, `Consumer`, `Supplier`.

**Interview line:** "`@FunctionalInterface` isn't mandatory but it's a compile-time safety net — if someone accidentally adds a second abstract method, the build breaks instead of failing silently at lambda call sites."

---

## 3. Predicate

`Predicate<T>` — takes an argument, returns a **boolean**. Used for filtering/testing conditions.

```java
Predicate<Integer> isEven = n -> n % 2 == 0;
isEven.test(4);  // true

// Combinators
Predicate<Integer> isPositive = n -> n > 0;
isEven.and(isPositive).test(4);   // true
isEven.or(isPositive).test(-3);   // true
isEven.negate().test(4);          // false
```

Method signature: `boolean test(T t)`

---

## 4. Consumer

`Consumer<T>` — takes an argument, returns **nothing** (void). Used for side effects like printing, logging, saving.

```java
Consumer<String> print = s -> System.out.println(s);
print.accept("Hello");

Consumer<String> printUpper = s -> System.out.println(s.toUpperCase());
print.andThen(printUpper).accept("hi");  // runs both in sequence
```

Method signature: `void accept(T t)`

---

## 5. Supplier

`Supplier<T>` — takes **no argument**, returns a value. Used for lazy generation/deferred computation.

```java
Supplier<Double> randomValue = () -> Math.random();
randomValue.get();

// Common use: lazy default in Optional
Optional<String> opt = Optional.empty();
opt.orElseGet(() -> computeExpensiveDefault());  // only called if empty
```

Method signature: `T get()`

**Interview line:** "Supplier is key for laziness — `orElseGet(Supplier)` only invokes the supplier if needed, unlike `orElse(value)` which is always eagerly evaluated even if the Optional is present."

---

## 6. Function

`Function<T, R>` — takes an argument of type T, returns a result of type R. Used for transformation.

```java
Function<String, Integer> length = s -> s.length();
length.apply("hello");  // 5

// Composition
Function<Integer, Integer> addOne = n -> n + 1;
Function<Integer, Integer> square = n -> n * n;

addOne.andThen(square).apply(3);  // (3+1)^2 = 16 → applies addOne first, then square
addOne.compose(square).apply(3);  // 3^2 + 1 = 10 → applies square first, then addOne
```

Method signature: `R apply(T t)`

**Interview line — andThen vs compose:** "`andThen` runs the current function first, then the argument function. `compose` runs the argument function first, then the current one. Easy way to remember: compose reads right-to-left like mathematical function composition, f(g(x))."

---

## 7. BiFunction / BiPredicate / BiConsumer

Two-argument versions of the above:

```java
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;
add.apply(3, 4);  // 7

BiPredicate<String, Integer> lengthCheck = (s, len) -> s.length() == len;
lengthCheck.test("hello", 5);  // true

BiConsumer<String, Integer> printPair = (k, v) -> System.out.println(k + "=" + v);
map.forEach(printPair);  // Map.forEach expects a BiConsumer<K,V>
```

Common real use: `Map.forEach()`, `Map.compute()`, `Map.merge()` all use Bi-variants.

---

## 8. Method References

Shorthand for a lambda that just calls an existing method. Four types:

```java
// 1. Static method reference
Function<String, Integer> f1 = Integer::parseInt;         // s -> Integer.parseInt(s)

// 2. Instance method on a particular object
String prefix = "Hello ";
Function<String, String> f2 = prefix::concat;              // s -> prefix.concat(s)

// 3. Instance method on an arbitrary object (of a particular type) — most common in streams
Function<String, Integer> f3 = String::length;              // s -> s.length()
list.stream().map(String::toUpperCase);

// 4. Constructor reference
Supplier<ArrayList<String>> f4 = ArrayList::new;            // () -> new ArrayList<>()
Function<String, StringBuilder> f5 = StringBuilder::new;    // s -> new StringBuilder(s)
```

**Interview line:** "The trickiest one to explain is #3 — 'instance method on an arbitrary object of a particular type,' like `String::length` used inside `map()`. The compiler figures out that the first parameter supplied by the stream becomes the object the method is invoked ON, not passed as an argument."

---

## 9. Stream API

A **Stream** is a sequence of elements supporting functional-style operations — **not a data structure**, it doesn't store elements; it pulls from a source (Collection, array, I/O channel) and processes them through a pipeline.

```java
List<String> names = List.of("John", "Jane", "Adam");
long count = names.stream()
                   .filter(n -> n.startsWith("J"))
                   .count();
```

Key properties:
- **Doesn't modify the source.**
- **Lazy** — nothing executes until a terminal operation is invoked.
- **Single-use** — once consumed (terminal op called), a stream **cannot be reused**; calling another operation on it throws `IllegalStateException: stream has already been operated upon or closed`.
- Can be **sequential** or **parallel**.

Pipeline structure: `Source → intermediate ops (0 or more) → terminal op (exactly 1)`

---

## 10. filter()

**Intermediate** operation. Takes a `Predicate<T>`, keeps only elements that match.

```java
List<Integer> evens = numbers.stream()
                              .filter(n -> n % 2 == 0)
                              .collect(Collectors.toList());
```

---

## 11. map()

**Intermediate** operation. Takes a `Function<T, R>`, transforms each element **1-to-1** into a new value/type.

```java
List<Integer> lengths = names.stream()
                              .map(String::length)
                              .collect(Collectors.toList());
```

---

## 12. flatMap()

**Intermediate** operation. Used when each element maps to **multiple values / a stream of values** (e.g., List<List<T>>), and you want to **flatten** them into a single stream.

```java
List<List<Integer>> nested = List.of(List.of(1,2,3), List.of(4,5), List.of(6));

List<Integer> flat = nested.stream()
                            .flatMap(List::stream)      // Stream<List<Integer>> -> Stream<Integer>
                            .collect(Collectors.toList());
// [1, 2, 3, 4, 5, 6]
```

Real use case: splitting sentences into words.
```java
List<String> sentences = List.of("hello world", "java streams");
List<String> words = sentences.stream()
                               .flatMap(s -> Arrays.stream(s.split(" ")))
                               .collect(Collectors.toList());
// [hello, world, java, streams]
```

---

## 13. distinct()

**Intermediate** operation. Removes duplicates using the element's `equals()`/`hashCode()` (internally uses a `HashSet`-like structure for stateful filtering).

```java
Stream.of(1, 2, 2, 3, 3, 3).distinct().forEach(System.out::println);  // 1 2 3
```

⚠️ For custom objects, `distinct()` only works correctly if `equals()`/`hashCode()` are properly overridden.

---

## 14. sorted()

**Intermediate**, **stateful** operation (must see all elements before emitting any — unlike filter/map which are element-at-a-time).

```java
names.stream().sorted().forEach(System.out::println);                         // natural order
names.stream().sorted(Comparator.reverseOrder()).forEach(System.out::println); // custom
employees.stream().sorted(Comparator.comparing(Employee::getSalary)
                                     .thenComparing(Employee::getName));
```

---

## 15. peek()

**Intermediate** operation meant for **debugging/side-effect inspection** — passes elements through unchanged while letting you "peek" at them.

```java
list.stream()
    .peek(x -> System.out.println("Before filter: " + x))
    .filter(x -> x > 2)
    .peek(x -> System.out.println("After filter: " + x))
    .collect(Collectors.toList());
```

⚠️ **Important gotcha:** `peek()` **won't execute at all** if there's no terminal operation, and JVM/JIT may even **skip peek() calls entirely** if it determines the result is unused (optimization). Never use `peek()` for actual business logic/mutation — it's for debugging only.

---

## 16. limit() / skip()

**Intermediate**, short-circuiting operations.

```java
numbers.stream().limit(5)              // take first 5 elements only
numbers.stream().skip(3)               // skip first 3 elements
numbers.stream().skip(3).limit(5)      // classic pagination pattern: page 2, size 5
```

`limit()` is **short-circuiting** — for an infinite stream, `Stream.iterate(1, n -> n+1).limit(5)` terminates correctly instead of running forever.

---

## 17. forEach()

**Terminal** operation. Performs an action for each element; returns `void`.

```java
names.stream().forEach(System.out::println);
```

⚠️ Cannot `break`/`return` out of it like a normal loop. Order is **not guaranteed** for parallel streams — use `forEachOrdered()` if you need guaranteed encounter order with parallel streams.

---

## 18. reduce()

**Terminal** operation. Combines all elements into a **single result** using a `BinaryOperator`.

Three overloads:
```java
// 1. No identity — returns Optional (empty stream possibility)
Optional<Integer> sum = numbers.stream().reduce((a, b) -> a + b);

// 2. With identity — always returns a value
int sum = numbers.stream().reduce(0, (a, b) -> a + b);

// 3. With identity + combiner (needed for parallel streams)
int totalLength = names.stream()
                        .reduce(0, (partial, name) -> partial + name.length(),
                                (a, b) -> a + b);  // combiner merges partial results from parallel threads
```

**Interview line:** "The 3-arg reduce is essential for parallel streams — the accumulator works on partial results per thread/chunk, and the combiner merges those partial results together. Without a combiner, parallel reduction wouldn't know how to merge sub-results."

---

## 19. collect()

**Terminal** operation. **Mutable reduction** — accumulates elements into a container (List, Set, Map, String, etc.) using a `Collector`.

```java
List<String> list = stream.collect(Collectors.toList());
Set<String> set = stream.collect(Collectors.toSet());
String joined = stream.collect(Collectors.joining(", "));

// Manual 3-arg collect (rarely used directly, but explains what Collectors do)
List<String> result = stream.collect(ArrayList::new, ArrayList::add, ArrayList::addAll);
//                                    supplier        accumulator    combiner
```

`collect()` is more flexible than `reduce()` for building mutable collections because it avoids creating a new immutable object on every step (reduce with immutable accumulation is O(n²) for something like string concatenation; collect with a mutable container is O(n)).

---

## 20. Collectors.toList()

Most common collector — gathers stream elements into a `List` (implementation not guaranteed to be `ArrayList`, though it is in the current JDK implementation).

```java
List<String> upperNames = names.stream()
                                .map(String::toUpperCase)
                                .collect(Collectors.toList());
```

Java 16+ shortcut: `.toList()` (returns an **immutable** list, unlike `Collectors.toList()`).

---

## 21. groupingBy()

Groups elements by a **classifier function** into a `Map<K, List<T>>`.

```java
Map<String, List<Employee>> byDept = employees.stream()
        .collect(Collectors.groupingBy(Employee::getDepartment));

// With downstream collector — group + count
Map<String, Long> countByDept = employees.stream()
        .collect(Collectors.groupingBy(Employee::getDepartment, Collectors.counting()));

// group + average salary
Map<String, Double> avgSalaryByDept = employees.stream()
        .collect(Collectors.groupingBy(Employee::getDepartment,
                                        Collectors.averagingDouble(Employee::getSalary)));

// Multi-level grouping
Map<String, Map<String, List<Employee>>> byDeptThenRole = employees.stream()
        .collect(Collectors.groupingBy(Employee::getDepartment,
                                        Collectors.groupingBy(Employee::getRole)));
```

**Interview line:** "groupingBy is basically SQL's `GROUP BY`. The second argument is a downstream collector that decides what to do with each group — collect them as a list (default), count them, sum a field, or even group again for a nested map."

---

## 22. partitioningBy()

Special case of grouping — splits elements into **exactly two groups** based on a `Predicate` (`Map<Boolean, List<T>>`). More efficient than `groupingBy` for binary conditions since it always produces exactly 2 keys (true/false), even if one group is empty.

```java
Map<Boolean, List<Integer>> partitioned = numbers.stream()
        .collect(Collectors.partitioningBy(n -> n % 2 == 0));

List<Integer> evens = partitioned.get(true);
List<Integer> odds = partitioned.get(false);
```

---

## 23. joining()

Concatenates stream of `CharSequence`/`String` elements.

```java
String csv = names.stream().collect(Collectors.joining());            // "JohnJaneAdam"
String withDelim = names.stream().collect(Collectors.joining(", "));  // "John, Jane, Adam"
String withPrefixSuffix = names.stream()
        .collect(Collectors.joining(", ", "[", "]"));                 // "[John, Jane, Adam]"
```

---

## 24. counting()

Downstream collector — counts elements, typically used inside `groupingBy`.

```java
long total = stream.collect(Collectors.counting());   // returns Long

Map<String, Long> countByDept = employees.stream()
        .collect(Collectors.groupingBy(Employee::getDepartment, Collectors.counting()));
```

Note: for a plain count without grouping, `stream.count()` (terminal op returning `long`) is simpler and preferred over `collect(Collectors.counting())`.

---

## 25. Optional

A container object that **may or may not** hold a non-null value — designed to avoid `NullPointerException` and force explicit handling of "value might be absent."

```java
Optional<String> opt = Optional.of("hello");        // throws NPE if value is null
Optional<String> opt2 = Optional.ofNullable(name);   // safe — wraps null as empty
Optional<String> empty = Optional.empty();

opt.isPresent();               // boolean check (old style)
opt.isEmpty();                 // Java 11+
opt.ifPresent(System.out::println);
opt.orElse("default");                    // eager — "default" always constructed
opt.orElseGet(() -> computeDefault());    // lazy — only called if empty
opt.orElseThrow(() -> new RuntimeException("not found"));
opt.map(String::toUpperCase).orElse("N/A");   // chainable transformation
```

**Interview line:** "Optional is meant as a **return type** for methods that might not have a result — not as a field type (adds serialization overhead) or a method parameter (forces callers to wrap things unnecessarily). It communicates absence explicitly in the type signature instead of relying on nullable references and hoping callers remember to null-check."

---

## 26. map() vs flatMap()

| Aspect | map() | flatMap() |
|---|---|---|
| Transformation | 1-to-1 | 1-to-many (then flattens) |
| Input function returns | A value: `Function<T, R>` | A **Stream**: `Function<T, Stream<R>>` |
| Result shape | Same structure, transformed values | Flattened single-level stream |
| Example use | `String -> Integer` (length) | `List<List<T>> -> List<T>`, `String -> words[]` |

```java
// map — wraps result, keeping nested structure
Stream<Stream<Integer>> nested = listOfLists.stream().map(List::stream);  // still nested!

// flatMap — flattens into one stream
Stream<Integer> flat = listOfLists.stream().flatMap(List::stream);       // flat!
```

Also applies to `Optional`: `Optional<Optional<T>>` (map) vs `Optional<T>` (flatMap) when chaining methods that themselves return `Optional`.

---

## 27. Intermediate vs Terminal operations

| Aspect | Intermediate | Terminal |
|---|---|---|
| Returns | Another Stream | Non-stream result (List, count, void, boolean, Optional...) |
| Laziness | **Lazy** — not executed until a terminal op is called | **Triggers execution** of the entire pipeline |
| Chainable | Yes, multiple can be chained | No — only one per stream, and it consumes the stream |
| Examples | `filter, map, flatMap, distinct, sorted, peek, limit, skip` | `forEach, collect, reduce, count, anyMatch, findFirst, toArray` |

**Interview line:** "None of the intermediate operations do anything by themselves — the whole pipeline is just a recipe until a terminal operation triggers a single pass through the data. This is what enables optimizations like short-circuiting and avoids unnecessary full traversals."

---

## 28. Lazy evaluation

Streams don't process elements until a terminal operation is invoked. Additionally, processing is **per-element vertically through the whole pipeline**, not **per-operation horizontally across all elements**.

```java
List<String> names = List.of("Alice", "Bob", "Charlie");

names.stream()
     .filter(n -> { System.out.println("filter: " + n); return n.length() > 3; })
     .map(n -> { System.out.println("map: " + n); return n.toUpperCase(); })
     .forEach(System.out::println);

// Output order proves per-element pipelining, NOT filter-all-then-map-all:
// filter: Alice
// map: Alice
// ALICE
// filter: Bob
// filter: Charlie
// map: Charlie
// CHARLIE
```

This enables **short-circuiting**: `findFirst()`, `anyMatch()`, `limit()` can stop processing as soon as they have their answer, without touching the rest of the source — critical for infinite streams (`Stream.iterate`, `Stream.generate`).

---

## 29. Sequential vs Parallel streams

```java
list.stream()          // sequential — single thread
list.parallelStream()  // parallel — uses ForkJoinPool.commonPool()
stream.parallel()      // convert an existing stream to parallel
```

- Parallel streams split the source (using `Spliterator`), process chunks on multiple threads from the **common ForkJoinPool** (default size = `Runtime.availableProcessors() - 1`), then merge results.
- **When it helps:** large datasets, CPU-intensive operations, easily divisible sources (ArrayList, arrays — good `Spliterator` support).
- **When it hurts:**
  - Small datasets — thread coordination overhead outweighs gains.
  - I/O-bound operations — threads just block, no CPU benefit; can starve the shared common pool used elsewhere in the app.
  - **Stateful/order-dependent operations** — sorting, `LinkedList` sources (bad splitting), or ops with side effects on shared mutable state (race conditions).
  - Using a **non-thread-safe** collector destination (e.g., manually collecting into a plain `ArrayList` via `forEach` instead of `collect()`).

**Interview line:** "Parallel streams aren't a free lunch — they use the shared common ForkJoinPool, so a long-running blocking parallel stream elsewhere in the app can starve yours. Only reach for parallelStream() after profiling shows a genuine CPU-bound bottleneck on a large enough dataset — for most business logic streams over a few thousand elements, sequential is faster due to overhead."

---

## 30. Common Stream interview problems

**a) Find duplicates in a list**
```java
Set<Integer> seen = new HashSet<>();
List<Integer> duplicates = numbers.stream()
        .filter(n -> !seen.add(n))   // add() returns false if already present
        .collect(Collectors.toList());
```

**b) Find the second-highest number**
```java
Optional<Integer> second = numbers.stream()
        .distinct()
        .sorted(Comparator.reverseOrder())
        .skip(1)
        .findFirst();
```

**c) Group words by their length**
```java
Map<Integer, List<String>> byLength = words.stream()
        .collect(Collectors.groupingBy(String::length));
```

**d) Find frequency count of each element**
```java
Map<String, Long> freq = words.stream()
        .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
```

**e) Convert List<Employee> to Map<Id, Name>**
```java
Map<Integer, String> idToName = employees.stream()
        .collect(Collectors.toMap(Employee::getId, Employee::getName));

// Handle duplicate keys with a merge function
Map<String, Double> deptToTotalSalary = employees.stream()
        .collect(Collectors.toMap(Employee::getDepartment, Employee::getSalary,
                                   Double::sum));  // merge function for collisions
```

**f) Sum/average/max/min of a numeric field**
```java
double totalSalary = employees.stream().mapToDouble(Employee::getSalary).sum();
OptionalDouble avg = employees.stream().mapToDouble(Employee::getSalary).average();
Optional<Employee> highestPaid = employees.stream()
        .max(Comparator.comparing(Employee::getSalary));

// Or all at once
DoubleSummaryStatistics stats = employees.stream()
        .mapToDouble(Employee::getSalary)
        .summaryStatistics();
stats.getMax(); stats.getMin(); stats.getAverage(); stats.getSum(); stats.getCount();
```

**g) Flatten and find distinct words across sentences**
```java
Set<String> uniqueWords = sentences.stream()
        .flatMap(s -> Arrays.stream(s.split(" ")))
        .collect(Collectors.toSet());
```

**h) Check if any/all/none match a condition**
```java
boolean anyAdult = people.stream().anyMatch(p -> p.getAge() >= 18);
boolean allAdults = people.stream().allMatch(p -> p.getAge() >= 18);
boolean noneMinor = people.stream().noneMatch(p -> p.getAge() < 18);
// All are short-circuiting — stop as soon as the result is determined
```

**i) Reverse a String using streams**
```java
String reversed = someString.chars()
        .mapToObj(c -> String.valueOf((char) c))
        .reduce("", (a, b) -> b + a);
```

**j) Convert array of ints to List<Integer> and back**
```java
int[] arr = {1, 2, 3};
List<Integer> list = Arrays.stream(arr).boxed().collect(Collectors.toList());
int[] backToArray = list.stream().mapToInt(Integer::intValue).toArray();
```

---

## Quick Revision Cheat-Sheet

- **Lambda** → anonymous function implementing a functional interface, compiled via `invokedynamic`, not an inner class file.
- **Functional interface** → exactly 1 abstract method (+ any default/static methods).
- **Predicate** → `T -> boolean`. **Consumer** → `T -> void`. **Supplier** → `() -> T`. **Function** → `T -> R`.
- **Method refs** → static (`Class::method`), bound instance (`obj::method`), unbound instance (`Class::method`), constructor (`Class::new`).
- **Stream** → lazy, single-use, doesn't store data, pipeline of intermediate ops + 1 terminal op.
- **map** = 1-to-1 transform. **flatMap** = 1-to-many + flatten.
- **reduce** = functional fold to single value. **collect** = mutable reduction into a container.
- **groupingBy** = SQL GROUP BY. **partitioningBy** = binary split (always 2 keys).
- **peek()** = debugging only, not guaranteed to run without a terminal op.
- **Optional** = explicit "might be absent" return type; use `orElseGet` for lazy defaults.
- **Intermediate ops are lazy**; nothing runs until a **terminal op** triggers per-element pipelined execution.
- **Parallel streams** use the shared `ForkJoinPool.commonPool()` — great for large CPU-bound work, risky for I/O-bound or small datasets.
