# Spring Boot 3.x Internals: Startup, Auto-Configuration, Config, Server, Actuator, Native

> Target: 5-year Java developer, JDK 17/21, Spring Boot 3.x (3.0 - 3.4 era).
> Scope: **Boot-specific** machinery. The IoC container (`refresh()`, BeanPostProcessors, AOP proxies,
> bean lifecycle) is covered in `Spring_Container_Internals_Deep.md` - only referenced here where Boot hooks in.
> Convention: class names below are real; where a detail differs between minor versions it is flagged.

---

## Table of contents
1. 60-second mental model + analogy
2. Deep internals
   - 2.1 `SpringApplication` constructor
   - 2.2 `run()` step by step + event order
   - 2.3 `@SpringBootApplication` anatomy
   - 2.4 Auto-configuration mechanism
   - 2.5 Conditions and the evaluation report
   - 2.6 `@ConfigurationProperties` and the `Binder`
   - 2.7 Property source precedence
   - 2.8 Profiles, `spring.config.import`, Kubernetes config
   - 2.9 Embedded server (Tomcat and friends)
   - 2.10 `DispatcherServlet` request flow in Boot
   - 2.11 Actuator
   - 2.12 Logging
   - 2.13 DevTools
   - 2.14 Graceful shutdown
   - 2.15 Fat jar, layers, Docker
   - 2.16 AOT and GraalVM native image
   - 2.17 Startup time optimisation
   - 2.18 Boot 2 -> 3 migration
3. Traced worked examples (DataSource x4 situations, custom starter in full)
4. Failure modes and production war stories
5. Interview questions (40) with follow-ups and wrong answers
6. One-page cheat sheet

---

# 1. 60-second mental model

Spring Boot is **not a new framework**. It is Spring Framework plus four things:

1. **A launcher** (`SpringApplication.run`) that builds an `Environment`, picks the right `ApplicationContext`
   type, runs `refresh()`, starts the embedded web server inside that refresh, then runs your runners.
2. **Opinionated defaults expressed as conditional configuration** ("auto-configuration"): hundreds of
   `@Configuration` classes, each guarded by `@Conditional...`, listed in one text file per jar.
   They are imported **after** your own configuration, and back off when you define your own bean.
3. **Typed, layered, externalised configuration** (`Environment` + ordered `PropertySource`s +
   `Binder` -> `@ConfigurationProperties`).
4. **Production packaging and operations**: executable fat jar, Actuator (health/metrics/loggers),
   graceful shutdown, layered images, AOT/native.

**Analogy: a smart hotel.**
- The hotel (Boot) has a room template per guest type. The **front desk checks what you brought**
  (classpath: `@ConditionalOnClass`), **what you told them** (properties: `@ConditionalOnProperty`) and
  **whether you already have your own gear** (`@ConditionalOnMissingBean`).
- You brought a JDBC driver and a URL -> you get Wi-Fi (a Hikari `DataSource`).
- You brought your own router (your own `DataSource` bean) -> the hotel's router is **not** installed.
- Your own furnishings are placed first (`DeferredImportSelector`), the hotel fills in the rest afterwards -
  otherwise "does the guest already have a router?" could not be answered.
- Room service menu = starters (dependency bundles), the guest register = the condition evaluation report.

One-line flow:

```
main -> new SpringApplication(...)            (deduce web type, load initializers/listeners from spring.factories)
     -> run(): Environment -> Context -> refresh (scan + autoconfig + start Tomcat) -> runners -> Ready
```

---

# 2. Deep internals

## 2.1 The `SpringApplication` constructor

`SpringApplication.run(App.class, args)` is `new SpringApplication(App.class).run(args)`.
The constructor (`SpringApplication(ResourceLoader, Class<?>... primarySources)`) does, in order:

```
1. this.resourceLoader   = resourceLoader (null by default)
2. this.primarySources   = your @SpringBootApplication class(es)
3. this.webApplicationType = WebApplicationType.deduceFromClasspath()
4. this.bootstrapRegistryInitializers = getSpringFactoriesInstances(BootstrapRegistryInitializer)
5. setInitializers( getSpringFactoriesInstances(ApplicationContextInitializer) )
6. setListeners  ( getSpringFactoriesInstances(ApplicationListener) )
7. this.mainApplicationClass = deduceMainApplicationClass()   // walks a stack trace for the frame named "main"
```

### Web application type deduction (`WebApplicationType.deduceFromClasspath()`)

```
                 DispatcherHandler present (spring-webflux)
                 AND DispatcherServlet ABSENT
                 AND Jersey ServletContainer ABSENT ?
                        │yes                       │no
                        ▼                          ▼
                    REACTIVE          jakarta.servlet.Servlet  AND
                                      ConfigurableWebApplicationContext both present?
                                            │no              │yes
                                            ▼                ▼
                                          NONE            SERVLET
```

- **Consequence:** `spring-boot-starter-web` + `spring-boot-starter-webflux` together -> SERVLET (MVC wins).
  Override with `spring.main.web-application-type=reactive|servlet|none` or `SpringApplicationBuilder.web(...)`.
- The type decides the `ApplicationContext` class *and* which web auto-configs match
  (`@ConditionalOnWebApplication(type=...)`).

### What is in `spring.factories` in Boot 3 (still!)

`META-INF/spring.factories` **still exists** in Boot 3 for *SpringApplication-level extension points*:

| Key (interface) | Examples |
|---|---|
| `ApplicationContextInitializer` | `ConfigurationWarningsApplicationContextInitializer`, `ContextIdApplicationContextInitializer`, `ServerPortInfoApplicationContextInitializer`, `SharedMetadataReaderFactoryContextInitializer`, `RSocketPortInfoApplicationContextInitializer` |
| `ApplicationListener` | `ClearCachesApplicationListener`, `ParentContextCloserApplicationListener`, `EnvironmentPostProcessorApplicationListener`, `AnsiOutputApplicationListener`, `LoggingApplicationListener`, `BackgroundPreinitializer`, `DelegatingApplicationListener`, `FileEncodingApplicationListener` |
| `SpringApplicationRunListener` | `EventPublishingRunListener` |
| `EnvironmentPostProcessor` | `ConfigDataEnvironmentPostProcessor`, `SpringApplicationJsonEnvironmentPostProcessor`, `SystemEnvironmentPropertySourceEnvironmentPostProcessor`, `RandomValuePropertySourceEnvironmentPostProcessor`, `DebugAgentEnvironmentPostProcessor` |
| `FailureAnalyzer` | `PortInUseFailureAnalyzer`, `NoSuchBeanDefinitionFailureAnalyzer`, `BeanCurrentlyInCreationFailureAnalyzer`, ... |
| `BootstrapRegistryInitializer`, `ApplicationContextFactory`, `PropertySourceLoader`, `ConfigDataLoader/Locations resolver`, `LoggingSystemFactory`, `DatabaseInitializerDetector`, `DependsOnDatabaseInitializationDetector` | |

What **moved out** in Boot 3.0: the `org.springframework.boot.autoconfigure.EnableAutoConfiguration` key.
Auto-configuration classes are registered ONLY in
`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (introduced in 2.7, sole mechanism in 3.0).

Loading uses `SpringFactoriesLoader.forDefaultResourceLocation(classLoader)`, which reads **every**
`META-INF/spring.factories` on the classpath (cached per classloader), so a jar you add can hook into startup without code changes.

## 2.2 `run()` step by step

Abbreviated real structure of `SpringApplication.run(String... args)` (3.x):

```java
public ConfigurableApplicationContext run(String... args) {
    Startup startup = Startup.create();                       // (3.2+) timing; older: StopWatch
    if (this.registerShutdownHook) SpringApplication.shutdownHook.enableShutdownHookAddition();
    DefaultBootstrapContext bootstrapContext = createBootstrapContext();
    ConfigurableApplicationContext context = null;
    configureHeadlessProperty();
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting(bootstrapContext, this.mainApplicationClass);          // (A)
    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments); // (B)
        Banner printedBanner = printBanner(environment);
        context = createApplicationContext();                                   // by web type
        context.setApplicationStartup(this.applicationStartup);
        prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner); // (C)(D)
        refreshContext(context);                                                // (E) refresh + shutdown hook
        afterRefresh(context, applicationArguments);                            // empty hook
        startup.started();
        if (this.logStartupInfo) new StartupInfoLogger(...).logStarted(getApplicationLog(), startup);
        listeners.started(context, startup.timeTakenToStarted());               // (F) + Liveness CORRECT
        callRunners(context, applicationArguments);                             // (G)
    } catch (Throwable ex) { throw handleRunFailure(context, ex, listeners); }  // (X) ApplicationFailedEvent
    try {
        if (context.isRunning()) listeners.ready(context, startup.ready());     // (H) ApplicationReadyEvent + Readiness ACCEPTING_TRAFFIC
    } catch (Throwable ex) { throw handleRunFailure(context, ex, null); }
    return context;
}
```

### Event / phase timeline (order matters in interviews)

```
 main()
   │  new SpringApplication()  -> read spring.factories (initializers, listeners)
   ▼
 (A) ApplicationStartingEvent            context: none, Environment: none
   │     LoggingApplicationListener initialises LoggingSystem here
   ▼
 (B) prepareEnvironment
   │     - create StandardServletEnvironment / StandardReactiveWebEnvironment / StandardEnvironment
   │     - configureEnvironment: ConversionService, command-line args as PropertySource "commandLineArgs"
   │     - ConfigurationPropertySources.attach(env)  (adds the "configurationProperties" adapter source)
   │     - ApplicationEnvironmentPreparedEvent  ──► EnvironmentPostProcessorApplicationListener
   │            runs EnvironmentPostProcessors: ConfigDataEnvironmentPostProcessor loads application.yml,
   │            profile files, spring.config.import; activates profiles; SPRING_APPLICATION_JSON; random.*
   │     - DefaultPropertiesPropertySource.moveToEnd(env)
   │     - bindToSpringApplication(env)   (spring.main.* -> SpringApplication fields: lazy-init, banner-mode ...)
   ▼
   printBanner  (banner.txt / banner.gif / spring.banner.location; spring.main.banner-mode=console|log|off)
   ▼
 createApplicationContext:
     SERVLET  -> AnnotationConfigServletWebServerApplicationContext
     REACTIVE -> AnnotationConfigReactiveWebServerApplicationContext
     NONE     -> AnnotationConfigApplicationContext
   ▼
 (C) prepareContext:
     - context.setEnvironment(env); postProcessApplicationContext (bean name generator, conversion service)
     - applyInitializers  (each ApplicationContextInitializer.initialize(ctx))
     - ApplicationContextInitializedEvent      <- "ContextPrepared"   (bootstrap context closed after)
     - register singletons: springApplicationArguments, springBootBanner
     - set allowBeanDefinitionOverriding=false (default in Boot), lazy-init BFPP if spring.main.lazy-initialization
     - load(context, sources): register your @SpringBootApplication class as a bean definition (BeanDefinitionLoader)
 (D) ApplicationPreparedEvent                <- "ContextLoaded"  (listeners are added to the context here)
   ▼
 (E) refreshContext -> AbstractApplicationContext.refresh():
       invokeBeanFactoryPostProcessors -> ConfigurationClassPostProcessor: parse @ComponentScan, @Import,
            *DeferredImportSelectors last* (auto-config), register bean defs
       onRefresh() -> ServletWebServerApplicationContext.createWebServer()  <- Tomcat is CREATED here
       finishBeanFactoryInitialization -> singletons instantiated
       finishRefresh -> Lifecycle beans start:
            WebServerStartStopLifecycle.start()  <- connectors bound, port opens NOW
            publishes ContextRefreshedEvent, then ServletWebServerInitializedEvent
   ▼
 afterRefresh (empty)
   ▼
 (F) ApplicationStartedEvent  then  AvailabilityChangeEvent(LivenessState.CORRECT)
   ▼
 (G) callRunners: ApplicationRunner and CommandLineRunner beans, sorted by @Order/Ordered
   ▼
 (H) ApplicationReadyEvent    then  AvailabilityChangeEvent(ReadinessState.ACCEPTING_TRAFFIC)

 any exception -> (X) handleRunFailure: exit code handling, ApplicationFailedEvent,
                       FailureAnalyzers print "APPLICATION FAILED TO START", context.close()
```

Precise class names for the events you asked about:

| Informal name | Real class | Published by |
|---|---|---|
| Starting | `ApplicationStartingEvent` | `listeners.starting` |
| EnvironmentPrepared | `ApplicationEnvironmentPreparedEvent` | `listeners.environmentPrepared` |
| ContextPrepared | `ApplicationContextInitializedEvent` | `listeners.contextPrepared` |
| ContextLoaded | `ApplicationPreparedEvent` | `listeners.contextLoaded` |
| (framework) | `ContextRefreshedEvent`, `WebServerInitializedEvent` | context refresh |
| Started | `ApplicationStartedEvent` (+ `AvailabilityChangeEvent` liveness) | `listeners.started` |
| Ready | `ApplicationReadyEvent` (+ `AvailabilityChangeEvent` readiness) | `listeners.ready` |
| Failed | `ApplicationFailedEvent` | `listeners.failed` |

**How events are delivered:** `EventPublishingRunListener` (the default `SpringApplicationRunListener`) uses an
`SimpleApplicationEventMulticaster` holding the listeners loaded from `spring.factories` for the events
**before** the context exists (Starting, EnvironmentPrepared, ContextInitialized). At `contextLoaded` it hands
the listeners to the context, and from `ApplicationPreparedEvent` on, `context.publishEvent(...)` is used - which means
**`@EventListener` methods and `@Component` listeners only see events from `ApplicationPreparedEvent` onward.**
You **cannot** receive `ApplicationStartingEvent`/`EnvironmentPreparedEvent` through a bean; you must register via
`spring.factories` or `SpringApplication.addListeners(...)`.

Why `ApplicationStartedEvent` and `ApplicationReadyEvent` differ:
- **Started** = context refreshed, web server accepting connections, **runners not yet executed**.
- **Ready** = runners finished; the app is ready to serve. Readiness probe flips to `ACCEPTING_TRAFFIC` here.
  A slow `CommandLineRunner` therefore delays readiness even though the port is already open
  (with default config, requests can hit a "started but not ready" app unless readiness gating is used).

### Runners

```java
@Component @Order(1)
class Warmup implements ApplicationRunner {          // gets parsed ApplicationArguments (--k=v, non-option args)
    public void run(ApplicationArguments a) { ... }
}
@Component @Order(2)
class Seed implements CommandLineRunner {            // gets raw String... args
    public void run(String... args) { ... }
}
```
An exception in a runner -> `handleRunFailure` -> `ApplicationFailedEvent`, context closed, process exit code 1
(unless `ExitCodeGenerator`s say otherwise). Runners run on the **main thread**, sequentially.

### `SpringApplicationBuilder`, non-web mode, and exit

```java
new SpringApplicationBuilder(App.class).web(WebApplicationType.NONE).bannerMode(Banner.Mode.OFF).run(args);
System.exit(SpringApplication.exit(ctx, () -> 42));   // ExitCodeGenerator beans + ExitCodeEvent
```
A `NONE` app with no non-daemon thread exits when `main` returns - a batch/CLI pattern.

## 2.3 `@SpringBootApplication` anatomy

```java
@Target(TYPE) @Retention(RUNTIME) @Documented @Inherited
@SpringBootConfiguration                    // meta: @Configuration  (proxyBeanMethods attribute)
@EnableAutoConfiguration                    // meta: @AutoConfigurationPackage + @Import(AutoConfigurationImportSelector)
@ComponentScan(excludeFilters = {
    @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
    @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication { exclude, excludeName, scanBasePackages, scanBasePackageClasses, nameGenerator, proxyBeanMethods }
```

- `@ComponentScan` with no `basePackages` scans **the package of the annotated class and below** - the classic
  "bean not found because my class is in a sibling package" bug.
- **`AutoConfigurationExcludeFilter`**: excludes any scanned class that is both `@Configuration` **and** listed as an
  auto-configuration. Reason: an auto-config class must be processed via the deferred import (ordering + condition
  semantics), not accidentally picked up early by scanning your package (matters if a library's auto-config
  lives under your scan root).
- **`TypeExcludeFilter`**: an extension hook. It delegates to all `TypeExcludeFilter` **beans** in the bean factory.
  Test slices (`@WebMvcTest`, `@DataJpaTest`) register `StandardAnnotationCustomizableTypeExcludeFilter`
  subclasses so that only the relevant components (`@Controller`, `@Repository`...) are scanned. That is why
  a `@WebMvcTest` does not load your `@Service` beans.
- **`@AutoConfigurationPackage`** registers `AutoConfigurationPackages.Registrar`, storing the package of your main
  class. JPA (`@EntityScan` default), Spring Data repositories, and MyBatis mappers use it as their default scan root.
  Moving the main class = silently changing where entities are discovered.
- `proxyBeanMethods=false` -> "lite mode" (no CGLIB subclass of the config class). All Boot auto-configs use
  `@Configuration(proxyBeanMethods = false)` for startup speed and AOT friendliness. Inter-`@Bean` method calls then
  create new instances - inject via parameters instead.

## 2.4 Auto-configuration mechanism

### The chain

```
@EnableAutoConfiguration
   └─ @Import(AutoConfigurationImportSelector.class)          implements DeferredImportSelector
         │
         ▼   (ConfigurationClassPostProcessor -> ConfigurationClassParser)
   1. Parse user's classes fully: @ComponentScan results, @Import, @Bean methods, nested classes
   2. LAST: deferredImportSelectorHandler.process() -> group by AutoConfigurationGroup
   3. AutoConfigurationImportSelector.getAutoConfigurationEntry(annotationMetadata):
        a. isEnabled?  (spring.boot.enableautoconfiguration != false)
        b. configurations = ImportCandidates.load(AutoConfiguration.class, classLoader)
                           = every line of every META-INF/spring/...AutoConfiguration.imports
        c. removeDuplicates
        d. exclusions = @EnableAutoConfiguration(exclude, excludeName) + spring.autoconfigure.exclude
           (checkExcludedClasses: excluded class must be on classpath and actually an auto-config)
        e. configurations.removeAll(exclusions)
        f. getConfigurationClassFilter().filter(configurations)
              = AutoConfigurationImportFilter implementations (OnClassCondition, OnBeanCondition, OnWebApplicationCondition)
                read from META-INF/spring-autoconfigure-metadata.properties (generated at BUILD time by
                spring-boot-autoconfigure-processor) -> cheap pre-filter WITHOUT loading classes via ASM/Class.forName
        g. fireAutoConfigurationImportEvents (feeds the ConditionEvaluationReport)
   4. AutoConfigurationSorter orders survivors:
        alphabetical  ->  @AutoConfigureOrder  ->  @AutoConfiguration(before/after) / @AutoConfigureBefore/After
   5. Each surviving class is parsed as a @Configuration; its own @Conditional* are evaluated
      (PARSE_CONFIGURATION for class-level type conditions, REGISTER_BEAN for bean conditions)
```

### Why "deferred" matters

`@ConditionalOnMissingBean(DataSource.class)` asks "is a `DataSource` **bean definition** already registered?".
That question is only meaningful if **all user bean definitions are already registered**. A normal `ImportSelector`
would run in parse order (possibly before a user `@Bean` further down). `DeferredImportSelector` guarantees it
is processed after **all** `@Configuration` classes have been parsed, so:

```
user config beans registered  ->  auto-config classes evaluated  ->  conditions see the user's beans  ->  back off correctly
```

Corollary: `@ConditionalOnBean` / `@ConditionalOnMissingBean` are **only reliable in auto-configuration classes**.
Using them in regular `@Configuration` classes depends on registration order and produces flaky results.

### The imports file (Boot 2.7+/3)

`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

```
# comments allowed
com.acme.autoconfigure.AcmeAutoConfiguration
com.acme.autoconfigure.AcmeMetricsAutoConfiguration
```
One FQCN per line. The class must be annotated `@AutoConfiguration` (a `@Configuration(proxyBeanMethods=false)`
meta-annotated with `@AutoConfigureBefore/After` aliases: `@AutoConfiguration(before = X.class, after = Y.class, beforeName/afterName)`).

Old (<= 2.6, and up to 2.7 deprecated): `spring.factories`:
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.acme.AcmeAutoConfiguration
```
Boot 3.0 **ignores** that key -> a library not migrated silently stops auto-configuring after upgrade (classic migration bug).

### Ordering of auto-configs

- `before/after` control **configuration class processing order** (and therefore `@ConditionalOnMissingBean` outcome
  between two auto-configs), *not* bean creation order (bean creation order is dependency-driven; use `@DependsOn` for that).
- Example from Boot: `DataSourceAutoConfiguration` is `before = SqlInitializationAutoConfiguration`;
  `HibernateJpaAutoConfiguration` is `after = DataSourceAutoConfiguration` and `before = ...`. Circular before/after
  cycles throw `IllegalStateException: AutoConfigure cycle detected`.

## 2.5 Conditions and the evaluation report

### Two evaluation phases (`ConfigurationCondition.ConfigurationPhase`)

| Phase | When | Typical conditions |
|---|---|---|
| `PARSE_CONFIGURATION` | while parsing the config class (may skip the whole class incl. nested/`@Import`) | `@ConditionalOnClass`, `@ConditionalOnMissingClass`, `@ConditionalOnWebApplication`, `@ConditionalOnProperty`, `@ConditionalOnExpression`, `@Profile`, `@ConditionalOnResource` |
| `REGISTER_BEAN` | when bean definitions are being registered | `@ConditionalOnBean`, `@ConditionalOnMissingBean`, `@ConditionalOnSingleCandidate` |

Order within a class: Spring sorts `Condition`s by `@Order`/`Ordered`; Boot's `OnClassCondition` etc. carry
`@Order(Ordered.HIGHEST_PRECEDENCE)` so the cheap class-presence check runs **before** bean checks. Class-level
conditions gate the entire class, then each `@Bean` method's own conditions are evaluated at registration.

### Catalogue

| Condition | Meaning / gotcha |
|---|---|
| `@ConditionalOnClass(X.class)` / `(name="x.Y")` | Read via ASM annotation metadata, so the annotated **class** loads even when `X` is absent. On a `@Bean` **method** use `name=` or move into a nested static `@Configuration` - otherwise the method signature referencing the class can fail with `NoClassDefFoundError`. |
| `@ConditionalOnMissingClass("x.Y")` | Only string form (a class ref would need the class). |
| `@ConditionalOnBean` / `@ConditionalOnMissingBean` | attributes `value`, `type`, `name`, `annotation`, `search` (`CURRENT`/`ANCESTORS`/`ALL`), `ignored`. With no attribute on a `@Bean` method it defaults to the method's return type. Considers **bean definitions registered so far**, including type prediction from `FactoryBean`/factory methods. |
| `@ConditionalOnSingleCandidate` | exactly one, or one `@Primary` |
| `@ConditionalOnProperty(prefix, name, havingValue, matchIfMissing)` | Missing property -> false unless `matchIfMissing=true`. If `havingValue` unset: matches when property exists and is **not** `"false"`. Uses relaxed names on the *name* only in the sense of canonical lookup through the `Environment`. |
| `@ConditionalOnWebApplication(type=SERVLET/REACTIVE/ANY)` / `@ConditionalOnNotWebApplication` | Uses the `ApplicationContext` type + presence of `WebApplicationContext` |
| `@ConditionalOnExpression("${x:true} and ...")` | SpEL; evaluated early, cannot see beans. |
| `@ConditionalOnResource(resources="classpath:x.xml")` | resource exists |
| `@ConditionalOnJava`, `@ConditionalOnCloudPlatform(KUBERNETES)`, `@ConditionalOnThreading(VIRTUAL)` (3.2+), `@ConditionalOnEnabledHealthIndicator`, `@ConditionalOnAvailableEndpoint`, `@ConditionalOnMissingFilterBean`, `@ConditionalOnDefaultWebSecurity` | specialised |
| `@Profile` | **Spring core** (`ProfileCondition`), not a Boot annotation. There is **no** `@ConditionalOnProfile` in Boot. |

### Reading the report

Run `java -jar app.jar --debug` (or `debug=true`; not the same as `logging.level.root=DEBUG`). Or expose
`/actuator/conditions` (`management.endpoints.web.exposure.include=conditions`). Shape (illustrative):

```
============================
CONDITIONS EVALUATION REPORT
============================
Positive matches:
-----------------
   DataSourceAutoConfiguration matched:
      - @ConditionalOnClass found required classes 'javax.sql.DataSource', 'org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseType' (OnClassCondition)
   DataSourceAutoConfiguration.PooledDataSourceConfiguration matched:
      - AnyNestedCondition 1 matched 1 did not; ... (DataSourceAutoConfiguration.PooledDataSourceCondition)
      - @ConditionalOnMissingBean (types: javax.sql.DataSource,javax.sql.XADataSource; SearchStrategy: all) did not find any beans (OnBeanCondition)

Negative matches:
-----------------
   MongoAutoConfiguration:
      Did not match:
         - @ConditionalOnClass did not find required class 'com.mongodb.client.MongoClient' (OnClassCondition)

Exclusions:
-----------
    org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration

Unconditional classes:
----------------------
    org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration
```
Sections: Positive, Negative, Exclusions, Unconditional. Technique: "Why is my bean X missing?" -> find the
auto-config in **Negative matches**, read the failing condition. "Why is Boot's bean overriding mine?" -> find it
in Positive matches; its `@ConditionalOnMissingBean` type probably differs from your bean's declared type
(you returned `Object`/an interface subtype the condition doesn't match by type prediction).

## 2.6 `@ConfigurationProperties` and the `Binder`

### The pipeline

```
@ConfigurationProperties("app") class AppProps
        │ registered by: @EnableConfigurationProperties(AppProps.class) | @ConfigurationPropertiesScan | @Component | @Bean
        ▼
 ConfigurationPropertiesBindingPostProcessor (BeanPostProcessor, before-initialization)
        │  ConfigurationPropertiesBean.get(...) -> ConfigurationPropertiesBinder
        ▼
 Binder.get(environment)  ── uses ── ConfigurationPropertySources (adapter over ALL PropertySources)
        │  names normalised to canonical form: lowercase, kebab, dots: "app.max-retries"
        ▼
 JavaBeanBinder | ValueObjectBinder (constructor)  | CollectionBinder | MapBinder | ArrayBinder
        │  conversion: ConversionService (Duration, DataSize, Period, Charset, List from CSV, enums case-insens.)
        ▼
 validation if @Validated (jakarta.validation; needs a provider e.g. spring-boot-starter-validation)
        ▼
 BindException -> startup failure "Binding to target ... failed" (BindFailureAnalyzer)
```

### Relaxed binding

For a Java property `maxRetries` under prefix `app`, all of these bind:

| Source | Accepted form |
|---|---|
| `.properties` / `.yml` | `app.max-retries` (recommended, kebab), `app.maxRetries`, `app.max_retries` |
| Env variable | `APP_MAXRETRIES` (uppercase, dots -> `_`, dashes dropped). Underscored `APP_MAX_RETRIES` is NOT the same key for a kebab property, so prefer the documented form. |
| Lists in env | `APP_HOSTS_0=a`, `APP_HOSTS_1=b` (indexed with underscore) |
| Map keys | `app.limits.[/key with slash]=1` bracket notation to preserve special characters |
| System properties | same as `.properties` |

`@Value("${app.max-retries}")` is NOT fully relaxed: it works with the canonical kebab-case key, but has no `Binder` semantics and supports SpEL. Prefer `@ConfigurationProperties`.

| | `@Value` | `@ConfigurationProperties` |
|---|---|---|
| Relaxed binding | limited | full |
| Metadata (IDE completion) | no | yes (`spring-boot-configuration-processor`) |
| SpEL | yes | no |
| Validation | manual | `@Validated` |
| Type-safe groups/nesting/lists/maps | no | yes |
| Rebind on change | no | no (not without Spring Cloud `@RefreshScope`) |

### Immutable binding (constructor binding)

```java
@ConfigurationProperties(prefix = "app.http")
public record HttpProps(
        @DefaultValue("5s") Duration timeout,
        @DefaultValue("3")  int retries,
        @DefaultValue List<String> hosts) {}        // Boot 3: single-constructor record => constructor binding automatically
```
- Boot 3.0: **`@ConstructorBinding` is only needed on a constructor when a class has multiple constructors.**
  Class-level `@ConstructorBinding` was Boot 2.2-2.7 style and was removed as the mandatory form.
- Register with `@EnableConfigurationProperties(HttpProps.class)` or `@ConfigurationPropertiesScan("com.acme")`.
  A constructor-bound type **cannot** be a `@Component` (the container would try normal autowiring construction).
- `@Validated` on the record + constraint annotations on components: `@NotNull @Min(1)` (jakarta.validation).
  Nested objects need `@Valid` to cascade.

### `@EnableConfigurationProperties` vs `@ConfigurationPropertiesScan`

| | Purpose |
|---|---|
| `@EnableConfigurationProperties(X.class)` | explicit registration; also switches on the binding infrastructure (already turned on by `ConfigurationPropertiesAutoConfiguration`) |
| `@ConfigurationPropertiesScan` | scans packages for `@ConfigurationProperties` classes (registers them as beans; skips classes that are already `@Component`) |
| `@Component`/`@Bean` on the props class | works only for JavaBean (setter) style, not constructor binding |

### Metadata

Add `org.springframework.boot:spring-boot-configuration-processor` (annotation processor, optional). It produces
`META-INF/spring-configuration-metadata.json` from `@ConfigurationProperties` classes + javadoc. Hand-written
extras go in `META-INF/additional-spring-configuration-metadata.json` (e.g. deprecations with `replacement`).
Metadata is only IDE tooling; binding never reads it.

## 2.7 Property source precedence

Boot's `Environment` holds an ordered list of `PropertySource`s; **first source that has the key wins**.
Documented order, **highest to lowest** (Boot 3.x reference "Externalized Configuration"):

```
  1. Devtools global settings  (~/.config/spring-boot/spring-boot-devtools*.properties, only when devtools active)
  2. @TestPropertySource                    (tests)
  3. @DynamicPropertySource                 (tests)
  4. `properties` attribute on @SpringBootTest and test slices
  5. Command-line arguments                 (--server.port=9000)
  6. SPRING_APPLICATION_JSON                (inline JSON in env var or -Dspring.application.json)
  7. ServletConfig init parameters
  8. ServletContext init parameters
  9. JNDI attributes from java:comp/env
 10. Java System properties                 (-Dserver.port=9000)
 11. OS environment variables               (SERVER_PORT=9000)
 12. RandomValuePropertySource              (random.*)
 13. Config data files:
        a. Profile-specific outside the jar   (application-{profile}.properties/yml)
        b. Application properties outside the jar
        c. Profile-specific inside the jar
        d. Application properties inside the jar
 14. @PropertySource on @Configuration classes   (added too late to influence some config like logging.* and spring.main.*)
 15. Default properties (SpringApplication.setDefaultProperties / DefaultPropertiesPropertySource)
```

Details that interviewers probe:
- **Later profile files override earlier**: `application-{profile}` beats `application`. With multiple active profiles
  (`dev,local`), the **last** listed profile wins.
- In a single location, **`.properties` beats `.yml`** for the same key.
- Config-data search locations, later overriding earlier:
  `classpath:/` -> `classpath:/config/` -> `file:./` -> `file:./config/` -> `file:./config/*/` (subdirectories).
  `spring.config.location` **replaces** defaults, `spring.config.additional-location` **adds** (and wins).
  `spring.config.name` changes `application` to something else.
- **`--` args beat env vars beat file config.** So Kubernetes `env:` overrides your `application.yml` - by design.
- `@PropertySource` does not support YAML, and is processed at `@Configuration` parse time, so it cannot supply
  values used to pick profiles or logging config.
- `spring.config.import` - imported files are inserted with a precedence *relative to the file that imports them*
  (imported document has higher priority than the importer, but below external overrides).
- To inspect at runtime: `/actuator/env/{property}` shows which property source won (values sanitised;
  since Boot 3.0 `management.endpoint.env.show-values` default is `NEVER`; use `WHEN_AUTHORIZED`/`ALWAYS` deliberately).

## 2.8 Profiles, `spring.config.import`, Kubernetes config

### Activation

```
spring.profiles.active=prod,eu            # env: SPRING_PROFILES_ACTIVE, arg: --spring.profiles.active=prod
spring.profiles.default=dev               # used only when nothing is active (default profile name is "default")
spring.profiles.include=metrics           # adds profiles unconditionally (not allowed inside profile-specific files)
spring.profiles.group.prod=proddb,prodmq  # activating "prod" also activates proddb and prodmq
```
- `SpringApplication.setAdditionalProfiles(...)`, `@ActiveProfiles` in tests.
- **Setting `spring.profiles.active` inside `application-prod.yml` is invalid** -> `InvalidConfigDataPropertyException`
  (profile activation must be decided before profile-specific files load). Same for `spring.profiles.include`.
- `@Profile("prod")`, `@Profile("!prod")`, `@Profile("a & b")` (Spring 5.1+ expressions). `@Profile` gates beans at registration.

### Multi-document files

```yaml
app.name: demo                 # default doc
---
spring.config.activate.on-profile: prod
app.name: demo-prod            # applies only when prod is active
---
spring.config.activate.on-cloud-platform: kubernetes
management.endpoint.health.probes.enabled: true
```
(Properties files use `#---` as the separator.) `spring.profiles` (the old key) was removed in 3.0 -> `spring.config.activate.on-profile`.

### `spring.config.import`

```yaml
spring:
  config:
    import:
      - optional:file:./local-overrides.yml        # optional: = no failure if missing
      - optional:configtree:/etc/config/           # each FILE = one property, filename = key, content = value
      - optional:configserver:http://cfg:8888      # Spring Cloud Config (needs the spring-cloud dependency)
      - optional:file:/run/secrets/db.properties[.properties]   # extension hint syntax
```
Without `optional:` a missing target fails startup with `ConfigDataLocationNotFoundException`.

### Kubernetes patterns

| Mechanism | How Boot sees it | Notes |
|---|---|---|
| ConfigMap -> `envFrom`/`env` | OS env var -> relaxed binding: `SPRING_DATASOURCE_URL`, `SERVER_TOMCAT_THREADS_MAX`, `APP_HOSTS_0` | Beats file config. Can't express keys with awkward characters -> use `SPRING_APPLICATION_JSON`. |
| ConfigMap holding `application.yml` mounted at `/config/application.yml` (working dir `/`?) | `file:./config/` search location if the process working dir contains `./config`, else add `spring.config.additional-location=file:/config/` | Update on the volume is **not** hot reloaded by plain Boot. |
| ConfigMap/Secret mounted as directory, `configtree:` import | one file per key | Ideal for Secrets (no env var exposure; permissions per file). |
| Spring Cloud Kubernetes | Reads ConfigMaps/Secrets via API, optional reload | Separate project; not in Boot itself. |

Boot's own Kubernetes awareness: `CloudPlatform.KUBERNETES` is detected when env vars `*_SERVICE_HOST` and
`*_SERVICE_PORT` exist; then **liveness/readiness probe health groups are enabled automatically**
(`management.endpoint.health.probes.enabled` implicit) and `@ConditionalOnCloudPlatform` applies.

## 2.9 Embedded server (Tomcat and friends)

### Wiring

```
ServletWebServerFactoryAutoConfiguration   (@ConditionalOnClass(ServletRequest), @ConditionalOnWebApplication(SERVLET))
   @EnableConfigurationProperties(ServerProperties)  (prefix "server")
   @Import: BeanPostProcessorsRegistrar,
            ServletWebServerFactoryConfiguration.EmbeddedTomcat   @ConditionalOnClass(Servlet, Tomcat, UpgradeProtocol) @ConditionalOnMissingBean(ServletWebServerFactory)
            ServletWebServerFactoryConfiguration.EmbeddedJetty    @ConditionalOnClass(Servlet, Server, Loader, WebAppContext)
            ServletWebServerFactoryConfiguration.EmbeddedUndertow @ConditionalOnClass(Servlet, Undertow, SslClientAuthMode)
   beans: ServletWebServerFactoryCustomizer, TomcatServletWebServerFactoryCustomizer (@ConditionalOnClass Tomcat) ...
   WebServerFactoryCustomizerBeanPostProcessor  -> applies ALL WebServerFactoryCustomizer beans to the factory bean
```
Flow inside refresh:

```
ServletWebServerApplicationContext.onRefresh()
   -> createWebServer()
        factory = getWebServerFactory()      // exactly ONE ServletWebServerFactory bean required
        webServer = factory.getWebServer(getSelfInitializer())   // ServletContextInitializer collects DispatcherServletRegistrationBean,
                                                                 // FilterRegistrationBeans, Servlet/Filter/Listener beans
        TomcatServletWebServerFactory.getWebServer:
           new Tomcat(); Connector; customizers (TomcatConnectorCustomizer, TomcatContextCustomizer, TomcatProtocolHandlerCustomizer)
           prepareContext -> TomcatEmbeddedContext, StandardWrapper for default servlet, initializers
           new TomcatWebServer(tomcat, autoStart) -> initialize(): tomcat.start() BUT connectors removed (port NOT yet bound);
                                                     a daemon "container-0" await thread keeps the JVM alive
finishRefresh() -> WebServerStartStopLifecycle.start() -> TomcatWebServer.start(): re-add connectors -> port bound
```
Why connectors are delayed: so the port opens only **after** all singletons are initialised - avoids serving
requests to a half-initialised app (and a failing bean means the port never opened).
Change the server by dependency: exclude `spring-boot-starter-tomcat` from `spring-boot-starter-web`
and add `spring-boot-starter-jetty` (or `-undertow`); the matching `@ConditionalOnClass` flips.
(Undertow lags Servlet-spec upgrades; check the current Boot release notes before choosing it for new projects.)

### Customising

```java
@Bean WebServerFactoryCustomizer<TomcatServletWebServerFactory> tomcatTuning() {
    return f -> f.addConnectorCustomizers(c -> c.setProperty("relaxedQueryChars", "[]"));
}
```
Or properties (preferred), `server.*`:

| Property | Meaning | Default (Tomcat) |
|---|---|---|
| `server.port` | listen port; `0` = random (see `local.server.port`) | 8080 |
| `server.tomcat.threads.max` | max worker threads | 200 |
| `server.tomcat.threads.min-spare` | idle threads kept | 10 |
| `server.tomcat.accept-count` | OS-level backlog queue when all connections are in use | 100 |
| `server.tomcat.max-connections` | max simultaneous open connections the acceptor allows (NIO) | 8192 |
| `server.tomcat.connection-timeout` | time to wait for request line after accept | Tomcat connector default |
| `server.tomcat.keep-alive-timeout` | idle keep-alive time | falls back to connection-timeout |
| `server.tomcat.max-keep-alive-requests` | requests per keep-alive connection | 100 |
| `server.max-http-request-header-size` | header cap (Boot 3 name; old `server.max-http-header-size` removed) | 8KB |
| `server.tomcat.max-http-form-post-size`, `server.tomcat.max-swallow-size` | body limits | 2MB |
| `server.compression.enabled`, `server.http2.enabled`, `server.ssl.*` | features | off |
| `server.servlet.context-path`, `server.shutdown` | | / , immediate |
| `spring.threads.virtual.enabled=true` | (3.2, JDK 21) Tomcat/Jetty use virtual threads for request handling; also affects `@Async`, scheduling defaults | false |

### Connection model - the numbers that get asked

```
client ──TCP──► [OS accept queue: accept-count (backlog)] ──► Acceptor thread
                          (queue full -> connection refused / timeout for client)
                                                       │  if open connections < max-connections
                                                       ▼
                                             Poller (NIO selector, cheap, holds idle keep-alive sockets)
                                                       │  request bytes ready
                                                       ▼
                                   worker pool: up to threads.max concurrent requests
                                         (busy -> job queued in executor queue / waits)
```
- `max-connections` (8192) >> `threads.max` (200) is intentional for NIO: idle keep-alive connections don't hold threads.
- Concurrency of *running* requests = `threads.max`. With blocking calls to a slow DB/HTTP dependency you exhaust the
  200 threads long before CPU is saturated -> latency cliff. Fixes: timeouts, bulkheads, more threads (with pool sizing
  awareness: Hikari pool default 10 is usually the real limiter), virtual threads, or reactive.
- `accept-count` only matters after `max-connections` is hit.

## 2.10 `DispatcherServlet` request flow in Boot

Boot registers `DispatcherServlet` through `DispatcherServletAutoConfiguration` (`DispatcherServletRegistrationBean`,
mapping `/`; `spring.mvc.servlet.load-on-startup` defaults to -1, so the servlet initialises lazily on the first request unless set >= 0).
`WebMvcAutoConfiguration` (with `@EnableWebMvcConfiguration` internals) supplies the strategy beans.
**Defining your own `@EnableWebMvc` or a `WebMvcConfigurationSupport` bean turns off `WebMvcAutoConfiguration`** (it is
`@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)`) - Boot's Jackson/resource/static defaults vanish. `WebMvcConfigurer` is the safe extension.

```
HTTP request
  │
  ▼ Tomcat -> FilterChain: CharacterEncodingFilter, FormContentFilter, RequestContextFilter, (Security filter chain),
  │           ObservationFilter / WebMvcMetricsFilter (http.server.requests), your Filters (@Order / FilterRegistrationBean)
  ▼
DispatcherServlet.doService -> doDispatch:
  1. checkMultipart (MultipartResolver)
  2. getHandler(request): iterate HandlerMappings in order
        RequestMappingHandlerMapping   (@RequestMapping methods) -> HandlerMethod
        BeanNameUrlHandlerMapping, RouterFunctionMapping, WelcomePageHandlerMapping, SimpleUrlHandlerMapping (static resources /**)
        => HandlerExecutionChain = handler + HandlerInterceptors (CorsInterceptor, your interceptors)
  3. getHandlerAdapter(handler): RequestMappingHandlerAdapter | HandlerFunctionAdapter | HttpRequestHandlerAdapter | SimpleControllerHandlerAdapter
  4. interceptors.preHandle (any false -> stop, afterCompletion of prior ones)
  5. adapter.handle -> RequestMappingHandlerAdapter.invokeHandlerMethod:
        ServletInvocableHandlerMethod:
          HandlerMethodArgumentResolvers  (@RequestParam, @PathVariable, @RequestHeader, @ModelAttribute,
              @RequestBody -> RequestResponseBodyMethodProcessor -> HttpMessageConverter.read (+ @Valid validation),
              HttpServletRequest, Principal, Pageable (Spring Data) ...)
          invoke controller method (via reflection)
          HandlerMethodReturnValueHandlers (@ResponseBody -> HttpMessageConverter.write, ResponseEntity, String view name,
              ModelAndView, Callable/DeferredResult/CompletableFuture async, StreamingResponseBody)
  6. interceptors.postHandle (reverse order)
  7. processDispatchResult: if exception -> processHandlerException -> HandlerExceptionResolvers; render view if any
  8. interceptors.afterCompletion (reverse)
  └ unhandled exception -> sendError -> Tomcat ERROR dispatch to /error -> BasicErrorController -> JSON/HTML error body
```

### HttpMessageConverters and Jackson

`HttpMessageConvertersAutoConfiguration` assembles converters: `ByteArray`, `String`, `Resource`, `Form`,
`MappingJackson2HttpMessageConverter` (if Jackson), Jaxb/Gson/Jsonb if present. Selection by `Content-Type`/`Accept`
and the return type. Missing converter -> `HttpMediaTypeNotAcceptableException` (406) or `HttpMediaTypeNotSupportedException` (415).

`JacksonAutoConfiguration` builds the `ObjectMapper` via `Jackson2ObjectMapperBuilder` and applies:
- Boot defaults different from raw Jackson: `WRITE_DATES_AS_TIMESTAMPS=false`, `FAIL_ON_UNKNOWN_PROPERTIES=false`,
  `DEFAULT_VIEW_INCLUSION=false`.
- All `com.fasterxml.jackson.databind.Module` **beans** (and `jackson-datatype-jsr310`, jdk8, parameter-names via the starter-json)
  are auto-registered; `Jackson2ObjectMapperBuilderCustomizer` beans customise.
- Config: `spring.jackson.property-naming-strategy=SNAKE_CASE`, `spring.jackson.default-property-inclusion=non_null`,
  `spring.jackson.serialization.indent-output=true`, `spring.jackson.deserialization.fail-on-unknown-properties=true`,
  `spring.jackson.time-zone`, `spring.jackson.date-format`.
- **Declaring your own `ObjectMapper @Bean` replaces the Boot one** (`@ConditionalOnMissingBean`) and loses all those defaults,
  `spring.jackson.*` and module auto-registration; also `WebMvc` and `RestTemplateBuilder`/`WebClient.Builder` use the
  context mapper. Prefer `@Bean Jackson2ObjectMapperBuilderCustomizer` or `@Primary` only with intent.
- `@JsonTest` slice for focused serialization tests.

### Exception handling chain

```
exception in handler
  ▼
HandlerExceptionResolverComposite (order):
   DefaultErrorAttributes (HIGHEST precedence: just stores the exception as request attribute, never resolves)
   ExceptionHandlerExceptionResolver   -> @ExceptionHandler in the controller, then @ControllerAdvice/@RestControllerAdvice
   ResponseStatusExceptionResolver     -> @ResponseStatus, ResponseStatusException
   DefaultHandlerExceptionResolver     -> Spring MVC std exceptions -> 400/404/405/406/415/500 via response.sendError
  ▼ nothing resolved -> exception propagates -> Tomcat ERROR dispatch -> /error
BasicErrorController (server.error.path=/error) + DefaultErrorAttributes -> {timestamp,status,error,path,(message,trace)}
   defaults since 2.3: server.error.include-message=never, include-stacktrace=never, include-binding-errors=never
```
(An exception thrown **inside a Filter** never reaches `@ControllerAdvice` - only the `/error` path.)

### `ProblemDetail` (RFC 7807 / 9457) in Boot 3 / Spring 6

- `ProblemDetail` is a Spring 6 type; `ErrorResponse` interface; `ErrorResponseException`.
- Set `spring.mvc.problemdetails.enabled=true` -> Boot registers `ProblemDetailsExceptionHandler`
  (a `@ControllerAdvice` extending `ResponseEntityExceptionHandler`) so standard MVC exceptions render as
  `application/problem+json`: `{ "type":"about:blank","title":"Bad Request","status":400,"detail":"...","instance":"/x" }`.
- To customise: extend `ResponseEntityExceptionHandler` in your own `@RestControllerAdvice` and return
  `ProblemDetail` from handlers (`ProblemDetail.forStatusAndDetail(HttpStatus.CONFLICT, "...")`, `setProperty("code", "E42")`).
  Note that once you write your own such advice, don't also enable the property (two advices extending the same base = ambiguity).
- Since Spring 6.x, `ProblemDetail` also applies to Boot's `/error` handling only in later versions when enabled; do not
  assume - test your version.

## 2.11 Actuator

`spring-boot-starter-actuator`. Endpoints have **enablement** (exists) and **exposure** (reachable over web/JMX).
Boot 3 default web exposure: **only `health`**. (JMX default exposes more.)

```
management.endpoints.web.exposure.include=health,info,metrics,prometheus,loggers,conditions
management.endpoints.web.exposure.exclude=env
management.endpoints.web.base-path=/actuator
management.server.port=9091                       # separate management port (not reachable from public ingress)
management.endpoint.health.show-details=when-authorized    # never | when-authorized | always
management.endpoint.health.show-components=...
```
Key endpoints: `health`, `info`, `metrics`, `prometheus`, `env`, `configprops`, `beans`, `mappings`, `conditions`,
`loggers`, `threaddump`, `heapdump`, `httpexchanges` (needs an `HttpExchangeRepository` bean; replaced `httptrace`),
`scheduledtasks`, `startup` (needs `BufferingApplicationStartup`), `shutdown` (disabled by default), `caches`, `flyway`, `liquibase`, `sbom`.

### Health

- `HealthEndpoint` aggregates `HealthIndicator` beans via a `StatusAggregator` (order `DOWN, OUT_OF_SERVICE, UP, UNKNOWN`;
  the worst wins). `DOWN` -> HTTP 503.
- Auto-provided: `DiskSpaceHealthIndicator`, `DataSourceHealthIndicator`, `RedisHealthIndicator`, `PingHealthIndicator`,
  broker/mail/ES ones - each `@ConditionalOnEnabledHealthIndicator` (`management.health.<name>.enabled=false`).
- **Health groups**: `management.endpoint.health.group.critical.include=db,redis` -> `/actuator/health/critical`, with own
  `show-details`, `status.http-mapping`, and `additional-path`.
- **Probes** (`management.endpoint.health.probes.enabled=true`, automatic on Kubernetes):
  `/actuator/health/liveness` (group contains `livenessState`) and `/actuator/health/readiness` (`readinessState`).
  `management.endpoint.health.probes.add-additional-paths=true` exposes `/livez` and `/readyz` on the main server port.
- `LivenessState` (`CORRECT`, `BROKEN`) and `ReadinessState` (`ACCEPTING_TRAFFIC`, `REFUSING_TRAFFIC`) are driven by
  `AvailabilityChangeEvent`; read via `ApplicationAvailability`. On shutdown, the context close event moves readiness
  to `REFUSING_TRAFFIC` (503) before connections drain.
- Rule: **liveness = "should I restart this process?" (internal only, e.g. deadlock/BROKEN state)**. Never put the DB in
  liveness: a DB outage would restart every pod - a cascading failure. Readiness may include dependencies you cannot serve without
  (`management.endpoint.health.group.readiness.include=readinessState,db`), with judgement.

```java
@Component   // bean name "payGateway" -> health component "payGateway" (suffix "HealthIndicator" is stripped)
class PayGatewayHealthIndicator extends AbstractHealthIndicator {
    private final PayClient client;
    PayGatewayHealthIndicator(PayClient c) { super("Pay gateway check failed"); this.client = c; }
    @Override protected void doHealthCheck(Health.Builder b) {
        long ms = client.ping();                                   // keep it FAST with its own timeout
        if (ms < 0) b.down().withDetail("reason", "timeout"); else b.up().withDetail("latencyMs", ms);
    }
}
// Manual toggling of liveness/readiness:
AvailabilityChangeEvent.publish(applicationEventPublisher, this, ReadinessState.REFUSING_TRAFFIC);
```
A slow indicator blocks `/health` calls (the probe times out). Health calls run on the request thread; wrap external calls
with timeouts or cache (`management.endpoint.health.cache.time-to-live`).

### Custom endpoint

```java
@Component
@Endpoint(id = "featureflags")                       // /actuator/featureflags and JMX; must be exposed to be reachable
class FeatureFlagsEndpoint {
    private final Map<String, Boolean> flags = new ConcurrentHashMap<>();
    @ReadOperation  Map<String, Boolean> all() { return flags; }
    @ReadOperation  Boolean one(@Selector String name) { return flags.get(name); }          // GET /actuator/featureflags/{name}
    @WriteOperation void set(@Selector String name, boolean enabled) { flags.put(name, enabled); }  // POST with JSON body
    @DeleteOperation void remove(@Selector String name) { flags.remove(name); }
}
```
`@WebEndpoint` = web-only, `@JmxEndpoint` = JMX-only, `@ControllerEndpoint`/`@RestControllerEndpoint` = full Spring MVC semantics (web only).
Access control: `management.endpoint.featureflags.enabled` (3.x) plus exposure include list (Boot 3.4 introduced `access` levels; check version).

### Metrics (Micrometer)

- Boot auto-configures a `MeterRegistry` per registry dependency (`micrometer-registry-prometheus` -> `/actuator/prometheus`;
  `CompositeMeterRegistry` if several). Binders: JVM memory/GC/threads, `system.cpu.usage`, `process.uptime`,
  `http.server.requests` (timer with `method,uri,status,outcome,exception` tags), `hikaricp.connections.*`, `tomcat.*` (needs
  `server.tomcat.mbeanregistry.enabled=true`), `logback.events`, cache, Kafka, rabbit.
- Custom:
```java
@Component class Orders {
    private final Counter placed; private final Timer latency;
    Orders(MeterRegistry r) { placed = Counter.builder("orders.placed").tag("channel","web").register(r);
                              latency = Timer.builder("orders.latency").publishPercentileHistogram().register(r); }
    void place() { latency.record(() -> { /* work */ }); placed.increment(); }
}
```
  `@Timed`/`@Counted` need `TimedAspect`/`CountedAspect` beans (AOP); Boot 3 adds the **Observation API**
  (`ObservationRegistry`, `@Observed` requires `ObservedAspect`) which produces both metrics and traces
  (with `micrometer-tracing` bridge; replaces Spring Cloud Sleuth).
- Beware **tag cardinality**: never tag with userId/URL-with-ids; `uri` tag uses the *template* (`/users/{id}`) - custom
  code that uses raw paths explodes the time series and OOMs the registry/Prometheus.
- Common tags: `management.metrics.tags.application=${spring.application.name}`.

### Securing endpoints

```java
@Bean SecurityFilterChain actuator(HttpSecurity http) throws Exception {
    http.securityMatcher(EndpointRequest.toAnyEndpoint())
        .authorizeHttpRequests(a -> a.requestMatchers(EndpointRequest.to("health","info")).permitAll()
                                     .anyRequest().hasRole("OPS"))
        .httpBasic(Customizer.withDefaults());
    return http.build();
}
```
Layers of defence: expose minimal endpoints, separate `management.server.port` bound to internal network / not routed by ingress,
Security rules, `show-details=when-authorized`, `management.endpoint.env.show-values`/`configprops.show-values` default never,
keep `heapdump` off (contains secrets in memory).

## 2.12 Logging

- Boot logs through **SLF4J -> Logback** by default (`spring-boot-starter-logging`, includes bridges `jul-to-slf4j`, `log4j-to-slf4j`).
  Abstraction: `LoggingSystem` (Logback/Log4j2/JUL implementations) initialised by `LoggingApplicationListener`
  on `ApplicationStartingEvent` (early console logging) and again after `EnvironmentPrepared` (with real config).
- Config files: `logback-spring.xml` (**use this, not `logback.xml`**: `logback.xml` is loaded before Spring's Environment is ready, so
  `<springProfile>`/`<springProperty>` don't work there). `logging.config=classpath:custom.xml` overrides.
- Properties: `logging.level.root=WARN`, `logging.level.com.acme.orders=DEBUG`, `logging.file.name=app.log` /
  `logging.file.path=/var/log`, `logging.logback.rollingpolicy.max-file-size=10MB`, `logging.logback.rollingpolicy.max-history=7`,
  `logging.pattern.console`, `logging.pattern.level`, `logging.charset.console`.
- **Groups**: `logging.group.db=org.hibernate.SQL,org.springframework.jdbc` then `logging.level.db=DEBUG`. Predefined groups: `web`
  (`org.springframework.core.codec, http, web, boot.actuate.endpoint.web, boot.web.servlet.ServletContextInitializerBeans`) and `sql`.
- Structured JSON logging is built in from Boot 3.4 (`logging.structured.format.console=ecs|logstash|gelf`); before that use
  `logstash-logback-encoder`.
- Switch to Log4j2: exclude `spring-boot-starter-logging` from **every** starter and add `spring-boot-starter-log4j2`.
  Two backends on the classpath -> "multiple SLF4J bindings" / `LoggerFactory` class-cast failure.
- **Change level at runtime:**
```
GET  /actuator/loggers/com.acme.orders
POST /actuator/loggers/com.acme.orders    Content-Type: application/json   {"configuredLevel":"DEBUG"}
POST ...                                   {"configuredLevel":null}        -> reset to inherited
```
  Needs `loggers` exposed; change is per-JVM and lost on restart; secure it (turning on TRACE for `org.springframework.security` in prod leaks data).
- `--debug` (Boot's own debug logging, includes conditions report) vs `--trace`.
- MDC: put `traceId`/`userId` in a filter; Boot 3 with Micrometer Tracing populates `traceId`/`spanId` into MDC automatically
  (`logging.pattern.level` includes them if tracing is present).

## 2.13 DevTools

`spring-boot-devtools` (scope `runtime`/`developmentOnly`, `optional`):
- **Automatic restart** using two classloaders: a base classloader (third-party jars, not reloaded) and a **restart classloader**
  (your project classes). On a classpath change, only the restart classloader is discarded and `main` re-run - much faster than a cold start.
  Trigger: IDE rebuild (classpath change detected by file watcher); `spring.devtools.restart.trigger-file`; exclusions
  `spring.devtools.restart.exclude`, `additional-paths`.
- LiveReload server (35729), property defaults for development (`spring.thymeleaf.cache=false`, `server.error.include-*`, `web` logging group DEBUG, H2 console),
  remote DevTools (do not use in prod).
- **Auto-disabled** when the app runs from a fully packaged jar (`java -jar`) or a production-like launcher (`isDevelopment` detection),
  and excluded from `repackage` by default. Still: keep it out of production classpaths.
- Gotchas: class-cast errors between "same" classes loaded by two loaders (e.g. a cache holding objects across restarts, `ServiceLoader`
  libs, some JDBC driver registration); heavy static state; use `META-INF/spring-devtools.properties` (`restart.include.*`) to move a jar into the restart loader.
- Global settings: `~/.config/spring-boot/spring-boot-devtools.properties` (highest precedence in 2.7).

## 2.14 Graceful shutdown

```
server.shutdown=graceful                                   # default: immediate
spring.lifecycle.timeout-per-shutdown-phase=30s            # default 30s: max wait per SmartLifecycle phase
```
Trace on SIGTERM:

```
SIGTERM ─► JVM shutdown hook (SpringApplication registers one; spring.main.register-shutdown-hook)
       ─► context.close() ─► publish ContextClosedEvent  (readiness -> REFUSING_TRAFFIC)
       ─► DefaultLifecycleProcessor.stopBeans(): SmartLifecycle beans grouped by getPhase(), stopped from HIGHEST phase to LOWEST
             WebServerGracefulShutdownLifecycle.stop(callback):
                  webServer.shutDownGracefully(result): Tomcat -> connectors PAUSED (stop accepting new connections),
                  waits for in-flight requests to finish, up to timeout-per-shutdown-phase, then logs
                  "Graceful shutdown aborted with one or more requests still active" and continues
             WebServerStartStopLifecycle.stop(): actually stop the server
       ─► destroy singletons (@PreDestroy / DisposableBean / DataSource close, Kafka listener containers, executors...)
```
- **Ordering principle:** things that *produce* work (web server, message listeners) must stop **before** things they depend on
  (DB pool). SmartLifecycle phases enforce this: higher phase = starts later / stops earlier.
- Custom:
```java
@Component
class ConsumerLifecycle implements SmartLifecycle {
    private volatile boolean running;
    public void start() { running = true; /* subscribe */ }
    public void stop(Runnable callback) { /* stop polling, drain */ running = false; callback.run(); }  // MUST call callback, else waits the timeout
    public void stop() { running = false; }
    public boolean isRunning() { return running; }
    public int getPhase() { return Integer.MAX_VALUE - 100; }   // relative to the web server lifecycle phases
    // isAutoStartup() defaults to true
}
```
- Kubernetes: after the pod is marked Terminating, kube-proxy/ingress endpoints update **asynchronously**, so new requests can still arrive for a few seconds.
  Use a `preStop` `sleep 5-10`, set `terminationGracePeriodSeconds` > preStop + `timeout-per-shutdown-phase`, keep readiness failing during drain.
- Does not help `kill -9`, OOM kill, `Runtime.halt`, or `System.exit` from a shutdown hook. `@PreDestroy` doesn't run on `kill -9`.
- In-flight `@Async`/executor tasks: configure `spring.task.execution.shutdown.await-termination=true` and `await-termination-period`.

## 2.15 Fat jar, layers, Docker

`spring-boot-maven-plugin:repackage` (or `bootJar`) rewrites the jar:

```
app.jar
 ├─ META-INF/MANIFEST.MF
 │     Main-Class:  org.springframework.boot.loader.launch.JarLauncher      (3.2+; <=3.1: org.springframework.boot.loader.JarLauncher)
 │     Start-Class: com.acme.App
 │     Spring-Boot-Version, Spring-Boot-Classes: BOOT-INF/classes/, Spring-Boot-Lib: BOOT-INF/lib/,
 │     Spring-Boot-Classpath-Index: BOOT-INF/classpath.idx, Spring-Boot-Layers-Index: BOOT-INF/layers.idx
 ├─ org/springframework/boot/loader/...   (launcher classes, the ONLY classes at the jar root)
 ├─ BOOT-INF/classes/                     (your compiled classes + application.yml)
 ├─ BOOT-INF/lib/*.jar                    (dependencies, stored UNCOMPRESSED so they can be read in place)
 ├─ BOOT-INF/classpath.idx                (ordered list of jars for classpath order)
 └─ BOOT-INF/layers.idx                   (layered jar definition)
```
`JarLauncher` creates a classloader that understands **nested jars** (special `jar:` URL handling since standard `URLClassLoader`
cannot read jars inside jars), sets the thread context classloader, then reflectively calls `Start-Class.main`.
`WarLauncher` (WAR: `WEB-INF/classes`, `WEB-INF/lib`, `WEB-INF/lib-provided`), `PropertiesLauncher` (`loader.path`, `loader.main` - add jars externally).
Consequences: no `File`-based access to classpath resources (`getResource(...).getFile()` fails - use streams);
`java -cp` style scanning behaves differently; IDE runs are **not** identical to the jar run.

### Layers (Docker cache efficiency)

`layers.idx` default order (least -> most volatile): `dependencies`, `spring-boot-loader`, `snapshot-dependencies`, `application`.
```dockerfile
FROM eclipse-temurin:21-jre AS build
WORKDIR /w
COPY target/app.jar app.jar
RUN java -Djarmode=tools -jar app.jar extract --layers --destination extracted    # 3.3+; earlier: -Djarmode=layertools extract

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /w/extracted/dependencies/ ./
COPY --from=build /w/extracted/spring-boot-loader/ ./
COPY --from=build /w/extracted/snapshot-dependencies/ ./
COPY --from=build /w/extracted/application/ ./
ENTRYPOINT ["java","-jar","app.jar"]     # extracted layout: app.jar + lib/ (tools extract) - launcher not used; check your Boot version's layout
```
Your code changes (few KB) rebuild only the `application` layer; the 50 MB dependency layer is reused from cache -> faster CI/pull.
Alternatives: `mvn spring-boot:build-image` / `gradle bootBuildImage` (Cloud Native Buildpacks/Paketo, no Dockerfile, memory calculator,
sets `JAVA_TOOL_OPTIONS`), Jib (no Docker daemon). Container tuning: `-XX:MaxRAMPercentage=75`, set CPU limits knowingly
(JVM ergonomics use them for GC threads / `ForkJoinPool`), `-XX:+UseZGC`/G1 choice.

## 2.16 AOT and GraalVM native image (basics and constraints)

- **AOT processing** (Spring Framework 6 / Boot 3): at *build* time `spring-boot-maven-plugin:process-aot` (or `processAot`) runs the app
  up to bean definition registration (no instantiation) and generates Java source: `*__BeanDefinitions` classes (bean registration as plain code,
  no reflection/classpath scanning at runtime), `*__BeanFactoryRegistrations`, and GraalVM **hints** JSONs (reflection, resources, proxies, serialization).
- **Native image** (`org.graalvm.buildtools:native-maven-plugin`, `mvn -Pnative native:compile`, or `mvn -Pnative spring-boot:build-image`):
  GraalVM `native-image` does a closed-world static analysis from `main`, producing an OS executable.
  Results: startup ~tens of ms, small RSS, no JIT warm-up; costs: build takes minutes and lots of RAM, peak throughput usually lower than
  JIT-warmed HotSpot (unless PGO with Oracle GraalVM), harder debugging, some libraries unsupported.
- Run AOT on the JVM too: `-Dspring.aot.enabled=true` (faster JVM startup even without native).
- **Constraints (the "closed world")**
  - **Conditions and profiles are fixed at build time.** Bean definitions are generated at build; `@ConditionalOnProperty`,
    `@Profile`, `@ConditionalOnClass` are decided during AOT. Changing `spring.profiles.active` or a condition-driving property at runtime
    doesn't add/remove beans. (Ordinary `@Value`/`@ConfigurationProperties` *values* still resolve at runtime.)
  - **No runtime bytecode generation**: CGLIB proxies and `@Configuration` subclasses are generated at build; runtime `Proxy`/CGLIB
    creation for un-hinted types fails. Runtime class loading/`Class.forName` on unknown classes, agents, JMX subtleties, dynamic classpath: unsupported/limited.
  - **Reflection/resources/serialization/JNI need hints** - Spring generates them for its own model; for your own reflective access provide
    `RuntimeHintsRegistrar` via `@ImportRuntimeHints`, or `@RegisterReflectionForBinding(Dto.class)` (Jackson DTOs), `@Reflective`.
    Missing hint -> works on JVM, fails only in native (`NoSuchMethodException`, empty JSON, missing resource).
  - Feature-poor libraries: check GraalVM reachability metadata repository; Hibernate needs its bytecode enhancement at build time.
  - Build on the target OS/arch; cross-compiling is not straightforward.
- Testing: `nativeTest`/`mvn -PnativeTest test`. Alternatives to reduce startup without native: CDS/AppCDS (2.17), CRaC (3.2+; needs a CRaC JDK),
  virtual threads for concurrency (not startup), Project Leyden (JDK, emerging).

## 2.17 Startup time optimisation

Measure first: `logging.level.org.springframework.boot=...`? Prefer:
- `BufferingApplicationStartup` -> `/actuator/startup` (timeline of every startup step: `spring.beans.instantiate`, `spring.context.refresh`)
  ```java
  new SpringApplicationBuilder(App.class).applicationStartup(new BufferingApplicationStartup(4096)).run(args);
  ```
- `-Xlog:class+load`, JFR, simple `Started X in 8.3 seconds (process running for 9.1)` log line.

Levers (biggest first, usually):
1. **Lazy initialisation**: `spring.main.lazy-initialization=true` (`LazyInitializationBeanFactoryPostProcessor`, individual `@Lazy(false)` opt-outs).
   Wins: fewer beans built at startup. Costs: first request latency, **misconfigurations surface at first use, not at boot** (fails readiness
   principle "fail fast"), `@Scheduled`/listeners still eager if depended-on, Hibernate/Hikari warm-up moves to first call.
2. **Trim the classpath and auto-config**: drop unused starters, `spring.autoconfigure.exclude=...`, `@SpringBootApplication(exclude=...)`.
   Use `--debug` -> count positive matches you don't need.
3. Narrow `@ComponentScan`/entity scan; avoid scanning giant packages including fat third-party trees.
4. JPA: `spring.data.jpa.repositories.bootstrap-mode=deferred|lazy` (repository proxies init in background), don't `ddl-auto=update` in prod,
   Flyway with many migrations, Hibernate `hibernate.temp.use_jdbc_metadata_defaults=false` with explicit dialect (only if you understand the trade-off).
5. **Class Data Sharing / AppCDS** (JDK 13+ dynamic archive, JDK 19+ `-XX:+AutoCreateSharedArchive`):
   ```
   java -XX:ArchiveClassesAtExit=app.jsa -Dspring.context.exit=onRefresh -jar app.jar     # training run (spring.context.exit is 3.3+)
   java -XX:SharedArchiveFile=app.jsa -jar app.jar
   ```
   Typically 20-40% faster startup, no code change; the archive must match the exact JDK + jar. In Docker, generate at image build.
6. JVM flags: `-XX:TieredStopAtLevel=1` (dev/short-lived only; hurts peak), `-Xss`, heap sizing so no early GC, `-XX:+UseSerialGC` for tiny pods.
7. `spring.main.banner-mode=off`, `spring.jmx.enabled=false` (default false since 2.2), fewer actuator health indicators.
8. Bigger lever: AOT (`-Dspring.aot.enabled`), native image, CRaC.
9. Kubernetes: `startupProbe` (`/actuator/health/liveness` with generous `failureThreshold`) so slow starts don't get killed by liveness;
   avoid CPU limit throttling during boot (JIT threads starve).

## 2.18 Boot 2 -> 3 migration gotchas

Do it stepwise: 2.7.latest first (fix deprecations) -> 3.0. Use `spring-boot-properties-migrator` (runtime, reports renamed/removed properties)
and OpenRewrite `org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_0`.

| Area | Change and symptom |
|---|---|
| **Java 17 baseline** | Boot 3 requires 17+ (compiled class 61). Runs on 21. Symptom: `UnsupportedClassVersionError`. |
| **`javax.*` -> `jakarta.*`** | Servlet, persistence (JPA), validation, mail, annotation (`jakarta.annotation.PostConstruct`, `@Resource`), transaction, websocket. Every dependency that uses these must ship a Jakarta build (Hibernate 6, Tomcat 10.1, Jetty 12, Jackson ok, Springfox dead -> springdoc-openapi 2.x, Swagger annotations `io.swagger.v3` jakarta variant). Symptom: `ClassNotFoundException: javax.servlet.Filter`, `NoSuchMethodError`, beans of a filter type not registered. |
| **Auto-config registration** | `spring.factories` `EnableAutoConfiguration` -> `AutoConfiguration.imports` (+ `@AutoConfiguration`). Custom starters/internal libs silently stop working. |
| **Property renames/removals** | `spring.redis.*` -> `spring.data.redis.*`; `server.max-http-header-size` -> `server.max-http-request-header-size`; `management.metrics.export.<x>.*` -> `management.<x>.metrics.export.*`; `spring.profiles` -> `spring.config.activate.on-profile`; `spring.security.saml2...` reorganised; Actuator `httptrace` -> `httpexchanges`; `spring.sleuth.*` gone (Micrometer Tracing: `management.tracing.*`, `management.zipkin.tracing.*`). |
| **Trailing-slash matching off** (Spring 6) | `GET /users/` no longer matches `/users`. Symptom: 404 after upgrade. Fix client URLs or `WebMvcConfigurer` redirect. |
| **Path matching** | `PathPatternParser` only (`AntPathMatcher` legacy usage removed for MVC in 6.0/6.x); Springfox/others requiring `ant_path_matcher` break. |
| **Parameter names** | Spring 6.1 removed `LocalVariableTableParameterNameDiscoverer` -> requires `-parameters` javac flag or you get `IllegalArgumentException: Name for argument of type [...] not specified` (`MissingParameterNameFailureAnalyzer` in 3.2). Boot's Maven/Gradle plugins set it; hand-rolled builds don't. |
| **Hibernate 6** | Stricter/different HQL (`select` required semantics, `count` return `Long`), **`hibernate_sequence` gone: `GenerationType.AUTO` now uses per-entity sequences (`<entity>_seq`)** -> ID clashes/`relation does not exist` on existing schemas (set `@SequenceGenerator`/`@GeneratedValue(strategy=IDENTITY)` or `hibernate.id.db_structure_naming_strategy=legacy`); dialect autodetected (remove `spring.jpa.database-platform` legacy dialect names); different `UUID`/`Instant`/`Duration`/JSON mappings; `@Type` changed; `javax.persistence` imports. |
| **Spring Security 6** | `WebSecurityConfigurerAdapter` removed -> `SecurityFilterChain` beans; `authorizeRequests` -> `authorizeHttpRequests`; `antMatchers/mvcMatchers` -> `requestMatchers`; `@EnableGlobalMethodSecurity` -> `@EnableMethodSecurity` (pre/post default on); lambda DSL (`.and()` deprecated); `SecurityContextHolderFilter` -> context no longer saved implicitly (explicit `SecurityContextRepository.saveContext`); authorization now applies to **all dispatch types** (error/async) - `/error` may need `permitAll`; `hasRole` prefix behaviour stable; `AuthorizationManager` replaces `AccessDecisionManager`. |
| **`@ConstructorBinding`** | Class-level no longer needed; on constructor only if multiple constructors. |
| **Logging/Observability** | `logging.pattern.dateformat` default now ISO-8601-ish `yyyy-MM-dd'T'HH:mm:ss.SSSXXX`; observation API replaces `WebMvcMetricsFilter` config props. |
| **HttpMethod** | `HttpMethod` is a class (not enum) in Spring 6: `switch` and `EnumSet` usages break. |
| **Elasticsearch/other clients** | `RestHighLevelClient` removed (new Java client); Reactor/Netty/Kafka/other major bumps. |
| **Tomcat 10.1 / Servlet 6** | Removed methods; `HttpServletRequest#getRealPath` semantics, cookie handling; `HttpServletResponse` API removals. |
| **CGLIB/Java 17 module encapsulation** | Illegal reflective access to JDK internals now hard errors (`--add-opens` for libs like older Lombok/Mockito/Kryo/Hazelcast). |

---

# 3. Traced worked examples

## 3.1 Why does (or doesn't) a `DataSource` get auto-configured? Four situations

Relevant Boot classes (shape simplified but structurally accurate):

```java
@AutoConfiguration(before = SqlInitializationAutoConfiguration.class)
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ DataSourcePoolMetadataProvidersConfiguration.class, DataSourceCheckpointRestoreConfiguration.class })
public class DataSourceAutoConfiguration {

    @Configuration(proxyBeanMethods = false)
    @Conditional(EmbeddedDatabaseCondition.class)          // true only when NO pooled DS lib is available/selected
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
    @Import(EmbeddedDataSourceConfiguration.class)
    protected static class EmbeddedDatabaseConfiguration {}

    @Configuration(proxyBeanMethods = false)
    @Conditional(PooledDataSourceCondition.class)          // spring.datasource.type set OR Hikari/Tomcat-JDBC/DBCP2/UCP on classpath
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
    @Import({ DataSourceConfiguration.Hikari.class, DataSourceConfiguration.Tomcat.class,
              DataSourceConfiguration.Dbcp2.class, DataSourceConfiguration.OracleUcp.class, DataSourceConfiguration.Generic.class })
    protected static class PooledDataSourceConfiguration {}
}
// DataSourceConfiguration.Hikari:
//   @ConditionalOnClass(HikariDataSource.class) @ConditionalOnMissingBean(DataSource.class)
//   @ConditionalOnProperty(name = "spring.datasource.type", havingValue = "com.zaxxer.hikari.HikariDataSource", matchIfMissing = true)
//   HikariDataSource dataSource(DataSourceProperties p)  -> p.initializeDataSourceBuilder().type(HikariDataSource.class).build()
```
(`spring-boot-starter-jdbc` and `spring-boot-starter-data-jpa` bring Hikari transitively.)

### Situation 1 - only `spring-boot-starter-web` (no JDBC)

```
1. AutoConfigurationImportSelector lists DataSourceAutoConfiguration (it is in the imports file).
2. Pre-filter (OnClassCondition via spring-autoconfigure-metadata.properties):
   requires javax.sql.DataSource (JDK: present) AND
            org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseType (spring-jdbc: ABSENT)
   -> auto-config removed before it is even parsed.
Report: Negative match: DataSourceAutoConfiguration - @ConditionalOnClass did not find required class '...EmbeddedDatabaseType'.
Result: no DataSource, no JdbcTemplate, no transaction manager. Anything injecting DataSource -> NoSuchBeanDefinitionException (with analyzer text).
```

### Situation 2 - `starter-data-jpa` + PostgreSQL driver + `spring.datasource.url/username/password`

```
1. OnClassCondition: DataSource + EmbeddedDatabaseType (spring-jdbc via starter) present -> class parsed.
2. @ConditionalOnMissingBean(type=io.r2dbc...ConnectionFactory): absent -> ok.
3. Nested PooledDataSourceConfiguration:
      PooledDataSourceCondition: spring.datasource.type unset, but Hikari on classpath -> match.
      @ConditionalOnMissingBean(DataSource, XADataSource): the deferred processing means user config is already registered;
          none -> match.
   Nested EmbeddedDatabaseConfiguration: EmbeddedDatabaseCondition -> "pooled available" -> NO match (Hikari wins over embedded).
4. DataSourceConfiguration.Hikari: class present, no DataSource yet, property `type` missing + matchIfMissing=true -> match.
5. Bean created lazily-ordered by dependencies: HikariDataSource built by DataSourceProperties:
      url from spring.datasource.url; driverClassName deduced from URL prefix `jdbc:postgresql:` (DatabaseDriver.fromJdbcUrl);
      pool options from spring.datasource.hikari.* (bound onto the HikariDataSource itself).
6. HibernateJpaAutoConfiguration (@AutoConfiguration(after = DataSourceAutoConfiguration)) sees `DataSource` bean via
   @ConditionalOnSingleCandidate(DataSource.class) -> creates EntityManagerFactory, JpaTransactionManager.
Result: one HikariDataSource, pool size 10 default, connection established at first getConnection (Hikari starts eagerly via fail-fast check
        `initializationFailTimeout` default 1 -> bean creation fails if DB unreachable).
```

### Situation 3 - `starter-data-jpa` + H2 on classpath, no `spring.datasource.url`

```
Pooled condition matches (Hikari). Hikari bean is built from DataSourceProperties, and DataSourceProperties.determineUrl():
      url empty -> EmbeddedDatabaseConnection.get(classLoader) finds H2 -> url "jdbc:h2:mem:<generated name>;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE",
      username "sa", password "".
Result: an in-memory H2 behind a HikariDataSource. `spring.jpa.hibernate.ddl-auto` defaults to create-drop for an embedded DB (none otherwise).
Danger: forgetting the URL in prod with H2 (test scope mistake -> compile scope) silently starts on H2 and "works" with empty data.
Report: EmbeddedDatabaseConfiguration Negative (pooled available); PooledDataSourceConfiguration Positive.
```

### Situation 4a - JPA starter, **no** driver on classpath, no URL

```
Same conditions match (Hikari present). Bean creation: determineUrl() -> no embedded DB found, url empty -> throws
DataSourceProperties.DataSourceBeanCreationException -> DataSourceBeanCreationFailureAnalyzer prints:

  Description: Failed to configure a DataSource: 'url' attribute is not specified and no embedded datasource could be configured.
               Reason: Failed to determine a suitable driver class
  Action:      Consider the following:
               If you want an embedded database (H2, HSQL or Derby), please put it on the classpath.
               If you have database settings to be loaded from a particular profile you may need to activate it (no profiles are currently active).
```
The "profile" hint is the real cause in many prod incidents: `application-prod.yml` wasn't loaded because `SPRING_PROFILES_ACTIVE` was missing.

### Situation 4b - you define your own `DataSource`

```java
@Configuration class Db { @Bean DataSource ds() { return new HikariDataSource(cfg()); } }
```
User config is parsed first (auto-config is deferred). `PooledDataSourceConfiguration` + `EmbeddedDatabaseConfiguration`:
`@ConditionalOnMissingBean(DataSource.class)` -> false. Negative match reason: "@ConditionalOnMissingBean (types: DataSource,XADataSource; SearchStrategy: all) found beans 'ds'".
`DataSourceProperties` is still bound (unused, harmless). `spring.datasource.hikari.*` **does not apply** to your bean (a common surprise) - bind it yourself:
`@Bean @ConfigurationProperties("app.datasource.hikari") HikariDataSource ds(...)`.
Two `DataSource` beans -> JPA `@ConditionalOnSingleCandidate` fails unless one is `@Primary` -> `EntityManagerFactory` never configured.

## 3.2 A complete custom starter: `acme-spring-boot-starter`

Goal: a `GreetingService` library with properties `acme.greeting.*`, conditional bean, health indicator, backs off when user defines their own.

Layout (two modules, naming rule: third-party starters are `acme-spring-boot-starter`, never `spring-boot-starter-acme`):

```
acme-parent/
 ├─ acme-spring-boot-autoconfigure/   (code)
 │    src/main/java/com/acme/greeting/...
 │    src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
 │    src/main/resources/META-INF/additional-spring-configuration-metadata.json   (optional)
 └─ acme-spring-boot-starter/         (pom only: depends on autoconfigure + the real library deps)
```

`acme-spring-boot-autoconfigure/pom.xml` essentials:

```xml
<dependencies>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-autoconfigure</artifactId></dependency>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-configuration-processor</artifactId><optional>true</optional></dependency>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-actuator</artifactId><optional>true</optional></dependency>
  <dependency><groupId>com.acme</groupId><artifactId>acme-greeting-core</artifactId></dependency>   <!-- the library itself -->
</dependencies>
<!-- Optionally spring-boot-autoconfigure-processor (optional) generates META-INF/spring-autoconfigure-metadata.properties
     -> fast OnClass/OnBean pre-filtering -->
```

`acme-spring-boot-starter/pom.xml`: only dependencies:

```xml
<dependencies>
  <dependency><groupId>com.acme</groupId><artifactId>acme-spring-boot-autoconfigure</artifactId><version>${project.version}</version></dependency>
  <dependency><groupId>com.acme</groupId><artifactId>acme-greeting-core</artifactId><version>${project.version}</version></dependency>
</dependencies>
```

Properties:

```java
package com.acme.greeting;

@ConfigurationProperties(prefix = "acme.greeting")
public class GreetingProperties {
    /** Master switch. */
    private boolean enabled = true;
    /** Greeting template; {0} is the name. */
    private String template = "Hello, {0}!";
    /** Upper-case output. */
    private boolean shout = false;
    // getters + setters
}
```

Service (library class, no Spring annotations):

```java
public class GreetingService {
    private final String template; private final boolean shout;
    public GreetingService(String template, boolean shout) { this.template = template; this.shout = shout; }
    public String greet(String name) { String s = MessageFormat.format(template, name); return shout ? s.toUpperCase() : s; }
}
```

Auto-configuration:

```java
package com.acme.greeting;

@AutoConfiguration                                   // = @Configuration(proxyBeanMethods = false) + ordering aliases
@ConditionalOnClass(GreetingService.class)           // library on classpath
@ConditionalOnProperty(prefix = "acme.greeting", name = "enabled", havingValue = "true", matchIfMissing = true)
@EnableConfigurationProperties(GreetingProperties.class)
public class GreetingAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean                        // user-defined GreetingService wins
    GreetingService greetingService(GreetingProperties p) {
        return new GreetingService(p.getTemplate(), p.isShout());
    }

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnClass(HealthIndicator.class)       // nested: actuator is optional; referenced class only touched when present
    static class GreetingHealthConfiguration {
        @Bean
        @ConditionalOnMissingBean(name = "greetingHealthIndicator")
        @ConditionalOnEnabledHealthIndicator("greeting")          // management.health.greeting.enabled
        HealthIndicator greetingHealthIndicator(GreetingService s) {
            return () -> Health.up().withDetail("sample", s.greet("probe")).build();
        }
    }
}
```

The registration file `src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

```
com.acme.greeting.GreetingAutoConfiguration
```

Additional metadata (deprecation, hints) - optional `META-INF/additional-spring-configuration-metadata.json`:

```json
{ "properties": [ { "name": "acme.greeting.legacy-format", "type": "java.lang.String",
    "deprecation": { "reason": "Replaced by template.", "replacement": "acme.greeting.template" } } ] }
```

Test with `ApplicationContextRunner` (no full Boot start, milliseconds):

```java
class GreetingAutoConfigurationTest {
    private final ApplicationContextRunner runner =
        new ApplicationContextRunner().withConfiguration(AutoConfigurations.of(GreetingAutoConfiguration.class));

    @Test void createsDefaultBean() {
        runner.run(ctx -> assertThat(ctx).hasSingleBean(GreetingService.class));
    }
    @Test void backsOffWhenUserDefinesBean() {
        runner.withBean(GreetingService.class, () -> new GreetingService("x", false))
              .run(ctx -> assertThat(ctx.getBean(GreetingService.class).greet("a")).isEqualTo("x"));
    }
    @Test void disabledByProperty() {
        runner.withPropertyValues("acme.greeting.enabled=false")
              .run(ctx -> assertThat(ctx).doesNotHaveBean(GreetingService.class));
    }
    @Test void hidesWhenLibraryMissing() {
        runner.withClassLoader(new FilteredClassLoader(GreetingService.class))
              .run(ctx -> assertThat(ctx).doesNotHaveBean(GreetingService.class));
    }
}
```

Consumer app: add `acme-spring-boot-starter`, set `acme.greeting.shout=true`, inject `GreetingService` - no `@Import`, no scan
config. Trace: `AutoConfigurationImportSelector` reads our imports file from the starter's autoconfigure jar -> filters -> `GreetingAutoConfiguration`
parsed after user config -> conditions pass -> bean registered -> `GreetingProperties` bound from the Environment.

Checklist for real starters: (1) auto-configs in a separate module from libs is convention, (2) every bean `@ConditionalOnMissingBean`,
(3) optional dependencies behind nested configs, (4) `@AutoConfiguration(after/before)` if you interact with other auto-configs (e.g.
`after = DataSourceAutoConfiguration.class` when you need a DataSource), (5) no `@ComponentScan` in auto-configs, (6) `proxyBeanMethods=false`,
(7) provide configuration metadata, (8) don't use `spring.*`/`server.*`/`management.*` prefixes, (9) native friendliness: declare
`RuntimeHintsRegistrar` if you use reflection.

---

# 4. Failure modes and production war stories

Format: **Symptom -> Diagnosis -> Fix**.

### FailureAnalyzer basics
Uncaught startup exceptions go through `handleRunFailure`: publish `ApplicationFailedEvent`, run `FailureAnalyzers` (loaded from `spring.factories`),
which walk the **cause chain** and, if one matches, print

```
***************************
APPLICATION FAILED TO START
***************************
Description:   <what happened>
Action:        <what to do>
```
instead of the stack trace (set `--debug` or `logging.level.org.springframework.boot.diagnostics=DEBUG` for the trace; it also logs the raw exception at debug).
Exit code is 1. Custom analyzers: implement `AbstractFailureAnalyzer<T extends Throwable>` and register under the `FailureAnalyzer` key in `spring.factories`.

| Message | Analyzer | Meaning / fix |
|---|---|---|
| `Web server failed to start. Port 8080 was already in use.` | `PortInUseFailureAnalyzer` | another process / previous instance; `lsof -i :8080`/`netstat -ano`; change `server.port`; in tests use `RANDOM_PORT` |
| `Parameter 0 of constructor in X required a bean of type 'Y' that could not be found.` | `NoSuchBeanDefinitionFailureAnalyzer` | wrong package (scan root), missing starter, condition negative (check report), missing `@Component`, profile mismatch |
| `The dependencies of some of the beans in the application context form a cycle:` | `BeanCurrentlyInCreationFailureAnalyzer` | Boot 2.6+ prohibits circular refs (`spring.main.allow-circular-references=true` is a band-aid). Fix design: extract a third bean, use events, `@Lazy` on one injection point |
| `Failed to configure a DataSource: 'url' attribute is not specified...` | `DataSourceBeanCreationFailureAnalyzer` | no url + no embedded DB; active profile missing, env var not passed, or exclude `DataSourceAutoConfiguration` if no DB is intended |
| `expected single matching bean but found 2` | `NoUniqueBeanDefinitionFailureAnalyzer` | `@Primary`, `@Qualifier`, or constructor parameter name matching bean name |
| `Binding to target ... failed: Property: app.timeout Value: "abc" Reason: failed to convert` | `BindFailureAnalyzer` / `InvalidConfigurationPropertyValueFailureAnalyzer` | check property source (`/actuator/env`) & type; use `Duration` syntax `5s` |
| `Name for argument of type [...] not specified, and parameter name information not found` | `MissingParameterNameFailureAnalyzer` | compile with `-parameters` |
| `Failed to bind properties under 'server.port' ... 'java.lang.String' to 'java.lang.Integer'` | bind failure | typos/placeholder unresolved: `${PORT}` env var missing -> `Could not resolve placeholder 'PORT'` |
| `A component required a bean named 'entityManagerFactory' that could not be found` | | JPA auto-config negative: no DataSource / no hibernate on classpath / excluded |
| `The bean 'x' could not be registered. A bean with that name has already been defined and overriding is disabled.` | `BeanDefinitionOverrideFailureAnalyzer` | Boot disallows overriding by default (2.1+); rename bean, don't enable `spring.main.allow-bean-definition-overriding` blindly |
| `Unable to start ServletWebServerApplicationContext due to missing ServletWebServerFactory bean` | `MissingWebServerFactory` | Tomcat excluded and no replacement, or web type NONE/servlet mismatch |
| `Unable to find a @SpringBootConfiguration` | (tests) | test class is not under the main package; add `classes=` |

### War story 1 - "Works on my machine, DataSource failed in prod pod"
- **Symptom:** CrashLoopBackOff, `Failed to configure a DataSource ... no profiles are currently active`.
- **Diagnosis:** `spring.profiles.active=prod` was set via a `ConfigMap` key `spring.profiles.active` used as a *env var name* (invalid env name with dots is dropped
  by some tooling); env var must be `SPRING_PROFILES_ACTIVE`. So `application-prod.yml` (holding the URL) never loaded.
- **Fix:** correct env var; add startup log of active profiles; fail-fast `@Validated` props for `spring.datasource.url`.

### War story 2 - Liveness probe restarts the fleet during a DB blip
- **Symptom:** DB failover for 40 s; every pod restarted, outage lasted 10 minutes (cold caches, thundering herd on the DB).
- **Diagnosis:** `management.endpoint.health.group.liveness.include=livenessState,db` (or the probe pointed at `/actuator/health`, which includes db).
- **Fix:** liveness = only internal state; readiness may include DB (pods just leave the Service, not killed); add `startupProbe`.

### War story 3 - 503s during every deployment
- **Symptom:** spike of 502/503 for ~5 s per rollout.
- **Diagnosis:** default `server.shutdown=immediate`; even with graceful, ingress kept routing to a terminating pod.
- **Fix:** `server.shutdown=graceful`, `spring.lifecycle.timeout-per-shutdown-phase=30s`, `preStop: sleep 10`, `terminationGracePeriodSeconds: 60`,
  readiness flips at shutdown, `maxUnavailable=0` in the rollout.

### War story 4 - Jackson behaviour changed after "a small refactor"
- **Symptom:** dates became arrays `[2024,5,1]`, unknown JSON fields started failing with 400, `spring.jackson.*` ignored.
- **Diagnosis:** someone added `@Bean ObjectMapper objectMapper() { return new ObjectMapper(); }` -> `JacksonAutoConfiguration` backed off
  (`@ConditionalOnMissingBean`), losing Boot defaults and module registration.
- **Fix:** delete the bean, or use `Jackson2ObjectMapperBuilder`/`Jackson2ObjectMapperBuilderCustomizer`; verify via `/actuator/beans` or the conditions report.

### War story 5 - `@EnableWebMvc` killed static resources and defaults
- **Symptom:** `/swagger-ui`/static content 404, date format changed, message converters different.
- **Diagnosis:** `@EnableWebMvc` (or extending `WebMvcConfigurationSupport`) disables `WebMvcAutoConfiguration`.
- **Fix:** remove, use `WebMvcConfigurer`.

### War story 6 - Beans missing after moving the main class
- **Symptom:** `NoSuchBeanDefinitionException` for a `@Service`; or JPA entity not found `Not a managed type`.
- **Diagnosis:** `@SpringBootApplication` in `com.acme.app` but components in `com.acme.shared` (sibling) - component scan and `@AutoConfigurationPackage`
  root are wrong.
- **Fix:** move main class to the common root, or `scanBasePackages`, `@EntityScan`, `@EnableJpaRepositories`.
  (Watch out: explicitly using `@EnableJpaRepositories` turns off Boot's repository auto-config defaults.)

### War story 7 - All requests hang; thread dump shows 200 threads in `SocketRead`
- **Symptom:** p99 climbs, CPU low, Tomcat `threads.busy=200`.
- **Diagnosis:** downstream service slow, no client timeouts (`RestTemplate` default infinite); worker pool exhausted; Hikari waiting (`hikaricp.connections.pending`).
- **Fix:** timeouts (`spring.http.client...`/`RestClient` factory settings), circuit breaker/bulkhead, size Hikari vs Tomcat threads, consider virtual threads on JDK 21
  (`spring.threads.virtual.enabled=true`, mind pinning with `synchronized` + blocking I/O on JDK 21 and DB pool limits, which now become the bottleneck).

### War story 8 - Configuration silently ignored
- **Symptom:** `server.tomcat.max-threads` (old name) has no effect; `spring.datasource.hikari.maximumPoolSize` typo.
- **Diagnosis:** unknown properties aren't errors unless `@ConfigurationProperties(ignoreUnknownFields=false)`; Boot 3 renamed (`server.tomcat.threads.max`).
- **Fix:** `spring-boot-properties-migrator` during upgrades; `/actuator/configprops` and `/actuator/env` to inspect; IDE with metadata.

### War story 9 - Runner never finishes -> pod never Ready
- **Symptom:** log shows "Started App in 12s" but readiness stays DOWN/`REFUSING_TRAFFIC`; port open, `/actuator/health/readiness` 503.
- **Diagnosis:** a `CommandLineRunner` performs a long loop (cache warm-up, `while(true)` consumer). `ApplicationReadyEvent` only fires after runners complete.
- **Fix:** move background loops to `SmartLifecycle`/`@Async`/own thread; keep runners short; or manage readiness via `AvailabilityChangeEvent`.

### War story 10 - `OutOfMemoryError: Metaspace`/leak after DevTools/hot deploy or repeated context creation
- **Symptom:** dev machine fine on first run, degrades after many restarts; or tests create hundreds of contexts.
- **Diagnosis:** `@MockBean`/different `@TestPropertySource` combinations produce distinct cached contexts (context cache key); each has its own Tomcat/Hikari.
- **Fix:** standardise test configs to share contexts, use slices, `@DirtiesContext` sparingly.

### War story 11 - Conditional bean surprise: `@ConditionalOnBean` in normal config
- **Symptom:** bean sometimes exists, sometimes not depending on component scan order / after refactor.
- **Diagnosis:** `@ConditionalOnBean` in a user `@Configuration` (not auto-config) evaluated before the referenced bean is registered.
- **Fix:** move to an auto-configuration with `@AutoConfiguration(after=...)`, or use `@Import`/`@DependsOn`, or `ObjectProvider`.

### War story 12 - Native image: JSON returns `{}` only in native
- **Symptom:** works on JVM, in native the DTO serialises empty or `NoSuchMethodException` in reflection code.
- **Diagnosis:** missing reflection hints for a class only used reflectively (e.g. deserialising into a class not discovered by AOT: generics, `Object` fields).
- **Fix:** `@RegisterReflectionForBinding(Dto.class)` / `RuntimeHintsRegistrar`; run `nativeTest`; check `META-INF/native-image` output.

### War story 13 - `/actuator/env` exposed publicly
- **Symptom:** security audit finds secrets; `/actuator/heapdump` downloadable.
- **Diagnosis:** `management.endpoints.web.exposure.include=*` copied from a dev sample and ingress routes port 8080.
- **Fix:** explicit include list, management port on an internal interface, Security rules, `show-values=NEVER`, rotate the leaked secrets.

### Debug toolbox (quick)
`--debug` (conditions), `/actuator/{beans,conditions,configprops,env,mappings,threaddump,startup,loggers}`,
`logging.level.org.springframework.boot.context.config=TRACE` (config data loading: which files were considered),
`logging.level.org.springframework.web=DEBUG` (`web` group), `jcmd <pid> Thread.print`, `-Dspring.aot.enabled=true` behaviour check,
`/actuator/mappings` to see which controller/handler matches a URL.

---

# 5. Interview questions (40)

Format: **Q (level)** -> model answer -> follow-up chain -> common wrong answers.

### Startup / SpringApplication

**Q1 (Easy). What does `@SpringBootApplication` do?**
Composite of `@SpringBootConfiguration` (a `@Configuration`), `@EnableAutoConfiguration` (imports auto-config through
`AutoConfigurationImportSelector`), and `@ComponentScan` (package of the class and below, with `TypeExcludeFilter` and `AutoConfigurationExcludeFilter`).
- F1: Why do I get "bean not found" for a class in another package? -> outside scan root; `scanBasePackages`/move main class.
- F2: What is `AutoConfigurationExcludeFilter` for? -> stops scanning from picking up auto-config classes that live under your scan root; they must go through deferred import.
- F3: What does `@AutoConfigurationPackage` do? -> records the main package for JPA/Data/Mybatis defaults.
- Wrong: "It starts Tomcat." (that's the refresh of a web context) / "It scans the whole classpath."

**Q2 (Easy). What is a starter?**
A dependency descriptor (POM) grouping libraries for a feature; the actual configuration lives in `spring-boot-autoconfigure` and
activates by classpath presence. Versions come from the BOM (`spring-boot-dependencies` via parent or import).
- F1: Difference starter vs autoconfigure module? -> starter has no code; autoconfigure has `@AutoConfiguration` classes.
- F2: How do you override a managed version? -> property like `<jackson-bom.version>`/`ext['x.version']`.
- Wrong: "A starter contains the auto-configuration classes" (only in tiny/in-house single-module cases).

**Q3 (Medium). Walk me through `SpringApplication.run()`.**
Answer with the timeline in 2.2: constructor (web type, factories) -> Starting event -> prepareEnvironment (+ EnvironmentPostProcessors load config) ->
banner -> create context by type -> prepareContext (initializers, ContextInitialized, load sources, ContextPrepared/Loaded events) -> refresh (scan, deferred auto-config,
create web server, instantiate singletons, start web server in finishRefresh) -> Started -> runners -> Ready.
- F1: Where exactly does the port open? -> `finishRefresh` via `WebServerStartStopLifecycle`; server object created in `onRefresh`.
- F2: Which event can a `@Component` listener NOT receive? -> Starting/EnvironmentPrepared/ContextInitialized (context not yet there); need `spring.factories` or `addListeners`.
- F3: Started vs Ready? -> runners in between.
- Wrong: "Tomcat starts first, then the context" / "auto-config runs before component scan."

**Q4 (Medium). How does Boot decide it's a servlet, reactive or non-web app?**
`WebApplicationType.deduceFromClasspath()`: WebFlux `DispatcherHandler` and no MVC `DispatcherServlet` (and no Jersey) -> REACTIVE; Servlet API or
`ConfigurableWebApplicationContext` missing -> NONE; else SERVLET. Override with `spring.main.web-application-type`.
- F1: Both MVC and WebFlux on classpath? -> SERVLET. F2: Why would you set NONE? -> CLI/batch/worker. F3: What differs? -> context class + web auto-configs.

**Q5 (Medium). What is `spring.factories` used for in Boot 3? What replaced it for auto-config?**
Still used for `ApplicationContextInitializer`, `ApplicationListener`, `EnvironmentPostProcessor`, `FailureAnalyzer`, `SpringApplicationRunListener`, etc.
Auto-config moved to `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (2.7 introduced, 3.0 exclusive).
- F1: What happens to an unmigrated library on 3.0? -> auto-config silently not applied. F2: Why change? -> dedicated format, faster, no key confusion, AOT.
- Wrong: "spring.factories was removed entirely."

**Q6 (Medium). List the run-listener events in order.**
`ApplicationStartingEvent`, `ApplicationEnvironmentPreparedEvent`, `ApplicationContextInitializedEvent`, `ApplicationPreparedEvent`,
(`ContextRefreshedEvent`, `WebServerInitializedEvent`), `ApplicationStartedEvent`, `AvailabilityChangeEvent(LivenessState.CORRECT)`,
runners, `ApplicationReadyEvent`, `AvailabilityChangeEvent(ReadinessState.ACCEPTING_TRAFFIC)`; failure: `ApplicationFailedEvent`.
- F1: Which class publishes them? -> `EventPublishingRunListener` (+ `SpringApplicationRunListeners` wrapper). F2: How do you hook the environment before beans? -> `EnvironmentPostProcessor` in `spring.factories`.

**Q7 (Hard). Explain how `application.yml` gets loaded and how `spring.config.import` fits.**
`ConfigDataEnvironmentPostProcessor` (triggered by `ApplicationEnvironmentPreparedEvent`) runs `ConfigDataEnvironment`: resolves locations
(`classpath:/`, `classpath:/config/`, `file:./`, `file:./config/`, `file:./config/*/`), loads contributors with `PropertySourceLoader`s
(`.properties`, `.yml`), activates profiles after the first pass, loads profile-specific files, and processes `spring.config.import`
recursively. Since Boot 2.4 replacing the legacy `ConfigFileApplicationListener`.
- F1: Why can't `application-prod.yml` set `spring.profiles.active`? -> profile determination precedes profile-specific loading; throws `InvalidConfigDataPropertyException`.
- F2: `optional:` prefix? -> ignore missing. F3: `spring.config.location` vs `additional-location`? -> replace vs add.
- Wrong: "`ConfigFileApplicationListener`" (that's legacy <= 2.3 with `spring.config.use-legacy-processing` removed later).

**Q8 (Medium). What does `prepareEnvironment` do and what is `configurationProperties` property source?**
Creates the `Environment` for the web type, adds command-line args, attaches `ConfigurationPropertySources` (adapter enabling relaxed name lookups over all real property sources),
fires the event so post-processors add sources, binds `spring.main.*`.
- F1: Can `@PropertySource` set `spring.main.banner-mode`? -> no; too late.

### Auto-configuration

**Q9 (Easy). How does auto-configuration know what to configure?**
Reads candidate class names from every `AutoConfiguration.imports`; filters by conditions - classpath (`@ConditionalOnClass`), properties, existing beans; registers matching ones.
- F1: How do I turn one off? -> `exclude`, `spring.autoconfigure.exclude`. F2: How to see why? -> `--debug`/`/actuator/conditions`.
- Wrong: "It scans all classes in all jars for @Configuration."

**Q10 (Medium). Why is `AutoConfigurationImportSelector` a `DeferredImportSelector`?**
So auto-config is processed after all user `@Configuration`s; that makes `@ConditionalOnMissingBean` correctly detect user beans and lets user bean win. Also enables group processing and
ordering via `AutoConfigurationSorter`.
- F1: Does that mean `@ConditionalOnBean` in my own config is safe? -> no. F2: How do you order two auto-configs? -> `@AutoConfiguration(before/after)`; alphabetic then `@AutoConfigureOrder` then before/after.
- Wrong: "Deferred means lazily loaded at first use" (no - deferred *processing order*, not lazy beans).

**Q11 (Medium). Difference between `@ConditionalOnClass` and `@ConditionalOnMissingBean`, and when are they evaluated?**
Class: parse phase, ASM metadata (safe when class missing on class-level); bean: registration phase, considers definitions registered so far.
- F1: `@ConditionalOnClass` on a `@Bean` method with a class reference - problem? -> `NoClassDefFoundError` when reflecting the config class; use `name=` or nested config.
- F2: Which conditions run first? -> class-level cheap ones (`@Order(HIGHEST_PRECEDENCE)` on Boot conditions).

**Q12 (Medium). What is the condition evaluation report and how do you use it?**
`ConditionEvaluationReport`: positive matches, negative matches, exclusions, unconditional classes. `--debug`, `/actuator/conditions`.
Use it to see why a bean is absent/present. Also `logging.level...autoconfigure=DEBUG` variations.
- F1: A bean from Boot overrides mine? -> check its `@ConditionalOnMissingBean` type vs your bean type.
- F2: Is `--debug` the same as root DEBUG? -> no.

**Q13 (Hard). A library's auto-config is present but nothing happens. Enumerate causes.**
(1) imports file wrong path/name/typo or not in the jar (shading/merging strategy dropped it - shade plugin `AppendingTransformer` needed);
(2) class not annotated properly / loaded via old `spring.factories` on Boot 3; (3) `@ConditionalOnClass` fails (optional dep missing);
(4) excluded by property; (5) class filtered because `spring-autoconfigure-metadata.properties` stale; (6) user bean of same type defined; (7) web type mismatch;
(8) conditional property not set/`matchIfMissing` false; (9) wrong ordering vs another auto-config; (10) `@ComponentScan` picked it early - excluded by filter, thus not processed twice.
- F1: Fat jar plugin merging? -> Maven Shade `ServicesResourceTransformer` isn't enough; merging `.imports` and `spring.factories` needs `AppendingTransformer`. Spring Boot's own repackage doesn't have this issue (nested jars).

**Q14 (Hard). Trace when `DataSourceAutoConfiguration` creates a bean and when it doesn't.** See 3.1.
Key points: requires spring-jdbc `EmbeddedDatabaseType`; pooled config takes precedence over embedded when Hikari is on classpath; backs off if any `DataSource`/`XADataSource` bean;
missing URL with no embedded DB -> `DataSourceBeanCreationException`.
- F1: Multiple DataSources? -> disable auto or define `@Primary` + explicit props; JPA needs single candidate.
- F2: Which pool is chosen if several? -> Hikari > Tomcat JDBC > DBCP2 > UCP (in the `@Import` order; `spring.datasource.type` forces one).

**Q15 (Medium). `@ConditionalOnProperty` semantics?**
`prefix`+`name`, `havingValue`, `matchIfMissing`. Unset havingValue: matches unless value is "false". Absent property fails unless `matchIfMissing=true`.
- F1: `havingValue="true"` and property `TRUE`? -> comparison is case-insensitive string equality (`equalsIgnoreCase`).
- Wrong: "matches if property is non-empty."

**Q16 (Hard). Why should Boot auto-configs use `proxyBeanMethods=false`?**
Avoids CGLIB subclass generation -> faster startup, less memory, AOT/native compat. Cost: `@Bean` method calls aren't intercepted so inter-bean references must use parameters.
- F1: When must you keep proxying? -> when config classes call `@Bean` methods of each other and expect singleton semantics (legacy code).

**Q17 (Medium). Can I write my own auto-configuration in an application (not a library)?**
Yes, but rarely needed; use normal `@Configuration`. If done, register in the app's own imports file. Key points for the conceptual difference: must use `@ConditionalOnMissingBean` pattern; not be under scan root or be excluded.

**Q18 (Hard). How do you write a custom starter? What could go wrong?**
See 3.2: two modules, `@AutoConfiguration`, imports file, properties + processor, conditional beans with `@ConditionalOnMissingBean`, nested configs for optional deps,
`ApplicationContextRunner` tests. Wrong things: no imports file (Boot 3), missing `@ConditionalOnMissingBean` so users cannot override, forcing optional deps, `@ComponentScan` inside, property prefix `spring.`.
- F1: How to test without full app? -> `ApplicationContextRunner`, `FilteredClassLoader`. F2: How to make IDE complete properties? -> configuration processor.

### Configuration

**Q19 (Easy). `@Value` vs `@ConfigurationProperties`?** Table in 2.6. Prefer `@ConfigurationProperties` for groups; `@Value` for one-off/SpEL.
- F1: Can `@ConfigurationProperties` be refreshed at runtime? -> not in plain Boot.

**Q20 (Medium). Explain relaxed binding. How do you set `app.hosts[0]` via env var?** `APP_HOSTS_0=...`; kebab in files; canonical form lower-kebab.
- F1: Env var for `my.property-name`? -> `MY_PROPERTYNAME`. F2: Why are dashes problematic in env names? -> not valid in shell names; use `SPRING_APPLICATION_JSON`.
- Wrong: "env var must be exactly the property with dots" (dots are illegal in most shells).

**Q21 (Medium). Property precedence from highest to lowest (main levels)?** CLI args > `SPRING_APPLICATION_JSON` > system properties > env vars > random > profile-specific external > external app props > profile-specific
packaged > packaged app props > `@PropertySource` > defaults. (test props and devtools above.)
- F1: Env var vs `application-prod.yml` - who wins? -> env var. F2: `.properties` vs `.yml`? -> properties.
- Wrong: "application.yml beats env vars" / "-D beats --args".

**Q22 (Medium). How do you do constructor binding in Boot 3 and why prefer it?** Immutable, thread safe, validated at bind time; single ctor -> automatic;
`@ConstructorBinding` only with multiple ctors; not a `@Component`.
- F1: Defaults? -> `@DefaultValue`. F2: Validation? -> `@Validated` + jakarta constraints, nested `@Valid`.

**Q23 (Medium). How do profiles work? Groups? `@Profile`?** `spring.profiles.active/default/include/group`, multi-doc `spring.config.activate.on-profile`, `@Profile` expression support; cannot activate inside profile files.
- F1: Two active profiles both define the same key? -> the last listed wins.
- F2: How do you pass profile in Kubernetes? -> env `SPRING_PROFILES_ACTIVE`.

**Q24 (Hard). Design config for 12-factor on Kubernetes: non-secret, secret, and dynamic.**
Non-secret: ConfigMap as env vars or mounted `application.yml` via `spring.config.additional-location` / `configtree:` import. Secrets: mounted files + `configtree:`
(avoid env vars, `/proc`, crash dumps); externalise per environment via profiles only for structure. Dynamic: restart via rollout (checksum annotation on the Deployment), or Spring Cloud Kubernetes/Config `@RefreshScope`.
Validation at startup via `@Validated` props; `env` endpoint values hidden.
- F1: Why restart on ConfigMap change? -> Boot doesn't watch files. F2: How to prevent secrets leaking in actuator? -> `show-values`, key sanitising rules (`password`, `secret`, `key`, `token`... regex), not exposing env.

### Web server / MVC

**Q25 (Medium). How to switch Tomcat to Jetty/Undertow?** Exclude `spring-boot-starter-tomcat` from the web starter, add the other starter. `@ConditionalOnClass` in `ServletWebServerFactoryConfiguration` flips.
- F1: Why does Boot need exactly one `ServletWebServerFactory`? -> `getWebServerFactory` throws if 0 or >1. F2: How to tune Tomcat? -> properties or `WebServerFactoryCustomizer`.

**Q26 (Hard). Explain `threads.max`, `max-connections`, `accept-count`. What happens at 10,000 concurrent clients?**
Per 2.9 diagram: acceptor admits up to 8192 connections; up to 200 concurrent requests; excess connections wait (idle in poller/socket buffer);
after 8192, 100 backlog; beyond that refused/time out. Bottleneck is usually downstream (DB pool 10) - increasing threads without pool changes does nothing.
- F1: How do virtual threads change this? -> request threads unbounded-ish; constraint moves to pools/downstreams. F2: Keep-alive effect? -> idle connections don't consume worker threads (NIO).

**Q27 (Medium). Describe request processing in `DispatcherServlet`.** Section 2.10: mappings, adapters, interceptors, argument resolvers, converters, exception resolvers.
- F1: Filter vs interceptor vs advice? -> servlet container vs MVC vs AOP-ish for controllers/body. F2: Where is `@Valid` performed? -> in argument resolvers (`RequestResponseBodyMethodProcessor`) -> `MethodArgumentNotValidException`.

**Q28 (Hard). How is an exception in a controller turned into a response? What about one thrown in a filter?**
`HandlerExceptionResolver` chain: `ExceptionHandlerExceptionResolver` -> `ResponseStatusExceptionResolver` -> `DefaultHandlerExceptionResolver`; unresolved -> `/error` -> `BasicErrorController`.
Filter exceptions bypass MVC resolvers; only error dispatch.
- F1: How does `DefaultErrorAttributes` get exception? -> it's also a resolver at highest precedence recording the attribute.
- F2: Why is my message missing in error JSON? -> `server.error.include-message=never` default.

**Q29 (Medium). What's `ProblemDetail` and how to enable it in Boot 3?** RFC 7807 body `application/problem+json`; `spring.mvc.problemdetails.enabled=true` or extend `ResponseEntityExceptionHandler`.
- F1: Custom fields? -> `setProperty`. F2: WebFlux? -> `spring.webflux.problemdetails.enabled`.

**Q30 (Medium). How to customise Jackson? What if I declare my own `ObjectMapper`?** properties, `Jackson2ObjectMapperBuilderCustomizer`, `Module` beans; own mapper disables auto-config and loses defaults (dates, unknown props).

### Actuator / ops

**Q31 (Easy). What is Actuator? How do you secure it?** Ops endpoints; exposure include list; separate port; Security `EndpointRequest`; `show-details=when-authorized`.
- F1: Default exposed? -> `health` only (web). F2: Danger endpoints? -> `env`, `heapdump`, `threaddump`, `loggers` (write), `shutdown`, `jolokia`.

**Q32 (Medium). Liveness vs readiness. What should each include?** Section 2.11. Liveness internal only; readiness gates traffic and can include dependencies. Startup probe for slow boots.
- F1: What happens to readiness on SIGTERM? -> `REFUSING_TRAFFIC`. F2: Why not DB in liveness? -> cascading restarts.
- Wrong: "Both should call the same `/health`."

**Q33 (Medium). How do you write a custom `HealthIndicator` and a custom endpoint?** Code in 2.11; name from bean name; `@Endpoint` + `@ReadOperation`; exposure needed.
- F1: Reactive? -> `ReactiveHealthIndicator`. F2: Timeouts? -> avoid blocking; cache TTL.

**Q34 (Medium). How does Micrometer work in Boot? Prevent cardinality explosion?** `MeterRegistry` per backend, auto binders, `http.server.requests`, custom meters, tags bounded, `MeterFilter` deny/limits (`MeterFilter.maximumAllowableTags`).
- F1: `@Timed` requires? -> `TimedAspect` bean. F2: Boot 3 change? -> Observation API/tracing replaces Sleuth.

**Q35 (Medium). Change log level of one package at runtime without restart.** `POST /actuator/loggers/{name}` `{"configuredLevel":"DEBUG"}`; reset with null; needs exposure; not persisted; per instance.
- F1: Difference `logback.xml` vs `logback-spring.xml`? -> Spring extensions and early init. F2: Logging groups? -> `logging.group.*`.

**Q36 (Hard). Explain graceful shutdown end to end in Kubernetes.** Section 2.14 - SIGTERM, hook, readiness change, connector pause, wait, phases (SmartLifecycle), destroy; K8s endpoint propagation race, preStop,
grace period > timeouts; unsent async tasks.
- F1: What's `SmartLifecycle.stop(Runnable)` contract? -> must call callback. F2: Why doesn't it work with `kill -9`? -> no hook.
- Wrong: "`@PreDestroy` is enough to drain HTTP requests."

### Packaging / native / performance

**Q37 (Medium). What's inside a Boot fat jar? How does it run?** Section 2.15; `JarLauncher`, nested jars, `Start-Class`, layers index. `getFile()` on classpath resource fails.
- F1: Why layered jars? -> Docker caching. F2: `PropertiesLauncher`? -> external classpath/`loader.path`.

**Q38 (Hard). What is AOT and what breaks in native image?** Section 2.16: build-time bean definition generation, hints; conditions/profiles fixed at build; reflection/proxies/resources need hints; no runtime bytecode generation;
build costs.
- F1: Does `@ConditionalOnProperty` work at native runtime? -> evaluated at AOT time. F2: Does it make throughput higher? -> no, startup/memory only.
- Wrong: "Native image is just a faster JVM" / "AOT = GraalVM only" (AOT also for JVM).

**Q39 (Hard). App takes 45 s to start. Plan of attack.** Measure (`/actuator/startup`, logs, JFR) -> identify heavy beans (JPA/Flyway/scanning/remote calls in `@PostConstruct`) -> trim starters/auto-config ->
lazy init selectively -> JPA deferred bootstrap -> AppCDS -> CPU limits on K8s -> AOT/CRaC/native as last resort. Add `startupProbe`.
- F1: Risks of global lazy init? -> late failures, first-request latency, hidden misconfig. F2: Where is CDS created? -> training run at image build.

**Q40 (Medium). Common startup failures and how Boot helps you read them.** `FailureAnalyzer` table in section 4 (port in use, NoSuchBean, circular, DataSource); `--debug` shows raw stack.
- F1: Circular dependency since 2.6? -> fails by default; `allow-circular-references`.
- F2: How to add a custom analyzer? -> `AbstractFailureAnalyzer` + `spring.factories`.

**Q41 (Hard). Boot 2.7 -> 3.x migration: what breaks first?** Java 17; `javax->jakarta` (dependencies!); imports file; renamed props; Hibernate 6 sequences/HQL; Security 6 lambdas/`requestMatchers`; trailing slash;
`-parameters`; `httptrace`; Sleuth -> Micrometer Tracing; third-party incompatibility (Springfox). Approach: 2.7 latest, remove deprecations, properties-migrator, OpenRewrite, staged testing with prod-like DB.
- F1: Hibernate sequence issue detail? -> per-entity sequence naming default vs old `hibernate_sequence`.

**Q42 (Hard). Why can `spring.jpa.open-in-view` be a production issue?** (Boot warns at startup.) OSIV keeps EntityManager/connection bound to the request thread until view rendering ->
connections held during serialisation/downstream calls; lazy loading N+1 hidden in controllers. Disable `spring.jpa.open-in-view=false`, fetch what you need in the service layer.
- Wrong: "It is a performance optimisation."

### Common wrong answers (quick list)
- "Boot uses its own DI container" -> it's Spring's.
- "Auto-config always overrides my beans" -> it backs off (`@ConditionalOnMissingBean`) - unless your bean type doesn't match the condition.
- "`application.properties` always wins" -> CLI/env/system/JSON beat it.
- "Actuator exposes all endpoints by default" -> only `health` over web.
- "Deferred import = lazy bean creation".
- "Embedded Tomcat starts before the ApplicationContext" -> created in `onRefresh`, connectors started at `finishRefresh`.
- "`@ConditionalOnProfile`" -> doesn't exist; `@Profile`.
- "`spring.factories` is gone in Boot 3" -> only the auto-config key moved.
- "Graceful shutdown is on by default" -> default is `immediate`.
- "Liveness should check the database".

---

# 6. One-page cheat sheet

```
RUN ORDER   new SpringApplication(web type, initializers, listeners from spring.factories)
            Starting -> prepareEnvironment (EnvironmentPreparedEvent; config data loaded) -> banner -> create ctx (servlet/reactive/none)
            -> prepareContext (initializers, ContextInitialized, ContextPrepared/Loaded=ApplicationPreparedEvent)
            -> refresh (scan -> DEFERRED auto-config -> onRefresh creates Tomcat -> singletons -> finishRefresh opens port)
            -> Started(+Liveness CORRECT) -> runners -> Ready(+Readiness ACCEPTING_TRAFFIC).  Failure: ApplicationFailedEvent + FailureAnalyzer.
LISTENERS   Bean listeners only from ApplicationPreparedEvent on; earlier ones need spring.factories/addListeners.

@SpringBootApplication = @SpringBootConfiguration + @EnableAutoConfiguration(@AutoConfigurationPackage + Import selector) + @ComponentScan(+TypeExclude/AutoConfigExclude filters)

AUTOCONFIG  META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports (Boot 3 only; spring.factories key is ignored)
            DeferredImportSelector => after user config => @ConditionalOnMissingBean sees user beans
            Order: alphabetical -> @AutoConfigureOrder -> before/after.  Exclude: exclude=, spring.autoconfigure.exclude
CONDITIONS  Class/property/web/expression/profile at parse; Bean/MissingBean/SingleCandidate at register.  No @ConditionalOnProfile - use @Profile.
            Debug: --debug, /actuator/conditions (Positive, Negative, Exclusions, Unconditional)

CONFIG PRECEDENCE (high->low) test props > devtools > CLI args > SPRING_APPLICATION_JSON > JNDI/servlet params > system props (-D) > env vars > random
            > profile files external > app files external > profile files in jar > app files in jar > @PropertySource > defaults
            locations: classpath:/ < classpath:/config < ./ < ./config < ./config/*/ ; .properties > .yml ; last profile wins
PROFILES    spring.profiles.active|default|include|group.x ; doc split: spring.config.activate.on-profile ; NOT active inside profile file
IMPORT      spring.config.import=optional:file:...|optional:configtree:/etc/cfg/ ; K8s: env SPRING_X_Y, mounted files, configtree for secrets
BINDING     relaxed (kebab in files, UPPER_SNAKE env, _0 indexes) ; records => ctor binding ; @Validated + jakarta ; @EnableConfigurationProperties | @ConfigurationPropertiesScan

TOMCAT      server.tomcat.threads.max=200 | threads.min-spare=10 | accept-count=100 | max-connections=8192 | max-keep-alive-requests=100
            server.shutdown=graceful ; spring.lifecycle.timeout-per-shutdown-phase=30s ; spring.threads.virtual.enabled (3.2/JDK21)
            switch server: exclude starter-tomcat + add starter-jetty|undertow
MVC         DispatcherServlet: HandlerMapping -> HandlerAdapter -> interceptors -> arg resolvers -> method -> return handlers/converters
            exceptions: @ExceptionHandler/@ControllerAdvice -> ResponseStatus -> DefaultHandler -> /error (BasicErrorController)
            ProblemDetail: spring.mvc.problemdetails.enabled=true ; own ObjectMapper bean = lose Boot Jackson defaults ; @EnableWebMvc = lose MVC autoconfig

ACTUATOR    web exposure default = health only ; management.endpoints.web.exposure.include ; management.server.port ; probes: /health/liveness, /health/readiness
            liveness=internal only ; readiness may include deps ; custom: HealthIndicator, @Endpoint+@ReadOperation ; Micrometer: MeterRegistry, avoid high-cardinality tags
LOGGING     logback-spring.xml ; logging.level.* ; logging.group.* ; POST /actuator/loggers/{name} {"configuredLevel":"DEBUG"}
SHUTDOWN    SIGTERM -> hook -> readiness REFUSING -> pause connectors -> drain (timeout) -> SmartLifecycle stop high->low phase -> destroy beans
JAR         Main-Class JarLauncher, Start-Class, BOOT-INF/{classes,lib,classpath.idx,layers.idx}; layers: dependencies/loader/snapshot/application
NATIVE      AOT at build: bean defs + hints; conditions/profiles frozen; reflection/proxy/resources need hints; -Dspring.aot.enabled=true on JVM
FAST START  trim starters/excludes, lazy-init (trade-offs), JPA deferred bootstrap, AppCDS, startupProbe, measure with /actuator/startup
FAILURES    port in use | NoSuchBean (scan root/condition) | circular (prohibited 2.6+) | DataSource url (missing profile!) | binding | -parameters
MIGRATE 3   Java 17, jakarta, .imports, property renames, Hibernate 6 sequences, Security 6 (SecurityFilterChain, requestMatchers), trailing slash, -parameters
```

Rules of thumb: (1) fail fast at startup, (2) every starter bean backs off, (3) liveness never checks dependencies, (4) never expose actuator wide open,
(5) know which PropertySource won (`/actuator/env`), (6) if unsure why a bean exists: conditions report.
