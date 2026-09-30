# Spring IoC Container Internals (Deep) - Spring 6 / Boot 3, JDK 17/21

> Audience: 5-year Java developer preparing for senior interviews. Read Part 1 first (simple), then go down. Everything here follows the real class names in `spring-beans`, `spring-context`, `spring-aop`. Where an internal detail is version-sensitive it is described conceptually and marked.
>
> Code marked **[VERIFIED]** was compiled and run with `javac`/`java` (plain-Java mini simulations, no Spring jar). Spring console output marked **[TRACED]** was derived from the documented/source call order, not from a machine run.

## Table of Contents
1. 60-second mental model + analogy
2. Deep internals
   2.1 BeanDefinition and BeanDefinitionRegistry
   2.2 `refresh()` step by step
   2.3 ConfigurationClassPostProcessor
   2.4 Bean creation: `doGetBean` to ready
   2.5 Lifecycle in exact order
   2.6 Injection styles and candidate resolution
   2.7 `@Configuration` full vs lite (CGLIB)
   2.8 Singleton registry and the three-level cache
   2.9 Scopes and scoped proxies
   2.10 BeanFactory vs ApplicationContext vs FactoryBean
   2.11 `@Conditional`, `@Import`, ImportSelector, Registrar
   2.12 Events
   2.13 Environment, PropertySource precedence, SpEL
   2.14 AOP internals
   2.15 Proxy-based features: `@Transactional`, `@Cacheable`, `@Async`
3. Traced worked examples
4. Failure modes and war stories
5. Interview questions (40) + wrong answers
6. One-page cheat sheet

---

# 1. 60-second mental model + analogy

**Analogy: a restaurant kitchen with a head chef.**

- **BeanDefinition** = the recipe card (class, scope, ingredients, init steps). No dish exists yet.
- **BeanDefinitionRegistry** = the recipe box. Scanning, `@Bean`, `@Import` all just put cards in the box.
- **BeanFactoryPostProcessor** = an editor who is allowed to rewrite recipe cards before cooking starts (placeholders, adding new cards from `@Configuration` classes).
- **BeanFactory (`DefaultListableBeanFactory`)** = the kitchen that turns cards into dishes on demand, and keeps singleton dishes on the pass (the singleton cache).
- **BeanPostProcessor** = quality-control stations every dish passes through (inject ingredients, run @PostConstruct, wrap in packaging = AOP proxy).
- **ApplicationContext** = the restaurant: kitchen + events + i18n + environment + lifecycle + opens the doors (web server).
- **`refresh()`** = "open the restaurant": read recipe cards, install QC stations, pre-cook all singleton dishes, open the doors.

**Five sentences that carry 80% of interviews**

1. Spring first builds *metadata* (BeanDefinitions), then *instances*. Configuration is processed by a `BeanFactoryPostProcessor`, not magic.
2. Every bean goes: instantiate -> populate -> Aware -> BPP-before -> init callbacks -> BPP-after -> ready. AOP proxies replace the bean in BPP-after.
3. What you get from `getBean` for an advised bean is a **proxy** (JDK or CGLIB); calls on `this` bypass it.
4. Circular singleton field/setter dependencies work because a *factory for the half-built bean* is exposed early (three-level cache); constructor cycles cannot, because there is no instance to expose yet.
5. `@Configuration` classes are CGLIB-subclassed so that `@Bean` method calls are routed through the container and return the singleton.

**Picture**

```
  @Configuration / @ComponentScan / @Import / auto-config
              |   (ConfigurationClassPostProcessor)
              v
   +-------------------------+      BFPPs edit definitions
   |  BeanDefinition registry| <--------------------------+
   +-------------------------+
              |  finishBeanFactoryInitialization (eager singletons)
              v
   instantiate -> populate -> aware -> BPP.before -> init -> BPP.after (proxy!)
              |
              v
   singletonObjects (L1)  ---- getBean() ----> your code holds PROXY or raw
```

---

# 2. Deep internals

## 2.1 BeanDefinition and BeanDefinitionRegistry

`BeanDefinition` (interface, `org.springframework.beans.factory.config`) describes *how to create* a bean. Main data:

| Property | Meaning |
|---|---|
| `beanClassName` | class to instantiate (null when a factory method is used) |
| `scope` | `singleton`, `prototype`, `request`, ... (empty string = default singleton) |
| `lazyInit` | skip pre-instantiation |
| `dependsOn` | explicit creation-order dependencies (also affects destroy order) |
| `autowireCandidate`, `primary` | participation in autowiring / tie-break |
| `factoryBeanName` + `factoryMethodName` | how `@Bean` methods are represented (config class bean + method name) |
| `constructorArgumentValues`, `propertyValues` | explicit args/properties (mostly XML / programmatic) |
| `initMethodName`, `destroyMethodName` | `@Bean(initMethod=..)`; destroy default for `@Bean` is `(inferred)` - public no-arg `close()` or `shutdown()` |
| `role` | APPLICATION / SUPPORT / INFRASTRUCTURE (infra beans are exempt from some warnings) |

Implementations you will see in a debugger:

| Class | Produced by |
|---|---|
| `ScannedGenericBeanDefinition` | `@ComponentScan` (ClassPathBeanDefinitionScanner) |
| `AnnotatedGenericBeanDefinition` | `AnnotationConfigApplicationContext.register(Foo.class)`, `@Import`ed classes |
| `ConfigurationClassBeanDefinition` | each `@Bean` method |
| `RootBeanDefinition` | **merged** definition (child + parent flattened); the factory always works on this via `getMergedBeanDefinition` |
| `GenericBeanDefinition` | programmatic / XML |

`BeanDefinitionRegistry` API: `registerBeanDefinition`, `removeBeanDefinition`, `getBeanDefinition`, `containsBeanDefinition`, `getBeanDefinitionNames`, `getBeanDefinitionCount`, `isBeanNameInUse`, `registerAlias`. `DefaultListableBeanFactory` implements it and stores definitions in `beanDefinitionMap` (a `ConcurrentHashMap`) plus `beanDefinitionNames` (registration order - this order drives eager instantiation order).

Bean-definition **overriding** (same name registered twice): Spring itself allows it, Boot disables it since 2.1 (`spring.main.allow-bean-definition-overriding=false`) and fails fast with `BeanDefinitionOverrideException`.

Key insight: **a definition is not a bean**. Until `preInstantiateSingletons`, almost nothing has been instantiated (except BFPPs/BPPs and their dependencies).

## 2.2 `refresh()` step by step

`AbstractApplicationContext.refresh()` (template method; guarded by `startupShutdownMonitor`). Order:

```
refresh()
 1  prepareRefresh()                    - startup date/active flag, init property sources (initPropertySources),
                                          validate required properties, earlyApplicationEvents buffer
 2  obtainFreshBeanFactory()            - refreshBeanFactory(): create DefaultListableBeanFactory
                                          (Generic/AnnotationConfig contexts: already exists)
 3  prepareBeanFactory(bf)              - classloader, SpEL BeanExpressionResolver, PropertyEditorRegistrar,
                                          + ApplicationContextAwareProcessor (BPP),
                                          ignoreDependencyInterface(EnvironmentAware, ResourceLoaderAware, ...),
                                          registerResolvableDependency(BeanFactory, ResourceLoader,
                                             ApplicationEventPublisher, ApplicationContext) -> "injecting ApplicationContext works"
                                          + ApplicationListenerDetector (BPP),
                                          register singletons: environment, systemProperties, systemEnvironment
 4  postProcessBeanFactory(bf)          - hook for subclasses (web contexts add request/session scopes here)
 5  invokeBeanFactoryPostProcessors(bf) - ***where @Configuration/@ComponentScan/@Import/@Bean become definitions***
 6  registerBeanPostProcessors(bf)      - instantiate BPP beans and add to the factory (not applied to them yet)
 7  initMessageSource()                 - bean "messageSource" or a delegating default
 8  initApplicationEventMulticaster()   - bean "applicationEventMulticaster" or SimpleApplicationEventMulticaster
 9  onRefresh()                         - hook; web contexts create the embedded server (createWebServer)
10  registerListeners()                 - register static + ApplicationListener beans; publish buffered early events
11  finishBeanFactoryInitialization(bf) - conversion service, embedded value resolver, freezeConfiguration,
                                          preInstantiateSingletons()  <- your beans are created here
12  finishRefresh()                     - clear resource caches, initLifecycleProcessor, LifecycleProcessor.onRefresh()
                                          (starts SmartLifecycle beans; Boot: WebServerStartStopLifecycle starts connectors),
                                          publish ContextRefreshedEvent
on exception: destroyBeans(); cancelRefresh(ex); rethrow
```

Details worth knowing:

- **Step 5 ordering** (`PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors`): first all `BeanDefinitionRegistryPostProcessor`s - `PriorityOrdered`, then `Ordered`, then the rest, **repeating until no new registry post-processors appear** (a registrar can register more registrars); then their `postProcessBeanFactory` callbacks; then plain `BeanFactoryPostProcessor`s, again priority-ordered / ordered / rest. `ConfigurationClassPostProcessor` is a `PriorityOrdered` registry post-processor, so it runs first.
- **Step 6** (`registerBeanPostProcessors`): BPP beans are registered in groups: `PriorityOrdered`, `Ordered`, ordinary, then internal ones (`MergedBeanDefinitionPostProcessor`) re-registered, and `ApplicationListenerDetector` moved to the end. A BPP and everything it depends on is created *during this step* - before other BPPs are in place - so they are not eligible for auto-proxying and you see the log `Bean 'x' of type [...] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)` (emitted by `BeanPostProcessorChecker`).
- **Step 9 (Boot web)**: `ServletWebServerApplicationContext.onRefresh()` -> `createWebServer()` builds Tomcat/Jetty/Undertow and initializes it, but **connectors are started only in `finishRefresh`** (Boot 2.3+, `WebServerStartStopLifecycle`), so the app never accepts traffic before all singletons exist.
- **Step 11** (`preInstantiateSingletons`): iterate `beanDefinitionNames` in registration order, for each non-abstract, singleton, non-lazy definition: if it is a `FactoryBean` create `&name` (product only if `SmartFactoryBean.isEagerInit`), else `getBean(name)`. After all are created, every bean implementing `SmartInitializingSingleton` gets `afterSingletonsInstantiated()` - this is when `@EventListener` methods are turned into listeners (`EventListenerMethodProcessor`) and `@Scheduled` tasks are registered.
- **Step 12**: `ContextRefreshedEvent` fires here. In a parent/child context hierarchy each context publishes its own, and child publishes propagate to the parent - so listeners can run twice.

`SpringApplication.run` wraps it: create Environment -> `ApplicationEnvironmentPreparedEvent` -> create context -> `prepareContext` (`ApplicationContextInitializer`s, load main class as a bean definition) -> `refreshContext` (= `refresh()` + shutdown hook) -> `ApplicationStartedEvent` (after refresh, before runners) -> `CommandLineRunner`/`ApplicationRunner` -> `ApplicationReadyEvent`. Also `AvailabilityChangeEvent` for liveness/readiness.

## 2.3 ConfigurationClassPostProcessor

Runs in step 5. Two entry points:

**A. `postProcessBeanDefinitionRegistry` -> `processConfigBeanDefinitions`**

```
1. Scan registry for "configuration candidates":
     full  : @Configuration (proxyBeanMethods = true)
     lite  : @Component, @ComponentScan, @Import, @ImportResource, any class with @Bean methods, @Configuration(proxyBeanMethods=false)
2. Sort candidates by @Order.
3. Loop (ConfigurationClassParser.parse on each candidate):
     processConfigurationClass(cfg)
        - @Conditional check (phase PARSE_CONFIGURATION) - skip whole class if false
        - process member classes (nested @Configuration)
        - @PropertySource      -> add to Environment
        - @ComponentScan       -> ComponentScanAnnotationParser -> ClassPathBeanDefinitionScanner.doScan
                                  (registers ScannedGenericBeanDefinition, THEN immediately parses each scanned
                                   class that is itself a config candidate -> recursion)
        - @Import              -> collect: (a) regular/@Configuration class -> process as config
                                          (b) ImportSelector -> selectImports() -> recurse on returned names
                                          (c) DeferredImportSelector -> queued; processed AFTER all other config
                                          (d) ImportBeanDefinitionRegistrar -> stored, invoked later at load time
        - @ImportResource      -> XML/Groovy readers, stored for later
        - @Bean methods        -> collected as BeanMethod (incl. default methods on interfaces)
        - superclass           -> processed too (loop)
   after the loop: DeferredImportSelectors are processed  <- Boot auto-configuration lives here
4. ConfigurationClassBeanDefinitionReader.loadBeanDefinitions(configClasses):
        - register imported classes as definitions
        - register each @Bean method as ConfigurationClassBeanDefinition (factoryBeanName = config bean, factoryMethodName)
        - run ImportBeanDefinitionRegistrars
        - @ImportResource readers
5. Repeat while newly registered definitions contain unparsed candidates.
```

Why Boot auto-config is "deferred": `AutoConfigurationImportSelector` is a `DeferredImportSelector`, so it runs after *all* user configuration; `@ConditionalOnMissingBean` inside auto-config can therefore see user beans. Boot 3 reads auto-config class names from `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Boot 2.7 introduced it, Boot 3 dropped the `spring.factories` route for auto-config), then filters them by `AutoConfigurationImportFilter`s (`OnClassCondition`, `OnBeanCondition`, `OnWebApplicationCondition` via ASM metadata - the class is not loaded).

**B. `postProcessBeanFactory`**: `enhanceConfigurationClasses` - for every *full* config class, swap the bean definition's class for a CGLIB subclass built by `ConfigurationClassEnhancer` (see 2.7). Also registers `ImportAwareBeanPostProcessor` (supports `ImportAware`).

Consequence: `@Bean` methods that return a `BeanFactoryPostProcessor` should be `static` - otherwise Spring must instantiate the whole config class very early (before BPPs exist) and `@Autowired`/`@Value` in that class will not be processed (warning is logged).

## 2.4 Bean creation: `doGetBean` to ready

Entry: `AbstractBeanFactory.getBean(name)` -> `doGetBean`.

```
doGetBean(name)
 1  transformedBeanName (strip '&', resolve alias)
 2  sharedInstance = getSingleton(beanName)             // 3-level lookup, see 2.8
 3  if found  -> getObjectForBeanInstance (FactoryBean unwrap: '&' returns factory, else getObject())
 4  else
      a  if prototype currently in creation -> BeanCurrentlyInCreationException
      b  parent factory has definition? -> delegate to parent
      c  mbd = getMergedLocalBeanDefinition(beanName)
      d  for each dependsOn: registerDependentBean + getBean(dep)
      e  singleton: getSingleton(beanName, () -> createBean(beanName, mbd, args))
                       - beforeSingletonCreation: add to singletonsCurrentlyInCreation
                         (throws BeanCurrentlyInCreationException if already there)
                       - singletonFactory.getObject() -> createBean
                       - afterSingletonCreation: remove; addSingleton -> L1, remove L2/L3
         prototype:    beforePrototypeCreation; createBean; afterPrototypeCreation
         other scope:  scope.get(name, objectFactory)
      f  type conversion of the result if requiredType given
```

`AbstractAutowireCapableBeanFactory.createBean` -> `doCreateBean`:

```
createBean
  resolveBeforeInstantiation        InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation
                                    (short-circuit: can return a ready object, e.g. custom TargetSource -> proxy)
  doCreateBean
    1 createBeanInstance            factory method (@Bean)  |  constructor autowiring  |  no-arg constructor
                                    constructor choice: SmartInstantiationAwareBPP.determineCandidateConstructors
                                    (AutowiredAnnotationBeanPostProcessor: single ctor => implicit @Autowired)
    2 applyMergedBeanDefinitionPostProcessors
                                    MergedBeanDefinitionPostProcessor: scans @Autowired/@Value/@Resource/@PostConstruct
                                    metadata ONCE per class and caches it
    3 early exposure                if (singleton && allowCircularReferences && currentlyInCreation)
                                        addSingletonFactory(name, () -> getEarlyBeanReference(name, mbd, bean))   // L3
    4 populateBean                  a) InstantiationAwareBPP.postProcessAfterInstantiation (veto)
                                    b) autowire by name/type if configured (legacy)
                                    c) InstantiationAwareBPP.postProcessProperties  <- @Autowired/@Value/@Resource injection
                                    d) applyPropertyValues
    5 initializeBean
        5a invokeAwareMethods       BeanNameAware, BeanClassLoaderAware, BeanFactoryAware ONLY
        5b applyBeanPostProcessorsBeforeInitialization
                                    ApplicationContextAwareProcessor: Environment/ResourceLoader/EventPublisher/
                                       MessageSource/ApplicationContext/EmbeddedValueResolver aware callbacks
                                    InitDestroyAnnotationBPP (CommonAnnotationBPP): @PostConstruct
                                    ... custom BPPs by order
        5c invokeInitMethods        InitializingBean.afterPropertiesSet(), then custom init-method
        5d applyBeanPostProcessorsAfterInitialization
                                    AbstractAutoProxyCreator -> proxy; AsyncAnnotationBPP, ... ; ApplicationListenerDetector
    6 early-reference reconciliation (see 2.8: "raw version" check)
    7 registerDisposableBeanIfNecessary (DisposableBeanAdapter for singletons with destroy hooks)
```

Point to memorize: `@PostConstruct` is **not** a container primitive; it is a `BeanPostProcessor.postProcessBeforeInitialization` hook (`CommonAnnotationBeanPostProcessor`, which is `PriorityOrdered`). So a plain custom BPP (no ordering) sees the bean **after** `@PostConstruct` ran.

## 2.5 Lifecycle in exact order

```
constructor
 -> @Autowired fields/setters injected            (populateBean, AutowiredAnnotationBeanPostProcessor)
 -> BeanNameAware.setBeanName
 -> BeanClassLoaderAware.setBeanClassLoader
 -> BeanFactoryAware.setBeanFactory
 -> EnvironmentAware / ResourceLoaderAware / ApplicationEventPublisherAware /
    MessageSourceAware / ApplicationContextAware      (ApplicationContextAwareProcessor - a BPP before-init)
 -> @PostConstruct                                  (CommonAnnotationBeanPostProcessor, before-init)
 -> other BPP.postProcessBeforeInitialization       (custom, by order)
 -> InitializingBean.afterPropertiesSet()
 -> @Bean(initMethod) / XML init-method
 -> BPP.postProcessAfterInitialization              (AOP proxy created HERE - may return a different object)
 == bean ready, stored in singletonObjects ==
 ... context.close() / shutdown hook ...
 -> DestructionAwareBPP.postProcessBeforeDestruction (@PreDestroy)
 -> DisposableBean.destroy()
 -> @Bean(destroyMethod) (default "(inferred)": close()/shutdown())
```

Rules:
- Prototypes: container does instantiate/inject/init and then **forgets** the instance - no destroy callbacks.
- Singletons are destroyed in reverse order of creation, dependents before their dependencies (dependency tracking in `dependentBeanMap`). `SmartLifecycle` beans stop first, in descending phase.
- Only `@PostConstruct`/`@PreDestroy` on the *same bean class*; they are package `jakarta.annotation` in Boot 3 (`javax` no longer works).
- Multiple init mechanisms with the same method name are de-duplicated (method invoked once).
- If an early reference exists (circular case), the object other beans hold is the pre-init one (raw or early proxy), but init still runs on the same underlying instance.

## 2.6 Injection styles and candidate resolution

| | Constructor | Setter | Field |
|---|---|---|---|
| Immutability (`final`) | yes | no | no |
| Mandatory deps enforced at compile time | yes | no | no |
| Unit test without Spring | `new X(mock)` | setter | reflection needed |
| Circular dependency | **fails** (see 2.8) | works (singleton) | works (singleton) |
| Hides SRP violations | no (huge ctor is a smell) | partly | yes |
| Spring mechanism | `autowireConstructor` at instantiation | BPP in populateBean | BPP in populateBean |

Constructor rule: one constructor => injected implicitly (Spring 4.3+); several => annotate exactly one with `@Autowired` (or `required=false` on several to let Spring choose the greediest satisfiable), otherwise the no-arg constructor is used. Records work with their canonical constructor.

**Resolution algorithm** (`DefaultListableBeanFactory.doResolveDependency`), for a dependency with type `T`:

```
1  @Value?                      -> resolve placeholder / SpEL, convert
2  multiple-bean types?         -> T is array / Collection / Map<String,X>: gather ALL candidates of X
                                   (sorted by @Order/Ordered/@Priority for List)
3  findAutowireCandidates(T)    -> names by type (incl. FactoryBean products, parent factories), keep only those where
        isAutowireCandidate:  - definition.autowireCandidate == true
                              - generics match  (Repository<User> does not match Repository<Order>)
                              - @Qualifier match (QualifierAnnotationAutowireCandidateResolver)
4  0 candidates                 -> required ? NoSuchBeanDefinitionException : null/empty
   1 candidate                  -> done
   >1 candidates                -> determineAutowireCandidate:
        a) exactly one @Primary  (>1 primary => NoUniqueBeanDefinitionException)
        b) highest @Priority (jakarta.annotation.Priority)
        c) FALLBACK BY NAME: dependency name (field name / constructor-or-setter parameter name)
           equals a candidate bean name or alias
        d) else NoUniqueBeanDefinitionException
```

Notes:
- Step (c) needs parameter names: compile with `-parameters` (Spring Boot parent POM / Gradle plugin do it). **Spring 6.1 removed bytecode-debug-info-based parameter name discovery**, so without `-parameters` this fallback silently stops working after an upgrade.
- `@Qualifier("x")` with no matching qualifier metadata falls back to matching the bean *name* `x`.
- `@Resource` (jakarta) resolves **by name first**, then by type; `@Autowired` type first.
- Wrappers that *defer* resolution: `ObjectProvider<T>` / `Provider<T>` / `Optional<T>` / `@Lazy`. `ObjectProvider` has `getIfAvailable()`, `getIfUnique()`, `stream()`, `orderedStream()`.
- `@Lazy` on an injection point injects a **lazy-resolution proxy** (built by `ContextAnnotationAutowireCandidateResolver`) that resolves the real bean on first method call.
- Spring 6.2 adds `@Fallback` (lowest-priority candidate) - use only if you are on 6.2+.
- A bean named `foo` produced by `FactoryBean` appears for type of its *product*, which can force early FactoryBean init during type matching.

## 2.7 `@Configuration` full vs lite (CGLIB)

```java
@Configuration            // full mode: proxyBeanMethods = true (default)
class AppConfig {
    @Bean Repo repo()       { return new Repo(); }
    @Bean Service service() { return new Service(repo()); }   // calls repo() directly
    @Bean Service2 service2(){ return new Service2(repo()); }
}
```

If `AppConfig` were used as-is, `repo()` would be called three times -> three `Repo` objects. Spring avoids it:

```
ConfigurationClassPostProcessor.postProcessBeanFactory
  -> ConfigurationClassEnhancer.enhance(AppConfig.class)    -> AppConfig$$SpringCGLIB$$0 (subclass)
  -> beanDefinition("appConfig").setBeanClass(enhanced)

call service() on the enhanced instance:
  BeanMethodInterceptor.intercept(...)
     is this the method the container itself is currently invoking as factory method?
        (SimpleInstantiationStrategy.getCurrentlyInvokedFactoryMethod())
        YES -> invokeSuper: run real code of service()
     inside it, repo() is called on `this` (the enhanced subclass) -> interceptor again:
        current factory method is service(), not repo()  -> NOT the same
        -> beanFactory.getBean("repo")   // singleton lookup / creation
```

So "`@Bean` method call returns the singleton" = CGLIB intercept + `getBean` redirect, and only inside full-mode config classes. Also intercepted: `BeanFactoryAwareMethodInterceptor` so the enhanced class receives the `BeanFactory`.

**Lite mode** - no proxy: `@Component`/`@Service` classes with `@Bean` methods, `@Import`ed plain classes, and `@Configuration(proxyBeanMethods = false)`. Calling `repo()` is a plain Java call -> a NEW `Repo`, not a bean. Benefits: no CGLIB subclass, faster startup, allows `final` classes/methods; that's why every Boot `*AutoConfiguration` uses `proxyBeanMethods = false` and takes dependencies as **method parameters**:

```java
@Bean Service service(Repo repo) { return new Service(repo); }   // safe in both modes
```

Restrictions for full mode: class non-final, `@Bean` methods non-private non-final (Spring logs/throws otherwise), no-arg (or `@Autowired`) constructor is fine since 4.3.

## 2.8 Singleton registry and the three-level cache

`DefaultSingletonBeanRegistry` state:

| Field | Level | Holds | Type |
|---|---|---|---|
| `singletonObjects` | L1 | **fully initialized** singletons (final objects, proxies included) | `ConcurrentHashMap` |
| `earlySingletonObjects` | L2 | **early references** already handed out to someone (raw bean or early proxy) | `ConcurrentHashMap` |
| `singletonFactories` | L3 | `ObjectFactory` lambdas: `() -> getEarlyBeanReference(name, mbd, bean)` | `HashMap` |
| `singletonsCurrentlyInCreation` | - | names being built right now | `Set` |
| `earlyProxyReferences` (in `AbstractAutoProxyCreator`) | - | beans already proxied via early reference | `Map` |

Lookup (`getSingleton(name, allowEarlyReference=true)`):

```
o = singletonObjects.get(name)                      // L1
if (o == null && isSingletonCurrentlyInCreation(name)) {
    o = earlySingletonObjects.get(name)             // L2
    if (o == null && allowEarlyReference) {
        factory = singletonFactories.get(name)      // L3
        if (factory != null) {
            o = factory.getObject()                 // -> getEarlyBeanReference: SmartInstantiationAwareBPPs
                                                    //    (AbstractAutoProxyCreator may create the PROXY now)
            earlySingletonObjects.put(name, o)      // promote L3 -> L2
            singletonFactories.remove(name)
        }
    }
}
return o
```

(Lock handling around this differs across 6.0/6.1/6.2, the shape is the same.)

**Why three levels, not two?** L2-only would mean creating the (possibly AOP) proxy for *every* bean right after instantiation, just in case somebody needs it. Normal design puts proxy creation in BPP-after-init. L3 stores a *lazy factory* so the early proxy is created **only if a circular reference actually happens**; otherwise the proxy is created at the normal place. L2 exists so that if two other beans ask for the early ref, both get the same object (the factory must run once).

**Why constructor cycles fail**: for `A(B b)`, `B(A a)`: creating A -> `createBeanInstance` needs B *before A exists* -> A's L3 factory is added only *after* instantiation (step 3 of `doCreateBean`), so nothing to expose. B is created, needs A -> A is in `singletonsCurrentlyInCreation` but L1/L2/L3 have nothing -> `beforeSingletonCreation` throws `BeanCurrentlyInCreationException: Error creating bean with name 'a': Requested bean is currently in creation: Is there an unresolvable circular reference?`.

Also fail regardless of style: **prototype** cycles (`prototypesCurrentlyInCreation`), and any cycle when `allowCircularReferences=false`.

**Boot 2.6+**: `SpringApplication` sets `allowCircularReferences=false` (`spring.main.allow-circular-references`, default false). Spring Framework itself still defaults to true. On failure Boot prints "The dependencies of some of the beans in the application context form a cycle:" with an ASCII loop diagram (`FailureAnalyzer`).

**The "raw version" check** (step 6 of `doCreateBean`):

```
exposedObject = initializeBean(...)                        // maybe wrapped by a BPP
if (earlySingletonExposure) {
    earlySingletonReference = getSingleton(name, false);   // L1/L2 only, do not trigger L3
    if (earlySingletonReference != null) {                 // someone consumed the early ref
        if (exposedObject == bean)  exposedObject = earlySingletonReference;   // OK: use what others hold
        else if (!allowRawInjectionDespiteWrapping && hasDependentBean(name)) {
            throw BeanCurrentlyInCreationException("Bean with name 'x' has been injected into other beans [y]
                  in its raw version as part of a circular reference, but has eventually been wrapped ...");
        }
    }
}
```

Why it happens: `AbstractAutoProxyCreator` is early-reference-aware: `getEarlyBeanReference` creates the proxy, records it in `earlyProxyReferences`, and its `postProcessAfterInitialization` then returns the bean **unchanged** (already proxied) - `exposedObject == bean`, consistent. But a BPP that wraps in after-init and does **not** implement early-reference support (the classic: `@Async`'s `AsyncAnnotationBeanPostProcessor`, an `AbstractAdvisingBeanPostProcessor`) returns a *different* object while someone already holds the raw one -> exception. Fixes: break the cycle, or `@Lazy` the injection, or move `@Async` to a separate bean, or (dangerous) `allowRawInjectionDespiteWrapping`.

`@Lazy` fix for constructor cycle: `A(@Lazy B b)` - a lazy proxy for B is injected, so A finishes; B resolves `getBean(B)` at first call. Works, but it only hides a design smell - better: extract a third bean / use events / invert the dependency.

## 2.9 Scopes and scoped proxies

| Scope | Where stored | Notes |
|---|---|---|
| singleton | `singletonObjects` | 1 per container (not per JVM, not per class) |
| prototype | nowhere | new per `getBean`/injection point; no destroy callbacks |
| request / session / application | `RequestScope`/`SessionScope`/`ServletContextScope` -> `RequestAttributes` bound by `RequestContextHolder` (ThreadLocal) via `RequestContextListener`/`RequestContextFilter` | registered in `postProcessBeanFactory` of web contexts |
| websocket | session attributes | |
| custom | implement `Scope` (`get`, `remove`, `registerDestructionCallback`, `resolveContextualObject`, `getConversationId`), `beanFactory.registerScope("x", scope)` | `SimpleThreadScope` exists but is not registered by default |

**Prototype in singleton**: injection happens once, at singleton creation -> the singleton holds ONE prototype forever.

Fixes:
1. `ObjectProvider<Proto>` / `jakarta.inject.Provider<Proto>` -> `provider.getObject()` each time (clearest, testable).
2. `@Lookup` method: Spring CGLIB-subclasses the bean (`CglibSubclassingInstantiationStrategy`, `LookupOverride`) and overrides the method to `getBean(Proto.class)`. Method must be non-private, non-final, class non-final; works with `abstract` methods; not on `@Bean` factory-produced beans.
3. Scoped proxy: `@Scope(value="prototype", proxyMode=TARGET_CLASS)` -> a singleton proxy that creates a **new target on every method call** (surprising - state does not survive between two calls on the same proxy).
4. Inject `ApplicationContext` and call `getBean` (works, couples to Spring - last resort).

**Request/session scope into a singleton**: use `proxyMode = ScopedProxyMode.TARGET_CLASS` (or `INTERFACES`). Mechanics: `ScopedProxyFactoryBean` registers two definitions - the real bean under `scopedTarget.myBean` (with the request scope) and a *singleton proxy* under `myBean`. Every method call on the proxy goes through `SimpleBeanTargetSource.getTarget()` -> `beanFactory.getBean("scopedTarget.myBean")` -> looks up the current request's instance. Outside a request (async thread, startup, scheduler) -> `IllegalStateException: No thread-bound request found` / `Scope 'request' is not active for the current thread`. Fix: pass `RequestAttributes` to the worker thread via a `TaskDecorator`, or don't use request scope there (pass data as method arguments).

## 2.10 BeanFactory vs ApplicationContext vs FactoryBean

- `BeanFactory`: `getBean`, `containsBean`, `isSingleton`, `getType`, `getAliases`. Root container interface. `DefaultListableBeanFactory` is the concrete engine (also `BeanDefinitionRegistry` and `ConfigurableListableBeanFactory`).
- `ApplicationContext` **extends** `ListableBeanFactory`, `HierarchicalBeanFactory`, `MessageSource`, `ApplicationEventPublisher`, `ResourcePatternResolver`, `EnvironmentCapable`. It *has-a* `DefaultListableBeanFactory` (via `GenericApplicationContext`) and delegates `getBean` to it.
- Practical differences: ApplicationContext pre-instantiates singletons (fail fast), auto-detects BPP/BFPP beans and registers them, gives events, i18n, resource loading, `@Configuration` processing, AOP infrastructure. A bare `DefaultListableBeanFactory` is lazy and needs BPPs added manually (`addBeanPostProcessor`) - `@Autowired` will not work unless you add `AutowiredAnnotationBeanPostProcessor`.
- Boot web: `AnnotationConfigServletWebServerApplicationContext` (servlet), `AnnotationConfigReactiveWebServerApplicationContext`, `AnnotationConfigApplicationContext` (non-web).

**FactoryBean<T>** (a *bean that produces beans*): `getObject()`, `getObjectType()`, `isSingleton()`. `getBean("x")` returns the **product**; `getBean("&x")` returns the factory itself. Examples: `ProxyFactoryBean`, `ScopedProxyFactoryBean`, `LocalContainerEntityManagerFactoryBean`, `JndiObjectFactoryBean`, Feign clients. Interview trap: `FactoryBean` (a bean) vs `BeanFactory` (the container). Product is cached in `factoryBeanObjectCache` for singleton factories; the product does **not** get the full lifecycle (it gets only `postProcessObjectFromFactoryBean` -> BPP-after, so it can be proxied).

## 2.11 `@Conditional`, `@Import`, ImportSelector, Registrar

**`@Conditional`** - interface `Condition.matches(ConditionContext ctx, AnnotatedTypeMetadata md)`. `ConditionContext` gives registry, bean factory, environment, resource loader, classloader. `ConditionEvaluator.shouldSkip(metadata, phase)` is called by:
- `ConfigurationClassParser` (phase `PARSE_CONFIGURATION`, for `ConfigurationCondition`s asking for that phase, e.g. class-presence checks),
- `ConfigurationClassBeanDefinitionReader` per `@Bean` and per imported class (phase `REGISTER_BEAN`: needs the registry populated - e.g. `@ConditionalOnBean`),
- `ClassPathScanningCandidateComponentProvider` for scanned components.

`@Profile` is just `@Conditional(ProfileCondition.class)`.

Boot conditions (`spring-boot-autoconfigure`): `OnClassCondition` (`@ConditionalOnClass` uses ASM annotation metadata, safe if class missing; use the `name=` form on methods), `OnBeanCondition` (`@ConditionalOnBean/MissingBean/SingleCandidate`, **order-sensitive**: only reliable in auto-configuration or after user configs), `OnPropertyCondition`, `OnWebApplicationCondition`, `OnExpressionCondition`, `OnResourceCondition`, `OnJavaCondition`, `OnCloudPlatform`. Debug: run with `--debug` or actuator `/actuator/conditions` -> the *condition evaluation report* (positive/negative matches).

**Three `@Import` targets**:

```java
// 1. plain/@Configuration class
@Import(SecurityConfig.class)

// 2. ImportSelector: pick classes by metadata (returns class NAMES)
class MySelector implements ImportSelector {
    public String[] selectImports(AnnotationMetadata meta) {
        return new String[]{"com.acme.CacheConfig"};
    }
}
// DeferredImportSelector: same but processed after all other @Configuration; supports Group; used by auto-config

// 3. ImportBeanDefinitionRegistrar: register raw BeanDefinitions programmatically
class MyRegistrar implements ImportBeanDefinitionRegistrar {
    public void registerBeanDefinitions(AnnotationMetadata meta, BeanDefinitionRegistry reg) {
        reg.registerBeanDefinition("foo", new RootBeanDefinition(Foo.class));
    }
}
```

The `@Enable*` pattern = a meta-annotation with `@Import`: `@EnableAsync`, `@EnableCaching`, `@EnableTransactionManagement` (ImportSelectors that pick proxy vs aspectj configuration), `@EnableJpaRepositories`, `@EnableFeignClients`, `@MapperScan` (Registrars: they scan and register FactoryBean-based definitions, which is how interface-only beans exist).

## 2.12 Events

- Publisher: `ApplicationEventPublisher.publishEvent(Object)` (the context itself). Non-`ApplicationEvent` objects are wrapped in `PayloadApplicationEvent`.
- Multicaster: `SimpleApplicationEventMulticaster` finds matching listeners (cached by event type + source type) and invokes them **on the publisher's thread, in order, synchronously**, unless an `Executor` is set (`setTaskExecutor`, then all listeners are async - an all-or-nothing switch) or `@Async` is used on a particular listener.
- A listener that throws: exception propagates to `publishEvent` caller (unless an `ErrorHandler` is set) and **later listeners do not run**. A sync listener runs inside the publisher's transaction: its failure rolls the publisher back.
- `@EventListener`: methods are adapted by `EventListenerMethodProcessor` (a `BeanFactoryPostProcessor` + `SmartInitializingSingleton`) into `ApplicationListenerMethodAdapter` after all singletons exist. Return value non-null -> published as a new event. `condition = "#event.x > 5"` is SpEL. Ordering: `@Order` on the method (or `Ordered` for `ApplicationListener` classes).
- `@TransactionalEventListener`: bound to a tx phase (`AFTER_COMMIT` default, `AFTER_ROLLBACK`, `AFTER_COMPLETION`, `BEFORE_COMMIT`); **if there is no active transaction the event is silently dropped** unless `fallbackExecution = true`. Work done in `AFTER_COMMIT` runs when the tx is committed but still bound: a DB write there needs `REQUIRES_NEW`.
- `@Async` + `@EventListener`: runs in another thread - loses ThreadLocals (security context, MDC, request scope), tx, exceptions go to `AsyncUncaughtExceptionHandler` (void).
- Early events (before multicaster exists) are buffered in `earlyApplicationEvents` and replayed in `registerListeners`.

## 2.13 Environment, PropertySource precedence, SpEL

`Environment` = profiles + `PropertyResolver` over an ordered list `MutablePropertySources`; first source that has the key wins. `@Value("${a.b:default}")` is resolved by `PropertySourcesPlaceholderConfigurer` (a BFPP; Boot auto-registers it) / embedded value resolver; `#{...}` by SpEL (`StandardBeanExpressionResolver`). Values are resolved once, at injection, so they do not change afterwards (unless `@RefreshScope`/rebinding of `@ConfigurationProperties` via Spring Cloud).

Boot 3 precedence, **highest to lowest** (reference docs):
1. DevTools global settings (`~/.config/spring-boot`)
2. `@TestPropertySource`
3. `@SpringBootTest(properties=...)`
4. Command-line args (`--server.port=9090`)
5. `SPRING_APPLICATION_JSON`
6. `ServletConfig` init params
7. `ServletContext` init params
8. JNDI `java:comp/env`
9. Java system properties (`-Dserver.port=..`)
10. OS environment variables (`SERVER_PORT`, relaxed binding)
11. `RandomValuePropertySource` (`random.*`)
12. Profile-specific `application-{profile}` **outside** the jar
13. Profile-specific `application-{profile}` **inside** the jar
14. `application.properties/yml` outside the jar
15. `application.properties/yml` inside the jar
16. `@PropertySource` on `@Configuration` classes
17. Default properties (`SpringApplication.setDefaultProperties`)

Consequences: env var beats `application.yml` (Kubernetes/12-factor override), `-D` beats env, command line beats all runtime sources, `@PropertySource` is nearly lowest **and is loaded too late for properties Boot reads before refresh** (`logging.*`, `spring.main.*`). YAML: profile documents later in the file override earlier. `spring.config.import` adds sources with precedence over the file that imports them. Later profile in `spring.profiles.active=a,b` wins over earlier.

**SpEL** essentials: `#{2 * 3}`, `#{systemProperties['user.name']}`, `#{@myBean.method()}`, `#{T(java.lang.Math).random()}`, `#{obj?.field}` (safe navigation), `#{a ?: 'dflt'}` (Elvis), `#{list.?[age > 18]}` (selection), `#{list.![name]}` (projection). Where Spring evaluates it: `@Value`, `@Cacheable(key/condition/unless)`, `@PreAuthorize`, `@EventListener(condition)`, `@ConditionalOnExpression`, `@Scheduled`. Security: evaluating **user-supplied** strings with `StandardEvaluationContext` allows arbitrary method calls / `T(Runtime)` = RCE; use `SimpleEvaluationContext`.

## 2.14 AOP internals

**Vocabulary quickly**: Aspect (class with `@Aspect`), Join point (in Spring AOP: always a *method execution* on a Spring bean), Pointcut (predicate over join points), Advice (code), **Advisor = Advice + Pointcut** (Spring's own unit; an `@Aspect` advice method becomes one `InstantiationModelAwarePointcutAdvisorImpl`), Weaving, Target, Proxy.

### 2.14.1 How the proxy gets created (`AbstractAutoProxyCreator`)

Registered as bean `org.springframework.aop.config.internalAutoProxyCreator` (Boot `AopAutoConfiguration` -> `@EnableAspectJAutoProxy`). Only one auto-proxy creator exists; `@EnableTransactionManagement`/`@EnableCaching` register `InfrastructureAdvisorAutoProxyCreator`, which **escalates** to `AnnotationAwareAspectJAutoProxyCreator` if AspectJ support is on.

It is a `SmartInstantiationAwareBeanPostProcessor`:

```
postProcessBeforeInstantiation(class, name)  - only for custom TargetSource; usually returns null
getEarlyBeanReference(bean, name)            - key = cacheKey; earlyProxyReferences.put(key, bean); return wrapIfNecessary(...)
postProcessAfterInitialization(bean, name)   - if (earlyProxyReferences.remove(key) != bean) return wrapIfNecessary(bean,name,key)
                                               else return bean (already proxied early)

wrapIfNecessary(bean, name, key)
   1  skip if: targetSourcedBeans, advisedBeans[key]==FALSE, infrastructure class (Advice/Pointcut/Advisor/AopInfrastructureBean),
             or shouldSkip (e.g. aspect beans themselves)
   2  specificInterceptors = getAdvicesAndAdvisorsForBean(class)
        -> findEligibleAdvisors:
             findCandidateAdvisors():  all Advisor beans  +  (AnnotationAware...) buildAspectJAdvisors() from @Aspect beans
             findAdvisorsThatCanApply: AopUtils.canApply(advisor, class)  -> ClassFilter then MethodMatcher on ANY method
             extendAdvisors: adds ExposeInvocationInterceptor (first) when AspectJ advisors exist
             sortAdvisors: AnnotationAwareOrderComparator (@Order / Ordered)
   3  none apply -> advisedBeans[key]=FALSE, return bean untouched (NO proxy - that is why plain beans stay raw)
   4  createProxy(class, name, interceptors, SingletonTargetSource(bean))
        ProxyFactory pf; pf.copyFrom(this); decide proxyTargetClass / evaluateProxyInterfaces
        pf.addAdvisors(...); return pf.getProxy(classLoader)
```

`DefaultAopProxyFactory.createAopProxy(config)`:

```
if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
    targetClass = config.getTargetClass()
    if (targetClass.isInterface() || Proxy.isProxyClass(targetClass) || ClassUtils.isLambdaClass(targetClass))
         return new JdkDynamicAopProxy(config)
    return new ObjenesisCglibAopProxy(config)
}
return new JdkDynamicAopProxy(config)
```

**Boot default: `spring.aop.proxy-target-class=true` -> CGLIB even if the bean has interfaces.** Plain Spring default: JDK proxy when there are interfaces. Consequence in plain Spring: injecting by concrete class fails with `BeanNotOfRequiredTypeException ... but was actually of type 'jdk.proxy2.$Proxy45'` (older JDKs: `com.sun.proxy.$Proxy45`).

### 2.14.2 Pointcut matching

`AspectJExpressionPointcut` uses AspectJ's pointcut parser only (Spring AOP does **not** use AspectJ weaving). Supported designators: `execution`, `within`, `this`, `target`, `args`, `@target`, `@args`, `@within`, `@annotation`, plus Spring's `bean(idOrNamePattern)`. Anything else (`call`, `get`, `set`, `initialization`, `handler`, `cflow`...) -> `IllegalArgumentException: Unsupported pointcut primitive`.

```
execution( modifiers? ret-type declaring-type? name(params) throws? )
execution(* com.acme.service..*Service.*(..))           any method of any *Service under com.acme.service..
execution(public * *(..))                                  public methods
within(com.acme.web..*)                                    join points in types matched
@annotation(com.acme.Audited)                              methods carrying the annotation
@within(org.springframework.stereotype.Service)           types carrying the annotation
this(X) vs target(X)                                       this = proxy is-a X; target = target object is-a X
args(String,..) / args(id)                                 argument types / binding to advice parameters
bean(*Repository)                                          Spring-only, by bean name
Combine: &&  ||  !     named: @Pointcut("...") void svc(){}   -> @Before("svc() && @annotation(a)")
```

Matching is two-phase for performance: `ClassFilter.matches(Class)` decides *whether the bean is proxied at all* (done once per bean), `MethodMatcher.matches(Method, Class)` (static, cached per method in `AdvisedSupport.methodCache`) decides *whether this method has an advice chain*. Designators that need runtime values (`args(..)` binding, `@args`, `this`/`target` with dynamic checks, `@target`) make the matcher dynamic (`isRuntime()`), and every call evaluates it again (`InterceptorAndDynamicMethodMatcher`) - slightly slower.

Pitfalls: `@annotation(X)` does **not** see annotations declared on an *interface* method when the class method has none (Spring's own `@Transactional` lookup handles this, your aspect does not); overly broad `execution(* *(..))` proxies everything and can break; `@Aspect` alone does not register a bean - the aspect must be a bean (`@Component`).

### 2.14.3 Advice chain: `ReflectiveMethodInvocation.proceed()`

Each advice becomes an `org.aopalliance.intercept.MethodInterceptor` (via `DefaultAdvisorAdapterRegistry`; AspectJ advice classes `AspectJAroundAdvice`, `AspectJMethodBeforeAdvice`, `AspectJAfterAdvice`, `AspectJAfterReturningAdvice`, `AspectJAfterThrowingAdvice` implement it directly or through adapters). On a call:

```
proxy.method(args)
  JdkDynamicAopProxy.invoke  |  CglibAopProxy.DynamicAdvisedInterceptor.intercept
     - equals/hashCode special-cased (JDK), exposeProxy -> AopContext.setCurrentProxy(proxy)
     - chain = advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass)   (cached)
     - chain empty ? invoke target reflectively/directly
                   : new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain).proceed()
proceed():
     if (currentInterceptorIndex == chain.size() - 1)  return invokeJoinpoint();   // the real method
     interceptor = chain.get(++currentInterceptorIndex);
     if dynamic matcher: matches(...) ? interceptor.invoke(this) : proceed()
     else return interceptor.invoke(this);           // interceptor calls invocation.proceed() recursively
```

It is an **onion** implemented with recursion:

```
  call --> [ExposeInvocation] --> [Tx interceptor] --> [Log interceptor] --> TARGET.method()
  ret  <-- [ExposeInvocation] <-- [Tx: commit/rollback] <-- [Log: after]   <--
  index:        -1 -> 0                  1                      2         (index == size-1 => invokeJoinpoint)
```

### 2.14.4 Advice ordering

Across aspects: `@Order(n)` / `Ordered`; **lower number = higher precedence = outermost** (first in, last out). No `@Order` => `LOWEST_PRECEDENCE`, unspecified relative order.

Inside one `@Aspect` class (Spring 5.2.7+, AspectJ precedence rules), highest to lowest precedence: `@Around`, `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`. `@After` is implemented as `try/finally` *outside* `@AfterReturning/@AfterThrowing`, so on the way out AfterReturning/AfterThrowing run first, then After.

```
success:    Around-pre -> Before -> METHOD -> AfterReturning -> After -> Around-post
exception:  Around-pre -> Before -> METHOD(throws) -> AfterThrowing -> After -> Around-post (finally) -> exception continues
```
Two advice of the same type in the same aspect: order undefined -> split into two aspects with `@Order`. `@Around` that does not call `proceed()` short-circuits everything inside it (cache hit, security deny). An `@Around` that catches and swallows exceptions makes `@AfterThrowing` / tx rollback never see them.

### 2.14.5 JDK proxy vs CGLIB

| | JDK dynamic proxy | CGLIB (`ObjenesisCglibAopProxy`) |
|---|---|---|
| Mechanism | `java.lang.reflect.Proxy` generates `$ProxyN implements <interfaces>`; `InvocationHandler` = `JdkDynamicAopProxy` | generates **subclass** `Foo$$SpringCGLIB$$0` overriding non-final methods; `MethodInterceptor` = `DynamicAdvisedInterceptor` |
| Needs | at least one interface | non-final class, visible (non-private) methods |
| Cast to concrete class | `ClassCastException` / `BeanNotOfRequiredTypeException` | fine |
| Intercepts | only interface methods | public/protected/package-private (package-private only if same package + same classloader), not `private`, not `final`, not `static` |
| Final class | n/a | `IllegalArgumentException: Cannot subclass final class` |
| Final method | n/a | silently NOT intercepted; runs on the *proxy instance*, whose fields are uninitialized -> `NullPointerException` |
| Constructor | target constructed normally; proxy has none | since Spring 4.0 **Objenesis** instantiates the subclass without calling any constructor (falls back to the constructor if Objenesis fails or `spring.objenesis.ignore=true`); before 4.0 the constructor ran twice (target + proxy), hence side effects/log lines duplicated |
| State | proxy holds target reference | proxy instance's own fields are empty; it delegates to a *separate* target object |
| Kotlin | interfaces fine | classes are `final` by default -> use `kotlin-spring` (all-open) plugin |

Invariant: the **proxy is not the target** - two objects exist. `this` inside target methods = target, never the proxy. That single fact explains self-invocation.

### 2.14.6 Self-invocation and fixes

```java
@Service class OrderService {
    public void place()  { save(); }          // this.save(): direct call -> NO advice
    @Transactional public void save() { ... } // ignored when called from place()
}
```

Fixes, best first:
1. **Move `save()` to another bean** and call through it (proxy is used) - cleanest.
2. Programmatic: `TransactionTemplate` / `CacheManager` calls inside the method.
3. Self-injection: `@Autowired @Lazy OrderService self;` then `self.save()` (works with setter/field; circular-safe with `@Lazy`).
4. `AopContext.currentProxy()` with `@EnableAspectJAutoProxy(exposeProxy = true)` (couples code to Spring).
5. **AspectJ weaving mode**: `@EnableTransactionManagement(mode = AdviceMode.ASPECTJ)`, `@EnableAsync(mode = AdviceMode.ASPECTJ)`, `@EnableCaching(mode = AdviceMode.ASPECTJ)` + `spring-aspects` + compile-time (ajc) or load-time weaving agent.

### 2.14.7 Spring AOP vs AspectJ

| | Spring AOP | AspectJ |
|---|---|---|
| Weaving | runtime proxies, at bean creation (BPP after-init) | compile-time (`ajc`), post-compile (binary weaving of jars), load-time (`-javaagent:aspectjweaver.jar` / `@EnableLoadTimeWeaving`) |
| Join points | method execution on Spring beans | method call & execution, constructor, field get/set, static init, exception handlers, any object |
| Self-invocation, private/final/static | not advised | advised |
| Overhead | proxy + chain per call | code woven inline, no proxy |
| Setup | just annotations | extra compiler/agent |
| Use when | 95% of app cross-cutting (tx, cache, security, logging) | domain-object aspects (`@Configurable`), field access audits, non-bean code, self-invocation must work |

### 2.14.8 Typical uses and how they hook

| Feature | Implementation |
|---|---|
| Logging/timing/audit | custom `@Aspect` `@Around` |
| Transactions | `BeanFactoryTransactionAttributeSourceAdvisor` + `TransactionInterceptor` (`@EnableTransactionManagement`) |
| Caching | `BeanFactoryCacheOperationSourceAdvisor` + `CacheInterceptor` (`@EnableCaching`) |
| Method security | `AuthorizationManagerBeforeMethodInterceptor` etc. (`@EnableMethodSecurity`, Security 6) |
| Retry | spring-retry `@Retryable` -> `RetryOperationsInterceptor` (`@EnableRetry`); *outside* the tx interceptor |
| Async | `AsyncAnnotationBeanPostProcessor` adds `AnnotationAsyncExecutionInterceptor` (`@EnableAsync`); Boot auto-enables nothing - you must add `@EnableAsync` |
| Validation | `MethodValidationPostProcessor` (`@Validated` on class) |
| Metrics | Micrometer `TimedAspect` bean for `@Timed` |

## 2.15 Proxy-based features: `@Transactional`, `@Cacheable`, `@Async`

**Common contract**: works only on calls that arrive **through the proxy**, from another bean, on a method the proxy type can intercept. Any of: self-invocation, `private` method, `final` method/class (CGLIB), object created with `new`, call in a constructor/`@PostConstruct` of the *same* bean, bean not managed -> annotation silently ignored. (Spring 6.0+: class-based (CGLIB) proxies also honour `@Transactional` on protected/package-visible methods; JDK proxies still only interface methods.)

### `@Transactional`
- `TransactionInterceptor.invoke` -> `TransactionAspectSupport.invokeWithinTransaction`: get `TransactionAttribute` -> `PlatformTransactionManager` -> `getTransaction` (propagation) -> proceed -> commit; on `Throwable` `completeTransactionAfterThrowing`: rollback **only for `RuntimeException`/`Error`** by default; checked exceptions commit unless `rollbackFor`.
- Connection bound to thread via `TransactionSynchronizationManager` (ThreadLocal): `@Async`/new thread/parallel stream = new (no) transaction.
- `REQUIRES_NEW` in a self-invoked method does nothing (no proxy).
- `readOnly=true` is a hint (Hibernate flush mode MANUAL, driver hints), not a security barrier.
- Swallowing the exception inside the method -> commit; catching in an outer bean that shares the tx -> `UnexpectedRollbackException` when inner marked rollback-only.
- Advisor order: `@EnableTransactionManagement(order=...)` default `LOWEST_PRECEDENCE` -> innermost. Retry/logging aspects that should wrap the whole tx need higher precedence (`@Order(1)`).

### `@Cacheable` / `@CachePut` / `@CacheEvict`
- `CacheInterceptor` (`CacheAspectSupport.execute`): evict-before -> lookup key -> hit returns (method **not called**) / miss calls method and puts -> `@CachePut` always calls -> evict-after.
- Key: `SimpleKeyGenerator` (no params -> `SimpleKey.EMPTY`, one param -> that param, many -> `SimpleKey` of all). Specify `key="#id"` / `"#a0"`. Two different methods with same key in the same cache name **collide** - include a prefix or different cache names.
- Default provider without config: Boot auto-detects by classpath (Caffeine, Redis, ...), otherwise falls back to `ConcurrentMapCacheManager` - **unbounded, no TTL, per-JVM**.
- `unless` is evaluated **after** invocation (can use `#result`); `condition` before (cannot use `#result`).
- `null` results are cached by default (unless `unless="#result == null"` or cache disallows null).
- Cache stampede: N concurrent misses -> N calls; `sync = true` makes the cache lock per key (only for single cache, no `unless`).
- Cached object is a shared reference in local caches: mutating it corrupts the cache; Redis needs serializable/serializer config; cached JPA lazy entities blow up outside session - cache DTOs.
- Self-invocation, private methods, and calls within the same class miss the cache.

### `@Async`
- `AsyncAnnotationBeanPostProcessor` (an `AbstractAdvisingBeanPostProcessor`): if the bean is already an `Advised` proxy it **adds** the async advisor to that proxy (at the front, `beforeExistingAdvisors=true`), else creates a new `ProxyFactory` proxy. Because it wraps after init and does not take part in early references, it triggers the "raw version" circular error.
- `AsyncExecutionInterceptor` submits to executor named by `@Async("name")`, else a `TaskExecutor` bean (unique / named `taskExecutor`), else `SimpleAsyncTaskExecutor` (new thread per call, **no pool** - unbounded). Boot auto-configures `applicationTaskExecutor` (`ThreadPoolTaskExecutor`: core 8, unbounded queue by default - so max pool never grows); Boot 3.2+ `spring.threads.virtual.enabled=true` on JDK 21 switches to virtual threads.
- Return type: `void`, `Future`, `CompletableFuture`. `void` exceptions -> `AsyncUncaughtExceptionHandler` (default logs); wrap/handle inside.
- Lost across threads: `SecurityContext`, MDC, request scope, transaction, `RequestContextHolder` -> `TaskDecorator`.
- `@EnableAsync` default `proxyTargetClass=false` (JDK proxy on classes with interfaces in plain Spring).
- Graceful shutdown: set `setWaitForTasksToCompleteOnShutdown` / `awaitTermination`.

---

# 3. Traced worked examples

## 3.1 Full lifecycle with a custom BPP [TRACED]

```java
@Component
class Ticket implements BeanNameAware, ApplicationContextAware, InitializingBean, DisposableBean {
    Ticket()                                  { p("1 constructor"); }
    @Autowired void setClock(Clock c)         { p("2 setter injection"); }
    public void setBeanName(String n)         { p("3 BeanNameAware " + n); }
    public void setApplicationContext(ApplicationContext c) { p("4 ApplicationContextAware"); }
    @PostConstruct void pc()                  { p("5 @PostConstruct"); }
    public void afterPropertiesSet()          { p("7 afterPropertiesSet"); }
    void customInit()                         { p("8 init-method"); }     // @Bean(initMethod="customInit")
    @PreDestroy void pd()                     { p("10 @PreDestroy"); }
    public void destroy()                     { p("11 DisposableBean.destroy"); }
}
@Component class Spy implements BeanPostProcessor {      // ordinary BPP, no Ordered
    public Object postProcessBeforeInitialization(Object b, String n) { if (b instanceof Ticket) p("6 BPP.before"); return b; }
    public Object postProcessAfterInitialization (Object b, String n)  { if (b instanceof Ticket) p("9 BPP.after");  return b; }
}
```

Console (registration of `Ticket` as a `@Bean(initMethod="customInit")` or via XML init-method for line 8) [TRACED]:

```
1 constructor
2 setter injection
3 BeanNameAware ticket
4 ApplicationContextAware
5 @PostConstruct            <- BEFORE the custom BPP.before! (CommonAnnotationBPP is PriorityOrdered)
6 BPP.before
7 afterPropertiesSet
8 init-method
9 BPP.after
... on close ...
10 @PreDestroy
11 DisposableBean.destroy
```
The common recited "BPP.before, then @PostConstruct" is a simplification; @PostConstruct is itself a BPP.before. To run before it, make your BPP `PriorityOrdered` with a higher precedence than `Ordered.LOWEST_PRECEDENCE - 3`.

## 3.2 The three-level cache with a circular dependency [VERIFIED mini-container]

Beans: `A` (field-injects B) and `B` (field-injects A), singletons. Real Spring call trace (numbers = order):

```
 1 getBean(A) -> doGetBean(A) -> getSingleton(A): L1 miss, A not in creation -> null
 2 getSingleton(A, factory): beforeSingletonCreation -> inCreation = {A}
 3 createBean(A) -> doCreateBean: createBeanInstance -> raw A (fields null)
 4 addSingletonFactory(A, () -> getEarlyBeanReference(A, rawA))          L3 = {A}
 5 populateBean(A): needs B -> getBean(B)
 6   doGetBean(B): L1 miss -> beforeSingletonCreation inCreation = {A, B}
 7   doCreateBean(B): raw B; addSingletonFactory(B, ...)                  L3 = {A, B}
 8   populateBean(B): needs A -> getBean(A) -> getSingleton(A, true)
 9     L1 miss; A in creation -> L2 miss -> L3 hit: factory.getObject()
        = getEarlyBeanReference: AbstractAutoProxyCreator.wrapIfNecessary -> rawA OR PROXY(A)
10    L2 = {A -> early}; L3 = {B}                                          (promotion L3 -> L2)
11   B.a = early A;  initializeBean(B);  addSingleton(B) -> L1 = {B}; L2/L3 cleaned of B
12 back in A: A.b = B (complete);  initializeBean(A): BPP.after
13 AbstractAutoProxyCreator.postProcessAfterInitialization: early proxy already made -> returns bean unchanged
14 reconciliation: earlySingletonReference != null && exposedObject == bean -> exposedObject = early ref
15 addSingleton(A) -> L1 = {A, B}; L2 = {}; L3 = {}
```

Mini-container run [VERIFIED, `javac` + `java`, ~60 lines: three maps + `inCreation` set + supplier]:

```
=== 1. plain circular A<->B ===
create MiniIoc$A   inCreation=[]
  instantiated raw A
create MiniIoc$B   inCreation=[MiniIoc$A]
  instantiated raw B
  L3 -> L2 : early reference for MiniIoc$A = A(raw)
  L1 <- MiniIoc$B = B(raw)
  L1 <- MiniIoc$A = A(raw)
final: L1 A=A(raw) ; B.a=A(raw) ; same=true
=== 2. A is proxied, BPP is early-aware (AbstractAutoProxyCreator style) ===
  ...
  L3 -> L2 : early reference for MiniIoc$A = PROXY[A]
  L1 <- MiniIoc$B = B(raw)
  L1 <- MiniIoc$A = PROXY[A]
final: L1 A=PROXY[A] ; B.a=PROXY[A] ; same=true
=== 3. A proxied only after init, BPP not early-aware (@Async style) ===
  L3 -> L2 : early reference for MiniIoc$A = A(raw)
  L1 <- MiniIoc$B = B(raw)
FAIL: Bean 'MiniIoc$A' has been injected into other beans in its raw version as part of a circular reference, but has eventually been wrapped
```

Core of the demo (the interesting 25 lines):

```java
Object getSingleton(String name) {
    Object o = singletonObjects.get(name);                                  // L1
    if (o == null && inCreation.contains(name)) {
        o = earlySingletonObjects.get(name);                                // L2
        if (o == null) {
            Supplier<Object> f = singletonFactories.get(name);              // L3
            if (f != null) { o = f.get(); earlySingletonObjects.put(name, o); singletonFactories.remove(name); }
        }
    }
    return o;
}
// in createBean, right after `raw = newInstance()`:
singletonFactories.put(name, () -> wrap && earlyAware ? proxy(raw) : raw);   // getEarlyBeanReference
// after populate + init:
Object exposed = (wrap && !alreadyEarlyProxied) ? proxy(raw) : raw;          // BPP.afterInit
Object early = earlySingletonObjects.get(name);
if (early != null) { if (exposed == raw) exposed = early; else throw new IllegalStateException("...raw version..."); }
```
Take-away: scenario 3 is exactly what the message `has been injected into other beans [b] in its raw version` means: B holds raw A, the container ends up publishing PROXY A - two different A's.

**Constructor cycle trace** (`A(B)`, `B(A)`), Boot 3 message shape:
```
Error creating bean with name 'a': Requested bean is currently in creation: Is there an unresolvable circular reference?
   -> UnsatisfiedDependencyException: ... Error creating bean with name 'b' ...
```
With Boot 2.6+ even the *field* version fails at startup: `The dependencies of some of the beans in the application context form a cycle: a -> b -> a` (allowCircularReferences=false).

## 3.3 Advice chain as an onion [VERIFIED plain-Java simulation of `ReflectiveMethodInvocation`]

A JDK `Proxy` + an `Inv` class with `idx = -1`, `proceed()` recursion (exactly the algorithm in 2.14.3), two interceptors (Order1-Tx outer, Order2-Log inner), target `SvcImpl` whose `outer()` calls `this.pay()`:

```
--- proxy.pay()
  Order1-Tx around-before
  Order2-Log around-before
    >> pay() body
  Order2-Log around-after
  Order1-Tx around-after
result=OK
--- proxy.outer() (self-invocation)
  Order1-Tx around-before
  Order2-Log around-before
    >> outer() body, calls this.pay()
    >> pay() body                           <- NO interceptors around the inner call
  Order2-Log around-after
  Order1-Tx around-after
result=OK
```
The second block is self-invocation proven: the inner `pay()` ran with no Tx/Log frames because `this` is the target, not the proxy.

## 3.4 One aspect, all five advice types [TRACED]

```java
@Aspect @Component @Order(1)
class Trace {
    @Pointcut("execution(* com.acme.PayService.pay(..))") void p() {}
    @Around("p()")          Object ar(ProceedingJoinPoint j) throws Throwable { out("around-pre");  try { return j.proceed(); } finally { out("around-post"); } }
    @Before("p()")          void b()                          { out("before"); }
    @AfterReturning("p()")  void afr()                        { out("afterReturning"); }
    @AfterThrowing("p()")   void aft()                        { out("afterThrowing"); }
    @After("p()")           void a()                          { out("after"); }
}
@Aspect @Component @Order(2) class Audit { @Before("execution(* com.acme.PayService.pay(..))") void x(){ out("audit-before"); } }
```
`pay()` succeeds, Spring 6 [TRACED per 2.14.4]:
```
around-pre
before
audit-before          <- Order(2) is inner: enters after Order(1) advice has run
>> pay body
afterReturning
after
around-post
```
`pay()` throws `IllegalStateException`:
```
around-pre, before, audit-before, >> pay body (throws), afterThrowing, after, around-post, then exception propagates
```

## 3.5 Prototype in singleton: three fixes [TRACED]

```java
@Component @Scope("prototype") class Cart { final String id = UUID.randomUUID().toString().substring(0,4); }

@Service class Broken   { @Autowired Cart cart; String id(){ return cart.id; } }
// Broken.id() -> "a1f3","a1f3","a1f3"   (one Cart, injected once)

@Service class Provided { @Autowired ObjectProvider<Cart> carts; String id(){ return carts.getObject().id; } }
// -> "a1f3","7c02","e9b8"

@Service abstract class Looked { @Lookup abstract Cart cart(); String id(){ return cart().id; } }
// -> new each call ("@Lookup": CGLIB subclass overrides cart() to getBean(Cart.class))

@Component @Scope(value="prototype", proxyMode=ScopedProxyMode.TARGET_CLASS) class ProxCart { ... }
// singleton proxy; EVERY method call creates a new ProxCart -> proxy.setX(1); proxy.getX() returns default!
```

## 3.6 `@Configuration` full vs lite [TRACED]

```java
@Configuration                       class Full { @Bean A a(){ return new A(); } @Bean B b(){ return new B(a()); } }
@Configuration(proxyBeanMethods=false) class Lite { @Bean A a(){ return new A(); } @Bean B b(){ return new B(a()); } }
```
`ctx.getBean(B.class).a == ctx.getBean(A.class)` -> Full: `true`; Lite: `false` (B got a *third-party* `new A()` not managed by Spring; `A` was constructed twice; `@PostConstruct` on the second is never run). Fix Lite by `@Bean B b(A a)`.

## 3.7 Autowiring resolution outcomes [TRACED]

```java
interface Notifier {}
@Component class EmailNotifier implements Notifier {}
@Component @Primary class SmsNotifier implements Notifier {}
@Component class PushNotifier implements Notifier {}

@Autowired Notifier n;                          // SmsNotifier  (@Primary)
@Autowired Notifier pushNotifier;               // PushNotifier (fallback by field name; needs -parameters for ctor params)
@Autowired @Qualifier("emailNotifier") Notifier n2;   // EmailNotifier (qualifier beats @Primary)
@Autowired List<Notifier> all;                  // all three, order by @Order else registration
@Autowired Map<String, Notifier> byName;        // keys: emailNotifier, smsNotifier, pushNotifier
@Autowired Repository<User> r;                  // only beans whose generic is User
```
Two `@Primary` -> `NoUniqueBeanDefinitionException`. Zero -> `NoSuchBeanDefinitionException` (unless `@Autowired(required=false)`, `Optional`, `ObjectProvider`).

---

# 4. Failure modes and production war stories (symptom -> diagnosis -> fix)

| # | Symptom | Diagnosis | Fix |
|---|---|---|---|
| 1 | Upgrade to Boot 2.6+/3: app fails with "The dependencies of some of the beans ... form a cycle" | Boot flipped `allowCircularReferences` to false; the cycle existed for years hidden by field injection | Break the cycle: extract a third bean, use events or `ObjectProvider`; short-term `@Lazy` on one injection point. Not `spring.main.allow-circular-references=true` (masks) |
| 2 | `@Transactional` method does not roll back / no tx | self-invocation, `private`/`final`, `new`-created object, checked exception without `rollbackFor`, exception swallowed, wrong `@Transactional` import | Call through another bean; check `TransactionSynchronizationManager.isActualTransactionActive()`; enable `logging.level.org.springframework.transaction.interceptor=TRACE` |
| 3 | `BeanCurrentlyInCreationException: ... in its raw version ... eventually been wrapped` after adding `@Async`/some aspect | early-reference-unaware BPP wraps a bean that is in a cycle | Break cycle; `@Lazy` injection; put `@Async` method in own bean |
| 4 | NPE inside a service method that clearly has injected fields, only when called from another bean | method is `final` (CGLIB can't intercept; runs on proxy instance with uninitialized fields, Objenesis skipped the constructor); or Kotlin final class | Remove `final`; `kotlin-spring` plugin; or expose via interface |
| 5 | `BeanNotOfRequiredTypeException ... $ProxyNN` | JDK proxy (plain Spring or `proxyTargetClass=false`) injected by class | Inject by interface, or `@EnableAspectJAutoProxy(proxyTargetClass=true)` / Boot default |
| 6 | Prototype bean state is shared / stale | prototype injected once in a singleton | `ObjectProvider`, `@Lookup` |
| 7 | `IllegalStateException: No thread-bound request found` in `@Async`/scheduler/Kafka thread | request-scoped bean accessed off the request thread | Pass values explicitly; `TaskDecorator` copying `RequestAttributes`; avoid request scope beyond web layer |
| 8 | `@Cacheable` never hits | self-invocation, key includes unstable object (no `equals/hashCode`), wrong cacheName, `condition` false, `@EnableCaching` missing | Log `org.springframework.cache=TRACE`, use explicit `key`, DTOs with equals |
| 9 | Cache returns wrong data for two methods | same cache name + same default key (single param `1L`) | key prefix `key="'user:'+#id"` or separate caches |
| 10 | Thundering herd on cache expiry | N misses call DB | `sync=true` / Caffeine refresh / single-flight loader |
| 11 | `@TransactionalEventListener` never fires | no active tx at publish time | publish inside tx; or `fallbackExecution=true` |
| 12 | Order confirmation event listener failure rolls back the order | sync `@EventListener` runs in publisher's tx and thread | `@TransactionalEventListener(AFTER_COMMIT)` + `@Async`, or try/catch in listener |
| 13 | Retry aspect retries but data written twice / "transaction marked rollback-only" | Retry advice *inside* tx advice (both LOWEST_PRECEDENCE) | `@Order` retry lower number than tx (outermost) so each attempt gets a fresh tx |
| 14 | `Bean 'x' is not eligible for getting processed by all BeanPostProcessors` and its `@Transactional`/`@Async` ignored | bean is a BPP or a dependency of one (e.g. a BPP `@Autowired` service) | Make it lazy-lookup via `ObjectProvider` in the BPP; make `@Bean` BFPP/BPP methods `static` |
| 15 | `@Autowired`/`@Value` null in a `@Configuration` class | config class instantiated early because a non-static BFPP `@Bean` lives in it | `static @Bean` for BFPP/BPP |
| 16 | Works locally, `NoUniqueBeanDefinitionException` in another module | a library added a second implementation; fallback-by-name broke (parameter names missing, Spring 6.1 no longer reads debug info) | `@Primary`/`@Qualifier` explicitly; compile with `-parameters` |
| 17 | `@Bean` singleton created twice in `@Configuration(proxyBeanMethods=false)` | inter-bean method call in lite mode | pass beans as method params |
| 18 | `@ConditionalOnBean` doesn't match in my `@Configuration` | evaluation order: bean definition not yet registered | move to auto-config or use `@ConditionalOnProperty`; or ordering via `@AutoConfiguration(after=...)` |
| 19 | Aspect not applied | aspect not a bean, wrong pointcut (`within` vs `execution`), advised bean was created before `AbstractAutoProxyCreator` was registered (BPP dependency), call via `this`, method `final`/`private`, `@Aspect` class not scanned | Check with `AopUtils.isAopProxy(bean)`, `--debug`, logging `org.springframework.aop=DEBUG` |
| 20 | Startup 60 s | eager creation of hundreds of beans, heavy `@PostConstruct` (remote calls), classpath scanning breadth | `spring.main.lazy-initialization=true` (with care: failures move to runtime), move IO out of `@PostConstruct`, restrict scan packages, Boot startup actuator endpoint (`ApplicationStartup`) |
| 21 | `@Async` method runs on caller thread | self-invocation, missing `@EnableAsync`, `void`+exception... | proxy rules |
| 22 | OOM with `@Async` default executor in old plain-Spring app | `SimpleAsyncTaskExecutor` spawns thread per call | define pooled `ThreadPoolTaskExecutor` with bounded queue + rejection policy |
| 23 | Memory leak with prototype beans holding resources | container does not call `@PreDestroy` for prototypes; also prototype registered as listener | clean up manually, or `DestructionAwareBeanPostProcessor`, or avoid prototype |
| 24 | Duplicate `ContextRefreshedEvent` handling | parent + child contexts (MVC `DispatcherServlet` + root) | check `event.getApplicationContext()`; prefer `ApplicationReadyEvent` |
| 25 | `spring.main.allow-bean-definition-overriding` needed after adding a starter | two definitions with same bean name (different config classes/`@Component` name clash `com.a.Cfg` vs `com.b.Cfg`) | give explicit unique names; don't enable overriding |
| 26 | Property change in `application.yml` has no effect | higher-precedence source (env var, `-D`, command line, K8s ConfigMap) is overriding | print `/actuator/env` (shows each source), remember precedence table |
| 27 | `@Value` static field is null | injection occurs on instances only | inject into instance setter and assign static, or `@ConfigurationProperties` |

**War story A - "the transaction that wasn't"**: Batch service method `process()` (no tx) looped calling `this.saveOne(item)` annotated `@Transactional(REQUIRES_NEW)` to isolate failures. In prod, one failing item polluted the persistence context and every following item failed. Root cause: self-invocation, so `REQUIRES_NEW` never happened; all items ran in one (or no) tx with one EntityManager. Fix: extract `ItemSaver` bean; loop calls `itemSaver.saveOne(item)` through the proxy.

**War story B - "Boot upgrade, startup dead"**: 3 services `OrderService <-> InventoryService <-> PricingService` wired by `@Autowired` fields for 6 years. Boot 2.5->2.7 upgrade failed at startup. Team's quick fix `allow-circular-references=true` shipped; three months later adding `@Async` to `InventoryService` produced the raw-version error. Real fix: `PricingService` needed only a price *lookup* -> extracted `PriceCatalog` (no back reference); cycle gone, flag removed.

**War story C - "final on a method killed the pod"**: A refactor by IDE-suggestion made helper methods `final` in a `@Service` with `@Transactional` class-level. Calls from controller ran the final method on the CGLIB proxy instance (fields null due to Objenesis), NPE only in the final method, and the tx was skipped. Detection: heap dump showed two instances; `ClassName$$SpringCGLIB$$0` with null fields.

**War story D - "cache poisoning via mutable object"**: `@Cacheable` on a service returning `List<Product>` from a `ConcurrentMapCache`. A caller sorted the list in place, mutating the cached reference; other users saw sorted output/`ConcurrentModificationException`. Fix: return unmodifiable copies (`List.copyOf`) or use a serializing cache (Redis) or DTO records.

---

# 5. Interview questions (40)

Format: **Qn [Level]** question - model answer - follow-ups - wrong answer to avoid.

**Q1 [Easy] What is IoC and DI? How does Spring implement it?**
A: IoC = the framework, not your code, controls creation and wiring of objects. DI is the mechanism: dependencies are supplied (constructor/setter/field). Spring implements it with an `ApplicationContext` that reads `BeanDefinition`s and builds beans using a `DefaultListableBeanFactory`. Follow-ups: IoC vs DI? (IoC is the principle, DI one implementation; service locator is another). How is it different from a factory? Wrong: "IoC means Spring uses `new` for you" (it uses reflection/constructor resolution, and IoC is broader).

**Q2 [Easy] BeanFactory vs ApplicationContext.**
A: BeanFactory is the root interface (`getBean`...). ApplicationContext extends it plus events, i18n, resources, environment/profiles, annotation config, BPP auto-detection, eager singleton init. Boot always uses an ApplicationContext; it delegates to a `DefaultListableBeanFactory`. Follow-ups: Which is lazy? (bare BeanFactory). Does ApplicationContext *contain* or *extend*? Both. Wrong: "BeanFactory is deprecated".

**Q3 [Easy] Name the bean scopes and defaults.**
A: singleton (default, per container), prototype, request, session, application, websocket, custom. Follow-ups: singleton scope vs GoF singleton? (per container per definition, not per class/JVM: two beans of the same class with different names = two instances). Wrong: "singleton = thread-safe".

**Q4 [Easy] Constructor vs setter vs field injection - which and why?**
A: Constructor for mandatory deps: immutability, testability, fail-fast, cycles are detected. Setter for optional/reconfigurable. Field: avoid (hidden deps, hard testing, allows cycles). Follow-ups: is `@Autowired` needed on a single ctor? (no, 4.3+). What if many deps? (SRP smell). Wrong: "field injection is faster".

**Q5 [Easy] `@Component` vs `@Service` vs `@Repository` vs `@Controller`.**
A: All are stereotypes (meta-annotated `@Component`) detected by scanning. `@Repository` also enables persistence-exception translation (`PersistenceExceptionTranslationPostProcessor` proxy); `@Controller` is recognised by MVC handler mapping; `@Service` is semantic only. Follow-up: how does scanning detect a meta-annotation? (ASM metadata, `AnnotationTypeFilter` with `considerMetaAnnotations`). Wrong: "they differ in scope".

**Q6 [Easy] `@Bean` vs `@Component`.**
A: `@Component` on your own class, discovered by scan; `@Bean` on a method in a config class, for third-party classes or complex construction; gives full control of name/init/destroy. Follow-up: can I mix? yes. Wrong: "@Bean is for prototypes".

**Q7 [Easy] What does `@SpringBootApplication` do?**
A: `@SpringBootConfiguration` (+`@Configuration`), `@EnableAutoConfiguration` (imports `AutoConfigurationImportSelector`), `@ComponentScan` (with exclude filters for `TypeExcludeFilter`/`AutoConfigurationExcludeFilter`). Follow-up: how does auto-config choose classes? (`.imports` file + conditions). Wrong: "it starts Tomcat" (Tomcat starts in `onRefresh`/`finishRefresh`).

**Q8 [Easy] Explain the bean lifecycle.**
A: Instantiate -> populate -> Aware -> BPP before (includes `@PostConstruct`) -> `afterPropertiesSet` -> init-method -> BPP after (proxy) -> ready -> `@PreDestroy` -> `destroy()` -> destroy-method. Follow-ups: where do proxies appear? (BPP after). Do prototypes get destroy callbacks? (no). Wrong: "@PostConstruct runs before injection".

**Q9 [Medium] Walk through `refresh()`.**
A: Use the 12-step list in 2.2; emphasise: BFPP step processes config classes -> definitions; BPPs registered before any ordinary bean; `onRefresh` creates web server; singletons pre-instantiated; `finishRefresh` starts lifecycle beans, publishes `ContextRefreshedEvent`. Follow-ups: where are `@Bean` definitions registered? (`ConfigurationClassPostProcessor`, step 5). Why are BPPs created before other beans? (so they can process them). When does Tomcat accept traffic? (finishRefresh). Wrong: "beans are created lazily when first requested" (singletons eager).

**Q10 [Medium] What is `ConfigurationClassPostProcessor` and what does it process?**
A: A `PriorityOrdered` `BeanDefinitionRegistryPostProcessor` that parses `@Configuration`/lite candidates: `@PropertySource`, `@ComponentScan`, `@Import` (selectors, registrars), `@ImportResource`, `@Bean`, then registers definitions; later CGLIB-enhances full config classes. Follow-ups: where does auto-config get processed? (deferred import selector, last). Why static `@Bean` for BFPP? Wrong: "it's a BeanPostProcessor".

**Q11 [Medium] BeanFactoryPostProcessor vs BeanPostProcessor.**
A: BFPP: acts on definitions, before instances (`PropertySourcesPlaceholderConfigurer`, `ConfigurationClassPostProcessor`). BPP: acts on instances around init (`AutowiredAnnotationBeanPostProcessor`, `AbstractAutoProxyCreator`). Follow-ups: can a BFPP `getBean`? technically yes but forces premature instantiation - bypasses BPPs. `BeanDefinitionRegistryPostProcessor`? Registers definitions too. Wrong: "BPP runs before instantiation".

**Q12 [Medium] Where is `@Autowired` processed?**
A: `AutowiredAnnotationBeanPostProcessor`, an `InstantiationAwareBeanPostProcessor` + `MergedBeanDefinitionPostProcessor`: caches injection metadata (merged phase), picks constructors (`determineCandidateConstructors`), injects fields/methods in `postProcessProperties` (populateBean). Follow-ups: bare BeanFactory? not applied unless registered. `@Resource`? `CommonAnnotationBeanPostProcessor`. Wrong: "reflection in ApplicationContext.refresh directly".

**Q13 [Medium] How does Spring resolve which bean to inject when several match?**
A: Algorithm in 2.6: candidates by type + generics + qualifiers; then `@Primary`, `@Priority`, dependency-name fallback; else `NoUniqueBeanDefinitionException`. Follow-ups: `@Qualifier` vs `@Primary` conflict? qualifier wins. Map/List injection? all beans. Generics? resolved from `ResolvableType`. Name fallback in ctor params? needs `-parameters`. Wrong: "always by bean name".

**Q14 [Medium] Why do `@Bean` method calls inside `@Configuration` return the same instance?**
A: Full-mode config classes are replaced with CGLIB subclasses; `BeanMethodInterceptor` checks if the call is the container's own factory-method call; if not it does `getBean(name)`. Follow-ups: what breaks it? `proxyBeanMethods=false`, `@Component` host, `final` class. Why does Boot use `proxyBeanMethods=false`? Startup/memory; pass params. Wrong: "because singleton scope caches method results" (plain Java has no idea).

**Q15 [Medium] Explain `@Lazy`.**
A: On a bean/definition: skip eager instantiation. On an injection point: inject a lazy-resolution proxy. On `@Configuration`: applies to all beans. Follow-ups: how does it fix a constructor cycle? proxy breaks the need for an instance. Downsides: failures at first use; proxy needs non-final type; hides design flaw. Wrong: "`@Lazy` makes the bean prototype".

**Q16 [Medium] Prototype in singleton - the problem and solutions.**
A: Injected once; see 2.9: `ObjectProvider`, `@Lookup`, scoped proxy (new per call), `getBean`. Follow-ups: request-scoped in singleton? scoped proxy with `scopedTarget.` bean. Which is best? ObjectProvider. Wrong: "use `@Scope("prototype")` on the singleton".

**Q17 [Medium] How do request-scoped beans work when injected into singletons?**
A: Singleton gets a CGLIB/JDK singleton proxy (`ScopedProxyFactoryBean`); each call resolves `scopedTarget.name` from the current thread's `RequestAttributes`. Follow-ups: outside request? `IllegalStateException`; `@Async`? attributes are ThreadLocal, lost. Wrong: "Spring creates a new singleton per request".

**Q18 [Medium] Explain `FactoryBean`, and `&`.**
A: 2.10. `getBean("x")` product, `getBean("&x")` factory. Follow-ups: FactoryBean vs `@Bean`? equivalent for simple creation, FactoryBean can be generic infra with type determination / used by Registrars. Where used? `LocalContainerEntityManagerFactoryBean`, Feign. Wrong: "FactoryBean is same as BeanFactory".

**Q19 [Medium] How does `@Conditional` work? Give Boot examples.**
A: `Condition.matches` evaluated by `ConditionEvaluator` while parsing config classes and registering definitions; `@Profile`, `@ConditionalOnClass`, `OnBean`, `OnProperty`, `OnMissingBean`. Follow-ups: why is `OnBean` order sensitive? depends on already-registered definitions; why does `OnClass` not blow up? ASM metadata. How do I debug? `--debug`, `/actuator/conditions`. Wrong: "runtime check on every request".

**Q20 [Medium] `@Import` variants and the `@Enable*` pattern.**
A: 2.11 (class, `ImportSelector`, `DeferredImportSelector`, `ImportBeanDefinitionRegistrar`). Follow-ups: how does `@MapperScan`/`@EnableJpaRepositories` create beans for interfaces? Registrar registers FactoryBean-based definitions. Deferred vs normal selector? runs after all other configuration. Wrong: "`@Import` only imports XML".

**Q21 [Medium] How does event publishing work? Sync or async?**
A: `ApplicationEventPublisher` -> multicaster; sync, publisher thread, exceptions propagate, same tx. Async only via executor on multicaster or `@Async` listener. Follow-ups: ordering? `@Order`; return value publishes another event; `@TransactionalEventListener` phases; event dropped without tx. Wrong: "events are async by default".

**Q22 [Medium] Property source precedence in Boot.**
A: Command line > system props > env vars > profile-specific files > application files > `@PropertySource` > defaults (full table 2.13). Follow-ups: env var name mapping? relaxed binding (`SERVER_PORT`). `@PropertySource` with YAML? not supported natively (needs factory). Wrong: "application.properties always wins".

**Q23 [Medium] `@Value` vs `@ConfigurationProperties`.**
A: `@Value` single SpEL/placeholder, no relaxed binding for SpEL, weak typing; `@ConfigurationProperties` type-safe object, relaxed binding, validation (`@Validated`), metadata, immutable constructor binding. Both resolved once at bean creation. Follow-up: dynamic refresh? Spring Cloud `@RefreshScope`. Wrong: "`@Value` re-reads on file change".

**Q24 [Medium] JDK proxy vs CGLIB - selection and limits.**
A: 2.14.5, `DefaultAopProxyFactory` logic, Boot default `proxyTargetClass=true`. Follow-ups: final method? not advised; Objenesis? no constructor call; why NPE with final? Wrong: "CGLIB modifies the class bytecode" (it generates a subclass).

**Q25 [Medium] Advice types and their order within an aspect.**
A: Around > Before > (method) > AfterReturning/AfterThrowing > After > Around-post; 5.2.7+ rules; across aspects `@Order`. Follow-ups: what if around swallows exception? afterThrowing never fires. Two `@Before` same aspect? undefined. Wrong: "`@After` runs only on success" (that's AfterReturning).

**Q26 [Medium] What is the self-invocation problem and fixes?**
A: 2.14.6. Follow-ups: exposeProxy details; AspectJ mode requirements; why does field-injected self reference work? proxy injected. Wrong: "make method public".

**Q27 [Medium] `@Cacheable` pitfalls.**
A: Section 2.15: self-invocation, key collisions, null caching, mutable shared objects, `unless` vs `condition`, default unbounded `ConcurrentMapCache`, stampede `sync=true`, serialization. Follow-ups: eviction with `@CacheEvict(allEntries)` cost; TTL? provider-specific. Wrong: "@Cacheable caches per thread".

**Q28 [Medium] `@Async` pitfalls.**
A: 2.15: needs `@EnableAsync`, proxy rules, default executor, `void` exceptions, lost context, circular raw-version. Follow-ups: how to return result? `CompletableFuture`. Config executor? `AsyncConfigurer` / named executor. Wrong: "@Async uses a pool by default in plain Spring" (SimpleAsyncTaskExecutor: no pool).

**Q29 [Hard] Explain the three-level cache and why exactly three.**
A: 2.8: L1 finished, L2 early references, L3 `ObjectFactory` producing early reference lazily via `getEarlyBeanReference` so AOP proxy is created only when needed; L2 caches so all consumers see one object. Trace of A<->B (3.2). Follow-ups: can it work with two levels? yes but proxies created eagerly for every bean or break BPP-after design. What does `earlyProxyReferences` do? prevents double proxy. Wrong: "L2 is the thread-local cache" / "three caches for performance".

**Q30 [Hard] Why do constructor circular dependencies fail but setter ones work?**
A: L3 factory registered after instantiation; ctor cycle needs the peer *during* instantiation, so no reference exists -> `BeanCurrentlyInCreationException`. Setter/field: instance exists (raw) before populate. Follow-ups: prototype cycles? always fail. With `@Lazy`? proxy provides. Boot 2.6+? banned. Mixed: A ctor->B, B setter->A? works if the creation starts with B? Depends on entry: start at A: A ctor needs B -> B created, B setter needs A -> A not exposed yet -> fails; start at B: B raw exposed -> A gets B -> works. Order-dependent (bean registration order, injection order) = fragile. Wrong: "constructor injection is not supported for cycles because of Java".

**Q31 [Hard] What is `has been injected into other beans in its raw version`?**
A: 2.8 reconciliation. Early ref was handed out (raw) and after init a BPP produced a different wrapper. Typical `@Async`. Follow-ups: why does `@Transactional` not trigger it? AutoProxyCreator supports early references. Fix options. `allowRawInjectionDespiteWrapping`? disables check, but consumers then hold un-advised object - silent bug. Wrong: "duplicate bean names".

**Q32 [Hard] How does `AbstractAutoProxyCreator` decide to proxy, and when?**
A: 2.14.1: `wrapIfNecessary` in `getEarlyBeanReference` or `postProcessAfterInitialization`; finds candidate advisors (Advisor beans + `@Aspect`), filters via ClassFilter + MethodMatcher, sorts, creates `ProxyFactory` proxy; caches negative. Follow-ups: why is a bean that is a BPP dependency not proxied? created before BPP registered. Which advisors would `@Transactional` add? `BeanFactoryTransactionAttributeSourceAdvisor`. Wrong: "proxies are created at first method call" (except lazy-init/`@Lazy` cases they are created at bean creation).

**Q33 [Hard] Trace what happens on a call to an advised method.**
A: 2.14.3: proxy -> (`JdkDynamicAopProxy.invoke` / `DynamicAdvisedInterceptor.intercept`) -> chain lookup (cached) -> `ReflectiveMethodInvocation.proceed` recursion with `currentInterceptorIndex`, last -> `invokeJoinpoint`. Follow-ups: how does `@Around` proceed? `ProceedingJoinPoint.proceed()` -> `invocation.proceed()`. Where does `ExposeInvocationInterceptor` fit? first; ThreadLocal current invocation. Dynamic pointcuts? per-call match. Wrong: "each advice is a separate proxy layer" (one proxy, one chain).

**Q34 [Hard] `@Configuration` full vs lite - internals, trade-offs.**
A: 2.7. Include `ConfigurationClassEnhancer`, `BeanMethodInterceptor`, `SimpleInstantiationStrategy.currentlyInvokedFactoryMethod`, Boot auto-config `proxyBeanMethods=false`, GraalVM/AOT (Spring AOT requires it - AOT-generated code handles this). Follow-ups: what if a `@Bean` method is `private`? not allowed in full mode. Wrong: "lite mode can't have @Bean".

**Q35 [Hard] How do you design around CGLIB proxy limitations (final, constructors, fields)?**
A: Avoid `final`, keep logic in methods not fields access from outside, keep constructors light (don't rely on them running on the proxy), expose interfaces for JDK proxies, Kotlin all-open, avoid `@Transactional` at private level, prefer constructor injection into the *target* (proxy has no fields). Follow-ups: how does the proxy get its target? `SingletonTargetSource` in `AdvisedSupport`. What is Objenesis? library bypassing constructors via `sun.reflect`/`Unsafe`-based instantiators. Wrong: "the constructor runs twice" (pre-4.0 only).

**Q36 [Hard] Pointcut expression evaluation - how does Spring decide, and what is expensive?**
A: 2.14.2: `AspectJExpressionPointcut` uses AspectJ parser; class filter once per bean, static method match cached, runtime `args()/@args()/this/target` dynamic re-evaluated each call. Follow-ups: cost of proxies? negligible relative to IO; startup cost of matching all methods of all beans with broad pointcuts (`execution(* *(..))`) is real. Wrong: "AspectJ compiler runs at startup".

**Q37 [Hard] Spring AOP vs AspectJ - when do you leave Spring AOP?**
A: 2.14.7: need field/constructor join points, self-invocation, non-bean objects, private methods, high-frequency hot paths where proxy overhead matters; weaving modes CTW/binary/LTW. Follow-ups: `@Configurable`. How do `@Transactional` semantics change in ASPECTJ mode? applies to any call incl. internal, private; needs weaver. Wrong: "AspectJ is just the annotation style".

**Q38 [Hard] Explain BPP ordering nuance: where does `@PostConstruct` sit relative to a custom BPP?**
A: 3.1: `CommonAnnotationBeanPostProcessor` is `PriorityOrdered` -> runs first; custom un-ordered BPP.before afterwards. Also `ApplicationContextAwareProcessor` first. Follow-ups: how to make a BPP run before @PostConstruct? `PriorityOrdered` with higher precedence. After-init ordering matters for proxy layering: an `Ordered` BPP running after `AbstractAutoProxyCreator` receives the proxy (and may wrap it). Wrong: "`@PostConstruct` is called by the container directly".

**Q39 [Hard] Why does Boot's `@ConditionalOnMissingBean` work in auto-config but is flaky in user config?**
A: Auto-config classes are processed via `DeferredImportSelector` after all user config definitions are registered; user config order vs. component scan order otherwise decides visibility (definitions registered when the config class is loaded). Follow-ups: `@AutoConfiguration(before/after)`; `@ConditionalOnMissingBean` type resolution can force early class/factory-bean init - avoid on FactoryBean-produced types. Wrong: "OnMissingBean checks instances".

**Q40 [Hard] Design question: startup is slow and you must add an async cross-cutting feature without breaking cycles. What do you check?**
A: Startup: `ApplicationStartup`/actuator startup endpoint, heavy `@PostConstruct`, scan breadth, lazy-init selectively, AOT/native or CDS. Cross-cutting: prefer `@Around` aspect with executor in its own bean; ensure it's early-ref-safe (use `AbstractAutoProxyCreator` advisors rather than wrapping BPPs), avoid putting `@Async` on beans in cycles, set order relative to tx (async boundary = new thread = new tx; context propagation via `TaskDecorator`/ Micrometer `ContextSnapshot`). Follow-ups: virtual threads (JDK 21, Boot 3.2 `spring.threads.virtual.enabled`), pinning with `synchronized`. Wrong: "just add more heap".

### Common wrong answers (quick list)
- "`@Transactional` works on any method" / "on private methods too".
- "Spring uses `new`" / "singleton means one per JVM".
- "`@PostConstruct` runs before dependency injection".
- "`@After` = only when successful".
- "Circular dependency isn't possible in Spring" (works for singleton setter/field pre-2.6) or "always fails".
- "BeanFactory is the same as FactoryBean".
- "CGLIB proxies call the constructor twice" (obsolete since 4.0).
- "Boot uses JDK proxies by default" (CGLIB since 2.0).
- "`@Async` uses a thread pool automatically" (not in plain Spring).
- "Prototype beans are destroyed by Spring".
- "`@Component` classes are always proxied" (only when an advisor applies).
- "`@EventListener` is async".

---

# 6. One-page cheat sheet

```
REFRESH:  prepareRefresh > obtainFreshBeanFactory > prepareBeanFactory > postProcessBeanFactory
          > invokeBFPPs (ConfigurationClassPostProcessor!) > registerBPPs > initMessageSource
          > initEventMulticaster > onRefresh (web server created) > registerListeners
          > finishBeanFactoryInitialization (singletons) > finishRefresh (lifecycle start, ContextRefreshedEvent)

BEAN:     ctor > inject > BeanName/BeanFactory aware > ApplicationContext aware > @PostConstruct
          > BPP.before > afterPropertiesSet > init-method > BPP.after (PROXY) > ready
          > @PreDestroy > destroy() > destroy-method      (prototype: no destroy)

CACHE:    L1 singletonObjects (done) | L2 earlySingletonObjects (early ref given) | L3 singletonFactories (lazy early-ref maker)
          ctor cycle => no instance to expose => BeanCurrentlyInCreationException
          Boot 2.6+: allowCircularReferences=false  | fix: redesign, @Lazy on injection point
          "raw version" error = early raw ref handed out + BPP (e.g. @Async) wrapped later

AUTOWIRE: type > generics > @Qualifier > @Primary > @Priority > name fallback (needs -parameters) > NoUnique/NoSuch
CONFIG:   @Configuration(full)=CGLIB, @Bean calls -> getBean | proxyBeanMethods=false/@Component=lite -> plain call
          static @Bean for BFPP/BPP | pass beans as method params
SCOPES:   prototype-in-singleton: ObjectProvider | @Lookup | scoped proxy (new per CALL) | getBean
          request/session into singleton: scoped proxy; no request thread => IllegalStateException
EVENTS:   sync, same thread, same tx, exception stops others | @TransactionalEventListener needs tx | @Async loses context
PROPS:    cmd line > -D > env > profile file (outside>inside) > application file > @PropertySource > defaults
CONDITIONS: ConfigurationClassParser + reader; OnBean order-sensitive; auto-config = DeferredImportSelector
IMPORT:   class | ImportSelector | DeferredImportSelector | ImportBeanDefinitionRegistrar

AOP:      AbstractAutoProxyCreator: early ref or BPP.after -> wrapIfNecessary -> advisors (ClassFilter+MethodMatcher) -> ProxyFactory
          JDK proxy (interfaces) vs CGLIB subclass (Boot default; no final/private; Objenesis, no ctor call)
          call: proxy -> chain -> ReflectiveMethodInvocation.proceed (index recursion) -> target
          order: lower @Order = outer. within aspect: Around, Before, [method], AfterReturning/Throwing, After
          self-invocation = no advice: other bean | self-inject @Lazy | AopContext | AspectJ mode
          Spring AOP: proxies, method execution on beans | AspectJ: CTW/binary/LTW, everything
PROXY FEATURES (@Transactional/@Cacheable/@Async): via proxy only, public/non-final, not self-call, tx rollback only unchecked,
          cache: key collisions/null/mutable/sync, async: executor choice, void exceptions, lost ThreadLocals

DEBUG:    --debug (conditions report), /actuator/conditions, /actuator/beans, /actuator/env, /actuator/startup,
          logging.level.org.springframework.aop=DEBUG, ...transaction.interceptor=TRACE, ...cache=TRACE,
          AopUtils.isAopProxy(x), AopProxyUtils.ultimateTargetClass(x), ctx.getBeanDefinition("x")
```
