# Defense-in-Depth Score: Postgres MCP Pro

**Repo:** https://github.com/crystaldba/postgres-mcp · **Commit:** `15c8e33353546148acc2d8bd784551cf3905d1e2` · **Reviewed:** 2026-10-03
**What it is:** Postgres MCP with configurable read/write access, index tuning and health analysis
**Category:** Data & Analytics
**Scored configuration:** postgres-mcp over stdio with DATABASE_URI set and no flags, i.e. the default --access-mode unrestricted (also used in every README configuration example).
**Agent surface (default):** code execution yes · filesystem write no · network egress opt-in · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 4.2 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L0 | L0 | L2 | 0.17 | — | **0.17** | High |
| C2 | Approval gates | L3 | L3 | L0 | L3 | 0.60 | G1 | **0.50** (alt) | High |
| C3 | Tool & action scoping | L3 | L3 | L0 | L2 | 0.55 | G1 | **0.50** (alt) | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L0 | 0.07 | C5-WORSTCASE | **0.07** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C9 | Audit & traceability | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | Medium |
| C10 | Limits & kill switch | L2 | L2 | L0 | L2 | 0.40 | G1 | **0.40** (alt) | High |

Controls where a risk surface exists: 2.15 / 8.0 (27%); 2 criteria scored SA (surface absent).

As shipped, Postgres MCP Pro runs in unrestricted mode: the model can execute any SQL, including multi-statement DROP/DELETE batches, with the operator's database login, and every statement is committed immediately with no timeout, record or undo. Risk annotations on some read-only-labelled tools are not accurate. The opt-in restricted mode is a genuinely strong parser allowlist plus read-only transaction with a 30-second timeout; deployers should always use it and a read-only database role.

## Critical gaps
- Default unrestricted mode: a hijacked host session can read any data the login can see and run irreversible SQL that the server commits immediately, with no server-side gate. (ASI01, ASI02; C5) — [src/postgres_mcp/server.py:565](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L565); [src/postgres_mcp/server.py:608-615](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L608-L615); [src/postgres_mcp/sql/sql_driver.py:255-256](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/sql_driver.py#L255-L256)

## Criterion details

### C1 Identity & least privilege — 0.17 (high)

The server connects to Postgres with the single connection string the operator supplies in DATABASE_URI and uses that one credential, through one connection pool, for every tool, reads and writes alike. Nothing in the server narrows the database role or checks who is asking: there is no per-caller authorization. Least privilege therefore depends entirely on the operator creating a restricted database role; the README's own examples use a generic user/password login with unrestricted mode.

- **S L1:** A dedicated, operator-supplied database login (DATABASE_URI) is the only identity; its privilege is whatever the operator chose and reads and writes share it. — [src/postgres_mcp/server.py:629](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L629); [src/postgres_mcp/sql/sql_driver.py:89-92](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/sql_driver.py#L89-L92) (verified)
  - *To reach the next level:* No role scoping in the server; L2 needs a role-scoped identity with specific privileges defined or required by the project.
- **C L0:** No authorization check runs on any tool call; every tool obtains the same pooled connection via get_sql_driver(). — [src/postgres_mcp/server.py:62-71](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L62-L71); searched `rg -n -i 'auth|token|bearer'` in `src` → 1 hits (single hit is the full-text function name ts_token_type in the SQL allowlist; no authorization code exists) (verified)
  - *To reach the next level:* No authorization layer; L1 needs at least the main tool path checked against a policy.
- **D L0:** The default install grants the model full use of whatever role the DSN holds (unrestricted mode by default); least privilege requires the operator to create a limited role and pass --access-mode=restricted. — [src/postgres_mcp/server.py:565](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L565); [README.md:141](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/README.md#L141) (verified)
  - *To reach the next level:* No narrower default; L1 needs a narrower default identity or mode out of the box.
- **B L2:** A hijacked session gets everything the configured login can do on one database system, including DDL and DELETE via execute_sql; with a superuser login Postgres features (COPY ... PROGRAM, untrusted PL languages) reach the DB host (inferred from Postgres behaviour). — [src/postgres_mcp/server.py:608-615](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L608-L615); [src/postgres_mcp/sql/sql_driver.py:255-256](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/sql_driver.py#L255-L256) (verified)
  - *To reach the next level:* Writes, including destructive DDL, are reachable; L3 needs scope limited to mostly-read with non-destructive writes.
- **Cap:** none
- **Notes:** The optional network transport as configured by the Docker entrypoint is not locked down. Not the scored default (stdio).

### C2 Approval gates — 0.50 (high)

As a tool server, Postgres MCP Pro cannot show approval prompts itself; it gives the host risk annotations. In the default unrestricted mode, execute_sql is correctly flagged destructive, but the read-only labels on some tools are not accurate. The opt-in restricted mode is much better: every tool goes through a SQL parser allowlist and a read-only transaction, so the read-only labels become true, but it is not the default.

- **default configuration** (default; raw 0.20 → 0.20)
  - **S L1:** Annotations exist (execute_sql destructiveHint), but annotations are not fully accurate in unrestricted mode. — [src/postgres_mcp/server.py:608-615](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L608-L615); [src/postgres_mcp/sql/sql_driver.py:255-256](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/sql_driver.py#L255-L256) (verified)
    - *To reach the next level:* Annotations are not accurate on every tool; L2 needs accurate read/write annotations on every tool.
  - **C L1:** Only execute_sql is flagged as destructive; risk signalling for other SQL-taking tools is not complete. (verified)
    - *To reach the next level:* Risk signalling is not accurate on every tool; L2 needs every built-in tool's risk signalled accurately.
  - **D L1:** Risk annotations ship by default, but the server-enforced read-only mode is opt-in and the default exposes the destructive execute_sql tool. — [src/postgres_mcp/server.py:565](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L565); [src/postgres_mcp/server.py:606-615](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L606-L615) (verified)
    - *To reach the next level:* The safe mode is not the default; L2 needs the read-only/annotated posture on by default with only a config change to widen it.
  - **B L0:** A wrongly approved or mislabelled call can DROP tables or DELETE rows and the driver commits immediately; there is no undo or dry-run. — [src/postgres_mcp/server.py:608-615](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L608-L615); [src/postgres_mcp/sql/sql_driver.py:255-256](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/sql_driver.py#L255-L256) (verified)
    - *To reach the next level:* Irreversible destructive SQL with no rollback; L1 needs at least some reversible or previewed operations.
- **opt-in restricted mode (--access-mode=restricted)** (alt; raw 0.60, cap G1 → 0.50) ← counted
  - **S L3:** In restricted mode every tool goes through SafeSqlDriver: pglast statement/node/function allowlists, EXPLAIN ANALYZE rejected, and execution in a READ ONLY transaction, so every tool's readOnlyHint is accurate and the mode is server-enforced; no destructive operations remain to preview. — [src/postgres_mcp/server.py:66-68](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L66-L68); [src/postgres_mcp/sql/safe_sql.py:958-975](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/safe_sql.py#L958-L975); [src/postgres_mcp/sql/safe_sql.py:909-913](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/safe_sql.py#L909-L913); [src/postgres_mcp/sql/sql_driver.py:230-231](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/sql_driver.py#L230-L231); [src/postgres_mcp/server.py:616-624](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L616-L624) (verified)
    - *To reach the next level:* No confirmation step the host must complete for elevation; L4 needs the read-only mode plus a host-completed confirmation or equivalent.
  - **C L3:** get_sql_driver() is the single source of drivers, so all tools, including explain and index tuning, pass the same validator and read-only transaction. — [src/postgres_mcp/server.py:62-71](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L62-L71); [src/postgres_mcp/sql/safe_sql.py:1030-1036](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/safe_sql.py#L1030-L1036) (verified)
    - *To reach the next level:* Unknown statements are rejected but no parsed-argument escalation exists; L4 needs explicit reject/escalate outcomes per call.
  - **D L0:** Restricted mode must be selected explicitly; the CLI default and README examples are unrestricted. — [src/postgres_mcp/server.py:565](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L565); [README.md:141](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/README.md#L141) (verified)
    - *To reach the next level:* Opt-in; L1 needs the read-only mode on by default.
  - **B L3:** In restricted mode a wrongly approved call can only read (subject to the role's grants) or run allowlisted VACUUM/ANALYZE maintenance; nothing is written. — [src/postgres_mcp/sql/sql_driver.py:230-231](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/sql_driver.py#L230-L231); [src/postgres_mcp/sql/safe_sql.py:102-120](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/safe_sql.py#L102-L120) (verified)
    - *To reach the next level:* Reads of all data the role can see are unbounded; L4 needs rate or quantity bounds.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.
- **Notes:** The restricted-mode validator is credited here as a server-enforced read-only mode (named in the tool-server anchor) and in C3 as argument validation; C4 does not credit it again.

### C3 Tool & action scoping — 0.50 (high)

In the default unrestricted mode the main tool takes any SQL string and runs it as-is, multi-statement batches included, against the whole database. A few helper tools are narrow and parameterised (schema listing, object details, top-queries ranking), but the general execute_sql tool is raw passthrough. The opt-in restricted mode adds real validation: SQL is parsed with Postgres's own parser and checked against allowlists of statement types, node types and functions, inside a read-only transaction with a 30-second timeout.

- **default configuration** (default; raw 0.07 → 0.07)
  - **S L0:** execute_sql passes the model's SQL string straight to the driver; nothing parses or bounds it in unrestricted mode. — [src/postgres_mcp/server.py:415-421](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L415-L421); [src/postgres_mcp/server.py:421](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L421); [src/postgres_mcp/sql/sql_driver.py:234-241](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/sql_driver.py#L234-L241) (verified)
    - *To reach the next level:* Raw SQL passthrough; L1 needs at least a denylist or statement filter in the default mode.
  - **C L1:** A few tools validate (parameterised listing queries, get_top_queries sort_by, a statement check in analyze_query_indexes); execute_sql and other SQL-taking tools do not. — [src/postgres_mcp/server.py:133-142](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L133-L142); [src/postgres_mcp/server.py:549-550](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L549-L550) (verified)
    - *To reach the next level:* Most SQL-taking tools are unvalidated; L2 needs most built-in tools to validate.
  - **D L0:** Everything, including arbitrary write SQL, is enabled by default. — [src/postgres_mcp/server.py:565](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L565); [src/postgres_mcp/server.py:608-615](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L608-L615) (verified)
    - *To reach the next level:* No least-agency default; L1 needs dangerous capabilities individually disableable or off by default.
  - **B L0:** A misused execute_sql reaches any table, schema or server function the login can touch, with no row or quantity bound. — [src/postgres_mcp/server.py:421](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L421); [src/postgres_mcp/sql/sql_driver.py:252](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/sql_driver.py#L252) (verified)
    - *To reach the next level:* General-purpose SQL against the whole database; L1 needs some limit on reach.
- **opt-in restricted mode SQL allowlist** (alt; raw 0.55, cap G1 → 0.50) ← counted
  - **S L3:** pglast parse with allowlists of statement types, AST node types and functions (no pg_read_file, dblink, pg_sleep), locking clauses and EXPLAIN ANALYZE rejected, executed in a READ ONLY transaction with a 30 s timeout. — [src/postgres_mcp/sql/safe_sql.py:958-975](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/safe_sql.py#L958-L975); [src/postgres_mcp/sql/safe_sql.py:896-903](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/safe_sql.py#L896-L903); [src/postgres_mcp/sql/safe_sql.py:905-907](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/safe_sql.py#L905-L907); [src/postgres_mcp/sql/sql_driver.py:230-231](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/sql_driver.py#L230-L231) (verified)
    - *To reach the next level:* Still a general SELECT tool over every table with no row bound; L4 needs narrow tools replacing general SQL.
  - **C L3:** All built-in tools obtain the SafeSqlDriver through get_sql_driver(); internal queries are parameterised with psycopg Literal quoting. — [src/postgres_mcp/server.py:62-71](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L62-L71); [src/postgres_mcp/sql/safe_sql.py:1020-1027](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/safe_sql.py#L1020-L1027) (verified)
    - *To reach the next level:* The policy is applied by driver selection, not a declarative layer new tools must inherit; L4 needs one central policy layer for all tools.
  - **D L0:** Restricted mode is opt-in; default is unrestricted. — [src/postgres_mcp/server.py:565](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L565) (verified)
    - *To reach the next level:* Opt-in; L1 needs dangerous tools off by default.
  - **B L2:** Reads across the whole database the role can see, without row caps; no writes. — [src/postgres_mcp/sql/sql_driver.py:230-231](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/sql_driver.py#L230-L231); [src/postgres_mcp/sql/sql_driver.py:252](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/sql_driver.py#L252) (verified)
    - *To reach the next level:* No quantity bound on reads; L3 needs max-row or output caps.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C4 Code-execution isolation — 0.05 (high)

The code-execution surface here is SQL: execute_sql and other SQL-taking tools send model-written statements straight to the database. In the default unrestricted mode there is no boundary at all: no statement filter, no read-only transaction, no timeout, and multi-statement batches are executed and committed. What that SQL can reach is everything the configured login can do; with a superuser login, Postgres features such as COPY ... TO PROGRAM or untrusted procedural languages extend that to the database host. The opt-in restricted mode's parser allowlist is credited under tool scoping rather than here.

- **S L0:** Model-written SQL is executed directly in the database session with no isolation primitive in the default mode. — [src/postgres_mcp/server.py:421](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L421) (verified)
  - *To reach the next level:* No isolation; L1 needs at least a filter on executed statements in the default path.
- **C L0:** No execution path is confined in the default mode. (verified)
  - *To reach the next level:* No path is confined; L1 needs the main exec path confined.
- **D L0:** Nothing confines execution by default; the only protective mode is opt-in. — [src/postgres_mcp/server.py:565](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L565); [src/postgres_mcp/server.py:58](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L58) (verified)
  - *To reach the next level:* Off by default; L1 needs a confinement on by default.
- **B L1:** Model SQL runs with the full privileges of the configured login across the database, including destructive DDL; with a superuser login server-side features reach the DB host (inferred from Postgres behaviour; the README concedes unsafe procedural languages defeat even restricted mode). — [src/postgres_mcp/server.py:608-615](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L608-L615); [README.md:606](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/README.md#L606) (verified)
  - *To reach the next level:* The whole database (and with superuser the DB host) is reachable; L2 needs reach limited to a workspace-like scope.
- **Cap:** none
- **Notes:** Restricted mode (pglast allowlist + READ ONLY transaction) is a filter in the same DB session (S L1 at most) and is already credited in C3; per the credit-once rule it is not added here as an alt.

### C5 Untrusted input blast radius — 0.07 (high)

The server returns database rows, catalog data and pg_stat_statements query texts to the model as plain stringified Python lists, with no marking of what came from the database versus the server. Any of that content can be attacker-written (user-submitted rows, logged queries). Nothing in the server limits what a hijacked host can then do: in the default mode the same session can read any data the login can see and run irreversible SQL, and the model-selectable index-tuning method 'llm' sends query text and plans to OpenAI. Restricted mode removes the write leg but not the read-everything leg, and it is opt-in.

- **S L1:** Outputs are plain str() of row dicts with no provenance or untrusted flag; tool descriptions contain no directives. — [src/postgres_mcp/server.py:424](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L424); [src/postgres_mcp/server.py:74-76](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L74-L76); searched `rg -n -i 'untrusted|provenance|source_url'` in `src` → 0 hits (no provenance or taint marking anywhere) (verified)
  - *To reach the next level:* No structure separating content from metadata; L2 needs structured outputs separating returned data from metadata.
- **C L0:** No source is distinguished: table rows, pg_stat_statements texts and error strings all return the same way. — [src/postgres_mcp/server.py:424](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L424); [src/postgres_mcp/server.py:427](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L427) (verified)
  - *To reach the next level:* Untrusted DB content is not distinguished; L1 needs at least one source handled.
- **D L0:** No untrusted-input control exists to be on by default; the default mode keeps the write leg. — [src/postgres_mcp/server.py:565](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L565) (verified)
  - *To reach the next level:* Off; L1 needs a protective mode on by default.
- **B L0:** Default config: a hijacked session can read every table the login can see and run irreversible DROP/DELETE that is committed immediately; exfiltration can go through the host's channels, and in-DB through extensions such as dblink when available (inferred). — [src/postgres_mcp/server.py:608-615](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L608-L615); [src/postgres_mcp/sql/sql_driver.py:255-256](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/sql_driver.py#L255-L256); [src/postgres_mcp/index/llm_opt.py:156-163](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/index/llm_opt.py#L156-L163) (verified)
  - *To reach the next level:* Leak and irreversible action are both possible unattended; L1 needs at least one of them removed or gated in code.
- **Cap:** C5-WORSTCASE — B is L0 in the default unrestricted mode: data reads and irreversible committed SQL from the same session with no server-side gate.

### C6 Memory, context & configuration integrity — 1.00 (high)

The server keeps no memory, vector store or conversation state, and it auto-loads no workspace files: configuration comes only from the DATABASE_URI environment variable or a command-line argument set by the operator. There is no .env loading. The database itself is persistent and writable in unrestricted mode, but that is the system being operated on (covered under the other criteria), not agent memory.

- **Structural absence:** searched `rg -n -i 'dotenv|load_dotenv|remember|save_memory|\.mcp\.json|AGENTS\.md|CLAUDE\.md'` in `src` → 0 hits (no memory, dotenv or auto-loaded instruction/config files); [src/postgres_mcp/server.py:629](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L629)

### C7 Third-party extensions — 1.00 (high)

The server loads no plugins, MCP servers, downloaded tools or model files, and installs nothing at runtime. Its only runtime third-party call is the optional OpenAI API request in the 'llm' index-tuning method, which is a fixed dependency, not loaded code. Database-side extensions created via SQL are scored as SQL execution under C4.

- **Structural absence:** searched `rg -n 'importlib|entry_points|__import__|pip install|subprocess|trust_remote_code|pickle|torch\.load'` in `src` → 0 hits (no dynamic code loading, installs, subprocesses or unsafe deserialization)

### C8 Secrets & sensitive-data protection — 0.25 (high)

The database password lives in the DATABASE_URI environment variable or command line. A password-obfuscation helper exists but is only applied to the startup connection-failure warning. Elsewhere, failed SQL is logged with its full text and raw error strings are returned to the model; the Docker entrypoint's credential handling is not locked down. There is no telemetry. The credential is a long-lived database password whose scope is whatever the operator chose.

- **S L1:** Secret comes from an env var; obfuscate_password masks it in one log path only. — [src/postgres_mcp/server.py:629](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L629); [src/postgres_mcp/server.py:641-643](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L641-L643); [src/postgres_mcp/sql/sql_driver.py:20-24](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/sql_driver.py#L20-L24) (verified)
  - *To reach the next level:* No type-level masking or log filters on main paths; L2 needs redaction on the main logging paths.
- **C L1:** Only the startup connection warning is masked; query error logs and tool error strings returned to the model are not. — [src/postgres_mcp/sql/sql_driver.py:271](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/sql_driver.py#L271); [src/postgres_mcp/server.py:427](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L427) (verified)
  - *To reach the next level:* Only one path is protected; L2 needs logs and transcripts redacted.
- **D L1:** No telemetry, but failed queries are logged verbatim and the Docker entrypoint's credential handling is not locked down. — searched `rg -n -i 'sentry|posthog|telemetry|segment|langfuse\.'` in `src` → 0 hits (no telemetry SDKs; _langfuse_trace is only a field in the tool response) (verified)
  - *To reach the next level:* Default logging is not redacted; L2 needs reasonable logging defaults with redaction.
- **B L1:** A leaked DSN is a long-lived database password whose privilege is the operator's chosen role, often an owner or superuser. — [src/postgres_mcp/server.py:629](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L629); [README.md:141](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/README.md#L141) (verified)
  - *To reach the next level:* Long-lived credential of unbounded scope; L2 needs a scoped credential.
- **Cap:** none
- **Notes:** When the model selects method='llm' in index tuning, query text and EXPLAIN plans (which may contain literals from data) go to OpenAI gpt-4o using the ambient OPENAI_API_KEY (llm_opt.py:129,156-163).

### C9 Audit & traceability — 0.20 (medium)

The server keeps no record of what it did. Successful tool calls, including every write executed through execute_sql, are not logged at all; only failures are logged, as free-text error lines that include the failing SQL. The repository configures no log handler, so these lines go wherever the MCP library or Python's default handler sends them (stderr), and nothing is flushed per action or protected against loss.

- **S L1:** Only failed queries are logged, as unstructured error lines; successful calls leave no trace. — [src/postgres_mcp/server.py:419-427](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L419-L427); searched `rg -n 'logger\.(info|warning)'` in `src` → 41 hits (all hits are startup/shutdown, index-tuning progress or warnings; none records a tool call with its arguments) (verified)
  - *To reach the next level:* No structured record of every tool call; L2 needs arguments, status and timestamps for each call.
- **C L1:** Error logging covers only some tool paths' failures; successful writes are never recorded. — [src/postgres_mcp/sql/sql_driver.py:271](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/sql_driver.py#L271) (verified)
  - *To reach the next level:* Not all tool calls are recorded; L2 needs every built-in tool call recorded.
- **D L1:** Error lines reach stderr by default via the library/stdlib handler (inferred; the repo configures no handler) inside the server process. — searched `rg -n 'basicConfig|FileHandler|StreamHandler|setLevel'` in `src` → 0 hits (no logging configuration in the repo; output depends on FastMCP/stdlib defaults) (inferred)
  - *To reach the next level:* Recorded by the same process with no protected sink; L2 needs an on-by-default record outside the agent's reach.
- **B L0:** Logging is best-effort stderr and actions proceed regardless; successful actions are never recorded. — [src/postgres_mcp/server.py:421](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L421); [src/postgres_mcp/sql/sql_driver.py:255-256](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/sql_driver.py#L255-L256) (verified)
  - *To reach the next level:* No per-action record; L1 needs at least a best-effort record of actions.
- **Cap:** none

### C10 Limits & kill switch — 0.40 (high)

In the default unrestricted mode, execute_sql and explain_query have no timeout and no row or output cap: a query runs until Postgres finishes and all rows are buffered. The index-tuning advisor stops itself after 30 seconds and accepts at most 10 queries, and the LLM optimizer stops after 5 attempts without progress. The opt-in restricted mode adds a 30-second timeout to every query. Shutdown closes the connection pool on SIGTERM/SIGINT.

- **default configuration** (default; raw 0.33 → 0.33)
  - **S L2:** Server-enforced caps exist on some operations (DTA max_runtime_seconds=30, max 10 tuning queries, LLM optimizer attempt cap), not on execute_sql. — [src/postgres_mcp/index/dta_calc.py:34](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/index/dta_calc.py#L34); [src/postgres_mcp/index/index_opt_base.py:25](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/index/index_opt_base.py#L25); [src/postgres_mcp/index/llm_opt.py:61](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/index/llm_opt.py#L61) (verified)
    - *To reach the next level:* No caps on every operation and no rate limits; L3 needs caps on every operation plus concurrency or rate limits.
  - **C L1:** Limits cover only index tuning; the general SQL tools are unbounded in unrestricted mode. — [src/postgres_mcp/server.py:62-71](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L62-L71); [src/postgres_mcp/sql/sql_driver.py:252](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/sql_driver.py#L252) (verified)
    - *To reach the next level:* Main SQL path has no timeout; L2 needs timeouts on the tool paths.
  - **D L1:** Defaults exist only where noted; the default mode has no statement timeout and the restricted-mode timeout is hard-coded and opt-in. — [src/postgres_mcp/server.py:565](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L565); searched `rg -n 'statement_timeout|asyncio\.timeout'` in `src` → 1 hits (the only timeout is in SafeSqlDriver (restricted mode)) (verified)
    - *To reach the next level:* No sensible default bound on the main tool; L2 needs sensible operator-configurable defaults.
  - **B L1:** A runaway query has no ceiling in the default mode; killing the server closes the pool but a query already running in Postgres is not shown to be cancelled. — [src/postgres_mcp/server.py:686-689](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L686-L689) (verified)
    - *To reach the next level:* No ceiling on default queries; L2 needs moderate ceilings.
- **opt-in restricted mode 30 s query timeout** (alt; raw 0.40, cap G1 → 0.40) ← counted
  - **S L2:** Every query in restricted mode runs under asyncio.timeout(30). — [src/postgres_mcp/sql/safe_sql.py:990-1003](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/safe_sql.py#L990-L1003); [src/postgres_mcp/server.py:66-68](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L66-L68) (verified)
    - *To reach the next level:* No row/output caps or rate limits; L3 needs caps on every operation plus rate limits.
  - **C L2:** All tools use the timed SafeSqlDriver in restricted mode. — [src/postgres_mcp/server.py:62-71](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L62-L71) (verified)
    - *To reach the next level:* Per-query only; no aggregate per-call budget across the many queries a tool issues.
  - **D L0:** Opt-in: restricted mode is not the default. — [src/postgres_mcp/server.py:565](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/server.py#L565) (verified)
    - *To reach the next level:* Opt-in; L1 needs bounds on by default.
  - **B L2:** A runaway read is bounded at 30 s per query; whether the server-side query is cancelled on timeout depends on psycopg cancellation behaviour (inferred). — [src/postgres_mcp/sql/safe_sql.py:998-1003](https://github.com/crystaldba/postgres-mcp/blob/15c8e33353546148acc2d8bd784551cf3905d1e2/src/postgres_mcp/sql/safe_sql.py#L998-L1003) (verified)
    - *To reach the next level:* No aggregate ceiling per tool call; L3 needs tight per-run ceilings with cancellation.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

## Rule-of-Two check
[A] untrusted input: database rows and pg_stat_statements query texts returned to the model (server.py:424) · [B] sensitive data/systems: the whole database behind DATABASE_URI (server.py:629) · [C] state change / egress: arbitrary committed SQL via execute_sql in unrestricted mode (server.py:608-615, sql_driver.py:255-256) · Same default session? Yes

## Highest-impact improvements
1. Make --access-mode restricted the default and require an explicit flag for unrestricted. — C3 D L0→L3, +0.150 before caps (Playbook 3, step 1)
2. Make every tool's risk annotation accurate in all modes. — C2 S L1→L2, +0.075 before caps (Playbook 5, step 1)
3. Set a default statement_timeout and a row cap for execute_sql in unrestricted mode. — C10 C L1→L2, +0.075 before caps (Playbook 3, step 3)
4. Log every tool call (tool, SQL, status, row count, timestamp) as a structured record. — C9 S L1→L2, +0.075 before caps (Playbook 1, step 3)
5. Apply obfuscate_password to all logged and returned errors. — C8 C L1→L2, +0.075 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of commit 15c8e33353546148acc2d8bd784551cf3905d1e2 only; nothing was installed, executed or probed.
- Postgres and psycopg behaviour (multi-statement execution without parameters, CREATE EXTENSION blocked in READ ONLY transactions, cancellation on asyncio timeout) is inferred from upstream documentation, not tested.
- Blast radius depends on the database role the operator puts in DATABASE_URI; ratings assume a typical owner-level login as shown in README examples.
- Tests, examples and CI workflows were not reviewed in depth; no reviewer-injection text was found in the repository.
