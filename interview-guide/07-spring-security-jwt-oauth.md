# 07 — Spring Security, JWT, OAuth 2.0, and OpenID Connect
## What this module teaches
This guide connects the course examples to Spring Boot **3.3.4**, Java **21**, and Spring Security **6.3**.
It distinguishes the configuration that actually runs from commented-out alternatives.
Treat the demonstration credentials, ephemeral keys, and relaxed headers as learning aids—not production defaults.

### Exact module paths
- `71-spring-security/`: JDBC users, Basic authentication, method authorization, and optional JWT examples.
- `72-oauth/`: browser login through Google's OAuth/OIDC integration.
- `spring-security-3.1.4-updates.md`: historical Spring Boot 3.1.x matcher migration notes.

All paths below are relative to the repository root.
The examples use the servlet stack, not reactive WebFlux security.

### How to run
Open a terminal at the repository root:

```bash
cd 71-spring-security
./gradlew bootRun
```

Run the OAuth example separately:

```bash
cd 72-oauth
./gradlew bootRun
```

Both web applications normally use port `8080`.
Do not run both on that port simultaneously.
To choose another port without editing properties:

```bash
./gradlew bootRun --args='--server.port=8081'
```

Tests, when you want to validate a local experiment:

```bash
./gradlew test
```

The Java toolchain in both `build.gradle` files requests Java 21.
A suitable JDK and working Gradle distribution/dependency access are prerequisites.
Commands here are instructions for study; this documentation task does not execute the applications.

### What is active right now?
| Feature | Repository state | Consequence |
|---|---|---|
| Basic configuration | `@Configuration` active | Basic authentication protects every request |
| Method security | Enabled in Basic configuration | All annotation families used in `TodoResource` are enabled |
| JWT configuration | `//@Configuration` commented out | RSA/JWT beans are not registered |
| JWT controller | `//@RestController` commented out | `/authenticate` is not currently an endpoint |
| CSRF protection in module 71 | Explicitly disabled | `/csrf-token` has no CSRF token to return |
| Form login in module 71 | Commented out | No generated form-login flow from this configuration |
| Google OAuth login in module 72 | Active | Real provider credentials are required for successful login |

## Project walkthrough
### 1. Build and application bootstrap
File: `71-spring-security/build.gradle`.

```groovy
id 'org.springframework.boot' version '3.3.4'
```

The Java toolchain requests:

```groovy
languageVersion = JavaLanguageVersion.of(21)
```

Relevant dependencies copied from the build:

```groovy
implementation 'org.springframework.boot:spring-boot-starter-security'
implementation 'org.springframework.boot:spring-boot-starter-jdbc'
implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
implementation 'com.h2database:h2'
testImplementation 'org.springframework.security:spring-security-test'
```

The web starter supplies Spring MVC and an embedded servlet container.
The resource-server starter supplies bearer-token/JWT support, but a dependency alone does not activate the commented JWT configuration.
File: `71-spring-security/src/main/java/com/in28minutes/learnspringsecurity/LearnSpringSecurityApplication.java`.

```java
@SpringBootApplication
public class LearnSpringSecurityApplication {
```

Its `main` method starts the application with `SpringApplication.run`.
Component scanning starts under `com.in28minutes.learnspringsecurity`.
That scan discovers the active Basic configuration and resource controllers.

### 2. Active Basic authentication configuration
File: `71-spring-security/src/main/java/com/in28minutes/learnspringsecurity/basic/BasicAuthSecurityConfiguration.java`.

```java
@Configuration
@EnableMethodSecurity(jsr250Enabled = true, securedEnabled = true)
public class BasicAuthSecurityConfiguration {
```

The configuration exposes a `SecurityFilterChain` bean.
Its active authorization rule is:

```java
http.authorizeHttpRequests(
                auth -> {
                    auth
                    .anyRequest().authenticated();
                });
```

No route is public through this rule.
`GET /hello-world`, `GET /todos`, and `GET /csrf-token` all require authentication.
More specific restrictions can then apply through method security.
The session policy and Basic authentication are:

```java
http.sessionManagement(
                session ->
                    session.sessionCreationPolicy(
                            SessionCreationPolicy.STATELESS)
                );
```

```java
http.httpBasic(withDefaults());
http.csrf(csrf -> csrf.disable());
```

The application also disables frame-options headers globally:

```java
http.headers(headers -> headers.frameOptions(frameOptionsConfig-> frameOptionsConfig.disable()));
```

This is not a recommended production hardening pattern.
If you need an H2 console locally, prefer a narrowly scoped development-only configuration.
An H2 dependency does not itself enable the web console.

### 3. JDBC schema and seeded users
The same configuration creates its own embedded data source:

```java
return new EmbeddedDatabaseBuilder()
        .setType(EmbeddedDatabaseType.H2)
        .addScript(JdbcDaoImpl.DEFAULT_USER_SCHEMA_DDL_LOCATION)
        .build();
```

There is no application-owned `schema.sql` in this module's main resources.
The schema is loaded from Spring Security's bundled JDBC schema resource.
It provides `users` and `authorities` tables suitable for the default JDBC queries.
Typical columns are `username`, `password`, and `enabled` in `users`;
`authorities` associates usernames with authority strings.
The user builder demonstrates encoding before persistence:

```java
var user = User.withUsername("in28minutes")
    //.password("{noop}dummy")
    .password("dummy")
    .passwordEncoder(str -> passwordEncoder().encode(str))
    .roles("USER")
    .build();
```

The other user is `admin`, also with password `dummy`, and roles `ADMIN` and `USER`.
These are deliberately public course credentials; do not copy them into a deployed system.

```java
var jdbcUserDetailsManager = new JdbcUserDetailsManager(dataSource);
jdbcUserDetailsManager.createUser(user);
jdbcUserDetailsManager.createUser(admin);
```

`JdbcUserDetailsManager` implements both user lookup and user-management operations.
The bean is declared with the `UserDetailsService` interface as its return type.
A commented alternative shows `InMemoryUserDetailsManager`; it is not active.

```java
@Bean
public BCryptPasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

File: `71-spring-security/src/main/resources/application.properties`.

```properties
spring.main.banner-mode=off
logging.pattern.console= %d{MM-dd HH:mm:ss} - %logger{36} - %msg%n
spring.datasource.url=jdbc:h2:mem:testdb
```

Important: the explicit `DataSource` bean uses `EmbeddedDatabaseBuilder`.
The property alone does not rename or configure that explicitly built database.
Do not infer its effective JDBC URL solely from `spring.datasource.url`.
The embedded users disappear when that in-memory database/application is destroyed.

### 4. Hello-world and CSRF demonstration endpoints
File: `71-spring-security/src/main/java/com/in28minutes/learnspringsecurity/resources/HelloWorldResource.java`.

```java
@GetMapping("/hello-world")
public String helloWorld() {
    return "Hello World v1";
}
```

The controller itself has no authorization annotation.
The filter-chain rule still protects it.
File: `71-spring-security/src/main/java/com/in28minutes/learnspringsecurity/resources/SpringSecurityPlayResource.java`.

```java
@GetMapping("/csrf-token")
public CsrfToken retrieveCsrfToken(HttpServletRequest request) {
    return (CsrfToken) request.getAttribute("_csrf");
}
```

When CSRF support is active, Spring exposes a `CsrfToken` as a request attribute.
The controller does not create tokens independently of the security filters.
With the current disabled-CSRF configuration, the attribute is normally absent.
Consequently, do not expect token JSON from this endpoint until CSRF is enabled.

### 5. Todo endpoints and method authorization
File: `71-spring-security/src/main/java/com/in28minutes/learnspringsecurity/resources/TodoResource.java`.
The data is a fixed list, not a database-backed todo store:

```java
private static final List<Todo> TODOS_LIST =
        List.of(new Todo("in28minutes", "Learn AWS"),
                new Todo("in28minutes", "Get AWS Certified"));
```

`GET /todos` returns this entire list to any authenticated caller.
The single-user GET combines four annotation families:

```java
@GetMapping("/users/{username}/todos")
@PreAuthorize("hasRole('USER') and #username == authentication.name")
@PostAuthorize("returnObject.username == 'in28minutes'")
@RolesAllowed({"ADMIN", "USER"})
@Secured({"ROLE_ADMIN", "ROLE_USER"})
public Todo retrieveTodosForSpecificUser(@PathVariable("username") String username) {
    return TODOS_LIST.get(0);
}
```

`@PreAuthorize` checks the user's role and requested username before execution.
`@PostAuthorize` checks the returned object after execution.
The result is always the first todo, whose username is `in28minutes`.
Thus `admin` requesting `/users/admin/todos` can pass these demonstration checks,
even though the returned todo belongs to `in28minutes`.
This is not correct general-purpose ownership enforcement.
Multiple enabled annotation families impose cumulative constraints; they are not alternatives.
An `ADMIN` role alone does not satisfy the explicit `hasRole('USER')` expression.

```java
@PostMapping("/users/{username}/todos")
public void createTodoForSpecificUser(@PathVariable String username
        , @RequestBody Todo todo) {
    logger.info("Create {} for {}", todo, username);
}
```

This POST only logs a request; it does not persist data or check username ownership.
Its current restriction is authentication at the HTTP layer.
The package-private record defines the request/response shape:

```java
record Todo (String username, String description) {}
```

### 6. Optional JWT filter chain and signing keys
File: `71-spring-security/src/main/java/com/in28minutes/learnspringsecurity/jwt/JwtSecurityConfiguration.java`.

```java
//@Configuration
public class JwtSecurityConfiguration {
```

It has the same JDBC users, BCrypt encoder, stateless policy, Basic auth, and relaxed CSRF/frame settings.
Its additional authentication mechanism is:

```java
http.oauth2ResourceServer(oauth2 -> oauth2.jwt(withDefaults()));
```

The signing key pair is generated at startup:

```java
var keyPairGenerator = KeyPairGenerator.getInstance("RSA");
keyPairGenerator.initialize(2048);
return keyPairGenerator.generateKeyPair();
```

The RSA JWK includes a public key, private key, and random key identifier:

```java
return new RSAKey
        .Builder((RSAPublicKey)keyPair.getPublic())
        .privateKey(keyPair.getPrivate())
        .keyID(UUID.randomUUID().toString())
        .build();
```

A `JWKSet` is adapted to a `JWKSource<SecurityContext>` for the encoder.
That Nimbus `SecurityContext` type is not Spring's `SecurityContextHolder` context.

```java
return NimbusJwtDecoder
        .withPublicKey(rsaKey.toRSAPublicKey())
        .build();
```

```java
return new NimbusJwtEncoder(jwkSource);
```

The decoder verifies using the public key; the encoder signs using the private key.
There is no HTTP JWKS endpoint simply because a `JWKSource` bean exists.
Keys rotate unintentionally on each restart in this example.
For production, securely persist/manage keys and plan intentional rotation with overlap.

### 7. Optional JWT issuance controller
File: `71-spring-security/src/main/java/com/in28minutes/learnspringsecurity/jwt/JwtAuthenticationResource.java`.

```java
//@RestController
public class JwtAuthenticationResource {
```

```java
@PostMapping("/authenticate")
public JwtResponse authenticate(Authentication authentication) {
    return new JwtResponse(createToken(authentication));
}
```

This accepts the already authenticated principal, not a username/password JSON login object.
In the optional JWT chain, Basic authentication can authenticate this POST.
It is not an OAuth authorization server or standards-compliant token endpoint.

```java
var claims = JwtClaimsSet.builder()
                        .issuer("self")
                        .issuedAt(Instant.now())
                        .expiresAt(Instant.now().plusSeconds(60 * 30))
                        .subject(authentication.getName())
                        .claim("scope", createScope(authentication))
                        .build();
```

Tokens expire after approximately 30 minutes.
The code joins existing authorities with spaces for the `scope` claim:

```java
return authentication.getAuthorities().stream()
    .map(a -> a.getAuthority())
    .collect(Collectors.joining(" "));
```

A Basic-authenticated `ROLE_USER` therefore becomes `scope: "ROLE_USER"`.
Default resource-server conversion maps that to **`SCOPE_ROLE_USER`**, not `ROLE_USER`.
Consequently a bearer token may pass HTTP authentication but fail `hasRole('USER')`.
Resolve this deliberately with a claim/authority design and `JwtAuthenticationConverter`.
Do not silently equate OAuth scopes with application roles.

### 8. OAuth client configuration
File: `72-oauth/build.gradle`.

```groovy
implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'
implementation 'org.springframework.boot:spring-boot-starter-web'
```

File: `72-oauth/src/main/java/com/in28minutes/learnoauth/OauthSecurityConfiguration.java`.

```java
@Configuration
public class OauthSecurityConfiguration {
```

```java
@Bean
@Order(SecurityProperties.BASIC_AUTH_ORDER)
SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {
```

```java
http.authorizeHttpRequests(request -> request.anyRequest().authenticated());
http.oauth2Login(Customizer.withDefaults());
return http.build();
```

The explicit chain replaces reliance on Boot's default web chain.
Its `@Order` establishes precedence if multiple chains are present.
With only one chain, it does not create additional authorization rules.
There is no stateless session policy here: browser login normally uses a session.
CSRF and standard security headers remain enabled unless otherwise configured.
File: `72-oauth/src/main/resources/application.properties`.

```properties
spring.security.oauth2.client.registration.google.client-id=YOUR_CLIENT_ID
spring.security.oauth2.client.registration.google.client-secret=YOUR_SECRET
```

These values are placeholders, not working credentials.
Use environment variables rather than committing a real secret:

```bash
export SPRING_SECURITY_OAUTH2_CLIENT_REGISTRATION_GOOGLE_CLIENT_ID='your-client-id'
export SPRING_SECURITY_OAUTH2_CLIENT_REGISTRATION_GOOGLE_CLIENT_SECRET='your-client-secret'
./gradlew bootRun
```

Configure Google's registered redirect URI to match the application's external URL:
`http://localhost:8080/login/oauth2/code/google` for this local example.
The login initiation endpoint is `/oauth2/authorization/google`.
Google provider defaults are inferred from the `google` registration name.
The common Google defaults include OpenID Connect scopes, including `openid`.

### 9. OAuth principal and application entry point
File: `72-oauth/src/main/java/com/in28minutes/learnoauth/HelloWorldResource.java`.

```java
@GetMapping("/")
public String helloWorld(Authentication authentication) {
    System.out.println(authentication);
    System.out.println(authentication.getPrincipal());
    return "Hello World";
}
```

The source declares `Authentication`, not `OAuth2AuthenticationToken` explicitly.
After successful `oauth2Login`, the runtime authentication is normally an `OAuth2AuthenticationToken`.
Its principal is an `OAuth2User`, or an `OidcUser` for OIDC login.
The principal holds provider attributes/claims; it is not the raw access token.
Production logging should avoid dumping claims, tokens, or personally identifiable information.
File: `72-oauth/src/main/java/com/in28minutes/learnoauth/LearnOauthApplication.java`.
The `@SpringBootApplication` entry point scans the OAuth package and starts the MVC application.

### 10. Historical matcher update document
File: `spring-security-3.1.4-updates.md` explicitly says:

> These changes are NOT NEEDED for 3.2.X and above!

It discusses the Spring Boot 3.1.x multi-servlet matcher situation and CVE-2023-34035 mitigations.
Its examples distinguish MVC endpoints from the H2 console servlet using explicit matchers.
Do not mechanically paste its historical “TO-BE” configuration into this Boot 3.3.4 project.
In particular, those snippets permit endpoints that the current source protects.
Keep servlet boundaries and matcher intent clear even when framework versions fix historical ambiguity.

## Core concepts explained
### Authentication versus authorization
| Question | Authentication | Authorization |
|---|---|---|
| Meaning | Who are you? | What may you do? |
| Inputs | Password, session, certificate, bearer token | Principal, authorities, resource, policy |
| Typical failure | `401 Unauthorized` for an API | `403 Forbidden` for an authenticated API caller |
| Repo example | Basic password verification | `hasRole('USER')` and username comparison |

Authentication is not ownership validation.
A valid `admin` login does not automatically justify reading every user's data.
Browser authentication failures can redirect to login rather than return an API-style 401.
Anonymous access denial can trigger authentication entry points rather than a simple 403.

### Security principles
| Principle | Practical application |
|---|---|
| Least privilege | Give only the scopes/roles a task needs |
| Deny by default | Require explicit grants and a restrictive fallback rule |
| Defense in depth | TLS, filter authorization, service authorization, validation, and audit |
| Complete mediation | Check every access path, not only a visible button |
| Fail securely | Reject access when identity/validation is uncertain |
| Separation of duties | Separate token issuance, consumption, and administration |
| Minimize attack surface | Remove unused endpoints and development consoles |
| Secure defaults | Retain CSRF/headers unless a justified design replaces them |

Never trust usernames in URLs or request bodies as proof of identity.
Validate ownership using the authenticated principal and actual stored resource.
Security-sensitive changes need negative tests, not merely happy-path curl calls.

### Servlet filter-chain architecture
```text
HTTP request
   |
Servlet container filter registration
   |
DelegatingFilterProxy("springSecurityFilterChain")
   | looks up a Spring-managed Filter bean
FilterChainProxy
   | selects FIRST matching SecurityFilterChain
   +--> Chain 1: /api/**       [API filters]
   +--> Chain 2: everything    [browser filters]
             |
      selected filters in framework-defined order
             |
       DispatcherServlet --> controller --> service proxy
                                               |
                                      method authorization
```

`DelegatingFilterProxy` bridges servlet-container registration and Spring dependency injection.
`FilterChainProxy` orchestrates Spring Security's chains and request firewall handling.
A `SecurityFilterChain` contains a request matcher and ordered filters.
Matching chains are not all executed; the first matching chain wins.
Within a chain, request authorization rules also use first matching rule semantics.
Illustrative filter roles—not a universal exhaustive order:

| Filter | Responsibility |
|---|---|
| `SecurityContextHolderFilter` | Load/defer the context and clear thread-local state afterward |
| `HeaderWriterFilter` | Write security response headers |
| `CorsFilter`, when configured | Handle CORS checks/preflight early |
| `CsrfFilter`, when enabled | Validate CSRF protection for unsafe requests |
| `UsernamePasswordAuthenticationFilter`, with form login | Process username/password form submissions |
| `BasicAuthenticationFilter` | Process an HTTP Basic header |
| `BearerTokenAuthenticationFilter`, with resource server | Resolve and authenticate bearer tokens |
| `AnonymousAuthenticationFilter` | Represent unauthenticated callers as anonymous |
| `ExceptionTranslationFilter` | Translate downstream authentication/access denial failures |
| `AuthorizationFilter` | Apply `authorizeHttpRequests` authorization rules |

OAuth login adds authorization-request redirect and login authentication filters.
Only configured mechanisms contribute their corresponding filters.
Use framework diagnostics to inspect exact filter composition; do not memorize one chain for every app.
With Spring Security 6, context persistence involves explicit-save semantics by default.
Built-in authentication flows handle their context repository integration.
A custom login implementation must not assume merely setting the holder saves a session.

### SecurityContextHolder and Authentication
```text
SecurityContextHolder
    -> SecurityContext
         -> Authentication
              -> principal
              -> credentials (usually erased after success)
              -> authorities
              -> authenticated state
```

The default storage strategy is thread-local.
Servlet request cleanup prevents one user's identity leaking into the next request on a reused thread.
A controller's `Authentication` parameter is a convenient access point.
Do not assume a thread-local context follows arbitrary executor, async, or scheduled work.
Use Spring's security-context-aware wrappers/integration for required propagation.
`isAuthenticated()` alone can be misleading because an anonymous token is also marked authenticated.
Use authorization APIs or an authentication trust resolver to distinguish anonymous authentication.

### Default form login
With the security starter and no replacing configuration, Boot commonly secures all endpoints,
creates a development user with a generated password, and supplies form login/Basic behavior.
Those defaults back off when the relevant custom security/user beans exist.
In this repo, custom JDBC users and the custom Basic chain are active.
To experiment with form login, the conceptual DSL is:

```java
http.formLogin(withDefaults());
```

This enables a generated login page unless you provide a custom one.
The login form includes a CSRF token when CSRF protection is enabled.
A successful browser flow usually stores the context in an HTTP session.
Session fixation protection changes the session identifier on authentication.
A stateless Basic chain is not a useful substitute for a stateful form-login design.

### HTTP Basic authentication
```text
Authorization: Basic base64(username:password)
```

Base64 is encoding, not encryption.
The credentials are presented repeatedly, so TLS is mandatory outside disposable local experiments.
The server decodes the header and authenticates through an `AuthenticationManager`/provider.
A typical `DaoAuthenticationProvider` loads a `UserDetails` and checks the password encoder.
Basic authentication does not imply a session, and a stateless policy can force per-request authentication.
Browser-managed Basic credentials may be sent automatically, which matters for CSRF.

### CSRF: the browser's ambient-credential problem
```text
User logs into trusted.example -> browser holds session cookie
User visits attacker.example
Attacker page submits POST to trusted.example/transfer
Browser attaches trusted.example cookie automatically
Server sees a valid session, but user never intended the transfer
```

CSRF exploits automatic credential attachment, not the theft of the password.
A synchronizer token ties an unpredictable value to the legitimate session/client workflow.
The legitimate page receives it and sends it in a hidden form field or request header.
The attacker cannot normally read it through the same-origin policy.
The server verifies it before accepting unsafe state-changing operations.
Spring's default servlet CSRF setup uses a session-backed token repository.
The default parameter is `_csrf`; the default header is `X-CSRF-TOKEN`.
Token loading is deferred, and Spring Security 6 includes BREACH-mitigation token handling.
A cookie-backed setup requires understanding `CookieCsrfTokenRepository` and its expected header.
Do not mix cookie/session recipes without checking their token request-handler behavior.

| Request/client design | Typical CSRF stance |
|---|---|
| Browser session cookie | Keep protection enabled |
| Browser automatically supplied Basic credentials | Still potentially vulnerable |
| Bearer token only in explicitly set `Authorization` header | Often reasonable to disable for that API chain |
| JWT stored in a cookie | Still a CSRF concern |
| GET that changes data | Fix the endpoint semantics; do not rely on GET CSRF checks |

`STATELESS` is not evidence that CSRF is impossible.
Disabling CSRF requires analyzing how the browser authenticates requests.
CORS is not a replacement for CSRF protection.
XSS can bypass many CSRF defenses by running inside the trusted origin.
Cookie `SameSite` options:

| Setting | Behavior summary |
|---|---|
| `Strict` | Restricts cross-site cookie sending most aggressively |
| `Lax` | Allows certain top-level navigations; not arbitrary cross-site unsafe requests |
| `None` | Allows cross-site sending and requires `Secure` in modern browsers |

SameSite is useful defense in depth, not a complete authorization/CSRF design.
Same-site and same-origin are different boundaries; sibling subdomains deserve attention.
OAuth redirects may require a cookie policy compatible with the chosen callback flow.

### CORS: permission to read cross-origin responses
An origin is the scheme, host, and port tuple.
CORS is enforced primarily by browsers, not by curl or backend HTTP clients.
A preflight `OPTIONS` request asks whether a method/header is allowed.
It usually lacks the credentials needed by a blanket authentication rule.
Configure CORS early through Spring Security integration and a `CorsConfigurationSource`.
Conceptual DSL, not present in the current Basic configuration:

```java
http.cors(withDefaults());
```

A custom MVC CORS configuration can also participate when appropriately integrated.
Use explicit origins for credentialed requests; `*` is not valid with credentialed wildcard origin access.
Allow only necessary methods/headers.
Permitting `OPTIONS` alone does not generate the necessary CORS response headers.
Neither source module currently contains an explicit CORS policy.

### Sessions versus stateless authentication
| Concern | Session | Stateless bearer JWT |
|---|---|---|
| Client sends | Session identifier | Token |
| Identity state | Server-side context/session | Usually reconstructed from token per request |
| Logout/revocation | Invalidate server session | Token remains valid unless additional mechanisms apply |
| Scaling | Shared/sticky session strategy may be needed | Shared verification keys/configuration |
| Main browser concern | Cookie theft, fixation, CSRF | Token theft/XSS; CSRF if cookie-carried |

`SessionCreationPolicy.STATELESS` means Spring Security neither creates nor uses a session
as the source of its security context.
It does not guarantee that no other application component creates an HTTP session.
Stateless does not mean “no database” or “no caches”.
Module 72 normally needs session state for authorization-request correlation and post-login browsing.

### User storage and password encoding
| Component | Purpose |
|---|---|
| `UserDetailsService` | Load a user by username; not itself a password verifier |
| `InMemoryUserDetailsManager` | Keep users in process memory |
| `JdbcUserDetailsManager` | Load/manage users through JDBC |
| `DaoAuthenticationProvider` | Combine user lookup and password verification |
| `PasswordEncoder` | Encode passwords and check raw input against stored encoding |

Plaintext storage exposes every password after a database breach.
Fast unsalted hashes such as raw SHA-256 enable very fast password guessing.
BCrypt is salted and deliberately expensive; identical raw passwords normally produce different encodings.
Verify with `matches(raw, encoded)`, not `encode(raw).equals(encoded)`.
Tune the cost using actual server performance and rate-limit authentication attempts.
BCrypt has an input-length limit of 72 bytes; establish a deliberate password-length/algorithm policy.
`DelegatingPasswordEncoder` supports encodings such as `{bcrypt}...` and migration across algorithms.
`PasswordEncoderFactories.createDelegatingPasswordEncoder()` defaults to BCrypt for new encodings.
The repo directly uses `BCryptPasswordEncoder`, so its BCrypt strings do not require a `{bcrypt}` prefix.
A delegating encoder needs a recognized `{id}` prefix or an explicitly configured fallback.
`{noop}` is only for disposable demonstrations, not a production password storage option.
Passwords are hashed for verification, not encrypted for later recovery.

### JWT anatomy and validation
```text
base64url(header).base64url(payload).base64url(signature)
       metadata          claims           integrity proof
```

A signed JWT is normally readable by anyone holding it.
Signing does not encrypt claims.
Never put passwords or unnecessary sensitive personal information into a readable token.
A JWT is a format; it is not automatically an OAuth access token or a login system.

| Claim/header | Meaning | Repo usage |
|---|---|---|
| `alg` | Signing algorithm | RSA-based encoder/decoder setup |
| `kid` | Key identifier | Random UUID on the RSA JWK |
| `iss` | Issuer | `self` |
| `sub` | Subject | Authentication name |
| `iat` | Issued at | Current instant |
| `exp` | Expiration | Now + 30 minutes |
| `aud` | Intended audience | Not set |
| `scope` | Space-separated scopes | Existing authority strings joined together |
| `jti` | Token identifier | Not set |

| Signing style | Verification | Trade-off |
|---|---|---|
| Symmetric, e.g. HMAC | Same secret signs and verifies | Every verifier with the secret can forge tokens |
| Asymmetric, e.g. RSA | Private key signs, public key verifies | Verifiers need not possess signing capability |

A JWK is a JSON representation of cryptographic key material and metadata.
A JWKS is a set of JWKs, often published by an authorization server.
Publish only public material; the repo's encoder-side JWK includes the private key.
A `kid` helps choose the verification key during rotation.
Maintain old public keys until tokens signed with them have expired, with clock skew considered.
Verification must check allowed algorithm, signature, timestamps, expected issuer, and audience as applicable.
The repo's public-key decoder provides default timestamp validation, not an application-specific audience rule.
Setting `iss: self` in issuance does not itself configure issuer validation in the decoder.
Add `JwtValidators.createDefaultWithIssuer(...)` and a suitable audience validator in a real system.
Never accept a token's unverified claims as authorization facts.
Expiry limits the theft window; it does not provide immediate logout or revocation.
Refresh tokens, if used, need a separately secured issuance/storage/rotation design.
This repository does not implement refresh tokens.

### OAuth 2.0 roles and grant flows
| Role | Job | Module 72 interpretation |
|---|---|---|
| Resource owner | Grants access | Browser user |
| Client | Requests access | This Spring application |
| Authorization server | Authenticates/consents and issues tokens | Google |
| Resource server | Serves protected resources using access tokens | Google APIs, or a separately configured API |

Authorization-code flow:

```text
Browser -> Spring client -> authorization redirect -> Google
                                                   |
                                         login + consent
                                                   |
Browser <- redirect with code + state <- Google
    |
Spring callback -> server-to-server token exchange -> Google
    |
Spring validates login result and creates local authenticated session
```

The short-lived code is not the access token.
`state` correlates the authorization response with the initiating browser flow.
Registered redirect URIs prevent arbitrary redirection of authorization responses.
Client secrets belong on confidential server clients, not inside browser/mobile apps.
PKCE adds a code verifier and derived challenge:

```text
Client keeps random verifier
Authorization request sends S256(verifier) as challenge
Token request sends verifier
Authorization server checks verifier matches challenge
```

PKCE protects against stolen/intercepted authorization codes.
It is essential for public clients and useful defense in depth for confidential clients.
It does not replace redirect-URI checking, `state`, or a full threat model.
Do not assume a confidential client configuration enables every desired PKCE behavior automatically.
Configure and verify the authorization request when PKCE is required by your provider/policy.
Client credentials flow is for machine-to-machine access:

```text
Backend client + client authentication -> token endpoint -> access token
Backend client + access token -> resource server
```

There is no interactive end-user identity in this grant.
`oauth2Login` is not the configuration used to obtain a machine token with this grant.
Avoid the legacy implicit flow and resource-owner-password grant for new designs.
Use standards-compliant authorization servers rather than expanding the demo token controller ad hoc.

### OpenID Connect and Spring's two OAuth entry points
OIDC is an identity layer built on OAuth 2.0.
Requesting `openid` introduces ID-token and identity semantics.
An ID token represents authentication for its client audience.
An access token authorizes access to a resource server.
A resource server must not treat an arbitrary ID token as its access token.
OIDC validation includes issuer, audience, signature, timestamps, and nonce when used by the flow.

| Feature | `oauth2Login` | `oauth2ResourceServer` |
|---|---|---|
| Main use | Browser sign-in through an external provider | Protect APIs receiving bearer tokens |
| Application role | OAuth client / OIDC relying party | Resource server |
| Typical request credential | Local authenticated session after login | `Authorization: Bearer ...` |
| Principal | `OAuth2User` / `OidcUser` | `Jwt` for JWT authentication |
| Repo | Module 72 | Optional JWT config in module 71 |

`OAuth2AuthenticationToken` identifies the local OAuth-login authentication and registration.
Authorized-client infrastructure separately manages provider access/refresh tokens.
A JWT resource server normally produces `JwtAuthenticationToken` instead.
Provider login does not automatically give the application a domain-specific `ROLE_ADMIN`.
Map provider identities to your application's authorization model explicitly.

### Method-level security and roles
```java
@EnableMethodSecurity(jsr250Enabled = true, securedEnabled = true)
```

Pre/post annotations are enabled by default with `@EnableMethodSecurity`.
`jsr250Enabled` activates Jakarta `@RolesAllowed`, `@PermitAll`, and `@DenyAll` support.
`securedEnabled` activates Spring's `@Secured` support.

| Annotation | Evaluation | Best fit |
|---|---|---|
| `@PreAuthorize` | Before invocation, SpEL | Role plus ownership/arguments |
| `@PostAuthorize` | After return, SpEL | Returned-object access checks |
| `@RolesAllowed` | Before invocation, role list | Jakarta role-based rules |
| `@Secured` | Before invocation, authority list | Simple Spring authority checks |

`hasRole('USER')` normally checks `ROLE_USER`.
`hasAuthority('ROLE_USER')` checks that exact authority string.
`roles("USER")` on the user builder adds the `ROLE_` prefix.
`authorities("USER")` does not add it.
Within a role list, listed roles are alternatives; across multiple different annotations, checks accumulate.
Method security uses proxies, so calling a method on `this` can bypass interception.
Place critical policy on a service boundary reached through a Spring-managed proxy.
Post-authorization does not undo writes or remote calls already performed by the method.

### Spring Security 6 migration essentials
- Use `SecurityFilterChain` beans; `WebSecurityConfigurerAdapter` is removed.
- Use lambda DSL configuration; older chained no-argument configurer styles are deprecated in 6.3.
- Use `requestMatchers`; `antMatchers` and `mvcMatchers` DSL methods were removed.
- Distinguish a chain's `securityMatcher` from authorization rules inside that chain.
- Use `@EnableMethodSecurity` rather than legacy `@EnableGlobalMethodSecurity` for new code.
- Use `jakarta.*` APIs, including `jakarta.annotation.security.RolesAllowed`.
- Verify request matcher behavior, CSRF integration, and authority mapping during migration.
- A migration that compiles but accidentally permits endpoints is not successful.

## Common pitfalls / gotchas
| Pitfall | Why it fails | Better approach |
|---|---|---|
| Calling `/authenticate` immediately | JWT controller is commented out | Activate the alternative deliberately in a practice copy |
| Enabling both configurations unchanged | Duplicate beans/competing all-request chains | Select one coherent configuration |
| Removing Basic config to enable JWT | Removes the currently declared method-security enablement | Enable method security explicitly in the selected setup |
| Expecting JWT roles to match Basic roles | Default scopes become `SCOPE_...` | Design and test an authority converter |
| Expecting `/csrf-token` JSON now | CSRF filter is disabled | Enable CSRF for the experiment |
| Disabling CSRF because sessions are disabled | Basic/cookie credentials can be ambient | Analyze browser credential transport |
| Treating CORS as authentication | Nonbrowser callers ignore CORS | Authenticate and authorize every request |
| Relying on username path variables | Attacker can choose them | Compare principal and stored ownership |
| Assuming POST persists a todo | It only logs | Implement persistence in a separate exercise |
| Equating post-authorization with rollback | Side effects may already have happened | Prefer preauthorization for writes |
| Expecting generated Boot user | Custom user service overrides defaults | Use the seeded JDBC accounts |
| Disabling all frame protection for H2 | Weakens unrelated pages | Development-only scoped console settings |
| Assuming `iss` is validated automatically here | Direct public-key decoder lacks issuer-specific validator | Configure issuer and audience validation |
| Restarting the token issuer | New RSA keys invalidate old tokens | Stable protected keys and planned rotation |
| Logging OAuth principal wholesale | Sensitive claims can leak | Redact and audit intentionally |

An all-request authentication rule is a useful baseline, not complete business authorization.
Test both API-level failures and method-level failures.
Remember that a 403 can mean CSRF denial, not just a missing role.

## Interview questions & answers
### Basic
#### 1. What is authentication?
Establishing an identity using evidence such as a password, session, or verified token.
In module 71, Basic authentication verifies the JDBC user's BCrypt password.
Authorization is a separate decision after identity establishment.

#### 2. What is authorization?
Deciding whether an identity may perform an operation on a resource.
The todo GET combines roles with username and returned-object checks.
Authentication alone does not establish resource ownership.

#### 3. What does HTTP Basic send?
A Base64-encoded `username:password` value in the `Authorization` header.
Base64 offers no secrecy; HTTPS is essential.
Credentials can be replayed if intercepted.

#### 4. What is the default Spring Security login experience?
Without custom replacing beans, Boot commonly supplies a development user and generated login page.
A custom `SecurityFilterChain` determines which login mechanisms are actually enabled.
This module explicitly enables Basic, not form login.

#### 5. What is a security filter chain?
An ordered collection of security filters selected for a matching request.
`FilterChainProxy` selects the first matching chain.
Controllers execute only after the selected filter flow permits them.

#### 6. Where is the current user stored during a servlet request?
In the `SecurityContext` accessible through `SecurityContextHolder`.
The default holder strategy is thread-local.
Spring clears it after request processing to avoid thread-reuse leaks.

#### 7. What is CSRF?
An attack that tricks a browser into performing an authenticated action using automatically attached credentials.
A synchronizer token requires proof of legitimate request intent beyond the cookie.
It is not the same vulnerability as XSS.

#### 8. Why use BCrypt instead of plaintext passwords?
BCrypt produces salted, adaptive one-way encodings that slow offline guessing.
Plaintext exposes passwords directly, and fast unsalted hashes are cheap to attack.
Password verification uses `matches`, not equality of fresh encodings.

#### 9. What is a JWT?
A compact claims format commonly represented as header, payload, and signature segments.
A signed JWT protects integrity but normally does not hide its contents.
Its claims are trustworthy only after appropriate validation.

#### 10. What is an OAuth access token?
A credential that lets a client access protected resources within granted permissions.
It may be JWT-shaped or opaque.
OAuth does not require all access tokens to be JWTs.

#### 11. What does OIDC add to OAuth?
Authentication/identity semantics, notably an ID token and the `openid` scope.
OAuth focuses on delegated resource access.
An ID token is intended for its relying-party client, not arbitrary API use.

#### 12. What is the difference between a role and an authority?
An authority is an exact permission string carried by authentication.
A role is conventionally an authority with the `ROLE_` prefix.
`hasRole('USER')` checks `ROLE_USER` by default.

### Intermediate
#### 13. What do the three servlet security layers do?

`DelegatingFilterProxy` bridges container registration to Spring.
`FilterChainProxy` selects the matching security chain.
`SecurityFilterChain` supplies that chain's matcher and filters.

#### 14. What does `UserDetailsService` do?

It loads a user by username and exposes password/account/authority data.
It does not itself authenticate by comparing a presented password.
`DaoAuthenticationProvider` normally coordinates lookup and verification.

#### 15. Why might a stateless API still need CSRF protection?

Stateless describes context/session policy, not browser credential attachment.
Browsers can automatically send cookies or cached Basic credentials.
Only an explicit nonambient token transport makes the usual bearer-only rationale applicable.

#### 16. Why does permitting OPTIONS not solve CORS?

Preflight authorization and CORS response headers are different concerns.
The browser also requires matching origin/method/header permissions.
Configure a real CORS policy and security integration.

#### 17. What is `DelegatingPasswordEncoder` useful for?

It routes verification using stored `{id}` prefixes and supports algorithm migrations.
New passwords can use a preferred encoder while old encodings remain verifiable.
The repo's direct BCrypt encoder is a simpler single-algorithm setup.

#### 18. Why use asymmetric JWT signing?

Services can validate with a public key without gaining token-signing power.
Only the issuer retains the private key.
This separates trust responsibilities better than a secret shared by every verifier.

#### 19. What is a JWK source?

A key-selection abstraction used here by `NimbusJwtEncoder`.
A JWK set can contain signing/verification metadata and keys.
Creating it as a bean does not publish a network JWKS endpoint.

#### 20. Explain authorization code with PKCE.

The client sends a challenge during authorization and proves the matching verifier during token exchange.
An intercepted code alone is insufficient to obtain tokens.
PKCE complements state and redirect-URI validation.

#### 21. When should a service use client credentials?

When a backend acts as itself rather than an interactive end user.
The client authenticates to the token endpoint and receives a scoped access token.
It is not a browser login or user-password flow.

#### 22. How are method-security annotations enabled?

`@EnableMethodSecurity` enables pre/post support by default.
Set `jsr250Enabled = true` for Jakarta role annotations and `securedEnabled = true` for `@Secured`.
The active Basic configuration enables both flags.

#### 23. What changed from `WebSecurityConfigurerAdapter`?

Spring Security 6 uses component-based configuration such as `SecurityFilterChain` beans.
Use lambda DSL and `requestMatchers` instead of removed older matcher methods.
Also migrate Java EE API imports to Jakarta where appropriate.

#### 24. How do OAuth login and resource-server support differ?

OAuth login makes the app a client/relying party and typically establishes a local browser session.
Resource-server support validates bearer access tokens on API requests.
The runtime principal and authentication token types differ.

### Advanced and scenario-based

#### 25. Scenario: Basic works but the same user's JWT gets 403 on the todo GET. Why?

The demo puts `ROLE_USER` into `scope`.
Default JWT conversion creates `SCOPE_ROLE_USER`, so `hasRole('USER')` no longer matches.
Configure a deliberate authority mapping and verify every method annotation.

#### 26. Scenario: JWT activation removes all method checks. What was missed?

`@EnableMethodSecurity` currently lives on the Basic configuration.
If that class is disabled, activate method security elsewhere in the selected JWT setup.
Endpoint annotations alone do not register the necessary infrastructure.

#### 27. Scenario: tokens stop working after every restart. Why?

The RSA key pair is regenerated at startup.
Previously issued signatures no longer validate against the new public key.
Persist protected signing keys and rotate intentionally with overlapping verification keys.

#### 28. Scenario: an admin gets another user's todo through `/users/admin/todos`. Why?

The seeded admin has `USER`; the path matches the admin's authentication name.
The method always returns the first todo, and the post-check only requires `in28minutes` ownership.
The policy is a teaching example, not a correct dynamic ownership check.

#### 29. Can `@PostAuthorize` safely protect a money transfer?

Not by itself: the transfer can execute before the returned object is checked.
Perform preauthorization and enforce domain invariants transactionally.
A later access denial is not a general rollback guarantee for external effects.

#### 30. Scenario: a custom login sets `SecurityContextHolder`, but the next request is anonymous. Why?

With explicit-save behavior, setting the thread-local holder does not automatically persist the context.
Save through the correct `SecurityContextRepository` when a stateful design requires it.
Also verify session policy, session authentication strategy, and cookie transport.

#### 31. What validation is missing from this simple public-key decoder?

Application-specific expected issuer and audience validation are not configured here.
Default timestamp validation and a valid signature do not establish the token's intended recipient.
Compose validators and restrict the trusted algorithm/key source.

#### 32. How would you implement JWT logout?

Deleting the client's token stops that client using it, but does not revoke stolen copies.
Use short lifetimes and, if necessary, denylist/version checks or introspected/reference tokens.
Revoke/rotate refresh tokens separately when that mechanism exists.

#### 33. Scenario: cross-origin curl succeeds but browser JavaScript fails. Why?

Curl does not enforce browser CORS rules.
Check the preflight and the allow-origin/method/header/credential response policy.
Do not weaken authentication just to make preflight appear successful.

#### 34. Scenario: Google returns `redirect_uri_mismatch`. What do you check?

Compare the registered URI with the actual scheme, host, port, and callback path.
Account for reverse-proxy external URLs and trusted forwarded-header configuration.
The local default callback is `/login/oauth2/code/google`.

#### 35. Why is arbitrary thread-local context propagation dangerous?

A worker can retain the wrong user's identity if context capture/cleanup is incorrect.
Use bounded security-aware task wrappers and clear state after execution.
Scheduled tasks should usually use an explicit service identity rather than a random request identity.

#### 36. How would you harden these examples for production?

Use TLS, external secret/key management, persistent users, restrictive headers, and transport-appropriate CSRF/CORS.
Enforce resource ownership, align JWT authorities, validate issuer/audience, and avoid sensitive logging.
Add authentication rate limiting, audit, dependency maintenance, and negative authorization tests.

## Hands-on exercises

Perform edits only in a separate practice copy or your own branch when studying.
The documentation-generation task itself does not change source configuration.

### Exercise 1 — Map the active Basic authorization behavior

Start module 71 and compare unauthenticated versus authenticated responses:

```bash
curl -i http://localhost:8080/hello-world
curl -i -u in28minutes:dummy http://localhost:8080/hello-world
curl -i -u in28minutes:dummy http://localhost:8080/todos
curl -i -u in28minutes:dummy http://localhost:8080/users/in28minutes/todos
curl -i -u in28minutes:dummy http://localhost:8080/users/admin/todos
curl -i -u admin:dummy http://localhost:8080/users/admin/todos
```

Expect 401 without valid credentials, 200 for permitted reads, and 403 for the regular user's admin-path GET.
Explain why the admin-path success still returns the first `in28minutes` todo.
Inspect response status as well as body; redirects and error bodies can otherwise mislead.

### Exercise 2 — Prove that the POST is not persistence or ownership enforcement

```bash
curl -i -u in28minutes:dummy \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","description":"Study method authorization"}' \
  http://localhost:8080/users/admin/todos
curl -i -u in28minutes:dummy http://localhost:8080/todos
```

Observe the application log and unchanged GET list.
Then add a preauthorization rule and a real ownership-aware persistence design in your practice copy.
Add tests for mismatched path/body/principal usernames rather than trusting client input.

### Exercise 3 — Enable and verify synchronizer-token protection

In a practice copy of the Basic setup, retain CSRF defaults instead of disabling them.
Retrieve a token while keeping the session cookie:

```bash
curl -s -u in28minutes:dummy -c csrf-cookies.txt \
  http://localhost:8080/csrf-token
```

Read the JSON `token` and `headerName` fields; the default header is `X-CSRF-TOKEN`.
Copy the returned token into a shell variable:

```bash
CSRF_TOKEN='copy-returned-token'
curl -i -u in28minutes:dummy -b csrf-cookies.txt \
  -H "X-CSRF-TOKEN: $CSRF_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"username":"in28minutes","description":"CSRF protected POST"}' \
  http://localhost:8080/users/in28minutes/todos
```

Repeat without the token and expect rejection for the unsafe request.
The session can exist for the CSRF repository even when the security context is stateless.
Remove the exercise cookie file afterward; never publish live cookies or tokens.

### Exercise 4 — Activate JWT as an alternative and expose the authority mismatch

In a practice copy, disable the Basic configuration and activate the JWT configuration/controller.
Ensure method security remains explicitly enabled with the same flags.
Do not enable both configurations unchanged.
Obtain a token using Basic authentication on the protected issuance endpoint:

```bash
curl -s -u in28minutes:dummy -X POST http://localhost:8080/authenticate
TOKEN='copy-token-field-from-response'
curl -i -H "Authorization: Bearer $TOKEN" http://localhost:8080/hello-world
curl -i -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/users/in28minutes/todos
```

Explain the successful hello response and possible role-related 403.
Add a converter that intentionally maps the demo authority strings to `ROLE_...` authorities.
Then test expiry, wrong signature, issuer, and audience validation.
Restart and observe the consequences of ephemeral signing keys.
Do not send real tokens to online JWT-decoding sites; decode locally for inspection only.

### Exercise 5 — Trace Google login and its local session

Provide client credentials through environment variables and register the local callback.
Start module 72 alone on port 8080.
Open `http://localhost:8080/` in a browser and follow the provider login.
Inspect `/oauth2/authorization/google`, the callback, and the local session cookie.
Observe the runtime authentication/principal types without publishing personal claims.
Explain why sending that local session cookie to an unrelated resource server is not OAuth API access.

## Quick revision cheat sheet

| Remember | One-line answer |
|---|---|
| Authentication | Identity evidence |
| Authorization | Permission to perform an operation |
| Filter registration bridge | `DelegatingFilterProxy` |
| Security chain selector | `FilterChainProxy`, first matching chain |
| Current request identity | `SecurityContextHolder` -> context -> authentication |
| Basic | Repeated Base64 credentials; always use TLS |
| CSRF | Prevent unwanted actions with ambient browser credentials |
| CORS | Browser permission to read cross-origin responses |
| Stateless | No session-based security context, not “CSRF impossible” |
| Password storage | Salted adaptive encoder, never plaintext |
| JWT | Readable claims plus integrity proof, not automatic encryption |
| RSA | Private key signs; public key verifies |
| JWK/JWKS | Key representation / key set |
| JWT validation | Algorithm, signature, timestamps, issuer, audience |
| OAuth code | Short-lived value exchanged for tokens |
| PKCE | Challenge/verifier binds token exchange to initiating client |
| OIDC | Identity layer and ID token on OAuth |
| `oauth2Login` | Browser client/relying party |
| `oauth2ResourceServer` | Bearer-token API validation |
| `hasRole('USER')` | Normally expects `ROLE_USER` |
| Default JWT scopes | Become `SCOPE_...` authorities |
| Method security | Enable it, then cross a Spring proxy boundary |
| Security 6 DSL | Beans, lambdas, `requestMatchers`, Jakarta APIs |

Repository-specific final checks:

- Basic configuration is active; JWT configuration and controller are commented out.
- JDBC schema comes from `JdbcDaoImpl`, not a local SQL script.
- Both seeded users have password `dummy`; admin has both `ADMIN` and `USER`.
- CSRF and frame-options are disabled in module 71's demonstration configuration.
- The todo GET is fixed-data teaching code; the POST logs without persistence.
- JWT scopes contain role-shaped strings, creating a default `SCOPE_ROLE_...` mismatch.
- RSA keys are generated at startup; issuer/audience policy needs additional validators.
- Module 72 uses Google OAuth login and normally maintains a local authenticated session.
- The root 3.1.x update document is historical, not a required Boot 3.3.4 patch.
