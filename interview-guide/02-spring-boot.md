# Spring Boot: Explain the Defaults, Then Prove Them in Code
## What this module teaches
This guide follows `02-springboot/` in the in28minutes **Master Spring and Spring Boot** repository.
It connects a small HTTP API to Boot's dependency management, auto-configuration, profiles, configuration binding, embedded server, developer tooling, and operational endpoints.
The current executable source uses **Spring Boot 3.3.4 and Java 21**, with Spring Framework 6.1.

**Repository root:** `/Users/rpathak/Documents/Learning/master-spring-and-spring-boot-main`.
**Exact module path:** `02-springboot/`.
**Main class:** `com.in28minutes.springboot.learnspringboot.LearnSpringBootApplication`.

Read the actual source first, then use these teaching references:

- `02-springboot/readme.md`: the fourteen-step course sequence and code snapshot.
- `02-springboot/notes.txt`: development/production logging levels.
- `02-springboot/final.md`: the final teaching snapshot.
- `spring-boot-3-1-updates.md`: Spring Security DSL and MySQL coordinate updates.
- `spring-boot-3.2.x-changes.md`: Jakarta JSTL dependency and taglib updates.

The Markdown snapshots include historical local paths and additional repository configuration.
The current `02-springboot/pom.xml` and `src/main` files determine the application's behavior. This module does not use Spring Security, MySQL, or JSP/JSTL dependencies itself.
The root update notes explain changes relevant to other course modules, not active features here.

### How to run it
From the repository root, with Maven available and JDK 21 selected:
```bash
mvn -f 02-springboot/pom.xml spring-boot:run
```
Equivalent from inside the module:
```bash
cd 02-springboot
mvn spring-boot:run
```
Or run `LearnSpringBootApplication.main()` in your IDE. The class has `@SpringBootApplication` and calls `SpringApplication.run`.
The default embedded HTTP server port is `8080` unless overridden. The checked-in base configuration activates the `dev` profile.

Try these endpoints while the application is running:
```bash
curl http://localhost:8080/courses
curl http://localhost:8080/currency-configuration
curl http://localhost:8080/actuator/health
curl http://localhost:8080/actuator/metrics
```
The currency endpoint deliberately exposes sample configuration, including its sample key. Do not copy that endpoint design into production with real credentials.
The values checked into this lesson are placeholders, not a secret-management pattern.

### Run a different profile without editing the source
```bash
mvn -f 02-springboot/pom.xml spring-boot:run \
  -Dspring-boot.run.profiles=prod
```
Or pass application arguments through the Maven plugin:
```bash
mvn -f 02-springboot/pom.xml spring-boot:run \
  -Dspring-boot.run.arguments="--spring.profiles.active=prod --server.port=8081"
```
`-Dspring-boot.run.arguments` configures the Maven plugin, which forwards application arguments.
It is not the same thing as expecting every Maven `-D` property to appear in the forked JVM. A direct executable-JAR invocation accepts Boot arguments after the JAR path.

### Build and run an executable JAR
```bash
mvn -f 02-springboot/pom.xml clean package
java -jar 02-springboot/target/learn-spring-boot-0.0.1-SNAPSHOT.jar \
  --spring.profiles.active=prod
```
The package command normally runs the test suite before packaging. A JDK 17 can run Boot 3 generally, but Java 21-targeted application bytecode needs Java 21 or newer.
The build's Java target and the framework's minimum Java baseline are different questions.

### Reading conventions
**Repo excerpts** below copy actual source with surrounding code omitted.
**Illustrations** and proposed improvements are not claims about existing classes.
All paths are repository-relative. Default behavior refers to Boot 3.3 unless this guide explicitly describes older versions.

## Project walkthrough
### 1. `pom.xml`: a small dependency declaration with a large effect
File: `02-springboot/pom.xml`.

Repo excerpt:
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.4</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>
```
Repo excerpt:
```xml
<properties>
    <java.version>21</java.version>
</properties>
```
The parent supplies curated dependency versions and useful Maven defaults. The declared dependencies are:

| Dependency | Scope in this POM | Why it is here |
|---|---|---|
| `spring-boot-starter-web` | Compile | Servlet MVC, JSON support, embedded Tomcat |
| `spring-boot-starter-actuator` | Compile | Health and management endpoint infrastructure |
| `spring-boot-devtools` | Compile | Development-time restart and related conveniences |
| `spring-boot-starter-test` | Test | JUnit Jupiter, assertions, mocking, Spring testing support |

The build declares `spring-boot-maven-plugin`. With the Boot parent, its repackage execution produces an executable archive during packaging.
The plugin and starters solve different problems: build/launch tooling versus dependency composition.
The actual POM has no database starter, security starter, validation starter, or JSP dependency.

### 2. `LearnSpringBootApplication`: the composition root
File: `02-springboot/src/main/java/com/in28minutes/springboot/learnspringboot/LearnSpringBootApplication.java`.

Repo excerpt:
```java
@SpringBootApplication
public class LearnSpringBootApplication {

    public static void main(String[] args) {
        SpringApplication.run(LearnSpringBootApplication.class, args);
    }

}
```
The main package is `com.in28minutes.springboot.learnspringboot`. Default component scanning includes that package and its descendants.
All controllers and the configuration-properties component are in that package. The web starter causes Boot to infer a servlet web application for this classpath.
Boot prepares an environment, creates a context, applies configuration, and starts the server.

This differs from the core module's direct `AnnotationConfigApplicationContext` launchers.
Here, Boot controls the startup workflow and integrates its externalized configuration system.
The main class is not a controller; it is the application bootstrap/configuration source.

### 3. `Course`: the JSON response model
File: `02-springboot/src/main/java/com/in28minutes/springboot/learnspringboot/Course.java`.

Repo excerpt:
```java
private long id;
private String name;
private String author;

public Course(long id, String name, String author) {
    super();
    this.id = id;
    this.name = name;
    this.author = author;
}
```
The class also supplies `getId()`, `getName()`, `getAuthor()`, and `toString()`. It is not a Spring bean, JPA entity, or Spring Data repository.
Controller-created instances are ordinary response objects. Jackson can serialize their properties through getters.
The absence of a no-argument constructor does not prevent this outgoing serialization.
Deserializing incoming JSON is a different operation and needs a suitable creation/binding strategy.

`toString()` is useful for Java logging, but it does not define the JSON response format. The JSON property names are `id`, `name`, and `author` under normal Jackson conventions.

### 4. `CourseController`: a minimal endpoint
File: `02-springboot/src/main/java/com/in28minutes/springboot/learnspringboot/CourseController.java`.

Repo excerpt:
```java
@RestController
public class CourseController {

    @RequestMapping("/courses")
    public List<Course> retrieveAllCourses() {
        return Arrays.asList(
                new Course(1, "Learn AWS", "in28minutes"),
                new Course(2, "Learn DevOps", "in28minutes"),
                new Course(3, "Learn Azure", "in28minutes"),
                new Course(4, "Learn GCP", "in28minutes")

                );
    }

}
```
`@RestController` combines controller semantics with response-body handling. Spring MVC invokes the method, and a message converter serializes the returned list as JSON.
There is no service layer or persistence call in this particular endpoint. The list is assembled for each method invocation using hardcoded data.

Important interview nuance: `@RequestMapping` without a `method` attribute is not GET-only.
This lesson commonly demonstrates it using GET, but the mapping itself is not restricted to GET. An explicit read-only API would normally use `@GetMapping("/courses")`.
That is a suggested exercise, not the current source.

Example GET response, reformatted for readability:
```json
[
  {"id": 1, "name": "Learn AWS", "author": "in28minutes"},
  {"id": 2, "name": "Learn DevOps", "author": "in28minutes"},
  {"id": 3, "name": "Learn Azure", "author": "in28minutes"},
  {"id": 4, "name": "Learn GCP", "author": "in28minutes"}
]
```
### 5. `CurrencyServiceConfiguration`: grouped property binding
File: `02-springboot/src/main/java/com/in28minutes/springboot/learnspringboot/CurrencyServiceConfiguration.java`.

Repo excerpt:
```java
@ConfigurationProperties(prefix = "currency-service")
@Component
public class CurrencyServiceConfiguration {

    private String url;
    private String username;
    private String key;
```
Repo excerpt:
```java
public String getUrl() {
    return url;
}

public void setUrl(String url) {
    this.url = url;
}
```
The class also has getter/setter pairs for `username` and `key`. The prefix groups `currency-service.url`, `.username`, and `.key` into one bean.
`@Component` registers the object; Boot's properties infrastructure binds its values. The current class uses mutable JavaBean-style binding, not constructor binding.
It has no validation annotations and treats all three properties as strings. It does not call the configured URL or authenticate to a currency service.

Missing properties can remain null in this unvalidated design. An invalid-looking URL string is not automatically rejected because the field type is `String`.
A validated immutable properties type would be a useful extension, not existing behavior.

### 6. `CurrencyConfigurationController`: the injected properties object
File: `02-springboot/src/main/java/com/in28minutes/springboot/learnspringboot/CurrencyConfigurationController.java`.

Repo excerpt:
```java
@Autowired
private CurrencyServiceConfiguration configuration;

@RequestMapping("/currency-configuration")
public CurrencyServiceConfiguration retrieveAllCourses() {
    return configuration;
}
```
This is field injection into a Spring-managed `@RestController`. The method name is `retrieveAllCourses`, despite returning currency configuration.
Request mapping determines the URL; the Java method name does not. The getters cause Jackson to serialize `url`, `username`, and `key`.

With the checked-in `dev` profile, the response contains:
```json
{
  "url": "http://dev.in28minutes.com",
  "username": "devusername",
  "key": "devkey"
}
```
In production, do not return a credentials-bearing configuration bean directly. Return a purpose-built non-sensitive DTO or remove the endpoint.
Constructor injection would make the controller's required dependency more explicit.

### 7. `application.properties`: base values and the active profile
File: `02-springboot/src/main/resources/application.properties`.

Repo excerpt:
```properties
logging.level.org.springframework=debug
spring.profiles.active=dev

currency-service.url=http://default1.in28minutes.com
currency-service.username=defaultusername
currency-service.key=defaultkey

management.endpoints.web.exposure.include=health,metrics
```
The base file supplies values usable by any active profile unless overridden. It explicitly activates `dev`; that is a course convenience rather than Boot's universal default.
Without that setting or an external active-profile setting, Boot uses the default-profile behavior.
The exposure include list replaces the default HTTP exposure set with `health,metrics`. It does not expose all Actuator endpoints.

### 8. `application-dev.properties`: development overrides
File: `02-springboot/src/main/resources/application-dev.properties`.

Repo excerpt:
```properties
logging.level.org.springframework=trace

currency-service.url=http://dev.in28minutes.com
currency-service.username=devusername
currency-service.key=devkey
```
Profile-specific values override base-file values at the corresponding config-data location. Development logging is very verbose because the Spring logger level is TRACE.
This can help observation, but it can also flood logs and expose sensitive diagnostic details.
Changing logger `org.springframework` does not necessarily change your application's logger level.

### 9. `application-prod.properties`: inheritance is easy to miss
File: `02-springboot/src/main/resources/application-prod.properties`.

Repo excerpt:
```properties
logging.level.org.springframework=info
```
The production file contains no currency overrides. If `prod` replaces the active `dev` profile, currency properties come from the base file.
They do not come from the development file just because that file exists on the classpath. The Actuator exposure setting also remains from the base file.

| Property | Checked-in active `dev` | Active `prod` only |
|---|---|---|
| Spring logging level | `trace` | `info` |
| Currency URL | `http://dev.in28minutes.com` | `http://default1.in28minutes.com` |
| Currency username | `devusername` | `defaultusername` |
| Currency key | `devkey` | `defaultkey` |
| HTTP Actuator exposure | `health,metrics` | `health,metrics` |

This is a concrete example of profile files overlaying base configuration, not replacing it wholesale. The sample production key is not safe real-world credential provisioning.

### 10. `notes.txt`, `readme.md`, and `final.md`
`notes.txt` contrasts TRACE development logging with INFO production logging. Its informal `warning` wording is not the usual Spring Boot configuration token; use `WARN`.
Standard levels include TRACE, DEBUG, INFO, WARN, ERROR, and OFF.

The readme's course order moves from manual setup problems to starters and auto-configuration, then DevTools, profiles, configuration properties, embedded servers, and Actuator.
`final.md` repeats the endpoint/property classes and final configuration snapshot. These files are teaching material; Boot does not parse them during startup.
Some snapshot build sections include Spring milestone repositories absent from the current POM. Do not add them just because they appear in historical documentation.

### 11. `LearnSpringBootApplicationTests`: context smoke test
File: `02-springboot/src/test/java/com/in28minutes/springboot/learnspringboot/LearnSpringBootApplicationTests.java`.

Repo excerpt:
```java
@SpringBootTest
class LearnSpringBootApplicationTests {

    @Test
    void contextLoads() {
    }

}
```
The empty test method is still meaningful: the Spring context must start for the test to pass. `@SpringBootTest` finds the Boot configuration and loads a broad application context.
Its default web environment is MOCK, not a live server on port 8080. This test does not assert `/courses` content, profile inheritance, or secret-redaction behavior.
A controller slice with `@WebMvcTest`, or a random-port integration test, would cover other concerns.

### 12. Trace one request through the actual stack
```text
curl GET /courses
        |
Embedded Tomcat receives HTTP request
        |
Servlet filters and DispatcherServlet
        |
Spring MVC handler mapping selects CourseController
        |
retrieveAllCourses() returns List<Course>
        |
Jackson-backed HTTP message converter creates JSON
        |
HTTP response to client
```
Boot configures much of this infrastructure because the web starter is present. Spring MVC supplies dispatch, mapping, argument handling, and message conversion abstractions.
Jackson supplies the concrete JSON serialization implementation. Tomcat supplies the servlet container implementation.
The user-written controller still supplies the endpoint-specific behavior.

## Core concepts explained
### Why Spring Boot exists
Spring Framework provides powerful abstractions, but applications need many coordinated choices:
compatible dependencies, configuration infrastructure, a server, logging, packaging, and observability. Historically, teams assembled and maintained more of those choices manually.
Boot reduces that setup work through convention, curated dependencies, and conditional defaults. It does not eliminate design decisions or make misconfiguration impossible.

| Before Boot-oriented conventions | With Spring Boot |
|---|---|
| Select many compatible dependency versions | Curated dependency management |
| Add individual web libraries | Web starter supplies a sensible group |
| Configure common infrastructure explicitly | Conditional auto-configuration supplies defaults |
| Install a separate servlet container for common deployments | Embedded container can travel with application |
| Hand-build operational endpoints | Actuator supplies supported management facilities |
| Configure executable packaging manually | Boot build plugins repackage an archive |

Boot's useful defaults are **opinions**, not unchangeable laws. You can replace compatible beans, change properties, exclude auto-configuration, and choose another server.
The best interview answer explains both convenience and the mechanism behind it.

### Spring Boot versus Spring versus Spring MVC
| Technology | Layer | Main job |
|---|---|---|
| Spring Framework | Foundation | Container, DI, transactions, data/web abstractions, testing |
| Spring MVC | Framework web module | Servlet-based request handling and controller model |
| Spring Boot | Application assembly/startup | Dependencies, conditional configuration, launch, packaging, operations |
```text
Application code: controllers, services, models
                         |
Spring Boot: startup + defaults + operational integration
                         |
Spring Framework: container, MVC, transactions, resources
                         |
Implementations: Tomcat, Jackson, database drivers, etc.
```
Boot can create non-web, servlet-web, or reactive-web applications. Spring Framework can be used without Boot, as the core module illustrates.
Spring MVC can also be configured without Boot. These names describe complementary layers, not competing products doing the same job.

### Starters: dependency bundles, not generated application code
A starter is a dependency descriptor for a coherent feature set. `spring-boot-starter-web` normally brings Spring MVC, JSON support, logging, and embedded Tomcat.
The exact dependency graph is version-specific; inspect it rather than memorize every transitive artifact.
```bash
mvn -f 02-springboot/pom.xml dependency:tree
```
| Common starter | Typical purpose | Present here? |
|---|---|---|
| `spring-boot-starter` | Basic Boot, context, logging | Transitive through other starters |
| `spring-boot-starter-web` | Servlet MVC and embedded server | Yes |
| `spring-boot-starter-webflux` | Reactive web stack | No |
| `spring-boot-starter-actuator` | Operational endpoints | Yes |
| `spring-boot-starter-data-jpa` | JPA and repository support | No |
| `spring-boot-starter-security` | Security integration/defaults | No |
| `spring-boot-starter-validation` | Bean Validation implementation | No |
| `spring-boot-starter-test` | Test libraries | Yes, test scope |

A starter makes relevant classes available; auto-configuration interprets that classpath. The starter itself does not create `CourseController` or infer application business rules.
A BOM or parent manages versions; it does not download every managed dependency automatically. Only declared/transitive dependencies are resolved.

### `@SpringBootApplication`: the accurate shorthand
Interview shorthand:
```text
@SpringBootApplication
    = configuration + auto-configuration + component scanning
```
More precisely, it combines `@SpringBootConfiguration`, `@EnableAutoConfiguration`, and `@ComponentScan`, with Boot-specific exclusions and aliases.
`@SpringBootConfiguration` is itself a specialized `@Configuration` annotation. So saying `@Configuration + @EnableAutoConfiguration + @ComponentScan` captures the main intent,
but it is not the complete annotation declaration.

Component scanning finds user components in the main package tree. Auto-configuration imports infrastructure configurations from dependencies.
Those are separate discovery mechanisms; auto-configurations need not be under your application's package.

### How auto-configuration works in Boot 3.3
At a practical level:

1. `SpringApplication` prepares startup state and an application context.
2. The main configuration enables auto-configuration imports.
3. Boot discovers candidate auto-configuration class names from dependency metadata.
4. Conditions determine which configurations and bean methods apply.
5. User definitions and the environment participate in those decisions.
6. Applicable configurations create infrastructure beans, often backing off from user-defined equivalents.
```text
Classpath dependencies + Environment + User bean definitions
                            |
               Candidate auto-configurations
                            |
                     @Conditional checks
                            |
           Applicable bean definitions and defaults
                            |
                 Managed running application
```
For this module, a servlet web classpath, server classes, and MVC/JSON libraries make web-server, dispatcher, MVC, and message-converter configuration candidates relevant.
Actuator dependencies make endpoint infrastructure candidates relevant. There is no JDBC starter or driver/database configuration to justify inventing a working database here.

### `spring.factories` versus `AutoConfiguration.imports`
| Version/context | Discovery metadata |
|---|---|
| Older Boot generations | Auto-configuration candidates were listed under `EnableAutoConfiguration` in `META-INF/spring.factories` |
| Boot 2.7 transition | Introduced the dedicated auto-configuration imports mechanism |
| Boot 3.x auto-configuration | `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` |
| Other supported Spring/Boot factory extension points | `spring.factories` still has uses; it is not universally deleted |

Boot 3 auto-configurations should be registered with the dedicated imports file.
Do not write a new Boot 3 library assuming the old `EnableAutoConfiguration` factories key is sufficient. An imports file contains auto-configuration class names, one per line.
These are dependency resources, not Java components discovered by your app's package scan.

Illustration of library metadata, **not a file in this module**:
```text
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.currency.autoconfigure.CurrencyClientAutoConfiguration
```
A library normally uses `@AutoConfiguration` for those configuration classes. Ordering can be expressed with supported before/after auto-configuration metadata.
Do not infer that source-file order or starter declaration order establishes bean creation order.

### Conditions: what the defaults depend on
| Condition | Question it asks |
|---|---|
| `@ConditionalOnClass` | Are specified classes available? |
| `@ConditionalOnMissingClass` | Are specified classes absent? |
| `@ConditionalOnBean` | Do matching beans/definitions exist? |
| `@ConditionalOnMissingBean` | Is a matching user/default bean absent? |
| `@ConditionalOnProperty` | Does a property meet the specified presence/value rules? |
| `@ConditionalOnWebApplication` | Is this the appropriate kind of web application? |
| `@ConditionalOnNotWebApplication` | Is the application non-web? |
| `@ConditionalOnResource` | Is a specified resource present? |
| `@ConditionalOnExpression` | Does an expression evaluate as required? |

`@ConditionalOnMissingBean` is a common basis for **backing off**. If the user supplies a suitable bean, the library's default need not be created.
Not every auto-configuration backs off on every bean type, so check its actual conditions.

Illustration of a custom Boot library default:
```java
@AutoConfiguration
@ConditionalOnClass(CurrencyClient.class)
class CurrencyClientAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean(CurrencyClient.class)
    @ConditionalOnProperty(prefix = "currency-client", name = "enabled",
                           havingValue = "true", matchIfMissing = true)
    CurrencyClient currencyClient() {
        return new CurrencyClient();
    }
}
```
`CurrencyClient` above is illustrative and is not implemented in this repo. Bean conditions depend on definitions processed so far, which is why configuration ordering matters.
A library factory's declared return type should expose enough type information for conditions.
A condition is not a general-purpose runtime feature toggle that rechecks on every method call.

### Inspecting and overriding auto-configuration
Start with a condition evaluation report rather than guessing:
```bash
mvn -f 02-springboot/pom.xml spring-boot:run \
  -Dspring-boot.run.arguments="--debug"
```
`--debug` activates selected diagnostic logging including a condition report. It does not set every application logger to DEBUG.
The base file's `logging.level.org.springframework=debug` is a logger setting, not the same as a command-line `--debug` switch.

To alter a default, prefer the smallest supported mechanism:

1. Change a documented configuration property.
2. Supply a suitable application bean when the auto-configuration supports backing off.
3. Exclude a specific auto-configuration when it is genuinely unwanted.
4. Replace more infrastructure only when you understand the defaults you are taking responsibility for.

Exclusion illustration:
```java
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
class ExampleApplication {
}
```
This is not necessary in the unchanged module and is not existing source. A configuration exclusion is not a universal fix for bad database settings.
Likewise, enabling bean-definition overriding is not the same as conditional backing off. Boot disables same-name bean-definition overriding by default.

### Embedded servers
An embedded server is started and managed as part of the application process. It does not mean that the server is a simplistic mock or a development-only feature.
This module's servlet web starter defaults to Tomcat. Boot also supports alternative servlet servers such as Jetty and Undertow when dependencies are changed correctly.
The reactive web starter commonly uses Reactor Netty instead; this module is not reactive.
```text
Traditional WAR deployment:
  separately managed servlet container -> deploy application WAR

Executable Boot JAR:
  java -jar application.jar -> application starts its embedded server
```
Useful settings include `server.port`, `server.servlet.context-path`, and server-specific options.
`server.port=0` requests an available ephemeral port, useful for integration tests. A port collision prevents the new server from binding successfully.
Changing `server.port` is usually the smallest solution in a local exercise.

Replacing Tomcat typically means excluding `spring-boot-starter-tomcat` from the web starter and adding the corresponding alternative server starter.
Adding two server implementations without understanding selection is not a clean replacement.
A non-web application can use `spring.main.web-application-type=none` where appropriate.

### Profiles: conditional environments, not separate applications
Profiles identify environment-specific configuration or bean groups. `application-dev.properties` is loaded when `dev` is active.
`application-prod.properties` is loaded when `prod` is active. The base `application.properties` still participates in both cases.

`@Profile("dev")` can restrict a bean/configuration to an active profile. Profile expressions support logical combinations such as `!prod` or `dev & local`.
Profiles can be combined, but overlapping values require a defined precedence strategy.
For multiple active profiles, later profiles generally override earlier ones at the same config location.
Property-source location precedence still matters; profile order is not the only dimension.

An externally supplied `spring.profiles.active=prod` replaces the lower-precedence base active-profile value. It does not automatically mean both `dev` and `prod` are active.
`spring.profiles.include` and profile groups can add profiles intentionally. Use active/include/group declarations in appropriate non-profile-specific documents.
Boot restricts placing profile activation control properties inside profile-specific documents.

A profile is not a secret store, an access-control rule, or proof that production is configured safely.
Here, the `prod` profile still inherits sample currency credentials from the base file.

### `@ConfigurationProperties` versus `@Value`
| Concern | `@ConfigurationProperties` | `@Value` |
|---|---|---|
| Best use | Grouped, typed application settings | Small individual values or expressions |
| Object binding | Nested objects, collections, structured properties | Usually a single injection expression |
| Relaxed binding | Dedicated binder support | Limited; canonical placeholder names are recommended |
| Type conversion | Binder converts to declared property types | Conversion for injected values |
| Validation | Supports Bean Validation on properties types | No equivalent grouped binding validation by itself |
| SpEL | Property binding does not evaluate SpEL expressions | Supports SpEL injection expressions |
| Registration | Needs component/scan/enable registration | Injection point must belong to a managed object |

Illustration of one value:
```java
@Value("${currency-service.url:http://localhost:9000}")
private String url;
```
The `:` supplies a placeholder default when appropriate. Without a default, an unresolved required placeholder can fail bean creation.
This illustration is not how the existing currency properties bean is implemented.

Boot's binder supports conversion to useful types such as `Duration`, `DataSize`, numbers, and enums.
String fields, as in the current class, do not automatically enforce domain validity.
Use nested properties objects for coherent settings rather than a large unrelated bag of strings.

### Registering and validating immutable configuration
The existing properties class is registered through `@Component` and setter binding.
Alternatives are `@ConfigurationPropertiesScan` or `@EnableConfigurationProperties(Type.class)`. Avoid blindly registering the same type through several mechanisms.
Constructor-bound properties types should use the supported scanning/enabling approach, not ordinary component construction with property strings treated as bean dependencies.

Illustration of a proposed record, **not current source**:
```java
@ConfigurationProperties(prefix = "currency-service")
@Validated
public record CurrencyProperties(
        @NotBlank String url,
        @NotBlank String username,
        @NotBlank String key) {
}
```
Register this type with properties scanning or explicit enabling. The example also requires a Bean Validation implementation, commonly the validation starter.
The current POM does not declare that starter, so merely adding constraints is not a complete exercise.
With a single parameterized constructor, Boot 3 usually infers constructor binding for such registered types.
Use `jakarta.validation` constraints, not legacy `javax.validation` imports.

Relaxed binding maps supported external forms to canonical property names. For example, the environment-variable form for `currency-service.url` is `CURRENCYSERVICE_URL`:
replace dots with underscores, remove dashes, and uppercase. Do not assume arbitrary added underscores have identical binding semantics.
Relaxed binding does not relax malformed values or bypass validation.

### Externalized configuration precedence
A key's effective value is determined by its highest-precedence applicable property source. Do not answer simply "profile files always win" or "environment variables always win."
A useful Boot 3.3 ordering, **lowest to highest**, is:

| Order | Source |
|---|---|
| 1 | Defaults set through `SpringApplication.setDefaultProperties` |
| 2 | `@PropertySource` on configuration classes |
| 3 | Config data, including `application.properties`/YAML and imported config |
| 4 | Random-value property source for `random.*` |
| 5 | Operating-system environment variables |
| 6 | Java system properties |
| 7 | JNDI attributes, where available |
| 8 | ServletContext initialization parameters |
| 9 | ServletConfig initialization parameters |
| 10 | `SPRING_APPLICATION_JSON` properties |
| 11 | Command-line arguments |
| 12 | Test-specific `properties` declarations, such as on `@SpringBootTest` |
| 13 | `@DynamicPropertySource` in tests |
| 14 | `@TestPropertySource` |
| 15 | DevTools global settings when DevTools is active |

Not all sources exist in every application.
Later applicable sources override earlier ones for the same key.
Command-line binding can be disabled explicitly, so describe normal defaults when answering.
`@PropertySource` is added during context refresh and is too late for some early startup settings,
including certain `logging.*` and `spring.main.*` values.

Within normal config-data locations, precedence generally grows in this order:

1. Packaged base application files.
2. Packaged profile-specific application files.
3. External base application files.
4. External profile-specific application files.

Explicit location groups and imports can refine ordering, so examine deployment-specific inputs.
If `.properties` and YAML exist in the same location, `.properties` takes precedence.
Prefer one format per location to avoid confusing overlays.

`spring.config.location` replaces the default search locations.
`spring.config.additional-location` adds locations alongside the defaults.
`spring.config.import` imports additional config data, such as a config tree.
These location settings are evaluated early and should be supplied through appropriate early sources.

### Apply precedence to this repository
Consider an illustrative packaged-JAR launch:
```bash
java -Dserver.port=8082 \
  -jar 02-springboot/target/learn-spring-boot-0.0.1-SNAPSHOT.jar \
  --server.port=8083 --spring.profiles.active=prod
```
The HTTP server uses `8083`, because a normal application argument overrides the system property.
The active profile is `prod`, overriding the base file's `dev` setting.
Currency values then inherit from the base file unless another higher-priority source overrides them.
A Java `-D` option belongs before `-jar`; afterward it is merely an application argument string.

Illustrative shell environment override:
```bash
CURRENCYSERVICE_URL=http://localhost:9000 \
  java -jar 02-springboot/target/learn-spring-boot-0.0.1-SNAPSHOT.jar
```
The environment URL overrides the config-data URL while the other properties can remain from `dev`.
Property sources overlay individual keys, not necessarily entire grouped objects.
Do not expect changing an external file to automatically rebind a running bean in plain Boot.
Dynamic refresh requires a deliberate additional mechanism or restart.

### Actuator: operational endpoints
Actuator adds management functionality through auto-configuration and endpoint abstractions.
An endpoint must be enabled/available and exposed through a transport to be reachable there.
Exposure and security are separate controls.
The module includes `health,metrics` for HTTP exposure, under the default `/actuator` base path.

| Endpoint | Typical use | HTTP-exposed in current module? |
|---|---|---|
| `/actuator` | Discovery links for exposed endpoints | Normally yes |
| `/actuator/health` | Aggregate health | Yes |
| `/actuator/metrics` | Names of registered metrics | Yes |
| `/actuator/metrics/{name}` | Measurement details for one metric | Yes, if that metric exists |
| `/actuator/info` | Configured application/build information | No |
| `/actuator/env` | Environment/property-source diagnostics | No |
| `/actuator/configprops` | Bound configuration-properties diagnostics | No |
| `/actuator/beans` | Bean metadata | No |
| `/actuator/mappings` | Handler mapping information | No |
| `/actuator/conditions` | Condition evaluation information | No |
| `/actuator/loggers` | Logger levels; potentially changeable | No |
| `/actuator/prometheus` | Prometheus scrape format | No; also requires appropriate registry |
| `/actuator/shutdown` | Application shutdown | Disabled by default and not exposed |

Boot's default HTTP exposure without the custom include list is health only.
Do not confuse the default endpoint-enablement policy with default HTTP exposure.
This module adds metrics exposure deliberately.
Adding Actuator does not automatically publish every endpoint or install every metrics backend.

`/actuator/metrics` returns a list of metric names, not all individual measurements.
A subsequent request such as `/actuator/metrics/jvm.memory.used` returns measurements and tags.
Metric availability depends on installed binders and activity; avoid asserting every possible metric exists.
Micrometer provides the metrics abstraction and integrations used by Boot.

### Actuator health, details, and security
A common healthy response is:
```json
{"status":"UP"}
```
Health details are hidden by default; configure detail visibility deliberately if needed.
A health contributor can report a dependency's status, but the current hardcoded currency settings
are not a currency-service connectivity health check.
UP does not prove that every business workflow is correct.

This POM does not include Spring Security.
Exposed management endpoints therefore must not be presumed authenticated by default.
Use authentication/authorization, network boundaries, and least-privilege exposure in a real deployment.
A separate management port can be useful but is not, by itself, authentication.

With Spring Security present, Boot can provide management security defaults when the application
has not supplied its own applicable security configuration.
Once you define your own `SecurityFilterChain`, you must deliberately handle the relevant rules.
Do not blindly expose `env`, `configprops`, `heapdump`, or mutable logger controls publicly.
Sanitization helps but does not replace access control.

### Liveness, readiness, and observability
Liveness asks whether the application is fundamentally alive and should remain running.
Readiness asks whether it should receive traffic now.
Boot supports health groups for these concerns, with probe behavior depending on environment/settings.
Do not make a transient external database failure automatically trigger endless liveness restarts.
Readiness can include carefully selected dependencies relevant to accepting traffic.

Metrics provide numeric measurements; logs describe events; traces relate work across boundaries.
Actuator and Micrometer integrate operational facilities, but they do not replace an observability plan.
The unchanged module exposes health and metrics only, not a complete tracing/monitoring deployment.

### DevTools: development convenience, not production hot deployment
DevTools includes automatic restart, LiveReload integration, and development-oriented defaults.
Its restart mechanism typically uses a base classloader for stable libraries and a restart classloader
for changing application classes/resources.
It restarts application context/application code when relevant classpath changes are detected.
It does not redefine every running class without restart like a full hot-swap tool.
```text
Developer changes source
          |
IDE/build recompiles into classpath output
          |
DevTools detects relevant changed files
          |
Restart classloader and application context are refreshed
          |
Developer retries request
```
Saving a `.java` file alone may not compile it in every IDE workflow.
Check compiler/autobuild settings before concluding DevTools is broken.
Some static/template changes are excluded from restart and can trigger LiveReload instead.
LiveReload requires suitable client/browser integration; it is not the same as Java recompilation.

DevTools is normally disabled for production-style packaged launches.
Boot's Maven repackage tooling excludes DevTools from executable archives by default.
The repo declares DevTools directly without `optional=true` or `runtime` scope.
Production-minded dependency declarations often use runtime scope and optional dependency propagation.
Do not enable remote DevTools in a public production deployment as a convenience shortcut.

### Executable fat JAR packaging
A normal library JAR mainly contains application classes/resources and expects dependencies elsewhere.
A Boot executable JAR includes application content, dependency archives, and Boot launch support.
Typical layout:
```text
application.jar
  META-INF/MANIFEST.MF
  org/springframework/boot/loader/...    Boot launcher support
  BOOT-INF/classes/...                 Application classes and resources
  BOOT-INF/lib/...                     Dependency JARs
```
The manifest's `Main-Class` identifies Boot's launcher.
`Start-Class` identifies the application main class.
The launcher loads nested dependencies and invokes the application.
Thus `java -jar` works without manually assembling a huge external classpath.

The repackage goal transforms the archive created by Maven's normal packaging steps.
The Boot parent supplies an inherited repackage execution for the declared plugin.
A BOM import alone provides dependency management, not all parent plugin configuration.
Reproducible deployment still requires a compatible JVM, external configuration, and operational controls.
Executable packaging does not make secrets inside the JAR private.

WAR deployment remains possible when explicitly configured for a servlet container.
An executable JAR is the natural demonstration in this module.
JSP support has executable-archive/container limitations; do not assume every JSP application works as a fat JAR.
This module returns JSON and has no JSP views.

### Boot 3 changes: baseline and Jakarta migration
Spring Boot 3 builds on Spring Framework 6 and requires Java 17 or later.
The course projects target Java 21, which exceeds that baseline.
Boot 3 moved relevant Jakarta EE APIs from legacy `javax.*` namespaces to `jakarta.*`.
This affects servlet, persistence, validation, and lifecycle APIs among others.

| Concern | Older example import | Boot 3-era equivalent |
|---|---|---|
| Servlet API | `javax.servlet.*` | `jakarta.servlet.*` |
| JPA | `javax.persistence.*` | `jakarta.persistence.*` |
| Bean Validation | `javax.validation.*` | `jakarta.validation.*` |
| Lifecycle annotations | `javax.annotation.PostConstruct` | `jakarta.annotation.PostConstruct` |
| Standard injection | `javax.inject.Inject` | `jakarta.inject.Inject` |

Do not mechanically replace every `javax` package.
Java SE APIs such as `javax.sql.DataSource`, `javax.crypto`, and `javax.net` remain in their Java SE namespaces.
An old dependency compiled against `javax.servlet` is not fixed merely by editing your own imports.
Dependencies and runtime containers must also be Jakarta-compatible.

Boot 3 introduced AOT/native-image support and broader observability integration.
A native executable is different from a normal executable JAR and has additional build/runtime constraints.
Java 21 features such as virtual threads are not automatically enabled merely by choosing Java 21.
Boot 3.2+ has virtual-thread support for supported use cases with explicit configuration such as
`spring.threads.virtual.enabled=true`; capacity and blocking behavior still require evaluation.

### Root note: `spring-boot-3-1-updates.md`
The course note recommends Spring Security's lambda DSL as older non-lambda methods were deprecated
in the associated Security generation.
Those are Security API changes encountered with Boot-managed dependencies, not new syntax for controllers.

Repo excerpt from the note:
```java
http.httpBasic(withDefaults());
```
The static import is `org.springframework.security.config.Customizer.withDefaults`.
The note also shows lambda forms for CSRF, frame options, and OAuth2 resource-server JWT configuration.
Deprecation does not mean every old call immediately stops compiling in Boot 3.3.
Disabling CSRF or frame protections is a policy decision, not a mandatory migration action.
Do not blindly apply such sample security disablement to a real application.

The MySQL coordinate update in the note is:
```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
</dependency>
```
The older coordinates shown are `mysql:mysql-connector-java`.
These reflect the driver's artifact relocation/renaming and course version guidance.
The present Boot module does not require either driver because it has no database access.

### Root note: `spring-boot-3.2.x-changes.md`
The course update recommends the Jakarta JSTL implementation coordinates:
```xml
<dependency>
    <groupId>org.glassfish.web</groupId>
    <artifactId>jakarta.servlet.jsp.jstl</artifactId>
</dependency>
```
It also uses this Jakarta Tags core declaration:
```jsp
<%@ taglib prefix="c" uri="jakarta.tags.core" %>
```
The note contrasts older `org.eclipse.jetty:glassfish-jstl` course configuration
and the legacy `http://java.sun.com/jsp/jstl/core` taglib URI.
It recommends rebuilding the relevant module after changing dependencies.
This is course migration guidance, not evidence that `02-springboot` serves JSPs.
Older URI compatibility can depend on the implementation; the Jakarta URI is the modern teaching target.

## Common pitfalls / gotchas
### Startup and dependency mistakes
- Boot is not a replacement DI framework; it builds on Spring Framework.
- A starter provides dependencies, not business endpoints.
- The Boot parent/BOM manages compatible versions; it does not include every starter automatically.
- Moving the main class can put controllers outside the default component scan.
- An embedded server can fail because port 8080 is already occupied.
- A Java 21-compiled application cannot run on Java 17 merely because Boot's baseline is 17.
- Adding `@EnableWebMvc` takes more direct responsibility for MVC configuration and can disable Boot MVC defaults.
- Excluding auto-configuration is not a substitute for diagnosing its conditions.

### Configuration mistakes
- This repository activates `dev` in the base file; Boot does not universally default to `dev`.
- The `prod` file only changes logging and inherits base currency values.
- An external active-profile value normally replaces, rather than merges with, the lower-precedence value.
- Property sources overlay keys; they do not replace an entire properties object as a unit.
- `@ConfigurationProperties` needs registration as well as its annotation.
- The current String fields do not validate a URL or require a nonblank key.
- Immutable constructor-bound properties should use properties scanning/enabling, not ordinary component injection.
- OS environment-variable normalization removes dashes; use the supported canonical mapping.
- A Maven `-D` option is not automatically an application argument in every forked launch.
- A Java system property belongs before `-jar`.
- Plain Boot does not automatically refresh a bound configuration bean when an external file changes.

### Endpoint and operations mistakes
- The current `@RequestMapping` annotations do not restrict endpoints to GET.
- `Course.toString()` does not control JSON output.
- The currency endpoint leaks every getter-visible field, including the sample key.
- Actuator presence, endpoint enablement, HTTP exposure, and authorization are different controls.
- Only health is HTTP-exposed by default; this module additionally exposes metrics.
- `/actuator/metrics` lists names; metric-specific requests provide measurements.
- A healthy process is not proof of correct business behavior.
- The module has no security starter; do not assume authentication protects exposed endpoints.
- TRACE logging can be excessive and leak diagnostics in production.
- DevTools restart requires relevant compiled/classpath changes; source saving alone may be insufficient.
- DevTools restart is not arbitrary class-level hot swapping.
- Root Security/JSTL/MySQL notes describe other course scenarios, not dependencies already used here.

## Interview questions & answers
### Basic
#### 1. Why use Spring Boot?
It reduces application setup through curated dependencies, conditional defaults, embedded-server support,
executable packaging, and operational facilities.
It retains Spring's programming model and lets applications override supported defaults.
#### 2. How do Spring, Boot, and MVC differ?
Spring Framework is the foundation, Spring MVC is its servlet web framework,
and Spring Boot configures/launches applications built on those technologies.
This module uses all three together rather than choosing only one.
#### 3. What is a starter?
A coherent dependency bundle for a feature such as web, testing, or JPA.
The web starter makes MVC, JSON, and default embedded-server libraries available.
It does not generate this repository's controllers.
#### 4. What is auto-configuration?
Conditional configuration of infrastructure based on the classpath, properties, application type,
and existing bean definitions.
Boot supplies defaults where its conditions apply, often backing off from user-defined beans.
#### 5. What does `@SpringBootApplication` combine?
Configuration, auto-configuration enablement, and component scanning.
Precisely, it includes `@SpringBootConfiguration`, which specializes `@Configuration`,
plus `@EnableAutoConfiguration` and `@ComponentScan` with Boot-specific details.
#### 6. Where does component scanning begin here?
At `com.in28minutes.springboot.learnspringboot`, the main class's package.
The current controllers and properties component are in that package.
Unrelated parent/sibling packages are not automatically scanned.
#### 7. What starts the application?
`LearnSpringBootApplication.main()` calls `SpringApplication.run`.
That workflow prepares the environment and context and starts the embedded web server.
The Maven plugin merely provides a convenient way to launch it.
#### 8. Which embedded server is used by default?
Tomcat, through the servlet web starter's dependencies.
Other server choices require suitable dependency changes.
This is not a WebFlux/Reactor Netty application.
#### 9. What does `/courses` return?
Four hardcoded course objects for AWS, DevOps, Azure, and GCP, serialized as JSON.
The author of each is `in28minutes`.
There is no database retrieval in this controller.
#### 10. What does `@RestController` mean?
It combines `@Controller` with response-body semantics.
Returned objects are written through HTTP message converters rather than normally interpreted as view names.
Jackson handles the JSON representation here.
#### 11. What is a profile?
A named configuration/bean activation context, commonly used for environment differences.
`dev` and `prod` select corresponding property overlays in this module.
Profiles are not credential storage or authorization.
#### 12. Which profile is active in the unchanged application?
`dev`, because the base `application.properties` declares it.
That makes the development currency values and TRACE Spring logging effective.
This is a repository setting, not Boot's universal profile default.

### Intermediate
#### 13. What values apply when only `prod` is activated externally?
Spring logging is INFO.
Currency URL/username/key come from the base file because the prod file does not override them.
Actuator HTTP exposure remains `health,metrics` from the base file.
#### 14. How can you activate prod without editing files?
Use `--spring.profiles.active=prod` for the application or the Maven plugin's profiles option.
A normal command-line value overrides the lower-precedence value in the packaged base file.
For example:
```bash
mvn -f 02-springboot/pom.xml spring-boot:run -Dspring-boot.run.profiles=prod
```
#### 15. What if both `dev` and `prod` are active?
Both profile-specific files participate.
At the same config location, later profile values generally win for overlapping keys.
Because prod has no currency overrides, dev currency values may still remain; examine all sources.
#### 16. `@ConfigurationProperties` or `@Value`: which for currency settings?
Use grouped configuration properties because URL, username, and key form a coherent settings group.
It offers structured binding, conversion, and validation support.
`@Value` is suitable for isolated placeholder or expression injection.
#### 17. Why is `@Component` present on the current properties class?
It registers the mutable properties object as a bean for Boot to bind.
The annotation alone is not an instruction to scan every properties class automatically.
Scanning or explicit enabling is an alternative registration approach.
#### 18. Are the current currency properties validated?
No. They are plain strings with setters and no validation annotations.
Missing settings can remain null and malformed-looking URL strings are not automatically rejected.
Add constraints, validation infrastructure, and an appropriate type to enforce stronger rules.
#### 19. Does `@ConfigurationProperties` support immutable records in Boot 3?
Yes, when registered through supported properties scanning/enabling and constructor binding.
A single suitable constructor is generally inferred for constructor binding.
Do not register such a record as an ordinary component expecting String bean dependencies.
#### 20. What is relaxed binding?
Boot's property binder maps supported naming forms to canonical fields/properties.
For environment variables, `currency-service.url` maps to `CURRENCYSERVICE_URL`.
Relaxed naming does not make invalid values valid or disable validation.
#### 21. Which wins: a system property or an application argument for `server.port`?
The application argument under normal Boot command-line processing.
`java -Dserver.port=8082 -jar app.jar --server.port=8083` uses 8083.
Test and DevTools-specific sources can have still higher precedence in their own contexts.
#### 22. Which wins: an environment URL or `application-dev.properties`?
The OS environment property under normal precedence.
Other keys can still come from the dev file, because overrides operate per property key.
The active profile itself can be supplied externally too.
#### 23. What is Actuator?
Boot's operational endpoint infrastructure for health, metrics, diagnostics, and related management.
The current project includes it and HTTP-exposes health and metrics.
Exposure is separate from endpoint enablement and authentication.
#### 24. Are all Actuator endpoints exposed after adding the starter?
No. Boot's default HTTP exposure is health only.
This module explicitly changes that to health and metrics.
Prometheus additionally needs an appropriate metrics registry and exposure configuration.
#### 25. What does `/actuator/metrics` return?
Names of registered metrics.
Use `/actuator/metrics/{name}` to inspect a particular metric's measurements and tags.
Available names depend on registered binders, dependencies, and application activity.
#### 26. Does the current application secure Actuator automatically?
Do not assume so: this POM has no Spring Security starter.
Use explicit security and deployment/network controls before exposing operational endpoints in production.
A management URL prefix is not an authentication boundary.
#### 27. What does DevTools actually restart?
The development application using a restart classloader/context lifecycle after relevant classpath changes.
It is not arbitrary in-place class redefinition.
Source changes must reach compiled output or another watched classpath location to trigger useful restart.
#### 28. What is an executable fat JAR?
An application archive containing application content, dependency JARs, and Boot launcher support.
The repackage goal makes it launchable with `java -jar`.
Its nested dependencies usually live under `BOOT-INF/lib`.

### Advanced and scenario-based
#### 29. Does Boot 3 still use `spring.factories` for auto-configuration candidates?
Boot 3 uses the dedicated `AutoConfiguration.imports` metadata file for that registration.
Older generations used the `EnableAutoConfiguration` key in `spring.factories`.
Other supported factory extension points still use `spring.factories`, so it is not universally obsolete.
#### 30. How can auto-configurations outside the main package be discovered?
They are imported through Boot's dependency metadata and auto-configuration import mechanism.
They are not required to be found by the application's component scan.
User components and dependency auto-configurations have different discovery paths.
#### 31. What does `@ConditionalOnMissingBean` accomplish?
It allows a default definition when a matching bean is absent.
A user-provided suitable bean can make that default back off.
The condition's specific types, search rules, and processing order determine the exact result.
#### 32. Is backing off equivalent to overwriting a bean of the same name?
No. Backing off prevents a conditional default from being registered.
Same-name overriding replaces a registered definition and is disabled by default in Boot.
Do not enable overriding merely to hide accidental duplicate configuration.
#### 33. How do you diagnose unexpected auto-configuration?
Run with `--debug` and inspect the condition evaluation report.
Check positive/negative matches, classpath libraries, user bean definitions, and effective properties.
Use the smallest supported property/bean/exclusion change rather than randomly removing starters.
#### 34. What happens if a controller is moved outside the main package tree?
Default component scanning may no longer register it, producing unmapped requests such as a 404.
Move it under the root package or deliberately adjust scanning/imports.
Adding a web starter does not scan every package in the JVM.
#### 35. Does `Course` need a default constructor for the current API?
Not for the outgoing getter-based JSON serialization demonstrated here.
Incoming deserialization requires a suitable creator or binding strategy and is a separate concern.
Do not apply a deserialization rule blindly to a response-only model.
#### 36. Are `/courses` and `/currency-configuration` GET-only?
No. The current `@RequestMapping` annotations omit the method restriction.
They are demonstrated using GET but not limited by annotation to GET.
Use `@GetMapping` when the intended contract is read-only GET handling.
#### 37. Why is returning `CurrencyServiceConfiguration` dangerous in production?
Jackson serializes its getters, including `getKey()` and `getUsername()`.
A credentials-bearing configuration object becomes a public response model.
Use a non-sensitive DTO or remove the endpoint; Actuator sanitization does not protect this custom API.
#### 38. Why might an external file override a packaged profile-specific file?
External config-data locations normally have higher precedence than packaged locations.
Profile specificity is only one part of source precedence.
Inspect explicit locations/imports and deployment inputs instead of assuming a profile file always wins.
#### 39. `spring.config.location` or `spring.config.additional-location`: what is the difference?
`location` replaces default search locations; `additional-location` adds to them.
A replacement can accidentally stop Boot from finding expected packaged defaults.
These settings are consumed early and should be supplied through appropriate early configuration sources.
#### 40. Will changing an external URL automatically update the running properties bean?
Not in plain Boot without a deliberate refresh/rebinding mechanism.
Restart or use additional supported dynamic-configuration infrastructure.
DevTools is a development restart tool, not a production dynamic-refresh guarantee.
#### 41. Can you replace every `javax` import with `jakarta` during a Boot 3 migration?
No. Relevant Jakarta EE APIs moved, but Java SE packages such as `javax.sql` remain.
Update affected imports and compatible dependencies together.
Old javax-based servlet dependencies cannot be fixed solely by editing application imports.
#### 42. Why can Boot require Java 17 while this project needs Java 21 to run?
Java 17 is Boot 3's minimum baseline.
This project's compiler target is Java 21, producing bytecode that needs a compatible newer JVM.
Framework minimum and application compilation target are different compatibility constraints.
#### 43. Does the empty `contextLoads()` test start a real server?
With default `@SpringBootTest`, the web environment is MOCK rather than a live random-port server.
It verifies context startup, not HTTP content or a successful socket bind.
Use RANDOM_PORT or suitable MVC testing tools when those behaviors need coverage.
#### 44. Does adding `@EnableWebMvc` improve a normal Boot MVC application?
Not automatically. It signals more direct MVC configuration responsibility and can disable Boot MVC defaults.
For ordinary customization, prefer supported properties or a `WebMvcConfigurer` without taking over MVC entirely.
Only use full enablement when you intend that responsibility.
#### 45. Should a database outage make the application fail liveness?
Usually not merely because the dependency is temporarily unavailable.
Liveness-driven restarts may amplify the outage without repairing the database.
Readiness and carefully chosen health contributors are often the appropriate traffic-control signal.
#### 46. Do Java 21 virtual threads activate just because this POM targets Java 21?
No. Boot's supported virtual-thread integration requires appropriate configuration and execution paths.
It does not remove limits such as database connection pool capacity or CPU-bound work.
Measure behavior and compatibility rather than assuming all performance improves automatically.
#### 47. What should you learn from the root Boot 3.1 and 3.2 notes?
Use compatible Jakarta-era libraries, updated MySQL coordinates, and the modern Security lambda DSL
where relevant; use the course's modern JSTL coordinates/URI for its JSP scenarios.
Do not add those dependencies or disable security controls in this JSON-only module without a reason.
#### 48. What happens if port 8080 is already occupied?
The embedded server cannot bind and application startup fails with a port-in-use diagnosis.
Select another port using `--server.port=8081` or stop only the known conflicting application safely.
A successful context-only MOCK test does not prove port 8080 is available.

## Hands-on exercises
Try these later in a personal working copy.
The guide describes the tasks but does not modify source, dependency manifests, or properties.

### Exercise 1: make profile inheritance visible
1. Run the unchanged module and request `/currency-configuration`.
2. Record the dev URL, username, and sample key.
3. Restart with only `prod` active and compare the result.
4. Explain why the base currency values remain while logging changes to INFO.
5. Run with `dev,prod` and reason about overlapping versus non-overlapping properties.

**Success criterion:** explain effective values using overlays, not "prod replaces the entire file."
**Interview lesson:** active-profile selection and property-source precedence are separate dimensions.

### Exercise 2: prove three-source precedence
1. Package the module in your working copy.
2. Run with a system-property port and a different application-argument port.
3. Confirm that the argument port wins.
4. Supply `CURRENCYSERVICE_URL` in the process environment and inspect the currency endpoint.
5. Add a command-line currency URL and confirm that it overrides the environment value.

**Success criterion:** predict each winning value before requesting the endpoint.
**Interview lesson:** overrides operate per key, and Java/Maven/application arguments are not interchangeable.

### Exercise 3: improve the configuration boundary
1. In your copy, change the controller to constructor injection.
2. Return a DTO containing only a non-sensitive URL or remove the endpoint.
3. Add a properly registered immutable configuration-properties record.
4. Add the validation starter and Jakarta constraints if using validated configuration.
5. Test that a missing required value fails startup and that no key appears in the endpoint response.

**Success criterion:** registration, binding, validation, and response serialization are all considered.
**Interview lesson:** strongly typed configuration is not the same as secure handling of secrets.

### Exercise 4: trace auto-configuration and Actuator
1. Start the application with `--debug` and find servlet-server/MVC condition matches.
2. Request health and metrics, then one available metric by name.
3. Request a non-exposed diagnostic endpoint and distinguish missing exposure from authentication.
4. In an isolated local copy, expose `conditions` deliberately and inspect it.
5. Keep production exposure minimal and remove the local diagnostic exposure afterward.

**Success criterion:** connect dependencies, conditions, managed infrastructure, and HTTP exposure.
**Interview lesson:** observing a condition report is more useful than calling Boot "magic."

### Exercise 5: verify API semantics and developer workflow
1. Change the two read endpoints to `@GetMapping` in your copy.
2. Add a controller test for the four course names and JSON property names.
3. Add a test proving a disallowed HTTP method receives the expected MVC rejection.
4. Change a course string while DevTools is running and ensure compilation reaches watched output.
5. Package a JAR, run it separately, and explain why this is not the same DevTools restart workflow.

**Success criterion:** HTTP contract, serialization, compilation, restart, and packaging are distinguished.
**Interview lesson:** developer convenience must not be mistaken for production runtime behavior.

## Quick revision cheat sheet
| Prompt | Fast answer |
|---|---|
| Why Boot? | Sensible assembly/startup defaults with supported customization |
| Starter | Feature-oriented dependency bundle |
| Parent/BOM | Curated version management; parent adds build conventions |
| Boot main annotation | Configuration + auto-config + component scan; precisely Boot-specialized configuration |
| Auto-config inputs | Classpath, properties, application kind, existing definitions |
| Boot 3 auto-config metadata | `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` |
| `spring.factories` | Historical auto-config registration; still used for other supported factory hooks |
| Backing off | Conditional default omitted when a suitable bean exists |
| Diagnose conditions | `--debug` condition evaluation report |
| Default server here | Embedded Tomcat through servlet web starter |
| Default port | 8080 unless configured |
| Default checked-in profile | `dev`, explicitly configured by the repository |
| Prod currency values here | Base-file values, because prod only changes logging |
| Grouped settings | `@ConfigurationProperties` with registration |
| Single value/expression | `@Value` |
| Current properties style | Mutable setter binding, no validation |
| Environment example | `currency-service.url` -> `CURRENCYSERVICE_URL` |
| Normal precedence anchor | Config data < environment < system properties < application arguments |
| Test/global exceptions | Test sources and active DevTools global settings can rank higher |
| Config location | Replaces defaults; additional-location augments defaults |
| Current Actuator exposure | HTTP health and metrics |
| Default HTTP exposure | Health only |
| Metrics root | Names; individual metric paths give measurements |
| Actuator security | Separate from exposure; no security starter here |
| DevTools | Development restart/LiveReload/defaults, not arbitrary production hot swapping |
| Executable JAR | Boot loader + classes/resources + nested dependencies |
| Boot 3 baseline | Java 17+, Spring Framework 6, relevant Jakarta EE namespaces |
| Repository target | Java 21, not Java 17 bytecode |
| Namespace exception | Java SE `javax.sql`, `javax.crypto`, etc. do not migrate wholesale |

Final source-specific memory anchors:

- Main class: `com.in28minutes.springboot.learnspringboot.LearnSpringBootApplication`.
- `/courses` returns four hardcoded objects; there is no persistence layer.
- Both application endpoints currently use unrestricted-by-method `@RequestMapping`.
- `Course` getters drive outgoing JSON serialization; `toString()` does not.
- `CurrencyServiceConfiguration` combines `@Component` and `@ConfigurationProperties(prefix = "currency-service")`.
- The currency controller field-injects that bean and returns its key-bearing configuration directly.
- Base logging is DEBUG, dev overrides it to TRACE, and prod overrides it to INFO.
- Base configuration activates dev and HTTP-exposes `health,metrics`.
- Prod alone inherits the base URL `http://default1.in28minutes.com` and sample credentials.
- The smoke test loads a Boot context but does not assert endpoint content or start a live default-port server.
- Security/MySQL/JSTL update notes explain relevant course migrations, not active dependencies here.
- Explain Boot as conditional configuration and application assembly, not unexplained magic.
