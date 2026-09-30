# Coding Problems: Low-Level Design + Spring/REST Snippets

## F1. Singleton, Factory, Builder, Strategy
```java
// Strategy: payment
interface Payment { void pay(double amt); }
class Upi implements Payment { public void pay(double a){ System.out.println("UPI " + a);} }
class Card implements Payment { public void pay(double a){ System.out.println("Card " + a);} }
class Checkout { private final Payment p; Checkout(Payment p){this.p=p;} void done(double a){ p.pay(a);} }
// Adds new payment type with NO change to Checkout → Open/Closed Principle.

// Factory
class PaymentFactory {
    static Payment of(String type) {
        return switch (type) { case "UPI" -> new Upi(); case "CARD" -> new Card();
            default -> throw new IllegalArgumentException(type); };
    }
}
```

## F2. Parking lot / Library / ATM / Elevator (structure to describe)
```
Entities: ParkingLot, Floor, Slot(type), Vehicle(type), Ticket
Flow: entry → find free slot of vehicle type (Strategy: nearest / first free)
      → create Ticket(entryTime, slot) → mark slot occupied
      exit  → fee = hours × rate (Strategy) → free slot
Concurrency: two cars must not get the same slot → synchronized / atomic compare-and-set on slot.
```
Say: classes, relationships, interfaces for change points, thread safety, extensibility.

## F3. Simple Observer (event notification)
```java
interface Listener { void onEvent(String e); }
class EventBus {
    private final List<Listener> ls = new CopyOnWriteArrayList<>();   // safe to iterate & modify
    void subscribe(Listener l) { ls.add(l); }
    void publish(String e) { ls.forEach(l -> l.onEvent(e)); }
}
```

## F4. Immutable class with Builder
```java
public final class Employee {
    private final String name; private final int age; private final String city;
    private Employee(Builder b) { name = b.name; age = b.age; city = b.city; }
    public static Builder builder() { return new Builder(); }
    public static class Builder {
        private String name; private int age; private String city;
        public Builder name(String n) { name = n; return this; }
        public Builder age(int a) { age = a; return this; }
        public Builder city(String c) { city = c; return this; }
        public Employee build() { return new Employee(this); }
    }
}
```

---

# G) SPRING / REST CODE SNIPPETS

## G1. Complete CRUD REST controller + service + exception handling
```java
@RestController @RequestMapping("/api/employees") @RequiredArgsConstructor
class EmployeeController {
    private final EmployeeService service;

    @GetMapping("/{id}")  ResponseEntity<EmployeeDto> get(@PathVariable Long id) { return ResponseEntity.ok(service.get(id)); }
    @GetMapping           Page<EmployeeDto> list(Pageable p) { return service.list(p); }
    @PostMapping          ResponseEntity<EmployeeDto> create(@Valid @RequestBody EmployeeDto d) {
        EmployeeDto saved = service.create(d);
        return ResponseEntity.created(URI.create("/api/employees/" + saved.id())).body(saved);
    }
    @PutMapping("/{id}")  EmployeeDto update(@PathVariable Long id, @Valid @RequestBody EmployeeDto d) { return service.update(id, d); }
    @DeleteMapping("/{id}") ResponseEntity<Void> delete(@PathVariable Long id) { service.delete(id); return ResponseEntity.noContent().build(); }
}

@Service @RequiredArgsConstructor
class EmployeeService {
    private final EmployeeRepository repo;
    @Transactional(readOnly = true)
    EmployeeDto get(Long id) {
        return repo.findById(id).map(this::toDto).orElseThrow(() -> new ResourceNotFoundException("Employee " + id));
    }
    // create/update/delete → @Transactional (default)
}
interface EmployeeRepository extends JpaRepository<Employee, Long> {
    List<Employee> findByDeptAndSalaryGreaterThan(String dept, double salary);   // derived query
}
```

## G2. Custom annotation + AOP (logging execution time)
```java
@Target(ElementType.METHOD) @Retention(RetentionPolicy.RUNTIME)
public @interface LogTime {}

@Aspect @Component
class TimeAspect {
    @Around("@annotation(LogTime)")
    Object measure(ProceedingJoinPoint pjp) throws Throwable {
        long s = System.currentTimeMillis();
        try { return pjp.proceed(); }
        finally { log.info("{} took {} ms", pjp.getSignature(), System.currentTimeMillis() - s); }
    }
}
```

## G3. JWT filter skeleton (ties to your Security notes)
```java
class JwtFilter extends OncePerRequestFilter {
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        String h = req.getHeader("Authorization");
        if (h != null && h.startsWith("Bearer ")) {
            String token = h.substring(7);
            if (jwtUtil.isValid(token)) {
                UserDetails u = userDetailsService.loadUserByUsername(jwtUtil.username(token));
                var auth = new UsernamePasswordAuthenticationToken(u, null, u.getAuthorities());
                SecurityContextHolder.getContext().setAuthentication(auth);
            }
        }
        chain.doFilter(req, res);
    }
}
```

## G4. Retry + circuit breaker (Resilience4j)
```java
@CircuitBreaker(name = "payment", fallbackMethod = "fallback")
@Retry(name = "payment")
public PaymentResponse charge(PaymentRequest r) { return client.charge(r); }
PaymentResponse fallback(PaymentRequest r, Throwable t) { return PaymentResponse.pending(); }
```

## G5. Kafka listener with idempotency
```java
@KafkaListener(topics = "orders", groupId = "billing")
@Transactional
public void on(OrderEvent e) {
    if (processedRepo.existsById(e.eventId())) return;      // already handled → skip
    billingService.bill(e);
    processedRepo.save(new Processed(e.eventId()));         // same tx as business change
}
```

## G6. N+1 fix
```java
@Query("select o from Order o join fetch o.items where o.customerId = :id")
List<Order> findWithItems(@Param("id") Long id);
// or @EntityGraph(attributePaths = "items"), or @BatchSize(size = 50)
```
