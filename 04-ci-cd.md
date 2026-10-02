# 04 — CI/CD & Docker

Selected: **CI1 — no CI/CD at all (undeclared)** · **CI2 — no Docker**.

**Section verdict (from the review):** nothing to assess in the repository, so the whole grade for this area
comes from the interview. The disciplined demo script and Maven wrapper suggest automation instincts, but CI/CD
is the one omission the candidate did **not** declare.

---

## CI1. No CI/CD — and it is not declared

### Evidence

**1. No pipeline definition of any kind.** Search of the repository (depth 4, excluding `.git`) for
`.github/`, `.gitlab-ci.yml`, `Jenkinsfile`, `azure-pipelines.yml`, `Dockerfile*`, `*compose*.yml`,
`Chart.yaml`, `.dockerignore`, `skaffold.yaml` → **0 results**.

Full list of non-source, non-spec files at the root:

```
.gitattributes  .gitignore  .mvn/wrapper/maven-wrapper.properties  mvnw  mvnw.cmd
NOTCLAUDE.md  README.md  pom.xml  scripts/demo.sh  scripts/demo.ps1
```

**2. Every other omission is declared — CI/CD is not.** The candidate's "Deliberately not built" list:

`README.md` — lines 433–447

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
```

`grep -i "CI|pipeline|github actions|docker|container" README.md` → only the Testcontainers line above.
The same is true of the CLAUDE.md out-of-scope list:

`NOTCLAUDE.md` — lines 374–378

```markdown
374  ## Out of scope — state in the README, do not build
375  
376  Real authentication (OIDC/Entra ID), Postgres + Flyway, Testcontainers, event
377  publishing to downstream consumers via a transactional outbox, persistent
378  idempotency store with TTL, observability.
```

**3. The only automation is a demo script** — well written, but run by hand:

`scripts/demo.sh` — lines 1–23

```bash
 1  #!/bin/sh
 2  # One scenario, end to end, narrated: create, replay, amend, a stale approval,
 3  # a real approval, a refused self-approval, then what the mortgage process
 4  # reads. scripts/demo.ps1 is the same scenario for PowerShell -- a change to
 5  # one is a change to both.
 6  #
 7  #   scripts/demo.sh [base-url]        (default http://localhost:8080)
 8  #
 9  # Needs curl and a running service, nothing else. There is no jq: it is not
10  # installed by default on macOS or Windows, and the values pulled from a
11  # response are fixed-shape fields the service formats itself, so sed is enough.
12  #
13  # Identity is the X-User-Id / X-User-Role pair, standing in for JWT claims.
14  # Nothing verifies it; that is why one script can play both people.
15  #
16  # A step that does not answer what it narrates stops the script. A demo that
17  # says "this is refused" and carries on after a 200 would be telling the
18  # reader the behaviour holds at the moment it has stopped holding.
19  #
20  # ASCII only, matching demo.ps1, which Windows PowerShell 5.1 would misread.
21  
22  set -eu
23  
```

It already behaves like a pipeline gate (`set -eu`, stops on the first unexpected response, unique run id) —
it is a smoke test in the wrong place.

**4. Nothing to version or promote.** The artefact is still a snapshot, with no release, SCM or
`distributionManagement` configuration:

`pom.xml` — lines 14–18

```xml
14      <groupId>com.example</groupId>
15      <artifactId>basis</artifactId>
16      <version>0.0.1-SNAPSHOT</version>
17      <name>basis</name>
18      <description>Pricing exception requests for mortgage applications</description>
```

**5. The candidate's own working agreement points at history as evidence** — which the zip doesn't contain:

`NOTCLAUDE.md` — lines 382–384

```markdown
382  ## Working agreement
383  
384  - Commit in small steps with honest messages; the history is part of what is reviewed.
```

→ **Action before the interview:** review the GitHub commit history (commit size, message honesty, whether
tests and implementation landed in separate commits as their process claims).

### Why it matters for this role

The role explicitly lists CI/CD. In an otherwise carefully scoped submission, leaving the pipeline out
*silently* is the most notable gap. There is also no AI in the delivery pipeline (PR review, spec
verification), although their OpenSpec setup has a `verify` command that could run as a CI gate.

### How to raise it

"Every omission in your README is declared except CI/CD. Why? Design the first pipeline for this repo,
from pull request to production."

Look for:
- PR: `./mvnw verify`, unit/integration split, SAST, dependency scan + SBOM, required checks / branch protection.
- Main: build the image **once**, tag with the commit SHA, sign, push to a registry.
- Deploy to dev via GitOps (Argo CD / Flux) + Helm; run `demo.sh` as the post-deploy smoke test; promote
  **the same artefact**; automatic rollback on a failed smoke test.
- Maven cache, OIDC federation to the cloud (no long-lived secrets), environment protection rules.
- Where AI fits (PR review, `openspec verify`, explaining failures) and what it must **not** do (merge, deploy).

---

## CI2. No Docker

### Evidence

- No `Dockerfile`, `.dockerignore` or Compose file (search above → 0 results).
- No image build configuration in the build — `grep -i "jib|build-image|image" pom.xml` → **0 matches**.
  The only plugin is the plain Boot plugin:

`pom.xml` — lines 58–65

```xml
58  
59      <build>
60          <plugins>
61              <plugin>
62                  <groupId>org.springframework.boot</groupId>
63                  <artifactId>spring-boot-maven-plugin</artifactId>
64              </plugin>
65          </plugins>
```

- No registry, no image tagging scheme, and the version is `0.0.1-SNAPSHOT` → nothing ties a running instance
  to a commit (no `build-info` / `git-commit-id` either).
- The runtime is designed around "nothing but a JDK" — the candidate's stated reason for H2 in-memory:

`README.md` — lines 438–440

```markdown
438  - **Postgres and Flyway.** H2 in-memory with `schema.sql` means the service
439    runs with nothing but a JDK.
440  - **Testcontainers.** The same reasoning.
```

  That is a reasonable *local-run* goal, but in a container world the equivalent is a single `docker run` /
  `docker compose up` with Postgres — which would also have removed the need for H2.

### Why it matters

Without an image there is no path to Kubernetes/Helm (the Infra topic), no reproducible runtime, and no
artefact a pipeline could promote. Together with CI1 it means the delivery side of the SDLC is entirely
absent from the submission.

### How to raise it

"Containerise this service. What does the Dockerfile — or the build-plugin config — look like, and what do
you get by doing it in the build rather than in a Dockerfile?"

Look for:
- Spring Boot `build-image` (buildpacks) or Jib vs a hand-written multi-stage Dockerfile; layered jars for
  cache-friendly layers.
- Non-root user, minimal/distroless base, pinned base-image digest, image scanning.
- JVM container awareness (`-XX:MaxRAMPercentage`), graceful shutdown, health endpoint for probes.
- Config and secrets injected at runtime (env / mounted secrets), never baked into the image.
- A Compose file with Postgres for local runs — replacing in-memory H2.
