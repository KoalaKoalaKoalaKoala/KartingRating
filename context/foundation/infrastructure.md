---
project: karting-rating
researched_at: 2026-10-19
recommended_platform: Fly.io (app) + Neon free Postgres (external)
runner_up: Railway
context_type: mvp
tech_stack:
  language: Java 21
  framework: Spring Boot 4.1.1 (Maven)
  runtime: JVM (container)
---

## Recommendation

**Deploy the app on Fly.io and use a free external Postgres (Neon free plan).**

Fly.io has the strongest CLI and deploy loop of the platforms that can run a JVM, and it is already the declared target in `tech-stack.md` (Fly + GitHub Actions auto-deploy on merge). The interview answers (cost and DX "roughly equal", no familiarity, single region, co-location "don't know") left no hard preference, but the user asked in follow-up to keep the cost low for a short-lived course app. Fly's Managed Postgres (from about $38/mo) was therefore rejected, and unmanaged Fly Postgres was rejected because the user would own backups. An external free managed Postgres gives the lowest cost with provider-run infrastructure. Expected cost: about $4-7/mo (one always-on Fly machine of 512MB-1GB) plus $0 for the database.

## Platform Comparison

Cloudflare Workers/Pages, Vercel and Netlify were dropped by the runtime filter: none runs a Spring Boot JVM app. Interview answers: persistent connections "don't know" (PRD lists no realtime or background jobs, so no filter applied), cost vs DX "equal", familiarity "none", single region, co-location "don't know" (external allowed).

| Platform | CLI-first | Managed | Agent docs | Deploy API | MCP | Notes |
|---|---|---|---|---|---|---|
| Fly.io | Pass | Partial | Pass | Pass | Partial | Machines about $2-7/mo; Managed Postgres from $38/mo |
| Railway | Pass | Pass | Partial | Partial | Pass | Hobby $5/mo incl. $5 usage; Postgres as add-on |
| Render | Partial | Pass | Partial | Pass | Pass | Starter $7/mo; free tier sleeps; free Postgres expires after 30 days |

Scores come from vendor docs and third-party pricing blogs checked 2026-10-19. Pricing figures in particular are third-party and must be confirmed on the vendors' pricing pages. No `llms.txt` was found for any of the three. Railway's official docs fetched did not document CLI deploy, rollback or logs commands (only MCP tools), which is why its CLI/API scores are not higher.

- Fly: managed VMs, so Partial on "managed" because the Dockerfile and JVM sizing are the developer's. `fly mcp server` ships in flyctl and supports "most" command groups (apps, certs, logs, machine, orgs, platform, secrets, status, volumes); it states no beta label, and the excluded commands are not listed (checked 2026-10-19).
- Railway: official hosted MCP at `mcp.railway.com` (redeploy, projects, an agent for log analysis); a local server (`railway mcp local`) exposes deploy and get_logs. No beta label on the docs page (checked 2026-10-19).
- Render: official hosted MCP (deploy, logs, read-only SQL, env var updates). It cannot change scaling, create image-backed services or delete resources. No beta label on the page (checked 2026-10-19).

### Shortlisted Platforms

#### 1. Fly.io (Recommended)

Best CLI and scriptable deploy loop, an official CLI MCP server, and already the declared target with CI on merge.

#### 2. Railway

Simplest Spring Boot + Postgres setup and a similar price. It lost on weaker documented CLI evidence and usage-based billing that is harder to predict.

#### 3. Render

Simplest and most predictable, but the free tier sleeps and its free Postgres expires after 30 days, so a realistic setup costs about $13/mo.

### Database choice (external Postgres)

| Option | Limits (checked 2026-10-19) |
|---|---|
| Neon Free (chosen) | 1 GB storage per project; 100 CU-hours/month; compute auto-suspends after 5 minutes idle (cannot be disabled); 6-hour instant-restore window; 1 manual snapshot; 5 GB network transfer/month. Page describes no expiry or inactivity deletion. |
| Supabase Free | 500 MB per project; 2 free projects; pauses on inactivity (days not stated in fetched docs); backup details not stated |

Neon region list was not confirmed on the fetched page. Choose the EU region closest to the Fly region at project creation and verify it.

## Anti-Bias Cross-Check: Fly.io

### Devil's Advocate - Weaknesses

1. Managed Postgres on Fly starts around $38/mo, and unmanaged Fly Postgres leaves backups, upgrades and failover to the developer. This is avoided here by using an external database, but it adds a second vendor.
2. Spring Boot is not a listed Fly framework; the Dockerfile and JVM flags are the developer's responsibility (the docs fetched confirm a Dockerfile is the general route for unlisted stacks, but not Java-specific guidance).
3. A 256MB machine is OOM-killed by a HotSpot JVM; 512MB-1GB plus explicit heap flags are needed.
4. The Fly MCP server covers only "most" commands, with the excluded ones undocumented.
5. New accounts get no free tier (trial of 2 hours or 7 days per third-party sources), so cost starts on day one.
6. Neon's free compute suspends after 5 minutes idle, so the first request after idle adds database wake-up latency on top of any JVM start time.

### Pre-Mortem - How This Could Fail

The team chose Fly because the stack file said so and launched on a 256MB machine. Random restarts followed, so they raised memory and enabled auto-stop to save money. After idle periods, the JVM cold start (tens of seconds) plus the Neon wake-up made the site look broken to ten users on race night, and they stopped checking it. Separately, the Neon free database held two race weekends of results and the only protection was a 6-hour restore window; nobody exported dumps. A bad migration shipped on merge, auto-deploy had no health check gating it, and rolling back the image did not undo the schema change. The points algorithm could not be re-verified because the raw results were gone. The solo developer, working evenings only, spent December on operations instead of the ranking tables. The wrong assumptions were "free managed" meaning "no ops" and "auto-deploy" meaning "safe deploy".

### Unknown Unknowns

- Auto-stop / scale-to-zero makes the JVM cold start 10-30+ seconds; keep one machine always running for ten users.
- Spring needs a `jdbc:postgresql://...` URL. The `postgres://` connection string that Neon shows will not work as-is; use the JDBC form (or split host/user/password into `spring.datasource.*`), and require SSL.
- Neon's 6-hour restore window is not a backup strategy. Schedule regular `pg_dump` exports (for example from a GitHub Actions cron to an artifact).
- Neon's 5-minute autosuspend interacts with Hikari connection pooling: stale connections after suspend can throw errors. Configure a small pool, connection validation and a connect timeout, and consider Neon's pooled connection string.
- Rolling back a Fly release redeploys the old image only; database migrations do not roll back. Use backward-compatible migrations (Flyway/Liquibase, expand-then-contract).
- Spring Boot 4.1 behavior for JVM memory in containers and Hikari defaults was not verified against Fly docs; test the memory settings on the actual image.

## Operational Story

- **Preview deploys**: Fly has no built-in per-PR previews. Optional and not needed for ten users; if wanted, use a throwaway app per PR via `fly deploy --app <name>-pr-<n>` from GitHub Actions (each costs machine time). Default: test locally and deploy only on merge.
- **Secrets**: runtime secrets (`SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD`, any admin bootstrap password) live in Fly secrets (`fly secrets set ...`). The CI token (`FLY_API_TOKEN`) lives in GitHub Actions secrets. Prefer a deploy-scoped token (`fly tokens create deploy`). Rotation: set a new Neon password, `fly secrets set` it (this triggers a restart), then revoke the old one.
- **Rollback**: `fly releases` to list releases, then redeploy a previous image with `fly deploy --image <registry.fly.io/app:deployment-id>` (confirm the exact flag against the installed flyctl version). Typical time to revert: a few minutes. Database migrations are not reverted; use backward-compatible migrations.
- **Approval**: a human approves merges to main (which auto-deploys), any secret rotation, changing the Neon plan, and deleting the Neon project or Fly app. An agent may run `fly status`, `fly logs`, `fly releases` and, with the deploy token, `fly deploy` from CI.
- **Logs**: `fly logs -a <app>` (live tail, read-only), `fly status -a <app>`, `fly machine list -a <app>`. Via MCP: `fly mcp server` exposes logs, status and machine commands (partial coverage). Neon logs/metrics are in the Neon console; the Neon project can be queried read-only with `psql`.

## Risk Register

| Risk | Source | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| Data loss: results exist only in a free Neon DB with a 6-hour restore window | Pre-mortem | M | H | Scheduled `pg_dump` (GitHub Actions cron) stored as artifact/other storage; test a restore once |
| OOM kills on small machine | Devil's advocate | H | M | Use 512MB-1GB machine; set `-XX:MaxRAMPercentage` (about 65-70) and `-XX:+UseSerialGC`; test on the real image |
| JVM cold start after auto-stop | Unknown unknowns | H | M | Set `auto_stop_machines = false` / `min_machines_running = 1` in `fly.toml` |
| Neon autosuspend causes first-request latency and stale pooled connections | Devil's advocate / Unknown unknowns | H | M | Small Hikari pool, validation/timeouts, accept ~1s wake-up or use the pooled endpoint; consider paid Neon only if it hurts |
| `postgres://` URL fails in Spring | Unknown unknowns | H | L | Use JDBC URL form via Fly secrets |
| Bad migration plus image-only rollback | Unknown unknowns | M | H | Backward-compatible migrations; health check (`/actuator/health`) gates the deploy; dump before risky migrations |
| Free-tier limits change or are hit (1 GB storage, 100 CU-hours) | Research finding | L | M | Ten users and a single season are far under the limits; monitor usage; Supabase free as fallback |
| Fly cost has no free tier and third-party pricing figures may be wrong | Research finding | M | L | Check fly.io/docs/about/pricing before the first deploy; set a budget alert |
| Fly MCP server covers only "most" commands | Devil's advocate | L | L | Use the flyctl CLI directly for anything MCP does not cover |
| Extra vendor (Neon) adds a second account and credential set | Devil's advocate | M | L | Keep credentials only in Fly secrets; document the project in the deploy plan |

## Getting Started

Commands are for flyctl and Spring Boot 4.1.1 / Java 21 / Maven as pinned in this repo. Confirm flags with `fly <command> --help`, because flyctl changes often and its Java-specific `fly launch` behavior was not confirmed in the docs fetched.

1. Install flyctl and sign in: `curl -L https://fly.io/install.sh | sh`, then `fly auth login`. This is a manual step because it needs a payment method.
2. Create the Neon project (free plan) in the EU region closest to the Fly region (verify the region list in the Neon console), and copy the host, database, user and password. Enable SSL (`sslmode=require`).
3. Add a `Dockerfile` for the Maven build (outside this skill's scope) and run `fly launch --no-deploy` to create the app and `fly.toml`. Pick a nearby region (for example Warsaw `waw`, if still available; check with `fly platform regions`). Set the machine to 512MB-1GB, `internal_port = 8080`, `auto_stop_machines = false`, `min_machines_running = 1`, and an HTTP health check on `/actuator/health` (requires `spring-boot-starter-actuator`, which is not yet in `pom.xml`).
4. Set secrets: `fly secrets set SPRING_DATASOURCE_URL="jdbc:postgresql://<host>/<db>?sslmode=require" SPRING_DATASOURCE_USERNAME=... SPRING_DATASOURCE_PASSWORD=...`. The JPA and PostgreSQL dependencies must be added to `pom.xml` before the app can use this.
5. Deploy with `fly deploy`, check with `fly status` and `fly logs`, then wire `fly deploy` into GitHub Actions on merge using a deploy token from `fly tokens create deploy` stored as the `FLY_API_TOKEN` secret.

## Out of Scope

The following were not evaluated in this research:
- Docker image configuration
- CI/CD pipeline setup
- Production-scale architecture (multi-region, HA, DR)
