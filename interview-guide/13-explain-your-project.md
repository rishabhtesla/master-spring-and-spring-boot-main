# 13 — Explaining This Project in an Interview

Almost every interview starts with **"Tell me about yourself / your project."** This file gives you a ready,
honest way to present the work in this repository, plus deep-dive answers for the follow-up questions.

> ⚠️ **Be honest:** this repository is based on the in28minutes *Master Spring and Spring Boot* course.
> Present it as a **hands-on learning project you built and extended**, not as production work.
> Interviewers respect "I built this to learn X, and here is what I learned" much more than an inflated claim.
> Make it truly yours by doing the **"Make it your own" upgrades** at the end of this file.

---

## 1. The 30-second elevator pitch

> "I built a set of Spring Boot 3 applications on Java 21, from Spring Core basics up to a full-stack app.
> The main project is a **Todo management application**: a **React** frontend talking to a **Spring Boot REST API**,
> secured with **Spring Security and JWT**, and persisting data with **Spring Data JPA / Hibernate** on H2 and MySQL.
> I also built a social-media-style REST API with proper exception handling, validation, versioning, HATEOAS and
> Swagger docs, tested the code with **JUnit 5 and Mockito**, containerised it with **Docker**, and deployed it to
> **AWS Elastic Beanstalk with RDS MySQL**, with the React frontend hosted on S3."

## 2. The 2-minute version (structure: Problem → Architecture → Your role → Learnings)

1. **What & why:** "I wanted to understand Spring end-to-end, not just annotations, so I built progressively:
   a plain Java app showing tight coupling → the same app with Spring IoC → Spring Boot → data → web → REST → security → cloud."
2. **Architecture:** see the diagram below.
3. **Key features I implemented:**
   - CRUD REST endpoints for users/posts and todos, with correct status codes (`201 Created` + `Location` header, `404`, `204`).
   - Global exception handling with `@ControllerAdvice` returning a consistent `ErrorDetails` JSON.
   - Bean Validation (`@Valid`, `@Size`, `@Past`) on request bodies.
   - JPA entities with a `@OneToMany` / `@ManyToOne` relationship (User → Posts).
   - Stateless security with JWT signed using an **RSA key pair** (`NimbusJwtEncoder`) and an OAuth2 resource server.
   - CORS configuration so the React app on port 3000 can call the API on port 8080.
   - AOP aspects for logging and execution-time tracking via a custom `@TrackTime` annotation.
   - Profiles + `@ConfigurationProperties` for environment-specific configuration, and Actuator for health/metrics.
   - Dockerised the app, and deployed to AWS Beanstalk with RDS (configured via environment variables such as `RDS_HOSTNAME`).
4. **What I learned:** proxies (why `@Transactional`/AOP don't work on self-invocation), the N+1 problem with lazy loading,
   why CSRF can be disabled for stateless JWT APIs, and how auto-configuration decides which beans to create.

## 3. Architecture diagram (draw this on the whiteboard)

```
 ┌────────────────────┐      HTTPS + JSON       ┌──────────────────────────────────────────┐
 │   React SPA        │  Authorization: Bearer  │        Spring Boot 3 REST API             │
 │  (13-full-stack/   │ ──────────────────────▶ │                                          │
 │   01-frontend)     │                         │  Security Filter Chain (JWT / Basic)     │
 │  - React Router    │ ◀────────────────────── │            │                             │
 │  - Context (auth)  │      JSON responses     │  @RestController (TodoJpaResource,       │
 │  - axios client    │                         │      UserJpaResource)                    │
 └────────────────────┘                         │            │                             │
        hosted on                               │  Service layer / @Transactional          │
        AWS S3 (static)                         │            │                             │
                                                │  Spring Data JPA Repositories            │
                                                │            │  Hibernate (ORM)            │
                                                └────────────┼─────────────────────────────┘
                                                             │ JDBC
                                                   ┌─────────▼─────────┐
                                                   │ H2 (dev) / MySQL  │
                                                   │ (Docker / AWS RDS)│
                                                   └───────────────────┘
 Cross-cutting: AOP logging/timing · Actuator /health · Swagger UI (springdoc) · Profiles
 Delivery: Maven/Gradle build → Docker image → AWS Elastic Beanstalk
```

### Request lifecycle you should be able to narrate
1. React calls `GET /users/in28minutes/todos` with `Authorization: Bearer <jwt>`.
2. Request passes the servlet filter chain → `DelegatingFilterProxy` → `FilterChainProxy` → `BearerTokenAuthenticationFilter`
   validates the JWT signature with the RSA public key and puts an `Authentication` in the `SecurityContextHolder`.
3. `DispatcherServlet` finds the handler method via `HandlerMapping` (`@GetMapping("/users/{username}/todos")`).
4. Controller calls the repository (`todoRepository.findByUsername(username)` — a **derived query method**).
5. Spring Data creates the JPQL, Hibernate generates SQL, executes it over a pooled connection (HikariCP).
6. Entities are serialised to JSON by **Jackson** (`HttpMessageConverter`), returned with `200 OK`.
7. If something throws `UserNotFoundException`, `@ControllerAdvice` converts it into a `404` JSON error body.

## 4. Module map — "what did each part teach you?"

| Module | One-line answer |
|--------|-----------------|
| `01-spring` | IoC/DI from first principles: gaming console example (tight → loose coupling), scopes, lifecycle, CDI, XML config |
| `02-springboot` | Starters, auto-configuration, profiles, `@ConfigurationProperties`, Actuator, DevTools |
| `03-jpa-getting-started` | Same CRUD implemented 3 ways: Spring JDBC (`JdbcTemplate`), JPA (`EntityManager`), Spring Data JPA |
| `11-web-application` | Server-side MVC Todo app with JSP, validation, sessions, form login, H2 → MySQL |
| `12-rest-api` | Production-style REST API: exceptions, validation, versioning, HATEOAS, filtering, Swagger, JPA relationships |
| `13-full-stack` | React frontend + REST backend, CORS, Basic auth then JWT |
| `51-junit`, `52-mockito` | Unit testing, stubs vs mocks, `@Mock`/`@InjectMocks` |
| `71-spring-security`, `72-oauth` | Filter chain, CSRF, BCrypt, JDBC users, JWT with RSA, method security, Google OAuth2 login |
| `73-spring-aop` | Logging & performance aspects, pointcuts, custom annotation |
| `81-maven`, `82-gradle` | Build lifecycle, dependency management |
| `83-docker`, `91-aws` | Dockerfiles, multi-stage builds, Beanstalk, RDS, S3 |

## 5. STAR stories (Situation · Task · Action · Result)

Prepare 3–4 of these. Use real issues you hit while running the code.

### Story 1 — Lazy loading / JSON serialisation
- **S:** Fetching a `User` with its `posts` caused infinite recursion / huge JSON (User → Post → User …).
- **T:** Return clean JSON without breaking the JPA relationship.
- **A:** Marked `Post.user` with `@JsonIgnore` and made the relationship `FetchType.LAZY`; exposed posts via a dedicated
  endpoint `/jpa/users/{id}/posts`.
- **R:** Smaller payloads, no recursion, and fewer unnecessary queries. Learned about the N+1 problem and `JOIN FETCH`.

### Story 2 — CORS between React and Spring
- **S:** React on `localhost:3000` got "blocked by CORS policy" calling `localhost:8080`.
- **T:** Allow the frontend without opening the API to everyone.
- **A:** Added a `WebMvcConfigurer` bean with `addCorsMappings` allowing the specific origin and methods; ensured
  pre-flight `OPTIONS` requests are permitted in the security filter chain.
- **R:** Frontend worked; I can now explain same-origin policy and pre-flight requests.

### Story 3 — Moving from session-based to stateless JWT security
- **S:** Basic auth sent credentials on every call and wasn't suitable for a SPA.
- **T:** Implement token-based auth.
- **A:** Generated an RSA key pair, configured `JwtEncoder`/`JwtDecoder` (Nimbus), an `/authenticate` endpoint that issues
  a token with claims (`sub`, `scope`, `exp`), `SessionCreationPolicy.STATELESS`, and `oauth2ResourceServer().jwt()`.
- **R:** Stateless, horizontally scalable API; learned JWT structure, signing vs encryption, and token-expiry trade-offs.

### Story 4 — Environment-specific config for cloud deployment
- **S:** App had hard-coded H2/MySQL URLs; on AWS the DB host differs.
- **A:** Used placeholders with defaults: `jdbc:mysql://${RDS_HOSTNAME:localhost}:${RDS_PORT:3306}/${RDS_DB_NAME:social-media-database}`,
  plus Spring profiles.
- **R:** Same artifact runs locally and on Beanstalk (12-factor "config in the environment").

## 6. Likely follow-up questions about the project (with answers)

1. **Why Spring Boot over plain Spring?** Auto-configuration, starters, embedded Tomcat, production-ready Actuator; less boilerplate, faster setup.
2. **Why JWT instead of sessions?** Stateless → easier horizontal scaling, works well with SPAs/mobile; trade-off: revocation is hard (use short expiry + refresh tokens / deny-list).
3. **How do you secure the JWT signing key?** In this project the RSA key is generated at startup (demo). In production: load from a keystore/secret manager (AWS Secrets Manager, Vault), rotate keys, expose public keys via JWKS.
4. **Why disable CSRF?** CSRF exploits browsers auto-sending cookies. A stateless API that reads a bearer token from a header isn't vulnerable in the same way. If auth used cookies, keep CSRF on.
5. **How would you scale this?** Stateless app instances behind a load balancer (Beanstalk auto-scaling), RDS read replicas, caching (`@Cacheable` + Redis), pagination on list endpoints, connection-pool tuning.
6. **How do you handle errors consistently?** `@ControllerAdvice` extending `ResponseEntityExceptionHandler`, a standard `ErrorDetails` (timestamp, message, details); validation errors → 400. Boot 3 also supports RFC 7807 `ProblemDetail`.
7. **How did you test it?** Unit tests with JUnit 5 + Mockito for business logic; could add `@WebMvcTest` with `MockMvc` for controllers and `@DataJpaTest` for repositories; `@SpringBootTest` for integration.
8. **What would you improve?** DTOs instead of exposing entities, MapStruct mapping, pagination, refresh tokens, Flyway migrations instead of `ddl-auto=update`, Testcontainers, CI/CD with GitHub Actions, observability (Micrometer + Prometheus/Grafana), rate limiting.
9. **Why H2 and MySQL?** H2 in-memory for fast local dev/tests; MySQL for realistic persistence. Switching is mostly configuration thanks to JPA.
10. **What's the hardest bug you hit?** Pick one of the STAR stories above, and tell it with concrete details.

## 7. "Make it your own" upgrades (do at least 2 before interviews)

| Upgrade | Skills it proves |
|---------|------------------|
| Introduce DTOs + MapStruct and stop returning entities | API design, layering |
| Add Flyway/Liquibase migrations | DB versioning in teams |
| Add `@WebMvcTest`, `@DataJpaTest` and Testcontainers MySQL tests | Testing maturity |
| Add pagination & sorting (`Pageable`) to list endpoints | Performance awareness |
| Add refresh tokens + role-based access (`@PreAuthorize("hasRole('ADMIN')")`) | Security depth |
| Replace custom `ErrorDetails` with `ProblemDetail` (RFC 7807) | Modern Boot 3 features |
| Add `docker-compose.yml` (app + MySQL) | DevOps |
| GitHub Actions pipeline: build → test → Docker image | CI/CD |
| Add Redis caching with `@Cacheable` | Caching & AOP |
| Micrometer metrics + Prometheus/Grafana dashboard | Observability |

Once done, update your pitch: *"…and I extended it with Flyway migrations, Testcontainers-based integration tests and a CI pipeline."*

## 8. Resume bullet examples

- Built a full-stack Todo application using **React, Spring Boot 3 (Java 21), Spring Data JPA and MySQL**, secured with **Spring Security + JWT (RSA-signed)**.
- Designed RESTful APIs with **global exception handling, Bean Validation, API versioning, HATEOAS and OpenAPI/Swagger** documentation.
- Implemented cross-cutting **logging and performance monitoring using Spring AOP** and a custom `@TrackTime` annotation.
- Wrote unit tests with **JUnit 5 and Mockito**; containerised services with **Docker** and deployed to **AWS Elastic Beanstalk + RDS**, frontend on **S3**.
