# Spring Boot Internals (Startup, Auto-Configuration, Starters, Profiles)

## What does `@SpringBootApplication` really contain?
```
@SpringBootApplication  =  @SpringBootConfiguration   (this class is a config class)
                         + @EnableAutoConfiguration   (magic: configure beans automatically)
                         + @ComponentScan             (find @Component/@Service in this package & below)
```

## Startup flow (`SpringApplication.run()`)

```
main() → SpringApplication.run()
   │
   ▼
1. Create Environment (application.properties/yml, env vars, CLI args)
   ▼
2. Create ApplicationContext (IoC container)
   ▼
3. Component scan → register your beans (@Service, @Controller…)
   ▼
4. Auto-configuration → read list of config classes from
   META-INF/spring/...AutoConfiguration.imports (in each starter jar)
   ▼
5. For each auto-config class, evaluate @Conditional...
   (is class on classpath? bean missing? property set?)  → register bean only if true
   ▼
6. Instantiate singletons, inject dependencies, run @PostConstruct
   ▼
7. Start embedded Tomcat (port 8080)
   ▼
8. Run CommandLineRunner / ApplicationRunner  → app READY
```

## Auto-configuration — "how does Boot know what to configure?"

**Analogy:** A smart hotel room. If you bring a laptop (JDBC driver on classpath), it auto-provides Wi-Fi (DataSource). If you bring your own router (you define your own `DataSource` bean), the hotel backs off.

```
spring-boot-starter-web added?
   → Tomcat + DispatcherServlet + Jackson on classpath
   → @ConditionalOnClass(DispatcherServlet)  ✔  → WebMvcAutoConfiguration kicks in

You define your own DataSource bean?
   → @ConditionalOnMissingBean(DataSource)   ✘  → Boot's default is skipped
```

Key conditionals:
| Annotation | Bean created only if… |
|---|---|
| `@ConditionalOnClass` | class present on classpath |
| `@ConditionalOnMissingBean` | you haven't defined that bean |
| `@ConditionalOnProperty` | property has certain value |
| `@ConditionalOnBean` | another bean exists |

Custom starter = your own `@AutoConfiguration` class + entry in `AutoConfiguration.imports`.

Debug: run with `--debug` → prints **conditions evaluation report** (positive/negative matches).

## Starters
A starter = a **dependency bundle** (no code). `spring-boot-starter-web` = Spring MVC + Tomcat + Jackson + validation… Versions managed by `spring-boot-starter-parent`/BOM → no version clashes.

## Config & profiles
Property priority (high → low): command-line args > env variables > `application-{profile}.yml` > `application.yml` > defaults.
```
java -jar app.jar --spring.profiles.active=prod
```
```java
@Value("${app.timeout}") int t;                 // single value
@ConfigurationProperties(prefix="app") class AppProps {...}   // typed group (preferred)
```

## Actuator (production monitoring)
`/actuator/health`, `/metrics`, `/info`, `/env`, `/loggers`. Used by Kubernetes for **liveness/readiness probes**. Secure it – never expose `/env` publicly.

## Frequent questions
- **Why Boot over Spring?** No XML, auto-config, embedded server, starters, actuator.
- **Change embedded server?** exclude `spring-boot-starter-tomcat`, add `-jetty`.
- **How to run code at startup?** `CommandLineRunner`.
- **Bean not found error?** Class is outside the component-scan base package.
