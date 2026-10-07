# Defense-in-Depth Score: Supabase MCP

**Repo:** https://github.com/supabase/mcp · **Commit:** `4602ee9ebf025611741cc127cee2a3c6a4f73854` · **Reviewed:** 2026-10-03
**What it is:** Connect Supabase projects to AI assistants (SQL, migrations, edge functions, branching)
**Category:** Data & Analytics
**Scored configuration:** The npm package's CLI (@supabase/mcp-server-supabase 0.13.0) over stdio with a personal access token from SUPABASE_ACCESS_TOKEN and no flags: default feature groups, all projects, read-write, no elicitation confirmations.
**Agent surface (default):** code execution yes · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 4.6 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L2 | L1 | 0.30 | G1 | **0.30** (alt) | High |
| C2 | Approval gates | L1 | L2 | L2 | L0 | 0.33 | — | **0.33** | High |
| C3 | Tool & action scoping | L1 | L2 | L2 | L0 | 0.33 | — | **0.33** | High |
| C4 | Code-execution isolation | L2 | L3 | L0 | L2 | 0.47 | G1 | **0.47** (alt) | Medium |
| C5 | Untrusted input blast radius | L1 | L1 | L2 | L0 | 0.25 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L2 | L2 | L0 | 0.33 | — | **0.33** | High |
| C9 | Audit & traceability | L1 | L2 | L0 | L1 | 0.28 | G1 | **0.28** (alt) | High |
| C10 | Limits & kill switch | L2 | L1 | L1 | L1 | 0.33 | — | **0.33** | High |

Controls where a risk surface exists: 2.60 / 8.0 (32%); 2 criteria scored SA (surface absent).

Run as shipped, this server gives the model your whole Supabase account: any SQL on any project's database (read-write by default), migrations, edge-function deploys, branch merges and paid project creation, all through one long-lived personal access token. Destructive actions are clearly labelled for the MCP host, and the server has good opt-in controls (--read-only, --project-ref, and HTTP-mode confirmations bound to the exact query), but none are on by default. A prompt injection in database rows can lead to data theft and irreversible changes unless the host stops it. Run it with --read-only and --project-ref against a development project.

## Critical gaps
- The default configuration uses an account-wide personal access token for every tool, so a hijacked session controls every organization and project the user can reach. (ASI03, T3; C1) — [packages/mcp-server-supabase/src/cli.ts:98](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/cli.ts#L98); [packages/mcp-server-supabase/src/server.ts:46-52](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/server.ts#L46-L52)
- Rule of Two violated by default: untrusted database rows enter the same session that can run destructive SQL and deploy public edge functions, with no server-side human step (C5-WORSTCASE). (ASI01, LLM01, T6; C5) — [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:518](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L518); [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:512-515](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L512-L515); [packages/mcp-server-supabase/src/tools/edge-function-tools.ts:124-146](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/edge-function-tools.ts#L124-L146)

## Criterion details

### C1 Identity & least privilege — 0.30 (high)

The server acts with whatever Supabase personal access token it is given, and that token reaches every organization and project the user belongs to, with full management rights. By default nothing narrows it: all projects are reachable, account-level tools (create, pause, restore projects) are on, and SQL runs read-write. An operator can pin the server to one project with --project-ref, which removes the project parameter from most tools, but the pin does not cover every path. The token itself is never narrowed or exchanged.

- **default configuration** (default; raw 0.07 → 0.07)
  - **S L0:** Every tool call uses the same long-lived personal access token, which grants the user's full Supabase account across organizations and projects. — [packages/mcp-server-supabase/src/cli.ts:98](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/cli.ts#L98); [packages/mcp-server-supabase/src/management-api/index.ts:19-24](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/management-api/index.ts#L19-L24); [packages/mcp-server-supabase/src/server.ts:46-52](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/server.ts#L46-L52) (verified)
    - *To reach the next level:* No scoped, per-capability or read-only credential; the full account PAT is used for reads and writes alike.
  - **C L1:** All tools share one platform client with the PAT, and the Supabase Management API authorizes each call against the token owner, but the server itself performs no authorization check per request. — [packages/mcp-server-supabase/src/management-api/index.ts:19-24](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/management-api/index.ts#L19-L24); [packages/mcp-server-supabase/src/platform/api-platform.ts:75-90](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/platform/api-platform.ts#L75-L90) (verified)
    - *To reach the next level:* No authorization layer in the server mapping each tool call to a least-privilege policy before the credential is attached.
  - **D L0:** Default CLI run exposes all projects, account tools and read-write SQL; least privilege needs --project-ref and --read-only. — [packages/mcp-server-supabase/src/cli.ts:36-39](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/cli.ts#L36-L39); [packages/mcp-server-supabase/src/server.ts:230](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/server.ts#L230); [packages/mcp-server-supabase/src/server.ts:110-118](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/server.ts#L110-L118) (verified)
    - *To reach the next level:* Default is not read-only or project-scoped; write access is not an explicit elevation.
  - **B L0:** A hijacked session holds the user's entire Supabase account: create/pause projects, run destructive SQL on any production database, deploy edge functions. — [packages/mcp-server-supabase/src/server.ts:46-52](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/server.ts#L46-L52); [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:512-515](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L512-L515); [packages/mcp-server-supabase/src/tools/edge-function-tools.ts:124-146](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/edge-function-tools.ts#L124-L146) (verified)
    - *To reach the next level:* Blast radius is not limited to one project or to read-mostly operations.
- **operator flag --project-ref (project pinning)** (alt; raw 0.30, cap G1 → 0.30) ← counted
  - **S L1:** With --project-ref the project_id is injected and removed from tool schemas, and account tools are dropped, but the same account-wide PAT is still attached to every call. — [packages/mcp-server-supabase/src/tools/util.ts:63-71](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/util.ts#L63-L71); [packages/mcp-server-supabase/src/server.ts:230](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/server.ts#L230) (verified)
    - *To reach the next level:* The credential itself is not narrowed; pinning is a schema rewrite, not an authorization check.
  - **C L1:** Enforcement of the project pin does not cover every path. (verified)
    - *To reach the next level:* Pin enforcement does not cover every tool path.
  - **D L2:** Project pinning is an operator CLI flag; removing it silently widens scope. — [packages/mcp-server-supabase/src/cli.ts:33-35](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/cli.ts#L33-L35) (verified)
    - *To reach the next level:* Not on by default; no warning when running unscoped.
  - **B L1:** Within the pinned project the model still has full write (destructive SQL, function deploys), and pin enforcement does not cover every path. — [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:512-515](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L512-L515) (verified)
    - *To reach the next level:* Writes are not limited to non-destructive operations.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.
- **Notes:** In --http mode the local entry takes the client's Bearer token and forwards it to the Management API (token passthrough, C1-PASSTHRU pattern); that mode is not the scored default. The hosted mcp.supabase.com OAuth flow is not in this repository.

### C2 Approval gates — 0.33 (high)

As a tool server, Supabase MCP leaves approval to the MCP host and its main contribution is risk signalling. All 36 tools carry readOnly/destructive hints, and the SQL, migration, deploy, branch and storage-config tools are marked destructive, but execute_sql mixes reads and writes in one tool and there is no dry-run. The server has its own confirmation step for destructive SQL and paid resources, bound to the exact query, but the stdio CLI never turns it on, and the confirmation logic does not cover every path. A wrongly approved call can drop or delete production data irreversibly.

- **default configuration** (default; raw 0.33 → 0.33) ← counted
  - **S L1:** Hints are present and accurate on every tool, but execute_sql mixes reads and writes and there is no dry-run in the default configuration. — [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:193-207](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L193-L207); [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:453-458](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L453-L458) (verified)
    - *To reach the next level:* No separate read-only SQL tool with accurate hints; no preview or dry-run for destructive operations.
  - **C L2:** Every tool goes out with hints, unknown tools are rejected and arguments are strictly parsed. — searched `rg -n 'destructiveHint:' --glob '!*.test.ts'` in `packages/mcp-server-supabase/src/tools` → 36 hits (one per tool definition: all 36 tools carry readOnly/destructive hints); [packages/mcp-utils/src/server.ts:510-521](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-utils/src/server.ts#L510-L521) (verified)
    - *To reach the next level:* Hints are advisory signalling only; coverage is limited one level above its strength.
  - **D L2:** Hints are hard-coded in the tool definitions and cannot be changed by the model or tool output. — [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:193-207](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L193-L207) (verified)
    - *To reach the next level:* No server-enforced gate on by default backs the hints.
  - **B L0:** execute_sql, apply_migration, delete_branch, merge_branch and deploy_edge_function act on production with no server-side undo. — [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:512-515](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L512-L515); [packages/mcp-server-supabase/src/tools/branching-tools.ts:280-289](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/branching-tools.ts#L280-L289); [packages/mcp-server-supabase/src/tools/edge-function-tools.ts:124-146](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/edge-function-tools.ts#L124-L146) (verified)
    - *To reach the next level:* No checkpoint, rollback or preview for destructive actions.
- **server-side elicitation confirmations (--http mode or elicitation option)** (alt; raw 0.23, cap G1 → 0.23)
  - **S L2:** Destructive SQL and paid resources trigger a server-issued confirmation bound by HMAC state to the exact query hash, but the prompt does not show the SQL. — [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:460-465](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L460-L465); [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:494-503](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L494-L503); [packages/mcp-server-supabase/src/tools/confirmation.ts:196-210](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/confirmation.ts#L196-L210) (verified)
    - *To reach the next level:* No preview/dry-run; approver sees a generic message, not the statement.
  - **C L1:** The destructive-SQL classifier is not a strict boundary; deploy, branch delete/merge and storage-config writes are never confirmed. — [packages/mcp-server-supabase/src/tools/branching-tools.ts:280-289](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/branching-tools.ts#L280-L289) (verified)
    - *To reach the next level:* Not all mutating paths reach the confirmation.
  - **D L0:** The stdio CLI never passes the elicitation option, and in --http mode a URL parameter skips confirmations. — [packages/mcp-server-supabase/src/cli.ts:119-129](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/cli.ts#L119-L129); [packages/mcp-server-supabase/src/server.ts:176-187](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/server.ts#L176-L187); [packages/mcp-server-supabase/src/transports/local-http-entry.ts:42-46](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/transports/local-http-entry.ts#L42-L46) (verified)
    - *To reach the next level:* Confirmation is not on in the default configuration.
  - **B L0:** Confirmed statements still run irreversibly on production. — [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:512-515](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L512-L515) (verified)
    - *To reach the next level:* No rollback or bounded quantities.
- **Cap:** none
- **Notes:** The legacy cost-confirmation flow does not cover every path.

### C3 Tool & action scoping — 0.33 (high)

Most tools are narrow and typed (strict zod schemas, region enums, ISO timestamps, injected project ids), and unknown arguments are rejected centrally. But the tools that matter most are raw passthroughs: execute_sql and apply_migration send any SQL the model writes, deploy_edge_function uploads arbitrary code, query_logs takes arbitrary log SQL, and search_docs any GraphQL. The default tool set includes all of these against every project in the account. Tool groups can be selected with --features and writes disabled with --read-only, but neither is the default.

- **S L1:** Typed strict schemas on narrow tools, but the dominant tools pass arbitrary SQL and code straight through. — [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:125-128](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L125-L128); [packages/mcp-utils/src/server.ts:510-521](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-utils/src/server.ts#L510-L521); [packages/mcp-server-supabase/src/tools/edge-function-tools.ts:48-57](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/edge-function-tools.ts#L48-L57) (verified)
  - *To reach the next level:* No allowlist validation or parameterization on the SQL and deploy tools.
- **C L2:** All tools go through the central strict-parse layer. — [packages/mcp-utils/src/server.ts:510-521](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-utils/src/server.ts#L510-L521) (verified)
  - *To reach the next level:* Central layer validates shapes only; capped one level above strength.
- **D L2:** Feature groups are selectable, but the default set includes database (read-write SQL), functions (deploy), account and branching. — [packages/mcp-server-supabase/src/server.ts:110-118](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/server.ts#L110-L118); [packages/mcp-server-supabase/src/cli.ts:36-39](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/cli.ts#L36-L39) (verified)
  - *To reach the next level:* Default tool set is not read-only.
- **B L0:** A misused execute_sql can touch any table in any project in the account, including production. — [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:512-515](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L512-L515); [packages/mcp-server-supabase/src/server.ts:46-52](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/server.ts#L46-L52) (verified)
  - *To reach the next level:* Not scoped to one project or bounded in quantity.
- **Cap:** none

### C4 Code-execution isolation — 0.47 (medium)

The code-execution surface is remote: SQL sent to the project's Postgres through the Management API, and edge-function code deployed to Supabase's runtime. Nothing runs on the machine hosting the MCP server, but by default the SQL runs read-write directly on the target (often production) database with no read-only session or restricted role. An opt-in --read-only flag asks the API to run SQL in read-only mode and blocks the write tools. Deployed functions are public endpoints with full network egress, and the model may set verify_jwt to false.

- **default configuration** (default; raw 0.05 → 0.05)
  - **S L0:** Model SQL is forwarded verbatim to the project database with read_only unset by default. — [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:512-515](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L512-L515); [packages/mcp-server-supabase/src/platform/api-platform.ts:193-206](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/platform/api-platform.ts#L193-L206) (verified)
    - *To reach the next level:* No read-only session or restricted role applied to model SQL by default.
  - **C L0:** execute_sql, apply_migration and deploy_edge_function all run unconfined. — [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:512-515](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L512-L515); [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:449](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L449); [packages/mcp-server-supabase/src/tools/edge-function-tools.ts:124-146](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/edge-function-tools.ts#L124-L146) (verified)
    - *To reach the next level:* No execution path goes through an isolation boundary by default.
  - **D L0:** Read-only execution is off unless --read-only is passed. — [packages/mcp-server-supabase/src/cli.ts:36-39](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/cli.ts#L36-L39) (verified)
    - *To reach the next level:* No isolation on by default.
  - **B L1:** SQL reaches the whole project database, and deployed functions get a public URL with full network egress (the platform also injects project keys into function environments, per Supabase docs, not visible in this repo). — [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:512-515](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L512-L515); [packages/mcp-server-supabase/src/tools/edge-function-tools.ts:42-47](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/edge-function-tools.ts#L42-L47) (verified)
    - *To reach the next level:* Network egress and project credentials remain reachable from executed code.
- **operator flag --read-only** (alt; raw 0.47, cap G1 → 0.47) ← counted
  - **S L2:** SQL is sent with read_only=true so the Management API runs it in read-only mode (enforcement is provider-side and inferred). — [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:512-515](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L512-L515); [packages/mcp-server-supabase/src/platform/api-platform.ts:193-206](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/platform/api-platform.ts#L193-L206) (inferred)
    - *To reach the next level:* Read-only mode is a database-level restriction, not a hardened sandbox; its enforcement is not visible in this repo.
  - **C L3:** Every write tool throws in read-only mode and is hidden from tools/list; execute_sql switches to read-only. — searched `rg -n 'if \(readOnly\)'` in `packages/mcp-server-supabase/src packages/mcp-utils/src` → 13 hits (12 per-tool read-only guards on every write tool plus the tool-hiding loop in server.ts); [packages/mcp-server-supabase/src/server.ts:325-331](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/server.ts#L325-L331); [packages/mcp-server-supabase/src/tools/edge-function-tools.ts:135-137](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/edge-function-tools.ts#L135-L137) (verified)
    - *To reach the next level:* Fails per tool rather than through one central gate; no fail-closed check for new tools.
  - **D L0:** Off by default. — [packages/mcp-server-supabase/src/cli.ts:36-39](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/cli.ts#L36-L39) (verified)
    - *To reach the next level:* Not on by default.
  - **B L2:** Read-only SQL still reads every table in the project, including secrets stored in the database. — [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:512-515](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L512-L515) (inferred)
    - *To reach the next level:* Sensitive data and any network-capable extensions callable in read-only mode remain reachable.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.25 (high)

Database rows, logs and notebook cells can contain attacker-written text, and the server wraps those results in a random-ID boundary with a note telling the model not to follow instructions inside it. That is spotlighting, a weak defence, and it is missing on other sources such as table and column comments, edge-function source, migrations and advisors. Results are plain text with no structured provenance, and the server's own instructions tell the model to run an npx install command. In the default configuration the same session reads untrusted rows, holds the whole account, and can run destructive SQL or deploy a public function that sends data anywhere, with no human required by the server.

- **S L1:** Untrusted results are spotlighted with a random-UUID boundary and a text directive; outputs are plain text, not structured content. — [packages/mcp-server-supabase/src/tools/util.ts:91-99](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/util.ts#L91-L99); [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:518](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L518) (verified)
  - *To reach the next level:* No structured separation of returned content from metadata, and no provenance flag the host can act on.
- **C L1:** Only execute_sql, get_logs, query_logs and get_notebook results are wrapped. — searched `rg -n 'wrapWithUntrustedDataBoundary\('` in `packages/mcp-server-supabase/src packages/mcp-utils/src` → 5 hits (definition plus 4 call sites: execute_sql, get_logs, query_logs, get_notebook; list_tables, get_edge_function, list_migrations, get_advisors, search_docs are not wrapped) (verified)
  - *To reach the next level:* Other sources (table/column comments, edge-function source, migrations, advisors, docs) are not marked.
- **D L2:** The wrapper is always applied on those tools and cannot be turned off. — [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:518](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L518) (verified)
  - *To reach the next level:* Limited to one level above strength.
- **B L0:** A hijacked session can read any project's data and exfiltrate it (deployed public function, data written to public tables) and also run irreversible SQL, unattended by default. — [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:512-515](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L512-L515); [packages/mcp-server-supabase/src/tools/edge-function-tools.ts:124-146](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/edge-function-tools.ts#L124-L146); [packages/mcp-server-supabase/src/cli.ts:119-129](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/cli.ts#L119-L129) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions do not require human approval in the default configuration.
- **Cap:** C5-WORSTCASE — Default configuration allows leaking private data and irreversible destructive SQL without any human step.
- **Notes:** The server instructions (server.ts:131) direct the host model to install `npx skills add supabase/agent-skills`; search_docs embeds a GraphQL schema fetched from supabase.com into its own tool description at runtime (first-party source). The repo's prompt-injection e2e test relies on model behaviour and is not a control.

### C6 Memory, context & configuration integrity — 1.00 (high)

The server keeps no memory and loads no workspace files: configuration comes from CLI flags and two environment variables, and nothing the model writes is read back as instructions by the server. Data written into a database can of course be read in a later session, but that is untrusted input (C5), not server memory.

- **Structural absence:** searched `rg -n -S 'dotenv|readFile|writeFile|node:fs|homedir|localStorage|AGENTS.md'` in `packages/mcp-server-supabase/src packages/mcp-utils/src` → 0 hits

### C7 Third-party extensions — 1.00 (high)

The server loads or launches no third-party code at runtime: no plugins, no child processes, no dynamic imports. Its tools call the Supabase Management API and the Supabase docs API only. Note that its instructions suggest the host model install a Supabase agent skill via npx, which is outside the server's own runtime.

- **Structural absence:** searched `rg -n -S 'child_process|spawn\(|execSync|eval\(|new Function|import\(|require\('` in `packages/mcp-server-supabase/src packages/mcp-utils/src` → 0 hits

### C8 Secrets & sensitive-data protection — 0.33 (high)

The access token comes from a CLI flag or environment variable, is attached only as a request header, and is never logged or returned to the model; errors are reduced to name and message. The API-key tool deliberately returns only client-safe anon/publishable keys. There is no telemetry. But nothing redacts sensitive data in SQL results, and the token is a long-lived, account-wide credential, so a leak (or a model reading secrets stored in the database) has a large blast radius. An opt-in URL flow keeps new edge-function secret values out of the model entirely.

- **S L1:** Token from env/flag held only in the client header; masking exists on one path (API keys filtered to client-safe ones). — [packages/mcp-server-supabase/src/cli.ts:98](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/cli.ts#L98); [packages/mcp-server-supabase/src/platform/api-platform.ts:398-402](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/platform/api-platform.ts#L398-L402) (verified)
  - *To reach the next level:* No redaction layer before model-bound tool results or type-level secret masking.
- **C L2:** Token is kept out of logs, errors and model context; error objects are reduced to name and message. — [packages/mcp-utils/src/server.ts:590-607](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-utils/src/server.ts#L590-L607); [packages/mcp-server-supabase/src/platform/api-platform.ts:398-402](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/platform/api-platform.ts#L398-L402) (verified)
  - *To reach the next level:* SQL and log results reach the model unredacted; capped one level above strength.
- **D L2:** No telemetry and no payload logging by default. — searched `rg -n -S 'sentry|posthog|opentelemetry'` in `packages/mcp-server-supabase/src packages/mcp-utils/src` → 4 hits (all hits are enum values in generated Management API types (log-drain backends), no SDK) (verified)
  - *To reach the next level:* No always-on redaction.
- **B L0:** The PAT is long-lived and grants the whole account; the model can also read any secret stored in project databases via SQL. — [packages/mcp-server-supabase/src/cli.ts:98](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/cli.ts#L98); [packages/mcp-server-supabase/src/server.ts:46-52](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/server.ts#L46-L52); [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:512-515](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L512-L515) (verified)
  - *To reach the next level:* No short-lived or scoped credential.
- **Cap:** none

### C9 Audit & traceability — 0.28 (high)

In the default stdio configuration the server records nothing about the tool calls it makes. The library offers an onToolCall callback that receives the tool name, arguments, annotations and result, but the CLI never sets it, and failures in it are swallowed. The local --http mode prints one line per request with the method, tool name and client, without arguments. There is no tamper-evident audit trail; any record of what happened lives in Supabase's own platform logs, which this repo does not control.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** No record of tool calls is written in the scored configuration. — searched `rg -n 'onToolCall'` in `packages/mcp-server-supabase/src/cli.ts packages/mcp-server-supabase/src/transports` → 0 hits (neither the stdio CLI nor the local HTTP entry supplies the tool-call callback); [packages/mcp-server-supabase/src/cli.ts:119-129](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/cli.ts#L119-L129) (verified)
    - *To reach the next level:* No structured record of each tool call.
  - **C L0:** Nothing is recorded. — searched `rg -n 'onToolCall'` in `packages/mcp-server-supabase/src/cli.ts packages/mcp-server-supabase/src/transports` → 0 hits (neither the stdio CLI nor the local HTTP entry supplies the tool-call callback) (verified)
    - *To reach the next level:* No tool path is recorded.
  - **D L0:** Recording requires an embedder to supply onToolCall. — [packages/mcp-server-supabase/src/server.ts:65-68](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/server.ts#L65-L68) (verified)
    - *To reach the next level:* Not on by default.
  - **B L0:** No record exists to survive a crash. — searched `rg -n 'onToolCall'` in `packages/mcp-server-supabase/src/cli.ts packages/mcp-server-supabase/src/transports` → 0 hits (neither the stdio CLI nor the local HTTP entry supplies the tool-call callback) (verified)
    - *To reach the next level:* No per-action record.
- **embedder-supplied onToolCall callback** (alt; raw 0.28, cap G1 → 0.28) ← counted
  - **S L1:** The callback receives name, arguments, annotations and result, but the server writes nothing itself; the embedder decides what to keep. — [packages/mcp-utils/src/server.ts:530-540](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-utils/src/server.ts#L530-L540) (verified)
    - *To reach the next level:* No server-written structured record with timestamps or actor attribution.
  - **C L2:** Every tools/call passes through the same dispatcher that invokes the callback. — [packages/mcp-utils/src/server.ts:530-540](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-utils/src/server.ts#L530-L540) (verified)
    - *To reach the next level:* No record of approvals beyond the result object; capped one level above strength.
  - **D L0:** Off unless supplied. — [packages/mcp-server-supabase/src/server.ts:65-68](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/server.ts#L65-L68) (verified)
    - *To reach the next level:* Not on by default.
  - **B L1:** Callback errors are caught and only printed; the action has already run. — [packages/mcp-utils/src/server.ts:537-540](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-utils/src/server.ts#L537-L540) (verified)
    - *To reach the next level:* Record failures are not surfaced and records are not flushed before the action.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C10 Limits & kill switch — 0.33 (high)

The server bounds very little of its own work. get_logs uses fixed queries with a 100-row limit, and the logs API caps windows at 24 hours, but execute_sql has no row or size cap, no API call has a timeout, and there are no rate limits on SQL, deploys, or creating paid projects and branches. Stopping the stdio process ends the session, but work already submitted to Supabase (migrations, branch creation, deploys) continues server-side. In --http mode a client disconnect closes the handler.

- **S L2:** Server-fixed limit of 100 rows on get_logs queries. — [packages/mcp-server-supabase/src/logs.ts:4-7](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/logs.ts#L4-L7) (verified)
  - *To reach the next level:* No caps on every operation and no concurrency or rate limits.
- **C L1:** Only the fixed log queries are bounded; SQL, deploys and API calls are not. — [packages/mcp-server-supabase/src/logs.ts:4-7](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/logs.ts#L4-L7); searched `rg -n -S 'timeout|AbortController|AbortSignal' --glob '!**/types.ts' --glob '!*.test.ts'` in `packages/mcp-server-supabase/src packages/mcp-utils/src` → 0 hits (no request timeout or cancellation signal on any Management API or Content API call (generated API type files excluded)) (verified)
  - *To reach the next level:* No timeouts on tool calls.
- **D L1:** Defaults exist only for get_logs; everything else is unlimited. — searched `rg -n -S 'timeout|AbortController|AbortSignal' --glob '!**/types.ts' --glob '!*.test.ts'` in `packages/mcp-server-supabase/src packages/mcp-utils/src` → 0 hits (no request timeout or cancellation signal on any Management API or Content API call (generated API type files excluded)) (verified)
  - *To reach the next level:* No sensible default bounds on SQL results or call duration.
- **B L1:** No ceiling on spend (paid projects/branches) or on result size; operations submitted to Supabase keep running after a stop. — [packages/mcp-server-supabase/src/tools/database-operation-tools.ts:512-515](https://github.com/supabase/mcp/blob/4602ee9ebf025611741cc127cee2a3c6a4f73854/packages/mcp-server-supabase/src/tools/database-operation-tools.ts#L512-L515) (verified)
  - *To reach the next level:* No tight per-run time or cost ceiling enforced by the server.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: SQL result rows, logs, table comments and edge-function source returned to the model (packages/mcp-server-supabase/src/tools/database-operation-tools.ts:518) · [B] sensitive data/systems: Account-wide PAT and every project's database (packages/mcp-server-supabase/src/cli.ts:98) · [C] state change / egress: Read-write execute_sql/apply_migration and deploy_edge_function (packages/mcp-server-supabase/src/tools/edge-function-tools.ts:139) · Same default session? Yes

## Highest-impact improvements
1. Make --read-only the CLI default and require an explicit --read-write flag for write tools. — C3 D L2→L3, +0.050 before caps (Playbook 3)
2. Split execute_sql into a read-only query tool (always sent with read_only=true) and a separate write tool annotated destructive. — C2 S L1→L2, +0.075 before caps (Playbook 5)
3. Write a structured record (timestamp, tool, arguments, result status) of each tool call to stderr or a file by default in the CLI. — C9 S L0→L2, +0.150 before caps (Playbook 1 step 3)
4. Return results as structuredContent with a source/untrusted flag and wrap every tool that returns user-controlled text. — C5 S L1→L2, +0.075 before caps (Playbook 1)
5. Add request timeouts and a result-size cap to execute_sql and all Management API calls. — C10 C L1→L2, +0.075 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The hosted endpoint https://mcp.supabase.com/mcp that the README leads with (OAuth login, its elicitation and read-only settings) is deployed from code outside this repository and was not scored; the score describes the npm package's CLI default.
- packages/mcp-server-postgrest is a separate MCP server in the same repository and was not scored.
- Provider-side behaviour (how the Management API enforces read_only, what credentials Supabase injects into edge functions, PAT lifetime) is inferred from the API contract, not verified in this repo.
- No reviewer-injection attempts were found in README, AGENTS.md, CLAUDE.md or source comments.
