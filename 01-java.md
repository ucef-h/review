# 01 — Java

Selected: questions **Q2, Q3, Q7** · findings **F1, F2, F4**.

**Section verdict (from the review):** strong. Concurrency, idempotency and transaction handling are
correct and non-trivial. Weaknesses are in class responsibilities, exception design and testability.
Because the mechanics appear mostly in AI-drafted designs and code comments, Q2 doubles as an
**ownership check**: can the candidate explain it without notes?

---

## Q2. Why is `create` using `TransactionTemplate` instead of `@Transactional`?

### Evidence

The create path opens its transaction programmatically and catches the duplicate key **outside** it:

`src/main/java/com/example/basis/request/ExceptionRequestStore.java` — lines 52–74

```java
52      /**
53       * Creates a request, or replays the create this key already stands for.
54       *
55       * <p>The transaction is opened explicitly rather than with an annotation
56       * because the duplicate key has to be caught <em>outside</em> it: the
57       * violation poisons the transaction, so the stored response cannot be read
58       * until it has rolled back. A self-invoked annotated method would not be
59       * proxied and would not be transactional at all.
60       *
61       * <p>Waiting for that rollback is safe. H2 holds the second insert until
62       * the transaction owning the key ends -- if it committed, the record is
63       * there to replay; if it rolled back, this insert succeeds and this caller
64       * is the one that created the request.
65       */
66      public CreateOutcome create(String applicationId, int discountBps, String reason,
67                                  String actor, String idempotencyKey) {
68          try {
69              return transactions.execute(status ->
70                      insertNew(applicationId, discountBps, reason, actor, idempotencyKey));
71          } catch (DuplicateKeyException alreadyUsed) {
72              return replay(idempotencyKey, actor, applicationId, discountBps, reason);
73          }
74      }
```

Every other write path uses the annotation, so the difference is deliberate:

`src/main/java/com/example/basis/request/ExceptionRequestStore.java` — lines 155–157

```java
155      @Transactional
156      public ExceptionRequest amend(UUID id, int version, String actor, Integer discountBps, String reason) {
157          refuseUnless(require(id).refuseAmendBy(actor));
```

The AI-drafted design records *why* the replay read must happen in a new transaction, but never names
`TransactionTemplate` or self-invocation — that choice appears **only in the code comment**:

`openspec/changes/archive/2026-09-20-idempotent-create/design.md` — lines 112–118

```markdown
112  a caller is told a key is taken but cannot be shown what it produced.
113  
114  The consequence for the code: the violation poisons the transaction, so the
115  replay read happens in a *new* one, after rollback. That differs from the
116  existing `classify()` on the amend and decide paths, where the failed `UPDATE`
117  leaves the transaction usable and the re-read happens inside it. Same shape,
118  different reason — worth stating because the two will sit next to each other.
```

The candidate's CLAUDE.md only says *"insert against the (key, caller) unique constraint and catch the
violation"* (`NOTCLAUDE.md` lines 44–46) — it does not specify the mechanism.

### What a good answer contains

- Spring `@Transactional` works through a proxy; a call from `create` to another method **in the same
  class** (self-invocation) bypasses the proxy, so an annotated helper would not be transactional.
- A unique-constraint violation marks the transaction rollback-only ("poisoned"); the stored record can
  only be read **after** rollback, i.e. outside the transaction that failed.
- `TransactionTemplate` makes the boundary explicit so the `catch (DuplicateKeyException)` sits outside it.
- Bonus: alternatives — a separate bean with `@Transactional(propagation = REQUIRES_NEW)`, or
  `INSERT ... ON CONFLICT DO NOTHING` on Postgres and checking the row count.

**Red flag:** cannot explain self-invocation, or says "Claude chose it".

---

## Q3. On Postgres instead of H2, does the duplicate-key replay still behave correctly? What if the first transaction is slow or rolls back?

### Evidence

Replay path — reads the stored record and compares the original create fields:

`src/main/java/com/example/basis/request/ExceptionRequestStore.java` — lines 109–128

```java
109      /**
110       * Answers a create whose key is already taken. The comparison is against
111       * the stored response, not against the request row: the row moves when the
112       * request is amended, and a retry of the original create must still be
113       * recognised as that create rather than told it differs.
114       */
115      private CreateOutcome replay(String key, String caller,
116                                   String applicationId, int discountBps, String reason) {
117          IdempotencyRecord stored = findRecord(key, caller).orElseThrow(() ->
118                  // Unreachable: the insert above only fails once the transaction
119                  // holding this key has committed. Failing loudly beats
120                  // inventing a response for a state that cannot occur.
121                  new IllegalStateException(
122                          "Idempotency key '" + key + "' was taken but no record was stored"));
123  
124          if (!isSameCreate(stored, applicationId, discountBps, reason)) {
125              throw new KeyReusedException(key);
126          }
127          return new CreateOutcome(stored.requestId(), stored.statusCode(), stored.responseBody(), true);
128      }
```

The guard is the composite primary key:

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

**Important:** the candidate's design doc already answers this question **for H2**, "established by
experiment". Expect a polished answer and push on Postgres:

`openspec/changes/archive/2026-09-20-idempotent-create/design.md` — lines 97–111

```markdown
 97  ### A losing retry replays after its own transaction rolls back
 98  
 99  Established by experiment, not assumed. Two connections at READ_COMMITTED
100  inserting the same primary key on H2 2.3.232:
101  
102  - The second insert **blocks** until the first transaction ends. It does not
103    fail immediately.
104  - If the first **commits**, the second then fails with `23505`, and after
105    rolling back it **can read the committed record** — so there is always
106    something to replay.
107  - If the first **rolls back**, the second insert **succeeds**, and that caller
108    becomes the creator.
109  
110  This is what makes the contract answerable without inventing a status code for
111  "the request you are retrying is still in flight". There is no window in which
```

The race is tested — two identical creates released together by a barrier:

`src/test/java/com/example/basis/request/IdempotentCreateTest.java` — lines 208–235

```java
208      @Test
209      @DisplayName("two retries of one create submitted together produce exactly one request")
210      void twoRetriesOfOneCreateRace() throws Exception {
211          String application = "APP-IDEM-RACE";
212          String key = UUID.randomUUID().toString();
213  
214          List<MvcResult> outcomes = runTogether(
215                  submitCreate(key, application),
216                  submitCreate(key, application));
217  
218          assertThat(outcomes)
219                  .as("neither caller was handed a server error")
220                  .allSatisfy(result -> assertThat(result.getResponse().getStatus()).isLessThan(500));
221  
222          assertThat(requestsFor(application))
223                  .as("only one of the two retries created a request")
224                  .isEqualTo(1);
225  
226          UUID id = onlyRequestFor(application);
227          assertThat(typesInHistoryOf(id))
228                  .as("exactly one CREATED entry exists for it")
229                  .containsExactly(EntryType.CREATED);
230  
231          assertThat(outcomes)
232                  .as("a caller answered 2xx was told about the request that was created")
233                  .filteredOn(result -> result.getResponse().getStatus() < 300)
234                  .allSatisfy(result -> assertThat(idOf(result)).isEqualTo(id));
235      }
```

Note the test runs on H2 only (no Testcontainers — declared as deliberately not built).

### What a good answer contains

- Postgres also **blocks** the second insert on the unique index until the first transaction ends:
  commit → `23505 unique_violation` → Spring `DuplicateKeyException` (vendor-neutral translation);
  rollback → the second insert succeeds and that caller becomes the creator.
- Slow first transaction: the second caller **waits** holding a connection — consider `lock_timeout`,
  statement timeouts, and pool exhaustion under retry storms.
- Would re-run the experiment on Postgres (Testcontainers) rather than assume H2 parity.
- Bonus: in-flight marker / `409 Retry-After` instead of blocking; records need a TTL (declared omission).

---

## Q7. If you had Java 21, what would you change in this code?

### Evidence

The project pins Java 17, justified as *"Runs on the reviewer's machine without a JDK upgrade"*
(`NOTCLAUDE.md` line 88):

`pom.xml` — lines 19–25

```xml
19  
20      <properties>
21          <java.version>17</java.version>
22          <!-- springdoc is not managed by the Spring Boot BOM, so its version is
23               pinned here. The 2.x line targets Boot 3; 3.x targets Boot 4. -->
24          <springdoc.version>2.9.1</springdoc.version>
25      </properties>
```

Code that Java 21 features would reshape — the error mapping is one handler per exception type:

`src/main/java/com/example/basis/ProblemDetailAdvice.java` — lines 143–181

```java
143      @ExceptionHandler(EmptyAmendmentException.class)
144      public ProblemDetail onEmptyAmendment(EmptyAmendmentException e) {
145          return problem(HttpStatus.BAD_REQUEST, "Invalid request", e.getMessage());
146      }
147  
148      @ExceptionHandler(RefusedException.class)
149      public ProblemDetail onRefused(RefusedException e) {
150          return problem(HttpStatus.FORBIDDEN, "Not permitted", e.refusal().detail());
151      }
152  
153      /**
154       * The key is well-formed; what it already stands for is the problem. That
155       * is 422 rather than 409: nothing about the request's state conflicts, and
156       * resubmitting the same body against the same key will never succeed.
157       */
158      @ExceptionHandler(KeyReusedException.class)
159      public ProblemDetail onKeyReused(KeyReusedException e) {
160          return problem(HttpStatus.UNPROCESSABLE_ENTITY, "Idempotency key already used", e.getMessage());
161      }
162  
163      @ExceptionHandler(RequestNotFoundException.class)
164      public ProblemDetail onNotFound(RequestNotFoundException e) {
165          return problem(HttpStatus.NOT_FOUND, "No such request", e.getMessage());
166      }
167  
168      @ExceptionHandler(ApprovedExceptionNotFoundException.class)
169      public ProblemDetail onNoApprovedException(ApprovedExceptionNotFoundException e) {
170          return problem(HttpStatus.NOT_FOUND, "No approved exception", e.getMessage());
171      }
172  
173      @ExceptionHandler(ConflictException.class)
174      public ProblemDetail onConflict(ConflictException e) {
175          ProblemDetail problem = problem(HttpStatus.CONFLICT, "Request has moved on", e.getMessage());
176          if (e.current() != null) {
177              problem.setProperty("currentVersion", e.current().version());
178              problem.setProperty("currentStatus", e.current().status());
179          }
180          return problem;
181      }
```

### What a good answer contains

- A **sealed** exception or result hierarchy (`sealed interface Outcome permits Success, Refused,
  Conflict, NotFound`) with an **exhaustive pattern-matching `switch`** in the advice — the compiler
  then catches a new failure type that has no HTTP mapping.
- Record patterns to destructure results; `switch` on `RequestStatus` for the transition table.
- Virtual threads: honest answer — JDBC is blocking; throughput is bounded by the connection pool, not by
  platform threads, so gains are limited without pool tuning.
- **Follow-up:** sealed classes already exist in Java 17 — why weren't they used? (Links to F2.)

---

## F1. One class does six jobs — `ExceptionRequestStore` (430 lines)

### Evidence

Annotated `@Repository`, but it is the whole application layer:

| Responsibility | Lines |
|---|---|
| SQL statements | 81–104, 160–231, 236–322 |
| Transaction boundaries | 69, 155, 187, 214, 328 |
| Authorisation (`refuseUnless`) | 157, 190, 216, 337–341 |
| Idempotency replay + comparison | 115–153 |
| JSON serialisation | 372–385 |
| Error classification | 343–360 |
| Row mappers | 391–429 |

Authorisation and transaction control inside the "repository":

`src/main/java/com/example/basis/request/ExceptionRequestStore.java` — lines 187–191

```java
187      @Transactional
188      public ExceptionRequest decide(UUID id, int version, String actor, UserRole role,
189                                     Decision decision, String note) {
190          refuseUnless(require(id).refuseDecisionBy(actor, role));
191  
```

HTTP-shaped error classification inside the "repository":

`src/main/java/com/example/basis/request/ExceptionRequestStore.java` — lines 343–360

```java
343      /**
344       * Decides what to say about a guarded write that matched no row. The write
345       * has already failed by this point, so re-reading here cannot reintroduce
346       * the race the guard exists to prevent.
347       */
348      private RuntimeException classify(UUID id, EntryType attempted) {
349          ExceptionRequest current = find(id).orElse(null);
350          if (current == null) {
351              return new RequestNotFoundException(id);
352          }
353          if (!current.status().allows(attempted)) {
354              return new ConflictException(current,
355                      "This request is " + current.status() + " and can no longer be changed.");
356          }
357          return new ConflictException(current,
358                  "This request has moved on and is now at version " + current.version()
359                          + ". Reload it and review the current version rather than resubmitting.");
360      }
```

The design that produced it (AI-drafted, accepted by the candidate under the CLAUDE.md "no layers"
constraint):

`openspec/changes/archive/2026-09-20-request-lifecycle/design.md` — lines 63–75

```markdown
63  ### Two classes on the write path, not three
64  
65  `ExceptionRequestController` handles HTTP. `ExceptionRequestStore` holds the
66  SQL and owns the transaction. There is no service layer between them.
67  
68  A transaction boundary is a real requirement — the guarded `UPDATE` and the
69  `request_history` `INSERT` must commit together — but it does not need a class
70  of its own. Each store method is one transactional unit corresponding to one
71  transition, which is the same granularity a service layer would have had.
72  
73  The domain rules that a service layer would usually hold live on the records
74  and enums instead, as pure functions. That is what makes them unit-testable
75  with no Spring context, as the testing section requires.
```

### Fix

```java
@Service    class ExceptionRequestService   { /* @Transactional, authz, classify, Clock */ }
@Repository class ExceptionRequestRepository { /* SQL + row mappers only */ }
class IdempotencyService                     { /* insert-or-replay */ }
class HistoryPayloadCodec                    { /* JSON in/out */ }
```

### Question to ask

"When does keeping all of this in one class stop being acceptable? What would you split first?"
Good answer: a reasoned threshold (second consumer of the rules, second persistence path, team size),
not dogma either way.

---

## F2. Exceptions used as control flow, with no hierarchy

### Evidence

A clean `Optional<Refusal>` is converted into a throw, and `classify` *returns* an exception for the
caller to throw:

`src/main/java/com/example/basis/request/ExceptionRequestStore.java` — lines 333–341

```java
333      private ExceptionRequest require(UUID id) {
334          return find(id).orElseThrow(() -> new RequestNotFoundException(id));
335      }
336  
337      private static void refuseUnless(Optional<Refusal> refusal) {
338          refusal.ifPresent(r -> {
339              throw new RefusedException(r);
340          });
341      }
```

Six unrelated `RuntimeException` subclasses; the HTTP status lives only in Javadoc:

`src/main/java/com/example/basis/request/ConflictException.java` — lines 1–19

```java
 1  package com.example.basis.request;
 2  
 3  /**
 4   * The write matched no row: the request had moved on. Answers 409, carrying
 5   * the state found when the failure was classified.
 6   */
 7  public class ConflictException extends RuntimeException {
 8  
 9      private final transient ExceptionRequest current;
10  
11      public ConflictException(ExceptionRequest current, String detail) {
12          super(detail);
13          this.current = current;
14      }
15  
16      public ExceptionRequest current() {
17          return current;
18      }
19  }
```

`src/main/java/com/example/basis/request/RefusedException.java` — lines 1–16

```java
 1  package com.example.basis.request;
 2  
 3  /** The caller may not do this, on the evidence of who they are. Answers 403. */
 4  public class RefusedException extends RuntimeException {
 5  
 6      private final transient Refusal refusal;
 7  
 8      public RefusedException(Refusal refusal) {
 9          super(refusal.detail());
10          this.refusal = refusal;
11      }
12  
13      public Refusal refusal() {
14          return refusal;
15      }
16  }
```

Each needs its own handler; forgetting one compiles fine and yields a `500`
(see `ProblemDetailAdvice.java` lines 143–181 in Q7 above).

Not specified in any spec, design or CLAUDE.md — **an implementation-time (AI) choice**.

### Fix

```java
public sealed abstract class DomainException extends RuntimeException
        permits ConflictException, RefusedException, NotFoundException, KeyReusedException {
    protected DomainException(String message) { super(message); }
    public abstract HttpStatus status();   // or a transport-neutral Kind enum mapped in the web adapter
    public abstract String title();
}

@ExceptionHandler(DomainException.class)
ProblemDetail on(DomainException e) { return problem(e.status(), e.title(), e.getMessage()); }
```

Better: the service returns a sealed result (`Success | Refused | Conflict | NotFound`) and the
controller maps it with an exhaustive `switch` (Java 21) or `instanceof` chain (Java 17).

---

## F4. Hard-wired time and ID generation

### Evidence

`Instant.now()` and `UUID.randomUUID()` called inline:

`src/main/java/com/example/basis/request/ExceptionRequestStore.java` — lines 76–79

```java
76      private CreateOutcome insertNew(String applicationId, int discountBps, String reason,
77                                      String actor, String idempotencyKey) {
78          UUID id = UUID.randomUUID();
79          Instant now = Instant.now();
```

`src/main/java/com/example/basis/request/ExceptionRequestStore.java` — lines 362–370

```java
362      private void appendHistory(UUID requestId, EntryType type, int version, String actor,
363                                 String payload, Instant when) {
364          db.sql("""
365                  INSERT INTO request_history (id, request_id, entry_type, version, actor, payload, occurred_at)
366                  VALUES (?, ?, ?, ?, ?, ?, ?)
367                  """)
368                  .params(UUID.randomUUID(), requestId, type.name(), version, actor, payload, at(when))
369                  .update();
370      }
```

(also `Instant.now()` at lines 159, 192, 218)

The design doc explicitly identifies clock granularity as a problem — and solves it with a four-level
`ORDER BY` instead of a controllable clock:

`openspec/changes/archive/2026-09-20-queries/design.md` — lines 43–51

```markdown
43  The clock alone does not separate that pair. `Instant.now()` is granular to a
44  millisecond or worse on some hosts, and an amendment followed by the decision
45  reviewing it can land in one tick, which made the order of the two a coin-toss
46  decided by a random UUID — the integration test written for this scenario
47  caught it. So the transition itself breaks the tie: a terminal entry sorts
48  after the `CREATED` or `AMENDED` entry of the same version, and the state
49  machine guarantees there is at most one such pair, because a version admits one
50  decision and a request that has been decided admits no further entries. That
51  makes the order total on domain facts rather than on timing.
```

The test then **spin-waits on the real clock** instead of injecting a fixed one:

`src/test/java/com/example/basis/request/RequestQueriesTest.java` — lines 268–283

```java
268      /**
269       * Two approvals a fraction of a millisecond apart share a {@code
270       * decided_at} on a host whose clock ticks in milliseconds, and which of
271       * them is "most recent" is then a coin-toss -- a tie design.md records as
272       * accepted rather than guarded, because two approvals on one application
273       * are minutes apart in practice and the guard that would fix it properly
274       * is the SUPERSEDED status this service deliberately does not model.
275       * Waiting for the clock to move keeps this test an assertion about the
276       * rule rather than about the tie.
277       */
278      private static void waitForTheClockToTick() {
279          Instant start = Instant.now();
280          while (!Instant.now().isAfter(start)) {
281              Thread.onSpinWait();
282          }
283      }
```

### Fix

```java
@Bean Clock clock() { return Clock.systemUTC(); }
@Bean Supplier<UUID> ids() { return UUID::randomUUID; }

public ExceptionRequestStore(JdbcClient db, ObjectMapper json, TransactionTemplate tx,
                             Clock clock, Supplier<UUID> ids) { ... }

Instant now = Instant.now(clock);
```

Tests use `Clock.fixed(...)` / a mutable test clock, making the same-millisecond tie reproducible
instead of avoided.

### Question to ask

"Your design says ties within a millisecond happen, and your test spins until the clock ticks. How would
you test that tie deterministically?" Good answer: inject `Clock`, fix it in the test, assert the
tiebreak order directly.
