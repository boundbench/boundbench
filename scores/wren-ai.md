# Defense-in-Depth Score: Wren AI

**Repo:** https://github.com/Canner/WrenAI · **Commit:** `2cc843fd87d6cfe9831554721559018dda2938a0` · **Reviewed:** 2026-10-03
**What it is:** GenBI / governed text-to-SQL with open context layer (semantic engine) for AI agents
**Category:** Data & Analytics
**Scored configuration:** The README-led flow: `pip install wrenai` CLI driven by a host coding agent through the shipped `wren` skill stub, with a connection profile and default ~/.wren config (strict mode off); the `wren serve mcp` server is footnoted.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents no · external communication yes

## Score: 2.6 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L0 | L1 | 0.12 | — | **0.12** | High |
| C2 | Approval gates | L2 | L3 | L3 | L2 | 0.62 | G1 | **0.50** (alt) | High |
| C3 | Tool & action scoping | L3 | L3 | L2 | L2 | 0.65 | G1 | **0.50** (alt) | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L1 | 0.12 | — | **0.12** | Medium |
| C6 | Memory, context & configuration integrity | L1 | L1 | L2 | L1 | 0.30 | C6-REPOCONFIG | **0.25** | High |
| C7 | Third-party extensions | L1 | L0 | L1 | L0 | 0.12 | — | **0.12** | Medium |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L3 | L1 | 0.50 | — | **0.50** | High |
| C9 | Audit & traceability | L1 | L0 | L1 | L0 | 0.12 | — | **0.12** | High |
| C10 | Limits & kill switch | L2 | L1 | L1 | L1 | 0.33 | — | **0.33** | High |


Wren puts an always-on read-only SQL check in front of your databases, and handles secrets carefully, but most other agent-safety controls are missing. Its shipped Claude Code skill pre-approves every `wren` command, so an agent can query anything your database login can read and publish it to Vercel or Cloudflare with `wren genbi deploy` without a human prompt. Queries run directly under your credentials with no row or cost cap on the CLI, nothing is logged, and project files (.env, wren_project.yml) silently choose credentials. Use a read-only database role, turn on strict mode, and remove the blanket skill permission before pointing it at production data.

## Critical gaps
- The shipped Claude Code skill pre-approves every `wren` command (`allowed-tools: Bash(wren:*)`), including `wren genbi deploy`, which publishes warehouse data to a public host with no confirmation. (ASI09, ASI02; C2) — [skills/wren/SKILL.md:5](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/skills/wren/SKILL.md#L5); [core/wren/src/wren/genbi/cli.py:355-362](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/cli.py#L355-L362)
- Model-written SQL executes directly on the target database under the operator's credential, and the wrangler deploy subprocess inherits the full environment, with no isolation layer. (ASI05, LLM05; C4) — [core/wren/src/wren/connector/postgres.py:299](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/connector/postgres.py#L299); [core/wren/src/wren/genbi/providers/cloudflare.py:80-84](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/providers/cloudflare.py#L80-L84)
- Workspace files auto-loaded without a trust decision (CWD/project `.env`, wren_project.yml `profile:`) choose the credential profile and supply secrets and deploy tokens. (ASI06, ASI03; C6) — [core/wren/src/wren/profile.py:72-91](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/profile.py#L72-L91); [core/wren/src/wren/profile.py:248-251](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/profile.py#L248-L251)
- `npx wrangler` (unpinned) and the in-process embedding model run with all of Wren's credentials and environment. (ASI04, LLM03; C7) — [core/wren/src/wren/genbi/providers/cloudflare.py:29-32](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/providers/cloudflare.py#L29-L32); [core/wren/src/wren/genbi/providers/cloudflare.py:80-84](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/providers/cloudflare.py#L80-L84)

## Criterion details

### C1 Identity & least privilege — 0.12 (high)

Wren runs every query with whatever database credential sits in the active (or project-pinned) connection profile; it does not scope, downscope, or check authorization per request. For BigQuery it requests the broad cloud-platform and Google Drive scopes and falls back to the operator's Application Default Credentials when no key is given. Row- and column-level access rules in the semantic model exist, but in the default non-strict mode a query can name raw database tables outside the model and skip them. A project's own wren_project.yml chooses which stored credential profile is used.

- **S L0:** The engine uses the profile's credential as-is and, for BigQuery, the operator's ADC chain with cloud-platform and Drive scopes. — [core/wren/src/wren/connector/bigquery.py:27-30](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/connector/bigquery.py#L27-L30); [core/wren/src/wren/connector/bigquery.py:53](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/connector/bigquery.py#L53); [core/wren/src/wren/cli.py:248](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/cli.py#L248) (verified)
  - *To reach the next level:* Request minimal scopes and use a dedicated, role-scoped identity rather than ambient ADC with cloud-platform scope.
- **C L1:** The query path goes through the semantic layer, but MDL row/column rules bind only to modelled tables (raw tables pass in non-strict mode) and the wrangler subprocess receives the full process environment. — [core/wren/src/wren/config.py:32](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/config.py#L32); [core/wren/src/wren/genbi/providers/cloudflare.py:80-84](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/providers/cloudflare.py#L80-L84) (verified)
  - *To reach the next level:* MDL access rules do not cover non-model tables by default and spawned tools inherit the whole environment.
- **D L0:** No narrower default: least privilege depends on the operator creating a restricted database role, and the workspace's wren_project.yml can pin which stored profile is used. — [core/wren/src/wren/profile.py:248-251](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/profile.py#L248-L251); [core/wren/src/wren/profile.py:220-226](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/profile.py#L220-L226) (verified)
  - *To reach the next level:* No read-only or minimal default identity; a repo-controlled project file selects the credential profile.
- **B L1:** A misused credential reads everything the operator's DB login can see across connected warehouses, and the same process holds Vercel/Cloudflare deploy tokens and the Wren Cloud key; the always-on read-only SQL check still blocks plain DML/DDL. — [core/wren/src/wren/connector/bigquery.py:27-30](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/connector/bigquery.py#L27-L30); [core/wren/src/wren/policy.py:308-310](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/policy.py#L308-L310); [core/wren/src/wren/genbi/tokens.py:14-27](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/tokens.py#L14-L27) (verified)
  - *To reach the next level:* Reach spans the operator's whole warehouse login plus deploy-provider accounts; L2 needs reach limited to one system's writes or read-only access.
- **Cap:** none

### C2 Approval gates — 0.50 (high)

Wren has no approval step of its own: queries, memory writes, context changes and `wren genbi deploy` (which publishes an app with bundled data to Vercel or Cloudflare) all run without confirmation. Worse, the skill stub it ships for Claude Code pre-approves every `wren` command, so the host agent's own permission prompt is skipped for deploys too. The deploy step only runs a structural and secret-pattern preflight. The MCP server mode is better behaved: every tool carries a read-only or write hint, and the single write tool is only registered when the operator passes --allow-write.

- **default configuration** (default; raw 0.05, cap C2-POWERBYPASS → 0.05)
  - **S L0:** No human approval in Wren for any action, and the shipped skill grants a blanket Bash(wren:*) permission to the host agent. — [skills/wren/SKILL.md:5](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/skills/wren/SKILL.md#L5); searched `rg -n -i 'confirm|approv'` in `core/wren/src/wren/genbi/cli.py core/wren/src/wren/cloud_cli.py` → 6 hits (All six hits are the skippable (--yes) confirmation for dropping a stored Wren Cloud API key; nothing gates deploy, push, or query.) (verified)
    - *To reach the next level:* No per-call approval of consequential commands (deploy, cloud push, memory store).
  - **C L0:** The most consequential path, `wren genbi deploy`, runs straight after an automated verify with no human gate. — [core/wren/src/wren/genbi/cli.py:333-342](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/cli.py#L333-L342); [core/wren/src/wren/genbi/cli.py:355-362](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/cli.py#L355-L362) (verified)
    - *To reach the next level:* Deploy and other mutating commands are not gated at all.
  - **D L0:** There is no gate to turn on, and the default skill install auto-approves the entire CLI in the host. — [skills/wren/SKILL.md:5](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/skills/wren/SKILL.md#L5) (verified)
    - *To reach the next level:* Approval does not exist by default; the shipped skill pre-authorizes every wren command.
  - **B L1:** A wrongly run deploy publishes warehouse data to a public hosting URL, which cannot be taken back; SQL DML/DDL is blocked by the read-only AST check, though that check is not a strict boundary. — [core/wren/src/wren/genbi/cli.py:355-362](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/cli.py#L355-L362); [core/wren/src/wren/genbi/cli.py:308-311](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/cli.py#L308-L311); [core/wren/src/wren/policy.py:308-310](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/policy.py#L308-L310) (verified)
    - *To reach the next level:* Public publication is irreversible and there are no previews, recipient limits, or rate limits on deploys.
- **MCP server mode (`wren serve mcp`), read-only unless --allow-write** (alt; raw 0.62, cap G1 → 0.50) ← counted
  - **S L2:** Separate read and write tools with readOnlyHint on every tool, and the only write tool registered solely under --allow-write; run_sql's read-only hint rests on an AST check that is not a strict boundary. — [core/wren/src/wren/mcp_server.py:118-121](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/mcp_server.py#L118-L121); [core/wren/src/wren/mcp_server.py:518-519](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/mcp_server.py#L518-L519); [core/wren/src/wren/mcp_server.py:694-695](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/mcp_server.py#L694-L695) (verified)
    - *To reach the next level:* No preview/dry-run for the write tool and the run_sql read-only hint is not fully enforced; L3 needs both.
  - **C L3:** Every MCP tool is registered with an annotation and the write tool is behind the operator flag; no deploy or cloud tool is exposed over MCP. — [core/wren/src/wren/mcp_server.py:694-695](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/mcp_server.py#L694-L695); [core/wren/src/wren/serve_cli.py:141-144](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/serve_cli.py#L141-L144) (verified)
    - *To reach the next level:* run_sql's read-only hint relies on an AST check that is not a strict boundary.
  - **D L3:** Write tool is off by default and enabled only by an explicit operator flag at server launch. — [core/wren/src/wren/serve_cli.py:141-144](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/serve_cli.py#L141-L144) (verified)
    - *To reach the next level:* The read-only mode is not bounded in time and not tied to an authenticated principal.
  - **B L2:** Through MCP, a wrongly approved call can at most run a read-only-checked query or write a local markdown memory file. — [core/wren/src/wren/policy.py:308-310](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/policy.py#L308-L310); [core/wren/src/wren/mcp_server.py:535-538](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/mcp_server.py#L535-L538) (verified)
    - *To reach the next level:* The read-only check is not a strict boundary and there are no rate limits on calls.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C3 Tool & action scoping — 0.50 (high)

Every SQL statement, from the CLI, MCP server and SDKs alike, passes through one policy function that parses it and rejects anything that is not a single read-only SELECT-family query, including hidden writes such as data-modifying CTEs or SELECT INTO. That check is real but not a full boundary: it does not cover every path, and the re-check of the final planned SQL is skipped when it cannot be parsed. The stronger strict mode, which limits queries to tables defined in the semantic model and blocks file/URL/remote-database reader functions, is off by default. Beyond SQL, the CLI exposes everything at once (query, deploy, cloud push, memory writes) with no narrower default tool set.

- **default configuration** (default; raw 0.40 → 0.40)
  - **S L2:** AST-based read-only allowlist on the statement root plus a forbidden-node walk, but it is not a strict boundary and the planned-SQL re-check fails open. — [core/wren/src/wren/policy.py:260-268](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/policy.py#L260-L268); [core/wren/src/wren/policy.py:279-287](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/policy.py#L279-L287) (verified)
    - *To reach the next level:* No table allowlist or function allowlist by default, so file/URL-reading functions inside a SELECT pass.
  - **C L2:** The same validate_sql_policy call sits in the shared engine used by the CLI, MCP server and SDK toolkits, and deploy validates app names, but memory store, cloud and profile commands accept their inputs without policy checks. — [core/wren/src/wren/engine.py:199-203](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/engine.py#L199-L203); [core/wren/src/wren/genbi/cli.py:18](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/cli.py#L18); [core/wren/src/wren/mcp_server.py:606](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/mcp_server.py#L606) (verified)
    - *To reach the next level:* Not every built-in command validates its inputs, and there is no central policy layer new commands inherit.
  - **D L1:** The default CLI exposes query, deploy, cloud and memory-write commands together; only the MCP server ships read-only by default. — [skills/wren/SKILL.md:5](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/skills/wren/SKILL.md#L5); [core/wren/src/wren/mcp_server.py:694-695](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/mcp_server.py#L694-L695) (verified)
    - *To reach the next level:* No read-only default tool set for the README-led CLI flow.
  - **B L1:** A misused query reaches any table the DB login can read (non-strict) with no row cap on the CLI; a misused deploy ships to a public host. — [core/wren/src/wren/config.py:32](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/config.py#L32); [core/wren/src/wren/cli.py:441](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/cli.py#L441); [core/wren/src/wren/cli.py:302-304](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/cli.py#L302-L304) (verified)
    - *To reach the next level:* Reach is not scoped to modelled tables or bounded in rows by default.
- **opt-in strict mode (~/.wren/config.json strict_mode: true)** (alt; raw 0.65, cap G1 → 0.50) ← counted
  - **S L3:** Strict mode fails closed on any table or table-valued function not in the MDL and blocks known data-reader functions in every AST position. — [core/wren/src/wren/policy.py:312-318](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/policy.py#L312-L318); [core/wren/src/wren/policy.py:344-384](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/policy.py#L344-L384) (verified)
    - *To reach the next level:* Readers outside the source position are matched by a blocklist and further gaps remain.
  - **C L3:** Applies through the shared engine to every SQL entry point. — [core/wren/src/wren/engine.py:199-203](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/engine.py#L199-L203) (verified)
    - *To reach the next level:* Not a single policy layer covering non-SQL commands.
  - **D L2:** Enabled only through the user-scope ~/.wren/config.json, which the operator controls. — [core/wren/src/wren/config.py:43](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/config.py#L43) (verified)
    - *To reach the next level:* Off by default and silently disabled by deleting the config key.
  - **B L2:** Queries are scoped to the tables modelled in the project. — [core/wren/src/wren/policy.py:312-318](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/policy.py#L312-L318) (verified)
    - *To reach the next level:* Still no default row or cost bound on the CLI.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C4 Code-execution isolation — 0.00 (high)

Wren's core job is to run model-written SQL against live databases, and it does so directly with the operator's database credential; there is no isolation boundary around that execution beyond the database's own permissions. The Cloudflare deploy path also launches the wrangler CLI (or `npx wrangler`) as the same user with the full process environment. DuckDB files are attached read-only, which is a narrow mitigation for that one connector.

- **S L0:** Generated SQL is executed by the target database under the profile credential, and wrangler runs as a same-user subprocess; no sandbox exists. — [core/wren/src/wren/connector/postgres.py:324-328](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/connector/postgres.py#L324-L328); [core/wren/src/wren/genbi/providers/cloudflare.py:36-42](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/providers/cloudflare.py#L36-L42) (verified)
  - *To reach the next level:* No isolation primitive (restricted DB session, separate low-privilege runtime, or sandboxed subprocess).
- **C L0:** No execution path is isolated; only DuckDB attachments are opened READ_ONLY. — [core/wren/src/wren/connector/duckdb.py:140](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/connector/duckdb.py#L140); [core/wren/src/wren/connector/postgres.py:299](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/connector/postgres.py#L299) (verified)
  - *To reach the next level:* Isolation is absent on the main SQL path and on the wrangler subprocess.
- **D L0:** No sandbox to enable. — [core/wren/src/wren/connector/postgres.py:299](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/connector/postgres.py#L299) (verified)
  - *To reach the next level:* No isolation, on or off.
- **B L0:** SQL runs with the operator's DB credential (autocommit), and the wrangler subprocess receives the whole environment including any .env-merged secrets. — [core/wren/src/wren/connector/postgres.py:299](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/connector/postgres.py#L299); [core/wren/src/wren/genbi/providers/cloudflare.py:80-84](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/providers/cloudflare.py#L80-L84); [core/wren/src/wren/profile.py:72-91](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/profile.py#L72-L91) (verified)
  - *To reach the next level:* Credentials are present on every execution path; L1 needs at least credentials kept out of spawned processes.
- **Cap:** none

### C5 Untrusted input blast radius — 0.12 (medium)

Wren returns database rows, project knowledge files, AGENTS.md and business rules to the host agent as plain content with no untrusted marking, and the onboarding and enrichment flows have the agent read raw documents. Nothing in Wren limits what a hijacked agent does next: with the shipped skill pre-approving every wren command, injected content in a database row or document could steer the agent to query sensitive tables and publish them with `wren genbi deploy`, unattended. Plain SQL writes are still blocked by the read-only check.

- **S L1:** Outputs are plain tables or JSON with no provenance or untrusted flag; project rule files and AGENTS.md are served as instructions. — [core/wren/src/wren/mcp_server.py:367-372](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/mcp_server.py#L367-L372); [core/wren/src/wren/mcp_server.py:589-597](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/mcp_server.py#L589-L597) (verified)
  - *To reach the next level:* No provenance or untrusted marking on returned content that a host could act on.
- **C L0:** No source is distinguished: DB results, knowledge files and memory exemplars all come back with the same standing. — [core/wren/src/wren/mcp_server.py:374-389](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/mcp_server.py#L374-L389) (verified)
  - *To reach the next level:* No untrusted-source handling for any source.
- **D L0:** No control exists. — [core/wren/src/wren/mcp_server.py:367-372](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/mcp_server.py#L367-L372) (verified)
  - *To reach the next level:* Nothing to enable.
- **B L1:** A hijacked host can read any table the credential sees and publish it via an auto-approved `wren genbi deploy`; destructive SQL is still blocked by the independent read-only check. — [skills/wren/SKILL.md:5](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/skills/wren/SKILL.md#L5); [core/wren/src/wren/genbi/cli.py:355-362](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/cli.py#L355-L362); [core/wren/src/wren/policy.py:308-310](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/policy.py#L308-L310) (inferred)
  - *To reach the next level:* Exfiltration through deploy runs unattended; L2 needs it behind human approval.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.25 (high)

Several files in the project directory steer Wren without any trust decision. Every run auto-loads `.env` from the current directory and the project root into the process environment, supplying values for credential placeholders and deploy tokens, and the project's wren_project.yml picks which stored credential profile is used. Rule files and AGENTS.md from the project are served to the agent as instructions, and stored question-to-SQL pairs are recalled as proven examples. Memory is plain markdown under knowledge/ that is easy to review in git, and the MCP write tool is off by default, but the CLI store command and Git Sync share it with the whole team.

- **S L1:** Memory writes are plain markdown files with a source tag but no validation; instruction files and AGENTS.md load silently, and repo .env/wren_project.yml influence credentials. — [core/wren/src/wren/profile.py:72-91](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/profile.py#L72-L91); [core/wren/src/wren/profile.py:248-251](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/profile.py#L248-L251); [core/wren/src/wren/memory/markdown.py:126](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/memory/markdown.py#L126) (verified)
  - *To reach the next level:* Project config can still change credential selection and environment without a workspace-trust decision.
- **C L1:** Only the MCP write path is gated (by --allow-write); CLI memory store, auto-loaded .env and project config are not. — [core/wren/src/wren/mcp_server.py:694-695](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/mcp_server.py#L694-L695); [core/wren/src/wren/memory/cli.py:397](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/memory/cli.py#L397) (verified)
  - *To reach the next level:* Auto-loaded files and the CLI memory path are uncontrolled.
- **D L2:** Memory and knowledge live per project directory, so separate projects do not share stores. — [core/wren/src/wren/memory/markdown.py:40-41](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/memory/markdown.py#L40-L41) (verified)
  - *To reach the next level:* Nothing stops the agent writing into other projects' namespaces via --path.
- **B L1:** Poisoned exemplars and rules persist across sessions and steer future SQL, and Git Sync propagates them to teammates. — [core/wren/src/wren/mcp_server.py:654](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/mcp_server.py#L654); [README.md:128](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/README.md#L128) (verified)
  - *To reach the next level:* Poisoned context persists and can drive tool use; L2 needs it limited to text or gated actions.
- **Cap:** C6-REPOCONFIG — `.env` files in the working directory and project root are auto-loaded into os.environ and wren_project.yml pins the credential profile, so workspace files redirect credentials and deploy tokens with no trust prompt.

### C7 Third-party extensions — 0.12 (medium)

Wren loads little third-party code, but what it does load is unpinned and unconfined. Cloudflare deploys fall back to `npx wrangler` (latest from npm) when wrangler is not installed and pass it the whole environment, and the optional memory extra downloads a Hugging Face embedding model, chosen by an environment variable, at whatever revision is current. Nothing is verified by hash or signature.

- **S L1:** User-chosen provider/extras, but wrangler is resolved via npx without a version and the embedding model has no pinned revision. — [core/wren/src/wren/genbi/providers/cloudflare.py:29-32](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/providers/cloudflare.py#L29-L32); [core/wren/src/wren/memory/embeddings.py:27-29](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/memory/embeddings.py#L27-L29) (verified)
  - *To reach the next level:* No version pins for wrangler or the embedding model.
- **C L0:** Neither extension type is verified. — [core/wren/src/wren/genbi/providers/cloudflare.py:29-32](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/providers/cloudflare.py#L29-L32); [core/wren/src/wren/memory/embeddings.py:114-118](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/memory/embeddings.py#L114-L118) (verified)
  - *To reach the next level:* No integrity verification for any extension type.
- **D L1:** wrangler/the model are fetched on first use of the Cloudflare provider or memory extra, without showing what will run. — [core/wren/src/wren/genbi/providers/cloudflare.py:29-32](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/providers/cloudflare.py#L29-L32) (inferred)
  - *To reach the next level:* No explicit install step that shows the exact package before it runs.
- **B L0:** wrangler runs as the same user with the full environment; the embedding model loads in-process. — [core/wren/src/wren/genbi/providers/cloudflare.py:80-84](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/providers/cloudflare.py#L80-L84); [core/wren/src/wren/memory/embeddings.py:106-111](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/memory/embeddings.py#L106-L111) (verified)
  - *To reach the next level:* Extensions get everything the CLI process holds; L1 needs at least a separate process with a scrubbed environment for the model path.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.50 (high)

Secret handling is one of Wren's better areas. Connection secrets are typed as masked values, profiles and Wren Cloud keys are stored in owner-only files, profiles can hold ${VAR} placeholders that are filled from the environment only at connection time, `profile debug` masks sensitive keys, tokens are never taken as command-line flags, and deploys refuse to ship `.env` files or obvious inlined credentials. There is no telemetry. Gaps: stored secrets are plaintext on disk, the wrangler subprocess receives the whole environment, and database passwords and service-account keys are long-lived.

- **S L2:** SecretStr connection fields, masked debug output, 0600 plaintext profile and cloud files, ${VAR} indirection resolved at connect time. — [core/wren/src/wren/model/__init__.py:118](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/model/__init__.py#L118); [core/wren/src/wren/profile.py:181-197](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/profile.py#L181-L197); [core/wren/src/wren/profile.py:358-369](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/profile.py#L358-L369) (verified)
  - *To reach the next level:* Secrets are not kept in an OS keychain/secret manager or encrypted at rest.
- **C L2:** Debug output, deploy bundles and data-reader error messages are covered; subprocess environments are not. — [core/wren/src/wren/genbi/verify.py:46-65](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/verify.py#L46-L65); [core/wren/src/wren/genbi/providers/vercel.py:49-52](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/providers/vercel.py#L49-L52); [core/wren/src/wren/genbi/providers/cloudflare.py:80-84](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/providers/cloudflare.py#L80-L84) (verified)
  - *To reach the next level:* The wrangler subprocess receives every environment secret.
- **D L3:** No telemetry at all and masking is always on in debug output. — searched `rg -n -i 'posthog|sentry|telemetry'` in `core/wren/src/wren core/wren/pyproject.toml` → 0 hits (No telemetry or crash-reporting SDK in the CLI package.); [core/wren/src/wren/profile.py:358-369](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/profile.py#L358-L369) (verified)
  - *To reach the next level:* Stored profiles are not encrypted or minimised by default.
- **B L1:** Long-lived DB passwords, service-account keys and deploy tokens; only Wren Cloud git pushes use 600-second tokens. — [core/wren/src/wren/model/__init__.py:118](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/model/__init__.py#L118); [core/wren/src/wren/cloud.py:118](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/cloud.py#L118) (verified)
  - *To reach the next level:* Most credentials are long-lived and not narrowly scoped by Wren.
- **Cap:** none

### C9 Audit & traceability — 0.12 (high)

Wren keeps no record of the queries it runs or the commands it executes: there are no log statements on the query path, and the MCP server does not log tool calls. The only durable trace of an action is the last-deploy entry written into the project's .wren/apps.yml, which sits inside the workspace and is overwritten on the next deploy. Reconstructing what an agent did through Wren depends entirely on the host agent's own transcript.

- **S L1:** Only the last deployment (provider, URL, date) is written to the project index. — [core/wren/src/wren/genbi/cli.py:367-376](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/cli.py#L367-L376); searched `rg -n 'logger\.|logging\.'` in `core/wren/src/wren/engine.py core/wren/src/wren/cli.py` → 0 hits (No log statement anywhere on the query path (engine facade and CLI).) (verified)
  - *To reach the next level:* No structured record of every query and command with arguments and timestamps.
- **C L0:** The main path (SQL execution) is not recorded at all. — searched `rg -n 'logger\.|logging\.'` in `core/wren/src/wren/engine.py core/wren/src/wren/cli.py` → 0 hits (No log statement anywhere on the query path (engine facade and CLI).) (verified)
  - *To reach the next level:* Query and command execution are unrecorded.
- **D L1:** The deploy record lives in <project>/.wren/apps.yml, editable by the agent. — [core/wren/src/wren/genbi/index.py:18](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/index.py#L18) (verified)
  - *To reach the next level:* Records live inside the agent-writable workspace.
- **B L0:** Actions proceed whether or not anything is recorded. — searched `rg -n 'logger\.|logging\.'` in `core/wren/src/wren/engine.py core/wren/src/wren/cli.py` → 0 hits (No log statement anywhere on the query path (engine facade and CLI).) (verified)
  - *To reach the next level:* No per-action record, so nothing is flushed or durable.
- **Cap:** none

### C10 Limits & kill switch — 0.33 (high)

Wren does not own the agent loop, so its limits are bounds on its own work. The MCP server caps query results at 1,000 rows by default and 10,000 maximum, deploy subprocesses time out after 10 minutes and Vercel calls after 2 minutes. The README-led CLI path, however, returns unlimited rows when no --limit is given, and there is no default statement timeout or cost cap, so a runaway BigQuery or Snowflake query bills until the warehouse stops it.

- **S L2:** Server-enforced row caps on MCP queries and timeouts on deploy calls. — [core/wren/src/wren/mcp_server.py:84-88](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/mcp_server.py#L84-L88); [core/wren/src/wren/genbi/providers/cloudflare.py:40-41](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/genbi/providers/cloudflare.py#L40-L41) (verified)
  - *To reach the next level:* No caps on every operation (CLI query rows, statement time, query cost).
- **C L1:** Row caps apply to MCP tools only; the CLI query path passes limit=None through. — [core/wren/src/wren/cli.py:441](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/cli.py#L441); [core/wren/src/wren/cli.py:302-304](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/cli.py#L302-L304) (verified)
  - *To reach the next level:* CLI queries are unbounded.
- **D L1:** CLI default is unlimited rows and no statement timeout (BigQuery job_timeout_ms defaults to None). — [core/wren/src/wren/model/__init__.py:53](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/model/__init__.py#L53); [core/wren/src/wren/cli.py:302-304](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/cli.py#L302-L304) (verified)
  - *To reach the next level:* No sensible default bounds on the primary CLI path.
- **B L1:** A runaway warehouse query is bounded only by the warehouse's own settings and can incur unbounded cost. — [core/wren/src/wren/model/__init__.py:53](https://github.com/Canner/WrenAI/blob/2cc843fd87d6cfe9831554721559018dda2938a0/core/wren/src/wren/model/__init__.py#L53) (verified)
  - *To reach the next level:* No default per-query time or cost ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Database rows, project knowledge/AGENTS.md and raw docs returned to the agent (core/wren/src/wren/mcp_server.py:367-372) · [B] sensitive data/systems: Warehouse data via stored connection profiles (core/wren/src/wren/cli.py:248) · [C] state change / egress: `wren genbi deploy` publishes to Vercel/Cloudflare (core/wren/src/wren/genbi/cli.py:355-362) · Same default session? Yes

## Highest-impact improvements
1. Drop `allowed-tools: Bash(wren:*)` from the skill stub (or narrow it to read-only commands) and require an interactive confirmation in `wren genbi deploy`. — C2 S L0→L2, +0.150 before caps (Playbook 5)
2. Turn strict mode on by default so queries are confined to modelled tables and file/URL readers are blocked. — C3 D L1→L3, +0.100 before caps (Playbook 3)
3. Stop auto-loading `.env` from the working directory/project and ignore the project `profile:` pin unless the user confirms it once. — C6 S L1→L3, +0.150 before caps (Playbook 2)
4. Open database sessions read-only (e.g. SET SESSION CHARACTERISTICS AS TRANSACTION READ ONLY) and pass wrangler a scrubbed environment. — C4 S L0→L2, +0.150 before caps (Playbook 3)
5. Apply a default row cap and statement timeout on the CLI query path, and append each executed query to a log outside the project. — C10 D L1→L2, +0.050 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- At this commit the hosted WrenAI GenBI app (wren-ui, wren-ai-service, docker/deployment) has moved to the legacy/v1 branch; the queue's shape `hosted_service` was kept as given, but the trust boundary actually scored is the `wrenai` CLI/MCP server driven by a host agent. Because Wren does not own the agent loop, tool-server anchors were used for C2 (alt) and C10.
- The Rust engine (core/wren-core) was reviewed only where it bears on access control (RLAC/CLAC); SDK packages (sdk/wren-langchain, sdk/wren-pydantic) were checked only to confirm they use the same WrenEngine policy path.
- Whether npx auto-installs wrangler without a prompt in a non-TTY session and whether lancedb's trust_remote_code defaults to False are third-party behaviours inferred, not verified in this repo.
- No reviewer-injection text aimed at AI auditors was found in the repository.
