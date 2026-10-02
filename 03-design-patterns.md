# 03 — Design Patterns & Architecture

Selected findings: **DP1, DP2, DP3, DP4, DP7** · selected questions: **D1, D2, D5, D7**.

**Section verdict (from the review):** excellent *tactical* patterns (optimistic concurrency, idempotency
key, append-only audit log, pure rule functions). Weak *strategic* architecture: dependencies point the
wrong way, the state machine is implicit, the domain can't act, there are no events, and **no boundary is
enforced**. The architecture style ("no layers") was the candidate's explicit constraint; the concrete
shape was drafted by AI and accepted.

The constraint, as written by the candidate:

`NOTCLAUDE.md` — lines 18–23

```markdown
18  ## Guiding constraint
19  
20  The brief asks for **a small, clear solution with thoughtful decisions over a
21  broad but fragile implementation**. Scope discipline is a graded property here,
22  not a shortcut. Do not add dependencies, layers, or abstractions beyond what is
23  listed below without being asked.
```

…and repeated as a rule for the AI's design step:

`openspec/config.yaml` — lines 103–103

```yaml
103          Do not introduce a layer (service/repository/mapper/DTO seam) unless the change cannot be expressed without it, and state why in that case.
```

---

## DP1. The database layer depends on the web layer

### Evidence

The repository imports the HTTP status type…

`src/main/java/com/example/basis/request/ExceptionRequestStore.java` — lines 16–21

```java
16  import org.springframework.dao.DuplicateKeyException;
17  import org.springframework.http.HttpStatus;
18  import org.springframework.jdbc.core.simple.JdbcClient;
19  import org.springframework.stereotype.Repository;
20  import org.springframework.transaction.annotation.Transactional;
21  import org.springframework.transaction.support.TransactionTemplate;
```

…builds the **HTTP response DTO** (`RequestBodies.View`) and persists it together with an HTTP status code:

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

The value object returned to the controller carries a status code:

`src/main/java/com/example/basis/request/CreateOutcome.java` — lines 5–12

```java
 5  /**
 6   * The answer to a create, whether it created anything or replayed a create that
 7   * already happened. The body travels as the stored bytes so that a replay
 8   * returns what the first caller received rather than a response rebuilt from
 9   * current state.
10   */
11  public record CreateOutcome(UUID requestId, int statusCode, String responseBody, boolean replayed) {
12  }
```

The response-shape factory takes Jackson's `ObjectMapper` (domain + wire + serialisation mixed):

`src/main/java/com/example/basis/request/RequestBodies.java` — lines 111–122

```java
111       * A request together with the trail that produced it, newest first. The
112       * two travel in one response so that a client cannot render a request
113       * beside a history that has moved on since.
114       */
115      public record Detail(
116              View request,
117              List<Entry> history) {
118  
119          public static Detail of(ExceptionRequest request, List<HistoryEntry> history, ObjectMapper json) {
120              return new Detail(View.of(request), history.stream().map(entry -> Entry.of(entry, json)).toList());
121          }
122      }
```

Origin — the AI-drafted design chose "built from the value the endpoint returns":

`openspec/changes/archive/2026-09-20-idempotent-create/design.md` — lines 82–87

```markdown
82  ### The idempotency record is written last, from the response that is returned
83  
84  Ordering within the transaction: request row, history entry, idempotency
85  record. The record is built from the same value the endpoint returns, so
86  "the stored response is the response the caller received" is true by
87  construction rather than by two pieces of code agreeing.
```

### Why it matters

A change to the API's JSON shape changes what the database stores. The persistence adapter cannot be reused
by anything that is not HTTP. It is deliberate (byte-exact replay), but in hexagonal terms the dependency is
inverted — and storing `HttpStatus` in the repository is not justified by "keep it small".

### Fix

Keep byte-exact replay **at the web edge**: an idempotency filter/interceptor stores the serialized response
it actually wrote, as an opaque blob. The repository persists domain facts only; status mapping lives in the
web adapter.

---

## DP2. The state machine is implicit

### Evidence

The specified state machine (candidate's CLAUDE.md):

`NOTCLAUDE.md` — lines 139–149

````text
139  ## State machine
140  
141  ```
142  PENDING --amend--> PENDING (version + 1)
143  PENDING --approve--> APPROVED
144  PENDING --decline--> DECLINED
145  PENDING --withdraw--> WITHDRAWN
146  APPROVED | DECLINED | WITHDRAWN --> (terminal)
147  ```
148  
149  Illegal transitions return `409`.
````

In code, the legal transitions live as **string literals inside three SQL statements**:

`src/main/java/com/example/basis/request/ExceptionRequestStore.java` — lines 160–167

```java
160          int rows = db.sql("""
161                  UPDATE exception_request
162                     SET discount_bps = COALESCE(?, discount_bps),
163                         reason       = COALESCE(?, reason),
164                         version      = version + 1,
165                         updated_at   = ?
166                   WHERE id = ? AND version = ? AND status = 'PENDING'
167                  """)
```

`src/main/java/com/example/basis/request/ExceptionRequestStore.java` — lines 193–197

```java
193          int rows = db.sql("""
194                  UPDATE exception_request
195                     SET status = ?, decided_by = ?, decided_at = ?, updated_at = ?
196                   WHERE id = ? AND version = ? AND status = 'PENDING'
197                  """)
```

`src/main/java/com/example/basis/request/ExceptionRequestStore.java` — lines 219–223

```java
219          int rows = db.sql("""
220                  UPDATE exception_request
221                     SET status = 'WITHDRAWN', updated_at = ?
222                   WHERE id = ? AND version = ? AND status = 'PENDING'
223                  """)
```

The enum has a transition function — but its own Javadoc says it **"does not gate any write"**:

`src/main/java/com/example/basis/request/RequestStatus.java` — lines 7–29

```java
 7  public enum RequestStatus {
 8  
 9      PENDING,
10      APPROVED,
11      DECLINED,
12      WITHDRAWN;
13  
14      public boolean isTerminal() {
15          return this != PENDING;
16      }
17  
18      /**
19       * Whether a transition is legal from this state.
20       *
21       * <p>This does not gate any write. The version and status guards live
22       * inside the UPDATE statement, because they race. This answers why a
23       * guarded write matched no row, once the row has been re-read: a terminal
24       * request explains the rejection differently from a stale version.
25       */
26      public boolean allows(EntryType transition) {
27          return this == PENDING && transition != EntryType.CREATED;
28      }
29  }
```

Usage in production code (`grep -rn ".allows(\|isTerminal()" src/main`):

```
ExceptionRequestStore.java:353:  if (!current.status().allows(attempted)) {      <- error-message wording only
```

`isTerminal()` is **never called from production code** — only from `RequestStatusTest`.

The task that produced it was satisfied literally (`request-lifecycle/tasks.md` line 23):

> 2.2 Add the request record carrying current state, and a function answering which transitions are legal
> from a given status.

### Fix

One explicit model — e.g. `EnumMap<RequestStatus, Set<Transition>>` or the State pattern — and derive the
SQL guard from it (`WHERE status IN (:allowedFrom)`), plus a test that asserts every SQL guard matches the
table. The model becomes the single answer to "what can happen from PENDING?".

---

## DP3. The domain object doesn't change itself

### Evidence

The whole domain record — it can **refuse**, but it cannot **approve, amend or withdraw**:

`src/main/java/com/example/basis/request/ExceptionRequest.java` — lines 16–60

```java
16  public record ExceptionRequest(
17          UUID id,
18          String applicationId,
19          int discountBps,
20          String reason,
21          RequestStatus status,
22          int version,
23          String requestedBy,
24          String decidedBy,
25          Instant decidedAt,
26          Instant createdAt,
27          Instant updatedAt) {
28  
29      /**
30       * Four-eyes is checked before the role, so that someone declining their
31       * own request is told the useful thing rather than that they lack a role.
32       */
33      public Optional<Refusal> refuseDecisionBy(String actor, UserRole role) {
34          if (requestedBy.equals(actor)) {
35              return Optional.of(new Refusal(RefusalReason.SELF_DECISION,
36                      "A request cannot be decided by the person who raised it."));
37          }
38          if (role != UserRole.REVIEWER) {
39              return Optional.of(new Refusal(RefusalReason.NOT_REVIEWER,
40                      "Deciding a request requires the " + UserRole.REVIEWER + " role."));
41          }
42          return Optional.empty();
43      }
44  
45      public Optional<Refusal> refuseAmendBy(String actor) {
46          return refuseUnlessRequester(actor, "amended");
47      }
48  
49      public Optional<Refusal> refuseWithdrawalBy(String actor) {
50          return refuseUnlessRequester(actor, "withdrawn");
51      }
52  
53      private Optional<Refusal> refuseUnlessRequester(String actor, String action) {
54          if (requestedBy.equals(actor)) {
55              return Optional.empty();
56          }
57          return Optional.of(new Refusal(RefusalReason.NOT_REQUESTER,
58                  "A request may only be " + action + " by the person who raised it."));
59      }
60  }
```

Every actual transition is an SQL `UPDATE` in the store, e.g. amend:

`src/main/java/com/example/basis/request/ExceptionRequestStore.java` — lines 155–185

```java
155      @Transactional
156      public ExceptionRequest amend(UUID id, int version, String actor, Integer discountBps, String reason) {
157          refuseUnless(require(id).refuseAmendBy(actor));
158  
159          Instant now = Instant.now();
160          int rows = db.sql("""
161                  UPDATE exception_request
162                     SET discount_bps = COALESCE(?, discount_bps),
163                         reason       = COALESCE(?, reason),
164                         version      = version + 1,
165                         updated_at   = ?
166                   WHERE id = ? AND version = ? AND status = 'PENDING'
167                  """)
168                  .params(discountBps, reason, at(now), id, version)
169                  .update();
170  
171          if (rows == 0) {
172              throw classify(id, EntryType.AMENDED);
173          }
174  
175          Map<String, Object> carried = new LinkedHashMap<>();
176          if (discountBps != null) {
177              carried.put("discountBps", discountBps);
178          }
179          if (reason != null) {
180              carried.put("reason", reason);
181          }
182          appendHistory(id, EntryType.AMENDED, version + 1, actor, payload(carried), now);
183  
184          return require(id);
185      }
```

### Why it matters

The domain behaviour ("amending bumps the version, keeps unchanged fields, records an AMENDED entry")
exists only in SQL (`COALESCE`, `version + 1`) and can only be tested with a database. The candidate's
"guards in the write" invariant explains the *guard*, not why the *new state* isn't computed by the domain.

### Fix

`ExceptionRequest amend(Amendment a, String actor, Instant now)` returns the new state (and the history
event); the repository persists it with `UPDATE … SET … WHERE id = ? AND version = ? AND status = 'PENDING'`.
Guard stays in SQL, behaviour moves to the domain, unit-testable without Spring.

---

## DP4. No domain events, no messaging (key gap for this role)

### Evidence

A decision writes the row and a history entry — nothing else leaves the service:

`src/main/java/com/example/basis/request/ExceptionRequestStore.java` — lines 187–212

```java
187      @Transactional
188      public ExceptionRequest decide(UUID id, int version, String actor, UserRole role,
189                                     Decision decision, String note) {
190          refuseUnless(require(id).refuseDecisionBy(actor, role));
191  
192          Instant now = Instant.now();
193          int rows = db.sql("""
194                  UPDATE exception_request
195                     SET status = ?, decided_by = ?, decided_at = ?, updated_at = ?
196                   WHERE id = ? AND version = ? AND status = 'PENDING'
197                  """)
198                  .params(decision.status().name(), actor, at(now), at(now), id, version)
199                  .update();
200  
201          if (rows == 0) {
202              throw classify(id, decision.entryType());
203          }
204  
205          Map<String, Object> carried = new LinkedHashMap<>();
206          if (note != null && !note.isBlank()) {
207              carried.put("note", note);
208          }
209          appendHistory(id, decision.entryType(), version, actor, payload(carried), now);
210  
211          return require(id);
212      }
```

The consumer (the mortgage process) must **poll**:

`src/main/java/com/example/basis/request/ApprovedExceptionController.java` — lines 32–44

```java
32      @GetMapping("/{id}/approved-exception")
33      public RequestBodies.ApprovedException approvedException(
34              @PathVariable String id,
35              @RequestHeader("X-User-Id")
36              @Size(max = RequestLimits.MAX_IDENTITY_LENGTH,
37                    message = "must be at most {max} characters")
38              String userId,
39              @RequestHeader("X-User-Role") UserRole role) {
40  
41          return store.approvedExceptionFor(id)
42                  .map(RequestBodies.ApprovedException::of)
43                  .orElseThrow(() -> new ApprovedExceptionNotFoundException(id));
44      }
```

Events were explicitly rejected in the design and declared out of scope:

`openspec/changes/archive/2026-09-20-request-lifecycle/design.md` — lines 77–82

```markdown
77  ### The history entry is written by the same method that made the transition
78  
79  Not by an event listener, an `@EntityListener`, or an aspect. Invariant 3 makes
80  history the audit record; a mechanism that could be silently bypassed or fail
81  independently would undermine it. Two statements in one transactional method,
82  in the obvious order, is the form that can be read and verified.
```

`NOTCLAUDE.md` — lines 374–378

```markdown
374  ## Out of scope — state in the README, do not build
375  
376  Real authentication (OIDC/Entra ID), Postgres + Flyway, Testcontainers, event
377  publishing to downstream consumers via a transactional outbox, persistent
378  idempotency store with TTL, observability.
```

### Why it matters

The role asks for **events, messaging and distributed systems**; the repo contains none, so the interview
has to supply the evidence. Lever: the `request_history` insert already happens in the same transaction as
the state change — it is ~80% of a transactional outbox.

### Question to ask (bridges to D5)

"Today the mortgage process polls you. Design it so it is told when an exception is approved."
Look for: `ExceptionApproved` domain event; outbox row in the same transaction; relay (Debezium/CDC or
poller) to Kafka/RabbitMQ; at-least-once + idempotent consumer; partition by `applicationId` for ordering;
versioned event schema; supersession as its own event.

---

## DP7. No enforced boundaries (upgraded to a top finding)

### Evidence

Every type, in two packages, all `public` (output of a scan of the first type declaration per file):

```
BasisApplication.java                            public class
ProblemDetailAdvice.java                         public class
request/ApprovedExceptionController.java         public class     <- web
request/ExceptionRequestController.java          public class     <- web
request/RequestBodies.java                       public final class <- web DTOs
request/ExceptionRequest.java                    public record    <- domain
request/RequestStatus.java                       public enum      <- domain
request/Decision.java                            public enum      <- domain
request/EntryType.java                           public enum      <- domain
request/UserRole.java                            public enum      <- domain
request/Refusal.java                             public record    <- domain
request/RefusalReason.java                       public enum      <- domain
request/RequestLimits.java                       public final class <- domain
request/ExceptionRequestStore.java               public class     <- persistence + tx + authz
request/HistoryEntry.java                        public record    <- persistence
request/IdempotencyRecord.java                   public record    <- persistence
request/CreateOutcome.java                       public record    <- persistence/web
request/ConflictException.java                   public class     <- errors (x7)
request/RefusedException.java                    public class
request/KeyReusedException.java                  public class
request/EmptyAmendmentException.java             public class
request/RequestNotFoundException.java            public class
request/ApprovedExceptionNotFoundException.java  public class
```

No application/use-case layer: controllers call the store directly. Domain exceptions are specified in
HTTP terms — e.g. `ConflictException` Javadoc: *"Answers 409"*; `RefusalReason`: *"All of these answer 403"*;
`UserRole`: *"Carried by the X-User-Role header"*.

Nothing enforces anything — no package-private types, no ArchUnit, no Spring Modulith, no Maven modules —
so the layer leaks in DP1 compile without complaint.

### Target structure

```
request/
  domain/                  ExceptionRequest, RequestStatus, Decision, Refusal, RequestLimits,
                           domain exceptions (no HTTP), ports: ExceptionRequestRepository, IdempotencyStore
  application/             CreateRequest, AmendRequest, DecideRequest, WithdrawRequest, RequestQueries
                           (@Transactional, Clock)
  adapter/in/web/          controllers, RequestBodies, ProblemDetailAdvice (domain -> HTTP), Caller resolver,
                           idempotent-replay filter
  adapter/out/persistence/ JdbcExceptionRequestRepository, row mappers, payload codec
```

Enforce with package-private defaults + an ArchUnit test, e.g.
`noClasses().that().resideInAPackage("..domain..").should().dependOnClassesThat().resideInAPackage("org.springframework..")`,
or Spring Modulith verification, or Maven modules.

### Expected defence and how to answer it

*"CLAUDE.md said no layers — it's graded."* Acceptable for three tables. Not acceptable for `HttpStatus`
inside the repository (costs nothing to fix), and a senior engineer should still sketch the target and say
**when** it becomes worth it.

---

## D1. Name the patterns you used here, and one you considered and rejected.

### Evidence of what is actually there (to check their vocabulary against)

| Pattern | Where |
|---|---|
| Optimistic concurrency (optimistic offline lock) | `ExceptionRequestStore.java` 160–173 — version + status in the `WHERE`, `rows == 0` → 409 |
| Idempotency key / idempotent receiver | `ExceptionRequestStore.java` 66–142, PK `("key", caller)` |
| Append-only audit log | `request_history`, insert-only (`schema.sql` 21–34) |
| Pure rule functions returning a verdict | `ExceptionRequest.refuse*` → `Optional<Refusal>` |
| Enum carrying behaviour | `Decision` → (`RequestStatus`, `EntryType`) |
| Static factory / DTO assembler | `View.of`, `Detail.of`, `ApprovedException.of` |
| Centralised error handling | `ProblemDetailAdvice` (RFC 7807) |
| Template method (framework) | `TransactionTemplate`, `ResponseEntityExceptionHandler` overrides |

Rejected alternatives the candidate documented (good material for follow-ups): JPA/Hibernate and Lombok
(README / CLAUDE.md), event listener for history (lifecycle design), fingerprint column (idempotency
design), `spring.mvc.problemdetails.enabled` (rejection-surface design), generated sequence column for
history ordering (queries design).

### What a good answer contains

Precise names, not "repository and controller". At least one real rejected option with a reason that is
theirs — cross-check against the design docs above.

---

## D2. Look at `ExceptionRequestStore:97`: your persistence layer builds the HTTP response. Which way do your dependencies point, and how would you restructure for hexagonal/clean architecture while keeping byte-exact replays?

### Evidence

See DP1 — lines 17 and 94–106 of `ExceptionRequestStore.java`, `CreateOutcome.statusCode`,
`RequestBodies.Detail.of(..., ObjectMapper)`.

### What a good answer contains

- Dependencies should point **inward**: web and persistence adapters depend on the domain/application, never
  the reverse.
- Ports & adapters: `ExceptionRequestRepository` port in the domain, JDBC adapter outside.
- Byte-exact replay moves to the web edge (filter/interceptor stores the written bytes as an opaque blob).
- Honest trade-off: today the response and the business write commit atomically; moving replay out of the
  transaction needs an in-flight marker or outbox-style reconciliation.

**Follow-up (DP7):** "Everything is public in one package. How would you *enforce* these boundaries — and what
would you still keep simple at this size?"
**Follow-up (AI5):** "Your CLAUDE.md forbids layers because it's graded. If this weren't a take-home, what would
the instruction say, and would the code look different?"
**Red flag:** "The AI / the brief did it this way" with no view of their own.

---

## D5. Your history table is append-only. Is that event sourcing? Would you go there for this domain?

### Evidence

Current-state table + history table, and the candidate's own description of the duplication:

`NOTCLAUDE.md` — lines 104–110

```markdown
104  ## Data model
105  
106  Two tables with different jobs. `exception_request` holds **what is true now** —
107  one row per request, updated in place — and is what the queue and the mortgage
108  process read. `request_history` holds **how it got there**: one row per
109  transition, `INSERT` only, never `UPDATE`d or `DELETE`d. The duplication is
110  deliberate; the current row is a cache of where the log ended up.
```

`src/main/resources/schema.sql` — lines 21–34

```sql
21  -- How it got there: one row per transition, INSERT only. Never UPDATEd or
22  -- DELETEd -- see invariant 3. The version column records the version the entry
23  -- concerned, which is not always the request's current version.
24  CREATE TABLE IF NOT EXISTS request_history (
25      id          UUID         NOT NULL PRIMARY KEY,
26      request_id  UUID         NOT NULL REFERENCES exception_request (id),
27      entry_type  VARCHAR(16)  NOT NULL,
28      version     INT          NOT NULL,
29      actor       VARCHAR(128) NOT NULL,
30      payload     VARCHAR(1024),
31      occurred_at TIMESTAMP WITH TIME ZONE NOT NULL
32  );
33  
34  CREATE INDEX IF NOT EXISTS idx_request_history_request_id ON request_history (request_id);
```

"Append-only" is not enforced by the database — the candidate's design acknowledges it:

`openspec/changes/archive/2026-09-20-request-lifecycle/design.md` — lines 107–112

```markdown
107  **Append-only is a convention, not a constraint.** Nothing in H2 prevents an
108  `UPDATE` or `DELETE` against `request_history`; only code discipline does. → No
109  code path writes anything but an `INSERT`, and this is the first thing to check
110  if audit integrity is ever questioned. In production the application's database
111  role would lack `UPDATE` and `DELETE` on that table, which is where the
112  guarantee belongs.
```

### What a good answer contains

- No: this is **state + audit log**. The row in `exception_request` is the source of truth; history is
  derived/parallel. In event sourcing the events *are* the source of truth and state is a projection.
- Event sourcing costs: projections, replay, snapshotting, event schema evolution, GDPR erasure.
- Reasoned view: likely overkill for a four-state workflow; an outbox (DP4) gets the integration benefit
  without it. Strong candidates mention enforcing append-only via DB grants/triggers.

---

## D7. The four-eyes rule is written in two places. How do you keep a business rule single-sourced when it has to apply both to a write and to a query?

### Evidence

Place 1 — the domain rule used on the write path:

`src/main/java/com/example/basis/request/ExceptionRequest.java` — lines 29–43

```java
29      /**
30       * Four-eyes is checked before the role, so that someone declining their
31       * own request is told the useful thing rather than that they lack a role.
32       */
33      public Optional<Refusal> refuseDecisionBy(String actor, UserRole role) {
34          if (requestedBy.equals(actor)) {
35              return Optional.of(new Refusal(RefusalReason.SELF_DECISION,
36                      "A request cannot be decided by the person who raised it."));
37          }
38          if (role != UserRole.REVIEWER) {
39              return Optional.of(new Refusal(RefusalReason.NOT_REVIEWER,
40                      "Deciding a request requires the " + UserRole.REVIEWER + " role."));
41          }
42          return Optional.empty();
43      }
```

Place 2 — the same rule as SQL in the reviewer queue:

`src/main/java/com/example/basis/request/ExceptionRequestStore.java` — lines 291–307

```java
291      /**
292       * The reviewer's queue: requests raised by someone else. Four-eyes lives
293       * in the WHERE clause, so no client can display a request its caller
294       * raised -- the API never returns one. Absent a status the queue holds
295       * what there is to review; a status given explicitly replaces that
296       * default, so a reviewer can look back at what was decided.
297       */
298      public List<ExceptionRequest> queueForReviewer(String reviewer, RequestStatus status) {
299          RequestStatus wanted = status == null ? RequestStatus.PENDING : status;
300          return db.sql(SELECT_COLUMNS + """
301                   WHERE requested_by <> ? AND status = ?
302                   ORDER BY created_at DESC
303                  """)
304                  .params(reviewer, wanted.name())
305                  .query(ExceptionRequestStore::toRequest)
306                  .list();
307      }
```

The AI-drafted lifecycle design promised this would never happen — the queries slice (written later the same
day) broke it:

`openspec/changes/archive/2026-09-20-request-lifecycle/design.md` — lines 35–49

```markdown
35  ### Guards are enforced in SQL; rules are evaluated in Java; they do not overlap
36  
37  The two are split by what they protect against, not by preference.
38  
39  *Enforced in the statement:* version match and current status. These race, so
40  they go in the `WHERE` clause of the guarded `UPDATE`, and `rowsAffected == 0`
41  is the rejection signal.
42  
43  *Evaluated in Java:* the four-eyes rule, the role check, ownership of an amend
44  or withdrawal, and field validation. None of these races — they depend on the
45  caller and the submitted body, not on concurrent state — so evaluating them
46  before the write costs nothing and produces a far better error message.
47  
48  The two sets are disjoint. No rule is checked in both places, which is what
49  would otherwise invite them to drift apart.
```

The queue filter itself was mandated by the candidate's CLAUDE.md API table ("four-eyes filtering happens
here, server-side"), so the duplication follows from two of their own requirements.

### What a good answer contains

- Specification pattern: one `FourEyes` spec that yields both an in-memory predicate and a SQL fragment /
  criteria, so both uses derive from one definition.
- Or: accept the duplication consciously and pin the two together with a test that runs the same cases
  through both paths.
- Bonus: notices that their own design document claims the sets are disjoint, and explains how the drift
  went unnoticed (ties to the AI section).
