# 12 — Rapid-Fire Interview Questions (150+)

Short, crisp answers for **last-minute revision**. For deeper explanations and code, see the linked topic guides.
Tip: cover the answer column and answer aloud first.

---

## A. Spring Core ([details](01-spring-core.md))

| # | Question | Answer |
|---|----------|--------|
| 1 | What is IoC? | Inversion of Control: the framework (container) creates and wires objects instead of your code doing `new`. |
| 2 | What is DI? | A way to implement IoC — dependencies are supplied (injected) to an object by the container. |
| 3 | Types of DI? | Constructor, setter, field. Constructor is recommended (immutability, mandatory deps, testable). |
| 4 | Why is field injection discouraged? | Hides dependencies, can't make fields `final`, hard to unit-test without reflection/Spring. |
| 5 | BeanFactory vs ApplicationContext? | ApplicationContext extends BeanFactory, adds events, i18n, AOP integration, eager singleton init. Always use ApplicationContext. |
| 6 | What is a Spring bean? | An object created, configured and managed by the Spring IoC container. |
| 7 | `@Component` vs `@Bean`? | `@Component` on a class (auto-detected by component scan). `@Bean` on a method inside `@Configuration` — for third-party classes or custom creation logic. |
| 8 | Stereotype annotations? | `@Component`, `@Service`, `@Repository` (adds exception translation), `@Controller`. |
| 9 | Default bean scope? | Singleton (one instance per container). |
| 10 | All scopes? | singleton, prototype, request, session, application, websocket. |
| 11 | Spring singleton vs GoF singleton? | Spring: one per container (per bean id). GoF: one per JVM/classloader. |
| 12 | Prototype bean injected into singleton? | Injected once → behaves like singleton. Fix: `ObjectProvider`, `@Lookup`, or scoped proxy. |
| 13 | Two beans of same type, no qualifier? | `NoUniqueBeanDefinitionException`. Fix with `@Primary` or `@Qualifier`. |
| 14 | `@Primary` vs `@Qualifier`? | `@Primary` = default choice. `@Qualifier` = injection point asks for a specific bean; it wins over `@Primary`. |
| 15 | No bean found? | `NoSuchBeanDefinitionException` (or `UnsatisfiedDependencyException` wrapping it). |
| 16 | Eager vs lazy init? | Singletons are eager by default (errors at startup). `@Lazy` creates on first use. |
| 17 | Bean lifecycle? | Instantiate → populate properties → Aware callbacks → BeanPostProcessor before → `@PostConstruct`/`afterPropertiesSet` → BPP after (proxies) → ready → `@PreDestroy`/`destroy`. |
| 18 | `@PostConstruct` use? | Init logic after dependencies are injected (e.g. warm cache, open connection). |
| 19 | Are `@PreDestroy` methods called for prototype beans? | No — container doesn't manage prototype destruction. |
| 20 | What is CDI? | Jakarta Contexts and Dependency Injection spec. `@Inject` ≈ `@Autowired`, `@Named` ≈ `@Component`. |
| 21 | Circular dependency? | A needs B, B needs A. Constructor injection fails fast (`BeanCurrentlyInCreationException`); Boot 2.6+ disallows cycles by default. Fix the design. |
| 22 | `@ComponentScan` default package? | Package of the annotated class and its sub-packages. |
| 23 | XML config still supported? | Yes (`ClassPathXmlApplicationContext`), but annotation/Java config is preferred. |
| 24 | What is a `BeanPostProcessor`? | Hook to modify beans before/after init — how proxies for AOP/`@Transactional` are created. |
| 25 | Is a singleton bean thread-safe? | Not automatically. Keep singletons stateless or synchronise shared state. |

## B. Spring Boot ([details](02-spring-boot.md))

| # | Question | Answer |
|---|----------|--------|
| 26 | What is Spring Boot? | Opinionated layer on Spring: auto-configuration, starters, embedded server, production features. |
| 27 | `@SpringBootApplication` = ? | `@SpringBootConfiguration` (`@Configuration`) + `@EnableAutoConfiguration` + `@ComponentScan`. |
| 28 | How does auto-config work? | Boot loads classes listed in `META-INF/spring/...AutoConfiguration.imports`; each is guarded by `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`, etc. |
| 29 | How to see which auto-configs applied? | `debug=true` (conditions evaluation report) or Actuator `/actuator/conditions`. |
| 30 | Exclude an auto-config? | `@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)` or `spring.autoconfigure.exclude`. |
| 31 | What is a starter? | A dependency descriptor bundling compatible libraries, e.g. `spring-boot-starter-web` = Spring MVC + Jackson + Tomcat. |
| 32 | Default embedded server? | Tomcat. Swap for Jetty/Undertow by excluding Tomcat and adding the other starter. |
| 33 | Change port? | `server.port=8081`. |
| 34 | Profiles? | Environment-specific config: `application-dev.properties`, activate with `spring.profiles.active=dev`. |
| 35 | `@Value` vs `@ConfigurationProperties`? | `@Value` for single values; `@ConfigurationProperties(prefix=...)` for type-safe grouped config with validation. |
| 36 | Config precedence (high → low, simplified)? | Command-line args → env vars → profile-specific properties → `application.properties` → defaults. |
| 37 | What is Actuator? | Production endpoints: `/actuator/health`, `/info`, `/metrics`, `/env`, `/beans`, `/mappings`. |
| 38 | Expose all actuator endpoints? | `management.endpoints.web.exposure.include=*` (not in prod!). |
| 39 | What is DevTools? | Auto-restart on classpath change, LiveReload; disabled in packaged jars. |
| 40 | `CommandLineRunner` vs `ApplicationRunner`? | Both run after startup; `ApplicationRunner` gets parsed `ApplicationArguments`. |
| 41 | What does a Boot fat JAR contain? | App classes + all dependency jars + embedded server + loader; run via `java -jar`. |
| 42 | Spring Boot 3 key changes? | Java 17 baseline, `javax.*` → `jakarta.*`, Spring 6, native image (GraalVM) support, Micrometer Observation, `ProblemDetail`. |
| 43 | Spring vs Spring MVC vs Spring Boot? | Spring = DI core + modules; Spring MVC = web framework module; Boot = makes building Spring apps fast. |
| 44 | Where do logs go / how to change level? | Console via Logback; `logging.level.com.example=debug`. |
| 45 | How to disable a bean in a profile? | `@Profile("!prod")` on the bean or `@ConditionalOnProperty`. |

## C. JPA, Hibernate, Spring Data ([details](03-jpa-hibernate.md))

| # | Question | Answer |
|---|----------|--------|
| 46 | JPA vs Hibernate? | JPA is a specification (interfaces); Hibernate is an implementation (provider). |
| 47 | Spring Data JPA? | Abstraction over JPA: repositories from interfaces, derived queries, paging — no boilerplate. |
| 48 | `JdbcTemplate` advantage over raw JDBC? | Handles connections, statements, exceptions (translated to `DataAccessException`). |
| 49 | `EntityManager`? | JPA API to persist/merge/find/remove entities; manages the persistence context. |
| 50 | Persistence context? | First-level cache of managed entities within a transaction; ensures identity and dirty checking. |
| 51 | Entity states? | Transient, Managed, Detached, Removed. |
| 52 | `persist` vs `merge`? | `persist` makes a new entity managed; `merge` copies a detached entity's state onto a managed copy and returns it. |
| 53 | `save()` in Spring Data does? | `persist` if new (id null / isNew), else `merge`. |
| 54 | `@GeneratedValue` strategies? | AUTO, IDENTITY, SEQUENCE, TABLE. |
| 55 | `CrudRepository` vs `JpaRepository`? | JpaRepository extends ListCrudRepository + ListPagingAndSortingRepository, adds `flush`, `saveAndFlush`, batch deletes. |
| 56 | Derived query example? | `List<Todo> findByUsername(String username)` → `where username = ?`. |
| 57 | `@Query` JPQL vs native? | JPQL queries entities (portable); `nativeQuery=true` uses raw SQL (DB-specific). |
| 58 | LAZY vs EAGER defaults? | `@OneToMany`/`@ManyToMany` LAZY; `@ManyToOne`/`@OneToOne` EAGER. |
| 59 | `LazyInitializationException`? | Accessing a lazy association after the session/transaction closed. Fix: fetch join, `@EntityGraph`, DTO projection, keep within transaction. |
| 60 | N+1 problem? | 1 query for parents + N for each child collection. Fix: `JOIN FETCH`, `@EntityGraph`, batch fetching. |
| 61 | `mappedBy`? | Marks the inverse (non-owning) side; the other side owns the foreign key. |
| 62 | `ddl-auto` values? | none, validate, update, create, create-drop. Use `validate`/`none` + Flyway in prod. |
| 63 | `@Transactional` default rollback? | Rolls back on unchecked exceptions (`RuntimeException`, `Error`), not checked ones (unless `rollbackFor`). |
| 64 | Propagation REQUIRED vs REQUIRES_NEW? | REQUIRED joins existing tx or creates one; REQUIRES_NEW suspends existing and starts a new one. |
| 65 | Why doesn't `@Transactional` work on a private/self-invoked method? | Proxy-based: calls inside the same class bypass the proxy; private methods can't be proxied. |
| 66 | Isolation levels? | READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE — prevent dirty / non-repeatable / phantom reads progressively. |
| 67 | Optimistic locking? | `@Version` column; update fails with `OptimisticLockException` if version changed. |
| 68 | First vs second-level cache? | 1st: per EntityManager, always on. 2nd: per SessionFactory, optional (Ehcache, etc.). |
| 69 | Pagination? | `repo.findAll(PageRequest.of(page, size, Sort.by("name")))` returns `Page<T>`. |
| 70 | `readOnly=true` benefit? | Hints to skip dirty checking/flush → performance. |

## D. Spring MVC & REST ([MVC](04-web-mvc-todo-app.md), [REST](05-rest-api.md))

| # | Question | Answer |
|---|----------|--------|
| 71 | DispatcherServlet? | Front controller: routes requests to handlers, resolves views, handles exceptions. |
| 72 | MVC request flow? | DispatcherServlet → HandlerMapping → HandlerAdapter → Controller → ModelAndView → ViewResolver → View. |
| 73 | `@Controller` vs `@RestController`? | `@RestController` = `@Controller` + `@ResponseBody` on every method (returns data, not views). |
| 74 | `@RequestParam` vs `@PathVariable`? | Query string `?name=x` vs URI segment `/users/{id}`. |
| 75 | `@RequestBody`? | Deserialises the HTTP body (JSON) into an object via `HttpMessageConverter`. |
| 76 | `ResponseEntity`? | Full control over status, headers, body. |
| 77 | Return 201 with Location? | `ResponseEntity.created(ServletUriComponentsBuilder.fromCurrentRequest().path("/{id}").buildAndExpand(id).toUri()).build()`. |
| 78 | Idempotent methods? | GET, PUT, DELETE, HEAD, OPTIONS. POST is not; PATCH not guaranteed. |
| 79 | Safe methods? | GET, HEAD, OPTIONS (no state change). |
| 80 | PUT vs PATCH? | PUT replaces the whole resource; PATCH partially updates. |
| 81 | 401 vs 403? | 401 = not authenticated; 403 = authenticated but not allowed. |
| 82 | Status for validation failure? | 400 Bad Request. |
| 83 | Global exception handling? | `@ControllerAdvice` / `@RestControllerAdvice` with `@ExceptionHandler` methods. |
| 84 | `@ResponseStatus` on an exception? | Maps that exception to an HTTP status automatically (e.g. `UserNotFoundException` → 404). |
| 85 | Bean validation trigger? | `@Valid` on `@RequestBody` param; failure → `MethodArgumentNotValidException`. |
| 86 | API versioning options? | URI (`/v1/person`), request param, custom header, media type (Accept). |
| 87 | Which versioning is best? | Trade-off: URI is simple & cache-friendly; headers/media types keep URIs clean but are harder to test/cache. Be consistent. |
| 88 | HATEOAS? | Responses include links to related actions (`EntityModel` + `WebMvcLinkBuilder`). Level 3 of Richardson maturity. |
| 89 | Content negotiation? | Client sends `Accept: application/xml`; add `jackson-dataformat-xml` to return XML. |
| 90 | Static vs dynamic filtering? | Static: `@JsonIgnore`/`@JsonIgnoreProperties`. Dynamic: `@JsonFilter` + `MappingJacksonValue`. |
| 91 | i18n in REST? | `MessageSource` + `LocaleContextHolder.getLocale()` from `Accept-Language`. |
| 92 | Swagger/OpenAPI in Boot 3? | `springdoc-openapi-starter-webmvc-ui` → `/swagger-ui.html`. |
| 93 | REST vs SOAP? | REST: architectural style, any format (JSON), HTTP verbs. SOAP: protocol, XML envelope, WSDL, WS-* standards. |
| 94 | Richardson maturity levels? | 0: single endpoint, 1: resources, 2: HTTP verbs + status codes, 3: HATEOAS. |
| 95 | Why use DTOs? | Decouple API contract from entities, hide fields, avoid lazy-loading/serialisation issues. |
| 96 | `@SessionAttributes`? | Stores model attributes in the HTTP session across requests (used for `name` in the Todo app). |
| 97 | `redirect:` prefix? | Tells view resolver to send HTTP redirect (PRG pattern after form POST). |
| 98 | `BindingResult`? | Holds validation errors; must directly follow the `@Valid` parameter. |
| 99 | RFC 7807 in Boot 3? | `ProblemDetail` standard error body; enable `spring.mvc.problemdetails.enabled=true`. |
| 100 | How to handle CORS? | `@CrossOrigin` or global `WebMvcConfigurer.addCorsMappings`; with Security, also `http.cors()`. |

## E. Spring Security, JWT, OAuth ([details](07-spring-security-jwt-oauth.md))

| # | Question | Answer |
|---|----------|--------|
| 101 | Authentication vs authorization? | Who you are vs what you can do. |
| 102 | How does Spring Security plug in? | Servlet filter: `DelegatingFilterProxy` → `FilterChainProxy` → one or more `SecurityFilterChain`s. |
| 103 | Default behaviour after adding the starter? | All endpoints secured, form login + basic auth, generated password in logs, CSRF on. |
| 104 | Configure security in Boot 3? | Declare a `SecurityFilterChain` `@Bean` with the lambda DSL; `WebSecurityConfigurerAdapter` is removed. |
| 105 | Where is the logged-in user stored? | `SecurityContextHolder.getContext().getAuthentication()` (ThreadLocal by default). |
| 106 | `UserDetailsService`? | Loads a user by username; implementations: `InMemoryUserDetailsManager`, `JdbcUserDetailsManager`, custom. |
| 107 | Why BCrypt? | Slow, salted, adaptive hash — resists brute force/rainbow tables. Never store plain or fast hashes (MD5/SHA). |
| 108 | Encoding vs hashing vs encryption? | Encoding: reversible, no key (Base64). Hashing: one-way. Encryption: reversible with key. |
| 109 | What is CSRF? | Attacker tricks a logged-in browser into sending a state-changing request using its cookies. |
| 110 | When can CSRF be disabled? | Stateless APIs that don't authenticate via cookies (e.g. bearer tokens). |
| 111 | JWT structure? | `header.payload.signature`, Base64URL-encoded; signature makes it tamper-evident (not encrypted!). |
| 112 | Symmetric vs asymmetric JWT signing? | HS256 shared secret vs RS256 private key signs / public key verifies. This repo uses RSA. |
| 113 | JWT drawbacks? | Hard to revoke, payload readable, size. Use short expiry + refresh tokens. |
| 114 | Stateless session config? | `sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))`. |
| 115 | OAuth2 roles? | Resource owner, client, authorization server, resource server. |
| 116 | Common OAuth2 grant flows? | Authorization Code (+PKCE) for users/SPAs, Client Credentials for service-to-service. (Implicit & password are deprecated.) |
| 117 | OAuth2 vs OpenID Connect? | OAuth2 = authorization (access tokens). OIDC = authentication layer on top (ID token, user info). |
| 118 | `oauth2Login` vs `oauth2ResourceServer`? | Login: app redirects user to provider (e.g. Google). Resource server: API validates incoming bearer tokens. |
| 119 | Method security? | `@EnableMethodSecurity` + `@PreAuthorize("hasRole('USER') and #username == authentication.name")`, `@PostAuthorize`, `@Secured`, `@RolesAllowed`. |
| 120 | Role vs authority? | Role is an authority with `ROLE_` prefix; `hasRole('ADMIN')` checks `ROLE_ADMIN`. |
| 121 | H2 console blank with Security? | Disable CSRF for it and allow frames: `headers(h -> h.frameOptions(f -> f.sameOrigin()))`. |
| 122 | `permitAll()` vs `web.ignoring()`? | `permitAll` still runs the filter chain; ignoring bypasses security entirely (avoid for dynamic endpoints). |

## F. AOP ([details](08-spring-aop.md))

| # | Question | Answer |
|---|----------|--------|
| 123 | What is AOP? | Modularising cross-cutting concerns (logging, security, tx) into aspects. |
| 124 | Aspect / Advice / Pointcut / JoinPoint? | Aspect = class with concern; Advice = what & when; Pointcut = where (expression); JoinPoint = specific method execution. |
| 125 | Advice types? | `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`, `@Around`. |
| 126 | Most powerful advice? | `@Around` — controls if/when target runs via `ProceedingJoinPoint.proceed()`. |
| 127 | Pointcut example? | `execution(* com.in28minutes.learnspringaop.aopexample.*.*.*(..))`. |
| 128 | Pointcut by annotation? | `@annotation(com.in28minutes...annotations.TrackTime)`. |
| 129 | Weaving? | Linking aspects to targets: compile-time, load-time (AspectJ), runtime proxies (Spring AOP). |
| 130 | Spring AOP vs AspectJ? | Spring AOP: proxy-based, method execution on beans only, simpler. AspectJ: full weaving, fields/constructors, faster. |
| 131 | JDK proxy vs CGLIB? | JDK proxies need interfaces; CGLIB subclasses the class. Boot defaults to CGLIB (`proxyTargetClass=true`). |
| 132 | AOP limitations? | No self-invocation, no private/final methods, only Spring-managed beans. |
| 133 | Order multiple aspects? | `@Order(n)` on the aspect (lower = higher precedence). |

## G. Testing ([details](09-testing-junit-mockito.md))

| # | Question | Answer |
|---|----------|--------|
| 134 | JUnit 5 modules? | Platform (launcher), Jupiter (new API), Vintage (JUnit 3/4). |
| 135 | `@BeforeEach` vs `@BeforeAll`? | Before every test vs once per class (static by default). |
| 136 | Test an exception? | `assertThrows(MyException.class, () -> code())`. |
| 137 | Stub vs mock? | Stub returns canned data; mock also verifies interactions. |
| 138 | Mock vs spy? | Mock: all methods fake. Spy: wraps real object, real methods unless stubbed. |
| 139 | `@Mock` / `@InjectMocks`? | Create mocks; create the class under test with mocks injected (needs `@ExtendWith(MockitoExtension.class)`). |
| 140 | Verify a call? | `verify(mock, times(1)).method(arg)`. |
| 141 | `@SpringBootTest`? | Loads the full context — integration tests. |
| 142 | `@WebMvcTest`? | Loads only MVC layer; use `MockMvc`; mock services with `@MockBean` (Boot 3.4+: `@MockitoBean`). |
| 143 | `@DataJpaTest`? | JPA slice with embedded DB, each test transactional & rolled back. |
| 144 | Test pyramid? | Many unit tests, fewer integration tests, few e2e tests. |
| 145 | What is TDD? | Red (failing test) → Green (minimal code) → Refactor. |

## H. Build, Docker, Cloud ([build](10-build-tools-maven-gradle.md), [deploy](11-docker-aws-deployment.md))

| # | Question | Answer |
|---|----------|--------|
| 146 | Maven lifecycle phases? | validate → compile → test → package → verify → install → deploy. |
| 147 | `mvn install` vs `mvn package`? | package builds the jar in `target/`; install also copies it to the local `~/.m2` repo. |
| 148 | Dependency scopes? | compile, provided, runtime, test, system, import. |
| 149 | Dependency conflict resolution in Maven? | Nearest definition wins; then first declared. Use `dependencyManagement` / exclusions. |
| 150 | `spring-boot-starter-parent` role? | Inherits managed dependency versions (BOM), plugin config, Java version, resource filtering. |
| 151 | Gradle `implementation` vs `api`? | `implementation` hides dep from consumers' compile classpath (faster builds); `api` exposes it. |
| 152 | Maven vs Gradle? | Maven: XML, convention, predictable. Gradle: Groovy/Kotlin DSL, incremental builds, build cache, faster. |
| 153 | Container vs VM? | Containers share host kernel (lightweight, fast start); VMs virtualise hardware with a full OS each. |
| 154 | Image vs container? | Image = read-only template; container = running instance of an image. |
| 155 | `CMD` vs `ENTRYPOINT`? | ENTRYPOINT = the fixed executable; CMD = default args (overridable). |
| 156 | Multi-stage build? | Build in a JDK/Maven stage, copy only the jar into a slim JRE stage → smaller, safer image. |
| 157 | Build image without Dockerfile? | `./mvnw spring-boot:build-image` (Cloud Native Buildpacks). |
| 158 | Elastic Beanstalk? | AWS PaaS: upload jar/Docker, it provisions EC2, load balancer, auto-scaling, monitoring. |
| 159 | How does the app get RDS credentials? | Environment variables (`RDS_HOSTNAME`, `RDS_USERNAME`, …) referenced in `application.properties`. |
| 160 | Host a React app on AWS? | Build static files (`npm run build`) → S3 static website (optionally CloudFront CDN). |

---

### 🧠 Last-minute "must not forget" list
- Constructor injection · Singleton default · `@Primary`/`@Qualifier`
- `@SpringBootApplication` = 3 annotations · auto-config uses `@Conditional`
- `@Transactional` = proxy → no self-invocation, rollback on unchecked only
- N+1 → `JOIN FETCH` / `@EntityGraph`
- 201 + Location for create, 204 for delete, 404 not found, 400 validation
- `@RestControllerAdvice` for global errors
- Filter chain → `SecurityFilterChain` bean; BCrypt; CSRF off only for stateless token APIs
- JWT = header.payload.signature (signed, not encrypted)
- AOP: `@Around` + `ProceedingJoinPoint.proceed()`; JDK vs CGLIB proxies
- `@WebMvcTest` vs `@DataJpaTest` vs `@SpringBootTest`
