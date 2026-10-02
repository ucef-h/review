# Interview Review — `basis-main`

**Role:** AI-Enabled Engineer (Senior)
**Repo under review:** `basis-main` — Spring Boot 3.5.16 / Java 17 service for mortgage pricing
exception requests (create, amend, approve/decline, withdraw, audit history). `JdbcClient` + in-memory H2.
**Prepared:** 2026-10-02, by the interviewer with Claude (Cowork).

This folder holds **only the points the interviewer selected to keep** during the review session.
Each file is self-contained: finding → evidence (code/doc excerpts with line numbers) → fix →
interview question → what a good answer contains.

## Files

| File | Section | Selected items |
|---|---|---|
| [01-java.md](01-java.md) | Java | Q2, Q3, Q7 · findings F1, F2, F4 |
| [02-spring.md](02-spring.md) | Spring | S3, S4, S5 |
| [03-design-patterns.md](03-design-patterns.md) | Design patterns & architecture | Findings DP1, DP2, DP3, DP4, DP7 · questions D1, D2, D5, D7 |
| [04-ci-cd.md](04-ci-cd.md) | CI/CD & Docker | CI1 (no CI/CD), CI2 (no Docker) |
| [05-ai-claude-usage.md](05-ai-claude-usage.md) | AI (Claude usage) | AI1–AI5 · questions A2, A3, A4, A5 |

Reviewed but **no items selected**: Maven, Infra (DB/Helm), Spec-Driven Development.

## Context the interviewer should know

- **Agent instruction files were renamed by the interviewer** so tools would not auto-load them:
  `CLAUDE.md` → `NOTCLAUDE.md`, `.claude/` → `notclaude/`. Code comments that cite "CLAUDE.md"
  refer to `NOTCLAUDE.md` in this copy.
- **The `notclaude/` folder is tool-generated**, not candidate work: every skill carries
  `author: openspec` / `generatedBy: "1.13.1"` (created by `openspec init`).
- **The candidate declared most omissions** in the README "Deliberately not built" section:
  real auth, Postgres + Flyway, Testcontainers, outbox/event publishing, idempotency TTL,
  observability. Frame those as *"you named it — now design it"*. **CI/CD and Docker are not in
  that list** — they are undeclared.
- **No git history** — the repo was downloaded as a zip. The candidate's own working agreement says
  *"the history is part of what is reviewed"*. Check the GitHub commit history before the interview.
- **Tests were not executed** during this review (Maven Central blocked in the review sandbox; the
  reviewer's Mac has only JDK 11). The README claims "72 tests, green". The payload-overflow defect in
  `01-java.md` is proven by arithmetic on the JSON encoding, not by running the service.

## Size of the submission (for calibration)

| Artefact | Lines |
|---|---|
| Java main (23 classes) | ~1,330 |
| UI (`static/index.html`) | 478 |
| Tests (11 classes, 7 of them `@SpringBootTest`) | ~2,100 |
| OpenSpec specs + 6 archived changes | ~3,240 |
| `NOTCLAUDE.md` (candidate's CLAUDE.md) | 394 |
