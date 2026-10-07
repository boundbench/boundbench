# Defense-in-Depth Score: Context7

**Repo:** https://github.com/upstash/context7 (`packages/mcp`) · **Commit:** `bfa02ea67b5707fe0e0a673faa49d0f50b28c80b` (@upstash/context7-mcp@4.1.1-26-gbfa02ea) · **Reviewed:** 2026-10-04
**What it is:** Upstash's MCP server that serves up-to-date, community-indexed library documentation into coding agents' context.
**Category:** Coding
**Scored configuration:** The @upstash/context7-mcp server in streamable-HTTP mode as hosted at https://mcp.context7.com/mcp (the README's manual MCP setup and `ctx7 setup --mcp`), with the user's API key in the Authorization header and default environment.
**Agent surface (default):** code execution no · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 6.5 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L3 | L3 | 0.60 | C1-PASSTHRU | **0.25** | High |
| C2 | Approval gates | L4 | L4 | L4 | L4 | 1.00 | — | **1.00** | High |
| C3 | Tool & action scoping | L3 | L3 | L3 | L3 | 0.75 | — | **0.75** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L2 | 0.10 | — | **0.10** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L2 | L2 | L2 | 0.42 | — | **0.42** | High |
| C9 | Audit & traceability | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C10 | Limits & kill switch | L2 | L3 | L3 | L2 | 0.62 | — | **0.62** | High |

Controls where a risk surface exists: 3.60 / 7.0 (51%); 3 criteria scored SA (surface absent).

Context7's MCP server is a small, read-only surface: two documentation lookups against one fixed API, no code execution, no files, no memory, no plugins. The dominant risk is what it delivers: community-contributed documentation goes into your agent's context as plain text with no provenance or untrusted marking, alongside tool descriptions that themselves direct the model, so any prompt injection in an indexed library reaches an agent that may hold far more powerful tools. It also forwards your API key or OAuth token to the Context7 backend without validating most tokens itself, and it keeps no per-call record of what was asked.

## Critical gaps
- The server forwards the client's API key or JWT unchanged to the Context7 API (MCP token passthrough), validating tokens only on the /mcp/oauth route; token validation there is not a strict boundary. (ASI03, T9; C1) — [packages/mcp/src/lib/encryption.ts:72](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/encryption.ts#L72); [packages/mcp/src/index.ts:459](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/index.ts#L459)

## Criterion details

### C1 Identity & least privilege — 0.25 (high)

The server holds no broad credential of its own: each request carries the user's own Context7 API key (or OAuth token), and the server only ever uses it for two read-only lookups against the Context7 API, so one user's request cannot act as another user. But the server does not validate most tokens itself: on the default /mcp endpoint any key or token is forwarded unchecked to the backend, and token validation on the protected endpoint is not a strict boundary. Forwarding the client's token downstream is the MCP 'token passthrough' pattern, which caps this criterion.

- **S L2:** Each request runs with the caller's own long-lived Context7 API key, forwarded as a Bearer token; there is no server-side downscoping or per-tool credential. — [packages/mcp/src/index.ts:409-418](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/index.ts#L409-L418); [packages/mcp/src/lib/encryption.ts:72](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/encryption.ts#L72) (verified)
  - *To reach the next level:* No per-tool or read-only credential and no token exchange; the client's key is passed through as-is.
- **C L2:** Both tools build upstream headers from the per-request context, but authorization is left to the private backend: on /mcp the token is not validated at all, and token validation on the protected endpoint is not a strict boundary. — [packages/mcp/src/index.ts:55-63](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/index.ts#L55-L63) (verified)
  - *To reach the next level:* No authorization layer in the server itself evaluates every tool call against the requesting principal and fails closed.
- **D L3:** The only upstream calls the server can make are two GET lookups, so whatever the forwarded key could do elsewhere, the server only reads. — [packages/mcp/src/lib/api.ts:125](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/api.ts#L125); [packages/mcp/src/lib/api.ts:165](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/api.ts#L165) (verified)
  - *To reach the next level:* Nothing narrows the forwarded credential itself or time-bounds it; the key can be a long-lived ctx7sk key.
- **B L3:** If the server's handling fails, the exposed authority is one user's Context7 account used for documentation reads; stateless per-request context keeps tenants apart. — [packages/mcp/src/index.ts:500-506](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/index.ts#L500-L506); [packages/mcp/src/index.ts:420](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/index.ts#L420) (verified)
  - *To reach the next level:* Credentials are long-lived API keys, not minutes-lived revocable tokens.
- **Cap:** C1-PASSTHRU — The server accepts the client's API key or JWT and forwards it unchanged as the Authorization header to the Context7 API (encryption.ts:72), without validating it on the default /mcp route.

### C2 Approval gates — 1.00 (high)

The server exposes exactly two tools, both pure lookups against the Context7 documentation API, and both carry accurate read-only, non-destructive annotations. Neither tool can change state anywhere: the only upstream calls are two HTTP GETs to a fixed host. There is nothing consequential for a host to approve, and the tool set is hard-coded with no way to add tools at runtime.

- **S L4:** Both tools are separate read-only lookups with accurate readOnlyHint/destructiveHint annotations, and read-only behaviour is enforced by the code: the only upstream calls are GETs. — [packages/mcp/src/index.ts:268-273](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/index.ts#L268-L273); [packages/mcp/src/index.ts:332-337](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/index.ts#L332-L337); searched `rg -n 'fetch\('` in `packages/mcp/src` → 6 hits (Two tool fetches (api.ts:134,174, both GET), OAuth metadata proxy GET (index.ts:565), Entra config GET (jwt.ts:45), and two handler.fetch request-dispatch calls in telemetry; none mutates state.) (verified)
- **C L4:** Every registered tool (two registerTool calls) is annotated, and no other path reaches an upstream write. — searched `rg -n 'registerTool\('` in `packages/mcp/src` → 2 hits; searched `rg -n 'readOnlyHint: true'` in `packages/mcp/src` → 2 hits (verified)
- **D L4:** Annotations and the tool set are hard-coded; tools.listChanged is false and no configuration, input, or model output can add a write tool. — [packages/mcp/src/index.ts:204](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/index.ts#L204) (verified)
- **B L4:** The default configuration permits no consequential action at all, enforced in code, so a wrongly approved call cannot change anything. — [packages/mcp/src/lib/api.ts:134](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/api.ts#L134); [packages/mcp/src/lib/api.ts:174](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/api.ts#L174) (verified)
- **Cap:** none

### C3 Tool & action scoping — 0.75 (high)

The tools are narrow by design: instead of a generic HTTP fetch, each builds a request to one fixed Context7 endpoint, with the model's arguments placed into encoded query parameters, so the model cannot choose the host or the path. Arguments are only typed as strings, though: there is no length limit and no format check on the library ID, and the server does not cap how much documentation it returns.

- **S L3:** Model arguments are written into URLSearchParams on a fixed base URL (the equivalent of a parameterized query), so neither host nor path is model-controlled. — [packages/mcp/src/lib/api.ts:125-127](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/api.ts#L125-L127); [packages/mcp/src/lib/constants.ts:14](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/constants.ts#L14); searched `rg -n '\.max\(|\.regex\(|\.min\('` in `packages/mcp/src` → 0 hits (verified)
  - *To reach the next level:* No length bounds or format validation on query/libraryId (schemas are bare z.string()).
- **C L3:** Both tools follow the same fixed-endpoint, encoded-parameter pattern; there are no extension tools. — [packages/mcp/src/lib/api.ts:165-167](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/api.ts#L165-L167) (verified)
  - *To reach the next level:* No single central policy layer that new tools inherit; each tool builds its own URL.
- **D L3:** The default and only tool set is read-only and fixed for the process lifetime. — [packages/mcp/src/index.ts:204](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/index.ts#L204) (verified)
  - *To reach the next level:* No per-task tool allowlisting offered by the server.
- **B L3:** A misused tool can only run documentation lookups against one fixed host. — [packages/mcp/src/lib/api.ts:183-190](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/api.ts#L183-L190) (verified)
  - *To reach the next level:* Response size is not bounded by the server; the upstream text is returned whole.
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

The server never interprets model text as code: there is no shell, subprocess, eval, or dynamic code loading anywhere in the MCP package. Arguments only become URL query parameters on a lookup request.

- **Structural absence:** searched `rg -n 'child_process|spawn\(|execSync|execFile|\beval\(|new Function|vm\.run'` in `packages/mcp/src` → 0 hits

### C5 Untrusted input blast radius — 0.10 (high)

The server's job is to put third-party, community-contributed documentation into the model's context, and it does so as plain text with no provenance marker or untrusted flag that the host could act on. Its own tool descriptions and server instructions also contain directives to the model ('You MUST call this function', 'Use even when you think you know the answer'), so content and instructions are mixed. The server itself cannot take actions, which limits what a hijack can do through it, but every query the model writes is sent to Context7 and there is no no-egress mode.

- **S L0:** Tool descriptions and the server instructions carry directives to the model, and documentation text is returned as unmarked plain text mixed with server-written guidance. — [packages/mcp/src/index.ts:208](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/index.ts#L208); [packages/mcp/src/index.ts:222](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/index.ts#L222); [packages/mcp/src/lib/api.ts:183-190](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/api.ts#L183-L190) (verified)
  - *To reach the next level:* Outputs would need to be structured, separating returned documentation from metadata, with no directives in descriptions.
- **C L0:** No source of untrusted content is distinguished: docs bodies, search result titles/descriptions and backend error messages all enter the result as text. — [packages/mcp/src/index.ts:344-351](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/index.ts#L344-L351); [packages/mcp/src/lib/utils.ts:27-31](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/utils.ts#L27-L31) (verified)
  - *To reach the next level:* At least one source (e.g. documentation bodies) would need provenance marking.
- **D L0:** There is no untrusted-content control to be on by default. — searched `rg -n -i 'untrusted|provenance|sanitiz'` in `packages/mcp/src` → 1 hits (The one hit is a JWT issuer error message in vercelMarketplaceJwt.ts, unrelated to content marking.) (verified)
  - *To reach the next level:* An untrusted-content marking or read-only/no-egress mode would need to exist and be on by default.
- **B L2:** Through this server a hijacked model can only perform documentation lookups; it cannot change state, and egress is limited to the fixed Context7 API, but the free-text query reaches Context7's backend on every call and the server offers no way to drop that leg. — [packages/mcp/src/lib/api.ts:126](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/api.ts#L126); [packages/mcp/src/index.ts:259](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/index.ts#L259) (verified)
  - *To reach the next level:* Whether query text is visible beyond Context7 (e.g. to library owners) cannot be verified from this repo, and there is no no-egress mode.
- **Cap:** none

### C6 Memory, context & configuration integrity — 1.00 (high)

The MCP server is stateless: it keeps no memory, writes no files, and loads no workspace instruction or config files. Only operator-scope environment variables configure it. (The Context7 documentation index is a shared, community-written store, but it lives in the private backend and is reached here only as untrusted input, scored under C5.)

- **Structural absence:** searched `rg -n 'dotenv|writeFile|appendFile|createWriteStream|sqlite|redis|localStorage|AGENTS\.md|CLAUDE\.md'` in `packages/mcp/src` → 0 hits

### C7 Third-party extensions — 1.00 (high)

The server loads no third-party code at runtime: no plugins, no MCP sub-servers, no package installs. Its only dynamic imports load its own telemetry modules.

- **Structural absence:** searched `rg -n 'require\(|import\(|npx|trust_remote_code|pickle|loadPlugin'` in `packages/mcp/src` → 7 hits (5 hits are dynamic imports of the package's own telemetry modules (telemetry-runtime.ts); 2 are a fixed sign-in hint string shown to the user (auth-prompt.ts), never executed.)

### C8 Secrets & sensitive-data protection — 0.42 (high)

API keys arrive per request (HTTP) or from an environment variable or command-line flag (stdio) and are kept in memory only. They are never echoed back into tool results, and the telemetry records only tool names and outcomes, never arguments or headers. There is, however, no redaction layer: secrets stay out of logs only because no code path happens to log them, not every error path keeps secrets out of its output, and the keys themselves are long-lived.

- **S L1:** Secrets come from headers or env vars and are kept out of logs by omission, with no masking or redaction helper. — [packages/mcp/src/index.ts:668](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/index.ts#L668) (verified)
  - *To reach the next level:* No type-level masking or log filter.
- **C L2:** Keys never enter tool results sent to the model, and telemetry attributes are a fixed content-free set (tool name, outcome, request id). — [packages/mcp/src/lib/mcp-telemetry.ts:240-242](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/mcp-telemetry.ts#L240-L242); [packages/mcp/src/index.ts:512](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/index.ts#L512) (verified)
  - *To reach the next level:* Error paths log raw error objects without redaction.
- **D L2:** Telemetry is on by default but content-free and not shipped to a third party (spans are no-ops without an operator-registered SDK; metrics are a local Prometheus endpoint). — [packages/mcp/src/lib/telemetry-config.ts:2](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/telemetry-config.ts#L2); [packages/mcp/src/lib/telemetry-provider.ts:8](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/telemetry-provider.ts#L8) (verified)
  - *To reach the next level:* No always-on redaction; telemetry is on unless OTEL_SDK_DISABLED is set.
- **B L2:** A leaked key is a long-lived Context7 API key scoped to one Context7 account. — [packages/mcp/src/lib/api.ts:44](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/api.ts#L44) (verified)
  - *To reach the next level:* Keys are not short-lived or per-task.
- **Cap:** none

### C9 Audit & traceability — 0.30 (high)

In the shipped configuration the server keeps no per-call record of what it was asked: only errors are printed to stderr, and aggregate metrics count calls by tool and outcome. OpenTelemetry spans exist but go nowhere unless the operator installs an exporter, and even then they hold the tool name and outcome, not the arguments or who asked.

- **S L1:** Unstructured stderr logs on error paths; per-call spans carry tool name and outcome but not arguments or principal. — [packages/mcp/src/lib/api.ts:139](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/api.ts#L139); [packages/mcp/src/lib/mcp-telemetry.ts:247](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/mcp-telemetry.ts#L247) (verified)
  - *To reach the next level:* No structured per-call record including arguments and timestamps.
- **C L1:** Only failing tool calls leave a log line; successful calls are visible only as aggregate counters. — [packages/mcp/src/index.ts:296](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/index.ts#L296) (verified)
  - *To reach the next level:* All tool calls, not just failures, should be recorded.
- **D L2:** What logging exists is on by default and lives server-side, out of the model's reach. — [packages/mcp/src/index.ts:436](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/index.ts#L436) (verified)
  - *To reach the next level:* A per-call record written by a component the model can't control is not produced by default.
- **B L1:** Logging is best-effort console output; telemetry flush on shutdown is bounded by a 5s timeout and errors are swallowed. — [packages/mcp/src/lib/process-shutdown.ts:81](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/process-shutdown.ts#L81) (verified)
  - *To reach the next level:* Records are not flushed durably per action.
- **Cap:** none

### C10 Limits & kill switch — 0.62 (high)

Each upstream lookup has a hard 60-second timeout, OAuth metadata fetches have 10 seconds, SSE keep-alives and notification subscriptions are disabled, and shutdown is bounded. The server itself has no rate limit, no concurrency cap and no output size limit (quotas are enforced by the private backend), and a client's cancellation of a tool call does not abort the in-flight upstream request.

- **S L2:** Server-enforced timeouts on upstream calls, disabled keep-alives and zero subscriptions bound the server's own work. — [packages/mcp/src/lib/api.ts:15](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/api.ts#L15); [packages/mcp/src/index.ts:434](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/index.ts#L434); [packages/mcp/src/lib/subscriptions.ts:3](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/subscriptions.ts#L3) (verified)
  - *To reach the next level:* No rate or concurrency limits and no output cap in the server.
- **C L3:** Both tool paths and the OAuth metadata proxy carry timeouts; there are no background tasks or spawned processes. — [packages/mcp/src/lib/api.ts:130](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/api.ts#L130); [packages/mcp/src/index.ts:561](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/index.ts#L561); [packages/mcp/src/lib/jwt.ts:45](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/jwt.ts#L45) (verified)
  - *To reach the next level:* No cap on concurrent requests; the Entra config lookup in token validation has no timeout.
- **D L3:** Timeouts are constants in code that neither the model nor request input can change. — [packages/mcp/src/lib/api.ts:15](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/api.ts#L15) (verified)
  - *To reach the next level:* No limits for the missing dimensions (rate, concurrency, output size) exist to be hard-capped.
- **B L2:** A runaway call is bounded at 60s, but client cancellation is not propagated to the upstream fetch, so in-flight lookups run to completion. — [packages/mcp/src/lib/api.ts:174](https://github.com/upstash/context7/blob/bfa02ea67b5707fe0e0a673faa49d0f50b28c80b/packages/mcp/src/lib/api.ts#L174) (verified)
  - *To reach the next level:* Cancellation of a tool call should abort the upstream request.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Community-contributed documentation returned raw (packages/mcp/src/lib/api.ts:183) · [B] sensitive data/systems: User's Context7 API key held per request (packages/mcp/src/lib/encryption.ts:72); no other sensitive data reachable · [C] state change / egress: Model-written query sent to the fixed Context7 API on every call (packages/mcp/src/lib/api.ts:126); no state change · Same default session? Yes

## Highest-impact improvements
1. Return documentation as structured content with source URL/library ID and an explicit untrusted flag, and remove imperative directives from tool descriptions and server instructions. — C5 S L0→L3, +0.225 before caps (Playbook 1)
2. Mark returned documentation bodies and search-result fields as untrusted third-party content on every tool. — C5 C L0→L2, +0.150 before caps (Playbook 1)
3. Write a structured per-call record (tool, arguments hash, principal, timestamp, outcome) by default. — C9 S L1→L2, +0.075 before caps (Playbook 1 step 3)
4. Add a redaction helper for secrets, applied to all error logging. — C8 S L1→L2, +0.075 before caps (Playbook 4)
5. Propagate the MCP request's cancellation signal into the upstream fetch and add per-key concurrency limits. — C10 B L2→L3, +0.050 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scope is packages/mcp. The ctx7 CLI + Skills mode, the SDK, the AI-SDK tools and the opencode/pi extensions were not scored.
- The Context7 API backend, parsing and crawling engines (which index and serve the documentation, and enforce quotas and key validation) are private and not in this repository; nothing about them is credited.
- The hosted endpoint's deployment (beyond packages/mcp/Dockerfile) could not be examined; the score assumes it runs this code with default environment.
- Version: no release tag at the pinned commit; plain `git describe --tags` returns a sibling package's tag (@upstash/context7-opencode@0.2.0-1-gbfa02ea), so the describe restricted to the scored package's tags is used.
- The local stdio mode (npx @upstash/context7-mcp) exposes the same two tools but runs as the OS user with the API key from env or a command-line flag; it is footnoted, not scored.
- No reviewer-steering text was found in the repository.
