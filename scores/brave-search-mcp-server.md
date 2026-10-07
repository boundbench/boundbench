# Defense-in-Depth Score: Brave Search MCP Server

**Repo:** https://github.com/brave/brave-search-mcp-server · **Commit:** `a75d7eff34c260c7bba3bdc0163fd6982c34e370` (v2.1.4-1-ga75d7ef) · **Reviewed:** 2026-10-04
**What it is:** Official MCP server exposing Brave Search API endpoints (web, news, images, video, local, place, summarizer, LLM context) as tools.
**Category:** AI Assistants
**Scored configuration:** STDIO transport (the default), all eight tools enabled, Brave API key supplied via BRAVE_API_KEY, as in the README's npx/docker host configs.
**Agent surface (default):** code execution no · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 4.9 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L3 | L3 | 0.60 | — | **0.60** | High |
| C2 | Approval gates | L1 | L2 | L2 | L4 | 0.53 | — | **0.53** | High |
| C3 | Tool & action scoping | L3 | L3 | L3 | L3 | 0.75 | — | **0.75** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | Medium |
| C6 | Memory, context & configuration integrity | L0 | L0 | L0 | L2 | 0.10 | C6-REPOCONFIG | **0.10** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L2 | 0.35 | — | **0.35** | High |
| C9 | Audit & traceability | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C10 | Limits & kill switch | L2 | L1 | L3 | L2 | 0.47 | — | **0.47** | High |

Controls where a risk surface exists: 2.85 / 8.0 (36%); 2 criteria scored SA (surface absent).

A small, read-only search server: it cannot execute code, write files or change anything, and its only credential is a search-only Brave API key sent to a hard-coded host. The dominant risk is the untrusted web content it feeds the model, which it returns without any untrusted marking while its image tool even instructs the model to embed remote images. It also silently loads a .env from its working directory, letting a cloned repo swap the API key or expose the server over unauthenticated HTTP, and it keeps no record of what it did.

## Critical gaps
- A .env file in the server's working directory is auto-loaded and can redirect the Brave API key (via BRAVE_API_KEY_FILE, which overrides BRAVE_API_KEY) or expose the server over unauthenticated HTTP on all interfaces. (ASI06, ASI04; C6) — [src/config.ts:7](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/config.ts#L7); [src/config.ts:144-152](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/config.ts#L144-L152)

## Criterion details

### C1 Identity & least privilege — 0.60 (high)

The server holds exactly one credential, a Brave Search API key read from the environment, a CLI argument or a key file, and attaches it only to requests to the hard-coded Brave Search API host. That key can only run searches, so a hijacked model can spend quota but cannot reach other systems. There is no per-tool or per-request scoping and no authorization layer; in the optional HTTP mode every client shares the operator's key with no authentication. No subprocesses are spawned, so the key is not passed on.

- **S L2:** A single dedicated Brave Search API key, fixed for the whole run, is used for every tool. — [src/config.ts:36](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/config.ts#L36); [src/BraveAPI/index.ts:21](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/BraveAPI/index.ts#L21) (verified)
  - *To reach the next level:* No per-tool or per-capability credential scoping (one key serves all eight tools).
- **C L2:** Every tool reaches the network only through API.issueRequest, which attaches the same key and targets the fixed Brave host; nothing else uses credentials. — [src/BraveAPI/index.ts:21](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/BraveAPI/index.ts#L21); [src/BraveAPI/index.ts:52](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/BraveAPI/index.ts#L52) (verified)
  - *To reach the next level:* No authorization layer in code that each tool path traverses; access is simply 'whoever can call the server'.
- **D L3:** By default the server holds only a search-only API key and no write-capable credential exists to elevate to. — [src/config.ts:36](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/config.ts#L36); [src/config.ts:58](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/config.ts#L58) (verified)
  - *To reach the next level:* A workspace .env (auto-loaded) can replace the key via BRAVE_API_KEY_FILE, so the identity is not tamper-proof against workspace input.
- **B L3:** A stolen or misused key allows read-only searches against one provider account (quota/billing abuse). — [src/BraveAPI/index.ts:52](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/BraveAPI/index.ts#L52); [src/BraveAPI/index.ts:21](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/BraveAPI/index.ts#L21) (verified)
  - *To reach the next level:* The key is long-lived and not rotated or short-lived by the server.
- **Cap:** none

### C2 Approval gates — 0.53 (high)

Every tool is a read-only search against Brave's API; no tool writes files, sends messages or changes state, so a wrongly approved call cannot change anything. However the tools advertise only a title and an 'open world' hint and never declare themselves read-only, so hosts get no machine-readable risk signal and will typically treat them as potentially mutating. There is no server-side read-only mode or confirmation step, but none is needed for the shipped tool set.

- **S L1:** Tools carry annotations (title, openWorldHint) but no readOnlyHint/destructiveHint on any tool. — [src/tools/web/index.ts:21-24](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/tools/web/index.ts#L21-L24); searched `rg -n 'readOnlyHint|destructiveHint'` in `src` → 0 hits (verified)
  - *To reach the next level:* readOnlyHint is not set on the read-only tools, so annotations are not accurate on every tool.
- **C L2:** All eight tools are registered through the same pattern with identical annotations; none has a consequential action. — [src/server.ts:23-25](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/server.ts#L23-L25) (verified)
  - *To reach the next level:* No accurate risk annotation on every tool path (readOnlyHint missing everywhere).
- **D L2:** There is nothing to switch off: no tool has a write path; operators can only narrow the tool list. — [src/config.ts:46-50](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/config.ts#L46-L50) (verified)
  - *To reach the next level:* No server-enforced read-only mode or confirmation the host must complete (signalled declaratively).
- **B L4:** In the default configuration no tool can change state: every handler only performs a GET to the fixed Brave Search API host. — [src/BraveAPI/index.ts:52](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/BraveAPI/index.ts#L52); [src/BraveAPI/index.ts:116](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/BraveAPI/index.ts#L116) (verified)
- **Cap:** none

### C3 Tool & action scoping — 0.75 (high)

Each tool is narrow (a specific Brave Search endpoint) and the destination host is hard-coded, so the model cannot point the server at arbitrary URLs or local services. Inputs are validated with typed schemas: query length and word count limits, enumerated countries and languages, and numeric bounds on result counts and token budgets. The weak spot is the 'goggles' parameter, which accepts any HTTPS URL or free-text definition that Brave's backend then fetches, and a few free-text header fields. All tools are enabled by default, but all are read-only and operators can restrict the list.

- **S L3:** Fixed host plus zod schemas with enums and numeric bounds (query max 400 chars/50 words, count 1-20). — [src/BraveAPI/index.ts:52](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/BraveAPI/index.ts#L52); [src/tools/web/params.ts:4-8](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/tools/web/params.ts#L4-L8); [src/tools/web/params.ts:158-162](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/tools/web/params.ts#L158-L162); [src/BraveAPI/index.ts:25-29](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/BraveAPI/index.ts#L25-L29) (verified)
  - *To reach the next level:* Goggles accept any https URL or arbitrary definition with no host allowlist, so not every argument is allowlisted.
- **C L3:** Every tool declares an inputSchema enforced by the MCP SDK and funnels through one request builder with a fixed host. — [src/server.ts:23-26](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/server.ts#L23-L26); [src/tools/summarizer/params.ts:4-6](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/tools/summarizer/params.ts#L4-L6) (verified)
  - *To reach the next level:* Validation is per-tool schemas rather than one central policy layer new tools inherit automatically (e.g., summarizer key is an unbounded string).
- **D L3:** The default tool set is entirely read-only search; tools are fixed at startup (listChanged false) and operators can allowlist via --enabled-tools. — [src/server.ts:17](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/server.ts#L17); [src/config.ts:66-70](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/config.ts#L66-L70) (verified)
  - *To reach the next level:* No per-task tool allowlist; all eight tools are exposed by default.
- **B L3:** A misused tool can only run bounded read-only searches against one API (counts capped per call). — [src/BraveAPI/index.ts:49](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/BraveAPI/index.ts#L49); [src/tools/web/params.ts:158-162](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/tools/web/params.ts#L158-L162) (verified)
  - *To reach the next level:* No rate limit or call-count bound, so quota spend is unbounded across calls.
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

The server never executes model-supplied or workspace-supplied text as code: there are no shell, eval, child-process or template-execution paths in the shipped source (the only child_process use is in an HTTP test). This risk surface is structurally absent.

- **Structural absence:** searched `rg -n -S 'child_process|spawn|execSync|exec\(|eval\(|new Function|vm\.'` in `src` → 2 hits (Both hits are in src/protocols/http.test.ts (test harness spawning the built server), not shipped runtime code.)

### C5 Untrusted input blast radius — 0.05 (medium)

The server's whole job is to pull untrusted web content (search snippets and, via the LLM-context tool, full page text) into the model's context. Results come back as JSON with source URLs, and some tools add structured output, but nothing marks content as untrusted, and the image tool's description actively tells the model to embed remote image URLs in markdown, an automatic-fetch channel that injected content can abuse. The server holds no private data, but a hijacked model can still push data out through model-chosen query text and goggle URLs that Brave's backend fetches. There is no read-only/no-egress mode to drop a Rule-of-Two leg.

- **S L0:** A tool description contains a directive to the model to render remote images in markdown, and no untrusted-content flag is attached to outputs. — [src/tools/images/index.ts:17](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/tools/images/index.ts#L17); [src/tools/web/index.ts:67-71](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/tools/web/index.ts#L67-L71) (verified)
  - *To reach the next level:* Remove model directives from descriptions and return structured outputs that separate content from metadata on every tool.
- **C L0:** No source is distinguished as untrusted; web, news, video, LLM-context and place results all return as plain text content. — [src/tools/llm_context/index.ts:41-45](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/tools/llm_context/index.ts#L41-L45) (verified)
  - *To reach the next level:* No untrusted/provenance marking applied to any source.
- **D L0:** No untrusted-input control exists to be on by default. — [src/tools/images/index.ts:17](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/tools/images/index.ts#L17) (verified)
  - *To reach the next level:* No default-on provenance or untrusted flag.
- **B L1:** No irreversible actions are reachable, but model-chosen query strings and https goggle URLs leave the machine unattended, and the image directive invites auto-loaded remote images, giving an exfiltration channel. — [src/BraveAPI/index.ts:91-100](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/BraveAPI/index.ts#L91-L100); [src/tools/images/index.ts:17](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/tools/images/index.ts#L17) (inferred)
  - *To reach the next level:* An outbound channel for model-chosen data remains (goggle URLs fetched by Brave; markdown image directive).
- **Cap:** none
- **Notes:** That Brave's backend fetches hosted goggle URLs is inferred from the parameter description; not verified against Brave's service.

### C6 Memory, context & configuration integrity — 0.10 (high)

The server has no memory or retrieval store, but at startup it silently loads a .env file from its current working directory, which for MCP hosts is often the user's project or a cloned repository. Such a file can swap the operator's API key for one named in a key file, switch the server to HTTP on all interfaces with no authentication, or change the tool list, all without any trust prompt. Process environment variables set by the host still win over .env values, but most of these settings are usually unset.

- **S L0:** dotenv.config() auto-loads .env from the working directory and its values feed every security-relevant option (key file, transport, host, tool lists). — [src/config.ts:7](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/config.ts#L7); [src/config.ts:58](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/config.ts#L58); [src/config.ts:64](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/config.ts#L64); [src/config.ts:84](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/config.ts#L84) (verified)
  - *To reach the next level:* Security-relevant settings should come only from the host-provided environment or explicit flags, or require a trust decision.
- **C L0:** The single auto-loaded config path is uncontrolled. — [src/config.ts:7](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/config.ts#L7) (verified)
  - *To reach the next level:* No control on the .env load path at all.
- **D L0:** The .env load is on by default with no option to disable it, and a key file from it overrides the operator's BRAVE_API_KEY. — [src/config.ts:7](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/config.ts#L7); [src/config.ts:144-152](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/config.ts#L144-L152) (verified)
  - *To reach the next level:* No isolation between workspace files and server configuration.
- **B L2:** A poisoned .env persists for every launch from that directory and can redirect the credential or expose an unauthenticated HTTP endpoint, but cannot directly trigger tool use. — [src/config.ts:144-152](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/config.ts#L144-L152); [src/protocols/http.ts:102](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/protocols/http.ts#L102) (verified)
  - *To reach the next level:* Effect persists across sessions and is not surfaced to the user for inspection.
- **Cap:** C6-REPOCONFIG — dotenv.config() at src/config.ts:7 auto-loads a workspace .env that can replace the API key (BRAVE_API_KEY_FILE) or switch to HTTP on 0.0.0.0, with no trust decision (dotenv reads process.cwd()/.env and does not override already-set variables).

### C7 Third-party extensions — 1.00 (high)

The server loads no plugins, MCP servers, remote code or model files at runtime; its tool set is fixed at build time. The README's 'npx -y' install line concerns how the server itself is installed (the project's own supply chain), which is out of scope for this criterion.

- **Structural absence:** searched `rg -n -S 'import\(|require\(|plugin|loadExtension'` in `src` → 0 hits

### C8 Secrets & sensitive-data protection — 0.35 (high)

The API key comes from an environment variable, a key file, or a command-line flag (the last is visible in process listings). It is never placed in tool output or model context and is not logged, and there is no telemetry. But there is no explicit redaction anywhere: upstream error bodies are passed back verbatim, and protection relies on the key simply not being on those paths. The key is long-lived but limited to search.

- **S L1:** Key sourced from env/file/argv and kept in a module-level state object; no masking or redaction helper exists. — [src/config.ts:36](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/config.ts#L36); [src/config.ts:54](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/config.ts#L54); [src/utils.ts:11](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/utils.ts#L11) (verified)
  - *To reach the next level:* No type-level masking or log/error redaction filters.
- **C L1:** The key is only sent in the request header to Brave; error paths forward Brave's response body unfiltered to the caller. — [src/BraveAPI/index.ts:21](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/BraveAPI/index.ts#L21); [src/BraveAPI/index.ts:119-130](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/BraveAPI/index.ts#L119-L130) (verified)
  - *To reach the next level:* No explicit protection on logs, error messages and model-bound output paths.
- **D L2:** No telemetry SDK and only minimal console error logging by default. — searched `rg -n -S 'sentry|posthog|telemetry|analytics|segment'` in `src package.json` → 0 hits; [src/protocols/http.ts:81](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/protocols/http.ts#L81) (verified)
  - *To reach the next level:* No always-on redaction.
- **B L2:** A leaked key is long-lived but scoped to Brave Search usage on one account. — [src/BraveAPI/index.ts:21](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/BraveAPI/index.ts#L21) (verified)
  - *To reach the next level:* Key is not short-lived or rotated by the server.
- **Cap:** none

### C9 Audit & traceability — 0.00 (high)

The server keeps no record of which tools were called, with what arguments, or what was returned. It declares the MCP logging capability but never emits log messages, and the only console output is startup errors and HTTP-mode exceptions. After an incident, the server itself offers nothing to reconstruct what happened; that would have to come from the host or Brave's API dashboard.

- **S L0:** No tool-call logging; console output is limited to configuration and HTTP errors. — searched `rg -n 'sendLoggingMessage|logger|audit'` in `src` → 0 hits; [src/server.ts:16](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/server.ts#L16) (verified)
  - *To reach the next level:* No structured record of tool calls.
- **C L0:** Nothing is recorded for any tool. — [src/tools/web/index.ts:41-45](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/tools/web/index.ts#L41-L45) (verified)
  - *To reach the next level:* No tool call is recorded.
- **D L0:** No audit log exists, on or off. — [src/config.ts:60](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/config.ts#L60) (verified)
  - *To reach the next level:* No default-on record.
- **B L0:** Actions proceed with no record at all. — [src/BraveAPI/index.ts:116](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/BraveAPI/index.ts#L116) (verified)
  - *To reach the next level:* No record is written per action.
- **Cap:** none

### C10 Limits & kill switch — 0.47 (high)

Each call's size is bounded by schema limits that the model cannot raise (result counts, query length, LLM-context token budgets), and the summarizer stops polling after 20 attempts. There is no rate limiting (a TODO in the code), no timeout on the outbound HTTP request, and no cancellation of in-flight requests. Spend ceilings depend on the Brave plan's own quota rather than anything the server enforces.

- **S L2:** Server-enforced caps on output size per call and a bounded summarizer polling loop. — [src/tools/summarizer/index.ts:86](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/tools/summarizer/index.ts#L86); [src/BraveAPI/index.ts:49](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/BraveAPI/index.ts#L49) (verified)
  - *To reach the next level:* No concurrency or rate limits (left as a TODO).
- **C L1:** Per-call size caps apply to every tool, but no request has a timeout. — [src/BraveAPI/index.ts:116](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/BraveAPI/index.ts#L116) (verified)
  - *To reach the next level:* No per-request timeout on the outbound fetch.
- **D L3:** Caps are hard-coded in schemas; the model cannot raise them and configuration does not expose them. — [src/tools/web/params.ts:158-162](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/tools/web/params.ts#L158-L162) (verified)
  - *To reach the next level:* No rate-limit defaults exist to be tamper-resistant; L4 needs L3 strength first.
- **B L2:** Each call is bounded, but call volume is unbounded and in-flight fetches run to completion; provider-side quotas are the only spend ceiling (not enforced by the server). — [src/BraveAPI/index.ts:116](https://github.com/brave/brave-search-mcp-server/blob/a75d7eff34c260c7bba3bdc0163fd6982c34e370/src/BraveAPI/index.ts#L116) (verified)
  - *To reach the next level:* No server-side cancellation of pending calls or spend ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Search results and page text from the web (src/tools/llm_context/index.ts:41-45, src/tools/web/index.ts:67-71) · [B] sensitive data/systems: Only the Brave API key, which is never returned to the model (src/BraveAPI/index.ts:21) · [C] state change / egress: Model-chosen query text and https goggle URLs sent to Brave (src/BraveAPI/index.ts:91-100); markdown image directive (src/tools/images/index.ts:17) · Same default session? Yes

## Highest-impact improvements
1. Stop auto-loading .env from the working directory (or load only from an explicit --env-file path). — C6 S L0→L3, +0.225 before caps (Playbook 2)
2. Add readOnlyHint: true (and idempotentHint) to every tool's annotations. — C2 S L1→L2, +0.075 before caps (Playbook 5)
3. Remove the markdown-image directive from the image tool description and add provenance/untrusted markers in structuredContent for every tool. — C5 S L0→L2, +0.150 before caps (Playbook 1)
4. Emit an MCP log / stderr record per tool call (tool, arguments, status, timestamp). — C9 S L0→L2, +0.150 before caps (Playbook 1 step 3)
5. Add an AbortSignal timeout to the outbound fetch and implement the TODO rate limiter. — C10 C L1→L2, +0.075 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Pinned commit is one commit past release tag v2.1.4 (git describe v2.1.4-1-ga75d7ef).
- Scored the STDIO default; the HTTP mode (no authentication, all clients share the operator key, Host check opt-in, Docker image binds 0.0.0.0) was reviewed only as a footnote.
- Behaviour of Brave's backend (goggle URL fetching, quotas) and of the dotenv/MCP SDK libraries is inferred, not verified.
- No text aimed at AI reviewers was found in the repository.
