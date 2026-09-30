# More Design Patterns (Builder, Decorator, Adapter, Proxy...)

| Pattern | Problem it solves | Real Java example |
|---|---|---|
| **Builder** | class with many optional params | `StringBuilder`, Lombok `@Builder`, `HttpRequest.newBuilder()` |
| **Decorator** | add behaviour without subclassing | `new BufferedReader(new FileReader(f))` |
| **Adapter** | make incompatible interfaces work together | `Arrays.asList()`, `InputStreamReader` |
| **Proxy** | control access / add cross-cutting logic | Spring AOP, `@Transactional`, Hibernate lazy loading |
| **Template Method** | fixed algorithm skeleton, subclass fills steps | `JdbcTemplate`, `RestTemplate` |
| **Chain of Responsibility** | request passes through handlers | Servlet Filter chain, Spring Security filter chain |
| **Dependency Injection** | don't `new` your collaborators | Spring IoC |

```java
// Builder
User u = User.builder().name("Ravi").age(30).city("BLR").build();
// Enum singleton – safest (thread-safe, reflection-safe, serialization-safe)
public enum Config { INSTANCE; public String get(String k){...} }
```
