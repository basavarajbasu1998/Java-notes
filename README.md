# Java Full-Stack Interview Notes (5-Year Experience) — Index

Every topic has its own file: **simple idea → analogy → flow chart → code → interview answer → traps**.
Files marked ★ are the ones most often asked in 5-year interviews.

## 1. Core Java
| Note | What is inside |
|---|---|
| `JavaFullstack.md` | Big overview: Collections, Strings, JVM, GC, Exceptions, OOP, Java 8, Multithreading, Design Patterns, SOLID, Spring, Microservices, JPA |
| `Collections.md` | HashMap internals, ArrayList vs LinkedList, fail-fast, Comparable/Comparator |
| `Java_8.md` | Lambda, functional interfaces, Streams, Optional |
| ★ `Java_Memory_Model_and_Locks.md` | volatile, happens-before, synchronized vs ReentrantLock, CAS |
| ★ `ThreadPoolExecutor.md` | Pool internals, rejection policies, CountDownLatch, `@Async` |
| `Core_Java_Deep_Points.md` | Generics, immutability, ClassLoader, reflection, serialization |
| `Modern_Java_11_to_21.md` | records, sealed, switch expressions, virtual threads |
| `More_Design_Patterns.md` | Builder, Decorator, Adapter, Proxy, Template |

## 2. Spring & Backend
| Note | What is inside |
|---|---|
| ★ `Spring_Transactional.md` | Proxy, propagation, isolation, rollback rules, self-invocation trap |
| ★ `SpringBoot_Internals.md` | Startup flow, auto-configuration, starters, profiles, Actuator |
| ★ `REST_API_Design.md` | HTTP methods, status codes, validation, `@ControllerAdvice`, idempotency |
| `SpringBoot_Order_Service.md` | Full layered example: entity → repository → service → controller |
| `Spring_Security_JWT_OAuth2.md` | Security flow, JWT, OAuth2, CORS/CSRF |
| `Testing_JUnit_Mockito.md` | Unit, slice and integration tests |
| `J2EE_Servlets_JDBC.md` | Servlet container flow, filters, JDBC → JPA → Spring Data |

## 3. Database
| Note | What is inside |
|---|---|
| ★ `SQL_Database_Performance.md` | Indexes, EXPLAIN flow, joins, ACID, order-schema example |

## 4. Messaging & Microservices
| Note | What is inside |
|---|---|
| `Microservices_Patterns_and_Flow.md` | Gateway, Feign, circuit breaker, Saga, pattern checklist |
| `RabbitMQ.md` | Complete RabbitMQ theory and interview answers |
| `RabbitMQ_Spring_Project_Flow.md` | RabbitMQ inside the order project (Spring AMQP) |
| ★ `Kafka.md` | Topics, partitions, consumer groups, guarantees, Spring code, outbox |

## 5. DevOps, Cloud, Delivery
| Note | What is inside |
|---|---|
| `Docker.md` | Dockerfile (multi-stage), docker-compose, commands |
| `Kubernetes.md` | Architecture, Deployment/Service/HPA YAML, rolling update, debugging |
| `AWS.md` | Core services, network layout, IAM, scaling, deploy flow |
| `Git.md` | Daily flow, branching, merge vs rebase, conflicts |
| `CI_CD_Pipeline.md` | Pipeline diagram, GitHub Actions, deployment strategies |
| `Production_Debugging.md` | OOM flow, high CPU, logging, tools |

## 6. Front-End
| Note | What is inside |
|---|---|
| `JavaScript.md` | Closures, event loop, promises, async/await |
| `CSS.md` | Box model, flexbox, grid, responsive |
| `React.md` | Rendering flow, hooks, state, auth flow |

## 7. Process, AI, Big Picture
| Note | What is inside |
|---|---|
| `SDLC_and_AI_SDLC.md` | Agile/Scrum, AI in every phase, spec-driven flow |
| `AI_LLM_RAG.md` | LLMs, RAG, agents/tools, Spring integration |
| `Project_Architecture_Overview.md` | One feature traced through every technology |
| `System_Design_Scenarios.md` | URL shortener, rate limiter, cache strategies |
| `Project_Interview_QA.md` | "Explain your architecture" + 8-week plan |
| `Behavioural_and_Rapid_Fire.md` | STAR stories, short answers to memorise |

## 8. Coding Practice
| Note | What is inside |
|---|---|
| `Coding_Strings.md` | reverse, palindrome, anagram, sliding window, parentheses |
| `Coding_Arrays.md` | Two Sum, Kadane, intervals, binary search |
| `Coding_Collections.md` | LRU cache, group anagrams, top-K |
| `Coding_Streams.md` | 20 employee-style stream problems |
| `Coding_Multithreading.md` | odd/even, producer-consumer, deadlock, CompletableFuture |
| `Coding_LLD_Spring.md` | Strategy/Factory/Builder, CRUD controller, AOP, JWT filter |
| `Coding_SQL.md` | Nth salary, window functions, duplicates |
| `Coding_Interview_Approach.md` | 5-step routine + pattern cheat-sheet |

## Suggested reading order
1. Revise `Collections.md`, `Java_8.md`, `JavaFullstack.md` (your original notes).
2. Fill the gaps: `Spring_Transactional` → `SpringBoot_Internals` → `REST_API_Design` → `Java_Memory_Model_and_Locks` → `ThreadPoolExecutor` → `SQL_Database_Performance` → `Kafka`.
3. Front-end + DevOps: `JavaScript` → `React` → `Docker` → `Kubernetes` → `AWS` → `CI_CD_Pipeline`.
4. Practice with the `Coding_*.md` files (2 problems a day).
5. Finish with `Project_Architecture_Overview`, `System_Design_Scenarios`, `Project_Interview_QA`.
