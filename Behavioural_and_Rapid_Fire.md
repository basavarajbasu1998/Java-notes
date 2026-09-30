# Behavioural Questions, Study Plan, Rapid-Fire Answers
Prepare a **story** (STAR: Situation, Task, Action, Result) for each:
1. Toughest production bug you fixed (how you found root cause).
2. A performance improvement you made (numbers: 3 s → 400 ms).
3. A design decision you owned and its trade-off.
4. A disagreement with a teammate / how you handled review comments.
5. How you mentor juniors, do code reviews, keep code quality.
6. Something you'd do differently in your last project.

---

# Study Order (suggested 4 weeks)
| Week | Topics |
|---|---|
| 1 | `Spring_Transactional.md`, `SpringBoot_Internals.md`, `REST_API_Design.md` |
| 2 | `Java_Memory_Model_and_Locks.md`, `ThreadPoolExecutor.md`, revise the Multithreading section of `JavaFullstack.md` |
| 3 | `SQL_Database_Performance.md`, `Kafka.md`, `Production_Debugging.md` |
| 4 | `Testing_JUnit_Mockito.md`, `Modern_Java_11_to_21.md`, `Core_Java_Deep_Points.md`, `System_Design_Scenarios.md`, behavioural stories + mock interviews |

## Rapid-fire answers to memorise
- **Why proxy in Spring?** To add cross-cutting behaviour (tx, security, caching) without changing your class.
- **`@Component` vs `@Bean`?** Class-level, scanned automatically vs method-level in `@Configuration`, for third-party classes.
- **Singleton bean thread-safe?** No – keep beans stateless.
- **`@Autowired` constructor vs field injection?** Constructor: immutable, testable, fails fast. Preferred.
- **Circular dependency?** A needs B, B needs A → redesign, or `@Lazy`; Boot 2.6+ forbids by default.
- **`@Primary` vs `@Qualifier`?** default choice vs explicit choice among multiple beans.
- **Lazy vs Eager fetch?** Lazy loads on access (LazyInitializationException risk), eager loads at once.
- **HashMap vs ConcurrentHashMap under threads?** HashMap can corrupt/loop; CHM per-bucket locking + CAS.
- **`List.of()` vs `Arrays.asList()`?** immutable vs fixed-size but settable; `List.of` rejects null.
- **`Optional` as field/parameter?** Avoid; use as return type only.
