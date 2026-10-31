# 🎯 Spring & Spring Boot — Learning + Interview Guide

This folder turns the course code in this repo into a **complete study path** to help you understand
every module and **prepare for Java / Spring Boot interviews**.

> **Stack used across the repo:** Java 21 · Spring Framework 6 · Spring Boot 3.3.4 · Spring Data JPA / Hibernate 6 ·
> Spring Security 6 · JUnit 5 · Mockito · Maven & Gradle · React · Docker · AWS

---

## 📚 Table of Contents

| # | Guide | Repo module(s) | What you'll master |
|---|-------|----------------|--------------------|
| 00 | [Study Roadmap & How to Use](00-study-roadmap.md) | — | Learning order, time plan, interview strategy |
| 01 | [Spring Core (IoC, DI, Beans)](01-spring-core.md) | `01-spring/` | Coupling, IoC container, DI, scopes, lifecycle |
| 02 | [Spring Boot Fundamentals](02-spring-boot.md) | `02-springboot/` | Starters, auto-config, profiles, actuator |
| 03 | [JPA, Hibernate & Spring Data](03-jpa-hibernate.md) | `03-jpa-getting-started/` | JDBC → JPA → Spring Data, transactions, N+1 |
| 04 | [Spring MVC Web App (Todo)](04-web-mvc-todo-app.md) | `11-web-application/` | DispatcherServlet, JSP, validation, form login |
| 05 | [REST API Design](05-rest-api.md) | `12-rest-api/` | REST, exception handling, versioning, HATEOAS |
| 06 | [Full Stack: React + Spring + JWT](06-full-stack-react-jwt.md) | `13-full-stack/` | CORS, JWT, React integration |
| 07 | [Spring Security, JWT & OAuth2](07-spring-security-jwt-oauth.md) | `71-spring-security/`, `72-oauth/` | Filter chain, CSRF, BCrypt, JWT, OAuth2 |
| 08 | [Spring AOP](08-spring-aop.md) | `73-spring-aop/` | Aspects, advices, pointcuts, proxies |
| 09 | [Testing: JUnit 5 & Mockito](09-testing-junit-mockito.md) | `51-junit/`, `52-mockito/` | Unit tests, mocks, Spring Boot test slices |
| 10 | [Build Tools: Maven & Gradle](10-build-tools-maven-gradle.md) | `81-maven/`, `82-gradle/` | POM, lifecycle, scopes, Gradle configs |
| 11 | [Docker & AWS Deployment](11-docker-aws-deployment.md) | `83-docker/`, `91-aws/` | Dockerfiles, Beanstalk, RDS, S3 |
| 12 | [Rapid-Fire Interview Questions (150+)](12-rapid-fire-interview-questions.md) | all | Last-minute revision |
| 13 | [Explaining This Project in an Interview](13-explain-your-project.md) | all | Pitch, architecture, STAR stories |

---

## 🗺️ How the modules connect

```
                 ┌───────────────────────────────────────────────┐
                 │               01  Spring Core                 │
                 │  IoC · DI · Beans · Scopes · Lifecycle        │
                 └───────────────────────┬───────────────────────┘
                                         │ Spring Boot automates the setup
                 ┌───────────────────────▼───────────────────────┐
                 │               02  Spring Boot                 │
                 │  Starters · Auto-config · Profiles · Actuator │
                 └──────┬───────────────┬────────────────┬───────┘
                        │               │                │
          ┌─────────────▼───┐   ┌───────▼────────┐   ┌───▼──────────────┐
          │ 03 JPA/Hibernate│   │ 04 Web MVC App │   │ 05 REST API      │
          │ data access     │──▶│ JSP + Todo     │   │ JSON + HATEOAS   │
          └─────────────────┘   └────────────────┘   └───┬──────────────┘
                                                         │
                                     ┌───────────────────▼───────────────┐
                                     │ 06 Full Stack (React + REST + JWT)│
                                     └───────────────────┬───────────────┘
   Cross-cutting topics                                  │ ship it
   ┌──────────────┐ ┌──────────┐ ┌───────────┐ ┌────────▼─────────┐
   │07 Security   │ │08 AOP    │ │09 Testing │ │11 Docker + AWS   │
   └──────────────┘ └──────────┘ └───────────┘ └──────────────────┘
                     ┌──────────────────────────┐
                     │10 Maven / Gradle (build) │
                     └──────────────────────────┘
```

## 🚀 Quick Start

1. Read [00-study-roadmap.md](00-study-roadmap.md) and pick a time plan.
2. For each guide: read the **Project walkthrough**, run the module code, then do the **Hands-on exercises**.
3. Answer the **Interview Q&A** aloud before reading the model answers.
4. The day before an interview, revise with [12-rapid-fire-interview-questions.md](12-rapid-fire-interview-questions.md)
   and practise your pitch from [13-explain-your-project.md](13-explain-your-project.md).

## 🔧 Prerequisites

- Java 21+ (Spring Boot 3 requires **Java 17+**)
- Maven 3.9+ (or use the bundled `./mvnw`) and/or Gradle (bundled `./gradlew`)
- IDE: IntelliJ IDEA / Eclipse / VS Code
- Docker Desktop (for MySQL and the Docker modules)
- Node.js 18+ (for the React frontend in `13-full-stack`)
