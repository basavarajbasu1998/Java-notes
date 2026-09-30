# Testing: JUnit 5 + Mockito

## Test pyramid
```
        /\        few  – E2E / UI       (slow, brittle)
       /  \
      /----\      some – Integration    (Spring context, real DB via Testcontainers)
     /------\
    /--------\    many – Unit tests     (fast, no Spring, no DB)
```

## Unit test with Mockito
**Mock** = fake dependency that we control. Test only ONE class.

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock PaymentClient paymentClient;
    @Mock OrderRepository repo;
    @InjectMocks OrderService service;          // mocks injected into this

    @Test
    void placeOrder_paymentSuccess_savesOrder() {
        // Arrange (Given)
        when(paymentClient.charge(100)).thenReturn(true);
        // Act (When)
        service.place(new Order(100));
        // Assert (Then)
        verify(repo, times(1)).save(any(Order.class));
    }

    @Test
    void placeOrder_paymentFails_throws() {
        when(paymentClient.charge(100)).thenReturn(false);
        assertThrows(PaymentException.class, () -> service.place(new Order(100)));
        verify(repo, never()).save(any());
    }
}
```
- **Stub** (`when().thenReturn`) = give canned answer. **Verify** = check interaction happened.
- **Mock vs Spy:** mock = fully fake; spy = real object, some methods overridden.
- `@Mock` (Mockito, plain) vs `@MockBean` / `@MockitoBean` (put mock into Spring context).
- Void method: `doThrow(...).when(mock).method()`.
- Capture argument: `ArgumentCaptor`.

## Spring test slices
| Annotation | Loads | Use |
|---|---|---|
| `@WebMvcTest(Controller.class)` | only web layer (+ MockMvc) | controller tests |
| `@DataJpaTest` | only JPA + in-memory DB | repository tests |
| `@SpringBootTest` | full context | integration |
```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mvc;
    @MockBean OrderService service;

    @Test void get_returns200() throws Exception {
        when(service.find(1L)).thenReturn(new OrderResponse(1L));
        mvc.perform(get("/api/orders/1")).andExpect(status().isOk())
           .andExpect(jsonPath("$.id").value(1));
    }
}
```
Use **Testcontainers** for real DB/Kafka in integration tests instead of H2 (H2 behaves differently from prod).

Good tests: FIRST (Fast, Independent, Repeatable, Self-validating, Timely); cover happy path + edge + failure; coverage (JaCoCo) is a guide, not a goal; **TDD** = write failing test → code → refactor.
