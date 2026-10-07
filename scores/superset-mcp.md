# Defense-in-Depth Score: Apache Superset MCP Service

**Repo:** https://github.com/apache/superset (`superset/mcp_service`) · **Commit:** `7a2913e2dbe892805f67138696ee5561d67ab7c5` · **Reviewed:** 2026-10-05
**What it is:** MCP service inside Superset exposing charts, dashboards, datasets and SQL to agents
**Category:** Data & Analytics
**Scored configuration:** `superset mcp run` (streamable HTTP on 127.0.0.1:5008) with shipped mcp_config.py defaults (no transport auth, RBAC on, tool search on) and MCP_DEV_USERNAME set as the README quickstart requires (admin in the README).
**Agent surface (default):** code execution yes · filesystem write no · network egress no · external credentials yes · persistent memory no · untrusted input yes · third party extensions opt-in · sub agents no · external communication no

## Score: 5.3 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L3 | L4 | L3 | L2 | 0.78 | G1 | **0.50** (alt) | High |
| C2 | Approval gates | L1 | L2 | L1 | L3 | 0.42 | — | **0.42** | High |
| C3 | Tool & action scoping | L2 | L3 | L1 | L2 | 0.53 | — | **0.53** | High |
| C4 | Code-execution isolation | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C5 | Untrusted input blast radius | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C9 | Audit & traceability | L3 | L3 | L3 | L2 | 0.70 | — | **0.70** | High |
| C10 | Limits & kill switch | L3 | L3 | L3 | L2 | 0.70 | — | **0.70** | High |

Controls where a risk surface exists: 4.30 / 9.0 (48%); 1 criterion scored SA (surface absent).

Superset's MCP service puts every tool behind Superset's own role-based permissions, guards SQL with parsing, per-table access checks and a DML switch that is off by default, and logs every call to the Superset audit log. The main gap is the setup the README leads with: no transport authentication, with every request run as one configured user, which the quickstart sets to admin. Enable JWT or API-key auth and use a least-privilege Superset user before exposing it. By default hosts also see a single generic call_tool proxy rather than the per-tool risk annotations, which weakens host-side approval.

## Critical gaps
- When extensions are enabled (off by default), extension backend code runs in-process with everything the server holds (C7 blast radius L0). (ASI04, T17; C7) — [superset/extensions/utils.py:69](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/extensions/utils.py#L69); [superset/initialization/__init__.py:659](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/initialization/__init__.py#L659)

## Criterion details

### C1 Identity & least privilege — 0.50 (high)

Every tool call runs as a Superset user and passes Superset's own role-based permission check before it touches data; a startup check refuses to serve any tool that skips this wrapper, and permission errors deny. In the README setup, though, there is no transport authentication and every caller is mapped to one fixed user named in the config, which the quickstart sets to admin. The opt-in JWT or API-key modes resolve each request to its own Superset user, intersect token scopes with that user's roles and refuse to start when misconfigured, but they are off by default. Warehouse queries use the connection credentials Superset stores, gated by Superset's dataset and database access checks.

- **default configuration** (default; raw 0.47 → 0.47)
  - **S L2:** Authority is the role set of the single Superset user named in MCP_DEV_USERNAME, static for the run, with reads and writes sharing it. — [superset/mcp_service/auth.py:875](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/auth.py#L875); [superset/mcp_service/auth.py:480-482](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/auth.py#L480-L482) (verified)
    - *To reach the next level:* Per-request principals with scoped tokens exist only in the opt-in JWT/API-key modes.
  - **C L3:** Every tool is wrapped by mcp_auth_hook, which runs the RBAC check; startup fails if any tool other than the bug-report tool lacks the wrapper, extension tools included, and permission-check errors deny. — [superset/core/mcp/core_mcp_injection.py:211-217](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/core/mcp/core_mcp_injection.py#L211-L217); [superset/mcp_service/auth.py:1299](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/auth.py#L1299); [superset/mcp_service/app.py:973-978](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/app.py#L973-L978); [superset/mcp_service/app.py:1132](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/app.py#L1132); [superset/mcp_service/auth.py:524-526](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/auth.py#L524-L526) (verified)
    - *To reach the next level:* In the scored mode every caller maps to the same configured user, so authorization is not evaluated per requesting principal.
  - **D L0:** The shipped code has no identity and denies every call until one is configured, but the documented quickstart sets MCP_DEV_USERNAME to admin with transport auth off. — [superset/mcp_service/README.md:113-115](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/README.md#L113-L115); [superset/mcp_service/mcp_config.py:279](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/mcp_config.py#L279); [superset/mcp_service/auth.py:909](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/auth.py#L909) (verified)
    - *To reach the next level:* A least-privilege service user is not the documented default.
  - **B L2:** If authorization fails, the process can read every database connection Superset holds and write Superset metadata; per-database allow_dml (off by default) and destructive-DDL blocking still stop warehouse writes. — [superset/sql/execution/executor.py:736](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/sql/execution/executor.py#L736); [superset/models/core.py:214](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/models/core.py#L214); [superset/mcp_service/sql_lab/tool/execute_sql.py:99](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/sql_lab/tool/execute_sql.py#L99) (verified)
    - *To reach the next level:* Reach is not limited to one project or tenant.
- **opt-in JWT / API-key transport auth** (alt; raw 0.78, cap G1 → 0.50) ← counted
  - **S L3:** Each request resolves to its own Superset user from a verified JWT or API key, and token scopes are intersected with that user's RBAC. — [superset/mcp_service/auth.py:194](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/auth.py#L194); [superset/mcp_service/auth.py:719](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/auth.py#L719) (verified)
    - *To reach the next level:* Warehouse access still uses Superset's shared stored connection credentials rather than per-request downscoped ones.
  - **C L4:** All tools pass the same per-principal RBAC check, a failed JWT resolution never falls back to the dev user, and permission errors deny. — [superset/mcp_service/auth.py:719](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/auth.py#L719); [superset/mcp_service/auth.py:524-526](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/auth.py#L524-L526); [superset/mcp_service/app.py:973-978](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/app.py#L973-L978) (verified)
  - **D L3:** When enabled, startup refuses a missing audience, missing key material or a dev username alongside JWT auth. — [superset/mcp_service/mcp_config.py:587](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/mcp_config.py#L587); [superset/mcp_service/mcp_config.py:601](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/mcp_config.py#L601) (verified)
    - *To reach the next level:* Enabling it is an explicit operator setting; nothing time-bounds elevated roles.
  - **B L2:** If authorization fails, the same stored warehouse credentials and Superset metadata are reachable. — [superset/models/core.py:214](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/models/core.py#L214) (verified)
    - *To reach the next level:* Reach is not limited to one project or tenant.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C2 Approval gates — 0.42 (high)

Every one of the 78 tools carries accurate MCP risk annotations, and deletes, ownership changes, role changes and execute_sql are flagged destructive. By default, though, the server swaps the tool list for a search tool plus a generic call_tool proxy that has no annotations of its own, so a host's approval policy sees one tool for reads and deletes alike. Mistakes are mostly recoverable by default: deletes go to trash, version history is on, execute_sql has a dry-run, chart generation previews unless asked to save, and warehouse writes are refused unless an admin enables DML on that database.

- **S L1:** Underlying tools are annotated and some have previews, but the default tool-search proxy call_tool fronts every read and write without annotations. — [superset/mcp_service/server.py:696](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/server.py#L696); [superset/mcp_service/server.py:694](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/server.py#L694); [superset/mcp_service/chart/tool/delete_chart.py:70-71](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/chart/tool/delete_chart.py#L70-L71); [superset/mcp_service/sql_lab/schemas.py:91](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/sql_lab/schemas.py#L91) (verified)
  - *To reach the next level:* Risk annotations must reach the host on every advertised tool, including the call_tool proxy.
- **C L2:** All 78 registered tools, including mutating ones such as delete_chart and manage_dashboard_owners, carry ToolAnnotations. — searched `rg -n '^@tool' --type py` in `superset/mcp_service` → 78 hits (78 tool registrations, every one carrying ToolAnnotations); searched `rg -n 'readOnlyHint=' --type py` in `superset/mcp_service` → 79 hits (78 tool annotations plus one comment in mcp_config.py); [superset/mcp_service/dashboard/tool/manage_dashboard_owners.py:249-250](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/dashboard/tool/manage_dashboard_owners.py#L249-L250) (verified)
  - *To reach the next level:* Coverage does not extend through the default proxy that the host actually gates.
- **D L1:** Tool search is enabled by default, so per-tool risk signalling is hidden from hosts unless the operator turns it off. — [superset/mcp_service/mcp_config.py:527-528](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/mcp_config.py#L527-L528); [superset/mcp_service/server.py:1108](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/server.py#L1108) (verified)
  - *To reach the next level:* Per-tool annotations should be what hosts see in the default configuration.
- **B L3:** By default deletes are soft and restorable, version history is captured, execute_sql offers dry_run, charts preview unless saved, and DML is refused per database by default. — [superset/config.py:724](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/config.py#L724); [superset/config.py:758](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/config.py#L758); [superset/mcp_service/chart/schemas.py:3956](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/chart/schemas.py#L3956); [superset/models/core.py:214](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/models/core.py#L214) (verified)
  - *To reach the next level:* No rate or quantity limits on consequential calls.
- **Cap:** none

### C3 Tool & action scoping — 0.53 (high)

Tools take typed request models and most are narrow (create a chart from a structured config, add a chart to a dashboard). The general execute_sql tool parses SQL with Superset's parser, requires access to every referenced table, blocks destructive DDL and client file-transfer commands, refuses DML unless the database allows it, and caps rows and timeouts. Every tool is exposed by default, including write tools and execute_sql, limited only by the user's Superset roles and an operator deny-list.

- **S L2:** execute_sql authorizes every referenced table and parses statements to block destructive DDL and gate DML on allow_dml, inputs are bounded (page size, timeout 1-300 s, SQL_MAX_ROW), but side-effecting SQL functions are handled by a per-engine function deny-list. — [superset/mcp_service/sql_lab/tool/execute_sql.py:189-195](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/sql_lab/tool/execute_sql.py#L189-L195); [superset/mcp_service/sql_lab/tool/execute_sql.py:99](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/sql_lab/tool/execute_sql.py#L99); [superset/sql/execution/executor.py:736](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/sql/execution/executor.py#L736); [superset/sql/execution/executor.py:723](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/sql/execution/executor.py#L723); [superset/mcp_service/sql_lab/schemas.py:86](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/sql_lab/schemas.py#L86); [superset/mcp_service/constants.py:28](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/constants.py#L28); [superset/config.py:2428](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/config.py#L2428) (verified)
  - *To reach the next level:* Raw SQL is still accepted; read-only enforcement relies on parsing plus a function deny-list rather than a read-only database role.
- **C L3:** All built-in tools validate through Pydantic request models and Superset command/DAO checks; extension tools get the same wrapper but only their own validation. — searched `rg -n '^@tool' --type py` in `superset/mcp_service` → 78 hits (78 tool registrations, every one carrying ToolAnnotations); [superset/core/mcp/core_mcp_injection.py:211-217](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/core/mcp/core_mcp_injection.py#L211-L217) (verified)
  - *To reach the next level:* No central argument-policy layer that extension tools inherit automatically.
- **D L1:** All tools, including write tools and execute_sql, are enabled by default; operators can remove individual tools with MCP_DISABLED_TOOLS. — [superset/mcp_service/mcp_config.py:132](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/mcp_config.py#L132) (verified)
  - *To reach the next level:* No read-only default tool set or selectable tool groups on the standard entry point.
- **B L2:** A misused tool reaches every Superset object and database the configured user can access, with row and response-size caps. — [superset/config.py:1769](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/config.py#L1769); [superset/mcp_service/constants.py:31](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/constants.py#L31) (verified)
  - *To reach the next level:* Write reach is the whole Superset instance rather than a bounded scope.
- **Cap:** none

### C4 Code-execution isolation — 0.38 (high)

The server never runs model-written code on its own host. Model-written SQL runs inside the connected warehouses with the stored connection credentials, after Superset's parser-based checks. Jinja templating in SQL is off by default and uses Jinja's sandboxed environment when enabled. These are filters around a full SQL engine, not an isolation boundary.

- **S L1:** Model-written SQL is filtered by parsing (DDL block, DML gate, file-transfer block) and Jinja runs in a SandboxedEnvironment; there is no separate execution boundary. — [superset/mcp_service/sql_lab/tool/execute_sql.py:99](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/sql_lab/tool/execute_sql.py#L99); [superset/sql/execution/executor.py:736](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/sql/execution/executor.py#L736); [superset/jinja_context.py:940](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/jinja_context.py#L940) (verified)
  - *To reach the next level:* No isolated runtime or restricted database role is applied per call by the server.
- **C L2:** execute_sql and the SQL Lab executor share the same checks, and virtual datasets go through Superset's dataset command layer. — [superset/mcp_service/sql_lab/tool/execute_sql.py:236](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/sql_lab/tool/execute_sql.py#L236); [superset/sql/execution/executor.py:736](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/sql/execution/executor.py#L736); [superset/sql/execution/executor.py:723](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/sql/execution/executor.py#L723) (verified)
  - *To reach the next level:* Coverage relies on each SQL entry point calling the shared validator.
- **D L2:** The checks are on by default; allow_dml is a per-database admin setting and template processing is a feature flag, both operator-changeable without warning. — [superset/models/core.py:214](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/models/core.py#L214); [superset/config.py:930](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/config.py#L930) (verified)
  - *To reach the next level:* Loosening DML or templating is not flagged loudly.
- **B L1:** If the filters fail, SQL runs in production warehouses with Superset's stored connection credentials. — [superset/mcp_service/sql_lab/tool/execute_sql.py:236](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/sql_lab/tool/execute_sql.py#L236) (verified)
  - *To reach the next level:* No per-call restricted credential or network limit around executed SQL.
- **Cap:** none

### C5 Untrusted input blast radius — 0.38 (high)

The server reads content other people wrote: chart, dashboard and dataset names and descriptions, and warehouse rows. It returns structured JSON, and its instructions tell the model that tool results carry no instruction authority, but outputs do not mark which fields are user-authored (one catalog field aside). A hijacked session acting as the configured user can read any data that user can reach and can share it inside Superset by adding owners or roles to a dashboard; deletes are reversible and warehouse writes are refused by default. The server has no outbound web or mail tools.

- **S L2:** Tool outputs are typed JSON models separating content from metadata; the data-boundary text in the server instructions is a prompt, not a control. — [superset/mcp_service/app.py:101-102](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/app.py#L101-L102); [superset/mcp_service/mcp_config.py:139](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/mcp_config.py#L139) (verified)
  - *To reach the next level:* No per-field provenance or untrusted flag the host can act on.
- **C L1:** Only a catalog description field is labelled untrusted; other user-authored fields are not distinguished. — [superset/mcp_service/catalog/schemas.py:102](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/catalog/schemas.py#L102) (verified)
  - *To reach the next level:* User-authored fields across tools are not marked.
- **D L2:** Structured JSON output is always on; outputSchema/structuredContent are opt-in and the operator can change output settings silently. — [superset/mcp_service/mcp_config.py:139](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/mcp_config.py#L139) (verified)
  - *To reach the next level:* No warned, locked-on setting.
- **B L1:** Unattended, a hijacked session can read sensitive data and expose it to other Superset users through ownership or role changes; irreversible actions are blocked by default. — [superset/mcp_service/dashboard/tool/manage_dashboard_owners.py:249-250](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/dashboard/tool/manage_dashboard_owners.py#L249-L250); [superset/mcp_service/dashboard/tool/manage_dashboard_roles.py:161-162](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/dashboard/tool/manage_dashboard_roles.py#L161-L162); [superset/config.py:724](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/config.py#L724) (verified)
  - *To reach the next level:* Sharing and ownership changes are not forced through human approval.
- **Cap:** none

### C6 Memory, context & configuration integrity — 1.00 (high)

The server keeps no agent memory, vector store or conversation history that feeds back into model context, and loads no workspace instruction files. Server instructions are generated from code and operator config. Response caching is off by default. Superset objects the model creates are application data that later sessions read as tool output, which is covered under untrusted input.

- **Structural absence:** searched `rg -n -i 'load_dotenv|vector_store|save_memory|remember\('` in `superset/mcp_service` → 0 hits; [superset/mcp_service/mcp_config.py:373](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/mcp_config.py#L373)

### C7 Third-party extensions — 0.25 (high)

Superset extensions, which can register their own MCP tools, are off by default. When an operator turns them on, extension bundles from configured local paths are loaded and executed inside the Superset process with no signature or hash check; only a deny-list and minimum-version policy exist. A malicious extension would run with everything the server holds, including database credentials.

- **S L1:** Extensions come from operator-configured paths with no signature or hash verification; a deny-list and minimum-version policy are the only checks. — [superset/config.py:3484-3485](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/config.py#L3484-L3485); [superset/config.py:3490-3496](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/config.py#L3490-L3496) (verified)
  - *To reach the next level:* Versions are not pinned or integrity-checked.
- **C L1:** The one extension type (Superset extension bundles) gets only deny-list and version-floor checks. — [superset/config.py:3490-3496](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/config.py#L3490-L3496) (verified)
  - *To reach the next level:* No integrity verification for any extension type.
- **D L2:** Extensions are disabled by default and can only be enabled through operator config. — [superset/config.py:733](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/config.py#L733); [superset/initialization/__init__.py:629](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/initialization/__init__.py#L629) (verified)
  - *To reach the next level:* Enabling does not show what will run; capped one level above strength.
- **B L0:** Extension backend code is exec'd in-process with the server's full credentials. — [superset/extensions/utils.py:69](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/extensions/utils.py#L69); [superset/initialization/__init__.py:659](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/initialization/__init__.py#L659) (verified)
  - *To reach the next level:* Extensions are not run in a separate process or sandbox.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.45 (high)

Database credentials stay inside Superset and are not returned by the MCP tools. Audit logging masks parameters with sensitive key names, error text is scrubbed of connection strings, tokens and emails before logging, and JWT key material is never logged. Masking is pattern- and key-name based, some tool progress notifications forward raw exception text to the client, and the optional error hook receives raw exceptions. Nothing is sent to third-party telemetry by default.

- **S L2:** Key-name masking for logged parameters and regex scrubbing for error text. — [superset/mcp_service/middleware.py:481-490](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/middleware.py#L481-L490); [superset/mcp_service/middleware.py:135](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/middleware.py#L135) (verified)
  - *To reach the next level:* No masking on every model-bound path or output scanning.
- **C L2:** Logs and error responses are scrubbed; some ctx.error notifications to the client carry raw exception strings. — [superset/mcp_service/middleware.py:135](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/middleware.py#L135); [superset/mcp_service/sql_lab/tool/execute_sql.py:282-286](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/sql_lab/tool/execute_sql.py#L282-L286) (verified)
  - *To reach the next level:* Not every client-bound error path is sanitized.
- **D L2:** No third-party telemetry by default; the error-capture hook is opt-in and receives raw exceptions. — [superset/mcp_service/mcp_config.py:167](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/mcp_config.py#L167) (verified)
  - *To reach the next level:* Masking is not enforced on the opt-in hook.
- **B L1:** The process holds long-lived warehouse connection credentials and Superset's metadata database access, though no tool exposes them to the model. — [superset/mcp_service/sql_lab/tool/execute_sql.py:236](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/sql_lab/tool/execute_sql.py#L236) (verified)
  - *To reach the next level:* Credentials are not short-lived or per-request.
- **Cap:** none

### C9 Audit & traceability — 0.70 (high)

A middleware layer writes a structured record of every tool call to Superset's event log (the logs table behind the Action Log UI): tool name, resolved user, agent id, a per-call id, sanitized arguments, success and duration. This covers extension tools and calls made through the call_tool proxy, and permission denials are recorded as failed calls. The record is written after the call finishes and a logging failure only produces a warning, so the action is not blocked.

- **S L3:** Each record carries user_id, agent_id, a random mcp_call_id, sanitized params, success and duration. — [superset/mcp_service/middleware.py:763](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/middleware.py#L763); [superset/mcp_service/middleware.py:710](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/middleware.py#L710); [superset/mcp_service/middleware.py:879](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/middleware.py#L879) (verified)
  - *To reach the next level:* Records are not tamper-evident or hash-chained.
- **C L3:** LoggingMiddleware wraps every tool call, including proxied and extension tools, and denials are logged as failures. — [superset/mcp_service/middleware.py:521](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/middleware.py#L521); [superset/mcp_service/middleware.py:763](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/middleware.py#L763) (verified)
  - *To reach the next level:* Configuration changes and credential use are not recorded.
- **D L3:** On by default through Superset's DBEventLogger, written by middleware the model cannot influence. — [superset/config.py:95](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/config.py#L95); [superset/mcp_service/middleware.py:763](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/middleware.py#L763) (verified)
  - *To reach the next level:* Disabling via EVENT_LOGGER is not itself logged.
- **B L2:** Records are written per call after execution; log failures are caught and warned. — [superset/mcp_service/middleware.py:951](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/middleware.py#L951) (verified)
  - *To reach the next level:* Consequential calls are not blocked when the record cannot be written.
- **Cap:** none

### C10 Limits & kill switch — 0.70 (high)

Every protected tool call runs in a bounded worker pool with a deadline (30 seconds by default, at most 300 for SQL) and returns a busy error rather than queueing when the pool is full. Responses are capped at about 50 KB, page sizes at 100 and rows at 100,000. On timeout the server asks the warehouse to cancel the query, but the worker keeps its slot until the database call returns. There is no per-client rate limit on the MCP transport.

- **S L3:** Deadlines on every protected call, concurrency admission limits, response-size, page and row caps. — [superset/mcp_service/worker.py:740](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/worker.py#L740); [superset/mcp_service/worker.py:69](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/worker.py#L69); [superset/mcp_service/constants.py:31](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/constants.py#L31); [superset/mcp_service/auth.py:1396](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/auth.py#L1396) (verified)
  - *To reach the next level:* Cancellation is best-effort; in-flight work is not forcibly stopped.
- **C L3:** All protected tools go through run_in_worker, and nested tool calls share the outer deadline and slot. — [superset/mcp_service/worker.py:662](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/worker.py#L662); [superset/mcp_service/worker.py:740](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/worker.py#L740) (verified)
  - *To reach the next level:* No cap on how many calls a single client can issue over time.
- **D L3:** Sensible defaults; the caller-set SQL timeout is bounded at 300 s by schema and worker counts are clamped to the pool budget. — [superset/mcp_service/sql_lab/schemas.py:86](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/sql_lab/schemas.py#L86); [superset/mcp_service/mcp_config.py:84](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/mcp_config.py#L84); [superset/config.py:2096](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/config.py#L2096) (verified)
  - *To reach the next level:* SQLLAB_TIMEOUT and response limits have no hard ceiling.
- **B L2:** Per-call ceilings are tight, but timed-out work continues until the database call returns and there is no transport rate limit. — [superset/mcp_service/worker.py:740](https://github.com/apache/superset/blob/7a2913e2dbe892805f67138696ee5561d67ab7c5/superset/mcp_service/worker.py#L740) (verified)
  - *To reach the next level:* Stopping does not end in-flight work.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: User-authored chart, dashboard and dataset text and warehouse rows returned by tools (superset/mcp_service/catalog/schemas.py:102) · [B] sensitive data/systems: All data the configured Superset user can query (superset/mcp_service/sql_lab/tool/execute_sql.py:236) · [C] state change / egress: Chart/dashboard/dataset writes and ownership/role changes (superset/mcp_service/dashboard/tool/manage_dashboard_owners.py:249) · Same default session? Yes

## Highest-impact improvements
1. Advertise per-tool annotations by default (disable tool search, or give call_tool a destructive annotation and surface the target tool's hints) so host approval policies can distinguish reads from writes. — C2 D L1→L2, +0.050 before caps (Playbook 5)
2. Make the quickstart use a dedicated least-privilege Superset user (or API key) instead of admin, and warn at startup when MCP_DEV_USERNAME resolves to an Admin role. — C1 D L0→L2, +0.100 before caps (Playbook 4)
3. Ship a read-only tool profile (exclude 'mutate'-tagged tools) on the standard `superset mcp run` path and make it the default. — C3 D L1→L3, +0.100 before caps (Playbook 3)
4. Mark user-authored fields (names, descriptions, query rows) as untrusted in every output schema. — C5 C L1→L2, +0.075 before caps (Playbook 1)
5. Route all client-bound error notifications through the existing error sanitizer. — C8 C L2→L3, +0.075 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scope is superset/mcp_service plus the Superset code it relies on for auth, permissions and SQL execution (security manager entry points, SQL executor, config defaults, extension loader); the wider Superset web app and frontend were not reviewed.
- HTTP-transport Origin/Host validation and DNS-rebinding behaviour depend on the FastMCP library and were not examined.
- Screenshot/preview rendering through the headless browser and chart plugins rendering in the frontend were not examined in depth.
- The repository contains AGENTS.md/CLAUDE.md files addressed to AI tools; they were treated as data and contained no attempt to steer reviewers' ratings.
