# Deep JPA / Hibernate 6 / Spring Data JPA 3 — Interview Deep Dive

> Target: 5-year Java developer. Versions assumed: **Jakarta Persistence 3.1, Hibernate ORM 6.x, Spring Data JPA 3.x, Spring Boot 3.x** (`jakarta.persistence.*` packages).
> Basics (annotation lists, simple CRUD, JPQL syntax) live in `JavaFullstack.md` ("JPA & Hibernate"). This note goes under the hood. Transaction propagation and rollback rules live in `Spring_Transactional.md` and are not repeated here.
>
> **Honesty note on SQL samples.** No Maven/Gradle was available when this was written, so nothing here was executed. All SQL is labelled "typical output": statement shape and count are reliable; alias names (`o1_0`), column order and formatting vary by Hibernate version and dialect. Verify on your own project with the logging setup in Part 9.

## Contents
1. 60-second mental model
2. Deep internals: persistence context, entity states, flush, dirty checking, action queue, proxies
3. Fetching and N+1 (with SQL), all fixes compared, pagination traps, cartesian product
4. Mapping traps: associations, cascade, orphanRemoval, equals/hashCode, ids, embeddables, inheritance, converters, enums, time
5. Batching and bulk operations
6. Locking, lost update, isolation, second-level cache
7. OSIV, LazyInitializationException, Spring Data JPA internals
8. Soft delete, auditing, multitenancy, pool, schema management
9. Observability: logging SQL, statistics, p6spy
10. Worked examples (exact SQL)
11. Case study: 5 s to 200 ms
12. Production war stories
13. Interview questions (50+) with follow-ups and wrong answers
14. One-page cheat sheet

---

# 1. The 60-second mental model

**JPA is a specification; Hibernate is the implementation; Spring Data JPA is a convenience layer on top of `EntityManager`.**

```
Your code -> Spring Data repository (interface)
          -> JDK proxy -> SimpleJpaRepository
          -> EntityManager (JPA API)  ==  Hibernate Session (native API)
          -> JDBC (PreparedStatement) -> Connection (HikariCP) -> DB
```

**The one idea that explains 80 % of Hibernate behaviour:** an `EntityManager` owns a *persistence context* - an in-memory map `(EntityType, id) -> entity instance` **plus a snapshot of each entity as it was loaded**. You do not run `UPDATE`; you *mutate managed objects* and Hibernate diffs them against the snapshot at **flush** time and writes SQL. SQL is **deferred and reordered**, not run at the line you wrote it (exceptions: IDENTITY inserts, queries, explicit flush).

**Analogy - the library desk clerk with a clipboard.**
You (the transaction) borrow books (entities). The clerk photocopies each page when you borrow it (snapshot) and keeps a list of who holds which book (identity map: ask twice for book #7 and you get the *same physical copy*). You scribble on your copy. Only when you leave (commit) - or when you ask a question that depends on your scribbles (query -> auto flush) - does the clerk compare copies to originals and phone the archive (SQL). If you leave the desk (session closed) while holding a "proxy slip" for a book not yet fetched, the slip is useless: `LazyInitializationException`.

**Five sentences to say out loud in an interview**
1. First-level cache = persistence context; per transaction (or per request with OSIV); guarantees `find(1) == find(1)`.
2. Managed entity changes are auto-persisted at flush via snapshot dirty checking - `save()` on a managed entity is redundant.
3. Flush happens before a query touching affected tables, on commit, or explicitly - never "when the setter is called".
4. Collections are LAZY by default; `@ManyToOne`/`@OneToOne` are EAGER by default (set them to LAZY).
5. N+1 is the #1 production issue; fix it by choosing the right *fetch shape per use case*, not by making everything EAGER.

---

# 2. Deep internals

## 2.1 The persistence context (first-level cache, identity map, unit of work)

Structure (conceptually; `StatefulPersistenceContext` in Hibernate):

```
+------------------------- EntityManager / Session -------------------------+
| PersistenceContext                                                        |
|  entitiesByKey     : EntityKey(type,id)  -> entity instance  (identity map)|
|  entityEntryContext: entity instance     -> EntityEntry                    |
|        EntityEntry = { status(MANAGED/DELETED/READ_ONLY...),               |
|                        loadedState[]  <-- SNAPSHOT of property values,     |
|                        version, lockMode, id }                             |
|  proxiesByKey      : EntityKey -> proxy                                    |
|  collectionEntries : PersistentCollection -> CollectionEntry (snapshot)    |
|                                                                           |
| ActionQueue (pending SQL): inserts, updates, deletes, collection ops      |
+---------------------------------------------------------------------------+
```

Consequences:
- **Identity guarantee:** within one context `em.find(User.class, 1L) == em.find(User.class, 1L)`; the second call emits **no SQL**.
- **Only `find`/`getReference` by id consult the cache first.** A JPQL/Criteria query *always hits the DB*; for each returned row Hibernate asks "is this id already in my context?" and if yes returns the *existing instance and ignores the row's column values*. You never see fresh DB state for already-loaded entities unless you `refresh()` or `detach()`/`clear()`.
- **Memory:** each loaded entity = live instance + snapshot. Loading 500 000 rows into one context = 2x objects and slow flushes. Page with `clear()`, use DTO projections, or `StatelessSession` for read-only bulk.
- **Scope:** Spring `@Transactional` binds one `EntityManager` to the thread for the tx (transaction-scoped context). With OSIV the context lives for the whole HTTP request (Part 7).

## 2.2 Entity states and the SQL at each transition

```
                    new Foo()
                       |
                   [TRANSIENT]  (no id / not in a context)
                       | persist()
                       v
   find()/query --> [MANAGED] <---- merge(detached) returns a managed COPY
                     |   |  \
        detach()/    |   |   remove()
        clear()/close|   |      \
                     v   |       v
                [DETACHED]|   [REMOVED] --flush--> row deleted
                          |       | persist() again before flush -> MANAGED
                          +-------+
```

| Call | State change | SQL at the call | SQL at flush/commit |
|---|---|---|---|
| `persist(e)` SEQUENCE id | transient -> managed | `select nextval` only if the id block is exhausted | `insert` |
| `persist(e)` IDENTITY | transient -> managed | **`insert` immediately** (needs generated id) | none |
| `persist(e)` assigned id | -> managed | none | `insert` |
| `find(T,id)` cache miss | -> managed | `select ... where id=?` | dirty check |
| `getReference(T,id)` | uninitialised proxy | **none** | none unless accessed |
| setter on managed entity | managed | none | `update` if snapshot differs |
| `remove(e)` | managed -> removed | none | `delete` (+ cascades) |
| `detach`/`clear`/`close` | -> detached | none | later changes silently lost |
| `merge(d)` | state copied onto managed instance | `select` by id if not in context | `update`/`insert` |
| `refresh(e)` | overwritten from DB | `select` | discards unflushed field changes |
| `flush()` | - | - | all pending SQL, now |

**persist vs merge** (frequent question): `persist` makes *that instance* managed (new entities; on a detached instance it throws `PersistentObjectException: detached entity passed to persist`). `merge` **never** makes its argument managed: it loads/creates a managed copy, copies state onto it and returns the copy.

```java
Order detached = ...;              // loaded in an earlier tx
Order managed = em.merge(detached);
detached.setNote("lost");          // NOT tracked
managed.setNote("saved");          // tracked
```

`merge` gotcha: with `cascade=MERGE`, merging a detached graph overwrites DB values with whatever the detached object holds (including nulls from a half-filled DTO->entity mapping). Prefer "load managed entity, copy allowed fields from the DTO".

## 2.3 Flush - what, when, in which order

**Flush = synchronise the persistence context to the DB by executing SQL. It is NOT commit.** Flushed rows are visible only to your own transaction until commit (isolation permitting), and rollback undoes them.

### Flush modes
| Mode | Meaning |
|---|---|
| `FlushModeType.AUTO` (default) | flush before commit; and before a query **when pending changes may affect the tables that query reads** |
| `FlushModeType.COMMIT` | flush only at commit; queries may miss your own unflushed changes |
| Hibernate `FlushMode.MANUAL` | never automatic; only explicit `flush()`. Spring applies this for `@Transactional(readOnly = true)` with Hibernate |

### When AUTO flush fires
1. Commit (Spring: end of the `@Transactional` method, before the actual commit).
2. Before a **JPQL/HQL/Criteria query** if its "query spaces" (tables it reads) overlap tables with pending actions. Unrelated tables: no flush.
3. Before a **native query**: under JPA bootstrapping (Spring) Hibernate typically flushes everything because it cannot know which tables the SQL touches.
4. `em.flush()`, `saveAndFlush`, `@Modifying(flushAutomatically = true)`.
5. **Not** on `find(id)` - it consults cache/DB by key only.

### Flush sequence (numbered trace)
```
1. Cascade PERSIST/MERGE from managed entities (a reachable transient entity not covered by cascade
   -> TransientPropertyValueException / "object references an unsaved transient instance")
2. Dirty check: every MANAGED entity's current values vs EntityEntry.loadedState -> schedule EntityUpdateAction
3. Collection check: PersistentCollection vs its snapshot -> schedule collection actions
4. Sort & execute the ActionQueue in FIXED order (2.5)
5. JDBC batch(es) executed; statements go to the DB (still inside the tx)
6. Snapshots refreshed to the just-written state
```

### Flush-before-query example
```java
@Transactional
void demo() {
  Product p = em.find(Product.class, 1L);   // SELECT
  p.setPrice(new BigDecimal("99"));         // no SQL yet
  List<Product> cheap = em.createQuery(
      "select x from Product x where x.price < 100", Product.class).getResultList();
}
```
Typical output:
```
select p1_0.id, p1_0.name, p1_0.price from product p1_0 where p1_0.id=?
update product set name=?, price=? where id=?                -- auto flush: query reads PRODUCT
select p1_0.id, p1_0.name, p1_0.price from product p1_0 where p1_0.price<100
```
If the query read only `Customer`, the `update` would wait until commit.

## 2.4 Automatic dirty checking

- At load, Hibernate stores `loadedState[]` (copies of property values; deep copy for mutable types like `Date`/arrays; collections tracked separately).
- At flush it compares each managed entity property-by-property (type-specific `equals`). **Cost = O(#managed entities x #properties) per flush**, and AUTO flush may run before *every* query. A context with 50k entities and a loop that runs a query per iteration is effectively O(n^2) - the classic slow batch job.
- **Default UPDATE writes all columns** (`update t set a=?, b=?, c=? where id=?`) so the statement text is stable (cacheable, batchable). `@DynamicUpdate` (Hibernate) generates `set` only for changed columns: helps wide tables or concurrent partial updates; costs SQL generation per update and defeats batching because differing SQL strings break a JDBC batch.
- **`@Transactional(readOnly = true)`** with Spring+Hibernate: sets flush mode MANUAL (commit-time dirty check skipped; accidental setters not persisted) and hints the JDBC connection as read-only. It does not stop an explicit `save()`+`flush()`. Hibernate also has a per-query read-only hint (`org.hibernate.readOnly`) so loaded entities need no snapshots.
- Bytecode enhancement (build-time plugin) can track dirtiness in instrumented setters instead of snapshot compare - opt-in niche optimisation, not default.
- **Mutable-attribute trap:** dirty check depends on equality. A JSON POJO/`Map` attribute without a proper `equals` is either always dirty (an update on every flush) or missed. Give value types real `equals`, or treat them as immutable/copy-on-write.

Trap: "I never called save() but the row changed" - the entity was managed inside a transaction. Reverse: "I called save() but nothing updated" - detached entity, `readOnly` tx, no active tx, or `@Transactional` self-invocation (see `Spring_Transactional.md`).

## 2.5 ActionQueue ordering (why SQL is not in code order)

At flush Hibernate executes queued actions in this fixed order:

```
1. Orphan removals
2. EntityInsertAction        (all inserts; IDENTITY ones already ran at persist)
3. EntityUpdateAction        (all updates)
4. Collection removals       (dropping whole collections)
5. Collection updates        (rows added/removed in join tables / element collections)
6. Collection recreations
7. EntityDeleteAction        (deletes LAST)
```
Within each kind, order = order of persist/mark, unless `hibernate.order_inserts` / `order_updates` re-sort for batching.

**Consequence - unique-constraint swap fails:**
```java
// unique index on users(email)
userRepo.delete(oldUser);                 // queued delete
userRepo.save(new User("a@x.com"));       // queued insert with the same email
// commit -> INSERT runs first, DELETE last -> duplicate key violation
```
Fix: `userRepo.delete(oldUser); userRepo.flush();` before the insert, or simply update the existing row.

Same reason "delete children, re-add children with the same natural key" or "replace a `@OneToMany` list contents with identical unique values" breaks: inserts precede deletes. FK problems arise only if the graph is malformed (child references a transient parent -> `TransientPropertyValueException`).

## 2.6 Proxies and lazy loading

### How a proxy works
- For a lazy `@ManyToOne`/`@OneToOne` and for `em.getReference`, Hibernate generates at runtime a **subclass of your entity (ByteBuddy)** whose methods are intercepted: the first non-identifier call makes the `LazyInitializer` load the real entity (`select ... where id=?`) into the persistence context and delegate all later calls to it.
- Collections use `PersistentBag/Set/List/Map` wrappers (not subclass proxies) that load on first read (`size()`, `iterator()`, `get`).

```
 Order.customer  ---> Customer$HibernateProxy (extends Customer)
                         id=7 (known, getId() needs no SQL)
                         target=null
   customer.getName() -> initialize: select ... from customer where id=7
                         target = real Customer (also stored in the PC)
                         delegate getName() to target
```
- **Requirements:** entity class not `final`, no `final` methods you rely on being intercepted, a no-arg constructor (protected is fine). A `final` entity class cannot be proxied: laziness silently degrades or fails.

### getReference vs find
| | `find(T,id)` | `getReference(T,id)` |
|---|---|---|
| SQL | immediate select | none until first non-id access |
| Missing row | returns `null` | `EntityNotFoundException` **later**, on init |
| Use | you need data | you need only an FK: `order.setCustomer(em.getReference(Customer.class, cid))` saves a select |
| Spring Data | `findById` | `getReferenceById` (older name `getOne`) |

### Proxy pitfalls
1. **`getClass()`** returns the proxy class, so `getClass() != Customer.class` breaks naive `equals`. Use `instanceof` (works on proxies) or `Hibernate.getClass(obj)`.
2. **`instanceof` with inheritance:** a lazy `Animal a` may be a proxy of `Animal` even when the row is a `Dog` -> `a instanceof Dog` is **false** until unproxied. Use `Hibernate.unproxy(a)` (returns the real implementation) or a visitor/polymorphic method instead of instanceof chains.
3. **Field access inside `equals`** (`other.name`) on a proxy reads the proxy's own empty field; use getters in entity `equals/hashCode`.
4. **Jackson** serialising an uninitialised proxy fails (`LazyInitializationException` or `hibernateLazyInitializer` error). Don't expose entities; use DTOs.
5. `Hibernate.initialize(x)` forces load; `Hibernate.isInitialized(x)` / `PersistenceUnitUtil.isLoaded(x)` check without loading.
6. A proxy created earlier stays a proxy even after the real entity is loaded in the same context; `proxy == real` is false. Compare by id.

### The inverse `@OneToOne` cannot really be lazy
The side that does *not* own the FK (`mappedBy` one-to-one) must query the other table to know whether to set `null` or a proxy, so Hibernate loads it eagerly. Options: make the FK side the only navigable side; share the PK using `@MapsId` and query from the owning side; or bytecode enhancement. A known source of "mystery" extra selects.

---

# 3. Fetching and N+1

## 3.1 Defaults per association (JPA spec)
| Association | Default | Advice |
|---|---|---|
| `@ManyToOne` | **EAGER** | set `fetch = LAZY` always |
| `@OneToOne` | **EAGER** | set LAZY on the FK-owning side (inverse side: see 2.6) |
| `@OneToMany` | LAZY | keep |
| `@ManyToMany` | LAZY | keep |
| `@ElementCollection` | LAZY | keep |
| `@Basic` (incl. `@Lob`) | EAGER | `@Basic(fetch=LAZY)` needs bytecode enhancement to be honoured |

`FetchType` on the mapping is the *default plan*. Per-use-case plans (join fetch, entity graph) override it for queries. **A mapping-level EAGER can never be turned off by a query**: `findById` uses a join for EAGER, but a JPQL query loads the root first and then fires *extra selects* for each EAGER association (hidden N+1). That is why "make it EAGER to fix LazyInitializationException" is an anti-pattern.

## 3.2 N+1 explained with real-looking SQL
Model: `Author 1--* Book` (LAZY).

```java
List<Author> authors = em.createQuery("select a from Author a", Author.class).getResultList(); // 1 query
for (Author a : authors) {
    System.out.println(a.getName() + " -> " + a.getBooks().size());   // +1 query per author
}
```
Typical output for 3 authors:
```
select a1_0.id, a1_0.name from author a1_0                                  -- 1
select b1_0.author_id, b1_0.id, b1_0.title from book b1_0 where b1_0.author_id=?   -- author 1
select b1_0.author_id, b1_0.id, b1_0.title from book b1_0 where b1_0.author_id=?   -- author 2
select b1_0.author_id, b1_0.id, b1_0.title from book b1_0 where b1_0.author_id=?   -- author 3
```
1 + N statements. With 1 000 authors and 2 ms round trip: 2 s of pure latency, plus pool occupancy. Also happens for `@ManyToOne` EAGER via JPQL (`select b from Book b` -> then `select ... from author where id=?` per *distinct* author not yet in the context), and through Jackson/mappers iterating lazy collections.

**Detecting:** SQL log with statement counts (Part 9); Hibernate statistics `prepareStatementCount`; test assertions with `datasource-proxy`/`QueryCountHolder`; APM (many identical short queries per request).

## 3.3 All the fixes, compared

| Fix | Mechanism | SQL shape | Pros | Cons / traps |
|---|---|---|---|---|
| `join fetch` (JPQL) | inner/left join, hydrates association in one query | 1 query, rows = parent x children | simple, exact | row duplication; **cannot paginate a collection fetch in DB**; two bag fetches impossible; can't alias/filter the fetched collection safely |
| `@EntityGraph` (`attributePaths`) | declarative fetch plan, usually left join | same as join fetch | works on derived methods (`findAll`), reusable | same limits as join fetch |
| `@BatchSize(size=N)` / `hibernate.default_batch_fetch_size` | load lazy things for *N owners at once* using `IN (?,?,...)` | 1 + ceil(N_owners/batch) | no query changes, pagination-safe, works for lazy collections and proxies | still extra queries; batch size tuning; `IN` list padding |
| `@Fetch(FetchMode.SUBSELECT)` | load all collections for all owners of the *original query* with a subselect | 2 queries | good for "load all then iterate" | re-executes the original query as a subquery; bad with big/complex queries; not for paginated lists |
| DTO projection | select only needed columns (constructor expression / interface projection / native) | 1 query, tailored | fastest, no PC overhead, no lazy issues | no managed entity (can't update); mapping code |
| Two-query approach | (1) page of ids/roots, (2) fetch children `where id in :ids` | 2 queries | correct pagination + no duplication | more code, keep order stable |

### join fetch (Hibernate 6)
```java
@Query("select a from Author a join fetch a.books where a.country = :c")
List<Author> findWithBooks(String c);
```
```
select a1_0.id, a1_0.name, b1_0.author_id, b1_0.id, b1_0.title
from author a1_0 join book b1_0 on a1_0.id=b1_0.author_id where a1_0.country=?
```
Hibernate 6 **deduplicates root entities automatically** in HQL results (the old `select distinct` to collapse duplicates is no longer required; in 5.x you needed it, and `distinct` also went to SQL unless the `hint.passDistinctThrough` hint was set).
Use `left join fetch` if an author with no books must still appear (inner `join fetch` drops them).

### @EntityGraph
```java
public interface AuthorRepo extends JpaRepository<Author, Long> {
    @EntityGraph(attributePaths = {"books"})          // fetch graph semantics
    List<Author> findByCountry(String country);
}
```
Also available as `@NamedEntityGraph` on the entity and referenced by name. `type = FETCH` (default: listed attributes eager, all others per Hibernate treated LAZY) vs `LOAD` (listed attributes eager, others as per mapping). Same pagination/cartesian caveats as join fetch.

### Batch fetching (great pragmatic default)
```properties
spring.jpa.properties.hibernate.default_batch_fetch_size=16
```
The N+1 becomes:
```
select a1_0.id, a1_0.name from author a1_0
select b1_0.author_id, b1_0.id, b1_0.title from book b1_0 where b1_0.author_id in (?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)   -- 16 authors per query
...
```
1000 authors -> 1 + 63 statements instead of 1001. Works with `Pageable` because the parent query stays simple. Per-entity/collection: `@BatchSize(size = 20)`.

### Subselect
```java
@OneToMany(mappedBy = "author") @Fetch(FetchMode.SUBSELECT) private List<Book> books;
```
```
select ... from author a1_0 where a1_0.country=?
select b1_0.author_id, ... from book b1_0 where b1_0.author_id in (select a1_0.id from author a1_0 where a1_0.country=?)
```

### DTO projection - often the best answer for read endpoints
```java
public record AuthorBookCount(Long id, String name, long books) {}

@Query("select new com.acme.AuthorBookCount(a.id, a.name, count(b)) " +
       "from Author a left join a.books b group by a.id, a.name")
List<AuthorBookCount> summary();
```
```
select a1_0.id, a1_0.name, count(b1_0.id) from author a1_0 left join book b1_0 on a1_0.id=b1_0.author_id group by a1_0.id, a1_0.name
```
One statement, nothing enters the persistence context (no snapshots, no dirty checking).

## 3.4 Cartesian product, MultipleBagFetchException, Set vs List

Fetching **two collections** of one root in one query multiplies rows: 10 orders x 5 items x 4 payments -> 200 rows for 10 roots.
```java
select o from Order o join fetch o.items join fetch o.payments      // items: List, payments: List
```
Hibernate refuses when both collections are *bags* (unordered `List` without `@OrderColumn`): `org.hibernate.loader.MultipleBagFetchException: cannot simultaneously fetch multiple bags`. It cannot tell which duplicate rows are real.

Wrong "fix" seen everywhere: change both `List` to `Set`. It compiles and stops the exception, but **the cartesian product remains** (200 rows, wasted bandwidth/CPU; result explodes with more children). Correct approaches:
1. Fetch one collection per query (two-step), relying on the persistence context to attach both to the same root instances:
```java
List<Order> orders = em.createQuery("select distinct o from Order o join fetch o.items where o.id in :ids", Order.class)
                       .setParameter("ids", ids).getResultList();
orders = em.createQuery("select distinct o from Order o join fetch o.payments where o in :orders", Order.class)
                       .setParameter("orders", orders).getResultList();   // same instances, payments now initialised
```
2. Use `@BatchSize`/`default_batch_fetch_size` for the second collection.
3. DTO/projection queries.
Use `Set` where the collection semantically is a set (and `equals/hashCode` are sound - 4.4); a `List` mapped as a bag has another cost: removing one element from a bag can make Hibernate **delete all join rows and re-insert** the rest (a `@ManyToMany` `List`), whereas a `Set` deletes just that row.

## 3.5 Pagination with collection fetch - the in-memory trap
```java
@Query("select a from Author a join fetch a.books")
Page<Author> page(Pageable p);     // DANGER
```
Hibernate cannot apply `LIMIT` to a joined result (limit would cut rows, not authors), so it **fetches every row, paginates in JVM memory** and logs:
- 5.x: `HHH000104: firstResult/maxResults specified with collection fetch; applying in memory!`
- 6.x: the same idea with a code in the `HHH90003004` range ("firstResult/maxResults specified with collection fetch. In memory pagination was about to be applied ..."). Exact code varies by version - search your logs for "firstResult/maxResults".
On a big table this means loading the entire join into heap: latency, OOM. Spring Data's count query for `Page` also does not like `join fetch` (`query specified join fetching, but the owner of the fetched association was not present in the select list` when the derived count query keeps the fetch) - supply `countQuery`.

**Correct pattern (two queries):**
```java
// 1) page only the roots (no collection fetch), stable ORDER BY with tie-breaker
Page<Long> ids = repo.findIds(pageable);            // select a.id from Author a order by a.name, a.id
// 2) fetch the graph for those ids
List<Author> list = repo.findWithBooksByIdIn(ids.getContent()); // join fetch a.books where a.id in :ids
// 3) restore the page order in Java (IN does not guarantee order) -> sort by ids list
```
or keep the parent query paginated and let batch fetching load `books`. `ToOne` joins (many-to-one) are safe to `join fetch` with pagination - only *collection* fetches are the problem.

## 3.6 Choosing a fetch plan (decision guide)
```
Read-only list/report screen?          -> DTO projection (fastest)
Need entities to modify, one root?     -> find + join fetch / @EntityGraph on the specific repo method
Paginated list of roots + children?    -> page roots, then batch fetch or 2nd query
Many ToOne on list rows?               -> join fetch ToOne (safe with paging), or projection
Legacy code with N+1 everywhere?       -> default_batch_fetch_size=16..64 as an emergency net
```

---

# 4. Mapping traps

## 4.1 Bidirectional associations: owning side, mappedBy, sync helpers
- The **owning side** is the one holding the FK column (usually `@ManyToOne` side). Only the owning side's state is written to the DB. `mappedBy` marks the inverse side, which is *read-only for persistence*.
- If you only do `parent.getChildren().add(child)` and never `child.setParent(parent)`, the FK stays `NULL` (or insert fails on NOT NULL). If you only set `child.setParent(p)`, the DB is right but your in-memory `p.getChildren()` is stale within the same context.
- Therefore keep both in sync with helpers on the parent:

```java
@Entity
public class Order {
    @Id @GeneratedValue private Long id;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private Set<OrderLine> lines = new HashSet<>();

    public void addLine(OrderLine l)    { lines.add(l);    l.setOrder(this); }
    public void removeLine(OrderLine l) { lines.remove(l); l.setOrder(null); }
}

@Entity
public class OrderLine {
    @Id @GeneratedValue private Long id;
    @ManyToOne(fetch = FetchType.LAZY, optional = false) @JoinColumn(name = "order_id")
    private Order order;
}
```
- **Unidirectional `@OneToMany` without `mappedBy`** (with or without `@JoinColumn`) is a trap: without `@JoinColumn`, Hibernate creates a *join table*; with `@JoinColumn` it inserts child then issues a separate `update child set order_id=?` (extra statement, NOT NULL problems). Prefer bidirectional with `@ManyToOne` as owner.
- `toString`/`equals`/`hashCode`/Lombok `@Data` on bidirectional entities: infinite recursion or accidental collection initialisation. Exclude associations from `toString`/`hashCode`.

## 4.2 Cascade types and orphanRemoval

| Cascade | Effect |
|---|---|
| `PERSIST` | `persist(parent)` also persists new children (and at flush for reachable transient ones) |
| `MERGE` | `merge` cascades |
| `REMOVE` | `remove(parent)` also removes children |
| `REFRESH`, `DETACH` | cascade those operations |
| `ALL` | all of the above |

Hibernate additionally has its own `org.hibernate.annotations.CascadeType` values (e.g. `SAVE_UPDATE`, `REPLICATE`, `LOCK`); the Hibernate `@Cascade` annotation is legacy-ish in 6.x - stick to JPA `CascadeType`.

**orphanRemoval = true** (only meaningful for `@OneToMany`/`@OneToOne`): a child removed from the parent's collection (or replaced) is **deleted** at flush. It implies "child lifecycle is owned by the parent".

Semantics differences: `cascade=REMOVE` deletes children when the *parent is deleted*; `orphanRemoval` also deletes when the child is merely *de-referenced*. Both together (`ALL` + `orphanRemoval`) = aggregate root pattern.

Traps:
1. **`CascadeType.REMOVE`/`ALL` on `@ManyToMany`** - deleting one `Student` would delete every `Course` it references, even though other students share them (FK violation or data loss). Never cascade REMOVE on many-to-many or on `@ManyToOne`.
2. **`cascade = ALL` on `@ManyToOne`** (child -> parent): deleting an `OrderLine` cascades a delete to its `Order` and (via order's cascade) all other lines. Cascade flows parent -> child only.
3. **Replacing the collection instance** `parent.setChildren(newList)` on a managed parent with orphanRemoval throws `HibernateException: A collection with cascade="all-delete-orphan" was no longer referenced by the owning entity instance`. Mutate the existing collection (`clear(); addAll()`), or use helpers.
4. `orphanRemoval` on an entity shared by two parents surprises: removed from one collection -> deleted for both.
5. Cascade `PERSIST` on a **detached** child (with id) -> `detached entity passed to persist`.
6. `@OnDelete(action = OnDeleteAction.CASCADE)` (Hibernate) pushes the delete down to the DB `ON DELETE CASCADE` - one statement, but skips entity callbacks/cache eviction/collection sync.

## 4.3 `@ElementCollection` and many-to-many with extra columns
`@ElementCollection` maps a collection of basics/embeddables to a *collection table* (`parent_id` + value columns); no identity of its own.
```java
@ElementCollection
@CollectionTable(name = "user_tag", joinColumns = @JoinColumn(name = "user_id"))
@Column(name = "tag")
private Set<String> tags = new HashSet<>();
```
Cost: modifying a **List** (bag) element collection typically deletes all rows and re-inserts; a Set still may rewrite on certain changes. There are no row ids, so no per-row targeting. For anything queried, audited, big, or shared: promote to a real entity.

**Many-to-many with extra columns cannot be `@ManyToMany`** (the join table has no entity to hold `enrolledAt`, `grade`). Use a **link entity** with two `@ManyToOne`s:
```java
@Embeddable
public class EnrollmentId implements Serializable {
    private Long studentId; private Long courseId;   // equals/hashCode required
}
@Entity
public class Enrollment {
    @EmbeddedId private EnrollmentId id = new EnrollmentId();
    @ManyToOne(fetch = LAZY) @MapsId("studentId") private Student student;
    @ManyToOne(fetch = LAZY) @MapsId("courseId")  private Course course;
    private Instant enrolledAt;
    private Integer grade;
}
```
Student and Course each have `@OneToMany(mappedBy = "student"/"course", cascade = ALL, orphanRemoval = true) Set<Enrollment>`.

## 4.4 equals / hashCode for entities
Problems:
- `hashCode` on a generated id changes after `persist` (id null -> value) -> an entity stored in a `HashSet` **before** persist is "lost" from the set afterwards.
- `equals` on all fields breaks with lazy proxies and mutable fields.
- Default `Object` identity breaks across detached/merged copies (two instances for the same row are unequal) - only OK if you never compare across contexts.

Safe patterns (choose one):
1. **Business/natural key** (immutable, unique, set at construction): `isbn`, `email` (if truly immutable), `@NaturalId` (Hibernate, supports natural-id lookup and caching). `equals` compares business key via getters; `hashCode` from it.
2. **Generated id with constant hashCode** (Vlad Mihalcea's pattern): 
```java
@Override public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Book other)) return false;            // instanceof handles proxies
    return getId() != null && getId().equals(other.getId()); // transient entities equal only to themselves
}
@Override public int hashCode() { return getClass().hashCode(); }  // stable across states; Set is a linear list-ish per type
```
(With proxies, `getClass()` may differ from the real class, so many teams use a fixed literal like `31` or `Hibernate.getClass(this).hashCode()` - the point is the value never changes over the object's life.)
3. **Assign the id yourself before persist** (UUID/ULID generated in constructor) - then id-based equals/hashCode is stable from birth.

Avoid: Lombok `@Data`/`@EqualsAndHashCode` on entities (includes all fields and associations, triggers lazy loads); `hashCode` using a lazy collection.

## 4.5 Identifier generation

| Strategy | How | Batching | Notes |
|---|---|---|---|
| `IDENTITY` | DB auto-increment; id known only after `INSERT` | **Disables JDBC insert batching** (Hibernate must execute each insert at `persist` to learn the id) | simple; fine for low-volume tables; MySQL classic |
| `SEQUENCE` | DB sequence; `allocationSize` (default **50**) ids reserved per `nextval` (pooled optimizer) | **Works** | best default on PostgreSQL/Oracle/SQL Server/DB2/H2; **DB sequence `INCREMENT BY` must equal `allocationSize`** |
| `AUTO` | Hibernate 6 picks per type: `Long` -> sequence-style generator (a table emulation on DBs without sequences, e.g. older MySQL), `UUID` -> UUID | depends | `Long` ids on MySQL get a `<entity>_seq` emulation table (surprise); be explicit |
| `TABLE` | row-locking id table | works but slow | portability only; contention |
| `UUID` (Hibernate 6.x: `@UuidGenerator`) / app-assigned | generated in JVM | works | 16 bytes; random v4 fragments B-tree indexes; time-ordered variants recommended |

**Pooled optimizer trace (allocationSize 50):** first `persist` -> Hibernate calls `nextval` (typically twice at startup to seed the block) and then hands out 50 consecutive ids **from memory**; the 51st id triggers the next `nextval`. So persisting 1 000 entities costs ~20 `nextval` calls, not 1 000. Trap: two apps/other tools inserting with a different increment, or a sequence created with `INCREMENT BY 1` while `allocationSize=50` -> **duplicate ids / gaps** (Hibernate validates increment vs allocationSize in newer versions, logging a warning/ error at startup).

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq")
@SequenceGenerator(name = "order_seq", sequenceName = "order_seq", allocationSize = 50)
private Long id;
// DDL: create sequence order_seq start with 1 increment by 50;
```
Gaps in ids after restart are normal (unused pre-allocated ids are discarded).

**UUID / ULID:** + no round trip, globally unique, safe to expose, can be assigned before persist (equals/hashCode stability, offline/merge friendly); - 16 B vs 8 B keys bloat every index and FK, random v4 UUIDs cause page splits and poor cache locality on clustered PKs (MySQL InnoDB, SQL Server); mitigate with time-ordered ids (UUIDv7-style, ULID) or a surrogate `bigint` PK plus a public UUID column. Store as native `uuid` (PostgreSQL) or `BINARY(16)`, not `varchar(36)`.

## 4.6 Composite keys, embeddables
- `@Embeddable` value object mapped into the owner's table (`Address` -> columns in `customer`). No identity, no lazy load, null-all-columns means null object. Override with `@AttributeOverride`. Embeddables can contain `@ManyToOne`.
- **`@EmbeddedId`** key class (must be `Serializable`, with correct `equals/hashCode`) - one field `id` in entity, navigable (`e.id.studentId` in JPQL).
- **`@IdClass`** - key fields declared directly on the entity plus a parallel key class with the same field names; queries use `e.studentId`. Slightly less nesting, more duplication.
- Prefer a surrogate key + unique constraint over composite keys unless the domain truly is a link table.

## 4.7 Inheritance strategies

| Strategy | Tables | Polymorphic query | Pros | Cons |
|---|---|---|---|---|
| `SINGLE_TABLE` (default) | one table + discriminator | one select, no joins | fastest, simple | subclass columns nullable (no NOT NULL per subclass), wide sparse table |
| `JOINED` | table per class, joined by PK | joins across hierarchy | normalised, NOT NULL possible | joins on every polymorphic read/write, multiple inserts |
| `TABLE_PER_CLASS` | one full table per concrete class | `UNION ALL` | no joins for concrete-type queries | polymorphic queries are unions; can't use IDENTITY; poor with associations to the base type; generally avoided |
| `@MappedSuperclass` | not an entity, just shared mapping (id, audit columns) | cannot query the superclass | good reuse | no polymorphic association to it |

Rule of thumb: shallow hierarchies with mostly shared columns -> SINGLE_TABLE; strong integrity needs -> JOINED; base-type only for code reuse -> `@MappedSuperclass`. Prefer composition over deep inheritance.

## 4.8 Converters, enums, dates
- **`AttributeConverter<X,Y>`** with `@Convert` (or `@Converter(autoApply = true)`) maps Java types to columns (e.g. `Money` -> long, `List<String>` -> csv/JSON, encrypted strings). Converted values are dirty-checked by `equals` of the *Java* type; converters can't be used on `@Id`, version, or relationship attributes and are not applied to queries' literal expressions in every case - pass parameters.
- **Enums:** `@Enumerated(EnumType.ORDINAL)` is the default and a trap - inserting a constant in the middle of the enum silently remaps every stored row. Use `EnumType.STRING` (or a converter to a stable code). Hibernate 6 can also generate DB check constraints/native enum types for schema generation; still prefer stable string codes.
- **Time:** use `java.time`. `Instant`/`OffsetDateTime` for points in time (store in UTC, `timestamp with time zone` where available); `LocalDate` for dates; `LocalDateTime` has *no zone* - only for "wall clock" concepts. Set `spring.jpa.properties.hibernate.jdbc.time_zone=UTC` so the driver doesn't shift by JVM default zone. Never use `java.util.Date`/`Calendar` in new code. Beware that `timestamp` (without tz) columns + JVM zone changes = silent hour shifts.
- `BigDecimal` for money with explicit `precision/scale` (`@Column(precision = 19, scale = 2)`); never `double`.

---

# 5. Batching and bulk operations

## 5.1 JDBC batching configuration
Hibernate does **not** batch by default (`hibernate.jdbc.batch_size` unset = one round trip per statement).
```properties
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
spring.jpa.properties.hibernate.jdbc.batch_versioned_data=true   # default true in modern versions; batches @Version updates
```
- `order_inserts`: sort pending inserts by entity type so consecutive statements share the same SQL (parent1, child1, parent2, child2 would otherwise interleave and *break* each batch). `order_updates` likewise for updates.
- Batching requires: **not IDENTITY**, same SQL text consecutively, driver support. MySQL Connector/J additionally needs `rewriteBatchedStatements=true` for real multi-row batching; PostgreSQL's driver has `reWriteBatchedInserts=true`.
- `@DynamicUpdate` produces varying SQL and thus smaller batches.

### Traced example - inserting 120 `Product` rows (SEQUENCE, allocationSize 50, batch_size 50)
```java
@Transactional
public void importAll(List<ProductDto> dtos) {
    int i = 0;
    for (ProductDto d : dtos) {
        em.persist(new Product(d.name(), d.price()));
        if (++i % 50 == 0) { em.flush(); em.clear(); }    // bounded memory, batch boundary
    }
}
```
Typical trace:
```
persist #1        -> select nextval('product_seq')  (pool seeded; further nextval only when 50 ids used up)
persist #1..#50   -> no INSERT yet (queued)
flush at #50      -> JDBC batch #1: 50 x "insert into product (name,price,id) values (?,?,?)"  (1 network batch, not 50 trips)
clear             -> persistence context emptied (snapshots gone)
...
flush at #100     -> batch #2 (50 rows)
commit            -> remaining 20 rows flushed as batch #3
```
**With IDENTITY** the same loop issues 120 individual `insert`s immediately at each `persist` and `flush` has nothing to batch.

### How to verify batching really happens
- SQL logging shows one line per statement even when batched - **not proof**. Use: Hibernate statistics `JDBC batches executed`/`statements prepared` (Part 9); log `org.hibernate.engine.jdbc.batch.internal.BatchingBatch` at DEBUG ("Executing batch size: 50"); datasource-proxy/p6spy batch markers; or the DB's own statement stats (round-trips in `pg_stat_statements` calls vs rows).
- Test: insert 1 000 rows and time it with batch_size 0 vs 50; expect order-of-magnitude difference on a remote DB.

### `saveAll` vs loop
`saveAll(list)` is just a loop of `save` inside one transaction; for new entities with generated ids it calls `persist` each. Batching still depends on the config above. For hundreds of thousands of rows: chunk + `flush/clear`, or `StatelessSession`, or JDBC `JdbcTemplate.batchUpdate`, or DB bulk loaders (`COPY`, `LOAD DATA`).

## 5.2 Bulk JPQL/HQL updates and deletes
```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("update Product p set p.price = p.price * 1.1 where p.category = :c")
int raise(@Param("c") String c);
```
- Executes **directly in the DB** as `update product set price=price*1.1 where category=?` - one statement, no entity loading, **bypasses the persistence context**, `@Version` increments (unless `update versioned`), lifecycle callbacks, cascades and second-level cache invalidation per entity (Hibernate does invalidate affected entity regions for HQL bulk updates, but not the query-level semantics you might assume - test it).
- **Stale context trap:** entities already loaded keep their old prices. `clearAutomatically = true` clears the whole context after the statement (detaching everything, including other unflushed-but-flushed-first entities) - hence `flushAutomatically = true` first so pending changes aren't lost. Alternative: `em.refresh(entity)` for specific ones.
- `@Modifying` is required for update/delete queries else `IllegalStateException`/`InvalidDataAccessApiUsageException` ("Not supported for DML operations"). Must run inside a transaction.
- Derived `deleteBy...` loads entities then deletes each (N+1 deletes); `@Modifying @Query("delete ...")` is a single statement.

## 5.3 StatelessSession
`sessionFactory.openStatelessSession()`: no persistence context, no dirty checking, no cascades, no lazy loading, no second-level cache, direct `insert/update/delete` (batchable). Ideal for ETL/import; unsuitable for entity graphs. Get the factory via `entityManager.getEntityManagerFactory().unwrap(SessionFactory.class)`.

---

# 6. Locking, lost updates, isolation, second-level cache

## 6.1 The lost-update anomaly - worked example (2 threads)
Account `id=1, balance=100`, two withdrawals of 30 with **no locking**, default READ COMMITTED.
```
T1: begin
T2: begin
T1: select balance from account where id=1        -> 100
T2: select balance from account where id=1        -> 100
T1: balance = 100 - 30 = 70
T2: balance = 100 - 30 = 70
T1: update account set balance=70 where id=1 ; commit
T2: update account set balance=70 where id=1 ; commit     -- overwrites T1
final balance = 70  (should be 40) -> LOST UPDATE
```
READ COMMITTED does not prevent this (both reads saw committed 100). REPEATABLE READ prevents it in some DBs (PostgreSQL: second updater gets a serialization failure; MySQL InnoDB: gap/next-key behaviour differs and a plain read-modify-write in application code can still lose updates) - never rely on isolation alone; make the intent explicit with locking.

## 6.2 Optimistic locking (`@Version`)
```java
@Entity
public class Account {
    @Id @GeneratedValue private Long id;
    private BigDecimal balance;
    @Version private long version;       // int/long/short/Integer/Long/Timestamp/Instant
}
```
Update statement becomes:
```
update account set balance=?, version=? where id=? and version=?      -- version=? new (n+1), last = value read
```
If 0 rows are updated (someone else committed first) Hibernate throws `jakarta.persistence.OptimisticLockException` (JPA) / Hibernate `StaleObjectStateException`; Spring translates to **`ObjectOptimisticLockingFailureException`** (subclass of `OptimisticLockingFailureException`, `DataAccessException`). Trace of the same race: T2's `update ... where version=0` finds version already 1 -> 0 rows -> exception -> T2 rolls back; T1's write is preserved.

Notes:
- No DB locks are held between read and write, so throughput is high with low contention. Under **high contention**, retries pile up -> use pessimistic or atomic SQL (`update account set balance = balance - :amt where id=:id and balance >= :amt`).
- Version is also incremented when a `@OneToMany` *collection owned by the entity* changes (unless `@OptimisticLock(excluded = true)`).
- Bulk JPQL updates don't touch the version unless `update versioned`.
- Detached-entity flows (web forms): send the version to the client, put it into the DTO, copy to the entity before `merge`/compare - otherwise the check protects nothing across requests (it only protects within a load-modify-flush window).
- **Retry:** catch `ObjectOptimisticLockingFailureException` **outside** the transactional boundary (a failed tx is rolled back and its context is dead), re-run the whole unit of work with a bounded attempt count and backoff (Spring Retry `@Retryable`, ordered before `@Transactional`; or a manual loop in a non-transactional facade). Return 409 Conflict to users when retry is inappropriate.
- `LockModeType.OPTIMISTIC` / `OPTIMISTIC_FORCE_INCREMENT`: `OPTIMISTIC` re-checks the version at commit for entities you *only read* (protect against write skew on inputs); `FORCE_INCREMENT` bumps the version even without changes (e.g. to serialise changes to an aggregate whose children changed).

## 6.3 Pessimistic locking
```java
public interface AccountRepo extends JpaRepository<Account, Long> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
    @Query("select a from Account a where a.id = :id")
    Optional<Account> lockById(@Param("id") Long id);
}
```
Typical SQL: `select ... from account a1_0 where a1_0.id=? for update` (PostgreSQL/MySQL/Oracle; SQL Server uses table hints `with (updlock, holdlock, rowlock)`; H2 `for update`).
- `PESSIMISTIC_READ` -> shared lock (`for share`/`for key share`-style depending on dialect; some dialects fall back to `for update`).
- `PESSIMISTIC_FORCE_INCREMENT` combines write lock and version bump.
- Lock is held **until the transaction ends** - keep the tx short; never call remote services while holding it.
- **Timeout:** hint `jakarta.persistence.lock.timeout` (ms; `0` = NOWAIT where supported). Failure surfaces as `PessimisticLockException`/`LockTimeoutException` -> Spring `PessimisticLockingFailureException`/`CannotAcquireLockException`. Deadlocks: `DeadlockLoserDataAccessException` (`CannotAcquireLockException` family) - order lock acquisition consistently (e.g. by id ascending).
- **SKIP LOCKED for work queues:** Hibernate supports it via `LockOptions.SKIP_LOCKED`, exposed on JPA as the lock-timeout hint value `-2` (Hibernate-specific meaning), giving `select ... for update skip locked` on PostgreSQL/MySQL 8+/Oracle. Workers grab distinct rows without blocking:
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "-2"))
List<Job> findTop10ByStatusOrderById(JobStatus s);      // then mark them RUNNING in the same tx
```
Combine with a status column; the lock alone doesn't survive commit. (Verify the generated SQL on your DB - support is dialect-dependent.)
- Pessimistic locks require a transaction; without one you get `TransactionRequiredException`.
- `select for update` on a query with `join fetch` locks rows in all joined tables (and Oracle rejects it with outer joins) - lock the root by id first, then load the rest.

**Choosing:** low conflict + web forms -> optimistic (+ retry); hot rows/counters/inventory with high conflict -> atomic `UPDATE ... SET x = x - ?` or pessimistic; queue consumption -> `SKIP LOCKED`; need cross-row invariants -> serializable isolation or explicit locks/constraints.

## 6.4 Isolation interplay (short)
JPA/Hibernate does not change the DB isolation; Spring's `@Transactional(isolation = ...)` sets it on the connection. READ COMMITTED (PostgreSQL/Oracle default) - non-repeatable reads across statements, lost updates without version/locks. REPEATABLE READ (MySQL default) - snapshot per tx; the **first-level cache** additionally makes a re-`find` return the cached object even if READ COMMITTED would see new data; a *query* re-reads rows but Hibernate keeps the old instance's values (2.1). SERIALIZABLE - DB may abort with serialization failure -> app must retry. Details of isolation levels: see `SQL_Database_Performance.md`.

## 6.5 Second-level cache (L2)
First-level cache = per EntityManager. **Second-level cache = per SessionFactory, shared across sessions**, storing *dehydrated entity state* (property arrays, not object instances) keyed by id, plus collection ids and (optionally) query results.

```
 find(Product,1)
   1. L1 (persistence context)?  hit -> return
   2. L2 (region "Product")?     hit -> hydrate new instance from cached state, put in L1
   3. DB select                  -> populate L2 and L1
```
Setup (Hibernate 6, JCache/Ehcache example):
```properties
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=jcache
spring.jpa.properties.hibernate.javax.cache.provider=org.ehcache.jsr107.EhcacheCachingProvider
spring.jpa.properties.hibernate.cache.use_query_cache=false      # only if you understand 6.5 below
```
(the `hibernate-jcache` module and a provider such as Ehcache 3 are required; Infinispan/Hazelcast have own region factories. `jakarta.persistence.sharedCache.mode` defaults to `ENABLE_SELECTIVE`: only entities annotated cacheable are cached.)
```java
@Entity
@Cacheable                                                     // jakarta.persistence
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Country { ... }
```
| Strategy | Semantics | Use for |
|---|---|---|
| `READ_ONLY` | never updated; throws if you try | reference data (countries, currencies) |
| `NONSTRICT_READ_WRITE` | no locking; small window of stale reads after update | rarely changed, staleness tolerable |
| `READ_WRITE` | soft locks; consistent with committed data within the node | mutable data, single node/cluster with a consistent provider |
| `TRANSACTIONAL` | full JTA-style transactional cache | needs JTA-capable provider; rare |

**Invalidation:** entity updates/deletes through Hibernate evict/update the entry at commit. **Direct SQL, native queries that modify data, other applications, DB triggers, and bulk operations bypass or over-invalidate** - stale data results. HQL bulk updates invalidate the affected entity region; native SQL updates don't unless you register the affected spaces / evict manually. Multi-node: use a distributed provider or accept per-node staleness.

**Collection cache** must be enabled separately (`@Cache` on the collection field) and caches only the *ids*; the elements must themselves be cached entities or each becomes a DB hit (N+1 in disguise).

**Query cache pitfalls:** caches (query string + parameters) -> list of ids; each returned entity is then re-fetched from L2 or DB. Any write to any table in the query spaces invalidates *all* cached results for those tables (timestamps region) - on write-heavy tables it thrashes and costs more than it saves. Enable per query via hint (`org.hibernate.cacheable`/`QueryHints.HINT_CACHEABLE`) only for stable, hot, read-mostly queries on cached entities.

**When NOT to use L2:** write-heavy or frequently changed data; large entities/collections; data updated by other systems or native SQL; multi-node without a coherent provider; when the real problem is N+1/bad queries (fix that first); when correctness requires the latest committed value (balances, inventory). Good candidates: small, reference, read-mostly tables and `@NaturalId` lookups. Measure with statistics: L2 hit/miss counts.

---

# 7. OSIV, LazyInitializationException, Spring Data JPA

## 7.1 Open Session In View
`spring.jpa.open-in-view` **defaults to `true` in Spring Boot** (with a startup warning `spring.jpa.open-in-view is enabled by default...`). `OpenEntityManagerInViewInterceptor` binds an `EntityManager` to the request thread at the start and closes it after the view is rendered.
```
request -> [EntityManager opened] -> controller -> @Transactional service (tx begin .. commit)
        -> DTO/entity mapping + JSON serialisation (lazy loads still work) -> [EntityManager closed]
```
The persistence context lives for the whole request, so lazy loading works in the controller and view layer. The widely reported production effect: with OSIV the database connection tends to stay tied up for (most of) the request rather than only for the transactional service call - lazy loads after the service returns run outside any transaction (autocommit) and re-acquire/keep a connection, and slow work in the controller (remote calls, serialisation) extends that window. Exact release timing depends on Spring/Hibernate connection-handling settings, so treat "connection held for the whole request" as the worst case to design against; the pool starvation it causes is real.

| Pros | Cons |
|---|---|
| Lazy loading "just works" in controllers/serialisers | Hides N+1: queries fire from the JSON layer, outside tx boundaries |
| Fewer `LazyInitializationException`s | Connection pool pressure; long request = long-held resources |
| | Data access in the view layer; loads happen in autocommit mode, possibly inconsistent snapshot |
| | Entities exposed in the API; security/coupling issues |

**Recommendation:** `spring.jpa.open-in-view=false`, and shape data in the service layer: load exactly what the use case needs (join fetch/entity graph/DTO projection) inside `@Transactional(readOnly = true)`, return DTOs/records, map in the service. Controllers then never touch lazy state; any missed fetch fails fast in tests instead of silently issuing queries.

## 7.2 LazyInitializationException (`could not initialize proxy - no Session`)
Cause: accessing an uninitialised proxy/collection when its session is closed (after tx end, outside service, after `detach`, in Jackson, in `@Async`/other thread).

Fixes, best to worst:
1. **Load what you need in the transaction**: `join fetch`, `@EntityGraph`, batch fetch, or a DTO projection.
2. **Map to DTO inside the service** (`@Transactional` method) before returning.
3. `Hibernate.initialize(order.getItems())` inside the tx (fine for one-off, but N+1 in loops).
4. Two-query fetch for collections + pagination.
5. Keep OSIV on (masks the problem).

Anti-patterns:
- Switching the mapping to `EAGER` globally (loads everything always, hidden N+1 via JPQL, cannot be undone per query).
- `spring.jpa.properties.hibernate.enable_lazy_load_no_trans=true` - each lazy access opens a *temporary session and connection*; N+1 with N connections; explicitly documented as an anti-pattern.
- Making the whole controller `@Transactional`.
- Catching the exception and returning `null`/empty.
- `FetchType.EAGER` + `@JsonIgnore` hacks; returning entities directly from controllers.

## 7.3 Spring Data JPA internals
**Repository bootstrapping:** an interface `OrderRepo extends JpaRepository<Order, Long>` -> at startup `JpaRepositoryFactory` creates a **JDK dynamic proxy** whose target is `SimpleJpaRepository<Order, Long>` (the default implementation of CRUD, holding a *shared, thread-safe EntityManager proxy* that resolves to the tx-bound EM per call). The proxy chain has interceptors: exception translation (`PersistenceExceptionTranslationInterceptor`), `TransactionInterceptor` (`SimpleJpaRepository` is annotated `@Transactional(readOnly = true)` at class level; write methods like `save/delete` override with `@Transactional`), then a `QueryExecutorMethodInterceptor` which routes declared query methods to `RepositoryQuery` objects and other calls to `SimpleJpaRepository`.

**Query method resolution (order):** (1) `@Query`, (2) named query `Entity.method` in `orm.xml`/`@NamedQuery`, (3) derived from the method name via `PartTree` (`findByLastNameAndAgeGreaterThanOrderByAgeDesc` -> `subject (find) + predicate parts (And/Or, operators) + OrderBy`) turned into a Criteria query at startup - **invalid property names fail application start**, not at runtime. Keywords: `Is/Equals`, `Like`, `Containing`, `In`, `Between`, `IgnoreCase`, `Top/First`, `Distinct`, `Exists`, `Count`, `Delete`.

**`save()`:** 
```java
@Transactional
public <S extends T> S save(S entity) {
    if (entityInformation.isNew(entity)) { em.persist(entity); return entity; }
    else                                  { return em.merge(entity); }
}
```
`isNew`: for an entity with a `@Version` of wrapper type -> new if version is `null`; otherwise new if the id is `null` (primitive id: `0`). **Assigned-id trap:** entities with a manually assigned id (UUID from constructor, natural keys) and no `@Version` look "not new" -> `merge` -> Hibernate first does `select ... where id=?` to see whether a row exists, *then* inserts: an extra SELECT for every insert and no batching benefit. Fixes: add a wrapper `@Version Long version` (null on new), or implement `Persistable<ID>` with an `isNew()` (typically a `@Transient` flag flipped by `@PostPersist/@PostLoad`), or use `em.persist` directly. On an update, `save` on a managed entity is pointless but harmless (merge of a managed entity is a no-op).

**`saveAll`** = loop of `save`, one tx by default. **`delete` semantics:**
| Method | SQL |
|---|---|
| `deleteById(id)` | `findById` (select) then delete; in Spring Data 3 a missing id is silently ignored |
| `delete(entity)` | merges/loads if detached, then `delete ... where id=?` (+ version check) |
| `deleteAll()` | `select *` then **one delete per entity** (N+1) - cascades and callbacks run |
| `deleteAllInBatch()` | single `delete from table` - bypasses persistence context/cascades/callbacks |
| `deleteAllByIdInBatch(ids)` / `deleteAllInBatch(entities)` | one `delete ... where id in (...)`/`or` statement |

**`existsBy`:** derived `existsByEmail` generates `select e.id from user e where e.email=? fetch first 1 rows only` (dialect-specific limit) instead of loading the entity or `count(*)`. `count` scans; `exists` short-circuits. `findById(id).isPresent()` loads the whole row.

**Pageable / Slice / Page cost:**
- `Page<T>` issues the data query **plus** a `select count(...)` (skipped when the first page is not full or the last page's size can be inferred). On big tables with complex joins the count dominates. Provide a `countQuery` that drops joins/fetches/ordering.
- `Slice<T>` fetches `size + 1` rows to know if a next page exists - no count query - best for infinite scroll/"next" UIs.
- `List<T> ... (Pageable)` returns a list with paging but no metadata.
- **Offset pagination degrades** with depth (`OFFSET 100000` still scans/discards 100000 rows). Prefer keyset (seek) pagination: `where (created_at, id) < (:ts, :id) order by created_at desc, id desc limit :n` (Spring Data 3.1+ has `ScrollPosition`/`Window` support - `Window<T> findFirst10By...(ScrollPosition)`; keep your own query if unsure).
- Sorting by a non-unique column without a tie-breaker gives unstable pages (rows duplicated/skipped): add `id` as the last sort key.

**Projections:**
| Kind | Example | Query | Notes |
|---|---|---|---|
| Interface, closed | `interface NameOnly { String getName(); }` | selects **only** those columns | no managed entities; best |
| Interface, open | `@Value("#{target.first + ' ' + target.last}")` | loads full entity | SpEL evaluated in memory - loses the optimisation |
| Class-based (DTO/record) | `record UserDto(String name, String email)` | constructor expression, selects only ctor args | must match ctor parameter names/types |
| Dynamic | `<T> List<T> findByActive(boolean a, Class<T> type)` | chosen by caller | one method, many shapes |
| Nested interface projections | | may cause entity/collection loading | check SQL |

**`@Query`:** JPQL by default; `nativeQuery = true` for SQL (bypasses portability; returns entities/`Tuple`/interface projections; **paging native queries needs `countQuery`**; native queries trigger full flush; `Sort` in native needs care). Named parameters (`:name` + `@Param`) over positional. Use `@Modifying` for DML (5.2).

**Specifications** (dynamic filters): `interface OrderRepo extends JpaRepository<Order, Long>, JpaSpecificationExecutor<Order>`; compose `Specification<Order>` with `and/or`. Combine with `Pageable`. Trap: no `join fetch` inside a spec that also paginates (in-memory paging warning; and count query with fetch join fails) - branch on `query.getResultType()` (`Long.class` for count) before adding fetches.

**`@EntityGraph` on repository methods** overrides the fetch plan for that method only: `@EntityGraph(attributePaths = {"customer","lines"})`. Multiple collections -> cartesian/`MultipleBagFetchException` (3.4).

**JPQL vs native vs Criteria:**
| | JPQL/HQL | Native SQL | Criteria API |
|---|---|---|---|
| Portability | high (entity model) | low | high |
| Power | joins on mapped associations, limited functions (HQL 6 much richer: CTEs, window functions, `union`, `lateral`-style features) | everything the DB offers | same as JPQL, programmatic |
| Type safety | strings (validated at startup for `@Query`) | none | good, verbose (with metamodel) |
| Best for | most queries | reporting, DB-specific features, hints, CTEs on old Hibernate | dynamic filters (or Specifications/QueryDSL) |

---

# 8. Soft delete, auditing, multitenancy, pool, schema management

## 8.1 Soft delete
```java
@Entity
@SQLDelete(sql = "update customer set deleted = true where id = ?")   // Hibernate: replaces the DELETE
@SQLRestriction("deleted = false")                                    // Hibernate 6.3+: replaces @Where
public class Customer { ... private boolean deleted; }
```
- `repo.delete(c)` now runs the `update`; every entity query (`find`, JPQL, and association loading) appends `and deleted=false`.
- `@Where` is deprecated in favour of `@SQLRestriction` (6.3+). On older 6.x use `@Where`.
- Traps: **unique constraints** (soft-deleted row with `email=x` blocks re-registering `x`: use a partial/filtered unique index `where deleted=false`, or include `deleted_at` in the key); native queries and `deleteAllInBatch`/bulk JPQL bypass the restriction; the `@SQLDelete` statement's param count/order must match; `@SQLRestriction` on an entity does **not** hide rows from joins written in native SQL; cascading soft-deletes to children requires annotating them too; you can't easily "see deleted" for admins - use `@Filter`/`@FilterDef` (enable per session) for optional visibility.
- Alternative: move rows to an archive table (trigger/CDC) keeps main tables lean.

## 8.2 Auditing
Spring Data auditing:
```java
@EnableJpaAuditing                                   // on a @Configuration class
@MappedSuperclass @EntityListeners(AuditingEntityListener.class)
public abstract class Auditable {
    @CreatedDate     @Column(updatable = false) private Instant createdAt;
    @LastModifiedDate private Instant updatedAt;
    @CreatedBy       @Column(updatable = false) private String createdBy;
    @LastModifiedBy  private String updatedBy;
}
// AuditorAware<String> bean supplies the current user (from SecurityContext)
```
Fills fields via JPA callbacks (`@PrePersist/@PreUpdate`). Bulk JPQL updates skip callbacks, hence skip auditing. For **full history (who changed what, old/new values)** use Hibernate Envers (`@Audited`, `_AUD` tables + `REVINFO`), or DB triggers/CDC. Callbacks must not call the EntityManager or touch other entities.

## 8.3 Multitenancy overview
- **Database per tenant / schema per tenant:** Hibernate `MultiTenantConnectionProvider` + `CurrentTenantIdentifierResolver` choose the connection/schema per request. Strong isolation, many pools/migrations to manage.
- **Discriminator column:** Hibernate 6 `@TenantId` on a column adds `tenant_id = ?` to reads and sets it on write, using the same tenant resolver. Cheapest to operate; isolation depends on never bypassing Hibernate (native SQL needs manual predicates); consider DB row-level security as a safety net.
- Caches (L2) and ids must be tenant-aware; batch jobs need explicit tenant context.

## 8.4 Connection pool interplay (HikariCP)
- Spring Boot's default pool is HikariCP. **`maximumPoolSize` defaults to 10**; `connectionTimeout` 30 s (threads wait for a connection, then `SQLTransientConnectionException: ... Connection is not available, request timed out after 30000ms`).
- Sizing rule of thumb (from the PostgreSQL wiki / HikariCP "About Pool Sizing"): `connections ~ (core_count * 2) + effective_spindle_count` for the **database server**, shared across *all* app instances. Small pools (10-20 per instance) with fast transactions beat big pools; more connections than DB cores usually means more contention, not throughput. Compute: `sum(instances x maximumPoolSize) <= DB max_connections budget`.
- **Long transactions starve the pool**: a tx holds its connection from first statement (or begin) to commit. Remote HTTP calls, file I/O, `Thread.sleep`, or big loops inside `@Transactional` multiply hold time. Little's law: connections needed ~ arrival rate x average hold time. Pool timeouts at load are typically a *hold-time* problem, not a pool-size problem.
- OSIV and `@Transactional` on controllers extend hold time (7.1).
- Diagnose with Hikari metrics (`hikaricp.connections.active/pending/timeout`), `leakDetectionThreshold` (logs stack of connections held too long), DB `pg_stat_activity` "idle in transaction".
- Batch jobs: give them a separate pool/datasource so they can't starve web traffic.

## 8.5 Schema management: Flyway/Liquibase vs `ddl-auto`
`spring.jpa.hibernate.ddl-auto`: `none`, `validate` (check entities vs schema, fail on mismatch), `update` (additive, best-effort), `create`, `create-drop`.
- **Production: `none` or `validate` + versioned migrations (Flyway `V1__init.sql`, `V2__add_col.sql` / Liquibase changelogs).** `update` cannot rename/drop, can't handle data migrations, is non-repeatable, and can lock big tables unpredictably; `create` in prod wipes data (a classic incident).
- Hibernate-generated DDL is a good *starting draft*: `jakarta.persistence.schema-generation.scripts.action=create` writes it to a file, review and put it in a migration.
- Migrations run before JPA validation in Boot (Flyway runs first). Test them against a real DB (Testcontainers), not H2 in "compatibility mode".
- Add indexes for FKs and frequent filters yourself - JPA does not create FK indexes (except PK/unique).

---

# 9. Observability

## 9.1 Logging SQL and bind parameters
```properties
# Preferred: logger-based (goes through logging framework, respects levels/format)
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.orm.jdbc.bind=TRACE          # Hibernate 6 bind values
# Hibernate 5 used: logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
spring.jpa.properties.hibernate.format_sql=true          # multi-line; dev only
```
- `spring.jpa.show-sql=true` (`hibernate.show_sql`) prints to **System.out**, bypasses the logging framework (no levels, no MDC, no file appender control) and never shows bind values - use it only for a quick look, never in production.
- Binds: the `?` in the logged SQL are values in the bind logger lines: `binding parameter (1:BIGINT) <- [42]`. Trace level can leak PII: dev/test only.
- **p6spy / datasource-proxy:** wrap the `DataSource` to log the final SQL *with values*, timing, and batch info at the JDBC layer (they show what the driver got, including batches). `datasource-proxy` can also count queries per test (`assertSelectCount(1)`) - ideal to lock in N+1 fixes in CI.

## 9.2 Hibernate statistics
```properties
spring.jpa.properties.hibernate.generate_statistics=true
logging.level.org.hibernate.stat=DEBUG
```
Gives per-session summary (statements prepared, entities loaded/fetched, collections fetched, flushes, L2 hits/misses/puts, query cache hits) and `SessionFactory.getStatistics()` (slowest query, `getPrepareStatementCount()`, `getEntityFetchCount()`, `getCollectionFetchCount()`, `getFlushCount()`). Overhead: turn on in test/staging, or sample in prod; expose via Micrometer (`hibernate.query.executions` etc. appear when statistics are enabled).

Session metrics log line shape (typical): `... JDBC statements: N; flushes: M; L2C hit count: X; query executions: Y`. If statements >> entities loaded, suspect N+1.

## 9.3 Verify a query plan
Copy the logged SQL to `EXPLAIN (ANALYZE, BUFFERS)` (PostgreSQL) / `EXPLAIN ANALYZE` (MySQL 8). Confirm index usage, rows actually scanned, and whether the DB executes `IN` lists efficiently. Hibernate can also add SQL comments (`hibernate.use_sql_comments=true`) to trace which HQL produced which SQL.

---

# 10. Worked examples (typical SQL)

## 10.1 Domain
```java
@Entity @Table(name = "orders")
class Order {
    @Id @GeneratedValue(strategy = SEQUENCE, generator = "orders_seq")
    @SequenceGenerator(name = "orders_seq", sequenceName = "orders_seq", allocationSize = 50)
    Long id;
    @ManyToOne(fetch = LAZY, optional = false) @JoinColumn(name = "customer_id") Customer customer;
    @OneToMany(mappedBy = "order", cascade = ALL, orphanRemoval = true) Set<OrderLine> lines = new HashSet<>();
    @Enumerated(EnumType.STRING) Status status;
    @Version long version;
    // addLine/removeLine helpers as in 4.1
}
```

## 10.2 Create with cascade
```java
@Transactional
Long place(Long customerId, List<LineDto> dto) {
    Order o = new Order();
    o.setCustomer(em.getReference(Customer.class, customerId));     // no SQL
    dto.forEach(d -> o.addLine(new OrderLine(d.sku(), d.qty())));
    em.persist(o);                                                  // cascades to lines
    return o.getId();
}
```
Typical output:
```
select nextval('orders_seq')            -- only when the id block is used up (plus one for order_line_seq likewise)
insert into orders (customer_id,status,version,id) values (?,?,?,?)
insert into order_line (order_id,qty,sku,id) values (?,?,?,?)       -- once per line (batched if batch_size>1 and SEQUENCE ids)
insert into order_line (order_id,qty,sku,id) values (?,?,?,?)
```
No SELECT for the customer because `getReference` gave a proxy; the FK value comes from the proxy's id.

## 10.3 Update by dirty checking and version
```java
@Transactional void ship(Long id) { Order o = repo.findById(id).orElseThrow(); o.setStatus(SHIPPED); }
```
```
select o1_0.id, o1_0.customer_id, o1_0.status, o1_0.version from orders o1_0 where o1_0.id=?
update orders set customer_id=?, status=?, version=? where id=? and version=?      -- at commit; all columns; version 0 -> 1
```
(`findById` on an entity whose `customer` is LAZY leaves `customer` a proxy - no join.)

## 10.4 Removing a child with orphanRemoval
```java
@Transactional void dropLine(Long orderId, Long lineId) {
    Order o = orderRepo.findWithLines(orderId);            // join fetch lines
    o.getLines().removeIf(l -> l.getId().equals(lineId));  // or o.removeLine(l)
}
```
```
select ... from orders o1_0 left join order_line l1_0 on o1_0.id=l1_0.order_id where o1_0.id=?
delete from order_line where id=?           -- orphan removal at flush
update orders set ..., version=? where id=? and version=?   -- collection change bumps parent's version too
```

## 10.5 N+1 before/after on the same code
Before (LAZY lines, JSON mapping touches `order.getLines()` for 20 orders on a page):
```
select ... from orders ... limit ?              -- 1
select ... from customer where id=?             -- x distinct customers (if EAGER/accessed)
select ... from order_line where order_id=?     -- x20
```
After (`default_batch_fetch_size=20` + projection for customer name):
```
select o.id, c.name, ... from orders o join customer c ... limit ?   -- 1  (DTO projection)
select ... from order_line where order_id in (?,?,...,?)             -- 1 (batch of 20)
```
Total statements: ~42 -> 2.

## 10.6 Bulk update staleness demo
```java
@Transactional void demo() {
    Product p = repo.findById(1L).get();                  // price = 10
    repo.raiseAll();                                       // @Modifying update ... set price = price*2  (no clearAutomatically)
    p.getPrice();                                          // still 10 (stale L1)
    repo.findById(1L).get().getPrice();                    // still 10! served from L1 - no SQL at all
}
```
With `clearAutomatically=true`: the second `findById` re-selects and returns 20. But also: `p` is now detached; later `p.setX()` is no longer saved.

## 10.7 Lost update prevented with `@Version` (two threads)
```
T1: select ... version=0 ; set balance ; commit -> update ... where id=1 and version=0  (1 row) -> version=1
T2: select ... version=0 ; set balance ; commit -> update ... where id=1 and version=0  (0 rows)
    -> OptimisticLockException -> Spring ObjectOptimisticLockingFailureException -> tx rolled back
```

## 10.8 Flush-order trap reproduced
```java
@Transactional void replace() {
    Coupon old = repo.findByCode("X1");     // unique(code)
    repo.delete(old);
    repo.save(new Coupon("X1"));
}                                            // commit
```
```
insert into coupon (code,id) values (?,?)   -- inserts run before deletes
-- ERROR: duplicate key value violates unique constraint "uk_coupon_code"
```
Fix: `repo.delete(old); repo.flush();` -> `delete` executes before the `insert`.

---

# 11. Case study: "Order history endpoint took 5 s -> 200 ms"

**Symptom.** `GET /customers/{id}/orders?page=0&size=50` p95 = 5.2 s (prod), CPU on DB low, app CPU moderate, pool `pending` spikes under load. Payload: orders with lines, product names, customer name, shipping address.

**Step 1 - measure (dev with prod-like data, statistics on):**
```
Session Metrics: 1 + 50 (lines) + 187 (products) + 50 (payments) + 1 (count) = 289 JDBC statements
entities loaded: ~4,900
```
Also a warning in logs: `HHH90003004 / HHH000104: firstResult/maxResults specified with collection fetch` -> the repository used `join fetch o.lines` with `Pageable`, loading *all* of a heavy customer's orders x lines (~12,000 rows) into memory and slicing in Java.

**Step 2 - causes found**
1. Collection `join fetch` + pagination (in-memory paging).
2. `Line.product` mapped `@ManyToOne` EAGER -> one select per distinct product (hidden N+1 through JPQL).
3. `payments` lazy touched by the mapper -> +50.
4. Entities returned to the mapper -> ~4,900 objects + snapshots, dirty check over all on commit (method was `@Transactional` non-readOnly).
5. OSIV on and the controller called an internal pricing service (HTTP, 80 ms) inside the request - connection tied longer.

**Step 3 - fixes (in order of payoff)**
| # | Change | Statements after |
|---|---|---|
| 1 | Page the roots only: `select o.id ... where o.customer.id=? order by o.createdAt desc, o.id desc` (keyset or offset) -> 1 query + count avoided (`Slice`) | 289 -> ~240 |
| 2 | Order-level DTO projection joined with customer: `select new OrderRow(o.id, o.createdAt, o.status, c.name) ...` | ~190 |
| 3 | Lines in one query: `select new LineRow(l.order.id, l.sku, p.name, l.qty, l.price) from OrderLine l join l.product p where l.order.id in :ids` (grouped in Java by orderId) | ~3 |
| 4 | Payments summary: one aggregate query `... group by order.id where order.id in :ids` | 4 |
| 5 | `@Transactional(readOnly = true)` on the service, DTOs returned, `open-in-view=false`, pricing call moved outside the transaction and cached | - |
| 6 | Added composite index `orders(customer_id, created_at desc, id desc)`; `order_line(order_id)` FK index confirmed via EXPLAIN | scan -> index |
| 7 | Set `Line.product` LAZY, `default_batch_fetch_size=32` as a safety net for other paths | - |

**Result (same dataset):** 289 -> 4 statements, 4,900 -> 0 managed entities (DTOs), p95 5.2 s -> ~190 ms, DB time dominated by one indexed query. Locked in with a `datasource-proxy` test asserting `select count <= 4`. (Figures are an illustrative case, not measured output from this repo.)

**Lesson:** the win came from *query shape + projection + tx scope*, not from the cache.

---

# 12. Production war stories (symptom -> diagnosis -> fix)

1. **"Nightly import takes 6 hours, DB idle."** Symptom: 200k inserts, one round trip each, app slow. Diagnosis: `GenerationType.IDENTITY` + no `batch_size`; single huge tx with growing persistence context (O(n^2) dirty checks). Fix: SEQUENCE(50), `batch_size=50`, `order_inserts`, chunk `flush()+clear()` every 50, or `StatelessSession`/`JdbcTemplate.batchUpdate`. -> 8 min.
2. **"Random duplicate key on unique email after 'update'."** Diagnosis: code deleted the old row and inserted the new in one tx: insert-before-delete ordering. Fix: flush after delete, or update in place.
3. **"Prices in report are wrong after bulk discount."** Diagnosis: `@Modifying` update without `clearAutomatically`; later code read stale entities from the persistence context. Fix: `clearAutomatically=true, flushAutomatically=true` or separate transactions.
4. **"App freezes under load; `Connection is not available, request timed out after 30000ms`."** Diagnosis: `pg_stat_activity` full of `idle in transaction`; service methods `@Transactional` around a payment-gateway HTTP call (2 s); pool size 10. Fix: shrink transaction to DB work only, call gateway outside, keep pool modest, add Hikari timeouts and leak detection, alert on `pending`.
5. **"Endpoint fine in dev, OOM in prod."** Diagnosis: `join fetch` collection + `Pageable` -> HHH in-memory pagination on a large table. Fix: two-query pagination.
6. **"Entity is inserted twice after retry."** Diagnosis: `save()` with assigned UUID id and no `@Version` -> merge path, plus a retry re-ran non-idempotent logic. Fix: idempotency key with unique constraint, `Persistable.isNew`.
7. **"Users see other users' cached data."** Diagnosis: query cache + L2 enabled for an entity updated via a native SQL job; stale until region eviction. Fix: disable cache for that entity, or evict after job, or route writes through Hibernate.
8. **"Sudden `MultipleBagFetchException` after adding a second `List` association."** Diagnosis: a repository method with two `join fetch` on Lists. Fix: two queries or `@BatchSize`; do not just switch to `Set`.
9. **"`LazyInitializationException` only in the scheduled job / `@Async` method."** Diagnosis: no OSIV outside web requests; entity passed across threads. Fix: pass ids, reload inside a transactional method, use DTOs.
10. **"Order total off by one update under concurrency."** Diagnosis: read-modify-write on `total` without version/lock. Fix: `@Version` + retry, or atomic `update ... set total = total + ?`.
11. **"Deleting one student deleted courses."** Diagnosis: `cascade = ALL` on `@ManyToMany`. Fix: cascade only PERSIST/MERGE (or none) on many-to-many; never REMOVE.
12. **"After enum refactor all statuses are wrong."** Diagnosis: `EnumType.ORDINAL` default; a constant inserted mid-enum. Fix: STRING, data migration.
13. **"Prod tables wiped after a deploy."** Diagnosis: `ddl-auto=create` (or `create-drop`) leaked into the prod profile from a dev config. Fix: Flyway/Liquibase + `ddl-auto=validate`/`none` in prod, profile-specific properties, DB user without DDL rights for the app.
14. **"Extra selects appeared everywhere after adding a bidirectional `@OneToOne`."** Diagnosis: the inverse (`mappedBy`) side cannot be lazy, so every load of the owner also queries the other table; plus untouched `@ManyToOne` EAGER defaults. Fix: explicit LAZY on the FK side, `@MapsId` shared-PK design, statement-count test guard.
15. **"Page 500 of the list shows duplicates and misses rows."** Diagnosis: sort on non-unique column without tie-breaker + offset. Fix: add `id` tie-breaker; keyset pagination.

---

# 13. Interview questions (50+) - Easy / Medium / Hard

Format: **Q** -> model answer -> *Follow-ups* -> Wrong answer to avoid.

## Easy

**Q1. JPA vs Hibernate vs Spring Data JPA?**
JPA = specification (API + JPQL + mapping annotations); Hibernate = a provider implementing it (plus extras like `@BatchSize`, `@NaturalId`); Spring Data JPA = repository abstraction generating implementations over `EntityManager`.
*Follow-up:* Can you use Hibernate without JPA? Yes, native `Session`/`SessionFactory` API. Can you swap Hibernate for EclipseLink? Yes if you avoid Hibernate-only features.
Wrong: "Spring Data JPA is an ORM."

**Q2. What is the first-level cache?**
The persistence context of an `EntityManager`: identity map of managed entities + snapshots. Always on, cannot be disabled, scoped to the tx (or request with OSIV). `find` by id checks it first; queries do not skip the DB but reuse existing instances.
*Follow-ups:* Is it shared between threads? No, one EM per unit of work, not thread-safe. Why does a second `find` fire no SQL? Identity map hit.
Wrong: "It caches query results."

**Q3. List the entity states.**
Transient, managed (persistent), detached, removed. Transitions in 2.2.
*Follow-up:* What state is an entity after `save` returns from a *merge*? The *returned* instance is managed; the argument stays detached.

**Q4. What does `@Transactional` have to do with Hibernate flush?**
The commit triggers flush; without a tx, managed changes are never flushed (no persistent context lifecycle unless OSIV).
*Follow-up:* What if the method is `private` or self-invoked? Proxy not applied, no tx (see `Spring_Transactional.md`).

**Q5. Default fetch types?**
`@ManyToOne`/`@OneToOne` EAGER; `@OneToMany`/`@ManyToMany`/`@ElementCollection` LAZY.
*Follow-up:* Why change ManyToOne to LAZY? Avoid loading graphs you don't need; EAGER is not overridable per query.

**Q6. `@Entity` requirements?**
Non-final class, no-arg constructor (public/protected), an `@Id`, no final persistent fields/methods you need proxied.

**Q7. What is `mappedBy`?**
Marks the inverse side of a bidirectional association; the owning side (with the FK) is the one that writes. Without keeping both sides in sync, FK may not be set.
Wrong: "mappedBy means the field is lazy."

**Q8. Why `EnumType.STRING`?**
ORDINAL stores position; reordering/inserting constants silently corrupts data.

**Q9. What does `@Version` do?**
Optimistic locking: `update ... where id=? and version=?`; zero rows -> `OptimisticLockException`.

**Q10. What is `LazyInitializationException`?**
Accessing an uninitialised proxy/collection outside an open session. Fix by fetching what you need inside the tx (join fetch/entity graph/DTO) - not by EAGER or `enable_lazy_load_no_trans`.
*Follow-up:* Does OSIV fix it? It hides it (7.1).

**Q11. `show-sql` vs logger?**
`show-sql` writes to stdout without binds; `logging.level.org.hibernate.SQL=DEBUG` + `org.hibernate.orm.jdbc.bind=TRACE` (Hibernate 6) uses the logging framework.

**Q12. `save()` vs `saveAndFlush()`?**
`saveAndFlush` also flushes immediately (executes pending SQL now) - use for constraint errors or when a native query needs to see the data.

**Q13. `findById` vs `getReferenceById`?**
Select now vs lazy proxy without SQL (`EntityNotFoundException` on later access if absent). Use reference for setting FKs.

**Q14. What is JPQL?**
Object-oriented query language over entities/attributes, translated to SQL per dialect.

## Medium

**Q15. Explain the N+1 problem and three fixes.**
1 query for roots + N lazy loads. Fixes: `join fetch`/`@EntityGraph`, batch fetching (`@BatchSize`, `default_batch_fetch_size`), DTO projection; subselect for load-all cases. Each has limits (3.3).
*Follow-ups:* Which fix works with pagination? Batch fetching, ToOne fetch joins, two-query. How do you detect it in CI? Query-count assertions (datasource-proxy). Does making it EAGER fix it? No - JPQL then issues N extra selects.
Wrong: "Use `FetchType.EAGER`."

**Q16. When exactly does Hibernate flush in AUTO mode?**
Before commit, before a query that reads tables with pending changes (native queries: everything), and on explicit `flush()`. Not on `find`.
*Follow-up:* What if I set `FlushMode.COMMIT`? Queries may not see unflushed changes - risk of stale reads within the tx.

**Q17. How does dirty checking work and what does it cost?**
Snapshot at load; property-wise compare of all managed entities at flush -> O(entities x props) per flush. Reduce: `readOnly=true` (MANUAL flush), smaller contexts (`clear()`), projections, `StatelessSession`.
*Follow-up:* Does `@DynamicUpdate` speed up dirty checking? No - it changes the SQL `set` list, not the comparison.
Wrong: "Hibernate calls a setter hook."

**Q18. What does `@Transactional(readOnly = true)` really do?**
Spring sets Hibernate flush mode to MANUAL and marks the connection read-only (driver/DB may optimise or route). Skips commit-time dirty check; doesn't forbid explicit writes.
*Follow-up:* Does it prevent updates at DB level? Only when the DB/driver enforces read-only connections; not guaranteed.

**Q19. persist vs merge vs save.**
`persist`: transient -> managed, void, fails for detached. `merge`: copies onto a managed instance and returns it. `save` (Spring Data) = `persist` if `isNew` else `merge`.
*Follow-up:* Why an extra SELECT before insert sometimes? Assigned id + no version -> `merge` path -> existence select. Fix with `@Version` wrapper or `Persistable`.

**Q20. Why does IDENTITY disable batching?**
The id is known only after the INSERT runs, so Hibernate must execute each insert at `persist` time to populate the id, leaving nothing to batch. SEQUENCE lets ids be pre-allocated in memory.
*Follow-up:* MySQL has no sequences - options? Keep IDENTITY and accept, use UUIDs/app-assigned ids, a table-emulated generator, or JDBC batch inserts outside Hibernate.

**Q21. Explain `allocationSize` and the pooled optimizer.**
One `nextval` reserves a block of `allocationSize` ids used from memory; sequence `INCREMENT BY` must equal `allocationSize`, else duplicate ids or gaps. Ids after restart show gaps.
*Follow-up:* Why default 50? Fewer round trips; costs id gaps.

**Q22. `MultipleBagFetchException` - cause and correct fix?**
Two `List` (bag) collections join-fetched together. Correct fix: fetch collections in separate queries or batch-fetch one. Switching to `Set` removes the exception but keeps the cartesian product.
*Follow-up:* How many rows for 10 orders x 5 items x 4 payments? 200.

**Q23. Why is `join fetch` + `Pageable` dangerous?**
Limit cannot be applied to joined rows, so Hibernate paginates in memory after loading everything (HHH000104 in 5.x; a similar "firstResult/maxResults specified with collection fetch" warning in 6.x). Use two-query pagination.
*Follow-up:* Is `join fetch` of a ManyToOne with paging OK? Yes.

**Q24. Bidirectional association hygiene?**
Owning side writes FK; use `addX/removeX` helpers to sync both; exclude associations from `toString/equals/hashCode`; `mappedBy` on inverse; `orphanRemoval`+cascade for aggregates.

**Q25. `cascade=REMOVE` vs `orphanRemoval`.**
REMOVE: deleting the parent deletes children. orphanRemoval: de-referencing a child from the collection deletes it (and, on parent delete, also deletes children). Never REMOVE/ALL on `@ManyToMany`/`@ManyToOne`.
*Follow-up:* What error appears when replacing the collection object? "A collection ... was no longer referenced by the owning entity instance" - mutate in place.

**Q26. `@ManyToMany` with an extra column?**
Not possible: introduce a link entity with two `@ManyToOne` and (usually) `@EmbeddedId` with `@MapsId`, plus `@OneToMany(mappedBy)` on both sides.

**Q27. How would you write entity `equals/hashCode`?**
Business key when a stable one exists (or `@NaturalId`); otherwise id-based `equals` guarded by `getId() != null` and `instanceof`, with constant `hashCode`; or assign UUID at construction. Never Lombok `@Data`, never include lazy associations.
*Follow-ups:* Why not id-based hashCode? Changes after persist, breaking `HashSet` membership. Why `instanceof` not `getClass()`? Proxies.

**Q28. Bulk JPQL update vs entity update - what's the catch?**
Bulk bypasses persistence context, version, callbacks, cascades - loaded entities go stale. Use `flushAutomatically`+`clearAutomatically` or reload/refresh.

**Q29. How do you avoid the extra SELECT on `save()` with assigned ids?**
`@Version` (wrapper type) or `Persistable<ID>.isNew()`, or `persist` directly via `EntityManager`.

**Q30. `deleteAll()` vs `deleteAllInBatch()`?**
`deleteAll()` selects all then deletes one by one, honouring cascades/callbacks; `deleteAllInBatch()` issues one `delete from` statement and skips them.

**Q31. `Page` vs `Slice`.**
Page runs an extra count query; Slice fetches `size+1` and has no total. Prefer Slice/keyset for large data or infinite scroll.

**Q32. Interface vs class vs dynamic projection.**
Closed interface and class DTO select only needed columns; open interface (SpEL) loads whole entity; dynamic lets the caller pass a `Class<T>`. Projections produce non-managed objects (no dirty checking).

**Q33. Inheritance strategies trade-offs.**
SINGLE_TABLE fastest but nullable columns; JOINED normalised with joins; TABLE_PER_CLASS unions and poor polymorphism; `@MappedSuperclass` for reuse only.

**Q34. OSIV - what and why disable?**
Keeps the EM open through the view so lazy loading works; downsides: connection held longer/pool starvation, N+1 hidden in serialisation, entity exposure. Disable and return DTOs.
*Follow-up:* After disabling, controller returns entity with lazy field - what happens? `LazyInitializationException` (or Jackson error) - good, it flags missing fetch planning.

**Q35. How do you log bind parameters in Hibernate 6?**
`logging.level.org.hibernate.orm.jdbc.bind=TRACE` (5.x: `BasicBinder`); or p6spy/datasource-proxy.

## Hard

**Q36. Walk through what happens at `flush()`.**
Flush-time cascade -> dirty check all managed entities -> collection checks -> ordered ActionQueue execution (orphan removals, inserts, updates, collection ops, deletes) -> JDBC batches -> snapshot refresh. (2.3, 2.5)
*Follow-ups:* Why does the unique swap fail? Inserts before deletes. What if a query runs mid-way? Auto flush before it if the tables overlap. Does flush commit? No.

**Q37. Explain the lost-update anomaly and how JPA prevents it.**
Two txs read the same row and both write based on the stale read (6.1). `@Version` compare-and-set; `PESSIMISTIC_WRITE` `select for update`; or atomic SQL update. Isolation alone (READ COMMITTED) does not prevent it.
*Follow-ups:* How to retry safely? Outside the tx boundary, whole unit of work, bounded with backoff, idempotent. Which do you pick for a hot inventory counter? Atomic conditional update; pessimistic if invariants span rows.
Wrong: "Use `synchronized`" (fails across instances) / "Use SERIALIZABLE everywhere."

**Q38. How does a Hibernate proxy work and what breaks?**
ByteBuddy subclass intercepting calls, loading the target on first non-id access. Breaks: `final` classes/methods, `getClass()` comparisons, `instanceof` on subtype, direct field access, serialisation after session close. Use `Hibernate.unproxy`, `instanceof`, getters. (2.6)
*Follow-up:* Why can't the inverse side of a OneToOne be lazy? Hibernate must query to know null vs proxy.

**Q39. SKIP LOCKED queue with JPA?**
Use `PESSIMISTIC_WRITE` with lock-timeout hint `-2` (Hibernate SKIP_LOCKED) -> `select ... for update skip locked` (PostgreSQL/MySQL 8/Oracle); claim rows and mark status in the same tx; keep it short. Verify SQL per dialect; `LIMIT` with locking semantics differ.
*Follow-ups:* What if the worker crashes after commit but before processing? Status column + visibility timeout/heartbeat. Ordering guarantees? None strict.

**Q40. Design correct pagination for orders with lines.**
Page order ids (stable sort + tie-breaker, keyset if deep), then fetch lines for those ids (second query or batch fetch), reassemble order; or DTO projections. Avoid in-memory pagination. (3.5)
*Follow-up:* How to keep the order of ids after `IN`? Sort in Java by the id list.

**Q41. When would you use a second-level cache, and what can go wrong?**
Read-mostly small reference data / natural-id lookups. Risks: stale data from native/bulk/external writes, query cache invalidation storms, collection cache only ids, multi-node coherence, complexity masking N+1. Choose READ_ONLY/NONSTRICT/READ_WRITE per staleness tolerance. (6.5)
*Follow-up:* How do you measure benefit? Statistics: L2 hit ratio versus DB calls saved; load test.

**Q42. `saveAll` of 100k entities is slow - diagnose.**
Check id strategy (IDENTITY), `batch_size`, `order_inserts`, one giant persistence context (snapshots, flush cost), statement logging overhead, `rewriteBatchedStatements`/`reWriteBatchedInserts`, index/trigger cost, FK checks. Fix: SEQUENCE + batch + chunked `flush/clear`, or JDBC/COPY.
*Follow-up:* How do you prove batching works? Statistics/`BatchingBatch` logs/proxy, DB call counts.

**Q43. Why can a query in the middle of a loop make a batch job O(n^2)?**
Each query triggers auto flush; each flush dirty-checks the whole context, which keeps growing. Fix: `clear()` periodically, use `FlushMode.COMMIT`/MANUAL carefully, read-only queries/projections, or stateless sessions.

**Q44. Explain `Persistable`/`isNew` and how `save` chooses persist vs merge.**
`JpaMetamodelEntityInformation.isNew`: version-attribute wrapper null -> new; else id null (or primitive 0) -> new; entities implementing `Persistable` use `isNew()`. (7.3)

**Q45. Compare fixes for `LazyInitializationException`; which are anti-patterns?**
Good: fetch plan in query/entity graph, DTO mapping in service, initialize in tx, projections. Anti-patterns: global EAGER, `enable_lazy_load_no_trans`, OSIV as a crutch, `@Transactional` controllers, swallowing the exception.

**Q46. Design soft delete correctly.**
`@SQLDelete` + `@SQLRestriction` (6.3+; `@Where` earlier), partial unique indexes for uniqueness among live rows, care with native/bulk queries, cascade to children, admin visibility via `@Filter`, audit deletion timestamp/user. Consider archival instead.

**Q47. Why `@ElementCollection` List is risky and what to do?**
No row identity: removing one element usually deletes all rows and re-inserts; can't query/update individual rows efficiently. Use a Set for a value collection or a real child entity with `orphanRemoval`.

**Q48. Explain HikariCP sizing and its link with transactions.**
Pool ~ small (DB-cores-based rule (cores x 2) + spindles as start, then measure). Needed connections = throughput x hold time; long `@Transactional` methods (remote calls, OSIV) inflate hold time, so fix hold time before growing the pool. Monitor pending threads and `idle in transaction`.

**Q49. How does Spring Data resolve a repository method and what fails at startup vs runtime?**
Order: `@Query` -> named query -> `PartTree` derivation. Property-name errors and JPQL syntax errors in `@Query` fail at startup (query is validated/parsed on bootstrap); data-dependent/native errors surface at runtime.

**Q50. `@Modifying @Query("delete ...")` vs `deleteBy...` derived - behaviours?**
Derived `deleteByX` loads entities then removes each (callbacks, cascades honoured, N statements); `@Modifying` delete is a single statement bypassing the context.

**Q51. Explain Hibernate's ordering problems with `order_inserts` and cascade.**
With cascade `ALL` from parent to child and batching, inserts interleave parent/child, splitting batches; `order_inserts=true` groups by entity type (respecting FK dependencies) restoring batch sizes.

**Q52. What isolation does Hibernate use? How does L1 interplay with isolation?**
Whatever the connection has (DB default or Spring `isolation`); L1 makes repeated `find` return cached state regardless of what committed elsewhere, so even under READ COMMITTED a re-`find` doesn't see other txs' changes; queries re-read rows but keep existing instances' values.

**Q53. How would you verify the `@Version` retry works in a test?**
Two threads/`CountDownLatch`, both load then update; assert one succeeds and one throws `ObjectOptimisticLockingFailureException`; or load twice in two transactions with `TransactionTemplate` sequentially, mutate the stale copy and expect failure.

**Q54. Multi-tenancy strategies?**
DB-per-tenant, schema-per-tenant (connection provider + tenant resolver), discriminator column (`@TenantId` in Hibernate 6). Trade isolation vs operations cost; watch native SQL, caches, batch jobs.

**Q55. How do you handle timezones?**
Store instants in UTC (`Instant`/`OffsetDateTime`, `timestamp with time zone`), set `hibernate.jdbc.time_zone=UTC`, convert at the edges, keep `LocalDateTime` only for zone-less wall-clock values.

## Common wrong answers (quick list)
- "`@Transactional(readOnly=true)` makes DB reject writes" - not guaranteed.
- "Hibernate updates the row when the setter is called" - at flush.
- "`Set` fixes MultipleBagFetchException" - only hides it, cartesian remains.
- "`FetchType.EAGER` fixes N+1 / lazy exceptions" - makes it worse.
- "`show_sql` is fine in prod" - no binds, stdout, overhead.
- "`save()` is needed to persist changes to a managed entity" - not needed.
- "L2 cache speeds everything" - stale data and invalidation costs; fix queries first.
- "OSIV is a Hibernate feature" - it is a Spring web-layer interceptor/filter.
- "`deleteAll()` issues one DELETE" - it issues N.
- "UUID PKs are always better" - index bloat/fragmentation with random v4.

---

# 14. One-page cheat sheet

```
MODEL      EM = persistence context (identity map + snapshots + ActionQueue). Mutate managed objects; flush writes SQL.
STATES     new -> persist -> managed -> remove -> removed ; detach/clear/close -> detached ; merge returns managed COPY
FLUSH      commit | before query on affected tables (native: all) | explicit.   flush != commit
ORDER      orphan removals, inserts, updates, collection ops, DELETES LAST  (unique swap -> flush after delete)
DIRTY      snapshot compare O(n x props) per flush.  readOnly=true -> MANUAL flush.  @DynamicUpdate changes SET list only
PROXY      ByteBuddy subclass; non-final entity; getReference = no SQL; use instanceof/Hibernate.unproxy; inverse OneToOne is eager
DEFAULTS   ToOne EAGER (set LAZY) | ToMany LAZY
N+1        fix: join fetch / @EntityGraph / @BatchSize or default_batch_fetch_size / subselect / DTO projection
COLLECTIONS  one bag fetch per query; Set does NOT remove cartesian; collection fetch + paging = IN-MEMORY (page ids first)
ASSOC      owning side = FK; mappedBy = inverse; addX/removeX helpers; no cascade REMOVE on ManyToMany/ManyToOne
EQUALS     business key | id!=null && id.equals + constant hashCode | assign UUID early
IDS        IDENTITY = no batching | SEQUENCE(allocationSize=50, DB increment must match) | UUID: prefer time-ordered
BATCH      hibernate.jdbc.batch_size=50, order_inserts, order_updates, SEQUENCE ids, flush()+clear() per chunk, verify via stats
BULK       @Modifying(clearAutomatically, flushAutomatically); bypasses PC/version/callbacks
LOCKING    @Version -> where version=?; ObjectOptimisticLockingFailureException; retry outside tx
           PESSIMISTIC_WRITE -> for update; lock.timeout hint; -2 = SKIP LOCKED (Hibernate); short tx; lock order
LOST UPDATE  read-modify-write w/o version|lock; READ COMMITTED does not stop it
L2 CACHE   @Cacheable + @Cache(strategy) + region factory; READ_ONLY for reference data; stale on native/bulk; query cache = last resort
OSIV       spring.jpa.open-in-view default TRUE; set false; DTOs from service in @Transactional(readOnly=true)
LIE        fix by fetch plan / DTO in tx.  NOT: global EAGER, enable_lazy_load_no_trans
SPRING DATA  proxy -> SimpleJpaRepository | save = persist if new else merge (assigned id -> extra SELECT; use @Version/Persistable)
           deleteAll = N deletes ; deleteAllInBatch = 1 | Page = +count ; Slice = size+1 | existsBy > count/findById
           projections: closed interface/DTO select cols; open (SpEL) loads entity
INHERIT    SINGLE_TABLE (fast, nullable) | JOINED (joins) | TABLE_PER_CLASS (unions, avoid) | @MappedSuperclass (reuse)
ENUM       STRING not ORDINAL | TIME: Instant/OffsetDateTime, UTC, jdbc.time_zone=UTC | money: BigDecimal
SOFT DEL   @SQLDelete + @SQLRestriction (6.3+; @Where before) ; partial unique index
POOL       Hikari default 10; size small ((cores*2)+spindles rule); needed = rate x hold time; kill long tx
SCHEMA     Flyway/Liquibase + ddl-auto=validate/none in prod
LOGGING    org.hibernate.SQL=DEBUG ; org.hibernate.orm.jdbc.bind=TRACE ; generate_statistics ; p6spy/datasource-proxy ; not show_sql
CHECKLIST  1 count statements per endpoint  2 LAZY everywhere  3 DTOs for reads  4 readOnly tx  5 batch config
           6 indexes on FKs/filters  7 keyset pagination  8 short transactions  9 OSIV off  10 EXPLAIN the top queries
```

**Performance checklist (endpoint slow?)**
1. Turn on SQL + statistics; count statements per request (expect a handful, not hundreds).
2. Find N+1 and hidden EAGER; fix with the right fetch plan or projection.
3. Look for in-memory pagination warnings and cartesian products.
4. Confirm indexes with EXPLAIN (FK columns, filter+sort columns).
5. Shrink transactions; no remote calls inside; `readOnly=true` for reads; OSIV off.
6. Bound the persistence context (`clear()`, streaming, stateless) for big jobs; enable batching properly.
7. Only then consider caching (L2/app-level) - with an invalidation story.
