# 02 — Spring

Selected: **S3, S4, S5**.

**Section verdict (from the review):** strong command of Spring MVC/JDBC internals (validation routing,
`ResponseEntityExceptionHandler`, transaction proxies). Weak on production readiness — no security,
observability, migrations or real-database testing. Most of that is **declared** in the README, so the
questions test design depth, not awareness.

The candidate's own declaration of what was not built:

`README.md` — lines 433–449

```markdown
433  ## Deliberately not built
434  
435  Named so that their absence reads as a decision rather than an omission.
436  
437  - **Real authentication** (OIDC / Entra ID). Identity is two headers.
438  - **Postgres and Flyway.** H2 in-memory with `schema.sql` means the service
439    runs with nothing but a JDK.
440  - **Testcontainers.** The same reasoning.
441  - **Event publishing to downstream consumers** via a transactional outbox.
442  - **A persistent idempotency store with a TTL.** Records live in the same
443    in-memory database and vanish on restart, so a retry arriving after a restart
444    creates a second request. Nothing expires or is swept.
445  - **Observability.** No metrics, no tracing. See the note on the `409` rate
446    below for the one number that would be worth watching first.
447  - **JPA/Hibernate** (dirty checking works against immutability, and mapping
448    buys nothing at three tables), **Lombok** (records suffice), **Spring
449    Security** (identity is stubbed), and any JavaScript framework.
```

---

## S3. Deploy this to Kubernetes against Postgres tomorrow. What breaks?

### Evidence

**1. `schema.sql` only runs against an embedded database.** Point the datasource at Postgres and no
tables are created:

`src/main/resources/application.properties` — lines 1–19

```properties
 1  spring.application.name=basis
 2  
 3  # In-memory only. DB_CLOSE_DELAY keeps the database alive between connections
 4  # for the lifetime of the JVM; without it H2 discards it when the pool closes
 5  # its last connection.
 6  spring.datasource.url=jdbc:h2:mem:basis;DB_CLOSE_DELAY=-1
 7  spring.datasource.driver-class-name=org.h2.Driver
 8  spring.datasource.username=sa
 9  spring.datasource.password=
10  
11  # schema.sql runs automatically for an embedded datasource.
12  spring.sql.init.mode=embedded
13  
14  springdoc.swagger-ui.path=/swagger-ui.html
15  
16  # A field the endpoint does not document is rejected rather than ignored, so
17  # that an attempt to change something immutable fails loudly instead of
18  # appearing to succeed.
19  spring.jackson.deserialization.fail-on-unknown-properties=true
```

Note also lines 6–9: datasource URL and credentials hard-coded, no environment placeholders, no
secret handling.

**2. H2-only SQL.** An untyped `NULL` parameter in `? IS NULL` fails on Postgres
(`could not determine data type of parameter $2`):

`src/main/java/com/example/basis/request/ExceptionRequestStore.java` — lines 309–322

```java
309      /**
310       * The requester's own requests, in every state unless one is asked for.
311       * The status is optional in the statement rather than in two statements,
312       * so there is one query to read and one plan to reason about.
313       */
314      public List<ExceptionRequest> queueForRequester(String requester, RequestStatus status) {
315          return db.sql(SELECT_COLUMNS + """
316                   WHERE requested_by = ? AND (? IS NULL OR status = ?)
317                   ORDER BY created_at DESC
318                  """)
319                  .params(requester, status == null ? null : status.name(), status == null ? null : status.name())
320                  .query(ExceptionRequestStore::toRequest)
321                  .list();
322      }
```

The schema quotes a reserved word because H2 2.x reserves `KEY` (portable, but a tell that the schema
was shaped around H2):

`src/main/resources/schema.sql` — lines 36–49

```sql
36  -- Idempotent create. The (key, caller) primary key IS the guard: the create
37  -- path inserts against it and catches the violation -- see invariant 7. H2 2.x
38  -- reserves KEY as a keyword, so the column CLAUDE.md names is quoted rather
39  -- than renamed; every statement touching it quotes it too. Scoped by caller so
40  -- that two relationship managers generating the same key do not collide.
41  CREATE TABLE IF NOT EXISTS idempotency_record (
42      "key"         VARCHAR(128)  NOT NULL,
43      caller        VARCHAR(128)  NOT NULL,
44      request_id    UUID          NOT NULL REFERENCES exception_request (id),
45      response_body VARCHAR(4096) NOT NULL,
46      status_code   INT           NOT NULL,
47      created_at    TIMESTAMP WITH TIME ZONE NOT NULL,
48      PRIMARY KEY ("key", caller)
49  );
```

**3. No migrations, no Actuator, no logging.** The full dependency list — no Flyway/Liquibase, no
`spring-boot-starter-actuator`, no Postgres driver:

`pom.xml` — lines 27–57

```xml
27      <dependencies>
28          <dependency>
29              <groupId>org.springframework.boot</groupId>
30              <artifactId>spring-boot-starter-web</artifactId>
31          </dependency>
32          <dependency>
33              <groupId>org.springframework.boot</groupId>
34              <artifactId>spring-boot-starter-jdbc</artifactId>
35          </dependency>
36          <dependency>
37              <groupId>org.springframework.boot</groupId>
38              <artifactId>spring-boot-starter-validation</artifactId>
39          </dependency>
40          <dependency>
41              <groupId>org.springdoc</groupId>
42              <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
43              <version>${springdoc.version}</version>
44          </dependency>
45  
46          <dependency>
47              <groupId>com.h2database</groupId>
48              <artifactId>h2</artifactId>
49              <scope>runtime</scope>
50          </dependency>
51  
52          <dependency>
53              <groupId>org.springframework.boot</groupId>
54              <artifactId>spring-boot-starter-test</artifactId>
55              <scope>test</scope>
56          </dependency>
57      </dependencies>
```

`grep -rn "Logger|LoggerFactory|log.info|log.warn|log.error|actuator" src/main pom.xml` → **0 matches**.
A refused four-eyes attempt, a 409 conflict or an idempotent replay leaves no trace.

**4. State is in-process.** `jdbc:h2:mem:basis` means every pod has its own database; with
`replicas: 2` requests, approvals and idempotency keys split across pods and vanish on restart.

### What a good answer contains

- `schema.sql` / `spring.sql.init.mode=embedded` won't run → Flyway `V1__baseline.sql`, run as a
  pre-deploy Job / Helm hook, not at app startup.
- The `? IS NULL` query → two statements, or `CAST(? AS VARCHAR)`; re-run the suite on Postgres
  (Testcontainers).
- Actuator liveness/readiness probes; graceful shutdown; Micrometer metrics (the 409 rate the README
  itself calls out); structured logs for refusals, conflicts, replays.
- Secrets from a vault / External Secrets, injected by environment — nothing in `application.properties`.
- Stateless pods with Postgres as the single source of truth (bridges to distributed systems).

---

## S4. Your tests all use `@SpringBootTest` on one shared database. What's the cost, and how would you structure the suite?

### Evidence

7 of 11 test classes boot the full context:

```
SeedAbsentByDefaultTest      @SpringBootTest
SeedFixtureTest              @SpringBootTest @ActiveProfiles("dev")   <- second context
ConcurrentAmendmentTest      @SpringBootTest @AutoConfigureMockMvc
RejectionSurfaceTest         @SpringBootTest @AutoConfigureMockMvc
IdempotentCreateTest         @SpringBootTest @AutoConfigureMockMvc
RequestQueriesTest           @SpringBootTest @AutoConfigureMockMvc
ExceptionRequestApiTest      @SpringBootTest @AutoConfigureMockMvc
```

No `@WebMvcTest`, `@JdbcTest`, `@Transactional` rollback, `@Sql` cleanup, `@DirtiesContext` or
Testcontainers anywhere in `src/test`.

Isolation is by **naming convention** — each test picks its own application identifier, because the
in-memory schema is shared across the whole suite:

`src/test/java/com/example/basis/request/IdempotentCreateTest.java` — lines 28–41

```java
28  /**
29   * The replay contract on {@code POST /requests}, through HTTP and against the
30   * real H2 schema -- acceptance criterion 2.
31   *
32   * <p>Each test works against an application identifier of its own, because the
33   * schema is shared across the suite and "exactly one request was created" is
34   * counted rather than inferred. Requests and records are read back through the
35   * store and the database rather than through an endpoint: a replay must return
36   * the stored response and create nothing new, and both halves of that are
37   * claims about what is in the tables rather than about what a read serves.
38   */
39  @SpringBootTest
40  @AutoConfigureMockMvc
41  class IdempotentCreateTest {
```

The convention is not universal — `APP-1` is reused across tests in `ExceptionRequestApiTest`:

`src/test/java/com/example/basis/request/ExceptionRequestApiTest.java` — lines 60–64

```java
60      @DisplayName("a created request is PENDING at version 1 with a CREATED history entry")
61      void createRecordsAPendingRequestAtVersionOne() throws Exception {
62          MvcResult result = mvc.perform(withKey(asRm(REQUESTER, post("/requests")))
63                          .content("{\"applicationId\":\"APP-1\",\"discountBps\":25,\"reason\":\"" + REASON + "\"}"))
64                  .andExpect(status().isCreated())
```

(also lines 407 and 450 of the same file)

The full-context approach was **the candidate's explicit choice** (CLAUDE.md testing section):

`NOTCLAUDE.md` — lines 289–302

```markdown
289  ## Testing
290  
291  Two levels, chosen because the risk sits in two different places.
292  
293  - **Unit tests on the domain** — the state machine, the version guard, the
294    four-eyes rule, validation. Plain JUnit, no Spring context, fast. These cover
295    the rules that make the service correct.
296  - **Integration tests through the API** — `@SpringBootTest` with MockMvc,
297    against the real H2 schema. One per acceptance criterion 1–6, because the
298    interesting behaviour (idempotent replay, stale-version rejection) only
299    exists once persistence and HTTP are involved.
300  
301  No mocking of the repository: the queries are the thing being tested. UI
302  criteria 7–9 are verified manually; say so rather than implying coverage.
```

### What a good answer contains

- Cost: slow suite, two cached contexts (default + `dev`), hidden ordering dependencies because state
  leaks between tests; a test that counts rows can break when another test is added.
- Structure: pure unit tests for rules (already exist: `ExceptionRequestRulesTest`,
  `StaleVersionRefusalTest`, `RequestStatusTest`); `@WebMvcTest` for the HTTP surface/error mapping;
  `@JdbcTest` + Testcontainers Postgres for SQL and the concurrency guarantees; a small number of full
  `@SpringBootTest` end-to-end tests.
- Isolation: per-test transaction rollback where possible, truncate/`@Sql` for the concurrency tests
  (which can't run inside one transaction), and no reliance on naming conventions.

---

## S5. Idempotency lives inside `create`. How would you make it cross-cutting, and how does it behave behind a load balancer with several instances?

### Evidence

The idempotency key is read as a header on the one endpoint that needs it…

`src/main/java/com/example/basis/request/ExceptionRequestController.java` — lines 39–66

```java
39      /**
40       * The response body travels as the stored bytes rather than as an object,
41       * so that a replay returns what the first caller received rather than a
42       * response rebuilt from state that may have moved since.
43       */
44      @PostMapping
45      public ResponseEntity<String> create(
46              @RequestHeader("X-User-Id")
47              @Size(max = RequestLimits.MAX_IDENTITY_LENGTH,
48                    message = "must be at most {max} characters")
49              String userId,
50              @RequestHeader("X-User-Role") UserRole role,
51              @RequestHeader("Idempotency-Key")
52              @NotBlank
53              @Size(max = RequestLimits.MAX_IDEMPOTENCY_KEY_LENGTH,
54                    message = "must be at most {max} characters")
55              String idempotencyKey,
56              @Valid @RequestBody RequestBodies.Create body) {
57  
58          CreateOutcome outcome = store.create(
59                  body.applicationId(), body.discountBps(), body.reason(), userId, idempotencyKey);
60  
61          return ResponseEntity
62                  .status(outcome.statusCode())
63                  .location(URI.create("/requests/" + outcome.requestId()))
64                  .contentType(MediaType.APPLICATION_JSON)
65                  .body(outcome.responseBody());
66      }
```

…and the record is written inside the same transaction as the business insert, in the repository:

`src/main/java/com/example/basis/request/ExceptionRequestStore.java` — lines 94–106

```java
 94          // Written last, from the value the endpoint returns, so that "the
 95          // stored response is the response the caller received" holds by
 96          // construction rather than by two places agreeing.
 97          String responseBody = serialise(RequestBodies.View.of(require(id)));
 98          db.sql("""
 99                  INSERT INTO idempotency_record ("key", caller, request_id, response_body,
100                                                  status_code, created_at)
101                  VALUES (?, ?, ?, ?, ?, ?)
102                  """)
103                  .params(idempotencyKey, actor, id, responseBody, HttpStatus.CREATED.value(), at(now))
104                  .update();
105  
106          return new CreateOutcome(id, HttpStatus.CREATED.value(), responseBody, false);
```

The design deliberately rejected a filter/interceptor:

`openspec/changes/archive/2026-09-20-idempotent-create/design.md` — lines 120–129

```markdown
120  ### The key is read and validated where the other headers are
121  
122  `@RequestHeader("Idempotency-Key")` on the create method, beside `X-User-Id`
123  and `X-User-Role`, validated to the bounds in CLAUDE.md's table. A missing
124  header is already a `400` through the existing advice; blank and over-length
125  join the other validation failures. No filter, no interceptor: the endpoint
126  that requires the key is the endpoint that reads it.
127  
128  `caller` is the `X-User-Id` the create arrived with, which is what makes
129  `(key, caller)` the primary key rather than `key` alone.
```

Storage is the same in-memory H2 (per-instance) and never expires — the candidate declares this:
*"Records live in the same in-memory database and vanish on restart, so a retry arriving after a restart
creates a second request. Nothing expires or is swept."* (README lines 442–444)

### What a good answer contains

- Cross-cutting: a servlet filter / `HandlerInterceptor` or an `@Idempotent` annotation + aspect that
  stores the response for any POST, keyed by `(key, caller, method, path)` plus a request fingerprint.
- Shared storage for all instances — Postgres table or Redis (`SET NX` + TTL). In-memory per pod breaks
  as soon as the load balancer sends the retry to another instance.
- **In-flight requests:** first write a `PROCESSING` marker; a concurrent retry gets `409` + `Retry-After`
  (or waits), then the final response replaces the marker. The candidate's current design relies on the DB
  blocking the second insert — fine in one DB, not portable to Redis.
- Expiry chosen from client retry behaviour (e.g. 24 h), with a sweeper or native TTL.
- Trade-off of moving it out of the transaction: the business write and the stored response are no longer
  atomic — how do they reconcile a crash between them?
