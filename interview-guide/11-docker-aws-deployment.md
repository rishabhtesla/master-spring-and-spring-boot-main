# 11 — Docker and AWS Deployment for Spring Boot and React

## What this module teaches

Containers package an application with a repeatable runtime environment.
Cloud deployment adds networking, identity, storage, scaling, and operations.
This guide connects the course's Docker recipes and AWS examples to those concepts.
The Java projects use **Spring Boot 3.3.4 and Java 21**.
Deployment procedures below are study instructions, not actions executed by this guide.

### Exact module paths and how to run

| Path | Application | Local endpoint / dependency |
|---|---|---|
| `83-docker/hello-world-java/` | Simple web app | `/` on port 5000 |
| `91-aws/01-hello-world-java/` | Beanstalk hello-world candidate | `/` on port 5000 |
| `91-aws/02-rest-api-mysql/` | User/post REST API | MySQL plus secured API on 5000 |
| `91-aws/03-rest-api-full-stack-h2/` | Todo API and JWT demo | In-memory H2 on 5000 |
| `91-aws/04-frontend-react/todo-app/` | React todo client | Development server on 3000 |

Only `hello-world-java/` exists as an application directory under `83-docker/`.
There are **no standalone Dockerfile variants** in the inspected extracted folders.
The three Dockerfiles are code blocks inside `83-docker/hello-world-java/README.md`.
There is also no existing Compose file to run in these directories.
Copy a recipe into a `Dockerfile` only in a separate practice copy.
The current documentation task does not add one to the source modules.

From repository root, with installed Maven and JDK 21:

```bash
mvn -f 83-docker/hello-world-java/pom.xml spring-boot:run
# In another terminal:
curl http://localhost:5000/

# Alternative application; stop the first app before starting this one.
mvn -f 91-aws/01-hello-world-java/pom.xml spring-boot:run
```

Build and run an executable JAR:

```bash
mvn -f 91-aws/01-hello-world-java/pom.xml package
java -jar 91-aws/01-hello-world-java/target/01-hello-world-0.0.1-SNAPSHOT.jar
```

For the full-stack example:

```bash
mvn -f 91-aws/03-rest-api-full-stack-h2/pom.xml spring-boot:run

# Separate terminal, exact frontend folder
cd 91-aws/04-frontend-react/todo-app
npm ci
npm start
```

`npm ci` uses the existing lockfile; choose a supported Node/npm combination.
The actual Create React App server defaults to `http://localhost:3000`.
The higher-level frontend note's port 4200 and old paths are historical leftovers.
API requests require the authentication configured by the selected backend.
Do not run multiple backends on port 5000 at the same time.

## Project walkthrough

This section uses real repository excerpts.
Later recipes explicitly marked **illustrative** are not already present files.
Whitespace is normalized in short excerpts for readability.

### 1. Docker hello-world behavior

Path: `83-docker/hello-world-java/src/main/java/com/in28minutes/rest/webservices/restfulwebservices/HelloWorldController.java`.

```java
@GetMapping(path = "/")
public String helloWorld() {
    // Implemented using String Templates
    return """
            { message: Hello World Java v1 }
           """;
}
```

This is a Java text block, not Java 21's preview string-template feature.
The comment's terminology is informal.
The returned text is not strict JSON: its key and value are unquoted.
Test it as a string unless the controller contract is deliberately changed.
Its mapping is `/`, not `/hello-world`.

Path: `83-docker/hello-world-java/src/main/resources/application.properties`.

```properties
logging.level.org.springframework = debug
server.port = 5000
```

Port 5000 is the process's listening port.
Docker publication must forward traffic to that container-side port.
Debug framework logs are useful for learning but can be too noisy in production.
The POM's artifactId is `hello-world-java`, and Boot's Maven plugin is declared.
The project does not declare Actuator, so `/actuator/health` is not automatically available.

### 2. Dockerfile variant 1: package outside the image build

Source: `83-docker/hello-world-java/README.md`, “Dockerfile - 1”.

```dockerfile
FROM openjdk:21-jdk-slim
COPY target/*.jar app.jar
EXPOSE 5000
ENTRYPOINT ["java","-jar","/app.jar"]
```

This expects a JAR to already exist under `target/` in the build context.
The Maven build happens before `docker build`, not in this Dockerfile.
The image contains a JDK plus the application JAR.
The exec-form entrypoint starts Java as the container's main process.
The wildcard should match exactly the intended runnable JAR.
The historical `openjdk` tag should not be assumed available or maintained today.
Choose a supported Java 21 runtime image and verify the tag/platform before practicing.
Pin an appropriate immutable digest when release provenance requires it.

The README's real build command is:

```bash
docker build -t in28min/hello-world-docker:v1 .
```

It is runnable only after creating the recipe file in a practice copy.
The dot is the build context, not a shorthand for an arbitrary host filesystem.

### 3. Dockerfile variant 2: multi-stage build

Source: same README, “Dockerfile - 2”.

```dockerfile
FROM maven:3.9.6-amazoncorretto-21-al2023 AS build
WORKDIR /home/app
COPY . /home/app
RUN mvn -f /home/app/pom.xml clean package

FROM openjdk:21-jdk-slim
EXPOSE 5000
COPY --from=build /home/app/target/*.jar app.jar
ENTRYPOINT [ "sh", "-c", "java -jar /app.jar" ]
```

The first stage has Maven, a JDK, source, and build outputs.
Only the selected JAR is copied into the final stage.
Build tools and source do not need to remain in the runtime image.
This improves separation even though the final recipe still uses a JDK image.
The shell entrypoint differs from variant 1's direct Java process.
Prefer exec form, or `exec java ...` within a necessary shell, for signal delivery.
The build runs tests unless explicitly configured otherwise.
Use exact output paths when multiple JARs could match the wildcard.

### 4. Dockerfile variant 3: attempt to improve caching

Source: same README, “Dockerfile - 3”.

```dockerfile
FROM maven:3.9.6-amazoncorretto-21-al2023 AS build
WORKDIR /home/app

COPY ./pom.xml /home/app/pom.xml
COPY ./src/main/java/com/in28minutes/rest/webservices/restfulwebservices/RestfulWebServicesApplication.java /home/app/src/main/java/com/in28minutes/rest/webservices/restfulwebservices/RestfulWebServicesApplication.java

RUN mvn -f /home/app/pom.xml clean package

COPY . /home/app
RUN mvn -f /home/app/pom.xml clean package
```

Its final stage is the same pattern as variant 2.
The early minimal project build warms dependency/plugin downloads.
Later source edits can leave those earlier layers reusable.
Changing the POM or copied application entry point invalidates that warm-up layer.
Two `clean package` executions are a teaching approach, not the only optimal strategy.
A POM-first dependency warm-up and BuildKit Maven-cache mount are alternatives.
`dependency:go-offline` may not pre-resolve every plugin/profile-specific requirement.
Caching must never substitute for the final real build and tests.

### 5. AWS hello-world module

Paths: `91-aws/01-hello-world-java/pom.xml` and its `HelloWorldController.java`.
The artifactId is `01-hello-world`.
The controller has the same root mapping and text-block response as the Docker module.

Path: `91-aws/01-hello-world-java/src/main/resources/application.properties`.

```properties
logging.level.org.springframework = debug

#CHANGE
server.port=5000
```

Port 5000 aligns with common Beanstalk Java SE proxy examples.
Confirm the selected platform's process/proxy port rather than assuming every platform uses it.
`91-aws/01-hello-world-java/README.md` is empty in this checkout.
There is no hidden set of deployment instructions in that empty file.
The Beanstalk procedure later in this guide is explanatory, not copied existing automation.

### 6. MySQL REST module and local container note

Path: `91-aws/02-rest-api-mysql/readme.md`.
The note launches MySQL, documents mysqlsh queries, and shows `/jpa/users` requests.
A short real command excerpt is:

```text
--name mysql --publish 3306:3306 mysql:8-oracle
```

The complete note supplies `MYSQL_ROOT_PASSWORD`, `MYSQL_USER`, `MYSQL_PASSWORD`,
and `MYSQL_DATABASE` as container initialization variables.
Its passwords are classroom values, not production credentials.
Binding host port 3306 on every interface can expose a learning database unnecessarily.
Prefer localhost publication for a host-running API and a private network for container peers.

The POM includes JPA, web, security, Actuator, validation, and MySQL Connector/J.
Its MySQL driver coordinates are `com.mysql:mysql-connector-j`.
A dependency on the driver does not create a database server.

Path: `91-aws/02-rest-api-mysql/src/main/resources/application.properties`.

```properties
server.port=5000

logging.level.org.springframework=info
management.endpoints.web.exposure.include=*
spring.jpa.defer-datasource-initialization=true
spring.jpa.show-sql=true
```

```properties
spring.datasource.url=jdbc:mysql://${RDS_HOSTNAME:localhost}:${RDS_PORT:3306}/${RDS_DB_NAME:social-media-database}
spring.datasource.username=${RDS_USERNAME:social-media-user}
```

The password property also reads `RDS_PASSWORD`, with a classroom fallback.
The other placeholders default to a localhost learning database.
Missing cloud variables can therefore make the app connect to the wrong host/database.
Explicitly supply all five `RDS_*` values in deployment configuration.

The same file has:

```properties
spring.jpa.hibernate.ddl-auto=update
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQLDialect
```

`ddl-auto=update` is convenient for lessons, not a controlled production migration strategy.
Use reviewed schema migrations and an intentional production setting.
`data.sql` initialization behavior depends on Spring's SQL-init settings and database type.
Deferring initialization alone does not force SQL scripts to run on every external database.

### 7. MySQL module security and health-check implications

Path: `91-aws/02-rest-api-mysql/src/main/java/com/in28minutes/rest/webservices/restfulwebservices/security/SpringSecurityConfiguration.java`.

```java
http.authorizeHttpRequests(
        auth -> auth
            .requestMatchers("/").permitAll() //#CHANGE
            .anyRequest().authenticated()
        );
http.httpBasic(withDefaults());
```

Only the root path is explicitly public in this configuration.
Actuator exists, but `/actuator/health` is not explicitly permitted by this rule.
An unauthenticated load-balancer probe can therefore receive 401.
A permitted path still needs an actual successful handler; permitAll does not create one.
Choose and configure a genuine health endpoint, its exposure, and its security policy.
The learning setting exposing `*` should not be copied to public production access.

### 8. Full-stack H2 backend

Path: `91-aws/03-rest-api-full-stack-h2/src/main/resources/application.properties`.

```properties
spring.datasource.url=jdbc:h2:mem:testdb

spring.jpa.defer-datasource-initialization=true

spring.main.banner-mode=off
logging.pattern.console= %d{MM-dd HH:mm:ss} - %logger{36} - %msg%n

#CHANGE
server.port=5000
```

An in-memory database loses state when the process stops.
Two instances have two independent H2 databases, not a shared persistence layer.
This module is suitable for a full-stack demonstration, not durable scaled todo storage.
Its POM includes the OAuth2 resource-server starter for JWT support.
It does not declare the Actuator starter despite security matchers mentioning Actuator paths.
A security matcher is not evidence that an endpoint exists.

Path: `.../todo/TodoJpaResource.java` under this module's Java package.

```java
@GetMapping("/users/{username}/todos")
public List<Todo> retrieveTodos(@PathVariable String username) {
    //return todoService.findByUsername(username);
    return todoRepository.findByUsername(username);
}
```

The actual endpoint is `/users/{username}/todos`, without an extra `/jpa` prefix.
The frontend's `TodoApiService.js` uses that same route.
Older readme examples with different paths should not override live mappings.

### 9. JWT demo caveats during horizontal scaling

Path: `91-aws/03-rest-api-full-stack-h2/src/main/java/com/in28minutes/rest/webservices/restfulwebservices/jwt/JwtSecurityConfig.java`.

```java
session.sessionCreationPolicy(
    SessionCreationPolicy.STATELESS)
```

```java
var keyPairGenerator = KeyPairGenerator.getInstance("RSA");
keyPairGenerator.initialize(2048);
return keyPairGenerator.generateKeyPair();
```

The demo generates signing keys at application startup.
After a restart, previously issued JWTs may no longer validate.
With multiple independently started instances, their keys can differ.
Use a stable managed issuer/key distribution strategy for real multi-instance deployment.
Stateless sessions do not automatically solve persistence or signing-key coordination.
The live user-details service is in-memory; old notes about database-created users are stale.

### 10. React frontend configuration

Path: `91-aws/04-frontend-react/todo-app/package.json`.

```json
"scripts": {
  "start": "react-scripts start",
  "build": "react-scripts build",
  "test": "react-scripts test",
  "eject": "react-scripts eject"
}
```

The project uses React 18 and `react-scripts` 5.0.1.
The generated production directory is `build/`, not an assumed Vite `dist/`.
`npm start` is a development server, not the process to deploy to S3.
S3 serves built static objects; it does not run Node or Spring Boot.

Path: `91-aws/04-frontend-react/todo-app/src/components/todo/api/ApiClient.js`.

```javascript
export const apiClient = axios.create(
    {
        baseURL: 'http://localhost:5000' //#CHANGE
```

A deployed browser interprets localhost as the end user's computer.
Change this base URL in a practice deployment to the real HTTPS API endpoint.
The note recommends CRA environment variables, but the live code is a literal.
Setting `REACT_APP_API_URL` alone will do nothing until code reads it.
CRA's client configuration is baked into JavaScript at build time.
Anything placed there is public to users, including accidental secrets.

### 11. Deployment notes and their historical assumptions

Read paths:

- `91-aws/02-rest-api-mysql/readme.md`.
- `91-aws/03-rest-api-full-stack-h2/readme.md`.
- `91-aws/04-frontend-react/readme.md`.
- `91-aws/04-frontend-react/todo-app/README.md`.

The full-stack notes show `npm run build` and a public-read S3 bucket-policy pattern.
They also contain historical frontend paths, HTTP Beanstalk URLs, and inconsistent ports.
Use the actual `ApiClient.js`, package scripts, and controller mappings as authority.
The note's JWT header examples are truncated; use `Authorization: Bearer <token>`.
Do not send real passwords to online example password-hashing sites.
Use local trusted tooling for any real credential preparation.
Public S3 website hosting is a learning option; private S3 with CloudFront is stronger production design.

## Core concepts explained

### Containers versus virtual machines

```text
Virtual machines                      Containers
+-----------+ +-----------+           +----------+ +----------+
| App       | | App       |           | App/libs | | App/libs |
| Guest OS  | | Guest OS  |           +----------+ +----------+
+-----------+ +-----------+           | Shared host kernel    |
| Hypervisor              |           +-----------------------+
+-------------------------+           | Host OS / hardware    |
| Host / hardware         |           +-----------------------+
+-------------------------+
```

| Dimension | Container | VM |
|---|---|---|
| Isolation boundary | Processes/namespaces/cgroups | Virtual hardware + guest OS |
| Kernel | Shares host kernel | Has guest kernel |
| Startup | Usually fast | Typically slower |
| Size | App/runtime layers | Guest OS image plus app |
| Resource overhead | Lower in many workloads | Additional guest OS overhead |
| Portability | Image OS/CPU compatibility required | Virtual machine platform compatibility |
| Security | Not an absolute security boundary | Often stronger isolation, still needs hardening |

Docker Desktop on macOS runs Linux containers through a Linux VM.
A Linux image does not run directly on the macOS kernel.
ARM and amd64 images differ; choose the architecture of the deployment target.
Containerization does not remove the need to patch runtime and base images.

### Docker architecture

```text
Developer / CI
    |
Docker CLI ---- API ---- Docker daemon / BuildKit
                              |             |
                         build images   run containers
                              |
                       registry push/pull
```

The client sends commands to a daemon, which may be remote.
An image is a packaged filesystem/configuration template with immutable layers.
A container is a running or stopped instance with a writable layer.
A registry stores images; repositories and tags organize them.
Tags can move, while a digest identifies specific content.
A container's writable layer is not durable database storage.
Use volumes or external managed services for persistence.
The Docker socket grants powerful control; do not expose it casually to workloads.

### Dockerfile instruction reference

| Instruction | Meaning | Important distinction |
|---|---|---|
| `FROM` | Select base image or stage | Can reference prior stage |
| `WORKDIR` | Set working directory for later instructions/process | Prefer over repeated shell `cd` |
| `COPY` | Copy build-context files or another stage's files | Cannot freely read outside context |
| `RUN` | Execute during image build | Not executed at every container start |
| `EXPOSE` | Document intended listening port | Does not publish host port |
| `ENTRYPOINT` | Main executable/startup contract | Runtime args usually append in exec form |
| `CMD` | Default command or arguments | Runtime command can replace it |
| `ENV` | Persist non-secret image environment settings | Not a safe secret store |
| `ARG` | Build-time parameter | Can leak in metadata/layers; not for secrets |
| `USER` | Runtime user | Non-root is preferable where supported |
| `HEALTHCHECK` | Container health probe | Requires suitable command/runtime |

Exec form uses a JSON array and avoids an implicit shell.
Shell form enables shell features but complicates PID 1 and signal handling.
Environment variables are not automatically expanded inside an exec-form argument string.
Spring can read environment variables directly, so shell expansion is usually unnecessary.

Illustrative entrypoint/default-argument combination:

```dockerfile
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
CMD ["--server.port=5000"]
```

`docker run image --server.port=5001` replaces CMD defaults but keeps the entrypoint.
`--entrypoint` can explicitly replace the entrypoint.
`docker stop` sends a termination signal before eventually forcing termination.
Direct process execution and graceful shutdown configuration help requests drain correctly.

### Layers, caching, and build context

A build step can reuse cached output when its relevant inputs are unchanged.
A changed early step generally invalidates following dependent steps in that stage.
Put slowly changing dependency metadata before frequently changing source.
Use a `.dockerignore` in a practice image project to exclude unnecessary context.
Exclude secrets, dependency directories, IDE data, and unrelated generated output as appropriate.
For variant 1, do **not** ignore the target JAR you need to copy.
For multi-stage source builds, ignoring host `target/` is usually desirable.
Deleting a secret in a later layer does not erase it from an earlier image layer.
Use BuildKit secret mounts rather than ARG/COPY for private build credentials.

### Multi-stage and image-size optimization

Separate build tools from runtime contents.
Choose a maintained Java 21 JRE/runtime image rather than retaining Maven and all source.
Copy only the intended artifact or intentionally extracted Boot layers.
Use a non-root user and writable directories with suitable ownership.
Avoid unnecessary package installation and remove package caches in the same RUN step.
Layered JARs can separate dependency changes from application-class changes.
A smaller image improves transfer/storage, but size alone is not security quality.
Distroless images reduce utilities and can make interactive troubleshooting harder.
Rebuild regularly for patched base images; pinning a digest does not patch itself.
Scan artifacts and document exceptions rather than assuming slim equals safe.

Illustrative maintained-runtime shape, not an existing Dockerfile:

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY target/hello-world-java-0.0.1-SNAPSHOT.jar app.jar
USER 10001
EXPOSE 5000
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Verify image tag/platform availability and required directory permissions before use.
Applications writing files must have appropriately owned writable paths.
The current hello-world request does not need to write application files.

### Spring Boot Buildpacks

Cloud Native Buildpacks inspect application contents and construct an OCI image.
Spring Boot exposes that workflow through its build plugins.
No project Dockerfile is needed for this path.
A configured builder/run image determines the runtime stack.
The image still needs versioning, scanning, registry distribution, and operations.

```bash
# From repository root; Docker-compatible daemon required
mvn -f 83-docker/hello-world-java/pom.xml spring-boot:build-image \
  -Dspring-boot.build-image.imageName=hello-world-java:study
```

For a compatible Gradle Boot project, the analogous task is `bootBuildImage`.
The Maven goal can build/package as part of its lifecycle-forking behavior.
Do not confuse Maven artifact deploy with image push or runtime deployment.
Use supported builder configuration for Java 21, proxies, and offline environments.

### Docker networking and environment variables

```text
Host-running API -> localhost:3306 -> published MySQL port

Container API -> mysql:3306 -> MySQL on user-defined network
              (Docker DNS)
```

Inside an API container, localhost refers to that API container.
It does not refer to the host or the MySQL container.
A user-defined bridge network provides name-based container/service discovery.
Legacy `--link` examples are historical; prefer a named network or Compose.
The API and MySQL do not need host publication to talk on their private network.
Publish only ports that host/external clients need.

MySQL's `MYSQL_*` variables initialize the database image.
The application's `RDS_*` variables configure its datasource.
They are different namespaces, even when the values describe the same database.
Initialization variables normally apply when the MySQL data directory is first initialized.
Changing them does not automatically reset users/passwords in an existing volume.

### Compose: a local multi-service application

Compose defines related containers, networks, volumes, and configuration in YAML.
It is valuable for local development and integration environments.
It is not automatically a production multi-host scheduler.
The following is **illustrative**; no `compose.yaml` currently exists in these modules.

```yaml
services:
  mysql:
    image: mysql:8-oracle
    environment:
      MYSQL_DATABASE: social-media-database
      MYSQL_USER: social-media-user
      MYSQL_PASSWORD: ${DB_PASSWORD:?set DB_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD:?set DB_ROOT_PASSWORD}
    volumes:
      - mysql-data:/var/lib/mysql
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 --silent"]
      interval: 10s
      timeout: 5s
      retries: 12
  api:
    image: ${API_IMAGE:?build and set API_IMAGE}
    ports:
      - "5000:5000"
    environment:
      RDS_HOSTNAME: mysql
      RDS_PORT: "3306"
      RDS_DB_NAME: social-media-database
      RDS_USERNAME: social-media-user
      RDS_PASSWORD: ${DB_PASSWORD:?set DB_PASSWORD}
    depends_on:
      mysql:
        condition: service_healthy
volumes:
  mysql-data:
```

`API_IMAGE` must be an image built from the MySQL backend, not the hello-world module.
The mysqladmin probe checks server responsiveness, not successful application credentials/schema.
A health-based dependency helps startup ordering but does not replace app retries.
Production should use managed secret handling, not plaintext checked-in `.env` files.
`docker compose down` keeps named volumes unless explicitly removed.
Removing a database volume destroys its stored data; do it only deliberately.

### Cloud fundamentals

| Model | You operate | Provider operates | Example |
|---|---|---|---|
| IaaS | OS, runtime, app, data | Physical infrastructure/virtualization | EC2 |
| PaaS | App/configuration/data | Much of platform/runtime management | Elastic Beanstalk-style platform |
| SaaS | Usage/settings/data governance | Application and infrastructure | Hosted collaboration software |

AWS responsibility remains shared; managed does not mean configuration-free.
A **region** is a geographic deployment area.
An **Availability Zone** is an isolated location/group of facilities within a region.
Multiple AZs improve resilience against localized failures.
Multi-region adds latency, replication, routing, and operational complexity.
Keep compute and database placement/network design intentional.
A security group is a stateful virtual firewall, not an IAM permission policy.

### AWS services in this learning architecture

| Service | Role | Practical deployment concern |
|---|---|---|
| Elastic Beanstalk | Managed app environment orchestration | Platform version, deployment policy, health |
| EC2 | Compute underlying/custom instances | OS patches, instance sizing, security groups |
| RDS | Managed MySQL persistence | Backups, credentials, private access, migrations |
| S3 | Static frontend objects and artifact storage | Access policy, versioning, cache headers |
| CloudFront | CDN and HTTPS frontend distribution | Origin access, caching, SPA routing |
| IAM | AWS identity and permission control | Least privilege, roles, temporary credentials |

The source notes demonstrate Beanstalk, RDS-style variables, and S3 frontend hosting.
EC2 is also visible in database security-group comments.
CloudFront and production IAM practices are explanatory extensions here,
not existing CloudFormation or infrastructure code in these modules.
Do not confuse Spring login users with AWS IAM identities.
Use instance/service roles for AWS service access rather than embedded access keys.

```text
Browser
  | HTTPS
CloudFront ---- private S3 origin (React HTML/JS/CSS)
  |
  | browser calls HTTPS API endpoint
  v
Beanstalk environment -> load balancer -> EC2 Spring Boot instances
                                           |
                                           | private MySQL traffic
                                           v
                                          RDS
```

Static hosting and API hosting are separate deployments.
Serving JavaScript from S3 does not proxy its API requests automatically.
CORS and TLS must match the browser's actual origin and API endpoint.

### Deploying a Spring Boot JAR to Elastic Beanstalk

1. Build/test using the selected JDK 21 and Maven versions.
2. Identify the executable Boot JAR, not its `.original` backup or a sources JAR.
3. Create/select a Beanstalk **Java SE** platform supporting the application's runtime.
4. Confirm process startup and reverse-proxy port configuration.
5. Upload the JAR as an application version using console or supported deployment tooling.
6. Configure runtime environment properties and instance/service roles.
7. Deploy to a non-production environment first.
8. Inspect logs and health before shifting real traffic.
9. Promote the same verified artifact using an intentional rollout policy.
10. Retain a known-good version and a compatible database rollback strategy.

A runnable JAR can be deployed directly with supported Java SE platform conventions.
For a source bundle/Procfile workflow, an illustrative startup command is:

```text
web: java -jar application.jar
```

That filename must match the artifact in the bundle.
The platform's expectations, supported JDK, and proxy port must agree with the application.
These are Java SE steps, not Tomcat WAR-deployment steps.
Beanstalk is not merely “upload JAR and ignore operations.”
Environment health depends on application response, platform resources, and configuration.

### RDS MySQL configuration and connectivity

Set the module's exact datasource variables:

| Variable | Expected value |
|---|---|
| `RDS_HOSTNAME` | RDS endpoint hostname, not a JDBC URL |
| `RDS_PORT` | Usually `3306` |
| `RDS_DB_NAME` | Provisioned application database |
| `RDS_USERNAME` | Least-privileged application database user |
| `RDS_PASSWORD` | Securely supplied database password |

The module assembles the JDBC URL from these values.
Use an appropriate TLS configuration and trust setup for production database connections.
Keep RDS private where practical.
Allow inbound database traffic from the application's security group, not the whole internet.
Validate routing, subnet placement, DNS, and credentials separately.
Do not reuse literal security-group IDs from course comments in another account.
Use migrations and backups before deploying changes affecting schema or data.
RDS Multi-AZ is an availability feature, not automatic read scaling.
Read replicas address different use cases and replication consistency concerns.

### Deploying React to S3 and CloudFront

1. In a practice copy, make `ApiClient.js` use the intended HTTPS API URL.
2. If adopting `REACT_APP_API_URL`, change code to read it before building.
3. Run tests and `npm run build` in the actual `todo-app/` directory.
4. Inspect generated files for localhost URLs and accidental secrets.
5. Upload the contents of `build/` to the intended frontend bucket prefix.
6. Configure index document and SPA navigation behavior.
7. Select either learning-only public website hosting or private S3 behind CloudFront.
8. Configure HTTPS, caching, and exact permitted API origins.
9. Test direct navigation to a nested React route and authenticated API calls.

Illustrative frontend configuration extension:

```javascript
const baseURL = process.env.REACT_APP_API_URL;
export const apiClient = axios.create({ baseURL });
```

Build-time example after that code change:

```bash
REACT_APP_API_URL=https://api.example.com npm run build
```

S3 website endpoints do not provide HTTPS by themselves.
CloudFront with a private S3 REST origin and Origin Access Control is a stronger default.
Origin Access Control is not for S3 website endpoints; those behave as custom origins.
Do not disable account-wide Block Public Access just to follow a classroom recipe.
Public website hosting requires carefully scoped bucket settings if deliberately chosen.
For SPA routes, configure a suitable fallback/rewrite strategy for missing object paths.
Avoid converting every API error into an HTML index page.
Use long-lived immutable caching for hashed assets and short/revalidated caching for index.html.

### Health checks and operational visibility

A process being alive is different from being ready to serve user requests.
**Liveness** asks whether restarting the process may help.
**Readiness** asks whether traffic should currently be sent to it.
A failed external database should not necessarily trigger endless liveness restarts.
Choose timeout thresholds that tolerate JVM startup and temporary dependencies.
Probe a genuine low-cost endpoint returning the expected success status.
If using Actuator, include its starter and intentionally configure exposure/security.
The hello-world modules do not already provide Actuator health.
The MySQL module has Actuator but protects it under its current security rules.

Capture application logs, startup failures, request/error metrics, and database connectivity.
Send logs to platform aggregation/stdout rather than relying on disposable container files.
Avoid logging passwords, bearer tokens, and excessive personal data.
Graceful shutdown and connection draining reduce failed requests during rollout.
A health-green deployment can still have a functional regression; run smoke tests too.

### 12-factor principles applied to this repo

| Factor | Application/deployment lesson |
|---|---|
| 1. Codebase | One traceable codebase with multiple deploys |
| 2. Dependencies | Declare with Maven/Gradle/npm, do not rely on ambient libraries |
| 3. Config | Externalize RDS connection values and environment settings |
| 4. Backing services | Treat database endpoints as replaceable attachments |
| 5. Build/release/run | Separate immutable artifact creation from runtime configuration |
| 6. Processes | Stateless app processes; no durable local H2 assumption |
| 7. Port binding | App exposes a service port; platform routes traffic |
| 8. Concurrency | Scale processes/instances appropriately |
| 9. Disposability | Fast startup, graceful termination |
| 10. Dev/prod parity | Exercise real database behavior before production |
| 11. Logs | Emit event streams for platform collection |
| 12. Admin processes | Run migrations/maintenance as controlled one-off tasks |

Environment variables are one configuration mechanism, not a perfect secret-management system.
CRA frontend variables are public build inputs, unlike server-side secret configuration.
The H2 database and startup-generated JWT keys limit the demo's scaling/restart behavior.
External durable state and stable authentication infrastructure address those concerns.

### CI/CD basics

```text
Change -> compile/lint -> unit tests -> integration tests
       -> package -> image/build scan -> publish versioned artifact
       -> staging deploy -> health/smoke checks -> approval/policy
       -> production rollout -> monitor -> rollback if necessary
```

CI means continuously integrating changes with automated validation.
Continuous delivery keeps changes releasable; production release may require approval.
Continuous deployment automatically releases changes that satisfy policy checks.
Build once and promote the same artifact rather than rebuilding for each environment.
Record version/digest, test results, dependencies, and provenance.
Use least-privileged short-lived CI credentials, such as role federation where supported.
Never place AWS access keys in frontend bundles, Dockerfiles, or source configuration.
Rolling, immutable, and blue/green policies trade cost, speed, and rollback options.
Database schema compatibility can constrain rollback even when an old JAR is available.

## Common pitfalls / gotchas

| Pitfall | Why it fails | Better approach |
|---|---|---|
| `docker build` in current Docker module | No standalone Dockerfile exists | Practice copy or Boot Buildpacks |
| Historical OpenJDK tag assumed current | Tag may be unavailable/unmaintained | Verify maintained Java 21 runtime |
| Wrong wildcard JAR | Multiple outputs or non-executable artifact | Copy exact runnable artifact |
| EXPOSE treated as publication | Host cannot reach container automatically | Use `-p host:container` |
| Container uses localhost for MySQL | It points to itself | Network service name / RDS endpoint |
| MySQL variables used for API | API reads `RDS_*` names | Configure each service's namespace |
| Reusing database initialization vars | Existing volume keeps old state | Manage users/schema deliberately |
| H2 on multiple instances | Separate volatile databases | Shared durable persistence |
| New JWT key on each instance | Tokens fail across restart/instance | Stable issuer/key management |
| Beanstalk port mismatch | Reverse proxy cannot reach Java | Align app/process/platform port |
| Health endpoint protected | Probe receives 401 | Intentional health route/security |
| Actuator matcher without dependency | Endpoint still absent | Add/configure actual endpoint in practice |
| Frontend localhost deployed | Browser calls user's machine | Real HTTPS API URL |
| Setting env without code reading it | Literal baseURL unchanged | Modify configuration code before build |
| HTTPS frontend + HTTP API | Browser blocks mixed content | HTTPS everywhere |
| S3 route reload gives 404 | BrowserRouter path lacks object | SPA fallback/routing policy |
| Public bucket used for private data | Unintended disclosure | Dedicated static bucket or private origin |
| Schema update copied to prod | Unreviewed destructive/incompatible changes | Controlled migrations and backups |

## Interview questions & answers

### Basic

#### 1. What is a container?
An isolated process environment created from an image, sharing the host kernel.
On macOS, Docker Desktop supplies a Linux VM for Linux containers.

#### 2. Image versus container?
An image is the packaged template; a container is an instance with runtime state and a writable layer.
Deleting a container is not the same as deleting its source image.

#### 3. What is a registry?
A service storing/distributing images, such as Docker Hub or a private registry.
Tags organize versions, while digests identify immutable content.

#### 4. What does FROM do?
It selects a base image or stage for subsequent Dockerfile instructions.
Multiple FROM instructions enable multi-stage builds.

#### 5. RUN versus ENTRYPOINT?
RUN executes during image construction; ENTRYPOINT defines container startup behavior.
A Maven RUN build does not run again at every container start.

#### 6. Does EXPOSE 5000 open the host port?
No; it documents the intended container port.
Use `-p` or orchestration networking to make it reachable externally.

#### 7. Why use multi-stage builds?
They separate build dependencies from runtime contents and copy only required outputs.
The runtime image need not contain Maven or source code.

#### 8. What are IaaS, PaaS, and SaaS?
They describe increasing provider responsibility for infrastructure, platform, and application operation.
Customer responsibility for data/access/configuration does not disappear.

#### 9. Region versus Availability Zone?
A region is geographic; AZs are isolated deployment locations within a region.
Multi-AZ placement improves resilience to localized failures.

#### 10. What does S3 host in this application?
The built React frontend's static HTML, JavaScript, CSS, and assets.
It does not execute the Spring Boot backend or the React development server.

### Intermediate

#### 11. ENTRYPOINT versus CMD?
ENTRYPOINT defines the startup executable; CMD provides replaceable defaults/arguments.
In exec form, runtime arguments normally replace CMD and append to ENTRYPOINT.

#### 12. Why copy pom.xml before source for caching?
Dependency metadata changes less often than application source.
An early dependency warm-up layer can be reused while later source layers rebuild.

#### 13. What are Buildpacks?
Build tooling that detects app requirements and produces an OCI image using a builder/runtime stack.
Spring Boot's `spring-boot:build-image` avoids requiring a project Dockerfile.

#### 14. Why does localhost MySQL fail inside an API container?
Localhost means the API container itself.
Use the MySQL service name on a shared user-defined network or the managed database hostname.

#### 15. Why are MYSQL_PASSWORD and RDS_PASSWORD different here?
MYSQL_PASSWORD configures MySQL-image initialization; RDS_PASSWORD configures Spring's datasource.
Set compatible values but use the variable names each component actually consumes.

#### 16. How do Compose volumes help?
They keep database data outside the disposable container writable layer.
Volume deletion is still destructive, and backups remain necessary.

#### 17. How do you configure the RDS JDBC URL?
Supply RDS_HOSTNAME, RDS_PORT, and RDS_DB_NAME; the properties file assembles the URL.
Also supply RDS_USERNAME and RDS_PASSWORD through secure runtime configuration.

#### 18. Why use security groups instead of exposing RDS publicly?
They restrict network access to approved application sources.
Database authentication remains necessary; network filtering and identity solve different layers.

#### 19. How is a Spring Boot JAR deployed to Beanstalk?
Use a compatible Java SE environment, upload the executable artifact, configure port/env/roles, and verify health.
Do not confuse this with Maven's deploy phase or a Tomcat WAR workflow.

#### 20. Why is a React environment variable not a secret?
CRA embeds values into JavaScript served to the browser.
Users can inspect that bundle, so only public configuration belongs there.

#### 21. What does CloudFront add to S3 hosting?
A CDN, HTTPS delivery, cache control, and private-origin integration options.
A private S3 REST origin with OAC is different from a public S3 website endpoint.

#### 22. What is the difference between readiness and liveness?
Readiness decides traffic eligibility; liveness decides whether a process needs restarting.
A dependency outage should not automatically be treated as a reason for endless restarts.

### Advanced and scenario-based

#### 23. The backend is healthy locally but Beanstalk returns 502. What do you check?
Check Java startup/logs, proxy target port, process command, and bind/listening configuration.
A successfully uploaded JAR is not evidence that the proxy can reach the process.

#### 24. The load balancer probes Actuator and gets 401 in the MySQL module. Why?
Its security configuration permits only `/` and authenticates other requests.
Configure an actual suitable health endpoint and a deliberate probe access policy.

#### 25. Todos disappear after deployment. Why in the H2 example?
It uses `jdbc:h2:mem:testdb`, so data is tied to the process lifetime.
Use durable shared persistence for data that must survive restarts and scaling.

#### 26. JWTs work on one instance but fail on another. What in this demo explains it?
Each startup generates its own RSA signing keys.
Use consistent trusted issuer/key distribution rather than independent random keys per instance.

#### 27. Changing DB_PASSWORD in Compose does not change the existing user's password. Why?
MySQL initialization environment values are normally applied only to a new data directory.
An existing volume preserves users; manage credential rotation explicitly.

#### 28. React loads but API calls target localhost. How do you fix it?
Update the actual ApiClient configuration and rebuild with the intended public HTTPS endpoint.
Setting a variable alone is insufficient while the source uses a hardcoded literal.

#### 29. React navigation works until the browser reloads `/todos`. Why?
The static origin has no object named `/todos` for that direct request.
Configure SPA fallback/rewrite behavior while preserving correct API error handling.

#### 30. Why does HTTPS frontend calling HTTP API fail?
Browsers block active mixed content.
Expose the API through HTTPS and configure CORS for the frontend's actual origin.

#### 31. Can deleting a credential file in a later Docker layer make the image safe?
No; earlier layers can retain the sensitive contents.
Use build-time secret mounts and rotate any exposed credentials.

#### 32. How do you design reliable rollback?
Keep known-good immutable artifacts and use a rollout policy with measurable health/smoke checks.
Ensure database migrations remain compatible, since reverting application code cannot undo arbitrary schema/data changes.

#### 33. Why not use latest tags in release deployments?
They are mutable and make the exact deployed contents ambiguous.
Use traceable version tags and recorded digests, with deliberate security rebuilds.

#### 34. Does Multi-AZ RDS mean automatic read scaling?
No; it primarily supports availability/failover configurations.
Read replicas serve different scaling/use cases and have consistency considerations.

## Hands-on exercises

### Exercise 1 — Compare the three Docker recipes

In a separate practice copy, build the JAR-first and multi-stage variants.
Replace historical base tags with verified supported Java 21 images as needed.
Compare final image contents, sizes, build times, and process startup commands.
Success: explain what remains in the runtime stage and why EXPOSE is not publication.

### Exercise 2 — Observe cache invalidation

Build the cache-oriented variant twice, then change only a controller response.
Compare this with changing pom.xml or the copied application entry point.
Explain which earlier inputs invalidate which layers.
Success: demonstrate useful caching without skipping the final build and tests.

### Exercise 3 — Run MySQL plus API privately

In a practice copy, build an API image from `02-rest-api-mysql` and adapt the Compose example.
Supply non-committed passwords and verify connectivity through the service name.
Restart the API, then MySQL, and observe volume persistence and readiness behavior.
Success: do not publish the database unnecessarily and do not destroy stored data casually.

### Exercise 4 — Plan a durable AWS backend deployment

Prepare a Java SE Beanstalk deployment checklist with runtime version, port, health route, and roles.
Specify private RDS access, all five RDS variables, backups, and migrations.
Explain how the H2 and generated-JWT-key demo would need to evolve before scaling.
Success: include rollback and functional smoke tests, not just artifact upload.

### Exercise 5 — Deploy a static frontend in an isolated learning account

Use a practice copy to replace the hardcoded API URL and produce a production build.
Choose private S3 plus CloudFront or explicitly scoped learning-only website hosting.
Test deep-link navigation, cache behavior, HTTPS, CORS, and authentication.
Success: no credentials in the bundle and no requests to the user's localhost.

## Quick revision cheat sheet / commands

### Existing modules: build and run

```bash
# Repository root, installed Maven, Java 21
mvn -f 83-docker/hello-world-java/pom.xml package
mvn -f 91-aws/01-hello-world-java/pom.xml package
mvn -f 91-aws/02-rest-api-mysql/pom.xml spring-boot:run
mvn -f 91-aws/03-rest-api-full-stack-h2/pom.xml spring-boot:run
curl http://localhost:5000/

# Buildpacks: no source Dockerfile required
mvn -f 83-docker/hello-world-java/pom.xml spring-boot:build-image \
  -Dspring-boot.build-image.imageName=hello-world-java:study
```

### Docker commands: only run against your own practice resources

```bash
docker version
docker images
docker ps
docker ps -a

# Existing Buildpacks image
docker run --rm --name hello-study -p 127.0.0.1:5000:5000 hello-world-java:study

# Separate terminal while that named container is running
docker logs hello-study
docker inspect hello-study
docker exec hello-study java -version
docker stop hello-study

# Generic practice Dockerfile commands
docker build -t example/app:study .
docker tag example/app:study registry.example.com/team/app:study
docker push registry.example.com/team/app:study
docker pull registry.example.com/team/app:study
docker network create study-net
```

`push` requires an intended registry/repository and authentication.
The foreground `docker run` remains attached; stop it explicitly when finished.
Avoid broad prune commands in shared environments.
Do not remove containers/images/volumes belonging to other users.

### Compose commands: practice directory containing compose.yaml

```bash
docker compose config
docker compose up --build
docker compose ps
docker compose logs api
docker compose stop
# Removes this project's containers/networks; normally preserves named volumes
docker compose down
```

### Frontend build and scoped S3 upload example

```bash
cd 91-aws/04-frontend-react/todo-app
npm ci
CI=true npm test -- --watchAll=false
npm run build

# Only after selecting/configuring your own bucket and credentials
aws s3 sync build/ s3://YOUR-FRONTEND-BUCKET/
```

That sync example does not delete objects absent from the local build.
Manage outdated assets/versioning/cache invalidation intentionally.
It does not enable website hosting, configure CloudFront, or deploy the backend.
Be aware that creating and running AWS resources can incur charges.

| Memory hook | Meaning |
|---|---|
| Image | Packaged immutable layers/configuration |
| Container | Runtime instance + writable layer |
| Registry | Image distribution service |
| EXPOSE | Metadata, not a firewall rule |
| `-p` | Host-to-container publication |
| Multi-stage | Build tools separated from runtime |
| Buildpacks | OCI image without project Dockerfile |
| MySQL peer | Service DNS name, not container localhost |
| RDS variables | Exact placeholders consumed by datasource |
| H2 memory | Volatile and per process |
| React config | Public, usually baked at build time |
| S3 | Static objects, not a Java process |
| CloudFront | CDN/HTTPS and origin access |
| IAM | AWS permissions, distinct from app login |
| Health | Real endpoint + correct access + useful semantics |
| CI/CD | Validate, version, promote, monitor, rollback |

Final memory hook: **package consistently, configure externally, verify before routing traffic**.
