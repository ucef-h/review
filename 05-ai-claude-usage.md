# 05 — AI (Claude usage)

Selected findings: **AI1–AI5** · questions **A2, A3, A5, A4** (AI5 is raised as a follow-up to D2).

**Section verdict (from the review):** strong *intent*, thin *tooling*. The candidate thinks carefully about
what the AI should do — concrete invariants, the "tests and implementation must not share a prompt" rule,
acceptance-criteria traceability. But everything relies on the model obeying prose: no hooks, subagents,
permission settings or CI integration; the context file is twice the recommended size and has already drifted
from the code.

### What is (and isn't) candidate-authored AI tooling

| Artefact | Authored by | Notes |
|---|---|---|
| `NOTCLAUDE.md` (their `CLAUDE.md`) | Candidate (claimed) | 394 lines, 19,260 bytes |
| `openspec/config.yaml` | Candidate | Context + per-phase rules for OpenSpec's planning skills |
| `notclaude/commands/opsx/*` (12), `notclaude/skills/openspec-*` (12) | **OpenSpec CLI** | Every skill: `author: openspec`, `generatedBy: "1.13.1"` — output of `openspec init` |
| `.claude/settings.json`, hooks, subagents, MCP config, own skills | — | **Not present** in the repository |

Evidence of generation (header of one generated skill):

`notclaude/skills/openspec-apply-change/SKILL.md` — lines 1–11

```yaml
 1  ---
 2  name: openspec-apply-change
 3  description: Implement tasks from an OpenSpec change. Use when the user wants to start implementing, continue implementation, or work through tasks. Also use when the user says "openspec apply", "opsx apply", or "openspec implement".
 4  allowed-tools: Bash(openspec:*)
 5  license: MIT
 6  compatibility: Requires openspec CLI.
 7  metadata:
 8    author: openspec
 9    version: "1.0"
10    generatedBy: "1.13.1"
11  ---
```

Reference — Anthropic's Claude Code documentation (*"How Claude remembers your project"*,
https://code.claude.com/docs/en/memory), quoted where relevant below.

---

## AI1. The CLAUDE.md is twice the recommended size

### Evidence

`wc -lc NOTCLAUDE.md` → **394 lines, 19,260 bytes**, loaded into every session.

Its sections — most are product specification, not working rules:

| Line | Section | Kind |
|---|---|---|
| 18 | Guiding constraint | working rule |
| 27 | Domain invariants — do not violate | working rule |
| 53 | The contested requirement, and how it is resolved | design decision (ADR) |
| 84 | Technology | design decision |
| 104 | Data model | spec |
| 139 | State machine | spec |
| 151 | API · Enumerations · Validation · Status codes | spec |
| 211 | User interface · Screens · Required behaviours (~54 lines) | spec |
| 266 | Acceptance criteria | spec |
| 289 | Testing | working rule |
| 306 | Deliverables | project plan |
| 330 | Assumptions (record in README) | spec / README content |
| 344 | Open questions for the business (record in README) | README content |
| 374 | Out of scope — state in the README, do not build | scope |
| 382 | Working agreement | working rule |

Roughly 60 of 394 lines are rules an agent needs in *every* session; the rest is spec that already exists in
`openspec/specs/` (78 scenarios).

Example — a full UI specification loaded even when the agent is editing SQL:

`NOTCLAUDE.md` — lines 211–223

```markdown
211  ## User interface
212  
213  A single page served by the service itself from
214  `src/main/resources/static/index.html`, reached at `/`. No controller, no
215  routing, no build step, no JavaScript framework. Pico.css is **vendored** under
216  `static/vendor/` rather than loaded from a CDN, so the page renders on a machine
217  with no network access; it is classless, so the markup stays semantic HTML with
218  no class names. All behaviour is plain JavaScript using `fetch` and the DOM API.
219  
220  The UI is a thin client over the same JSON API — it holds no business rules of
221  its own. Every invariant above is enforced server-side; the UI only reflects
222  what the API returns. Never implement a rule in JavaScript that the API does
223  not also enforce.
```

Anthropic guidance:

> **Size**: target under 200 lines per CLAUDE.md file. Longer files consume more context and reduce
> adherence. If your instructions are growing large, use path-scoped rules so instructions load only when
> Claude works with matching files.

> Keep it to facts Claude should hold in every session: build commands, conventions, project layout,
> "always do X" rules. If an entry is a multi-step procedure or only matters for one part of the codebase,
> move it to a skill or a path-scoped rule instead.

### A2. Your CLAUDE.md is almost 400 lines. What does that cost, and how would you restructure it?

Look for:
- Cost: context budget on every session, lower adherence, more chance of contradictions.
- Core file < 200 lines: invariants, build/test commands, working agreement.
- UI rules → `.claude/rules/ui.md` with `paths: ["src/main/resources/static/**"]`; SQL/persistence rules
  path-scoped to `schema.sql` / the store; procedures → skills.
- API, validation, status codes, acceptance criteria → `openspec/specs/` (already there) and referenced,
  not copied.

---

## AI2. Rules are written as prose, with nothing enforcing them

### Evidence

The most important rules are absolute — and only text:

`NOTCLAUDE.md` — lines 27–49

```markdown
27  ## Domain invariants — do not violate
28  
29  1. A request in a terminal state (`APPROVED`, `DECLINED`, `WITHDRAWN`) is
30     **immutable**. Never add a code path that mutates one.
31  2. A decision **binds to an exact version** of the request. Deciding against a
32     stale version is rejected with `409`.
33  3. Every state change appends a row to `request_history`. That table is
34     **append-only** — never `UPDATE` or `DELETE` it.
35  4. The requester may not decide their own request (four-eyes).
36  5. The discount is stored as a **positive integer in basis points**. Never a
37     float, never negative.
38  6. All timestamps are `Instant`, UTC.
39  7. **Guards are enforced in the write, not before it.** The version and status
40     checks live inside the statement:
41     `UPDATE exception_request SET ... WHERE id = ? AND version = ? AND status = 'PENDING'`,
42     and `rowsAffected == 0` means `409`. Never read the row, compare in Java,
43     then write — that passes every test and races under concurrency.
44     Idempotency follows the same rule: insert against the `(key, caller)` unique
45     constraint and catch the violation to return the stored response; never
46     "check if it exists, then insert".
47  
48  If a change appears to require breaking one of these, stop and raise it rather
49  than working around it.
```

`NOTCLAUDE.md` — lines 382–394

```markdown
382  ## Working agreement
383  
384  - Commit in small steps with honest messages; the history is part of what is reviewed.
385  - Comments explain **why**, not what.
386  - Do not add a dependency without stating the reason. This includes front-end
387    assets: nothing is loaded from a CDN, and nothing introduces a build step.
388  - The UI is built last, after the API and its tests are green. If time runs
389    short it is dropped and said so in the README — never at the cost of the
390    service or the decision record.
391  - Do not generate tests and implementation from the same prompt in one pass —
392    the tests would encode the same misreading as the code. Implement against the
393    acceptance criteria above, then review each test against its criterion.
394  - Anything an agent produces must be read and understood before it is committed.
```

The process claim that tests are written in a separate session is also only text, in an AI-generated task list:

`openspec/changes/archive/2026-09-20-request-lifecycle/tasks.md` — lines 1–7

```markdown
1  # Tasks
2  
3  Groups 1–4 are implementation. Groups 5 and 6 are verification and are written
4  in a **separate session**, from the scenarios in
5  `specs/exception-request/spec.md` and the numbered acceptance criteria in
6  CLAUDE.md — not from the code produced by groups 1–4. Tests and implementation
7  written in one pass would encode the same misreading twice.
```

What would enforce it — and is absent:

| Rule | Possible enforcement | Present? |
|---|---|---|
| Never `UPDATE`/`DELETE` `request_history` | PreToolUse hook rejecting such SQL; DB grants | No (design admits "a convention, not a constraint") |
| Never read-then-write | `ConcurrentAmendmentTest` + CI; review hook | Test exists, no CI |
| Tests not written from the implementation | Test-writer subagent without access to `src/main` | No subagents |
| Run the suite after edits | PostToolUse hook running `./mvnw -q test` | No hooks |
| Don't touch settled files | `permissions.deny` in `.claude/settings.json` | No settings file |
| Small honest commits | commit hook / PR checks | Unknown (no history) |

Anthropic guidance:

> Claude treats them as context, not enforced configuration. To block an action regardless of what Claude
> decides, use a PreToolUse hook instead.

> If the instruction is something that must run at a specific point, such as before every commit or after
> each file edit, write it as a hook instead.

### A3. "Never read-then-write" and "tests and implementation in separate sessions" are written as prose. How do you know the agent followed them? How would you *enforce* them?

Look for:
- Admits CLAUDE.md is advisory.
- Hooks: PostToolUse runs tests; PreToolUse blocks edits to history SQL / schema.
- A test-writer subagent with restricted tools and no `src/main` access, fed only spec + criteria.
- `settings.json` permissions; CI as the final gate.
- Points to `ConcurrentAmendmentTest` as the real enforcer of invariant 7 — and that only works if CI runs it.

---

## AI3. Two sources of truth, already drifted

### Evidence

CLAUDE.md says **two** tables:

`NOTCLAUDE.md` — lines 88–100

```markdown
 88  | Language | Java 17 | Runs on the reviewer's machine without a JDK upgrade |
 89  | Framework | Spring Boot 3 | Client-standard stack |
 90  | Persistence | `JdbcClient` + H2 in-memory, `schema.sql` | Two tables, no associations; explicit SQL keeps audit-critical writes visible |
 91  | Build | Maven wrapper (`./mvnw`) | Only a JDK is required to run |
 92  | API docs | `springdoc-openapi` | Browsable API at `/swagger-ui.html` |
 93  | Errors | `ProblemDetail` (RFC 7807) | Structured errors via one `@RestControllerAdvice` |
 94  | Tests | JUnit 5, AssertJ, MockMvc | Included in the starter |
 95  | UI | Single static page, plain JavaScript + Pico.css (vendored) | No build step, no framework |
 96  
 97  **Deliberately not used:** JPA/Hibernate (dirty checking conflicts with
 98  immutability; mapping buys nothing at two tables), Lombok (records suffice),
 99  Spring Security (identity is stubbed), messaging, Flyway, Testcontainers, and
100  any JavaScript framework.
```

The schema has **three** (`exception_request`, `request_history`, `idempotency_record` — see
`src/main/resources/schema.sql` lines 3, 24, 41). The data-model section of the same file lists all three
(lines 104–137), so the file contradicts itself.

`config.yaml` copies CLAUDE.md and declares itself the stale side:

`openspec/config.yaml` — lines 1–18

```yaml
 1  schema: spec-driven
 2  
 3  # Constraints for planning artifacts. Derived from CLAUDE.md — if the two
 4  # disagree, CLAUDE.md is the source and this file is stale. OpenSpec does not
 5  # read CLAUDE.md; this field is the only project context its skills load.
 6  context: |
 7    Basis is an internal service for a financial-services client. A relationship
 8    manager (RM) requests a discount on the standard mortgage interest rate; an
 9    authorised reviewer approves or declines it before the application process may
10    use it. Repository `basis`, Maven artifactId `basis`, root package
11    `com.example.basis`. Keep the client's name out of all of them.
12  
13    GUIDING CONSTRAINT — this is graded. The brief asks for a small, clear
14    solution with thoughtful decisions over a broad but fragile implementation.
15    Do not introduce dependencies, layers, or abstractions beyond those listed
16    below. Prefer the fewest moving parts that satisfy the acceptance criteria.
17    Scope discipline is not a shortcut here; breadth is a defect.
18  
```

…and subordinates the specs to CLAUDE.md:

`openspec/config.yaml` — lines 64–72

```yaml
64    PRECEDENCE — CLAUDE.md at the repository root is the source of truth. It
65    already fixes the data model, the API surface, the enumerations, the
66    validation rules, the status codes and the acceptance criteria. Generated
67    specs restate that contract in scenario form; they do not extend or revise
68    it. Where a generated artifact and CLAUDE.md disagree, CLAUDE.md is correct
69    and the artifact is the defect - fix the artifact, never reconcile by
70    changing the service's behaviour. Where CLAUDE.md is silent on something an
71    artifact needs, that is a gap: say so and ask, rather than inventing a value
72    and recording it as though it were specified.
```

The code cites CLAUDE.md, not the OpenSpec specs, as its source:

```
src/main/java/.../RequestBodies.java:17          * The wire shapes. Bounds come from the validation table in CLAUDE.md ...
src/main/java/.../ExceptionRequestStore.java:275 * ORDER BY is that rule, and it is the weaker choice CLAUDE.md records --
src/main/resources/schema.sql:38                 -- reserves KEY as a keyword, so the column CLAUDE.md names is quoted ...
src/test/java/.../RequestStatusTest.java:11       * The state machine, asserted against the diagram in CLAUDE.md ...
src/test/java/.../RequestValidationTest.java:14   * The bounds in CLAUDE.md's validation table, asserted at both edges ...
```

So the same facts live in three places (CLAUDE.md, `config.yaml`, `openspec/specs/`), with a fourth layer
(code comments) pointing at the prose file.

### A5. CLAUDE.md says two tables; the schema has three. `config.yaml` copies CLAUDE.md. How do you keep agent context from rotting as the code changes?

Look for:
- Single source of truth: specs own behaviour; CLAUDE.md references them (`@openspec/specs/...`) instead of
  copying.
- Context changes ship in the same PR as the code they describe; reviewed like code.
- Periodic audit — Claude Code's `/doctor prompt-audit` checks CLAUDE.md, rules and skills for outdated or
  conflicting instructions.
- A CI check (e.g. `openspec validate` / a script that diffs documented tables against `schema.sql`).

---

## AI4. AI output reached the code without catching its false claims

### Evidence

**(a) "cannot disagree" — but they can.** The constants class claims a single source:

`src/main/java/com/example/basis/request/RequestLimits.java` — lines 3–23

```java
 3  /**
 4   * Field bounds, in one place so that validation and the database constraint
 5   * cannot disagree. The discount range in particular is a policy decision
 6   * rather than a known business rule, so it is named rather than repeated as a
 7   * literal wherever it is enforced.
 8   */
 9  public final class RequestLimits {
10  
11      public static final int MIN_DISCOUNT_BPS = 1;
12      public static final int MAX_DISCOUNT_BPS = 200;
13  
14      public static final int MIN_REASON_LENGTH = 10;
15      public static final int MAX_REASON_LENGTH = 500;
16  
17      public static final int MAX_APPLICATION_ID_LENGTH = 64;
18      public static final int MAX_IDENTITY_LENGTH = 128;
19      public static final int MAX_IDEMPOTENCY_KEY_LENGTH = 128;
20  
21      private RequestLimits() {
22      }
23  }
```

The schema repeats the same numbers as literals:

`src/main/resources/schema.sql` — lines 5–7

```sql
5      application_id VARCHAR(64)  NOT NULL,
6      discount_bps   INT          NOT NULL,
7      reason         VARCHAR(500) NOT NULL,
```

`src/main/resources/schema.sql` — lines 15–18

```sql
15      -- Invariant 5: a discount is a positive integer in basis points. Validation
16      -- rejects out-of-range values first; this makes the storage itself unable
17      -- to hold one.
18      CONSTRAINT discount_bps_in_range CHECK (discount_bps BETWEEN 1 AND 200)
```

The task the AI ticked said the opposite of what was delivered:

`openspec/changes/archive/2026-09-20-request-lifecycle/tasks.md` — lines 25–25

```markdown
25  - [x] 2.4 Add the discount bound 1–200 as a named constant. Verify it is referenced by validation rather than repeated as a literal
```

**(b) "from the same moment" — not under READ COMMITTED.**

`src/main/java/com/example/basis/request/ExceptionRequestStore.java` — lines 324–331

```java
324      /**
325       * A request with its history. Read together so that the two cannot be
326       * reported from different moments.
327       */
328      @Transactional(readOnly = true)
329      public RequestBodies.Detail detailOf(UUID id) {
330          return RequestBodies.Detail.of(require(id), historyOf(id), json);
331      }
```

Two `SELECT`s in a read-only transaction at the default READ COMMITTED isolation can see different commits;
only REPEATABLE READ / snapshot (or a single joined query) makes the claim true.

**(c) The payload column can overflow.** `reason` is validated up to 500 characters, but the history
`payload` stores it JSON-encoded in a 1,024-character column:

`src/main/resources/schema.sql` — lines 24–32

```sql
24  CREATE TABLE IF NOT EXISTS request_history (
25      id          UUID         NOT NULL PRIMARY KEY,
26      request_id  UUID         NOT NULL REFERENCES exception_request (id),
27      entry_type  VARCHAR(16)  NOT NULL,
28      version     INT          NOT NULL,
29      actor       VARCHAR(128) NOT NULL,
30      payload     VARCHAR(1024),
31      occurred_at TIMESTAMP WITH TIME ZONE NOT NULL
32  );
```

Encoded length of `{"discountBps":200,"reason":"…"}` (computed with a standard JSON encoder; Jackson escapes
the same way):

| `reason` (500 chars, passes validation) | Stored payload length | Fits `VARCHAR(1024)`? |
|---|---|---|
| 500 × `"` | 1,031 | **No** |
| 500 × `\` | 1,031 | **No** |
| 500 × control char `\u0001` | 3,031 | **No** |

Result: the insert fails inside the transaction → HTTP 500 on create or amend. Not executed against the
running service (see README of this folder); the arithmetic is deterministic.

**The candidate's own rule these violate:**

`NOTCLAUDE.md` — lines 394–394

```markdown
394  - Anything an agent produces must be read and understood before it is committed.
```

### A4. Show `RequestLimits` ("cannot disagree") next to `schema.sql` (`BETWEEN 1 AND 200`). Who wrote this comment, and how did it get past review?

Look for:
- Ownership — "I accepted it, I missed it" — not "Claude wrote that".
- How they review AI *prose* (comments, design claims), not only AI code.
- Techniques: ask the agent to prove each claim with a test or a pointer; adversarial review by a second agent;
  treat comments as code in review; tests for invariants the comments assert (e.g. a test that reads the
  schema constraint and compares it with `RequestLimits`).
- Bonus: spots (b) and (c) unprompted when shown the files.

---

## AI5. The context file is written for the grader

### Evidence

`NOTCLAUDE.md` — lines 18–23

```markdown
18  ## Guiding constraint
19  
20  The brief asks for **a small, clear solution with thoughtful decisions over a
21  broad but fragile implementation**. Scope discipline is a graded property here,
22  not a shortcut. Do not add dependencies, layers, or abstractions beyond what is
23  listed below without being asked.
```

`openspec/config.yaml` — lines 13–17

```yaml
13    GUIDING CONSTRAINT — this is graded. The brief asks for a small, clear
14    solution with thoughtful decisions over a broad but fragile implementation.
15    Do not introduce dependencies, layers, or abstractions beyond those listed
16    below. Prefer the fewest moving parts that satisfy the acceptance criteria.
17    Scope discipline is not a shortcut here; breadth is a defect.
```

This one instruction explains much of the architecture findings: no service layer, headers read at every call
site, a store that does everything (see `03-design-patterns.md` DP1/DP7). The AI followed it faithfully — the
lifecycle design is titled *"Two classes on the write path, not three"*.

### How to raise it (follow-up to D2)

"Your CLAUDE.md forbids layers because it's graded. If this weren't a take-home, what would the instruction
say — and would the code look different?"

Look for: a clear distinction between *their* architectural judgement and the instruction they gave the AI; a
concrete description of the instruction they'd write for a production codebase (layering, boundaries,
enforcement). **Red flag:** cannot separate the two — the constraint was the design.
