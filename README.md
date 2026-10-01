# Prisma schema and deterministic seed

PortfolioPilot is a teaching stock portfolio application. This activity builds its database foundation: application tables, a saved migration, reproducible sample data, and database queries that respect user ownership. The frontend still displays its earlier labeled demo fixtures; seeding the database does not update those screens yet.

## 1. Purpose: what this activity teaches

The prompt asks the coding agent to design and migrate PostgreSQL records for users, authentication sessions, portfolios, securities, buys and sells, watchlists, quotes, and news. It also asks for repeatable demo data and real database verification of ownership isolation.

A **schema** describes tables and their relationships. A **migration** is a saved SQL change to that structure. A **seed** inserts sample data. An **idempotent** seed can run repeatedly without duplicating records or changing fixture timestamps.

Students learn to:

- Design ownership relationships, indexes for common queries, and unique constraints that reject duplicates.
- Choose what happens to related records when something is deleted.
- Keep transactions as the authoritative history from which positions will be derived.
- Store financial values precisely and return them as decimal strings over JSON.
- Identify a stock listing by exchange and symbol, rather than assuming a ticker is globally unique.
- Require a trusted, session-derived owner context for private database operations.
- Verify migrations and data isolation against a fresh, disposable PostgreSQL database.

**Fixture prices and news are synthetic. Their fixed clock is January 15, 2025 at 16:00 UTC. They are not current market prices or actual news events.**

## 2. Steps performed, in order

### Inspect and initialize the workspace

The milestone folder was initially empty. The milestone 05 project was copied into it without its Git directory, dependencies, or npm cache. The preceding folder was left unchanged. The agent read the project contract, state, plan, database client, and pinned versions.

```powershell
npm ci --no-audit --no-fund
```

Installation first failed because registry access was denied. After network access was permitted, it installed 328 packages from the existing lockfile. No dependency versions were upgraded.

### Choose the identities and authentication structure

The agent checked Prisma 7 documentation and installed CLI/types, plus Better Auth's documented core database fields. [ADR 0002](docs/decisions/0002-database-ledger-and-auth.md) records the decisions.

Better Auth's database-session model was chosen for the later authentication milestone. This activity adds `User`, `Session`, `Account`, and `Verification` tables, but does not install or integrate the authentication library. Milestone 07 must verify its maintained version and generated schema before integration.

Securities use the unique pair `(exchangeMic, symbol)`. A **MIC**, or market identifier code, identifies a trading venue. Two fictional `ACME` listings on `XNAS` and `XNYS` demonstrate why a symbol alone is insufficient.

### Create the schema and initial migration

[packages/db/prisma/schema.prisma](packages/db/prisma/schema.prisma) now defines:

| Models | Purpose |
| --- | --- |
| `User`, `Session`, `Account`, `Verification` | Identity, sessions, provider accounts, and temporary verification records |
| `Portfolio` | A named USD portfolio owned by one user |
| `Security` | An exchange-aware USD stock listing |
| `PortfolioTransaction` | Buy/sell facts, including quantity, price, fees, amount, and occurrence time |
| `WatchlistEntry` | A user's saved listing |
| `QuoteSnapshot` | A provider price at a recorded time, with a synthetic-data flag |
| `NewsArticle`, `NewsArticleSecurity` | Provider news and links to mentioned listings |

Every model has creation/update timestamps stored in timezone-aware PostgreSQL columns. Indexes support owner lists, ordered transaction histories, quote/news lookup, and expiry cleanup. Unique constraints cover emails, session tokens, provider account identities, portfolio names per owner, listing identities, watchlist membership, and provider records.

Quantity, price, and fees use `numeric(28,10)`; amounts use `numeric(38,10)`. These exact decimal columns allow ten digits after the decimal point. There is no stored position or mutable share balance.

```text
BUY amount  = quantity × price + fees
SELL amount = quantity × price − fees
```

Amounts represent positive cash outflows or proceeds, rounded to ten decimal places. SQL **CHECK constraints** reject invalid numeric values, inconsistent amounts, and unsupported currency/asset types. Prisma does not represent these checks in its schema, so they were added to the generated SQL migration.

Deleting a user removes their private records; deleting a portfolio removes its transaction history. A security referenced by transactions or watchlists cannot be deleted. Removing an article removes its listing links. The [database diagram](docs/database-diagram.md) and ADR explain these choices.

The agent validated/generated the Prisma client and generated migration SQL with the installed CLI. Standard npm-accessible equivalents are:

```powershell
npx --no-install prisma validate --config packages/db/prisma.config.ts
npx --no-install prisma generate --config packages/db/prisma.config.ts
npx --no-install prisma migrate diff --config packages/db/prisma.config.ts --from-empty --to-schema packages/db/prisma/schema.prisma --script --output packages/db/prisma/base.sql
```

The generated table SQL was combined with the reviewed checks in [the initial migration](packages/db/prisma/migrations/20261001000000_initial/migration.sql). Students should apply that existing migration rather than regenerate it. [prisma.config.ts](packages/db/prisma.config.ts) supplies datasource and migration configuration.

### Add owner-scoped repositories and decimal contracts

[repositories.ts](packages/db/src/repositories.ts) resolves an unexpired database session into an opaque **owner context**: a trusted object identifying the current user. A fabricated `{ userId }` is rejected before a query runs.

Portfolio reads and mutations, transaction reads, and watchlist operations enforce ownership. Quote/news reads only include listings referenced by the owner's ledger or watchlist. Contexts must be resolved for each request and not cached across logout. This is a server-side database boundary; cookie parsing and sign-in endpoints are still future work.

[packages/contracts/src/index.ts](packages/contracts/src/index.ts) adds listing and transaction formats. A **DTO**, or data transfer object, describes data passed between application layers. Financial fields use strings. Serialization uses `toFixed()` to preserve small values without scientific notation:

```json
{
  "quantity": "0.0000000001",
  "price": "1",
  "fees": "0",
  "amount": "0.0000000001"
}
```

This is an illustrative excerpt, not a new HTTP endpoint. The tiny quantity and amount serialization passed integration checks.

### Add a fixed-clock, idempotent seed

[seed.ts](packages/db/src/seed.ts) accepts a **fixture clock**, a supplied clock returning a reproducible time. Fixed IDs and atomic upserts keep records stable. An **upsert** updates an existing record or inserts it if missing. A PostgreSQL advisory lock serializes concurrent fixture runs.

The seed creates two distinct demo users, three portfolios, three fictional listings, five buys/sells, two watchlist entries, three synthetic quotes, three synthetic articles, and three article/listing links. It creates no passwords, authentication accounts, verification records, or sessions.

[seed-cli.ts](packages/db/src/seed-cli.ts) requires explicit opt-in and a loopback PostgreSQL URL. It rejects production. Build, migrations, and server startup do not seed automatically.

### Verify and update the learning records

An isolated PostgreSQL 17.6 container was started on localhost port 5546, separate from the normal Compose volumes. The first migration attempt failed before application tables were created because the generated base SQL was missing. After schema-engine execution was permitted, the complete SQL was generated. The failed attempt was marked rolled back using `migrate resolve --rolled-back 20261001000000_initial`, then successfully deployed. No database reset ran.

The final migration was applied directly to a second entirely empty database, `portfolio_m06_verify_final`. The real-database verifier passed there. The compiled seed CLI also ran twice successfully.

Files added or changed included the database schema/config/migration/lock, database package scripts, [client exports](packages/db/src/index.ts), repositories, seed and CLI, [boundary tests](packages/db/test/boundaries.test.ts), [database verifier](packages/db/test/verify-database.mjs), shared contracts and [ledger tests](packages/contracts/test/ledger.test.ts). The diagram, ADR/index, [lesson notes](docs/lessons/06-database-and-seed.md), versions, README, and [project state](docs/project-state.md) were also updated.

## 3. Results achieved and observed

The database foundation stores the core records and rejects invalid row values and duplicate identities. Repository integration checks proved that Alice cannot read or mutate Bob's private portfolio data. Sequential and concurrent seed runs produced identical fixture rows and timestamps.

Both observed CLI seed runs printed:

```text
Synthetic demo seeded at 2025-01-15T16:00:00Z; prices are not current market data.
```

The database verifier printed:

```text
PASS: fresh migration, sequential/concurrent seed idempotency, fixture clock, synthetic labels and exchange-aware identity
PASS: anonymous/forged/expired/revoked contexts; cross-owner reads and writes; decimal string DTOs
PASS: SQL numeric/USD/check constraints, unique identities, restrictive security deletion and owner cascades
```

| Check run during implementation | Observed result |
| --- | --- |
| Prisma 7.10.0 validate/generate | Passed |
| Final migration deployed directly to a fresh PostgreSQL 17.6 database | Passed |
| Repeated/concurrent seed and ownership integration verifier | Passed, including tiny decimal serialization |
| Compiled seed CLI run twice with local opt-in | Passed |
| `npm run build` | Passed with existing warnings |
| `npm run typecheck` | Passed |
| `npm run test` | Passed: 27 unit tests |
| `node scripts/check-browser-boundary.mjs` | Passed: browser dependencies stayed within web, config, and contracts |
| Optional Prisma engine schema-drift comparison | Failed with database connection error `P1001` |

These are recorded implementation results. This documentation update does not claim a new application test run. The frontend remains unchanged: it still shows its labeled sample portfolios, watchlist, and news.

## 4. How to run and verify

### Prerequisites and installation

Use Node.js **24.21.0**, npm **11.19.0**, and Docker Desktop with an accessible Linux container engine. Network access is needed for installation and the first image download. Ports 5432/6379 serve normal local PostgreSQL/Redis, 5546 serves the isolated database test, and 3001/5173 serve the application.

Run PowerShell commands from this directory. No AI or market-data credentials are needed for this database activity. The example database password is public local teaching data, not a production credential.

```powershell
node --version
npm --version
docker info
npm ci --no-audit --no-fund
npm run build
```

On the recorded Windows host, inactive Node/npm shims were bypassed with the installed runtime:

```powershell
$nodeDir = 'C:/Users/luisc/AppData/Local/Author Software/nvm/installs/v24.21.0'
$env:PATH = "$nodeDir;$env:PATH"
& "$nodeDir/node.exe" "$nodeDir/node_modules/npm/bin/npm-cli.js" run build
```

Replace `run build` with the required npm command. Other machines should use their own installation path or normal npm commands.

### Migrate and seed the normal local database

```powershell
npm run infra:start
npm run infra:status
$env:DATABASE_URL = 'postgresql://portfolio_local:local_only_change_me@127.0.0.1:5432/portfolio_pilot'
npm run migrate:deploy --workspace=@portfolio-pilot/db
$env:NODE_ENV = 'development'
$env:ALLOW_DEMO_SEED = 'true'
npm run seed:demo --workspace=@portfolio-pilot/db
npm run seed:demo --workspace=@portfolio-pilot/db
Remove-Item Env:ALLOW_DEMO_SEED
```

**Expected:** migration succeeds and both seed runs print the synthetic-data message. An initially empty database receives the counts listed above. These behaviors were observed on the isolated port-5546 databases; the normal Compose/API smoke test remains pending.

Set `DATABASE_URL` in the terminal for CLI commands. The API's `.env.local` file does not automatically set variables in a separate terminal.

### Verify on a fresh disposable database

The verifier requires an empty, migrated database named `portfolio_m06_verify` or `portfolio_m06_verify_final`. It changes and deletes fixtures to test deletion rules. Do not seed it first or point it at the application database.

Choose an unused container name and ensure port 5546 is free. If the implementation's earlier verification container is still running, stop it first:

```powershell
# Only if the earlier verification container is running:
docker stop portfolio-pilot-m06-verify

# Start a NEW disposable container, separate from normal Compose services.
docker run --detach --name portfolio-pilot-m06-student --publish 127.0.0.1:5546:5432 --env POSTGRES_USER=portfolio_local --env POSTGRES_PASSWORD=local_only_change_me --env POSTGRES_DB=portfolio_m06_verify postgres:17.6-alpine
docker exec portfolio-pilot-m06-student pg_isready -U portfolio_local -d portfolio_m06_verify
```

Repeat `pg_isready` until it reports `accepting connections`, then run:

```powershell
$env:DATABASE_URL = 'postgresql://portfolio_local:local_only_change_me@127.0.0.1:5546/portfolio_m06_verify'
npm run migrate:deploy --workspace=@portfolio-pilot/db
$env:NODE_ENV = 'test'
npm run verify:database --workspace=@portfolio-pilot/db
npm run test
npm run typecheck
npm run check:browser-boundary
```

**Expected:** the verifier prints the three PASS groups above and the project checks pass. It compares fixture rows/timestamps after repeated and concurrent seeds; checks missing, fabricated, expired, and revoked contexts; tests cross-owner reads/mutations; and exercises numeric checks, uniqueness, restrictive security deletion, and owner cascades. Test session tokens are random and are not printed.

The verifier should refuse a repeat run against the populated database. For another run, stop the container and create a new container with an unused name; update commands accordingly. Stopping a container does not empty its database.

```powershell
docker stop portfolio-pilot-m06-student
```

### Run the application and retained health checks

Restore the normal application URL and development mode. Preserve an existing environment file:

```powershell
$env:NODE_ENV = 'development'
$env:DATABASE_URL = 'postgresql://portfolio_local:local_only_change_me@127.0.0.1:5432/portfolio_pilot'
npm run infra:start
if (-not (Test-Path apps/api/.env.local)) {
  Copy-Item apps/api/.env.example apps/api/.env.local
}
npm run dev
```

Open `http://127.0.0.1:5173`. The frontend still uses its own fixtures. In another terminal:

```powershell
Invoke-WebRequest http://127.0.0.1:5173/api/health/live
Invoke-WebRequest http://127.0.0.1:5173/api/health/ready
```

**Liveness** checks whether the API responds; **readiness** checks PostgreSQL and Redis. Expected status is 200 when both services are healthy. These API responses passed simulated unit tests; the full running API/Compose readiness transition was not verified. [Lesson 05](docs/lessons/05-local-infrastructure.md) retains the service-stop demonstration. `npm run infra:stop` stops normal services without deleting their named volumes. Stop development servers with Ctrl+C.

### Retained Assistant demo from milestone 04

In mock mode, open `http://127.0.0.1:5173/assistant`, enter a question, and select **Send message**. The earlier activity verified a visible response:

```text
[Mock answer] This is a deterministic demo response. Portfolio analysis and current market facts are unavailable in this question-and-answer demo.
```

With Chrome installed and development servers running, the existing browser tests can be run with:

```powershell
npm run test:browser --workspace=@portfolio-pilot/web
```

Three Chrome tests passed in milestone 04, not during this database activity. Live Claude mode remains optional and unverified. It needs a server-side Anthropic application API key, an available model ID, and an absolute SDK workspace outside this repository. [Lesson 04](docs/lessons/04-first-agent-slice.md) documents setup and limits: no built-in tools, one turn, a USD 0.02 estimated SDK budget, and a 20-second timeout. Never put credentials in browser variables or committed files.

## 5. Limitations and unfinished work

- **No sign-in yet.** The schema and repository boundary are implemented, but auth library integration, cookies, demo login, and Microsoft Entra OIDC remain milestone 07. Seeded users have no login credentials. Resolve owner contexts per request rather than caching them across logout.
- **No database-backed portfolio UI or transaction API yet.** Trade creation, pagination cursors, aggregate oversell checks under concurrent writes, and weighted-average valuation follow in milestones 08/09. Row constraints cannot prove aggregate long-only holdings.
- **The optional schema-drift check failed.** A drift check compares the live structure with the Prisma schema. It reported `P1001` despite passing adapter tests and container readiness. The cause is unresolved. With a verification database running and engine connectivity fixed, retry:

  ```powershell
  $env:DATABASE_URL = 'postgresql://portfolio_local:local_only_change_me@127.0.0.1:5546/portfolio_m06_verify'
  npx --no-install prisma migrate diff --config packages/db/prisma.config.ts --from-config-datasource --to-schema packages/db/prisma/schema.prisma --exit-code
  ```

  This retry has not passed. Prisma schema comparison does not cover manual SQL CHECK constraints; their rejection behavior passed real database tests.
- **Existing build warnings remain.** Vite reported third-party `use client` warnings; Next.js reported two Node/Edge analysis warnings for the shutdown hook. Builds passed.
- **Earlier live checks remain pending.** Database container access succeeded with appropriate permissions in milestone 06, but the full API/Redis readiness smoke test and credentialed Claude request were not verified.
- **The Assistant endpoint is temporary.** Its unauthenticated `/api/demo/ask` route is restricted to local development. Authenticated run endpoints, stored chat, streaming, portfolio grounding, and news citations are future work.
- **No commit or deployment occurred.** This workspace was not a Git repository; nothing was pushed or provisioned in the cloud.

## Further reading

- [Project contract](AGENTS.md) and [milestone plan](docs/project-plan.md)
- [Project state and observed checks](docs/project-state.md)
- [Lesson 06](docs/lessons/06-database-and-seed.md)
- [Database diagram](docs/database-diagram.md)
- [Database/auth decision](docs/decisions/0002-database-ledger-and-auth.md)
- [Pinned versions](docs/versions.md)
