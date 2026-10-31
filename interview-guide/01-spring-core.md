# Spring Core: From Java Objects to a Managed Application
## What this module teaches
This guide follows the in28minutes **Master Spring and Spring Boot** course code. It explains Spring Framework 6 concepts using the actual gaming and business examples.
The Maven projects use **Spring Boot 3.3.4 dependency management and Java 21**. That does not make every launcher a Boot application: most explicitly create a Spring context.

**Repository root:** `/Users/rpathak/Documents/Learning/master-spring-and-spring-boot-main`.

**Exact module paths:**

- `01-spring/learn-spring-framework-01/`: plain Java, Java configuration, and bean selection.
- `01-spring/learn-spring-framework-02/`: scanning, injection, scopes, lifecycle, CDI annotations, and XML.
- `01-spring/01-Step*.md`, `02-Step*.md`, and `03-Step*.md`: historical lesson snapshots.
- `01-spring/step-by-step-code-changes/step-by-step-guide.md`: the combined progression.

Treat the current `src/main` files as authoritative. The snapshots sometimes contain older project names or intermediate implementations.
In particular, the earliest tightly coupled runner is a teaching stage, not the final runner.

### How to run the examples
Run commands from the repository root with a JDK 21 installation and Maven available. These modules contain several `main` methods, so choose the main class explicitly.
The following command runs the Java-configuration gaming example:
```bash
mvn -f 01-spring/learn-spring-framework-01/pom.xml spring-boot:run \
  -Dspring-boot.run.main-class=com.in28minutes.learnspringframework.App03GamingSpringBeans
```
The following command runs the annotation-based gaming example:
```bash
mvn -f 01-spring/learn-spring-framework-02/pom.xml spring-boot:run \
  -Dspring-boot.run.main-class=com.in28minutes.learnspringframework.game.GamingAppLauncherApplication
```
Alternatively, open a launcher in your IDE and run its `main` method. The launchers create a context, print results, close the context, and exit.
They do not start an HTTP server or require a database.

| Project | Main class suffix under `com.in28minutes.learnspringframework` | Demonstration |
|---|---|---|
| Framework 01 | `App01GamingBasicJava` | Manual object construction and constructor injection |
| Framework 01 | `helloworld.App02HelloWorldSpring` | Named beans, records, primary beans, qualifiers |
| Framework 01 | `App03GamingSpringBeans` | `@Bean` wiring of the gaming graph |
| Framework 01 | `test.ExploringVar` | Java local-variable type inference |
| Framework 01 | `test.SpringBeanVsJavaBean` | POJO and JavaBean terminology |
| Framework 02 | `game.GamingAppLauncherApplication` | Component scan and qualified injection |
| Framework 02 | `examples.a0.SimpleSpringContextLauncherApplication` | Bean-definition listing |
| Framework 02 | `examples.a1.DepInjectionLauncherApplication` | Constructor injection |
| Framework 02 | `examples.c1.RealWorldSpringContextLauncherApplication` | Service/repository discovery |
| Framework 02 | `examples.d1.LazyInitializationLauncherApplication` | Lazy creation |
| Framework 02 | `examples.e1.BeanScopesLauncherApplication` | Singleton and prototype identities |
| Framework 02 | `examples.f1.PrePostAnnotationsContextLauncherApplication` | Initialization and cleanup |
| Framework 02 | `examples.g1.CdiContextLauncherApplication` | Jakarta `@Named` and `@Inject` |
| Framework 02 | `examples.h1.XmlConfigurationContextLauncherApplication` | Explicit XML definitions |

For any Framework 02 example, replace the main-class value in the command above. For example, the lifecycle main class is:
`com.in28minutes.learnspringframework.examples.f1.PrePostAnnotationsContextLauncherApplication`.

### Reading conventions
Code labeled **repo excerpt** is copied from the current source, with surrounding code omitted.
Code labeled **illustration** is an explanatory alternative, not an existing implementation.
Paths are repository-relative; package-private helper classes often share their launcher's file.
A Spring bean means an object managed by a Spring container, not merely an annotated class.

## Project walkthrough
### 1. Build files: Spring libraries without Boot startup
Files:

- `01-spring/learn-spring-framework-01/pom.xml`
- `01-spring/learn-spring-framework-02/pom.xml`

Both inherit from `spring-boot-starter-parent` version `3.3.4`. Both declare `<java.version>21</java.version>` and `spring-boot-starter`.
Both include the test starter and Spring Boot Maven plugin. Framework 02 additionally declares `jakarta.inject:jakarta.inject-api:2.0.1`.

The starter supplies the Spring infrastructure used by the context launchers. The parent manages compatible versions; the Maven plugin can launch a selected main method.
Neither the parent nor the plugin secretly calls `SpringApplication.run` for these examples. The code chooses how the application context is created.

### 2. `App01GamingBasicJava`: construction before containers
File: `01-spring/learn-spring-framework-01/src/main/java/com/in28minutes/learnspringframework/App01GamingBasicJava.java`.

Repo excerpt:
```java
var game = new PacmanGame(); //1: Object Creation

var gameRunner = new GameRunner(game);
```
The method then invokes `gameRunner.run()`. The commented alternatives instantiate `MarioGame` or `SuperContraGame`. The active implementation is `PacmanGame`, not Mario.

This is already dependency injection: another object supplies the runner's dependency. Spring is not required to practice DI.
Here, application startup code owns object creation and wiring.

### 3. `GamingConsole`: an interface that separates policy from implementation
File: `01-spring/learn-spring-framework-01/src/main/java/com/in28minutes/learnspringframework/game/GamingConsole.java`.
The same interface also exists in Framework 02's `game` package.

Repo excerpt:
```java
public interface GamingConsole {
    void up();
    void down();
    void left();
    void right();
}
```
This is a contract for four operations, not a graphical console or hardware interface. `GameRunner` needs the contract, not the concrete game class.
All three games implement it, allowing the runner to remain unchanged.

### 4. `MarioGame`, `PacmanGame`, and `SuperContraGame`
Files in Framework 01:

- `01-spring/learn-spring-framework-01/src/main/java/com/in28minutes/learnspringframework/game/MarioGame.java`
- `01-spring/learn-spring-framework-01/src/main/java/com/in28minutes/learnspringframework/game/PacmanGame.java`
- `01-spring/learn-spring-framework-01/src/main/java/com/in28minutes/learnspringframework/game/SuperContraGame.java`

Repo excerpt from `MarioGame`:
```java
public void up() {
    System.out.println("Jump");
}
```
Repo excerpt from `SuperContraGame`:
```java
public void right() {
    System.out.println("Shoot a bullet");
}
```
| Operation | Mario | Pacman | Super Contra |
|---|---|---|---|
| `up()` | `Jump` | `up` | `up` |
| `down()` | `Go into a hole` | `down` | `Sit down` |
| `left()` | `Go back` | `left` | `Go back` |
| `right()` | `Accelerate` | `right` | `Shoot a bullet` |

In Framework 01 these are ordinary unannotated classes. They become Spring beans only when a configuration method registers an instance.
These output differences make implementation selection visible without external infrastructure.

### 5. `GameRunner`: the consumer of the abstraction
File: `01-spring/learn-spring-framework-01/src/main/java/com/in28minutes/learnspringframework/game/GameRunner.java`.

Repo excerpt:
```java
private GamingConsole game;

public GameRunner(GamingConsole game) {
    this.game = game;
}
```
Repo excerpt from its `run()` method:
```java
System.out.println("Running game: " + game);
game.up();
game.down();
game.left();
game.right();
```
There is no game construction inside the runner. The field is not `final` in the repo, but the design uses constructor injection.
An immutable-dependency version would make it `private final GamingConsole game`. That is a proposed improvement, not a claim about the existing source.

### 6. `GamingConfiguration`: explicit bean factories
File: `01-spring/learn-spring-framework-01/src/main/java/com/in28minutes/learnspringframework/GamingConfiguration.java`.

Repo excerpt:
```java
@Bean
public GamingConsole game() {
    var game = new PacmanGame();
    return game;
}

@Bean
public GameRunner gameRunner(GamingConsole game) {
    var gameRunner = new GameRunner(game);
    return gameRunner;
}
```
The class carries `@Configuration`. The default bean names are `game` and `gameRunner`, matching the method names.
Spring resolves the parameter to the registered `GamingConsole` bean. `new` is appropriate inside a factory method: the returned object becomes managed.

The graph is:
```text
GamingConfiguration
   |
   +-- @Bean game ---------> PacmanGame
   |
   +-- @Bean gameRunner ---> GameRunner
                                  |
                                  +---- GamingConsole reference to PacmanGame
```
### 7. `App03GamingSpringBeans`: creating and closing a context
File: `01-spring/learn-spring-framework-01/src/main/java/com/in28minutes/learnspringframework/App03GamingSpringBeans.java`.

Repo excerpt:
```java
try (var context =
        new AnnotationConfigApplicationContext
            (GamingConfiguration.class)) {

    context.getBean(GamingConsole.class).up();
    context.getBean(GameRunner.class).run();
}
```
The constructor registers configuration and refreshes the context. The first lookup returns the one `GamingConsole` bean, which prints `up`.
The runner then prints all four Pacman operations. Try-with-resources closes the configurable context and triggers eligible destruction callbacks.

### 8. `HelloWorldConfiguration`: values, records, names, and selection
File: `01-spring/learn-spring-framework-01/src/main/java/com/in28minutes/learnspringframework/helloworld/HelloWorldConfiguration.java`.

Repo excerpt:
```java
record Person (String name, int age, Address address) { };
record Address(String firstLine, String city){ };
```
These package-private Java records are ordinary Java types. Their instances become beans because factory methods return them.
Records generate component accessors, equality, hash code, and `toString`. They do not generate mutable JavaBean setters.

The configuration defines:

| Bean name | Type | Important behavior |
|---|---|---|
| `name` | `String` | Returns `Ranga` |
| `age` | `Integer` after boxing | Factory returns primitive `15` |
| `person` | `Person` | Ravi, 20, a directly constructed Utrecht address |
| `person2MethodCall` | `Person` | Calls configuration methods |
| `person3Parameters` | `Person` | Uses injected factory-method arguments |
| `person4Parameters` | `Person` | Primary `Person` bean |
| `person5Qualifier` | `Person` | Explicitly selects qualified Hyderabad address |
| `address2` | `Address` | Primary address, Baker Street, London |
| `address3` | `Address` | Motinagar, Hyderabad; custom qualifier |

Repo excerpt:
```java
@Bean(name = "address2")
@Primary
public Address address() {
    return new Address("Baker Street", "London");
}

@Bean(name = "address3")
@Qualifier("address3qualifier")
public Address address3() {
    return new Address("Motinagar", "Hyderabad");
}
```
Repo excerpt:
```java
@Bean
public Person person5Qualifier(String name, int age, @Qualifier("address3qualifier") Address address) {
    //name,age,address2
    return new Person(name, age, address); //name, age
}
```
The custom qualifier is `address3qualifier`; it is not the bean name `address3`. The old inline comment mentioning `address2` should not override the annotation's meaning.
The qualified person receives Hyderabad even though London is primary.

`person3Parameters` names its `Address` parameter `address3`. In this source, the primary `address2` still wins the ordinary single-value type resolution.
A matching parameter name is a fallback, not a way to override a primary candidate. Use a qualifier when implementation choice must be explicit.

`person2MethodCall()` calls `name()`, `age()`, and `address()`. Full `@Configuration` processing intercepts those inter-bean calls.
Calling `address()` obtains the managed `address2`, rather than creating an extra singleton.
The `new Address(...)` inside `person()` is different: it is not a separately registered bean.

### 9. `App02HelloWorldSpring`: lookup by name and by type
File: `01-spring/learn-spring-framework-01/src/main/java/com/in28minutes/learnspringframework/helloworld/App02HelloWorldSpring.java`.

Repo excerpt:
```java
System.out.println(context.getBean("name"));
System.out.println(context.getBean("age"));
System.out.println(context.getBean("person"));
```
The launcher also retrieves `Person.class`, `Address.class`, and `person5Qualifier`. The type lookups succeed because the respective primary beans resolve ambiguity.
A name lookup selects that bean directly and does not need a primary designation. The commented bean-definition stream is useful for inspecting registrations.

### 10. Java terminology helpers
Files:

- `01-spring/learn-spring-framework-01/src/main/java/com/in28minutes/learnspringframework/test/ExploringVar.java`
- `01-spring/learn-spring-framework-01/src/main/java/com/in28minutes/learnspringframework/test/SpringBeanVsJavaBean.java`

Repo excerpt from `ExploringVar`:
```java
var myList = new ArrayList<String>();
myList.add("someString");
```
`var` is local-variable type inference, not dynamic typing. The commented field declaration demonstrates that `var` cannot declare an instance field.
It cannot declare an ordinary method parameter or return type either.

Repo excerpt from the JavaBean helper:
```java
class JavaBean implements Serializable  { //EJB
    private static final long serialVersionUID = 1L;

    //1: public no-arg constructor
    public JavaBean() {

    }
```
The helper also defines private properties and getters/setters. `Pojo` is a simple class whose `toString()` prints its two fields.
The launcher constructs `new Pojo()` without any Spring context. The source's `//EJB` comment does not make this class an Enterprise JavaBean.
EJB is a separate Jakarta Enterprise Beans technology.

### 11. Framework 02 gaming: automatic discovery
Files under `01-spring/learn-spring-framework-02/src/main/java/com/in28minutes/learnspringframework/game/`:

- `GamingConsole.java`: same four-operation contract.
- `MarioGame.java`: `@Component` and `@Primary`.
- `PacmanGame.java`: `@Component`.
- `SuperContraGame.java`: `@Component` and a custom qualifier.
- `GameRunner.java`: `@Component` and qualified constructor injection.
- `GamingAppLauncherApplication.java`: configuration and explicit package scan.

Repo excerpt from `GameRunner`:
```java
public GameRunner(@Qualifier("SuperContraGameQualifier") GamingConsole game) {
    this.game = game;
}
```
Repo excerpt from `SuperContraGame`:
```java
@Component
@Qualifier("SuperContraGameQualifier")
public class SuperContraGame implements GamingConsole{
```
Repo excerpt from the launcher:
```java
@Configuration
@ComponentScan("com.in28minutes.learnspringframework.game")
public class GamingAppLauncherApplication {
```
The launcher performs both a direct `GamingConsole.class` lookup and a runner lookup. The direct lookup selects primary Mario and prints `Jump`.
The runner receives qualified Super Contra and prints `Shoot a bullet` for `right()`. There is no contradiction: different injection/lookup sites have different selection rules.

### 12. `a0`: inspect the container
File: `01-spring/learn-spring-framework-02/src/main/java/com/in28minutes/learnspringframework/examples/a0/SimpleSpringContextLauncherApplication.java`.

Repo excerpt:
```java
Arrays.stream(context.getBeanDefinitionNames())
    .forEach(System.out::println);
```
The launcher has `@Configuration` and parameterless `@ComponentScan`. The scan starts in `examples.a0`, not in every package in the project.
The output includes the configuration bean and infrastructure definitions.
A bean definition describes how to obtain a bean; it is not proof the instance already exists.
Lazy and prototype definitions may be listed before their objects are constructed.

### 13. `a1`: constructor, setter, and field injection
File: `01-spring/learn-spring-framework-02/src/main/java/com/in28minutes/learnspringframework/examples/a1/DepInjectionLauncherApplication.java`.

The file defines `YourBusinessClass`, `Dependency1`, `Dependency2`, and the launcher.
The first three are package-private `@Component` classes.

Repo excerpt:
```java
//@Autowired
public YourBusinessClass(Dependency1 dependency1, Dependency2 dependency2) {
    super();
    System.out.println("Constructor Injection - YourBusinessClass ");
    this.dependency1 = dependency1;
    this.dependency2 = dependency2;
}
```
The annotation is commented out because a single constructor is automatically selected.
The file's setter-injection methods are commented out; they are not active callbacks.
Its fields currently have no active `@Autowired` annotation either.
The launcher prints definitions and the injected business object's `toString()`.

Setter-injection illustration based on the commented teaching alternative:
```java
@Autowired
public void setDependency1(Dependency1 dependency1) {
    this.dependency1 = dependency1;
}
```
Field-injection illustration, **not active repo code**:
```java
@Autowired
private Dependency1 dependency1;
```
Constructor injection is normally preferred for mandatory dependencies.
Setter injection can represent optional or replaceable dependencies.
Field injection works on managed objects but hides requirements from their public construction API.

### 14. `c1`: a real-world-shaped service boundary
Files under `01-spring/learn-spring-framework-02/src/main/java/com/in28minutes/learnspringframework/examples/c1/`:

- `DataService.java`: interface with `int[] retrieveData()`.
- `MySqlDataService.java`: repository returning `{1, 2, 3, 4, 5}`.
- `MongoDbDataService.java`: primary repository returning `{11, 22, 33, 44, 55}`.
- `BusinessCalculationService.java`: service using constructor-injected `DataService`.
- `RealWorldSpringContextLauncherApplication.java`: package-local scanning and definition listing.

Repo excerpt from `BusinessCalculationService`:
```java
public int findMax() {
    return Arrays.stream(dataService.retrieveData()).max().orElse(0);
}
```
Repo excerpt from `MongoDbDataService`:
```java
@Repository
@Primary
public class MongoDbDataService implements DataService {
```
The business service receives Mongo by default, so `findMax()` would return `55`.
The current launcher only lists bean definitions; it does **not** call `findMax()`.
Do not promise a printed `55` from the unchanged launcher.

These repositories are hardcoded data providers, not actual MySQL or MongoDB integrations.
There is no connection URL, driver, database client, query, or transaction in this example.
The point is swappable implementations and a clean business dependency.
A fake `DataService` can test `findMax()` without Spring or a database.

### 15. `d1`: lazy construction is observable
File: `01-spring/learn-spring-framework-02/src/main/java/com/in28minutes/learnspringframework/examples/d1/LazyInitializationLauncherApplication.java`.

The file defines `ClassA`, lazy `ClassB`, and the launcher.
`ClassB` depends on `ClassA` and prints during construction.

Repo excerpt:
```java
@Component
@Lazy
class ClassB {
```
Repo excerpt from the launcher:
```java
System.out.println("Initialization of context is completed");
context.getBean(ClassB.class).doSomething();
```
Expected relevant output order:
```text
Initialization of context is completed
Some Initialization logic
Do Something
```
`ClassA` is a normal singleton and is eagerly constructed during context refresh.
`ClassB` waits until the explicit lookup.
If an eager singleton required `ClassB` directly, that dependency could force earlier creation.

### 16. `e1`: object identity and scope
File: `01-spring/learn-spring-framework-02/src/main/java/com/in28minutes/learnspringframework/examples/e1/BeanScopesLauncherApplication.java`.

The file defines `NormalClass`, `PrototypeClass`, and the launcher.

Repo excerpt:
```java
@Scope(value=ConfigurableBeanFactory.SCOPE_PROTOTYPE)
@Component
class PrototypeClass {

}
```
The launcher looks up `NormalClass` six times and `PrototypeClass` four times.
Singleton lookups return the same instance for that bean definition in that context.
Each prototype lookup asks the container to construct a new instance.
The default object strings make this visible, but identity assertions are stronger evidence.

Request and session scopes are **extensions to this lesson**, not present in `e1`.
They require a web-aware context and a relevant active request/session.
Do not expect `@RequestScope` to work in this plain annotation context.

### 17. `f1`: managed initialization and cleanup
File: `01-spring/learn-spring-framework-02/src/main/java/com/in28minutes/learnspringframework/examples/f1/PrePostAnnotationsContextLauncherApplication.java`.

The file defines `SomeClass`, `SomeDependency`, and the launcher.
It imports `jakarta.annotation.PostConstruct` and `jakarta.annotation.PreDestroy`.

Repo excerpt:
```java
@PostConstruct
public void initialize() {
    someDependency.getReady();
}

@PreDestroy
public void cleanup() {
    System.out.println("Cleanup");
}
```
Constructor injection makes `SomeDependency` available to `SomeClass`'s constructor.
Initialization runs after dependency population and before normal use of the bean.
The dependency method prints `Some logic using SomeDependency`.
Closing the context prints `Cleanup` for the managed singleton.

The constructor's message `All dependencies are ready!` describes this constructor-injected example.
It is not a universal guarantee that all field/setter injection is finished inside a constructor.

### 18. `g1`: Jakarta injection annotations inside Spring
File: `01-spring/learn-spring-framework-02/src/main/java/com/in28minutes/learnspringframework/examples/g1/CdiContextLauncherApplication.java`.

The file defines a `BusinessService`, a concrete local `DataService`, and the launcher.
This `DataService` is not the `c1.DataService` interface; packages distinguish them.

Repo excerpt:
```java
@Named
class BusinessService {
    private DataService dataService;

    //@Autowired
    @Inject
    public void setDataService(DataService dataService) {
        System.out.println("Setter Injection");
        this.dataService = dataService;
    }
```
Spring component scanning recognizes `jakarta.inject.Named`.
Spring processes `jakarta.inject.Inject` at injection points.
The launcher prints the resolved dependency from `getDataService()`.
This is Spring interoperability with standard injection annotations, not a full CDI container.
Adding `jakarta.inject-api` alone does not install Weld or implement all CDI semantics.

### 19. `h1`: explicit XML still works
Files:

- `01-spring/learn-spring-framework-02/src/main/java/com/in28minutes/learnspringframework/examples/h1/XmlConfigurationContextLauncherApplication.java`
- `01-spring/learn-spring-framework-02/src/main/resources/contextConfiguration.xml`
- `01-spring/learn-spring-framework-02/src/main/resources/contextConfiguration.out.xml`

The `.out.xml` file is empty and is not loaded by the launcher.
The real XML declares `name`, `age`, `game`, and `gameRunner`.

Repo excerpt from `contextConfiguration.xml`:
```xml
<bean id="game" class="com.in28minutes.learnspringframework.game.PacmanGame"/>

<bean id="gameRunner"
    class="com.in28minutes.learnspringframework.game.GameRunner">
    <constructor-arg ref="game" />
</bean>
```
Repo excerpt from the launcher:
```java
new ClassPathXmlApplicationContext("contextConfiguration.xml")
```
`name` is the string `Ranga`, and `age` is an `Integer` with value `35`.
The component-scan element is commented out.
Therefore the explicit XML constructor reference supplies Pacman to the runner.
The constructor's Java qualifier does not replace that explicit XML argument.
No scanned Mario or Super Contra bean is needed by this configuration.

### 20. Tests and historical lesson files
Each module contains `src/test/java/com/in28minutes/learnspringframework/LearnSpringFrameworkApplicationTests.java`.
These are course test scaffolding; the launchers are the primary demonstrations.
Do not confuse a test class with the configuration passed to an example's context constructor.

The `01-Step*.md` snapshots trace basic construction, interfaces, and Java configuration.
The `02-Step*.md` snapshots trace managed objects and component-based wiring.
The `03-Step*.md` snapshots develop advanced container features.
The combined step-by-step guide groups those stages under the same three sections.
Use them to understand why code changed, not as additional classes on the runtime classpath.

## Core concepts explained
### Tight versus loose coupling
Tight coupling means a consumer depends directly on implementation details.
An early teaching-stage runner tied to `MarioGame` cannot accept another game unchanged.
Loose coupling means the runner depends on the stable `GamingConsole` interface.

Illustration of tight coupling:
```java
class TightRunner {
    private final MarioGame game = new MarioGame();
}
```
Illustration of loose coupling:
```java
class LooseRunner {
    private final GamingConsole game;

    LooseRunner(GamingConsole game) {
        this.game = game;
    }
}
```
```text
Tight:  Runner ------> MarioGame concrete class

Loose:  Runner ------> GamingConsole contract
                              ^
                    +---------+---------+
                    |         |         |
                  Mario     Pacman   SuperContra
```
An interface is useful when it describes a meaningful substitution boundary.
Merely adding interfaces everywhere does not guarantee good design.
An interface whose every method leaks vendor-specific details is still tightly coupled.

### IoC and DI are related, not identical
**Inversion of Control:** framework/container code controls part of object creation or execution.
**Dependency Injection:** an object's dependencies are supplied from outside that object.
DI is one way to implement IoC for object collaboration.
Other IoC examples include framework-invoked callbacks and event listeners.

Without Spring, `App01GamingBasicJava` chooses the dependency graph.
With Spring, configuration describes the graph and the container constructs it.
Business code focuses on operations rather than locating or assembling services.

| Style | Dependency supplied by | Typical concern |
|---|---|---|
| Internal `new` | The consumer itself | Hardcoded implementation |
| Manual DI | Startup code or tests | Manual graph maintenance |
| Spring DI | Spring container | Correct registration and selection |
| Service locator | Consumer asks a registry | Hidden dependencies and framework coupling |

Calling `getBean()` is reasonable in these launcher demonstrations.
Repeatedly calling it from business methods turns DI into service-location-style code.

### Constructor, setter, and field injection
| Aspect | Constructor | Setter | Field |
|---|---|---|---|
| Dependencies visible at construction | Yes | Not necessarily | No |
| Supports final fields | Yes | No for reassignment | Not recommended |
| Plain unit-test setup | Straightforward | Straightforward with setters | Awkward without reflection/container |
| Good default for mandatory dependency | Yes | Usually not | Usually not |
| Supports optional configuration | With optional/provider forms | Yes | Possible, but less explicit |
| Injection happens | While constructing | After constructing | After constructing |

A single constructor does not require `@Autowired` in Spring 6.
With multiple constructors, make the intended injection strategy explicit.
`@Autowired` defaults to required dependency resolution.
`@Autowired(required=false)` on a setter can allow the setter to be skipped.
`Optional<T>`, `@Nullable`, or `ObjectProvider<T>` can communicate optionality more clearly.
Do not make mandatory collaborators optional merely to hide configuration errors.

### BeanFactory versus ApplicationContext
`BeanFactory` is the fundamental API for obtaining and managing beans.
`ApplicationContext` builds on it and supplies application-level services.
Most applications use an `ApplicationContext`, including every context launcher here.

| Capability | BeanFactory | ApplicationContext |
|---|---|---|
| Bean lookup, scopes, dependency resolution | Core responsibility | Inherited |
| Resource access | Not the core contract | Integrated resource loading |
| Events | Not the core contract | Publication/listeners |
| Internationalization | Not the core contract | `MessageSource` |
| Environment/profiles | Not the core contract | Integrated environment |
| Processor discovery during refresh | Requires explicit orchestration in bare factories | Automatic context infrastructure |
| Typical eager singleton initialization | Bare factory commonly waits for lookup | Non-lazy singletons created on refresh |

Avoid saying BeanFactory can never do lifecycle processing.
A configured bean factory can run registered processors and lifecycle methods.
The practical difference is the richer application-context infrastructure and refresh workflow.

`AnnotationConfigApplicationContext` reads Java configuration and annotations.
`ClassPathXmlApplicationContext` reads classpath XML definitions.
Web contexts add request/session scope support and web integration.

### Definitions, instances, and configuration
A bean definition is metadata: type, factory, scope, qualifiers, dependencies, lifecycle settings.
A bean instance is the actual object produced from that definition.
Registration and instantiation are distinct operations.
The container may register a lazy definition now and instantiate it much later.

| Annotation | Put it on | Purpose |
|---|---|---|
| `@Configuration` | Configuration class | Supplies configuration metadata and bean methods |
| `@Bean` | Factory method | Registers the returned object |
| `@Component` | Class | Marks a scan candidate |
| `@ComponentScan` | Configuration class | Defines where to discover components |

Use `@Bean` for third-party classes, explicit construction logic, or multiple configured instances.
Use `@Component` and stereotypes for owned application classes with straightforward construction.
Both mechanisms produce beans in the same container.
Neither has a separate lifetime system.

Full `@Configuration` defaults to `proxyBeanMethods=true`.
Inter-bean method calls are intercepted to honor container-managed semantics.
With `proxyBeanMethods=false`, direct Java method calls are not intercepted.
Prefer factory-method parameter injection when methods must work without inter-bean proxies.
`@Bean` in a non-configuration component is also processed in lite mode.

### Component scanning and stereotypes
Parameterless `@ComponentScan` starts from the declaring class's package.
It recursively scans subpackages, not parent packages or arbitrary siblings.
The example launchers intentionally isolate topics by scanning their own package.

| Annotation | Intended layer | Additional meaning |
|---|---|---|
| `@Component` | Generic managed object | Basic component marker |
| `@Service` | Business/service layer | Communicates role; no automatic transaction by itself |
| `@Repository` | Persistence layer | Eligible for persistence exception translation with infrastructure |
| `@Controller` | MVC presentation layer | MVC controller stereotype |
| `@RestController` | Response-body MVC controller | `@Controller` plus `@ResponseBody` |

`@Repository` does not create a database connection or generate CRUD operations.
Exception translation requires a configured translator/post-processor and relevant persistence exceptions.
`@Service` does not automatically add retries, security, or transactions.
Those behaviors need additional annotations and infrastructure.

### Resolving multiple beans: primary, qualifier, name, collection
For a typical required single-value injection point:

1. Find candidates compatible with the requested type and injection metadata.
2. Apply qualifier restrictions if present.
3. Among eligible candidates, use a unique primary when appropriate.
4. Other resolution rules, including priority and name fallback, may apply.
5. If no unique choice remains, report ambiguity.

This is a practical model, not a full implementation listing of Spring's resolver.
A qualifier narrows the eligible set; it is not simply a global ranking.
A primary is a default preference among suitable candidates.

| Situation | Result |
|---|---|
| One `GamingConsole` bean | It is injected |
| Several, unique primary Mario | Ordinary unqualified injection selects Mario |
| Super Contra qualifier at injection point | Matching Super Contra is selected |
| Several unresolved candidates, no other discriminator | `NoUniqueBeanDefinitionException` |
| Required type absent | `NoSuchBeanDefinitionException`, often wrapped during creation |
| `List<GamingConsole>` injection | All matching beans, not just primary |
| `getBean("pacmanGame")` | Direct selection by bean name |

An injection failure while creating another bean often surfaces as `UnsatisfiedDependencyException`.
Inspect the nested cause for the precise missing/ambiguous-bean diagnosis.
The default component names here are `marioGame`, `pacmanGame`, and `superContraGame`.
The qualifier `SuperContraGameQualifier` is case-sensitive metadata, not that default name.

### Bean scopes
| Scope | Lifetime/key | Example and warning |
|---|---|---|
| `singleton` | One instance per definition per container | Default `NormalClass`; not one instance for the entire JVM |
| `prototype` | New instance each time container creation is requested | `PrototypeClass`; destruction is caller-managed |
| `request` | One instance per HTTP request | Web context and active request required |
| `session` | One instance per HTTP session | Shared within session; concurrent requests still possible |
| `application` | One per ServletContext | Web scope, distinct from each Spring context's singleton |
| `websocket` | One per WebSocket session | Specialized web integration |

Singleton scope does not make mutable fields thread-safe.
Prefer stateless services and local variables for request-specific data.
A session-scoped object can also face concurrent access from multiple requests.

A prototype injected directly into a singleton is resolved once during singleton construction.
It does not become new every time the singleton method is called.
Use `ObjectProvider<T>.getObject()` or another deliberate lookup strategy for repeated creation.

Illustration:
```java
@Component
class JobRunner {
    private final ObjectProvider<JobState> states;

    JobRunner(ObjectProvider<JobState> states) {
        this.states = states;
    }

    void runJob() {
        JobState state = states.getObject();
    }
}
```
`JobState` would need a prototype definition; these classes are not in the repo.
For injecting a shorter-lived web bean into a singleton, a scoped proxy delegates per request.
`@RequestScope` and `@SessionScope` provide convenient scoped-proxy defaults.
Calling such a proxy outside its active web scope can still fail.

### Lazy versus eager initialization
Eager singleton creation happens during normal application-context refresh.
Benefits: detect wiring errors early and avoid first-use construction latency.
Costs: startup time, resource use, and initializing features that might never be used.

Lazy creation delays the object until needed.
Benefits: lower initial cost and potentially shorter startup.
Costs: delayed failures and first-use latency.
It is not inherently a performance optimization for total workload cost.

`@Lazy` on a bean definition delays that bean when nothing eagerly requires it.
`@Lazy` on an injection point can inject a lazy-resolution proxy.
These are related but different uses.
Boot's `spring.main.lazy-initialization=true` is a broader application setting.
The `d1` launcher does not read Boot startup settings through `SpringApplication.run`.

### Lifecycle: creation, initialization, use, destruction
Simplified lifecycle:
```text
Read/register bean definitions
           |
Instantiate dependencies and bean (constructor)
           |
Populate fields and call injection setters
           |
Aware callbacks and before-initialization bean processors
           |   (@PostConstruct is processed in this stage)
InitializingBean.afterPropertiesSet(), if implemented
           |
Custom init method, if configured
           |
After-initialization processors (may return an AOP proxy)
           |
Application uses managed bean
           |
Context closes / relevant managed scope ends
           |
@PreDestroy -> DisposableBean.destroy() -> custom destroy method
```
This diagram omits advanced processor hooks and special early references.
If several distinct initialization mechanisms exist, annotation callbacks precede interface callbacks,
which precede configured custom initialization methods.
The analogous distinct destruction callbacks follow the same annotation/interface/custom ordering.
Avoid redundantly configuring the same lifecycle operation through several mechanisms.

`@PostConstruct` should be a suitable no-argument instance callback.
Do not assume calling an annotated method manually simulates container lifecycle.
Do not rely on `@PreDestroy` after forced process termination or JVM failure.
Spring does not automatically destroy prototype instances when the context closes.

A bean's `@PostConstruct` runs before its normal final proxy is ready.
Do not assume self-invoked `@Transactional` work there receives proxy advice.
A constructor also should not perform slow remote operations unnecessarily.

### Jakarta injection versus a full CDI runtime
| Spring-specific annotation | Standard Jakarta alternative | Caveat |
|---|---|---|
| `@Component` | `@Named` | Spring recognizes it during scanning |
| `@Autowired` | `@Inject` | No `required=false` attribute on `@Inject` |
| String bean qualifier usage | `@Named("name")` at injection point | Can qualify by bean name |
| Spring custom qualifier annotation | Jakarta custom `@Qualifier` annotation | Different annotation types/packages |

The package is `jakarta.inject`, not legacy `javax.inject`, in this Boot 3 example.
Jakarta `@Named`/`@Inject` integration does not mean all CDI scopes and extensions work in Spring.
For scope choices in this lesson, use Spring's documented scope facilities.
Avoid accidentally importing the wrong `Qualifier` or an unrelated `Named` annotation.

### XML configuration and mixed configurations
XML remains supported in Spring Framework 6.
`<constructor-arg ref="game"/>` supplies another bean.
`<constructor-arg value="35"/>` supplies a literal converted to the target argument type.
`<property name="..." ref="..."/>` supplies a setter property.

`<context:component-scan>` discovers annotated classes and enables related annotation infrastructure.
`<context:annotation-config>` processes annotations on already registered objects without scanning.
Neither is active in the current explicit `h1` XML.
An annotation-based configuration can also import XML with `@ImportResource`.
Mixing formats works, but duplicate definitions and confusing ownership should be avoided.

### Circular dependencies
A circular dependency exists when A needs B and B needs A to be created.
```text
A constructor needs B
       |
       v
B constructor needs A
       |
       +----------> A is still being constructed
```
Constructor cycles cannot normally be satisfied through early singleton references.
Typical failure involves `BeanCurrentlyInCreationException`, often nested in creation exceptions.
Some setter/field singleton cycles can be resolved by a suitably configured plain Spring container.
That ability is not a design recommendation and does not cover every proxy/scoping situation.
Spring Boot 3.3 disallows circular references by default.
Setting `spring.main.allow-circular-references=true` does not solve a genuine constructor cycle.

Better solutions:

- Extract shared responsibility into a third service.
- Make one dependency directional through a clear orchestration layer.
- Publish events where asynchronous or decoupled notification is appropriate.
- Use a provider or lazy injection only when deferred access is a legitimate design choice.
- Do not access a lazy collaborator during construction and recreate the same cycle immediately.

### Spring Framework modules versus Spring projects
Spring Framework is modular; Spring projects build on or alongside it.
The module name and Maven artifact name are related but not always identical.

| Framework area | Common artifacts | Responsibility |
|---|---|---|
| Core container | `spring-core`, `spring-beans`, `spring-context`, `spring-expression` | IoC, resources, contexts, SpEL |
| AOP | `spring-aop`, `spring-aspects` | Proxy-based cross-cutting behavior and AspectJ integration |
| Data access | `spring-jdbc`, `spring-tx`, `spring-orm` | JDBC helpers, transactions, ORM integration |
| Web | `spring-web`, `spring-webmvc`, `spring-webflux` | Web support, servlet MVC, reactive web stack |
| Messaging | `spring-messaging`, `spring-jms` | Messaging abstractions and JMS integration |
| Testing | `spring-test` | Context integration and web testing support |

| Separate Spring project | Main purpose |
|---|---|
| Spring Boot | Starters, auto-configuration, executable packaging, operational support |
| Spring Data | Repository abstractions for supported data stores |
| Spring Security | Authentication, authorization, security integration |
| Spring Cloud | Distributed-system integrations and patterns |
| Spring Batch | Restartable, structured batch workloads |
| Spring Integration | Enterprise integration patterns |
| Spring Session | Externalized/distributed HTTP session management |

Spring MVC is part of Spring Framework, not an alternative DI container.
Spring Boot uses Spring Framework rather than replacing it.
Spring Data MongoDB is not automatically activated by naming a class `MongoDbDataService`.

## Common pitfalls / gotchas
### Registration mistakes
- Creating `new YourBusinessClass(...)` yourself does not trigger annotation injection.
- Annotating a class does not help if its package is outside the configured scan.
- A classpath dependency is not automatically an application bean.
- Two registration mechanisms can accidentally define the same logical collaborator twice.
- A bean-definition listing is metadata inspection, not an object-construction log.
- Moving an example launcher changes parameterless scan boundaries.

### Selection mistakes
- A qualifier may be custom metadata rather than the default bean name.
- Parameter names are not a reliable substitute for explicit qualifiers.
- Two primary candidates for the same required type can remain ambiguous.
- An incompatible primary is not eligible just because it is primary.
- Primary selection does not discard other beans from collection injection.
- In the gaming launcher, direct lookup uses Mario while runner injection uses Super Contra.
- In the XML launcher, an explicit Pacman reference wins because it supplies the constructor argument directly.

### Scope and lifecycle mistakes
- Spring singleton means per bean definition per context, not per JVM.
- Singleton does not mean immutable or thread-safe.
- Prototype injected once into a singleton is not recreated per method invocation.
- Prototype destruction is not automatically managed after the container hands it out.
- Request/session scopes are not available in the plain non-web example context.
- A lazy dependency can still be constructed to satisfy an eager consumer.
- Field-injected dependencies are not ready during object construction.
- `@PreDestroy` is not guaranteed after a forced kill.
- Lifecycle callbacks on objects never managed by Spring will not run automatically.

### Source-reading mistakes
- `c1` does not connect to real databases and its launcher does not print the maximum.
- `a1`'s setter methods and `@Autowired` constructor annotation are commented teaching alternatives.
- `g1.DataService` and `c1.DataService` are different types.
- `contextConfiguration.out.xml` is empty and unused.
- The JavaBean helper's EJB comment is not an EJB implementation.
- Older step snapshots can use historical paths absent from current source.
- Java 21 is the repo target; Java 17 is Spring 6/Boot 3's minimum baseline.

## Interview questions & answers
### Basic
#### 1. What is Spring Framework?
A modular Java framework supplying IoC/DI, data-access integration, transaction support,
web frameworks, and testing infrastructure.
In this module, the central feature is the application context managing Java objects.
#### 2. What problem does loose coupling solve?
It reduces the need to change a consumer when one implementation changes.
`GameRunner` depends on `GamingConsole`, so Mario, Pacman, or Super Contra can be supplied.
The runner still requires the contract to remain compatible.
#### 3. Is dependency injection possible without Spring?
Yes. `new GameRunner(new PacmanGame())` supplies the dependency externally.
Spring automates graph creation and provides management around those objects.
#### 4. What is IoC?
IoC transfers some control from application code to a framework/container.
Spring decides when configured beans are created, injected, initialized, and destroyed.
DI is one concrete mechanism used to accomplish that transfer.
#### 5. What is a Spring bean?
An object managed by the Spring container.
It can be a POJO, record, third-party object, or infrastructure object.
It does not need JavaBean getters/setters or to implement a Spring interface.
#### 6. What are the three common DI types?
Constructor, setter, and field injection.
Constructor injection is the preferred default for required collaborators.
The current `a1` example actively uses constructor injection.
#### 7. Why is `@Autowired` unnecessary in `a1`'s constructor?
The class has one constructor, so Spring selects it automatically.
All required constructor arguments must still be resolvable beans.
The commented annotation does not disable injection.
#### 8. What does `@Configuration` do?
It marks a class as a source of bean definitions and configuration.
In full mode, Spring enhances it so calls between `@Bean` methods use container semantics.
It must still be registered, imported, or discovered to take effect.
#### 9. What does `@Bean` do?
It registers the object returned by an annotated factory method.
The default bean name is the method name, unless a name is explicitly specified.
`GamingConfiguration.game()` registers a Pacman instance as `game`.
#### 10. How does `@ComponentScan` choose packages?
With no argument, it scans the declaring class's package and descendants.
The gaming launcher explicitly scans `com.in28minutes.learnspringframework.game`.
It does not scan every sibling topic automatically.
#### 11. What is the default scope?
Singleton: one shared instance for that bean definition within one container.
Repeated `NormalClass` lookups in `e1` return the same object.
Another context can have another singleton for the same Java class.
#### 12. What is prototype scope?
The container creates a new instance for each creation request for that definition.
Repeated `PrototypeClass` lookups in `e1` produce separate objects.
The caller must manage their later cleanup when necessary.

### Intermediate
#### 13. BeanFactory or ApplicationContext: which should an application normally use?
ApplicationContext, because it integrates bean management with events, resources,
environment, messages, and automatic processor orchestration.
BeanFactory is the foundational abstraction, useful for lower-level infrastructure.
#### 14. When would you choose `@Bean` over `@Component`?
For third-party classes, multiple differently configured instances, or explicit factory logic.
Use component scanning for owned classes where straightforward discovery is appropriate.
Both register ordinary Spring beans with the same scope/lifecycle model.
#### 15. What happens if two beans have the same type and neither is primary?
If no qualifier, applicable name fallback, or other discriminator resolves the single-value request,
Spring reports `NoUniqueBeanDefinitionException`.
During constructor wiring this is commonly nested in `UnsatisfiedDependencyException`.
#### 16. What happens when no required bean exists?
Resolution fails with a missing-bean diagnosis such as `NoSuchBeanDefinitionException`.
If the failure happens while constructing a consumer, inspect its nested creation exception.
Check registration and scanning before making the dependency optional.
#### 17. How do `@Primary` and `@Qualifier` differ?
Primary supplies a default preference among eligible candidates.
Qualifier narrows candidate eligibility at a specific injection point.
The runner's Super Contra qualifier excludes Mario despite Mario being primary.
#### 18. What does the annotation-based gaming launcher print first?
The direct `getBean(GamingConsole.class).up()` call selects primary Mario and prints `Jump`.
The subsequently retrieved runner executes Super Contra because its constructor is qualified.
Identity-string details in `Running game: ...` are runtime-dependent.
#### 19. Which address reaches `person5Qualifier`?
The `address3` bean containing Motinagar, Hyderabad.
Its qualifier is `address3qualifier`, explicitly requested by the parameter annotation.
The primary London bean is not eligible for that qualified injection.
#### 20. Does the parameter named `address3` always select that bean?
No. Parameter-name matching is a fallback, and the primary `address2` wins in the ordinary
unqualified single-value resolution used by the current configuration.
Use `@Qualifier` for deliberate selection.
#### 21. Does `@Primary` affect injection of `List<GamingConsole>`?
It does not reduce the list to one element.
All matching candidate beans are injected, subject to qualifier filtering if present.
If order matters, specify ordering semantics rather than relying on incidental discovery order.
#### 22. What result would `c1.findMax()` return, and why?
`55`, because primary `MongoDbDataService` supplies `{11, 22, 33, 44, 55}`.
The unchanged launcher only lists definitions; you must retrieve the service and call the method.
These repositories do not query actual databases.
#### 23. How would you unit-test that business service without Spring?
Construct it with a fake implementation and call `findMax()`.
For example, this illustration uses the single-method interface as a lambda:
```java
DataService fake = () -> new int[] {3, 9, 1};
var service = new BusinessCalculationService(fake);
assertEquals(9, service.findMax());
```
#### 24. What happens if the fake returns an empty array?
`findMax()` returns `0` because the stream maximum is empty and `orElse(0)` supplies a default.
For an all-negative nonempty array, it returns the actual maximum, not zero.
A null array would cause a null-related failure rather than use the empty-stream default.
#### 25. Does `@Service` automatically make a method transactional?
No. It communicates a service role and supports component discovery.
Transactions need transaction infrastructure and an applicable transaction declaration.
Similarly, `@Repository` alone does not implement database access.
#### 26. What is the output order in `d1`?
Context-completed message, `Some Initialization logic`, then `Do Something`.
Lazy `ClassB` is instantiated when explicitly requested after context refresh.
The eager `ClassA` can already exist before that lookup.
#### 27. When does `@PostConstruct` run?
After object construction and dependency population during initialization processing.
In `f1`, it calls `SomeDependency.getReady()` before normal use of `SomeClass`.
It is not invoked merely because a programmer creates an object with `new`.
#### 28. When does `@PreDestroy` run?
For eligible managed beans when the context or relevant managed scope ends normally.
The `f1` context is closed by try-with-resources, so its singleton cleanup callback runs.
It is not automatically applied to handed-out prototype instances.

### Advanced and scenario-based
#### 29. A prototype is constructor-injected into a singleton. Is it new on each call?
No. Direct injection resolves it once while the singleton is created.
Use `ObjectProvider<T>.getObject()` per operation if each operation needs a fresh prototype.
Do not confuse a new bean-creation request with a normal call on an existing object.
#### 30. Can a singleton service safely store the current user's cart in a field?
Not as shared mutable request-specific state.
Different requests can use the same singleton concurrently and overwrite one another's data.
Use an explicit per-request parameter or an appropriate web scope with concurrency considerations.
#### 31. What if `@RequestScope` is added to the plain `e1` example?
The plain `AnnotationConfigApplicationContext` does not provide the normal web request scope.
Looking up the scoped target can fail because the scope is unregistered or inactive.
Use a web-aware application context and access it within an HTTP request.
#### 32. Does lazy initialization always keep a bean uncreated until direct lookup?
No. An eager singleton's ordinary dependency resolution can require it sooner.
Injection-point `@Lazy` can defer resolution through a proxy where appropriate.
Deferred creation also defers configuration errors and moves latency to first use.
#### 33. Why can constructor cycles fail even when circular references are enabled?
Neither object's constructor can complete before the other object exists.
Early references normally depend on an already instantiated object, which a pure constructor cycle lacks.
Refactor the graph rather than relying on a setting to solve the impossible ordering.
#### 34. Why does Boot reject some cycles that plain Spring might resolve?
Boot disables circular references by default as an application policy.
A suitably configured plain Spring factory can resolve some setter/field singleton cycles.
This does not mean all cycles are supported or that resolved cycles are good design.
#### 35. What changes with `@Configuration(proxyBeanMethods=false)`?
Calls between configuration methods become ordinary Java calls rather than intercepted bean lookups.
A direct `address()` call can create another instance if invoked from a factory method.
Parameter injection preserves explicit container wiring without needing those method-call proxies.
#### 36. Can `@PostConstruct` safely call its own `@Transactional` method?
Do not assume transaction advice applies.
Initialization occurs before normal final proxy use, and self-invocation bypasses proxy interception.
Move transactional startup work to an appropriately invoked managed collaborator or startup phase.
#### 37. Does importing `jakarta.inject.Inject` start a CDI container?
No. The API defines annotations; Spring can interpret supported ones using its own container.
The `g1` launcher is still `AnnotationConfigApplicationContext`, not Weld or another CDI runtime.
Full CDI features require their own supported runtime/integration.
#### 38. Why does XML wiring select Pacman despite the runner's qualifier?
The XML explicitly supplies `ref="game"` as the constructor argument.
There is no unresolved annotated candidate choice at that argument site.
The active XML's `game` definition is a Pacman instance and component scanning is commented out.
#### 39. What is the difference between a BeanFactoryPostProcessor and BeanPostProcessor?
A BeanFactoryPostProcessor works with container metadata before normal bean instantiation.
A BeanPostProcessor participates around individual bean initialization and may return a wrapped object.
Auto-proxying and annotation lifecycle processing use the latter kind of extension point.
#### 40. Is a Spring singleton the same as a GoF singleton?
No. Spring scope is enforced by a container for a particular bean definition.
The Java class need not restrict its constructors, and another container can manage another instance.
Two singleton definitions of the same class can also produce two separate instances.
#### 41. What if a bean's initialization callback throws an exception?
Bean creation fails; eager startup may fail the context refresh.
For a lazy bean, the failure may occur only when it is first requested.
Report the underlying initialization error rather than assuming DI alone is at fault.
#### 42. How should you explain Spring modules versus Spring Boot in one sentence?
Spring Framework supplies the core container and application abstractions;
Spring Boot assembles and auto-configures those abstractions with useful defaults and packaging.
The core examples use Boot-managed libraries without delegating startup to `SpringApplication`.

## Hands-on exercises
Try these later in a personal working copy; they are not changes already made by this guide.
Predict the result before running each modified launcher.

### Exercise 1: swap the game without editing the runner
1. Run `App01GamingBasicJava` with its current Pacman construction.
2. Change only its chosen game to `MarioGame` in your copy.
3. Confirm that `GameRunner.java` does not need to change.
4. Repeat in Framework 01 by changing the object returned by `GamingConfiguration.game()`.

**Success criterion:** the same runner executes different operation messages.
**Interview lesson:** the composition root changes while the consumer depends on an abstraction.

### Exercise 2: prove primary and qualifier selection
1. Run Framework 02's gaming launcher unchanged.
2. Explain why the first operation is Mario but the runner is Super Contra.
3. Remove the runner's qualifier in your copy and predict primary Mario selection.
4. Remove Mario's primary marker too and observe ambiguity with three candidates.
5. Restore an explicit qualifier and confirm deterministic selection.

**Success criterion:** distinguish direct type lookup, qualified injection, and ambiguous injection.
**Interview lesson:** inspect nested creation exceptions and injection-site metadata.

### Exercise 3: test business logic independently
1. Add a unit test in your copy that constructs `BusinessCalculationService` with a fake `DataService`.
2. Test `{3, 9, 1}`, an empty array, and `{-8, -2, -4}`.
3. Expect `9`, `0`, and `-2` respectively.
4. Add a launcher call to `findMax()` and compare the primary repository's result.

**Success criterion:** unit tests require neither a Spring context nor a database.
**Interview lesson:** DI improves testability; a business service need not know persistence details.

### Exercise 4: separate scope from lookup frequency
1. Run `e1` and compare repeated object identities.
2. In your copy, inject a prototype into a singleton and call a method twice.
3. Show that the directly injected prototype reference remains the same.
4. Replace direct injection with an `ObjectProvider` and request an object per method call.
5. Add cleanup logging and observe that prototype cleanup is not automatic on context close.

**Success criterion:** explain when a new container creation request occurs.
**Interview lesson:** scope belongs to bean creation/lifetime, not arbitrary Java method calls.

### Exercise 5: observe initialization timing
1. Run `d1` and `f1` separately with explicit main-class selection.
2. Remove `@Lazy` from `ClassB` in your copy and predict the changed output order.
3. Create an eager consumer of `ClassB` and compare definition-level versus injection-point laziness.
4. In `f1`, observe initialization before use and cleanup after the try block closes.
5. Construct an unmanaged `SomeClass` in a suitable same-package test and compare callback behavior.

**Success criterion:** distinguish constructor execution, dependency population, initialization, and close.
**Interview lesson:** annotations are metadata interpreted by the container, not magical Java keywords.

## Quick revision cheat sheet
| Prompt | Fast answer |
|---|---|
| Tight coupling | Consumer depends on a concrete implementation |
| Loose coupling | Consumer depends on a meaningful stable contract |
| DI | Supply collaborators externally |
| IoC | Container/framework owns part of construction or execution control |
| Preferred injection | Constructor for mandatory dependencies |
| Single constructor | No `@Autowired` needed |
| Spring bean | Container-managed object, regardless of JavaBean conventions |
| BeanFactory | Foundational bean-management API |
| ApplicationContext | Bean management plus environment, resources, events, messages, refresh |
| `@Bean` | Factory method registers returned object |
| `@ComponentScan` | Discovers candidates in chosen package tree |
| Primary | Default preference among eligible single-value candidates |
| Qualifier | Injection-site candidate restriction |
| Ambiguous required type | `NoUniqueBeanDefinitionException`, possibly wrapped |
| Default scope | Singleton per definition per container |
| Prototype | New object per creation request; caller owns later destruction |
| Request/session | Web context plus active relevant scope |
| Lazy definition | Delay unless earlier dependency resolution needs it |
| `@PostConstruct` | Initialization after dependency population |
| `@PreDestroy` | Eligible managed cleanup on normal close/scope end |
| Jakarta imports | `jakarta.annotation.*`, `jakarta.inject.*` for these examples |
| CDI annotation support | Does not turn Spring into a full CDI implementation |
| Constructor cycle | Refactor; early singleton references cannot generally solve it |
| XML in Spring 6 | Supported; explicit refs and values still configure objects |

Final source-specific memory anchors:

- Framework 01 manual and `@Bean` gaming both currently use Pacman.
- Framework 02 direct gaming lookup uses primary Mario; its runner uses qualified Super Contra.
- `person4Parameters` is primary `Person`; `address2` is primary `Address`.
- `person5Qualifier` explicitly receives Hyderabad through `address3qualifier`.
- `a1` actively demonstrates constructor injection, not its commented setters.
- `c1` chooses hardcoded Mongo data; `findMax()` would return `55`, but the launcher only lists beans.
- `d1` prints context completion before lazy `ClassB` construction.
- `e1` compares singleton and prototype retrieval; it does not demonstrate HTTP scopes directly.
- `f1` uses Jakarta lifecycle annotations and closes its context explicitly.
- `g1` uses Jakarta standard injection annotations inside Spring.
- `h1` loads `contextConfiguration.xml` and explicitly wires Pacman.
- Boot manages dependency versions here; the example code still controls context startup explicitly.
