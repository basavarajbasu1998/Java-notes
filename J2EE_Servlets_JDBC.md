# Java + J2EE (Servlets, Filters, JDBC to JPA)

## What is J2EE / Java EE / Jakarta EE?
A **set of specifications** for enterprise (server-side) apps. Old name J2EE → Java EE → now **Jakarta EE** (`jakarta.*` packages). Spring builds on/replaces much of it.

| Spec | What | Spring Boot equivalent |
|---|---|---|
| **Servlet** | Java class handling HTTP | `@RestController` (runs on DispatcherServlet) |
| **JSP** | HTML with Java (server-rendered pages) | Thymeleaf / React |
| **JDBC** | talk to database | Spring JDBC / JPA |
| **JPA** | ORM standard | Spring Data JPA (Hibernate) |
| **EJB** | managed business components, transactions | `@Service` + `@Transactional` |
| **JMS** | messaging API | RabbitMQ/Kafka templates |
| **JNDI** | lookup resources by name | dependency injection / properties |
| **CDI** | dependency injection | Spring IoC |
| **JAX-RS** | REST | Spring MVC |

## Web container flow (Tomcat) — foundation of every Spring Boot request
```
Browser ─HTTP─► Tomcat (Servlet container)
                   │ 1. parse request → HttpServletRequest/Response objects
                   │ 2. Filter chain (Filter.doFilter): auth, logging, CORS, encoding
                   │ 3. Find servlet by URL mapping (web.xml or @WebServlet)
                   │ 4. servlet.service() → doGet()/doPost()
                   │ 5. Write response → back through filters → client
```
Servlet lifecycle: **load class → `init()` (once) → `service()` (each request, multithreaded, ONE instance!) → `destroy()`**. Because one servlet instance serves all threads, servlets must be stateless/thread-safe.

```java
@WebServlet("/hello")
public class HelloServlet extends HttpServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse res) throws IOException {
        String name = req.getParameter("name");
        res.setContentType("text/plain");
        res.getWriter().println("Hello " + name);
    }
}
```
- **Session tracking:** cookie `JSESSIONID` ↔ `HttpSession` on server (stateful). JWT = stateless alternative.
- **forward vs redirect:** forward = server-side, same request, URL unchanged; redirect = 302 to client, new request, URL changes.
- **GET vs POST**, **request/session/application scope**, **Filter vs Listener vs Servlet**.
- **WAR** (deploy to external Tomcat) vs **executable JAR** (Boot embeds Tomcat).

## JDBC → JPA → Spring Data (evolution of DB access)
```java
// 1. JDBC (raw, verbose)
try (Connection c = ds.getConnection();
     PreparedStatement ps = c.prepareStatement("SELECT * FROM orders WHERE id=?")) {
    ps.setLong(1, id);
    try (ResultSet rs = ps.executeQuery()) { if (rs.next()) { /* map manually */ } }
}
// PreparedStatement prevents SQL injection & is pre-compiled.

// 2. JPA/Hibernate: map class ↔ table (@Entity), EntityManager
// 3. Spring Data: interface OrderRepository extends JpaRepository<Order, Long> {}  → zero implementation code
```

## Core Java revision map (jump to your notes)
Collections → `Collections.md` · Streams/Lambda → `Java_8.md` · Multithreading, JVM, GC, OOP, Exceptions → `JavaFullstack.md` · Deep dives → `Java_Memory_Model_and_Locks.md`, `Core_Java_Deep_Points.md` · Problems → `Coding_*.md` files. Full index: `README.md`.
