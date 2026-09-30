# Spring Boot: Layered Order Service (Working Example)

## Layered architecture & flow
```
HTTP Request
   ▼
Filters (Security/JWT, logging)
   ▼
DispatcherServlet ── HandlerMapping ──► Controller  (HTTP only: validate, map DTO, status codes)
                                             ▼
                                        Service      (business rules, @Transactional)
                                             ▼
                                       Repository    (Spring Data JPA – DB access only)
                                             ▼
                                          Database
Response ◄─ DTO ◄─ @RestControllerAdvice (errors) ◄────────────────────────
```
Rule: each layer talks only to the layer below. Controllers never touch repositories; entities never leave the service layer (use DTOs).

## Working code — Order feature
```java
// ---- Entity
@Entity @Table(name = "orders")
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    private Long customerId;
    private BigDecimal total;               // NEVER double for money
    @Enumerated(EnumType.STRING) private OrderStatus status;
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
    @Version private Long version;          // optimistic locking
    private Instant createdAt = Instant.now();
}
public enum OrderStatus { CREATED, PAID, SHIPPED, CANCELLED }

// ---- DTOs (records)
public record OrderRequest(@NotNull Long customerId,
                           @NotEmpty List<@Valid ItemDto> items) {}
public record ItemDto(@NotNull Long productId, @Min(1) int qty) {}
public record OrderResponse(Long id, OrderStatus status, BigDecimal total) {}

// ---- Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByCustomerIdOrderByCreatedAtDesc(Long customerId);
}

// ---- Service
@Service @RequiredArgsConstructor @Slf4j
public class OrderService {
    private final OrderRepository repo;
    private final PaymentClient paymentClient;      // Feign client to Payment service
    private final OrderEventPublisher publisher;

    @Transactional
    public OrderResponse place(OrderRequest req) {
        Order order = new Order(req);                    // build entity, compute total
        order.setStatus(OrderStatus.CREATED);
        repo.save(order);

        PaymentResponse p = paymentClient.charge(new PaymentRequest(order.getId(), order.getTotal()));
        if (!p.success()) throw new PaymentFailedException(order.getId());   // RuntimeException → rollback

        order.setStatus(OrderStatus.PAID);
        publisher.orderPlaced(order);                    // messages (see `RabbitMQ_Spring_Project_Flow.md` and `Kafka.md`; use the outbox pattern for safety)
        return new OrderResponse(order.getId(), order.getStatus(), order.getTotal());
    }
}

// ---- Controller
@RestController @RequestMapping("/api/orders") @RequiredArgsConstructor
public class OrderController {
    private final OrderService service;

    @PostMapping
    public ResponseEntity<OrderResponse> place(@Valid @RequestBody OrderRequest req) {
        OrderResponse r = service.place(req);
        return ResponseEntity.created(URI.create("/api/orders/" + r.id())).body(r);
    }
}
```
`application.yml`
```yaml
server.port: 8081
spring:
  datasource: { url: jdbc:mysql://${DB_HOST:localhost}:3306/shop, username: ${DB_USER}, password: ${DB_PASS} }
  jpa: { hibernate.ddl-auto: validate, open-in-view: false }   # never 'update/create' in prod; use Flyway
management.endpoints.web.exposure.include: health,info,metrics
```

## Concepts to attach to this code
| Concept | Where it shows |
|---|---|
| IoC/DI | `@RequiredArgsConstructor` constructor injection of repo, client |
| Bean lifecycle / scopes | all `@Service` are singleton → keep stateless |
| AOP | `@Transactional`, security, logging = proxies |
| Auto-config, starters, profiles | see `SpringBoot_Internals.md` |
| Security | JWT filter + `@PreAuthorize` (your Spring_Security notes) |
| Validation / error handling | `@Valid`, `@RestControllerAdvice` |
| Testing | `@WebMvcTest`, Mockito, Testcontainers |
| Caching | `@Cacheable("product")` + Redis |
| Scheduling | `@Scheduled(cron = "0 0 2 * * *")` nightly report |
| Flyway/Liquibase | version-controlled DB migrations |

Database migration (never edit prod schema by hand):
```
V1__create_orders.sql, V2__add_status_column.sql → Flyway runs new ones in order at startup
```
