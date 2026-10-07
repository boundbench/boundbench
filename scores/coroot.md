# Defense-in-depth score: Coroot (MCP server and AI RCA)

**Repo:** https://github.com/coroot/coroot · **Commit:** `cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855` · **Reviewed:** 2026-10-05
**What it is:** Open-source observability platform; scored here for its built-in MCP server for AI agents and its AI root-cause analysis feature.
**Category:** Infrastructure & Ops
**Scored configuration:** Community Edition from the shipped docker-compose with the /mcp endpoint served by default (OAuth or API-key bearer auth) and AI root-cause analysis off until a Coroot Cloud API key is configured.
**Agent surface (default):** code execution no · filesystem write no · network egress opt-in · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions no · sub agents no · external communication yes

## Score: 6.7 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L3 | L3 | L2 | L2 | 0.65 | none | **0.65** | High |
| C2 | Approval gates | L2 | L3 | L2 | L2 | 0.57 | none | **0.57** | High |
| C3 | Tool & action scoping | L2 | L3 | L2 | L2 | 0.57 | none | **0.57** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L2 | L3 | L3 | L2 | 0.62 | none | **0.62** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L2 | 0.30 | none | **0.30** | High |
| C9 | Audit & traceability | L1 | L1 | L2 | L1 | 0.30 | none | **0.30** | High |
| C10 | Limits & kill switch | L2 | L3 | L3 | L2 | 0.62 | none | **0.62** | High |

Controls where a risk surface exists: 3.65 / 7.0 (52%); 3 criteria scored SA (surface absent).

Coroot's MCP server gives AI agents read access to production logs, traces and metrics, checked on every call against the connecting user's Coroot role, plus one write: resolving alerts. The tools are narrow, carry accurate risk hints, cap their output size and run no code. The main weaknesses are what the data contains and what isn't recorded: logs and traces reach the agent's model unredacted and unmarked as untrusted, and apart from alert resolutions nothing records what an agent read. AI root-cause analysis is off by default; once enabled it sends telemetry to Coroot Cloud and posts model-written analyses to notification channels without review.

## Critical gaps
- Apart from alert resolutions, MCP tool calls leave no record: what an agent read, with which arguments and through which client, cannot be reconstructed. (T8, ASI10; C9). Evidence: [api/mcp.go:137-143](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L137-L143)
- Log bodies and trace data are returned to the agent's model with no redaction of secrets or personal data. (LLM02, ASI03; C8). Evidence: [api/mcp.go:1651](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1651)

## Criterion details

### C1 Identity & least privilege: 0.65 (high confidence)

Every MCP tool call runs as the Coroot user or service account that authenticated, and each tool checks that user's role in code before Coroot queries Prometheus or ClickHouse with its own server-held credentials. That is a real per-request authorization gate, and it fails closed if roles can't be loaded. The weak points are reach and defaults: the first account is an Admin and an MCP grant carries the user's whole role, refresh grants last 30 days, and authorization is not applied the same way on every AI feature request path. If the gate fails, the server's credentials can read telemetry for every project.

- **S L3:** Each tool resolves the authenticated user and checks project and per-object RBAC permissions in code before the server attaches its own backend credentials. Evidence: [api/mcp.go:442-458](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L442-L458); [api/mcp.go:1685](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1685); [api/mcp.go:1212](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1212) (verified)
  - *To reach the next level:* The grant itself is not narrowed: an MCP token carries the user's full role for 30 days rather than a short-lived, task-scoped credential.
- **C L3:** All MCP tools pass through the same user-scoped RBAC check, which fails closed when role resolution errors. Evidence: [api/auth.go:291-296](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/auth.go#L291-L296); [api/mcp.go:1100](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1100) (verified)
  - *To reach the next level:* Authorization against the requesting user is not applied uniformly on every request path.
- **D L2:** The default first account is an Admin, so an MCP client authorized by it gets the Admin role's tools; read-only use requires the operator to connect a Viewer user or service account. Evidence: [db/user.go:86](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/db/user.go#L86); [api/mcp_oauth.go:24-25](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp_oauth.go#L24-L25) (verified)
  - *To reach the next level:* No read-only default for MCP grants; write tools are available whenever the connecting user's role allows them, with no separate elevation.
- **B L2:** If authorization fails, the server-held Prometheus and ClickHouse credentials read logs, traces and metrics for every project, while writes through MCP are limited to alert state. Evidence: [api/mcp.go:1212](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1212); [rbac/role.go:30](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/rbac/role.go#L30) (verified)
  - *To reach the next level:* Backend credentials are not scoped per project or per user, so a failure exposes all projects' telemetry rather than one tenant.
- **Cap:** none

### C2 Approval gates: 0.57 (high confidence)

As a tool server, Coroot leaves approval to the connecting agent, so what matters is what it tells that agent. All 17 MCP tools carry read-only and destructive hints, reads and writes are separate tools, and the only state-changing tool, resolving alerts, is checked against the user's edit permission on the server. There is no preview or dry-run for resolving alerts, and no operator switch that makes the server read-only apart from connecting a Viewer account. A wrongly approved call can silence alerts and send resolution notices to Slack or PagerDuty; alerts whose conditions persist fire again.

- **S L2:** Every tool declares readOnly/destructive/openWorld hints and reads and writes are separate tools; resolve_alerts is marked non-read-only and open-world. Evidence: [api/mcp.go:210-223](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L210-L223); searched `rg -n -S 'dry.?run|preview|confirm'` in `api/mcp.go` → 1 hits (the single hit is the resolve_alerts description text asking the model to confirm the issue is fixed; no preview or confirmation mechanism exists) (verified)
  - *To reach the next level:* No preview or dry-run for the state-changing tool and no server-enforced confirmation step.
- **C L3:** All 17 registered tools carry annotations and the one write tool is additionally checked server side against the alerts edit permission. Evidence: searched `rg -n 'WithReadOnlyHintAnnotation'` in `api/mcp.go` → 17 hits (one per registered tool; there are 17 mcp.NewTool registrations); [api/mcp.go:1100](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1100) (verified)
  - *To reach the next level:* No server-side policy that rejects or holds write calls independent of the user's role.
- **D L2:** Annotations are always present, but whether the write tool is usable depends only on the connecting user's role, and the default first account is an Admin. Evidence: [api/mcp.go:218](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L218); [db/user.go:86](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/db/user.go#L86) (verified)
  - *To reach the next level:* No operator setting to run the MCP server in read-only mode regardless of role.
- **B L2:** A wrongly approved call resolves alerts (reversible, and alerts whose condition holds re-fire) but also sends resolution notifications to configured channels, which cannot be recalled. Evidence: [api/api.go:1440-1470](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/api.go#L1440-L1470); [api/mcp.go:1103-1109](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1103-L1109) (verified)
  - *To reach the next level:* No bound on how many alerts one call can resolve and no rate limit on the write tool.
- **Cap:** none

### C3 Tool & action scoping: 0.57 (high confidence)

Most MCP tools are narrow, purpose-built reads (list alerts, get a trace, node details) with typed arguments, clamped limits, and log searches passed to ClickHouse as bound parameters. The exception is query_metrics, which forwards an arbitrary PromQL expression to the project's metrics backend, bounded only by a 24-hour range and a series limit. The default tool set includes alert resolution for any Editor or Admin and there is no way to switch tool groups off. Misuse stays within read-only telemetry queries and alert state for the selected project.

- **S L2:** Typed arguments with clamped numeric bounds and parameterized log filters, but query_metrics passes a raw PromQL expression straight to the metrics backend. Evidence: [api/mcp.go:1207-1210](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1207-L1210); [api/mcp.go:1235](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1235); [clickhouse/logs.go:457-458](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/clickhouse/logs.go#L457-L458) (verified)
  - *To reach the next level:* The general query tool is not replaced or constrained by an allowlist of metrics or label matchers.
- **C L3:** Every built-in tool validates and clamps its own arguments; the server loads no extension tools. Evidence: [api/mcp.go:1577-1584](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1577-L1584); [api/mcp.go:1219-1222](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1219-L1222) (verified)
  - *To reach the next level:* Validation is per tool rather than one central policy layer that new tools inherit.
- **D L2:** All 17 tools are always registered; the only write tool is usable by any Editor or Admin, including the default first account. Evidence: [api/mcp.go:145-155](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L145-L155); [rbac/role.go:27](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/rbac/role.go#L27) (verified)
  - *To reach the next level:* No operator setting to expose only the read-only tool set.
- **B L2:** A misused tool reaches all telemetry of the selected project, with per-call row and size bounds on reads, and can resolve an unbounded list of alerts. Evidence: [api/mcp.go:1159-1165](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1159-L1165); [api/mcp.go:1103-1109](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1103-L1109) (verified)
  - *To reach the next level:* The write tool has no quantity bound, so scoped-and-bounded is not met for every tool.
- **Cap:** none

### C4 Code-execution isolation: 1.00 (high confidence)

Nothing in Coroot's AI surface runs model-supplied text as code: the MCP tools and the AI root-cause path contain no shell, interpreter, template or plugin execution. The closest thing is query_metrics, which sends a PromQL expression to the metrics backend as a read-only query; that is scored as tool scoping in C3.

- **Structural absence:** searched `rg -n -S 'os/exec|exec\.Command|syscall\.Exec|text/template|html/template|goja|otto|plugin\.Open'` in `api/mcp.go api/mcp_oauth.go api/rca.go cloud` → 0 hits (MCP server, MCP OAuth, AI RCA client and Coroot Cloud client)

### C5 Untrusted input blast radius: 0.62 (high confidence)

Coroot's tools return application logs, trace attributes and Kubernetes events, all of which can contain text written by outsiders, to the connecting agent. Results are structured JSON with the originating service or trace attached, and the server instructions contain only tool-selection guidance, but nothing marks returned content as untrusted and there is no mode that drops the alert-resolution write. A hijacked agent can read every project's telemetry its user can see and resolve alerts without the server asking anyone. When AI root-cause analysis is enabled, the same telemetry is also sent to Coroot Cloud's model and its output is posted to notification channels without review.

- **S L2:** Tool results are structured JSON that separate log bodies and span data from metadata such as service, severity and trace id. Evidence: [api/mcp.go:1651](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1651); [api/mcp.go:1770-1780](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1770-L1780) (verified)
  - *To reach the next level:* No untrusted flag on returned content the host could act on, and no server mode that drops the write or egress leg.
- **C L3:** Every tool returns structured JSON through the same encoder, and the server instructions are a fixed constant with no directives drawn from data. Evidence: [api/mcp.go:28](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L28); [api/mcp.go:1653](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1653) (verified)
  - *To reach the next level:* Third-party text inside telemetry is never distinguished from Coroot's own fields.
- **D L3:** The structured output format is fixed in code and nothing in a request or in telemetry can change it. Evidence: [api/mcp.go:1770-1780](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1770-L1780) (verified)
  - *To reach the next level:* Capped one level above strength; a stronger mechanism (provenance flags, read-only mode) would be needed first.
- **B L2:** A hijacked agent can read all telemetry its user may view, which often holds personal data or secrets, and resolve alerts unattended; the server itself offers no outbound channel, and alert resolution is reversible though it sends notices. Evidence: [api/mcp.go:1115](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1115); [api/rca.go:149](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/rca.go#L149); [notifications/notifications.go:100](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/notifications/notifications.go#L100) (verified)
  - *To reach the next level:* Sensitive telemetry and an unattended state change are both available in the same session; no server mode separates them.
- **Cap:** none

### C6 Memory, context & configuration integrity: 1.00 (high confidence)

In the scored default, nothing an MCP client or model produces persists into later behaviour: the server's instructions are a compiled-in constant, session state is only the selected project held in memory, and the only write records a fixed 'via MCP' attribution on resolved alerts. No workspace files or .env are auto-loaded. Enabling AI root-cause analysis changes this: model-written analyses are then stored on each incident and returned to every later MCP client and notification channel for that project, with no review or purge step.

- **Structural absence:** searched `rg -n -S 'db\.(Update|Set|Save|Add|Insert|Create|Delete)|resolveAlerts\('` in `api/mcp.go` → 1 hits (the only write from MCP tools is alert resolution with a fixed resolvedBy attribution; no model-written text is stored); searched `rg -n -S 'AGENTS\.md|CLAUDE\.md|godotenv|load_dotenv|\.mcp\.json'` in `.` → 0 hits

### C7 Third-party extensions: 1.00 (high confidence)

The AI surface loads no third-party code: the MCP server is compiled into Coroot, launches no MCP servers or plugins of its own, and the AI root-cause feature calls only Coroot's own cloud service over HTTPS.

- **Structural absence:** searched `rg -n -S 'plugin\.Open|exec\.Command|go-plugin|npx|stdio|NewStdioMCPClient'` in `api/mcp.go api/mcp_oauth.go api/rca.go cloud` → 0 hits

### C8 Secrets & sensitive-data protection: 0.30 (high confidence)

Coroot stores service-account API keys only as SHA-256 hashes and keeps its Prometheus and ClickHouse credentials on the server, never in tool results. But nothing redacts what the tools return: log bodies, trace attributes and events go to the agent's model as collected, apart from length truncation, and with AI root-cause analysis enabled, traces and Kubernetes events are sent to Coroot Cloud the same way. The Coroot Cloud API key is stored in plain text in the database. Anonymous usage statistics, including per-tool MCP call counts, are sent to coroot.com by default unless disabled.

- **S L1:** API keys are hashed at rest, but no redaction is applied to telemetry returned to MCP clients or sent for AI analysis. Evidence: [db/user.go:271-274](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/db/user.go#L271-L274); searched `rg -n -S 'redact|mask|scrub|sanitiz|obfuscat'` in `api/mcp.go api/rca.go cloud` → 0 hits (no redaction anywhere in the MCP or RCA paths); [cloud/api.go:70-72](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/cloud/api.go#L70-L72) (verified)
  - *To reach the next level:* No redaction before model-bound tool results or the RCA payload, and the cloud API key is stored unencrypted.
- **C L1:** Only credential storage is protected; model-bound tool results, the RCA payload and notification text are not. Evidence: [api/mcp.go:1651](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1651); [api/rca.go:134](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/rca.go#L134) (verified)
  - *To reach the next level:* Model-bound messages and outbound RCA payloads carry no protection.
- **D L1:** Usage statistics, including MCP tool call counts but not content, are sent to coroot.com unless the operator disables them. Evidence: [stats/stats.go:30](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/stats/stats.go#L30); [config/flags.go:30](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/config/flags.go#L30); [api/mcp.go:140](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L140) (verified)
  - *To reach the next level:* Telemetry is on by default rather than opt-in.
- **B L2:** An exposed MCP grant is role-scoped with a 1-hour access token and a 30-day refresh token; service-account keys are long-lived until an operator deletes them. Evidence: [api/mcp_oauth.go:24-25](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp_oauth.go#L24-L25) (verified)
  - *To reach the next level:* Refresh grants and API keys are long-lived rather than short-lived and automatically rotated.
- **Cap:** none

### C9 Audit & traceability: 0.30 (high confidence)

Coroot keeps no record of what an agent did through MCP beyond one case: resolving alerts stores the user's name with a 'via MCP' tag on the alert. Read tools that pull logs, traces and metrics leave no trace except an anonymous per-tool counter in usage statistics, and the server logs only errors. After an incident you could see which alerts an agent resolved and on whose account, but not what data it read or which OAuth client acted.

- **S L1:** Only alert resolution leaves an attributed record (user name plus 'via MCP'); other tool calls are not recorded. Evidence: [api/mcp.go:1110-1115](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1110-L1115); [api/mcp.go:137-143](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L137-L143) (verified)
  - *To reach the next level:* No structured record of every tool call with arguments, result status and timestamps.
- **C L1:** The write path is recorded; read tools, grant issuance and AI analysis requests are not. Evidence: searched `rg -n 'klog\.(Info|Infof|Infoln|V\()'` in `api/mcp.go api/mcp_oauth.go` → 0 hits (MCP code logs only errors via klog.Errorln) (verified)
  - *To reach the next level:* Read tool calls and OAuth grants are not recorded at all.
- **D L2:** The alert attribution is written to Coroot's database by server code that the agent cannot edit. Evidence: [api/api.go:1451](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/api.go#L1451) (verified)
  - *To reach the next level:* Capped one level above strength; a fuller record would be needed first.
- **B L1:** The attribution is written together with the alert change, but nothing else is durable and a trajectory cannot be replayed. Evidence: [api/api.go:1451](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/api.go#L1451) (verified)
  - *To reach the next level:* No per-action durable record for most tools.
- **Cap:** none

### C10 Limits & kill switch: 0.62 (high confidence)

Coroot bounds the work of each MCP call: list results are cut at about 50 KB and any response at 80 KB, row limits are clamped to fixed maximums, metric queries are limited to 24 hours, and backend queries follow the request's context so they stop when the client goes away. There are no rate or concurrency limits, so a runaway agent can keep issuing expensive queries against Prometheus and ClickHouse. AI root-cause analysis stops when Coroot Cloud credits run out, a ceiling enforced by the provider, but background incident investigations run without a timeout of their own.

- **S L2:** Server-enforced caps on output size, row counts and metric time range, compiled in as constants. Evidence: [api/mcp.go:1765-1768](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1765-L1768); [api/mcp.go:1165](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1165); searched `rg -n -i 'ratelimit|rate\.Limit|throttle|semaphore'` in `api/mcp.go api/mcp_oauth.go` → 0 hits (verified)
  - *To reach the next level:* No rate or concurrency limits on tool calls, and log and trace queries have no time-range cap.
- **C L3:** Every tool's output goes through the same size budget and every list tool clamps its row limit. Evidence: [api/mcp.go:1791-1808](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1791-L1808); [watchers/incidents.go:122](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/watchers/incidents.go#L122) (verified)
  - *To reach the next level:* Background AI investigations are not bounded by a timeout like interactive requests.
- **D L3:** Limits are constants a caller cannot raise; out-of-range requests fall back to defaults. Evidence: [api/mcp.go:1207-1210](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/api/mcp.go#L1207-L1210) (verified)
  - *To reach the next level:* Capped one level above strength.
- **B L2:** Each call is bounded, AI analyses stop at the provider-side credit limit, but a client can loop indefinitely and the cloud call has no client timeout. Evidence: [cloud/rca.go:69](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/cloud/rca.go#L69); [cloud/api.go:96](https://github.com/coroot/coroot/blob/cc7c1bf75be9c3d08c751d3d4f9942cf8c81e855/cloud/api.go#L96) (verified)
  - *To reach the next level:* No session-level ceiling on MCP calls and no timeout on background RCA requests.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Application log bodies, span attributes and Kubernetes events returned by query_logs / get_trace (api/mcp.go:1651) · [B] sensitive data/systems: Production telemetry of every project the user can view, read with server-held backend credentials (api/mcp.go:1212) · [C] state change / egress: resolve_alerts changes alert state and sends notifications (api/mcp.go:1115); RCA payload to Coroot Cloud when enabled (api/rca.go:157) · Same default session? Yes

## Highest-impact improvements
1. Write an audit record for every MCP tool call (user, OAuth client, tool, arguments, result status, timestamp) and for each grant issued. (C9 S L1→L2, +0.075 before caps; Playbook 1 step 3)
2. Redact common secret and personal-data patterns from log bodies, span attributes and events before returning them to MCP clients or sending them for AI analysis. (C8 S L1→L2, +0.075 before caps; Playbook 4)
3. Add an operator setting (or a read-only OAuth scope) that exposes only the read-only tool set over MCP regardless of the user's role. (C3 D L2→L3, +0.050 before caps; Playbook 3)
4. Add per-user rate and concurrency limits on MCP tool calls and a time-range cap on log and trace queries. (C10 S L2→L3, +0.075 before caps; Playbook 3 step 3)
5. Mark third-party text in tool results (log bodies, span attributes, events) with an untrusted flag the host can act on. (C5 S L2→L3, +0.075 before caps; Playbook 1)

## Re-audit log
- C2 S: L3 → L2. First rated L3 because no tool is destructive, but resolve_alerts changes state and sends notifications with no preview or dry-run, so the L3 element is not met.
- C3 B: L3 → L2. Reads are row- and size-bounded, but resolve_alerts accepts an unbounded id list, so 'scoped and quantity-bounded' does not hold for every tool.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scope is Coroot's AI-agent surface only: the built-in MCP server and its OAuth/API-key authentication (api/mcp.go, api/mcp_oauth.go) and the AI root-cause analysis client (api/rca.go, cloud/, its incident auto-investigation and notification text). The rest of the observability platform (UI authentication and first-run setup, collectors, node and cluster agents, alerting engine, dashboards) was not scored.
- The AI root-cause model runs in Coroot Cloud, outside this repository; how it handles the telemetry it receives and what it returns was not examined. The front end was read only where it displays AI analyses.
- Enterprise Edition features (custom roles, EE-only MCP tools) are not in this repository and were not reviewed; with the Community Edition's built-in roles every user can view every project.
- The scored default has AI root-cause analysis off; C6 is structurally absent only in that configuration, and enabling it adds a persistent, project-wide store of model-written analyses.
- Behaviour of the mark3labs/mcp-go library (session handling, transport) was assumed from its use here, not audited.
- No release tag points at the pinned commit.
