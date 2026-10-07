# Defense-in-Depth Score: Neon MCP

**Repo:** https://github.com/neondatabase/mcp-server-neon · **Commit:** `00d82d4e5c925380fc077fe1932780035f564d8b` · **Reviewed:** 2026-10-03
**What it is:** MCP server for Neon Management API and databases
**Category:** Data & Analytics
**Scored configuration:** Hosted remote server (mcp.neon.tech) as the README leads with it: OAuth or API-key bearer auth, default grant with no readonly, category or projectId parameters (consent page with writes pre-checked).
**Agent surface (default):** code execution yes · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 4.6 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L2 | L0 | L0 | 0.23 | C1-PASSTHRU | **0.23** | High |
| C2 | Approval gates | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C3 | Tool & action scoping | L1 | L2 | L2 | L0 | 0.33 | — | **0.33** | High |
| C4 | Code-execution isolation | L2 | L3 | L0 | L2 | 0.47 | G1 | **0.47** | High |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L0 | 0.07 | C5-WORSTCASE | **0.07** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L2 | L1 | L0 | 0.28 | C8-MODELSECRETS | **0.25** | High |
| C9 | Audit & traceability | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |

Controls where a risk surface exists: 2.58 / 8.0 (32%); 2 criteria scored SA (surface absent).

Neon MCP gives an AI client full write and delete authority over a user's whole Neon account by default: arbitrary SQL as the database owner, project and branch deletion, function deploys and auth changes. Read-only, category and project restrictions exist and are enforced in code, but they are opt-in, and credential handling in the OAuth flow is not strictly scoped. The server also routinely returns database passwords to the model. Use a readonly=true or projectId-scoped URL.

## Critical gaps
- A hijacked session or stolen token has write and delete authority over the user's whole Neon account, including organizations and every database. (ASI03, T3; C1) — [lib/oauth/client.ts:116-129](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/lib/oauth/client.ts#L116-L129); [lib/oauth/client.ts:208](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/lib/oauth/client.ts#L208)
- Worst case under prompt injection: a hijacked default session can exfiltrate data and secrets and run irreversible SQL and deletions with no server-side human step. (ASI01, LLM01, T6; C5) — [mcp/tools/tools.ts:92](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/tools.ts#L92); [mcp/tools/tools.ts:1181](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/tools.ts#L1181)
- Database owner-role passwords are routinely returned to the model through get_connection_string, which the create tools tell the model to call. (ASI03, LLM02, T9; C8) — [mcp/tools/tools.ts:1181](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/tools.ts#L1181); [mcp/tools/generated/adapt.ts:28](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/generated/adapt.ts#L28)

## Criterion details

### C1 Identity & least privilege — 0.23 (high)

The server acts on Neon with the user's own authority. In the OAuth flow it requests broad project and organization scopes (create, update, delete, org permissions); in the API-key flow it forwards the user's Neon API key. Read-only, category and project restrictions are enforced inside the server by filtering tools and pinning project_id; handling of the issued credential is not strictly scoped. The default grant is read-write across every project and organization the user can reach.

- **S L1:** Per-client OAuth grant, but the upstream token carries full projects:* and orgs:* scopes (including orgs:permission) and API keys are the user's full key; narrowing is a server-side tool filter only, and credential handling is not strictly scoped. — [lib/oauth/client.ts:116-129](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/lib/oauth/client.ts#L116-L129); [lib/oauth/client.ts:208](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/lib/oauth/client.ts#L208); [app/api/[transport]/route.ts:385-388](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/app/api/[transport]/route.ts#L385-L388) (verified)
  - *To reach the next level:* Request minimal Neon OAuth scopes that match the user's grant.
- **C L2:** Every tool, host and generated, is registered through the grant filter and invoked through invokeTool, which pins project_id, but enforcement is not a strict boundary. — [app/api/[transport]/route.ts:474-480](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/app/api/[transport]/route.ts#L474-L480); [mcp/tools/grant-filter.ts:286-292](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/grant-filter.ts#L286-L292); [app/api/[transport]/route.ts:1085](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/app/api/[transport]/route.ts#L1085) (verified)
  - *To reach the next level:* Make the authorization layer cover every path by which the credential can be used.
- **D L0:** With no scope requested the OAuth flow defaults to read+write, the consent page pre-checks writes, and the API-key path is read-write unless readonly is set; all projects and categories are in scope. — [mcp/oauth/issued-scopes.ts:4](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/oauth/issued-scopes.ts#L4); [app/api/authorize/route.ts:284](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/app/api/authorize/route.ts#L284); [mcp/utils/read-only.ts:76](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/utils/read-only.ts#L76) (verified)
  - *To reach the next level:* Default to read-only (or one project) and require an explicit opt-in to writes.
- **B L0:** A hijacked session or leaked token can create, update and delete any project in the user's personal account and organizations, manage org permissions, and read and write every database. — [lib/oauth/client.ts:116-129](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/lib/oauth/client.ts#L116-L129); [mcp/tools/handlers/connection-string.ts:66](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/handlers/connection-string.ts#L66) (verified)
  - *To reach the next level:* Bound the token to one project, read-only, short-lived scopes so a hijack cannot reach the whole Neon account.
- **Cap:** C1-PASSTHRU — Credential handling between the MCP client and the Neon API is not strictly scoped.

### C2 Approval gates — 0.38 (high)

As a hosted tool server, Neon MCP leaves approval to the MCP client and contributes risk signalling. Every tool carries readOnly/destructive hints, deletes and updates are marked destructive, and the descriptions of dangerous tools tell the model to ask first; but run_sql and run_sql_transaction mix reads and writes in one tool, and there is no server-side confirmation step. Migrations and query tuning use a prepare-on-temporary-branch, then complete pattern that works as a preview, but project and branch deletion, raw SQL, function deploys and auth changes have none. A server-enforced read-only mode exists but is off by default.

- **S L1:** Annotations are present on all tools and broadly accurate, but the most powerful tools (run_sql, run_sql_transaction) mix reads and writes, and 'ask the user first' lives only in description text. — [mcp/tools/definitions.ts:59-71](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/definitions.ts#L59-L71); [mcp/tools/definitions.ts:62](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/definitions.ts#L62); [mcp/tools/generated/adapt.ts:210-218](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/generated/adapt.ts#L210-L218) (verified)
  - *To reach the next level:* Split SQL into separate read and write tools so the hints let a host gate writes without gating reads.
- **C L2:** Host and generated tools all receive annotations from one path (explicit for host tools, method-based for generated ones), and the read-only filter uses the same readOnlySafe flag for every tool. — [mcp/tools/generated/adapt.ts:183](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/generated/adapt.ts#L183); [mcp/tools/generated/adapt.ts:143](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/generated/adapt.ts#L143); [mcp/tools/grant-filter.ts:203](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/grant-filter.ts#L203) (verified)
  - *To reach the next level:* Offer previews or a server confirmation step that covers destructive generated tools, not just migrations and tuning.
- **D L2:** Hints are always sent and cannot be changed by the model, but the server's enforceable control (read-only mode) is off by default and the consent page pre-checks writes. — [app/api/authorize/route.ts:284](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/app/api/authorize/route.ts#L284); [mcp/utils/read-only.ts:76](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/utils/read-only.ts#L76) (verified)
  - *To reach the next level:* Ship read-only as the default grant, with writes as an explicit, visible opt-in.
- **B L1:** Wrongly approved calls can delete branches and data, run arbitrary DML/DDL on main branches, deploy code and change auth settings; only project deletion (recover_project) and migrations (temporary branch) have an undo or preview. — [mcp/tools/generated/operations.ts:10](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/generated/operations.ts#L10); [mcp/tools/tools.ts:92](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/tools.ts#L92); [mcp/tools/generated/operations.ts:85](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/generated/operations.ts#L85) (verified)
  - *To reach the next level:* Add server-side previews or dry-runs and quantity limits for destructive operations.
- **Cap:** none

### C3 Tool & action scoping — 0.33 (high)

Most of the roughly 110 tools are narrow, typed Neon API operations with strict schemas, and a project-scoped grant pins project_id in code. The tools that matter most are raw passthroughs, though: run_sql, run_sql_transaction and explain_sql_statement send any SQL the model writes, as the database owner role, and deploy_function uploads arbitrary code. By default every category is enabled, including writes, across every project; category and project filters are opt-in. SQL has no row or statement limit.

- **S L1:** Strict zod schemas on all tools and project_id pinning, but the SQL tools pass arbitrary SQL straight to Postgres with no allowlist, parameterisation or bounds. — [mcp/tools/tools.ts:92](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/tools.ts#L92); [mcp/tools/grant-filter.ts:286-292](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/grant-filter.ts#L286-L292) (verified)
  - *To reach the next level:* Constrain SQL (restricted role, statement classes, row caps) rather than passing it raw.
- **C L2:** Every tool is parsed by its schema before execution (host tools via parseHost, generated via tool.inputSchema.parse), but the validation on the SQL arguments is just a string type. — [mcp/tools/generated/adapt.ts:276](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/generated/adapt.ts#L276); [mcp/tools/toolsSchema.ts:24](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/toolsSchema.ts#L24) (verified)
  - *To reach the next level:* Apply a shared policy layer that bounds what SQL and code-deploy arguments can do, not just their types.
- **D L2:** Tool categories and a read-only mode are selectable, but the default grant (scopes null) publishes every category including write, delete and code-deploy tools. — [mcp/utils/grant-context.ts:43-46](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/utils/grant-context.ts#L43-L46); [mcp/utils/read-only.ts:76](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/utils/read-only.ts#L76) (verified)
  - *To reach the next level:* Make the default tool set read-only, with write categories requiring explicit enabling.
- **B L0:** A misused run_sql can read or modify any table in any database of any project the account reaches, with no row or statement cap. — [mcp/tools/tools.ts:92](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/tools.ts#L92); [mcp/tools/handlers/connection-string.ts:66](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/handlers/connection-string.ts#L66); searched `rg -n -i 'statement_timeout|rowLimit|max_rows' --glob '!**/__tests__/**'` in `mcp app lib` → 0 hits (no SQL statement timeout or row cap) (verified)
  - *To reach the next level:* Scope SQL to one project and bound its quantity (rows, statement time).
- **Cap:** none

### C4 Code-execution isolation — 0.47 (high)

Nothing the model writes runs on the server: SQL goes to Neon's managed Postgres over HTTPS, and deployed functions run on Neon's platform. That is a real separation from the MCP host, but SQL runs as the database owner role (a neon_superuser member) directly on the chosen branch, usually the main one. The opt-in read-only mode wraps SQL in a read-only transaction using the same owner role rather than a read-only role. Only the migration and tuning flows use a throwaway branch first.

- **S L2:** Model SQL executes remotely in Neon's Postgres compute, not in the server process, but with the owner role and no restricted role or sandboxed copy by default. — [mcp/tools/tools.ts:92](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/tools.ts#L92); [mcp/tools/handlers/connection-string.ts:66](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/handlers/connection-string.ts#L66) (verified)
  - *To reach the next level:* Run model SQL as a restricted role or on an ephemeral branch by default.
- **C L3:** Every execution path (run_sql, run_sql_transaction, explain, migrations, tuning, describe/inspect queries, function deploys) runs on Neon infrastructure; nothing spawns a process on the server host. — [mcp/tools/tools.ts:92](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/tools.ts#L92); searched `rg -n 'child_process|spawn\(|execSync|await import\(|new Function|eval\(' --glob '!**/__tests__/**'` in `mcp app lib` → 3 hits (all three are redis.eval calls of fixed, in-repo Lua scripts in mcp/oauth/refresh-lock.ts (216, 238, 246); no subprocess or dynamic code loading) (verified)
  - *To reach the next level:* Route every SQL path through a restricted or ephemeral execution context with no fallback to the owner role.
- **D L0:** No restriction on executed SQL is on by default: the read-only transaction is used only when the opt-in read-only mode is set. — [mcp/tools/tools.ts:84-87](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/tools.ts#L84-L87); [mcp/utils/read-only.ts:76](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/utils/read-only.ts#L76) (verified)
  - *To reach the next level:* Turn the read-only transaction or restricted role on by default and require an explicit opt-in for writes.
- **B L2:** Inside the database the SQL has the owner role's rights over the whole branch's data, but no server secrets or host filesystem are reachable from it. — [mcp/tools/handlers/connection-string.ts:66](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/handlers/connection-string.ts#L66) (verified)
  - *To reach the next level:* Execute on an ephemeral branch with a role limited to the task, so a misuse cannot reach production data.
- **Cap:** G1 — The only restriction on executed SQL, the read-only transaction, applies only in the opt-in read-only mode.

### C5 Untrusted input blast radius — 0.07 (high)

Database rows, column comments, logs and function code can all hold attacker-written text, and the server returns them to the model as plain JSON with no provenance or untrusted marking. If the model is hijacked by that content, the same session can run any SQL, delete projects and branches, fetch connection strings with passwords, and deploy functions with outbound network access, with nothing in the server asking a human. The read-only and project-scoped modes would cut most of that but are off by default.

- **S L1:** Results are plain JSON text with no provenance; the only directives are static safety notes in tool descriptions, not mixed into returned data. — [mcp/tools/generated/adapt.ts:242-250](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/generated/adapt.ts#L242-L250); searched `rg -n -i 'untrusted|injection|provenance' --glob '!**/__tests__/**'` in `mcp app lib` → 0 hits (no provenance tagging or untrusted-content handling in server code) (verified)
  - *To reach the next level:* Return structured content with source metadata and an untrusted flag the host can act on.
- **C L0:** No source is distinguished: SQL rows, schemas, logs and API results all enter context the same way. — searched `rg -n -i 'untrusted|injection|provenance' --glob '!**/__tests__/**'` in `mcp app lib` → 0 hits (no provenance tagging or untrusted-content handling in server code) (verified)
  - *To reach the next level:* Mark every source of user-controlled data (rows, comments, logs, function code) as untrusted.
- **D L0:** There is no untrusted-input limit to switch on; the modes that drop a Rule-of-Two leg (read-only, project scope) are off by default. — [mcp/utils/read-only.ts:76](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/utils/read-only.ts#L76); [mcp/oauth/issued-scopes.ts:4](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/oauth/issued-scopes.ts#L4) (verified)
  - *To reach the next level:* Default sessions to read-only so a hijack cannot change state.
- **B L0:** A hijacked default session can both leak secrets and data (get_connection_string, run_sql, function deploys with egress) and take irreversible actions (DROP, delete_branch, delete_project) with no human step in the server. — [mcp/tools/tools.ts:1181](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/tools.ts#L1181); [mcp/tools/tools.ts:92](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/tools.ts#L92); [mcp/tools/generated/operations.ts:85](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/generated/operations.ts#L85) (verified)
  - *To reach the next level:* Remove either the state-change leg or the sensitive-data leg from default sessions.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 1.00 (high)

The server keeps no memory the model can write and loads no workspace or instruction files. Its persistent state is OAuth tokens, grants and session bindings, which the model cannot influence. Data written into a database can be read in a later session, but that is untrusted input (C5), not server memory.

- **Structural absence:** searched `rg -n 'load_dotenv|dotenv|readFileSync|readFile\(|AGENTS\.md|CLAUDE\.md' --glob '!**/__tests__/**'` in `mcp app lib` → 0 hits (no auto-loaded workspace/config files at runtime); searched `rg -n 'memory|remember|vector|embedding' --glob '!**/__tests__/**'` in `mcp app lib` → 3 hits (all three are comments (tools.ts:311 'No in-memory state storage', docs.ts:8 'vector without authentication', refresh-lock.ts:8 'in-memory singleflight'); no memory store)

### C7 Third-party extensions — 1.00 (high)

The server loads or launches no third-party code at runtime: no plugins, no subprocesses, no dynamic imports, and it does not connect to other MCP servers. Its tools call the Neon API, Neon Postgres and neon.com docs only. Its @neon/tools dependency is a build-time package and out of scope here.

- **Structural absence:** searched `rg -n 'child_process|spawn\(|execSync|await import\(|new Function|eval\(' --glob '!**/__tests__/**'` in `mcp app lib` → 3 hits (all three are redis.eval calls of fixed, in-repo Lua scripts in mcp/oauth/refresh-lock.ts (216, 238, 246); no subprocess or dynamic code loading); searched `rg -n 'StdioClientTransport|StreamableHTTPClientTransport|plugin|loadExtension' --glob '!**/__tests__/**'` in `mcp app lib` → 0 hits (server acts as no MCP client and has no plugin loader)

### C8 Secrets & sensitive-data protection — 0.25 (high)

The server deliberately puts database secrets into the model's context: get_connection_string returns the full Postgres URI including the owner role's password, and tool descriptions tell the model to call it after creating a project or branch; role creation and password reset also return passwords. It does strip connection URIs and client secrets from other Management API results and redacts Neon Auth secrets. Token storage and logging do not fully protect credentials. Segment analytics (with user email) and Sentry with default PII and 100% tracing are on by default.

- **S L1:** A sanitizer drops connection URIs and secrets from generated results and refresh tokens are masked in logs, but connection-string passwords go to the model by design, and token storage and logging do not fully protect credentials. — [mcp/tools/tools.ts:1181](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/tools.ts#L1181); [mcp/tools/generated/sanitize.ts:3-6](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/generated/sanitize.ts#L3-L6); [mcp/tools/generated/sanitize.ts:8-13](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/generated/sanitize.ts#L8-L13); [app/api/[transport]/route.ts:958](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/app/api/[transport]/route.ts#L958) (verified)
  - *To reach the next level:* Keep credentials out of model-bound results and harden token storage.
- **C L2:** Masking covers some log lines and generated model-bound results, but not get_connection_string, role passwords, or the telemetry path. — [mcp/tools/generated/sanitize.ts:8-13](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/generated/sanitize.ts#L8-L13); [lib/oauth/client.ts:55](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/lib/oauth/client.ts#L55); [mcp/tools/tools.ts:1181](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/tools.ts#L1181) (verified)
  - *To reach the next level:* Extend redaction to every model-bound result and to telemetry.
- **D L1:** Segment analytics with a hardcoded write key (identifying users by email) and Sentry with sendDefaultPii are on by default; tool-call events carry no arguments. — [lib/config.ts:32](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/lib/config.ts#L32); [mcp/analytics/analytics.ts:10](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/analytics/analytics.ts#L10); [mcp/analytics/analytics.ts:36](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/analytics/analytics.ts#L36); [mcp/sentry/instrument.ts:13](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/sentry/instrument.ts#L13) (verified)
  - *To reach the next level:* Make telemetry opt-in and turn off default PII collection.
- **B L0:** Leaked material includes long-lived owner-role database passwords and server-held refresh tokens for full-scope Neon OAuth grants. — [mcp/tools/tools.ts:1181](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/tools.ts#L1181); [lib/oauth/client.ts:116-129](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/lib/oauth/client.ts#L116-L129) (verified)
  - *To reach the next level:* Use short-lived, narrowly scoped credentials so a leak is bounded.
- **Cap:** C8-MODELSECRETS — get_connection_string returns the owner role's password to the model, and create_project/create_branch descriptions tell the model to call it routinely.

### C9 Audit & traceability — 0.38 (high)

Every tool call passes through one wrapper that writes a log line (tool name, read-only and project-scope flags, client, trace id) before running, and sends a matching analytics event with the account id. Arguments are not recorded, so after an incident you can see that run_sql or delete_project was called but not on what or with which SQL. Logs go to the console, which Vercel collects off-host, beyond the model's reach. There is no tamper-evident audit trail or record of results.

- **S L1:** Log lines record the tool name and context flags but not arguments or targets; analytics adds the account id. — [app/api/[transport]/route.ts:435-446](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/app/api/[transport]/route.ts#L435-L446); [app/api/[transport]/route.ts:452-462](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/app/api/[transport]/route.ts#L452-L462) (verified)
  - *To reach the next level:* Record arguments (redacted), target project/branch and result status for every call.
- **C L2:** All tools, host and generated, go through the same logging wrapper, including the anonymous docs-only path. — [app/api/[transport]/route.ts:435-446](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/app/api/[transport]/route.ts#L435-L446); [app/api/[transport]/route.ts:757](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/app/api/[transport]/route.ts#L757) (verified)
  - *To reach the next level:* Also record OAuth consent decisions, grant changes and credential use in the same audit stream.
- **D L2:** Logging is on by default and goes to stdout collected by the hosting platform, outside anything a tool can write. — [mcp/utils/logger.ts:11-19](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/utils/logger.ts#L11-L19) (verified)
  - *To reach the next level:* Write audit records through a component with an integrity guarantee and logged configuration.
- **B L1:** The log line is written before execution, but logging and analytics are best-effort and actions proceed if they fail. — [app/api/[transport]/route.ts:435-446](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/app/api/[transport]/route.ts#L435-L446); [app/api/[transport]/route.ts:474-480](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/app/api/[transport]/route.ts#L474-L480) (verified)
  - *To reach the next level:* Surface logging failures and flush a durable record per action.
- **Cap:** none

### C10 Limits & kill switch — 0.45 (high)

The server bounds some of its own work: each request is capped by the 800-second Vercel function limit, create operations wait at most 120 seconds, docs fetches time out after 10 seconds, and inspection queries cap their row limit. Generated Management API calls receive the client's cancellation signal. But SQL has no statement timeout or row cap, and there are no rate limits on SQL, deletions, or creating projects, branches and computes, all of which keep running in Neon after a request ends.

- **S L2:** Server-enforced caps exist on some operations (function duration, create waits, docs timeout, inspect limit) and API calls get the abort signal. — [app/api/[transport]/route.ts:112](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/app/api/[transport]/route.ts#L112); [mcp/tools/generated/adapt.ts:63](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/generated/adapt.ts#L63); [mcp/tools/handlers/docs.ts:9](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/handlers/docs.ts#L9); [mcp/inspect/queries.ts:558](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/inspect/queries.ts#L558); [mcp/tools/generated/adapt.ts:294-297](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/generated/adapt.ts#L294-L297) (verified)
  - *To reach the next level:* Cap every operation, including SQL statement time and result size, and add rate limits.
- **C L2:** The platform duration cap covers every request and some tools add their own timeouts, but run_sql and run_sql_transaction have none. — [vercel.json:5](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/vercel.json#L5); searched `rg -n -i 'statement_timeout|rowLimit|max_rows' --glob '!**/__tests__/**'` in `mcp app lib` → 0 hits (no SQL statement timeout or row cap) (verified)
  - *To reach the next level:* Apply timeouts and size caps to SQL tools as well.
- **D L2:** Limits are hard-coded constants the model cannot raise, but caller-supplied limit parameters and unbounded SQL leave the defaults loose. — [mcp/inspect/queries.ts:558](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/inspect/queries.ts#L558); [mcp/tools/generated/adapt.ts:63](https://github.com/neondatabase/mcp-server-neon/blob/00d82d4e5c925380fc077fe1932780035f564d8b/mcp/tools/generated/adapt.ts#L63) (verified)
  - *To reach the next level:* Enforce server-side ceilings that caller arguments cannot exceed on every tool.
- **B L1:** A runaway client can create projects, branches and computes or run heavy SQL repeatedly with no server-side rate limit, and upstream operations continue after the request stops. — searched `rg -n -i 'rate.?limit' --glob '!**/__tests__/**'` in `mcp app lib` → 2 hits (both hits are the NeonApiError kind 'rate_limit' type check in mcp/server/errors.ts (classifying upstream 429s), not a limiter) (verified)
  - *To reach the next level:* Add per-account rate and quantity limits on consequential and billable operations.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Database rows, comments and logs returned as plain JSON (mcp/tools/generated/adapt.ts:247, mcp/tools/tools.ts:92) · [B] sensitive data/systems: Every database in the account via owner-role connection strings (mcp/tools/handlers/connection-string.ts:66, mcp/tools/tools.ts:1181) · [C] state change / egress: run_sql writes, delete_project/delete_branch, deploy_function (mcp/tools/tools.ts:92, mcp/tools/generated/operations.ts:85) · Same default session? Yes

## Highest-impact improvements
1. Make read-only the default grant (unchecked 'Allow writes', readonly default for API keys) so writes need an explicit opt-in. — C3 D L2→L3, +0.050 before caps (Playbook 3)
2. Request Neon OAuth scopes that match each grant (read-only scopes for read-only grants). — C1 S L1→L3, +0.150 before caps (Playbook 4)
3. Default to read-only so a hijacked session cannot change state (drop the state-change leg of the Rule of Two). — C5 D L0→L2, +0.100 before caps (Playbook 1)
4. Record redacted arguments and target project/branch for every tool call. — C9 S L1→L2, +0.075 before caps (Playbook 1 step 3)
5. Add a SQL statement timeout, row cap, and per-account rate limits on create/delete operations. — C10 B L1→L2, +0.050 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The ~95 generated Management API tools come from the external @neon/tools package (v2.0.1), which is not in this repo; their schemas, requiresApproval flags and annotations were judged from the adapter code and the committed snapshot mcp/__tests__/__snapshots__/hosted-tools-catalog.md.
- Not verified: the strength of the read-only SQL transaction, which uses the owner role and depends on PostgreSQL transaction semantics that were not tested.
- Neon platform behaviour (point-in-time restore, project recovery windows, upstream OAuth scope enforcement, Sentry's header scrubbing under sendDefaultPii) was not examined and is not credited.
- No reviewer-steering text was found in README, AGENTS.md, .agents/skills or code comments.
