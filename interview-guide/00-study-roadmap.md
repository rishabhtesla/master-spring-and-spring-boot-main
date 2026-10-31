# 00 — Study Roadmap & How to Use This Guide

## 1. Who this is for

- **Freshers / 0–2 yrs:** focus on modules 01–05, 09, 10, plus the rapid-fire questions.
- **2–5 yrs:** everything, with deeper focus on 03 (transactions, N+1), 05 (API design), 07 (security), 08 (AOP/proxies).
- **5+ yrs:** be ready to discuss trade-offs: proxy internals, transaction propagation, security architecture,
  containerisation, observability, scaling, and design decisions (see "Advanced" Q&A sections).

## 2. Recommended learning order

| Phase | Guides | Goal | Suggested time |
|-------|--------|------|----------------|
| 1. Foundations | 01, 02 | Understand IoC/DI and how Boot wires everything | 2–3 days |
| 2. Data | 03 | Persist data with JDBC → JPA → Spring Data JPA | 2 days |
| 3. Web | 04, 05 | Build MVC + REST apps, handle errors, validation | 3–4 days |
| 4. Security | 07, then 06 | Basic auth, JWT, OAuth2, CORS, full-stack integration | 3 days |
| 5. Cross-cutting | 08, 09 | AOP, unit/integration testing | 2 days |
| 6. Build & Ship | 10, 11 | Maven/Gradle, Docker, AWS | 2 days |
| 7. Interview polish | 12, 13 | Rapid-fire revision, project pitch, mock interviews | 1–2 days |

### Fast tracks
- **7-day crash plan:** Day 1: 01 · Day 2: 02 · Day 3: 03 · Day 4: 05 · Day 5: 07 · Day 6: 08 + 09 · Day 7: 12 + 13.
- **1-day "interview tomorrow" plan:** Cheat-sheet sections of every guide (bottom of each file) → 12 → 13.

## 3. How to study each module (the loop)

```
 Read concept ─▶ Open the real code ─▶ Run it ─▶ Break it on purpose ─▶ Fix it
       ▲                                                                 │
       └──── Answer interview Q&A out loud ◀── Do hands-on exercise ◀────┘
```

1. **Read** the "What this module teaches" + "Core concepts" sections.
2. **Open the code** referenced in "Project walkthrough" side by side.
3. **Run it** (`./mvnw spring-boot:run` or `./gradlew bootRun`, or run the `main` class in the IDE).
4. **Break it deliberately**: remove an `@Component`, add a second bean of the same type,
   remove `@Transactional`, call an endpoint without credentials. Read the exception — interviewers love
   "what happens if…" questions, and you will have *seen* the answer.
5. **Do the exercises**, then answer the Q&A **without looking** at the answers.

## 4. How to run modules

| Module type | Command (from module folder) |
|-------------|------------------------------|
| Maven (`pom.xml`) | `./mvnw spring-boot:run` (or `mvn spring-boot:run`) |
| Gradle (`build.gradle`) | `./gradlew bootRun` |
| Plain Java (`51-junit`) | Import into IDE, run tests with JUnit 5 |
| React (`13-full-stack/...frontend...`) | `npm install && npm start` |
| H2 console | `http://localhost:8080/h2-console` (JDBC URL from `application.properties`) |
| MySQL via Docker | `docker run --detach --env MYSQL_ROOT_PASSWORD=dummypassword --env MYSQL_USER=todos-user --env MYSQL_PASSWORD=dummytodos --env MYSQL_DATABASE=todos --name mysql --publish 3306:3306 mysql:8-oracle` |

> Many modules contain `StepXX.md` / `StepXX.zip` files: these are snapshots of the code at each course step.
> Use them to see **how the code evolved** — an excellent way to understand *why* each piece exists.

## 5. Interview strategy

### How to answer any technical question (the 4-part formula)
1. **Definition** — one crisp sentence. *"Dependency Injection is when the container supplies an object's dependencies instead of the object creating them."*
2. **Why it matters** — the problem it solves. *"It gives loose coupling, so implementations can be swapped and mocked in tests."*
3. **Example** — preferably from **this project**. *"In my gaming example, `GameRunner` takes a `GamingConsole` in its constructor; Spring injects `PacmanGame` or `MarioGame`."*
4. **Trade-off / gotcha** — shows depth. *"I prefer constructor injection because dependencies are final and it surfaces circular dependencies early."*

### Top 10 topics interviewers ask most for Spring Boot roles
1. IoC & DI, bean scopes, bean lifecycle
2. How Spring Boot auto-configuration works
3. `@Component` vs `@Bean`, `@Controller` vs `@RestController`
4. `@Transactional` — propagation, rollback rules, proxy self-invocation pitfall
5. JPA relationships, lazy loading, N+1 problem
6. REST design: status codes, idempotency, exception handling with `@ControllerAdvice`
7. Spring Security filter chain, JWT flow, CSRF
8. AOP & proxies (JDK vs CGLIB)
9. Testing: `@SpringBootTest` vs `@WebMvcTest` vs `@DataJpaTest`, mocks
10. Profiles, externalised config, Actuator, Docker deployment

### Red flags to avoid
- Saying "Spring Boot is a different framework from Spring" (it is built **on** Spring).
- Using field injection in examples without acknowledging its downsides.
- Not knowing what HTTP status code your API returns for create (201) / delete (204) / not found (404).
- Disabling CSRF without being able to explain *why* it's safe for a stateless JWT API.

## 6. Progress checklist

- [ ] 01 Spring Core
- [ ] 02 Spring Boot
- [ ] 03 JPA & Hibernate
- [ ] 04 Web MVC
- [ ] 05 REST API
- [ ] 06 Full Stack + JWT
- [ ] 07 Security / OAuth
- [ ] 08 AOP
- [ ] 09 Testing
- [ ] 10 Maven / Gradle
- [ ] 11 Docker / AWS
- [ ] 12 Rapid-fire revision done twice
- [ ] 13 Project pitch rehearsed out loud (2-minute and 5-minute versions)
