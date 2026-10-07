# Defense-in-Depth Score: Exa MCP

**Repo:** https://github.com/exa-labs/exa-mcp-server · **Commit:** `f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97` · **Reviewed:** 2026-10-05
**What it is:** MCP server exposing Exa web search, page fetching and Exa Agent research tools, deployed as the hosted Streamable-HTTP server at mcp.exa.ai and as a local stdio package.
**Category:** AI Assistants
**Scored configuration:** The Streamable-HTTP server in api/mcp.ts as hosted at https://mcp.exa.ai/mcp (the README's lead option), with the default tool set (web_search_exa, web_fetch_exa) and either no credential (free tier on the operator's key) or the caller's own Exa API key or OAuth token.
**Agent surface (default):** code execution no · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents opt-in · external communication no

## Score: 5.7 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L3 | L3 | L3 | 0.68 | C1-PASSTHRU | **0.25** | High |
| C2 | Approval gates | L1 | L2 | L2 | L4 | 0.53 | none | **0.53** | High |
| C3 | Tool & action scoping | L2 | L3 | L3 | L2 | 0.62 | none | **0.62** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L1 | 0.05 | none | **0.05** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L2 | L1 | L2 | 0.38 | none | **0.38** | High |
| C9 | Audit & traceability | L1 | L2 | L2 | L1 | 0.38 | none | **0.38** | High |
| C10 | Limits & kill switch | L2 | L3 | L1 | L2 | 0.53 | none | **0.53** | High |

Controls where a risk surface exists: 2.88 / 7.0 (41%); 3 criteria scored SA (surface absent).

A small, read-only web search and fetch server: it runs no code, touches no files and only talks to Exa's API, and its default tools cannot change anything. Its main gaps in posture are around untrusted content and authority: web pages come back as unmarked text, the fetch tool gives a hijacked model an outbound channel to any URL, and callers' API keys and OAuth tokens are forwarded unchanged to Exa's API. Limits exist as timeouts and free-tier rate limits, but result sizes are model-chosen and logging is plain console text.

## Critical gaps
- OAuth access tokens and API keys presented to the MCP server are forwarded unchanged to Exa's API rather than exchanged for a server-scoped credential (token passthrough). (ASI03, T3; C1) — [api/mcp.ts:456-462](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/api/mcp.ts#L456-L462); [src/tools/config.ts:23-25](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/config.ts#L23-L25)

## Criterion details

### C1 Identity & least privilege — 0.25 (high)

The hosted server runs each request with whatever credential the caller brings: the caller's own Exa API key (header or URL), an Exa OAuth access token, or, for anonymous callers, the operator's shared Exa key. A fresh server instance is built per request, so one caller's key is never reused for another. The agent tool is only registered for callers who bring their own credential, and a token that fails verification is rejected rather than downgraded. OAuth access tokens issued for the MCP server are verified and then forwarded unchanged to Exa's API (token passthrough), so the server's own authority is not separated from the downstream API's.

- **S L2:** Authority is per request: the caller's own API key or verified OAuth token is used for that request, and anonymous callers fall back to the operator's single shared Exa key; there is no per-tool or downscoped credential. — [api/mcp.ts:433-449](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/api/mcp.ts#L433-L449); [api/mcp.ts:456-462](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/api/mcp.ts#L456-L462); [src/tools/config.ts:47-58](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/config.ts#L47-L58) (verified)
  - *To reach the next level:* No per-tool or read-only credential; the forwarded key or token carries the caller's full Exa API authority.
- **C L3:** Every tool builds its client from the per-request config, and one registration check (requiresUserProvidedApiKey) withholds the agent and deep-research tools from anonymous callers both at registration and at the HTTP layer. — [src/mcp-handler.ts:65-75](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/mcp-handler.ts#L65-L75); [api/mcp.ts:737-739](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/api/mcp.ts#L737-L739); [api/mcp.ts:565-574](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/api/mcp.ts#L565-L574) (verified)
  - *To reach the next level:* Authorization is not evaluated against a server-side policy for the requesting principal; whatever the forwarded credential allows downstream is allowed.
- **D L3:** The default tool set is the two read-only search and fetch tools, and the credit-heavy agent tools need both an explicit ?tools= selection and the caller's own credential; invalid OAuth tokens return 401 instead of falling back to the shared key. — [src/toolRegistry.ts:12-17](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/toolRegistry.ts#L12-L17); [src/toolRegistry.ts:86-93](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/toolRegistry.ts#L86-L93); [api/mcp.ts:733-735](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/api/mcp.ts#L733-L735) (verified)
  - *To reach the next level:* Elevation is not time-bounded, and tool selection is a URL parameter rather than an operator-only setting.
- **B L3:** If this layer fails, the exposure is one Exa account per caller (search, content fetch and agent-run spend) or the operator's shared free-tier key; no other system is reachable and no Exa data is modified. — [src/tools/config.ts:54-58](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/config.ts#L54-L58) (verified)
  - *To reach the next level:* Forwarded API keys are long-lived and account-wide rather than short-lived and task-scoped.
- **Cap:** C1-PASSTHRU — OAuth access tokens verified for the mcp.exa.ai audience are forwarded unchanged as the Authorization header to Exa's API, and client-supplied API keys are forwarded likewise.

### C2 Approval gates — 0.53 (high)

As a tool server it relies on the MCP host to ask the user before calls. It gives the host risk labels on every tool, and the two default tools are accurately labelled read-only lookups that cannot change anything. The labels are not consistent across the opt-in tools: the agent tool, which starts billable multi-minute runs, is labelled read-only while the older equivalent research tool is not, and the advanced search tool says it does not reach the open web. There is no dry-run, confirmation step or server-enforced read-only mode, but in the default configuration no tool can change state, so a wrongly approved call costs only search credits.

- **S L1:** Every tool carries readOnlyHint/destructiveHint annotations, accurate on the default tools, but agent_run (which creates billable runs) is marked readOnlyHint: true while deep_researcher_start is marked false, and web_search_advanced_exa sets openWorldHint: false. — [src/tools/agentRun.ts:547](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/agentRun.ts#L547); [src/tools/deepResearchStart.ts:39](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/deepResearchStart.ts#L39); [src/tools/webSearchAdvanced.ts:133-135](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/webSearchAdvanced.ts#L133-L135); [src/tools/webSearch.ts:44-46](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/webSearch.ts#L44-L46) (verified)
  - *To reach the next level:* Annotations must be accurate on every tool, including agent_run and web_search_advanced_exa.
- **C L2:** All 11 tool registrations carry annotations and no tool in any mode reaches a write or delete API; there is no server-side gate on any path. — searched `rg -n readOnlyHint` in `src` → 11 hits (one per tool registration; 11 server.tool calls in src/tools); [src/tools/webFetch.ts:63-65](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/webFetch.ts#L63-L65) (verified)
  - *To reach the next level:* No server-side confirmation or preview exists for the credit-spending agent run path.
- **D L2:** Annotations and the default tool list are hard-coded; callers widen the set only through the ?tools= URL parameter, and the model cannot add tools. — [api/mcp.ts:494-503](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/api/mcp.ts#L494-L503); [src/mcp-handler.ts:58-63](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/mcp-handler.ts#L58-L63) (verified)
  - *To reach the next level:* No server-enforced read-only mode or confirmation step the host must complete.
- **B L4:** The default configuration permits no consequential action: the two default tools only POST queries and URLs to Exa's /search and /contents endpoints, enforced in code. — [src/tools/webSearch.ts:87-99](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/webSearch.ts#L87-L99); [src/tools/webFetch.ts:86-97](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/webFetch.ts#L86-L97) (verified)
- **Cap:** none

### C3 Tool & action scoping — 0.62 (high)

Each tool calls one fixed Exa endpoint through the official client, so the model cannot point the server at an arbitrary host or service, and the default tool set is two read-only lookups. Arguments are typed and validated with zod schemas, but the schemas are deliberately lenient and carry no upper bounds: the model chooses how many results, how many URLs and how many characters per page. The fetch tool accepts any URL, which Exa's crawler then retrieves on the caller's behalf.

- **S L2:** Typed zod schemas are enforced by the MCP SDK and endpoints are fixed, but numResults, maxCharacters and the URL list have no upper bounds and web_fetch_exa forwards any URL string. — [src/tools/webSearch.ts:39-41](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/webSearch.ts#L39-L41); [src/tools/webFetch.ts:56-60](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/webFetch.ts#L56-L60); [src/tools/validation.ts:18-20](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/validation.ts#L18-L20) (verified)
  - *To reach the next level:* No numeric maxima or URL allowlist enforced in code.
- **C L3:** Every tool declares its arguments through the same zod helpers and fixed-endpoint client; there are no extension tools. — searched `rg -n -F server.tool(` in `src` → 11 hits (all tool registrations use the SDK's zod-validated tool API); [src/tools/config.ts:38-44](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/config.ts#L38-L44) (verified)
  - *To reach the next level:* There is no central policy layer applying bounds that new tools inherit.
- **D L3:** The default tool set is web_search_exa and web_fetch_exa only; the agent and advanced tools need explicit selection. — [src/toolRegistry.ts:12-17](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/toolRegistry.ts#L12-L17); [src/toolRegistry.ts:18-24](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/toolRegistry.ts#L18-L24) (verified)
  - *To reach the next level:* Tool selection is per connection, not per task, and is not operator-only.
- **B L2:** A misused tool can make Exa fetch any public URL and spend the account's credits, but cannot touch any host or system other than Exa's API. — [src/tools/webFetch.ts:76-81](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/webFetch.ts#L76-L81) (verified)
  - *To reach the next level:* Quantities (results, URLs, characters) are not bounded by the server.
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

The server never interprets model text as code: there is no shell, eval, subprocess or script execution anywhere in its source. All work is HTTPS requests to Exa's API through the official client.

- **Structural absence:** searched `rg -n -S 'child_process|execSync|spawn\(|eval\(|new Function|vm\.run'` in `src api` → 0 hits (No code execution path.)

### C5 Untrusted input blast radius — 0.05 (high)

Search highlights and fetched page text are returned as flat text mixed with the server's own labels (Title, URL, Author), with no structured separation and no marker that the content is untrusted. Tool descriptions include usage directives to the model. The fetch tool lets a hijacked host model send data to any URL by encoding it in the address, which Exa's crawler then requests, so exfiltration needs no human step. The server itself can take no irreversible action.

- **S L0:** Third-party page text is concatenated with server-written labels in one text block, and tool descriptions carry directives ("follow up with web_fetch_exa"). — [src/tools/webSearch.ts:124-137](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/webSearch.ts#L124-L137); [src/tools/webFetch.ts:18-27](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/webFetch.ts#L18-L27); [src/tools/webSearch.ts:26-34](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/webSearch.ts#L26-L34) (verified)
  - *To reach the next level:* Return page content separately from metadata, with provenance the host can act on.
- **C L0:** No source is distinguished: search highlights, fetched pages and agent output all enter context the same way. — searched `rg -n -i 'untrusted|structuredContent'` in `src` → 0 hits (no provenance or structured output anywhere) (verified)
  - *To reach the next level:* At least the main content sources should carry an untrusted marker.
- **D L0:** No provenance or isolation mechanism exists to be on by default. — searched `rg -n -i 'untrusted|structuredContent'` in `src` → 0 hits (verified)
  - *To reach the next level:* A mechanism must exist before it can be on by default.
- **B L1:** A hijacked host can exfiltrate data unattended by having web_fetch_exa request an attacker URL with data in it; the server offers no irreversible action. — [src/tools/webFetch.ts:76-81](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/webFetch.ts#L76-L81) (verified)
  - *To reach the next level:* An outbound channel to arbitrary URLs remains available without approval.
- **Cap:** none

### C6 Memory, context & configuration integrity — 1.00 (high)

The server keeps no memory the model can write and loads no workspace files. Its only persistence is a 24-hour record of the connecting client's name and version, kept to label requests to Exa's API and never returned into model context. Its bundled agent guide is read from the package itself.

- **Structural absence:** searched `rg -n -S 'writeFile|appendFile|dotenv|AGENTS.md|CLAUDE.md'` in `src api` → 0 hits (No model-writable persistence or workspace config load.); searched `rg -n -S 'redisClient\.(set|get)\('` in `api` → 2 hits (Both are client-metadata storage for analytics headers (api/mcp.ts:87, 112), not model context.)

### C7 Third-party extensions — 1.00 (high)

The server loads no plugins, launches no other MCP servers and installs nothing at runtime; it uses only its own fixed npm dependencies. The skill files in the repo are read by host clients, not executed by the server.

- **Structural absence:** searched `rg -n -S 'npx|npm install|loadPlugin|child_process'` in `src api` → 0 hits (No runtime extension loading.); searched `rg -n -S 'import\(|require\('` in `src api` → 3 hits (two are the server's own Upstash dependencies (api/mcp.ts:146-147), one is a TypeScript type import)

### C8 Secrets & sensitive-data protection — 0.38 (high)

API keys come from headers, the URL or the environment and are never placed in tool results. Before each request is handed to the MCP layer and its third-party analytics wrapper, the server strips the x-api-key and Authorization headers and the exaApiKey query parameter. There is no general redaction or log filter, though: every search query and fetched URL is written to the server log by default, the README recommends putting the API key in the URL, and content-free usage analytics are sent to a third party (Agnost) by default.

- **S L1:** Secrets come from headers or env; one masking step removes credentials from the request before the MCP handler and analytics see it; there is no redaction helper or log filter. — [api/mcp.ts:815-830](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/api/mcp.ts#L815-L830) (verified)
  - *To reach the next level:* No type-level masking or log filter on the main logging paths.
- **C L2:** Credentials are kept out of analytics and model-bound tool results; error text returned to the model is the API's message only. — [api/mcp.ts:821-824](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/api/mcp.ts#L821-L824); [src/utils/errorHandler.ts:110-116](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/utils/errorHandler.ts#L110-L116) (verified)
  - *To reach the next level:* Logs carry no redaction step of their own.
- **D L1:** Agnost usage analytics are on by default and sent to api.agnost.ai with input, output and errors disabled; queries and URLs are logged on every call. — [src/mcp-handler.ts:275-285](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/mcp-handler.ts#L275-L285); [src/utils/logger.ts:14-16](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/utils/logger.ts#L14-L16) (verified)
  - *To reach the next level:* Telemetry should be opt-in and default logging should not record query content.
- **B L2:** A leaked key is a long-lived Exa API key scoped to one Exa account; OAuth access tokens are JWTs with expiry. — [src/utils/auth.ts:38-41](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/utils/auth.ts#L38-L41) (verified)
  - *To reach the next level:* API keys are not short-lived or task-scoped.
- **Cap:** none

### C9 Audit & traceability — 0.38 (high)

Every tool call writes a few unstructured lines to the server log with a request id, the tool name, the query or URLs, and whether it succeeded. That gives the operator a basic trail, but it records no caller identity, is plain console text rather than a structured record, and is best effort. Third-party analytics checkpoints are not an audit trail.

- **S L1:** createRequestLogger writes plain-text start/complete/error lines with a request id and tool name to stderr; no actor or structured fields. — [src/utils/logger.ts:4-26](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/utils/logger.ts#L4-L26) (verified)
  - *To reach the next level:* No structured per-call record with arguments, status and timestamps.
- **C L2:** All 11 tool handlers create a request logger and log start and outcome. — searched `rg -n -F createRequestLogger(` in `src/tools` → 11 hits; [src/utils/logger.ts:8-24](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/utils/logger.ts#L8-L24) (verified)
  - *To reach the next level:* Caller identity and auth method are not recorded on the per-call log lines.
- **D L2:** Logging is always on and goes to the hosting platform's logs, out of the model's reach. — [src/utils/logger.ts:4-6](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/utils/logger.ts#L4-L6) (verified)
  - *To reach the next level:* Not written by a separate component; the server process controls it.
- **B L1:** Logging is best-effort console output with no durability guarantees. — [src/utils/logger.ts:4-6](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/utils/logger.ts#L4-L6) (verified)
  - *To reach the next level:* Records are not flushed durably per action.
- **Cap:** none

### C10 Limits & kill switch — 0.53 (high)

Every tool call is wrapped in a server-side timeout (60 seconds for the default tools, 5 minutes for advanced search), retries are capped at two, and the agent tool's call window is clamped below the platform's 800-second function limit. Anonymous callers are rate limited per IP when the operator configures the rate-limit store, and the limiter fails open when that store is unavailable. Result counts and page sizes are chosen by the model with no upper bound, timeouts stop waiting without cancelling the upstream request, and callers with their own key have no spend limit from the server.

- **S L2:** Server-enforced timeouts on every tool call, bounded retries, and per-IP QPS/daily limits for anonymous callers. — [src/tools/config.ts:81-85](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/config.ts#L81-L85); [src/utils/errorHandler.ts:44-46](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/utils/errorHandler.ts#L44-L46); [api/mcp.ts:155-166](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/api/mcp.ts#L155-L166) (verified)
  - *To reach the next level:* No caps on output size or result counts.
- **C L3:** All tool paths run under withTimeout and the platform maxDuration; the agent call window is clamped to the function ceiling. — [src/tools/webFetch.ts:86-99](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/webFetch.ts#L86-L99); [src/tools/agentRun.ts:96-115](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/agentRun.ts#L96-L115); [vercel.json:3-5](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/vercel.json#L3-L5) (verified)
  - *To reach the next level:* No cap on concurrent calls per caller and no delegation-style limits on agent runs.
- **D L1:** Defaults are sensible (10 results, 3000 characters) but the model can raise numResults and maxCharacters freely, and rate limiting is off unless the store is configured and fails open on errors. — [src/tools/config.ts:86-87](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/tools/config.ts#L86-L87); [api/mcp.ts:352-358](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/api/mcp.ts#L352-L358) (verified)
  - *To reach the next level:* The model should not be able to raise the server's own limits.
- **B L2:** A runaway call is bounded at 60s (300s advanced, about 750s agent) but withTimeout only stops waiting, and keyed callers have no server-side spend ceiling. — [src/utils/errorHandler.ts:48-70](https://github.com/exa-labs/exa-mcp-server/blob/f3d71fb6b0ff4b4683f108f05bc2bae61a9f7e97/src/utils/errorHandler.ts#L48-L70) (verified)
  - *To reach the next level:* In-flight upstream requests are not cancelled when the timeout fires.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Search highlights and fetched pages returned as plain text (src/tools/webSearch.ts:124-137, src/tools/webFetch.ts:18-27) · [B] sensitive data/systems: Server holds only the caller's or operator's Exa credential, never exposed to the model (api/mcp.ts:815-830); sensitive data comes from the host session · [C] state change / egress: web_fetch_exa has Exa fetch any model-chosen URL (src/tools/webFetch.ts:76-81) · Same default session? Yes

## Highest-impact improvements
1. Return results as structured content with source URL and an untrusted flag, kept separate from server labels, and drop model directives from tool descriptions. — C5 S L0→L2, +0.150 before caps (Playbook 1)
2. Fix the agent_run and web_search_advanced_exa annotations so every tool's hints are accurate. — C2 S L1→L2, +0.075 before caps (Playbook 5)
3. Emit one structured JSON record per tool call with tool, arguments, status, auth method and timestamp. — C9 S L1→L2, +0.075 before caps (Playbook 1 step 3)
4. Enforce server-side maxima on numResults, URL count and maxCharacters. — C10 D L1→L2, +0.050 before caps (Playbook 3 step 3)
5. Make third-party usage analytics opt-in and stop logging raw queries and URLs by default. — C8 D L1→L2, +0.050 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the hosted Streamable-HTTP server (api/mcp.ts) that the README leads with; the local stdio package (src/stdio.ts) and the AgentCore container (src/runtime-server.ts, which relies on the hosting platform for caller authentication) share the same tools but were not scored separately.
- Production Vercel environment values (rate-limit store, OAuth issuer, ENABLED_TOOLS) are not in the repository; defaults in code were scored.
- No release tag points at the pinned commit; package.json says 3.4.1, not used as the version.
- Exa's backend behaviour (crawler URL handling, account-level spend limits, agent-run retention, Exa Connect providers) and the Agnost analytics SDK internals were not examined.
- No reviewer-steering text was found in the repository.
