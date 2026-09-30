# Modern Java (11 to 21)

| Version | Feature | Example |
|---|---|---|
| 9 | `List.of()`, `Map.of()` immutable factories | `List.of(1,2,3)` |
| 10 | `var` local type inference | `var list = new ArrayList<String>();` |
| 11 | `String.isBlank/strip/lines/repeat`, HTTP Client | |
| 14/17 | **Switch expressions** | see below |
| 15/17 | **Text blocks** `"""` | multi-line JSON/SQL |
| 16 | **Records** | immutable DTO |
| 16 | **Pattern matching `instanceof`** | `if (o instanceof String s)` |
| 17 | **Sealed classes** | restrict subclasses |
| 17 (LTS), 21 (LTS) | | |
| 21 | **Virtual threads**, record patterns, sequenced collections, pattern-matching switch | |

```java
// Record: constructor, getters (name()), equals, hashCode, toString auto-generated. Fields are final.
public record User(String name, int age) {
    public User { if (age < 0) throw new IllegalArgumentException(); }   // compact constructor
}

// Switch expression: returns value, no fall-through, no break
String type = switch (day) {
    case SAT, SUN -> "Weekend";
    case MON, TUE, WED, THU, FRI -> "Weekday";
};

// Sealed: only listed classes may extend
public sealed interface Shape permits Circle, Square {}
record Circle(double r) implements Shape {}
record Square(double s) implements Shape {}

// Pattern matching switch (21) – compiler checks all cases covered
double area = switch (shape) {
    case Circle c -> Math.PI * c.r() * c.r();
    case Square s -> s.s() * s.s();
};
```
**Virtual threads (21):** very lightweight threads managed by JVM (not OS). You can create millions; blocking IO no longer wastes an OS thread. `Executors.newVirtualThreadPerTaskExecutor()`; Spring Boot 3.2: `spring.threads.virtual.enabled=true`. Not for CPU-bound work. Avoid `synchronized` pinning for long blocking calls (use `ReentrantLock`).

**Records vs Lombok `@Data`:** records are immutable & built-in; Lombok makes mutable classes.
