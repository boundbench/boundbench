# Defense-in-depth score: Netdata (MCP server)

**Repo:** https://github.com/netdata/netdata · **Commit:** `4106699b00be39dddbf3e9416db250ec3a5a7c5c` · **Reviewed:** 2026-10-05
**What it is:** Real-time infrastructure monitoring agent with a built-in MCP server that exposes metrics, alerts, logs and live system functions to AI assistants.
**Category:** Infrastructure & Ops
**Scored configuration:** Built-in Agent MCP server at /mcp on the default web listener (bind *, bearer protection off, 'allow mcp from' = dashboard default), claimed agent, AI client configured with the auto-generated MCP API key as the README instructs.
**Agent surface (default):** code execution no · filesystem write no · network egress no · external credentials no · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 5.1 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L1 | L1 | 0.25 | G2 | **0.25** | High |
| C2 | Approval gates | L1 | L1 | L1 | L2 | 0.30 | none | **0.30** | High |
| C3 | Tool & action scoping | L2 | L2 | L2 | L1 | 0.45 | none | **0.45** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L1 | L0 | L1 | L1 | 0.17 | none | **0.17** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L1 | 0.25 | none | **0.25** | High |
| C9 | Audit & traceability | L1 | L1 | L2 | L1 | 0.30 | none | **0.30** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | none | **0.40** | High |

Controls where a risk surface exists: 2.12 / 7.0 (30%); 3 criteria scored SA (surface absent).

Netdata's built-in MCP server is a small, mostly read-only surface: no code execution, memory or extensions. The weak point is access control: anonymous MCP access is open to any address by default, the only API key grants full administrator access across every node a Parent sees, and the authorization check does not hold on every path in the default configuration. Logs and process data reach the model unmarked and unredacted, and per-tool-call audit records exist only at debug level.

## Critical gaps
- The MCP server's per-request authorization check does not hold on every path in the default configuration. (ASI03, T3, LLM06; C1). Evidence: [src/nrpc/nrpc-calls.c:548](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/nrpc/nrpc-calls.c#L548); [src/web/mcp/mcp-tools-execute-function.c:2037-2040](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-execute-function.c#L2037-L2040)

## Criterion details

### C1 Identity & least privilege: 0.25 (high confidence)

Anyone who can reach the Netdata web port gets anonymous MCP access to metrics, alerts and node information, because bearer protection is off by default and the MCP access list follows the dashboard default of allowing everyone. The auto-generated MCP API key, once presented, grants full administrator access on that agent and, on a Parent, across every streamed child. Every function call does pass a deterministic access check mapped to Netdata's permission levels, but the API key satisfies all of them, there is no per-user or per-tool scoping, and the authorization check does not hold on every path in the default configuration.

- **S L1:** The MCP API key maps to full administrator access (all permission bits), and anonymous callers get the anonymous-data tier; there is no narrower identity per tool or per user. Evidence: [src/web/mcp/adapters/mcp-http.c:79](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/adapters/mcp-http.c#L79); [src/web/api/web_api.c:25](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/api/web_api.c#L25); [src/nrpc/nrpc-calls.c:548](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/nrpc/nrpc-calls.c#L548) (verified)
  - *To reach the next level:* No per-tool or per-capability scoping: the key used for metric reads is the same admin-equivalent key used for logs and live functions.
- **C L1:** The function path checks the caller's access level against each method, but the check does not hold on every path the tools take in the default configuration. Evidence: [src/nrpc/nrpc-calls.c:540](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/nrpc/nrpc-calls.c#L540); [src/nrpc/nrpc-calls.c:548](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/nrpc/nrpc-calls.c#L548); [src/web/mcp/mcp-tools-execute-function.c:2037-2040](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-execute-function.c#L2037-L2040) (verified)
  - *To reach the next level:* Not every tool path is held to the requesting caller's access level.
- **D L1:** Bearer protection defaults to off, the MCP access list defaults to all addresses on a listener bound to every interface, and the documented client setup uses the full-access API key. Evidence: [src/web/api/http_auth.c:8](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/api/http_auth.c#L8); [src/daemon/config/netdata-conf-web.c:88](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/daemon/config/netdata-conf-web.c#L88); [src/daemon/config/netdata-conf-web.c:82](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/daemon/config/netdata-conf-web.c#L82); [src/web/server/web_server.c:34](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/server/web_server.c#L34); [src/web/api/mcp_auth.c:165](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/api/mcp_auth.c#L165) (verified)
  - *To reach the next level:* Default exposure is not minimal: anonymous MCP is reachable from any address, and the only credential offered is admin-equivalent.
- **B L1:** A hijacked or stolen key reads logs, processes and connections on the agent and, on a Parent, on every child, with no expiry or scope reduction. Evidence: [src/web/mcp/adapters/mcp-http.c:79](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/adapters/mcp-http.c#L79); [src/web/api/mcp_auth.c:179](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/api/mcp_auth.c#L179); [src/web/api/mcp_auth.c:32](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/api/mcp_auth.c#L32) (verified)
  - *To reach the next level:* Key is long-lived and fleet-wide on a Parent; no scoping to a single node or to read-only operations.
- **Cap:** G2: The MCP server's per-request authorization check does not hold on every path in the default configuration.

### C2 Approval gates: 0.30 (high confidence)

As a tool server, Netdata leaves approval to the AI client and gives it risk annotations on every tool: each one is flagged read-only, and the function tool is flagged as reaching other nodes. That signalling is coarse. The function-execution tool is a generic dispatcher: it runs whatever named function the node exposes, so a single annotation cannot describe every call, and the server has no enforced read-only mode, dry-run or confirmation step of its own. Most tools only read metrics and alerts, so a wrongly approved call usually changes nothing.

- **S L1:** Annotations (readOnlyHint, openWorldHint) are emitted for every tool, but the generic function tool takes a free-form function name, so one annotation covers every function the node exposes. Evidence: [src/web/mcp/mcp-tools.c:301-302](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools.c#L301-L302); [src/web/mcp/mcp-tools.c:134](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools.c#L134); [src/web/mcp/mcp-tools-execute-function.c:2037-2040](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-execute-function.c#L2037-L2040) (verified)
  - *To reach the next level:* No per-call risk signalling for the generic function tool and no server-enforced read-only mode.
- **C L1:** All tools carry the same annotation set, so a client gate keyed on annotations treats every function call alike. Evidence: [src/web/mcp/mcp-tools.c:301-302](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools.c#L301-L302); [src/web/mcp/mcp-tools.c:338-342](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools.c#L338-L342) (verified)
  - *To reach the next level:* Function calls are not split into separately annotated tools the client's gate can tell apart.
- **D L1:** Annotations ship on by default, but the model chooses the function name per call, which decides what the generic tool does. Evidence: [src/web/mcp/mcp-tools-execute-function.c:2037-2040](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-execute-function.c#L2037-L2040); [src/web/mcp/mcp-tools.c:301-302](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools.c#L301-L302) (verified)
  - *To reach the next level:* No server-side restriction of the function tool to a fixed read-only set that the model cannot widen.
- **B L2:** Most tools are pure reads of metrics, alerts and metadata; what the function tool does depends on which functions the node registers. Evidence: [src/web/mcp/mcp-tools.c:338-342](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools.c#L338-L342); [src/web/mcp/mcp-tools-execute-function.c:2049](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-execute-function.c#L2049) (verified)
  - *To reach the next level:* No preview or dry-run and no rate limit on function calls.
- **Cap:** none

### C3 Tool & action scoping: 0.45 (high confidence)

Tool arguments are parsed with typed helpers and several bounds: data points are capped at 1,000, query cardinality at 1,000, selected columns and filter conditions have maximums, and enums such as sort order and direction are checked. The function tool, however, is a general dispatcher keyed on a free-form function name that is resolved against whatever the node registers. Some bounds are missing (the weights cardinality and function row limit are unbounded), and every tool is on by default. With the API key, the reach covers logs and live system data across every node a Parent sees.

- **S L2:** Typed parameter extraction with numeric bounds on most query parameters, but the function tool accepts any registered function name and some limits are unbounded. Evidence: [src/web/mcp/mcp-tools-query-metrics.c:433](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-query-metrics.c#L433); [src/web/mcp/mcp-tools-execute-function.c:2114](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-execute-function.c#L2114); [src/web/mcp/mcp-tools-weights.c:84](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-weights.c#L84); [src/web/mcp/mcp-tools-execute-function.c:2082](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-execute-function.c#L2082); [src/web/mcp/mcp-tools-execute-function.c:2037-2040](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-execute-function.c#L2037-L2040) (verified)
  - *To reach the next level:* Replace the generic function dispatcher with narrow per-function tools and bound every numeric argument.
- **C L2:** Most tools use the shared parameter helpers; the weights cardinality and function row limit have no upper bound. Evidence: [src/web/mcp/mcp-tools-query-metrics.c:433](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-query-metrics.c#L433); [src/web/mcp/mcp-tools-weights.c:84](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-weights.c#L84); [src/web/mcp/mcp-tools-execute-function.c:2082](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-execute-function.c#L2082) (verified)
  - *To reach the next level:* No single policy layer that bounds every argument of every tool.
- **D L2:** All 13 tools, including the generic function tool, are exposed by default with no per-tool enable switch; sensitive functions need the API key. Evidence: [src/web/mcp/mcp-tools.c:338-342](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools.c#L338-L342); [src/web/mcp/adapters/mcp-http.c:79](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/adapters/mcp-http.c#L79) (verified)
  - *To reach the next level:* No read-only-by-default tool set with the function tool disabled until enabled.
- **B L1:** A misused function tool reaches logs, process tables and connections on every node visible to the agent or Parent. Evidence: [src/web/mcp/mcp-tools-execute-function.c:2037-2040](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-execute-function.c#L2037-L2040); [src/web/mcp/mcp-tools-execute-function.c:2049](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-execute-function.c#L2049) (verified)
  - *To reach the next level:* Reach is not scoped to a node or bounded in quantity for log and function reads.
- **Cap:** none

### C4 Code-execution isolation: 1.00 (high confidence)

The MCP server never interprets model-generated text as code: no shell, eval or subprocess path exists in the MCP code, and tool arguments are passed as function names and filter values to Netdata's existing function handlers.

- **Structural absence:** searched `rg -n -S -g '*.c' -g '*.h' -e 'popen|system\(|execv|execl|fork\(|spawn|eval'` in `src/web/mcp` → 5 hits (all hits are false positives: mcp_initialize_subsystem() (3) and prose containing 'evaluate'/'evaluated' (2))

### C5 Untrusted input blast radius: 0.17 (high confidence)

The server returns logs, process command lines, connection tables and alert text to the AI client as plain text, with no marking that this content is untrusted and no provenance the client could act on. It offers no mode that drops sensitive data or state-changing functions for sessions that read such content. Anything written into a log line is therefore handed to the model with the same standing as the server's own guidance, and with the API key the same session can read sensitive data across the fleet.

- **S L1:** Function output is passed through as plain text content; server guidance is placed in separate content items, but nothing marks returned data as untrusted. Evidence: [src/web/mcp/mcp-tools-execute-function.c:2274](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-execute-function.c#L2274); searched `rg -n -S -g '*.c' -g '*.h' -e 'untrusted|provenance|redact|sanitiz'` in `src/web/mcp` → 0 hits (no provenance marking or redaction of returned content) (verified)
  - *To reach the next level:* No provenance or untrusted flag on returned content.
- **C L0:** No source is distinguished: journal logs, process tables and alert text all enter as plain text. Evidence: searched `rg -n -S -g '*.c' -g '*.h' -e 'untrusted|provenance|redact|sanitiz'` in `src/web/mcp` → 0 hits (no provenance marking or redaction of returned content); [src/web/mcp/mcp-tools-execute-function.c:2274](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-execute-function.c#L2274) (verified)
  - *To reach the next level:* No untrusted source is marked or separated.
- **D L1:** Plain-text output with server guidance in separate items is always on, but nothing in it stops returned content from steering the client. Evidence: [src/web/mcp/mcp-tools-execute-function.c:2274](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-execute-function.c#L2274); searched `rg -n -S -g '*.c' -g '*.h' -e 'untrusted|provenance|redact|sanitiz'` in `src/web/mcp` → 0 hits (no provenance marking or redaction of returned content) (verified)
  - *To reach the next level:* No untrusted-content marking ships on by default.
- **B L1:** With the documented API-key setup a hijacked client can read logs and live system data across all nodes; the server itself has no outbound channel, but it offers no mode that removes sensitive reads. Evidence: [src/web/mcp/adapters/mcp-http.c:79](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/adapters/mcp-http.c#L79); [src/web/mcp/mcp-tools-execute-function.c:2037-2040](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-execute-function.c#L2037-L2040) (verified)
  - *To reach the next level:* No read-only or low-sensitivity mode the client can select for sessions that ingest untrusted content.
- **Cap:** none

### C6 Memory, context & configuration integrity: 1.00 (high confidence)

The MCP server keeps no memory the model can write and loads no workspace files; the only cache holds function metadata fetched from the node and expires on a timer.

- **Structural absence:** searched `rg -n -S -g '*.c' -g '*.h' -e 'fopen|fwrite|unlink|rename\(|memory|remember|persist'` in `src/web/mcp` → 14 hits (all hits are prose examples (metric names like redis.memory, 'if issues persist', 'Remember:') or comments on in-memory buffers; no file writes or memory store)

### C7 Third-party extensions: 1.00 (high confidence)

The MCP server loads no third-party code: no plugin loading, package installation or model files exist in the MCP code. The functions it dispatches are Netdata's own collector functions installed by the operator.

- **Structural absence:** searched `rg -n -S -g '*.c' -g '*.h' -e 'dlopen|npx|uvx|pip install|npm install|load_plugin'` in `src/web/mcp` → 0 hits (no extension loading in the MCP server)

### C8 Secrets & sensitive-data protection: 0.25 (high confidence)

The MCP API key is a random UUID stored in a file only the netdata user can read, and the stdio bridge can take it from an environment variable. Beyond that there is no secret handling: logs and process command lines are returned to the model unredacted, so any secret that appears in them goes to the model provider. The key never expires, and the legacy query-string form of the key is still accepted. Netdata's anonymous telemetry is on by default (content-free), with an opt-out file or environment variable.

- **S L1:** Key stored in a 0600 file; no masking or redaction of logs or function output sent to the model. Evidence: [src/web/api/mcp_auth.c:32](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/api/mcp_auth.c#L32); [src/web/api/mcp_auth.c:64](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/api/mcp_auth.c#L64); searched `rg -n -S -g '*.c' -g '*.h' -e 'untrusted|provenance|redact|sanitiz'` in `src/web/mcp` → 0 hits (no provenance marking or redaction of returned content) (verified)
  - *To reach the next level:* No redaction of secrets in returned logs/command lines and no secret store beyond a plaintext file.
- **C L1:** Only the at-rest key file is protected; model-bound output and logs are not. Evidence: [src/web/api/mcp_auth.c:64](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/api/mcp_auth.c#L64); [docs/netdata-ai/mcp/README.md:659](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/docs/netdata-ai/mcp/README.md#L659) (verified)
  - *To reach the next level:* Model-bound messages and logs are not covered.
- **D L1:** Anonymous telemetry is on unless an opt-out file or DISABLE_TELEMETRY is set; the legacy query-string key form remains accepted. Evidence: [src/daemon/analytics.c:1528-1530](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/daemon/analytics.c#L1528-L1530); [docs/netdata-ai/mcp/README.md:659](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/docs/netdata-ai/mcp/README.md#L659) (verified)
  - *To reach the next level:* Telemetry is not opt-in and the query-string key form is not disabled by default.
- **B L1:** The key is long-lived (persisted until the file is deleted) and grants admin-equivalent access to the agent. Evidence: [src/web/mcp/adapters/mcp-http.c:79](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/adapters/mcp-http.c#L79); [src/web/api/mcp_auth.c:32](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/api/mcp_auth.c#L32); [src/web/mcp/bridges/stdio-golang/nd-mcp.go:113](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/bridges/stdio-golang/nd-mcp.go#L113) (verified)
  - *To reach the next level:* No short-lived or scoped credential.
- **Cap:** none

### C9 Audit & traceability: 0.30 (high confidence)

Each HTTP request to the MCP endpoint lands in Netdata's access log with client IP, role and access level, written by the daemon outside anything the model controls. Which tool was called and with which arguments is logged only at debug level, and individual function calls likewise only at debug level, so an incident cannot be reconstructed per tool call from default logs. WebSocket sessions are a single long-lived request.

- **S L1:** Access-log entries per HTTP request with caller attribution; tool names and arguments only in debug logs. Evidence: [src/web/server/web_client.c:281-282](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/server/web_client.c#L281-L282); [src/web/server/web_client.c:255-258](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/server/web_client.c#L255-L258); [src/web/mcp/mcp-tools.c:357](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools.c#L357) (verified)
  - *To reach the next level:* No structured record of each tool call with its arguments and result status.
- **C L1:** Only the HTTP request is recorded; tool calls inside batches or WebSocket sessions are not. Evidence: [src/web/mcp/mcp-tools.c:357](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools.c#L357); [src/nrpc/nrpc-calls.c:774](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/nrpc/nrpc-calls.c#L774) (verified)
  - *To reach the next level:* Tool calls are not recorded individually across transports.
- **D L2:** The access log is on by default and written by the daemon, outside anything the model can write. Evidence: [src/web/server/web_client.c:281-282](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/server/web_client.c#L281-L282) (verified)
  - *To reach the next level:* Tool-call records themselves are not produced by default.
- **B L1:** Logging is best-effort through the daemon's log pipeline; actions do not depend on a record being written. Evidence: [src/web/server/web_client.c:281-282](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/server/web_client.c#L281-L282) (verified)
  - *To reach the next level:* No per-action durable record.
- **Cap:** none

### C10 Limits & kill switch: 0.40 (high confidence)

The server enforces some bounds on its own work: function calls time out after 60 seconds by default, responses are capped at 16 MiB, and data queries are capped at 1,000 points. The caller can raise function timeouts to an hour, metric queries have no timeout unless one is passed, batched requests are unbounded in count, and the server does not implement cancellation of in-flight calls.

- **S L2:** Server-enforced caps on some operations (function timeout, response size, query points). Evidence: [src/web/mcp/mcp-tools-execute-function.c:2049](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-execute-function.c#L2049); [src/web/mcp/mcp.c:165](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp.c#L165); [src/web/mcp/mcp-tools-query-metrics.c:433](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-query-metrics.c#L433); searched `rg -n -S -g '*.c' -g '*.h' -e 'cancel'` in `src/web/mcp` → 0 hits (no MCP cancellation (notifications/cancelled) handling in the server) (verified)
  - *To reach the next level:* No caps on every operation, no rate limit, and no cancellation of in-flight work.
- **C L2:** Function and weights calls have timeouts; metric queries default to no timeout and batches are unbounded. Evidence: [src/web/mcp/mcp-tools-query-metrics.c:455](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-query-metrics.c#L455); [src/web/api/queries/query.c:313](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/api/queries/query.c#L313); [src/web/mcp/adapters/mcp-http.c:170-175](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/adapters/mcp-http.c#L170-L175) (verified)
  - *To reach the next level:* Limits do not cover every operation and request batches.
- **D L1:** Defaults exist but the caller (the model) can raise timeouts up to 3600 seconds per call. Evidence: [src/web/mcp/mcp-tools-execute-function.c:2049](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-execute-function.c#L2049); [src/web/mcp/mcp-tools-query-metrics.c:455](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/mcp-tools-query-metrics.c#L455) (verified)
  - *To reach the next level:* Caller can raise its own limits.
- **B L1:** A runaway client can hold hour-long calls and unbounded batches; nothing cancels in-flight calls. Evidence: [src/web/mcp/adapters/mcp-http.c:170-175](https://github.com/netdata/netdata/blob/4106699b00be39dddbf3e9416db250ec3a5a7c5c/src/web/mcp/adapters/mcp-http.c#L170-L175); searched `rg -n -S -g '*.c' -g '*.h' -e 'cancel'` in `src/web/mcp` → 0 hits (no MCP cancellation (notifications/cancelled) handling in the server) (verified)
  - *To reach the next level:* No tight ceilings and no stop for in-flight work.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: journal logs, process command lines and alert text returned as plain text (src/web/mcp/mcp-tools-execute-function.c:2274) · [B] sensitive data/systems: API key grants full access to logs and live functions on all nodes (src/web/mcp/adapters/mcp-http.c:79) · [C] state change / egress: generic function tool dispatches any registered node function by name (src/web/mcp/mcp-tools-execute-function.c:2037-2040); egress is the AI client's · Same default session? Yes

## Highest-impact improvements
1. Authorize every MCP tool path against the requesting caller's access level, failing closed. (C1 C L1→L2, +0.075 before caps; Playbook 4)
2. Restrict execute_function to an explicit allowlist of functions and add a server-enforced read-only mode. (C2 S L1→L2, +0.075 before caps; Playbook 5)
3. Issue scoped MCP keys (read-only metrics vs logs/functions) instead of one key that maps to full administrator access. (C1 S L1→L2, +0.075 before caps; Playbook 4)
4. Log every tool call (tool name, node, function, caller, result) at info level in the access or audit log. (C9 S L1→L2, +0.075 before caps; Playbook 1 step 3)
5. Default MCP access to localhost or require the API key for all MCP calls when the agent is reachable from the network. (C1 D L1→L2, +0.050 before caps; Playbook 4)

## Re-audit log
- C3 C: L3 → L2. Weights cardinality_limit and function row limit are read without upper bounds (mcp-tools-weights.c:84, mcp-tools-execute-function.c:2082), so not every built-in tool validates.
- C9 D: L3 → L2. Access log is written by a component the model cannot control, but D may be at most one level above S (L1).
- C5 S: L0 → L1. Defended L0: server guidance text is emitted as separate content items from returned data; outputs are plain text with no provenance (tool-server L1).

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scope is the AI/MCP surface Netdata ships in this repository: the built-in Agent MCP server (src/web/mcp, its authentication and routing, and the function-dispatch layer it calls) and the stdio bridges. The rest of the monitoring agent (collectors, streaming, exporting, dashboard) was examined only where the MCP tools call into it.
- Netdata Cloud MCP (app.netdata.cloud) and the Netdata AI troubleshooting, investigations and MCP-client features are closed-source cloud services documented in docs/netdata-ai; they are not in this repository and were not scored.
- Individual collector functions reachable through execute_function were not each traced; ratings assume the function set Netdata registers by default.
- The repository contains AGENTS.md, CLAUDE.md and GEMINI.md developer instruction files; they were treated as data. No text aimed at reviewers was found in the MCP sources.
