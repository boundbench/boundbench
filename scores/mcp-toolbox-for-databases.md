# Defense-in-Depth Score: MCP Toolbox for Databases

**Repo:** https://github.com/googleapis/mcp-toolbox · **Commit:** `a24e5e68567fa014a42fc6cc711faa82b964d4c0` · **Reviewed:** 2026-10-03
**What it is:** Open source MCP server for ~30 databases (AlloyDB, Cloud SQL, BigQuery, Spanner, Postgres, MySQL...)
**Category:** Data & Analytics
**Scored configuration:** README quick start: `npx -y @toolbox-sdk/server --prebuilt=postgres --stdio` with POSTGRES_* environment variables, all prebuilt postgres tools (no toolset suffix), default flags.
**Agent surface (default):** code execution yes · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 3.8 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L0 | L2 | 0.25 | G1 | **0.25** (alt) | High |
| C2 | Approval gates | L1 | L2 | L2 | L0 | 0.33 | — | **0.33** | High |
| C3 | Tool & action scoping | L2 | L1 | L2 | L0 | 0.33 | — | **0.33** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L0 | 0.07 | C5-WORSTCASE | **0.07** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C9 | Audit & traceability | L1 | L2 | L0 | L1 | 0.28 | G1 | **0.28** (alt) | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |

Controls where a risk surface exists: 1.80 / 8.0 (22%); 2 criteria scored SA (surface absent).

As shipped for IDE use, Toolbox hands the connected AI an execute_sql tool that runs any SQL against your database with the full privileges of the login you configure, with no read-only mode, timeout, row cap, or authorization check for the postgres source. Its safety depends almost entirely on the database role you give it and on your MCP host's approval prompts; the tool is correctly marked destructive so hosts can gate it. Tool calls are not recorded unless debug logging or telemetry export is enabled. Stronger controls (session-locked read-only mode, BigQuery write blocking, per-tool auth) exist for some sources but are opt-in.

## Critical gaps
- In the default postgres prebuilt, a hijacked session can both read sensitive data and run irreversible SQL (DROP/DELETE) through execute_sql with no server-side gate. (ASI01, ASI02, LLM01; C5) — [internal/tools/postgres/postgresexecutesql/postgresexecutesql.go:103](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/tools/postgres/postgresexecutesql/postgresexecutesql.go#L103); [internal/prebuiltconfigs/tools/postgres.yaml:26-29](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/prebuiltconfigs/tools/postgres.yaml#L26-L29)

## Criterion details

### C1 Identity & least privilege — 0.25 (high)

The server connects to Postgres with whatever database user and password the operator puts in environment variables and uses that one static credential for every tool call, reads and writes alike. Nothing in the server narrows it, and in the default stdio mode there is no per-caller authorization at all. An opt-in feature (auth services with authRequired tools and parameters filled from verified ID-token claims) can tie individual custom tools to an authenticated caller over HTTP, but it never narrows the database credential and tools without authRequired stay open. For the Cloud SQL, AlloyDB and BigQuery prebuilts the server instead uses the operator's Application Default Credentials.

- **default configuration** (default; raw 0.17 → 0.17)
  - **S L1:** A single operator-supplied database credential (POSTGRES_USER/POSTGRES_PASSWORD) is used for all tools; it is a dedicated DB login but its privilege is whatever the operator chose, and read and write share it. — [internal/prebuiltconfigs/tools/postgres.yaml:21-22](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/prebuiltconfigs/tools/postgres.yaml#L21-L22); [internal/sources/postgres/postgres.go:118](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/sources/postgres/postgres.go#L118) (verified)
    - *To reach the next level:* No per-tool or read/write credential split and no deterministic authorization gate mapped to database privileges.
  - **C L0:** In the scored stdio mode no authorization check runs on any tool call: the auth-service block is skipped when headers are nil, and tools without authRequired are authorized unconditionally. — [internal/server/mcp/v20251125/method.go:270](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/server/mcp/v20251125/method.go#L270); [internal/tools/tools.go:163-165](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/tools/tools.go#L163-L165) (verified)
    - *To reach the next level:* No authorization layer covers the main tool path in the default configuration.
  - **D L0:** The default install runs with whatever role the operator configures (commonly the database owner); the plain postgres source has no read-only mode, so least privilege depends entirely on manual role hardening outside the server. — [internal/sources/postgres/postgres.go:100-101](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/sources/postgres/postgres.go#L100-L101); [internal/prebuiltconfigs/tools/postgres.yaml:21-22](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/prebuiltconfigs/tools/postgres.yaml#L21-L22) (verified)
    - *To reach the next level:* No narrower default: a read-only or minimal mode does not exist for the postgres source.
  - **B L2:** A hijacked session gets everything the configured database login can do on one database system, including DDL and DELETE through execute_sql. — [internal/tools/postgres/postgresexecutesql/postgresexecutesql.go:103](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/tools/postgres/postgresexecutesql/postgresexecutesql.go#L103); [internal/prebuiltconfigs/tools/postgres.yaml:26-29](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/prebuiltconfigs/tools/postgres.yaml#L26-L29) (verified)
    - *To reach the next level:* Writes are not limited to non-destructive operations or a single schema/tenant.
- **opt-in auth services: authRequired tools and claim-bound authenticated parameters (HTTP mode, custom tools.yaml)** (alt; raw 0.25, cap G1 → 0.25) ← counted
  - **S L1:** Tools can require a verified token from a named auth service and bind parameters to token claims, so a custom tool can be limited to the authenticated principal's rows, but authorization is only 'holds a token from service X' and the database credential stays one broad static login. — [internal/server/mcp/v20251125/method.go:303](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/server/mcp/v20251125/method.go#L303); [internal/util/parameters/parameters.go:156](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/util/parameters/parameters.go#L156) (verified)
    - *To reach the next level:* No role-scoped credential: the DB login is not narrowed per tool or per verb.
  - **C L1:** Only tools that list authRequired are checked; any tool without it, including generic execute_sql tools, is authorized unconditionally, and stdio sessions skip auth entirely. — [internal/tools/tools.go:163-165](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/tools/tools.go#L163-L165); [internal/server/mcp/v20251125/method.go:270](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/server/mcp/v20251125/method.go#L270) (verified)
    - *To reach the next level:* Authorization does not cover every tool path, and unannotated tools are allowed rather than denied.
  - **D L0:** Auth services and authRequired are opt-in configuration; the prebuilt postgres config declares none. — [internal/prebuiltconfigs/tools/postgres.yaml:26-29](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/prebuiltconfigs/tools/postgres.yaml#L26-L29) (verified)
    - *To reach the next level:* Not on by default.
  - **B L2:** If the per-tool check is bypassed, the same database-wide credential applies. — [internal/tools/postgres/postgresexecutesql/postgresexecutesql.go:103](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/tools/postgres/postgresexecutesql/postgresexecutesql.go#L103) (verified)
    - *To reach the next level:* Writes are not limited to non-destructive operations or a single tenant.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.
- **Notes:** C1-PASSTHRU was considered: BigQuery and several Google sources offer useClientOAuth, which forwards the caller's OAuth token to the downstream API (e.g. internal/prebuiltconfigs/tools/bigquery.yaml:22, default false). It is off in the scored postgres configuration, so the cap was not applied; deployments that enable it should expect it.

### C2 Approval gates — 0.33 (high)

As a tool server, Toolbox leaves approval to the MCP host and its contribution is risk signalling. Every postgres tool carries MCP annotations: execute_sql is marked destructive and the catalog/listing tools read-only, and other config-defined SQL tools default to destructive. But the main SQL tool mixes reads and writes, there is no read-only SQL tool or dry-run, and the plain postgres source has no server-enforced read-only mode (Cloud SQL, AlloyDB and BigQuery sources do, opt-in). A wrongly approved execute_sql call can DROP or DELETE production data irreversibly.

- **S L1:** Annotations are present and conservative (execute_sql destructiveHint true), but the one SQL tool mixes reads and writes and no separate read-only SQL tool, dry-run, or server-enforced read-only mode exists for postgres. — [internal/tools/postgres/postgresexecutesql/postgresexecutesql.go:74](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/tools/postgres/postgresexecutesql/postgresexecutesql.go#L74); [internal/tools/tools.go:90-97](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/tools/tools.go#L90-L97); [internal/sources/postgres/postgres.go:100-101](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/sources/postgres/postgres.go#L100-L101) (verified)
  - *To reach the next level:* No separate read and write SQL tools with accurate hints; no dry-run or read-only mode.
- **C L2:** All 24 postgres tool types set default annotations through GetAnnotationsOrDefault, so every tool reaches the host with risk hints. — searched `rg -n GetAnnotationsOrDefault` in `internal/tools/postgres` → 24 hits (one per postgres tool type (24 directories); 22 read-only defaults, 2 destructive (execute-sql, postgres-sql)); [internal/tools/postgres/postgreslisttables/postgreslisttables.go:153](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/tools/postgres/postgreslisttables/postgreslisttables.go#L153) (verified)
  - *To reach the next level:* Hints cover every tool but the mechanism itself is only advisory signalling (coverage capped one level above strength).
- **D L2:** Annotations are on by default and come from the embedded prebuilt config; only an operator-written custom config can override them. — [internal/prebuiltconfigs/prebuiltconfigs.go:25](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/prebuiltconfigs/prebuiltconfigs.go#L25); [internal/tools/postgres/postgresexecutesql/postgresexecutesql.go:74](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/tools/postgres/postgresexecutesql/postgresexecutesql.go#L74) (verified)
  - *To reach the next level:* Strength caps this; no server-enforced confirmation or read-only mode to default on.
- **B L0:** execute_sql passes any statement to the production database, so a wrongly approved call can irreversibly drop or delete data with no preview or undo. — [internal/tools/postgres/postgresexecutesql/postgresexecutesql.go:103](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/tools/postgres/postgresexecutesql/postgresexecutesql.go#L103); [internal/sources/postgres/postgres.go:118](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/sources/postgres/postgres.go#L118) (verified)
  - *To reach the next level:* No dry-run, transaction preview, or rollback for destructive statements.
- **Cap:** none
- **Notes:** Server-enforced read-only modes exist for other sources: Cloud SQL PostgreSQL/AlloyDB lock the session read-only (internal/sources/cloudsqlpg/cloud_sql_pg.go:181), and BigQuery offers writeMode blocked/protected. They are opt-in (env default false) and unavailable for the scored postgres source.

### C3 Tool & action scoping — 0.33 (high)

Toolbox has a real argument-validation mechanism: typed parameters and fixed SQL statements that receive arguments as bind parameters ($1), which is how most of the 29 prebuilt postgres tools work. However, the default tool set also includes execute_sql, which runs any SQL the model writes, and get_query_plan, which pastes a model string into the statement with a Go text template (its own description warns about SQL injection). Toolsets can be selected with --prebuilt=postgres/<toolset>, but the default loads every tool, so the agent can touch any table the credential reaches.

- **S L2:** Typed parameters with bind-parameter statements are a sound mechanism, but template parameters do raw string substitution into SQL and are escapable by design. — [internal/prebuiltconfigs/tools/postgres.yaml:126](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/prebuiltconfigs/tools/postgres.yaml#L126); [internal/util/parameters/parameters.go:289-294](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/util/parameters/parameters.go#L289-L294) (verified)
  - *To reach the next level:* Template parameters are not restricted to allowlisted values, so the mechanism is escapable.
- **C L1:** The most powerful tool, execute_sql, takes raw SQL, and get_query_plan injects a raw string; only the fixed listing tools are bound by validated parameters. — [internal/tools/postgres/postgresexecutesql/postgresexecutesql.go:103](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/tools/postgres/postgresexecutesql/postgresexecutesql.go#L103); [internal/prebuiltconfigs/tools/postgres.yaml:178](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/prebuiltconfigs/tools/postgres.yaml#L178) (verified)
  - *To reach the next level:* The main SQL tool and the plan tool bypass parameter validation.
- **D L2:** Toolsets (data, monitor, health, view-config, replication) are selectable, but with no toolset suffix all tools load, including execute_sql in the default. — [cmd/internal/options.go:240](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/cmd/internal/options.go#L240); [internal/prebuiltconfigs/tools/postgres.yaml:260-263](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/prebuiltconfigs/tools/postgres.yaml#L260-L263) (verified)
  - *To reach the next level:* No read-only default tool set; write-capable execute_sql ships enabled.
- **B L0:** execute_sql reaches any table, schema, or DDL the credential allows, with no row, table, or statement-type bounds. — [internal/tools/postgres/postgresexecutesql/postgresexecutesql.go:103](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/tools/postgres/postgresexecutesql/postgresexecutesql.go#L103); [internal/prebuiltconfigs/tools/postgres.yaml:26-29](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/prebuiltconfigs/tools/postgres.yaml#L26-L29) (verified)
  - *To reach the next level:* No scoping to schemas/tables or quantity bounds on SQL.
- **Cap:** none

### C4 Code-execution isolation — 0.05 (high)

The code-execution surface here is SQL: execute_sql (and the template-built get_query_plan) send model-written statements straight to the database through the connection pool. There is no isolation boundary: no read-only transaction, statement-type filter, or sandboxed role is applied for the postgres source, and nothing is opt-in either. What that SQL can reach is the whole database under the configured login; with a superuser login Postgres features such as COPY ... PROGRAM extend that to the database host.

- **S L0:** Model-written SQL is executed directly on the target database session with no isolation primitive. — [internal/tools/postgres/postgresexecutesql/postgresexecutesql.go:103](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/tools/postgres/postgresexecutesql/postgresexecutesql.go#L103); [internal/sources/postgres/postgres.go:118](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/sources/postgres/postgres.go#L118) (verified)
  - *To reach the next level:* No isolation such as a read-only or restricted session for model SQL.
- **C L0:** No execution path is isolated; both execute_sql and template-substituted statements run unconfined. — [internal/util/parameters/parameters.go:289-294](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/util/parameters/parameters.go#L289-L294); [internal/tools/postgres/postgresexecutesql/postgresexecutesql.go:103](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/tools/postgres/postgresexecutesql/postgresexecutesql.go#L103) (verified)
  - *To reach the next level:* No path goes through an isolation boundary.
- **D L0:** No isolation exists to be on by default for the postgres source (IsReadOnly is hard-coded false). — [internal/sources/postgres/postgres.go:100-101](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/sources/postgres/postgres.go#L100-L101) (verified)
  - *To reach the next level:* No default-on isolation.
- **B L1:** Model SQL runs with the full privileges of the configured login across the database, including destructive DDL; with a superuser login Postgres server-side features can reach the DB host (inferred from Postgres behaviour, not this repo). — [internal/tools/postgres/postgresexecutesql/postgresexecutesql.go:103](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/tools/postgres/postgresexecutesql/postgresexecutesql.go#L103); [internal/prebuiltconfigs/tools/postgres.yaml:21-22](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/prebuiltconfigs/tools/postgres.yaml#L21-L22) (verified)
  - *To reach the next level:* Not confined to a schema or read-only session; credential not scoped.
- **Cap:** none

### C5 Untrusted input blast radius — 0.07 (high)

Database rows can contain attacker-written text (customer records, comments, tickets), and Toolbox returns them to the model as plain JSON text with nothing marking them as untrusted. The same default session can read sensitive data and run destructive SQL, so a successful injection could make the host read data and delete or alter it with no server-side barrier. The server offers no read-only or no-write mode for postgres that would drop one of those legs.

- **S L1:** Tool results are serialized rows in plain text content, with no provenance or untrusted flag. — [internal/server/mcp/v20251125/method.go:447](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/server/mcp/v20251125/method.go#L447); searched `rg -n -i "untrusted|provenance"` in `internal/server/mcp/v20251125/method.go internal/tools/postgres/postgresexecutesql/postgresexecutesql.go` → 0 hits (verified)
  - *To reach the next level:* No structured separation of content and metadata, and no provenance on returned data.
- **C L0:** No returned source is distinguished from any other. — [internal/server/mcp/v20251125/method.go:447](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/server/mcp/v20251125/method.go#L447) (verified)
  - *To reach the next level:* No source is marked or limited.
- **D L0:** There is no mechanism to enable. — [internal/server/mcp/v20251125/method.go:447](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/server/mcp/v20251125/method.go#L447) (verified)
  - *To reach the next level:* No default-on mechanism.
- **B L0:** A hijacked session can read sensitive database data and run irreversible writes (DROP/DELETE) through execute_sql in the same session, with no server-side approval. — [internal/tools/postgres/postgresexecutesql/postgresexecutesql.go:103](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/tools/postgres/postgresexecutesql/postgresexecutesql.go#L103); [internal/prebuiltconfigs/tools/postgres.yaml:26-29](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/prebuiltconfigs/tools/postgres.yaml#L26-L29) (verified)
  - *To reach the next level:* Writes and data reads are not separated or gated in the default configuration.
- **Cap:** C5-WORSTCASE — In the default configuration the same session reads sensitive data and can execute irreversible SQL with no server-side gate.

### C6 Memory, context & configuration integrity — 1.00 (high)

In the scored configuration (--prebuilt=postgres) the tool definitions come from YAML embedded in the binary and the connection settings come from the process environment; the server keeps no memory, writes nothing back, and loads no workspace files. Note that a different mode behaves differently: when started with neither --prebuilt nor --config, the server loads tools.yaml from the current directory and watches it for changes, so a launch from inside an untrusted repository would take that repository's tool definitions.

- **Structural absence:** searched `rg -n -i "godotenv|dotenv"` in `cmd internal main.go` → 0 hits (no .env auto-loading anywhere); searched `rg -n "os\.WriteFile|os\.Create\(|os\.OpenFile\("` in `internal/server/server.go internal/server/mcp.go internal/server/mcp internal/sources/postgres/postgres.go internal/tools/postgres internal/prebuiltconfigs` → 0 hits (no persistence written by the server or postgres tools)
- **Notes:** Unscored no-flag mode: cmd/internal/options.go:176-179 falls back to ./tools.yaml when no --prebuilt and no --config is given, and dynamic reload is on unless --disable-reload is passed. That would trigger C6-REPOCONFIG if scored.

### C7 Third-party extensions — 1.00 (high)

The server loads no third-party code at runtime in the scored configuration: tool types and database drivers are compiled into the Go binary, and the prebuilt configs are embedded. It does not launch other MCP servers, install packages, or load plugins. The only process spawn in the codebase is the optional Dataform compile-local tool, which is not part of the postgres prebuilt.

- **Structural absence:** searched `rg -n "exec\.Command|plugin\.Open"` in `internal/server internal/sources/postgres internal/tools/postgres internal/prebuiltconfigs cmd main.go` → 0 hits (elsewhere the only non-test hit is internal/tools/dataform/dataformcompilelocal (not in the scored config)); searched `rg -n -i "npx|pip install|go install|trust_remote_code"` in `internal/server internal/sources/postgres internal/tools/postgres internal/prebuiltconfigs/prebuiltconfigs.go` → 0 hits
- **Notes:** Install-time supply chain (out of scope here): the npm platform packages download the binary from storage.googleapis.com in a postinstall script without a checksum check (npm/server-darwin-arm64/scripts/downloadBinary.js), and the README launches it with an unpinned `npx -y @toolbox-sdk/server`.

### C8 Secrets & sensitive-data protection — 0.25 (high)

The database password arrives through an environment variable and is kept as a plain string in the source config and the connection URL; there is no secret type or redaction layer in the server. It is not sent to the model, telemetry export is opt-in, and the default info log level does not record SQL or parameters. Debug logging, which is a single flag away, logs full SQL text and parameters unredacted, and query results (which may contain sensitive data) go to the model unfiltered.

- **S L1:** Secrets come from environment variables into a plain string field; no masking or redaction exists in the server. — [internal/sources/postgres/postgres.go:59](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/sources/postgres/postgres.go#L59); [internal/prebuiltconfigs/tools/postgres.yaml:21-22](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/prebuiltconfigs/tools/postgres.yaml#L21-L22); searched `rg -n -i "redact|mask"` in `internal/sources/postgres/postgres.go internal/server/server.go internal/server/mcp.go internal/log cmd/internal/options.go cmd/internal/config.go` → 0 hits (verified)
  - *To reach the next level:* No type-level masking or log redaction on main paths.
- **C L1:** The only protection is that the default log level omits payloads; debug logs, tool results to the model, and errors are not filtered. — [internal/server/mcp/v20251125/method.go:350](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/server/mcp/v20251125/method.go#L350); [internal/tools/postgres/postgresexecutesql/postgresexecutesql.go:101](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/tools/postgres/postgresexecutesql/postgresexecutesql.go#L101) (verified)
  - *To reach the next level:* Logs and transcripts are not redacted.
- **D L1:** Telemetry exporters are opt-in and default logging is at info, which omits SQL and parameters, but a content-free startup version check calls GitHub by default and verbose debug logging (full SQL and parameters) is one flag away with no redaction. — [cmd/internal/flags.go:35-37](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/cmd/internal/flags.go#L35-L37); [internal/server/config.go:152](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/server/config.go#L152); [cmd/internal/flags.go:41](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/cmd/internal/flags.go#L41) (verified)
  - *To reach the next level:* No redaction layer exists to keep on, and the content-free version check is on by default rather than opt-in.
- **B L1:** A leaked POSTGRES_PASSWORD is a long-lived database login with whatever privileges the operator gave it. — [internal/prebuiltconfigs/tools/postgres.yaml:21-22](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/prebuiltconfigs/tools/postgres.yaml#L21-L22) (verified)
  - *To reach the next level:* Credentials are not short-lived or rotated by the server.
- **Cap:** none

### C9 Audit & traceability — 0.28 (high)

In the default configuration, tool calls leave no record: the tool name and parameters are only logged at debug level, the default level is info, and in stdio mode there is no request log. OpenTelemetry tracing is built in and creates a span per tool call with the tool name, but exporters are off unless the operator passes --telemetry-otlp or --telemetry-gcp, and the spans do not include arguments or caller identity. There is no tamper-evident audit trail.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Tool invocations are logged only at debug level. — [internal/server/mcp/v20251125/method.go:172](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/server/mcp/v20251125/method.go#L172); [internal/server/mcp/v20251125/method.go:350](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/server/mcp/v20251125/method.go#L350); [internal/server/config.go:152](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/server/config.go#L152) (verified)
    - *To reach the next level:* No record of tool calls at the default log level.
  - **C L0:** Nothing about successful tool calls is recorded by default. — searched `rg -n "InfoContext|WarnContext"` in `internal/server/mcp/v20251125/method.go` → 0 hits (tools/call handler has no info/warn-level logging) (verified)
    - *To reach the next level:* No tool path is recorded by default.
  - **D L0:** Any useful record requires --log-level debug or an opt-in telemetry exporter. — [cmd/internal/flags.go:35-37](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/cmd/internal/flags.go#L35-L37) (verified)
    - *To reach the next level:* Not on by default.
  - **B L0:** With no record, actions proceed unlogged. — [internal/server/mcp/v20251125/method.go:370](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/server/mcp/v20251125/method.go#L370) (verified)
    - *To reach the next level:* No per-action record.
- **opt-in OpenTelemetry export (--telemetry-otlp / --telemetry-gcp)** (alt; raw 0.28, cap G1 → 0.28) ← counted
  - **S L1:** Spans and metrics per tool call carry tool name, operation and error type, exported via OTLP or to Google Cloud, but not arguments or caller identity. — [internal/server/mcp/v20251125/method.go:176-179](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/server/mcp/v20251125/method.go#L176-L179); [cmd/internal/flags.go:35-37](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/cmd/internal/flags.go#L35-L37) (verified)
    - *To reach the next level:* No arguments or actor attribution in the record.
  - **C L2:** Every tools/call passes through the instrumented handler, regardless of tool type. — [internal/server/mcp/v20251125/method.go:370](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/server/mcp/v20251125/method.go#L370) (verified)
    - *To reach the next level:* Approvals, config changes and credential use are not recorded.
  - **D L0:** Exporters are off unless flagged. — [cmd/internal/flags.go:35-37](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/cmd/internal/flags.go#L35-L37) (verified)
    - *To reach the next level:* Not on by default.
  - **B L1:** Spans use a batching exporter, so records are best-effort and flushed late. — [internal/telemetry/telemetry.go:127](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/telemetry/telemetry.go#L127) (verified)
    - *To reach the next level:* Records are not durable per action.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C10 Limits & kill switch — 0.25 (high)

For the postgres source the server puts no time, row or size bound on queries: execute_sql runs until the database finishes, results are fully buffered, and the server has no MCP cancellation handling. Some listing tools take an optional limit with a default of 50, but the caller can raise it. An operator can add a statement_timeout through POSTGRES_QUERY_PARAMS, but it is empty by default. Other sources do better (BigQuery caps rows at 50 by default and supports a billing limit).

- **S L1:** Only caller-chosen limits exist (optional limit parameters); there is no server-enforced timeout, row cap or output cap for postgres queries. — [internal/prebuiltconfigs/tools/postgres.yaml:126](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/prebuiltconfigs/tools/postgres.yaml#L126); searched `rg -n "WithTimeout|statement_timeout|notifications/cancelled"` in `internal/sources/postgres/postgres.go internal/tools/postgres/postgresexecutesql/postgresexecutesql.go internal/server/mcp.go internal/server/mcp/v20251125/method.go` → 0 hits (verified)
  - *To reach the next level:* No server-enforced caps such as a statement timeout or maximum rows.
- **C L1:** Default limits exist only on a few listing tools; execute_sql has none. — [internal/tools/postgres/postgresexecutesql/postgresexecutesql.go:103](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/tools/postgres/postgresexecutesql/postgresexecutesql.go#L103) (verified)
  - *To reach the next level:* Limits do not cover the main SQL tool.
- **D L1:** Defaults (limit 50) can be raised by the caller per call, and statement_timeout is only available if the operator sets query params. — [internal/prebuiltconfigs/tools/postgres.yaml:23](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/prebuiltconfigs/tools/postgres.yaml#L23); [internal/prebuiltconfigs/tools/postgres.yaml:126](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/prebuiltconfigs/tools/postgres.yaml#L126) (verified)
  - *To reach the next level:* No sensible default the caller cannot raise.
- **B L1:** A runaway query has no ceiling, and killing the server does not reliably stop a query already running in Postgres. — [internal/sources/postgres/postgres.go:118](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/sources/postgres/postgres.go#L118); [internal/tools/postgres/postgresexecutesql/postgresexecutesql.go:103](https://github.com/googleapis/mcp-toolbox/blob/a24e5e68567fa014a42fc6cc711faa82b964d4c0/internal/tools/postgres/postgresexecutesql/postgresexecutesql.go#L103) (verified)
  - *To reach the next level:* No moderate ceilings or in-flight cancellation.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Database row contents returned as plain text to the model (internal/server/mcp/v20251125/method.go:447) · [B] sensitive data/systems: Full read access to the configured database (internal/sources/postgres/postgres.go:118) · [C] state change / egress: execute_sql runs arbitrary writes and DDL (internal/tools/postgres/postgresexecutesql/postgresexecutesql.go:103) · Same default session? Yes

## Highest-impact improvements
1. Ship execute_sql in a separate write toolset and add a read-only execute_sql (or a readOnly source option that opens READ ONLY transactions) for postgres, enabled by default in the prebuilt. — C2 S L1→L3, +0.150 before caps (Playbook 3)
2. Default the postgres prebuilt to a session-level read-only setting (default_transaction_read_only plus a statement_timeout) the model cannot change, mirroring the Cloud SQL locked read-only session. — C1 D L0→L3, +0.150 before caps (Playbook 4)
3. Set a default statement_timeout and a server-side maximum row/byte cap on SQL results, and honour MCP cancellation. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)
4. Log every tool call (tool name, arguments hash or SQL, result status, caller) at info level by default, outside the model's reach. — C9 S L0→L2, +0.150 before caps (Playbook 1 step 3)
5. Restrict template parameters to allowlisted values or quoted identifiers and replace get_query_plan's raw substitution with a bound parameter. — C3 C L1→L2, +0.075 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the postgres prebuilt over stdio as the README's lead example; the ~40 other prebuilt configs and source types (BigQuery, Cloud SQL, AlloyDB, Spanner, Looker, Dataproc, HTTP tool, etc.) were sampled, not reviewed exhaustively, and several have stronger opt-in controls (read-only sessions, write modes, byte caps).
- HTTP mode (not scored) has request-origin defaults that are not locked down. It binds to 127.0.0.1 by default.
- Running the binary with no --prebuilt or --config auto-loads ./tools.yaml from the working directory with hot reload (not scored; would trigger C6-REPOCONFIG).
- Database-side behaviour (e.g. Postgres superuser COPY ... PROGRAM) is inferred from Postgres documentation, not this repo.
- No reviewer-steering text was found in AGENTS.md/CLAUDE.md/GEMINI.md/README.
