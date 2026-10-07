# Defense-in-depth score: Metabase MCP server

**Repo:** https://github.com/metabase/metabase (`src/metabase/mcp`) · **Commit:** `23c7a5d8f662e425dd304adc30a76465ff98859f` · **Reviewed:** 2026-10-05
**What it is:** The MCP server built into Metabase, exposing search, browsing, querying, charting and content-writing tools over Streamable HTTP with Metabase's embedded OAuth server.
**Category:** Data & Analytics
**Scored configuration:** Open-source Metabase build with default settings (AI features and the MCP server on), a client connecting over OAuth with the baseline scopes it is offered first.
**Agent surface (default):** code execution opt-in · filesystem write no · network egress no · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication opt-in

## Score: 5.9 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L3 | L3 | L3 | L2 | 0.70 | none | **0.70** | High |
| C2 | Approval gates | L1 | L2 | L2 | L2 | 0.42 | none | **0.42** | High |
| C3 | Tool & action scoping | L2 | L3 | L3 | L2 | 0.62 | none | **0.62** | High |
| C4 | Code-execution isolation | L1 | L1 | L2 | L1 | 0.30 | none | **0.30** | Medium |
| C5 | Untrusted input blast radius | L2 | L2 | L3 | L2 | 0.55 | none | **0.55** | High |
| C6 | Memory, context & configuration integrity | L3 | L3 | L3 | L3 | 0.75 | none | **0.75** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L2 | L3 | L2 | L3 | 0.62 | none | **0.62** | High |
| C9 | Audit & traceability | L1 | L1 | L2 | L1 | 0.30 | none | **0.30** | High |
| C10 | Limits & kill switch | L3 | L2 | L3 | L1 | 0.57 | none | **0.57** | High |

Controls where a risk surface exists: 4.85 / 9.0 (54%); 1 criterion scored SA (surface absent).

Metabase's built-in MCP server is carefully bounded for a tool server: every call is checked against OAuth scopes and the connecting user's own Metabase permissions, and a new connection gets only read and query access until the user ticks wider permissions on a consent screen. Once those are granted, the main risks are raw SQL running with the instance's shared warehouse account, where only a read-only hint stands in the way of writes, and scheduled deliveries to any email address. In the open-source build individual tool calls are not logged, and some write tools understate their risk in the hints clients use to decide what to confirm.

## Critical gaps
- None: no criterion is capped and no blast radius is at the worst level.

## Criterion details

### C1 Identity & least privilege: 0.70 (high confidence)

Every MCP request runs as the Metabase user who authorized the client, and every tool call is checked twice: against the OAuth scopes on the token and against that user's own Metabase permissions. Access tokens last one hour and can be revoked; a fresh connection is granted only reading and query scopes, and each wider permission has to be ticked by the user on a consent screen, with a loud warning on the full-account scope. The weak spots are that refresh tokens live 30 days, a token is not bound to the resource it was requested for, and browser-cookie or API-key sessions get unrestricted scopes. Behind the user's permissions, queries reach the warehouse through the instance's single shared database connection.

- **S L3:** Authority is the requesting user's Metabase permissions intersected with per-capability OAuth scopes on a one-hour access token. Evidence: [src/metabase/mcp/v2/registry.clj:300](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/registry.clj#L300); [src/metabase/oauth_server/settings.clj:7-11](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/oauth_server/settings.clj#L7-L11); [src/metabase/oauth_server/settings.clj:29](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/oauth_server/settings.clj#L29) (verified)
  - *To reach the next level:* Credentials are not task-scoped or short-lived end to end: refresh tokens last 30 days and warehouse access uses one shared connection.
- **C L3:** Every tools/call passes the registry's scope gate and handlers check the calling user's permissions; unknown, expired, revoked or deactivated-user tokens fail closed. Evidence: [src/metabase/mcp/v2/registry.clj:300](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/registry.clj#L300); [src/metabase/mcp/v2/tools/query.clj:283-285](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/tools/query.clj#L283-L285); [src/metabase/oauth_server/core.clj:284-286](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/oauth_server/core.clj#L284-L286); [src/metabase/oauth_server/core.clj:190](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/oauth_server/core.clj#L190) (verified)
  - *To reach the next level:* Tokens are not bound to the resource they were requested for, which the source notes, so authorization is not tied to the intended audience.
- **D L3:** A new client starts with read, query and resource-read scopes only; writes, raw SQL and delivery need the user to tick each scope on the consent screen. Evidence: [src/metabase/mcp/paths.clj:64-72](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/paths.clj#L64-L72); [src/metabase/oauth_server/api/oauth.clj:151-157](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/oauth_server/api/oauth.clj#L151-L157); [src/metabase/mcp/transport.clj:744](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/transport.clj#L744) (verified)
  - *To reach the next level:* Elevation lasts as long as the token and its refresh chain rather than being time-bounded, and cookie or API-key sessions are unrestricted.
- **B L2:** A hijacked token reaches whatever its user can see in Metabase and, through the shared warehouse connection, the data behind it, with writes to Metabase content once write scopes are granted. Evidence: [src/metabase/mcp/v2/tools/query.clj:283-285](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/tools/query.clj#L283-L285); [src/metabase/oauth_server/consent_page.clj:84](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/oauth_server/consent_page.clj#L84) (verified)
  - *To reach the next level:* Blast radius is one system but includes broad read access to sensitive warehouse data and write access to shared content.
- **Cap:** none

### C2 Approval gates: 0.42 (high confidence)

As a tool server, Metabase leaves the approval prompt to the client but gives it two things to work with: risk annotations on every tool and OAuth scopes that keep writes, raw SQL and deliveries behind a consent step the user completes. The scope gate covers every tool call and rejects unknown tools. The annotations are not consistent: some tools that can trash, rewrite or overwrite content, or email results to outside addresses, are marked as non-destructive and closed-world. There are no dry-runs for destructive writes, and once a scope is granted, deliveries to any email address and raw SQL run without further confirmation.

- **S L1:** Every tool carries readOnly/destructive hints and raw SQL is honestly marked destructive, but several content and delivery tools that can archive, rewrite or send externally are marked non-destructive. Evidence: [src/metabase/mcp/v2/tools/document.clj:507](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/tools/document.clj#L507); [src/metabase/mcp/v2/tools/transform.clj:447](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/tools/transform.clj#L447); [src/metabase/mcp/v2/tools/subscription.clj:496](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/tools/subscription.clj#L496); [src/metabase/mcp/v2/tools/question.clj:556](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/tools/question.clj#L556); [src/metabase/mcp/v2/tools/query.clj:457](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/tools/query.clj#L457) (verified)
  - *To reach the next level:* Hints must be accurate on every mutating tool, with destructive operations flagged consistently.
- **C L2:** The scope gate runs centrally on every tools/call and unknown tools are rejected, but its value as a risk signal is limited by the annotations. Evidence: [src/metabase/mcp/v2/registry.clj:300](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/registry.clj#L300); [src/metabase/mcp/v2/registry.clj:290-294](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/registry.clj#L290-L294) (verified)
  - *To reach the next level:* Coverage credit is held at one level above the risk signalling it carries.
- **D L2:** A fresh connection holds only read and query scopes and consequential scopes require a consent-screen tick by the user; browser-cookie and API-key sessions skip scopes. Evidence: [src/metabase/mcp/paths.clj:64-72](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/paths.clj#L64-L72); [src/metabase/oauth_server/api/oauth.clj:151-157](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/oauth_server/api/oauth.clj#L151-L157); [src/metabase/mcp/transport.clj:744](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/transport.clj#L744) (verified)
  - *To reach the next level:* Unrestricted cookie and API-key sessions and grants that persist for the token's lifetime keep this below a time-bounded, principal-only elevation.
- **B L2:** Archived content goes to the trash and can be restored, but scheduled deliveries can email results to any address and raw SQL can change warehouse data where the connection allows. Evidence: [src/metabase/mcp/v2/recipients.clj:29-30](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/recipients.clj#L29-L30); [src/metabase/mcp/v2/tools/query.clj:457](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/tools/query.clj#L457); [src/metabase/driver/sql_jdbc/execute.clj:428-432](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/driver/sql_jdbc/execute.clj#L428-L432) (verified)
  - *To reach the next level:* No previews or dry-runs for external deliveries or SQL, and nothing bounds recipients.
- **Cap:** none

### C3 Tool & action scoping: 0.62 (high confidence)

Most tools are narrow and typed: queries are built in Metabase's structured query language, every tool's arguments are checked against a closed schema in one central registry, filter values bind as prepared-statement parameters, and result pages are capped. The exception is the raw SQL tool, which passes the model's SQL verbatim to the warehouse; it needs a separate scope, native-query permission and an instance switch that is on by default. A fresh connection gets only read and query tools; everything else needs the user's consent.

- **S L2:** Arguments are validated by closed schemas and structured queries, but the raw SQL tool forwards arbitrary SQL. Evidence: [src/metabase/mcp/v2/tools/query.clj:432](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/tools/query.clj#L432); [src/metabase/mcp/v2/tools/query.clj:436](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/tools/query.clj#L436); [src/metabase/agent_api/settings.clj:17-21](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/agent_api/settings.clj#L17-L21) (verified)
  - *To reach the next level:* The raw SQL tool has no allowlist or read-only enforcement in code, so validation is escapable through it.
- **C L3:** Registration requires an argument schema and the registry validates every call against it before dispatch, so new tools inherit validation. Evidence: [src/metabase/mcp/v2/registry.clj:58-60](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/registry.clj#L58-L60); [src/metabase/mcp/v2/registry.clj:317](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/registry.clj#L317) (verified)
  - *To reach the next level:* Coverage credit is held one level above argument-validation strength because the raw SQL path passes its main argument through.
- **D L3:** The baseline token reaches only read and query tools; write, raw SQL and delivery tools need scopes the user must grant. Evidence: [src/metabase/mcp/paths.clj:64-72](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/paths.clj#L64-L72); [src/metabase/oauth_server/api/oauth.clj:151-157](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/oauth_server/api/oauth.clj#L151-L157) (verified)
  - *To reach the next level:* No per-task tool allowlist; every tool is listed to every client whatever it holds.
- **B L2:** Misuse stays inside the user's Metabase permissions with capped result pages, but raw SQL reaches whatever the warehouse connection can touch. Evidence: [src/metabase/mcp/v2/tools/query.clj:51](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/tools/query.clj#L51); [src/metabase/mcp/v2/tools/query.clj:436](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/tools/query.clj#L436) (verified)
  - *To reach the next level:* Writes and raw SQL are not quantity-bounded or limited to non-destructive operations.
- **Cap:** none

### C4 Code-execution isolation: 0.30 (medium confidence)

The server never runs model-written code on its own host, but its raw SQL tool runs model-written SQL inside the connected warehouse with the instance's stored database account. The only containment is the database's own permissions and a read-only connection flag that Metabase's own code describes as a hint some databases ignore, and which fails silently when it cannot be set. The tool needs a separate consent scope, native-query permission and an instance setting that is on by default. Saved native questions and transforms reuse the same path, and transforms write by design.

- **S L1:** Model-written SQL runs in the warehouse under the instance's connection account, constrained only by permission checks and a read-only connection hint. Evidence: [src/metabase/mcp/v2/tools/query.clj:436](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/tools/query.clj#L436); [src/metabase/driver/sql_jdbc/execute.clj:428-432](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/driver/sql_jdbc/execute.clj#L428-L432); [src/metabase/mcp/v2/tools/query.clj:283-285](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/tools/query.clj#L283-L285) (verified)
  - *To reach the next level:* No separate low-privilege execution identity or enforced read-only boundary for agent-authored SQL.
- **C L1:** execute_sql and native question saves go through the same checks, while native transforms run on the writable transform path. Evidence: [src/metabase/mcp/v2/queries.clj:205](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/queries.clj#L205); [src/metabase/mcp/v2/tools/transform.clj:122-128](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/tools/transform.clj#L122-L128) (verified)
  - *To reach the next level:* Not every model-reachable SQL path shares even the read-only hint.
- **D L2:** The read-only hint is applied by default to query connections but a failure to set it is only logged at debug level, and the raw SQL switch defaults on. Evidence: [src/metabase/driver/sql_jdbc/execute.clj:434](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/driver/sql_jdbc/execute.clj#L434); [src/metabase/agent_api/settings.clj:17-21](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/agent_api/settings.clj#L17-L21) (verified)
  - *To reach the next level:* Fall-back to a writable connection is silent and the raw SQL capability ships enabled.
- **B L1:** If the hint does not hold, SQL runs with whatever the operator's warehouse account can do across that database. Evidence: [src/metabase/driver/sql_jdbc/execute.clj:428-432](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/driver/sql_jdbc/execute.clj#L428-L432) (inferred)
  - *To reach the next level:* Nothing in code limits the warehouse account or the reach of executed SQL.
- **Cap:** none

### C5 Untrusted input blast radius: 0.55 (high confidence)

The server returns content other people wrote: question, dashboard and document text, glossary entries, and warehouse rows. It wraps that data in randomly delimited blocks labelled as data, keeps it out of the structured output channel, and tells the model in its instructions that such text carries no authority. Those markers are aimed at the model, not machine-readable flags a client could act on. A hijacked session on a default connection can read sensitive data but has no server-side way to send it out or change anything; once the user grants write and delivery scopes, it could email results to any address or edit shared content without further confirmation.

- **S L2:** Instance and warehouse content is rendered inside random-boundary data blocks separate from the server's own prose. Evidence: [src/metabase/mcp/v2/message.clj:158-164](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/message.clj#L158-L164); [src/metabase/mcp/v2/common.clj:53](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/common.clj#L53); [src/metabase/mcp/v2/api.clj:180](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/api.clj#L180) (verified)
  - *To reach the next level:* No machine-readable provenance or untrusted flag on returned content that a host can act on.
- **C L2:** The shared rendering path wraps non-message values as data, and tool descriptions are static server text. Evidence: [src/metabase/mcp/v2/common.clj:54-55](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/common.clj#L54-L55); [src/metabase/mcp/v2/message.clj:158-164](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/message.clj#L158-L164) (verified)
  - *To reach the next level:* Not shown that every user-authored field interpolated into messages across all tools is marked as data.
- **D L3:** The boundary wrapping is built into the renderer and has no setting to turn it off. Evidence: [src/metabase/mcp/v2/message.clj:158-164](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/message.clj#L158-L164) (verified)
  - *To reach the next level:* Default credit is held one level above the strength of the wrapping.
- **B L2:** With the default baseline scopes a hijacked session can read sensitive data but has no server-side egress or write; after a step-up it can email any address or edit shared content unattended. Evidence: [src/metabase/mcp/paths.clj:64-72](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/paths.clj#L64-L72); [src/metabase/mcp/v2/recipients.clj:29-30](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/recipients.clj#L29-L30) (verified)
  - *To reach the next level:* Granted delivery and write scopes act without a per-action check, so exfiltration by email becomes possible unattended.
- **Cap:** none

### C6 Memory, context & configuration integrity: 0.75 (high confidence)

The server keeps no agent memory, conversation history or instruction files, and its instructions are a fixed string in code. The one thing a model can make it persist is a query handle: a stored query plus the user's prompt, readable only by the same user, re-validated against current permissions whenever it is used, and deleted after 24 hours by default. Metabase content written through the tools is application data that later sessions read as tool output, which is covered under untrusted input.

- **S L3:** Stored query handles are validated at use against the query schema and current permissions and expire on a schedule. Evidence: [src/metabase/mcp/v2/queries.clj:177-178](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/queries.clj#L177-L178); [src/metabase/mcp/task/mcp_query_handle_gc.clj:30](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/task/mcp_query_handle_gc.clj#L30) (verified)
  - *To reach the next level:* Handles are not versioned or integrity-protected.
- **C L3:** The handle store is the only model-influenced persistence and all of it goes through the same lookup and expiry. Evidence: [src/metabase/mcp/session.clj:439](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/session.clj#L439); [src/metabase/mcp/task/mcp_query_handle_gc.clj:30](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/task/mcp_query_handle_gc.clj#L30) (verified)
  - *To reach the next level:* Stored prompts carried with handles are not provenance-tagged.
- **D L3:** Handle lookup is scoped to the owning user in the query itself, and a retention TTL is on by default. Evidence: [src/metabase/mcp/session.clj:439](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/session.clj#L439); [src/metabase/mcp/settings.clj:59-62](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/settings.clj#L59-L62) (verified)
  - *To reach the next level:* No per-tenant storage separation.
- **B L3:** A poisoned handle affects only its owner's sessions and is purged after the TTL. Evidence: [src/metabase/mcp/settings.clj:59-62](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/settings.clj#L59-L62); [src/metabase/mcp/session.clj:439](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/session.clj#L439) (verified)
  - *To reach the next level:* Handles persist across the user's sessions for up to a day rather than being ephemeral or reviewed.
- **Cap:** none

### C7 Third-party extensions: 1.00 (high confidence)

The MCP server loads no third-party code at runtime: no plugins, no downloaded tools, no MCP servers of its own and no model files. Its tools are compiled into Metabase and registered in code.

- **Structural absence:** searched `rg -n -e 'load-plugin|classloader|require-resolve|dynamic-require|npx|pip install|trust_remote|pickle'` in `src/metabase/mcp` → 0 hits (tools register through deftool in compiled namespaces required from v2/api.clj)

### C8 Secrets & sensitive-data protection: 0.62 (high confidence)

The server keeps secrets out of what it returns: tool results are built from closed projections that never include connection details, internal errors are replaced with a generic message, and the short-lived credential the chart iframe uses travels in a host-only field and is stripped from traces. Its signing secret is masked when read through settings. At rest, that secret and other MCP settings are encrypted only if the operator sets an encryption key. Access tokens are short-lived and refresh tokens rotate, but the instance-wide signing secret cannot be rotated through settings.

- **S L2:** Secrets are masked in settings reads and credentials are redacted from traces, but at-rest encryption depends on an operator-set key. Evidence: [src/metabase/mcp/settings.clj:19](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/settings.clj#L19); [src/metabase/mcp/settings.clj:28](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/settings.clj#L28); [src/metabase/mcp/v2/registry.clj:381](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/registry.clj#L381) (verified)
  - *To reach the next level:* Encryption at rest is conditional rather than default.
- **C L3:** Model-bound results, error messages and eval traces are covered by projections, a single error sanitizer and credential redaction. Evidence: [src/metabase/mcp/v2/common.clj:328](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/common.clj#L328); [src/metabase/mcp/v2/registry.clj:381](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/registry.clj#L381) (verified)
  - *To reach the next level:* Server-side logs keep full exceptions for debugging, so not every path is redacted.
- **D L2:** The MCP module emits no telemetry and eval capture is off by default; the error sanitizer cannot be switched off. Evidence: searched `rg -n -i -e 'snowplow|track-event'` in `src/metabase/mcp` → 0 hits (no telemetry calls in the MCP module); [src/metabase/ai_tracing/core.clj:18](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/ai_tracing/core.clj#L18); [src/metabase/analytics/settings.clj:20-23](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/analytics/settings.clj#L20-L23) (verified)
  - *To reach the next level:* The host application's anonymous usage tracking defaults on, so telemetry is not opt-in instance-wide.
- **B L3:** Leaked access tokens are scoped and expire in an hour, and refresh tokens rotate on use. Evidence: [src/metabase/oauth_server/settings.clj:7-11](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/oauth_server/settings.clj#L7-L11); [src/metabase/oauth_server/core.clj:229](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/oauth_server/core.clj#L229); [src/metabase/mcp/settings.clj:10](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/settings.clj#L10) (verified)
  - *To reach the next level:* The instance-wide signing secret behind UI credentials and session keys is not rotatable through settings.
- **Cap:** none

### C9 Audit & traceability: 0.30 (high confidence)

In the open-source build the server records no row per tool call: its usage recorder is a no-op there. What does get recorded is every query it runs, attributed to the calling user and tagged as agent-originated, in Metabase's query-execution history, but those rows are batched and written asynchronously and can be lost on an unclean shutdown. Content writes go through Metabase's normal model code. The enterprise build adds a per-call row with the tool name, user, client, status and duration, but no arguments, and writes it best-effort.

- **S L1:** Only query executions are recorded, with executing user and an agent context, not every tool call. Evidence: [src/metabase/mcp/usage.clj:96-100](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/usage.clj#L96-L100); [src/metabase/mcp/usage.clj:6-7](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/usage.clj#L6-L7); [src/metabase/metabot/query_execution.clj:31-32](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/metabot/query_execution.clj#L31-L32) (verified)
  - *To reach the next level:* No structured per-call record of tool name and arguments in the default build.
- **C L1:** Query-running tools reach the recorded path; writes, deliveries and other tools have no MCP-level record. Evidence: [src/metabase/mcp/v2/tools/query.clj:127](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/tools/query.clj#L127); [src/metabase/mcp/usage.clj:96-100](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/usage.clj#L96-L100) (verified)
  - *To reach the next level:* Non-query tools and scope denials are not recorded.
- **D L2:** Query-execution recording is on by default and written by the query processor outside the model's control, in the application database. Evidence: [src/metabase/metabot/query_execution.clj:31-32](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/metabot/query_execution.clj#L31-L32) (verified)
  - *To reach the next level:* Default credit is held one level above the strength of the record.
- **B L1:** Execution rows are batched asynchronously and can be lost on a non-graceful shutdown. Evidence: [src/metabase/query_processor/middleware/process_userland_query.clj:78](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/query_processor/middleware/process_userland_query.clj#L78) (verified)
  - *To reach the next level:* Records are not written durably per action.
- **Cap:** none
- **Notes:** The enterprise build writes mcp_tool_call_log rows for every tools/call (tool, user, session, client, status, duration, error code), best-effort, without arguments.

### C10 Limits & kill switch: 0.57 (high confidence)

The server bounds its own work well: each user is throttled to 1,000 JSON-RPC messages a minute (batches count per message), can hold at most 25 event streams, result pages are capped at 2,000 rows, listings and searches have hard maximums, and warehouse statements carry a query timeout. There is no way to cancel a call already in flight. Alerts, subscriptions and transform jobs the agent creates keep running on their schedules after the session ends.

- **S L3:** Server-enforced caps on result and listing sizes plus a per-user message rate limit and stream cap. Evidence: [src/metabase/mcp/transport.clj:620](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/transport.clj#L620); [src/metabase/mcp/transport.clj:483-494](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/transport.clj#L483-L494); [src/metabase/mcp/v2/tools/query.clj:51](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/tools/query.clj#L51); [src/metabase/mcp/v2/tools/search.clj:493](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/tools/search.clj#L493); [src/metabase/mcp/v2/api.clj:146-155](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/api.clj#L146-L155) (verified)
  - *To reach the next level:* No cancellation of in-flight calls; cancellation notifications are not handled.
- **C L2:** The throttle covers every message on the surface and warehouse statements carry a timeout. Evidence: [src/metabase/mcp/transport.clj:620](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/transport.clj#L620); [src/metabase/driver/sql_jdbc/execute.clj:620](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/driver/sql_jdbc/execute.clj#L620) (verified)
  - *To reach the next level:* Scheduled work the tools create does not count against any limit.
- **D L3:** Caps are constants and schema maximums the model cannot raise. Evidence: [src/metabase/mcp/v2/tools/query.clj:51](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/tools/query.clj#L51); [src/metabase/mcp/v2/tools/search.clj:493](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/tools/search.clj#L493) (verified)
  - *To reach the next level:* The per-query timeout comes from instance configuration with no hard ceiling shown here.
- **B L1:** Stopping a client leaves in-flight queries running to their timeout and agent-created schedules active, down to hourly deliveries. Evidence: [src/metabase/mcp/v2/tools/subscription.clj:46](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/tools/subscription.clj#L46); [src/metabase/mcp/v2/api.clj:146-155](https://github.com/metabase/metabase/blob/23c7a5d8f662e425dd304adc30a76465ff98859f/src/metabase/mcp/v2/api.clj#L146-L155) (verified)
  - *To reach the next level:* Nothing scheduled by the agent stops with the session.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Content written by other users and warehouse rows returned as boundaried data (src/metabase/mcp/v2/message.clj:158-164) · [B] sensitive data/systems: Warehouse data the user can query through the instance connection (src/metabase/mcp/v2/tools/query.clj:127) · [C] state change / egress: Content writes and email deliveries to any address after a scope step-up (src/metabase/mcp/v2/recipients.clj:29-30) · Same default session? Yes

## Highest-impact improvements
1. Mark every tool that can archive, rewrite or overwrite content as destructive, and mark email-delivering tools as open-world, so clients confirm them. (C2 S L1→L2, +0.075 before caps; Playbook 5)
2. Record every tools/call in the open-source build with tool name, sanitized arguments, user and outcome, written synchronously. (C9 S L1→L2, +0.075 before caps; Playbook 1 step 3)
3. Run agent-authored SQL on a dedicated read-only warehouse connection by default and refuse when read-only mode cannot be set. (C4 S L1→L2, +0.075 before caps; Playbook 3)
4. Add a machine-readable provenance and untrusted flag to tool results so hosts can gate actions after reading user-authored content. (C5 S L2→L3, +0.075 before caps; Playbook 1)
5. Honor MCP cancellation notifications by cancelling the in-flight warehouse query. (C10 S L3→L4, +0.075 before caps; Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Sparse checkout: the MCP module plus the OAuth server, agent API, query-processor middleware, SQL-JDBC driver, sessions and settings it calls. Pulse/notification internals (e.g. recipient domain allowlists), the permission model internals and the frontend iframe were not reviewed.
- Scored the open-source build; the enterprise build adds per-call usage logging, noted under audit.
- No text aimed at AI reviewers was found in the reviewed files.
