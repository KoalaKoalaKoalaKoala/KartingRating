# First deploy: KartingRating → Fly.io + Neon Postgres

## Context

`context/foundation/infrastructure.md` picked **Fly.io (app) + Neon Free (Postgres)**. `tech-stack.md` sets **GitHub Actions auto-deploy on merge to `main`**. The repo is still a bare Spring Boot 4.1.1 scaffold: only `spring-boot-starter-webmvc` and devtools, with no actuator, JPA, driver or Dockerfile, and no `.github/`. Locally, `gh` and Docker Desktop are available, but **`flyctl` and Java are not installed in WSL**.

Goal: a production baseline running on Fly that connects to Neon, applies a Flyway migration, has a health check gating each deploy, and redeploys automatically on every merge to `main`. Decisions made: **App + Neon DB now**. **pg_dump backups deferred** and recorded as a follow-up.

Legend: 🤖 = the agent does it, 👤 = manual/human gate (account, payment, secrets, merge).

---

## Phase 0: Prerequisites (CLI and tooling setup)

All commands run in the WSL shell (Ubuntu, bash). Each item says how to verify it.

Status checked on 2026-10-19, re-checked 2026-10-22:

| Tool | Status |
|---|---|
| `gh` | ✅ logged in, scopes `repo` + `workflow` |
| `origin` remote | ✅ set |
| Docker | ✅ Docker Desktop binary present |
| `flyctl` | ✅ v0.4.120, logged in, org `personal` |
| Java | ✅ Temurin 21.0.12.1, Maven wrapper 3.10.0 |
| SDKMAN | ✅ installed |
| `psql` | ✅ 18.6 |
| `node` | ✅ v22.22.1 (`neonctl` 8.11.0, logged in, no projects yet) |

### 0.1 Fly.io CLI (`flyctl`)

- [x] 👤 Install: `curl -L https://fly.io/install.sh | sh`
- [x] 👤 Add it to `PATH`. Append these lines to `~/.bashrc`, then run `source ~/.bashrc`:
  ```bash
  export FLYCTL_INSTALL="$HOME/.fly"
  export PATH="$FLYCTL_INSTALL/bin:$PATH"
  ```
- [x] 👤 Verify with `fly version`. `flyctl version` also works, since `fly` is an alias.
- [x] 👤 Create a Fly account (fly.io/app/sign-up) and **add a payment method**. New accounts have no free tier, so the app won't start without one.
- [x] 👤 Log in with `fly auth login`. In WSL, if no browser opens, copy the printed URL into the Windows browser.
- [x] 👤 Verify with `fly auth whoami`, which shows your email. Then `fly orgs list` should show the `personal` org.
- [ ] 👤 Check current pricing at fly.io/docs/about/pricing (the risk register flags the third-party figures).
- [x] 🤖 Run `fly platform regions` to confirm `waw` (Warsaw) is available. Fallback: `fra` (Frankfurt). Record the choice.
  - **Result (2026-10-22): `waw` is not in the region list. Chosen region: `fra` (Frankfurt)**, which matches Neon `aws-eu-central-1`.
- [ ] Note: the personal `fly auth login` session is **only for local use**. CI gets a separate app-scoped deploy token in Phase 6. Never put the personal token in GitHub.
- [ ] Optional: `fly mcp server --help` confirms the MCP server is available. Use the CLI by default for MVP.
- [ ] Optional: keep flyctl current with `fly version upgrade`, because flyctl changes often.

### 0.2 Java 21 (local build and tests)

- [x] 👤 Install SDKMAN: `curl -s "https://get.sdkman.io" | bash`, then `source "$HOME/.sdkman/bin/sdkman-init.sh"`.
- [x] 👤 Run `sdk list java | grep -- -tem`, then `sdk install java <21.x.y>-tem` and `sdk default java <21.x.y>-tem`.
  - Alternative without SDKMAN: `sudo apt install openjdk-21-jdk`.
- [x] 👤 Verify:
  - `java -version` shows 21
  - `echo $JAVA_HOME` is set
  - `./mvnw -v` shows Java 21 and the Maven wrapper version
- [ ] **Edge case: `mvnw` is not executable in git.** It's stored as mode `100644`, so `./mvnw` would fail with `Permission denied` in GitHub Actions and in the Docker build.
  - 🤖 Fix: `git update-index --chmod=+x mvnw`, committed with the Phase 2 changes.
  - The Dockerfile also runs `chmod +x mvnw` before building, as a backstop.
  - The `/mnt/d` NTFS mount shows every file as 777, which hides the problem locally.
- [ ] Edge case: line endings. `.gitattributes` already forces `mvnw` to LF, so keep it that way. A CRLF `mvnw` fails with `/bin/sh^M: bad interpreter`.

### 0.3 Docker (Testcontainers and the local image build)

- [x] 👤 In Docker Desktop, go to Settings → Resources → WSL integration and turn it on for this distro.
- [x] 👤 Verify:
  - `docker info` works from WSL, with no permission error
  - `docker run --rm hello-world` succeeds
- [ ] Note: deploys use Fly's **remote builder** (`fly deploy --remote-only`), so local Docker is only needed for tests and the optional local image smoke test.

### 0.4 GitHub CLI and repository

- [x] ✅ `gh auth status` shows you logged in with the `repo` and `workflow` scopes. `workflow` is required to push files under `.github/workflows/`.
  - If it's ever missing: `gh auth refresh -s workflow`.
- [x] ✅ `git remote -v` shows `origin` → `github.com/KoalaKoalaKoalaKoala/KartingRating`.
- [x] 🤖 Run `gh repo view --json defaultBranchRef` to confirm the default branch is `main`, so that `push` to `main` triggers the deploy.
- [x] 🤖 Check that `main` is pushed and up to date: `git status -sb`, `git log origin/main -1`.

### 0.5 Postgres client (Neon sanity checks, later backups)

- [x] 👤 Install: `sudo apt install postgresql-client`
- [x] 👤 Verify with `psql --version`. Version 15 or newer is fine for connecting. For later `pg_dump` backups, the client should match Neon's server major version.

### 0.6 Neon CLI (optional)

The console is enough for this plan. The CLI only helps with scripting.

- [x] Optional 👤: `neonctl` needs Node.js, which isn't installed. Either `sudo apt install nodejs npm` then `npm i -g neonctl`, or skip it and use the Neon console.
- [x] Optional 👤: `neonctl auth`, then `neonctl projects list` to verify.

### 0.7 Prerequisites done: one-shot check

- [x] 🤖 Run this and expect every line to succeed (passed 2026-10-22):
  ```bash
  fly version && fly auth whoami && java -version && ./mvnw -v && docker info --format '{{.ServerVersion}}' && gh auth status && psql --version
  ```

## Phase 1: Neon database (👤 external integration)

- [x] 👤 Create a Neon account and a project `karting-rating` on the **Free** plan, in the EU region closest to the Fly region (Frankfurt `aws-eu-central-1`, matching Fly `fra`). Verify the region in the console.
  - **Result (2026-10-22):** project `karting-rating` (id `summer-pond-39742626`), region `aws-eu-central-1`, default branch `main` (`br-gentle-river-bahjo94k`), 0.25 CU.
- [x] 👤 Note the **direct** (non-pooler) host, database name, role and password. Use the direct endpoint, not `-pooler`, so Flyway and Hikari prepared statements work without PgBouncer caveats.
  - Host: `ep-solitary-queen-ba2b2bps.c-8.eu-central-1.aws.neon.tech` (direct, pooler off). DB: `neondb`. Role: `neondb_owner`. Password: not recorded here.
- [x] 👤 Build the JDBC URL: `jdbc:postgresql://<host>/<db>?sslmode=require`. **Not** the `postgres://` string Neon shows.
  - `jdbc:postgresql://ep-solitary-queen-ba2b2bps.c-8.eu-central-1.aws.neon.tech/neondb?sslmode=require`
- [ ] 👤 Optional sanity check: `psql "postgresql://<user>@<host>/<db>?sslmode=require" -c 'select 1'`.
- [ ] 👤 Keep the password out of the chat. You'll paste it yourself in Phase 4.

## Phase 2: App changes (🤖, on branch `deploy/fly-first-release`)

- [ ] `pom.xml`: add
  - `spring-boot-starter-actuator`
  - `spring-boot-starter-data-jpa`
  - `spring-boot-starter-flyway` + `org.flywaydb:flyway-database-postgresql`
  - `org.postgresql:postgresql` (runtime)
  - `spring-boot-docker-compose` (runtime, optional, for a local DB)
  - test: `spring-boot-testcontainers` + Testcontainers PostgreSQL module

  Confirm the exact Boot 4.1 / Testcontainers 2.x artifact names before editing.
- [ ] `src/main/resources/db/migration/V1__baseline.sql`: a minimal baseline (e.g. a `schema_info` table) that proves migrations run.
- [ ] `application.properties` (shared):
  - `spring.jpa.hibernate.ddl-auto=validate`
  - `spring.jpa.open-in-view=false`
  - `management.endpoints.web.exposure.include=health`
  - `management.endpoint.health.probes.enabled=true`
- [ ] `application-prod.properties`, Hikari tuned for Neon autosuspend:
  - `maximum-pool-size=3`
  - `minimum-idle=0`
  - `idle-timeout=60000`
  - `max-lifetime=240000`
  - `connection-timeout=15000` (allows for Neon wake-up)
  - `keepalive-time=0`
- [ ] `compose.yaml` with `postgres:17` for local `./mvnw spring-boot:run`. Boot's docker-compose support starts it automatically.
- [ ] Test: `KartingRatingApplicationTests` uses Testcontainers PostgreSQL (`@ServiceConnection`) so `contextLoads` covers Flyway + JPA.
- [ ] `Dockerfile`:
  - multi-stage: `eclipse-temurin:21-jdk` runs `chmod +x mvnw && ./mvnw -DskipTests package`, then `eclipse-temurin:21-jre`
  - non-root user, `EXPOSE 8080`
  - `JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=70 -XX:+UseSerialGC"`
- [ ] `.dockerignore`: `target/`, `.git/`, `context/`, `.claude/`.
- [ ] `fly.toml`, hand-written so `fly launch` doesn't auto-generate a Java config:
  - `app`, `primary_region = "fra"`, `[build] dockerfile`
  - `[env] SPRING_PROFILES_ACTIVE = "prod"`
  - `[http_service] internal_port = 8080`, `force_https = true`, `auto_stop_machines = "off"`, `auto_start_machines = true`, `min_machines_running = 1`
  - `[[http_service.checks]] path = "/actuator/health/liveness"`, `grace_period = "60s"`, `interval = "30s"`, `timeout = "5s"`
  - `[[vm]] memory = "512mb"`, `cpu_kind = "shared"`, `cpus = 1`
  - ⚠️ **Edge case:** the health check uses **liveness**, not `/actuator/health`. The full health endpoint includes the DB indicator, so every check would wake Neon. That keeps the compute up 24/7: about 0.25 CU × 730 h ≈ 182 CU-h, which is over the free 100 CU-h/month.
- [ ] Run `./mvnw verify` locally. It must pass.
- [ ] Run `docker build -t karting-rating .`, then `docker run` with `SPRING_DATASOURCE_*` pointing at the compose DB. Smoke-test `/actuator/health`.

## Phase 3: Fly app creation (🤖 commands, 👤 approves cost)

- [ ] Run `fly apps create karting-rating`. If the name is taken globally, use `karting-rating-<suffix>` and update `fly.toml`.
- [ ] Check with `fly apps list`.

## Phase 4: Secrets (👤, run yourself with the `!` prefix so the values stay out of the transcript)

- [ ] `! fly secrets set -a <app> --stage SPRING_DATASOURCE_URL="jdbc:postgresql://<host>/<db>?sslmode=require" SPRING_DATASOURCE_USERNAME=<role> SPRING_DATASOURCE_PASSWORD=<pw>`
- [ ] 🤖 Run `fly secrets list -a <app>`. It shows names and digests only.

## Phase 5: First manual deploy (🤖)

- [ ] Run `fly deploy -a <app> --remote-only`. Fly's remote builder means no local image push is needed.
- [ ] Check `fly status -a <app>`: 1 machine, `started`, checks `passing`.
- [ ] Check `fly logs -a <app> --no-tail`: Flyway applied `V1`, Hikari connected, no OOM.
- [ ] Run `curl -fsS https://<app>.fly.dev/actuator/health`. Expect `{"status":"UP"}`, with the DB component UP.
- [ ] Watch memory with `fly machine status`. If it hits OOM or restarts, bump `[[vm]] memory = "1gb"` and redeploy.
- [ ] **Edge-case support if the deploy fails:**
  - Health check timeout: raise `grace_period` and check JVM startup time in the logs.
  - `PSQLException SSL`: check `sslmode=require` in the URL.
  - Auth failure: re-stage the secrets.
  - Flyway checksum mismatch: never edit applied migrations; add `V2`.

## Phase 6: Auto-deploy on merge (🤖 writes it, 👤 provides the token)

- [ ] 👤 Run `! fly tokens create deploy -a <app>`. The token is scoped to this app only. Then `! gh secret set FLY_API_TOKEN` and paste the token.
- [ ] 🤖 `.github/workflows/ci-deploy.yml`:
  - Triggers: `pull_request` and `push` to `main`. `concurrency: deploy-main` (no cancel-in-progress for deploys).
  - Job `verify`: `actions/setup-java@v4` (temurin 21, maven cache), then `./mvnw -B verify`. Testcontainers works on `ubuntu-latest`.
  - Job `deploy`: `needs: verify`, `if: github.event_name == 'push' && github.ref == 'refs/heads/main'`. Uses `superfly/flyctl-actions/setup-flyctl` (pinned version), then `flyctl deploy --remote-only`, with `FLY_API_TOKEN` in env.
- [ ] 👤 Optional: enable branch protection on `main` that requires the `verify` check.
- [ ] 🤖 Push the branch and open a PR with `gh pr create`. The `verify` job must be green.
- [ ] 👤 **Merge the PR. This is the human approval gate.**
- [ ] 🤖 Run `gh run watch` on the `main` run. The deploy job must succeed and `fly releases -a <app>` must show a new version.

## Phase 7: Rollback rehearsal and verification (🤖)

- [ ] `fly releases -a <app> --image` lists the image refs.
- [ ] Rehearse a rollback with `fly deploy -a <app> --image <previous-ref>` (confirm the flag with `fly deploy --help`), then re-run the health check, then roll forward again.
- [ ] Confirm in the Neon console that the compute **suspends** after about 5 minutes without traffic, so the liveness check isn't keeping it awake. Check CU-hours usage after 24 h.

## Phase 8: Documentation (🤖)

- [ ] Save this approved plan to `context/deployment/deploy-plan.md`, with checkboxes ticked to match reality, the app name, region, URL and secret **names** (no values).
- [ ] Update `CLAUDE.md`: Postgres/JPA/Flyway are now in `pom.xml`; Fly + GH Actions auto-deploy are set up; add the local DB note (compose).
- [ ] Add follow-up items to `deploy-plan.md`:
  - [ ] **Deferred:** scheduled `pg_dump` backup (GitHub Actions cron to an artifact) plus one restore test, done **before real race results are entered**
  - [ ] Fly billing check after the first month

## Verification (end-to-end done criteria)

- [ ] `./mvnw verify` is green locally and in CI.
- [ ] `https://<app>.fly.dev/actuator/health` returns `UP` with the DB component UP.
- [ ] A trivial merge to `main` auto-deploys a new Fly release with no manual step.
- [ ] Rollback to the previous image is rehearsed successfully.
- [ ] Neon autosuspend is observed working, so the free CU-hours budget is safe.
