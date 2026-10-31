# 04 — Spring MVC and the Todo Web Application
## What this module teaches
**Exact module path:** `11-web-application/`.

This module builds a server-rendered todo application using Spring MVC and JSP. It starts with strings and HTML, introduces model-backed forms and validation, adds Spring Security form login, and
replaces an in-memory list with JPA/MySQL.

The current `pom.xml` declares **Spring Boot 3.3.4** and **Java 21**. The relevant modern APIs are Spring Framework 6, Spring Security 6, Hibernate 6, and Jakarta Servlet/Persistence/Validation APIs.
Do not copy old `javax.*` imports or `WebSecurityConfigurerAdapter` examples into this build.

### How to run the actual current module
Prerequisites: JDK 21, Maven, and a MySQL server reachable at `localhost:3306`. The current datasource points to database `todos` and username `todos-user`. The current H2 dependency and H2 datasource
URL are commented out. Starting the module without MySQL therefore is not an H2-only demo.

From the repository root:

```bash
mvn -f 11-web-application/pom.xml spring-boot:run
```

There is no module Maven wrapper, so use an installed Maven rather than `./mvnw`. For credential overrides, configure the standard `SPRING_DATASOURCE_PASSWORD` environment variable. Do not place real
credentials in shell history, URLs, logs, or committed properties.

For a disposable local MySQL instance, an illustrative Docker command is:

```bash
docker run --detach --name todo-interview-mysql \
  --publish 127.0.0.1:3306:3306 \
  --env MYSQL_DATABASE=todos \
  --env MYSQL_USER=todos-user \
  --env MYSQL_PASSWORD \
  --env MYSQL_ROOT_PASSWORD \
  mysql:8.4
```

Set `MYSQL_PASSWORD` and `MYSQL_ROOT_PASSWORD` in your shell first using your secure local workflow. Set `SPRING_DATASOURCE_PASSWORD` to the same application password. The Docker process must finish
initializing MySQL before Spring connects. If port 3306 is occupied, use the existing server or choose another host port and override the JDBC URL. The repository's historical command uses
`mysql:8-oracle`; the command above is an illustrative alternative. No Docker command was executed while writing this guide.

Open `http://localhost:8080/`. Spring Security should redirect an unauthenticated browser to its generated login page. The current code contains local demo users `in28minutes` and `ranga`. Consult
`SpringSecurityConfiguration.java` for their demo-only passwords. These hard-coded accounts are course examples, not a production authentication design.

A second module may already be using port 8080. Run this one on another port if needed:

```bash
mvn -f 11-web-application/pom.xml spring-boot:run \
  -Dspring-boot.run.arguments="--server.port=8081"
```

With the current MySQL datasource, Boot does not normally run `data.sql` automatically. An initially empty todo list is therefore consistent with the checked-in configuration. For a disposable
database only, explicitly opt in to SQL initialization if desired:

```bash
mvn -f 11-web-application/pom.xml spring-boot:run \
  -Dspring-boot.run.arguments="--spring.sql.init.mode=always"
```

The script uses fixed IDs, so repeated startup against persistent seeded data can cause duplicate-key failures. `spring.jpa.defer-datasource-initialization=true` orders scripts after JPA schema
creation; it does not enable external-database scripts on its own.

### Important source-location correction
The current JSPs are **not** under `src/main/webapp` in this checkout. They are under:

```text
11-web-application/src/main/resources/META-INF/resources/WEB-INF/jsp/
```

`src/main/webapp/WEB-INF/jsp` is a common servlet/WAR layout, not this module's actual layout. The controller also contains comments pointing to the actual resource location. JSP runtime support is
provided by Jasper and Jakarta JSTL dependencies in the POM.

Use Maven/IDE development launch for this course example. Spring Boot documents limitations for JSPs in executable JARs; do not assume `java -jar` packaging works merely because the development server
renders JSPs. For production JSP packaging, prefer a deliberately configured, tested WAR layout/runtime.

## Project walkthrough
### Source map
All paths below are relative to `11-web-application/` unless explicitly stated otherwise.

```text
src/main/java/com/in28minutes/springboot/myfirstwebapp/
  MyfirstwebappApplication.java
  hello/SayHelloController.java
  login/WelcomeController.java
  todo/Todo.java
  todo/TodoController.java       [@Controller commented out]
  todo/TodoControllerJpa.java    [active MVC controller]
  todo/TodoService.java          [static-list implementation]
  todo/TodoRepository.java       [active JPA abstraction]
  security/SpringSecurityConfiguration.java
src/main/resources/
  application.properties
  data.sql
  META-INF/resources/WEB-INF/jsp/
    sayHello.jsp
    welcome.jsp
    listTodos.jsp
    todo.jsp
    common/header.jspf
    common/navigation.jspf
    common/footer.jspf
```

### 1. `MyfirstwebappApplication.java`
Path: `src/main/java/com/in28minutes/springboot/myfirstwebapp/MyfirstwebappApplication.java`.

Actual excerpt:

```java
@SpringBootApplication
public class MyfirstwebappApplication {

	public static void main(String[] args) {
		SpringApplication.run(MyfirstwebappApplication.class, args);
	}

}
```

The application package sits above `hello`, `login`, `todo`, and `security`. Component scanning therefore finds controllers, services, and configuration classes there. Auto-configuration assembles the
servlet server, MVC, datasource, JPA, validation, and security support. The application class is the composition root, not the place for request-specific todo logic.

### 2. `SayHelloController.java`: three response styles
Path: `src/main/java/com/in28minutes/springboot/myfirstwebapp/hello/SayHelloController.java`.

Actual plain-body endpoint:

```java
@RequestMapping("say-hello")
@ResponseBody
public String sayHello() {
	return "Hello! What are you learning today?";
}
```

`@Controller` marks this as an MVC controller. `@ResponseBody` on this method means the returned string becomes HTTP response content. The view resolver does not search for a JSP with that long
greeting as its name.

The second method, `sayHelloHtml`, builds an HTML string using a `StringBuffer`. It also has `@ResponseBody`, so MVC writes that string rather than resolving a JSP. That is a teaching bridge, not a
maintainable template strategy. For a guaranteed HTML media type, declare `produces="text/html"` in a real endpoint. Returning a string containing HTML tags does not by itself declare that media type.

Actual JSP endpoint:

```java
@RequestMapping("say-hello-jsp")
public String sayHelloJsp() {
	return "sayHello";
}
```

There is no `@ResponseBody` here. `"sayHello"` is a logical view name resolved to `/WEB-INF/jsp/sayHello.jsp`. The mappings have no HTTP-method restriction, unlike later GET/POST-specific examples.
Security still protects these hello URLs because its rule covers all requests.

### 3. `WelcomeController.java`: user identity and model
Path: `src/main/java/com/in28minutes/springboot/myfirstwebapp/login/WelcomeController.java`.

Actual excerpt:

```java
@Controller
@SessionAttributes("name")
public class WelcomeController {

	@RequestMapping(value="/",method = RequestMethod.GET)
	public String gotoWelcomePage(ModelMap model) {
		model.put("name", getLoggedinUsername());
		return "welcome";
	}
```

Actual identity lookup:

```java
private String getLoggedinUsername() {
	Authentication authentication = 
			SecurityContextHolder.getContext().getAuthentication();
	return authentication.getName();
}
```

The authenticated principal supplies the username, not a submitted form parameter. The model attribute `name` is rendered by `welcome.jsp` as `${name}`. `@SessionAttributes("name")` promotes that
named model attribute into the HTTP session. It does not make the controller itself session-scoped. It does not create the Spring Security authentication session; security manages that separately.

Earlier step snapshots contain a hand-written login controller and credential handling. That is not the active login mechanism in the current Java sources. Spring Security now supplies its generated
login page and authentication filter.

### 4. `Todo.java`: both entity and form-backing object
Path: `src/main/java/com/in28minutes/springboot/myfirstwebapp/todo/Todo.java`.

Actual fields:

```java
@Id
@GeneratedValue
private int id;

private String username;

@Size(min=10, message="Enter atleast 10 characters")
private String description;
private LocalDate targetDate;
private boolean done;
```

The class has `@Entity`, a no-argument constructor, an all-fields constructor, and setters/getters. JPA maps it to database storage; MVC binds request values to its properties. `@GeneratedValue` does
not explicitly select IDENTITY; its default strategy is AUTO. Hibernate 6's chosen allocation mechanism depends on the database/provider. The `int` ID begins at zero for a new form object.

`@Size(min=10)` validates description length when validation is invoked. It permits `null`; it is not equivalent to `@NotBlank`. Ten spaces can satisfy the length constraint. The current target date
has no `@Future`/`@NotNull` annotation. HTML `required` helps the browser but is not a server-side persistence invariant.

Binding directly into entities is convenient for the course. A dedicated form DTO is safer when fields such as ID, owner, or completion status need restricted binding. A hidden form field is still
client-controlled data.

### 5. `TodoService.java`: the earlier static-list implementation
Path: `src/main/java/com/in28minutes/springboot/myfirstwebapp/todo/TodoService.java`.

Actual storage excerpt:

```java
@Service
public class TodoService {
	
	private static List<Todo> todos = new ArrayList<>();
	
	private static int todosCount = 0;
```

The static initializer creates three initial `in28minutes` todos. Actual filtering:

```java
public List<Todo> findByUsername(String username){
	Predicate<? super Todo> predicate = 
			todo -> todo.getUsername().equalsIgnoreCase(username);
	return todos.stream().filter(predicate).toList();
}
```

The stream returns a filtered list rather than exposing the original list directly. The `Todo` objects inside that result are still the same mutable objects. `Stream.toList()` returns an unmodifiable
list, but not deeply immutable entities.

`addTodo` increments a static counter and adds an object. `deleteById` removes matches with `removeIf`. `findById` ends in `findFirst().get()` and throws if the ID is absent. `updateTodo` deletes the
old entry and appends the supplied object.

These are not thread-safe transaction operations. Concurrent mutations of `ArrayList` and increments of `todosCount` can race. State is lost on process restart and not shared consistently across
application instances. This class is a learning implementation, not durable storage.

Its `@Valid` method parameter does not alone enable service method validation. That requires appropriate method-validation configuration and an intercepted validated bean. The active request
validation occurs in the MVC handler instead.

### 6. `TodoController.java`: why it is not active
Path: `src/main/java/com/in28minutes/springboot/myfirstwebapp/todo/TodoController.java`.

Actual class annotation excerpt:

```java
//@Controller
@SessionAttributes("name")
public class TodoController {
```

The class still illustrates the static-service version, but it is not registered as an MVC controller. Activating it alongside `TodoControllerJpa` would duplicate route mappings. Spring can then fail
startup with ambiguous mappings. Switch implementations deliberately rather than annotating both.

Actual static-service add excerpt:

```java
String username = getLoggedInUsername(model);
todoService.addTodo(username, todo.getDescription(), 
		LocalDate.now().plusYears(1), false);
return "redirect:list-todos";
```

This earlier implementation ignores the submitted target date during creation. It replaces it with one year from today and forces `done=false`. The JPA controller instead saves the validated bound
Todo, preserving its bound date. That is a real behavioral difference, not merely a repository-type swap.

### 7. `TodoRepository.java`: active persistence interface
Path: `src/main/java/com/in28minutes/springboot/myfirstwebapp/todo/TodoRepository.java`.

Actual excerpt:

```java
public interface TodoRepository extends JpaRepository<Todo, Integer>{
	public List<Todo> findByUsername(String username);
}
```

Spring Data provides inherited CRUD methods and derives the username query. The ID type is `Integer`, matching the entity's primitive `int` identifier. The finder has no explicit sort or pagination.
Result ordering is therefore not a business guarantee.

The static service compares usernames case-insensitively in Java. The JPA query's comparison follows the database's collation/operator behavior. Do not assume their case sensitivity is identical on
every database.

### 8. `TodoControllerJpa.java`: active request handlers
Path: `src/main/java/com/in28minutes/springboot/myfirstwebapp/todo/TodoControllerJpa.java`.

The class uses `@Controller`, `@SessionAttributes("name")`, and constructor injection. Actual constructor:

```java
public TodoControllerJpa(TodoRepository todoRepository) {
	super();
	this.todoRepository = todoRepository;
}
```

A single constructor does not need `@Autowired` for Spring injection. Constructor injection makes dependencies explicit and facilitates focused tests.

#### List todos
Actual excerpt:

```java
@RequestMapping("list-todos")
public String listAllTodos(ModelMap model) {
	String username = getLoggedInUsername(model);
			
	List<Todo> todos = todoRepository.findByUsername(username);
	model.addAttribute("todos", todos);
	
	return "listTodos";
}
```

The current user determines the query filter. The model's `todos` attribute becomes the JSP loop input. This route is not restricted to GET in the source. A read-only page would normally use
`@GetMapping` explicitly.

#### Display the creation form
Actual excerpt:

```java
@RequestMapping(value="add-todo", method = RequestMethod.GET)
public String showNewTodoPage(ModelMap model) {
	String username = getLoggedInUsername(model);
	Todo todo = new Todo(0, username, "", LocalDate.now().plusYears(1), false);
	model.put("todo", todo);
	return "todo";
}
```

The model must contain the form-backing object before the JSP renders. The default description is empty, but validation is performed on submission rather than this GET. The initial date makes the form
useful without requiring immediate date selection.

#### Handle creation
Actual excerpt:

```java
@RequestMapping(value="add-todo", method = RequestMethod.POST)
public String addNewTodo(ModelMap model, @Valid Todo todo, BindingResult result) {
	
	if(result.hasErrors()) {
		return "todo";
	}
	
	String username = getLoggedInUsername(model);
	todo.setUsername(username);
	todoRepository.save(todo);
```

After this excerpt, the method returns `"redirect:list-todos"`. `@Valid` triggers validation of the bound object. `BindingResult` immediately follows the parameter it describes. On failure, the method
redisplays the same view and preserves bound values/errors. On success, it overwrites username from the security context and saves.

Overwriting username prevents simple submitted-owner spoofing on create. It does not make every other field safe: a submitted nonzero ID can affect save/merge behavior. A robust creation DTO should
exclude ID and owner, and construct a fresh entity server-side.

#### Handle deletion
Actual excerpt:

```java
@RequestMapping("delete-todo")
public String deleteTodo(@RequestParam int id) {
	//Delete todo
	todoRepository.deleteById(id);
	return "redirect:list-todos";
	
}
```

The JSP uses an ordinary link, so the operation currently happens through a GET request. The mapping has no method restriction. Deleting state through GET violates safe-method expectations and invites
accidental activation. Use a POST form or appropriate API DELETE endpoint with authorization and CSRF protection.

The method does not restrict deletion to the current username. Authentication is not object-level authorization. Checking ownership in a transactional service is a necessary production improvement.

#### Display and process updates
Actual load excerpt:

```java
Todo todo = todoRepository.findById(id).get();
model.addAttribute("todo", todo);
return "todo";
```

Absent IDs lead to `Optional.get()` failure rather than a deliberate 404 response. There is no owner filter in this lookup. The POST update validates, sets username to the current principal, and calls
`save`. Blindly merging that bound object can overwrite fields or transfer ownership of an existing row. A safer service loads by ID **and owner**, then copies only editable fields.

### 9. JSP views and shared fragments
Base path: `src/main/resources/META-INF/resources/WEB-INF/jsp/`.

Actual `common/header.jspf` tag declarations:

```jsp
<%@ taglib prefix="c" uri="jakarta.tags.core" %>
<%@ taglib prefix="form" uri="http://www.springframework.org/tags/form" %>
```

The JSTL core URI is the Jakarta-compatible URI used by this source. Spring's form tag URI remains `http://www.springframework.org/tags/form`. That URI is an identifier for the tag library, not a
browser HTTP request.

Actual `listTodos.jsp` excerpt:

```jsp
<c:forEach items="${todos}" var="todo">
	<tr>
		<td>${todo.description}</td>
		<td>${todo.targetDate}</td>
		<td>${todo.done}</td>
```

`c:forEach` iterates through the model collection. JSP EL reads bean properties through their accessor conventions. `${todo.done}` uses `isDone()` for the boolean property. Plain JSP EL is not
automatic HTML escaping; use `<c:out>` or another safe rendering strategy for untrusted text.

Actual `todo.jsp` form declaration and description field:

```jsp
<form:form method="post" modelAttribute="todo">

	<fieldset class="mb-3">
		<form:label path="description">Description</form:label>
		<form:input type="text" path="description" required="required"/>
		<form:errors path="description" cssClass="text-warning"/>
	</fieldset>
```

Actual hidden fields:

```jsp
<form:input type="hidden" path="id"/>

<form:input type="hidden" path="done"/>
```

`path` names a property of the modelAttribute object. The form has no explicit action, so it posts to the current add/update URL. The same JSP therefore serves both creation and editing. The
`targetDate` field is a text input decorated by a datepicker. The widget improves input convenience, not server-side trust.

The views include `common/header.jspf`, `navigation.jspf`, and `footer.jspf`. The include directive combines JSP source at translation time. It is different from a request-time `<jsp:include>`.
`welcome.jsp` displays the name and a link to manage todos. `sayHello.jsp` is the simplest standalone JSP demonstration.

### 10. `SpringSecurityConfiguration.java`
Path: `src/main/java/com/in28minutes/springboot/myfirstwebapp/security/SpringSecurityConfiguration.java`.

The class declares beans for an in-memory user manager, BCrypt password encoder, and filter chain. It constructs two demo users and grants each `USER` and `ADMIN` roles.

Actual user-building excerpt:

```java
UserDetails userDetails = User.builder()
							.passwordEncoder(passwordEncoder)
							.username(username)
							.password(password)
							.roles("USER","ADMIN")
							.build();
```

The local function named `passwordEncoder` encodes the supplied password during construction. `roles("USER")` conventionally creates authority `ROLE_USER`. The code does not store these users in the
Todo table. `InMemoryUserDetailsManager` is authentication storage, not todo persistence.

Actual password bean:

```java
@Bean
public PasswordEncoder passwordEncoder() {
	return new BCryptPasswordEncoder();
}
```

BCrypt is an adaptive salted hash, not reversible encryption. Login checks use `matches(rawPassword, encodedPassword)` rather than encoding twice and comparing strings. Different salts can produce
different hashes for the same raw password.

Actual filter-chain body:

```java
http.authorizeHttpRequests(
		auth -> auth.anyRequest().authenticated());
http.formLogin(withDefaults());

http.csrf(csrf -> csrf.disable());
```

Actual frame configuration:

```java
http.headers(headers -> headers.frameOptions(HeadersConfigurer.FrameOptionsConfig::disable));
```

The source disables CSRF and frame-options globally for course/demo convenience. It does not restrict those relaxations to H2 paths. This remains true even though the active database is now MySQL. Do
not promote those global exceptions to production defaults.

Form login and HTTP Basic are different authentication mechanisms. This configuration enables the generated browser login form, not `httpBasic` explicitly. Unauthenticated browser requests normally
redirect to login. Successful login stores authentication in the security context/session for later requests.

### 11. `application.properties`, `data.sql`, and the POM
Path: `src/main/resources/application.properties`.

Actual relevant excerpts:

```properties
spring.mvc.view.prefix=/WEB-INF/jsp/
spring.mvc.view.suffix=.jsp
logging.level.org.springframework=info
logging.level.com.in28minutes.springboot.myfirstwebapp=info

spring.mvc.format.date=yyyy-MM-dd
```

```properties
spring.jpa.defer-datasource-initialization=true

spring.datasource.url=jdbc:mysql://localhost:3306/todos
spring.datasource.username=todos-user
```

```properties
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQLDialect

spring.jpa.hibernate.ddl-auto=update
```

A datasource password is also configured in the actual source; it is not reproduced here. Hibernate can usually infer the dialect from JDBC metadata; the explicit dialect is optional.
`ddl-auto=update` is convenient for this demo but not a production migration plan. The date formatter converts strings such as `2027-05-01` to `LocalDate` for MVC binding. It is not a global timezone
configuration or a universal JSON serialization setting.

Path: `src/main/resources/data.sql`.

Actual first insert:

```sql
insert into todo (ID, USERNAME, DESCRIPTION, TARGET_DATE, DONE)
values(10001,'in28minutes', 'Get AWS Certified', CURRENT_DATE(), false);
```

Three more inserts seed Azure, GCP, and DevOps todos with IDs 10002–10004. Their username is `in28minutes`, so `ranga` would not see them in the username-filtered list. Fixed seed IDs and generated ID
allocation are separate concerns. Verify the generator cannot later collide with manually seeded identifiers.

The actual POM supplies these additional pieces:

| Dependency | Role |
|---|---|
| `spring-boot-starter-web` | MVC and embedded servlet server |
| `spring-boot-starter-data-jpa` | Hibernate-backed repository integration |
| `mysql-connector-j` | MySQL JDBC driver |
| `spring-boot-starter-security` | Authentication/authorization filters |
| `spring-boot-starter-validation` | Bean Validation provider/integration |
| `tomcat-embed-jasper`, provided | JSP compilation/runtime support |
| Jakarta JSTL API + GlassFish implementation | JSTL tags |
| Bootstrap 5.1.3 WebJar | CSS/layout styling |
| jQuery 3.6.0 WebJar | Datepicker dependency |
| bootstrap-datepicker 1.9.0 WebJar | Date input widget |
| DevTools | Development restart conveniences |

### 12. How the historical step notes fit
The `Step*.md` files are complete snapshots, not executable application sources. `99-step-by-step-changes.md` records the teaching progression. The snapshots contain earlier Spring Boot versions and
older security configuration syntax. Use their narrative as context; use current `src/` and `pom.xml` as runtime truth.

| Step group | Learning progression |
|---|---|
| 03–05 | Controller, response body, HTML, JSP view resolution |
| 06–13 | Earlier hand-written login, request parameters, model, logging |
| 15–19 | Todo list, session attributes, JSTL, Bootstrap |
| 22–27 | Validation, command/form beans, delete/update, target date |
| 28 | Navigation and JSP fragments |
| 29–33 | Spring Security, encoded passwords, principal-based identity |
| 34–38 | H2, Todo entity, JPA repository, database-backed features |
| 39–41 | Optional Docker/MySQL transition |

The step heading "redirect to a JSP" should not obscure HTTP semantics: returning a normal view name typically renders through an internal forward; returning `redirect:...` causes a client-visible
redirect and a new request.

## Core concepts explained
### 1. MVC responsibilities and complete request flow
MVC means **Model–View–Controller**. The controller coordinates a request, the model supplies rendering data, and the view generates the response presentation. Repositories/services are not themselves
the MVC view or the model map.

```text
Browser: GET /list-todos
          |
          v
Servlet container (embedded Tomcat)
          |
          v
Spring Security filter chain
  authentication / authorization / CSRF / headers
          |
          v
DispatcherServlet                 [front controller]
          |
          v
HandlerMapping                    [which handler matches?]
          |
          v
HandlerAdapter                    [bind/invoke the handler]
          |
          v
TodoControllerJpa.listAllTodos
          |                 |
          |                 +-> TodoRepository -> JPA -> Hibernate -> MySQL
          v
Model: todos + view name "listTodos"
          |
          v
ViewResolver
  /WEB-INF/jsp/ + listTodos + .jsp
          |
          v
JSP View -> Jasper/JSTL -> rendered HTML
          |
          v
HTTP response -> browser
```

Security filters run before the MVC handler. HandlerMapping selects; HandlerAdapter invokes and manages arguments/return values. ViewResolver chooses a View; it does not itself query the database or
render every response type. A `@ResponseBody` path uses message conversion instead of normal JSP view resolution.

Servlet filters and MVC interceptors are not interchangeable. Filters wrap servlet requests; interceptors operate around mapped MVC handlers. Cross-cutting business transactions belong in service
boundaries, not JSP rendering code.

### 2. `@Controller`, `@RestController`, and `@ResponseBody`
| Annotation | Default return handling | Typical use |
|---|---|---|
| `@Controller` | Resolve view names when returning String | HTML pages/templates |
| `@ResponseBody` | Write return value to response | One body-producing controller method |
| `@RestController` | Controller + response body semantics | JSON/REST APIs |

Returning `"todo"` from this MVC controller selects `todo.jsp`. Returning `"todo"` from a RestController normally sends the text `todo`. A RestController does not automatically serialize every string
into a JSON object. Actual media type depends on mapping, negotiation, and available message converters.

Illustrative REST endpoint, not present in this module:

```java
@RestController
@RequestMapping("/api/todos")
class TodoApiController {
    @GetMapping("/{id}")
    TodoResponse read(@PathVariable("id") int id) {
        // Return a deliberately chosen response DTO.
        throw new UnsupportedOperationException("Illustrative only");
    }
}
```

A controller method can also return `ResponseEntity` for explicit status, headers, and body. Use an API DTO rather than exposing every persistence field automatically.

### 3. Mapping paths and request data
`@RequestMapping` can constrain path, method, headers, parameters, and media types. `@GetMapping` and `@PostMapping` are composed shortcuts for method-specific mappings.

Illustrative equivalent declarations:

```java
@RequestMapping(value = "/add-todo", method = RequestMethod.GET)
```

```java
@GetMapping("/add-todo")
```

| Input | URL/example | Annotation |
|---|---|---|
| Query/form parameter | `/update-todo?id=10001` | `@RequestParam("id")` |
| Path segment | `/todos/10001` | `@PathVariable("id")` |
| Form object | Fields description, targetDate, etc. | `@ModelAttribute` or implicit complex binding |
| JSON request body | `{"description":"..."}` | `@RequestBody` |

The actual update/delete routes use query parameters, not path variables. A required primitive parameter fails binding when missing or nonnumeric. Use explicit parameter names in annotations when
portability across build parameter metadata matters. The Boot build config normally retains Java parameter-name metadata, which supports the current shorthand.

GET should retrieve information without changing state. POST commonly submits form changes. An unrestricted mapping is not automatically a GET-only endpoint. Missing required parameters are request
errors, not database failures.

### 4. Model, ModelMap, request, and session scope
| Mechanism | Purpose/lifetime |
|---|---|
| `Model` | Interface for attributes passed to a view |
| `ModelMap` | Map-oriented model container |
| Request attributes | Available for the current request/render |
| `@SessionAttributes` | Promote selected controller model attributes into session |
| `@SessionAttribute` | Read an existing session attribute |
| `HttpSession` | General servlet session access |
| Flash attributes | Temporary attributes intended for a redirect's next request |

`model.addAttribute("todos", todos)` provides view data. It does not serialize the list to the browser as JSON on its own. The JSP renders the list using the data in request scope.

```text
Request 1: model.put("name", username)
                    |
         @SessionAttributes("name")
                    v
HTTP session: name available between requests
                    |
Request 2: controller/view can use promoted attribute
```

Keep session state small and avoid storing large managed entity graphs there. Session attributes can become stale and can be awkward in multiple browser tabs. For a conversation-style form,
`SessionStatus.setComplete()` clears attributes managed by that controller's `@SessionAttributes` workflow; it is not general session invalidation or logout.

Authentication is obtained from the security principal, not trusted from the display-name model attribute. Directly opening `/list-todos` need not first visit `/` because the JPA controller reads the
principal itself.

### 5. Form backing object and binding lifecycle
A form-backing object represents editable form state. `modelAttribute="todo"` points to the object stored under model key `todo`. For a complex non-simple handler parameter, MVC applies
model-attribute binding even without an explicit annotation. An explicit `@ModelAttribute("todo")` can make intent clearer in new code.

```text
GET /add-todo
  -> new Todo(0, user, "", defaultDate, false)
  -> model["todo"]
  -> JSP form fields

POST /add-todo
  -> request parameters
  -> WebDataBinder + conversion
  -> populated Todo
  -> Bean Validation
  -> BindingResult
      errors? -> redisplay form
      valid?  -> service/repository -> redirect
```

For `targetDate=2027-05-01`, the MVC formatter converts a string to `LocalDate`. For an invalid date string, conversion itself can create a binding error. Conversion errors and constraint violations
can both appear in BindingResult.

The hidden ID field preserves the edited row identifier between GET and POST. It does not prove that the user is authorized to edit that ID. A client can change any hidden field with developer tools
or a custom request.

Illustrative safer binding shape:

```java
public class TodoForm {
    @NotBlank
    @Size(min = 10, max = 200)
    private String description;

    @NotNull
    private LocalDate targetDate;

    // Accessors required for ordinary bean binding.
}
```

Use an authenticated service to construct a new entity or load an owned existing one. Do not copy user-supplied owner or ID into new entities blindly. `@InitBinder` allowed-fields configuration is
another binding defense, not a substitute for ownership checks.

### 6. Bean Validation and BindingResult
| Annotation/API | Role |
|---|---|
| `@Size` | Length/size range; null accepted |
| `@NotNull` | Reject null |
| `@NotBlank` | Reject null/blank character sequences |
| `@Future` | Date/time must be in the future |
| `@Valid` | Trigger cascaded Bean Validation in supported contexts |
| `BindingResult` | Binding and validation errors for the preceding object |
| `form:errors` | Render relevant form errors |

Correct parameter ordering:

```java
@PostMapping("/add-todo")
public String add(@Valid @ModelAttribute("todo") TodoForm form,
                  BindingResult result, Model model) {
    if (result.hasErrors()) {
        return "todo";
    }
    return "redirect:/list-todos";
}
```

This is illustrative; the actual handler uses the entity directly. BindingResult belongs immediately after the object being bound/validated. It is not an arbitrary final parameter for every argument
error in a method. Spring Framework 6.1 also supports MVC method validation with its own exception flow; that is distinct from this simple model-attribute validation example.

Return the form view on errors, not a success redirect. Otherwise a new request can lose field errors and rejected values. A database NOT NULL constraint can protect persistence but cannot supply
friendly per-field form messages alone. Browser validation is only a convenience because requests can bypass the browser entirely.

### 7. JSP, JSTL, and view resolution
The resolver combines a logical view name with configured prefix and suffix:

```text
"todo"
  -> prefix /WEB-INF/jsp/
  -> suffix .jsp
  -> /WEB-INF/jsp/todo.jsp
```

JSP is compiled to a servlet by Jasper. JSTL adds standard control-flow/output tags such as `c:forEach` and `c:out`. Spring form tags integrate bean binding and validation errors with generated HTML
inputs. Use templates for presentation rather than putting SQL or service logic into JSP scriptlets.

`WEB-INF` views are intended for internal rendering, not direct browser access. A user navigates to `/add-todo`, not `/WEB-INF/jsp/todo.jsp`. Controller mappings expose the workflow; the template path
is an implementation detail.

Illustrative escaping improvement:

```jsp
<td><c:out value="${todo.description}"/></td>
```

Escaping helps prevent untrusted descriptions from being interpreted as executable markup. Validation of minimum length alone provides no output-encoding protection. Do not confuse CSRF protection
with XSS prevention; they solve different problems.

### 8. Redirect and Post/Redirect/Get
```text
Browser POST /add-todo
  -> bind / validate / save
  <- 3xx Location: list-todos
Browser GET /list-todos
  -> query / render
  <- 200 HTML
```

This is **Post/Redirect/Get**, often abbreviated PRG. Refreshing the final page repeats GET rather than the original creation POST. It reduces accidental resubmission but does not provide full
duplicate-request idempotency.

`return "todo"` renders a view inside the current request. `return "redirect:list-todos"` sends the browser a redirect. The source uses a relative target; `redirect:/list-todos` is a clearer
context-relative form in many applications. Consider path resolution when endpoints are nested under class-level mappings.

Ordinary request model state does not persist automatically across a redirect. Use `RedirectAttributes.addFlashAttribute` for a one-time success message. Flash attributes are for display state, not
reliable durable business events.

### 9. Logging with SLF4J and levels
The current properties set Spring and application package logging to INFO. The current controllers do not declare an active SLF4J logger field. Earlier course steps discuss logging; the following is
an illustrative extension:

```java
private static final Logger log = LoggerFactory.getLogger(TodoControllerJpa.class);

log.debug("Listing todos for {}", username);
log.info("Created todo with id {}", savedTodo.getId());
```

Use `org.slf4j.Logger` and `org.slf4j.LoggerFactory` imports. SLF4J is a facade; Boot's default logging implementation is normally Logback. Parameterized messages avoid eager string concatenation and
are easier to structure. Avoid logging passwords, session IDs, tokens, or entire form/entity objects containing private information.

| Level | Typical purpose |
|---|---|
| TRACE | Extremely detailed investigation |
| DEBUG | Developer diagnostic detail |
| INFO | Useful ordinary application events |
| WARN | Unexpected/recoverable condition |
| ERROR | Failure requiring attention |
| OFF | Disable logging |

At INFO, DEBUG and TRACE are suppressed; WARN and ERROR remain enabled. A package-specific level can override a broader/root level. Enable narrow package logging before enabling global TRACE.
`spring.jpa.show-sql` is not the same as application SLF4J logging. Production diagnostics should balance useful context with privacy and volume.

### 10. WebJars, Bootstrap, and browser assets
WebJars package client libraries inside dependency JARs. Spring Boot serves their static assets from `/webjars/...` paths. Actual `common/footer.jspf` excerpts:

```html
<script src="webjars/bootstrap/5.1.3/js/bootstrap.min.js"></script>
<script src="webjars/jquery/3.6.0/jquery.min.js"></script>
<script src="webjars/bootstrap-datepicker/1.9.0/js/bootstrap-datepicker.min.js"></script>
```

Bootstrap styles the container, table, navigation bar, and buttons. Bootstrap 5 itself does not require jQuery; the selected datepicker does. Load jQuery before the datepicker plugin and run
initialization after those scripts. The datepicker's `yyyy-mm-dd` format agrees visually with the server's `yyyy-MM-dd` formatter syntax. They are different formatting libraries; do not assume every
token is interchangeable.

These relative asset URLs work for the current root-level routes. Nested paths or a non-root context can require context-aware URL construction. With `anyRequest().authenticated()`, asset requests
also require authentication unless separately permitted. The generated login page has its own framework handling, not these protected JSP fragments.

### 11. Spring Security form login and H2 console exceptions
```text
Unauthenticated request
  -> security chain
  -> generated login form
POST login credentials
  -> authentication filter
  -> authentication provider
  -> UserDetailsService + PasswordEncoder
  -> authenticated SecurityContext
  -> session-backed subsequent requests
```

`InMemoryUserDetailsManager` implements UserDetailsService for the demo accounts. PasswordEncoder verifies hashes. SecurityFilterChain configures which requests require authentication and other
protections. Authorization is more than checking whether the user logged in. Granting USER and ADMIN roles does not automatically enforce any object ownership rule.

CSRF protects state-changing requests using browser-carried credentials against unwanted cross-site actions. Cookie/session authentication makes that protection important for this application. The
current global CSRF disable removes that protection. Re-enable it for normal forms; Spring form tags can cooperate with Spring Security's request-data processing and include the CSRF token when
configured normally.

The H2 console uses frames and its own request handling, so development exceptions may be necessary. An illustrative development-only exception configuration is:

```java
http.csrf(csrf -> csrf.ignoringRequestMatchers(PathRequest.toH2Console()));
http.headers(headers -> headers.frameOptions(frame -> frame.sameOrigin()));
```

This snippet requires `org.springframework.boot.autoconfigure.security.servlet.PathRequest`. It leaves CSRF protection enabled for non-console routes. `sameOrigin` still relaxes framing policy and
must be a deliberate decision. For tighter separation, use a development-only console-specific filter chain with its own matcher and header policy, followed by the application chain. Do not expose an
H2 console in production or permit all application URLs just to make it work.

The current MySQL module cannot show H2 merely by changing the security headers. An H2 learning variant also needs the H2 runtime dependency, console-enabled property, and an H2 datasource
configuration without the MySQL-specific dialect override.

### 12. Getting the logged-in user
The actual controllers use SecurityContextHolder. In ordinary servlet processing, its context is usually associated with the current thread. Security filters populate and clean it around request
processing. It does not automatically follow an arbitrary new thread or asynchronous task.

Illustrative MVC alternatives:

```java
@GetMapping("/list-todos")
public String list(Authentication authentication, Model model) {
    String username = authentication.getName();
    // Query current user's todos and render the model.
    return "listTodos";
}
```

You can also accept `Principal` or use `@AuthenticationPrincipal` for a custom principal. Do not trust a request parameter named username as the identity of the current user. Protected controller
routes can normally expect authenticated requests, but tests/background work need explicit security context setup or passed identity. Anonymous authentication can also produce a non-null
Authentication; non-null alone is not a universal authorization check.

### 13. Static list to JPA/MySQL: what changes architecturally?
```text
Earlier version
Controller -> TodoService -> static ArrayList

Current course version
TodoControllerJpa -> TodoRepository -> EntityManager -> Hibernate -> MySQL

Production-oriented extension
Controller -> authenticated transactional TodoService -> repository -> database
```

Persistence brings durable storage, database constraints, transactions, and multi-instance sharing. It does not automatically bring service-level authorization or safe form binding. The current
controller directly uses repository transactions. A sequence of multiple repository calls is not automatically one business transaction.

Illustrative secure update algorithm:

1. Obtain the principal's username.
2. Begin a service transaction.
3. Load by ID and username, returning a deliberate not-found/forbidden result if absent.
4. Copy description/date from a restricted form DTO.
5. Keep the loaded owner and ID unchanged.
6. Commit with dirty checking, optionally guarded by `@Version`.

For MySQL, verify network reachability, database existence, credentials, grants, and JDBC driver. A connection-refused error is not fixed by adding `@Entity`. A missing-table error is not necessarily
a controller mapping problem. A dialect mismatch is not fixed by changing JSP tags.

`ddl-auto=update` can create/adjust local tables, but use explicit migrations for production. Container restart persistence requires deliberate database storage/volume configuration. The illustrative
Docker command is for disposable learning, not a durable deployment plan.

## Common pitfalls / gotchas
| Pitfall | Actual reason | Better approach |
|---|---|---|
| Looking only under src/main/webapp | This checkout uses META-INF/resources | Follow the actual source paths |
| Assuming executable JAR JSP support | Boot JSP packaging limitations | Test a deliberately supported runtime/layout |
| Both Todo controllers activated | Duplicate URL handlers | Register only one implementation |
| H2 console expected on current app | MySQL active; H2 dependency commented | Create a deliberate H2 learning configuration |
| Empty MySQL list despite data.sql | External SQL initialization defaults | Explicit disposable opt-in or managed seed migration |
| Duplicate seed errors after restart | Fixed IDs in persistent database | Idempotent fixtures/migrations |
| BindingResult in the wrong position | It belongs to the preceding form object | Place it immediately after that object |
| Redirecting on validation errors | Request errors/rejected values are lost | Redisplay the form view |
| Trusting HTML required | Clients can bypass browser validation | Server Bean Validation |
| Size assumed to reject null/blank | Size only constrains non-null length | Add NotBlank as appropriate |
| Trusting hidden IDs | Hidden != trusted | Restricted DTO plus owned-entity load |
| Logged in assumed to mean authorized | No per-row ownership check | Service checks principal and row ownership |
| Delete via a link | GET mutates state | POST/DELETE with CSRF and authorization |
| CSRF disabled globally | Demo convenience persists in current code | Re-enable and scope console exceptions |
| Frames globally allowed | Removes clickjacking defense | Deliberate same-origin/dev-only exception |
| JSP EL assumed to escape | Raw EL can output HTML | c:out/context-appropriate escaping |
| Optional.get for unknown ID | No deliberate not-found handling | orElseThrow with mapped response |
| Static ArrayList treated as safe storage | Races, restarts, instance-local state | Repository/database/service transactions |
| Expecting static/JPA behaviors identical | Date and case comparison differ | Test contracts during the storage switch |
| Logging all submitted data | Privacy/credential leakage | Minimal, structured diagnostics |
| AUTO assumed IDENTITY | Provider/database chooses strategy | Inspect Hibernate mapping and SQL |
| Header changes assumed to enable H2 | Console also needs dependency/configuration | Configure all pieces consistently |

## Interview questions & answers
### Basic
#### Q1. What is Spring MVC?
**Answer:** Spring's servlet-based web framework that coordinates request handlers, binding, validation, model data, and response rendering. This module uses it to generate JSP HTML pages.

#### Q2. What is DispatcherServlet?
**Answer:** MVC's front controller. It delegates handler selection and invocation, then processes the return value through view rendering or response-body handling.

#### Q3. What is the difference between HandlerMapping and ViewResolver?
**Answer:** HandlerMapping selects the controller handler for a request. ViewResolver translates a logical view name into a View used for presentation.

#### Q4. What does `return "listTodos"` mean in this controller?
**Answer:** Resolve and render a view named listTodos using the configured JSP prefix/suffix. It is not automatically a JSON string or a client-visible redirect.

#### Q5. Compare Controller and RestController.
**Answer:** Controller commonly returns view names; RestController adds response-body semantics. The hello controller demonstrates method-level ResponseBody alongside a JSP-producing method.

#### Q6. What does ResponseBody do?
**Answer:** It makes MVC write the method's return value through HTTP message conversion rather than use normal view-name resolution for that result.

#### Q7. What are GetMapping and PostMapping?
**Answer:** Composed RequestMapping annotations restricted to GET and POST respectively. The actual add/update methods use equivalent RequestMapping method attributes.

#### Q8. How do RequestParam and PathVariable differ?
**Answer:** RequestParam binds query/form parameters, such as `?id=10001`; PathVariable binds a URI-template segment, such as `/todos/10001`.

#### Q9. What is ModelMap used for?
**Answer:** Passing named attributes to a view. Here `todos` drives the list table and `todo` backs the form inputs.

#### Q10. Where are the actual JSPs stored?
**Answer:** Under `src/main/resources/META-INF/resources/WEB-INF/jsp/` in this module, not under the conventional src/main/webapp layout.

#### Q11. What is JSTL?
**Answer:** Standard JSP tag libraries for tasks such as iteration and escaped output. This application uses `c:forEach` and declares the Jakarta core tag URI in header.jspf.

#### Q12. What is a WebJar?
**Answer:** A dependency JAR packaging browser assets. This build uses WebJars for Bootstrap, jQuery, and the datepicker, served through `/webjars/...` resource paths.

### Intermediate
#### Q13. What does modelAttribute="todo" mean?
**Answer:** The Spring form reads/writes properties of the object stored as `todo` in the model. The GET handler must provide that backing object before the form renders.

#### Q14. What does Valid do in the POST handler?
**Answer:** It invokes Bean Validation for the bound Todo. The adjacent BindingResult holds validation/binding errors so the controller can redisplay the form.

#### Q15. Why must BindingResult follow Todo immediately?
**Answer:** Its association is positional: it describes the preceding bound argument. Moving it elsewhere can make MVC raise an exception instead of handing those errors to the method.

#### Q16. Does Size(min=10) reject all empty descriptions?
**Answer:** It rejects non-null strings shorter than ten characters, but accepts null and sufficiently long whitespace. Add NotBlank when the requirement is meaningful nonblank text.

#### Q17. How is targetDate converted from the form?
**Answer:** MVC binds the string using configured date formatting, here yyyy-MM-dd. Bad date syntax creates a binding error even without an explicit date constraint.

#### Q18. What does SessionAttributes("name") do?
**Answer:** It promotes that named controller model attribute into the HTTP session. It neither makes all model state session-scoped nor creates authentication by itself.

#### Q19. What is Post/Redirect/Get?
**Answer:** Successful POST processing returns a redirect, and the browser performs a GET. Refreshing the final page usually repeats the read instead of resubmitting the original form.

#### Q20. Why return the same JSP on validation failure?
**Answer:** The current request still contains the rejected values and BindingResult. A success-style redirect creates a new request and can lose that feedback.

#### Q21. How does this app obtain its username?
**Answer:** It reads Authentication from SecurityContextHolder and calls getName. Principal or Authentication handler parameters would be cleaner injectable alternatives in new code.

#### Q22. What stores the demo users?
**Answer:** InMemoryUserDetailsManager, configured with UserDetails objects. TodoRepository stores todos; it does not store or authenticate these demo accounts.

#### Q23. Why use a PasswordEncoder?
**Answer:** Passwords should be verified against adaptive hashes, not stored/recovered as plaintext. BCrypt includes salt; use matches rather than comparing independently encoded strings.

#### Q24. What logging appears at INFO level?
**Answer:** INFO, WARN, and ERROR are normally emitted; DEBUG and TRACE are suppressed. The current properties apply INFO to both Spring and the application package.

### Advanced and scenario-based
#### Q25. The app starts with ambiguous mappings. What is the first thing to check here?
**Answer:** Whether the commented Controller annotation on TodoController was activated while TodoControllerJpa remained active. Both classes define the same todo routes.

#### Q26. A user changes id in the delete URL and deletes another user's todo. What is missing?
**Answer:** Object-level authorization. The current delete loads/deletes by ID only. Require an ID-and-owner lookup/check in a transactional service before changing the row.

#### Q27. Is setting username from the principal enough to secure an update?
**Answer:** No. Merging a client-selected ID can still overwrite or reassign an existing row. Load the existing owned entity first and copy only allowed editable properties.

#### Q28. Why are hidden fields not a security boundary?
**Answer:** They are ordinary request data and can be altered by a browser tool or custom client. Ownership and allowed fields must be enforced on the server.

#### Q29. Why is deleting through GET dangerous?
**Answer:** GET is expected to be safe; crawlers, prefetchers, links, or unwanted requests can activate it. Use a state-changing method with CSRF protection and explicit row authorization.

#### Q30. Does form login mean HTTP Basic is enabled?
**Answer:** No. Form login processes browser credentials and session authentication. HTTP Basic uses an Authorization header and must be configured deliberately if desired.

#### Q31. Why does H2 console sometimes need special security rules?
**Answer:** Its frames and requests conflict with normal frame/CSRF protections. Use scoped development exceptions, not globally disabled protections for the entire app.

#### Q32. Can you restore H2 just by enabling spring.h2.console.enabled?
**Answer:** Not with the current POM/configuration alone. Add the H2 runtime dependency, use an H2 datasource, remove the MySQL-specific override, and configure development console access.

#### Q33. Why is data.sql not seeding the current MySQL app by default?
**Answer:** Boot normally initializes scripts automatically only for embedded databases. Defer-datasource-initialization changes ordering, not opt-in; external seeding needs explicit configuration.

#### Q34. Why can enabling SQL initialization fail on the second startup?
**Answer:** The script inserts fixed IDs into persistent tables, so existing rows cause duplicate keys. Use disposable fixtures or deliberately idempotent/versioned initialization instead.

#### Q35. The form datepicker works, but POST rejects the date. What do you inspect?
**Answer:** Compare browser/plugin format with MVC's server formatter, and inspect BindingResult. A visual widget cannot override server conversion rules or constraints.

#### Q36. A submitted description contains markup and appears as HTML. Why?
**Answer:** The list JSP outputs raw EL and minimum-length validation does not escape HTML. Use context-appropriate output escaping, for example c:out for ordinary text content.

#### Q37. Will switching ArrayList to MySQL fix concurrent edit conflicts automatically?
**Answer:** It adds real persistence/transactions, but lost updates still need a concurrency policy. A Version field and user-visible conflict handling are common optimistic-locking solutions.

#### Q38. Why might static and JPA username filtering behave differently?
**Answer:** Static filtering uses equalsIgnoreCase. Derived JPA queries follow database comparison/collation. Declare and test the intended username normalization/uniqueness policy instead of assuming
equivalence.

#### Q39. A JSP works through Maven but fails from an executable JAR. Is that unexpected?
**Answer:** JSP has documented Boot packaging limitations, especially executable JAR support. Validate a supported JSP packaging/runtime, often a deliberately configured WAR, rather than assuming dev
launch proves it.

#### Q40. How would you organize production-ready controller tests?
**Answer:** Test mapping, principal handling, validation errors, model attributes, and redirects with MVC tests. Also test security/CSRF and owned-row update/delete behavior, plus repository
integration against the intended database.

## Hands-on exercises
These are learner tasks for an isolated copy or disposable environment. They describe improvements; they were not applied to existing application files while writing this guide.

### Exercise 1: trace a rendered request
1. Start MySQL and the application with matching credentials.
2. Login and visit `/list-todos`.
3. Follow the active controller, model attribute, logical view name, and JSP.
4. Compare `/say-hello` with `/say-hello-jsp` in browser network tools.
5. Record body, status, media type, and whether JSP resolution occurs.

**Acceptance:** draw the request flow and explain the ResponseBody branch without guessing. Do not confuse an internal JSP forward with a browser redirect.

### Exercise 2: strengthen and test form validation
1. Introduce a restricted TodoForm or deliberately strengthen existing validation.
2. Add NotBlank and an appropriate date constraint.
3. Submit short text, whitespace, bad date syntax, and valid values.
4. Check BindingResult and field-level JSP error display.
5. Ensure invalid submissions do not create rows.

**Acceptance:** invalid requests redisplay the form with preserved values and useful errors. Server behavior remains correct when browser required validation is bypassed.

### Exercise 3: enforce ownership on update/delete
1. Add repository/service lookup by ID and principal username.
2. Load existing rows before applying update fields.
3. Replace delete links with CSRF-protected POST forms.
4. Test as both demo users against each other's IDs.
5. Define whether unauthorized row access yields 404 or 403.

**Acceptance:** one user cannot read an edit form, update, or delete another user's todo by changing ID. A creation request cannot overwrite an existing row through a submitted ID.

### Exercise 4: restore normal security defaults
1. Re-enable CSRF for ordinary application forms.
2. Verify generated/form-tag token handling and POST logout behavior.
3. If experimenting with H2, add a development-only console configuration.
4. Restrict console exceptions rather than disabling all headers/CSRF globally.
5. Keep production console exposure disabled.

**Acceptance:** authenticated POST without a valid CSRF token fails; normal valid form submissions succeed, and H2 exceptions do not weaken unrelated routes.

### Exercise 5: compare persistence and pagination
1. Run the current JPA version with deliberate seed initialization.
2. Add enough owned todos to need pagination.
3. Sort deterministically and show a Page or Slice in the model.
4. Restart the app and confirm intended database persistence.
5. Compare the static version's create-date behavior with the JPA version.

**Acceptance:** page contents/order are predictable and owner-filtered. Explain fixed seed ID risks, external SQL initialization, and at least one behavior changed by the storage switch.

## Quick revision cheat sheet
| Topic | Remember |
|---|---|
| Actual active controller | TodoControllerJpa |
| Earlier controller | TodoController has Controller commented out |
| Active database | MySQL todos at localhost:3306 |
| H2 in current POM | Dependency commented out |
| JSP location | META-INF/resources/WEB-INF/jsp |
| Boot/Java | 3.3.4 / 21 |
| Jakarta migration | persistence, validation, servlet namespaces |
| Front controller | DispatcherServlet |
| HandlerMapping | Find handler |
| HandlerAdapter | Bind and invoke |
| ViewResolver | Logical name -> View |
| Controller | Typically view-oriented |
| RestController | Controller + ResponseBody |
| RequestParam | Query/form parameter |
| PathVariable | URI-template segment |
| ModelMap | View attributes |
| SessionAttributes | Selected model attributes promoted to session |
| Form backing object | Model key todo |
| BindingResult | Immediately after validated object |
| Size | Length check; null permitted |
| Valid | Trigger validation in supported context |
| Date format | MVC yyyy-MM-dd |
| redirect: | Client redirect/new request |
| PRG | POST -> redirect -> GET |
| JSTL | Jakarta core tags and loops/output |
| form tags | Binding-aware HTML inputs/errors |
| JSP escaping | Raw EL is not automatic safe output |
| WebJars | Packaged browser assets |
| Logging | SLF4J facade; INFO suppresses DEBUG/TRACE |
| Demo users | InMemoryUserDetailsManager |
| Password hashing | BCrypt; matches for verification |
| Login mechanism | Form login, not implicit HTTP Basic |
| Principal lookup | SecurityContextHolder / Authentication / Principal |
| Current CSRF policy | Disabled globally; demo caveat |
| Current frame policy | Disabled globally; demo caveat |
| Row authorization | Not provided by authenticated() alone |
| Dangerous current delete | GET link with ID-only delete |
| Hidden fields | Untrusted client input |
| data.sql + MySQL | Needs explicit initialization opt-in |
| defer initialization | Ordering only |
| ddl-auto update | Local convenience, not production migration policy |
| Static storage | Non-durable, non-thread-safe learning implementation |
| JPA storage | Durable engine plus explicit service/authorization design |

**Thirty-second answer:** Spring Security filters protect the request before DispatcherServlet selects and invokes the MVC handler. The controller binds and validates a Todo, obtains the principal's
identity, queries or saves through JPA, and either renders a JSP using model data or redirects after success. The course demonstrates the framework mechanics; production readiness also needs ownership
checks, restricted form binding, CSRF, safe output, and deliberate database initialization.
