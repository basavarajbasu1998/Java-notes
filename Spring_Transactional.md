# Spring Transactions — Expert Interview Note (Spring 6 / Boot 3, JDK 17/21)

> Scope: `@Transactional` end to end: proxy creation, interceptor, transaction manager, thread binding,
> propagation, rollback rules, isolation, readOnly, timeout, pitfalls, events, messaging, testing, retries, distributed tx.
> Confidence legend: statements about *behaviour* are stable across Spring 5.3 -> 6.x. Where an internal detail
> changed between versions or I am not certain of the exact name, the text says "conceptually" or "verify".

## Table of contents
1. 60-second mental model + analogy
2. Deep internals (setup, call trace, flowcharts, thread binding)
3. Traced worked examples (every propagation, rollback-only, savepoints)
4. Rollback rules, isolation, readOnly, timeout
5. Proxy pitfalls and fixes, TransactionTemplate
6. Events, messaging, outbox, long transactions, OSIV
7. Retries, deadlocks, optimistic locking, distributed transactions, testing
8. Failure modes and production incident stories
9. Interview questions (35) with follow-ups and common wrong answers
10. Runnable code
11. One-page cheat sheet

---
# 1. The 60-second mental model

## Simple version
A transaction = "all my DB changes happen together or not at all".
`@Transactional` = "Spring, please open the transaction before this method, and close it (commit/rollback) after".

Spring does that by **wrapping your bean in a proxy**. The proxy is not magic: it is an AOP advice
(`TransactionInterceptor`) that does `begin -> call your method -> commit or rollback`.

## Analogy: the hotel key card
- Transaction = **one hotel room booking** (physical). Everything you do in the room is on one bill.
- `REQUIRED` = you walk into a room someone else already booked: same bill (you *participate*).
- `REQUIRES_NEW` = you ask reception for a **separate room and bill**; the first room is locked/put on hold (*suspended*).
- `NESTED` = same room, but you take a **photo of the room before an experiment** (savepoint). If the experiment goes wrong, restore the photo, not check out.
- The **key card is a ThreadLocal**: it works only for the person (thread) who got it. Hand the task to another thread and the card does not open the door (no tx).
- Rollback-only = a housekeeping note on the bill: "this bill must be voided". Anyone in the room can write it; only the person who booked (outermost) can act on it, and if they try to pay (commit) they get `UnexpectedRollbackException`.

## The 8 sentences you must be able to say
1. Spring creates a **proxy** (JDK or CGLIB; Boot defaults to CGLIB) via an auto-proxy creator that finds a transaction **advisor**.
2. The advisor's pointcut = "does `AnnotationTransactionAttributeSource` return a `TransactionAttribute` for this method?".
3. The advice `TransactionInterceptor.invoke` -> `TransactionAspectSupport.invokeWithinTransaction`.
4. That asks a `PlatformTransactionManager` for a `TransactionStatus` (`getTransaction`), runs the method, then `commit` or `rollback`.
5. The connection (or `EntityManager`) is bound to the **current thread** in `TransactionSynchronizationManager` ThreadLocals, so `JdbcTemplate`/Hibernate find it without you passing it.
6. Default rollback: `RuntimeException` and `Error` only. Checked exceptions **commit**.
7. Propagation decides physical vs logical transaction; a *logical* inner scope that fails marks the *physical* tx **rollback-only**.
8. Everything above only happens if the call **goes through the proxy** (self-invocation, private, final, non-bean = no tx).

---
# 2. Deep internals

## 2.1 Setup: how the advisor gets into the container

`@EnableTransactionManagement` (Boot: `TransactionAutoConfiguration` does it for you) does:

```
@EnableTransactionManagement
   └─ @Import(TransactionManagementConfigurationSelector)
        ├─ mode = PROXY (default) imports:
        │     ├─ AutoProxyRegistrar
        │     │     └─ registers InfrastructureAdvisorAutoProxyCreator  (a BeanPostProcessor)
        │     │        (only picks up advisors with ROLE_INFRASTRUCTURE)
        │     └─ ProxyTransactionManagementConfiguration   (@Configuration, @Role(INFRASTRUCTURE))
        │           ├─ @Bean BeanFactoryTransactionAttributeSourceAdvisor
        │           │        ├─ pointcut: TransactionAttributeSourcePointcut
        │           │        └─ advice:   TransactionInterceptor
        │           ├─ @Bean TransactionAttributeSource  = AnnotationTransactionAttributeSource
        │           └─ @Bean TransactionInterceptor      (holds attributeSource + txManager)
        └─ mode = ASPECTJ imports AspectJTransactionManagementConfiguration
              (needs spring-aspects; weaving does the work, no proxies)
```

Attributes of `@EnableTransactionManagement`: `proxyTargetClass` (default false in plain Spring, but Boot sets
`spring.aop.proxy-target-class=true` so CGLIB is the Boot default), `mode`, `order` (default `Ordered.LOWEST_PRECEDENCE`).

**Bean creation time:** for every bean, `AbstractAutoProxyCreator.postProcessAfterInitialization`
-> `wrapIfNecessary` -> `getAdvicesAndAdvisorsForBean` -> for each advisor, `AopUtils.canApply(pointcut, targetClass)`.
The pointcut's `matches(method, targetClass)` = `transactionAttributeSource.getTransactionAttribute(method, targetClass) != null`.
If any method of the class matches -> a proxy is created (`ProxyFactory` -> `DefaultAopProxyFactory` -> `JdkDynamicAopProxy` or `ObjenesisCglibAopProxy`).

Consequences:
- The proxy is created **after** `@PostConstruct`/`afterPropertiesSet` of the target. So a `@Transactional` call *inside* `@PostConstruct` is on the raw object: no tx. Use `ApplicationReadyEvent`/`SmartInitializingSingleton` or `TransactionTemplate`.
- A bean created too early (e.g. injected into a `BeanPostProcessor`) logs *"Bean 'x' is not eligible for getting processed by all BeanPostProcessors"* and is never proxied.
- Only beans matter. `new OrderService()` is never proxied.

## 2.2 Attribute lookup: `AnnotationTransactionAttributeSource`

`AbstractFallbackTransactionAttributeSource.getTransactionAttribute(method, targetClass)` (result cached per `(method, targetClass)`):

Order of lookup in `computeTransactionAttribute`:
1. If `allowPublicMethodsOnly()` and the method is not public -> `null`.
   - Spring 5.x: `AnnotationTransactionAttributeSource` defaults to `publicMethodsOnly = true`.
   - **Spring 6.0+**: protected and package-private methods are also transactional with **class-based (CGLIB)** proxies. With JDK proxies only interface methods are reachable anyway. `private` never works.
2. `specificMethod = AopUtils.getMostSpecificMethod(method, targetClass)` (the implementation method on the target class).
3. Look for annotation on **specificMethod** (target class method).
4. Then on the **target class** itself (class-level `@Transactional`).
5. If `specificMethod != method` (i.e. the call came through an interface): look on the **interface method**, then on the **interface's declaring class**.
6. None -> `null` -> no advice for that method.

So: method-level beats class-level; implementation beats interface; class-level applies to all *eligible* methods. The attribute is *not* merged: method-level annotation replaces class-level completely (`@Transactional(readOnly=true)` on class + bare `@Transactional` on a method = the method is **read-write, default rollback**).

Parsers: `SpringTransactionAnnotationParser` (`org.springframework.transaction.annotation.Transactional`),
`JtaTransactionAnnotationParser` (`jakarta.transaction.Transactional`; attribute names `rollbackOn`/`dontRollbackOn`), `Ejb3TransactionAnnotationParser`.
Spring's is normally what you want.

Guidance in the reference docs: annotate concrete classes, not interfaces. Interface annotations are only reliably honoured with interface-based (JDK) proxies.

## 2.3 The call trace (numbered)

Scenario: `controller -> orderService.placeOrder(o)` where `OrderService` is a CGLIB proxy, `JpaTransactionManager`.

```
 1. Controller invokes proxy.placeOrder(o)
 2. CglibAopProxy.DynamicAdvisedInterceptor.intercept()
      builds/gets cached interceptor chain for the Method  -> [TransactionInterceptor]
      wraps in CglibMethodInvocation, calls proceed()
 3. TransactionInterceptor.invoke(invocation)
      targetClass = AopUtils.getTargetClass(invocation.getThis())
      -> invokeWithinTransaction(method, targetClass, invocation::proceed)
 4. TransactionAspectSupport.invokeWithinTransaction
      a. tas = getTransactionAttributeSource(); txAttr = tas.getTransactionAttribute(method, targetClass)
      b. tm = determineTransactionManager(txAttr)      // @Transactional("qualifier") or by type/bean name
      c. (reactive types/Kotlin suspend handled by a separate branch — skipped here)
      d. ptm = asPlatformTransactionManager(tm); joinpointIdentification = "com...OrderService.placeOrder"
      e. txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification)
            └─ ptm.getTransaction(txAttr)   ── see 2.4 ──►  TransactionStatus
            └─ prepareTransactionInfo(...)  binds TransactionInfo to a ThreadLocal (transactionInfoHolder)
 5. try { retVal = invocation.proceedWithInvocation();   // YOUR METHOD (real target object)
    } catch (Throwable ex) {
         completeTransactionAfterThrowing(txInfo, ex);   // rollbackOn(ex)? rollback : commit
         throw ex;
    } finally {
         cleanupTransactionInfo(txInfo);                 // restore previous TransactionInfo (nesting)
    }
 6. commitTransactionAfterReturning(txInfo) -> ptm.commit(txInfo.getTransactionStatus())
 7. return retVal
```

Details worth quoting:
- `completeTransactionAfterThrowing`: if `txInfo.transactionAttribute.rollbackOn(ex)` -> `ptm.rollback(status)`; else -> `ptm.commit(status)` (yes: commit, even though an exception is propagating). If the rollback itself throws, that exception is logged/thrown and the original is masked in some paths (`TransactionSystemException` is thrown carrying the app exception as `getApplicationException()`).
- `DefaultTransactionAttribute.rollbackOn(ex)` = `ex instanceof RuntimeException || ex instanceof Error`.
- `RuleBasedTransactionAttribute` (what `@Transactional(rollbackFor..)` builds) evaluates rules first (closest match by class hierarchy depth wins), falls back to the default.
- If the manager is a `CallbackPreferringPlatformTransactionManager` (WebSphere-style) the callback branch is used; irrelevant for normal setups.
- The commit happens **after your method returned**, in the interceptor. Exceptions thrown by the flush/commit (constraint violation, deferred FK, serialization failure, optimistic lock at flush) surface **outside** your method's `try/catch`.

## 2.4 `AbstractPlatformTransactionManager.getTransaction` (the state machine)

```
getTransaction(definition)
  │
  ├─ transaction = doGetTransaction()      // subclass: looks up resource holder bound to THIS thread
  │
  ├─ isExistingTransaction(transaction)? ────────────────────────────►  handleExistingTransaction()  (2.5)
  │
  ├─ timeout < TIMEOUT_DEFAULT ? -> InvalidTimeoutException
  │
  ├─ MANDATORY ?      ─► throw IllegalTransactionStateException("No existing transaction found for transaction marked with propagation 'mandatory'")
  │
  ├─ REQUIRED / REQUIRES_NEW / NESTED ?
  │        suspendedResources = suspend(null)      // nothing bound, but clears leftover synchronizations
  │        status = newTransactionStatus(def, tx, newTransaction=true, newSynchronization, ...)
  │        doBegin(tx, def)                        // subclass: connection/EntityManager acquired + bound
  │        prepareSynchronization(status, def)     // sets TSM: actualTransactionActive, isolation, readOnly, name, initSynchronization()
  │        return status
  │
  └─ otherwise (SUPPORTS / NOT_SUPPORTED / NEVER with no tx):
           "empty" transaction: no physical tx, but synchronization is still initialised
           (transactionSynchronization = SYNCHRONIZATION_ALWAYS, the default)
           return status(newTransaction=false)
```

### `DataSourceTransactionManager.doBegin` (JDBC)
```
1. con = obtainDataSource().getConnection()                       (from HikariCP)
2. previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition)
      - if readOnly && enforceReadOnly:  (default false in DSTM: no SET TRANSACTION READ ONLY sent by Spring itself;
                                           the con.setReadOnly(true) hint is still set)
      - if isolation != DEFAULT:         con.setTransactionIsolation(level)
3. if con.getAutoCommit(): mustRestoreAutoCommit = true; con.setAutoCommit(false)   // this is the "BEGIN"
4. txObject.getConnectionHolder().setTransactionActive(true)
5. timeout != DEFAULT -> holder.setTimeoutInSeconds(timeout)      // deadline, see 4.4
6. if new holder: TransactionSynchronizationManager.bindResource(dataSource, holder)   // ThreadLocal put
```
Note: `setAutoCommit(false)` is the physical begin: with Hikari's default `autoCommit=true` that is one extra round trip; set `spring.datasource.hikari.auto-commit=false` and (for Hibernate) `hibernate.connection.provider_disables_autocommit=true` to save it and to delay connection acquisition until first statement.

### `JpaTransactionManager.doBegin`
```
1. em = createEntityManagerForTransaction()   (unless bound by OpenEntityManagerInView)
2. HibernateJpaDialect.beginTransaction(em, def):
      - applies isolation (only via HibernateJpaDialect; plain JpaDialect throws InvalidIsolationLevelException)
      - readOnly -> session.setDefaultReadOnly(true) and FlushMode.MANUAL, plus connection.setReadOnly(true) hint
      - timeout  -> transaction timeout on the Hibernate Transaction / queries
      - em.getTransaction().begin()
3. bindResource(entityManagerFactory, new EntityManagerHolder(em))
4. If a DataSource is known and dialect exposes the JDBC connection:
      bindResource(dataSource, new ConnectionHolder(connectionHandle))
      -> so JdbcTemplate and JPA in the SAME method share ONE connection and ONE tx
```

## 2.5 `handleExistingTransaction` (what propagation actually does)

```
existing physical tx found on this thread:
  NEVER          ─► IllegalTransactionStateException("Existing transaction found for transaction marked with propagation 'never'")
  NOT_SUPPORTED  ─► suspend(tx); run with NO tx (status.newTransaction=false); resume afterwards
  REQUIRES_NEW   ─► suspended = suspend(tx)   // unbind resources+synchronizations from thread, keep them in SuspendedResourcesHolder
                     doBegin(...)             // NEW connection from pool (a 2nd connection! pool pressure)
                     on failure in begin: resumeAfterBeginException
  NESTED         ─► if !nestedTransactionAllowed → NestedTransactionNotSupportedException
                     if useSavepointForNestedTransaction():        // JDBC-based managers (DSTM, JpaTransactionManager+Hibernate)
                         status.createAndHoldSavepoint()           // con.setSavepoint()
                     else: nested begin/commit/rollback via synchronization (JTA only)
  SUPPORTS / REQUIRED / MANDATORY (default branch) ─► PARTICIPATE
                     if (validateExistingTransaction) check isolation & readOnly mismatch → IllegalTransactionStateException
                     else:  silently ignore inner isolation/readOnly/timeout   <---- important
                     return status(newTransaction=false)
```
`validateExistingTransaction` defaults to **false**: `@Transactional(isolation = SERIALIZABLE)` joining a READ_COMMITTED tx is **silently ignored**. Only the outermost (creating) scope applies isolation, timeout and readOnly. Set `txManager.setValidateExistingTransaction(true)` to fail fast.

## 2.6 Commit and rollback (the logical/physical rules)

```
commit(status):
  if status.isCompleted → IllegalTransactionStateException
  if status.isLocalRollbackOnly()  ── (setRollbackOnly() on THIS scope)
        processRollback(status, unexpected=false)   ; return
  if !shouldCommitOnGlobalRollbackOnly() && status.isGlobalRollbackOnly()   ── (tx marked by someone else, e.g. inner scope)
        processRollback(status, unexpected=true)    ; return     ← ends in UnexpectedRollbackException (if this is the outer/new tx)
  processCommit(status):
        triggerBeforeCommit(status)      // TransactionSynchronization.beforeCommit
        triggerBeforeCompletion(status)
        if status.hasSavepoint():   status.releaseHeldSavepoint()           // NESTED "commit" = release savepoint only
        else if status.isNewTransaction(): doCommit(status)                 // ONLY the outermost scope of a physical tx really commits
        (else participating: nothing happens)
        triggerAfterCommit(status)       // @TransactionalEventListener(AFTER_COMMIT) fires here
        triggerAfterCompletion(status, STATUS_COMMITTED)
        cleanupAfterCompletion(status)   // unbind, restore autocommit/isolation/readOnly, release connection, RESUME suspended tx

rollback(status): processRollback(status, false)
  processRollback:
        triggerBeforeCompletion
        if status.hasSavepoint():        status.rollbackToHeldSavepoint()   // NESTED: undo just inner part
        else if status.isNewTransaction(): doRollback(status)               // real ROLLBACK
        else (participating):
              if status.hasTransaction() && (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure() /*default true*/)
                    doSetRollbackOnly(status)     // mark the SHARED physical tx: connectionHolder.setRollbackOnly()
        triggerAfterCompletion(ROLLED_BACK)
        if unexpectedRollback → throw UnexpectedRollbackException("Transaction rolled back because it has been marked as rollback-only")
```
The key insight: **an inner `@Transactional` (REQUIRED) that fails does not roll back; it *marks* the shared tx rollback-only.** The real rollback only happens when the outermost scope finishes, and if that outer scope tried to commit it gets `UnexpectedRollbackException`.

`failEarlyOnGlobalRollbackOnly` (default false) lets the manager throw at the inner scope's completion instead of at the outermost commit.

## 2.7 `TransactionSynchronizationManager` and thread binding

All state is in `NamedThreadLocal`s (not `InheritableThreadLocal`):

| ThreadLocal | Holds |
|---|---|
| `resources` | `Map<Object, Object>`: key = `DataSource` / `EntityManagerFactory` -> `ConnectionHolder` / `EntityManagerHolder` |
| `synchronizations` | `Set<TransactionSynchronization>` (callbacks registered for this tx) |
| `currentTransactionName` | name = `com...Service.method` |
| `currentTransactionReadOnly` | `Boolean` |
| `currentTransactionIsolationLevel` | `Integer` |
| `actualTransactionActive` | `Boolean` (true only if a *physical* tx is active; false for SUPPORTS-empty) |

How `JdbcTemplate` joins your tx: `DataSourceUtils.getConnection(dataSource)` -> `TransactionSynchronizationManager.getResource(dataSource)`; if a holder exists it reuses its connection, else opens a fresh **autocommit** connection (each statement its own tx!). Same idea for `EntityManagerFactoryUtils.doGetTransactionalEntityManager` and the shared `EntityManager` proxy injected by `@PersistenceContext`.

### Why tx is lost on other threads
```
   main thread (http-nio-8080-exec-3)                 pool thread (task-1)
   ┌───────────────────────────────┐                  ┌───────────────────────────┐
   │ ThreadLocal resources:        │   submit()       │ ThreadLocal resources:    │
   │   DataSource -> ConnHolder#42 │ ───────────────► │   (empty)                 │
   │ actualTransactionActive=true  │                  │ actualTransactionActive=∅ │
   └───────────────────────────────┘                  └───────────────────────────┘
      order saved, NOT committed yet                      cannot see the row (READ_COMMITTED)
                                                          repo call runs in its own autocommit tx
```
Applies to: `@Async`, `CompletableFuture.supplyAsync`, `ExecutorService`, `parallelStream()` (ForkJoin common pool),
`@Scheduled` (own thread), Reactor `publishOn/subscribeOn` (blocking `@Transactional` + reactive = wrong tool; use `ReactiveTransactionManager` + `TransactionalOperator`, which stores the tx in the Reactor `Context`, not a ThreadLocal).
Virtual threads (JDK 21) still have ThreadLocals, so behaviour is identical per virtual thread; the tx does not "follow" a task to another virtual thread either.
Also: an ambient `@Transactional` **does not propagate** into a `@Async` method; `@Async` + `@Transactional` on the same method = the async method opens **its own** tx on the async thread, which is fine and often intended.

The 40-line runnable simulation in section 10.1 demonstrates the thread-hop (verified with `javac`/`java` on JDK 21).

## 2.8 Which transaction manager

| Manager | Resource bound | Notes |
|---|---|---|
| `DataSourceTransactionManager` | `DataSource -> ConnectionHolder` | JDBC / JdbcTemplate / MyBatis / jOOQ. Savepoints supported. |
| `JpaTransactionManager` | `EMF -> EntityManagerHolder` (+ `DataSource -> ConnectionHolder` if exposed) | JPA/Hibernate. Also serves plain JDBC on the *same* DataSource in the same tx. Supports isolation/readOnly via `HibernateJpaDialect`. Boot auto-configures this when JPA present. |
| `HibernateTransactionManager` | `SessionFactory` | Native Hibernate API (legacy, removed from Spring 6 core support of Hibernate 5 native; use JPA). |
| `JtaTransactionManager` | `UserTransaction`/`TransactionManager` | Global XA tx across several resources; needs a JTA provider (Narayana; app-server). |
| `KafkaTransactionManager`, `JmsTransactionManager`, `RabbitTransactionManager` | broker resource | **Independent** local transactions per resource, not XA. |
| `R2dbcTransactionManager` etc. | `ReactiveTransactionManager` | Context based. |

Boot auto-configures **one** manager. With several DataSources you must pick: `@Transactional("ordersTm")` (the `transactionManager` / `value` attribute), otherwise `NoUniqueBeanDefinitionException`.
A tx on `tmA` does **not** cover a repository bound to `dataSourceB`: silent non-atomicity.

## 2.9 Ordering with other advice
`TransactionInterceptor` order = `@EnableTransactionManagement(order=...)`, default `LOWEST_PRECEDENCE` (innermost, closest to your code).
Any other advisor (`@Aspect` for logging, `@Retryable`, `@Cacheable`, `@Async`, security) is ordered relative to it by `@Order`/`Ordered`.
Lower number = outer = runs first. When two advisors have the same order, the order is unspecified: **always set explicit order** when order matters (retry, caching of transactional results, audit).

---
# 3. Traced worked examples

Assume `JpaTransactionManager`, Hikari pool, table `orders(id, status)` and `audit(id, msg)`.
Log lines are representative/abridged output with
`logging.level.org.springframework.transaction.interceptor=TRACE` and
`logging.level.org.springframework.orm.jpa.JpaTransactionManager=DEBUG`. Exact wording can vary by version, the *sequence* is what matters.

## 3.1 REQUIRED + REQUIRED (join) - one physical, two logical

```java
@Service class OrderService {
  @Autowired PaymentService payments;
  @Transactional public void place(long id) { orders.save(new Order(id,"NEW")); payments.charge(id); }
}
@Service class PaymentService {
  @Transactional public void charge(long id) { pay.save(new Payment(id)); }
}
```
```
TRACE ... Getting transaction for [OrderService.place]            <- outer: logical #1 = PHYSICAL tx
DEBUG ... Creating new transaction with name [OrderService.place]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
DEBUG ... Opened new EntityManager [SessionImpl] for JPA transaction
DEBUG ... Exposing JPA transaction as JDBC [HikariProxyConnection@1001]
sql> insert into orders (status,id) values ('NEW',7)               (deferred until flush/commit, shown for order)
TRACE ... Getting transaction for [PaymentService.charge]         <- inner: logical #2
DEBUG ... Participating in existing transaction
TRACE ... Completing transaction for [PaymentService.charge]      <- NOTHING is committed here
TRACE ... Completing transaction for [OrderService.place]
DEBUG ... Initiating transaction commit
DEBUG ... Committing JPA transaction on EntityManager [...]
sql> insert ... orders ; insert ... payment ; COMMIT               <- flush + one COMMIT on connection 1001
```
Connections used: **1**. Commits: **1**. Inner failure -> marks rollback-only (see 3.6).

## 3.2 REQUIRED -> REQUIRES_NEW (suspend) - two physical transactions

```java
@Transactional public void placeOrder(long id) {           // Tx-1
    orders.save(new Order(id,"NEW"));
    audit.log("attempt " + id);                              // Tx-2
    throw new IllegalStateException("payment failed");
}
@Transactional(propagation = REQUIRES_NEW) public void log(String m) { auditRepo.save(new Audit(m)); }
```
```
Creating new transaction with name [OrderService.placeOrder]        conn A (HikariProxyConnection@1001)
Suspending current transaction, creating new transaction with name [AuditService.log]   conn B (@1002)
   ── TSM resources for thread: A unbound & parked in SuspendedResourcesHolder, B bound ──
Initiating transaction commit  (Tx-2)   sql> insert into audit ...; COMMIT   on B   <- visible to everyone NOW
Resuming suspended transaction after completion of inner transaction                 A rebound
Completing transaction for [OrderService.placeOrder] after exception: java.lang.IllegalStateException
Initiating transaction rollback   (Tx-1) sql> ROLLBACK on A
```
Result: `orders` empty, `audit` has the row. **Two connections held simultaneously by one thread**: with pool size N and N concurrent requests, all N threads each hold connection A and wait for connection B => **pool deadlock** (Section 8, incident 3).
Also: Tx-2 cannot see Tx-1's uncommitted `orders` row (READ_COMMITTED); if Tx-2 inserts a child with FK to an uncommitted parent it **blocks on the parent's lock/ FK check** until Tx-1 ends, while Tx-1 waits for Tx-2 -> self-deadlock (DB shows a lock wait, app hangs until lock timeout).

## 3.3 NESTED (savepoint) - one physical tx, partial rollback

```java
@Transactional public void batch(List<Item> items) {                 // physical tx
    for (Item i : items) {
        try { itemService.importOne(i); }                             // NESTED
        catch (BadItemException e) { log.warn("skipped {}", i.id()); }
    }
}
@Transactional(propagation = NESTED) public void importOne(Item i) { repo.save(i); validate(i); }
```
JDBC-level trace (item 2 invalid):
```
BEGIN (autocommit off)
SAVEPOINT SAVEPOINT_1          insert item 1       RELEASE SAVEPOINT_1      <- "commit" of nested = release only
SAVEPOINT SAVEPOINT_2          insert item 2 -> validate throws
                               ROLLBACK TO SAVEPOINT_2                       <- only item 2 undone
SAVEPOINT SAVEPOINT_3          insert item 3       RELEASE SAVEPOINT_3
COMMIT                                                                       <- items 1 and 3 saved
```
Important caveats:
- Requires a JDBC-savepoint-capable manager: `DataSourceTransactionManager`, or `JpaTransactionManager` with `HibernateJpaDialect` (Boot default). Not supported by plain JPA-only dialects (`NestedTransactionNotSupportedException`).
- With JPA the **persistence context is not rolled back**: the entity `i` persisted in the failed nested scope may still be managed in the `EntityManager` and be flushed later (or cause `PersistenceException` inconsistencies). Use `em.clear()`/`detach` in the catch, or prefer `REQUIRES_NEW` for isolation of failures with JPA. Nested is most natural with `JdbcTemplate`.
- Outer failure rolls back everything including released savepoints. Nested inner is **not** durable on its own.
- With `catch` outside the proxy boundary (as above, in a different bean) the nested marks nothing global: `NESTED` failure does *not* set the outer tx rollback-only (the savepoint absorbs it). This is exactly the difference from REQUIRED.

## 3.4 SUPPORTS / NOT_SUPPORTED / MANDATORY / NEVER

| Caller has tx? | SUPPORTS | NOT_SUPPORTED | MANDATORY | NEVER |
|---|---|---|---|---|
| yes (Tx-1) | joins Tx-1 | **suspends** Tx-1, runs non-tx, resumes | joins Tx-1 | `IllegalTransactionStateException` |
| no | runs non-tx (but synchronization active) | runs non-tx | `IllegalTransactionStateException` | runs non-tx |

`NOT_SUPPORTED` use: call a slow reporting/HTTP-ish method without holding the connection -> the connection stays *bound but suspended*; **it is still checked out of the pool** while suspended (the physical connection is held in the suspended holder!). So `NOT_SUPPORTED` does NOT relieve pool pressure; it only stops the work being part of the tx. To free the connection, structure the code so the outer tx ends first.

`SUPPORTS` gotcha: with no tx, a `JdbcTemplate` call runs in autocommit, and JPA `readOnly` semantics / flush behaviour differ: a `SUPPORTS` method that *modifies* JPA entities will not flush without a tx.

## 3.5 Which of these are physical?

```
 Case                       Physical tx created?    Connection count    Commit happens at
 REQUIRED (none exists)     yes (owner)             1                   this scope's exit
 REQUIRED (joins)           no  (participant)       0 extra             owner's exit
 REQUIRES_NEW               yes always              +1                  this scope's exit
 NESTED                     no (savepoint)          0 extra             owner's exit (release at scope exit)
 NOT_SUPPORTED              no (suspends)           0 extra (held)      n/a
 SUPPORTS(with tx)          no (participant)        0 extra             owner's exit
```

## 3.6 Rollback-only and `UnexpectedRollbackException` (worked example)

```java
@Service class A {
  @Autowired B b;
  @Transactional
  public void run() {
      repo.save(new Order(1));
      try { b.risky(); }                              // B is a different bean -> goes via proxy
      catch (RuntimeException e) { log.warn("ignored", e); }   // "I handled it"
      repo.save(new Order(2));
  }                                                   // proxy tries COMMIT
}
@Service class B {
  @Transactional                                       // REQUIRED -> participates
  public void risky() { throw new IllegalArgumentException("bad"); }
}
```
Trace:
```
A.run  : getTransaction  -> new physical tx (Tx-1)
B.risky: getTransaction  -> Participating in existing transaction
B.risky: throws IllegalArgumentException (RuntimeException)
         TransactionInterceptor(B).completeTransactionAfterThrowing -> rollbackOn=true
         -> ptm.rollback(status of B):  status.isNewTransaction()==false
              -> participating: doSetRollbackOnly()   DEBUG "Participating transaction failed - marking existing transaction as rollback-only"
A.run  : catch swallows; save(Order 2)
A.run  : returns normally -> commitTransactionAfterReturning -> ptm.commit(statusA)
              isGlobalRollbackOnly() == true  -> processRollback(unexpected=true)
              -> real ROLLBACK, then
              throw UnexpectedRollbackException:
                 "Transaction silently rolled back because it has been marked as rollback-only"
```
The caller of `A.run()` gets an exception even though `A` "caught everything" and both saves are lost.
Fixes (pick by intent):
1. Really want partial success -> `B.risky` as `REQUIRES_NEW` (separate tx; its rollback does not poison Tx-1) or `NESTED` (savepoint).
2. Want B's failure to abort all -> **don't swallow**; rethrow or let it propagate.
3. `noRollbackFor = IllegalArgumentException.class` on `B.risky` -> B does not mark the tx (only if the exception really is benign to DB state).
4. Never rely on "catch inside the outer method" to cancel a REQUIRED inner rollback.

Note the JPA twist: after a `PersistenceException` from Hibernate the `EntityManager`/session is in an inconsistent state and Hibernate/JPA itself marks the tx rollback-only, regardless of Spring. Catching it and continuing is not valid.

## 3.7 Self-invocation trace
```java
@Service class UserService {
  public void register(User u) { saveUser(u); }        // this.saveUser(...)
  @Transactional(propagation = REQUIRES_NEW) public void saveUser(User u) { repo.save(u); }
}
```
```
controller -> [CGLIB proxy].register()   -> TransactionAttributeSource: register has no annotation -> null -> plain pass-through
                                            └─ real.register() -> this.saveUser()   (this == REAL object, not proxy)
                                                                  no interceptor, no REQUIRES_NEW, likely no tx at all
                                                                  repo.save() itself is @Transactional (SimpleJpaRepository) -> its own tx per call
```
(Spring Data repositories carry `@Transactional` on `SimpleJpaRepository`, so writes "seem to work" even in a broken design, but each call is its own tx -> silently non-atomic.)

---
# 4. Rules: rollback, isolation, readOnly, timeout

## 4.1 rollbackFor / noRollbackFor
- Default: `RuntimeException` and `Error` roll back. Checked `Exception` (including your `class PaymentException extends Exception`) **commits**.
  Historical reason: EJB semantics - checked = "business exception, caller may recover".
- `rollbackFor = Exception.class` fixes; team-wide, many put it on a meta-annotation:
```java
@Target({METHOD, TYPE}) @Retention(RUNTIME)
@Transactional(rollbackFor = Exception.class)
public @interface Tx {}
```
- `noRollbackFor` = commit despite a runtime exception, e.g. `noRollbackFor = DuplicateKeyLikeBusinessEx.class`.
- Combination: rules are matched by **inheritance depth**; the rule whose class is *closest* to the thrown exception wins. `rollbackFor=Exception.class, noRollbackFor=MyBizException.class` + throw `MyBizException` -> no rollback (closer). If no rule matches -> default rule.
- Also `rollbackForClassName` / `noRollbackForClassName` (substring match on class name; Spring 6.2 extends to patterns - verify your version).
- `jakarta.transaction.Transactional` uses `rollbackOn` / `dontRollbackOn`; semantics of default are the same (runtime rolls back, checked doesn't).
- **Where does the exception have to be visible?** The interceptor only sees exceptions that *escape* your method. Caught inside -> commit (unless something else marked rollback-only).
- The *cause chain is not inspected*: `throw new RuntimeException(checkedEx)` rolls back; `throw new Exception(runtimeEx)` commits.
- Rollback rules apply only to the scope that **owns the rule**; in participation the inner failure marks global rollback-only if *its* rules say rollback (see 3.6).
- Programmatic: `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` (works only inside a transactional method, with the proxy in the chain) or `status.setRollbackOnly()` in `TransactionTemplate`.

## 4.2 Isolation

Spring's `Isolation` enum: `DEFAULT`, `READ_UNCOMMITTED`, `READ_COMMITTED`, `REPEATABLE_READ`, `SERIALIZABLE`. `DEFAULT` = "use the database's default" (no `setTransactionIsolation` call).
Spring only calls `Connection.setTransactionIsolation` when creating the physical tx (in `doBegin`) and restores the previous level afterwards. **On join (REQUIRED participating, NESTED) it is ignored** unless `validateExistingTransaction=true`.

Anomalies: dirty read, non-repeatable read, phantom, **lost update**, **write skew**. The SQL standard table is the *minimum*; real engines differ:

| | MySQL InnoDB | PostgreSQL | Oracle |
|---|---|---|---|
| Default | REPEATABLE READ | READ COMMITTED | READ COMMITTED |
| Mechanism | MVCC via undo log; **snapshot fixed at the first consistent read** in the tx (not at `BEGIN`) | MVCC via tuple versions (xmin/xmax), no undo; snapshot **per statement** at RC, **per tx** at RR | MVCC via undo/rollback segments, statement-level read consistency at RC |
| READ_UNCOMMITTED | truly reads uncommitted | **behaves as READ COMMITTED** | not supported |
| READ_COMMITTED | fresh snapshot per statement; gap locks mostly off | default | default |
| REPEATABLE_READ | default. Plain `SELECT` = snapshot. `SELECT ... FOR UPDATE/UPDATE/DELETE` read **latest committed** (current read) + **next-key/gap locks** (blocks inserts in ranges, prevents phantoms for locking reads) | = snapshot isolation: no dirty/non-repeatable/**phantom**. Concurrent update of the same row -> `ERROR 40001 could not serialize access due to concurrent update`: **you must retry** | **not available** (Spring/JDBC request -> driver error) |
| SERIALIZABLE | plain SELECT becomes `LOCK IN SHARE MODE`: readers block writers -> lock waits, deadlocks | SSI: optimistic; aborts with 40001 "could not serialize access due to read/write dependencies" -> **retry** | snapshot isolation, `ORA-08177 can't serialize access` on conflict -> retry |
| Readers block writers? | no (snapshot) except SERIALIZABLE | no | no (`ORA-01555 snapshot too old` if undo is too small) |

Things seniors say:
- Higher isolation does not fix **lost update** by magic in MySQL RR: `SELECT balance` (snapshot), compute in Java, `UPDATE ... SET balance=?` -> the write is based on stale data and overwrites a concurrent commit. Fix with **atomic SQL** (`UPDATE acc SET bal = bal - :x WHERE id=:id AND bal >= :x`, check rows affected), **optimistic `@Version`**, or **pessimistic** `SELECT ... FOR UPDATE` (`@Lock(PESSIMISTIC_WRITE)`).
- In PostgreSQL RC the same read-modify-write is also lost-update prone. In PG RR it fails loudly with 40001 instead (good, but need retry).
- Write skew (two doctors on call) survives snapshot isolation (PG RR, Oracle SERIALIZABLE, MySQL RR); only PG SERIALIZABLE (SSI) or explicit locks/constraints stop it.
- Isolation is a property of the **physical** tx/connection: `REQUIRES_NEW` with a different level really gets it; `REQUIRED` joining does not.
- JPA: `JpaTransactionManager` with plain `DefaultJpaDialect` throws `InvalidIsolationLevelException` ("Standard JPA does not support custom isolation levels - use a special JpaDialect for your JPA implementation"). Boot's `HibernateJpaDialect` supports it.
- Hibernate first-level cache masks isolation: in one persistence context `em.find(id)` twice returns the same entity **without hitting the DB** - you see "repeatable read" behaviour even at READ_COMMITTED (and stale data).

## 4.3 readOnly

`@Transactional(readOnly = true)` is a **hint plus a few concrete effects**, not a security guarantee:

| Layer | What happens |
|---|---|
| Spring TSM | `isCurrentTransactionReadOnly()` = true (for routing/AOP checks) |
| Hibernate (`HibernateJpaDialect`) | session default read-only, **`FlushMode.MANUAL`**: no automatic dirty checking flush at commit or before queries. Entities loaded are not snapshotted (less memory/CPU). Changes made to managed entities are **silently discarded** |
| JDBC driver | `Connection.setReadOnly(true)` - driver-dependent. PostgreSQL JDBC: sets the tx read-only (`SET SESSION CHARACTERISTICS ... READ ONLY` semantics; writes fail with "cannot execute INSERT in a read-only transaction"). MySQL Connector/J: can send `START TRANSACTION READ ONLY`. Oracle: `SET TRANSACTION READ ONLY` = transaction-level read consistency (snapshot for the whole tx!). H2/others: often ignored |
| `DataSourceTransactionManager.enforceReadOnly` | default false; when true Spring explicitly executes `SET TRANSACTION READ ONLY` |
| Replica routing | Not automatic. You implement `AbstractRoutingDataSource.determineCurrentLookupKey()` using `TransactionSynchronizationManager.isCurrentTransactionReadOnly()` |

Replica-routing gotcha: `DataSourceTransactionManager.doBegin` fetches the connection **before** the read-only flag is published to the TSM thread state, so a naive routing DataSource always sees `false`. Solution: wrap with `LazyConnectionDataSourceProxy` (connection is obtained lazily at first statement, after the flag is set) - the standard recipe. Also beware replica lag: read-your-writes after a write tx on the primary may hit a stale replica.

readOnly pitfalls:
- **Inner write in a readOnly outer tx**: `@Transactional(readOnly=true) list()` calls `svc.update()` (REQUIRED, not readOnly) -> joins, inherits the *outer* readOnly (inner flag ignored) -> with Hibernate, flush MANUAL -> **update silently lost** (or DB error on PG).
- class-level `@Transactional(readOnly = true)` + method `@Transactional` without readOnly is the right way to override.
- `readOnly` does not prevent `JdbcTemplate.update` on drivers that ignore the hint (MySQL w/o support, H2).
- A read-only tx still holds a connection and (Oracle) a consistent snapshot; not free.

## 4.4 timeout
`@Transactional(timeout = 5)` seconds. Semantics:
- Applies only when Spring **creates** the tx (REQUIRED-new / REQUIRES_NEW / NESTED-new). Ignored on join.
- Spring stores a **deadline** on the `ResourceHolder`. It is checked **when Spring-managed code touches the holder** (`ConnectionHolder.getTimeToLiveInSeconds` while creating statements via `DataSourceUtils.applyTimeout` -> `Statement.setQueryTimeout(remaining)`), which throws `TransactionTimedOutException` once past.
- Consequence: it is **not a watchdog thread**. A method that computes 60 s in Java and then does no DB work never times out; a `Thread.sleep` inside the tx is not interrupted. It bounds *DB statements after the deadline*, and Hibernate applies the remaining time as query timeout via the dialect (verify per version).
- For real bounds combine: `timeout`, JDBC `socketTimeout`/`connectTimeout`, DB `statement_timeout`/`lock_wait_timeout`/`innodb_lock_wait_timeout`, Hikari `connectionTimeout`, and proper client timeouts on remote calls.

---
# 5. Proxy pitfalls and fixes

## 5.1 Matrix: when does `@Transactional` silently do nothing?

| # | Cause | Why | Fix |
|---|---|---|---|
| 1 | **Self-invocation** `this.m()` | `this` is the target, not the proxy | Move to another bean; `TransactionTemplate`; inject self (`@Lazy`); AspectJ mode |
| 2 | `private` method | not visible to proxy / attribute source never matches | Make it non-private (public best); or extract bean |
| 3 | `protected`/package-private | Spring <=5: ignored. Spring 6+: OK with CGLIB only | Use public to be safe |
| 4 | `final` method (CGLIB) | CGLIB cannot override -> **no advice, no error**; final *class* -> startup error (Boot: cannot create CGLIB proxy) | Remove `final` (Kotlin: `kotlin-spring` plugin opens classes) |
| 5 | `static` method | not an instance call | Refactor |
| 6 | JDK proxy + injecting by class | Bean is `$Proxy12` implementing interfaces; `BeanNotOfRequiredTypeException` at startup | Inject by interface or `proxyTargetClass=true` |
| 7 | Annotation on interface + CGLIB | May be ignored depending on lookup; not recommended | Annotate class |
| 8 | Bean not managed (`new`) | no proxy | Let Spring create it |
| 9 | Call from `@PostConstruct` / constructor | proxy not yet applied | `ApplicationReadyEvent`, `TransactionTemplate` |
| 10 | Exception swallowed / checked exception | interceptor sees no rollback trigger | Rethrow; `rollbackFor` |
| 11 | Other thread (`@Async`, executor, parallel stream) | ThreadLocal | Own tx in async method / pass ids, not entities |
| 12 | Wrong/absent `PlatformTransactionManager` for the DataSource | tx begins on a different resource | `@Transactional("name")` |
| 13 | Multiple `@EnableTransactionManagement` / `@Transactional` on a `@Configuration`-created bean too early | not proxied | check `BeanPostProcessor` warnings |
| 14 | Catch + continue after inner REQUIRED failure | rollback-only -> `UnexpectedRollbackException` | Section 3.6 |

## 5.2 JDK vs CGLIB
```
JDK dynamic proxy                          CGLIB subclass proxy
- implements target's interfaces           - generated subclass of target class (Objenesis: no ctor call)
- only interface methods are advised       - public/protected/package (Spring6) non-final methods advised
- cannot be cast to the class              - class must be non-final; needs visible (non-private) methods
- Spring default when interfaces exist     - Boot default (spring.aop.proxy-target-class=true)
  and proxyTargetClass=false               - fields on the proxy are NOT the target's fields:
                                             accessing `proxy.field` reads the (empty) proxy instance -> use getters
```
Classic CGLIB trap: `service.publicField` or a `protected` field access from another class reads the proxy's own uninitialised field, not the target's. Also a `final` method invoked on the proxy runs on the **proxy instance** (with null fields) because it is not delegated - typically NPE.

## 5.3 Fixes for self-invocation, ranked

1. **Extract to another bean** (best design; also makes the tx boundary visible).
2. **`TransactionTemplate`** inside the same class - explicit boundary, no proxy dependency:
```java
@Service
class UserService {
    private final TransactionTemplate tx;
    UserService(PlatformTransactionManager tm) {
        this.tx = new TransactionTemplate(tm);
        this.tx.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
    }
    public void register(User u) {
        tx.executeWithoutResult(status -> repo.save(u));   // Spring 5.2+
    }
}
```
3. **Self injection** - works but smells; Boot 2.6+ forbids circular references by default so use `@Lazy`:
```java
@Autowired @Lazy private UserService self;
public void register(User u) { self.saveUser(u); }
```
4. `AopContext.currentProxy()` with `@EnableAspectJAutoProxy(exposeProxy = true)` - couples code to Spring AOP; avoid.
5. **AspectJ mode**: `@EnableTransactionManagement(mode = AdviceMode.ASPECTJ)` + `spring-aspects` + compile-time (`aspectj-maven-plugin`) or load-time weaving (`-javaagent:spring-instrument`/aspectjweaver). Advice is woven into the bytecode of the class itself: **self-invocation works, private methods work in practice, works on non-Spring-managed objects** (e.g. entities woven with `@Configurable`). Cost: build complexity. Annotate the *class/method*, interface annotations are not honoured in AspectJ mode.

## 5.4 Programmatic transactions: `TransactionTemplate`
```java
TransactionTemplate tt = new TransactionTemplate(transactionManager);   // implements TransactionOperations
tt.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
tt.setTimeout(3);
tt.setReadOnly(false);
Long id = tt.execute(status -> {                 // TransactionCallback<T>
    Order o = repo.save(new Order());
    if (o.isSuspicious()) status.setRollbackOnly();   // rollback without exception
    return o.getId();
});
tt.executeWithoutResult(status -> repo.flush());       // TransactionCallbackWithoutResult
```
Semantics: `execute` -> `tm.getTransaction(this)`; callback `RuntimeException`/`Error` -> rollback (same default rule as annotation: `rollbackOnException`); checked exceptions can't escape the lambda (wrap them); `status.isRollbackOnly()` -> rollback. Thread-safe once configured. Use it for: self-invocation, tx boundary around only part of a method (do the HTTP call *outside*), loops with per-item tx, and **so the commit happens before you publish events/messages**:

```java
Order saved = tt.execute(s -> repo.save(order));      // tx fully committed here
kafka.send("orders", saved.getId(), payload);          // after commit (still: not atomic, see 6.3)
```
`TransactionOperations.withoutTransaction()` exists for tests/optional-tx wiring.

---
# 6. Events, messaging, long transactions, OSIV

## 6.1 `@TransactionalEventListener`
```java
@Service class OrderService {
  @Transactional public void place(Order o) { repo.save(o); publisher.publishEvent(new OrderPlaced(o.getId())); }
}
@Component class Mailer {
  @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
  void on(OrderPlaced e) { mail.send(e.orderId()); }
}
```
Mechanism: `TransactionalEventListenerFactory` creates `TransactionalApplicationListenerMethodAdapter`. When the event is published inside an active synchronization, the adapter **registers a `TransactionSynchronization`** and defers the listener invocation to the chosen phase.

| Phase | When | Notes |
|---|---|---|
| `BEFORE_COMMIT` | just before commit, tx still open | can still write; exception -> rollback |
| `AFTER_COMMIT` (default) | after successful commit | **tx already committed; resources still bound to the thread**; exceptions are logged, do NOT affect the caller |
| `AFTER_ROLLBACK` | after rollback | cleanup/compensation |
| `AFTER_COMPLETION` | after either | |

Pitfalls:
1. **No transaction active -> the event is silently dropped** unless `fallbackExecution = true`. (Classic: publisher method lacks `@Transactional`, listener never runs.)
2. **AFTER_COMMIT + `@Transactional` REQUIRED in the listener = writes lost.** In `afterCommit` the old connection/EntityManager is still bound and its tx is logically finished: the listener's REQUIRED scope *joins* that dead physical tx, nothing commits it again -> data never persisted, no error. Use `@Transactional(propagation = REQUIRES_NEW)` (or `NOT_SUPPORTED`). Recent Spring versions (6.1+, as I recall) even validate this at startup and reject other propagation modes on such listeners - verify for your version.
3. Listener runs **synchronously on the publisher's thread** unless you add `@Async` (then it also has no tx and no ambient anything).
4. Exceptions in AFTER_COMMIT are swallowed (logged at error) -> caller thinks success. A crash between commit and listener = **event lost** -> not reliable delivery (see outbox).
5. Ordering across multiple listeners: `@Order`. Works with any ApplicationEvent/POJO event.
6. Entity state in AFTER_COMMIT: lazily loading from a detached/closed session may fail; pass ids/DTOs in events.
7. Spring Data domain events (`AbstractAggregateRoot`, `@DomainEvents`) publish on `save()` -> combine with the phase rules above.

## 6.2 Programmatic hook without an event
```java
TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
    @Override public void afterCommit() { cache.evict(key); }
});
```
Requires `TransactionSynchronizationManager.isSynchronizationActive()` else `IllegalStateException("Transaction synchronization is not active")`.

## 6.3 Transactions + messaging (Kafka / RabbitMQ / SNS)

The **dual write problem**: a DB and a broker are two resources; a local `@Transactional` covers only the DB.

```
Publish INSIDE tx (dangerous)                       Publish AFTER commit (also lossy)
 begin                                               begin
 insert order                                        insert order
 kafka.send(OrderCreated)  ── message is OUT         commit                       ── crash here
 ... later exception / rollback / commit fails       kafka.send(OrderCreated)     ── message never sent
 rollback                                            → DB has order, consumers never hear of it
 → consumers process an order that doesn't exist    (or: broker down → send fails → tx already committed)
```
Failure modes when sending inside the tx: (a) rollback after send = **phantom event**, (b) consumer receives event **before the commit** and queries the DB -> row not found (race), (c) slow broker blocks the tx and holds a pooled connection, (d) `KafkaTemplate.send` is async: returns success before the broker ack; failure surfaces later, nobody rolls anything back.

The correct pattern: **Transactional Outbox**
```
tx:  insert orders(...)
     insert outbox(id, aggregate_id, type, payload, created_at, published=false)     ← SAME local tx, atomic
commit
relay (separate thread/process):  poll `outbox WHERE published=false ORDER BY id` (SELECT ... FOR UPDATE SKIP LOCKED)
                                  OR CDC (Debezium reads the binlog/WAL) → publishes to Kafka → mark published / delete
consumer: at-least-once ⇒ must be idempotent (dedupe on event id / inbox table)
```
- Guarantees at-least-once + ordering per aggregate (key by aggregate id). Exactly-once is achieved by **idempotent consumers**, not by the pipe.
- `@TransactionalEventListener(AFTER_COMMIT)` + broker send is the "poor man's" version: fine for non-critical notifications, wrong for money/inventory.
- Kafka transactions (`transactional.id`, `KafkaTransactionManager`) make *producer writes + consumer offsets* atomic in Kafka (exactly-once *within Kafka*). They do **not** include your JDBC tx. `ChainedTransactionManager` = best-effort 1PC ordering (commit B then A) with a failure window; it is deprecated in Spring Data. Prefer outbox.
- RabbitMQ: `channelTransacted` / publisher confirms similarly local to the broker.

## 6.4 Long transactions and connection-pool exhaustion

HikariCP defaults: `maximumPoolSize=10`, `connectionTimeout=30000` ms. Failure: 
`HikariPool-1 - Connection is not available, request timed out after 30000ms` (`SQLTransientConnectionException`), Boot: `CannotCreateTransactionException: Could not open JPA EntityManager for transaction`.

Little's law: needed connections ≈ requests/s x average *transaction* time. A tx that includes a 2 s remote call at 50 rps needs ~100 connections - you'll exhaust 10 quickly.

Anti-patterns inside `@Transactional`:
- **Remote HTTP/gRPC calls** (payment gateway, KYC): connection idle but checked out; also rollback can't undo the remote side effect.
- Sending email/SMS/Kafka; file IO; `Thread.sleep`; waiting on user; big loops with per-row `flush`.
- Reading large result sets fully; "fat" service methods that orchestrate everything.
- `REQUIRES_NEW` inside a loop / nested per request (2 connections per thread).
- Locks held across the call: row locks (`SELECT FOR UPDATE`) stay until commit -> other requests queue -> pool drains.

Pattern to fix: **shrink the tx** to only DB work:
```java
public Result process(Cmd c) {                         // NOT @Transactional
    Quote q = pricingClient.quote(c);                  // remote, no tx, no connection
    Order o = tt.execute(s -> orderRepo.save(build(c, q)));    // short tx
    notifier.publishAfterCommit(o);                    // outside
    return map(o);
}
```
Observability: Hikari `leakDetectionThreshold` (ms) logs a stack trace of who holds a connection too long; metrics `hikaricp_connections_active/pending/timeout_total`; DB side `pg_stat_activity` (`state='idle in transaction'` is the smoking gun), MySQL `information_schema.innodb_trx`. Set PG `idle_in_transaction_session_timeout`.

## 6.5 Lazy loading, `LazyInitializationException`, and OSIV

- Entities are managed only while the persistence context (EntityManager) is open. Outside the tx (no OSIV), touching a lazy association -> `LazyInitializationException: could not initialize proxy - no Session`.
- **OSIV** (Open Session In View, `spring.jpa.open-in-view`, **default true in Boot**, Boot logs a warning at startup) registers `OpenEntityManagerInViewInterceptor` binding an `EntityManager` to the request thread for the *whole* request including view rendering/JSON serialization. Then:
  - lazy loading works in controllers/serializers (the hidden N+1 factory),
  - lazy loads *outside* a tx run in autocommit mode each statement; a connection can be acquired and kept until the request ends, so slow clients/serialization pin pool connections,
  - `JpaTransactionManager` **reuses** the request's EM: the persistence context is shared across several transactions in a request; first-level cache may show stale state; a failed tx leaves entities in an odd state.
  - Only covers the servlet thread; `@Async`/other threads have no OSIV.
- Recommended for APIs: `spring.jpa.open-in-view=false`, fetch what you need inside the service: `JOIN FETCH`, `@EntityGraph`, DTO projections, `@BatchSize`; map to DTOs **inside** the tx boundary. Never return entities from `@Transactional` methods to the web layer.
- `spring.jpa.properties.hibernate.enable_lazy_load_no_trans=true` is an anti-pattern (each lazy load opens a temporary session/connection).

## 6.6 JPA flush behaviour that surprises people
- `save()` on a new entity with `GenerationType.IDENTITY` inserts immediately; with SEQUENCE/AUTO SQL is **deferred until flush** (commit, or before a JPQL query touching the table with `FlushMode.AUTO`).
- Constraint violations therefore surface at **commit** (in the interceptor), possibly as `DataIntegrityViolationException` thrown from the proxy, not from `repo.save()`. To catch inside the method call `repo.saveAndFlush()`/`em.flush()` (but then the tx is likely rollback-only anyway).
- Dirty checking: modify a managed entity and don't call `save` -> still updated at flush (unless `readOnly`).

---
# 7. Retries, deadlocks, locking, distributed tx, testing

## 7.1 Transient failures and retry

Exceptions Spring surfaces (all under `org.springframework.dao`):
| Exception | Typical cause | Retry? |
|---|---|---|
| `DeadlockLoserDataAccessException` (subtype of `PessimisticLockingFailureException`) | DB chose you as deadlock victim (MySQL 1213, PG 40P01) | yes, whole tx |
| `CannotAcquireLockException` | lock wait timeout (MySQL 1205 / `innodb_lock_wait_timeout`), PG `lock_timeout` | maybe (with backoff) |
| `CannotSerializeTransactionException` | PG 40001, Oracle ORA-08177 | yes |
| `OptimisticLockingFailureException` (`ObjectOptimisticLockingFailureException` for JPA `@Version`) | stale version | yes, **re-read** and recompute |
| `TransientDataAccessException` family | generic transient | case by case |
| `DuplicateKeyException` / `DataIntegrityViolationException` | constraint | usually not (business error), except upsert races |

**Rule: the retry must wrap the transaction, never sit inside it.** A tx that has hit a deadlock/serialization failure is already rolled back or rollback-only at DB level; retrying *inside* the same tx just fails again or hits `current transaction is aborted, commands ignored until end of transaction block` (PostgreSQL).

Order of advice:
```
call ─► [Retry advice (outer, @Order lower value)] ─► [Transaction advice (inner)] ─► your method
          attempt 1: new tx → fails → tx rolled back, connection released
          backoff
          attempt 2: NEW tx, fresh snapshot / fresh @Version read
```
With Spring Retry (`spring-retry` + `@EnableRetry`):
```java
@Service class TransferFacade {                    // NOT @Transactional
  @Retryable(retryFor = { OptimisticLockingFailureException.class, DeadlockLoserDataAccessException.class },
             maxAttempts = 3, backoff = @Backoff(delay = 50, multiplier = 2, random = true))
  public void transfer(long from, long to, BigDecimal amt) { transferTx.doTransfer(from, to, amt); }
}
@Service class TransferTx {
  @Transactional public void doTransfer(long from, long to, BigDecimal amt) { ... }
}
```
- Safest design: **retry on a separate facade bean whose method is not transactional**, so the ordering question cannot bite. (`retryFor` is the Spring Retry 2.0 name; older versions used `value`/`include` - verify.)
- If both annotations are on the same bean/method, ordering is decided by advisor `order`: `TransactionInterceptor` order is `@EnableTransactionManagement(order)` (default `LOWEST_PRECEDENCE`); the Spring Retry advisor must have a **smaller** order value (outer). Spring Retry's own default puts retry slightly outside the tx advisor, but do not rely on defaults you have not verified: set `@EnableTransactionManagement(order = Ordered.LOWEST_PRECEDENCE)` and configure retry's order (`@EnableRetry(order = ...)` in recent spring-retry, or your own `RetryOperationsInterceptor` advisor) to a lower value; then add a **test** that asserts each attempt opens a new tx (log "Creating new transaction" appears N times).
- Alternative w/o annotations, explicit and testable:
```java
retryTemplate.execute(ctx -> tt.execute(status -> { ... return null; }));
```
- Retried code must be **idempotent** aside from the DB (no emails/HTTP inside).
- Deadlock avoidance: access rows in a **consistent order** (`ORDER BY id` when locking multiple rows, sort ids before updating), keep tx short, index the WHERE columns (MySQL gap locks on unindexed scans lock whole ranges), use `SKIP LOCKED` for queue tables.
- Deadlock **visibility**: MySQL `SHOW ENGINE INNODB STATUS` -> "LATEST DETECTED DEADLOCK"; PG log with `log_lock_waits=on`, `deadlock_timeout`.

## 7.2 Optimistic vs pessimistic
```java
@Entity class Account { @Id Long id; BigDecimal balance; @Version long version; }
// UPDATE account SET balance=?, version=? WHERE id=? AND version=?   → 0 rows ⇒ ObjectOptimisticLockingFailureException at flush/commit
```
- Optimistic: no locks held, conflict detected at write, cheap when contention is low. The exception typically surfaces **at commit**, i.e. from the proxy, after your method returned -> the retry must be outside (7.1). Version check is at **flush**; `@Version` doesn't help with `JdbcTemplate` updates unless you write the predicate.
- Pessimistic: `@Lock(LockModeType.PESSIMISTIC_WRITE)` -> `SELECT ... FOR UPDATE`; use with lock timeout hint (`jakarta.persistence.lock.timeout`), keep the tx tiny, consistent order. Use for hot rows (inventory, sequences) where retries would storm.
- Atomic update SQL is often the best answer to "concurrent balance updates".
- Idempotency keys (unique constraint) turn retry storms into safe no-ops.

## 7.3 Distributed transactions

**2PC / XA**: a coordinator asks all resource managers `prepare`; if all vote yes -> `commit` else `rollback`. `JtaTransactionManager` + XA DataSources / JMS (Narayana is the supported embedded option in Boot 3; Boot 3 dropped Atomikos/Bitronix auto-config).
Costs: blocking (in-doubt transactions if coordinator dies between phases, locks held), latency (two round trips + forced log writes), not supported by Kafka/most HTTP services, availability coupling. Rarely chosen for microservices; still valid for two DBs / DB+JMS in a monolith.

**Saga**: sequence of local transactions with **compensating actions**; eventual consistency.
- Choreography: services react to events (simple, hard to follow at scale).
- Orchestration: a coordinator (Temporal, Camunda, custom state machine) issues commands.
- Requirements: each step **idempotent**, compensations idempotent and commutative-safe, **outbox** for reliable events, semantic locks/status fields (`PENDING`), timeouts, dead-letter handling, explicit "pivot" step (after which you only move forward).
- No isolation between steps -> handle dirty reads at the business level (reservations, versioned state).

Decision: single DB -> plain ACID. Two DBs same process and must be atomic -> XA (or redesign). Across services -> saga + outbox + idempotency.

## 7.4 Testing transactions

`@Transactional` on a **test** (Spring TestContext `TransactionalTestExecutionListener`) starts a test-managed tx around each test method and **rolls it back** at the end by default. Consequences (bugs hidden):
1. **No commit ever happens** -> commit-time failures don't show: deferred constraints, unique violations that would fire at flush, `@Version` conflicts, DDL/trigger behaviour. Flush only occurs on queries; add `entityManager.flush()` + `clear()` in the test to force SQL and to reload from DB.
2. `@TransactionalEventListener(AFTER_COMMIT)` **never fires** (the tx is rolled back) - the code path looks untested. Use `@Commit` / `TestTransaction.flagForCommit(); TestTransaction.end();`.
3. **Lazy loading works** because the session stays open across the test - masks `LazyInitializationException` that production (with OSIV off) would throw.
4. `REQUIRES_NEW` code **commits for real** (separate physical tx) and is **not rolled back** by the test -> data pollution between tests and order-dependent failures.
5. Everything runs on the test thread: a full server (`webEnvironment = RANDOM_PORT`) handles requests on **other threads**, so the test tx doesn't cover them and won't roll their changes back; `MockMvc` runs in the same thread and does share the tx.
6. `@Async`, executors: separate threads, no test tx.
7. First-level cache: `repo.save(x); repo.findById(id)` returns the same instance without SQL, hiding mapping errors.
Rules: `@Commit`/`@Rollback(false)` when verifying commit semantics; `@BeforeTransaction/@AfterTransaction`; prefer Testcontainers with the real DB (H2 differs on isolation, locks, `readOnly`, SQL dialect); assert the *DB state via a separate connection/JdbcTemplate after `TestTransaction.end()`*; test propagation with log assertions or `TransactionSynchronizationManager.isActualTransactionActive()`/`getCurrentTransactionName()`.

Detecting a tx in a test (or production sanity check):
```java
assertThat(TransactionSynchronizationManager.isActualTransactionActive()).isTrue();
assertThat(TransactionSynchronizationManager.getCurrentTransactionName()).contains("OrderService.place");
```

---
# 8. Failure modes and production incident stories

Format: symptom -> diagnosis -> fix.

**Incident 1 - "Money debited, no transfer record" (checked exception commits).**
Symptom: support tickets: account debited, no transfer row, no error to user. Diagnosis: `transfer()` `throws InsufficientFundsException extends Exception`; default rules commit; debit had been flushed before exception. Fix: `rollbackFor = Exception.class` (team meta-annotation `@Tx`), plus an architecture test (ArchUnit) forbidding bare `@Transactional`. Lesson: read the SQL log: you see `COMMIT`, not `ROLLBACK`.

**Incident 2 - "UnexpectedRollbackException after we added a try/catch".**
Symptom: batch job logs `UnexpectedRollbackException: Transaction silently rolled back because it has been marked as rollback-only`; some rows missing. Diagnosis: outer `@Transactional` loop caught exceptions from an inner REQUIRED service to continue with the next record; first failure poisoned the whole tx. Fix: per-record `REQUIRES_NEW` (or `TransactionTemplate` per item / NESTED with JDBC), one tx per item, outer method non-transactional. Trace confirmation via DEBUG log "Participating transaction failed - marking existing transaction as rollback-only".

**Incident 3 - Pool deadlock from `REQUIRES_NEW`.**
Symptom: traffic spike -> all requests hang -> after 30 s `Connection is not available, request timed out after 30000ms`; threads dump shows all threads in `HikariPool.getConnection` called from `AuditService.log`. Diagnosis: pool size 10, 10 concurrent requests each hold connection A (outer) and need B for `REQUIRES_NEW`; nobody can proceed. Fix: (a) pool >= max concurrent threads x 2 (fragile), (b) better: call audit **after** outer commit (`@TransactionalEventListener` + REQUIRES_NEW, or outbox), (c) `leakDetectionThreshold`. Rule: every `REQUIRES_NEW` inside another tx increases required connections per thread by 1.

**Incident 4 - Remote call inside tx exhausts connections.**
Symptom: latency of the payment gateway rises from 200 ms to 5 s; entire service (even health/`/orders`) times out; DB CPU idle. Diagnosis: `@Transactional placeOrder()` called the gateway; connections held `idle in transaction`. Fix: move call outside tx, timeouts on HTTP client, bulkhead/circuit breaker, `idle_in_transaction_session_timeout`, alert on Hikari pending threads.

**Incident 5 - "@Transactional does nothing" (self-invocation).**
Symptom: partial data after failure; unit tests pass with mocks. Diagnosis: `processAll()` (no annotation) calls `this.processOne()` (`@Transactional REQUIRES_NEW`) - no proxy, no tx per item; each `repo.save` ran in its own tx. Fix: extract `ItemProcessor` bean. Detection: log `org.springframework.transaction.interceptor` at TRACE - no "Getting transaction for [..processOne]" lines.

**Incident 6 - Silent data loss with `readOnly=true`.**
Symptom: "status not updated" only when called from the list endpoint. Diagnosis: class-level `@Transactional(readOnly = true)` on a service; a write method forgot to override, so joined a read-only tx; Hibernate `FlushMode.MANUAL` discarded changes. Fix: override at method level `@Transactional` (readOnly=false); ArchUnit rule / integration test asserting writes commit; on PG the failure would have been loud - test on the real DB.

**Incident 7 - AFTER_COMMIT listener writes vanish.**
Symptom: "welcome-bonus" row missing though listener logs "granted". Diagnosis: listener annotated `@Transactional` (REQUIRED) joined the already-committed tx. Fix `REQUIRES_NEW`. Also add integration test with `@Commit`.

**Incident 8 - `@Async` method sees no data / `LazyInitializationException`.**
Symptom: async e-mail job throws `EntityNotFoundException`/lazy exception; sometimes works (race). Diagnosis: async ran before the caller's tx committed (READ_COMMITTED can't see) and detached entity passed across threads. Fix: publish event `AFTER_COMMIT`, pass ids, load inside the async method's own `@Transactional`.

**Incident 9 - Phantom Kafka event.**
Symptom: downstream shows orders that don't exist in orders DB. Diagnosis: `kafkaTemplate.send` inside tx, later flush failed (constraint) -> rollback. Fix: outbox; consumers idempotent; never trust "sent inside tx".

**Incident 10 - Deadlocks under load after adding a batch.**
Symptom: `Deadlock found when trying to get lock; try restarting transaction` (MySQL 1213) ~1% of requests. Diagnosis: two flows update `accounts` rows (A then B) vs (B then A). `SHOW ENGINE INNODB STATUS` shows both. Fix: sort by id before locking; retry facade (7.1) with backoff; shorter tx; covering index to avoid gap locks.

**Incident 11 - Retry that never helped.**
Symptom: `@Retryable` on a `@Transactional` method retried 3x but always hit "current transaction is aborted..." Diagnosis: advisor order: retry executed **inside** the tx. Fix: facade bean + explicit order, log assertion of 3 tx begins.

**Incident 12 - Stale read from replica after write.**
Symptom: user updates profile, page shows old data. Diagnosis: readOnly txs routed to replica via `AbstractRoutingDataSource`; replica lag 300 ms. Fix: read-your-writes routing (sticky to primary for N seconds/after write), or read from primary for user-facing flows; and remember `LazyConnectionDataSourceProxy` or routing never engages.

**Incident 13 - Parallel stream in a tx corrupts nothing (or everything).**
Symptom: `items.parallelStream().forEach(repo::save)` inside `@Transactional` - some rows missing on failure, others committed. Diagnosis: ForkJoin threads run repo calls in their own autocommit txs; the outer tx's rollback does nothing to them; also `EntityManager` not thread-safe. Fix: sequential loop inside tx, or batch with partitioning + one tx per partition on a bounded executor.

**Incident 14 - Test suite green, prod fails.**
Symptom: unique-constraint violation only in prod; test used `@Transactional` + H2 + no flush. Fix: flush+clear in tests, Testcontainers, tests that commit.

**Incident 15 - Two DataSources, one tx manager.**
Symptom: `@Transactional` writes to the second DB are not rolled back. Fix: named managers `@Transactional("secondTm")` and repository config per DataSource; or XA/outbox if cross-DB atomicity is really needed.

Diagnosis toolbox:
```
logging.level.org.springframework.transaction=TRACE
logging.level.org.springframework.orm.jpa=DEBUG
logging.level.org.hibernate.engine.transaction.internal.TransactionImpl=DEBUG
logging.level.com.zaxxer.hikari=DEBUG      (pool stats)
spring.jpa.show-sql=false; use p6spy/datasource-proxy to see BEGIN/COMMIT/ROLLBACK and real SQL
```
`TransactionSynchronizationManager.getCurrentTransactionName()` / `isActualTransactionActive()` in a debugger; `AopUtils.isAopProxy(bean)`, `AopUtils.isCglibProxy(bean)`, `AopProxyUtils.ultimateTargetClass(bean)` to check what you injected.

---
# 9. Interview questions

Legend: **E** easy, **M** medium, **H** hard. Each has a model answer, then "Interviewer then asks...".

### Q1 (E) What does `@Transactional` do and how?
**A:** Declarative transaction demarcation using Spring AOP. A proxy around the bean intercepts calls; `TransactionInterceptor` gets a transaction from the `PlatformTransactionManager` before the method, commits on normal return and rolls back on `RuntimeException`/`Error`.
- *Then asks:* "What kind of proxy?" -> JDK if interfaces and `proxyTargetClass=false`, CGLIB otherwise; Boot defaults to CGLIB.
- *Then asks:* "When is the proxy created?" -> `BeanPostProcessor` (`InfrastructureAdvisorAutoProxyCreator`) after initialization.

### Q2 (E) Default rollback behaviour?
**A:** Unchecked exceptions and Errors roll back; checked exceptions commit. Override with `rollbackFor`/`noRollbackFor`.
- *Then:* "Why?" -> EJB heritage: checked = recoverable business condition.
- *Then:* "`throw new Exception(new RuntimeException())`?" -> commits (cause chain not inspected).

### Q3 (E) Which methods can be transactional?
**A:** Public methods called through the proxy from another bean. Spring 6 with CGLIB also allows protected/package-private; never private/static; final methods are silently not advised under CGLIB.

### Q4 (E) Name the propagation types, and default.
**A:** REQUIRED (default), REQUIRES_NEW, NESTED, SUPPORTS, NOT_SUPPORTED, MANDATORY, NEVER. Give one-line semantics (table in section 3).
- *Then:* "Difference between REQUIRES_NEW and NESTED?" -> REQUIRES_NEW = separate physical tx/connection, independent commit, outer suspended. NESTED = same physical tx with savepoint, inner commit is only a release, outer rollback undoes it.

### Q5 (E) Why doesn't `@Transactional` work when I call another method in the same class?
**A:** Self-invocation bypasses the proxy. Fix: another bean, `TransactionTemplate`, self-injection, AspectJ mode.
- *Then:* "Does making it public help?" -> No; visibility isn't the issue.
- *Then:* "And if the annotation is on the outer method only?" -> Then the whole call is in one tx, which may be fine; but inner `REQUIRES_NEW` won't be honoured.

### Q6 (E) What is a dirty read / non-repeatable read / phantom?
**A:** As in the table in 4.2; plus lost update and write skew, which the standard table doesn't name.

### Q7 (E) Does `readOnly=true` make it read only?
**A:** Not guaranteed. It's a hint: Hibernate flush mode MANUAL and no dirty checking; driver-dependent `setReadOnly`; usable for replica routing. On PostgreSQL writes actually fail; on others they may succeed via JDBC.

### Q8 (M) Walk me through what happens between the controller calling a `@Transactional` method and the SQL `COMMIT`.
**A:** Give the numbered trace of 2.3: CGLIB intercept -> chain -> `TransactionInterceptor` -> `invokeWithinTransaction` -> determine attribute and manager -> `getTransaction` (doGetTransaction, propagation, doBegin binds resource to TSM) -> method -> `commitTransactionAfterReturning` -> `processCommit` -> synchronization callbacks -> `doCommit` -> cleanup/resume.
- *Then:* "Where does the connection come from for JdbcTemplate?" -> `DataSourceUtils.getConnection` looks in TSM ThreadLocal.
- *Then:* "What if two DataSources?" -> two TSM entries, need two managers; `@Transactional("name")`.

### Q9 (M) How do `JdbcTemplate` and JPA share the same transaction?
**A:** `JpaTransactionManager` binds `EntityManagerHolder` for the EMF *and* a `ConnectionHolder` for the underlying DataSource (when the dialect exposes the connection and the DataSource is known). `DataSourceUtils.getConnection` therefore returns the JPA tx's connection. With a plain `DataSourceTransactionManager`, JPA would not be in the tx.
- *Then:* "Flush order problem?" -> JdbcTemplate reads don't trigger Hibernate flush: unflushed entity changes invisible to your JDBC query; call `em.flush()` first.

### Q10 (M) Explain `UnexpectedRollbackException` with an example.
**A:** 3.6. Inner REQUIRED failure marks the shared tx rollback-only; outer catches and commits; commit converts to rollback and throws. Fixes: REQUIRES_NEW/NESTED/noRollbackFor/don't swallow.
- *Then:* "Does it happen if the inner is in the same class?" -> No - no proxy, no participation logic; the exception simply propagates or is caught with no marking (but Hibernate may mark rollback-only itself on `PersistenceException`).
- *Then:* "How to detect earlier?" -> `failEarlyOnGlobalRollbackOnly`.

### Q11 (M) Isolation on an inner `@Transactional(isolation = SERIALIZABLE)` when the outer is default?
**A:** Ignored on join; isolation is set only when the physical tx starts, on the connection. Only `validateExistingTransaction=true` makes Spring throw. For real effect use REQUIRES_NEW.
- *Then:* "And REQUIRES_NEW with different isolation?" -> New connection, level applied, restored at end.

### Q12 (M) MySQL RR vs PostgreSQL RC vs Oracle: what does a repeated `SELECT` see?
**A:** MySQL RR: same snapshot (established at the first read) for plain selects; PG RC: each statement sees the latest committed; Oracle RC: statement-level; PG RR: tx snapshot, aborts with 40001 on write conflicts. Locking reads/updates in MySQL RR see the latest data.
- *Then:* "Does RR prevent lost update in MySQL?" -> No for read-compute-write; use atomic update/version/`FOR UPDATE`.
- *Then:* "Does Spring set isolation for you?" -> Only if != DEFAULT.

### Q13 (M) What happens when `REQUIRES_NEW` is used inside a transaction: threads, connections?
**A:** Outer resources are unbound from TSM and parked in a `SuspendedResourcesHolder`; a second connection is fetched; inner commits independently; outer resumed. Two connections per thread -> pool exhaustion risk; and FK/lock self-deadlock on rows the outer holds.

### Q14 (M) `@Transactional` + `@Async`?
**A:** Async runs on another thread; ThreadLocals are not inherited so the caller's tx isn't visible. If the async method itself is `@Transactional`, it starts its own tx. Consequence: race with the caller's commit; pass ids and publish after commit.
- *Then:* "Parallel streams?" -> ForkJoin common pool threads, same problem, plus non-thread-safe EntityManager.
- *Then:* "Virtual threads?" -> ThreadLocals still per thread; behaviour same; watch pinning by synchronized in old drivers.

### Q15 (M) How to make a self-invoked method transactional?
**A:** Ranked: extract bean, `TransactionTemplate`, `@Lazy` self injection, `AopContext.currentProxy()`, AspectJ mode. Explain why each works (2.x/5.3).

### Q16 (M) `@TransactionalEventListener` phases; what if no tx?
**A:** BEFORE_COMMIT, AFTER_COMMIT, AFTER_ROLLBACK, AFTER_COMPLETION. No tx -> dropped unless `fallbackExecution=true`.
- *Then:* "Can the AFTER_COMMIT listener write to the DB?" -> Only with REQUIRES_NEW; REQUIRED joins the dead tx and nothing commits.
- *Then:* "Is it reliable delivery?" -> No; process crash after commit loses it; outbox.

### Q17 (M) Why is calling an HTTP API inside `@Transactional` bad?
**A:** Holds a pooled connection and DB locks while idle; latency amplifies into pool exhaustion; can't be rolled back; on rollback the remote effect remains. Do it outside; use saga/outbox for consistency.

### Q18 (M) Optimistic vs pessimistic; where does the exception surface?
**A:** `@Version`; `ObjectOptimisticLockingFailureException` at flush/commit -> after the method returns, from the proxy; catching inside the method won't work. Retry outside the tx with re-read.

### Q19 (M) How would you implement read replica routing using `readOnly`?
**A:** `AbstractRoutingDataSource` keyed on `TransactionSynchronizationManager.isCurrentTransactionReadOnly()`, wrapped in `LazyConnectionDataSourceProxy` so the connection is chosen after the flag is set. Beware lag, read-your-writes, and that class-level readOnly with writes joins read-only.

### Q20 (M) What are the test pitfalls of `@Transactional` tests?
**A:** 7.4: rollback hides commit-time errors, AFTER_COMMIT never fires, lazy loading hides bugs, REQUIRES_NEW pollution, other-thread servers, first-level cache. Use flush/clear, `@Commit`, Testcontainers.

### Q21 (H) Explain how the advisor is registered and how Spring decides a bean needs a proxy.
**A:** 2.1/2.2. `@EnableTransactionManagement` -> selector -> `AutoProxyRegistrar` (`InfrastructureAdvisorAutoProxyCreator`) + `ProxyTransactionManagementConfiguration` (`BeanFactoryTransactionAttributeSourceAdvisor`, `AnnotationTransactionAttributeSource`, `TransactionInterceptor`). During `postProcessAfterInitialization`, `AopUtils.canApply` checks whether `getTransactionAttribute` is non-null for any method of the class (target method, then class, then interface method/class). Cached in the attribute cache.
- *Then:* "Method-level vs class-level annotation merge?" -> No merge; method replaces.
- *Then:* "What if two advisors match?" -> One proxy with a chain ordered by `Ordered`/`@Order`.
- *Then:* "Where does the proxy sit relative to @PostConstruct?" -> after.

### Q22 (H) Suspend vs join at the resource level. What exactly does `suspend()` do?
**A:** `AbstractPlatformTransactionManager.suspend`: runs `doSuspend` (unbind the resource holder, e.g. `TransactionSynchronizationManager.unbindResource(dataSource)`), snapshots and clears name/readOnly/isolation/actualActive, calls `synchronization.suspend()` on registered synchronizations and clears them (`clearSynchronization`), returns `SuspendedResourcesHolder`. `resume` does the reverse after the inner completes (`cleanupAfterCompletion`).
- *Then:* "What if `doBegin` for the inner throws?" -> `resumeAfterBeginException` restores the outer.
- *Then:* "Which hooks does `SUPPORTS`/`NOT_SUPPORTED` see?" -> synchronization is initialised in "empty" tx (SYNCHRONIZATION_ALWAYS) so `afterCompletion` callbacks still work.

### Q23 (H) How are NESTED transactions implemented, and with JPA?
**A:** JDBC savepoints via `status.createAndHoldSavepoint()` (`Connection.setSavepoint`), release on success, rollback-to-savepoint on failure. JPA/Hibernate works through the dialect's connection handle but the persistence context isn't rewound, so entity state can diverge. JTA managers don't support it by default (`nestedTransactionAllowed=false`).
- *Then:* "Does a nested failure mark the outer rollback-only?" -> No; that's REQUIRED's participation behaviour, not NESTED (as long as the exception is caught before the outer's completion).
- *Then:* "Cost?" -> savepoints are cheap on PG/MySQL but PG subtransaction overhead can hurt with >64 open per tx (subxid cache overflow).

### Q24 (H) What does Spring do exactly at commit if the tx is rollback-only?
**A:** 2.6. Local rollback-only -> rollback silently. Global rollback-only and `shouldCommitOnGlobalRollbackOnly()==false` (default) -> `processRollback(status, true)` -> rollback + `UnexpectedRollbackException` if the scope is the physical owner. JTA managers historically set `shouldCommitOnGlobalRollbackOnly` true so the transaction manager itself decides.

### Q25 (H) Design: process 10,000 records, some invalid; commit valid ones, log failures durably. Choices?
**A:** Outer non-transactional; per-record tx via `TransactionTemplate`/separate bean (`REQUIRED` at method boundary = new tx each since none exists); failures recorded with `REQUIRES_NEW` audit or after per-record catch; chunk commits (e.g. 100) for throughput with fallback to single-record on chunk failure; JDBC batching; `flush/clear` every N. Avoid one giant tx (undo growth, locks, replica lag) and avoid REQUIRED inner + swallow. Spring Batch gives chunk/skip/retry out of the box.

### Q26 (H) How do you guarantee "save order and publish event" atomically?
**A:** Outbox (6.3): same-tx insert, relay by poller (`FOR UPDATE SKIP LOCKED`) or CDC, at-least-once, idempotent consumer with inbox/dedup key. Mention why `@TransactionalEventListener` and Kafka transactions are insufficient, and why XA is unavailable for Kafka.
- *Then:* "Ordering?" -> partition by aggregate id; poller must ORDER BY id; parallel relays need per-key ordering.
- *Then:* "Outbox table growth?" -> delete/partition after publish, or CDC with delete.

### Q27 (H) Where must `@Retryable` sit relative to `@Transactional` and how to guarantee it?
**A:** Outside. Retry advice must have higher precedence (lower order number) than the tx advisor; simplest is a facade bean that is not transactional, and calls a transactional bean; or `RetryTemplate` around `TransactionTemplate`. Verify by test counting "Creating new transaction" log or a counting `TransactionSynchronization`. Reasons: after a deadlock/serialization error the tx is dead; the retry needs a fresh tx, snapshot, and re-read for optimistic locking. The retried unit must be idempotent.
- *Then:* "What if I need different order?" -> `@EnableTransactionManagement(order=...)`, `@Order` on your aspects, never leave equal orders.
- *Then:* "Which exceptions?" -> Deadlock/CannotAcquireLock/CannotSerialize/OptimisticLocking; not business exceptions.

### Q28 (H) Why did adding a `try/catch` around `repository.save()` not save my transaction from a constraint violation?
**A:** (1) with JPA, the violation appears at flush/commit, later than `save()`; (2) even if you flush, Hibernate marks the tx rollback-only after a `PersistenceException`; on PG the connection is in "aborted" state until rollback. Use pre-checks/upsert (`ON CONFLICT DO NOTHING`), or a separate REQUIRES_NEW/TransactionTemplate for the risky insert.

### Q29 (H) Explain `TransactionSynchronizationManager` and what breaks with reactive code.
**A:** ThreadLocal store (2.7). Reactive pipelines hop threads; blocking `@Transactional` semantic doesn't follow. `ReactiveTransactionManager` + `TransactionalOperator` / reactive `@Transactional` puts the tx in the Reactor `Context` (`TransactionContextManager`), so `Mono`/`Flux` chains are correct if subscribed within the operator; JDBC blocking calls inside must not be used on event-loop threads.

### Q30 (H) What does `DataSourceTransactionManager.doBegin` do w.r.t. `autoCommit`, and why does it matter for HikariCP?
**A:** `setAutoCommit(false)` is the begin; and restore at end. If pool default is autocommit=true, Spring toggles per tx (extra round trips). Setting `hikari.auto-commit=false` + `provider_disables_autocommit=true` lets Hibernate defer connection acquisition until first SQL, shortening how long connections are held.

### Q31 (H) Compare XA/2PC and saga; when do you pick which?
**A:** 7.3. XA = atomic, blocking, coordinator log, in-doubt txns, limited resource support. Saga = availability, eventual consistency, needs compensations/idempotency/outbox.
- *Then:* "Pivot transaction?" -> the step after which the saga can't be compensated; steps before are compensable, after are retriable.
- *Then:* "Compensation fails?" -> retry forever/alert/manual queue; compensations must be idempotent.

### Q32 (H) Your service intermittently returns stale data right after an update. Transaction-related causes?
**A:** replica lag with readOnly routing; long-lived persistence context (OSIV) first-level cache; MySQL RR snapshot created at first read earlier in the tx; `@Cacheable` around tx result cached before commit (cache advisor inside vs outside tx; use afterCommit eviction); async consumer reading before commit.

### Q33 (M) How do you handle two databases / two tx managers?
**A:** Separate `DataSource`, EMF, `PlatformTransactionManager` beans (one `@Primary` or qualified), `@Transactional(transactionManager = "x")`, repository packages bound to the right EMF. No atomicity across them unless XA or saga.

### Q34 (H) Timeout: does `@Transactional(timeout=2)` kill a method that takes 10 s?
**A:** Not necessarily. Deadline is checked when Spring-managed resources are used (statement creation, query timeout derived from remaining time). CPU/sleep/remote-call time isn't interrupted; the next statement after the deadline throws `TransactionTimedOutException`. Use DB/driver/client timeouts for hard limits.

### Q35 (H) Production: `Connection is not available, request timed out after 30000ms`. Your process?
**A:** (1) Is it a leak or contention? Hikari metrics: active==max, pending>0. (2) Thread dump: who holds connections (`idle in transaction`, blocked in remote call, or `REQUIRES_NEW` nesting). (3) `leakDetectionThreshold` stack traces. (4) DB view of sessions/locks (`pg_stat_activity`, `innodb_trx`). (5) Fix root cause: shrink tx, remove remote calls, fix nested tx, OSIV off, indexes for slow queries; only then resize pool (pool = DB can handle; more connections can hurt). (6) Guardrails: timeouts, circuit breaker, alerts.

## Common wrong answers (and the correction)

| Wrong answer | Correct |
|---|---|
| "`@Transactional` rolls back on any exception." | Only Runtime/Error by default. |
| "`REQUIRES_NEW` and `NESTED` are basically the same." | Different connection/commit vs savepoint in same tx. |
| "Nested = child tx that commits independently." | Nested commit is a savepoint release; outer rollback undoes it. |
| "Private methods work if the class has `@Transactional`." | Never for proxies. |
| "The inner method rolls back the transaction when it throws." | It only *marks rollback-only*; real rollback at the outer boundary. |
| "If I catch the exception in the outer method, no problem." | `UnexpectedRollbackException`. |
| "`readOnly=true` forbids writes." | Hint; may silently drop Hibernate changes or do nothing. |
| "Inner `isolation=SERIALIZABLE` upgrades the tx." | Ignored on join. |
| "`@Transactional` is thread-safe across `@Async`." | ThreadLocal; not propagated. |
| "`@TransactionalEventListener` guarantees message delivery." | It's in-memory after-commit callback; can be lost. |
| "Higher isolation fixes all concurrency bugs / SERIALIZABLE is the same everywhere." | Engines differ; lost update needs locks/versions/atomic SQL. |
| "Spring rolls back my Kafka publish." | Kafka isn't in the DB tx. |
| "Timeout kills the thread." | No, checked at resource access. |
| "Put `@Transactional` on the controller to be safe." | Keeps connection open during serialization; service layer, short. |
| "H2 tests prove transaction behaviour." | Different isolation/locking/readOnly semantics. |

---
# 10. Runnable code

## 10.1 Plain-Java simulation (compiled and run with JDK 21)
Demonstrates: proxy around a target, "begin/commit/rollback" decided by exception type, join-if-exists (REQUIRED), and the **ThreadLocal binding that is lost on another thread**.

```java
import java.lang.reflect.*;
import java.util.*;
import java.util.concurrent.*;

public class MiniTx {
    // --- mini TransactionSynchronizationManager: thread-bound resources ---
    static final ThreadLocal<Map<Object,Object>> RESOURCES = ThreadLocal.withInitial(HashMap::new);
    static int connCounter = 0;
    static final Object DS = "dataSource";

    interface OrderService { void place(boolean fail); String txState(); }

    static class OrderServiceImpl implements OrderService {
        public void place(boolean fail) {
            System.out.println("  business: conn bound on " + Thread.currentThread().getName() + " = " + RESOURCES.get().get(DS));
            if (fail) throw new IllegalStateException("boom");
        }
        public String txState() { return String.valueOf(RESOURCES.get().get(DS)); }
    }

    // --- mini TransactionInterceptor (JDK dynamic proxy, like JdkDynamicAopProxy) ---
    static OrderService proxy(OrderService target) {
        return (OrderService) Proxy.newProxyInstance(MiniTx.class.getClassLoader(),
            new Class<?>[]{OrderService.class}, (p, m, a) -> {
                boolean owner = !RESOURCES.get().containsKey(DS);   // REQUIRED: join if exists
                if (owner) { RESOURCES.get().put(DS, "conn#" + (++connCounter)); System.out.println("BEGIN " + RESOURCES.get().get(DS)); }
                try {
                    Object r = m.invoke(target, a);
                    if (owner) System.out.println("COMMIT");
                    return r;
                } catch (InvocationTargetException e) {
                    Throwable t = e.getCause();
                    if (owner) System.out.println(t instanceof RuntimeException || t instanceof Error ? "ROLLBACK" : "COMMIT (checked)");
                    throw t;
                } finally { if (owner) RESOURCES.get().remove(DS); }
            });
    }

    public static void main(String[] x) throws Exception {
        OrderService s = proxy(new OrderServiceImpl());
        s.place(false);
        try { s.place(true); } catch (Exception e) { System.out.println("caught " + e.getMessage()); }
        // thread hop loses the binding
        ExecutorService ex = Executors.newSingleThreadExecutor(r -> new Thread(r, "async-1"));
        RESOURCES.get().put(DS, "conn#main");
        System.out.println("main sees: " + s.txState());
        System.out.println("async sees: " + ex.submit(() -> new OrderServiceImpl().txState()).get());
        ex.shutdown();
    }
}
```
Actual output (verified):
```
BEGIN conn#1
  business: conn bound on main = conn#1
COMMIT
BEGIN conn#2
  business: conn bound on main = conn#2
ROLLBACK
caught boom
main sees: conn#main
async sees: null
```
Reading it: the `BEGIN/COMMIT/ROLLBACK` lines are the interceptor; `conn#main` vs `null` is the ThreadLocal boundary.
(Also visible: a self-call `this.place()` inside `OrderServiceImpl` would skip the proxy lines entirely.)

## 10.2 Spring: configuration (Boot 3, JPA)
```java
@Configuration
@EnableTransactionManagement                     // Boot does this automatically; explicit only to set order/mode
class TxConfig {
    // Boot auto-configures JpaTransactionManager. Explicit only if you need customisation:
    @Bean
    PlatformTransactionManager transactionManager(EntityManagerFactory emf) {
        JpaTransactionManager tm = new JpaTransactionManager(emf);
        tm.setValidateExistingTransaction(true);          // fail fast on isolation/readOnly mismatch when joining
        tm.setDefaultTimeout(10);                         // seconds, applies to newly created tx
        return tm;
    }
}
```
`application.yml` (relevant):
```yaml
spring:
  jpa:
    open-in-view: false
  datasource:
    hikari:
      maximum-pool-size: 20
      connection-timeout: 5000
      leak-detection-threshold: 20000
      auto-commit: false
logging.level:
  org.springframework.transaction.interceptor: TRACE
  org.springframework.orm.jpa.JpaTransactionManager: DEBUG
```

## 10.3 Spring: propagation demo (REQUIRES_NEW audit + rollback-only trap)
```java
@Service
public class OrderService {
    private final OrderRepository orders;
    private final AuditService audit;
    private final RiskService risk;
    public OrderService(OrderRepository o, AuditService a, RiskService r) { orders = o; audit = a; risk = r; }

    @Transactional(rollbackFor = Exception.class)
    public void place(long id) throws Exception {
        orders.save(new Order(id, "NEW"));
        audit.log("attempt " + id);                          // REQUIRES_NEW: survives rollback
        try {
            risk.check(id);                                   // REQUIRED: may poison this tx
        } catch (RuntimeException e) {
            // WRONG if risk.check is REQUIRED: outer commit -> UnexpectedRollbackException
            throw e;                                          // correct: propagate
        }
    }
}

@Service
class AuditService {
    private final AuditRepository repo;
    AuditService(AuditRepository r) { repo = r; }
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(String msg) { repo.save(new Audit(msg)); }
}

@Service
class RiskService {
    @Transactional                                            // REQUIRED: joins caller's tx
    public void check(long id) { if (id % 2 == 0) throw new IllegalArgumentException("risky " + id); }
}
```

## 10.4 Spring: per-item transactions with `TransactionTemplate`
```java
@Service
public class ImportService {
    private final TransactionTemplate perItem;
    private final ItemRepository repo;
    private final FailureLog failures;      // its own REQUIRES_NEW / or JdbcTemplate autocommit

    public ImportService(PlatformTransactionManager tm, ItemRepository repo, FailureLog failures) {
        this.perItem = new TransactionTemplate(tm);                       // REQUIRED, new tx as none is active
        this.repo = repo; this.failures = failures;
    }

    public void importAll(List<Item> items) {                             // deliberately NOT @Transactional
        for (Item i : items) {
            try {
                perItem.executeWithoutResult(s -> repo.saveAndFlush(i));  // flush -> constraint errors here, inside the tx
            } catch (RuntimeException e) {
                failures.record(i.id(), e.getMessage());                  // tx of that item already rolled back
            }
        }
    }
}
```

## 10.5 Spring: retry outside the transaction
```java
@Configuration
@EnableRetry
class RetryConfig {}

@Service
public class TransferFacade {                       // no @Transactional here
    private final TransferTx tx;
    public TransferFacade(TransferTx tx) { this.tx = tx; }

    @Retryable(retryFor = { OptimisticLockingFailureException.class, PessimisticLockingFailureException.class,
                            CannotSerializeTransactionException.class },
               maxAttempts = 4, backoff = @Backoff(delay = 30, multiplier = 2, random = true))
    public void transfer(long from, long to, BigDecimal amount) { tx.doTransfer(from, to, amount); }
}

@Service
class TransferTx {
    private final AccountRepository accounts;
    TransferTx(AccountRepository a) { accounts = a; }

    @Transactional
    public void doTransfer(long from, long to, BigDecimal amount) {
        // lock in a consistent order to avoid deadlocks
        long first = Math.min(from, to), second = Math.max(from, to);
        Account a = accounts.findByIdForUpdate(first);        // @Lock(PESSIMISTIC_WRITE) query
        Account b = accounts.findByIdForUpdate(second);
        Account src = a.getId() == from ? a : b, dst = src == a ? b : a;
        src.debit(amount); dst.credit(amount);                // flush at commit
    }
}
```
(`PessimisticLockingFailureException` is the parent of `DeadlockLoserDataAccessException` and `CannotAcquireLockException`, so it covers both.)

## 10.6 Spring: `@TransactionalEventListener` done right
```java
record OrderPlaced(long orderId) {}

@Service
class OrderCommands {
    private final ApplicationEventPublisher publisher; private final OrderRepository repo;
    OrderCommands(ApplicationEventPublisher p, OrderRepository r) { publisher = p; repo = r; }

    @Transactional
    public void place(Order o) { repo.save(o); publisher.publishEvent(new OrderPlaced(o.getId())); }
}

@Component
class OrderPlacedHandler {
    private final BonusRepository bonus;
    OrderPlacedHandler(BonusRepository b) { bonus = b; }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Transactional(propagation = Propagation.REQUIRES_NEW)     // REQUIRED would silently lose the write
    public void on(OrderPlaced e) { bonus.save(new Bonus(e.orderId())); }
}
```

## 10.7 Outbox (schema + write path)
```sql
CREATE TABLE outbox (
  id           BIGINT AUTO_INCREMENT PRIMARY KEY,   -- ordering
  aggregate_id VARCHAR(64) NOT NULL,
  type         VARCHAR(64) NOT NULL,
  payload      JSON        NOT NULL,
  created_at   TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,
  published_at TIMESTAMP   NULL,
  INDEX idx_unpublished (published_at, id)
);
```
```java
@Transactional
public void place(Order o) {
    orders.save(o);
    outbox.save(new OutboxEvent(String.valueOf(o.getId()), "OrderPlaced", json(o)));   // same tx = atomic
}
// relay: scheduled poller, `SELECT ... WHERE published_at IS NULL ORDER BY id LIMIT 100 FOR UPDATE SKIP LOCKED`,
// send to Kafka with key=aggregate_id, set published_at in its own short tx. Consumers dedupe by outbox id.
```

## 10.8 Diagnostics snippet (what am I injected with, is a tx active?)
```java
@Component
class TxProbe {
    void dump(Object bean) {
        System.out.println("proxy=" + AopUtils.isAopProxy(bean) + " cglib=" + AopUtils.isCglibProxy(bean)
            + " target=" + AopProxyUtils.ultimateTargetClass(bean).getSimpleName());
        System.out.println("txActive=" + TransactionSynchronizationManager.isActualTransactionActive()
            + " name=" + TransactionSynchronizationManager.getCurrentTransactionName()
            + " readOnly=" + TransactionSynchronizationManager.isCurrentTransactionReadOnly());
    }
}
```

---
# 11. One-page cheat sheet

```
ARCHITECTURE
 @EnableTransactionManagement → AutoProxyRegistrar + ProxyTransactionManagementConfiguration
   Advisor(BeanFactoryTransactionAttributeSourceAdvisor) = Pointcut(AnnotationTransactionAttributeSource) + TransactionInterceptor
 call → proxy → TransactionInterceptor.invoke → TransactionAspectSupport.invokeWithinTransaction
      → ptm.getTransaction → [your method] → commitTransactionAfterReturning / completeTransactionAfterThrowing
 Resources bound in TransactionSynchronizationManager ThreadLocals (DataSource→ConnectionHolder, EMF→EntityManagerHolder)
 Attribute lookup: target method → target class → interface method → interface class  (no merge; method wins)
 Visibility: public always; protected/package OK in Spring 6 CGLIB; private/static never; final silently skipped

DEFAULTS
 propagation REQUIRED | isolation DEFAULT (DB default) | readOnly false | timeout none
 rollback: RuntimeException + Error only. Checked → COMMIT. Cause chain ignored.
 Boot: CGLIB proxies, open-in-view=true (turn it off), Hikari pool 10 / 30 s wait.

PROPAGATION (outer has tx → inner)
 REQUIRED     join          (inner fail ⇒ mark rollback-only ⇒ outer commit → UnexpectedRollbackException)
 REQUIRES_NEW suspend outer, NEW connection, independent commit   (+1 connection per thread!)
 NESTED       savepoint in same tx; inner fail ⇒ rollback to savepoint; outer fail undoes all (JDBC savepoints)
 SUPPORTS     join if any    | NOT_SUPPORTED suspend, run non-tx (connection still held)
 MANDATORY    require tx     | NEVER  fail if tx exists

ONLY THE OUTERMOST/NEW SCOPE: applies isolation, readOnly, timeout, and really commits/rolls back.
 (joins ignore inner isolation/readOnly/timeout unless validateExistingTransaction=true)

ISOLATION QUICK
 MySQL RR (default): snapshot at first read; locking reads see latest + gap locks; lost update still possible
 PG RC (default): statement snapshot | PG RR: snapshot iso, 40001 → retry | PG SERIALIZABLE: SSI, 40001 → retry | PG RU = RC
 Oracle RC (default) / SERIALIZABLE (snapshot, ORA-08177) / no RR
 Lost update fix: atomic UPDATE, @Version, SELECT FOR UPDATE.  Write skew: SERIALIZABLE(PG)/constraints/locks.

readOnly: Hibernate FlushMode.MANUAL + driver hint; NOT a guard; replica routing needs LazyConnectionDataSourceProxy.
timeout : deadline checked at Spring resource access; not a watchdog.

WHEN NOTHING HAPPENS
 self-invocation | private/static/final | not a bean | @PostConstruct | other thread (@Async, executor, parallelStream)
 | interface annotation w/ CGLIB | wrong tx manager | swallowed / checked exception | JDK proxy injected by class
 FIXES: other bean · TransactionTemplate · @Lazy self · AopContext.currentProxy · AspectJ mode

EVENTS / MESSAGING
 @TransactionalEventListener: BEFORE_COMMIT / AFTER_COMMIT(default) / AFTER_ROLLBACK / AFTER_COMPLETION; no tx ⇒ dropped (fallbackExecution)
 AFTER_COMMIT + @Transactional needs REQUIRES_NEW. Not reliable delivery.
 Don't send to Kafka/Rabbit inside tx → OUTBOX (same-tx insert + relay/CDC + idempotent consumer)
 2PC/XA: atomic, blocking, rare | Saga: local tx + compensation + idempotency + outbox

LONG TX
 No remote calls, IO, sleeps inside. Tx = DB work only. Hikari: leakDetectionThreshold; pg 'idle in transaction'.
 REQUIRES_NEW inside tx doubles connections. OSIV off; DTOs inside the tx; JOIN FETCH/EntityGraph.

RETRY
 Retry OUTSIDE tx (facade bean / lower @Order number). Retry: deadlock, lock timeout, serialization, optimistic lock.
 Optimistic exception appears at commit (from the proxy). Lock rows in consistent order. Retried code idempotent.

TESTS
 @Transactional test = auto rollback ⇒ hides flush/commit errors, AFTER_COMMIT never fires, lazy loading masked.
 Use flush()+clear(), @Commit/@Rollback(false), TestTransaction, Testcontainers. REQUIRES_NEW data is NOT rolled back.

DEBUG
 logging: org.springframework.transaction.interceptor=TRACE, ...orm.jpa.JpaTransactionManager=DEBUG
 TransactionSynchronizationManager.isActualTransactionActive()/getCurrentTransactionName(); AopUtils.isAopProxy(bean)
 Look for: "Creating new transaction", "Participating in existing", "Suspending current", "marking existing transaction as rollback-only"

30-SECOND ANSWER
 "@Transactional is AOP: a proxy whose TransactionInterceptor asks a PlatformTransactionManager to begin, binds the
  connection/EntityManager to the thread, and commits on return or rolls back on RuntimeException/Error. Propagation
  decides whether a call joins (logical scope on the same physical tx), suspends (REQUIRES_NEW), or uses a savepoint
  (NESTED). Inner failures mark the shared tx rollback-only, which is why swallowing them causes UnexpectedRollbackException.
  It breaks on self-invocation, private/final methods and other threads. Keep transactions short, no remote calls inside,
  retry outside, and use an outbox for messaging."
```
