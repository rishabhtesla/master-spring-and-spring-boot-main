# 06 — Full-Stack React, Spring REST, and JWT: Interview Study Guide
## What this module teaches
**Exact module:** `13-full-stack/`. **Backend:** `13-full-stack/02-rest-api/`. **Frontend:** `13-full-stack/01-frontend/todo-app/`. The backend POM declares **Spring Boot 3.3.4** and **Java 21**. The frontend uses **React 18**, **React Router 6**, **axios**, **Formik**, and **Moment**.

This course module connects a browser-based todo application to authenticated Spring REST endpoints. It progresses from component/state exercises to API calls, Basic authentication, JWT, and JPA. The checked-in final source uses JWT authentication while retaining an inactive Basic-only configuration. It intentionally remains a teaching application rather than a production security blueprint.

### How to run: two independent processes
From the repository root, in terminal 1:
```bash
java -version
mvn -version
cd 13-full-stack/02-rest-api
mvn spring-boot:run
```

Use an installed Maven and Java 21 JDK; this module has no checked-in Maven wrapper. The server listens on `http://localhost:8080` unless explicitly overridden. Do not run the `12-rest-api` application simultaneously on the same port. H2 is in-memory; no MySQL instance is required for the current configuration.

In terminal 2, again starting from the repository root:
```bash
cd 13-full-stack/01-frontend/todo-app
node --version
npm --version
npm ci
npm start
```

`npm ci` installs from the existing lockfile; run it when dependencies need installation. This guide documents learner commands; its creation does not install or build the project. Use a Node version compatible with the checked-in Create React App 5 toolchain. The development UI is `http://localhost:3000`. Do not substitute `http://127.0.0.1:3000`: the backend's origin allowlist names `localhost` exactly.

The demonstration login is:
```text
username: in28minutes
password: dummy
```

Those values come from the actual in-memory UserDetailsService. `{noop}` marks its password as unencoded and is strictly a classroom choice. Never use this credential, plaintext password storage, or HTTP transport for a deployed application.

Backend-only smoke requests:
```bash
BASE=http://localhost:8080
curl -i -H 'Content-Type: application/json'   -d '{"username":"in28minutes","password":"dummy"}'   "$BASE/authenticate"
curl -i -u in28minutes:dummy "$BASE/users/in28minutes/todos"
```

The second call works because the active JWT chain also enables Basic authentication. The browser's active login path uses `/authenticate` and subsequent Bearer headers instead.

Optional learner validation commands:
```bash
# From 13-full-stack/02-rest-api/
mvn test
# From 13-full-stack/01-frontend/todo-app/
npm run build
CI=true npm test -- --watchAll=false
```

The backend test only checks context startup. `src/App.test.js` is an old Create React App test looking for a "learn react" link absent from the todo UI. Do not claim the existing frontend test already proves the authentication flow works.

## Project walkthrough
### 1. Architectural and file map
```text
Browser at localhost:3000
  React components + React Router + AuthContext
                 |
                 | axios JSON HTTP requests
                 | Authorization: Bearer <JWT>
                 v
Spring application at localhost:8080
  SecurityFilterChain -> DispatcherServlet -> REST controller
                 |
                 | TodoJpaResource
                 v
Spring Data JPA TodoRepository
                 |
                 v
Hibernate ORM -> JDBC -> H2 in-memory database testdb
```

The requested high-level architecture is:
```text
React  -->  REST/HTTP  -->  Spring  -->  JPA/Hibernate  -->  DB
 UI          contract      server       persistence       data
```

The final todo controller calls its repository directly. `TodoService` is still injectable but is no longer the active persistence path. Do not draw a mandatory TodoService step for every final-source request.

Backend Java root:
```text
13-full-stack/02-rest-api/src/main/java/
  com/in28minutes/rest/webservices/restfulwebservices/
```

Frontend component root:
```text
13-full-stack/01-frontend/todo-app/src/components/
```

Backend short paths below are relative to the backend Java root. Frontend paths beginning `todo/` or `counter/` are relative to the component root. All **source excerpts** are copied from actual source, with normalized indentation. Any code labeled **illustrative improvement** is not an existing implementation.

### 2. Backend dependencies and resource configuration
File: `13-full-stack/02-rest-api/pom.xml`.

| Dependency | Responsibility |
|---|---|
| `spring-boot-starter-web` | MVC controllers, embedded server, JSON converters |
| `spring-boot-starter-data-jpa` | Repository support, Hibernate, relational persistence |
| `spring-boot-starter-security` | Authentication and authorization infrastructure |
| `spring-boot-starter-oauth2-resource-server` | Bearer/JWT resource-server processing |
| `spring-boot-configuration-processor` | Configuration metadata generation |
| `spring-boot-devtools` | Optional development restart support |
| `h2` | Runtime embedded database |
| `spring-boot-starter-test` | JUnit/Spring test support |

The final backend does **not** declare the validation starter used by `12-rest-api`. Its Todo request handlers have `@RequestBody` but not `@Valid`. Do not infer server-side description/date validation from Formik's client-side checks.

File: `13-full-stack/02-rest-api/src/main/resources/application.properties`.

**Source excerpt:**
```properties
logging.level.org.springframework=debug
spring.datasource.url=jdbc:h2:mem:testdb
spring.h2.console.enabled=true
spring.jpa.defer-datasource-initialization=true
spring.main.banner-mode=off
logging.pattern.console= %d{MM-dd HH:mm:ss} - %logger{36} - %msg%n
```

Security-user property examples in this file are commented out. The active JWT configuration supplies its own user instead of Boot's generated user. Debug logging and an enabled H2 console are development conveniences, not deployment defaults.

File: `13-full-stack/02-rest-api/src/main/resources/data.sql`.

**Source excerpt:**
```sql
insert into todo (id, description, done, target_date,username)
values (10001, 'Learn AWS', false, CURRENT_DATE(), 'in28minutes');
```

The file seeds IDs `10001`, `10002`, and `10003` for `in28minutes`. The other descriptions are `Get AWS Certified` and `Learn DevOps`. All seed dates are the database's current date and all `done` values are false. These differ from the static service's todo descriptions, IDs, and future dates.

File: `13-full-stack/02-rest-api/readme.md`. It describes an **optional MySQL transition**, including driver and datasource changes. The current POM/properties still use H2. For Boot 3.3, the modern MySQL driver coordinate is `com.mysql:mysql-connector-j`. Do not copy an old dialect name blindly; Hibernate can normally infer a supported dialect. External datasource migration is an exercise, not the final source's existing runtime requirement.

### 3. Bootstrap and actual CORS configuration
File: `RestfulWebServicesApplication.java`. The application class also contains a WebMvcConfigurer bean.

**Source excerpt:**
```java
@Bean
public WebMvcConfigurer corsConfigurer() {
    return new WebMvcConfigurer() {
        public void addCorsMappings(CorsRegistry registry) {
            registry.addMapping("/**")
                .allowedMethods("*")
                .allowedOrigins("http://localhost:3000");
        }
    };
}
```

This maps MVC CORS policy across controller paths. The allowlist names one exact browser origin, not arbitrary origins. Methods are broadly allowed for the lesson. `CorsRegistration` defaults also matter; allowed request headers default broadly here. No explicit `allowCredentials(true)` or exposed response-header list is configured. The active chain permits OPTIONS but does not explicitly call `http.cors(...)`.

For successful controller requests, MVC can apply its CORS handling. For a rejection occurring earlier in Security, the browser may not receive usable CORS headers. That can hide an underlying 401 as a browser CORS/network error. An explicit Security CORS integration is a production-oriented improvement discussed below.

### 4. Todo entity and Spring Data repository
Files: `todo/Todo.java`, `todo/repository/TodoRepository.java`.

**Source excerpt from the entity:**
```java
@Entity
public class Todo {
    @Id
    @GeneratedValue
    private Integer id;
    private String username;
    private String description;
    private LocalDate targetDate;
    private boolean done;
```

A public no-argument constructor and getters/setters support persistence and JSON binding. `isDone()` exposes JSON property `done`. `LocalDate` normally serializes as an ISO date string, not a JavaScript Date object. The database stores a `username` string, not a JPA relationship to an account entity. There is no `@Version` field or declared required-field validation on Todo.

**Source excerpt from the repository:**
```java
public interface TodoRepository extends JpaRepository<Todo, Integer>{
    List<Todo> findByUsername(String username);
}
```

Spring Data derives the username query from the method name. The plain derived method does not request case-insensitive matching. That differs from the old static service's `equalsIgnoreCase` predicate. A username parameter is a data filter, not proof of authorization.

### 5. Earlier in-memory service and inactive resource
Files: `todo/TodoService.java`, `todo/TodoResource.java`.

**Source excerpt:**
```java
@Service
public class TodoService {
    private static List<Todo> todos = new ArrayList<>();
    private static int todosCount = 0;
```

It initializes three future-dated todos with IDs 1–3. It supplies `findByUsername`, `addTodo`, `deleteById`, `findById`, and `updateTodo`. `findById()` calls `findFirst().get()`, so a missing entry is not handled gracefully. `updateTodo()` deletes and re-adds to the list. The static list and counter are neither durable nor concurrency-safe storage.

**Source excerpt from the old resource:**
```java
//@RestController
public class TodoResource {
```

The mappings in this class are inactive because its controller annotation is commented out. The service remains an active bean, but the final JPA controller uses the repository. Enabling both controllers with identical mappings would create an ambiguous mapping problem. The source is a record of the progression, not two simultaneous implementations of every route.

### 6. Active TodoJpaResource: inspect every operation
File: `todo/TodoJpaResource.java`.

**Source excerpt:**
```java
@GetMapping("/users/{username}/todos")
public List<Todo> retrieveTodos(@PathVariable String username) {
    return todoRepository.findByUsername(username);
}
```

The constructor injects both `TodoService` and `TodoRepository`. The commented service calls show the transition; the repository calls determine current behavior. The list endpoint filters by path username but does not compare it to the authenticated principal.

**Source excerpt:**
```java
@GetMapping("/users/{username}/todos/{id}")
public Todo retrieveTodo(@PathVariable String username,
        @PathVariable int id) {
    return todoRepository.findById(id).get();
}
```

The username is accepted but ignored in the lookup. An absent ID leads to `NoSuchElementException` and normally a server-error response, not a deliberate 404. A successful login alone therefore does not implement ownership authorization. This is a critical interview distinction between authenticating a caller and securing their records.

**Source excerpt:**
```java
@DeleteMapping("/users/{username}/todos/{id}")
public ResponseEntity<Void> deleteTodo(@PathVariable String username,
        @PathVariable int id) {
    todoRepository.deleteById(id);
    return ResponseEntity.noContent().build();
}
```

Deletion explicitly returns 204 and has no response body. Current repository deletion does not add a parent/owner check. Do not expect axios `response.data` to contain the deleted entity.

**Source excerpt:**
```java
@PutMapping("/users/{username}/todos/{id}")
public Todo updateTodo(@PathVariable String username,
        @PathVariable int id, @RequestBody Todo todo) {
    todoRepository.save(todo);
    return todo;
}
```

The body ID—not the path ID—is the value passed into save. The path username is not assigned or validated either. A null/new body ID can cause creation despite the update-looking route. A mismatched existing body ID can target a different row. These are limitations of the sample, not correct PUT resource-identity enforcement.

**Source excerpt:**
```java
@PostMapping("/users/{username}/todos")
public Todo createTodo(@PathVariable String username,
         @RequestBody Todo todo) {
    todo.setUsername(username);
    todo.setId(null);
    return todoRepository.save(todo);
}
```

Creation replaces the client-supplied ID with null so JPA generates a new identity. It assigns the path username, preventing a conflicting body username from choosing the owner. But it still does not compare that path username with the authenticated principal. It returns a bean directly: **200 OK**, not 201, and no Location header is set. A production-oriented contract would normally consider 201 plus a retrieval Location.

### 7. Greeting and Basic-auth check endpoints
Files: `helloworld/HelloWorldController.java`, `helloworld/HelloWorldBean.java`.

**Source excerpt:**
```java
@GetMapping(path = "/basicauth")
public String basicAuthCheck() {
    return "Success";
}
```

`/basicauth` is simply a protected endpoint returning a string. It does not parse credentials or create a session itself. The active filter chain can authenticate either Basic or JWT before it reaches this handler. The other mappings are `/hello-world`, `/hello-world-bean`, and `/hello-world/path-variable/{name}`. They return `Hello World v2`, a bean containing `Hello World Bean v1`, and a personalized bean respectively. This backend does not have the MessageSource i18n route from module 12.

### 8. Basic-only security: retained but not active
File: `basic/BasicAuthenticationSecurityConfiguration.java`.

**Source excerpt:**
```java
//@Configuration
public class BasicAuthenticationSecurityConfiguration {
```

Its `@Bean filterChain` method is not discovered as configuration in the present source. The class demonstrates permitting OPTIONS, authenticating other requests, Basic authentication, `SessionCreationPolicy.STATELESS`, and disabled CSRF. Do not activate it alongside the JWT chain without carefully separating chain matchers and order. The final active JWT chain independently retains `.httpBasic(...)`.

### 9. JwtSecurityConfig: active filter chain
File: `jwt/JwtSecurityConfig.java`.

**Source excerpt:**
```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/authenticate").permitAll()
    .requestMatchers(PathRequest.toH2Console()).permitAll()
    .requestMatchers(HttpMethod.OPTIONS,"/**")
    .permitAll()
    .anyRequest()
    .authenticated())
.csrf(AbstractHttpConfigurer::disable)
.sessionManagement(session -> session.
    sessionCreationPolicy(SessionCreationPolicy.STATELESS))
.oauth2ResourceServer(oauth2 -> oauth2.jwt(withDefaults()))
.httpBasic(Customizer.withDefaults())
```

`@Configuration` and `@EnableWebSecurity` make this the active configuration. The lambda resource-server DSL is appropriate for Spring Security 6 / Boot 3.3. The source retains commented deprecated DSL examples only as historical context. H2 servlet routes are matched with `PathRequest.toH2Console()` rather than business-controller assumptions. Frame options are set to same-origin to support the local console.

Every other request requires authentication, but no per-todo owner or scope rule is configured. OPTIONS permission helps a browser preflight reach CORS processing without a bearer token. `permitAll` changes authorization policy; it does not turn off all authentication filters. An invalid supplied Bearer token can still be rejected on `/authenticate` before the controller runs. That matters when stale axios interceptors attach old tokens to future login attempts.

### 10. AuthenticationManager and demonstration user
**Source excerpt from `JwtSecurityConfig`:**
```java
@Bean
public AuthenticationManager authenticationManager(UserDetailsService userDetailsService) {
    var authenticationProvider = new DaoAuthenticationProvider();
    authenticationProvider.setUserDetailsService(userDetailsService);
    return new ProviderManager(authenticationProvider);
}
```

`ProviderManager` delegates authentication to configured providers. The DAO provider uses the UserDetailsService and password-encoding infrastructure. The login controller asks this manager to check the supplied credentials. It does not authenticate by comparing strings inside the REST method.

**Source excerpt:**
```java
UserDetails user = User.withUsername("in28minutes")
                        .password("{noop}dummy")
                        .authorities("read")
                        .roles("USER")
                        .build();
```

The user is stored in `InMemoryUserDetailsManager`. **Subtlety:** `.roles("USER")` replaces the builder's previously configured authorities. Do not confidently claim the final user simultaneously has `read` and `ROLE_USER` from this chain. Its effective authority is `ROLE_USER`, which affects the generated scope string. Password-prefix delegation recognizes `{noop}`; a production password encoder should use an adaptive hash. The backend does not have an account registration or persistent credential repository.

### 11. RSA keys, JWK source, encoder, decoder
**Source excerpt from `JwtSecurityConfig`:**
```java
var keyPairGenerator = KeyPairGenerator.getInstance("RSA");
keyPairGenerator.initialize(2048);
return keyPairGenerator.generateKeyPair();
```

The configuration creates a 2048-bit RSA key pair at application startup. A `RSAKey` JWK wraps the public/private keys and a randomly generated key ID. A `JWKSet` supplies the key to a selector-driven `JWKSource<SecurityContext>`.

**Source excerpt:**
```java
@Bean
JwtEncoder jwtEncoder(JWKSource<SecurityContext> jwkSource) {
    return new NimbusJwtEncoder(jwkSource);
}
@Bean
JwtDecoder jwtDecoder() throws JOSEException {
    return NimbusJwtDecoder
            .withPublicKey(rsaKey().toRSAPublicKey())
            .build();
}
```

The encoder signs with the private key; the decoder verifies using the public key. The normal encoder defaults in this configuration use an RSA signing algorithm such as RS256. The exact header is best observed from an issued token rather than inferred from a hand-written string. The full `@Configuration` proxy preserves singleton bean identity for calls such as `rsaKey()`. It does not generate a different key on every decoder call during this configuration flow.
```text
private RSA key --sign--> JWT -----------------------> client
                                                           |
public RSA key <--verify-- JWT + Authorization header <-----+
```

Keys are generated in memory, not loaded from durable secure storage. Restarting the server invalidates signatures on previously issued tokens. Multiple replicas generating independent keys cannot reliably verify one another's tokens. No public JWKS HTTP endpoint, key-rotation schedule, or issuer discovery endpoint is implemented. The JWKSource bean alone does not publish a JWKS URL.

### 12. JWT issuance: request/response records and token service
Files: `jwt/JwtTokenRequest.java`, `jwt/JwtTokenResponse.java`.

**Source excerpts:**
```java
public record JwtTokenRequest(String username, String password) {}
public record JwtTokenResponse(String token) {}
```

Records express compact immutable request/response carriers. Jackson binds their named components; the response field is `token`, not `access_token`. This is a custom login contract, not the standard OAuth token endpoint response format.

File: `jwt/JwtAuthenticationController.java`.

**Source excerpt:**
```java
var authenticationToken =
        new UsernamePasswordAuthenticationToken(
                jwtTokenRequest.username(),
                jwtTokenRequest.password());
var authentication =
        authenticationManager.authenticate(authenticationToken);
var token = tokenService.generateToken(authentication);
return ResponseEntity.ok(new JwtTokenResponse(token));
```

These lines are inside `POST /authenticate` with `@RequestBody JwtTokenRequest`. Only a successfully authenticated result is passed to the token service. Failed credentials do not legitimately produce a token; ordinary failures map to authentication errors. There is no explicit `@Valid` DTO presence validation or custom login error envelope.

File: `jwt/JwtTokenService.java`.

**Source excerpt:**
```java
var claims = JwtClaimsSet.builder()
                .issuer("self")
                .issuedAt(Instant.now())
                .expiresAt(Instant.now().plus(90, ChronoUnit.MINUTES))
                .subject(authentication.getName())
                .claim("scope", scope)
                .build();
return this.jwtEncoder
        .encode(JwtEncoderParameters.from(claims))
        .getTokenValue();
```

The scope string is formed by joining `GrantedAuthority.getAuthority()` values with spaces. The token lifetime is **90 minutes**, not a default framework value. The issuer string is `self`; no audience claim is set. The configured public-key decoder verifies signatures and normal timestamp validity. It does **not** explicitly validate that `iss == self` or enforce an audience in this configuration. Issuer/audience validation should be added deliberately if those claims define trust boundaries.

The default resource-server scope converter generally prefixes scopes with `SCOPE_`. A scope string `ROLE_USER` consequently maps to `SCOPE_ROLE_USER`, not automatically `ROLE_USER`. This sample only requires `.authenticated()`, so that distinction does not block its current todo routes. It matters immediately when adding `hasRole("USER")` or `hasAuthority(...)` rules.

### 13. Frontend package, bootstrapping, and learning components
Files: `13-full-stack/01-frontend/todo-app/package.json`, `src/index.js`, `src/App.js`.

| Package/script | Actual role |
|---|---|
| `react`, `react-dom` | Components, hooks, rendering |
| `react-router-dom` | BrowserRouter, Routes, Route, Navigate, links, hooks |
| `axios` | Shared HTTP client and request interceptors |
| `formik` | Form state, fields, errors, submission |
| `moment` | Client date parsing/validation |
| `bootstrap` | UI styles imported by index.js |
| `react-scripts` | Create React App development/build/test tooling |
| `npm start` | Development server |
| `npm run build` | Production asset bundle |
| `npm test` | Test runner |

`src/index.js` uses `ReactDOM.createRoot` and wraps `<App />` in `<React.StrictMode>`. `src/App.js` renders `<TodoApp />`. StrictMode can replay effect setup/cleanup in development to reveal unsafe side effects. It does not mean the production server intentionally runs every API call twice.

The `learning-examples/` folder illustrates function/class components, fragments, nested objects, array mapping, and default/named exports. The current App does not mount LearningComponent. The `counter/` folder illustrates local state and callback props.

**Source excerpt from `counter/Counter.jsx`:**
```javascript
const [count, setCount] = useState(0)
function incrementCounterParentFunction(by) {
    setCount(count + by)
}
```

`CounterButton` receives `by`, `incrementMethod`, and `decrementMethod` as props. For multiple queued increments, a functional update such as `setCount(c => c + by)` is safer. That functional form is an improvement, not the source excerpt above.

### 14. TodoApp routing and client-side route guards
File: `todo/TodoApp.jsx`.

**Source excerpt:**
```jsx
function AuthenticatedRoute({children}) {
    const authContext = useAuth()
    if(authContext.isAuthenticated)
        return children
    return <Navigate to="/" />
}
```

The component tree places AuthProvider outside BrowserRouter and mounts HeaderComponent inside the router. Protected routes wrap their elements in AuthenticatedRoute. This controls UI navigation; it cannot prevent a caller from sending curl requests directly. Server authorization remains essential regardless of a hidden button or protected route.

| Frontend route | Component | Guard |
|---|---|---|
| `/`, `/login` | LoginComponent | Public |
| `/welcome/:username` | WelcomeComponent | AuthenticatedRoute |
| `/todos` | ListTodosComponent | AuthenticatedRoute |
| `/todo/:id` | TodoComponent | AuthenticatedRoute |
| `/logout` | LogoutComponent | AuthenticatedRoute |
| `*` | ErrorComponent | Catch-all UI |

Frontend `/todo/-1` means “create a new todo” by convention. It is not a database record with a meaningful negative ID. React Router path parameters are strings, which explains the loose comparisons to `-1` in the source. The UI catch-all is not the same as an HTTP 404 status returned by Spring. Production SPA hosting must route deep links to the application entry page.

### 15. LoginComponent and shared AuthContext
Files: `todo/LoginComponent.jsx`, `todo/security/AuthContext.js`. LoginComponent stores username, password, and error visibility in `useState`. Its controlled inputs call change handlers that store `event.target.value`. Its asynchronous submit awaits `authContext.login` before navigating to `/welcome/${username}`. There is no Formik usage in this login form; Formik is used for editing todos.

**Source excerpt from AuthContext:**
```javascript
export const AuthContext = createContext()
export const useAuth = () => useContext(AuthContext)
const [isAuthenticated, setAuthenticated] = useState(false)
const [username, setUsername] = useState(null)
const [token, setToken] = useState(null)
```

`AuthProvider` exposes these values and `login`/`logout` through a Context provider. This avoids passing authentication props manually through every intermediate component. A provider state update causes consumers to observe the new context value during rendering. State is in memory; there is no localStorage/sessionStorage/cookie persistence in the active code. Refreshing the page resets authentication even if the backend token remains unexpired.

**Source excerpt from the active login implementation:**
```javascript
const response = await executeJwtAuthenticationService(username, password)
if(response.status==200){
    const jwtToken = 'Bearer ' + response.data.token
    setAuthenticated(true)
    setUsername(username)
    setToken(jwtToken)
    apiClient.interceptors.request.use(
        (config) => {
            console.log('intercepting and adding a token')
            config.headers.Authorization = jwtToken
            return config
        }
    )
```

`login` returns true on success; failures call logout and return false. Earlier mock and Basic-auth versions remain inside comments and are not active. The stored token string includes the `Bearer ` prefix, not just the compact JWT. There is no response interceptor to handle expiry, refresh, or centralized 401 recovery.

**Source excerpt from logout:**
```javascript
function logout() {
    setAuthenticated(false)
    setToken(null)
    setUsername(null)
}
```

It clears React state but does not eject request interceptors or contact the backend. Previously installed interceptors still hold their captured token values. Repeated successful logins accumulate interceptors rather than replacing one interceptor. This causes stale authorization behavior discussed in the pitfalls section.

### 16. Shared axios client and API service modules
File: `todo/api/ApiClient.js`.

**Source excerpt:**
```javascript
export const apiClient = axios.create(
    {
        baseURL: 'http://localhost:8080'
    }
);
```

This is a dedicated axios instance; interceptors apply to calls made through that instance. The base URL is hard-coded for local development, not environment-configured. There is no global proxy assumption in these service modules. There is no `withCredentials: true` setting because the normal path uses an explicit Authorization header.

File: `todo/api/AuthenticationApiService.js`.

**Source excerpt:**
```javascript
export const executeJwtAuthenticationService
    = (username, password) =>
        apiClient.post(`/authenticate`,{username,password})
```

The module also retains a Basic helper calling `/basicauth` with an explicit Authorization header. The active AuthContext imports the JWT helper, not the Basic helper. axios serializes the object into JSON and returns a promise resolving to a response object. Read API payload from `response.data`, status from `response.status`, and headers from `response.headers`.

File: `todo/api/TodoApiService.js`.

**Source excerpt:**
```javascript
export const retrieveAllTodosForUsernameApi
    = (username) => apiClient.get(`/users/${username}/todos`)
export const updateTodoApi
    = (username, id, todo) => apiClient.put(`/users/${username}/todos/${id}`, todo)
export const createTodoApi
    = (username, todo) => apiClient.post(`/users/${username}/todos`, todo)
```

The same file supplies individual retrieval and DELETE helpers. They rely on AuthContext's installed interceptor to add credentials. `HelloWorldApiService.js` uses this client for bean and path-variable greetings. Its function accepts a token parameter in one signature but the explicit header block is commented out. The active interceptor—not that unused parameter—is responsible for authentication.

### 17. ListTodosComponent: effects and rendering
File: `todo/ListTodosComponent.jsx`.

**Source excerpt:**
```javascript
const [todos,setTodos] = useState([])
const [message,setMessage] = useState(null)
useEffect ( () => refreshTodos(), [])
```

`refreshTodos()` calls the username collection API and stores `response.data`. Rendering maps todos into `<tr key={todo.id}>` rows. Stable record IDs help React preserve item identity during updates and deletions. A Date field from JSON is a string; the component displays its string form. `deleteTodo()` displays a message and reloads the list after successful deletion. `updateTodo()` navigates to `/todo/${id}`; add-new navigates to `/todo/-1`.

The empty dependency array means mount-triggered fetching under this component's assumptions. The effect still captures username/refreshTodos values; future account switching needs correct dependencies. The source mainly logs request errors; it has no robust loading/retry/error UI. An unused computed targetDate uses `today.getDay()` rather than day-of-month `getDate()`. Do not confuse that leftover teaching variable with dates fetched from the backend.

### 18. TodoComponent: Formik editor, dates, and save
File: `todo/TodoComponent.jsx`. It reads `id` with `useParams`, username from Context, and navigation through `useNavigate`. It stores initial description/date in React state and retrieves an existing todo when `id != -1`. Its retrieval effect depends on `[id]`.

**Source excerpt:**
```jsx
<Formik initialValues={ { description, targetDate } }
    enableReinitialize = {true}
    onSubmit = {onSubmit}
    validate = {validate}
    validateOnChange = {false}
    validateOnBlur = {false}
>
```

`enableReinitialize` lets asynchronously fetched React state become the form's initial values. `Field` binds a named form value; `ErrorMessage` renders the named validation error. `Form` integrates Formik's submission handling. Changing initial values later can reset form editing, so deliberate initialization boundaries matter.

**Source excerpt:**
```javascript
if(values.description.length<5) {
    errors.description = 'Enter atleast 5 characters'
}
if(values.targetDate == null || values.targetDate=='' || !moment(values.targetDate).isValid()) {
    errors.targetDate = 'Enter a target date'
}
```

These are client-side rules: minimum five characters and a parseable nonempty date. They do not require the date to be in the future or ensure equivalent server validation. Validation on change/blur is disabled; submit still invokes the validate function.

**Source excerpt from submission:**
```javascript
const todo = {
    id: id,
    username: username,
    description: values.description,
    targetDate: values.targetDate,
    done: false
}
```

For `id == -1`, it POSTs; otherwise it PUTs. The backend's create handler resets ID to null before persistence. Every save sets `done: false`, potentially resetting a previously completed todo during editing. On success the editor navigates to `/todos`; on failure it mainly logs the error. A production form should preserve relevant fields and avoid logging personal form data unnecessarily.

### 19. Header, welcome, logout, errors, and footer
Files: `todo/HeaderComponent.jsx`, `WelcomeComponent.jsx`, `LogoutComponent.jsx`, `ErrorComponent.jsx`, `FooterComponent.jsx`.

- Header renders links conditionally based on Context authentication state.
- Its Home link hard-codes `/welcome/in28minutes`, not the current username.
- Its Logout link calls `authContext.logout()` in its click handler.
- Welcome reads username from the route but calls the greeting with hard-coded `Ranga`.
- Welcome reads the returned greeting from `response.data.message`.
- LogoutComponent only displays text; it does not itself clear authentication.
- The logout route is guarded, so clearing auth before navigation can redirect to `/` instead of showing it.
- ErrorComponent is a frontend fallback display, not backend exception translation.
- FooterComponent references an old Context `number` field that the current provider does not expose.
- FooterComponent is not mounted by the final TodoApp.

These details illustrate why source inspection matters more than a generic todo-app description. UI state, route state, and backend principal identity are distinct sources of information. Only the server's verified principal should drive security decisions.

### 20. REST endpoint inventory
All business endpoints except `/authenticate` require authentication in the active chain. OPTIONS requests and H2-console routes are explicitly permitted. No general `@ControllerAdvice` from module 12 is present in this backend. Missing data and security responses therefore do not share module 12's ErrorDetails contract.

| Method | URL | Description | Current outcomes / notable errors |
|---|---|---|---|
| POST | `/authenticate` | Verify credentials and return `{token}` | 200 success; 401 normal bad credentials; 400 malformed body |
| GET | `/basicauth` | Protected check returning `Success` | 200; 401 without acceptable authentication |
| GET | `/hello-world` | Plain `Hello World v2` | 200; 401 |
| GET | `/hello-world-bean` | JSON greeting | 200; 401 |
| GET | `/hello-world/path-variable/{name}` | Personalized JSON greeting | 200; 401 |
| GET | `/users/{username}/todos` | Filter todos by path username | 200 collection, including empty; 401 |
| GET | `/users/{username}/todos/{id}` | Lookup by ID only | 200; missing ID usually 500; bad numeric ID 400; 401 |
| POST | `/users/{username}/todos` | Save new generated-ID todo | 200 with entity; no Location; 400 binding errors; 401 |
| PUT | `/users/{username}/todos/{id}` | Save supplied body entity | 200 ordinarily; body ID controls save; 400 binding errors; 401 |
| DELETE | `/users/{username}/todos/{id}` | Delete by ID only | 204 with no body; 401 |
| OPTIONS | Any controller path with CORS policy | Browser preflight | Typically 200 if accepted; 403 for rejected origin/policy |
| GET | `/h2-console` | Local database UI servlet | UI/redirect; permitted with same-origin frames |

No PATCH endpoint, refresh-token endpoint, server logout endpoint, or account registration endpoint exists. For unauthenticated requests, the exact authentication challenge can depend on request/entry-point selection. An invalid Bearer token is normally rejected even on otherwise permitted routes. Do not label the current POST status as 201 merely because that would be a better creation contract.

## Core concepts explained
### Full-stack boundaries and responsibilities
| Layer | Responsibility | Must not be trusted to do alone |
|---|---|---|
| React UI | User interaction, rendering, client validation | Enforce server permissions or protect secrets |
| HTTP/REST contract | Methods, URIs, media types, statuses | Guarantee database ownership without server checks |
| Spring Security | Authenticate and apply authorization policies | Infer todo ownership from a URL string automatically |
| MVC/controller | Bind requests and orchestrate responses | Treat client body IDs/owners as verified identity |
| Service/domain layer | Business rules and transaction boundaries | Assume frontend validation covers all callers |
| Repository/JPA | Query and persist entities | Replace security policy by existence checks alone |
| Database | Durable constraints/storage when configured | Safely recover all invariants if API policies are missing |

The final sample merges some responsibilities for teaching simplicity. A stronger design adds a transactional service and explicit account-scoped DTO operations. React and Spring need agreement on JSON property names and status/body expectations. They do not need to share implementation languages or database entity classes.

### Same-origin policy and CORS: what and why
An origin is the tuple **scheme + hostname + port**. These are different origins:
```text
http://localhost:3000        React development server
http://localhost:8080        Spring backend
http://127.0.0.1:3000        Different host spelling
https://localhost:3000       Different scheme
```

Browsers restrict scripts from reading cross-origin responses without permission. CORS means **Cross-Origin Resource Sharing**: response headers grant narrowly defined browser access. It is enforced by browsers, not by curl as a server access-control mechanism. A successful curl request does not prove the browser's origin is allowed. CORS is not authentication and does not stop a non-browser client from making HTTP requests.

| Header | Direction | Meaning |
|---|---|---|
| `Origin` | Request | Browser's initiating origin |
| `Access-Control-Request-Method` | Preflight request | Intended actual method |
| `Access-Control-Request-Headers` | Preflight request | Intended non-safelisted header names |
| `Access-Control-Allow-Origin` | Response | Allowed browser origin |
| `Access-Control-Allow-Methods` | Preflight response | Permitted methods |
| `Access-Control-Allow-Headers` | Preflight response | Permitted request headers |
| `Access-Control-Allow-Credentials` | Response | Whether credentialed browser access is permitted |
| `Access-Control-Expose-Headers` | Response | Extra response headers scripts may read |

A simple cross-origin request may be sent without a preflight, then blocked from being read. A non-safelisted request is preflighted before the browser sends the actual request. JSON `Content-Type: application/json` and an Authorization header typically trigger preflight. Preflight generally does not carry the application's Bearer token. That is why requiring authentication for OPTIONS would break the browser flow.

### Preflight OPTIONS sequence
```text
React browser                  Spring Security                    Spring MVC CORS
     |                                |                                  |
     | OPTIONS /users/in28minutes/todos                                  |
     | Origin: http://localhost:3000  |                                  |
     | Access-Control-Request-Method: GET                                |
     | Access-Control-Request-Headers: authorization                     |
     |------------------------------->| OPTIONS permitted                 |
     |                                |--------------------------------->|
     |                                |                   check origin/method/headers
     |<------------------------------------------------------------------|
     | Access-Control-Allow-Origin: http://localhost:3000                  |
     | Access-Control-Allow-Methods / Allow-Headers                       |
     |                                |                                  |
     | GET /users/in28minutes/todos    |                                  |
     | Authorization: Bearer <token>  |                                  |
     |------------------------------->| verify JWT, authenticate           |
     |                                |--------------------------------->|
     |<-------------------------- 200 JSON + CORS headers ----------------|
```

Allowing OPTIONS in Security does not itself add CORS response headers. MVC's WebMvcConfigurer supplies the policy in the current source. For early authentication failures, integrate CORS directly into Security as well.

**Illustrative improvement, not checked-in code:**
```java
http.cors(Customizer.withDefaults());
```

With Spring MVC present, Security can use the MVC CORS configuration in the appropriate setup. Alternatively supply a deliberate CorsConfigurationSource/CorsFilter with narrow allowed origins. Do not use `allowedOrigins("*")` together with credentialed cookie access. For same-origin deployment behind a reverse proxy, the development CORS problem may disappear.

### Basic authentication versus JWT bearer authentication
| Concern | Basic Auth | JWT Bearer |
|---|---|---|
| Wire header | `Authorization: Basic <base64(user:password)>` | `Authorization: Bearer <signed token>` |
| Sent repeatedly | Reusable username/password representation | Previously issued bearer credential |
| Encoding | Base64, not encryption | Base64url segments plus cryptographic signature |
| Initial login | Each request can authenticate credentials | Custom login authenticates credentials and issues token |
| Server verification | Typically UserDetailsService/password encoder | Signature and claim validators |
| Expiry | Password remains valid until account/credential changes | `exp` limits token lifetime |
| Revocation | Change password/account policy | Needs additional strategy or wait for expiry |
| Principal loading | Provider often loads user every call | Can reconstruct principal from trusted token claims |
| Storage concern | Avoid storing reusable password in UI | Protect bearer token from theft and replay |
| Transport | HTTPS required | HTTPS required |

JWT does not eliminate credential handling; it changes what later requests carry. A stolen JWT acts as the user until it expires or is otherwise invalidated. A signed JWT is usually readable, not confidential. Basic can be stateless; JWT can be used in a stateful application. The authentication format alone does not define session policy.

### JWT structure: header.payload.signature
A compact signed JWT/JWS has three dot-separated base64url components:
```text
base64url(header).base64url(payload).base64url(signature)
       |                  |                     |
    metadata           claims             integrity proof
```

An **illustrative decoded** token for this app resembles:
```json
{
  "alg": "RS256",
  "kid": "generated-key-id"
}
```
```json
{
  "iss": "self",
  "sub": "in28minutes",
  "iat": 1793440000,
  "exp": 1793445400,
  "scope": "ROLE_USER"
}
```

The numeric times and key ID above are examples, not captured execution values. Do not assume the header always includes `typ` unless you inspect actual encoder output. The signature protects the encoded header and payload against undetected modification. Decoding base64url does not establish authenticity; signature verification does. Do not put passwords, secrets, or unnecessary sensitive personal data into readable JWT claims.

| Claim/header | Meaning | Present app |
|---|---|---|
| `alg` | Signing algorithm | RSA encoder/decoder defaults, commonly RS256 |
| `kid` | Key identifier for selection | Random RSA JWK key ID |
| `iss` | Issuer | `self` |
| `sub` | Subject identity | Authenticated username |
| `iat` | Issued-at time | Instant.now() |
| `exp` | Expiration time | Issued around 90 minutes earlier |
| `scope` | Space-separated authority-derived values | Joined Authentication authorities |
| `aud` | Intended recipient audience | Not set/explicitly enforced |
| `nbf` | Not valid before | Not set by token service |
| `jti` | Unique token identifier | Not set by token service |

Expiration is evaluated by the server, normally with limited clock-skew tolerance. The frontend can inspect exp for UX timing, but must not decide security validity by decoding alone.

### RSA signing, JWK, and Nimbus
RSA is asymmetric: private and public keys serve different roles. RS256 signs using RSA with SHA-256; it does not encrypt the claim payload. The issuing service holds the private key; resource servers only need trusted public verification keys. That separation is useful when several APIs consume tokens from one authorization service.
```text
credentials --> AuthenticationManager --> authenticated identity
                                              |
                                              v
claims + private key --> NimbusJwtEncoder --> compact signed JWT
                                              |
Authorization header + public key --> NimbusJwtDecoder
                                              |
                             signature + claim validation
                                              |
                                  trusted Authentication
```

JWK is a JSON Web Key representation; JWKSet contains keys; JWKSource selects keys. NimbusJwtEncoder adapts Nimbus JOSE support to Spring's JwtEncoder interface. NimbusJwtDecoder adapts verification/validation to JwtDecoder. A decoder configured with a public key is not automatically fetching remote issuer metadata. A custom `/authenticate` endpoint is not a full OAuth 2.0/OIDC authorization server.

Production considerations include durable key storage, key IDs, rotation overlap, issuer/audience validation, algorithm allowlists, accurate clocks, and safe distribution/caching of public verification keys. Never let an untrusted token's algorithm header alone choose an arbitrary verification strategy. Do not share the private key with every frontend or resource server.

### oauth2ResourceServer jwt and stateless sessions
`oauth2ResourceServer(oauth2 -> oauth2.jwt(withDefaults()))` enables Bearer-token authentication. The bearer authentication filter extracts the token and authenticates it through JWT infrastructure. Spring converts trusted claims into an Authentication and authorities in the SecurityContext. That context supports authorization for the current request.

`SessionCreationPolicy.STATELESS` means Spring Security will not create/use an HTTP session as the repository of security authentication for subsequent requests. It does not mean there is no database, no cache, no current request SecurityContext, or no state anywhere. Other application functionality can still have independent state and lifecycle behavior. Each protected request must bring its own acceptable authentication credential.
```text
Request A + JWT --> verify --> principal for A --> response --> request context cleaned
Request B + JWT --> verify --> principal for B --> response --> request context cleaned
```

The browser's AuthContext is separate in-memory UI state, not a server session. The current chain's disabled CSRF fits the explicit bearer-header demonstration better than a cookie session. However, Basic authentication is also enabled, and browser-automatically-attached credentials need a separate threat assessment. Do not say “stateless” automatically proves CSRF is impossible.

### Browser token storage: concerns and alternatives
The source stores its token in React state, not localStorage. Any request interceptor closure can also retain a token in memory. That is less persistent than browser storage, but it is not immune to XSS. Malicious same-origin JavaScript can still make authenticated requests or intercept credentials.

| Storage option | Benefit | Main concern |
|---|---|---|
| In-memory state | Lost on reload; no persistent token file | XSS can still act; refresh loses login |
| sessionStorage | Survives reload within tab | Readable by JavaScript/XSS |
| localStorage | Survives reload and browser restart | Long-lived JavaScript-readable token theft risk |
| HttpOnly cookie | JavaScript cannot directly read token | Browser auto-sends it; design CSRF/SameSite/CORS correctly |
| Backend-for-frontend session | Browser can use controlled session cookie | Requires server session/BFF lifecycle and CSRF policy |

There is no universal storage choice independent of application threat model. Use HTTPS, short-lived access tokens, secure cookies where selected, a strong CSP, and XSS prevention. Refresh tokens require stronger protections, rotation/reuse handling, and careful lifecycle design. This app has no refresh token or token renewal implementation. Do not recommend a long-lived JWT in localStorage as an unqualified “best practice.”

### React components, props, state, and hooks
A React component is a reusable function/class that describes UI from props and state. JSX is syntax transformed into JavaScript UI element construction; it is not a raw HTML template engine. `className` is the JSX DOM class attribute spelling. Fragments let a component group children without adding an extra DOM wrapper.

| Concept | Actual module example | Model answer |
|---|---|---|
| Function component | LoginComponent, TodoComponent | Function returns a UI description |
| Class component | ThirdComponent/FourthComponent | Extends Component and implements render |
| Props | CounterButton callback/step props | Inputs passed from parent; do not mutate them |
| Local state | Username, todos, message | useState schedules rerender with new value |
| Effect | List retrieval and editor retrieval | Synchronize with an external system after render |
| Route hook | useParams/useNavigate | Read matched params and navigate programmatically |
| Context | AuthProvider/useAuth | Share auth state/functions across consumers |
| Controlled input | Login username/password fields | Value is driven by state and onChange updates it |
| List identity | `<tr key={todo.id}>` | Stable keys support correct reconciliation |

State setters schedule updates; reading the variable immediately afterward still sees that render's snapshot. Use functional updates when next state depends on prior state under queued updates. Keep ordinary render logic pure; perform requests and subscriptions in handlers/effects appropriately. Avoid logging/side effects from JSX expression evaluation as a production pattern.

### useEffect dependency and cleanup reasoning
`useEffect(..., [])` runs for a mount lifecycle, subject to development StrictMode checks. `useEffect(..., [id])` runs again when the route ID changes. An effect's closure captures values from the render that created it. Missing dependencies can leave stale values; unstable function dependencies can cause repeated effects.

The list currently assumes username stays unchanged while mounted. A more general account-switching flow should rerun/cancel retrieval when username changes. The editor should also reset defaults when entering a new-item route rather than retaining old edit state. AbortController or another cancellation/ignore strategy can prevent late responses from overwriting newer state. Do not make the effect callback itself async if it would return a promise instead of cleanup.

### Context API versus real authentication
Context shares the UI's belief about login state. `isAuthenticated: true` is not proof that a token is still valid at the server. A 90-minute-old token, restarted signing server, or changed authorization policy can invalidate a request. The backend must verify credentials on every protected request regardless of Context state.

A route parameter username is also not verified identity. Current list calls use Context username; the server nevertheless must scope records to its own principal. `AuthenticatedRoute` protects user experience, not the REST endpoint from unauthorized direct calls. Never rely on hiding an Update button as a security control.

### axios interceptors and lifecycle
An axios request interceptor runs before dispatch and can attach shared headers. An axios response interceptor can normalize errors or react to 401, but none is implemented here. `use(...)` returns an interceptor ID that can be ejected later. Interceptors on one axios instance do not automatically apply to every other axios client.

The source adds a new request interceptor on every successful login. Each callback captures its own `jwtToken` and writes `config.headers.Authorization`. axios request interceptors run in reverse registration order by default. An older interceptor can overwrite a newer token later in the chain. Logout's state clearing does not erase those captured credentials.

A safer **illustrative design** is one stable interceptor reading current token state through a controlled reference/store, or a single installed interceptor that is ejected/replaced whenever credentials change. Exclude the authentication endpoint from stale bearer injection when appropriate. Handle 401 without an infinite refresh/retry loop and without retrying non-idempotent writes blindly. Never log Authorization headers or full tokens for routine troubleshooting.

### Formik and server validation
Formik centralizes field values, validation errors, touched state, and submission behavior. `initialValues` provides the initial model; `enableReinitialize` handles changed initial values. `Field` connects a named input; `ErrorMessage` renders its named validation error. A validate function returns a field-keyed error object.
```text
User types --> Formik values --> validate(values)
                                  |
                   errors --------+-------- no errors
                     |                          |
                ErrorMessage                 onSubmit
                                                |
                                             axios
                                                |
                                      server must validate again
```

Formik improves user experience, not server trust. curl or malicious clients can bypass it entirely. The source backend currently has no matching @Valid constraints on Todo. Date parsing is not business-rule validation, and string length does not necessarily mean nonblank content. When updating, preserve fields such as done instead of hard-coding false.

### HTTP identity, ownership, and optimistic updates
The URI `/users/{username}/todos/{id}` promises a relationship between owner and record. A correct implementation should verify that relationship, not merely accept both path parameters. Use the authenticated principal and an ownership-scoped repository query such as `findByIdAndUsername(id, principalName)` in an intentional service operation. That method is a proposed design, not a current repository method.

For PUT, check that path and body IDs agree or ignore the body ID and assign the path identity deliberately. Define whether absent targets may be created; do not let save's behavior accidentally define API policy. Use DTOs to avoid mass assignment of username/ID/security-related fields. For concurrent updates, use a JPA @Version column and/or ETag/If-Match contract. This sample has no optimistic concurrency control, so last-writer-wins effects are possible.

### Login request end-to-end sequence
The following describes a fresh login without leftover interceptors. For a browser JSON POST, a CORS preflight typically comes first.
```text
User        LoginComponent     AuthContext/axios      Security       AuthController      AuthManager      TokenService
 |                |                    |                  |               |                 |                |
 | type + click   |                    |                  |               |                 |                |
 |--------------->| login(user,pass)   |                  |               |                 |                |
 |                |------------------->| OPTIONS /authenticate             |                 |                |
 |                |                    |----------------->| allow OPTIONS; MVC CORS policy approves           |
 |                |                    |<-----------------|               |                 |                |
 |                |                    | POST /authenticate {username,password}             |                |
 |                |                    |----------------->| permit route  |                 |                |
 |                |                    |                  |-------------->| build credential token            |
 |                |                    |                  |               |---------------->| load user + check password
 |                |                    |                  |               |<----------------| authenticated identity
 |                |                    |                  |               |--------------------------------->| build claims
 |                |                    |                  |               |                                  | sign with RSA
 |                |                    |                  |               |<---------------------------------| compact JWT
 |                |                    |<------------------------------- 200 {"token":"..."} ----------------|
 |                |                    | set auth/user/token; install request interceptor    |                |
 |                |<-------------------| true             |               |                 |                |
 |                | navigate /welcome/in28minutes          |               |                 |                |
 |<---------------| render protected UI                   |               |                 |                |
```

Credentials are sent once in this login contract, but they are still sensitive and require TLS. The controller is permitted for authorization; password verification still happens explicitly. A bad password should leave Context unauthenticated and show an error instead of navigating. A forged token is not accepted simply because the browser stores it.

### Authenticated request end-to-end sequence
```text
User       ListTodosComponent       axios/interceptor       Security/JWT          TodoJpaResource       Repository/JPA      H2
 |                 |                        |                    |                      |                    |             |
 | open /todos     |                        |                    |                      |                    |             |
 |---------------->| useEffect refresh      |                    |                      |                    |             |
 |                 |----------------------->| add Authorization: Bearer <token>           |                    |             |
 |                 |                        | OPTIONS if browser needs uncached preflight                      |             |
 |                 |                        |------------------->| permit + CORS         |                    |             |
 |                 |                        |<-------------------|                      |                    |             |
 |                 |                        | GET /users/in28minutes/todos               |                    |             |
 |                 |                        |------------------->| decode + verify RSA   |                    |             |
 |                 |                        |                    | check token timestamps|                    |             |
 |                 |                        |                    | set Authentication    |                    |             |
 |                 |                        |                    |--------------------->| findByUsername      |             |
 |                 |                        |                    |                      |------------------->| SELECT      |
 |                 |                        |                    |                      |                    |------------>|
 |                 |                        |                    |                      |                    |<------------|
 |                 |                        |                    |                      |<-------------------| List<Todo>  |
 |                 |                        |<--------------------------- 200 JSON -------------------------|             |
 |                 |<-----------------------| response.data       |                      |                    |             |
 |                 | setTodos(data) -> rerender rows              |                      |                    |             |
 |<----------------|                        |                    |                      |                    |             |
```

This is authentication plus the **current** username query, not a claim of owner enforcement. If the token fails validation, Security stops the call before the controller/repository. With current CORS integration, the browser may mask an early 401 as a CORS error. The backend does not refresh the token or re-run the password login for a normal Bearer request.

## Common pitfalls / gotchas
### Authentication and authorization limitations
1. A valid JWT proves a trusted asserted identity, not permission for every todo ID.
2. List routes trust path username; individual GET/DELETE ignore it when selecting records.
3. PUT saves the body ID and does not enforce path/body identity agreement.
4. Create assigns the path username but does not verify that the caller owns that username.
5. The filter chain uses authenticated-only rules, not per-owner or per-scope business authorization.
6. The builder's roles call replaces earlier authorities, so do not assume a read scope exists.
7. Default JWT scope conversion yields SCOPE-prefixed authorities, not ordinary role names automatically.
8. `{noop}` passwords and in-memory generated keys are classroom conveniences.

### JWT lifecycle traps
- Restarted backend means a new RSA key; existing browser tokens stop verifying.
- Independent replicas cannot share this generated-key trust implicitly.
- Payload decoding does not verify integrity or expiry.
- Issuer/audience claims need validators, not just string values in a token.
- The sample has no refresh, revoke, introspection, or server-side logout endpoint.
- Clearing Context state does not invalidate an already issued token.
- A bearer credential can be replayed by anyone who steals it.
- Do not paste real tokens into public online JWT decoders or logs.

### CORS and browser traps
- Different ports are different origins even when both servers run on one machine.
- `localhost` and `127.0.0.1` do not match the same allowlist entry.
- Authorization and JSON POST usually require an OPTIONS preflight.
- Permitting OPTIONS is not the entire CORS configuration.
- MVC CORS handling may not decorate early Security failures.
- CORS does not prevent curl from sending the same request.
- A response lacking accessible CORS headers may look like a network error in axios.
- Explicit Bearer headers do not require `withCredentials`, but cookie-based designs need other choices.

### React/axios traps in the actual source
- Login adds an interceptor every time and never ejects it.
- Logout clears state but old interceptors retain and can still send old tokens.
- Reverse request-interceptor order can let older tokens overwrite newer tokens.
- A stale Bearer header on `/authenticate` can fail before a permitted login handler runs.
- Reload clears Context; backend expiry and frontend login state are not synchronized.
- The logout page does not call logout; the Header click does.
- A guarded logout route can redirect immediately after state is cleared.
- The Home path and greeting argument contain hard-coded usernames.
- Missing effect dependencies matter when identity changes without unmounting.
- StrictMode development replay can expose duplicate request effects.
- No centralized response error handling or refresh behavior is implemented.

### Forms, persistence, and contract traps
- Formik length/date checks do not apply to direct backend callers.
- A valid parsed date is not necessarily a future date or acceptable business date.
- Todo edits always submit done:false, possibly undoing completion.
- Entering the create route from an edit route may retain old editor state without reset logic.
- JPA rows use IDs 10001–10003 initially; the old service uses IDs 1–3.
- Missing findById().get() results are not deliberately mapped to 404.
- Creation currently returns 200 rather than 201 plus Location.
- DELETE returns 204; do not parse an empty response as a todo representation.
- In-memory H2 is recreated on restart, so it is not durable production storage.
- App.test.js's starter assertion is stale and does not test the rendered todo UI.

## Interview questions & answers
### Basic
#### 1. What are the main full-stack layers in this module?
React UI calls REST endpoints in Spring, which use Spring Data JPA/Hibernate and H2. The active TodoJpaResource calls the repository directly rather than the old static service.
#### 2. What is a React component?
A reusable function or class describing UI from props and state. The current todo UI uses function components, while learning examples also contain class components.
#### 3. What is the difference between props and state?
Props are inputs from a parent; state is component-managed data that can change across renders. CounterButton receives props, while LoginComponent owns input state with useState.
#### 4. What does useState do?
It stores a value across renders and provides a setter that schedules an update. The current render retains its state snapshot even immediately after calling the setter.
#### 5. Why use useEffect for fetching todos?
Fetching synchronizes the component with an external system after rendering. The list effect triggers retrieval and then stores the returned array for a rerender.
#### 6. Why is a key needed in a rendered list?
A stable key identifies the same item across insertions, deletions, and reordering. The source uses each todo's database ID as its table-row key.
#### 7. What is CORS?
Browser-enforced cross-origin response-access permission expressed through HTTP headers. It is not authentication and does not restrict curl as a security barrier.
#### 8. Why are ports 3000 and 8080 cross-origin?
Origin includes port as well as scheme and hostname. Two services on localhost can therefore require CORS permission for browser JavaScript calls.
#### 9. What is Basic authentication?
A request carries Base64-encoded username:password in an Authorization Basic header. Base64 is reversible encoding, so transport must be protected by HTTPS.
#### 10. What is JWT?
A claims token format commonly carried as a compact signed header.payload.signature value. A signature supplies integrity/authenticity, not confidentiality of readable claims.
#### 11. Where does this frontend store its JWT?
In React AuthContext state, with captured copies also retained by installed interceptor closures. It does not actively persist the token in localStorage or a cookie.
#### 12. What does axios return here?
A promise resolving to a response containing data, status, and headers. The UI reads todo arrays from response.data rather than the entire response object.

### Intermediate
#### 13. How does the browser learn whether Authorization is allowed cross-origin?
It sends a preflight OPTIONS request listing the intended method and header names. The server must answer with an allowed origin and compatible CORS method/header permissions.
#### 14. Is permitting OPTIONS sufficient to solve CORS?
No: it avoids an authentication barrier, but the response must still contain correct CORS permission headers. The current application provides its MVC policy through WebMvcConfigurer.
#### 15. What is the purpose of AuthContext?
It shares login state, username, token, and login/logout operations across components without prop drilling. It improves UI coordination but is not a server authorization boundary.
#### 16. What does AuthenticatedRoute protect?
It redirects unauthenticated UI navigation before rendering protected components. It cannot stop a direct HTTP client; the backend must enforce access independently.
#### 17. How does POST /authenticate work?
The controller constructs a username/password Authentication token and passes it to AuthenticationManager. On successful verification, JwtTokenService signs claims and the controller returns a token record with 200.
#### 18. Why are RSA public and private keys different roles useful?
Only the issuer needs the private signing key; resource servers can verify using a public key. That avoids sharing signing capability with every API consumer.
#### 19. What does NimbusJwtEncoder do?
It implements Spring's JWT encoding/signing abstraction using Nimbus JOSE support and the JWKSource. The module supplies an RSA key set for selecting its signing key.
#### 20. What does oauth2ResourceServer jwt enable?
Bearer-token extraction, decoding/verification, and conversion to a trusted Authentication for request authorization. It does not by itself implement a full OAuth authorization server or refresh endpoint.
#### 21. What does STATELESS mean in Spring Security?
Security does not persist authentication in an HTTP session for subsequent requests. Each protected request carries credentials; database state and a per-request SecurityContext still exist.
#### 22. Why does Formik need enableReinitialize in the editor?
Initial description/date arrive asynchronously after a backend retrieval. Reinitialization lets changed React initial values populate the form instead of remaining empty.

#### 23. Does Formik validation secure the backend?
No: clients can bypass it by calling the API directly. Server-side request and business validation must enforce equivalent rules independently.

#### 24. How does the final repository differ from TodoService?
The repository persists relational rows and derives findByUsername through Spring Data. TodoService stores a static list and uses case-insensitive username filtering, but is not the active storage path.

### Advanced and scenario-based
#### 25. Why can the UI stay “logged in” while every API call fails?
AuthContext's flag can outlive token validity or survive until the signing server restarts with a new key. The UI needs deliberate 401/expiry handling; the server must keep rejecting invalid credentials.

#### 26. Why can logout still leave Authorization headers on later requests?
Logout clears state but does not eject interceptors that captured previous jwtToken values. Manage one interceptor lifecycle and remove/replace stored credential behavior explicitly.

#### 27. Why might a second login use the first token?
Successful logins install multiple interceptors, and request interceptors normally execute in reverse registration order. An older interceptor can overwrite Authorization after the newer interceptor has set it.

#### 28. Why might a permitted login endpoint reject an invalid old Bearer token?
permitAll changes authorization, not authentication-filter execution for supplied credentials. A Bearer filter can reject an invalid header before the controller receives the new password request.

#### 29. Does the source's JWT issuer string get enforced automatically?
Not by this explicit public-key decoder configuration alone. It verifies signature/timestamps, but issuer and audience validation need intentional validators.

#### 30. What happens when two replicas generate their own RSA keys?
A token signed by one replica may fail verification on the other. Use shared durable issuer keys or a trusted public-key distribution/JWKS and rotation strategy.

#### 31. Is the final demo user both read and ROLE_USER?
No: User.Builder.roles replaces its previously configured authority list. The generated scope reflects the effective ROLE_USER authority rather than guaranteed read plus role values.

#### 32. Why could hasRole("USER") fail for the generated JWT?
Default JWT scope conversion prefixes values with SCOPE_, so ROLE_USER becomes SCOPE_ROLE_USER. Configure a deliberate converter/authority contract instead of assuming JWT scopes are automatically roles.

#### 33. Is URL username sufficient to protect ownership?
No: it is caller-controlled input, and individual GET/DELETE ignore it in current repository lookups. Scope queries and mutation policy to the verified principal, with consistent missing/forbidden behavior.

#### 34. What is wrong with the current PUT implementation?
It saves the body entity without enforcing that body ID/username matches the path and authenticated caller. Validate or assign target identity server-side and prevent unintended creation or cross-record updates.

#### 35. Why does a missing todo currently look like a server failure?
The active retrieval method calls Optional.get() without checking absence. A deliberate not-found mapping would produce a clearer 404 contract instead of an unhandled exception.

#### 36. Why can curl show 401 while the browser shows a CORS error?
Security may reject the actual request before MVC adds CORS headers. Integrate CORS into the Security layer and inspect both the preflight and actual response.

#### 37. Should access tokens always go in localStorage?
No: localStorage is persistent and accessible to injected JavaScript. Choose memory, secure cookie/BFF, or another deliberate design based on XSS, CSRF, and persistence requirements.

#### 38. Does local logout revoke a JWT?
No: the existing signed token can remain valid until expiry unless additional revocation logic exists. This module implements neither a revocation list nor a server logout/token invalidation endpoint.

#### 39. Why might editing a completed todo mark it incomplete?
TodoComponent always submits done:false rather than preserving the retrieved done value. Initialize and submit all relevant editable/preserved fields instead of hard-coding unrelated state.

#### 40. What would you test first before deploying this app?
Principal-scoped ownership, path/body ID enforcement, auth/expiry, CORS failure paths, and server validation. Also cover interceptor cleanup, form behavior, correct statuses, persistence, and concurrent updates.

## Hands-on exercises
These exercises operate against the current local application or propose changes in a learner's study copy. No existing application files were modified in producing this guide. Do not use production credentials/tokens in shell history or share decoded tokens with third-party tools.

### Exercise 1 — Observe preflight before browser login
Start both processes and open the browser's Network panel. Submit the demonstration login and inspect OPTIONS and POST `/authenticate`. Then reproduce the preflight with curl:
```bash
BASE=http://localhost:8080
curl -i -X OPTIONS "$BASE/authenticate"   -H 'Origin: http://localhost:3000'   -H 'Access-Control-Request-Method: POST'   -H 'Access-Control-Request-Headers: content-type'
curl -i -X OPTIONS "$BASE/users/in28minutes/todos"   -H 'Origin: http://localhost:3000'   -H 'Access-Control-Request-Method: GET'   -H 'Access-Control-Request-Headers: authorization'
```

Expected: permitted preflight with the configured localhost origin and appropriate allowed values. Repeat with `Origin: http://127.0.0.1:3000` and explain the rejected-origin behavior. Remember: curl prints the response; it does not enforce the browser CORS read restriction.

### Exercise 2 — Login, inspect JWT locally, and make authenticated requests
Capture only the token using Python JSON parsing:
```bash
TOKEN="$(curl -sS -H 'Content-Type: application/json'   -d '{"username":"in28minutes","password":"dummy"}'   "$BASE/authenticate" | python3 -c 'import json,sys; print(json.load(sys.stdin)["token"])')"
curl -i -H "Authorization: Bearer $TOKEN"   "$BASE/users/in28minutes/todos"
curl -i -H "Authorization: Bearer $TOKEN"   "$BASE/users/in28minutes/todos/10001"
curl -i "$BASE/users/in28minutes/todos"
```

Expect 200 for valid protected reads and normally 401 for the unauthenticated API request. Decode the header/payload **locally**, without claiming this verifies the signature:
```bash
printf '%s' "$TOKEN" | python3 -c 'import base64,json,sys; parts=sys.stdin.read().split("."); [print(json.dumps(json.loads(base64.urlsafe_b64decode(p + "=" * (-len(p)%4))),indent=2)) for p in parts[:2]]'
```

Inspect `iss`, `sub`, `scope`, and `exp - iat` around 5400 seconds. Do not post the token to an online decoder. A simple invalid bearer smoke test:
```bash
curl -i -H 'Authorization: Bearer not-a-valid-jwt'   "$BASE/users/in28minutes/todos"
```

### Exercise 3 — Follow the actual todo CRUD contract
Create and capture the generated ID; POST currently returns 200 with the entity:
```bash
NEW_TODO="$(curl -sS -H "Authorization: Bearer $TOKEN"   -H 'Content-Type: application/json'   -d '{"description":"Practice full stack flows","targetDate":"2030-06-01","done":false}'   "$BASE/users/in28minutes/todos")"
NEW_ID="$(printf '%s' "$NEW_TODO" | python3 -c 'import json,sys; print(json.load(sys.stdin)["id"])')"
curl -i -H "Authorization: Bearer $TOKEN"   "$BASE/users/in28minutes/todos/$NEW_ID"
curl -i -X PUT -H "Authorization: Bearer $TOKEN"   -H 'Content-Type: application/json'   -d "{\"id\":$NEW_ID,\"username\":\"in28minutes\",\"description\":\"Practice updated full stack flows\",\"targetDate\":\"2030-07-01\",\"done\":true}"   "$BASE/users/in28minutes/todos/$NEW_ID"
curl -i -X DELETE -H "Authorization: Bearer $TOKEN"   "$BASE/users/in28minutes/todos/$NEW_ID"
```

Observe 200 for update and 204 with an empty body for delete. Retrieve after deletion to observe the current missing-ID behavior and explain the Optional.get() issue. In a study copy, add a deliberate 404 and a 201 + Location creation response with tests.

### Exercise 4 — Repair token/interceptor lifecycle in a study copy
Log in, log out, and log in again while observing request headers in the Network panel. Record that old interceptor closures can survive even after AuthContext clears its token state. Replace repeated registration with a single controlled interceptor and an explicit cleanup strategy. Add a response-error policy for expired tokens without infinite retry loops.
```bash
# Basic still works in the active JWT chain:
curl -i -u in28minutes:dummy "$BASE/basicauth"
# Bad credentials should not issue a token:
curl -i -H 'Content-Type: application/json'   -d '{"username":"in28minutes","password":"wrong"}' "$BASE/authenticate"
```

Also restart only the backend and demonstrate that an old JWT fails under the new generated key. Check that the improved frontend transitions to a login-required state when this happens. A new login should succeed without being poisoned by stale Authorization on `/authenticate`.

### Exercise 5 — Add validation, ownership, and robust editor behavior
In a study copy, add DTO validation and a transactional ownership-scoped service. Reject or normalize mismatched PUT identity; do not trust a body username for permissions. Add a second local test user and tests proving one user cannot read/update/delete another user's todo. Preserve done state in the editor and reset description/date when entering `/todo/-1`.

A baseline demonstration of missing backend validation, using only newly created local test data:
```bash
curl -i -H "Authorization: Bearer $TOKEN"   -H 'Content-Type: application/json'   -d '{"description":"x","targetDate":"2030-06-01","done":false}'   "$BASE/users/in28minutes/todos"
```

The backend currently accepts a short description even though Formik would reject it. After adding server constraints in your study copy, expect a deliberate 400 and structured field errors. Test those cases directly at the HTTP layer as well as through the browser. Clean up only the records you created in the exercise, using their returned generated IDs.

## Quick revision cheat sheet
| Topic | Key fact |
|---|---|
| Paths | Backend 13-full-stack/02-rest-api; frontend 13-full-stack/01-frontend/todo-app |
| Run | Maven spring-boot:run; npm ci when needed; npm start |
| Local ports | React 3000; Spring 8080; different origins |
| Data | Active JPA/H2 rows 10001–10003; old static service is not active storage |
| Active controller | TodoJpaResource; TodoResource annotation commented out |
| Active security | JwtSecurityConfig; Basic-only Configuration annotation commented out |
| Demo login | in28minutes / dummy, with {noop}; classroom-only |
| Login endpoint | POST /authenticate → AuthenticationManager → JwtTokenService → 200 {token} |
| Token lifetime | 90 minutes |
| JWT shape | base64url(header).base64url(payload).base64url(signature) |
| JWT visibility | Signed does not mean encrypted; payload is readable |
| RSA | Private key signs; public key verifies |
| Nimbus | Encoder uses JWKSource; decoder trusts configured public key |
| Key lifetime | Generated per application context; restart breaks old tokens |
| Resource server | Bearer extraction + JWT authentication, not a full OAuth authorization server |
| Sessions | STATELESS security; each request must authenticate |
| CSRF | Disabled in active chain; assess auto-attached credentials separately |
| Basic | Also enabled in active JWT chain |
| Authorities | roles replaces earlier authorities; JWT scopes get SCOPE_ prefix by default |
| Issuer/audience | self is issued; explicit issuer/audience validators are not configured |
| CORS | Exact localhost:3000 allowlist in WebMvcConfigurer |
| Preflight | OPTIONS usually before JSON POST or Authorization-bearing requests |
| CORS integration | OPTIONS allowed; Security cors(...) not explicit in source |
| AuthContext | UI auth state in memory, not proof of server validity |
| Interceptors | Source accumulates request interceptors; logout does not eject them |
| Storage | No active localStorage, sessionStorage, refresh token, or server logout |
| Router | Guard is UX; backend must enforce ownership |
| useState | Render snapshot + scheduled state update |
| useEffect | External synchronization; dependency and cleanup correctness matter |
| Formik | initialValues + enableReinitialize + validate + Field/ErrorMessage |
| Server validation | No Todo @Valid/constraints equivalent to Formik in current backend |
| CRUD statuses | GET/POST/PUT 200; DELETE 204; missing GET ID can be 500 |
| Identity gap | Individual routes ignore owner; PUT trusts body ID |
| Editor gap | done:false on every save; reset new-item state deliberately |
| Production priorities | TLS, durable keys, scoped ownership, validation, safe error handling, tests |

Final interview habit: trace **browser behavior**, **HTTP policy**, **security verification**, and **database access** separately, then explain how their contracts fit together.
