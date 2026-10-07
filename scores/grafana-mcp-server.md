# Defense-in-Depth Score: Grafana MCP Server

**Repo:** https://github.com/grafana/mcp-grafana · **Commit:** `9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f` · **Reviewed:** 2026-10-03
**What it is:** MCP server for Grafana dashboards, datasources (Prometheus/Loki), alerting, OnCall, incidents
**Category:** Infrastructure & Ops
**Scored configuration:** stdio transport (the README Quick Start) with default flags: default tool categories enabled, write tools on, a single GRAFANA_SERVICE_ACCOUNT_TOKEN from the environment.
**Agent surface (default):** code execution yes · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions yes · sub agents no · external communication yes

## Score: 3.3 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L2 | L1 | L1 | 0.33 | C1-SELFESC | **0.25** | High |
| C2 | Approval gates | L1 | L2 | L2 | L0 | 0.33 | — | **0.33** | High |
| C3 | Tool & action scoping | L2 | L3 | L2 | L1 | 0.53 | — | **0.53** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | Medium |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |
| C9 | Audit & traceability | L2 | L2 | L0 | L1 | 0.35 | G1 | **0.35** (alt) | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |

Controls where a risk surface exists: 2.30 / 9.0 (26%); 1 criterion scored SA (surface absent).

Grafana's MCP server is carefully engineered around its credentials: the token never reaches the model, redirects cannot carry it elsewhere, and every tool is annotated so hosts can tell reads from writes. But it ships with write tools on, including a general-purpose Grafana API tool that takes any method and path and can run raw SQL through datasources, plus a plugin installer the model can drive. Nothing in the server marks untrusted log or dashboard content or records what tools did, so a prompt-injected session can read sensitive telemetry, send it out through a new contact point, and delete resources. Run it with --disable-write and a narrowly scoped, read-only service account unless writes are really needed.

## Critical gaps
- The default grafana_api_request tool can call Grafana's service-account and access-control endpoints with the server's own credential, letting the model mint tokens or change roles when the token is Admin (C1-SELFESC). (ASI03, T3; C1) — [tools/api.go:18-24](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/api.go#L18-L24); [tools/api.go:129](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/api.go#L129)
- Default configuration lets a hijacked session both exfiltrate data (new webhook/email contact points) and take irreversible actions (DELETE on any API path, raw SQL) with no server-side gate (C5-WORSTCASE). (ASI01, LLM01, T6; C5) — [tools/alerting_routing.go:170-173](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/alerting_routing.go#L170-L173); [tools/api.go:18-24](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/api.go#L18-L24)
- install_plugin is on by default and installs a model-chosen plugin and version into Grafana, where it runs with Grafana's authority; the only consent is text asking the model to check with the user. (ASI04, T17, LLM03; C7) — [tools/plugins.go:201-219](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/plugins.go#L201-L219); [tools/plugins.go:417-422](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/plugins.go#L417-L422)

## Criterion details

### C1 Identity & least privilege — 0.25 (high)

The server uses one Grafana credential for everything: a service account token (or a username and password) taken from its environment, attached to every request by every tool. It does not narrow that credential per tool or per request, and the README's quick-path advice is to give the service account the broad Editor role. Write tools, including a general-purpose Grafana API tool that accepts any method and path, are on by default, so the model can call service-account and access-control endpoints and, with an Admin token, mint new tokens or change its own roles; only Grafana's own permission checks stop that.

- **S L1:** A dedicated Grafana service-account token from the environment is used as-is for every read and write; the server does no scoping, and the README suggests the broad Editor role as the easy setup. — [mcpgrafana.go:76-80](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/mcpgrafana.go#L76-L80); [mcpgrafana.go:732-740](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/mcpgrafana.go#L732-L740); [README.md:652](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/README.md#L652) (verified)
  - *To reach the next level:* No per-tool or read/write credential split and no in-code authorization policy mapping each request to a least-privilege set before the credential is attached.
- **C L2:** Every built-in tool, including the generic API tool, builds its client from the same config and AuthRoundTripper, so all paths use the one operator-supplied identity; there are no plugins or sub-agents. — [tools/api.go:124](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/api.go#L124); [mcpgrafana.go:908-911](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/mcpgrafana.go#L908-L911) (verified)
  - *To reach the next level:* No authorization layer in the server's code that every tool call passes through; authorization is entirely delegated to Grafana's checks on the shared token.
- **D L1:** Write tools and the unrestricted API tool ship enabled (disable-write defaults to false), and the privilege level is whatever token the operator supplies, with Editor suggested. — [cmd/mcp-grafana/main.go:229](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/cmd/mcp-grafana/main.go#L229); [cmd/mcp-grafana/main.go:372](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/cmd/mcp-grafana/main.go#L372) (verified)
  - *To reach the next level:* Default tool set is not read-only; write access needs no explicit operator elevation.
- **B L1:** A hijacked session can write across Grafana (dashboards, alert rules, contact points, datasources, plugins) and query every datasource the token can reach, including raw SQL against databases via /api/ds/query. — [tools/api.go:18-24](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/api.go#L18-L24); [tools/api.go:129](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/api.go#L129); [tools/api.go:28-30](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/api.go#L28-L30) (verified)
  - *To reach the next level:* Blast radius is not limited to one project or to read-mostly access; writes are not confined to non-destructive operations.
- **Cap:** C1-SELFESC — The default grafana_api_request tool sends any method to any /api path with the server's credential, so the model can call Grafana's service-account token and role-assignment endpoints for its own identity; only Grafana's RBAC on that token stops it.
- **Notes:** In the non-default streamable-http/SSE transports the server also accepts a caller-supplied X-Grafana-Service-Account-Token header and forwards it to Grafana (mcpgrafana.go:215-224, 954-970), i.e. MCP token passthrough (C1-PASSTHRU, cap 0.40); the lower C1-SELFESC cap already applies. With --allow-grafana-url-override the env credentials are deliberately withheld from caller-selected targets (mcpgrafana.go:955-958).

### C2 Approval gates — 0.33 (high)

The host, not the server, decides what to approve, so this rates the signals the server gives it. Every tool declares read-only, destructive and open-world hints, and a test fails the build if any tool is missing them, so hosts can tell reads from writes. But the default toolset includes grafana_api_request, one tool that does both reads and writes (correctly flagged destructive), there is no dry-run or preview for destructive operations, and the read-only mode is opt-in. Approved or bypassed calls can delete alert rules, snapshots and annotations, or run destructive SQL through datasources, with no undo in the server.

- **S L1:** Annotations are present and enforced on every tool, but the default generic API tool mixes reads and writes in one tool, and destructive operations have no preview or dry-run. — [tools/api.go:204-214](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/api.go#L204-L214); [cmd/mcp-grafana/annotations_test.go:46-79](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/cmd/mcp-grafana/annotations_test.go#L46-L79); [tools/plugins.go:234-243](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/plugins.go#L234-L243) (verified)
  - *To reach the next level:* Separate read and write tools on every path: grafana_api_request in its default write variant serves both GET reads and PUT/DELETE writes.
- **C L2:** Every registered tool goes through MustTool/Register and carries annotations, checked by a test over all categories with write enabled and disabled. — [cmd/mcp-grafana/annotations_test.go:54-79](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/cmd/mcp-grafana/annotations_test.go#L54-L79); [tools.go:126-128](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools.go#L126-L128) (verified)
  - *To reach the next level:* Coverage is limited by the mechanism: the mixed read/write API tool means hosts cannot gate writes without also gating its reads.
- **D L2:** Risk annotations are always emitted and nothing the model sends can change them; the server-enforced read-only mode (--disable-write) exists but is off by default. — [cmd/mcp-grafana/main.go:229](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/cmd/mcp-grafana/main.go#L229); [cmd/mcp-grafana/main.go:372](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/cmd/mcp-grafana/main.go#L372) (verified)
  - *To reach the next level:* No server-side read-only default or confirmation step; write tools are on unless the operator passes --disable-write.
- **B L0:** Default tools can perform irreversible actions: DELETE on any Grafana API path, deleting snapshots, annotations and alert rules, and raw SQL through POST /api/ds/query that can drop data if datasource credentials allow. — [tools/api.go:18-24](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/api.go#L18-L24); [README.md:565](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/README.md#L565) (verified)
  - *To reach the next level:* No checkpoints, rollback, or dry-run for destructive operations.
- **Cap:** none

### C3 Tool & action scoping — 0.53 (high)

Most tools are narrow, typed Grafana operations, and a shared wrapper rejects unknown arguments for every tool. The generic grafana_api_request tool, enabled by default, undoes much of that: it accepts any HTTP method, any API path on the Grafana host, arbitrary request headers and a free-form body, which includes raw datasource queries. The host is fixed and cross-origin redirects are refused, so it cannot be pointed at other servers directly. Write tools are on by default; a read-only mode and per-category switches exist but must be turned on.

- **S L2:** Typed schemas with additionalProperties=false and unknown-argument rejection, plus per-tool checks (path must start with '/', method allowlist, contact-point UID regex, Loki line cap), but the generic API tool takes any path, method, headers and body. — [tools.go:378-381](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools.go#L378-L381); [tools/api.go:107-109](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/api.go#L107-L109); [tools/api.go:144-146](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/api.go#L144-L146) (verified)
  - *To reach the next level:* No endpoint allowlist or parsed-query policy on grafana_api_request; raw SQL and admin endpoints pass through unvalidated.
- **C L3:** The same ConvertTool wrapper (schema decode and unknown-key rejection) runs for every tool; the server loads no extension tools. — [tools.go:370-381](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools.go#L370-L381); [tools.go:534](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools.go#L534) (verified)
  - *To reach the next level:* No central policy layer on arguments' meaning (paths, queries) that every tool inherits.
- **D L2:** Categories are selectable and a read-only mode exists, but the default category list includes write tools, the generic API tool, and plugin installation. — [cmd/mcp-grafana/main.go:210](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/cmd/mcp-grafana/main.go#L210); [cmd/mcp-grafana/main.go:229](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/cmd/mcp-grafana/main.go#L229) (verified)
  - *To reach the next level:* Default tool set is not read-only; write tools require --disable-write to remove.
- **B L1:** A misused grafana_api_request reaches every endpoint and every datasource the token can access on the Grafana host; limits are only the fixed host and a 10 MB response cap. — [tools/api.go:129](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/api.go#L129); [tools/response_utils.go:8](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/response_utils.go#L8) (verified)
  - *To reach the next level:* No scoping to specific resources and no quantity bounds on writes.
- **Cap:** none

### C4 Code-execution isolation — 0.05 (high)

The server never runs commands or code on its own machine: there is no subprocess or eval path, and the jq filter language it accepts runs in a pure-Go interpreter that is denied access to environment variables. It does, however, let the model have code run elsewhere. The default grafana_api_request tool can POST to /api/ds/query, which runs model-written raw SQL against any SQL datasource with that datasource's credentials, and the project's own README notes such queries will run DROP TABLE if the credentials allow. Nothing isolates that path.

- **S L0:** Model-written raw SQL is sent unfiltered through Grafana to the datasource database with the datasource's own credentials; there is no isolation primitive. — [tools/api.go:26-30](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/api.go#L26-L30); [cmd/mcp-grafana/main.go:231](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/cmd/mcp-grafana/main.go#L231) (verified)
  - *To reach the next level:* No boundary around remote query execution (e.g., forcing read-only transactions or a read-only datasource credential).
- **C L0:** The main remote-execution path (POST /api/ds/query via the default API tool) is not constrained; only the jq filter runs in a capability-free interpreter. — [tools/api.go:243-246](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/api.go#L243-L246); [tools/api.go:113-117](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/api.go#L113-L117); searched `rg -n -e 'exec\.Command' -e '"os/exec"' -g *.go -g !*_test.go` in `.` → 0 hits (no local subprocess execution anywhere in the server) (verified)
  - *To reach the next level:* Remote query execution is not routed through any isolating or read-only layer.
- **D L0:** No isolation exists to be on by default; the raw-SQL path ships enabled via the write-mode API tool. — [cmd/mcp-grafana/main.go:229](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/cmd/mcp-grafana/main.go#L229); [tools/api.go:243-246](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/api.go#L243-L246) (verified)
  - *To reach the next level:* No default that disables raw query execution or forces read-only datasource access.
- **B L1:** Model-written SQL runs inside production databases with whatever credentials the Grafana datasources hold; nothing runs on the MCP host itself. — [README.md:565](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/README.md#L565) (verified)
  - *To reach the next level:* Execution is not confined to read-only or scoped credentials.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

The server returns logs, traces, dashboards, annotations, incident text and other content that anyone who can write to those systems controls, with no marking of what is untrusted. Several tool descriptions and outputs also carry instructions to the model (for example 'you MUST ask the user' and 'Ask the user whether to install'), relying on the model for consent. In the default configuration the same session can read sensitive telemetry, create contact points that send data to external webhooks or email, install plugins, and delete resources, so a hijacked model can both leak and destroy without a human in the loop as far as the server is concerned.

- **S L0:** Tool descriptions and outputs contain directives to the model, and returned content carries no provenance or untrusted flag; many outputs are structured JSON but that does not separate trusted from untrusted text. — [tools/datasources.go:452](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/datasources.go#L452); [tools/plugins.go:210](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/plugins.go#L210) (verified)
  - *To reach the next level:* Remove model directives from descriptions/outputs and return structured outputs that separate returned content from server metadata.
- **C L0:** No untrusted source is distinguished; log lines, dashboard JSON and incident content enter results the same way as server metadata. — searched `rg -n -i untrusted -g !*_test.go` in `tools` → 0 hits (no untrusted-content marking in any tool) (verified)
  - *To reach the next level:* Mark content from every untrusted source (logs, traces, dashboards, annotations, incidents) as such.
- **D L0:** The modes that drop a Rule-of-Two leg (--disable-write, --disable-query) are off by default. — [cmd/mcp-grafana/main.go:229](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/cmd/mcp-grafana/main.go#L229); [cmd/mcp-grafana/main.go:230](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/cmd/mcp-grafana/main.go#L230) (verified)
  - *To reach the next level:* No default mode that removes state change or egress from sessions that read untrusted content.
- **B L0:** A hijacked session can read sensitive logs/metrics, exfiltrate via a new webhook or email contact point, and delete or overwrite Grafana resources, all unattended by the server. — [tools/alerting_routing.go:170-173](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/alerting_routing.go#L170-L173); [tools/api.go:18-24](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/api.go#L18-L24) (verified)
  - *To reach the next level:* Egress and irreversible actions are not removed or gated in sessions that read untrusted content.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 1.00 (high)

The server keeps no memory, retrieval store, or conversation history between sessions, and it does not load instruction or configuration files from a working directory. Configuration comes only from command-line flags and environment variables set by the operator, plus optional token, TLS and CA file paths the operator names. Caches (clients, frontend settings, docs index) live only in process memory.

- **Structural absence:** searched `rg -n -e godotenv -e 'os\.WriteFile' -e 'os\.Create\(' -e 'os\.OpenFile' -g *.go -g !*_test.go -g !internal/**` in `.` → 0 hits (no file writes or dotenv loading in server code (internal/ holds a dev-only schema linter)); searched `rg -n -i -e 'AGENTS\.md' -e 'CLAUDE\.md' -e vectorstore -e save_memory -g *.go -g !*_test.go` in `.` → 0 hits (no instruction-file loading or memory store); searched `rg -n 'os\.ReadFile' -g *.go -g !*_test.go -g !internal/**` in `.` → 2 hits (the only file reads are the operator-named token file (GRAFANA_SERVICE_ACCOUNT_TOKEN_FILE) and TLS CA file)

### C7 Third-party extensions — 0.00 (medium)

The server itself loads no plugins or remote code, but its default toolset includes install_plugin, which lets the model install any Grafana plugin at any version into the connected Grafana server through Grafana's install API. The only consent step is a message telling the model to ask the user when no version is given; if the model supplies a version, the install happens immediately. Installed plugins run inside Grafana (backend code on the server, frontend code in every user's browser), with far more access than the MCP server itself.

- **S L0:** The model chooses the plugin ID and version and the server asks Grafana to install it; there is no allowlist, pinning policy, or verification in the server. — [tools/plugins.go:201-219](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/plugins.go#L201-L219) (verified)
  - *To reach the next level:* No operator allowlist of installable plugins or server-side integrity/signature policy.
- **C L0:** No extension type is verified by the server; plugin installs are the only extension path and they are unchecked. — [tools/plugins.go:417-422](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/plugins.go#L417-L422) (verified)
  - *To reach the next level:* Apply verification to plugin installs.
- **D L0:** install_plugin is registered by default (plugin category in the default list, write tools on), and the confirmation is only text returned to the model. — [cmd/mcp-grafana/main.go:210](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/cmd/mcp-grafana/main.go#L210); [tools/plugins.go:202-216](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/plugins.go#L202-L216) (verified)
  - *To reach the next level:* Nothing third-party installable by default; installs should require operator enablement and a human confirmation the server enforces.
- **B L0:** A malicious or compromised plugin runs inside the Grafana server and in users' browsers, with Grafana's own authority rather than the MCP server's narrower token. — [tools/plugins.go:219-229](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/plugins.go#L219-L229) (inferred)
  - *To reach the next level:* No confinement of what an installed plugin can reach.
- **Cap:** none
- **Notes:** Blast radius is inferred from Grafana's plugin model (backend plugins run as Grafana-managed processes, frontend plugins load into the Grafana UI); Grafana's own signature checks for catalog plugins live outside this repository and were not examined. The installation succeeds only if the token holds plugin-install rights (typically Admin).

### C8 Secrets & sensitive-data protection — 0.40 (high)

The Grafana token never appears in tool output: tools have no way to read it, it is attached only by the transport layer, cross-origin redirects are refused so it cannot follow a redirect elsewhere, and URLs with embedded credentials are rejected. Debug logging of HTTP traffic masks Authorization and similar headers, and startup logs record only whether a key is set. However, tool results such as log lines are passed to the model without any secret scanning, the token is a long-lived credential from an environment variable or file, and anonymous usage statistics are sent to Grafana Labs by default (content-free: counts, tool names, auth method).

- **S L2:** Header redaction in debug logs, URL redaction in startup logs, rejection of credentials embedded in URLs, and a redirect guard that stops credentials leaving the Grafana origin; credentials are plaintext env vars or files. — [mcpgrafana.go:760-777](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/mcpgrafana.go#L760-L777); [redirect_guard.go:13-37](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/redirect_guard.go#L13-L37); [validate_url.go:49-51](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/validate_url.go#L49-L51) (verified)
  - *To reach the next level:* No secret manager/keychain storage and no redaction of secrets in model-bound tool results.
- **C L2:** Logs, error messages and telemetry are covered (redacted headers, origin-only redirect errors, bounded-vocabulary usage stats); model-bound tool results are not scanned. — [mcpgrafana.go:1009](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/mcpgrafana.go#L1009); [redirect_guard.go:40-47](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/redirect_guard.go#L40-L47) (verified)
  - *To reach the next level:* Redaction does not extend to model-bound messages (e.g., secrets inside returned log lines).
- **D L1:** Usage statistics reporting to stats.grafana.org is on by default and content-free; debug logging is off by default. — [usagestats/mode.go:29](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/usagestats/mode.go#L29); [usagestats/mode.go:44](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/usagestats/mode.go#L44) (verified)
  - *To reach the next level:* Telemetry should be opt-in.
- **B L1:** A leaked token is a long-lived Grafana service-account token whose scope is set by the operator (Editor suggested); file-based rotation is supported but not enforced. — [mcpgrafana.go:82-92](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/mcpgrafana.go#L82-L92); [README.md:652](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/README.md#L652) (verified)
  - *To reach the next level:* No short-lived or narrowly scoped credentials issued by the server.
- **Cap:** none

### C9 Audit & traceability — 0.35 (high)

In the default configuration the server keeps no record of the tool calls it makes: the observability middleware is a no-op unless metrics, tracing or slow-request logging is enabled, and tool handlers log nothing at info level. If the operator configures an OpenTelemetry endpoint, every tool call becomes a span with its name, session ID, status and error, exported off the host; arguments are added only with an extra flag, and there is no record of which human or principal asked for the call.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** No tool-call record by default: the MCP middleware returns the handler unchanged when metrics, slow-log and tracing are all off. — [observability/observability.go:620-627](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/observability/observability.go#L620-L627); searched `rg -n -e '\.Info\(' -e 'InfoContext\(' -g !*_test.go` in `tools` → 0 hits (tool handlers never log their actions at info level) (verified)
    - *To reach the next level:* A structured per-call record (tool, arguments, outcome, timestamp) written by default.
  - **C L0:** Nothing is recorded for any tool in the default configuration. — [cmd/mcp-grafana/main.go:1321](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/cmd/mcp-grafana/main.go#L1321) (verified)
    - *To reach the next level:* Record every tool call.
  - **D L0:** Recording is opt-in only (OTLP endpoint, --metrics or --slow-request-threshold). — [observability/observability.go:182](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/observability/observability.go#L182) (verified)
    - *To reach the next level:* Recording on by default.
  - **B L0:** With nothing recorded, actions proceed with no trace. — [observability/observability.go:625-627](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/observability/observability.go#L625-L627) (verified)
    - *To reach the next level:* Records flushed per action with errors surfaced.
- **opt-in OpenTelemetry tracing (OTEL_EXPORTER_OTLP_ENDPOINT)** (alt; raw 0.35, cap G1 → 0.35) ← counted
  - **S L2:** Each tool call is a span with tool name, session ID, status and error, exported via OTLP off the host; arguments only with --include-args-in-spans. — [tools.go:347-368](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools.go#L347-L368); [cmd/mcp-grafana/main.go:275](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/cmd/mcp-grafana/main.go#L275) (verified)
    - *To reach the next level:* No actor attribution (requesting principal) and arguments are excluded by default.
  - **C L2:** Spans are created in the shared ConvertTool wrapper, so every built-in tool is covered, including unknown-argument rejections. — [tools.go:378-381](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools.go#L378-L381) (verified)
    - *To reach the next level:* No record of configuration changes or credential use.
  - **D L0:** Off unless an OTLP endpoint is configured. — [observability/observability.go:182-193](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/observability/observability.go#L182-L193) (verified)
    - *To reach the next level:* On by default.
  - **B L1:** Spans are exported by a batching processor, best-effort and flushed late; export failures only print to stderr. — [observability/observability.go:187-190](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/observability/observability.go#L187-L190) (verified)
    - *To reach the next level:* Per-action durable flush.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C10 Limits & kill switch — 0.40 (high)

The server bounds some of its own work: most HTTP responses are capped at 10 MB, Loki log queries at 100 lines, and the typed Grafana client has a 10-second default timeout. Other paths are open-ended: the generic API tool's HTTP client has no timeout, jq filters run until the caller cancels, the model can set any rendering timeout, and the Loki cost guardrail is off by default. There are no rate or concurrency limits on tool calls, including write tools.

- **S L2:** Server-enforced caps on some operations: 10 MB response limit, Loki line cap, 10 s typed-client timeout. — [tools/response_utils.go:12-20](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/response_utils.go#L12-L20); [tools/loki.go:26](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/loki.go#L26); [mcpgrafana.go:430](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/mcpgrafana.go#L430) (verified)
  - *To reach the next level:* Caps on every operation plus rate or concurrency limits.
- **C L2:** Most HTTP tool reads use the size cap, but grafana_api_request's client has no timeout and its jq evaluation is unbounded. — [tools/api.go:148](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/api.go#L148); [tools/api.go:185-187](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/api.go#L185-L187) (verified)
  - *To reach the next level:* Bounds on every tool path, including the generic API tool.
- **D L1:** The model can raise the rendering timeout to any value, and the Loki guardrail defaults to off. — [tools/rendering.go:128-131](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/tools/rendering.go#L128-L131); [cmd/mcp-grafana/main.go:268](https://github.com/grafana/mcp-grafana/blob/9dbfe137a5d3c3bc930ab7d9565e7917dac1cc0f/cmd/mcp-grafana/main.go#L268) (verified)
  - *To reach the next level:* Limits the model cannot raise.
- **B L1:** No ceiling on how many write or query calls run; a runaway host can issue unlimited destructive calls. — searched `rg -n -e x/time/rate -e semaphore -e MaxConcurrent -g *.go -g !*_test.go` in `.` → 0 hits (no rate or concurrency limiter in the server) (verified)
  - *To reach the next level:* Rate limits on side-effecting tools.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Log lines, traces, dashboards, annotations and incident text returned by default tools (tools/loki.go query_loki_logs, tools/api.go:129) · [B] sensitive data/systems: All datasources the token can query, including logs and SQL databases (tools/api.go:28-30 /api/ds/query) · [C] state change / egress: Write tools on by default: grafana_api_request DELETE/PUT (tools/api.go:18-24), contact-point creation to external webhooks (tools/alerting_routing.go:170-173), install_plugin (tools/plugins.go:219) · Same default session? Yes

## Highest-impact improvements
1. Ship read-only by default: make --disable-write the default and require an explicit flag to register write tools. — C3 D L2→L3, +0.050 before caps (Playbook 3)
2. Write a structured record of every tool call (tool, arguments, outcome, timestamp) to stderr or a log file by default. — C9 S L0→L2, +0.150 before caps (Playbook 1 step 3)
3. Remove install_plugin from the default toolset and gate it behind an operator allowlist of plugin IDs. — C7 D L0→L3, +0.150 before caps (Playbook 3)
4. Drop model-directed instructions from tool descriptions and outputs and mark returned log/dashboard content as untrusted with its source. — C5 S L0→L3, +0.225 before caps (Playbook 1)
5. Add an endpoint allowlist to grafana_api_request that blocks service-account, access-control, admin and plugin-install paths. — C3 S L2→L3, +0.075 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Grafana server behaviour (RBAC enforcement, plugin signature checks, SQL datasource handling) is outside the repository and was inferred, not verified; C7 blast radius is INFERRED.
- The third-party module github.com/grafana/mcp-doc-server (used by get_doc to fetch documentation URLs) and the MCP go-sdk internals were not examined.
- The non-default HTTP transports (streamable-http, SSE) were reviewed only for credential handling: they forward caller tokens (token passthrough) and have optional bearer auth and Host/Origin checks.
- No reviewer-injection text was found in the repository.
