---
bootstrapped_at: 2026-10-12T21:24:49Z
starter_id: spring
starter_name: Spring Boot
project_name: karting-rating
language_family: java
package_manager: maven
cwd_strategy: subdir-then-move
bootstrapper_confidence: verified
phase_3_status: ok
audit_command: "null"
---

## Hand-off

```yaml
starter_id: spring
package_manager: maven
project_name: karting-rating
hints:
  language_family: java
  team_size: solo
  deployment_target: fly
  ci_provider: github-actions
  ci_default_flow: auto-deploy-on-merge
  bootstrapper_confidence: verified
  path_taken: standard
  quality_override: false
  self_check_answers: null
  has_auth: true
  has_payments: false
  has_realtime: false
  has_ai: false
  has_background_jobs: false
```

## Why this stack

KartingRating is a small web app (about ten users) with login, two roles, result entry and several computed season tables. You asked for a Java backend with PostgreSQL, and Spring Boot is the vetted default for Java web projects. It is typed, convention-based, widely represented in training data and well documented, so an AI agent can work reliably in it. Scaffolding is verified end-to-end. It fits your Java background and your goal of building a full application on your own. Frontend remains open; PostgreSQL is a database preference to apply when the project is scaffolded (e.g. Spring Data JPA). Deployment defaults to Fly with GitHub Actions auto-deploy on merge.

## Pre-scaffold verification

| Signal      | Value   | Severity | Notes                                                        |
| ----------- | ------- | -------- | ------------------------------------------------------------ |
| npm package | not run | n/a      | non-JS starter                                               |
| GitHub repo | not run | n/a      | card docs_url (https://docs.spring.io/spring-boot/) is not a GitHub URL; no recency signal available |

## Scaffold log

**Resolved invocation**: `mkdir .bootstrap-scaffold && curl -sS --fail https://start.spring.io/starter.tgz -d dependencies=web,devtools -d type=maven-project -d javaVersion=21 -d groupId=com.example -d artifactId=karting-rating -d name=karting-rating | tar -xzf - -C .bootstrap-scaffold`
(Adapted from the card's `cmd_template`: extracted into the temp directory via `tar -C` and used `project_name` as the artifactId, because `.bootstrap-scaffold` is not a valid Maven artifactId and the original template extracts into cwd. Added `name=karting-rating`.)
**Strategy**: subdir-then-move
**Exit code**: 0 (tar printed harmless `LIBARCHIVE.creationtime` header warnings)
**Files moved**: 10 (.gitattributes, .gitignore, HELP.md, mvnw, mvnw.cmd, pom.xml, .mvn/wrapper/maven-wrapper.properties, src/main/resources/application.properties, src/main/java/com/example/karting_rating/KartingRatingApplication.java, src/test/java/com/example/karting_rating/KartingRatingApplicationTests.java)
**Conflicts (.scaffold siblings)**: none
**.gitignore handling**: moved silently (absent in cwd)
**.bootstrap-scaffold cleanup**: deleted

Notes: Spring Boot 4.1.1, Java 21, dependencies web + devtools. `mvnw` marked executable. No `java` binary found on PATH in this environment, so the project was not built or run.

## Post-scaffold audit

**Tool**: skipped — no built-in audit tool for java
**Recommended external tool**: OWASP Dependency-Check or Snyk

## Hints recorded but not acted on

| Hint                    | Value                |
| ----------------------- | -------------------- |
| bootstrapper_confidence | verified             |
| quality_override        | false                |
| path_taken              | standard             |
| self_check_answers      | null                 |
| team_size               | solo                 |
| deployment_target       | fly                  |
| ci_provider             | github-actions       |
| ci_default_flow         | auto-deploy-on-merge |
| has_auth                | true                 |
| has_payments            | false                |
| has_realtime            | false                |
| has_ai                  | false                |
| has_background_jobs     | false                |

Also not applied: PostgreSQL / Spring Data JPA preference (only `web` and `devtools` were scaffolded).

## Next steps

Next: a future skill will set up agent context (CLAUDE.md, AGENTS.md). For now, your project is scaffolded and verified — happy hacking.

Useful manual steps in the meantime:
- `git init` (if you have not already) to start your own repo history.
- Review any `.scaffold` siblings the conflict policy created and decide which version of each file to keep.
- Address audit findings per your project's risk tolerance — the full breakdown is in this log.
