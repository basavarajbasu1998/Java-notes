# Java Interview Notes

## Table of Contents
1. [Collections](#collections)
2. [Strings](#strings)
3. [JVM Memory](#jvm-memory)
4. [Garbage Collection](#garbage-collection)
5. [Exception Handling](#exception-handling)
6. [OOP Concepts](#oop-concepts)
7. [Java 8 Features](#java-8-features)
8. [Multithreading](#multithreading)
9. [Design Patterns](#design-patterns)
10. [SOLID Principles](#solid-principles)
11. [Spring Framework](#spring-framework)
12. [Microservices](#microservices)
13. [JPA & Hibernate](#jpa--hibernate)

---

## Collections

### HashMap

1. HashMap stores data in key-value pairs.
2. HashMap is not thread-safe.
3. It allows one null key and multiple null values.
4. HashMap uses `hashCode()` to calculate bucket index.
5. Internally it stores data as `Node(hash, key, value, next)`.
6. Buckets are maintained in an array.
7. If multiple keys map to the same bucket, it is called a collision.
8. Before Java 8, collisions were handled using a LinkedList.
9. From Java 8 onward, if bucket size exceeds 8, the LinkedList is converted to a Red-Black Tree for better performance.
10. HashMap uses both `hashCode()` and `equals()` to locate and compare keys.
11. Average time complexity for `put()` and `get()` is O(1).

### ConcurrentHashMap

1. Thread-safe implementation of `Map`.
2. Multiple threads can read and write concurrently.
3. Provides better concurrency than `Hashtable`.
4. Java 8 uses CAS (Compare-And-Swap) and bucket-level synchronization.
5. Does not lock the entire map.
6. Null keys and null values are not allowed.
7. Read operations are mostly lock-free.
8. Performance is better than `Hashtable` in multithreaded applications.
9. Used when multiple threads need to access and modify shared data.

### Hashtable

1. Stores data in key-value pairs.
2. Thread-safe because methods are synchronized.
3. Locks the entire Hashtable for every operation.
4. Does not allow null keys or null values.
5. Performance is slower due to full-table locking.
6. Legacy class introduced in JDK 1.0.

**Quick summary:**
- HashMap → Fast but Not Thread-Safe
- Hashtable → Thread-Safe but Slow
- ConcurrentHashMap → Thread-Safe and Fast

| Feature | HashMap | Hashtable | ConcurrentHashMap |
|---|---|---|---|
| Thread Safe | ❌ No | ✅ Yes | ✅ Yes |
| Performance | Fastest | Slow | Faster than Hashtable |
| Locking | No Lock | Entire Map Lock | Bucket/Node Level Lock + CAS |
| Multiple Threads | Not Safe | One thread at a time | Multiple threads simultaneously |
| Null Key | ✅ One null key | ❌ Not allowed | ❌ Not allowed |
| Null Value | ✅ Multiple null values | ❌ Not allowed | ❌ Not allowed |
| Introduced | JDK 1.2 | JDK 1.0 | JDK 1.5 |
| Use Case | Single-threaded | Legacy code | Multi-threaded applications |

---

### Comparable

1. Used for natural/default sorting.
2. Implemented at the class level.
3. Contains only one method: `compareTo()`.
4. We can define only one default sorting logic.
5. Present in the `java.lang` package.
6. Used when the class itself knows how objects should be sorted.

### Comparator

1. Used for custom sorting.
2. Implemented outside the class.
3. Contains the `compare()` method.
4. We can create multiple Comparator classes.
5. Present in the `java.util` package.
6. Used when we need different sorting logics for the same class.

| Comparable | Comparator |
|---|---|
| Class Level | Separate Class |
| `compareTo()` | `compare()` |
| One Sorting Logic | Multiple Sorting Logics |
| `java.lang` package | `java.util` package |
| Natural Sorting | Custom Sorting |

---

### ArrayList

1. Dynamic array.
2. Maintains insertion order.
3. Allows duplicate elements.
4. Fast for searching and accessing elements by index.
5. Internally uses a resizable array.
6. Insertion/deletion in the middle is slower because elements need to be shifted.

### LinkedList

1. Based on a doubly linked list.
2. Maintains insertion order.
3. Allows duplicate elements.
4. Fast for insertion and deletion.
5. Slow for searching and random access because it traverses nodes.
6. Each node stores data, a previous node reference, and a next node reference.

| Feature | ArrayList | LinkedList |
|---|---|---|
| Internal Structure | Dynamic Array | Doubly Linked List |
| Random Access (`get(index)`) | Fast O(1) | Slow O(n) |
| Insert/Delete Middle | Slow O(n) | Faster O(1)* |

---

### HashSet

1. Stores only unique elements.
2. Duplicate values are not allowed.
3. Insertion order is not maintained.
4. Internally uses a HashMap.
5. Allows one null value.
6. Best performance for `add()`, `remove()`, `contains()`.

### LinkedHashSet

1. Stores only unique elements.
2. Duplicate values are not allowed.
3. Maintains insertion order.
4. Internally uses a LinkedHashMap.
5. Allows one null value.
6. Slightly slower than HashSet due to maintaining order.

### TreeSet

1. Stores only unique elements.
2. Duplicate values are not allowed.
3. Stores elements in sorted order (ascending by default).
4. Internally uses a Red-Black Tree.
5. Does not allow null values.
6. Uses `Comparable` or `Comparator` for sorting.

| Feature | HashSet | LinkedHashSet | TreeSet |
|---|---|---|---|
| Duplicate Elements | ❌ Not Allowed | ❌ Not Allowed | ❌ Not Allowed |
| Order | No Order | Insertion Order | Sorted Order |
| Internal Structure | HashMap | LinkedHashMap | Red-Black Tree |
| Null Values | ✅ One Null | ✅ One Null | ❌ No Null |
| Performance | Fastest | Slightly Slower | Slower (sorting overhead) |

---

### Fail-Fast vs Fail-Safe Iterators

| Feature | Fail-Fast | Fail-Safe |
|---|---|---|
| Meaning | Detects modification during iteration and throws exception | Allows modification during iteration without exception |
| Exception | Throws `ConcurrentModificationException` | No `ConcurrentModificationException` |
| Works on | Original collection | Copy of collection / separate structure |
| Memory | No extra memory | Requires extra memory |
| Performance | Faster | Slower compared to fail-fast |
| Thread Safe | ❌ Not thread-safe | ✅ Generally safer for concurrent access |

---

### Iterator

1. Used to traverse collection elements.
2. Works with all Collection classes.
3. Supports only forward direction traversal.
4. Can remove elements using `remove()`.
5. Cannot add or update elements.
6. Part of the `java.util` package.

### ListIterator

1. Used only with List implementations.
2. Supports both forward and backward traversal.
3. Can add elements.
4. Can update elements using `set()`.
5. Can remove elements.
6. Can start from any index position.

---

## Strings

### String

1. Immutable (cannot be changed after creation).
2. Any modification creates a new String object.
3. String objects are stored in the String Pool (for literals).
4. Thread-safe because it is immutable.
5. Slow for frequent modifications.

### StringBuilder

1. Mutable (can be changed).
2. Modifies the existing object.
3. Faster than StringBuffer.
4. Not thread-safe.
5. Used in single-threaded applications.

### StringBuffer

1. Mutable.
2. Thread-safe.
3. Methods are synchronized.
4. Slower than StringBuilder.
5. Used in multithreaded applications.

### String Constant vs String Pool

| String Constant | String Pool |
|---|---|
| A string value written directly in double quotes. | A special memory area where Java stores string literals. |
| Example: `"Hello"` | The location where `"Hello"` is stored. |
| Created using string literals. | Managed by the JVM. |
| Represents the actual string literal. | Represents the collection/storage of literals. |
| Written in source code. | Exists in Heap memory (String Constant Pool area). |
| Duplicate constants are avoided. | Maintains only one copy of identical literals. |
| Compiler identifies constants. | JVM manages storage and reuse. |
| Used to improve readability. | Used to improve memory efficiency. |

---

## JVM Memory

Topics covered: Stack memory, Heap memory, Method area, String pool, Garbage Collection, JVM memory structure.

### Stack Memory Allocation

- Used for method calls, local variables, and references.
- Memory is automatically allocated when a method starts and cleared when it ends.
- Data exists only during the method's execution.
- If the stack runs out of space, a `StackOverflowError` occurs.
- Stack is faster compared to Heap.
- Memory is allocated and deallocated automatically.
- Follows LIFO (Last In, First Out) order.

### Heap Memory Allocation

- Used for objects and instance variables created using the `new` keyword.
- Size depends on the class structure.
- The Garbage Collector manages this area by removing unused objects.
- Used for dynamic memory allocation.
- Programmer allocates memory at runtime.
- Memory remains allocated until it is freed.

---

## Garbage Collection

### What is Garbage Collection?

Garbage Collection is the process by which the JVM automatically identifies and removes objects that are no longer reachable/referenced by the application, freeing up heap memory without manual intervention (unlike C/C++, where developers manually free memory).

### JVM Memory Areas (Relevant to GC)

**Heap** — where all objects are created. Divided into:
- **Young Generation** — newly created objects, further split into:
  - **Eden Space** — where objects are first allocated.
  - **Survivor Spaces (S0 and S1)** — objects that survive garbage collection in Eden are moved here.
- **Old Generation (Tenured)** — long-lived objects that survived multiple GC cycles in the young generation get promoted here.

**Metaspace (Java 8+)** — stores class metadata; replaced PermGen from earlier Java versions. Unlike PermGen, Metaspace grows dynamically using native memory instead of a fixed heap size, reducing `OutOfMemoryError: PermGen space` issues.

**Stack** — stores method call frames and local variables; not managed by GC (cleared automatically when a method returns).

### How an Object Becomes Eligible for GC

An object becomes eligible for garbage collection when it's no longer reachable from any live thread or static reference. Common ways this happens:

- Setting a reference to `null`.
- Reassigning a reference to point to another object.
- An object created inside a method goes out of scope after the method returns.
- **Island of isolation** — two or more objects reference each other but have no external reference (still eligible for GC).

> **Note:** Calling `System.gc()` only *suggests* garbage collection to the JVM — it doesn't guarantee immediate execution.

### Types of Garbage Collectors in Java

- **Serial GC** — single-threaded, stops all application threads during collection ("stop-the-world"). Best for small applications with small heaps.
- **Parallel GC (Throughput Collector)** — uses multiple threads for young generation collection; default GC in Java 8. Focuses on maximizing throughput.
- **CMS (Concurrent Mark Sweep)** — performs most of its work concurrently with application threads to minimize pause times. Deprecated in Java 9, removed in Java 14.
- **G1 GC (Garbage First)** — divides the heap into regions instead of fixed young/old generation spaces; balances throughput and low pause times. Default GC starting Java 9.
- **ZGC / Shenandoah** — newer low-latency collectors (Java 11+) for very large heaps with sub-millisecond pause times; typically asked about only in senior/advanced interviews.

### GC Algorithms/Phases

1. **Mark** — identify which objects are still reachable (live).
2. **Sweep** — remove unreachable (dead) objects, reclaiming memory.
3. **Compact** — rearrange remaining live objects to eliminate fragmentation (not all collectors do this every cycle).

### Minor GC vs Major GC vs Full GC

- **Minor GC** — cleans the Young Generation only; relatively fast and frequent.
- **Major GC** — cleans the Old Generation.
- **Full GC** — cleans the entire heap (Young + Old + Metaspace); more expensive, causes longer pause times, and is a common performance-tuning target.

### Object Promotion

Objects that survive multiple Minor GC cycles in the Young Generation (based on an age threshold) get promoted to the Old Generation. This is based on the **weak generational hypothesis** — most objects die young, so focusing collection effort on young objects is more efficient.

### Common Interview Follow-ups

- **What causes `OutOfMemoryError`?** Heap exhausted because too many objects remain reachable, memory leaks (e.g., static collections holding references, unclosed resources), or heap size configured too small (`-Xmx`).
- **`finalize()` method** — deprecated since Java 9; was called before an object was garbage collected, but unreliable (no guarantee of timing or even execution), so `try-with-resources` / `AutoCloseable` is preferred instead.
- **Strong vs Soft vs Weak vs Phantom References:**
  - **Strong reference** — normal reference; prevents GC as long as it exists.
  - **Soft reference** — collected only when the JVM is low on memory (used for caches).
  - **Weak reference** — collected in the next GC cycle regardless of memory pressure (used in `WeakHashMap`).
  - **Phantom reference** — object already finalized; used for cleanup actions after an object is removed from memory, queued via `ReferenceQueue`.
- **How to tune GC** — common JVM flags: `-Xms`, `-Xmx` (initial/max heap size), `-XX:+UseG1GC`, `-XX:+PrintGCDetails` (for logging).

---

## Exception Handling

### What is an Exception?

An exception is an event that disrupts the normal flow of a program's execution during runtime. Java uses an object-oriented approach to handle these events through the exception hierarchy, allowing the program to catch and recover from errors gracefully instead of crashing abruptly.

### Exception Hierarchy

- **`Throwable`** — root class for all errors and exceptions.
  - **`Error`** — serious problems that applications shouldn't try to catch (e.g., `OutOfMemoryError`, `StackOverflowError`). Usually indicates JVM-level issues.
  - **`Exception`** — conditions a program might want to catch and handle.
    - **Checked Exceptions** — checked at compile time; must be either caught or declared using `throws` (e.g., `IOException`, `SQLException`).
    - **Unchecked Exceptions (`RuntimeException`)** — not checked at compile time; occur due to programming errors (e.g., `NullPointerException`, `ArrayIndexOutOfBoundsException`, `ArithmeticException`, `ClassCastException`).

### Checked vs Unchecked Exceptions

| Feature | Checked | Unchecked |
|---|---|---|
| Checked at | Compile time | Runtime |
| Must handle? | Yes (catch or declare) | No, optional |
| Examples | `IOException`, `SQLException` | `NullPointerException`, `ArithmeticException` |
| Typical cause | External factors (file not found, network issue) | Programming logic errors |

### Core Keywords

- **`try`** — block of code that might throw an exception.
- **`catch`** — handles a specific exception type thrown in the try block.
- **`finally`** — always executes, regardless of whether an exception occurred or was caught; commonly used for closing resources like streams or connections.
- **`throw`** — used to explicitly throw an exception instance.
- **`throws`** — declares that a method might throw a checked exception; placed in the method signature.

### try-with-resources

Introduced in Java 7, allows automatic closing of resources (like file streams, database connections) that implement the `AutoCloseable` interface, without needing an explicit `finally` block.

```java
try (BufferedReader br = new BufferedReader(new FileReader("file.txt"))) {
    // use br
} catch (IOException e) {
    // handle exception
}
```

The resource is closed automatically at the end of the try block, even if an exception occurs — cleaner than manually closing in `finally`.

### Multi-catch Block

Introduced in Java 7, allows catching multiple exception types in a single catch block using `|`, avoiding duplicated handling code:

```java
try {
    // code
} catch (IOException | SQLException e) {
    // common handling
}
```

### Custom Exceptions

Developers can create their own exception classes by extending `Exception` (checked) or `RuntimeException` (unchecked), useful for representing business-specific error conditions clearly.

```java
class InsufficientBalanceException extends Exception {
    public InsufficientBalanceException(String message) {
        super(message);
    }
}
```

### Exception Propagation

If an exception is not caught in the method where it occurs, it propagates up the call stack to the calling method, and so on, until it's either caught or reaches the JVM (which then terminates the thread/program and prints the stack trace).

### Common Interview Follow-ups

- **Difference between `throw` and `throws`:** `throw` is used to actually throw an exception instance inside a method body; `throws` is used in a method signature to declare that the method might throw a certain exception, shifting responsibility to the caller.
- **Can the `finally` block be skipped?** Yes — if `System.exit()` is called inside try/catch, or if the JVM crashes, the `finally` block won't execute.
- **What if both `try` and `finally` have return statements?** The `finally` block's return value overrides the `try` block's return value — a common tricky interview question. Best practice: avoid `return` inside `finally`.
- **Exception chaining:** wrapping one exception inside another using constructors that accept a cause parameter (e.g., `throw new ServiceException("failed", originalException)`), which preserves the original stack trace for debugging.
- **Why is catching `Exception` or `Throwable` broadly considered bad practice?** It can accidentally catch unrelated exceptions (including `Error` types like `OutOfMemoryError` if catching `Throwable`), masking real bugs and making debugging harder. Best practice is to catch specific exception types.
- **Difference between `Exception` and `Error`:** `Exception` is meant to be handled by the application (recoverable); `Error` represents serious problems (like the JVM running out of memory) that applications generally shouldn't try to catch or recover from.

---

## OOP Concepts

### 1. Encapsulation

Wrapping data (fields) and methods that operate on that data into a single unit (class), and restricting direct access to some of the object's components using access modifiers (`private`, `protected`, `public`).

- Achieved using private fields + public getter/setter methods.
- Benefits: data hiding, controlled access, easier maintenance, and validation logic can be added inside setters.

```java
class Account {
    private double balance;
    public double getBalance() { return balance; }
    public void deposit(double amount) {
        if (amount > 0) balance += amount;
    }
}
```

### 2. Inheritance

Mechanism where one class (child/subclass) acquires the properties and behaviors (fields and methods) of another class (parent/superclass), promoting code reusability.

- Uses the `extends` keyword for classes, `implements` for interfaces.
- Types: single, multilevel, hierarchical. Java does not support multiple inheritance with classes (to avoid the diamond problem) but allows it through interfaces.

### 3. Polymorphism

The ability of an object to take many forms — the same method behaves differently based on the object or arguments.

- **Compile-time polymorphism (Method Overloading)** — same method name, different parameter list, resolved at compile time.
- **Runtime polymorphism (Method Overriding)** — subclass provides a specific implementation of a method already defined in its parent class, resolved at runtime using dynamic method dispatch.

```java
class Animal { void sound() { System.out.println("Some sound"); } }
class Dog extends Animal { void sound() { System.out.println("Bark"); } }
```

### 4. Abstraction

Hiding implementation details and showing only essential features to the user.

- Achieved using abstract classes (can have both abstract and concrete methods) and interfaces (traditionally only abstract methods, but can now have default/static methods since Java 8).
- Focuses on "what" an object does rather than "how" it does it.

### Abstract Class vs Interface

| Feature | Abstract Class | Interface |
|---|---|---|
| Methods | Abstract + concrete | Abstract, default, static (Java 8+) |
| Fields | Any type (instance variables allowed) | Only `public static final` (constants) |
| Multiple inheritance | Not supported (single class extension) | Supported (a class can implement multiple interfaces) |
| Constructor | Can have one | Cannot have one |
| Access modifiers | Any (`private`, `protected`, `public`) | `public` by default |
| When to use | Share code among closely related classes | Define a contract/capability across unrelated classes |

### Method Overloading vs Method Overriding

| Feature | Overloading | Overriding |
|---|---|---|
| Definition | Same method name, different parameters | Same method name and signature, subclass redefines parent's method |
| Binding | Compile-time (static binding) | Runtime (dynamic binding) |
| Class relationship | Same class | Parent-child (inheritance) |
| Return type | Can differ | Must be same or covariant |
| Access modifier | Can differ freely | Cannot reduce visibility of overridden method |

### Constructor Concepts

- **Constructor overloading** — multiple constructors with different parameter lists in the same class.
- **`this()`** — used to call another constructor in the same class (constructor chaining).
- **`super()`** — used to call the parent class's constructor; must be the first statement if used explicitly, and is called implicitly if not written.
- Constructors are not inherited by subclasses.

### Common Interview Follow-ups

- **Can you override a static method?** No — static methods belong to the class, not the instance, so they're hidden (method hiding), not overridden, if redefined in a subclass.
- **Can a constructor be private?** Yes — commonly used in the Singleton design pattern to prevent external instantiation.
- **What is the diamond problem, and how does Java avoid it?** Occurs when a class inherits from two classes that both define the same method — ambiguity in which one to use. Java avoids this for classes (single inheritance only) but faces a similar issue with interface default methods (Java 8+), where a class implementing two interfaces with the same default method must explicitly override it to resolve ambiguity.
- **Difference between `==` and `.equals()`:** `==` compares references (memory addresses) for objects; `.equals()` compares actual content/value, provided it's properly overridden (default `Object.equals()` behaves like `==`).
- **Why override `hashCode()` when overriding `equals()`?** To maintain the contract that equal objects must have equal hash codes — critical for correct behavior in hash-based collections like `HashMap`, `HashSet`.
- **Association vs Aggregation vs Composition:**
  - **Association** — general relationship between two independent classes.
  - **Aggregation** — "has-a" relationship where the child can exist independently of the parent (weak ownership, e.g., Department has Employees).
  - **Composition** — "has-a" relationship where the child cannot exist without the parent (strong ownership, e.g., House has Rooms).

---

## Java 8 Features

### Lambda Expressions

Introduced in Java 8, lambda expressions allow developers to write concise, functional-style code by representing anonymous functions. They enable passing code as parameters or assigning it to variables, resulting in cleaner and more readable programs.

- Lambda expressions implement a functional interface (an interface with only one abstract method).
- Enable passing code as data (method arguments).
- Can access only final or effectively final variables from the enclosing scope.
- Cannot throw checked exceptions unless the functional interface declares them.

**Structure:**
- **Parameter list** — parameters for the lambda expression.
- **Arrow token (`->`)** — separates the parameter list and the body.
- **Body** — logic to be executed.

### Functional Interfaces

A functional interface in Java is an interface that has only one abstract method, making it suitable for use with lambda expressions and method references.

- Use `@FunctionalInterface` to ensure only one abstract method (the annotation is optional but recommended).
- Enable clean, concise code using lambdas and method references.

**Example:** `Runnable` has one abstract method `run()`, so it qualifies as a functional interface.

```java
new Thread(() -> System.out.println("New thread created")).start();
```

Here, the lambda `() -> System.out.println("New thread created")` defines the `run()` method, and `start()` runs it on a new thread.

### The Four Core Functional Interface Types

Java 8 introduced four main functional interface types under `java.util.function`, widely used in the Stream API, collections, and lambda-based operations.

- **`Consumer<T>`** — accepts one argument and performs an action (e.g., printing, logging), returns nothing. Variants: `DoubleConsumer`, `IntConsumer`, `LongConsumer`.
- **`Predicate<T>`** — represents a boolean-valued function of one argument; commonly used for filtering in streams. Variants: `IntPredicate`, `DoublePredicate`, `LongPredicate`.
- **`Function<T, R>`** — takes one argument and returns a result; commonly used for transforming data. Variants:
  - `BiFunction` — takes two arguments and returns a result.
  - `UnaryOperator` — input and output are of the same type.
  - `BinaryOperator` — like `BiFunction` but with the same input/output type.
- **`Supplier<T>`** — takes no arguments and returns a single output. Variants: `BooleanSupplier`, `DoubleSupplier`, `LongSupplier`, `IntSupplier`.

### Method References

Method references are a shorthand way to refer to an existing method without invoking it, making lambda expressions shorter, cleaner, and more readable. They use the double-colon (`::`) operator and are mainly used with functional interfaces.

```java
// Using method reference to print each name
Arrays.stream(names).forEach(Geeks::print);
```

Here, `print` is a static method used to print names; the array of names is streamed and each element is passed directly to `print` via the method reference.

**Forms:**
- `ClassName::methodName` — static method reference.
- `objectReference::methodName` — instance method reference.
- `ClassName::new` — constructor reference.

### Stream API — Intermediate Operations

Intermediate operations transform a stream into another stream:

- **`filter()`** — filters elements based on a specified condition.
- **`map()`** — transforms each element in a stream to another value.
- **`sorted()`** — sorts the elements of a stream.
- **`distinct()`** — removes duplicates.
- **`skip()`** — skips the first *n* elements.
- **`forEach()`** — iterates over all elements in a stream.
- **`collect(Collectors.toList())`** — collects stream elements into a list (or other collections like set/map).
- **`reduce()`** — reduces stream elements into a single aggregated result.
- **`count()`** — returns the total number of elements in a stream.
- **`anyMatch()` / `allMatch()` / `noneMatch()`** — check whether elements match a given condition.
- **`findFirst()` / `findAny()`** — return the first or any element from a stream.

`Stream.reduce()` performs a reduction on the elements of a stream using an associative accumulation function and returns an `Optional`. It's commonly used to aggregate or combine elements into a single result, such as computing the maximum, minimum, sum, or product.

### Parallel Streams

Parallel Stream is a feature introduced in Java 8 that allows data processing operations to run in parallel and concurrently using the Stream API. Normally, Stream API operations (filtering, mapping, collecting) run linearly (sequentially); using a parallel stream, these operations run concurrently across multiple threads for improved performance.

**Example:**

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
int sum = numbers.parallelStream().mapToInt(Integer::intValue).sum();
```

Using `parallelStream()` instead of `stream()` runs the operations across multiple threads concurrently; `mapToInt()` and `sum()` then compute the total.

> Using Parallel Stream should be done with care, since incorrect usage can lead to issues such as race conditions and deadlocks.

**Advantages:**
1. **High speed** — improved performance and faster execution for concurrent, parallel operations.
2. **Readable code** — data processing code becomes simpler and more intuitive to write.
3. **Easy implementation** — no boilerplate code needed for concurrent operations.

**Disadvantages:**
1. **High memory consumption** — creating threads for parallel operations incurs additional memory overhead.
2. **Non-determinism** — output may vary between executions.
3. **Concurrency issues** — incorrect usage can lead to race conditions and deadlocks.

**When not to use parallel streams:**
Parallel streams internally use the shared common `ForkJoinPool`, which is also used by other parallel streams and by any `CompletableFuture.supplyAsync()` calls without a custom executor. Because of this:

- Blocking I/O operations (network calls, file reads, database calls) inside a parallel stream can starve the shared thread pool, slowing down unrelated parts of the application.
- Parallel streams are best suited for CPU-bound tasks on large datasets, not I/O-bound tasks.
- For small datasets, the overhead of splitting and merging tasks across threads can make parallel streams slower than sequential ones.

### Comparable vs Comparator (Recap)

| Feature | Comparable | Comparator |
|---|---|---|
| Definition | Defines natural ordering within the class | Defines external or custom sorting logic |
| Method | `compareTo()` | `compare()` |
| Implementation | Implemented in the class | Implemented in a separate class |
| Sorting criteria | Natural order sorting | Custom order sorting |
| Usage | Used for a single sorting order | Used for multiple sorting orders |

### Default Methods in Interfaces

Before Java 8, interfaces could only have abstract methods (no body); the implementation had to be provided in a separate class. Java 8 introduced **default methods** in interfaces, allowing methods with a body, making interfaces more flexible and backward-compatible.

**Key features:**
- Interfaces can now have both abstract and default methods.
- Default methods provide backward compatibility without breaking existing code.
- They allow API evolution and support new features like Streams and Lambdas.

### Optional Class

`Optional` is a container introduced in Java 8 to represent a value that may or may not be present, mainly to avoid `NullPointerException` and reduce explicit null checks.

- **`Optional.of(value)`** — wraps a non-null value; throws `NullPointerException` immediately if the value is null.
- **`Optional.ofNullable(value)`** — wraps a value that may be null; returns an empty `Optional` if null.
- **`Optional.empty()`** — returns an empty `Optional`.
- **`isPresent()` / `isEmpty()`** — check if a value exists.
- **`get()`** — returns the value if present, else throws `NoSuchElementException`. Common interview trap: calling `get()` without checking presence defeats the purpose of `Optional`.
- **`orElse(default)`** — returns the value if present, else returns the default.
- **`orElseGet(Supplier)`** — like `orElse()`, but lazily computes the default only when needed (better for expensive defaults).
- **`orElseThrow()`** — throws a custom exception if the value is absent.
- **`map()` / `filter()`** — allow chaining transformations on the wrapped value without manual null checks.

> **Best practice:** use `Optional` as a return type for methods, not as a field type or method parameter.

### Date and Time API (`java.time` package)

Before Java 8, `java.util.Date` and `Calendar` had problems: mutability, poor design, not thread-safe, confusing (0-based) month indexing. Java 8 introduced a new, immutable, thread-safe Date-Time API inspired by the Joda-Time library.

**Key classes:**
- **`LocalDate`** — date only (year, month, day), no time or timezone.
- **`LocalTime`** — time only, no date or timezone.
- **`LocalDateTime`** — combines date and time, no timezone.
- **`ZonedDateTime`** — date and time with timezone information.
- **`Duration`** — represents a time-based amount (hours, minutes, seconds).
- **`Period`** — represents a date-based amount (years, months, days).
- **`DateTimeFormatter`** — used for formatting/parsing date-time objects.

All classes in this API are immutable — every "modification" method (like `plusDays()`) returns a new object instead of changing the existing one, making them thread-safe by design.

### Stream API — Deeper Concepts

- **`map()` vs `flatMap()`:** `map()` transforms each element into another single value (one-to-one). `flatMap()` is used when each element maps to a stream of values (like a list of lists), and it flattens the result into a single stream (one-to-many, then merged).
- **Collectors:**
  - **`Collectors.groupingBy()`** — groups elements by a classifier function into a `Map`.
  - **`Collectors.partitioningBy()`** — splits elements into two groups based on a boolean predicate (true/false map).
  - **`Collectors.joining()`** — concatenates String elements, optionally with a delimiter, prefix, suffix.
  - **`Collectors.toMap()`** — collects elements into a `Map` using key and value mapping functions.
- **Lazy vs eager evaluation:** Intermediate operations (`filter`, `map`, `sorted`) are lazy — they don't execute until a terminal operation (`collect`, `forEach`, `reduce`) is invoked. This lets the JVM optimize the pipeline execution.
- **Streams can only be consumed once:** Once a terminal operation is called, the stream is considered closed. Attempting to reuse it throws `IllegalStateException`. A new stream must be created from the source if further processing is needed.
- **`Stream.of()` vs `Arrays.stream()`:** `Stream.of()` creates a stream from individual elements or a varargs array. `Arrays.stream()` is specifically designed to convert arrays (including primitive arrays like `int[]`) into a stream, and offers overloads for primitive types to avoid boxing overhead.

### Lambda Internals

Unlike anonymous inner classes, lambda expressions are not compiled into separate `.class` files. Instead, Java uses the `invokedynamic` bytecode instruction (introduced in Java 7, leveraged by Java 8) to generate the implementing class at runtime, which is more efficient and reduces class-loading overhead.

"Effectively final" means a local variable is not explicitly declared `final`, but its value is never changed after initialization. Lambdas can only capture such variables because they may be invoked on a different thread or at a later time, and allowing mutation could lead to inconsistent or unsafe state.

### Functional Interface Corner Cases

- A functional interface can extend another interface, but only if the parent interface's methods are also abstract and don't add a second abstract method overall.
- A functional interface can have any number of default and static methods — the "single abstract method" rule applies only to abstract methods, not default/static ones.
- `Object` class methods (like `toString()`, `equals()`, `hashCode()`) declared in an interface don't count toward the abstract method limit, since they're implicitly implemented by any class.

### CompletableFuture — Quick Recap

*(See the [Multithreading](#multithreading) section for the full theory.)* `CompletableFuture` (in `java.util.concurrent`) is used for asynchronous, non-blocking programming, improving on the older `Future` interface, which only allowed blocking `get()` calls with no way to chain or combine results.

- **`supplyAsync()`** — runs a task asynchronously and returns a result.
- **`runAsync()`** — runs a task asynchronously with no return value.
- **`thenApply()`** — transforms the result once available (synchronous continuation).
- **`thenApplyAsync()`** — same, but runs in a separate thread.
- **`thenCompose()`** — chains two dependent async operations (flattens nested `CompletableFuture`s, similar to `flatMap`).
- **`thenCombine()`** — combines results of two independent `CompletableFuture`s.
- **`exceptionally()`** — handles exceptions in the async pipeline.
- **`join()`** — like `get()` but throws unchecked exceptions, easier to use in lambdas.

**Difference from `Future`:** `Future` only supports blocking retrieval of a result; `CompletableFuture` supports callbacks, chaining, combining multiple futures, and manual completion.

---

## Multithreading

### Thread Creation

A thread is a lightweight unit of execution that allows a Java program to perform multiple tasks concurrently. Threads share the same memory and resources of a process. The main thread starts automatically when a Java program begins execution. New threads can be created to execute tasks independently and concurrently.

```java
class MyThread extends Thread {
    public void run() {
        // task to perform
    }
}

MyThread t = new MyThread();
t.start();
```

### Runnable Interface

The `Runnable` interface defines a task that can be executed by a thread. It contains a single method, `run()`, which holds the code to be executed.

- Contains only one method: `run()`.
- Separates task logic from thread creation.
- Supports multiple inheritance since it is an interface.

```java
class MyTask implements Runnable {
    public void run() {
        // task to perform
    }
}

Thread t = new Thread(new MyTask());
t.start();
```

### Thread Lifecycle

A Java thread transitions through several states from creation to termination:

1. **New** — Created but not yet started; code has not begun running.
   `public static final Thread.State NEW`
2. **Runnable** — Executing in the JVM but may be waiting for OS resources such as a processor.
   `public static final Thread.State RUNNABLE`
3. **Blocked** — Waiting to acquire a lock currently held by another thread.
   `public static final Thread.State BLOCKED`
4. **Waiting** — Waiting indefinitely due to `Object.wait()` (no timeout), `Thread.join()` (no timeout), or `LockSupport.park()`.
   `public static final Thread.State WAITING`
5. **Timed Waiting** — Waiting for a specified time due to `Thread.sleep()`, `Object.wait(timeout)`, `Thread.join(timeout)`, `LockSupport.parkNanos()`, or `LockSupport.parkUntil()`.
   `public static final Thread.State TIMED_WAITING`
6. **Terminated** — Thread has completed execution.
   `public static final Thread.State TERMINATED`

### Synchronization

Synchronization controls the execution of multiple threads so that shared resources are accessed in an orderly manner, avoiding conflicts and ensuring correct results.

- Controls access to shared resources.
- Avoids data inconsistency.
- Ensures proper execution of processes.

**Synchronized Methods**
Locks an entire method so only one thread can execute it at a time for a particular object. Uses the object-level (instance) lock. Safe, but may reduce performance due to full-method locking.

**Synchronized Blocks**
Locks only a specific section of code instead of the entire method, providing better performance through fine-grained control.

**Static Synchronization**
Locks at the class level instead of the object level, protecting static data/methods shared across all instances.

**Volatile Keyword**
Ensures all threads have a consistent view of a variable's value by preventing caching.
- Applies only to variables.
- Guarantees visibility — any write is immediately visible to other threads.
- Does **not** guarantee atomicity — operations like `count++` can still be inconsistent.

### Race Condition

A race condition occurs when two or more threads access the same shared resource simultaneously, and at least one modifies it, causing the final result to depend on execution order and potentially producing incorrect results.

### Deadlock

A deadlock occurs when two or more threads are permanently blocked because each is waiting for a resource held by another.

**Real-life example:**
- Person A has a Pen and needs a Notebook.
- Person B has a Notebook and needs a Pen.
- Without rules, both wait forever → Deadlock.

**How to Avoid Deadlock:**

1. **Acquire resources in the same order** — e.g., always take the Pen, then the Notebook. This creates waiting but no circular dependency, so no deadlock.
2. **Release resources quickly** — don't hold a lock longer than necessary.
3. **Don't wait forever (timeout)** — if a second resource isn't available in time, release the first and retry later instead of blocking permanently.

### Cooperation (Inter-thread Communication)

Allows multiple threads to coordinate execution by communicating instead of competing for a resource, using `wait()`, `notify()`, and `notifyAll()` from the `Object` class.

**Example:** In a restaurant scenario, the Cook thread prepares food while the Customer thread waits until it's ready; once done, the cook notifies the customer to proceed.

These methods must be called inside a synchronized block or method.

**Types of Thread Synchronization:**
- Mutual Exclusive: synchronized method, synchronized block, static synchronization.
- Cooperation (inter-thread communication).

### Executor Framework

A high-level API introduced in Java 5 (`java.util.concurrent`) to simplify thread management. Instead of manually creating threads with `new Thread()`, tasks are submitted to an Executor, which manages thread creation, scheduling, and reuse.

### Thread Pool

- **What is it?** A collection of reusable worker threads that execute submitted tasks, reducing the cost of repeatedly creating new threads.
- **Why better than new threads?** Thread creation is expensive; pooling reuses threads, improving performance and reducing memory usage.
- **What happens when all threads are busy?** New tasks queue up until a worker becomes available.
- **What is thread reuse?** After finishing a task, a worker doesn't terminate — it picks up the next task from the queue.
- **What if `shutdown()` isn't called?** The `ExecutorService` keeps running and its threads stay alive, which may prevent the JVM from exiting.

### Atomic Classes

Found in `java.util.concurrent.atomic` (e.g., `AtomicInteger`, `AtomicLong`). They offer atomic operations on variables without explicit synchronization, using the hardware's low-level atomic operations for thread safety.

### Daemon Threads

A daemon thread is a low-priority background thread that supports user threads (e.g., garbage collection, monitoring). Daemon threads run in the background and terminate automatically once all user threads finish.

- Marked using `setDaemon(boolean)`.
- Status checked using `isDaemon()`.

### Callable Interface

Part of `java.util.concurrent`, introduced in Java 5. Represents a task executed by a thread that can **return a result** and **throw checked exceptions** (unlike `Runnable`).

- Used with `ExecutorService` for asynchronous/concurrent execution.
- Result obtained via a `Future` object.
- Functional interface — supports lambda expressions.

**Example flow:** A single-threaded `ExecutorService` runs a `Callable` that prints "Calculating...", waits 1 second, and returns 20. It's submitted via `executor.submit(task)`, returning a `Future`. Calling `future.get()` waits for completion and retrieves the result (20). The executor is then shut down.

### Future Interface

Part of `java.util.concurrent`, introduced in Java 5. Represents the result of an asynchronous computation — a value available only after the task completes.

- Used to check task status (completed, running, cancelled).
- Retrieves the result of a `Callable` once done.
- Supports cancellation of still-running tasks.

**Example flow:** A single-threaded executor runs a task that sleeps 1 second and returns 50. Before completion, `isDone()` returns `false`; after `get()` retrieves the result, it returns `true`. The executor is then shut down.

### CompletableFuture

One of the most important concurrency features introduced in Java 8, used for asynchronous (non-blocking) programming — a task runs in the background while the main thread continues other work. It implements both `Future` and `CompletionStage`.

**Why introduced?** Before Java 8, `Future` had several limitations:
- `get()` blocks the current thread.
- Cannot chain multiple tasks.
- Cannot combine multiple futures easily.
- Poor exception handling.
- No callbacks on completion.

**Synchronous vs Asynchronous:**
- Synchronous: Task A → Task B → Task C (each waits for the previous).
- Asynchronous: Task A, B, and C run independently, in parallel.

**Example (e-commerce):**
Without async — fetching customer details (2s), orders (3s), recommendations (4s) sequentially takes 9s total.
With `CompletableFuture` — all three run simultaneously, taking ≈4s total.

**Future vs CompletableFuture:**

| Feature | Future | CompletableFuture |
|---|---|---|
| Asynchronous execution | ✅ | ✅ |
| Blocking | `get()` blocks | Can avoid blocking |
| Callback support | ❌ | ✅ |
| Chaining | ❌ | ✅ |
| Combine tasks | ❌ | ✅ |
| Exception handling | Poor | Excellent |
| Manual completion | ❌ | ✅ |

**Creating a CompletableFuture:**

1. `completedFuture()` — already completed.
   ```java
   CompletableFuture<String> future = CompletableFuture.completedFuture("Hello");
   System.out.println(future.get()); // Hello
   ```
2. `runAsync()` — runs a task with no return value (`CompletableFuture<Void>`). Good for sending email, logging, notifications.
   ```java
   CompletableFuture.runAsync(() -> System.out.println("Running..."));
   ```
3. `supplyAsync()` — runs a task and returns a value.
   ```java
   CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "Java");
   ```

**Default Thread Pool:** If no executor is provided, Java uses `ForkJoinPool.commonPool()`. You can supply your own:
```java
ExecutorService executor = Executors.newFixedThreadPool(5);
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "Hello", executor);
```

**Getting the Result:**
- `get()` — blocks; throws checked exceptions (`InterruptedException`, `ExecutionException`).
- `join()` — also blocks, but throws an unchecked `CompletionException`. Most developers prefer `join()` in application code.

**Key chaining/combining methods:**
- `thenApply()` — transforms the result. `supplyAsync(() -> "java").thenApply(String::toUpperCase)` → `"JAVA"`.
- `thenAccept()` — consumes the result, returns nothing.
- `thenRun()` — runs another task without receiving the previous result.
- `thenCompose()` — chains dependent asynchronous operations (returns a `CompletableFuture`, unlike `thenApply` which returns a plain value).
- `thenCombine()` — combines two independent futures into a single result.
- `allOf()` — waits for all given futures to complete.
- `anyOf()` — returns as soon as the first future completes.
- `exceptionally()` — handles exceptions and supplies a fallback value.
- `handle()` — handles both success and failure in one callback.
- `whenComplete()` — executes after completion; cannot change the result.
- `complete()` — manually completes a future.
- `cancel()` — cancels execution.

**Typical execution flow:**
`supplyAsync()` → `thenApply()` → `thenCompose()` → `thenCombine()` → `exceptionally()` → `join()`

**Real-time example:** An online shopping order page needs customer details (2s), order details (3s), payment details (4s), and product reviews (2s). Sequentially this is 11s; run in parallel with `CompletableFuture`, the total is only as long as the slowest task (≈4s).

**Where it's used:** REST API calls, microservices, Spring Boot apps, database queries, file processing, email sending, report generation, payment gateways, notification services, parallel data fetching, background tasks.

**Advantages:** Non-blocking, better CPU utilization, easy chaining and combining, powerful exception handling, improved responsiveness, cleaner code than nested callbacks.

**Disadvantages:** More complex than plain threads, harder to debug, misuse of `get()`/`join()` can negate benefits, requires careful thread pool management under load.

### Java Virtual Threads (Java 21)

Virtual threads are a lightweight thread implementation, standardized in Java 21 (previewed earlier), that let Java applications create millions of concurrent threads without consuming large amounts of OS resources. They make concurrent programming simpler and more scalable, especially for I/O-heavy workloads (database calls, REST APIs, file access, network communication).

**Why introduced?** Before Java 21, Java used **platform threads**, each mapped 1:1 to an OS thread. Creating an OS thread is expensive (≈1 MB stack memory, OS scheduling, context switching, kernel resources). Thousands of threads meant high memory usage, high context-switching overhead, and poor performance.

**Key points:**
- Virtual threads are lightweight, JVM-managed threads that allow millions of concurrent tasks using only a small number of underlying OS threads.
- They solve the scalability limitations of platform threads.
- Platform threads map 1:1 to OS threads; virtual threads are scheduled by the JVM onto a much smaller set of platform threads.
- They benefit applications with heavy blocking I/O — web servers, REST APIs, database-driven systems, microservices.
- Millions of virtual threads can be created, limited mainly by available memory and workload rather than OS thread limits.

### ExecutorService

An interface in `java.util.concurrent` used to manage and execute multiple tasks asynchronously using a pool of threads. Instead of creating a new thread per task, it maintains a thread pool, assigns tasks to available threads, and returns threads to the pool once finished — improving performance, reducing memory use, and simplifying thread management.

**Why introduced?** Before Java 5, threads were created manually via the `Thread` class, which was expensive, memory-heavy, hard to manage, and didn't allow reuse. The Executor Framework (with `ExecutorService` at its core) solved these problems.

**How it works:**
`Task Submitted → ExecutorService → Thread Pool → Available Worker Thread → Task Execution → Thread Returns to Pool`

**Main responsibilities:**
- Manage thread creation and reuse existing threads.
- Execute tasks asynchronously.
- Queue tasks when all threads are busy.
- Return results through `Future`.
- Manage thread lifecycle and graceful shutdown.

**Types of ExecutorService:**

1. **Fixed Thread Pool** — fixed number of worker threads; extra tasks wait in a queue. Best for known workloads.
2. **Single Thread Executor** — one thread executes tasks one after another, preserving order.
3. **Cached Thread Pool** — creates new threads as needed and reuses idle ones; suited for many short-lived tasks.
4. **Scheduled Thread Pool** — executes tasks after a delay or periodically.
5. **Virtual Thread Per Task Executor (Java 21)** — creates one virtual thread per submitted task; best for highly concurrent I/O-bound applications.

**Advantages:** Improves performance via thread reuse, reduces memory usage, simplifies multithreading, better resource management, supports async programming, improves scalability, built-in scheduling/lifecycle management.

**Disadvantages:** Incorrect pool sizing can hurt performance; deadlocks are still possible with poorly designed tasks; the executor must be shut down manually; debugging concurrent tasks can be difficult.

**Where used:** Web servers, REST APIs, Spring Boot apps, database operations, file processing, email services, background jobs, batch processing, microservices.

---

## Design Patterns

### Singleton Pattern

Ensures a class has only one instance and provides a global access point to it. Used for centralized control of resources like database connections, configuration settings, or logging.

- Prevents accidental creation of multiple instances; ensures efficient use of memory and connections.
- Simplifies coordination by providing a single shared instance.
- **Example:** A Database Connection Manager uses one shared instance instead of creating multiple connections.

**Real-world applications:** Logging systems, configuration managers, database connections, thread pools.

**Features:** Single instance, global access point, lazy or eager initialization, can be made thread-safe, useful for shared-resource management.

```java
class Singleton {
    private static Singleton instance;

    private Singleton() {
        // Initialization code here
    }

    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

**Thread-safety issue with the naive version:**
1. Thread 1 checks `instance == null` → true (no object yet).
2. Thread 1 is paused (context switch) before creating the object.
3. Thread 2 checks `instance == null` → also true.
4. Thread 2 creates a new Singleton object.
5. Thread 1 resumes, unaware an instance now exists, and creates another one — breaking the singleton guarantee.

**Advantages:** Prevents inconsistent states, saves memory/resources, provides a global access point.
**Disadvantages:** Creates tight coupling, difficult to unit test due to shared global state, can cause issues in multithreaded environments if not implemented carefully.

### Factory Method Pattern

A creational pattern that defines an interface for creating objects but lets subclasses decide which class to instantiate. Promotes loose coupling by delegating object creation to a method.

- Subclasses override the factory method to create specific object types.
- Supports easy addition of new product types and improves maintainability.
- **Example:** A Notification System that creates Email, SMS, or Push notification objects via a Factory Method instead of direct instantiation.

**Real-world applications:** Web browsers (plugin/renderer creation), Android activity instantiation, payment gateways (Stripe, PayPal), game development (spawning enemies/items/NPCs).

**Components:**
- **Creator** — abstract class/interface declaring the factory method.
- **Concrete Creator** — subclass implementing the factory method for a specific product.
- **Product** — interface/abstract class for created objects.
- **Concrete Product** — actual object implementing the Product interface.

**Example problem:** A vehicle application needs to handle Two Wheelers, Three Wheelers, and Four Wheelers, each with distinct properties.

**Issues with direct creation:** Tight coupling to product classes, violation of Single Responsibility Principle, hard to extend without modifying the client.

**Solution:** Define a `VehicleFactory` interface; implement concrete factories like `TwoWheelerFactory` and `FourWheelerFactory`; have the client depend on `VehicleFactory` instead of concrete classes, so new vehicle types can be added without altering client code.

**Advantages:** Encapsulates creation logic, promotes loose coupling, supports scalability/extensibility, improves reusability and maintainability.
**Disadvantages:** Increases number of classes, adds complexity, may be unnecessary for small/simple applications.

### Prototype Pattern

A creational pattern used when object creation is time-consuming or costly — new objects are created by cloning existing ones rather than building from scratch.

- The cloned object can modify only the properties it needs, leaving the original untouched.
- Saves time and resources for expensive or complex object creation.
- The `clone()` method is a simple way to implement this, though cloning logic depends on business requirements.
- **Example:** A Game Character System where creating a character involves loading graphics, skills, and configurations — cloning an existing character is far cheaper.

**Real-world applications:**
- **Document/Content Management Systems** — existing templates can be cloned to quickly create new documents, then have layout/fonts/content modified.
- **Game Engines** — complex characters or environment objects are cloned instead of reinitialized, improving runtime performance.

**Components:** Prototype interface/abstract class (defines `clone()`), Concrete Prototype (implements cloning), Client (uses the prototype to create objects), Clone Method (defines how copying works).

**When to use:** Object creation is time-consuming/resource-intensive; many similar objects with slight variations are needed; initialization involves database calls or heavy computation.

**Advantages:** Simplifies object creation, reduces subclassing, allows dynamic addition/removal of object types at runtime, promotes flexibility.
**Disadvantages:** Cloning complex objects can be difficult, deep-copy implementation can be complicated, requires careful handling of references to avoid shared-state issues, every class must implement cloning logic properly.

### Facade Pattern

A structural pattern that provides a simple, unified interface to a complex subsystem, hiding internal complexity to make the system easier to use and maintain.

- Structuring a system into subsystems reduces overall complexity and improves organization.
- The Facade introduces a single entry-point object providing a simplified interface to the subsystem.
- **Example:** In a home automation system, a user controls lights, AC, and security cameras through one app/panel — the control system acts as a Facade hiding the complexity of multiple devices.

**Real-world applications:** Home automation systems, video streaming platforms (unified interface for encoding, buffering, playback), banking systems (simple balance-check/transfer methods hiding complex backend operations).

**Advantages:** Simplified interface, reduced coupling, encapsulation of subsystem changes, improved maintainability.
**Disadvantages:** Adds an extra abstraction layer (can complicate debugging), limits direct access to specific subsystem functionality, potential overengineering for simple systems, possible performance overhead from indirection.

### Strategy Pattern

*(Last updated: 14 May 2026)*

A behavioral pattern that defines a family of related algorithms, encapsulates each in its own class, and makes them interchangeable — allowing the algorithm to vary independently from the client using it, with behavior changes possible at runtime without altering existing code.

- Encapsulates different algorithms into separate strategy classes for dynamic selection/switching at runtime.
- Reduces complex conditional logic and improves maintainability.
- **Example:** A payment system where a user selects a payment method, and the system applies a strategy — Credit Card, UPI, or PayPal — to process the payment.

### Observer Pattern

*(Last updated: 7 Jun 2026)*

A behavioral pattern that creates a one-to-many relationship between a subject and its observers. When the subject's state changes, all dependent observers are notified and updated automatically, ensuring synchronized communication.

- Enables automatic updates to multiple objects when one object changes — useful for event-driven or publish-subscribe systems.
- Promotes loose coupling between subject and observers, improving flexibility and maintainability.
- **Example:** A YouTube channel (Subject) notifies all its subscribers (Observers) whenever a new video is uploaded.

**Real-world applications:**
- **Social Media Notifications** — users are notified when someone they follow posts new content.
- **Stock Market Apps** — investors get real-time updates when stock prices change.
- **GUI Event Listeners** — UI components respond to clicks or keyboard input.
- **Weather Monitoring Systems** — multiple displays update automatically when central weather data changes.

---

## SOLID Principles

### 1. Single Responsibility Principle (SRP)

A class should have only one reason to change — i.e., a single responsibility, job, or purpose.

**Example:** A baker's role is to focus solely on baking bread, ensuring quality and consistency, without taking on unrelated responsibilities.

### 2. Open/Closed Principle (OCP)

Software entities (classes, modules, functions) should be open for extension but closed for modification — you should be able to extend behavior without modifying existing code.

**Example:** A `PaymentProcessor` class initially supports only credit card payments. To add PayPal support, you extend its functionality rather than modifying the existing credit card logic.

### 3. Liskov Substitution Principle (LSP)

Introduced by Barbara Liskov in 1987: derived/child classes must be substitutable for their base/parent classes without causing unexpected behavior.

**Example:** A rectangle has variable width and height; a square is a rectangle with equal width and height. This shows how care is needed when extending a Rectangle class into a Square class so substitution doesn't break expected behavior.

### 4. Interface Segregation Principle (ISP)

Similar in spirit to SRP but applied to interfaces: clients should not be forced to depend on methods irrelevant to them. Prefer several small, client-specific interfaces over one large "fat" interface.

**Example:** A pure vegetarian customer at a restaurant shouldn't be handed a menu that mixes vegetarian items, non-vegetarian items, drinks, and sweets into a single undivided list — separate, focused menus serve each need better.

### 5. Dependency Inversion Principle (DIP)

High-level modules should not depend on low-level modules — both should depend on abstractions. Abstractions should not depend on details; details should depend on abstractions.

In simpler terms: classes should rely on interfaces/abstract classes rather than concrete implementations, allowing more flexible, decoupled code where implementations can change without affecting other parts of the codebase.

**Example:** A software development team depends on an abstract version control system (e.g., Git) to manage code changes, without depending on the specific internal details of how Git works.

---

## Spring Framework

### Beans and Bean Scopes

In Spring, a **bean** is an object that is instantiated, assembled, and managed by the Spring IoC container. Beans are the backbone of a Spring application, representing the various components that make up its business logic.

**Bean scopes** define the lifecycle and visibility of beans within the Spring context:

- **Singleton** *(default)* — a single instance per Spring IoC container.
- **Prototype** — a new instance is created every time the bean is requested.
- **Request** *(web)* — a single instance per HTTP request.
- **Session** *(web)* — a single instance per HTTP session.
- **Application** *(web)* — a single instance per `ServletContext`.
- **WebSocket** — a single instance per WebSocket.

**Lifecycle callbacks by scope:**
- **Singleton** — Spring calls `@PostConstruct` and `@PreDestroy`.
- **Prototype** — Spring calls `@PostConstruct` but *not* `@PreDestroy`; you must clean up prototypes yourself.
- **Request/Session** — lifecycle tied to the HTTP request/session; `@PreDestroy` is called when the request/session ends.

**Typical use cases:**

| Scope | Example use |
|---|---|
| Singleton | Services, DAOs, controllers (stateless) |
| Prototype | Objects with per-use state (e.g., builder objects, input processors) |
| Request | Per-request context, e.g., request-scoped metrics or locale collectors |
| Session | Shopping cart, user preferences during login session |
| Application | Shared resources tied to `ServletContext` (rare) |
| WebSocket | Per WebSocket user connection state |

```java
@Component
@Scope("prototype")
public class MyPrototypeBean {}

@Component
@Scope("singleton")
public class MySingletonBean {}

@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestBean {}
```

> **Interview soundbite:** "I mostly use the singleton scope because most services in Spring Boot are stateless. I use prototype when I need new independent instances, and I use request/session scopes in web apps when managing request-level or user-session-level data."

### Configuration

Configuration means telling Spring how to create beans and manage dependencies. Spring supports three major approaches: **XML**, **Java-based** (`@Configuration`), and **annotation-based** (stereotypes). Spring Boot mainly uses Java and annotations.

1. **Java-Based Configuration** *(most important in Spring Boot)*
   - Uses `@Configuration` and `@Bean`.
   - A class annotated with `@Configuration` tells Spring: "this class contains bean definitions."
   - Methods annotated with `@Bean` return objects managed by Spring.
2. **Annotation-Based Configuration (Component Scanning)**
   - Uses `@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController`.
   - Spring scans packages and automatically creates beans.
3. **XML-Based Configuration** *(legacy, rarely used now)* — only found in very old Spring projects.
4. **Spring Boot Auto-Configuration**
   - Uses `@SpringBootApplication` and `@EnableAutoConfiguration`.
   - Spring Boot automatically creates beans based on the classpath and configuration.
5. **Property-Based Configuration** *(externalized config)* — values injected from `application.properties` / `application.yml` using `@Value` and `@ConfigurationProperties`.
6. **Conditional Configuration** — loads beans only if a condition matches, e.g. `@ConditionalOnClass`, `@ConditionalOnProperty`, `@Profile`.

> **Interview soundbite:** "Spring provides multiple ways of configuration: Java Configuration (type-safe, preferred in Spring Boot), annotation-based (auto component scanning), XML (old style, rarely used today), auto-configuration (Boot creates beans based on classpath), property-based configuration, and conditional/profile-based configuration for environment-specific beans."

### IoC (Inversion of Control)

IoC is a design principle where the control of creating objects and managing their dependencies is given to the Spring Container instead of the developer.

- You don't create objects manually using `new`.
- Spring creates them for you and injects the required dependencies automatically.

**Before IoC:**
```java
UserService service = new UserService(new UserRepository());
```

**With IoC (Spring):**
```java
@Service
public class UserService {
    @Autowired
    UserRepository repository;
}
```

Spring creates `UserRepository`, creates `UserService`, and injects the repository into the service — `new` is never used directly.

**How IoC is implemented:**
- **`BeanFactory`** — basic IoC container.
- **`ApplicationContext`** — advanced IoC container (used by Spring Boot), adding bean creation, dependency injection, bean lifecycle, AOP, event handling, and property binding.

> **Interview soundbite:** "IoC in Spring means the control of object creation is inverted from the application code to the Spring container. Instead of using `new`, Spring creates objects (beans) and injects dependencies automatically, using the IoC container (`ApplicationContext`) to manage the full bean lifecycle. IoC is implemented through Dependency Injection — constructor, setter, or field injection."

### Dependency Injection (DI)

DI is a design pattern where the dependencies of a class are provided by Spring instead of the class creating them itself.

**Manual way (no DI):**
```java
EmailService service = new EmailService(new SMTPClient());
```

**With Spring DI:**
```java
@Service
public class EmailService {
    private final SMTPClient smtp;

    @Autowired
    public EmailService(SMTPClient smtp) {
        this.smtp = smtp;
    }
}
```

**Types of Dependency Injection:**

1. **Constructor Injection** *(recommended)*
   ```java
   public MyService(MyRepo repo) { ... }
   ```
2. **Setter Injection**
   ```java
   @Autowired
   public void setRepo(MyRepo repo) { ... }
   ```
3. **Field Injection** *(not recommended)*
   ```java
   @Autowired
   private MyRepo repo;
   ```

### BeanFactory vs ApplicationContext

| | `BeanFactory` | `ApplicationContext` |
|---|---|---|
| Features | Basic IoC container, lazy loading | All `BeanFactory` features, plus eager loading, AOP support, event listeners, message sources, environment & profiles |

**When to use which?**
- **`BeanFactory`** — only in lightweight or memory-sensitive applications; rare in modern Spring Boot.
- **`ApplicationContext`** *(default in Boot)* — enterprise apps, web apps, REST APIs, AOP-required or event-driven apps. Spring Boot always initializes `ApplicationContext`, never `BeanFactory`.

> **Interview soundbite:** "`BeanFactory` is the basic IoC container, whereas `ApplicationContext` is a more advanced container with features like AOP, events, i18n, auto-wiring, and environment support. Spring Boot always uses `ApplicationContext`."

### Component Scanning

Component scanning is the mechanism Spring uses to automatically discover and register beans by scanning classpath packages for classes annotated with stereotype annotations such as `@Component`, `@Service`, `@Repository`, `@Controller`, and other meta-annotated types.

**Why use it?**
- Removes manual `@Bean` boilerplate for most beans.
- Keeps configuration declarative and modular.
- Encourages convention-over-configuration (package structure matters).

**How it's triggered:**
- Via `@ComponentScan` on a `@Configuration` class.
- Implicitly via `@SpringBootApplication`, which includes `@ComponentScan`.
- In legacy XML: `<context:component-scan base-package="..."/>`.

If `@SpringBootApplication` is placed on `com.example.app.Application`, Spring Boot scans `com.example.app` and all subpackages — which is why the main class is normally in the top-level package.

```java
// explicit component scan
@Configuration
@ComponentScan(basePackages = "com.example.services")
public class AppConfig {}

// multiple packages
@ComponentScan(basePackages = {"com.example.services", "com.example.util"})

// preferred style — refactoring-safe
@ComponentScan(basePackageClasses = { SomeMarkerClass.class })
public class AppConfig {}
```

**What gets detected by default:**
`@Component`, `@Service`, `@Repository`, `@Controller` / `@RestController` (all meta-annotated with `@Component`), `@Configuration` (treated specially), and any custom annotation meta-annotated with `@Component`.

**Custom stereotypes:**
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Component
public @interface MyService { }
```

**Filters — include/exclude:**
```java
@ComponentScan(
  basePackages = "com.example",
  includeFilters = @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = Special.class),
  excludeFilters = @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE, classes = UnwantedService.class)
)
```

`FilterType` options: `ANNOTATION`, `ASSIGNABLE_TYPE`, `ASPECTJ`, `REGEX`, `CUSTOM` (via a custom `TypeFilter` implementation).

**Scoped proxies and scanning:** if a scanned bean is request/session scoped and injected into a singleton, use a proxy mode:
```java
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestBean {}
```

**Component scan + `@Configuration` interplay:** `@Configuration` classes are themselves registered as bean definitions, and by default use CGLIB proxying to ensure `@Bean` methods return singletons.

**Common pitfalls:**
- Placing the main class outside the root package (so subpackages aren't scanned).
- Brittle string-based package names — prefer `basePackageClasses` with marker classes.
- Duplicate beans — resolve with `@Primary` or `@Qualifier`.
- Unintended component discovery from libraries/tests — use `excludeFilters`.
- Large scanned package trees increasing startup time.
- Field injection silently masking circular dependencies — prefer constructor injection, which surfaces circular references as errors.

**Restricting to certain stereotypes:**
```java
@ComponentScan(
  basePackages = "com.example",
  includeFilters = @Filter(type = FilterType.ANNOTATION, classes = Service.class),
  useDefaultFilters = false
)
```

**When *not* to use component scanning:** for a small number of beans that are configuration-heavy or created conditionally with complex constructor logic, or for third-party classes you can't annotate — prefer explicit `@Bean` in `@Configuration`.

**Testing tips:** use `ApplicationContextRunner` (or `@SpringBootTest`) plus `getBeanNamesForType(...)` / `getBeanDefinitionNames()` to inspect what's registered.

**Final summary:** Component scanning is Spring's automatic bean-discovery system. Configure it via `@ComponentScan` (or implicitly via `@SpringBootApplication`), refine it with filters, and prefer `basePackageClasses` over string paths.

### Bean Lifecycle

Understanding the bean lifecycle helps you handle initialization/destruction properly, write custom init/destroy logic, and understand when beans are created and injected.

**Lifecycle steps:**

1. **Instantiation** — Spring creates the bean instance (via constructor or factory).
2. **Populate properties** — Spring injects dependencies (`@Autowired`, setter, constructor).
3. **`BeanNameAware` / `BeanFactoryAware` / `ApplicationContextAware`** — optional callbacks.
4. **Pre-initialization** (`BeanPostProcessor` before-init) — custom logic can be applied.
5. **Initialization** — `@PostConstruct` or a custom `init-method` executes.
6. **Post-initialization** (`BeanPostProcessor` after-init) — further processing possible.
7. **Bean is ready to use** — available for injection/use.
8. **Destruction** — `@PreDestroy` or a custom `destroy-method` executes before container shutdown.

**Annotations/interfaces to control lifecycle:**

| Feature | Usage |
|---|---|
| `@PostConstruct` | Method called after dependency injection |
| `@PreDestroy` | Method called before the bean is destroyed |
| `InitializingBean` | Implement `afterPropertiesSet()` for init logic |
| `DisposableBean` | Implement `destroy()` for cleanup |
| `init-method` / `destroy-method` | XML/`@Bean` way to specify lifecycle methods |

```java
@Component
public class MyBean {

    @PostConstruct
    public void init() {
        System.out.println("Bean is initialized");
    }

    @PreDestroy
    public void destroy() {
        System.out.println("Bean is destroyed");
    }
}

@Configuration
public class AppConfig {

    @Bean(initMethod = "customInit", destroyMethod = "customDestroy")
    public AnotherBean anotherBean() {
        return new AnotherBean();
    }
}
```

**Key interview points:**
- Spring calls `@PostConstruct` after dependencies are injected.
- Lifecycle differs for singleton vs. prototype: singletons are initialized at container startup and destroyed at shutdown; prototypes are created on request, and Spring does not manage their destruction.
- `BeanPostProcessor` is powerful — it can intercept bean creation before/after initialization.

### AOP (Aspect-Oriented Programming)

AOP is a programming paradigm that separates cross-cutting concerns (like logging, security, transaction management) from business logic, modularizing concerns that cut across multiple classes and methods and reducing duplication.

**Key terminology:**
- **Aspect** — a module encapsulating a cross-cutting concern (e.g., `LoggingAspect`).
- **Join Point** — a point in program execution where an aspect can be applied (e.g., method execution, exception handling).
- **Advice** — the action taken by an aspect at a join point (`Before`, `After`, `Around`, etc.).
- **Pointcut** — an expression matching join points where advice should be applied.
- **Target Object** — the object being advised by aspects.
- **Weaving** — the process of linking aspects with application objects.

**Types of advice:**
- `@Before` — executes before method execution.
- `@After` — executes after method execution (regardless of outcome).
- `@AfterReturning` — executes after successful method execution.
- `@AfterThrowing` — executes if the method throws an exception.
- `@Around` — wraps the method execution; the most powerful advice type.

**Enabling AOP in Spring Boot:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```
Then annotate the main class or a configuration class with `@EnableAspectJAutoProxy`.

**Spring AOP vs AspectJ:**

| | Spring AOP | AspectJ |
|---|---|---|
| Weaving | Runtime, proxy-based | Compile-time / load-time |
| Join points | Method execution only | All join points (field access, constructor calls, etc.) |
| Complexity | Easier to use | More powerful, more complex |
| Scope | Spring beans only | Any Java object |

**Pointcut expressions** define where advice applies — common patterns include `execution` expressions for method matching, `within` for type matching, `@annotation`-based matching, and `bean()` name patterns.

**Limitations of Spring AOP:**
- Only applies to Spring beans.
- Only supports method-level join points.
- Proxy-based approach means self-invocation doesn't trigger aspects.
- Cannot advise `final` classes or methods.
- Slightly lower performance due to runtime proxies.

**JDK Dynamic Proxy vs CGLIB Proxy:**
- **JDK Dynamic Proxy** — used when the target implements an interface; creates a proxy implementing the same interface; faster to create.
- **CGLIB Proxy** — used when the target doesn't implement an interface; creates a subclass of the target; slightly slower but more flexible.

**`@Around` advice** wraps the target method completely, receives a `ProceedingJoinPoint` parameter, and can control whether to proceed with execution, modify arguments/return values, handle exceptions, and measure execution time.

**Ordering multiple aspects:** use `@Order` (lower values = higher priority) or implement the `Ordered` interface. Higher-priority aspects execute first for `@Before` advice and last for `@After` advice.

**Self-invocation problem:** when a method within the same class calls another method, the call bypasses the proxy (it happens on `this`, not the proxy), so aspects won't be triggered. Solutions: inject a self-reference, use AspectJ weaving, or restructure code to call through another bean.

**Accessing method parameters in advice:** use the `JoinPoint` parameter to get the method signature and arguments, bind parameters via pointcut expressions with `args()`, or use `@annotation` with custom annotations that expose parameters.

---

### Spring Security

#### Authentication and Authorization Flow

```
User → Filter → AuthenticationManager → AuthenticationProvider → UserDetailsService
     → PasswordEncoder → SecurityContext → Authorization → Controller → Response
```

Authentication in Spring Boot is the process of verifying a user's identity using Spring Security's filters, `AuthenticationManager`, `UserDetailsService`, and `PasswordEncoder`. Once verified, the user info is stored in `SecurityContext` for the rest of the request.

**Authentication** answers "who is this user?" — **Authorization** answers "what can this user access?" (checked against the user's roles/authorities: allow if sufficient, else `403 Forbidden`).

#### AuthenticationManager

The core engine of Spring Security that performs the actual authentication process — it decides whether the user's credentials are valid.

- Receives an `Authentication` object (username/password, or token).
- Forwards it to the correct `AuthenticationProvider`.
- If valid → returns an authenticated `Authentication` object.
- If invalid → throws an exception.

#### AuthenticationProvider

The component that actually checks the user's credentials and decides if authentication is successful — it does the real validation work, while `AuthenticationManager` just delegates to it.

#### SecurityFilterChain

The modern way to configure Spring Security (replacing the older `WebSecurityConfigurerAdapter`). Every request passes through a chain of security filters that check authentication, authorization, sessions, CSRF, CORS, and token validation. The `SecurityFilterChain` bean lets you customize this chain — e.g., allow `/public/**` without authentication, restrict `/admin/**` to the `ADMIN` role, and require login for everything else, all via the fluent `HttpSecurity` API.

This approach is modular, clear, and avoids method overriding — making it easier to add custom behavior like JWT validation via `addFilterBefore` / `addFilterAfter`.

#### UserDetailsService

Loads user-specific data from a source — database, in-memory, LDAP, or an API. It does **not** perform authentication itself; it only supplies the user details needed for authentication.

- Spring Security calls `loadUserByUsername()`, which must return a `UserDetails` object (username, encrypted password, roles/authorities).
- If the user doesn't exist, it throws `UsernameNotFoundException`.

**In simple terms:** `UserDetailsService` loads user data → `AuthenticationProvider` validates the credentials → `AuthenticationManager` coordinates the process.

#### PasswordEncoder

Responsible for encrypting and validating passwords, ensuring passwords are never stored or compared in plain text — they're always stored in a one-way hashed format.

- On registration, the raw password is hashed before being saved.
- On login, the raw password is hashed the same way and compared to the stored hash.
- The recommended encoder is `BCryptPasswordEncoder`, which adds a random salt and performs multiple hashing rounds, making brute-force/rainbow-table attacks impractical.
- `PasswordEncoder` never decrypts — hashing is one-way; there's no "decode" step.
- Modern Spring Security refuses plain-text passwords outright.

#### In-Memory Authentication

Users, passwords, and roles are stored directly in application memory (RAM) rather than a database; data is lost on restart. Mainly used for learning, testing, or small internal tools.

Configured via an `InMemoryUserDetailsManager` bean, still requiring a `PasswordEncoder` to hash passwords.

```java
@Bean
public UserDetailsService userDetailsService(PasswordEncoder encoder) {
    UserDetails user = User.withUsername("user")
            .password(encoder.encode("user123"))
            .roles("USER")
            .build();
    UserDetails admin = User.withUsername("admin")
            .password(encoder.encode("admin123"))
            .roles("ADMIN")
            .build();

    return new InMemoryUserDetailsManager(user, admin);
}
```

#### Database Authentication

Credentials (username, password, roles) are stored in a database — the standard, recommended approach for real applications.

- Create a `User` entity mapped to a table and a `UserRepository` to fetch users.
- Implement `UserDetailsService` to load a user by username and convert it into a `UserDetails` object.
- A `PasswordEncoder` (usually BCrypt) hashes passwords on save and validates them on login.
- Flow: request → filters → `AuthenticationManager` → `AuthenticationProvider` → `UserDetailsService` loads the user → password verified → authenticated user stored in `SecurityContext` → authorization rules applied.

#### JWT (JSON Web Token) Authentication

JWT is a compact, secure token used to authenticate and authorize users in **stateless** applications — no server-side session is stored.

**Structure** (three parts separated by dots):
- **Header** — token type (JWT) and signing algorithm (e.g., HS256, RS256).
- **Payload** — claims such as user id, username, roles, and expiration time.
- **Signature** — created with a secret/private key to detect tampering.

**Flow:**
1. User logs in with username/password; Spring Security authenticates via `UserDetailsService` + `PasswordEncoder`.
2. On success, the server generates a JWT containing user info and expiration, and sends it to the client.
3. The client sends the token on every request via the `Authorization: Bearer <JWT_TOKEN>` header.
4. A JWT filter (typically extending `OncePerRequestFilter`) intercepts each request, validates the token's signature/expiration, and if valid, populates the `SecurityContext`.

**`SecurityFilterChain` for JWT typically:**
- Disables CSRF.
- Uses stateless session management.
- Permits login endpoints.
- Adds the JWT filter before `UsernamePasswordAuthenticationFilter`.

If the token is invalid or missing, the request is rejected.

#### OAuth2 in Spring Boot

A standard authorization framework letting users log in via a trusted external provider (Google, GitHub, Facebook, Okta, etc.) without sharing passwords with your application.

**Flow:** user clicks "Login with Google" → redirected to Google's login page → Google returns an authorization code → Spring Security exchanges the code for an access token, verifies it, and creates an authenticated user.

**Benefits:** your app never sees the user's password; authentication is delegated to trusted providers; tokens are used instead of credentials.

**Used for:**
- Social login (OAuth2 Client).
- Securing APIs using access tokens (OAuth2 Resource Server).

**Implementation steps:**
1. Register your app with the provider to get a client ID and secret.
2. Add the OAuth2 client dependency and configure provider details in `application.yml`.
3. Enable `oauth2Login()` in the `SecurityFilterChain`.
4. Spring Security handles the redirect, token exchange, and user authentication automatically.

#### CORS (Cross-Origin Resource Sharing)

A browser security rule controlling whether a web app on one origin (domain/port/protocol) can access resources on another origin. Browsers block cross-origin requests by default — e.g., a frontend on `localhost:3000` can't call a backend on `localhost:8080` unless CORS explicitly allows it.

Spring Boot's CORS configuration specifies allowed origins, HTTP methods, headers, and whether credentials (cookies, auth headers) are allowed. For cross-origin requests, the browser may first send a **preflight** `OPTIONS` request, and Spring Boot responds with headers like `Access-Control-Allow-Origin`.

#### CSRF (Cross-Site Request Forgery)

Enabled by default in Spring Security to protect session-based applications. For REST APIs, JWT-based auth, or stateless services, CSRF protection is usually unnecessary and commonly disabled.

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf.disable())
        .authorizeHttpRequests(auth -> auth
            .anyRequest().authenticated()
        );
    return http.build();
}
```

**Safe to disable CSRF for:** stateless REST APIs, JWT-based auth, mobile/SPA backends, OAuth2 resource servers.
**Do *not* disable CSRF for:** traditional web apps using sessions + cookies, form-based login applications.

#### Method-Level Security

- **`@PreAuthorize`** — checks access *before* method execution.
- **`@PostAuthorize`** — checks access *after* method execution.

Both control access to methods (not URLs).

---

### Spring Boot Caching

#### What is Spring Boot Cache?

A mechanism to store frequently accessed data in memory (or another fast store) so repeated requests can be served quickly without hitting the database or re-running expensive computations. Spring Boot provides a **cache abstraction**, so business code is independent of the actual cache provider — you can start with simple in-memory caching and later plug in Redis, EhCache, Caffeine, or Hazelcast without changing business logic.

#### `@EnableCaching`

Activates Spring's caching support. Without it, `@Cacheable`, `@CachePut`, and `@CacheEvict` do nothing. Once enabled, Spring creates proxies around beans that intercept method calls and apply caching logic before/after execution. Usually placed on the main application class or a configuration class.

#### Cache Annotations

**`@Cacheable`** — stores the method result in the cache; if called again with the same parameters, the cached result is returned instead of re-executing the method.
```java
@Cacheable(value = "users", key = "#id")
public User getUserById(Long id) {
    return userRepository.findById(id).get();
}
```

**`@CacheEvict`** — removes data from the cache, used when cached data becomes outdated.
```java
@CacheEvict(value = "users", key = "#id")
public void deleteUser(Long id) {
    userRepository.deleteById(id);
}
// clear the entire cache:
@CacheEvict(value = "users", allEntries = true)
```

**`@CachePut`** — updates the cache without skipping method execution; the method always runs and its result is placed into the cache.
```java
@CachePut(value = "users", key = "#user.id")
public User updateUser(User user) {
    return userRepository.save(user);
}
```

**`@Caching`** — allows multiple caching rules on the same method.
```java
@Caching(
    cacheable = { @Cacheable(value = "users", key = "#id") },
    evict = { @CacheEvict(value = "usersList", allEntries = true) }
)
public User getUser(Long id) {
    return userRepository.findById(id).get();
}
```

**`@CacheConfig`** — defines common cache configuration at the class level, avoiding repetition of cache names in every method.
```java
@CacheConfig(cacheNames = "users")
@Service
public class UserService {

    @Cacheable(key = "#id")
    public User getUser(Long id) { }

    @CacheEvict(key = "#id")
    public void deleteUser(Long id) { }
}
```

#### Configuring Redis Cache

Redis is an external, fast in-memory data store commonly used in production because it's fast, scalable, and shared across multiple application instances.

1. **Add dependencies:**
   ```xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-data-redis</artifactId>
   </dependency>
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-cache</artifactId>
   </dependency>
   ```
2. **Configure the connection** in `application.yml`/`application.properties`:
   ```properties
   spring.redis.host=localhost
   spring.redis.port=6379
   ```
3. **Enable caching:**
   ```java
   @SpringBootApplication
   @EnableCaching
   public class Application {
   }
   ```
4. **Redis is auto-configured as the cache provider** when its dependency is present (optionally customize with a `RedisCacheManager` bean).
5. **Use cache annotations** as usual — first call hits the DB, subsequent calls return data from Redis.

#### Configuring EhCache

EhCache is a Java-based in-memory caching provider, commonly used for fast caching inside a single application instance.

1. **Add dependencies** (`spring-boot-starter-cache` + `org.ehcache:ehcache`).
2. **Create `ehcache.xml`** in `src/main/resources` defining cache name, size, and expiration:
   ```xml
   <config xmlns="http://www.ehcache.org/v3">
       <cache alias="users">
           <key-type>java.lang.Long</key-type>
           <value-type>com.example.User</value-type>
           <expiry>
               <ttl unit="minutes">10</ttl>
           </expiry>
           <resources>
               <heap unit="entries">1000</heap>
           </resources>
       </cache>
   </config>
   ```
3. **Enable caching** with `@EnableCaching`.
4. **Point Spring at the config file:**
   ```properties
   spring.cache.jcache.config=classpath:ehcache.xml
   ```
5. **Use cache annotations** as usual — first call hits the DB and populates EhCache, subsequent calls return data from memory.

#### Cache Abstraction

A common caching layer that lets developers use standard annotations (`@Cacheable`, `@CachePut`, `@CacheEvict`) without writing cache-specific code for Redis, EhCache, Caffeine, or others. Internally, Spring uses a `CacheManager` interface; depending on which cache dependency is on the classpath, Spring Boot automatically creates the appropriate manager (`RedisCacheManager`, `EhCacheCacheManager`, etc.) — letting you switch underlying cache technology via configuration alone, without touching business logic.

---

## Microservices

### What Are Microservices?

Microservices are small, independent services that each focus on a single business capability. Each service can be developed, deployed, and scaled separately.

- Loosely coupled and independently deployable.
- Focused on one specific business function.
- Can be built using different programming languages and frameworks.
- Each microservice can be developed, deployed, and scaled independently of the others.

**Example:** An e-commerce platform uses separate microservices for product catalog, user authentication, cart, payments, and order management, communicating through APIs.

### Real-World Applications

- **Amazon** — initially a monolithic app; broke its platform into smaller components early on, allowing individual feature updates and enhanced functionality.
- **Banking & FinTech** — independent services for accounts, transactions, fraud detection, and customer support, ensuring high security, reliability, and regulatory compliance.
- **Netflix** — after facing service outages while transitioning to streaming in 2007, adopted microservices, improving reliability and performance.
- **Social media platforms** — separate services for feed, chat, notifications, and user profiles, enabling scalability and real-time interactions for millions of users.
- **Healthcare systems** — patient records, appointment scheduling, billing, and reporting as separate services, improving data management and reliability.
- **Uber** — switching from a monolithic structure to microservices smoothed operations, increasing page views and search efficiency.

### How Microservices Work

- Each microservice handles one particular business feature (e.g., user authentication, product management), allowing specialized development.
- Services interact via APIs, enabling standardized information exchange and integration.
- Each service runs independently and communicates with others through lightweight protocols such as HTTP or messaging systems.
- User requests are routed to the appropriate microservice, which processes the request and may interact with other services or databases before returning a response.

### API Gateway

A centralized entry point that manages client requests and directs them to the appropriate backend services, simplifying communication between clients and multiple microservices while enforcing security and performance policies.

- Routes client requests, handles authentication and rate limiting, and acts as a reverse proxy that hides internal service complexity.
- Provides a single, unified API interface for clients.

**Example:** In an e-commerce system, one API Gateway can route requests for product details, user orders, and payment processing to separate microservices, while checking authentication and limiting request rates.

**Typical responsibilities:**
- **Routing** — determining which backend service a request should go to.
- **Authentication** — verifying a user's identity (e.g., via a User ID and password, as in a login/signup flow).
- **SSL (Secure Socket Layer)** — establishes an encrypted link between server and client.
- **Protocol translation** — translating incoming requests from one channel/protocol to another.
- **Request aggregation** — the gateway sends requests to multiple backend services and combines their responses into a single result for the client. This reduces the number of client requests but may increase latency, since it waits for all services to respond.

### Synchronous vs Asynchronous Communication

**Synchronous communication** — the sender sends a request and waits for an immediate response before proceeding; the requesting service pauses execution until the receiving service processes the request and returns a result.
- Used in real-time scenarios needing immediate feedback (APIs, database queries, tightly coupled systems).
- Ensures consistency and simplicity but can introduce latency if response time is delayed.
- Analogy: a phone call or face-to-face conversation.

**Asynchronous communication** — the sender sends a request and continues execution without waiting for an immediate response; the receiver processes it independently, and the response (if any) arrives later via callbacks, message queues, or event-driven mechanisms.
- Common in distributed systems, microservices, and applications requiring high scalability and flexibility.
- Reduces latency and improves performance by decoupling sender and receiver, though it adds complexity in managing responses.
- Analogy: email or text messages.

### Service Discovery and Service Registry

As the number of microservices grows, managing communication between them becomes complex. Service Discovery and Service Registry solve this by dynamically locating services.

- Helps services find and communicate with each other.
- Eliminates the need to manually manage IP addresses and ports.
- Essential for scalable and distributed systems.

**Example:** In an e-commerce app, services like Order, Payment, and Product need to communicate. Service discovery lets them do so dynamically, without knowing each other's exact locations.

**Service Discovery** — a mechanism that allows services to automatically find each other without manual configuration, removing the need to remember service locations.
- Automatically locates services.
- Avoids hardcoded URLs.
- Improves flexibility and scalability.

**Service Registry** — a centralized database storing all service details (IP address, port, instances); all services register themselves here.
- Stores service location information.
- Keeps track of all active instances.
- Must be highly available and kept up to date.

**Advantages:** no need to manage IPs manually, supports dynamic scaling, improves flexibility.
**Disadvantages:** requires additional setup, creates a dependency on registry availability, slightly increases system complexity.

### Load Balancing

As traffic to a website grows, load on a single server increases; concurrent traffic can overload it, slowing the site down. Scaling by adding more servers and distributing requests across them solves this.

**What is a Load Balancer?**
A networking device or software application that distributes incoming traffic among servers to provide high availability, efficient utilization, and high performance. Widely used in cloud computing, data centers, and large-scale web applications. Its primary goal is to avoid overburdening any single server, which could lead to crashes or high latency.

**Without a load balancer:** a client connects directly to a single server — no traffic distribution, so a single overloaded or failed server disrupts the entire service.

**How it works:** placing a load balancer in front of the web servers lets the service handle more requests by adding more servers to the pool.
- Requests are spread across multiple servers.
- If one server goes offline, the service continues.
- Latency drops because no single server is bottlenecked on RAM/Disk/CPU.

**Load balancing algorithms:**

1. **Round Robin** — a simple static approach distributing requests sequentially/rotationally across servers. Easy to implement, but doesn't account for current server load, risking overload on some servers.
2. **Weighted Round Robin** — similar to Round Robin, but each server is given a weighted score, and requests are distributed according to that score.
3. **Source IP Hash** — distributes requests based on a hash of the source IP address, ensuring requests from the same client are consistently routed to the same server.
4. **Least Connection Method** — a dynamic approach assigning new requests to the server with the fewest active connections, balancing current load across servers.
5. **Least Response Time Method** — a dynamic approach directing new requests to the server with the quickest response time, minimizing overall response times.

### Fault Tolerance

Fault tolerance is a system's capacity to keep working despite hardware or software issues. It relies on redundancy, error detection, and error recovery techniques to avoid costly failures, letting the system continue operating (or degrade gracefully) instead of failing completely.

- Prevents complete system failure by letting the system continue working even if some components stop functioning.
- Improves reliability by detecting errors and switching to backup components when needed.

**Systems that require fault tolerance:**
- **RAID (Redundant Array of Independent Disks)** — distributes data across multiple disks with redundancy, so the system keeps functioning even if one disk fails.
- **Load Balancing** — distributes traffic across servers so others can absorb the load if one fails.
- **Clustering** — clusters of servers let one take over seamlessly if another fails.
- **Virtualization** — VMs can be migrated to another server in case of hardware failure.
- **Microservices Architecture** — breaking an app into small independent services isolates faults, preventing one failing service from bringing down the whole system.
- **Distributed Cloud Architecture** — distributing applications across multiple cloud regions/providers reduces the impact of a failure in any one region or service.

**Challenges in implementing fault tolerance:**
- **Scalability issues** — fault-tolerant mechanisms must scale alongside the system's growth without sacrificing performance or availability.
- **Performance impacts** — redundancy and error correction can degrade performance; the challenge is minimizing this while maintaining fault tolerance.
- **Cost considerations** — robust fault tolerance requires redundant hardware, software licenses, maintenance, and monitoring, all adding cost.

### System Resilience

System resilience is the capability of a system — engineered, organizational, or software-based — to handle disruptions or failures and keep functioning, withstanding and rapidly recovering from failures, disruptions, or stress without significant downtime or loss of functionality.

- Designed to handle unexpected issues (hardware failures, software bugs, heavy traffic, cyber-attacks) and remain operational.
- Rooted in a system's capacity to anticipate, absorb, adapt to, and/or quickly recover from such events.

**Why resilience matters:**
- **Maintaining continuous operations** — resilient systems withstand and recover from hardware, software, or network failures, keeping critical services available and avoiding costly downtime.
- **Minimizing disruptions and downtime** — anticipating failures and implementing proactive measures reduces the impact and duration of disruptions.
- **Protecting against cyber threats** — robust security measures (encryption, authentication, intrusion detection) mitigate breach risk and protect data integrity/confidentiality.
- **Ensuring data integrity and recovery** — robust backup and recovery mechanisms protect against data loss/corruption and preserve business continuity.
- **Adapting to change and scaling** — resilient systems are flexible and scalable, adjusting dynamically to traffic spikes or new technologies without sacrificing performance or reliability.

---

### Additional Microservices Concepts (Common Interview Follow-ups)

#### Monolithic vs Microservices Architecture

| Feature | Monolithic | Microservices |
|---|---|---|
| Codebase | Single, unified codebase | Multiple independent codebases |
| Deployment | Entire app deployed as one unit | Each service deployed independently |
| Scaling | Scales the whole application | Scales individual services as needed |
| Technology stack | Usually one stack for the whole app | Each service can use a different stack |
| Fault isolation | A bug can bring down the whole app | Failures are isolated to individual services |
| Development | Simpler initially, harder to maintain at scale | More complex initially, easier to maintain/extend at scale |
| Communication | In-process method calls | Network calls (REST, gRPC, messaging) |
| Team structure | Works well for small, centralized teams | Works well for multiple independent teams owning separate services |

#### Circuit Breaker Pattern

Prevents a failing service from being repeatedly called, protecting the overall system from cascading failures — similar to an electrical circuit breaker that "trips" to stop current flow when there's a fault.

**States:**
- **Closed** — requests flow normally; failures are counted.
- **Open** — once failures exceed a threshold, the circuit "opens" and requests fail fast (or return a fallback) without calling the failing service.
- **Half-Open** — after a timeout, a limited number of test requests are allowed through; if they succeed, the circuit closes again, otherwise it re-opens.

Commonly implemented in Spring Boot using **Resilience4j** (the modern choice) or the older, now-in-maintenance-mode **Netflix Hystrix**.

#### Saga Pattern (Distributed Transactions)

Since microservices each typically own their own database, traditional ACID transactions spanning multiple services aren't possible. The Saga pattern manages data consistency across services using a sequence of local transactions, each publishing an event/message that triggers the next step.

- **Choreography-based Saga** — each service listens for events and decides what to do next; no central coordinator, but can become hard to track as complexity grows.
- **Orchestration-based Saga** — a central orchestrator tells each service what local transaction to execute, and coordinates compensating transactions if a step fails.
- **Compensating transactions** — if one step in the saga fails, previously completed steps are undone via compensating actions (e.g., refunding a payment if an order can't be fulfilled).

#### Distributed Tracing and Monitoring

In a system made of many services, a single user request can pass through several services, making failures and performance issues hard to diagnose without a way to trace the whole path.

- **Distributed tracing** — tools like Spring Cloud Sleuth, Zipkin, and Jaeger tag each request with a unique trace ID, allowing engineers to follow it across every service it touches.
- **Centralized logging** — tools like the ELK stack (Elasticsearch, Logstash, Kibana) aggregate logs from all services into one searchable place.
- **Metrics/monitoring** — tools like Prometheus and Grafana track service health, latency, and error rates across the system.

#### Centralized Configuration

Managing configuration (database URLs, feature flags, credentials) separately for dozens of services is error-prone. A **Config Server** (e.g., Spring Cloud Config) centralizes configuration, letting services fetch their settings at startup (or refresh them at runtime) from one place, often backed by a Git repository for version control.

#### Containerization and Orchestration

- **Docker** — packages a microservice and its dependencies into a lightweight, portable container that runs consistently across environments.
- **Kubernetes** — an orchestration platform that automates deployment, scaling, load balancing, and recovery of containerized microservices across a cluster of machines.

#### CAP Theorem

In a distributed system, it's impossible to simultaneously guarantee all three of the following — a system can provide at most two:

- **Consistency** — every read receives the most recent write (or an error).
- **Availability** — every request receives a (non-error) response, without guaranteeing it's the most recent data.
- **Partition Tolerance** — the system continues to operate despite network partitions (communication failures between nodes).

Since network partitions are unavoidable in real distributed systems, the practical trade-off is usually between consistency and availability (CP vs. AP systems).

#### API Versioning

As microservices evolve, APIs need to change without breaking existing clients. Common versioning strategies:

- **URI versioning** — e.g., `/api/v1/orders`, `/api/v2/orders`.
- **Header versioning** — version specified in a custom request header.
- **Query parameter versioning** — e.g., `/api/orders?version=1`.
- **Content negotiation (Accept header)** — version embedded in the media type, e.g., `Accept: application/vnd.company.v1+json`.

---

## JPA & Hibernate

### What is JPA?

JPA (Java Persistence API) is a **specification** for managing relational data in Java applications, providing an object-relational mapping (ORM) approach to map Java objects to database tables. JPA itself is just a specification, not an implementation — **Hibernate**, EclipseLink, and OpenJPA are popular implementations.

| | JPA | Hibernate |
|---|---|---|
| What it is | Specification/standard for ORM — defines interfaces and annotations | An implementation of the JPA specification |
| Portability | Portable across different ORM frameworks | Hibernate-specific features go beyond the JPA spec |

### Entities

An **Entity** is a lightweight persistence domain object representing a table in a relational database — each instance corresponds to a row. Entities are plain Java objects (POJOs) annotated with `@Entity`; the class must have a no-arg constructor and cannot be `final`.

```java
@Entity
@Table(name = "employees")
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "emp_name")
    private String name;

    private String email;
    private Double salary;
}
```

### Core Annotations

- **`@Entity`** — marks a class as a JPA entity.
- **`@Table`** — specifies the table name (and optionally schema); defaults to the class name if omitted.
- **`@Id`** — marks the primary key field; every entity must have at least one.
- **`@GeneratedValue`** — specifies the auto-generation strategy for the primary key.
- **`@Column`** — maps a field to a column and its properties (name, `nullable`, `length`, `unique`).
- **`@Transient`** — excludes a field from persistence entirely.
- **`@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany`** — relationship mappings.

```java
@Column(name = "emp_name", nullable = false, length = 100, unique = true)
private String name;
```

### EntityManager

The primary interface for interacting with the **persistence context** — provides methods to create, read, update, and delete entities, and manages the entity lifecycle.

```java
@PersistenceContext
private EntityManager entityManager;

public void saveEmployee(Employee emp) {
    entityManager.persist(emp);
}
```

`persistence.xml` (in `META-INF`) configures persistence units, database connection details, and JPA properties — used in plain JPA setups (Spring Boot typically configures this via `application.properties` instead).

### Entity Lifecycle States

1. **Transient** — a new object, not yet associated with a persistence context.
2. **Managed/Persistent** — associated with a persistence context; changes are tracked.
3. **Detached** — was managed, but the session/persistence context has closed.
4. **Removed** — marked for deletion.

```java
Employee emp = new Employee();      // Transient
entityManager.persist(emp);         // Managed
entityManager.detach(emp);          // Detached
entityManager.remove(emp);          // Removed
```

### Primary Key Generation Strategies

`@GeneratedValue` specifies how the primary key is generated:

- **`AUTO`** — the JPA provider chooses the strategy.
- **`IDENTITY`** — uses a database identity/auto-increment column.
- **`SEQUENCE`** — uses a database sequence.
- **`TABLE`** — uses a separate table to generate unique values.

```java
// IDENTITY
@GeneratedValue(strategy = GenerationType.IDENTITY)

// SEQUENCE
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "emp_seq")
@SequenceGenerator(name = "emp_seq", sequenceName = "employee_seq", allocationSize = 1)

// TABLE
@GeneratedValue(strategy = GenerationType.TABLE, generator = "emp_gen")
@TableGenerator(name = "emp_gen", table = "id_gen", pkColumnName = "gen_name")
```

### Core EntityManager Operations

- **`persist()`** — for new (transient) entities; makes the entity managed; throws an exception if the entity already exists.
- **`merge()`** — for detached entities; creates a new managed copy and returns it (the original reference stays detached).
- **`find()`** — retrieves an entity by primary key; returns `null` if not found.
- **`remove()`** — marks a managed entity for deletion.
- **`getReference()`** — returns a lazy proxy instead of hitting the database immediately; throws `EntityNotFoundException` when accessed if the row doesn't exist (unlike `find()`, which hits the DB right away and returns `null`).

```java
// persist
Employee emp = new Employee();
emp.setName("John");
entityManager.persist(emp);

// merge
Employee detachedEmp = new Employee();
detachedEmp.setId(1L);
detachedEmp.setName("Updated");
Employee managedEmp = entityManager.merge(detachedEmp);

// find vs getReference
Employee emp1 = entityManager.find(Employee.class, 1L);        // DB hit immediately
Employee emp2 = entityManager.getReference(Employee.class, 1L); // Proxy; DB hit on first field access
```

### JPQL (Java Persistence Query Language)

An object-oriented query language for entities — similar to SQL, but works with entity objects/fields instead of tables/columns, and is database-independent.

```java
String jpql = "SELECT e FROM Employee e WHERE e.salary > :minSalary";
List<Employee> employees = entityManager.createQuery(jpql, Employee.class)
    .setParameter("minSalary", 50000.0)
    .getResultList();
```

| | JPQL | SQL |
|---|---|---|
| Operates on | Entity objects and their properties | Database tables and columns |
| Portability | Database independent | Database specific |

### Spring Data JPA — CrudRepository

Provides CRUD operations without writing implementation code — Spring Data JPA auto-implements the methods.

```java
public interface EmployeeRepository extends CrudRepository<Employee, Long> {
    // No implementation needed
}

employeeRepository.save(employee);
employeeRepository.findById(1L);
employeeRepository.findAll();
employeeRepository.deleteById(1L);
```

---

### Entity Relationships

| Relationship | Description | Example |
|---|---|---|
| One-to-One | Single entity related to a single instance of another | `User` ↔ `UserProfile` |
| One-to-Many | Single entity related to multiple instances of another | `Department` → `Employees` |
| Many-to-One | Multiple entities related to a single instance | `Employees` → `Department` |
| Many-to-Many | Multiple entities related to multiple instances | `Students` ↔ `Courses` |

**`@OneToOne`:**
```java
@Entity
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String username;

    @OneToOne(cascade = CascadeType.ALL)
    @JoinColumn(name = "profile_id", referencedColumnName = "id")
    private UserProfile profile;
}

@Entity
public class UserProfile {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String bio;

    @OneToOne(mappedBy = "profile")
    private User user;
}
```

**`@OneToMany` / `@ManyToOne`:**
```java
@Entity
public class Department {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Employee> employees = new ArrayList<>();
}

@Entity
public class Employee {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "dept_id")
    private Department department;
}
```

**`@ManyToMany`:**
```java
@Entity
public class Student {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();
}

@Entity
public class Course {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;

    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();
}
```

### `mappedBy` — Owning vs Inverse Side

`mappedBy` defines the owning side of a bidirectional relationship. The side **with** `mappedBy` is the **inverse** (non-owning) side; the owning side holds the foreign key (`@JoinColumn`).

```java
// Department is the inverse side
@OneToMany(mappedBy = "department")
private List<Employee> employees;

// Employee is the owning side (holds the FK)
@ManyToOne
@JoinColumn(name = "dept_id")
private Department department;
```

**Unidirectional vs bidirectional:** unidirectional means only one entity knows about the relationship; bidirectional means both sides know about it (via `mappedBy` on the inverse side).

### Cascade Types

`CascadeType` defines which operations cascade from parent to child entities:

- **`PERSIST`** — cascades the persist operation.
- **`MERGE`** — cascades the merge operation.
- **`REMOVE`** — cascades the delete operation.
- **`REFRESH`** — cascades the refresh operation.
- **`DETACH`** — cascades the detach operation.
- **`ALL`** — all of the above.

```java
@OneToMany(cascade = CascadeType.ALL)
private List<Employee> employees;

department.getEmployees().add(newEmployee);
entityManager.persist(department); // newEmployee also persisted
```

### Fetch Types

- **`LAZY`** — data is loaded on-demand when accessed; generally better performance.
- **`EAGER`** — data is loaded immediately with the parent entity; can cause performance issues.

```java
@ManyToOne(fetch = FetchType.LAZY)
private Department department;

@ManyToOne(fetch = FetchType.EAGER)
private Department department;
```

### The N+1 Problem

Occurs when fetching a collection results in one query for the parent plus **N** additional queries — one per child — typically due to lazy loading accessed in a loop.

```java
// 1 query for departments + N queries for employees
List<Department> depts = entityManager
    .createQuery("SELECT d FROM Department d", Department.class)
    .getResultList();

for (Department dept : depts) {
    dept.getEmployees().size(); // triggers a separate query each time
}
```

**Solutions:**

1. **`JOIN FETCH`:**
   ```java
   String jpql = "SELECT DISTINCT d FROM Department d JOIN FETCH d.employees";
   ```
2. **`EntityGraph`:**
   ```java
   EntityGraph<Department> graph = entityManager.createEntityGraph(Department.class);
   graph.addAttributeNodes("employees");
   entityManager.createQuery("SELECT d FROM Department d", Department.class)
       .setHint("javax.persistence.fetchgraph", graph)
       .getResultList();
   ```
3. **Batch fetching:**
   ```java
   @BatchSize(size = 10)
   @OneToMany(mappedBy = "department")
   private List<Employee> employees;
   ```

### Join Mappings

**`@JoinColumn`** — specifies the foreign key column on the owning side:
```java
@ManyToOne
@JoinColumn(name = "department_id", nullable = false,
            foreignKey = @ForeignKey(name = "fk_emp_dept"))
private Department department;
```

**`@JoinTable`** — creates a join table for Many-to-Many relationships:
```java
@ManyToMany
@JoinTable(
    name = "student_course",
    joinColumns = @JoinColumn(name = "student_id"),
    inverseJoinColumns = @JoinColumn(name = "course_id"),
    uniqueConstraints = @UniqueConstraint(columnNames = {"student_id", "course_id"})
)
private Set<Course> courses;
```

### `orphanRemoval`

Automatically deletes child entities when they're removed from the parent's collection.

```java
@OneToMany(mappedBy = "department", orphanRemoval = true)
private List<Employee> employees;

department.getEmployees().remove(employee); // employee is deleted from the DB
```

### Embeddable Objects and Composite Keys

**`@Embeddable` / `@Embedded`** — embed an object as part of an entity instead of a separate table:
```java
@Embeddable
public class Address {
    private String street;
    private String city;
    private String zipCode;
}

@Entity
public class Employee {
    @Id
    private Long id;
    private String name;

    @Embedded
    private Address address; // fields embedded in the employee table
}
```

**Composite primary keys** — a key made of multiple fields; implemented with `@IdClass` or `@EmbeddedId`.

```java
// @IdClass
public class OrderId implements Serializable {
    private Long customerId;
    private Long productId;
    // equals/hashCode
}

@Entity
@IdClass(OrderId.class)
public class Order {
    @Id private Long customerId;
    @Id private Long productId;
    private Integer quantity;
}

// @EmbeddedId
@Embeddable
public class OrderId implements Serializable {
    private Long customerId;
    private Long productId;
    // equals/hashCode
}

@Entity
public class Order {
    @EmbeddedId
    private OrderId id;
    private Integer quantity;
}
```

---

### Named Queries and Native SQL

**`@NamedQuery`** — reusable JPQL defined at the entity level:
```java
@Entity
@NamedQuery(
    name = "Employee.findByDepartment",
    query = "SELECT e FROM Employee e WHERE e.department.name = :deptName"
)
public class Employee { }

entityManager.createNamedQuery("Employee.findByDepartment", Employee.class)
    .setParameter("deptName", "IT")
    .getResultList();
```

**`@NamedNativeQuery`** — reusable native SQL defined at the entity level.

**`createQuery` vs `createNativeQuery`:**

| | `createQuery` (JPQL) | `createNativeQuery` (SQL) |
|---|---|---|
| Portability | Database independent | Database specific |
| Flexibility | Returns entities | More flexible, works with raw SQL |

### Criteria API

A type-safe way to build dynamic queries programmatically, without string concatenation.

```java
CriteriaBuilder cb = entityManager.getCriteriaBuilder();
CriteriaQuery<Employee> cq = cb.createQuery(Employee.class);
Root<Employee> root = cq.from(Employee.class);

cq.select(root).where(cb.greaterThan(root.get("salary"), 50000.0));
List<Employee> employees = entityManager.createQuery(cq).getResultList();
```

| | JPQL | Criteria API |
|---|---|---|
| Style | String-based, simpler syntax | Type-safe, programmatic |
| Risk | Prone to typos (caught at runtime) | Compile-time checked, but verbose |
| Best for | Static/simple queries | Dynamic queries built conditionally |

### Field-Level Annotations

- **`@Temporal`** — specifies precision for `java.util.Date`/`Calendar` fields (`DATE`, `TIME`, `TIMESTAMP`); not needed for Java 8+ `LocalDate`/`LocalDateTime`.
- **`@Enumerated`** — specifies how enums are persisted: `EnumType.STRING` (as text) or `EnumType.ORDINAL` (as integer index) — `STRING` is generally safer since it's resilient to enum reordering.
- **`@Lob`** — stores large objects: `String` → CLOB (Character Large Object), `byte[]` → BLOB (Binary Large Object).

```java
@Enumerated(EnumType.STRING)
private Status status;

@Lob
private String description;

@Lob
private byte[] profileImage;
```

### Optimistic vs Pessimistic Locking

**Optimistic locking** — assumes conflicts are rare; uses a `@Version` field to detect concurrent updates. If the version doesn't match on update, an `OptimisticLockException` is thrown.

```java
@Entity
public class Employee {
    @Id
    private Long id;

    @Version
    private Long version; // auto-incremented on each update

    private String name;
}
```

**Pessimistic locking** — assumes conflicts are likely; locks the database row on read.

```java
Employee emp = entityManager.find(Employee.class, 1L, LockModeType.PESSIMISTIC_WRITE);
// row is locked until the transaction completes
```

### Entity Lifecycle Callbacks

```java
@Entity
public class Employee {
    @Id
    private Long id;

    @PrePersist  public void prePersist() { }   // before insert
    @PostPersist public void postPersist() { }  // after insert
    @PreUpdate   public void preUpdate() { }    // before update
    @PostUpdate  public void postUpdate() { }   // after update
    @PreRemove   public void preRemove() { }    // before delete
    @PostRemove  public void postRemove() { }   // after delete
    @PostLoad    public void postLoad() { }     // after loading from DB
}
```

**`@EntityListeners`** — specifies external callback listener classes instead of defining callback methods on the entity itself.
```java
public class AuditListener {
    @PrePersist
    public void prePersist(Object entity) { }
}

@Entity
@EntityListeners(AuditListener.class)
public class Employee { }
```

### EntityGraph

Defines which attributes to fetch eagerly for a specific query, helping avoid the N+1 problem without changing the entity's default fetch type.

```java
@Entity
@NamedEntityGraph(
    name = "Employee.detail",
    attributeNodes = {
        @NamedAttributeNode("department"),
        @NamedAttributeNode("projects")
    }
)
public class Employee {
    @Id
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    private Department department;

    @ManyToMany(fetch = FetchType.LAZY)
    private Set<Project> projects;
}

// Spring Data JPA repository usage
@EntityGraph(attributePaths = {"department", "projects"})
List<Employee> findAll();
```

---

### Caching

**First Level Cache** — the persistence context cache, enabled by default at the `EntityManager` level. Stores entities during a transaction, so the same entity instance is returned for the same ID within the same session.

```java
Employee emp1 = entityManager.find(Employee.class, 1L); // DB hit
Employee emp2 = entityManager.find(Employee.class, 1L); // Cache hit
System.out.println(emp1 == emp2); // true — same instance
```

**Second Level Cache** — shared across `EntityManager` instances at the `EntityManagerFactory` level; optional and requires configuration.

```properties
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.jcache.JCacheRegionFactory
```

```java
@Entity
@Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Employee { }
```

### Persistence Context Management

- **`flush()`** — synchronizes the persistence context with the database, executing pending SQL without committing the transaction.
- **`clear()`** — detaches *all* entities from the persistence context (clears the first-level cache).
- **`detach()`** — detaches a *specific* entity from the persistence context.

```java
entityManager.persist(employee);
entityManager.flush(); // INSERT executed, not yet committed

Employee emp = entityManager.find(Employee.class, 1L); // Managed
entityManager.clear();                                  // Detached
emp.setName("New Name");
entityManager.flush();                                   // No update — emp is detached
```

**Dirty checking** — Hibernate automatically detects changes to managed entities and issues `UPDATE` statements without an explicit save/merge call.

```java
@Transactional
public void updateEmployee(Long id) {
    Employee emp = entityManager.find(Employee.class, id); // Managed
    emp.setSalary(75000.0); // Change detected automatically → UPDATE on commit
}
```

### Soft Delete, `@Where`, and `@Filter` (Hibernate-specific)

```java
@Entity
@SQLDelete(sql = "UPDATE employee SET deleted = true WHERE id = ?")
@Where(clause = "deleted = false")
public class Employee {
    @Id
    private Long id;
    private boolean deleted = false;
}
```

- **`@SQLDelete`** — overrides the default `DELETE` statement (e.g., to perform a soft delete instead).
- **`@Where`** — adds a permanent `WHERE` clause to all queries for the entity.
- **`@Filter`** — provides dynamic, parameterized filtering that can be enabled/disabled at runtime:

```java
@Entity
@FilterDef(name = "departmentFilter", parameters = @ParamDef(name = "deptName", type = "string"))
@Filter(name = "departmentFilter", condition = "dept_name = :deptName")
public class Employee { }

Session session = entityManager.unwrap(Session.class);
session.enableFilter("departmentFilter").setParameter("deptName", "IT");
```

### Auditing

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class Auditable {
    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdDate;

    @LastModifiedDate
    private LocalDateTime lastModifiedDate;

    @CreatedBy
    @Column(updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String lastModifiedBy;
}

@Entity
public class Employee extends Auditable { }

@Configuration
@EnableJpaAuditing
public class JpaConfig {
    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.of("CurrentUser"); // typically pulled from security context
    }
}
```

### Batch Processing

Groups multiple SQL statements together for execution, improving performance over one-statement-at-a-time processing.

```java
@Transactional
public void batchInsert(List<Employee> employees) {
    int batchSize = 50;
    for (int i = 0; i < employees.size(); i++) {
        entityManager.persist(employees.get(i));
        if (i > 0 && i % batchSize == 0) {
            entityManager.flush();
            entityManager.clear();
        }
    }
}
```

```properties
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
spring.jpa.properties.hibernate.batch_versioned_data=true
```

### `persist`/`save`/`merge` — Key Differences

| Method | Behavior |
|---|---|
| `persist()` (JPA) | Only for new entities; throws if the entity already exists |
| `merge()` (JPA) | For detached entities; returns a new managed copy |
| `save()` (Spring Data JPA) | Can insert or update (delegates to `merge` internally if an ID is present) |
| `saveAndFlush()` | Persists *and* immediately flushes to the database |
| `update()` (Hibernate-specific) | Reattaches a detached entity; no return value |

### StatelessSession (Hibernate)

A session with **no** persistence context — no first-level caching, no dirty checking, no cascading. Used for high-performance bulk operations (bulk import/export, batch processing).

```java
StatelessSession session = sessionFactory.openStatelessSession();
Transaction tx = session.beginTransaction();

for (Employee emp : employees) {
    session.insert(emp); // no caching
}

tx.commit();
session.close();
```

### HQL vs JPQL

**HQL (Hibernate Query Language)** is Hibernate's own object-oriented query language — a superset of JPQL with additional Hibernate-specific features. JPQL is the JPA-standard, portable subset.

### Pagination

```java
// Plain JPA
List<Employee> employees = entityManager
    .createQuery("SELECT e FROM Employee e ORDER BY e.id", Employee.class)
    .setFirstResult(pageNumber * pageSize)
    .setMaxResults(pageSize)
    .getResultList();
```

**Spring Data JPA `Pageable`:**
```java
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    Page<Employee> findByDepartmentName(String deptName, Pageable pageable);
}

Pageable pageable = PageRequest.of(0, 10, Sort.by("name").ascending());
Page<Employee> page = employeeRepository.findByDepartmentName("IT", pageable);

page.getTotalPages();
page.getTotalElements();
page.getContent();
```

### Specifications (Dynamic Queries)

`Specification` allows building dynamic queries programmatically using the Criteria API under the hood.

```java
public interface EmployeeRepository extends JpaSpecificationExecutor<Employee> { }

public class EmployeeSpecification {
    public static Specification<Employee> hasSalaryGreaterThan(Double salary) {
        return (root, query, cb) -> cb.greaterThan(root.get("salary"), salary);
    }
    public static Specification<Employee> belongsToDepartment(String deptName) {
        return (root, query, cb) -> cb.equal(root.get("department").get("name"), deptName);
    }
}

Specification<Employee> spec = EmployeeSpecification.hasSalaryGreaterThan(50000.0)
    .and(EmployeeSpecification.belongsToDepartment("IT"));
List<Employee> employees = employeeRepository.findAll(spec);
```

### Projections

Retrieve only specific fields instead of an entire entity, improving performance.

- **DTO (class-based) projection:**
  ```java
  public class EmployeeDTO {
      private String name;
      private Double salary;
      public EmployeeDTO(String name, Double salary) { this.name = name; this.salary = salary; }
  }

  @Query("SELECT new com.example.EmployeeDTO(e.name, e.salary) FROM Employee e")
  List<EmployeeDTO> findAllEmployeeDTOs();
  ```
- **Interface-based projection:**
  ```java
  public interface EmployeeProjection {
      String getName();
      Double getSalary();
  }

  List<EmployeeProjection> findByDepartmentName(String deptName);
  ```
- **Dynamic projection** — the repository method's return type is generic, letting the caller choose the projection type at call time.

### `@Query` and `@Modifying`

```java
@Query("SELECT e FROM Employee e WHERE e.salary > :minSalary")
List<Employee> findHighEarners(@Param("minSalary") Double minSalary);

@Query(value = "SELECT * FROM employees WHERE salary > ?1", nativeQuery = true)
List<Employee> findHighEarnersNative(Double minSalary);

@Modifying
@Transactional
@Query("UPDATE Employee e SET e.salary = e.salary * 1.1 WHERE e.department.name = :deptName")
int giveDepartmentRaise(@Param("deptName") String deptName);
```

> `@Modifying` marks a query as `UPDATE`/`DELETE`; it must always be paired with `@Transactional`.

### LazyInitializationException

Thrown when accessing a lazy-loaded association **outside** the persistence context (e.g., after the transaction/session has closed).

```java
@Transactional
public Employee getEmployee(Long id) {
    return employeeRepository.findById(id).orElse(null);
} // transaction ends, session closes

employee.getDepartment().getName(); // LazyInitializationException
```

**Solutions:**
1. Use `EAGER` fetching.
2. Use `JOIN FETCH` in the query.
3. Keep the transaction open around the code that accesses the association.
4. Initialize lazy associations within the transaction.
5. Use `EntityGraph`.
6. Use Open Session In View (generally *not* recommended — see below).

### Open Session In View (OSIV)

Keeps the Hibernate session open for the entire HTTP request, allowing lazy loading in the view/controller layer.

```properties
# Disable OSIV (recommended)
spring.jpa.open-in-view=false
```

**Problems with OSIV:**
- Performance issues from long-held database connections.
- Can cause N+1 problems to surface in the view layer instead of being caught early.
- Ties database transactions to web request lifecycles.
- Makes lazy-loading issues harder to detect during development.
- Not suitable for high-traffic applications.

### Connection Pooling

Maintains a pool of reusable database connections, avoiding the overhead of creating a new connection per request. **HikariCP** is the default connection pool in Spring Boot.

```properties
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=20000
spring.datasource.hikari.idle-timeout=300000
spring.datasource.hikari.max-lifetime=1200000
```

### Inheritance Strategies

| Strategy | Description |
|---|---|
| **Single Table** | All classes in the hierarchy stored in one table, distinguished by a discriminator column |
| **Joined** | Each class has its own table, joined via foreign keys |
| **Table Per Class** | Each concrete class has its own complete table with all fields |

```java
// Single Table
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "employee_type")
public abstract class Employee {
    @Id private Long id;
    private String name;
}

@Entity
@DiscriminatorValue("FT")
public class FullTimeEmployee extends Employee {
    private Double salary;
}

// Joined
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Employee { /* ... */ }

@Entity
@Table(name = "full_time_employees")
public class FullTimeEmployee extends Employee { /* ... */ }

// Table Per Class
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Employee { /* ... */ }
```

### Attribute Converters

Convert entity attributes to database columns and back — useful for custom mapping logic not natively supported.

```java
@Converter(autoApply = true)
public class BooleanToYNConverter implements AttributeConverter<Boolean, String> {
    @Override
    public String convertToDatabaseColumn(Boolean value) {
        return value != null && value ? "Y" : "N";
    }
    @Override
    public Boolean convertToEntityAttribute(String dbData) {
        return "Y".equals(dbData);
    }
}

@Entity
public class Employee {
    @Convert(converter = BooleanToYNConverter.class)
    private Boolean active;
}
```

### Multi-Tenancy

Allows a single application to serve multiple tenants with data isolation. Approaches:

1. **Separate Database** — each tenant has its own database.
2. **Separate Schema** — shared database, separate schemas per tenant.
3. **Shared Schema** — a single schema with a `tenant_id` discriminator column on shared tables.

```java
@Entity
@FilterDef(name = "tenantFilter", parameters = @ParamDef(name = "tenantId", type = "long"))
@Filter(name = "tenantFilter", condition = "tenant_id = :tenantId")
public class Employee {
    @Id
    private Long id;

    @Column(name = "tenant_id")
    private Long tenantId;
}
```

### Custom ID Generators

```java
public class CustomIdGenerator implements IdentifierGenerator {
    @Override
    public Serializable generate(SharedSessionContractImplementor session, Object object) {
        String prefix = "EMP";
        Long maxId = (Long) session.createQuery("SELECT MAX(id) FROM Employee").uniqueResult();
        return prefix + (maxId == null ? 1 : maxId + 1);
    }
}

@Entity
public class Employee {
    @Id
    @GeneratedValue(generator = "custom-generator")
    @GenericGenerator(name = "custom-generator", strategy = "com.example.CustomIdGenerator")
    private String id;
}
```

### `EntityManagerFactory` vs `EntityManager`

`EntityManagerFactory` creates `EntityManager` instances — it's thread-safe and expensive to create, so it's typically created once per application. `EntityManager` (Hibernate's equivalent is `Session`) is **not** thread-safe — one instance is used per thread/transaction.