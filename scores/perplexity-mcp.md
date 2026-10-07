# Defense-in-Depth Score: Perplexity MCP Server

**Repo:** https://github.com/perplexityai/modelcontextprotocol · **Commit:** `c58e4ad254608952606f09a40934ab6cca65bfad` · **Reviewed:** 2026-10-04
**What it is:** Perplexity's official MCP server exposing web search, Q&A, reasoning and deep-research tools backed by the Perplexity Search and Agent APIs.
**Category:** AI Assistants
**Scored configuration:** Local stdio server launched with `npx -y @perplexity-ai/mcp-server` and PERPLEXITY_API_KEY set, all other environment variables at their defaults.
**Agent surface (default):** code execution no · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 6.2 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | Medium |
| C2 | Approval gates | L2 | L3 | L3 | L2 | 0.62 | — | **0.62** | High |
| C3 | Tool & action scoping | L3 | L3 | L3 | L3 | 0.75 | — | **0.75** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | Medium |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L2 | L2 | L2 | 0.42 | — | **0.42** | Medium |
| C9 | Audit & traceability | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C10 | Limits & kill switch | L2 | L3 | L3 | L2 | 0.62 | — | **0.62** | High |

Controls where a risk surface exists: 3.22 / 7.0 (46%); 3 criteria scored SA (surface absent).

This is a small, read-only server: four tools that send a question to Perplexity and return web-derived text, with no shell, file access, memory or plugins, and the API key never leaves the request header. The main risks are what it can carry outward and what it costs: any text the host model puts in a question goes to Perplexity's hosted agent, which browses the web, so a prompt-injected host can use it to send data out, and returned web content arrives unlabelled. The server keeps no record of the queries it ran and has no rate or spend limit beyond a per-call timeout.

## Critical gaps
- None: no criterion is capped and no blast radius is at the worst level.

## Criterion details

### C1 Identity & least privilege — 0.50 (medium)

The server holds one credential, the operator's Perplexity API key, read from the environment and attached to every outbound call to the Perplexity API. It forwards no client tokens in the scored local mode and runs no subprocesses, so the key never reaches anything else. There is no narrower or per-tool credential and no authorization check of its own: every call, including deep research runs, bills the same long-lived account key. In the optional HTTP mode nothing authenticates inbound callers, so anyone who can reach the port spends the operator's key; the defaults keep that port on loopback.

- **S L2:** A single dedicated Perplexity API key from PERPLEXITY_API_KEY, scoped to one service, is used statically for the whole run. — [src/server.ts:21](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L21); [src/server.ts:111-117](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L111-L117) (verified)
  - *To reach the next level:* No per-tool or per-request narrowing of the key (no separate, cheaper or read-only credential, no short-lived tokens).
- **C L2:** Every tool reaches the API only through makeApiRequest, which attaches the same key; there are no subprocesses, extensions or sub-agents. — [src/server.ts:72-92](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L72-L92); [src/http.ts:127-135](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/http.ts#L127-L135) (verified)
  - *To reach the next level:* No authorization layer in code checks a request before the key is attached; the HTTP transport has no inbound caller authentication.
- **D L2:** Default use is the operator's full account key; nothing the model or a tool result can send changes which key is used. — [src/server.ts:21](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L21); [src/server.ts:82-92](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L82-L92) (verified)
  - *To reach the next level:* Default is not minimal: the full billing key is used for every call with no operator-level elevation step for expensive presets.
- **B L2:** If misused or stolen, the key can run unlimited Perplexity API requests billed to the operator's account; it reaches no other system (scope of a Perplexity key is inferred from the API, not code). — [src/server.ts:111-117](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L111-L117); [src/server.ts:22](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L22) (inferred)
  - *To reach the next level:* The credential is long-lived and not limited to read-only, quota-bounded use.
- **Cap:** none
- **Notes:** Library embedders can supply a per-call apiKey provider (src/types.ts) that fails closed when it returns nothing; this is not the scored default. PERPLEXITY_BASE_URL and proxy env vars can redirect the key to another host, but only via operator environment.

### C2 Approval gates — 0.62 (high)

The server's four tools only search the web or ask Perplexity's hosted agent; none writes files, sends messages or changes state. Every tool is marked read-only and non-destructive with an open-world flag, and those hints are accurate for local state. The one consequential effect is spending money on the operator's Perplexity account, which the annotations don't signal, and there is no dry-run, confirmation step or rate limit the host could use to slow repeated calls.

- **S L2:** All four tools carry hardcoded readOnlyHint true, destructiveHint false and openWorldHint true, which accurately describe tools with no local side effects. — [src/server.ts:604-609](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L604-L609); searched `rg -n 'readOnlyHint: true'` in `src/server.ts` → 4 hits (one per registered tool: perplexity_ask, perplexity_research, perplexity_reason, perplexity_search) (verified)
  - *To reach the next level:* No cost signalling, preview or server-enforced confirmation for expensive calls such as deep research runs.
- **C L3:** Every registered tool carries the same annotations and the tool set is fixed at registration; no tool reaches a mutating path. — searched `rg -n 'readOnlyHint: true'` in `src/server.ts` → 4 hits (one per registered tool: perplexity_ask, perplexity_research, perplexity_reason, perplexity_search); [src/server.ts:741-757](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L741-L757) (verified)
  - *To reach the next level:* No server-side rejection or confirmation hook applies uniformly to calls, so coverage of anything beyond hints can't rise.
- **D L3:** Annotations are literals in the registration code; no argument, env var or file can change them. — [src/server.ts:651-656](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L651-L656) (verified)
  - *To reach the next level:* Annotations are informational only and no approval-related control is enforced by the server itself.
- **B L2:** Nothing local can change, but a wrongly approved call irreversibly spends API credit and discloses host-provided text to Perplexity; per-call size (max_results 1-20, tokens per page 256-2048) and duration are bounded. — [src/server.ts:725-728](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L725-L728); [src/server.ts:429-431](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L429-L431) (verified)
  - *To reach the next level:* Outbound calls have no preview or dry-run, and there is no rate limit or spend cap on repeated calls.
- **Cap:** none

### C3 Tool & action scoping — 0.75 (high)

Each tool is a narrow wrapper around one fixed Perplexity endpoint; the model cannot choose the host, path or method, which come from code and operator environment. Arguments are typed with enums for filters and numeric bounds on result counts and page size, and message arrays are re-checked in code. Free-text fields (the question, domain filters, country) are passed through as data to Perplexity. The tool set is read-only and fixed, but all four tools are always exposed.

- **S L3:** Fixed endpoints built from an operator-set base URL, enums for recency/context/search type/role, and numeric bounds on max_results and max_tokens_per_page; messages are re-validated in code. — [src/server.ts:725-728](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L725-L728); [src/server.ts:561-575](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L561-L575); [src/server.ts:53-69](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L53-L69); [src/server.ts:97](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L97) (verified)
  - *To reach the next level:* Free-text fields such as search_domain_filter and country are not allowlisted, and the remote agent may fetch any URL named in a message.
- **C L3:** All four tools declare zod input schemas validated by the MCP SDK, and the three agent tools also call validateMessages. — [src/server.ts:618](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L618); [src/server.ts:723-735](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L723-L735) (verified)
  - *To reach the next level:* Validation is per-tool schema rather than a central policy layer new tools would inherit automatically.
- **D L3:** The default tool set is read-only (search and Q&A only) with no write, exec or local network tools, and the model cannot register new tools. — searched `rg -n 'readOnlyHint: true'` in `src/server.ts` → 4 hits (one per registered tool: perplexity_ask, perplexity_research, perplexity_reason, perplexity_search); [src/server.ts:592-603](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L592-L603) (verified)
  - *To reach the next level:* No per-task tool allowlist; all four tools, including the expensive research tool, are always registered.
- **B L3:** A misused tool can only send a query to Perplexity and return bounded results; request size and duration are capped per call. — [src/server.ts:725-728](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L725-L728); [src/server.ts:429-431](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L429-L431) (verified)
  - *To reach the next level:* Operations are not fully bounded: research-run cost is set by the remote preset, not by the server.
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

Nothing in the server interprets model text as code: there is no shell, eval, dynamic code loading or file access in the shipped source. All work is an HTTPS POST to the Perplexity API. Code execution is therefore structurally absent.

- **Structural absence:** searched `rg -n 'child_process|exec\(|eval\(|spawn|new Function|node:vm'` in `src/server.ts src/index.ts src/http.ts src/logger.ts src/validation.ts src/types.ts` → 0 hits (no execution primitive in any non-test source file)

### C5 Untrusted input blast radius — 0.30 (medium)

Everything the tools return is untrusted web content: search snippets and Perplexity's synthesized answers, merged into plain text with URLs inline. The structured output repeats the same string rather than separating source content from metadata, and nothing flags it as untrusted beyond the open-world annotation. The outbound side matters too: whatever the host model puts in a question goes to Perplexity's hosted agent, which searches and fetches web pages, so a hijacked host can use these tools as a channel to send data out. The server itself cannot change local state.

- **S L1:** Outputs are plain formatted text with titles, URLs and snippets concatenated; structuredContent wraps the same string. — [src/server.ts:466-471](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L466-L471); [src/server.ts:632-635](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L632-L635) (verified)
  - *To reach the next level:* Returned content is not separated from metadata in structured output and carries no untrusted flag.
- **C L1:** The same plain-text treatment applies to every tool, and web text, upstream error bodies and citations are all mixed in one string. — [src/server.ts:365-374](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L365-L374); [src/server.ts:140-149](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L140-L149) (verified)
  - *To reach the next level:* No tool output distinguishes web-derived content from server-generated text.
- **D L2:** The output format is fixed in code and nothing the server reads can change it. — [src/server.ts:466-471](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L466-L471) (verified)
  - *To reach the next level:* No server mode removes a Rule-of-Two leg (for example a no-egress or search-only mode).
- **B L1:** A hijacked host can send arbitrary text (including secrets from its context) to Perplexity's hosted agent, which fetches URLs during runs, giving an unattended exfiltration channel; no irreversible local action exists. — [src/server.ts:225-227](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L225-L227); [src/server.ts:413-424](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L413-L424) (inferred)
  - *To reach the next level:* The server offers no way to stop outbound content (no egress-free mode, no approval tie-in) so exfiltration needs no human.
- **Cap:** none

### C6 Memory, context & configuration integrity — 1.00 (high)

The server keeps no memory between calls, writes nothing to disk and loads no files from the working directory (no .env loading, no project config). Its settings come only from environment variables the host sets when launching it. The memory and configuration surface is structurally absent.

- **Structural absence:** searched `rg -n 'dotenv|readFile|writeFile|node:fs|process\.cwd|homedir|localStorage'` in `src/server.ts src/index.ts src/http.ts src/logger.ts src/validation.ts src/types.ts` → 0 hits (no file or persistence access in non-test source)

### C7 Third-party extensions — 1.00 (high)

The server loads no plugins, MCP servers, models or packages at runtime; its tool set is fixed in code. How the host installs this package (the README uses unpinned npx -y) is the project's own supply chain and outside this criterion. Third-party extension loading is structurally absent.

- **Structural absence:** searched `rg -n 'import\(|require\(|npx|install|plugin'` in `src/server.ts src/index.ts src/http.ts src/logger.ts src/validation.ts src/types.ts` → 0 hits (no dynamic loading or install path in non-test source)

### C8 Secrets & sensitive-data protection — 0.42 (medium)

The API key comes from the environment and is only placed in the Authorization header of requests to the configured Perplexity base URL; it is never put in tool output, logged, or passed to a subprocess. There is no telemetry and logging defaults to errors only. That safety is by construction rather than by a redaction layer: nothing masks secrets if they ever appear in an error, and upstream error bodies are relayed to the model verbatim. The key itself is a long-lived account key.

- **S L1:** Key read from an env var and used only in an outbound header; no masking, redaction or secret type exists anywhere. — [src/server.ts:21](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L21); [src/server.ts:111-117](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L111-L117); searched `rg -n 'redact|mask|scrub|sanitiz'` in `src/server.ts src/index.ts src/http.ts src/logger.ts src/validation.ts src/types.ts` → 0 hits (no redaction helper exists) (verified)
  - *To reach the next level:* No type-level masking or log/error redaction filters.
- **C L2:** Logs, model-bound tool output and subprocess environments never receive the key by construction; error paths relay upstream response text and stringified network errors to the model. — [src/server.ts:140-149](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L140-L149); [src/server.ts:136](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L136) (verified)
  - *To reach the next level:* Error messages and stack-derived strings are not filtered before reaching the model or stderr.
- **D L2:** No telemetry; log level defaults to ERROR and the server's tool path logs nothing. — [src/logger.ts:24-36](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/logger.ts#L24-L36); searched `rg -n -i 'telemetry|sentry|posthog|analytics'` in `src/server.ts src/index.ts src/http.ts src/logger.ts package.json` → 0 hits (no telemetry SDK or code) (verified)
  - *To reach the next level:* No always-on redaction; verbose levels are a plain env var.
- **B L2:** A leaked key is a single-service Perplexity API key that is long-lived and billing-capable (scope inferred from the provider). — [src/server.ts:21](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L21) (inferred)
  - *To reach the next level:* The key is not short-lived or rotated by the server.
- **Cap:** none

### C9 Audit & traceability — 0.00 (high)

The server keeps no record of what it did. The tool handlers and the API client contain no logging at all, and the logger used elsewhere only covers HTTP-transport rejections and errors at its default level. After an incident there is no server-side trace of which queries were sent, by which caller, or what came back; only the host's logs and Perplexity's own account usage would show it.

- **S L0:** Tool calls are not recorded anywhere by the server. — searched `rg -n 'logger|console\.'` in `src/server.ts` → 0 hits (the tool handlers and API client contain no logging statements at all) (verified)
  - *To reach the next level:* No structured record of tool calls (arguments, status, timestamps).
- **C L0:** No tool path is logged. — searched `rg -n 'logger|console\.'` in `src/server.ts` → 0 hits (the tool handlers and API client contain no logging statements at all) (verified)
  - *To reach the next level:* No tool call is recorded.
- **D L0:** Even raising PERPLEXITY_LOG_LEVEL adds no tool-call logging because no tool code logs. — [src/logger.ts:24-36](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/logger.ts#L24-L36); searched `rg -n 'logger|console\.'` in `src/server.ts` → 0 hits (the tool handlers and API client contain no logging statements at all) (verified)
  - *To reach the next level:* No audit record exists to enable by default.
- **B L0:** With no record at all, calls proceed with nothing to lose or flush. — searched `rg -n 'logger|console\.'` in `src/server.ts` → 0 hits (the tool handlers and API client contain no logging statements at all) (verified)
  - *To reach the next level:* No per-action record is written or flushed.
- **Cap:** none

### C10 Limits & kill switch — 0.62 (high)

Every API call is bounded by a deadline (5 minutes by default) that covers the whole streamed response, and if the host cancels or the deadline fires mid-stream the server asks Perplexity to cancel the run so it stops billing. Search size is bounded by schema. There are no rate or concurrency limits, no spend cap, and no ceiling on how high the operator can set the timeout; research cost is set by Perplexity's preset rather than by the server.

- **S L2:** Server-enforced whole-call timeout on every request plus result-size bounds; host aborts and deadlines trigger best-effort remote cancellation. — [src/server.ts:429-431](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L429-L431); [src/server.ts:95-99](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L95-L99); [src/server.ts:284-289](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L284-L289) (verified)
  - *To reach the next level:* No concurrency or rate limits on calls.
- **C L3:** The deadline covers all four tools (search through makeApiRequest, agent tools through a whole-stream deadline), and the remote agent run is cancelled on abort or deadline once its id is known. — [src/server.ts:441-444](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L441-L444); [src/server.ts:284-289](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L284-L289) (verified)
  - *To reach the next level:* No cap on how many runs can be in flight at once.
- **D L3:** Default 300000 ms comes from operator env only; no tool argument can raise it. — [src/server.ts:429-431](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L429-L431); [.claude-plugin/marketplace.json:42](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/.claude-plugin/marketplace.json#L42) (verified)
  - *To reach the next level:* No hard ceiling: PERPLEXITY_TIMEOUT_MS accepts any value, and the plugin manifest raises it to 600000.
- **B L2:** Each call stops within minutes and cancellation is propagated, but nothing limits how many calls or how much spend accumulates. — [src/server.ts:155-162](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L155-L162); [src/server.ts:429-431](https://github.com/perplexityai/modelcontextprotocol/blob/c58e4ad254608952606f09a40934ab6cca65bfad/src/server.ts#L429-L431) (verified)
  - *To reach the next level:* No per-session time or spend ceiling, and remote cancellation is best-effort (skipped if the deadline fires before the run id arrives).
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Web search snippets and agent answers returned as tool output (src/server.ts:459-479, src/server.ts:337-386) · [B] sensitive data/systems: Operator's Perplexity API key held by the server (src/server.ts:21) plus whatever the host passes in messages · [C] state change / egress: Host-supplied text sent to Perplexity's Agent API, which fetches web pages (src/server.ts:413-424, src/server.ts:225) · Same default session? Yes

## Highest-impact improvements
1. Log every tool call (tool name, argument summary, status, duration, response id) as structured records at the default level. — C9 S L0→L2, +0.150 before caps (Playbook 1 step 3)
2. Return search results as structured items (url, title, snippet) with an explicit untrusted-source flag instead of one formatted string. — C5 S L1→L3, +0.150 before caps (Playbook 1)
3. Add a per-process rate/concurrency limit and an optional per-session call or spend budget. — C10 S L2→L3, +0.075 before caps (Playbook 3 step 3)
4. Redact the API key and proxy credentials from error strings relayed to the model and to stderr. — C8 S L1→L2, +0.075 before caps
5. Offer a search-only mode that disables the agent tools whose remote run can fetch arbitrary URLs. — C5 D L2→L3, +0.050 before caps (Playbook 1)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The README leads with Perplexity's hosted remote endpoint (https://api.perplexity.ai/mcp); its code is not in this repository and was not scored. The scored configuration is the local stdio server, the first mode the repository's code implements.
- The self-hosted HTTP mode (src/http.ts) was reviewed but not scored: it binds to loopback with Host and CORS allowlists by default but has no inbound authentication, so any caller that reaches it spends the operator's key.
- Behaviour of the remote Perplexity Agent API (URL fetching, preset step budgets, key scoping) is outside the repo and is inferred from event names and docs; affected parameters are marked inferred.
- The repository has no git tags, so version is null; package.json declares 1.3.0 but was not used.
- The README's recommended install (`npx -y @perplexity-ai/mcp-server`) is unpinned; that is the project's own distribution supply chain and not scored under C7.
- No reviewer-steering text was found in the repository.
