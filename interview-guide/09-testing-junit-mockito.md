# 09 — Testing with JUnit 5, Mockito, and Spring Boot

## What this module teaches

Testing gives executable evidence that behavior matches requirements.
This chapter connects the course's small Java examples to realistic Spring tests.
The baseline is **Java 21 and Spring Boot 3.3.4** where a Boot project exists.
The standalone JUnit folder is not a Spring Boot Maven module.

### Exact module paths and how to run

| Module path | Layout | Run strategy |
|---|---|---|
| `51-junit/` | `src/` and `test/`; no POM or Gradle build | IDE JUnit runner |
| `52-mockito/` | Standard Maven `src/main/java` and `src/test/java` | Installed Maven |

From the repository root:

```bash
# All Mockito-module tests, including its Spring context smoke test
mvn -f 52-mockito/pom.xml test

# Just the business unit tests and list examples
mvn -f 52-mockito/pom.xml -Dtest=SomeBusinessImplMockTest,SomeBusinessImplStubTest,ListTest test

# One test class
mvn -f 52-mockito/pom.xml -Dtest=SomeBusinessImplMockTest test
```

For `51-junit/`, import the directory as a Java project into your IDE.
Mark `src` as source and `test` as test source.
Supply JUnit Jupiter and the JUnit 4 assertion API used by `MyAssertTest`.
Run `MyMathTest` or `MyBeforeAfterTest` using the IDE's JUnit runner.
Running all three classes includes an intentionally failing array assertion.
Do not assume `./mvnw` exists in either extracted module.
These instructions describe commands; this documentation task does not execute them.

## Project walkthrough

All snippets in this section are excerpts of actual repository files.
Later examples marked **illustrative** are extensions, not existing course code.

### 1. The production unit: `MyMath`

Path: `51-junit/src/com/in28minutes/junit/MyMath.java`.

```java
public int calculateSum(int[] numbers) {
    
    int sum = 0;
    
    for(int number:numbers) {
        sum += number;
    }
    
    return sum;
}
```

This method has no database, HTTP call, or Spring dependency.
Its output depends only on the supplied array.
It is a good unit-testing target because setup is inexpensive.
An empty array returns zero because the loop never executes.
Negative values work with the same accumulation logic.
A null array currently throws `NullPointerException`.
Integer arithmetic can overflow; there is no overflow check.
Neither null validation nor overflow handling should be invented as existing behavior.

### 2. Existing sum tests

Path: `51-junit/test/com/in28minutes/junit/MyMathTest.java`.

```java
private MyMath math = new MyMath();

@Test
void calculateSum_ThreeMemberArray() {
    assertEquals(6, math.calculateSum(new int[] {1,2,3}));
}

@Test
void calculateSum_ZeroLengthArray() {
    assertEquals(0, math.calculateSum(new int[] {}));
}
```

Read an assertion as **expected value, actual value**.
These tests encode the normal case and an empty-input boundary.
The names identify the method and scenario rather than implementation details.
Each test should remain independently executable.
The `math` object is created directly: no Spring container is involved.

### 3. Assertions demonstration and a deliberate failure

Path: `51-junit/test/com/in28minutes/junit/MyAssertTest.java`.

```java
import static org.junit.Assert.assertArrayEquals;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertTrue;
```

```java
assertTrue(test);
assertFalse(test2);
//assertNull,assertNotNull
assertArrayEquals(new int[] {1,2}, new int[] {2, 1});

assertEquals(3, todos.size());
```

The class uses Jupiter's `@Test` but imports array equality from JUnit 4.
That import requires the JUnit 4 API to compile.
It does **not** make the class a Vintage test: the test annotation is Jupiter.
The arrays have the same members in different positions, so equality fails.
Ordinary assertions stop execution at the first failure.
Therefore the following list-size assertion is not reached in this run.
For a new test, prefer consistent `org.junit.jupiter.api.Assertions` imports.

### 4. Lifecycle hooks

Path: `51-junit/test/com/in28minutes/junit/MyBeforeAfterTest.java`.

```java
@BeforeAll
static void beforeAll() {
    System.out.println("beforeAll");
}

@BeforeEach
void beforeEach() {
    System.out.println("BeforeEach");
}
```

```java
@AfterEach
void afterEach() {
    System.out.println("AfterEach");
}

@AfterAll
static void afterAll() {
    System.out.println("afterAll");
}
```

The class has three test methods: `test1`, `test2`, and `test3`.
Their names do not guarantee execution order.
An illustrative execution is:

```text
beforeAll
  BeforeEach -> test2 -> AfterEach
  BeforeEach -> test1 -> AfterEach
  BeforeEach -> test3 -> AfterEach
afterAll
```

Jupiter normally creates a new test-class instance for each test method.
Static all-class hooks work with this default `PER_METHOD` lifecycle.
`@TestInstance(PER_CLASS)` permits non-static all-class hooks.
Shared mutable state in that mode needs careful reset.

### 5. Business logic with a collaborator

Path: `52-mockito/src/main/java/com/in28minutes/mockito/mockitodemo/business/SomeBusinessImpl.java`.

```java
public SomeBusinessImpl(DataService dataService) {
    super();
    this.dataService = dataService;
}

public int findTheGreatestFromAllData() {
    int[] data = dataService.retrieveAllData();
    int greatestValue = Integer.MIN_VALUE;
    for(int value:data) {
        if(value > greatestValue)
            greatestValue = value;
    }
    return greatestValue;
}
```

The same source file declares its collaborator:

```java
interface DataService {
    int[] retrieveAllData();
}
```

`DataService` is package-private, not a separate public source file.
Tests referencing it must be in the business package.
Constructor injection lets a test provide any compatible implementation.
No `@Service` annotation appears on this business class.
Its empty-array result is the sentinel `Integer.MIN_VALUE`.
Starting at zero would be wrong for arrays containing only negative numbers.
Null data currently fails in the loop; that contract is not explicitly validated.

### 6. Handwritten stub tests

Path: `52-mockito/src/test/java/com/in28minutes/mockito/mockitodemo/business/SomeBusinessImplStubTest.java`.

```java
DataService dataServiceStub = new DataServiceStub1();
SomeBusinessImpl businessImpl = new SomeBusinessImpl(dataServiceStub);
int result = businessImpl.findTheGreatestFromAllData();
assertEquals(25, result);
```

```java
class DataServiceStub1 implements DataService {

    @Override
    public int[] retrieveAllData() {
        return new int[]{25, 15, 5};
    }
    
}
```

`DataServiceStub2` returns `new int[]{35}`.
The two tests expect 25 and 35 respectively.
A stub supplies predetermined data; these tests assert returned state.
A separate stub class for every scenario becomes repetitive.
A configurable handwritten stub would also be a valid alternative.
Mockito removes that boilerplate without changing the production contract.

### 7. Annotation-based Mockito tests

Path: `52-mockito/src/test/java/com/in28minutes/mockito/mockitodemo/business/SomeBusinessImplMockTest.java`.

```java
@ExtendWith(MockitoExtension.class)
class SomeBusinessImplMockTest {
    
    @Mock
    private DataService dataServiceMock;
    
    @InjectMocks
    private SomeBusinessImpl businessImpl;
```

```java
@Test
void findTheGreatestFromAllData_basicScenario() {
    when(dataServiceMock.retrieveAllData()).thenReturn(new int[]{25, 15, 5});
    assertEquals(25, businessImpl.findTheGreatestFromAllData());
}
```

The other tests return `{35}` and `{}`.
They expect 35 and `Integer.MIN_VALUE`.
`MockitoExtension` initializes annotated mocks and manages Mockito validation.
`@InjectMocks` creates the subject and attempts injection, here through its constructor.
It is not Spring dependency injection and does not discover an application context.
Although called a mock, this test double is used as a stub in these assertions.
There are no `verify` calls in the existing business tests.

### 8. Every `ListTest` scenario

Path: `52-mockito/src/test/java/com/in28minutes/mockito/mockitodemo/list/ListTest.java`.

**`simpleTest`:**

```java
List listMock = mock(List.class);
//listMock.size() => 3
when(listMock.size()).thenReturn(3);
assertEquals(3, listMock.size());
```

The file repeats the assertion four times; the stub is reusable.
The raw `List` is course shorthand; use `List<String>` in new examples.
A mock is not a real list with three stored items.
Stubbing `size()` does not make `get()` return actual elements.

**`multipleReturns`:**

```java
when(listMock.size()).thenReturn(1).thenReturn(2);
assertEquals(1, listMock.size());
assertEquals(2, listMock.size());
assertEquals(2, listMock.size());
```

After the configured sequence is exhausted, the last answer repeats.
This can model retry outcomes or evolving collaborator responses.
Do not use call sequencing when input-specific stubbing is clearer.

**`specificParameters`:**

```java
when(listMock.get(0)).thenReturn("SomeString");
assertEquals("SomeString", listMock.get(0));
assertEquals(null, listMock.get(1));
```

An unstubbed reference-returning method typically returns null.
Unlike a real empty list, this mock does not throw for index one by default.
Mockito defaults can hide missing setup unless assertions exercise the behavior.

**`genericParameters`:**

```java
when(listMock.get(Mockito.anyInt())).thenReturn("SomeOtherString");
assertEquals("SomeOtherString", listMock.get(0));
assertEquals("SomeOtherString", listMock.get(1));
```

`anyInt()` matches any integer argument, including a negative integer.
It does not implement real list bounds checking.
Use exact arguments when the requested argument is part of the requirement.

### 9. Context smoke test and Maven dependency

Path: `52-mockito/src/test/java/com/in28minutes/mockito/mockitodemo/MockitoDemoApplicationTests.java`.

```java
@SpringBootTest
class MockitoDemoApplicationTests {

    @Test
    void contextLoads() {
    }

}
```

This verifies that the Spring Boot context starts.
It does not prove every business method is correct.
`52-mockito/pom.xml` declares `spring-boot-starter-test` with test scope.
The parent is Boot 3.3.4 and `java.version` is 21.
The starter supplies Jupiter, Mockito, Spring testing, and assertion libraries.
The production application uses `@SpringBootApplication` and `SpringApplication.run`.

## Core concepts explained

### Unit, integration, and end-to-end tests

```text
                 /-----------\
                / Few E2E     \
               / browser + API \
              /-----------------\
             / Integration/slices \
            / real wiring or DB    \
           /------------------------\
          / Many fast unit tests      \
         / deterministic business rules\
        /------------------------------\
```

| Level | Boundary under test | Example | Typical trade-off |
|---|---|---|---|
| Unit | One behavior in isolation | Sum or greatest-value algorithm | Fast, limited wiring confidence |
| Integration | Collaborating real components | Repository against MySQL | Slower, realistic infrastructure |
| E2E | User workflow through deployed system | Login and save todo in browser | Broad confidence, fragile setup |

The pyramid is a heuristic, not a mandatory percentage distribution.
A controller slice sits between isolated logic and a full system test.
An integration test does not always mean loading every application component.
Use the smallest boundary capable of detecting the failure you care about.
Keep a few representative E2E flows rather than repeating every unit case there.

### Arrange, Act, Assert

1. **Arrange:** construct the subject and prepare collaborator responses.
2. **Act:** invoke one meaningful operation.
3. **Assert:** check observable results and required interactions.

Keep setup close to the scenario unless shared setup improves clarity.
Avoid asserting private fields or every internal method call.
Tests should describe contracts that survive sensible refactoring.
A test that passes only in suite order is not isolated.

### JUnit 5 architecture

```text
IDE / Maven Surefire / Gradle test task
                   |
              JUnit Platform
            discovery + execution
              /            \
       Jupiter engine    Vintage engine
       JUnit 5 tests     JUnit 3/4 tests
```

**Platform** provides the launcher and test-engine contracts.
**Jupiter** provides the modern programming model and its execution engine.
**Vintage** allows legacy JUnit 3/4 test execution when deliberately included.
JUnit 5 is the umbrella, not a single annotation package.
Jupiter imports are under `org.junit.jupiter`.
JUnit 4's `@Test` is `org.junit.Test`.
Mixing assertion imports is distinct from mixing execution engines.

### Annotation reference

| Annotation | Purpose | Useful caution |
|---|---|---|
| `@Test` | Ordinary test method | Public modifier is not required |
| `@BeforeEach` | Setup for each invocation | Parameterized invocations get separate setup |
| `@AfterEach` | Cleanup after each invocation | Prefer resource-safe cleanup |
| `@BeforeAll` | One-time class setup | Normally static |
| `@AfterAll` | One-time class cleanup | Normally static |
| `@DisplayName` | Human-readable report label | Method name still helps searches |
| `@Disabled` | Skip test/class with reason | Do not hide failures indefinitely |
| `@ParameterizedTest` | Repeat logic across data | Needs an argument source |
| `@Nested` | Group related scenarios | Usually non-static inner class |
| `@ExtendWith` | Register extension behavior | Mockito lifecycle is an extension |

Illustrative Jupiter examples using the existing `MyMath`:

```java
@DisplayName("Adding numbers")
@Nested
class AddingNumbers {
    @ParameterizedTest(name = "{0} + {1} = {2}")
    @CsvSource({"1,2,3", "-2,-3,-5", "0,0,0"})
    void sumsTwoInputs(int left, int right, int expected) {
        assertEquals(expected, math.calculateSum(new int[]{left, right}));
    }
}
```

Import `ParameterizedTest` from `org.junit.jupiter.params`.
Import `CsvSource` from `org.junit.jupiter.params.provider`.
Use `@MethodSource` for arrays, objects, and more complex datasets.
A source method is normally static unless the lifecycle permits otherwise.
`@Disabled("Pending agreed overflow contract")` should include a trackable reason.

### Assertions that communicate intent

| Assertion | Meaning | Example |
|---|---|---|
| `assertEquals` | Equal expected/actual values | `assertEquals(6, sum)` |
| `assertTrue` / `assertFalse` | Boolean condition | Membership check |
| `assertNull` / `assertNotNull` | Reference presence | Optional lookup result |
| `assertArrayEquals` | Same elements in same order | Sum-input transformations |
| `assertThrows` | Expected exception type or subtype | Invalid/null input |
| `assertAll` | Collect failures from grouped executables | Response fields |
| `assertTimeout` | Check completion time after execution | Bounded operation |

Illustrative examples, not additional repository tests:

```java
@Test
void nullInput_currentlyThrows() {
    assertThrows(NullPointerException.class,
        () -> math.calculateSum(null));
}

@Test
void groupedChecks() {
    assertAll("sum boundaries",
        () -> assertEquals(0, math.calculateSum(new int[]{})),
        () -> assertEquals(-5, math.calculateSum(new int[]{-2, -3})));
}

@Test
void completesWithinBudget() {
    assertTimeout(Duration.ofSeconds(1),
        () -> math.calculateSum(new int[]{1, 2, 3}));
}
```

`assertThrows` returns the exception for further checks.
`assertThrowsExactly` rejects subtypes when that distinction matters.
`assertTimeout` is not a hard stop; the executable runs on the calling thread.
`assertTimeoutPreemptively` uses another thread and can disrupt transaction context.
Prefer generous timeout budgets; tight timing assertions cause CI flakiness.

### Stubs, mocks, spies, and fakes

| Double | Purpose | Real behavior? | Example |
|---|---|---|---|
| Stub | Supply canned answers | No meaningful implementation | `DataServiceStub1` |
| Mock | Program answers and verify interactions | Usually no | Mockito `DataService` |
| Spy | Observe/partially override real object | Yes unless stubbed | Spy around an existing adapter |
| Fake | Simplified working implementation | Yes, simplified | In-memory repository backed by a map |

A dummy is merely supplied to satisfy a parameter and is never meaningfully used.
The categories describe usage, not rigid Java class hierarchies.
A Mockito-created mock can be used only for stubbing.
A fake should honor enough of the production contract to be trustworthy.
Prefer real value objects and collections over mocking everything.

### Mockito stubbing and failures

Illustrative test in the same package as `DataService`:

```java
when(dataServiceMock.retrieveAllData()).thenReturn(new int[]{-7, -2, -9});
assertEquals(-2, businessImpl.findTheGreatestFromAllData());

when(dataServiceMock.retrieveAllData())
    .thenThrow(new IllegalStateException("Data unavailable"));
assertThrows(IllegalStateException.class,
    businessImpl::findTheGreatestFromAllData);
```

Checked exceptions must be allowed by the mocked method's signature.
For void methods, use `doThrow(...).when(mock).operation()`.
For spies, prefer `doReturn(value).when(spy).method()`.
`when(spy.method())` invokes the real method while defining the stub.
That can cause I/O, mutation, or exceptions before the test's Act phase.

### Interaction verification

```java
businessImpl.findTheGreatestFromAllData();
verify(dataServiceMock, times(1)).retrieveAllData();
```

`verify(mock).method()` defaults to one invocation.
`never()` expresses a forbidden call.
`atLeastOnce()` is weaker than an exact invocation count.
`InOrder` verifies sequence only when sequence is part of the contract.
Avoid blanket `verifyNoMoreInteractions` checks that couple tests to internals.
Assert state first; verify interactions for requirements such as one notification.

### Matchers and argument capture

Illustrative collaborators below are not in the repository:

```java
when(client.lookup(eq("tenant-a"), anyInt())).thenReturn("found");

ArgumentCaptor<Message> captor = ArgumentCaptor.forClass(Message.class);
verify(publisher).send(captor.capture());
assertEquals("order-created", captor.getValue().type());
```

If one argument uses a matcher, all arguments in that call must use matchers.
`eq(...)` turns a literal requirement into a matcher.
`any()` matches reference values broadly; typed `any(Class)` excludes null.
Use `isNull()` when null is the expected input.
Matchers are not general-purpose values; use them only in stubbing/verification.
A captor inspects what was passed after the subject executes.
Use `getAllValues()` when multiple captured invocations are meaningful.
Prefer exact stubbing over capture during setup.

### Strict stubs

`MockitoExtension` uses strict-stubbing behavior by default.
Unused stubs often indicate unnecessary setup or a wrong execution path.
Argument mismatches can reveal that the subject called a different operation.
Put scenario-specific stubs in the relevant test rather than universal setup.
Use `lenient()` selectively for genuinely optional shared behavior.
Do not disable strictness merely to make a broken test turn green.

### Static mocking

Mockito's inline mock maker supports static mocking; Mockito 5 uses inline by default.
This is separate from Spring's bean substitution.
Static mocks should have a short, explicit lifetime.

```java
// Illustrative: ClockSource is an example helper, not repository code.
try (MockedStatic<ClockSource> clock = mockStatic(ClockSource.class)) {
    clock.when(ClockSource::today).thenReturn(LocalDate.of(2030, 1, 1));
    assertEquals(LocalDate.of(2030, 1, 1), ClockSource.today());
}
```

The scope is thread-local; work on another thread is not automatically mocked.
Close the scope to avoid contamination of later tests.
For new design, inject `java.time.Clock` rather than hiding time behind statics.
On Java 21, agent instrumentation warnings can require build-level configuration.
Use a Mockito-supported agent setup rather than ignoring instrumentation forever.

### Spring Boot test boundaries

| Tool | Loads | Starts real server? | Good for |
|---|---|---|---|
| Plain Jupiter + Mockito | No Spring context | No | Business rules |
| `@WebMvcTest` | MVC slice, relevant web infrastructure | No | Routing, JSON, validation, security |
| `@DataJpaTest` | JPA slice | No | Mappings and repository queries |
| `@SpringBootTest` default | Full application context | Usually no | Wiring and configuration |
| `@SpringBootTest(RANDOM_PORT)` | Full context + embedded server | Yes | Real HTTP integration |

`MockMvc` drives the MVC pipeline without opening a network socket.
A slice generally excludes service implementations; replace or import collaborators.
`@DataJpaTest` is transactional and normally rolls back after each test.
Its default database replacement behavior can select an embedded database.
For production SQL confidence, test with the same database engine, often Testcontainers.
H2 success does not prove MySQL compatibility.

### MVC slice example

Illustrative controller/service types; not existing Mockito-module beans:

```java
@WebMvcTest(GreetingController.class)
class GreetingControllerTest {
    @Autowired MockMvc mvc;
    @MockBean GreetingService service;

    @Test
    void greetingReturnsText() throws Exception {
        when(service.greet()).thenReturn("Hello");
        mvc.perform(get("/greeting"))
            .andExpect(status().isOk())
            .andExpect(content().string("Hello"));
    }
}
```

Security filters can make unauthenticated requests return 401 or redirects.
Use an appropriate authenticated test user when security is in the contract.
CSRF tokens may be needed for state-changing requests.
Do not disable filters just to avoid testing an intended security requirement.

### `@MockBean` versus `@MockitoBean`: version matters

For this repository's **Boot 3.3.4 / Spring Framework 6.1**, use `@MockBean`.
Its package is `org.springframework.boot.test.mock.mockito`.
It adds or replaces a bean in the Spring test application context.
A plain `@Mock` does not replace an autowired Spring bean.

`@MockitoBean` is a Spring Framework **6.2+** feature, used with Boot 3.4+.
Its package is `org.springframework.test.context.bean.override.mockito`.
Boot 3.4 deprecates Boot's `@MockBean` in favor of that mechanism.
Do not paste `@MockitoBean` into a Boot 3.3.4 project without upgrading its baseline.
Spring bean overriding and `@InjectMocks` solve different problems.

### Real HTTP with `TestRestTemplate`

```java
// Illustrative full-server test; this pattern requires a web application.
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class GreetingHttpTest {
    @Autowired TestRestTemplate http;

    @Test
    void returnsGreetingOverHttp() {
        ResponseEntity<String> response =
            http.getForEntity("/greeting", String.class);
        assertEquals(HttpStatus.OK, response.getStatusCode());
    }
}
```

A random port prevents collisions between parallel test processes.
Real server requests execute outside the test method's transaction.
A test-level rollback does not automatically undo server-side writes.
Clean data explicitly or isolate databases for those tests.
Spring's context cache improves repeated compatible context startup.
Unnecessary `@DirtiesContext` usage discards that benefit.

### TDD: red, green, refactor

```text
Requirement -> Red failing test -> Green simplest implementation
     ^                                      |
     |----------- Refactor safely ----------|
```

Red proves the test detects the missing behavior.
Green supplies the smallest correct behavior, not necessarily final design.
Refactor improves structure while keeping observable behavior unchanged.
For greatest-value logic, an all-negative test catches initialization to zero.
Agree the empty-input contract before changing its sentinel behavior.
TDD is a feedback discipline, not a ban on architectural thinking.

### Coverage and useful test names

Line coverage reports executed lines; branch coverage reports decision outcomes.
JaCoCo is a common JVM coverage tool, but is not configured in these small examples.
High coverage does not prove correct assertions, boundary coverage, or good requirements.
Mutation testing asks whether small wrong-code changes are detected.
Avoid chasing 100% coverage of trivial generated code at the expense of risky paths.

Good naming forms include:

- `method_scenario_expectedOutcome`.
- `returnsGreatestValue_whenAllValuesAreNegative`.
- `rejectsUpdate_whenUserDoesNotOwnTodo`.

The existing `findTheGreatestFromAllData_EmptyArray` is scenario-oriented.
A name such as `test1` is useful for lifecycle output, not business documentation.

## Common pitfalls / gotchas

| Pitfall | Why it fails | Better approach |
|---|---|---|
| Running `51-junit` with Maven | No build descriptor exists | IDE runner with correct libraries |
| Expecting every course assertion to pass | Array-order example deliberately fails | Read expected/actual values |
| Jupiter test with missing JUnit 4 API | Mixed static import does not compile | Consistent Jupiter assertions |
| Mock fields stay null | No Mockito extension/init | Register `MockitoExtension` |
| Mock returns null data | Collaborator was never stubbed | Arrange required responses |
| Empty array expected as zero | Existing maximum contract returns MIN_VALUE | Assert actual agreed contract |
| Spying with `when` | Real method executes during setup | `doReturn` for risky methods |
| Mixing matcher and literal | Mockito matcher count is invalid | Use `eq` for literals |
| Full context for every unit | Slow, unrelated failures | Use plain unit tests |
| H2 equals production MySQL | Dialects/constraints differ | Real-engine integration test |
| Hardcoded port | Concurrent test collision | Random server port |
| Shared mutable fixtures | Order dependence | Reset per test |
| Sleeps for async work | Timing-dependent failures | Await a condition with a deadline |
| Excessive interaction checks | Refactoring breaks valid behavior | Verify contract-significant calls |
| Using `@MockitoBean` on Boot 3.3 | Annotation not on this baseline | Use `@MockBean` here |

## Interview questions & answers

### Basic

#### 1. What is a unit test?
A fast test of a narrowly scoped behavior in isolation.
`MyMath.calculateSum` is an example because it has no external collaborator.

#### 2. What is an integration test?
A test proving that real collaborating components work together.
A repository query against a real MySQL database checks more than Java logic.

#### 3. Why prefer many unit tests and fewer E2E tests?
Unit tests are fast and localize failures; E2E tests provide broad but costly confidence.
Use both, without duplicating every algorithm case through a browser.

#### 4. What are Platform, Jupiter, and Vintage?
Platform launches engines, Jupiter provides modern JUnit tests, and Vintage runs legacy tests.
Adding JUnit 4 assertions alone does not turn a Jupiter test into a Vintage test.

#### 5. What does `@BeforeEach` do?
It prepares each test invocation, including each parameterized invocation.
Use it for independent fixtures rather than relying on previous tests.

#### 6. Why is `@BeforeAll` usually static?
Jupiter's default per-method lifecycle does not share a test instance for all methods.
`PER_CLASS` lifecycle permits non-static all-class hooks.

#### 7. What is the correct `assertEquals` argument order?
Expected first, actual second.
Correct order makes failure reports easier to diagnose.

#### 8. Why does `MyAssertTest` fail?
It compares `{1,2}` with `{2,1}` using ordered array equality.
The later size assertion is never reached after that failure.

#### 9. What is a stub?
A test double returning predetermined answers.
The repository's `DataServiceStub1` always returns `{25,15,5}`.

#### 10. What does `@Mock` do?
It declares a Mockito mock initialized by Mockito lifecycle support.
It does not create or replace a Spring bean on its own.

### Intermediate

#### 11. What does `@InjectMocks` do?
It attempts to build the subject and inject available mocks, preferring constructor injection.
It is a convenience, not a complete dependency-injection container.

#### 12. How do you test a dependency throwing an exception?
Stub `thenThrow` and use `assertThrows` on the subject's operation.
Assert propagation or translation according to the intended contract.

#### 13. What does `verify(mock, times(1))` prove?
That a matching method invocation occurred exactly once.
It does not prove the returned business result is correct.

#### 14. When should you use `ArgumentCaptor`?
When you need to inspect an argument produced by the subject, such as an emitted event.
Capture during verification and assert meaningful fields.

#### 15. Why must matcher arguments be consistent?
Mockito records matcher expressions for an invocation.
If any argument is matched, wrap required literals with `eq(...)` as well.

#### 16. What happens after chained `thenReturn(1, 2)` answers run out?
Mockito repeats the final answer, two.
That is exactly the behavior demonstrated by `ListTest.multipleReturns`.

#### 17. What is a spy, and why can stubbing it be dangerous?
A spy normally calls real methods.
`when(spy.method())` can execute production effects during setup; use `doReturn` where needed.

#### 18. What does strict stubbing catch?
Unused setup and likely mismatches between expected and actual collaborator calls.
Fix the scenario rather than globally making all stubs lenient.

#### 19. What is `@WebMvcTest` for?
A focused MVC test of controllers, serialization, validation, and relevant security filters.
It does not load the entire service/data application by default.

#### 20. What does `@DataJpaTest` normally roll back?
Changes performed in its test transaction.
It cannot automatically roll back separate server-side transactions from real HTTP requests.

#### 21. `@Mock`, `@MockBean`, or `@MockitoBean` on this repo?
Use `@Mock` for isolated Mockito tests and `@MockBean` for Boot 3.3.4 context substitution.
`@MockitoBean` requires the later Spring Framework 6.2 baseline.

#### 22. What is the difference between `assertAll` and ordinary assertions?
Ordinary assertions stop at the first failure; `assertAll` aggregates failures from its executables.
Group related checks, not unrelated test scenarios.

### Advanced and scenario-based

#### 23. A greatest-value test with positive values passes, but production negatives fail. Why?
An implementation initialized to zero would pass positive-only tests and fail negative-only arrays.
Add an all-negative test; the current course implementation correctly starts at MIN_VALUE.

#### 24. An MVC test returns 401 instead of 200. What do you investigate?
Check whether security filters require authentication and whether the endpoint is intentionally protected.
Supply a suitable test principal instead of reflexively removing security.

#### 25. Why does a full-server test leave database rows after rollback?
HTTP handlers run on different threads and their own transactions.
Use explicit cleanup or isolated databases for those integration tests.

#### 26. A static mock works on the test thread but not an async worker. Why?
Static mocking is thread-scoped and does not transparently propagate to worker threads.
Inject the collaborator, especially `Clock`, to make asynchronous logic testable.

#### 27. Does 100% line coverage mean the application is correct?
No; lines can execute without strong assertions or meaningful edge cases.
Combine branch coverage, risk-based tests, and optionally mutation testing.

#### 28. How do you test retries without sleeping?
Stub sequential failures/success and inject a scheduler or time abstraction.
Assert eventual outcomes and meaningful invocation counts deterministically.

#### 29. When is a fake better than a mock?
When many tests benefit from coherent stateful behavior, such as save-then-find semantics.
Still test the real adapter because fake behavior can drift from production.

#### 30. How would you introduce TDD for empty input?
First agree whether empty input returns a sentinel or throws.
Write a failing test for the new contract, implement it, then refactor with all other cases green.

#### 31. A test passes alone and fails in the suite. What are likely causes?
Shared static state, unclosed static mocks, unreleased resources, or order-dependent fixtures.
Run independently and inspect lifecycle cleanup rather than adding ordering annotations.

#### 32. Why not use `assertTimeoutPreemptively` everywhere?
It runs work in another thread, potentially losing transaction/security thread-local context.
Use it only when its interrupting execution semantics match the test boundary.

## Hands-on exercises

### Exercise 1 — Strengthen sum boundaries

Add tests in a separate practice copy for negative numbers and a single zero.
Add a characterization test for null input without changing production behavior.
Explain the current integer overflow behavior before proposing a new contract.
Success: each test has one clear scenario and runs independently.

### Exercise 2 — Compare three doubles

Test greatest-value behavior with a handwritten configurable stub and a Mockito mock.
Create a simple stateful fake only if its richer behavior has a meaningful purpose.
Explain which test checks state and which checks interactions.
Success: no database or Spring context is required.

### Exercise 3 — Verify collaborator usage

Add an all-negative case and verify retrieval happens once.
Add an exception case using `thenThrow` and `assertThrows`.
Keep the empty-array sentinel test unchanged unless deliberately changing the contract.
Success: setup is local and strict stubbing remains enabled.

### Exercise 4 — Add focused Spring testing

In a practice web module, add a controller slice with `MockMvc` and `@MockBean`.
Then add a random-port HTTP integration test with `TestRestTemplate`.
Document which wiring bugs the full test can catch that the slice cannot.
Success: tests use the correct Boot 3.3.4 annotations and handle security intentionally.

### Exercise 5 — Review test quality

Choose a method with a conditional branch and inspect line/branch coverage.
Deliberately change a comparison operator in a practice copy.
Explain why surviving tests reveal a gap even if coverage remains high.
Success: improve a meaningful assertion or boundary, not just execution counts.

## Quick revision cheat sheet / commands

```bash
# From repository root; installed Maven and JDK 21 required
mvn -f 52-mockito/pom.xml test
mvn -f 52-mockito/pom.xml -Dtest=SomeBusinessImplMockTest test
mvn -f 52-mockito/pom.xml -Dtest=ListTest test
mvn -f 52-mockito/pom.xml -Dtest=SomeBusinessImplMockTest#findTheGreatestFromAllData_EmptyArray test

# Inspect test dependencies without changing source files
mvn -f 52-mockito/pom.xml dependency:tree -Dscope=test
```

| Remember | One-line answer |
|---|---|
| Test strategy | Many fast rules tests; fewer wiring and E2E checks |
| AAA | Arrange, Act, Assert |
| Jupiter lifecycle | New instance per test by default |
| Stub | Supplies answers |
| Mock | Can supply answers and verify interactions |
| Spy | Calls real code unless overridden |
| Fake | Simplified working implementation |
| Mockito annotations | Extension initializes; Mock creates; InjectMocks injects |
| Matchers | All arguments use matchers when any one does |
| Captor | Inspect subject-generated arguments after execution |
| Strict stubs | Remove unused or mismatched setup |
| Boot 3.3.4 context replacement | `@MockBean`, not the newer `@MockitoBean` |
| MVC test | Slice + MockMvc, no real socket |
| Real HTTP | RANDOM_PORT + TestRestTemplate |
| Coverage | Execution evidence, not correctness proof |
| Naming | Behavior + scenario + expected outcome |

Final memory hook: **test the contract at the smallest trustworthy boundary**.
