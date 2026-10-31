# 03 — JPA, Hibernate, and Spring Data JPA
## What this module teaches
**Exact module path:** `03-jpa-getting-started/`.

This module saves the same `Course` using three progressively higher-level APIs: Spring JDBC, a hand-written JPA repository, and a Spring Data JPA interface. The currently active
startup runner uses Spring Data JPA. The other implementations remain in the source for comparison.

The build declares Spring Boot **3.3.4** and Java **21**. Boot manages the compatible Spring Framework 6, Spring Data 3, and Hibernate 6 dependencies. Persistence imports use
`jakarta.persistence`, not the old `javax.persistence` namespace.

### How to run
From the repository root, with JDK 21 and Maven installed:

```bash
mvn -f 03-jpa-getting-started/pom.xml spring-boot:run
```

There is no Maven wrapper in this module; do not assume `./mvnw` exists. The web starter keeps the application running on port 8080 by default. Stop another application using that
port before starting this one. Alternatively, supply a different port:

```bash
mvn -f 03-jpa-getting-started/pom.xml spring-boot:run \
  -Dspring-boot.run.arguments="--server.port=8081"
```

Open `http://localhost:8080/h2-console` when using the default port. Use JDBC URL `jdbc:h2:mem:testdb`, username `sa`, and an empty password. The console must connect to the same
URL as the application. An in-memory database disappears when its JVM/database lifetime ends.

The runner inserts IDs 1, 2, and 3, then deletes ID 1. Its final repository count is therefore **2** in a fresh successful run. `findById` prints `Optional` values, not just raw
entities. The AWS-name finder returns an empty list after its row has been deleted.

These are expected behaviors from reading the code, not a claim of an executed build. The examples below distinguish **actual repository excerpts** from **illustrative
extensions**.

## Project walkthrough
### Source map
All paths in this subsection are relative to `03-jpa-getting-started/`.

```text
pom.xml
src/main/java/com/in28minutes/springboot/learnjpaandhibernate/
  LearnJpaAndHibernateApplication.java
  course/
    Course.java
    CourseCommandLineRunner.java
    jdbc/CourseJdbcRepository.java
    jpa/CourseJpaRepository.java
    springdatajpa/CourseSpringDataJpaRepository.java
src/main/resources/
  application.properties
  schema.sql
```

### 1. `pom.xml`: what provides the behavior?
The actual build includes:

| Dependency | Purpose in this module |
|---|---|
| `spring-boot-starter-data-jdbc` | JDBC infrastructure, including `JdbcTemplate` |
| `spring-boot-starter-data-jpa` | JPA integration, Hibernate, Spring Data repositories |
| `spring-boot-starter-web` | Embedded servlet server and browser access |
| `h2`, runtime scope | Local embedded SQL database |
| `spring-boot-starter-test` | Testing support |

The JDBC starter does not make `CourseJdbcRepository` a Spring Data JDBC repository. That class explicitly calls `JdbcTemplate` and manually supplies SQL. Similarly, importing a
JPA starter does not turn every Java class into an entity. Entity annotations and scanning determine which classes participate in ORM.

### 2. `LearnJpaAndHibernateApplication.java`: the entry point
Path: `src/main/java/com/in28minutes/springboot/learnjpaandhibernate/LearnJpaAndHibernateApplication.java`.

Actual excerpt:

```java
@SpringBootApplication
public class LearnJpaAndHibernateApplication {

	public static void main(String[] args) {
		SpringApplication.run(LearnJpaAndHibernateApplication.class, args);
	}

}
```

`@SpringBootApplication` combines configuration, component scanning, and auto-configuration. Its package is above the `course` package, so the repository components are
discoverable. Boot also uses the application's auto-configuration packages for entity/repository discovery. A repository or entity moved outside that hierarchy may need explicit
scanning configuration.

Startup is not only `main`:

```text
main
  -> build ApplicationContext
  -> configure DataSource
  -> initialize SQL/JPA schema as configured
  -> create repository infrastructure
  -> invoke CommandLineRunner
  -> application ready for normal use
```

### 3. `Course.java`: a Java object mapped to a row
Path: `src/main/java/com/in28minutes/springboot/learnjpaandhibernate/course/Course.java`.

Actual excerpt:

```java
@Entity
public class Course {
	
	@Id
	private long id;
	
	private String name;
	
	private String author;

	public Course() {

	}
```

The no-argument constructor allows the persistence provider to instantiate the entity. The public constructor accepting ID, name, and author supports the startup examples. The
getters/setters also allow `BeanPropertyRowMapper` to populate the same object.

Important observations:

- `Course` has **assigned IDs**; it does not declare `@GeneratedValue`.
- `@Id` is on a field, so JPA uses field access by default.
- There are no `@Table`, `@Column`, relationship, or `@Version` annotations here.
- No annotation states that `name` or `author` must be non-null.
- The SQL script has stronger nullability constraints than this entity declaration.
- The primitive `long` ID cannot represent `null`; the runner explicitly assigns it.

An entity is not automatically a Spring singleton bean. Spring manages repository/service components; Hibernate manages entity instances. The entity's lifecycle is associated with
a persistence context, not component scanning alone.

### 4. `CourseJdbcRepository.java`: SQL remains explicit
Path: `src/main/java/com/in28minutes/springboot/learnjpaandhibernate/course/jdbc/CourseJdbcRepository.java`.

Actual excerpt:

```java
@Repository
public class CourseJdbcRepository {
	
	@Autowired
	private JdbcTemplate springJdbcTemplate;
```

Actual insert implementation:

```java
public void insert(Course course) {
	springJdbcTemplate.update(INSERT_QUERY, 
			course.getId(), course.getName(), course.getAuthor());
}
```

The actual `INSERT_QUERY` contains:

```sql
insert into course (id, name, author)
values(?, ?,?);
```

The `?` placeholders are bound parameters. Values are not concatenated into SQL text. This is both convenient and safer than building SQL from untrusted strings.

Actual row lookup:

```java
public Course findById(long id) {
	//ResultSet -> Bean => Row Mapper => 
	return springJdbcTemplate.queryForObject(SELECT_QUERY,
			new BeanPropertyRowMapper<>(Course.class), id);
	
}
```

`BeanPropertyRowMapper` translates result columns into bean properties. It relies on conventions and setters rather than JPA entity lifecycle management. `queryForObject` expects
exactly one row. No row normally produces `EmptyResultDataAccessException`. Multiple rows produce an incorrect-result-size exception. There is no `Optional` wrapper in this JDBC
method.

`deleteById` executes `DELETE_QUERY` using `JdbcTemplate.update`. Deleting a missing row usually affects zero rows rather than throwing a not-found error. The current methods
ignore the returned affected-row count.

### 5. `CourseJpaRepository.java`: work with entities
Path: `src/main/java/com/in28minutes/springboot/learnjpaandhibernate/course/jpa/CourseJpaRepository.java`.

Actual imports/annotations and field:

```java
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import jakarta.transaction.Transactional;
```

```java
@Repository
@Transactional
public class CourseJpaRepository {
	
	@PersistenceContext
	private EntityManager entityManager;
```

Actual operations:

```java
public void insert(Course course) {
	entityManager.merge(course);
}

public Course findById(long id) {
	return entityManager.find(Course.class, id);
}

public void deleteById(long id) {
	Course course = entityManager.find(Course.class, id);
	entityManager.remove(course);
}
```

Despite its name, `insert` calls `merge`, not `persist`. It can copy state into a new or existing managed instance. The returned managed instance is discarded in this example. The
supplied object does not become managed merely because it was passed to `merge`.

`find` returns a managed entity when found, or `null` when missing. `remove` expects a managed entity. If `find` returns `null`, passing it to `remove` is an error. A robust delete
implementation must define a missing-ID policy before removal.

`@PersistenceContext` injects a Spring-managed shared EntityManager proxy. The proxy delegates to the EntityManager associated with the current transaction. It is not a single
mutable EntityManager shared directly between all request threads.

The class uses Jakarta's transaction annotation. Spring recognizes it, but Spring's richer `propagation`/`isolation` attributes require
`org.springframework.transaction.annotation.Transactional` instead. Do not paste Spring-specific attributes onto `jakarta.transaction.Transactional`.

### 6. `CourseSpringDataJpaRepository.java`: an interface is enough
Path: `src/main/java/com/in28minutes/springboot/learnjpaandhibernate/course/springdatajpa/CourseSpringDataJpaRepository.java`.

Actual excerpt:

```java
public interface CourseSpringDataJpaRepository extends JpaRepository<Course, Long>{
	
	List<Course> findByAuthor(String author);
	List<Course> findByName(String name);

}
```

`Course` is the domain type; `Long` is its identifier type. Generic parameters use wrapper classes even though the entity field is primitive `long`. Spring Data creates a
repository proxy during startup. The proxy combines CRUD implementations with generated query methods.

`findByAuthor` is parsed against the **Java property** `author`. `findByName` is parsed against `name`. A typo such as `findByAuthro` generally fails repository initialization. No
handwritten implementation and no explicit `@Repository` are needed on this interface.

The return type `List<Course>` means zero matches produce an empty list. It permits multiple courses with the same author or name. It does not imply a unique database constraint.

### 7. `CourseCommandLineRunner.java`: see APIs in action
Path: `src/main/java/com/in28minutes/springboot/learnjpaandhibernate/course/CourseCommandLineRunner.java`.

Actual active injection:

```java
@Autowired
private CourseSpringDataJpaRepository repository;
```

The JDBC and custom JPA injections are commented out. Those comments document the teaching progression, not simultaneous writes through all three APIs.

Actual startup excerpt:

```java
repository.save(new Course(1, "Learn AWS Jpa!", "in28minutes"));
repository.save(new Course(2, "Learn Azure Jpa!", "in28minutes"));
repository.save(new Course(3, "Learn DevOps Jpa!", "in28minutes"));

repository.deleteById(1l);

System.out.println(repository.findById(2l));
System.out.println(repository.findById(3l));
```

Later it prints `findAll`, `count`, and both derived finder methods. Because IDs are already nonzero and there is no version-based new-state detection, Spring Data typically
considers these `Course` instances not new and uses `merge`. Hibernate can still insert the corresponding rows when no row exists. Assigned-ID merge can require an existence lookup
before insertion.

The runner itself is not annotated transactional. Each inherited repository write method normally supplies its own transaction. Three saves and one delete are therefore not
automatically one atomic unit. A service-level transaction would be appropriate if they represented one business operation.

### 8. `application.properties` and `schema.sql`
Path: `src/main/resources/application.properties`.

Actual content:

```properties
spring.h2.console.enabled=true
spring.datasource.url=jdbc:h2:mem:testdb
spring.jpa.show-sql=true
```

Path: `src/main/resources/schema.sql`.

Actual content:

```sql
create table course
(
	id bigint not null,
	name varchar(255) not null,
	author varchar(255) not null,
	primary key (id)

);
```

The whitespace above is normalized; the SQL clauses are copied from the source. Boot normally runs SQL initialization for an embedded database. This module does not explicitly set
`spring.jpa.hibernate.ddl-auto`. For an embedded database without Flyway/Liquibase, Boot normally defaults it to `create-drop`. Consequently, do not assume this script is the sole
schema owner. Hibernate may recreate the table from entity metadata after the script runs. That can affect the final constraints because the entity has no matching `nullable=false`
declarations.

For a predictable learning experiment, choose one schema authority:

| Approach | Illustrative settings |
|---|---|
| SQL owns schema | `spring.jpa.hibernate.ddl-auto=none` |
| Hibernate owns schema | Disable SQL initialization with `spring.sql.init.mode=never` |
| Production migrations own schema | Flyway/Liquibase plus `ddl-auto=validate` or `none` |

Do not change these settings blindly in a production database. `show-sql` prints SQL but does not guarantee bound parameter visibility. Hibernate 6 bind logging can be enabled
selectively:

```properties
# Illustrative diagnostic settings, not current module properties.
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.orm.jdbc.bind=TRACE
```

Bind logs may expose private data; avoid enabling them broadly in production.

## Core concepts explained
### 1. JDBC vs Spring JDBC vs JPA vs Hibernate vs Spring Data JPA
| Technology | What it is | You usually write | Who maps rows? | Current example |
|---|---|---|---|---|
| JDBC | Java database API | Connections, statements, SQL, result handling | You | Underlying driver/API |
| Spring JDBC | Convenience layer over JDBC | SQL and row mapping | RowMapper | `CourseJdbcRepository` |
| JPA | Persistence specification/API | Entities and EntityManager calls | Provider | `CourseJpaRepository` |
| Hibernate | ORM implementation of JPA, plus native features | Mappings, queries, optional provider settings | Hibernate | Boot's provider |
| Spring Data JPA | Repository abstraction built on JPA | Interfaces and finder declarations | JPA provider | `CourseSpringDataJpaRepository` |

JPA is not a database and Spring Data JPA is not an alternative SQL engine. Hibernate does not eliminate JDBC; it ultimately uses JDBC to execute database work.

```text
Spring Data JPA repository
         |
         v
JPA EntityManager API
         |
         v
Hibernate ORM provider
         |
         v
JDBC driver -> H2 / MySQL / another relational database
```

Use JDBC when precise SQL control and simple row-oriented operations are important. Use JPA when object lifecycle, relationships, and domain modeling justify ORM complexity. Mixing
them is possible, but keep their transaction and cache interactions explicit. A JDBC update is not automatically reflected in an already-managed entity.

### 2. What ORM solves—and what it does not
ORM means **object-relational mapping**. It bridges Java objects with tables, columns, keys, and relationships.

```text
Java                              Relational database
Course instance                   course row
id = 2            <----------->   id = 2
name = "Learn Azure Jpa!"          name = 'Learn Azure Jpa!'
author = "in28minutes"             author = 'in28minutes'
```

Objects have references and behavior; relational tables have foreign keys and sets of rows. ORM translates between those models. It also tracks changes and coordinates writes
inside transactions. It cannot design your indexes, authorization policy, or efficient fetch plan for you.

A query returning ten entities may issue more than one SQL statement. Calling a getter may issue SQL when a lazy association is initialized. That is why reading generated SQL is
part of learning JPA, not an optional distraction.

### 3. Entity mapping annotations
| Annotation | Meaning | Common misunderstanding |
|---|---|---|
| `@Entity` | Marks a persistent entity class | Does not make it a service bean |
| `@Entity(name="...")` | Changes the JPQL entity name | Not identical to `@Table` |
| `@Table(name="...")` | Configures the physical table mapping | Does not change the JPQL entity name |
| `@Id` | Identifies the primary-key field/property | Required even for natural keys |
| `@GeneratedValue` | Requests provider/database ID generation | Not present on this `Course` |
| `@Column` | Column name, length, nullability, uniqueness metadata | Not request validation |
| `@Transient` | Excludes a persistent field/property | Different from detached state |
| `@Version` | Optimistic concurrency token | Not a business revision counter |

Illustrative extension, not the actual `Course`:

```java
@Entity
@Table(name = "courses")
public class Course {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "course_name", nullable = false, length = 200)
    private String name;

    protected Course() {}
}
```

A JPA entity needs a public/protected no-argument constructor. For portable JPA and easy proxying, avoid final entity classes and final persistent methods. Field access lets the
provider read fields directly. Property access applies when mapping annotations are on getters. Use `@Access` deliberately if mixing access strategies.

`@Column(nullable=false)` describes database mapping/DDL requirements. `@NotNull` validates Java values when Bean Validation is invoked. Use both when both concerns matter; neither
is a complete substitute for the other. Existing production schema changes still require an explicit migration.

### 4. Generated ID strategies in Hibernate 6
| Strategy | Mechanism | Typical implications |
|---|---|---|
| `AUTO` | Provider chooses an appropriate strategy | Database/provider dependent |
| `IDENTITY` | Database identity/auto-increment column | Insert needed to obtain ID; limits insert batching |
| `SEQUENCE` | Database sequence | Can allocate IDs before insert and support batching |
| `TABLE` | Dedicated table stores allocation state | More coordination and potential contention |
| `UUID` | Provider generates UUID identifiers | Available in Jakarta Persistence 3.1 |

Omitting the strategy means `AUTO`. It does not mean `IDENTITY` on every database. MySQL has auto-increment columns but not PostgreSQL-style standalone sequences. Hibernate 6 may
use a table-backed mechanism for sequence-style generation there.

Illustrative sequence configuration:

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "course_seq")
@SequenceGenerator(name = "course_seq", sequenceName = "course_seq", allocationSize = 50)
private Long id;
```

The database sequence configuration must match the mapping's allocation expectations. Generated identifiers can have gaps after rollback or allocation. Do not promise gap-free
business numbering from an ORM primary-key generator.

### 5. EntityManager and persistence context
An `EntityManager` is the JPA API for operations such as `find`, `persist`, `merge`, and `remove`. Its persistence context tracks managed entities and their identity.

```text
Transaction A
  EntityManager / persistence context
    Course#2 -> one managed Java object
    Course#3 -> one managed Java object
    snapshots -> values used for dirty checking

Transaction B
  different persistence context
    Course#2 -> potentially a different Java object
```

Within one context, repeated `find(Course.class, 2L)` normally returns the same instance. A query can still execute SQL even if some returned entities are already managed.
First-level caching is identity/state management, not universal query-result caching.

Typical service transaction:

```java
// Illustrative; imports Spring's @Transactional.
@Transactional
public void renameCourse(long id, String name) {
    Course course = entityManager.find(Course.class, id);
    if (course == null) {
        throw new IllegalArgumentException("Course not found");
    }
    course.setName(name);
}
```

No explicit `save` is needed for a managed entity changed inside this write transaction. Dirty checking detects the changed `name` and schedules an update at flush. An
EntityManager is not generally thread-safe. Do not hand managed entities to worker threads and assume their context follows them.

### 6. Entity states
```text
new Course(...)
     |
     v
TRANSIENT --persist--> MANAGED --detach/clear/close--> DETACHED
                         |                              |
                       remove                        merge
                         |                              |
                         v                              v
                       REMOVED                managed copy returned
```

| State | In context? | Scheduled database effect |
|---|---|---|
| Transient/new | No | None until persistence operation |
| Managed | Yes | Changes can be detected and flushed |
| Detached | No longer | Changes are not tracked automatically |
| Removed | Yes, pending removal | Delete normally occurs during flush |

`persist` makes a new entity managed. `merge` copies supplied state into a managed entity and returns that managed entity. `detach(entity)` removes one instance from tracking.
`clear()` detaches everything in the context. `refresh(entity)` reloads database state and can overwrite unsaved changes.

Illustrative merge distinction:

```java
Course detached = new Course(2, "Updated title", "in28minutes");
Course managed = entityManager.merge(detached);
managed.setAuthor("New author");
```

Do not infer `detached == managed`. Mutating `detached` after merge does not reliably update the managed copy. After a normal transaction-scoped context closes, entities returned
to callers are detached. Open EntityManager in View can extend context availability in web applications, but not a business transaction.

### 7. `@Transactional`: boundaries before convenience
Spring transaction support commonly works through AOP proxies. Put business atomicity in a service rather than relying on unrelated repository calls.

```text
Caller -> service proxy -> begin/join transaction
                         -> target method
                            -> repository A
                            -> repository B
                         -> flush and commit, or rollback
```

Illustrative Spring annotation:

```java
@Transactional(rollbackFor = IOException.class)
public void importCourses() throws IOException {
    // Multiple related repository operations belong to one unit.
}
```

Spring's default rollback rule is rollback for `RuntimeException` and `Error`. Checked exceptions do not trigger rollback by default. `rollbackFor` and `noRollbackFor` customize
the rule. Jakarta's annotation offers `rollbackOn` and `dontRollbackOn` instead.

Catching an exception and returning normally can prevent the interceptor from applying its rule. A participating transaction may nevertheless already be marked rollback-only. That
can cause `UnexpectedRollbackException` at the outer commit. Always distinguish "exception handled" from "transaction safe to commit."

#### Propagation
| Spring propagation | Existing transaction | No existing transaction |
|---|---|---|
| `REQUIRED` | Join it | Create one |
| `REQUIRES_NEW` | Suspend it; create independent one | Create one |
| `SUPPORTS` | Join it | Run without one |
| `MANDATORY` | Join it | Throw exception |
| `NOT_SUPPORTED` | Suspend it | Run without one |
| `NEVER` | Throw exception | Run without one |
| `NESTED` | Use savepoint if supported | Behave like `REQUIRED` |

`REQUIRED` is the default. `REQUIRES_NEW` may need another database connection while the outer transaction holds one. Do not treat it as a free fix for rollback problems. `NESTED`
depends on the transaction manager and savepoint support. JPA transaction managers do not provide universally portable nested entity-state semantics.

#### Isolation
| Isolation | Intended protection | Remaining issues |
|---|---|---|
| `DEFAULT` | Use database default | Depends on database |
| `READ_UNCOMMITTED` | Lowest isolation requested | Dirty reads may occur |
| `READ_COMMITTED` | Prevent dirty reads | Non-repeatable/phantom reads may occur |
| `REPEATABLE_READ` | Prevent dirty/non-repeatable reads | Phantom behavior depends on implementation |
| `SERIALIZABLE` | Strongest standard isolation | More contention/retries possible |

Dirty read: observe another transaction's uncommitted write. Non-repeatable read: reread a row and see a changed committed value. Phantom read: repeat a predicate query and see a
changed qualifying set. Actual database engines implement these levels differently. MySQL InnoDB commonly defaults to `REPEATABLE_READ`; that is not a universal SQL default.
Isolation is generally applied when starting a new transaction, not by magically changing one already joined.

#### Self-invocation pitfall
```java
// Illustrative: a proxy-based transaction will not intercept this internal call.
public void importAll() {
    saveOne();
}

@Transactional
public void saveOne() {
    // ...
}
```

Calling `this.saveOne()` bypasses the proxy. Move the transactional method to another injected bean or use `TransactionTemplate` deliberately. If the outer method is already
transactional, its transaction still exists; the inner annotation simply does not add its own propagation behavior through self-invocation. Annotating private methods is not a
reliable proxy-based transaction design.

`readOnly=true` is an optimization hint, not a general write-protection guarantee. Do not depend on it instead of authorization or database permissions.

### 8. Derived queries and explicit queries
Illustrative additions to the existing repository:

```java
List<Course> findByAuthorAndNameContainingIgnoreCase(String author, String name);
List<Course> findTop5ByAuthorOrderByIdDesc(String author);
boolean existsByName(String name);
long countByAuthor(String author);
```

Keywords express predicates, ordering, limiting, and result shape. Property names must match the domain model. A long derived method can be harder to maintain than an explicit
query.

JPQL uses **entity names and Java attributes**:

```java
@Query("select c from Course c where c.author = :author")
List<Course> searchByAuthor(@Param("author") String author);
```

Native SQL uses **database tables and columns**:

```java
@Query(value = "select * from course where author = :author", nativeQuery = true)
List<Course> searchByAuthorNative(@Param("author") String author);
```

| Concern | JPQL | Native SQL |
|---|---|---|
| Names | Entities/properties | Tables/columns |
| Portability | Usually greater | Database-specific syntax possible |
| Joins | Entity associations supported | Foreign-key SQL joins |
| Database features | Limited to query language/provider support | Direct access to SQL features |

JPQL is not "SQL with capitalized table names." Hibernate HQL adds features beyond JPQL; do not call every HQL extension portable JPA. Bind parameters in both query styles. Native
pageable queries may need an explicit `countQuery` for reliable totals.

Bulk JPQL updates/deletes use `@Modifying` and a write transaction. They bypass normal per-entity dirty checking and can leave managed instances stale. Consider flushing before and
clearing afterward when mixing bulk operations with entity access.

### 9. Relationships: use the actual REST module as a bridge
`Course` has no relationships, so this topic refers to `12-rest-api/`.

Actual path: `12-rest-api/src/main/java/com/in28minutes/rest/webservices/restfulwebservices/user/User.java`.

Actual excerpt:

```java
@Entity(name = "user_details")
public class User {
```

```java
@OneToMany(mappedBy = "user")
@JsonIgnore
private List<Post> posts;
```

Actual path: `12-rest-api/src/main/java/com/in28minutes/rest/webservices/restfulwebservices/user/Post.java`.

Actual excerpt:

```java
@ManyToOne(fetch = FetchType.LAZY)
@JsonIgnore
private User user;
```

```text
User#1
  posts -> [Post#10, Post#11]
                |        |
                +--- user reference ---> User#1

user_details                     post
id = 1                           id = 10, user_id = 1
                                 id = 11, user_id = 1
```

`Post.user` is the owning side in this bidirectional relationship. Its foreign-key mapping controls association updates. `User.posts` is inverse because `mappedBy="user"` names the
**Post property**. It is not the name of a SQL column.

Both sides should be kept consistent in memory:

```java
// Illustrative association helper, not present in User.java.
public void addPost(Post post) {
    posts.add(post);
    post.setUser(this);
}
```

The actual classes do not specify cascade or orphan removal. Saving a user does not therefore imply saving every transient post. Deleting a user with referencing posts can fail a
foreign-key constraint. `cascade` propagates entity operations; it does not determine fetch timing. `orphanRemoval=true` removes children disassociated from an appropriate owned
lifecycle. Use cascading carefully, especially `REMOVE` on many-to-one relationships.

`@JsonIgnore` controls Jackson serialization. It neither changes relationship ownership nor prevents Hibernate from loading an association. DTOs often make REST payload and fetch
requirements clearer than serializing entities directly.

### 10. LAZY vs EAGER and N+1
| Association | JPA default |
|---|---|
| `@OneToMany` | `LAZY` |
| `@ManyToMany` | `LAZY` |
| `@ManyToOne` | `EAGER` |
| `@OneToOne` | `EAGER` |

LAZY requests deferred loading; it is a JPA hint whose exact implementation can vary. EAGER requires the association to be available, but does not require a single SQL join.
Hibernate often implements lazy to-one references using proxies/enhancement. The REST module deliberately overrides the many-to-one default to LAZY.

```text
Query all users                      1 SQL statement
Loop: user.getPosts().size()
  load posts for User#1               1 SQL statement
  load posts for User#2               1 SQL statement
  ...
  load posts for User#N               1 SQL statement
Total                                1 + N
```

EAGER can also produce N+1 through secondary selects. Switching every relationship to EAGER is not a reliable fix. Loading lazy state after context closure can cause
`LazyInitializationException`. Fetch the required data inside a service transaction and map it to a suitable result DTO.

Illustrative JPQL fetch plan for the REST entities:

```java
@Query("select distinct u from user_details u left join fetch u.posts")
List<User> findAllWithPosts();
```

The JPQL entity name is `user_details` because of the actual `@Entity(name=...)`. `LEFT JOIN FETCH` keeps users with no posts; an inner join would omit them. `distinct` expresses
entity-level uniqueness; Hibernate 6 also handles many fetch-join duplicates. A fetch join loads associations, unlike an ordinary join used only for filtering.

Illustrative entity graph alternative:

```java
@EntityGraph(attributePaths = "posts")
@Query("select u from user_details u")
List<User> findUsersWithPosts();
```

An entity graph describes attributes to fetch for that operation. The provider chooses SQL; inspect it rather than assuming one query in all cases. Batch fetching can reduce N+1 to
grouped secondary queries. DTO projections can avoid materializing a complete graph when only summary fields are needed.

Do not paginate a collection fetch join casually. SQL rows represent parent-child combinations, not one row per parent. Hibernate may paginate in memory or reject the operation
with fail-fast configuration. A common fix is paging parent IDs, then fetching those parents and their children separately.

### 11. `save`, `saveAndFlush`, and transaction commit
`save` decides between `persist` and `merge` based on new-entity detection. With no suitable version property, a null ID—or zero for supported primitive numeric IDs—usually means
new. Custom `Persistable.isNew()` can alter that decision. Always use the returned entity when merge may occur.

| Operation | Meaning | Does it commit? |
|---|---|---|
| `save(entity)` | Persist or merge entity state | Not by itself inside an outer transaction |
| `flush()` | Synchronize pending context changes with database | No |
| `saveAndFlush(entity)` | Save, then flush the persistence context | No |
| Commit | Finish transaction successfully | Yes, normally flushes first |

A repository call without an outer transaction may commit when its own transactional method returns. That is transaction completion, not a special promise built into the word
`save`. `saveAndFlush` flushes pending changes in the context, not necessarily only its argument. SQL execution does not imply durability while the surrounding transaction remains
uncommitted. IDENTITY generation can trigger an insert before the usual flush point. Queries can also cause an automatic flush under AUTO flush mode.

Use an explicit flush to surface a constraint failure earlier when necessary. Do not flush after every row just to "make sure": it reduces batching opportunities.

### 12. Repository interfaces in Spring Data 3
| Interface | Main purpose |
|---|---|
| `Repository<T, ID>` | Marker/base abstraction |
| `CrudRepository<T, ID>` | Basic CRUD; iterable multi-result operations |
| `ListCrudRepository<T, ID>` | CRUD variant returning lists for multi-result operations |
| `PagingAndSortingRepository<T, ID>` | Pageable/sorted reads |
| `ListPagingAndSortingRepository<T, ID>` | List-oriented sorted reads |
| `JpaRepository<T, ID>` | JPA CRUD, paging/sorting, flush, batch-related operations |

**Version-sensitive interview point:** since Spring Data 3, `PagingAndSortingRepository` does not itself extend `CrudRepository`. If you build a generic repository needing both,
extend both appropriate interfaces. `JpaRepository` already combines relevant list-based CRUD and paging abstractions. `saveAll` is not a guarantee of one SQL statement or JDBC
batching.

Illustrative pagination extension:

```java
Page<Course> findByAuthor(String author, Pageable pageable);
```

Illustrative caller:

```java
Pageable request = PageRequest.of(0, 10, Sort.by("name").ascending().and(Sort.by("id")));
Page<Course> page = repository.findByAuthor("in28minutes", request);
```

Page indexes start at **0**. A `Page` provides totals, usually requiring a count query when totals cannot be inferred. A `Slice` provides whether another slice exists without
requiring a total count. A pageable `List` can limit results without page metadata. Use deterministic ordering; add ID as a tie-breaker when names repeat. Validate client-supplied
page sizes and allowed sort fields. Deep offset pagination can become expensive; keyset/scroll approaches may be better at scale.

### 13. `ddl-auto` and database choice
| Value | Effect | Suitable context |
|---|---|---|
| `none` | No Hibernate schema management | Explicit schema ownership |
| `validate` | Check schema compatibility; fail if incompatible | Migrated environments |
| `update` | Attempt schema adjustment | Convenient local learning, not migration governance |
| `create` | Recreate schema on startup | Disposable databases |
| `create-drop` | Create and drop at shutdown | Disposable tests/development |

These settings control schema, not automatic business data migrations. `update` cannot reliably plan every destructive rename, backfill, or rollout requirement. Use versioned
migrations for important databases. A successful `validate` is not proof that every index or business constraint is ideal.

| H2 | MySQL |
|---|---|
| Embedded/in-memory options | Separate server process |
| Very fast local feedback | Closer to MySQL production behavior |
| Data often disposable | Persistent server storage when configured |
| Own SQL dialect and type behavior | MySQL dialect, collation, engine semantics |
| Useful small tests | Better compatibility tests for MySQL deployments |

H2's MySQL compatibility mode is not an actual MySQL server. It cannot reproduce every locking, SQL, collation, timezone, or identifier behavior. Use integration tests against the
intended engine when those details matter. A MySQL JDBC URL requires the MySQL driver and a reachable database with credentials. SQL initialization defaults differ: external
databases do not normally run `data.sql` automatically.

### 14. CommandLineRunner, caches, and concurrency
`CommandLineRunner.run(String... args)` is called after context refresh during startup. It is useful for demonstrations and controlled one-time initialization. `ApplicationRunner`
instead receives parsed `ApplicationArguments`. Multiple runners can be ordered using `@Order` or `Ordered`. Uncaught runner exceptions can fail application startup. Do not use
every-start demo inserts as production migration machinery.

| Cache | Scope | Default/requirements |
|---|---|---|
| First-level | Persistence context | Always inherent to entity management |
| Second-level | EntityManagerFactory/session factory | Optional; requires deliberate provider/configuration |
| Query cache | Query results/identifiers | Separate optional feature, not implied by L2 |

Second-level cache can share entity state across contexts. It does not replace the database or make every query disappear. External JDBC writes can cause stale cached state unless
invalidation is coordinated. Cache concurrency strategies must match the consistency needs of the domain. This module does not configure a second-level cache.

Illustrative optimistic locking extension:

```java
@Version
private Long version;
```

```text
Reader A sees Course#2 version 7
Reader B sees Course#2 version 7
A updates WHERE id=2 AND version=7 -> row changed; version becomes 8
B updates WHERE id=2 AND version=7 -> zero rows; optimistic conflict
```

The provider manages the version value. A stale update can raise `OptimisticLockException`, commonly translated by Spring into an optimistic-locking data-access exception. The
error may appear at flush/commit rather than exactly at `save`. Handle conflicts with a clear message, merge policy, or carefully bounded retry. Do not blindly retry a
non-idempotent operation. Optimistic locking is not a replacement for a unique constraint or all transaction isolation requirements.

## Common pitfalls / gotchas
| Pitfall | Why it happens | Better approach |
|---|---|---|
| Treating JPA and Hibernate as synonyms | Specification confused with implementation | Name the API and provider separately |
| Assuming Course IDs are generated | No `@GeneratedValue` exists | Recognize assigned-ID workflow |
| Assuming `merge` attaches its argument | It returns a managed copy | Keep the return value |
| Calling `remove` after a missing `find` | `find` returns null | Handle not-found explicitly |
| Calling JDBC `queryForObject` expecting null | Single-row contract throws on absence | Define a deliberate optional lookup |
| Trusting `schema.sql` as final schema | Hibernate can also manage DDL | Choose one schema owner |
| Using Spring attributes on Jakarta `@Transactional` | Similar name, different API | Check the import |
| Internal call to transactional method | Proxy bypass | Use a bean boundary or transaction template |
| Expecting rollback on every exception | Checked exceptions are excluded by default | Configure exact rollback requirements |
| Thinking flush equals commit | SQL can still roll back | Explain transaction lifetime |
| Setting all relationships EAGER | Secondary selects/overfetching | Use operation-specific fetch plans |
| Collection fetch join with pagination | Joined rows distort parent paging | Page IDs, then fetch graph |
| Returning entities directly to REST clients | Lazy loading, cycles, overexposure | Use intentional DTOs |
| Expecting cascade from `mappedBy` | Ownership and lifecycle are distinct | Configure cascade only when justified |
| Treating `saveAll` as SQL batching | API iteration != JDBC batch | Configure and measure batching |
| Assuming H2 proves MySQL compatibility | Different engines | Test relevant behavior on MySQL |
| Using `ddl-auto=update` as production governance | Not versioned migration planning | Flyway/Liquibase |
| Assuming repository calls are one atomic workflow | Each may own its transaction | Add service transaction boundary |
| Updating via JDBC while entity is managed | Context retains old object state | Refresh/clear and coordinate caches |
| Sharing EntityManager across threads | Context is not thread-safe | Independent managed transaction boundaries |

When SQL surprises you, inspect these three facts first:

1. Is the entity new, managed, or detached?
2. Which transaction and persistence context are active?
3. What SQL actually executed, and when did flush happen?

## Interview questions & answers
### Basic
#### Q1. What is JDBC?
**Answer:** The standard Java API for interacting with relational databases through drivers, connections, prepared statements, and result sets. It is the layer underneath this
module's ORM stack.

#### Q2. How does Spring JDBC improve JDBC?
**Answer:** `JdbcTemplate` handles repetitive resource and exception work while you retain SQL control. You still define queries and row mapping, as `CourseJdbcRepository`
demonstrates.

#### Q3. What is JPA?
**Answer:** A specification for Java persistence: entity mapping, EntityManager operations, queries, and lifecycle rules. Hibernate is a provider that implements it.

#### Q4. What is Spring Data JPA?
**Answer:** A repository abstraction over JPA that generates CRUD and query-method implementations. `CourseSpringDataJpaRepository` supplies an interface rather than CRUD
implementation code.

#### Q5. What does `@Entity` do?
**Answer:** It declares that a class participates in persistence mapping. It does not declare the class a Spring-managed singleton service.

#### Q6. What identifies a Course row?
**Answer:** The `id` field annotated `@Id`. The runner assigns IDs explicitly; this entity does not request generated IDs.

#### Q7. Why does Course have a no-argument constructor?
**Answer:** The provider needs a way to instantiate the entity during loading. A public/protected no-argument constructor is part of the portable JPA entity contract.

#### Q8. What does `@PersistenceContext` inject?
**Answer:** A container-managed EntityManager reference, typically a Spring shared proxy that delegates to the context associated with the current transaction.

#### Q9. What does `EntityManager.find` return for a missing ID?
**Answer:** `null`. In contrast, Spring Data `findById` returns `Optional.empty()` and the current JDBC single-row lookup throws when no row exists.

#### Q10. What is a derived finder method?
**Answer:** A method whose property names and keywords define its query. `findByAuthor(String author)` filters Course entities using their `author` property.

#### Q11. What does the startup runner demonstrate?
**Answer:** Three saves, a delete, individual and bulk reads, a count, and derived queries. With a fresh successful run, two courses remain after ID 1 is deleted.

#### Q12. Is H2 console the database itself?
**Answer:** No. It is a browser interface for opening a JDBC connection. Its JDBC URL must match the application's database, or you may inspect a different database.

### Intermediate
#### Q13. Explain transient, managed, and detached states.
**Answer:** Transient objects are new and untracked; managed objects belong to an active context; detached objects no longer do. Only managed changes are automatically
dirty-checked.

#### Q14. How do `persist` and `merge` differ?
**Answer:** `persist` makes a new instance managed. `merge` copies state into a managed instance and returns it; the supplied instance may remain detached. This module's custom
`insert` uses merge.

#### Q15. Must a managed entity be saved after every setter?
**Answer:** Not inside a suitable write transaction. Dirty checking schedules changes for flush. An explicit save is usually unnecessary for a managed entity loaded in the same
transaction.

#### Q16. What does `saveAndFlush` guarantee?
**Answer:** It saves and synchronizes pending context changes to the database. It does not independently commit an outer transaction, and later rollback remains possible.

#### Q17. What is the default transaction rollback rule?
**Answer:** Spring rolls back for unchecked exceptions and errors by default, not checked exceptions. Spring's `rollbackFor` or Jakarta's `rollbackOn` can customize that rule.

#### Q18. Why does calling your own transactional method fail to start its transaction?
**Answer:** Normal Spring AOP interception happens at the proxy boundary. Self-invocation stays inside the target instance, bypassing the inner method's interceptor.

#### Q19. Compare `REQUIRED` and `REQUIRES_NEW`.
**Answer:** REQUIRED joins an existing transaction or starts one. REQUIRES_NEW suspends an existing transaction and starts an independent one, often needing another connection.

#### Q20. What are JPQL names based on?
**Answer:** Entity names and Java attribute names, not raw tables/columns. The REST `User` entity uses JPQL name `user_details` because its `@Entity` specifies that name.

#### Q21. What does `mappedBy="user"` mean?
**Answer:** The inverse side points to the owning Java property `Post.user`. The many-to-one side controls the foreign-key association; `mappedBy` is not a SQL column name.

#### Q22. What are the defaults for one-to-many and many-to-one fetching?
**Answer:** One-to-many defaults LAZY; many-to-one defaults EAGER. The REST Post explicitly sets many-to-one LAZY, and EAGER does not guarantee a single join query.

#### Q23. What changed in PagingAndSortingRepository in Spring Data 3?
**Answer:** It no longer inherits CRUD operations itself. Combine the appropriate interfaces when needed; JpaRepository already provides CRUD and paging support.

#### Q24. How do Page and Slice differ?
**Answer:** Page includes total-count metadata, often requiring a count query. Slice exposes whether more results exist without requiring an overall count.

### Advanced and scenario-based
#### Q25. A list of 100 users triggers 101 SQL statements. What would you inspect?
**Answer:** Look for one root query followed by per-user association initialization: N+1. Use an explicit fetch join, entity graph, batch fetching, or DTO query and verify the new
SQL count.

#### Q26. Will making all associations EAGER fix N+1?
**Answer:** No. EAGER can load associations through secondary selects and overfetch unrelated data. Choose a fetch plan for the actual use case rather than changing every default.

#### Q27. A controller accesses posts after the service returns and fails. Why?
**Answer:** A lazy association may require an EntityManager whose context is already closed. Fetch and map the needed data inside the transaction; do not fix the design by blindly
keeping it open.

#### Q28. Can a collection fetch join be safely combined with any pageable query?
**Answer:** No. Joined rows multiply parents, so database pagination can truncate collections or force in-memory paging. Page parent IDs first, then fetch their associations.

#### Q29. Two users edit the same course. How do you avoid silent lost updates?
**Answer:** Add a provider-managed `@Version` and handle optimistic-lock conflicts. The version predicate causes a stale write to fail rather than overwrite a newer version
silently.

#### Q30. Why can a successful `save` still be followed by a constraint exception?
**Answer:** SQL or constraint checking may happen at flush/commit rather than at the method call. An early flush can surface errors, but it still does not make the transaction
committed.

#### Q31. A checked import exception leaves partial data. What went wrong?
**Answer:** The transaction boundary may be missing, split across calls, or using default rollback rules. Wrap the business operation in one intercepted transaction and declare the
checked rollback requirement.

#### Q32. An inner REQUIRED method fails, the caller catches it, and commit fails. Why?
**Answer:** The shared transaction may already be rollback-only. Catching the exception does not clear that marker; outer completion can raise UnexpectedRollbackException.

#### Q33. Why might schema.sql constraints disappear in this module?
**Answer:** Boot's embedded-database default can let Hibernate recreate schema after script initialization. Choose a single schema authority and align entity metadata with explicit
migrations/scripts.

#### Q34. A JDBC update succeeds, but the next entity getter shows old data. Why?
**Answer:** A managed entity's first-level cached state was not changed by the JDBC write. Refresh/clear appropriately and consider second-level cache invalidation when external
writes are used.

#### Q35. Does `saveAll` guarantee efficient batch inserts?
**Answer:** No. It is a repository API, not a batching promise. Generation strategy, JDBC batch settings, flush cadence, and driver support determine actual batching.

#### Q36. Is a first-level cache shared across all application users?
**Answer:** No. It belongs to a persistence context. An optional second-level cache can share state across contexts but requires a deliberate consistency strategy.

#### Q37. Why can changing AUTO to IDENTITY alter performance on MySQL?
**Answer:** IDENTITY needs database insertion to obtain an identifier, limiting normal insert batching. AUTO's Hibernate 6 strategy may use a different allocation mechanism;
measure before changing it.

#### Q38. Would you use `ddl-auto=update` for a zero-downtime production deployment?
**Answer:** Not as the migration plan. Use reviewed, versioned, backward-compatible migrations with appropriate validation and rollout sequencing; update is convenient for
disposable learning environments.

#### Q39. How would you test this module's repositories?
**Answer:** Test derived finders and write/read behavior with focused JPA tests, flushing/clearing when needed. Use the target database for dialect/locking-sensitive tests;
transaction rollback can otherwise hide commit-time errors.

#### Q40. A finder returns several courses but the UI expects one. Where is the bug?
**Answer:** The method's `List` contract permits multiple matches, and no unique name constraint exists. Either enforce uniqueness as a business/database invariant or design the UI
to handle multiple results.

## Hands-on exercises
These are suggested learner changes, not changes already made to the module. Use a disposable database and keep the original examples available for comparison.

### Exercise 1: compare missing-row behavior
1. Look up an existing course through JDBC, custom JPA, and Spring Data.
2. Repeat each lookup with ID 9999.
3. Record exception, `null`, and `Optional.empty()` differences.
4. Define a consistent service-layer not-found policy.

**Acceptance:** a test explicitly covers a missing ID for each API. Explain why identical method names do not imply identical return contracts.

### Exercise 2: add a paginated course search
1. Add a pageable finder by author.
2. Sort by name and ID for deterministic results.
3. Insert enough sample courses for multiple pages.
4. Compare Page and Slice query counts.

**Acceptance:** page 0 and page 1 contain the expected non-overlapping ordered IDs. A zero-match author yields an empty result without a controller/server error.

### Exercise 3: demonstrate dirty checking and rollback
1. Create a separate service bean with Spring's `@Transactional`.
2. Load a Course and change its name without calling save.
3. Verify the update after transaction completion.
4. Throw an unchecked exception and verify rollback.
5. Repeat with a checked exception and an explicit rollback rule.

**Acceptance:** SQL and follow-up reads match the chosen rollback policy. Call the service through injection, not through self-invocation.

### Exercise 4: optimistic locking
1. Add a nullable `Long` version property annotated `@Version`.
2. Create the schema deliberately using one schema owner.
3. Load the same course in two separate transactions/contexts.
4. Commit one edit, then try to commit the stale edit.

**Acceptance:** the stale write fails and does not overwrite the committed name. Document whether the conflict occurred during flush or commit.

### Exercise 5: inspect the REST relationship fetch plan
1. Use the existing User/Post mapping in `12-rest-api/` with disposable data.
2. Read multiple users and iterate through their posts inside a transaction.
3. Count SQL statements with logging or Hibernate statistics.
4. Add an explicit fetch query or entity graph and repeat.
5. Explain the separate strategy needed for parent pagination.

**Acceptance:** demonstrate a reduced query count while retaining users with no posts. Do not claim that EAGER alone is the optimization.

## Quick revision cheat sheet
| Remember | Short answer |
|---|---|
| JDBC | Low-level database API |
| Spring JDBC | SQL + JdbcTemplate + RowMapper |
| JPA | Specification/API |
| Hibernate | ORM provider |
| Spring Data JPA | Repository abstraction over JPA |
| Current runner | Spring Data implementation only |
| Course ID | Assigned primitive long; no generation annotation |
| Custom JPA insert | Calls merge and ignores managed return value |
| Missing lookup | JDBC exception / JPA null / Spring Data Optional.empty |
| Persistence context | Tracks identity and managed state |
| persist | Manage a new entity |
| merge | Copy into and return a managed entity |
| Dirty checking | Managed mutations flushed in transaction |
| Default rollback | RuntimeException and Error |
| Proxy trap | Self-invocation bypasses inner advice |
| REQUIRED | Join/create transaction |
| REQUIRES_NEW | Independent transaction |
| Flush | SQL synchronization, not commit |
| JPQL | Entity names + Java properties |
| Native query | Tables + columns + database syntax |
| Owning relationship | Actual Post.user |
| mappedBy | Name of owning Java property |
| To-many default | LAZY |
| To-one default | EAGER |
| N+1 | Root query plus per-row association queries |
| Fetch fixes | Fetch join, graph, batching, projection |
| Pagination caution | Collection fetch joins multiply rows |
| Page vs Slice | Totals vs only next-slice information |
| Spring Data 3 | PagingAndSortingRepository is not a CRUD subtype |
| ddl-auto | Choose one schema authority |
| H2 vs MySQL | Different SQL engines, not interchangeable proofs |
| L1 cache | Persistence-context scope |
| L2 cache | Optional cross-context state cache |
| @Version | Detect stale concurrent writes |

**Thirty-second answer:** this module moves from explicit SQL and row mapping to EntityManager operations and finally a generated repository interface. Hibernate implements JPA
underneath. The abstraction reduces boilerplate, but transaction boundaries, entity state, schema ownership, fetch plans, and concurrency still require deliberate design.
