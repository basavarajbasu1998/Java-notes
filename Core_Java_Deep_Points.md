# Core Java Deep Points (Generics, Immutability, ClassLoader, Reflection...)

## Generics
Type safety at compile time; **type erasure** – generics are removed at runtime (`List<String>` and `List<Integer>` are both just `List`), so you can't do `new T()` or `instanceof List<String>`.

**PECS: Producer Extends, Consumer Super**
```java
void copy(List<? extends Number> src, List<? super Number> dest) { // read from src, write to dest
    for (Number n : src) dest.add(n);
}
```
- `List<? extends Number>` → can READ Numbers, can't add (unknown exact type).
- `List<? super Integer>` → can ADD Integers, reading gives Object.
- `List<Object>` is NOT a supertype of `List<String>` (generics are invariant; arrays are covariant → `ArrayStoreException`).

## Immutable class (5 steps)
1. `final` class  2. `private final` fields  3. no setters
4. **defensive copy** of mutable fields in constructor and getter  5. no leaking `this`.
```java
public final class Person {
    private final String name;
    private final List<String> phones;
    public Person(String name, List<String> phones) {
        this.name = name;
        this.phones = List.copyOf(phones);             // defensive copy
    }
    public List<String> getPhones() { return phones; } // already immutable
}
```
Benefits: thread-safe, safe HashMap keys, cache-friendly.

## `equals()` / `hashCode()` contract
Equal objects **must** have equal hash codes (reverse not required). If you override `equals` but not `hashCode` → HashMap can't find the key (you have this in Collections.md – revise with JPA entities: don't use mutable/generated ID in `hashCode` carelessly).

## `final` / `finally` / `finalize`; `static`; `==` vs `equals`
- `final`: variable can't reassign, method can't override, class can't extend. `finally`: always runs (except `System.exit`, JVM crash). `finalize`: deprecated, never use.
- `static` block runs once when class loads; static methods can't be overridden (only hidden).
- `String s1 = new String("a")` vs literal → `==` false, `equals` true (revise your String pool notes).
- `Integer` cache: `Integer a=127,b=127; a==b` → **true**; `128` → **false** (cache −128…127).

## ClassLoader
```
Bootstrap  (java.lang.*, rt)         loads core JDK classes
   ▲ parent
Platform/Extension                   JDK modules
   ▲ parent
Application (System)                 your classpath
   ▲ parent
Custom loader                        plugins, hot reload
```
**Parent delegation:** loader asks its parent first; only loads itself if parent can't → prevents someone replacing `java.lang.String`.
Class loading steps: **Load → Link (verify, prepare, resolve) → Initialize** (static blocks run).
`ClassNotFoundException` (checked, dynamic load failed, e.g. `Class.forName`) vs `NoClassDefFoundError` (Error; class existed at compile time but missing at runtime).

## Reflection
Inspect/modify classes at runtime (`Class.forName`, `getDeclaredFields`, `setAccessible(true)`, `invoke`). Used by Spring (DI), Hibernate, Jackson, JUnit. Cons: slower, breaks encapsulation, no compile-time safety. Can break Singleton (use enum singleton).

## Serialization
`Serializable` marker; `serialVersionUID` version check; `transient` fields skipped; static not serialized. Prefer JSON/Protobuf over Java serialization (security risk).

## Other frequent questions
- **Shallow vs deep copy**, `clone()` pitfalls → copy constructor.
- **Pass by value:** Java always passes value; for objects the *reference* is copied → you can mutate the object but can't reassign caller's variable.
- **Autoboxing NPE:** `Integer x = null; int y = x;` → NPE.
- **Interface vs abstract class (Java 8+):** interface can have `default`/`static` methods, still no state.
- **Overloading resolution with null:** `f(null)` picks most specific type.
- **String immutability reasons:** pool, hash caching, security, thread safety.
- **`hashCode` of `String`** is cached.
