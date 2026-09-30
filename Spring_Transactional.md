# Spring `@Transactional` (Propagation, Isolation, Rollback, Proxy Trap)

## Simple idea
A **transaction** = "all steps succeed, or none of them happen".

**Analogy:** Bank transfer. Debit ₹100 from A, credit ₹100 to B. If debit works and credit fails, money vanishes. So both are wrapped in one transaction → on failure, **rollback** (undo everything).

## How Spring does it (the magic = Proxy)

Spring does NOT change your class. It wraps your bean in a **proxy** object.

```
Caller ──► [ PROXY of OrderService ] ──► [ real OrderService ]
               │  1. begin transaction
               │  2. call real method
               │  3. no exception?  → COMMIT
               │     RuntimeException? → ROLLBACK
```

Flow chart:

```
Controller calls orderService.placeOrder()
            │
            ▼
   Spring Proxy intercepts (AOP)
            │
            ▼
   Is there an existing transaction? ──Yes──► apply PROPAGATION rule
            │No
            ▼
   Open connection, setAutoCommit(false)
            │
            ▼
   Run your method
       │            │
   success        exception
       │            │
       │      RuntimeException / Error ? ──No (checked)──► COMMIT (default!)
       │            │Yes
       ▼            ▼
    COMMIT       ROLLBACK
       └─────┬──────┘
             ▼
   Release connection to pool
```

## Rollback rule (very common trap)
- **Default:** rolls back only on **unchecked** (`RuntimeException`) and `Error`.
- **Checked exception (`IOException`, your custom `extends Exception`) → COMMITS!**
- Fix: `@Transactional(rollbackFor = Exception.class)`.

## Propagation — "what if a transactional method calls another transactional method?"

**Analogy:** You are already inside a taxi (transaction A). You call another service:
- Join the same taxi? (`REQUIRED`)
- Always take a new separate taxi? (`REQUIRES_NEW`)

| Propagation | Meaning | Real use |
|---|---|---|
| **REQUIRED** (default) | Join existing; if none, create new | 95% of code |
| **REQUIRES_NEW** | Suspend current, start a **brand new independent** tx | Audit log / notification that must be saved even if main tx rolls back |
| **SUPPORTS** | Join if exists, else run without tx | Read-only helpers |
| **NOT_SUPPORTED** | Suspend tx, run without | Long report that shouldn't hold a tx |
| **MANDATORY** | Must already have tx, else exception | Enforce caller rules |
| **NEVER** | Must NOT have tx, else exception | – |
| **NESTED** | Savepoint inside current tx; inner rollback can undo only inner part | Partial rollback |

Example — REQUIRES_NEW:

```java
@Service
class OrderService {
    @Autowired AuditService audit;

    @Transactional                       // Tx-1
    public void placeOrder(Order o) {
        orderRepo.save(o);
        audit.log("Order attempted");    // runs in Tx-2 (independent)
        throw new RuntimeException("payment failed");   // Tx-1 rolls back
    }
}

@Service
class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(String msg) { auditRepo.save(new Audit(msg)); }   // Tx-2 COMMITS
}
```
Result: order NOT saved, audit log IS saved.

REQUIRED trap: if inner method (same tx) throws RuntimeException and outer **catches** it, the tx is already marked **rollback-only** → outer commit throws `UnexpectedRollbackException`.

## Isolation levels — "what can other concurrent transactions see?"

Three problems:
- **Dirty read:** I read data another tx has *not committed yet* (may be rolled back).
- **Non-repeatable read:** I read row twice in my tx; it changed between (someone updated + committed).
- **Phantom read:** I run same query twice; *new rows* appear (someone inserted).

| Isolation | Dirty | Non-repeatable | Phantom |
|---|---|---|---|
| READ_UNCOMMITTED | ✗ possible | possible | possible |
| **READ_COMMITTED** (Oracle, PostgreSQL default) | prevented | possible | possible |
| **REPEATABLE_READ** (MySQL InnoDB default) | prevented | prevented | possible* |
| SERIALIZABLE | prevented | prevented | prevented (slowest) |

Higher isolation = more safety, less speed.

## The #1 trap: SELF-INVOCATION (proxy bypass)

```java
@Service
class UserService {
    public void register() {
        this.saveUser();          // ❌ calls real object directly, NOT via proxy
    }
    @Transactional
    public void saveUser() { ... }   // transaction NEVER starts
}
```

```
Controller ─► PROXY ─► register() [no tx annotation, plain call]
                          └─► this.saveUser()   (inside the real object, proxy skipped)
```

Fixes: move `saveUser()` to another bean; or inject self; or annotate `register()` itself.

Other reasons `@Transactional` silently does nothing:
1. Method is `private` / `final` / `static` (proxy can't override).
2. Class is not a Spring bean (`new UserService()`).
3. Checked exception thrown (default = commit).
4. Exception caught inside method and swallowed.
5. `@Transactional` on `@Async` thread → new thread has no tx.

## Other attributes
- `readOnly = true` → hint to DB/Hibernate (no dirty checking, can route to replica). Use on read methods.
- `timeout = 5` → tx aborted if exceeds 5 s.
- Put on **service layer**, not controller/repository.

## Interview answer (30 sec)
> "`@Transactional` is implemented by Spring AOP using a proxy. The proxy opens a transaction before the method, commits on success and rolls back on RuntimeException by default. Propagation controls what happens when transactional methods call each other – REQUIRED joins, REQUIRES_NEW starts an independent one. Common pitfalls are self-invocation, private methods, and checked exceptions not rolling back."
