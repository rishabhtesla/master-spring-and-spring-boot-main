# 10 — Build Tools: Maven and Gradle

## What this module teaches

A build tool turns source code and dependency declarations into tested artifacts.
This chapter explains how the course expresses the same Spring stack in two tools.
The source baseline is **Spring Boot 3.3.4 and Java 21**.
Build-tool compatibility must also be checked: old wrappers are not automatically updated.

### Exact module paths and how to run

| Path | Purpose | Build descriptor |
|---|---|---|
| `81-maven/` | Maven dependencies, POM, Boot packaging | `pom.xml` |
| `82-gradle/` | Gradle plugins, tasks, configurations | `build.gradle`, `settings.gradle` |
| `71-spring-security/` | Security/JDBC/resource-server dependencies | `build.gradle` |
| `72-oauth/` | OAuth client dependencies | `build.gradle` |
| `73-spring-aop/` | AOP starter dependency | `build.gradle` |

From the repository root:

```bash
mvn -f 81-maven/pom.xml test
mvn -f 81-maven/pom.xml package
mvn -f 81-maven/pom.xml spring-boot:run

# Stop the running app before starting another module on the same port.
java -jar 81-maven/target/learn-maven-0.0.1-SNAPSHOT.jar
```

The extracted `81-maven/` directory does **not** contain `mvnw`.
`81-maven/Step10.zip` contains archived `mvnw` and `mvnw.cmd` scripts.
Use installed Maven for the current extracted project.
Do not pretend an archived script is an executable file already in the module.

`82-gradle/gradlew` exists, but two limitations matter:

1. Its properties pin **Gradle 7.5.1**, which cannot run on Java 21.
2. The extracted folder has no `gradle/wrapper/gradle-wrapper.jar`.

Boot 3.3 requires supported Gradle 7.5+ or 8.x versions, but Java 21 runtime support
requires **Gradle 8.5+**; select a supported 8.x release for this practice baseline.
The Java 21 toolchain support also needs a sufficiently recent Gradle version.
A practical route for the unchanged extracted project is installed Gradle 8.5+:

```bash
gradle -p 82-gradle helloWorld
gradle -p 82-gradle test
gradle -p 82-gradle bootJar
gradle -p 82-gradle bootRun
```

Wrapper repair or generation is an exercise for a separate practice copy.
These commands are study instructions, not build executions performed by this chapter.

## Project walkthrough

This section quotes actual source/build snippets.
Examples labeled illustrative later are proposed patterns, not current declarations.

### 1. Maven POM identity and inheritance

Path: `81-maven/pom.xml`.

```xml
<modelVersion>4.0.0</modelVersion>

<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.4</version>
    <relativePath /> <!-- lookup parent from repository -->
</parent>

<groupId>com.in28minutes</groupId>
<artifactId>learn-maven</artifactId>
<version>0.0.1-SNAPSHOT</version>
```

`modelVersion` is the POM model format, not the installed Maven version.
The project's coordinates are `com.in28minutes:learn-maven:0.0.1-SNAPSHOT`.
The parent supplies dependency management and useful plugin/default configuration.
An empty `relativePath` disables Maven's usual local parent-file lookup.
The project then resolves this parent through configured repositories.

### 2. Java version and actual dependencies

Same path: `81-maven/pom.xml`.

```xml
<properties>
    <java.version>21</java.version>
</properties>
```

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.amqp</groupId>
    <artifactId>spring-amqp</artifactId>
</dependency>
```

The file also declares `spring-boot-starter` explicitly.
The web starter brings in the base starter transitively, so this declaration overlaps.
That overlap does not mean duplicate versions are loaded as separate dependencies.
`spring-amqp` is a library dependency, not a declaration of a running RabbitMQ broker.
Dependency management supplies omitted versions.
The test starter has `<scope>test</scope>`.
Test-scoped dependencies are not part of the production runtime classpath.

### 3. Packaging plugin

Same path: `81-maven/pom.xml`.

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

The Boot parent supplies management for this plugin.
Declaring the plugin activates Boot packaging behavior, including managed repackage setup.
An executable Boot JAR contains application classes and nested dependency JARs.
A plain Java library JAR usually does not bundle all dependencies.
`java -jar` needs an appropriate manifest and launcher layout.

### 4. Dependency repositories versus plugin repositories

Same path: `81-maven/pom.xml`.

```xml
<repositories>
    <repository>
        <id>spring-milestones</id>
        <name>Spring Milestones</name>
        <url>https://repo.spring.io/milestone</url>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </repository>
</repositories>
```

The POM separately declares the same URL under `<pluginRepositories>`.
Dependencies and build plugins have separate repository configuration concepts.
Central remains available through Maven's inherited defaults.
The milestone repository is an additional repository, not a replacement for Central.
Its snapshot policy disables snapshot retrieval from that declaration.
Most stable Boot artifacts are available from Maven Central.

### 5. Application and test sources

Path: `81-maven/src/main/java/com/in28minutes/learnmaven/LearnMavenApplication.java`.

```java
@SpringBootApplication
public class LearnMavenApplication {

    public static void main(String[] args) {
        SpringApplication.run(LearnMavenApplication.class, args);
    }

}
```

This is a normal Boot application entry point; Maven does not replace Spring startup.
The project has no custom controller in this class.
Starting the web starter does not guarantee a useful endpoint exists at `/`.
`src/main/resources/application.properties` is empty in this module.
`src/test/java/com/in28minutes/learnmaven/LearnMavenApplicationTests.java`
contains a `@SpringBootTest` with an empty `contextLoads()` method.
It verifies context startup rather than Maven's dependency mediation algorithm.

### 6. Maven notes and archived wrapper

Paths: `81-maven/HELP.md` and `81-maven/Stepzz-final-code.md`.
The HELP file explains that a hyphenated Java package was normalized to `learnmaven`.
Its links refer to older Boot 3.0.0-RC1 documentation.
The live POM, not an old help URL, defines the actual Boot 3.3.4 baseline.
The final-code note reproduces the POM, entry point, empty properties, and context test.
Its historical directory comment says `21-maven`; the current module is `81-maven/`.

Archive member: `81-maven/Step10.zip` → `mvnw`.
A real excerpt of the archived launcher is:

```sh
WRAPPER_LAUNCHER=org.apache.maven.wrapper.MavenWrapperMain
```

The archived script refers to `.mvn/wrapper/maven-wrapper.jar`.
It contains a fallback download for Maven Wrapper 3.1.0 bootstrap code.
That wrapper-library version is **not** the Maven distribution version.
A complete wrapper setup also needs its configuration and supported distribution URL.
Do not copy only a script and assume that the distribution is pinned correctly.

### 7. Gradle plugin block and Java toolchain

Path: `82-gradle/build.gradle`.

```groovy
plugins {
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.6'
    id 'java'
}

group = 'com.in28minutes'
version = '0.0.1-SNAPSHOT'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}
```

The Java plugin adds compilation, test, and JAR tasks.
The Boot plugin adds Boot-specific tasks such as `bootJar` and `bootRun`.
The dependency-management plugin integrates Spring's managed dependency versions.
When used with the Boot plugin, it imports the matching Boot dependency BOM.
The toolchain describes the JDK needed for Java tasks.
It does not automatically make an old Gradle runtime support Java 21.
The JVM running Gradle and the JDK compiling the application are distinct concerns.

### 8. Gradle repositories and configurations

Same path: `82-gradle/build.gradle`.

```groovy
repositories {
    mavenCentral()
    maven { url 'https://repo.spring.io/milestone' }
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework:spring-core'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

`implementation` expresses an internal production dependency.
`spring-core` is also normally reached through Spring starters.
Its explicit declaration demonstrates syntax; it does not remove transitivity.
Managed versions explain why coordinates omit the version segment here.
`testImplementation` is for test compilation and runtime, not application packaging.

### 9. JUnit Platform and a custom task

Same path: `82-gradle/build.gradle`.

```groovy
tasks.named('test') {
    useJUnitPlatform()
}

tasks.register('helloWorld'){
    doLast {
        System.out.println("Hello World");
    }
}
```

`useJUnitPlatform()` enables Jupiter-engine execution in the Gradle test task.
`tasks.named` configures an existing task lazily.
`tasks.register` creates a task lazily rather than eagerly realizing it.
`doLast` executes when the task runs, not merely when the build script is evaluated.
`helloWorld` prints a message and does not compile application sources by itself.
A task graph links work through declared dependencies, not file order in the script.

### 10. Gradle settings and wrapper

Path: `82-gradle/settings.gradle`.

```groovy
pluginManagement {
    repositories {
        maven { url 'https://repo.spring.io/milestone' }
        gradlePluginPortal()
    }
}
rootProject.name = 'learn-gradle'
```

Plugin resolution here is distinct from dependency repositories in `build.gradle`.
The root project name contributes to the default archive name.
No `include` declarations make this a multi-project build in the live settings file.

Path: `82-gradle/gradle/wrapper/gradle-wrapper.properties`.

```properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-7.5.1-bin.zip
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

Path: `82-gradle/gradlew`.
The generated POSIX launcher resolves its project home and launches wrapper Java code.
The script is not itself the Gradle distribution.
The live wrapper JAR is absent; `82-gradle/learn-gradle.zip` contains an archived JAR.
The distribution URL still identifies an old runtime incompatible with Java 21.
These are setup limitations, not failures in the Java application algorithm.

### 11. Gradle notes are not a second authoritative build

Paths: `82-gradle/HELP.md` and `82-gradle/Stepzz-final-code.md`.
The HELP file includes old Boot RC documentation links.
The large final-code note mostly contains generated test-report CSS/HTML/JavaScript,
compiled-data dumps, a manifest, and resource snapshots.
Generated output is evidence of a past run, not a recommended source of build configuration.
Read `build.gradle`, `settings.gradle`, and wrapper properties for current configuration.
Do not copy `build/` report files into a production build definition.

### 12. Related Gradle modules: actual differences

Path: `71-spring-security/build.gradle`.

```groovy
implementation 'org.springframework.boot:spring-boot-starter-security'
implementation 'org.springframework.boot:spring-boot-starter-web'
implementation 'org.springframework.boot:spring-boot-devtools'
implementation 'org.springframework.boot:spring-boot-starter-jdbc'
implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
implementation 'com.h2database:h2'
testImplementation 'org.springframework.boot:spring-boot-starter-test'
testImplementation 'org.springframework.security:spring-security-test'
```

Notice devtools is declared as `implementation` in this file.
That differs from the more purpose-specific configuration in the OAuth module.
Boot packaging has its own handling for development dependencies; inspect the resulting artifact.
Do not claim the declaration is already `developmentOnly` here.

Path: `72-oauth/build.gradle`.

```groovy
implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'
implementation 'org.springframework.boot:spring-boot-starter-web'
developmentOnly 'org.springframework.boot:spring-boot-devtools'
testImplementation 'org.springframework.boot:spring-boot-starter-test'
```

`developmentOnly` is a Boot-oriented configuration for local development support.
OAuth **client** and OAuth **resource server** starters serve different application roles.
The build file makes that distinction visible before reading security configuration.

Path: `73-spring-aop/build.gradle`.

```groovy
implementation 'org.springframework.boot:spring-boot-starter'
implementation 'org.springframework.boot:spring-boot-starter-aop'
testImplementation 'org.springframework.boot:spring-boot-starter-test'
```

All three related build files use Boot 3.3.4, dependency management 1.1.6,
a Java 21 toolchain, Maven Central/milestones, and `useJUnitPlatform()`.
Do not assume that matching plugin versions imply matching wrapper versions.

## Core concepts explained

### What build tools do

```text
Source + resources + dependency metadata + build configuration
                             |
                  resolve -> compile -> test
                             |
                 package -> verify -> publish
                             |
                 JAR / WAR / reports / image
```

Build tools automate repeatable steps rather than merely downloading libraries.
They manage classpaths, compiler options, resource copying, tests, and packaging.
Plugins extend them with code generation, analysis, documentation, and publishing.
A reproducible build also needs pinned tools and controlled dependency inputs.
A successful local build is not proof that production configuration is valid.

### Maven POM anatomy

| Element | Meaning | Common misconception |
|---|---|---|
| `modelVersion` | POM schema/model version | Not Maven executable version |
| `parent` | Inherit configuration and management | Not always the aggregator |
| `groupId` | Organization/namespace identity | Not necessarily a Java package |
| `artifactId` | Artifact/module name | Not the main class |
| `version` | Published artifact version | Not the Java language version |
| `packaging` | Artifact type, default `jar` | `pom` often describes an aggregator |
| `properties` | Reusable values | Does not itself install JDKs |
| `dependencies` | Dependencies actually requested | Not just version constraints |
| `dependencyManagement` | Managed defaults | Does not automatically add libraries |
| `build/plugins` | Active build plugins | Distinct from application dependencies |
| `build/pluginManagement` | Plugin defaults | Generally needs activation by lifecycle/declaration |
| `profiles` | Conditional configuration | Not Spring runtime profiles |
| `modules` | Reactor children | Not application package scanning |

### Coordinates and versions

The common coordinate form is `groupId:artifactId:version`.
Full artifact identity also involves type/extension and sometimes a classifier.
A `sources` classifier refers to source artifacts, not a different library version.
Maven repository paths follow group segments and artifact/version directories.

```text
com.in28minutes:learn-maven:0.0.1-SNAPSHOT
       namespace    name       evolving version
```

A release version should identify an immutable published artifact.
A SNAPSHOT is an evolving development version resolved using snapshot metadata.
Remote snapshots may have timestamped filenames while retaining a SNAPSHOT coordinate.
`-U` requests checks for updated snapshots and missing releases.
It is not a universal instruction to arbitrarily upgrade every dependency.
Use releases, locking/pinning, and recorded provenance for dependable releases.

### Maven dependency scopes

| Scope | Main compile | Main runtime | Tests | Notes |
|---|---|---|---|---|
| `compile` | Yes | Yes | Yes | Default; typically transitive |
| `provided` | Yes | No | Yes | Runtime environment supplies it; not transitive |
| `runtime` | No | Yes | Yes | Example: driver needed only at runtime |
| `test` | No | No | Yes | Testing libraries; not transitive to consumers |
| `system` | Yes | No standard managed runtime resolution | Yes | Like provided but uses explicit local `systemPath`; avoid |
| `import` | Not a classpath scope | Not a classpath scope | Not a classpath scope | Imports a `pom` BOM into dependencyManagement |

`system` dependencies are local-machine-specific and undermine portability.
Imported BOM entries are management data, not JARs to execute.
Transitive scope propagation has rules; not every dependency keeps its original scope.
A runtime dependency can still appear on the test runtime classpath.
A provided servlet API is useful when an external container supplies that API.
An embedded Boot application has different packaging needs from a container-deployed WAR.

### Transitivity and Maven conflict resolution

```text
application
  +-- library-a -> common:1.0     (depth 2)
  +-- common:2.0                 (depth 1)

Selected common version: 2.0 (nearest declaration)
```

Maven normally selects the nearest version in the dependency graph.
If competing declarations are equally near, first declaration order decides.
An explicit direct dependency can override a transitive selection.
Dependency management can control transitive versions without adding a direct dependency.
An exclusion removes a dependency edge, not every possible path to the artifact.
Optional dependencies are not automatically inherited by consumers.
Always examine the effective graph rather than guessing from one starter declaration.

```bash
mvn -f 81-maven/pom.xml dependency:tree
mvn -f 81-maven/pom.xml dependency:tree -Dincludes=org.springframework:spring-core
mvn -f 81-maven/pom.xml dependency:tree -Dverbose
mvn -f 81-maven/pom.xml help:effective-pom
```

`dependency:tree` reports what was selected and where it came from.
`help:effective-pom` shows inherited and managed configuration.
Verbose output behavior depends on the dependency-plugin version.
If two paths introduce an unsafe version, fix management/exclusions deliberately.
Do not remove a required transitive library just to make the tree shorter.

### BOM, parent, and dependency management

A BOM is a POM containing coordinated dependency version recommendations.
Spring Boot tests a compatible library set and publishes `spring-boot-dependencies`.
The starter parent also supplies build defaults beyond that BOM.
Maven permits one parent, but multiple imported BOMs in dependency management.
Corporate inheritance can therefore use a corporate parent and import Boot's BOM.
Plugin management must then be considered separately.

Illustrative BOM import, not this module's current parent configuration:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.3.4</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

This import manages versions but adds no web, JPA, or test starter by itself.
Declare actual dependencies separately.
Overriding a managed version can be necessary but should be tested for compatibility.
Do not confuse a starter, which brings dependencies, with a BOM, which manages versions.

### Maven lifecycles, phases, and goals

Maven has separate **default**, **clean**, and **site** lifecycles.
The commonly remembered default lifecycle sequence is:

```text
validate -> compile -> test -> package -> verify -> install -> deploy
```

There are additional phases between these, such as resource processing and test compilation.
Running a later phase executes all earlier phases of that lifecycle.
`mvn package` therefore normally compiles and runs unit tests first.
`mvn clean package` invokes the clean lifecycle and then the default lifecycle.

| Phase | Typical result |
|---|---|
| `validate` | Project configuration checked |
| `compile` | Production sources compiled |
| `test` | Unit tests run |
| `package` | JAR/WAR produced |
| `verify` | Integration/quality checks bound before/at verification |
| `install` | Artifact installed into local repository |
| `deploy` | Artifact published to configured remote artifact repository |

**Deploy does not mean deploying the application to AWS.**
It is Maven artifact publication.
An integration-test class is not automatically run simply because `verify` is invoked.
Configure Failsafe and its integration-test/verify goals for that convention.
Surefire normally handles unit tests during `test`.

A goal is a plugin operation, commonly invoked as `pluginPrefix:goal`.
Examples: `dependency:tree`, `spring-boot:run`, `spring-boot:build-image`.
Goals can be bound to phases or invoked directly.
Direct goal invocation need not execute the whole default lifecycle.
`pluginManagement` records defaults; it is not the same as requesting every goal to execute.

### Local, Central, and remote repositories

```text
Project requests an artifact
            |
     local repository cache (~/.m2/repository)
            |
       configured remote repositories
        Central / company proxy / releases / snapshots
```

The local repository caches downloads and stores artifacts from `install`.
Maven Central is a public remote repository, not the local cache.
A company repository manager can proxy Central and host internal artifacts.
Use `settings.xml` for credentials, mirrors, and environment-specific repository settings.
`distributionManagement` identifies publication destinations for Maven deploy.
Do not place passwords in a shared POM.
Offline mode works only if required artifacts/plugins are already cached.
A corrupted cache entry is different from a missing repository credential.

### Multi-module builds

Illustrative aggregator, not an existing root reactor in these course folders:

```xml
<packaging>pom</packaging>
<modules>
    <module>domain</module>
    <module>service</module>
    <module>web</module>
</modules>
```

The reactor builds modules in dependency-aware order.
An aggregator lists children; a parent provides inherited configuration.
One POM can play both roles, but they are distinct relationships.
The course's numbered folders should not be assumed to form one reactor.
Use `-pl` to select modules and `-am` to also build required reactor dependencies.
Use `-rf` to resume from a failed module when appropriate.

### Maven wrapper and profiles

The wrapper pins a Maven distribution for team/CI consistency.
It still requires a compatible JVM and downloadable/cached distribution artifacts.
Verify distribution checksums where supported; review wrapper upgrades intentionally.
Common commands in a **complete** wrapper-enabled project are `./mvnw test` and `./mvnw package`.
They are not runnable from the current extracted `81-maven/` folder as-is.

A Maven profile changes build configuration using explicit or conditional activation.
Examples include extra checks, alternate repositories, or packaging behavior.
`mvn -Pquality verify` activates a profile named `quality` if it is declared.
This guide's module does not currently declare that profile.
`help:active-profiles` shows effective activation.
Spring profiles select application runtime beans/configuration instead.
Do not bake environment secrets into differently built production artifacts.
Prefer one artifact promoted between environments with external runtime configuration.

### Gradle DSL, tasks, and configurations

Gradle supports Groovy (`build.gradle`) and Kotlin (`build.gradle.kts`) DSLs.
Both configure the same underlying Gradle model.
Kotlin DSL gives stronger compile-time typing and IDE completion.
Groovy is concise and dynamic; this repository uses it.
Do not mix the two syntaxes verbatim.

| Concept | Meaning | Example |
|---|---|---|
| Plugin | Adds/configures build capabilities | `java`, Spring Boot |
| Task | A unit of work | `compileJava`, `test`, `bootJar` |
| Task dependency | Scheduling relation | Test needs compiled test classes |
| Configuration | Dependency bucket or resolvable/consumable view | `implementation`, `runtimeClasspath` |
| Settings | Build identity and included projects | `rootProject.name` |

```text
bootJar -> classes -> compileJava + processResources
 test   -> testClasses -> compileTestJava + processTestResources
 build  -> assemble + check
```

The actual graph depends on applied plugins and configuration.
`build` is a lifecycle task, not a compiler itself.
`bootRun` runs the application and is different from producing `bootJar`.
Configuration happens before task execution; `doLast` is execution-time work.
Keep expensive I/O out of configuration where possible.

### Gradle dependency configurations

| Configuration | Consumer-facing purpose | Typical use |
|---|---|---|
| `implementation` | Internal compile/runtime dependency; hidden from consumer compile API | Service implementation library |
| `api` | Exposed dependency on consumer compile classpath | Public library signatures |
| `compileOnly` | Compile-time only | Externally supplied API |
| `runtimeOnly` | Runtime only | JDBC driver |
| `testImplementation` | Test compile/runtime | Jupiter, Mockito |
| `testRuntimeOnly` | Test runtime only | Additional engine/launcher |
| `developmentOnly` | Boot development support | Devtools |

`api` is supplied by `java-library`, not the plain Java plugin alone.
If a public method returns another library's type, consumers may need that library as `api`.
`implementation` reduces accidental dependency exposure and recompilation pressure.
Configurations are not exactly one-to-one equivalents of Maven scopes.
Gradle normally resolves conflicting versions to the highest candidate version,
subject to constraints, platforms, strict versions, and other resolution rules.
That is different from Maven's default nearest-wins strategy.
Gradle `platform(...)` supports BOM-based alignment without the Spring management plugin.

### Incremental builds, caching, and the daemon

An up-to-date task can be skipped when inputs/outputs are unchanged locally.
Incremental compilation recompiles only affected work where supported.
The build cache reuses task outputs by cache key, potentially across workspaces.
A configuration cache reuses configured task-graph state; it is a different mechanism.
Correct inputs/outputs are prerequisites for trustworthy cache behavior.
A custom task that reads undeclared external state can make cached output wrong.

```text
Unchanged inputs + existing matching outputs -> UP-TO-DATE
Changed workspace + reusable cached task key -> FROM-CACHE
No reusable output                         -> execute task
```

The Gradle daemon reuses a warm JVM across compatible builds.
It reduces startup overhead but does not replace task caching.
Different JVM/options/version requirements can produce multiple daemon processes.
`gradle --stop` stops daemons for that Gradle version.
`--no-daemon` can still require a single-use daemon for JVM compatibility.
Use CI isolation and cache policies deliberately, not blanket cache deletion.

### Maven versus Gradle

| Dimension | Maven | Gradle |
|---|---|---|
| Build language | Declarative XML POM | Groovy or Kotlin DSL |
| Main model | Lifecycle phases with plugin goals | Task graph and plugins |
| Default layout | Strong conventions | Conventions through plugins |
| Dependency conflicts | Nearest wins, with management rules | Highest candidate by default, with constraints |
| Custom logic | Usually plugin/configuration | DSL/tasks/plugins |
| Performance features | Incremental support varies by plugins | Extensive incremental/cache/daemon support |
| Multi-module | Reactor | Multi-project builds |
| Version alignment | dependencyManagement/BOM | Platforms or management plugin |
| Wrapper | Maven wrapper | Gradle wrapper |
| Learning trade-off | Predictable conventional builds | Flexible but more build-programming concepts |

Neither tool inherently makes application code faster at runtime.
Choose based on team expertise, project scale, plugins, and maintainability.
A poorly written Gradle script can be slower and less reliable than a simple Maven build.
A Maven project can still support sophisticated automation through plugins.

## Common pitfalls / gotchas

| Pitfall | Symptom | Diagnosis / correction |
|---|---|---|
| Old Gradle runtime + JDK 21 | Startup/plugin/toolchain errors | Use supported Gradle 8.x, at least 8.5 for runtime |
| Missing wrapper JAR | Wrapper main class cannot be loaded | Complete wrapper setup in a practice copy |
| `./mvnw` in extracted Maven folder | File not found | Use installed Maven |
| POM modelVersion confused with Maven | Wrong version reasoning | Inspect executable `mvn -version` |
| Managed dependency mistaken for included dependency | Missing imports/classes | Add actual dependency |
| Adding a library installs its server | Broker/database connection fails | Provision external infrastructure separately |
| `install` mistaken for OS installation | Expect app/service to run | It installs artifacts into Maven local repo |
| `deploy` mistaken for cloud rollout | No Beanstalk deployment occurs | Use deployment tooling separately |
| `package` expected to run all integration tests | Integration bugs missed | Bind Failsafe and run `verify` |
| Skipped tests called tested build | False release confidence | Preserve CI verification gates |
| `api` on plain Java plugin | Unknown configuration | Apply `java-library` where appropriate |
| Wrong Gradle test engine setup | Jupiter tests not executed | Use JUnit Platform and required engines |
| Java toolchain mistaken for Gradle JVM | Wrapper startup still fails | Check both versions separately |
| Old HELP links treated as baseline | Wrong Boot API advice | Read live descriptors |
| Repeated clean builds everywhere | Cache benefits lost | Clean only when necessary |
| Secrets in build profiles | Credentials leak | Use secure settings/CI secret injection |

`-DskipTests` skips execution but ordinarily still compiles tests.
`-Dmaven.test.skip=true` also skips test compilation in standard supported plugins.
These flags are troubleshooting conveniences, not recommended release defaults.
Gradle `-x test` excludes the test task from the requested graph.
A build that packages successfully may still depend on unvalidated runtime settings.

## Interview questions & answers

### Basic

#### 1. What does a build tool do?
It resolves dependencies and automates compilation, testing, packaging, and publication.
It makes a shared build procedure executable rather than a checklist of manual steps.

#### 2. What is a POM?
Maven's Project Object Model expressed in XML.
It declares project identity, dependencies, inheritance, plugins, and related build settings.

#### 3. What are this Maven module's coordinates?
`com.in28minutes:learn-maven:0.0.1-SNAPSHOT`.
The artifact name and Java package do not need identical spelling.

#### 4. What does `modelVersion` 4.0.0 mean?
It identifies the POM model format.
It does not imply the build is using Maven 4.

#### 5. SNAPSHOT versus release?
A SNAPSHOT is an evolving development artifact; a release should be immutable.
Snapshots trade reproducibility for convenient development updates.

#### 6. What is a transitive dependency?
A dependency introduced because a directly requested library depends on it.
The web starter brings many Spring and server libraries without explicit declarations.

#### 7. What is Maven's default dependency scope?
`compile`, usable for main compilation/runtime and tests.
Use narrower scopes when the dependency is not needed everywhere.

#### 8. What does `mvn package` do?
Runs preceding default-lifecycle phases, normally including compilation and unit tests, then packages.
The packaging type and bound plugins determine the resulting artifact.

#### 9. What is a Gradle task?
A unit of build work, such as compilation, testing, or printing `helloWorld` output.
Tasks execute according to their dependency graph.

#### 10. Why use wrappers?
They make the build-tool distribution version explicit and consistent across machines.
They still require a complete wrapper setup and a compatible JDK.

### Intermediate

#### 11. What does Maven nearest-wins mean?
When versions conflict, the dependency declared at the shortest graph distance normally wins.
At equal depth, declaration order breaks ties unless management changes selection.

#### 12. How does Gradle's default conflict selection differ?
Gradle normally selects the highest candidate version rather than the nearest path.
Constraints, platforms, and custom rules can alter that result.

#### 13. Why can dependencies omit versions in these projects?
Boot's parent/BOM or Gradle dependency management supplies compatible managed versions.
Omitting an unmanaged version would not be valid just because it is a Spring application.

#### 14. Does dependencyManagement add a dependency?
No; it supplies defaults such as versions for dependencies requested elsewhere.
The same principle applies to importing a BOM.

#### 15. Parent versus BOM?
A parent can supply dependency and build/plugin defaults through inheritance.
A BOM primarily supplies coordinated dependency management and can be imported without using that parent.

#### 16. Goal versus phase?
A goal is a plugin operation; a phase is a lifecycle position to which goals can be bound.
`dependency:tree` is a goal, while `package` is a phase.

#### 17. `install` versus `deploy`?
Install writes the artifact to the local Maven repository; deploy publishes it remotely.
Neither command inherently runs the application on a cloud server.

#### 18. `provided` versus `runtime`?
Provided is needed to compile but supplied by the runtime environment.
Runtime is not needed for main compilation but must be available when the app executes.

#### 19. What does `implementation` hide?
It does not expose the dependency as part of a library consumer's compile API.
Consumers still obtain required runtime dependencies through published metadata.

#### 20. When use `api`?
When the dependency's types are exposed in a library's public API.
The `java-library` plugin supplies that configuration.

#### 21. Why call `useJUnitPlatform()`?
It tells Gradle's test task to use the JUnit Platform for Jupiter and other installed engines.
Dependencies alone do not express every test-runner setting.

#### 22. What is the daemon's benefit?
A reused warm JVM reduces repeated build startup overhead.
It is different from caching compiled/task outputs.

### Advanced and scenario-based

#### 23. The repo says Java 21 but `gradlew` fails before compiling. Why?
The wrapper pins Gradle 7.5.1 and its extracted wrapper JAR is missing.
Use a complete supported Gradle 8.x setup; Java source settings do not fix launcher compatibility.

#### 24. A corporate parent is mandatory. How do you retain Boot version alignment?
Import `spring-boot-dependencies` as a BOM in dependencyManagement.
Configure Boot/plugin/compiler defaults separately instead of assuming BOM import supplies them.

#### 25. `mvn verify` passed, but no integration tests ran. What is missing?
The appropriate integration-test plugin execution and discovery convention may not be configured.
Bind Failsafe's integration-test and verify goals and inspect reports.

#### 26. Two versions of spring-core appear in the tree. What should you do?
Check which version was selected, its paths, and the effective dependency management.
Align deliberately through the Boot BOM or a tested explicit override, not random exclusions.

#### 27. A Gradle task reports FROM-CACHE. How is that different from UP-TO-DATE?
FROM-CACHE restores reusable outputs for a matching task key.
UP-TO-DATE skips work because matching outputs already exist for unchanged local inputs.

#### 28. How can a custom task break cache correctness?
By reading external files/environment or generating outputs without declaring inputs/outputs properly.
Model its inputs and outputs before marking its work cacheable.

#### 29. A library compiles but its consumer cannot compile a public return type. Why?
The returned type's dependency may be incorrectly declared as implementation instead of api.
Expose required public-API dependencies through the library's published compile metadata.

#### 30. How do you build only a Maven module and its prerequisites?
Use `-pl module-name -am` from the aggregator build.
This applies to a declared reactor, not arbitrary sibling course folders.

#### 31. Maven profile versus Spring profile?
Maven profiles alter build configuration; Spring profiles select runtime application configuration/beans.
Prefer one release artifact with environment-specific runtime configuration.

#### 32. Would you clean on every CI step to guarantee correctness?
Not automatically: it sacrifices useful caching and can mask poor task modeling.
Use isolated workspaces, declared inputs, and targeted cleaning when stale-output evidence warrants it.

#### 33. Why avoid `systemPath` dependencies?
They bind the build to a machine-specific local file and bypass normal repository resolution.
Publish internal artifacts to a controlled repository instead.

#### 34. How do you investigate builds that work locally but fail offline in CI?
Check cached dependencies/plugins, repositories, credentials, tool/JDK versions, and snapshot state.
A local cache can conceal undeclared publication or remote-resolution assumptions.

## Hands-on exercises

### Exercise 1 — Explain the effective Maven project

Run `dependency:tree` and `help:effective-pom` for `81-maven/`.
Locate the managed version of `spring-amqp` and the web starter's transitive libraries.
Explain why explicit base starter and web starter declarations overlap.
Success: distinguish requested dependencies from managed version entries.

### Exercise 2 — Trace the lifecycle

In a practice copy, compare logs and outputs for `test`, `package`, and `install`.
Identify compilation, test reports, the executable JAR, and local-repository publication.
Explain what `deploy` would require without publishing anything unintentionally.
Success: describe phases without confusing artifact publication with AWS deployment.

### Exercise 3 — Complete a compatible Gradle wrapper

In a separate practice copy, use supported installed Gradle 8.x to generate the wrapper.
Pin a Java-21-compatible distribution and configure checksum verification where appropriate.
Run `helloWorld`, `test`, and `bootJar`; record the actual tool/JDK versions.
Success: wrapper bootstrap and compilation both work, with no reliance on an archived-only file.

### Exercise 4 — Design a small multi-module library

Create a practice domain library and web app with an aggregator or multi-project settings.
Expose a type from a dependency and compare Gradle `api` with `implementation` behavior.
Explain parent inheritance versus reactor aggregation in the Maven variant.
Success: consumers compile without accidental transitive API assumptions.

### Exercise 5 — Observe incremental execution

Run a compatible Gradle build twice and inspect UP-TO-DATE tasks.
Change one Java file and observe which tasks execute.
Repeat in another workspace with build caching configured, if appropriate.
Success: explain daemon, incremental compilation, build cache, and configuration cache separately.

## Quick revision cheat sheet / commands

Commands below are from the repository root unless otherwise noted.
Use installed tools for the extracted modules with incomplete/incompatible wrappers.

```bash
# Versions and Java runtime
java -version
mvn -version
gradle -version

# Maven verification and packaging
mvn -f 81-maven/pom.xml validate
mvn -f 81-maven/pom.xml test
mvn -f 81-maven/pom.xml verify
mvn -f 81-maven/pom.xml package
mvn -f 81-maven/pom.xml spring-boot:run

# Maven diagnostics
mvn -f 81-maven/pom.xml dependency:tree
mvn -f 81-maven/pom.xml help:effective-pom
mvn -f 81-maven/pom.xml help:active-profiles
mvn -f 81-maven/pom.xml -U test
mvn -f 81-maven/pom.xml -o test

# Supported installed Gradle 8.x, Java 21
gradle -p 82-gradle tasks --all
gradle -p 82-gradle helloWorld
gradle -p 82-gradle test
gradle -p 82-gradle bootJar
gradle -p 82-gradle bootRun
gradle -p 82-gradle dependencies --configuration runtimeClasspath
gradle -p 82-gradle dependencyInsight --dependency spring-core --configuration runtimeClasspath
gradle -p 82-gradle build --build-cache
gradle -p 82-gradle build --dry-run
gradle -p 82-gradle --status
```

Generic commands for **other complete wrapper-enabled practice projects**:

```bash
./mvnw verify
./gradlew build
./gradlew test --tests 'com.example.SomeTest'
./gradlew bootBuildImage
```

Publication and profile examples require configured destinations/profiles:

```bash
# Generic Maven reactor examples, not this repo's numbered folders
mvn -pl web -am verify
mvn -rf :web verify
mvn -Pquality verify
mvn install
mvn deploy
```

| Memory hook | Meaning |
|---|---|
| GAV | groupId, artifactId, version |
| SNAPSHOT | Mutable development line |
| BOM | Coordinated version management |
| Starter | Actual related dependencies |
| Maven conflict | Nearest wins by default |
| Gradle conflict | Highest candidate by default |
| Phase | Lifecycle position |
| Goal | Plugin operation |
| Install | Local repository publication |
| Deploy | Remote artifact publication |
| `api` | Public consumer compile dependency |
| Toolchain | Java tasks' JDK selection |
| Wrapper | Build distribution selection |
| Daemon | Warm reused Gradle JVM |
| UP-TO-DATE | Existing outputs remain valid |
| FROM-CACHE | Reusable outputs restored |

Final memory hook: **the build descriptor is executable project policy, not just a library list**.
