# 05 — REST APIs with Spring Boot: Interview Study Guide
## What this module teaches
**Exact module:** `12-rest-api/`. **Runtime in its actual POM:** Spring Boot **3.3.4**, Java **21**, Spring MVC 6.x. This is the social-media REST API portion of in28minutes' *Master Spring and Spring Boot* course. It progresses from returning a greeting to persistence, validation, hypermedia, and API evolution.

Use this guide to connect an HTTP request to the exact controller and supporting objects. Distinguish **present source behavior** from **course snapshots** and **recommended improvements**. The repository intentionally retains commented experiments and earlier lesson snapshots.

### How to run
From the repository root, use an installed Maven and a Java 21 JDK:

```bash
java -version
mvn -version
cd 12-rest-api
mvn spring-boot:run
```

There is no Maven wrapper in this module, so do not assume `./mvnw` exists. The default server port is `8080`. No React frontend belongs to this module; curl, a browser, or an HTTP client is its client. Do not run the full-stack backend on the same port simultaneously.

For reproducible **local-only** examples, override the default generated credentials:

```bash
mvn spring-boot:run   -Dspring-boot.run.arguments="--spring.security.user.name=study --spring.security.user.password=study-only"
```

These are example credentials supplied at startup, not credentials configured in the source. Without that override, Boot supplies username `user` and logs a generated password. Do not use either demonstration credential strategy in production.

```bash
BASE=http://localhost:8080
curl -i -u study:study-only "$BASE/hello-world"
curl -i -u study:study-only -H 'Accept: application/json' "$BASE/users"
```

**Important:** the checked-in security configuration has no active `SecurityFilterChain` bean. Boot's default security remains active, including CSRF protection. Basic authentication alone does not make POST or DELETE succeed without a CSRF token. The hands-on section shows a token-aware curl flow without changing source files.

Optional commands for a learner working in a writable study copy:

```bash
mvn test
mvn package
java -jar target/restful-web-services-0.0.1-SNAPSHOT.jar
```

The only checked-in test is a context-loading test, not a complete endpoint suite. This documentation task does not execute those commands or change the application.

## Project walkthrough
### 1. Source and lesson map
The Java package root is:

```text
12-rest-api/src/main/java/
  com/in28minutes/rest/webservices/restfulwebservices/
```

Paths below beginning with `helloworld/`, `user/`, and similar prefixes are relative to that root. Resource paths are relative to `12-rest-api/src/main/resources/`. All snippets labeled **source excerpt** reproduce actual checked-in code, with indentation normalized. Examples labeled **illustrative** are explanations or proposed improvements, not existing endpoints.

```text
HTTP client
    |
    v
Spring Security filter chain
    |
    v
DispatcherServlet --> handler mapping --> controller
                                            |
                      +---------------------+-------------------+
                      |                                         |
                UserDaoService                       UserRepository/PostRepository
                 static List                                JPA/Hibernate
                      |                                         |
                 memory only                                H2 testdb
                      +---------------------+-------------------+
                                            |
                                  Jackson message converter
                                            |
                                      HTTP response
```

The following documentation was read alongside the current Java files:

| Module document | Main lesson context |
|---|---|
| `12-rest-api/readme.md` | Course goals, tools, and lecture inventory |
| `12-rest-api/01-step-by-step-changes/v2.md` | Full progression, URLs, request samples, and changes |
| `Step01.md`, `Step03.md`, `Step05.md` | Setup, greetings, beans, path variables |
| `Step07.md`, `Step08.md`, `Step09.md` | Users, reads, creation |
| `Step10.md`, `Step11.md`, `Step12.md`, `Step14.md` | Location, exceptions, global errors, validation |
| `Step17.md`, `Step21.md` | OpenAPI and versioning stages |
| `Step26.md` | Filtering, HATEOAS, operational exploration |
| `Step29.md`, `Step32.md` | JPA, users, posts, relational association |
| `Step35.md`, `Step37.md` | Optional MySQL and custom Basic authentication |

The `Step*.md` files are historical complete-code snapshots, not additional runtime modules. There are no `.txt` files in this module's inspected file inventory. Some snapshots enable libraries or annotations that the current POM/source comments out. The actual source wins when predicting what a running application does.

### 2. Bootstrap and dependency choices
File: `RestfulWebServicesApplication.java`.

**Source excerpt:**

```java
@SpringBootApplication
public class RestfulWebServicesApplication {
    public static void main(String[] args) {
        SpringApplication.run(RestfulWebServicesApplication.class, args);
    }
}
```

`@SpringBootApplication` combines configuration, component scanning, and auto-configuration. Its package placement allows controllers, repositories, and advice below it to be discovered. Boot supplies embedded servlet infrastructure and MVC configuration from the classpath.

File: `12-rest-api/pom.xml`.

| Dependency | What it contributes | Present state |
|---|---|---|
| `spring-boot-starter-web` | MVC, embedded Tomcat, Jackson JSON | Active |
| `spring-boot-starter-data-jpa` | Spring Data repositories and Hibernate | Active |
| `spring-boot-starter-validation` | Jakarta Bean Validation implementation | Active |
| `spring-boot-starter-security` | Authentication/authorization filters | Active |
| `spring-boot-starter-hateoas` | `EntityModel`, MVC link builders, HAL support | Active |
| `spring-boot-starter-actuator` | Operational endpoints | Active |
| `spring-data-rest-hal-explorer` | Browser-based HAL exploration | Active |
| `h2` | In-memory relational database | Runtime |
| `spring-boot-devtools` | Development restart support | Optional runtime |
| `spring-boot-starter-test` | JUnit and Spring testing | Test scope |
| `springdoc-openapi-starter-webmvc-ui`, `2.6.0` | OpenAPI JSON and Swagger UI | Commented out |
| `jackson-dataformat-xml` | XML conversion | Commented out |
| MySQL driver lesson | External persistent database | Commented out |

The SpringDoc coordinates shown in the current comment differ from older lesson snapshots. For Boot 3, use the SpringDoc **2.x starter**, not the older Springfox integration. A commented dependency is not on the runtime classpath.

### 3. Resource configuration and data
File: `12-rest-api/src/main/resources/application.properties`.

**Source excerpt:**

```properties
logging.level.org.springframework=info
management.endpoints.web.exposure.include=*
spring.datasource.url=jdbc:h2:mem:testdb
spring.h2.console.enabled=true
spring.jpa.defer-datasource-initialization=true
spring.jpa.show-sql=true
```

- `testdb` is an in-memory H2 database and is recreated across application restarts.
- JPA creates the embedded-database schema under Boot's normal defaults here.
- Deferred SQL initialization allows `data.sql` to run after Hibernate schema creation.
- SQL logging illustrates what repository and relationship access execute.
- H2 console enablement does not automatically override Security or frame protections.
- Exposing Actuator endpoints does not automatically make them anonymous.

File: `12-rest-api/src/main/resources/data.sql`.

**Source excerpt:**

```sql
insert into user_details(id,birth_date,name)
values(10001, current_date(), 'Ranga');

insert into post(id,description,user_id)
values(20001,'I want to learn AWS', 10001);
```

Users `10001`, `10002`, and `10003` and posts `20001` through `20004` are seeded. The static-list users instead have IDs `1`, `2`, and `3`. These stores are independent; creating a `/users` user does not insert a `/jpa/users` row. Seeded birth dates use today, while request validation requires a date strictly in the past. The SQL seed bypasses request validation, explaining that apparent inconsistency.

### 4. Greetings, binding, and internationalization
Files: `helloworld/HelloWorldController.java`, `helloworld/HelloWorldBean.java`.

**Source excerpt:**

```java
@GetMapping(path = "/hello-world-bean")
public HelloWorldBean helloWorldBean() {
    return new HelloWorldBean("Hello World");
}

@GetMapping(path = "/hello-world/path-variable/{name}")
public HelloWorldBean helloWorldPathVariable(@PathVariable String name) {
    return new HelloWorldBean(String.format("Hello World, %s", name));
}
```

The controller also returns plain `"Hello World"` at `/hello-world`. A bean response is serialized through Jackson; `getMessage()` exposes the `message` property. `toString()` is not the JSON serialization mechanism. `@PathVariable` extracts a URI segment, not a query parameter or JSON property.

**Source excerpt:**

```java
@GetMapping(path = "/hello-world-internationalized")
public String helloWorldInternationalized() {
    Locale locale = LocaleContextHolder.getLocale();
    return messageSource.getMessage("good.morning.message", null, "Default Message", locale );
}
```

`MessageSource` arrives through constructor injection. Boot discovers the default `messages.properties` bundle and locale variants. The MVC locale resolver normally derives the request locale from `Accept-Language`.

| File | Exact message |
|---|---|
| `messages.properties` | `good.morning.message=Good Morning` |
| `messages_nl.properties` | `good.morning.message=Goedemorgen` |

The source comments mention French and German, but no such bundles are checked in. An unsupported language falls back according to message-source/locale configuration. Do not promise a French translation merely because a comment says `Bonjour`.

### 5. In-memory user service and controller
Files: `user/UserDaoService.java`, `user/UserResource.java`.

**Source excerpt:**

```java
public User save(User user) {
    user.setId(++usersCount);
    users.add(user);
    return user;
}

public User findOne(int id) {
    Predicate<? super User> predicate = user -> user.getId().equals(id);
    return users.stream().filter(predicate).findFirst().orElse(null);
}
```

The service is a `@Component` holding a static mutable `ArrayList`. It teaches service separation without requiring SQL for the first CRUD lessons. `findAll()` returns the list itself; there is no paging or defensive copy. The counter and list are not a concurrent, durable repository. Deletion uses `removeIf`, which silently does nothing when the ID is absent.

**Source excerpt:**

```java
@PostMapping("/users")
public ResponseEntity<User> createUser(@Valid @RequestBody User user) {
    User savedUser = service.save(user);

    URI location = ServletUriComponentsBuilder.fromCurrentRequest()
                    .path("/{id}")
                    .buildAndExpand(savedUser.getId())
                    .toUri();

    return ResponseEntity.created(location).build();
}
```

`@RequestBody` asks an HTTP message converter to deserialize request bytes into `User`. `@Valid` asks Bean Validation to inspect the resulting object before invoking the method. `created(location)` selects `201 Created` and sets `Location`. `build()` returns no body, even though the declared generic type is `User`. Use `.body(savedUser)` only if the chosen API contract should return the created representation. The actual DELETE method returns `void`, so its ordinary success is **200**, not automatically 204.

### 6. User entity, post entity, and persistence
Files: `user/User.java`, `user/Post.java`.

**Source excerpt from `User`:**

```java
@Entity(name = "user_details")
public class User {
    @Id
    @GeneratedValue
    private Integer id;

    @Size(min=2, message = "Name should have atleast 2 characters")
    private String name;

    @Past(message = "Birth Date should be in the past")
    private LocalDate birthDate;

    @OneToMany(mappedBy = "user")
    @JsonIgnore
    private List<Post> posts;
```

The source also provides a protected no-argument constructor and public getters/setters. The explicit entity name avoids treating `user` as the entity/table name in this example. Boot's physical naming convention produces columns such as `birth_date`. Do not confuse a Java field name, an entity name, and a physical SQL column name.

**Source excerpt from `Post`:**

```java
@Size(min = 10)
private String description;

@ManyToOne(fetch = FetchType.LAZY)
@JsonIgnore
private User user;
```

```text
user_details                              post
+-------+-------+------------+            +-------+----------------+---------+
| id PK | name  | birth_date |  1 ------ N | id PK | description    | user_id |
+-------+-------+------------+            +-------+----------------+---------+
              User.posts                         Post.user owns FK
              mappedBy="user"                     user_id references User.id
```

The `Post.user` side owns the association because it supplies the foreign key. `mappedBy="user"` names the **Java association field**, not the database column `user_id`. Both association properties are ignored in JSON in the current source. This prevents recursive user→posts→user serialization and hides persistence internals. It does not remove the SQL relationship or prevent Java access to it. No cascade removal is configured; deleting a user who still has posts may violate the FK.

Files: `jpa/UserRepository.java`, `jpa/PostRepository.java`.

**Source excerpt:**

```java
public interface UserRepository extends JpaRepository<User, Integer> {
}
```

The post interface similarly extends `JpaRepository<Post, Integer>`. Spring Data builds repository implementations with `findAll`, `findById`, `save`, and `deleteById`. JPA is the persistence specification; Hibernate is the provider; H2 is the database. Those names are not interchangeable.

### 7. JPA-backed REST endpoints
File: `user/UserJpaResource.java`.

The constructor injects both repositories. Unlike `UserResource`, this controller reads and writes database rows. It still uses the same `User` representation and validation annotations.

**Source excerpt:**

```java
@GetMapping("/jpa/users/{id}/posts")
public List<Post> retrievePostsForUser(@PathVariable int id) {
    Optional<User> user = userRepository.findById(id);

    if(user.isEmpty())
        throw new UserNotFoundException("id:"+id);

    return user.get().getPosts();
}
```

The absent-parent check precedes collection access. A valid user with no posts returns an empty collection, not 404. Collection access may issue an additional SQL query because one-to-many is lazy by default. Boot's default Open EntityManager in View normally keeps persistence access available here. For production, prefer transaction-scoped loading and DTO mapping over reliance on that default.

**Source excerpt:**

```java
post.setUser(user.get());

Post savedPost = postRepository.save(post);

URI location = ServletUriComponentsBuilder.fromCurrentRequest()
        .path("/{id}")
        .buildAndExpand(savedPost.getId())
        .toUri();

return ResponseEntity.created(location).build();
```

These lines belong to `createPostForUser`, mapped to `POST /jpa/users/{id}/posts`. The server assigns the parent from the path rather than trusting a JSON relationship. The resulting `Location` is `/jpa/users/{userId}/posts/{postId}`. **There is no GET mapping for that individual-post URL in the actual controller.** Treat this as a contract gap to fix in an exercise, not an endpoint already available. The collection GET is the existing way to inspect the new post.

### 8. Exception handling and error contract
Files: `user/UserNotFoundException.java`, `exception/CustomizedResponseEntityExceptionHandler.java`.

**Source excerpt:**

```java
@ResponseStatus(code = HttpStatus.NOT_FOUND)
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(String message) {
        super(message);
    }
}
```

This exception carries a default status policy when MVC's response-status resolver handles it. The controller advice supplies a more explicit status and custom body for the same exception. The advice's matching handler handles the exception before fallback resolution is needed.

**Source excerpt:**

```java
@ExceptionHandler(UserNotFoundException.class)
public final ResponseEntity<ErrorDetails> handleUserNotFoundException(Exception ex, WebRequest request) throws Exception {
    ErrorDetails errorDetails = new ErrorDetails(LocalDateTime.now(),
            ex.getMessage(), request.getDescription(false));

    return new ResponseEntity<ErrorDetails>(errorDetails, HttpStatus.NOT_FOUND);
}
```

The advice extends `ResponseEntityExceptionHandler` and also handles generic `Exception` as 500. It overrides `handleMethodArgumentNotValid` using Spring 6's **`HttpStatusCode`** parameter. It returns `400 Bad Request` with a total error count and the first field error message. The exact source calls `ex.getFieldError().getDefaultMessage()` without a null guard. That assumes a field error exists; future object-level validation could break that assumption.

File: `exception/ErrorDetails.java`. Its three properties are `timestamp: LocalDateTime`, `message: String`, and `details: String`. Getters let Jackson serialize a response similar to:

```json
{
  "timestamp": "2026-10-31T12:00:00",
  "message": "id:999999",
  "details": "uri=/users/999999"
}
```

The timestamp above is illustrative, not copied execution output. Authentication failures occur in filters and need security entry points/denied handlers. A controller advice is not a universal handler for every filter-chain failure. Avoid exposing raw internal exception messages as the public production contract.

### 9. Static and dynamic Jackson filtering
Files: `filtering/SomeBean.java`, `filtering/FilteringController.java`.

**Source excerpt:**

```java
//@JsonIgnoreProperties({"field1","field2"})
@JsonFilter("SomeBeanFilter")
public class SomeBean {
    private String field1;
    private String field2;
    //@JsonIgnore
    private String field3;
```

The static annotations in this bean are **commented examples**. Active `@JsonIgnore` usages exist on the JPA relationship fields discussed earlier. `@JsonIgnore` excludes a property; class-level `@JsonIgnoreProperties` can list several names. Neither is a request-specific authorization decision.

**Source excerpt from `/filtering`:**

```java
SimpleBeanPropertyFilter filter =
        SimpleBeanPropertyFilter.filterOutAllExcept("field1","field3");

FilterProvider filters =
        new SimpleFilterProvider().addFilter("SomeBeanFilter", filter );

mappingJacksonValue.setFilters(filters );
```

`MappingJacksonValue` wraps the payload with request-specific serialization metadata. The filter name must match `@JsonFilter("SomeBeanFilter")` exactly. The other handler, `/filtering-list`, selects `field2` and `field3` instead.

| Request | JSON fields visible | Example |
|---|---|---|
| `GET /filtering` | `field1`, `field3` | `{"field1":"value1","field3":"value3"}` |
| `GET /filtering-list` | `field2`, `field3` | First object contains `value2`, `value3` |

A bean marked with `@JsonFilter` can fail serialization if no matching provider is supplied. Comments beside mappings are historical hints; the actual filter whitelist determines output. For stable public contracts, explicit response DTOs often communicate intent more clearly.

### 10. Four versioning strategies in one controller
Files: `versioning/VersioningPersonController.java`, `PersonV1.java`, `PersonV2.java`, `Name.java`.

**Source excerpt:**

```java
@GetMapping("/v2/person")
public PersonV2 getSecondVersionOfPerson() {
    return new PersonV2(new Name("Bob", "Charlie"));
}

@GetMapping(path = "/person", params = "version=1")
public PersonV1 getFirstVersionOfPersonRequestParameter() {
    return new PersonV1("Bob Charlie");
}
```

`PersonV1` exposes a string `name`; `PersonV2` exposes a nested `Name` object. That type change is breaking for a client expecting a string. The sample does not persist people; it demonstrates representation selection.

**Source excerpt:**

```java
@GetMapping(path = "/person/header", headers = "X-API-VERSION=1")
public PersonV1 getFirstVersionOfPersonRequestHeader() {
    return new PersonV1("Bob Charlie");
}

@GetMapping(path = "/person/accept", produces = "application/vnd.company.app-v2+json")
public PersonV2 getSecondVersionOfPersonAcceptHeader() {
    return new PersonV2(new Name("Bob", "Charlie"));
}
```

Use an explicit `Accept` vendor type to select the media-type version. `Accept: */*` does not express a reliable preference between these versions. An unsupported representation can produce `406 Not Acceptable` after authentication. Missing custom version headers may leave no matching handler rather than select v1 automatically.

### 11. HATEOAS, HAL Explorer, Swagger, and Actuator
**Source excerpt from `user/UserResource.java`:**

```java
EntityModel<User> entityModel = EntityModel.of(user);

WebMvcLinkBuilder link =  linkTo(methodOn(this.getClass()).retrieveAllUsers());
entityModel.add(link.withRel("all-users"));

return entityModel;
```

`EntityModel` combines representation data with links. `methodOn` creates a controller invocation description; it does not execute the real service method. `linkTo` uses mapping metadata to construct a URI. The same pattern in the JPA controller links back to `/jpa/users`.

```json
{
  "id": 1,
  "name": "Adam",
  "birthDate": "1996-10-31",
  "_links": {
    "all-users": {"href": "http://localhost:8080/users"}
  }
}
```

This is an illustrative HAL-shaped response; the date changes with initialization time. Prefer `Accept: application/hal+json` when explicitly exploring HAL. The HAL Explorer dependency serves an exploration UI, commonly at `/` or `/explorer/index.html`. Security and UI asset access still matter. It does **not** turn every Spring Data JPA repository into an automatically exported REST API.

SpringDoc would ordinarily provide `/v3/api-docs` and `/swagger-ui.html` when enabled. Those routes are **not available merely because their dependency appears inside a POM comment**. OpenAPI is a machine-readable contract; Swagger UI is an interactive viewer/client for that contract. HAL Explorer follows hypermedia; it is not a substitute for schema documentation.

Actuator is active, with broad web exposure requested in properties. `/actuator/health` provides health information; `/actuator` discovers exposed endpoints. Only enabled/available endpoints are exposed; `*` does not create nonexistent capabilities. Boot defaults normally permit health access while protecting other actuator routes. Production should limit exposure and carefully configure authorization and health detail disclosure.

### 12. Security configuration: source versus final lesson
File: `security/SpringSecurityConfiguration.java`.

**Source excerpt:**

```java
@Configuration
public class SpringSecurityConfiguration {
//  @Bean
//  public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
```

Every filter-chain line is commented out in the present file. An empty `@Configuration` class does not replace Boot's default security. The final course snapshot demonstrates authenticated requests, Basic auth, and disabled CSRF. Do not report that snapshot's filter chain as currently active.

A Spring 6/Boot 3-style **illustrative** explicit REST chain would use the lambda DSL:

```java
http.authorizeHttpRequests(auth -> auth.anyRequest().authenticated());
http.httpBasic(Customizer.withDefaults());
http.csrf(AbstractHttpConfigurer::disable);
return http.build();
```

Disabling CSRF requires a considered threat model; Basic credentials may be browser-attached. Do not infer CSRF safety from calling an endpoint a REST API. Use `jakarta.persistence` and `jakarta.validation`, not legacy `javax` imports, with Boot 3.

### 13. Endpoint inventory
Statuses below separate implemented controller outcomes from useful design conventions. Protected GETs without authentication generally return 401 for a curl/API request. Protected writes may return 403 for missing CSRF before reaching MVC. Validation/binding failures occur only after the security layer allows the request.

| Method | URL / selection | Description | Normal MVC result / relevant errors |
|---|---|---|---|
| GET | `/hello-world` | Plain greeting | 200; security 401 |
| GET | `/hello-world-bean` | JSON greeting bean | 200; 406 if no converter can satisfy Accept |
| GET | `/hello-world/path-variable/{name}` | Personalized greeting | 200 |
| GET | `/hello-world-internationalized` | MessageSource greeting | 200 |
| GET | `/users` | Static-list users | 200, including empty list |
| GET | `/users/{id}` | Static-list user with link | 200; 404 absent; 400 bad numeric ID |
| POST | `/users` | Validate and create static-list user | 201 + Location; 400 invalid; 403 CSRF |
| DELETE | `/users/{id}` | Remove from static list | 200 empty body, also for absent ID; 403 CSRF |
| GET | `/jpa/users` | Database users | 200 |
| GET | `/jpa/users/{id}` | Database user with link | 200; 404 absent |
| POST | `/jpa/users` | Validate and save user | 201 + Location; 400 invalid |
| DELETE | `/jpa/users/{id}` | Repository deletion | 200 empty body; possible 500 FK conflict |
| GET | `/jpa/users/{id}/posts` | User's posts | 200 collection; 404 absent parent |
| POST | `/jpa/users/{id}/posts` | Validate and save associated post | 201 + Location; 400 invalid; 404 parent absent |
| GET | `/filtering` | Dynamic whitelist of two fields | 200 |
| GET | `/filtering-list` | Different whitelist for a list | 200 |
| GET | `/v1/person` | URI version 1 | 200 |
| GET | `/v2/person` | URI version 2 | 200 |
| GET | `/person?version=1` | Parameter version 1 | 200 |
| GET | `/person?version=2` | Parameter version 2 | 200 |
| GET | `/person/header`, `X-API-VERSION: 1` | Header version 1 | 200 |
| GET | `/person/header`, `X-API-VERSION: 2` | Header version 2 | 200 |
| GET | `/person/accept`, vendor v1 Accept | Media-type version 1 | 200; 406 unsupported Accept |
| GET | `/person/accept`, vendor v2 Accept | Media-type version 2 | 200; 406 unsupported Accept |
| GET | `/actuator`, `/actuator/health` | Operational discovery and health | 200 with applicable access; health can be 503 |
| GET | `/h2-console` | Database UI entry, not business REST | Redirect/UI; security and frames may block |
| GET | `/`, `/explorer/index.html` | HAL Explorer UI entry | UI/redirect depending routing and access |
| GET | `/v3/api-docs`, `/swagger-ui.html` | Optional SpringDoc routes | Not registered by current POM |

No PUT or PATCH user endpoint is implemented. No individual-post GET handler exists despite its creation Location. Unsupported methods on existing mappings normally yield 405 when security permits MVC processing.

## Core concepts explained
### REST principles and constraints
REST means **Representational State Transfer**. It is an architectural style, not a Java library and not a synonym for JSON over HTTP. A resource is a conceptual thing; a representation is the transferable view of it. `/users/1` identifies a user; its JSON and HAL forms are representations of that resource.

| Constraint | Beginner explanation | Consequence for API design |
|---|---|---|
| Client–server | UI and storage responsibilities are separated | React or curl can use the same server |
| Stateless | Each request carries needed request context/authentication | Do not depend on earlier requests to interpret a call |
| Cacheable | Responses indicate whether/how they can be reused | Set suitable cache metadata, especially for private data |
| Uniform interface | Use standard resource and representation semantics | Predictable methods, identifiers, and links |
| Layered system | Clients need not know intermediaries | Gateways, proxies, load balancers can intervene |
| Code on demand, optional | Server may send executable code to a client | Not required for this JSON API |

The uniform interface includes resource identification, manipulation through representations, self-descriptive messages, and hypermedia as the engine of application state. Strict REST is more demanding than simply having plural nouns in URLs. Server-side database state does not violate statelessness; server-side conversational client state can. Boot's current default session/CSRF behavior means this sample is not a pure stateless security setup.

### HTTP methods: safety, idempotency, and intent
**Safe** means the client does not request a change to resource state. **Idempotent** means repeating the same request has the same intended effect as sending it once. Response codes, timestamps, access logs, and counters can differ without changing that definition.

| Method | Safe? | Idempotent by HTTP semantics? | Typical purpose | Example |
|---|---|---|---|---|
| GET | Yes | Yes | Retrieve representation | `GET /users/1` |
| HEAD | Yes | Yes | Retrieve corresponding metadata without body | `HEAD /users/1` |
| OPTIONS | Yes | Yes | Describe communication options | CORS preflight uses OPTIONS |
| POST | No | No, normally | Create subordinate resource or process command | `POST /users` |
| PUT | No | Yes | Replace/create state at known target URI | Proposed `PUT /users/1` |
| PATCH | No | Not guaranteed | Apply partial modification | Proposed `PATCH /users/1` |
| DELETE | No | Yes | Remove target resource | `DELETE /users/1` |

A repeated DELETE can return 204 first and 404 later while remaining idempotent. A POST can be made operationally idempotent with a persisted idempotency-key mechanism. A PATCH assigning a fixed name can be idempotent; a PATCH incrementing a balance is not. A GET that sends email or deletes rows is an unsafe misuse of GET.

### Status codes and API contracts
| Code | Meaning | Use and caveat |
|---|---|---|
| 200 | OK | Successful read or update with body; current void DELETE also uses it |
| 201 | Created | Resource created; send Location; body is optional |
| 204 | No Content | Successful action without representation; must have no response body |
| 400 | Bad Request | Malformed JSON, invalid input, or conversion/validation failure |
| 401 | Unauthorized | Missing/invalid authentication; challenge commonly in WWW-Authenticate |
| 403 | Forbidden | Refused access; also missing/invalid CSRF in this sample |
| 404 | Not Found | Requested resource absent, or deliberate concealment policy |
| 409 | Conflict | State conflict, duplicate business key, or incompatible operation |
| 500 | Internal Server Error | Unexpected server failure, not a normal business validation result |

Also know 405 for an unsupported method, 406 for unacceptable response representation, 415 for unsupported request media type, 412 for failed conditional preconditions, and 429 for rate limits. The module does not currently map a database integrity conflict to 409. Its generic advice can instead yield 500, which is a useful improvement discussion. Do not turn every exception into 200 plus an `error` field.

### Request binding and the MVC conversion pipeline
```text
Content-Type: application/json
           |
raw body -> Jackson converter -> User instance -> @Valid constraints
                                                   |
                       invalid --------------------+---- valid
                         |                               |
              MethodArgumentNotValidException       controller
                         |                               |
                    advice -> 400                    service/save
```

`Content-Type` tells the server what the **request body** is. `Accept` tells the server which **response representation** the client can consume. `@PathVariable` binds URI template values; `@RequestParam` binds query/form parameters. `@RequestBody` binds through converters; it does not pull fields from the query string. `@RestController` combines `@Controller` with response-body behavior. A plain string returned from it is a response body, not a view template name.

Constraints and presence are different questions:

| Constraint | Current field | Important nuance |
|---|---|---|
| `@Size(min=2)` | User name | Null is valid unless combined with NotNull/NotBlank |
| `@Past` | User birth date | Today is invalid; null is valid unless presence is constrained |
| `@Size(min=10)` | Post description | Null is valid; short present strings are invalid |
| `@Valid` | User/post request argument | Triggers traversal; does not invent required-field rules |

Use `@NotBlank` for a mandatory non-whitespace string and `@NotNull` for required dates. Use separate request DTOs when creation and updates require different rules. Bean Validation can also execute during JPA lifecycle operations; request validation is not the only layer. Direct SQL seed execution is outside those Java validation hooks.

### ResponseEntity and Location construction
`ResponseEntity<T>` controls status, headers, and optional body together. Direct bean returns are convenient for straightforward 200 responses. Creation benefits from an explicit response because it needs a status and discovery header.

```text
POST http://localhost:8080/users
          |
    assign id = 4
          |
fromCurrentRequest()        -> http://localhost:8080/users
.path("/{id}")             -> http://localhost:8080/users/{id}
.buildAndExpand(4).toUri()  -> http://localhost:8080/users/4
          |
HTTP/1.1 201 Created
Location: http://localhost:8080/users/4
```

Reverse proxies may change scheme, host, port, or path prefixes. Configure trusted forwarded headers appropriately rather than hard-coding deployment hosts. A Location should ideally identify a resource the client can subsequently retrieve. A valid absolute URI alone does not guarantee a implemented GET contract.

### Exception handling strategies
| Strategy | Where it lives | Strength | Limitation |
|---|---|---|---|
| `@ResponseStatus` exception | Exception class | Simple status mapping | Couples exception to HTTP; limited custom body control |
| `ResponseStatusException` | Throw site | Dynamic status/reason | Scattered HTTP policy can grow inconsistent |
| Local `@ExceptionHandler` | Controller | Feature-specific error policy | Duplicates cross-cutting logic |
| `@ControllerAdvice` + handlers | Shared advice bean | Uniform errors across controllers | Does not cover every servlet filter error |
| `ResponseEntityExceptionHandler` | Advice superclass | Built-in MVC exception hooks | Override the Spring-version-correct signatures |
| Security entry point/denied handler | Security filter chain | Consistent 401/403 bodies | Separate from controller advice |

`@RestControllerAdvice` includes response-body semantics on top of controller advice. Here, `@ControllerAdvice` works because handlers return `ResponseEntity`. Prefer a stable code, safe message, trace ID, and structured field errors for production consumers. Spring 6 offers `ProblemDetail` and RFC 9457-style problem responses. This sample uses custom `ErrorDetails`; it does not currently return ProblemDetail for its custom errors. Log stack traces server-side while returning only safe client-facing explanations.

### Versioning pros and cons
| Strategy | Repo example | Advantages | Disadvantages / operational work |
|---|---|---|---|
| URI | `/v1/person` | Obvious, easy curl/browser testing, simple cache separation | Version enters URI identity; links and routes multiply |
| Query parameter | `/person?version=1` | Visible, simple incremental routing | Cache/CDN query-key policy must include version |
| Custom header | `X-API-VERSION: 1` | Clean resource URI | Less browser-visible; document header; cache must vary appropriately |
| Media type/Accept | `application/vnd.company.app-v1+json` | Models representation evolution; standard Accept mechanism | More complex negotiation/tooling; use Vary: Accept |

There is no universally best versioning scheme. Choose one consistently based on client ecosystem, gateway behavior, and contract governance. Adding an optional response property is often backward-compatible for tolerant consumers. Changing `name` from string to object is not backward-compatible. Plan deprecation timelines, migration documentation, and usage measurement before removing a version. Ensure hypermedia links preserve or negotiate the intended version context. Header/query versioning needs cache correctness, not just controller mappings.

### HATEOAS and the Richardson maturity model
HATEOAS means **Hypermedia As The Engine Of Application State**. A client discovers related resources or allowed next actions through representation links. Links supplement—not replace—authentication, authorization, and domain invariants.

```text
GET /users/1
   |
   +-- user properties
   +-- rel="all-users" --> GET /users

Client follows link relation instead of duplicating every route-construction rule.
```

| Richardson level | What is added | Example |
|---|---|---|
| 0 | HTTP as a transport tunnel | POST everything to `/service` with action names |
| 1 | Separate resources | `/users`, `/users/1`, but mostly one method |
| 2 | HTTP methods and statuses | GET, POST, DELETE, 201, 404 |
| 3 | Hypermedia controls | Links describing navigation and next actions |

The sample contains level-2 practices plus illustrative level-3 links. Do not claim a single `all-users` link proves a complete hypermedia-driven API. HAL commonly represents links under `_links` and embedded resources under `_embedded`. `EntityModel` holds one resource; `CollectionModel` and `PagedModel` serve broader cases. The current collection methods return plain lists, not fully linked paged models.

### REST versus SOAP
| Dimension | REST | SOAP |
|---|---|---|
| Nature | Architectural style | Message protocol/specification family |
| Representation | JSON, XML, text, HAL, and others | XML SOAP envelope |
| Interface emphasis | Resources and uniform semantics | Defined operations/messages |
| Contract | Often OpenAPI plus conventions | Often WSDL/XSD |
| HTTP usage | Uses methods/status/cache semantics when implemented well | Commonly POST over HTTP; other transports possible |
| Tool ecosystem | Browser, curl, HTTP clients, OpenAPI | Generated proxies, WS-* tooling |
| Security | TLS and application auth schemes | TLS and potentially WS-Security message features |
| Good fit | Web/mobile integration and resource-oriented APIs | Existing enterprise contracts and message-level requirements |

REST is not inherently insecure and SOAP is not inherently secure. REST need not use JSON; SOAP over HTTP does not become REST by using port 8080. Compare actual contract and operational needs rather than choosing based on slogans.

### PUT versus PATCH versus POST
| Concern | POST | PUT | PATCH |
|---|---|---|---|
| Target | Collection or command endpoint | Known resource URI | Known resource URI |
| Main intent | Create/process | Replace current representation | Apply partial change |
| Identifier choice | Often server-assigned | Usually client already knows URI | Already identifies target |
| Idempotency | Not guaranteed | Required by method semantics | Depends on patch operation |
| Missing fields | Contract-specific | Replacement contract must define their effect | Usually unchanged unless explicitly removed |
| Typical media type | `application/json` | `application/json` | Merge Patch or JSON Patch media type |

JSON Merge Patch uses object-shaped changes and gives null special removal semantics. JSON Patch uses operations such as `add`, `replace`, and `remove` at JSON Pointer paths. An arbitrary partial JSON payload is not automatically compliant with either patch format. Validate the resulting state as well as the supplied modifications. Use ETag/If-Match or optimistic locking to avoid silently overwriting concurrent edits. These update/patch features are conceptual here: no User PUT/PATCH handlers exist.

### Jackson JSON serialization and contract design
Jackson serializes bean properties and deserializes JSON through constructors/accessors/fields it supports. Boot's configured mapper includes useful modules such as Java time support. `LocalDate` typically appears as an ISO date string such as `"1990-04-12"`. `LocalDateTime` has no time-zone/offset semantics; do not label it automatically as UTC.

| Annotation/tool | Purpose | Repo state |
|---|---|---|
| `@JsonIgnore` | Exclude property from JSON binding/serialization | Active on relationships |
| `@JsonIgnoreProperties` | Exclude named properties at class level | Commented SomeBean example |
| `@JsonProperty` | Explicit external property name | Commented User examples |
| `@JsonFilter` | Named dynamic serialization filter | Active on SomeBean |
| `MappingJacksonValue` | Per-response Jackson metadata wrapper | Active in FilteringController |
| DTO | Separate persistence from external representation | Recommended, not current architecture |

Do not accept server-owned ID, owner, or authorization attributes without explicit policy. Returning entities directly can expose schema changes and cause lazy-loading surprises. Omitting secrets in one serializer path is not a substitute for authorization and safe logging. Filtering controls representation fields, not access to a resource as a whole.

### i18n and content negotiation are different
```text
Accept: application/json              -> choose representation format
Accept-Language: nl                   -> choose localization preference
Content-Type: application/json        -> parse request body format
Authorization: Basic ...             -> identify authenticated caller
```

Boot's message source resolves `good.morning.message` against the selected locale. The default message parameter is used if that message code cannot be resolved. Locale fallback and missing-code fallback are related but not identical mechanisms. For cacheable localized content, make language variation part of cache behavior. Do not translate JSON property names between locales; keep machine contracts stable. If enabling XML in a study copy, add `jackson-dataformat-xml` and test with XML Accept/Content-Type. The current active classpath does not include that converter dependency.

### Best practices checklist
- Name resources consistently; use nouns such as `/users` and nested collections sensibly.
- Honor HTTP safety/idempotency instead of tunneling all behavior through POST.
- Use 201 plus a usable Location for creation and explicit 204 for bodyless success where appropriate.
- Validate server-side even when a frontend already validates.
- Define required fields separately from length/range constraints.
- Use request/response DTOs to avoid exposing persistence and server-owned fields.
- Centralize safe, stable error contracts and distinguish business failures from crashes.
- Scope data queries and mutation checks to the authenticated principal.
- Add paging, sorting, and bounds to collections rather than unbounded `findAll()`.
- Use service transactions for multi-repository invariants.
- Consider optimistic locking and conditional requests for concurrent updates.
- Configure cache headers correctly for private and versioned representations.
- Use TLS and carefully chosen authentication/CSRF/session policies.
- Restrict Actuator and H2-console exposure outside local development.
- Maintain OpenAPI documentation and endpoint/security integration tests together.

## Common pitfalls / gotchas
### Runtime versus snapshot confusion
1. Swagger and XML dependencies are commented out; the lessons alone do not enable them.
2. The current Security configuration is empty; the Step37 snapshot is not the active chain.
3. `/users/1` and `/jpa/users/10001` demonstrate different storage systems.
4. Post creation emits a Location with no individual-post GET implementation.
5. Plain `void` DELETE methods do not automatically produce 204.
6. Entity IDs assigned by JPA are not guaranteed to continue from the manually seeded high IDs.
7. H2 rows and static-list rows disappear on application restart.
8. Boot defaults may reject writes before validation or exception advice can run.

### Validation and error handling traps
- `@Size` permits null; it does not mean a field is mandatory.
- `@Past` rejects today, even though SQL seeds use today.
- A malformed date fails JSON conversion before ordinary field validation.
- The first field error is not a stable, complete ordering of every validation problem.
- A generic 500 handler does not replace deliberate 409 mapping for business conflicts.
- `ex.getMessage()` can leak internal implementation information.
- MVC advice does not automatically normalize security-filter authentication errors.
- Validation imports must be `jakarta.validation` in Boot 3.

### Persistence and serialization traps
- `mappedBy` names a Java relationship field, not a table or column.
- Lazy loading can issue queries during controller processing or serialization.
- Disabling Open EntityManager in View exposes accidental out-of-transaction access.
- Adding eager relationships indiscriminately is not a general fix for lazy-loading problems.
- Bidirectional entities can recurse in JSON unless carefully shaped.
- No cascade on user deletion means posts can prevent deleting their parent.
- A static `ArrayList` and `++usersCount` are not concurrency-safe persistence.
- Class-level static filters and request-specific dynamic filters solve different problems.

### HTTP, security, and deployment traps
- 401 is about authentication; 403 is about refusal and can include CSRF rejection.
- Encoding Basic credentials is not encrypting them; always use HTTPS outside local exercises.
- `Accept` does not describe the submitted JSON body; `Content-Type` does.
- Hard-coded URI hosts break behind proxies and alternate deployments.
- A DELETE can be idempotent even when the second response differs.
- A browser API call and a curl call can differ in credentials, headers, and CSRF state.
- Broad Actuator exposure must not become a production copy-and-paste default.
- Versioning headers require compatible caching and client tooling, not just mappings.

## Interview questions & answers
### Basic
#### 1. What is REST?
An architectural style built around resources, representations, and constraints including statelessness and a uniform interface. It is not restricted to JSON and is not a protocol like SOAP.

#### 2. What does `@RestController` do?
It marks a Spring MVC controller and applies response-body semantics to handler return values. A returned bean is written through a message converter rather than resolved as a view name.

#### 3. What is the difference between a resource and a representation?
A resource is the conceptual user identified by `/users/1`. JSON or HAL is a transferable representation of that user.

#### 4. How does a Java bean become JSON here?
The Jackson HTTP message converter serializes visible properties such as `HelloWorldBean.getMessage()`. It does not call `toString()` to build JSON.

#### 5. What does `@RequestBody` do?
It reads the request body through an appropriate HTTP message converter and binds it to an object. JSON syntax/type errors can fail before the controller method runs.

#### 6. What is the difference between `@PathVariable` and `@RequestParam`?
A path variable comes from a URI template segment such as `/users/{id}`. A request parameter comes from query/form parameters such as `?version=1`.

#### 7. Why return 201 for creation?
It tells the client that a new resource was created. A Location header identifies that resource; a response body is optional.

#### 8. Is GET idempotent and safe?
Yes under HTTP semantics: it requests retrieval rather than modification. Incidental logging does not turn a properly designed GET into a requested state-changing operation.

#### 9. Are DELETE requests idempotent?
Yes: deleting the same target repeatedly leaves it absent. The first and second response statuses do not need to be identical.

#### 10. How do 401 and 403 differ?
401 means authentication is missing or unacceptable; 403 means the request is refused. In this module, a write with valid Basic credentials can still receive 403 for missing CSRF.

#### 11. What does `@Valid` guarantee?
It triggers the declared validation constraints on the bound request object. It does not make unconstrained fields mandatory or replace domain/business validation.

#### 12. Is the static user service a database?
No: it holds an in-memory static list and counter. The JPA endpoints use separate repository-backed H2 rows.

### Intermediate
#### 13. What are the parts of ResponseEntity?
HTTP status, headers, and an optional response body. The sample uses it to emit 201 plus Location while deliberately building an empty body.

#### 14. How is Location built without a hard-coded host?
`ServletUriComponentsBuilder.fromCurrentRequest()` starts from the request URI and appends a URI template. `buildAndExpand(savedUser.getId())` substitutes the created ID.

#### 15. How does the application return a custom 404 body?
It throws `UserNotFoundException`, and a matching global advice handler returns `ErrorDetails` with NOT_FOUND. The exception's `@ResponseStatus` also defines a fallback status policy.

#### 16. Why extend ResponseEntityExceptionHandler?
It supplies hooks for Spring MVC exceptions such as validation and unreadable message errors. Overrides must match Spring 6 signatures, including HttpStatusCode where used.

#### 17. How do static and dynamic filtering differ?
Static Jackson annotations exclude fixed properties across uses of a class. The dynamic SomeBean filter supplies a per-response whitelist through MappingJacksonValue.

#### 18. What happens if a JsonFilter provider is missing?
Serialization can fail because Jackson cannot resolve the named filter. The `@JsonFilter` name and registered provider ID must match.

#### 19. Why is User.posts marked mappedBy="user"?
The association is inverse; `Post.user` owns the foreign key. The string identifies a Java field on Post, not the SQL column name.

#### 20. What problem does JsonIgnore solve on relationships?
It hides relationship properties from the JSON contract and prevents recursive serialization. It does not configure SQL fetching or authorization.

#### 21. How is an unsupported locale handled?
Message resolution falls back through available bundles/configured fallback behavior. Only the default and Dutch messages exist here; French is not automatically translated.

#### 22. What is the difference between Accept and Content-Type?
Accept expresses desired response media types; Content-Type identifies the submitted body media type. `Accept-Language` is a separate localization preference.

#### 23. How does the controller implement versioning?
It offers distinct URI paths, query-parameter conditions, custom-header conditions, and vendor media types. V1 has a string name; V2 has a structured Name, demonstrating a breaking representation change.

#### 24. What does methodOn do in link construction?
It supplies a proxy-based description of a controller method invocation. The actual retrieve-all business operation is not executed just to create the URI.

### Advanced and scenario-based
#### 25. A POST returns 403 instead of the expected validation 400. Why?
Security processing happens before MVC binding and validation. The current default chain requires CSRF for unsafe methods, even with Basic credentials.

#### 26. A POST succeeds but following Location returns 404. What would you inspect?
Check whether the server has a GET mapping for the emitted URI. Post creation here emits an individual-post Location but implements only the collection GET.

#### 27. Why can deleting a database user produce 500?
Existing posts can violate a foreign-key constraint because cascade deletion is not configured. The generic handler currently maps unexpected exceptions to 500; an explicit conflict policy would be better.

#### 28. A missing name passes User validation. Is Bean Validation broken?
No: @Size treats null as valid, and there is no @NotBlank on name. Mandatory presence and minimum length must both be declared if both are required.

#### 29. How would you prevent two clients from losing each other's updates?
Use optimistic locking, typically a JPA @Version field, and/or HTTP ETag with If-Match. Reject stale writes rather than silently applying last-writer-wins replacement.

#### 30. Does EntityModel automatically make the entire API Richardson level 3?
No: it introduces hypermedia links for particular responses. A mature hypermedia API consistently exposes meaningful controls and expects clients to follow them.

#### 31. Why can header versioning return the wrong cached representation?
A cache may key only on the URI and ignore the version-selection header. Configure Vary/cache keys and intermediary behavior to include the relevant selector.

#### 32. Should a public REST API return entities directly?
It can for learning, but DTOs give a more stable contract and avoid accidental relationship/schema exposure. Map under an intentional transaction boundary and constrain server-owned fields.

#### 33. Is disabling CSRF always correct for REST?
No: CSRF risk depends on whether browsers automatically attach authentication credentials. Explicit bearer headers and cookie-based authentication have different threat models.

#### 34. What is a production alternative to this custom error shape?
Spring 6 ProblemDetail supports standardized problem responses with extensions for codes and trace IDs. Keep field errors structured and avoid leaking stack traces or internal exception text.

#### 35. How would you design safe automatic POST retries?
Accept an idempotency key, persist its request/result association, and handle concurrent duplicate submissions atomically. Retry policy alone cannot make an ordinary creation POST idempotent.

#### 36. What should be tested beyond contextLoads?
Status, headers, payload shape, validation, absence, auth/CSRF behavior, association ownership, and persistence boundaries. MockMvc tests plus repository/integration tests cover much more than successful application bootstrap.

## Hands-on exercises
These tasks describe work for a learner's study copy; no source changes are made by this guide. Start with the local-only startup credentials documented above. Use `-i` so status and headers stay visible rather than judging only the JSON body.

### Exercise 1 — Trace greetings, links, and localization
```bash
BASE=http://localhost:8080
curl -i -u study:study-only "$BASE/hello-world-bean"
curl -i -u study:study-only "$BASE/hello-world/path-variable/Ada"
curl -i -u study:study-only -H 'Accept-Language: nl'   "$BASE/hello-world-internationalized"
curl -i -u study:study-only -H 'Accept: application/hal+json' "$BASE/users/1"
curl -i -u study:study-only "$BASE/users/999999"
```

Expected learning: bean conversion, URI binding, `Goedemorgen`, an `all-users` link, and custom 404. Repeat `/jpa/users/10001` to prove that the database user is not the static-list user.

### Exercise 2 — Obtain CSRF, create a user, and inspect Location
Boot's default login page publishes a CSRF form token tied to its session. This shell-only example keeps both values in variables; it creates no cookie file. It assumes Boot's generated login page and JSESSIONID cookie remain enabled as in the present source.

```bash
LOGIN_PAGE="$(curl -sS -i "$BASE/login")"
SESSION_COOKIE="$(printf '%s' "$LOGIN_PAGE" | python3 -c 'import re,sys; s=sys.stdin.read(); print(re.search(r"(?im)^set-cookie: (JSESSIONID=[^;\r\n]+)",s).group(1))')"
CSRF_TOKEN="$(printf '%s' "$LOGIN_PAGE" | python3 -c 'import re,sys; s=sys.stdin.read(); print(re.search(r"name=\"_csrf\"[^>]*value=\"([^\"]+)\"",s).group(1))')"

curl -i -u study:study-only   -H "Cookie: $SESSION_COOKIE" -H "X-CSRF-TOKEN: $CSRF_TOKEN"   -H 'Content-Type: application/json'   -d '{"name":"Ada Lovelace","birthDate":"1815-12-10"}'   "$BASE/users"
```

Expect 201 and a Location identifying the newly allocated static-list ID. Read that exact Location with an authenticated GET; do not assume a fixed generated ID. Repeat with `"name":"A"` to exercise the validation handler and expect 400. Remove the CSRF header to contrast Security's 403 with MVC's validation 400. A login-page/session configuration change would require a different CSRF acquisition mechanism.

### Exercise 3 — Compare filtering and versioned representations
```bash
curl -sS -u study:study-only "$BASE/filtering"
curl -sS -u study:study-only "$BASE/filtering-list"
curl -sS -u study:study-only "$BASE/v1/person"
curl -sS -u study:study-only "$BASE/person?version=2"
curl -sS -u study:study-only -H 'X-API-VERSION: 1' "$BASE/person/header"
curl -i -u study:study-only   -H 'Accept: application/vnd.company.app-v2+json' "$BASE/person/accept"
curl -i -u study:study-only -H 'Accept: application/pdf' "$BASE/person/accept"
```

Record the exact exposed fields and string-versus-object `name` types. Explain why an explicit vendor Accept selects a version and unsupported Accept can yield 406. Explain why static annotations in SomeBean comments do not determine its current output.

### Exercise 4 — Work with posts and repair the Location contract in a study copy
```bash
curl -sS -u study:study-only "$BASE/jpa/users/10001/posts"
curl -i -u study:study-only   -H "Cookie: $SESSION_COOKIE" -H "X-CSRF-TOKEN: $CSRF_TOKEN"   -H 'Content-Type: application/json'   -d '{"description":"Practice Spring REST relationships"}'   "$BASE/jpa/users/10001/posts"
```

Expected: 201; inspect the collection again to find the newly created post. In a separate study copy, add GET `/jpa/users/{userId}/posts/{postId}`. Verify both post existence and association with the requested user. Add tests for absent parent, absent post, wrong parent, and valid creation Location. Do not merely fetch by post ID and ignore the parent portion of the URI.

### Exercise 5 — Turn sample behavior into a documented contract
In a study copy, return explicit 204 for successful DELETE and decide the absent-resource policy. Add required-field constraints to request DTOs and structured validation errors. Map an intentional user-with-posts conflict to 409 rather than generic 500. Enable SpringDoc/XML only if you intentionally add their commented dependencies.

```bash
curl -i -u study:study-only   -H "Cookie: $SESSION_COOKIE" -H "X-CSRF-TOKEN: $CSRF_TOKEN"   -X DELETE "$BASE/users/999999"
curl -i -u study:study-only -H 'Accept: application/xml' "$BASE/users"
curl -i -u study:study-only "$BASE/v3/api-docs"
```

Baseline expectation: DELETE is 200; XML is unavailable without its converter; SpringDoc is not registered. After intentional changes, verify the new contract rather than silently assuming the lesson snapshots apply.

## Quick revision cheat sheet
| Remember | Interview-ready summary |
|---|---|
| REST | Style with client–server, stateless, cacheable, uniform, layered constraints |
| Safe | No requested state change: GET, HEAD, OPTIONS |
| Idempotent | Repetition has same intended effect: GET, HEAD, OPTIONS, PUT, DELETE |
| POST | Normally non-idempotent creation/processing |
| PATCH | Partial changes; idempotency depends on operation |
| 201 | Created; Location; optional body |
| 204 | Success with no body; not automatic for void methods |
| 400 / 401 / 403 | Invalid request / authentication issue / refused access or CSRF |
| 404 / 409 / 500 | Missing resource / state conflict / unexpected server failure |
| RequestBody | Message-converter body binding |
| Valid | Executes constraints; does not imply NotNull/NotBlank |
| ResponseEntity | Status + headers + body |
| Advice | Shared MVC errors; not every Security filter error |
| Spring 6 errors | HttpStatusCode override signatures; ProblemDetail available |
| Jackson | Bean properties, Java-time modules, configurable inclusion |
| Static filter | JsonIgnore or JsonIgnoreProperties |
| Dynamic filter | JsonFilter + FilterProvider + MappingJacksonValue |
| i18n | Accept-Language → locale → MessageSource bundle |
| Negotiation | Accept selects output; Content-Type describes input |
| Versioning | URI, parameter, header, vendor Accept; define cache behavior |
| HATEOAS | Data plus discoverable link relations |
| Richardson | 0 transport; 1 resources; 2 HTTP semantics; 3 hypermedia |
| Persistence | JPA specification → Hibernate provider → H2 database |
| Ownership | Post.user owns FK; User.posts is mappedBy inverse |
| Actual stores | /users static IDs 1–3; /jpa/users seeded IDs 10001–10003 |
| Actual security | Empty custom config; Boot default auth and CSRF active |
| Actual optional tools | Swagger/XML commented out; Actuator/HAL Explorer active |
| Contract gap | Created individual-post Location has no corresponding GET |

Final interview habit: explain **what the code does now**, **why it does it**, and **what you would improve** as three separate claims.
