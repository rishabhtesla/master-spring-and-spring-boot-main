# 08 — Spring AOP: Proxies, Pointcuts, Logging, and Timing
## What this module teaches
The course module demonstrates how to apply logging and performance measurement
without putting those responsibilities inside every business method.
This guide follows the actual code in `73-spring-aop/` and explains its behavior
under Spring Boot **3.3.4**, Java **21**, and Spring Framework **6.1**.

Spring AOP is proxy-based interception of method execution on Spring-managed objects.
It uses AspectJ annotations and pointcut syntax without necessarily using AspectJ weaving.
The distinction is crucial in interviews and when debugging missed advice.

### Exact module path and how to run
Module: `73-spring-aop/`.
From the repository root:

```bash
cd 73-spring-aop
./gradlew bootRun
```

The application implements `CommandLineRunner` and runs two calculations at startup.
It has the basic Boot starter, not the web starter.
There is no REST endpoint or embedded HTTP server to curl in this example.
A normal successful run completes its calculations and exits when no non-daemon work remains.

Run the existing test task from the same module directory:

```bash
./gradlew test
```

The Gradle Java toolchain requests Java 21.
Ensure the JDK/toolchain and dependency download prerequisites are available.
These are study instructions; writing this guide does not run or alter the application.

### What you should observe
- `BusinessService1.calculateMax()` returns `55`.
- `BusinessService2.calculateMin()` returns `111`.
- Before advice applies to bean names matching `*Service*`.
- After advice applies to business-package methods.
- After-returning advice applies to data-package methods.
- After-throwing advice is configured, but normal sample execution throws no business exception.
- Timing advice currently uses `@TrackTime`, not the commented package-wide expression.
- `BusinessService1.calculateMax()` and `DataService2.retrieveData()` are the two annotated target methods.

Expect roughly 30 ms or more around paths containing the demonstration sleep,
not an exact timing assertion: OS scheduling, logging, JIT compilation, and startup affect observations.
Do not assume the two aspects have a deterministic relative order; neither declares `@Order`.

## Project walkthrough
### 1. Build configuration and AOP activation
File: `73-spring-aop/build.gradle`.

```groovy
id 'org.springframework.boot' version '3.3.4'
id 'io.spring.dependency-management' version '1.1.6'
id 'java'
```

```groovy
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}
```

Relevant dependencies:

```groovy
implementation 'org.springframework.boot:spring-boot-starter'
implementation 'org.springframework.boot:spring-boot-starter-aop'
testImplementation 'org.springframework.boot:spring-boot-starter-test'
```

The AOP starter supplies Spring AOP and AspectJ support libraries.
Boot's AOP auto-configuration enables proxy support when its conditions are satisfied.
The application does not need to add `@EnableAspectJAutoProxy` manually for this normal Boot setup.
Plain non-Boot Spring configuration typically enables that infrastructure explicitly.
Adding `aspectjweaver` as a library does not mean compile-time or load-time weaving is active.

File: `73-spring-aop/src/main/resources/application.properties`.

```properties
spring.main.banner-mode=off
logging.pattern.console= %d{MM-dd HH:mm:ss} - %logger{36} - %msg%n
```

The properties customize output, not pointcuts or advice ordering.
There is no explicit `spring.aop.proxy-target-class` property here.
Boot normally defaults that property to `true`, favoring class-based CGLIB proxies.

### 2. Application entry point and CommandLineRunner
File: `73-spring-aop/src/main/java/com/in28minutes/learnspringaop/LearnSpringAopApplication.java`.

```java
@SpringBootApplication
public class LearnSpringAopApplication implements CommandLineRunner{
```

Both business services are constructor-injected:

```java
public LearnSpringAopApplication(BusinessService1 businessService1,
        BusinessService2 businessService2) {
    this.businessService1 = businessService1;
    this.businessService2 = businessService2;
}
```

The startup runner invokes:

```java
@Override
public void run(String... args) throws Exception {
    logger.info("businessService1 Value returned is {}", businessService1.calculateMax());
    logger.info("businessService2 Value returned is {}", businessService2.calculateMin());
}
```

Calls originate outside the business objects and go through their injected references.
If a bean is advised, that reference is normally its proxy rather than its raw target.
This external call pattern is why advice can execute without explicit logging calls in the services.
The runner is not matched merely because it calls matched services.
Its package/name do not satisfy the demonstrated service pointcuts.

### 3. BusinessService1: maximum and opt-in timing
File: `73-spring-aop/src/main/java/com/in28minutes/learnspringaop/aopexample/business/BusinessService1.java`.

```java
@Service
public class BusinessService1 {
```

Its constructor receives `DataService1`.
The business operation is:

```java
@TrackTime
public int calculateMax() {
    int[] data = dataService1.retrieveData();

    //throw new RuntimeException("Something Went Wrong!");
    return Arrays.stream(data).max().orElse(0);
}
```

`@TrackTime` opts this operation into the annotation timing pointcut.
The data service's default data is `[11, 22, 33, 44, 55]`.
The maximum is `55`; empty data would yield `0` because of `orElse(0)`.
The exception line is commented out and does not run.

### 4. BusinessService2: minimum without direct timing
File: `73-spring-aop/src/main/java/com/in28minutes/learnspringaop/aopexample/business/BusinessService2.java`.

```java
public int calculateMin() {
    int[] data = dataService2.retrieveData();

    //throw new RuntimeException("Something Went Wrong!");
    return Arrays.stream(data).min().orElse(0);
}
```

This class is also a `@Service` with constructor injection.
The operation does **not** have `@TrackTime`.
Its data service does have the annotation, so timing appears for the nested data call,
not directly for the unannotated business operation under the current pointcut.
The default result is `111`.

### 5. DataService1: repository stereotype and delayed array
File: `73-spring-aop/src/main/java/com/in28minutes/learnspringaop/aopexample/data/DataService1.java`.

```java
@Repository
public class DataService1 {
```

```java
public int[] retrieveData() {
```

The method sleeps as a simple latency demonstration:

```java
try {
    Thread.sleep(30);
} catch (InterruptedException e) {
    // TODO Auto-generated catch block
    e.printStackTrace();
}
```

It returns:

```java
return new int[] {11, 22, 33, 44, 55};
```

This is not actual database access despite the repository stereotype.
The method has no `@TrackTime` annotation.
It is still matched by the logging aspect's data-package and bean-name expressions.
The swallowed interrupt is a course-code caveat; a production design should preserve interruption semantics.

### 6. DataService2: annotation on the data boundary
File: `73-spring-aop/src/main/java/com/in28minutes/learnspringaop/aopexample/data/DataService2.java`.

```java
@Repository
public class DataService2 {

    @TrackTime
    public int[] retrieveData() {
```

The same `Thread.sleep(30)` pattern appears here.
The returned array is:

```java
return new int[] {111, 222, 333, 444, 555};
```

Because the injected data-service reference is a bean proxy, the nested external call
from `BusinessService2` crosses another interceptable boundary.
Nested calls are not automatically self-invocation: self-invocation means calling a method on the same target.

### 7. TrackTime custom annotation
File: `73-spring-aop/src/main/java/com/in28minutes/learnspringaop/aopexample/annotations/TrackTime.java`.

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface TrackTime {

}
```

`METHOD` restricts where the marker can be placed.
`RUNTIME` makes the annotation available to runtime interception and reflection.
The annotation itself has no timing behavior.
The `@Around` advice supplies the behavior by matching annotated method executions.
The name expresses intent without making the business service depend on timer implementation details.

### 8. CommonPointcutConfig: named reusable selections
File: `73-spring-aop/src/main/java/com/in28minutes/learnspringaop/aopexample/aspects/CommonPointcutConfig.java`.

This helper is an ordinary class containing named `@Pointcut` declarations.
It is not annotated with `@Configuration` or `@Aspect` in the source.
The advice declarations refer to its public pointcuts by fully qualified name.
These declarations are expression definitions, not business methods that must run.

```java
@Pointcut("execution(* com.in28minutes.learnspringaop.aopexample.*.*.*(..))")
public void businessAndDataPackageConfig() {}
```

```java
@Pointcut("execution(* com.in28minutes.learnspringaop.aopexample.business.*.*(..))")
public void businessPackageConfig() {}
```

```java
@Pointcut("execution(* com.in28minutes.learnspringaop.aopexample.data.*.*(..))")
public void dataPackageConfig() {}
```

```java
@Pointcut("bean(*Service*)")
public void allPackageConfigUsingBean() {}
```

```java
@Pointcut("@annotation(com.in28minutes.learnspringaop.aopexample.annotations.TrackTime)")
public void trackTimeAnnotation() {}
```

The broad package pattern matches the current business/data package structure.
It is not a recursive `..` pattern for arbitrarily deep future subpackages.
`allPackageConfigUsingBean` is a descriptive method name, not a guarantee of matching all beans.
It selects bean names containing the case-sensitive text `Service`.
Default bean names include `businessService1`, `businessService2`, `dataService1`, and `dataService2`.

### 9. LoggingAspect: before advice
File: `73-spring-aop/src/main/java/com/in28minutes/learnspringaop/aopexample/aspects/LoggingAspect.java`.

```java
@Configuration
@Aspect
public class LoggingAspect {
```

`@Aspect` declares aspect metadata.
`@Configuration` also registers this scanned class as a Spring bean.
An unregistered `@Aspect` class would not supply advice to Spring AOP.
A common alternative is `@Component` plus `@Aspect`.

Real before advice:

```java
@Before("com.in28minutes.learnspringaop.aopexample.aspects.CommonPointcutConfig.allPackageConfigUsingBean()")//WHEN
public void logMethodCallBeforeExecution(JoinPoint joinPoint) {
    logger.info("Before Aspect - {} is called with arguments: {}"
            ,  joinPoint, joinPoint.getArgs());//WHAT
}
```

The expression supplies the selection: “when”.
The method body supplies the behavior: “what”.
All four service beans match the active bean-name pointcut.
The current calculation/retrieval methods take no parameters, so their argument arrays are empty.
Do not copy unrestricted argument logging into methods handling passwords or tokens.

### 10. LoggingAspect: after-finally advice
```java
@After("com.in28minutes.learnspringaop.aopexample.aspects.CommonPointcutConfig.businessPackageConfig()")
public void logMethodCallAfterExecution(JoinPoint joinPoint) {
    logger.info("After Aspect - {} has executed",  joinPoint);
}
```

This advice applies to business-package methods, not to the data methods.
`@After` means after-finally: it applies on normal and exceptional completion.
The log message “has executed” is not proof of successful completion.
The distinction from `@AfterReturning` is an important interview question.

### 11. LoggingAspect: after-throwing advice
```java
@AfterThrowing(
pointcut = "com.in28minutes.learnspringaop.aopexample.aspects.CommonPointcutConfig.businessAndDataPackageConfig()",
throwing = "exception"
)
public void logMethodCallAfterException(JoinPoint joinPoint, Exception exception) {
    logger.info("AfterThrowing Aspect - {} has thrown an exception {}"
            ,  joinPoint, exception);
}
```

`throwing = "exception"` binds the escaping exception to the advice parameter.
Its declared `Exception` type restricts the binding to compatible exceptions.
It does not generally bind JVM `Error` subclasses.
This advice logs but does not swallow or replace the exception.
An exception caught inside a target method is not an escaping exception for that method's after-throwing advice.

### 12. LoggingAspect: after-returning advice
```java
@AfterReturning(
pointcut = "com.in28minutes.learnspringaop.aopexample.aspects.CommonPointcutConfig.dataPackageConfig()",
returning = "resultValue"
)
public void logMethodCallAfterSuccessfulExecution(JoinPoint joinPoint,
        Object resultValue) {
    logger.info("AfterReturning Aspect - {} has returned {}"
            ,  joinPoint, resultValue);
}
```

It observes normal return values from the data package.
The returned values here are primitive `int[]` arrays bound as objects.
Array formatting depends on the logger's argument formatting; avoid assuming raw Java `toString` semantics.
The advice cannot replace a return value by returning a different object.
Use around advice if replacement is genuinely required.

### 13. PerformanceTrackingAspect: around advice
File: `73-spring-aop/src/main/java/com/in28minutes/learnspringaop/aopexample/aspects/PerformanceTrackingAspect.java`.

```java
@Aspect
@Configuration
public class PerformanceTrackingAspect {
```

Its active pointcut is:

```java
@Around("com.in28minutes.learnspringaop.aopexample.aspects.CommonPointcutConfig.trackTimeAnnotation()")
public Object findExecutionTime(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
```

The package-wide around annotation immediately above it is commented out.
The active timer body contains:

```java
long startTimeMillis = System.currentTimeMillis();
Object returnValue = proceedingJoinPoint.proceed();
long stopTimeMillis = System.currentTimeMillis();
long executionDuration = stopTimeMillis - startTimeMillis;
```

```java
logger.info("Around Aspect - {} Method executed in {} ms"
        ,proceedingJoinPoint, executionDuration);

return returnValue;
```

`proceed()` invokes the remainder of the interceptor chain, ultimately reaching the target.
It is not necessarily a direct jump to the target method.
Returning the result preserves the target's normal result semantics.
The method declares `throws Throwable` because `proceed()` can propagate arbitrary throwables.
If the target throws, this implementation does not reach its stop/log lines.
For exception-inclusive timing, use `try/finally` in a practice improvement.
For elapsed durations, use monotonic `System.nanoTime()` rather than wall-clock milliseconds.

### 14. Trace the actual object graph
```text
LearnSpringAopApplication runner
  |
  +--> BusinessService1 proxy
  |       | Before logging / annotation timing / After business logging
  |       v
  |    BusinessService1 target.calculateMax()
  |       |
  |       +--> DataService1 proxy
  |               | Before logging / AfterReturning data logging
  |               v
  |            DataService1 target.retrieveData() -> [11..55]
  |       |
  |       +--> max = 55
  |
  +--> BusinessService2 proxy
          | Before logging / After business logging
          v
       BusinessService2 target.calculateMin()
          |
          +--> DataService2 proxy
                  | Before logging / annotation timing / AfterReturning data logging
                  v
               DataService2 target.retrieveData() -> [111..555]
          |
          +--> min = 111
```

The diagram lists applicable advice, not a promised relative order between aspects.
The timer for `calculateMax` includes its nested data retrieval.
The timer for `DataService2.retrieveData` does not measure the whole `calculateMin` calculation.
Nested timing spans overlap; summing all durations can double-count elapsed time.

## Core concepts explained
### Cross-cutting concerns
A business service should primarily express the business operation.
Logging, timing, transactions, caching, and authorization often apply across many services.
Repeating those concerns in every method creates duplication and inconsistent policy.
AOP selects operations and attaches reusable behavior at an interception boundary.

```text
Without AOP                         With AOP
Service A: log -> logic -> log       Logging aspect
Service B: log -> logic -> log          |
Service C: log -> logic -> log       proxies -> focused service logic
```

AOP is not a reason to hide essential domain logic.
“Calculate maximum” belongs in the business service.
“Measure selected method executions” belongs naturally in an aspect.
Critical security/transaction requirements should remain discoverable through annotations and tests.

### AOP terminology
| Term | Meaning | Repository example |
|---|---|---|
| Aspect | Module of cross-cutting behavior and selections | `LoggingAspect` |
| Advice | Code executed at a selected interception point | `logMethodCallBeforeExecution` |
| Pointcut | Predicate selecting join points | `trackTimeAnnotation()` |
| Join point | A point in execution eligible for interception | `calculateMax()` execution |
| Target | Object whose behavior is advised | Raw `BusinessService1` instance |
| Proxy | Object callers use to trigger interceptors | Injected business-service reference |
| Weaving | Connecting aspects with execution | Runtime proxy creation in Spring AOP |
| Advisor | Spring combination of advice and matching metadata | Infrastructure built from an aspect method |
| Introduction | Adding an interface/behavior through a proxy | Not used in this module |

In full AspectJ, join points include more than methods.
In Spring AOP, supported join points are method executions on proxied Spring beans.
The term “weaving” is sometimes used loosely for proxy creation here;
AspectJ bytecode weaving is a different implementation mechanism.

### The proxy model
```text
caller
  |
  v
proxy
  |
  v
interceptor 1 -> interceptor 2 -> target method
  ^                                    |
  +------------- result/exception -----+
```

Spring creates proxies during bean lifecycle processing where advice applies.
Dependency injection supplies the proxy to other beans.
Calling `new BusinessService1(...)` yourself bypasses that managed proxy.
Having the same Java annotations on a manually created object does not activate Spring advice.

The proxy can intercept before execution and while results/exceptions unwind.
An around interceptor may stop the chain by never calling `proceed()`.
Therefore AOP can change behavior, not merely observe it.
Choose the least powerful advice needed to reduce accidental semantic changes.

### Advice types and contracts
| Advice | Runs when | Access | Main caveat |
|---|---|---|---|
| `@Before` | Before invocation | Arguments/signature | Cannot replace result; can abort by throwing |
| `@After` | After normal/exceptional completion | Join-point metadata | Not evidence of success |
| `@AfterReturning` | On normal completion | Compatible returned value | Cannot replace return value |
| `@AfterThrowing` | On compatible escaping throwable | Bound exception | Does not recover automatically |
| `@Around` | Wraps continuation | `ProceedingJoinPoint` | Must preserve proceed/result/error semantics deliberately |

For a simple conceptual composition:

```text
Around enters
  Before
    Target runs
  AfterReturning on success OR AfterThrowing on compatible failure
  After-finally
Around completes or propagates exception
```

Actual details depend on aspect precedence and which advice is nested where.
Within an aspect, Spring defines advice-type precedence for the modern annotation model.
Do not infer order from source line order, method names, or console output from one run.
Multiple same-type advice methods in one aspect should not depend on an undocumented ordering.
Use explicit aspect ordering or consolidate dependent behavior.

### JoinPoint versus ProceedingJoinPoint
`JoinPoint` provides reflective execution metadata:

```java
joinPoint.getSignature();
joinPoint.getArgs();
joinPoint.getTarget();
joinPoint.getThis();
```

`getTarget()` identifies the advised target.
`getThis()` identifies the proxy seen in this Spring AOP context.
`getArgs()` returns the current invocation arguments.
`getSignature()` describes the matched method signature.

`ProceedingJoinPoint` adds continuation control:

```java
Object result = proceedingJoinPoint.proceed();
```

A `proceed(Object[] args)` overload can invoke using replacement arguments.
That is powerful and should preserve method contracts and validation assumptions.
For simple timing, call `proceed()` exactly once and return the result unchanged.
Calling twice executes the remaining chain twice, which can duplicate writes or external calls.

### Pointcut expression anatomy: execution
Repository expression:

```text
execution(* com.in28minutes.learnspringaop.aopexample.business.*.*(..))
          ^                  package                       ^ ^  ^
          |                                                | |  |
     any return type                                  any type | any argument list
                                                          any method
```

General shape:

```text
execution([modifiers] return-type declaring-type.method-name(parameters) [throws-pattern])
```

Square-bracketed components are optional notation, not literal characters.
Examples below are explanatory alternatives, not current source declarations:

```java
@Pointcut("execution(public int com.in28minutes.learnspringaop.aopexample.business.*.*(..))")
public void publicIntBusinessMethods() {}
```

```java
@Pointcut("execution(* com.in28minutes.learnspringaop.aopexample.data.*.retrieveData())")
public void noArgumentDataRetrieval() {}
```

| Symbol | Meaning |
|---|---|
| `*` in return position | Any return type |
| `*` in type/method-name position | Wildcard for that name segment |
| `()` | No arguments |
| `(*)` | Exactly one argument of any type |
| `(..)` | Zero or more arguments |
| `..` in package pattern | Recursive package traversal |
| `+` after a type pattern | Include matching subtypes |

The repository's `aopexample.*.*.*(..)` matches one package segment beneath `aopexample`,
one type segment, then one method segment.
It currently reaches `business` and `data` classes.
For arbitrary nested subpackages, an intentionally recursive pattern is different:

```text
execution(* com.in28minutes.learnspringaop.aopexample..*(..))
```

Broad recursive pointcuts can unintentionally capture future beans; test boundaries explicitly.

### within: select by declaring type
`within` is not used by the current source; it is a useful comparison syntax.
It selects method executions within matching types in Spring AOP.

```java
@Pointcut("within(com.in28minutes.learnspringaop.aopexample.business..*)")
public void insideBusinessTypes() {}
```

Unlike a full `execution` signature, it does not specify return type, method name, or parameters.
Use it for package/type-level architectural boundaries.
Remember that Spring's method-execution/proxy limitation still applies.
A matching type does not make constructors, fields, or internal self-calls interceptable.

### bean: select by Spring bean name
Current source:

```text
bean(*Service*)
```

This is a Spring AOP-specific pointcut designator, not portable full AspectJ syntax.
It targets bean names/aliases rather than the fully qualified class name.
Default component bean names decapitalize the class's first letter.
`businessService1` contains `Service`, so it matches.
An explicit bean name such as `@Service("maxCalculator")` could stop matching this pointcut.
Name-based selection is convenient but can make harmless naming refactors change behavior.

### @annotation: select opt-in methods
Current source:

```text
@annotation(com.in28minutes.learnspringaop.aopexample.annotations.TrackTime)
```

This selects executions whose matched method carries the annotation.
It currently selects the business maximum operation and second data retrieval operation.
The marker is meaningful only when the matching aspect bean/infrastructure is active.
Put operational annotations on concrete implementation methods unless you have verified interface resolution behavior.
Interface method annotations are not inherited as ordinary implementation method annotations.
Class-level annotations require a different design, such as `@within`, and an appropriate annotation target.

### Compose and reuse pointcuts
Named pointcut methods centralize selection decisions.
The repo's logging/timing aspects reference `CommonPointcutConfig` instead of repeating package strings.
A future package rename can then be reflected in one selection definition.
Their empty method bodies are intentional.
They are not callback implementations.

Conceptual composition:

```java
@Pointcut("businessPackageConfig() || dataPackageConfig()")
public void businessOrData() {}

@Pointcut("businessPackageConfig() && trackTimeAnnotation()")
public void timedBusinessMethods() {}
```

| Operator | Meaning |
|---|---|
| `&&` | Both selections match |
| `||` | Either selection matches |
| `!` | Selection must not match |

Use fully qualified references when referring across packages/classes as the repo does.
Use stable package boundaries or explicit annotations instead of brittle accidental name conventions.
Named selections are part of an architecture contract and deserve tests.

### Spring AOP versus AspectJ
| Dimension | Spring AOP | Full AspectJ |
|---|---|---|
| Mechanism | Runtime proxies | Compile-time or load-time bytecode weaving |
| Object coverage | Proxied Spring beans | Woven objects, including non-Spring objects |
| Join points | Method execution | Execution/call, fields, constructors, and more |
| Self-invocation | Bypasses ordinary proxy interception | Can be advised through weaving |
| Final/private behavior | Restricted by proxy mechanism | Broader woven join-point support |
| Syntax | AspectJ annotations/selected pointcuts plus Spring `bean` | Full AspectJ language/tooling |
| Setup | Straightforward Boot starter | Additional weaving build/runtime configuration |
| Typical fit | Service-level enterprise concerns | Deep instrumentation beyond bean boundaries |

`@Aspect` alone does not select between these mechanisms.
The module uses `@Aspect` metadata with Spring runtime proxies.
Do not enable weaving merely to solve a problem that a clean service boundary can solve.
Build complexity, deployment compatibility, and observability should influence the choice.

### JDK dynamic proxies versus CGLIB
| Dimension | JDK dynamic proxy | CGLIB class proxy |
|---|---|---|
| Proxy shape | Implements target interfaces | Subclasses target class |
| Requires interface? | Yes, for exposed operations | No |
| Calls exposed | Interface methods | Overridable class methods |
| Final target class | Can proxy its interfaces | Cannot subclass it |
| Final target method | Can intercept via an interface proxy | Cannot override/advice the final method |
| Private method | Not an exposed interface operation | Cannot override/advice it |
| Injection guidance | Inject interfaces | Concrete injection possible |

Spring Framework proxy selection traditionally prefers interfaces when available unless class proxying is forced.
Boot normally defaults to class proxying through `spring.aop.proxy-target-class=true`.
The service classes in this module do not declare interfaces, so class proxies are appropriate here.
Spring repackages CGLIB support; no standalone manually added CGLIB dependency is needed for this example.
If you switch to JDK proxies, code that injects concrete implementations may no longer fit the proxy type.
Final methods are not universally “unadvisable”: the interface-proxy case is an important qualification.

### Public-method guidance and the actual visibility nuance
Use public methods on Spring beans as the safe application/service interception contract.
JDK proxies expose public interface methods.
CGLIB can intercept eligible protected and package-visible methods too, subject to visibility/overridability.
It cannot intercept private methods or override final methods.
Thus “Spring AOP only supports public methods” is a useful service-design guideline,
but too absolute as a statement about class-proxy technical capability.
The actual fundamental limit is method execution reached through a compatible Spring proxy.
All business/data operations demonstrated in this repository are public.

### Self-invocation: the most common trap
```java
public int outer() {
    return this.calculateMax();
}
```

If an external caller invokes `outer` on a proxy, the target's internal `this.calculateMax()`
does not return through the proxy and therefore does not trigger a fresh interceptor chain.
The annotation on `calculateMax` does not change that fact.

```text
external caller -> proxy -> target.outer()
                              |
                              +--> this.calculateMax()  [proxy bypass]
```

Prefer moving the intercepted operation to another injected bean:

```text
external caller -> outer-service proxy -> outer target
                                          |
                                          +--> injected calculator proxy -> calculator target
```

Self-injection or `AopContext.currentProxy()` can work in specific configurations,
but couple application code to proxy mechanics and can complicate lifecycle/testing.
`AopContext` requires exposed-proxy configuration and is not enabled by this module.
AspectJ weaving is another option when architectural refactoring is unsuitable.

### Advice ordering with @Order
Neither current aspect specifies ordering.
If behavior depends on nesting, order the aspect beans explicitly.
Smaller `@Order` values have higher precedence.
The higher-precedence aspect generally runs first on entry and last on exit.

Illustrative aspect declarations:

```java
@Aspect
@Component
@Order(1)
class AuthorizationAuditAspect {
}
```

```java
@Aspect
@Component
@Order(2)
class TimingAspect {
}
```

```text
Order 1 enters
  Order 2 enters
    target
  Order 2 exits
Order 1 exits
```

Put `@Order` on the aspect type rather than relying on it to order individual advice methods.
Do not confuse aspect precedence with `SecurityFilterChain` order or servlet filter order.
When infrastructure advisors participate, inspect their actual order instead of guessing.
A timer outside a transaction measures transaction overhead; one inside may measure only the inner work.
A retry outside a transaction can create one transaction per attempt; reversed nesting can be problematic.

### Around advice that preserves exceptional outcomes
The following is an **improvement example**, not code currently present in the repository:

```java
@Around("com.in28minutes.learnspringaop.aopexample.aspects.CommonPointcutConfig.trackTimeAnnotation()")
public Object timeEveryOutcome(ProceedingJoinPoint point) throws Throwable {
    long started = System.nanoTime();
    try {
        return point.proceed();
    } finally {
        long elapsedNanos = System.nanoTime() - started;
        logger.info("{} took {} ns", point.getSignature(), elapsedNanos);
    }
}
```

`finally` measures both success and failure paths.
The original result/throwable propagates unchanged if the logging itself does not fail.
For metrics, record low-cardinality tags rather than arguments or arbitrary exception messages.
An exception thrown by advice can still mask the target exception; observational advice should be robust.
Avoid swallowing exceptions and returning `null` merely to “keep the application running”.

### Real-world AOP uses
| Feature | Typical proxy behavior | Important boundary |
|---|---|---|
| `@Transactional` | Begin/join transaction, invoke, commit/rollback | External proxy call; configured rollback rules |
| `@Cacheable` | Return cached value or invoke and store | Key correctness; method might not execute |
| Method security | Authorize before/after execution | Service call through proxy |
| `@Async` | Dispatch work to executor | Thread/context and return-type semantics |
| Custom logging | Record selected operation metadata | Redaction and performance |
| Timing/metrics | Measure invocation duration/outcome | Meaningful span boundaries |

These features commonly use Spring AOP infrastructure, not necessarily user-written `@Aspect` classes.
HTTP authentication/authorization filters are not themselves “all AOP”; method security is proxy-based.
Transaction management also has an AspectJ mode, so proxy caveats describe the default common mode.
Caching can intentionally skip execution; a timer's location determines whether it measures hits or misses.
A proxy crossing is not a database transaction by itself; the corresponding advisor must be enabled/configured.

### Exception propagation and startup behavior
An after-throwing advice logs an escaping exception but does not consume it.
If `calculateMax()` throws during `CommandLineRunner.run`, the runner does not continue normally
into the second logged calculation.
Boot reports startup failure; the aspect is not an exception recovery framework.
If a data method throws and the business method does not catch it,
matching after-throwing advice can execute at both the data and business boundaries.
This can create duplicate-looking logs for one root cause.
Choose a consistent logging responsibility rather than logging every exception everywhere.

### Testing AOP behavior correctly
A unit test using `new` is valuable for the pure calculation logic.
It does not prove that proxy advice is active.
For AOP integration tests, obtain beans from a Spring application context.
Use an isolated test configuration where appropriate so the startup runner does not contaminate observations.

Conceptual assertions:

```java
assertThat(AopUtils.isAopProxy(businessService1)).isTrue();
assertThat(businessService1.calculateMax()).isEqualTo(55);
assertThat(businessService2.calculateMin()).isEqualTo(111);
```

`AopUtils` is a diagnostic helper from Spring AOP.
A proxy assertion alone does not prove that the intended pointcut/advice matched.
Capture logs or use a recording advice bean to assert which operations were observed.
Verify both included and excluded methods, as well as failure paths.
Avoid asserting exact timing durations or unspecified cross-aspect log ordering.

## Common pitfalls / gotchas
| Pitfall | Symptom | Explanation / remedy |
|---|---|---|
| Manually creating a service | No advice | Use a context-managed proxy for AOP tests/use |
| `@Aspect` without bean registration | No advice | Register with component/configuration or `@Bean` |
| Wrong component scan root | Aspect/target not found | Keep packages scanned or import intentionally |
| Calling through `this` | Annotated inner method not advised | Extract a collaborator or choose a deliberate alternative |
| Final class with class proxy | Proxy creation fails | Use an interface proxy or make design proxy-compatible |
| Final method with class proxy | Advice never runs for that method | Use overridable service methods |
| Private method annotation | No proxy interception | Move the boundary to an exposed operation |
| Missing runtime retention | Annotation pointcut misses | Use `RetentionPolicy.RUNTIME` |
| Broad execution wildcard | Unexpected logs or overhead | Narrow pointcuts and test architectural boundaries |
| Bean-name refactor | Before advice disappears | Name-based pointcut matching changed |
| Forgetting `proceed()` | Business method never runs | Preserve continuation unless intentionally short-circuiting |
| Calling `proceed()` twice | Duplicate work | Use exactly once for ordinary observational advice |
| Returning `null` from around advice | Wrong result/unboxing failure | Return the actual compatible result |
| Timer after `proceed()` only | No failure timing | Use `try/finally` |
| Wall-clock timer | Distorted/negative durations | Use a monotonic elapsed-time source |
| Unspecified aspect order | Inconsistent nesting expectations | Add explicit aspect precedence if needed |
| Treating `@After` as success | Misleading logs | Use after-returning for success |
| Logging raw arguments | Sensitive data leakage | Redact/whitelist values |
| Summing nested timing spans | Inflated total latency | Distinguish inclusive and exclusive duration |
| Blocking 30 ms in production | Artificial throughput loss | Sleep exists only as a timing demonstration |

### Repo-specific checks before claiming an AOP bug
1. Is the object injected by Spring rather than constructed directly?
2. Is the method called externally through its proxy?
3. Does the active named pointcut really match it?
4. Does the bean name contain the expected `Service` text?
5. Is `@TrackTime` present on the matched method?
6. Did the method throw before the current timer's logging code?
7. Are you expecting ordering that the source never declares?
8. Are you testing a command-line app as though it exposes HTTP endpoints?

`BusinessService2.calculateMin()` lacking a direct timing log is expected.
`DataService1.retrieveData()` lacking a direct timing log is also expected.
Both are still eligible for other matching logging advice.
An annotation is not a promise that every internal call will be intercepted.

## Interview questions & answers
### Basic
#### 1. What is AOP?
Aspect-oriented programming separates concerns that apply across many operations from core logic.
Spring AOP attaches advice to selected method executions through proxies.
The repo separates logging/timing from maximum/minimum calculations.

#### 2. What is a cross-cutting concern?
A responsibility such as logging, timing, authorization, or transaction management that spans many components.
Centralizing it can improve consistency and reduce duplication.
Essential domain rules should not be hidden as incidental logging-style advice.

#### 3. What is an aspect?
A module that contains cross-cutting behavior and matching metadata.
`LoggingAspect` is the example containing several advice methods.
It must be registered as a Spring bean for Spring AOP to use it.

#### 4. What is advice?
The behavior executed before, after, or around selected operations.
For example, `logMethodCallBeforeExecution` logs metadata before matching service calls.
The advice body is separate from the pointcut selecting calls.

#### 5. What is a pointcut?
A predicate selecting join points for advice.
The repo uses package executions, bean names, and `@TrackTime` annotations.
A named `@Pointcut` method has an intentionally empty body.

#### 6. What is a join point in Spring AOP?
A method execution reachable through a proxied Spring bean.
`BusinessService1.calculateMax()` is one example.
Field reads and constructors are not Spring AOP join points.

#### 7. What is a target versus a proxy?
The target contains the original business implementation.
The proxy is the object callers use to trigger interception before reaching that target.
Injected service references normally point to proxies when advice applies.

#### 8. What does `@Before` do?
It runs before the matched invocation and can inspect metadata/arguments.
It cannot substitute a result; throwing can prevent execution.
The repo uses it for bean-name-based logging.

#### 9. What is the difference between `@After` and `@AfterReturning`?
`@After` runs on normal and exceptional completion like finally.
`@AfterReturning` runs only on successful return and can observe a compatible result.
A log from `@After` is not proof that the method succeeded.

#### 10. What does `@AfterThrowing` do?
It observes compatible throwables escaping matched method execution.
It does not automatically recover from or swallow the exception.
A throwable caught inside the target is not an escaping exception at that boundary.

#### 11. What does `@Around` do?
It wraps the invocation and controls whether/how the remaining chain executes.
Use `ProceedingJoinPoint.proceed()` to continue, then return the compatible result.
It is the most powerful advice and should be used carefully.

#### 12. What are the default calculation results?
The first business service computes maximum `55` from `[11,22,33,44,55]`.
The second computes minimum `111` from `[111,222,333,444,555]`.
AOP observation does not change those values in the current code.

### Intermediate
#### 13. How does `execution(* package.Type.method(..))` work?
The first wildcard selects any return type; the declaring type and method name constrain the operation.
`(..)` means any number of arguments.
The repo uses wildcards for types/methods inside specific business/data packages.

#### 14. What does `bean(*Service*)` match?
Spring bean names or aliases containing the case-sensitive text `Service`.
It is instance/name-oriented rather than just class/package-oriented.
All four default service bean names in this module match.

#### 15. How does `within` differ from `execution`?
`within` selects by enclosing/declaring type pattern.
`execution` can constrain a full method signature, including name, return type, and parameters.
Both still operate within Spring's proxy method-execution model.

#### 16. Why does `TrackTime` need runtime retention?
Spring's runtime matching must be able to see the annotation.
`RetentionPolicy.RUNTIME` preserves it for reflection/interception.
The annotation has no standalone timing behavior without matching advice.

#### 17. Why reuse named pointcuts?
They centralize architecture-level selections and reduce duplicated expression strings.
The logging/timing aspects both refer to `CommonPointcutConfig`.
Update and test the shared selection when packages or naming conventions change.

#### 18. What is the difference between `JoinPoint` and `ProceedingJoinPoint`?
`JoinPoint` supplies signature, arguments, proxy, and target metadata.
`ProceedingJoinPoint` additionally controls continuation with `proceed()`.
Around advice uses the latter because it wraps execution.

#### 19. Why might the timer not log an exception path?
The current timer's stop/log statements are after `proceed()` without `finally`.
A propagated throwable skips those statements.
A monotonic timer in `try/finally` measures every completion path.

#### 20. Why use `System.nanoTime()` for duration?
It provides a monotonic elapsed-time source rather than wall-clock time.
Wall-clock corrections can distort differences computed with `currentTimeMillis()`.
Nanoseconds are a unit of measurement, not a guarantee of nanosecond clock resolution.

#### 21. How do JDK proxies differ from CGLIB?
JDK proxies expose interfaces; CGLIB proxies subclass target classes.
Class proxies cannot override final/private methods or subclass final classes.
Boot defaults to class proxies, and these service classes expose no interfaces.

#### 22. Does Spring AOP support only public methods?
Public service methods are the safest intended boundary, and JDK proxies expose public interface operations.
Eligible nonpublic overridable methods can also be intercepted through CGLIB subject to visibility rules.
Private/final class-proxy methods and self-invoked calls remain limitations.

#### 23. Why is no explicit `@EnableAspectJAutoProxy` present?
The AOP starter triggers Boot's conditional AOP auto-configuration in this setup.
That creates the proxy infrastructure.
Plain Spring without Boot generally needs explicit enabling/imported configuration.

#### 24. Are `@Transactional`, `@Cacheable`, and security examples of AOP?

Transaction management, caching, and method security commonly use Spring proxy advisors.
They need their respective infrastructure and compatible proxy boundaries.
HTTP security filters are a different mechanism, not simply method AOP.

### Advanced and scenario-based

#### 25. Scenario: a unit test sees correct results but no aspect logs. Why?

If the test constructs services with `new`, it bypasses Spring's proxy lifecycle.
Test pure logic that way, but use context-managed beans for interception tests.
Also verify pointcut matches instead of merely checking for any proxy.

#### 26. Scenario: `this.calculateMax()` ignores `@TrackTime`. Why?

The target's internal call never re-enters the proxy.
Extract the operation into an injected collaborator for a clean interception boundary.
Annotation presence alone cannot fix ordinary proxy self-invocation.

#### 27. Scenario: renaming a bean removes before logging. Why?

The active before advice uses `bean(*Service*)`, so the bean's name is part of selection.
An explicit name without `Service` no longer matches.
Use an annotation or stable type/package pointcut if renaming should not affect logging.

#### 28. Scenario: adding a deeper subpackage misses the broad package pointcut. Why?

The repo uses single-segment `*` package matching, not recursive `..` matching.
A new deeper package can lie outside its selection.
Choose and test an intentionally recursive boundary if that is desired.

#### 29. Scenario: failed startup prevents the second calculation. Is after-throwing broken?

No: the advice logs but leaves the throwable propagating.
A failure in the first `CommandLineRunner` calculation interrupts startup/normal control flow.
Recovery requires an explicit application policy, not just an observation annotation.

#### 30. How does ordering affect timing versus retries or transactions?

An outer timer includes all inner retries/transaction overhead; an inner timer measures narrower work.
Lower `@Order` values have higher precedence and generally wrap lower-precedence aspects.
Test actual advisor composition; reversed retry/transaction nesting can change correctness.

#### 31. Can around advice change parameters or results?

Yes, through replacement arguments to `proceed(args)` and a compatible returned value.
It can also short-circuit execution, as caching may do.
Those changes must be intentional because they can violate validation, idempotency, or caller contracts.

#### 32. Can after-returning advice replace the target's result?

No; returning a different value from the advice does not replace the caller's result.
It may observe or mutate a returned mutable object, which has separate correctness concerns.
Use around advice for intentional result substitution.

#### 33. Scenario: after-throwing logs twice for one failure. Why?

A data-method failure can propagate through the matching business method.
Each proxy boundary can observe the same escaping failure.
Define a logging boundary/correlation policy instead of treating duplicate logs as separate root causes.

#### 34. When would you choose AspectJ instead of Spring AOP?

When required interception covers self-invocation, constructors, fields, or non-Spring objects beyond proxies.
AspectJ weaving can instrument those execution points with additional tooling.
Prefer simpler Spring boundaries when they meet the requirement.

#### 35. Scenario: a final method needs advice. Is it impossible?

A CGLIB proxy cannot override and intercept it.
A JDK interface proxy can intercept an interface invocation even if the underlying implementation method is final.
Distinguish proxy mechanism and exposed operation before choosing a remedy.

#### 36. How would you validate this module's exact matching contract?

Obtain beans from a context and record advice events for both calculation paths.
Assert timing for `calculateMax` and `DataService2.retrieveData`, but not the other two operations.
Check failure semantics and results without asserting exact durations or undeclared relative ordering.

## Hands-on exercises

Use a separate practice copy or your own development workspace for source experiments.
These tasks extend the existing code; this guide does not apply those changes.

### Exercise 1 — Build an advice coverage matrix

Run:

```bash
cd 73-spring-aop
./gradlew bootRun
```

Before running, predict this coverage:

| Method | Before | After-finally | After-returning | Timing |
|---|---|---|---|---|
| `BusinessService1.calculateMax` | Yes | Yes | No | Yes |
| `BusinessService2.calculateMin` | Yes | Yes | No | No |
| `DataService1.retrieveData` | Yes | No | Yes | No |
| `DataService2.retrieveData` | Yes | No | Yes | Yes |

After-throwing is eligible for all four, but only runs for compatible escaping failures.
Compare the table with actual logs and confirm values `55` and `111`.
Do not interpret the observed cross-aspect order as a guaranteed contract.

### Exercise 2 — Make timing exception-safe

In a practice copy, change the performance timer to use `System.nanoTime()` and `try/finally`.
Temporarily activate the commented exception in `BusinessService1.calculateMax()`.
Observe that before/after/after-throwing advice can execute and timing now records failure too.
Verify that the original exception still propagates and startup fails rather than silently returning zero.
Restore the normal calculation afterward.
Add a test that records a duration event for both successful and failing invocations.

### Exercise 3 — Demonstrate self-invocation and fix the boundary

Add a public outer method on `BusinessService1` that internally calls `calculateMax()`.
Invoke it through the injected service and compare timing logs with direct external invocation of `calculateMax`.
Explain why the inner `@TrackTime` method does not create a fresh timing interception.
Move the timed operation to another injected bean and repeat the experiment.
Prefer that refactoring over hidden `AopContext` coupling.
Verify results remain unchanged through both designs.

### Exercise 4 — Compare selection strategies and ordering

Add a new annotated public method in a deeper business subpackage in your practice copy.
Compare the current single-segment `execution` pattern with a recursive `within` expression.
Rename one service bean explicitly and observe the effect on `bean(*Service*)`.
Give timing and logging aspects explicit `@Order` values, then reverse them.
Explain which work is included in a timer at each nesting position.
Add tests for inclusion/exclusion rather than relying on visual log inspection only.

### Exercise 5 — Write a focused AOP integration test

Create a test context containing just the services, pointcut helper references, and aspects needed for the scenario.
Avoid invoking the application runner unintentionally before assertions begin.
Retrieve services from the context rather than constructing them directly.
Check proxy status with `AopUtils`, preserve expected results, and record matched advice events.
Assert the exact annotation-based timing coverage and both normal/exceptional logging semantics.
Do not assert that a 30 ms sleep produces exactly 30 ms elapsed time.

## Quick revision cheat sheet

| Topic | Remember |
|---|---|
| Aspect | Cross-cutting module |
| Advice | What executes |
| Pointcut | Which executions are selected |
| Join point | Method execution in Spring AOP |
| Target | Original object |
| Proxy | Intercepting object injected into callers |
| `@Before` | Before invocation; can abort by throwing |
| `@After` | Finally-style completion, not necessarily success |
| `@AfterReturning` | Observe compatible normal result |
| `@AfterThrowing` | Observe compatible escaping throwable |
| `@Around` | Control continuation and outcome |
| `proceed()` | Continue the remaining chain; normally once |
| `execution` | Match method signature |
| `within` | Match declaring type/package |
| `bean` | Match Spring bean name; Spring-specific |
| `@annotation` | Match method annotation |
| `*` / `..` | Single wildcard segment / recursive package or arbitrary arguments by context |
| Named pointcut | Empty method declaring reusable selection |
| JDK proxy | Interface-based |
| CGLIB proxy | Class-subclass-based, requires overridability |
| Public methods | Safe service contract; CGLIB has qualified nonpublic support |
| Self-invocation | Bypasses ordinary proxy interception |
| AspectJ | Weaving with broader join points/object coverage |
| `@Order` | Smaller value enters first and normally exits last |
| Duration clock | `System.nanoTime()` for elapsed measurements |
| AOP test | Obtain context-managed proxy, then assert actual advice coverage |

Repository-specific memory hooks:

- One `CommandLineRunner`, two business services, two data services, and two aspects.
- Default calculations return `55` and `111`.
- Both data methods sleep for 30 ms as a demonstration, not real database latency.
- `TrackTime` is a runtime-retained method marker.
- Only `calculateMax` and `DataService2.retrieveData` are annotated.
- Before logging uses `bean(*Service*)`.
- After-finally logging uses the business-package pointcut.
- After-returning logging uses the data-package pointcut.
- After-throwing logging uses the broad business/data structure pattern and binds `Exception`.
- The current timer misses exceptional completion because its log is after `proceed()`.
- Neither aspect currently specifies `@Order`.
- The AOP starter enables Boot's normal proxy infrastructure; it does not activate full AspectJ weaving.
