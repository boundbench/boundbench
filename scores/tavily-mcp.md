# Defense-in-Depth Score: Tavily MCP Server

**Repo:** https://github.com/tavily-ai/tavily-mcp · **Commit:** `1c1d54c2a619afe52544f775c8d82a56ac6d8bb5` · **Reviewed:** 2026-10-04
**What it is:** MCP server exposing Tavily web search, extract, crawl, map, research and feedback tools over stdio.
**Category:** AI Assistants
**Scored configuration:** Local stdio server launched via npx -y tavily-mcp@latest with TAVILY_API_KEY in the client's env (the repo's runnable configuration; the README's lead option, the hosted remote server at mcp.tavily.com, is not in this repo).
**Agent surface (default):** code execution no · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication yes

## Score: 3.8 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L1 | L3 | 0.50 | — | **0.50** | High |
| C2 | Approval gates | L0 | L0 | L0 | L2 | 0.10 | — | **0.10** | High |
| C3 | Tool & action scoping | L1 | L1 | L1 | L2 | 0.30 | — | **0.30** | Medium |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | SA | L2 | 0.30 | C6-REPOCONFIG | **0.25** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L2 | 0.30 | — | **0.30** | High |
| C9 | Audit & traceability | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C10 | Limits & kill switch | L2 | L1 | L1 | L1 | 0.33 | — | **0.33** | High |

Controls where a risk surface exists: 1.88 / 8.0 (24%); 2 criteria scored SA (surface absent).

A small, read-mostly web search server: it runs no code, touches no files and only ever talks to Tavily's API, so it cannot do much damage on its own. Its gaps in posture are around that core: web content comes back unmarked and mixed with the server's own instructions to the model, there are no risk labels for hosts and no audit log, and the extract and feedback tools give a hijacked model an outbound channel. Its configuration loading is also not integrity-protected.

## Critical gaps
- None: no criterion is capped and no blast radius is at the worst level.

## Criterion details

### C1 Identity & least privilege — 0.50 (high)

The server holds one credential, a Tavily API key read from the environment, and uses it only to call Tavily's own API at hardcoded addresses. It does not touch files, spawn processes or reach other systems, so a stolen or misused key can spend Tavily credits and submit feedback, but nothing else. All six tools share that one key and there is no per-request authorization. Credential loading is also not integrity-protected.

- **S L2:** A single dedicated Tavily API key from TAVILY_API_KEY is attached to every request to api.tavily.com; no other credentials or ambient authority are used. — [src/index.ts:14-17](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L14-L17); [src/index.ts:91-93](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L91-L93) (verified)
  - *To reach the next level:* No per-tool or read/write credential split; search and the feedback write share one long-lived key.
- **C L2:** Every tool posts through the one shared axios instance carrying that key to hardcoded Tavily endpoints; there are no subprocesses or extensions. — [src/index.ts:87-97](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L87-L97); [src/index.ts:56-63](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L56-L63) (verified)
  - *To reach the next level:* No authorization layer checks individual requests; any tool call the host sends is executed with the key.
- **D L1:** The scoped default is undermined because credential loading is not integrity-protected. — [src/index.ts:14-17](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L14-L17) (verified)
  - *To reach the next level:* Credential source should come only from the operator-supplied environment.
- **B L3:** A hijacked key reaches only the Tavily account: credit spend, usage and feedback records; no write to any other system. — [src/index.ts:56-63](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L56-L63) (verified)
  - *To reach the next level:* Key is long-lived and not task-scoped or short-lived.
- **Cap:** none

### C2 Approval gates — 0.10 (high)

As a tool server it relies on the MCP host to ask the user before calls, but it gives the host nothing to decide with: none of the six tools carries a read-only or destructive label, there is no dry-run, and no read-only mode. The tools mostly read the web, but the feedback tool sends free text (including the agent's final answer) to Tavily and the research tool can spend significant credits. No tool can delete or change data anywhere, which keeps the damage from a wrongly approved call low.

- **S L0:** No tool annotations (readOnlyHint/destructiveHint/openWorldHint) are declared on any tool, including the feedback write tool. — searched `rg -n 'annotations|readOnlyHint|destructiveHint|openWorldHint'` in `src` → 0 hits (No risk annotations anywhere.); [src/index.ts:431-433](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L431-L433) (verified)
  - *To reach the next level:* Tools need accurate risk annotations so hosts can gate the feedback write and credit-spending research calls.
- **C L0:** No server-side gate or confirmation exists on any tool path; the dispatcher executes every call directly. — [src/index.ts:536-541](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L536-L541) (verified)
  - *To reach the next level:* Consequential tools (feedback, research) should require a confirmation step or be separable.
- **D L0:** There is no approval or read-only setting to enable; the full tool set is always offered. — [src/index.ts:155-157](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L155-L157) (verified)
  - *To reach the next level:* A server-enforced read-only/no-feedback mode on by default would be needed.
- **B L2:** No tool deletes or modifies data; the irreversible effects are disclosure of submitted text to Tavily (feedback, queries) and credit spend. — [src/index.ts:771-777](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L771-L777); [src/index.ts:310-314](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L310-L314) (verified)
  - *To reach the next level:* No preview or quantity bounds on credit-spending calls such as pro research or large crawls.
- **Cap:** none

### C3 Tool & action scoping — 0.30 (medium)

Each tool calls one fixed Tavily endpoint, so the model cannot point the server at an arbitrary host or service. Beyond that, arguments are passed straight through: the declared schemas (enums, a max_results ceiling) are advisory and not checked in the server, crawl and map limits have no upper bound, and any URL can be handed to Tavily to fetch. All six tools are always on and cannot be switched off individually.

- **S L1:** Arguments are forwarded as received with only Array.isArray coercion on domain lists; schema bounds such as maximum: 20 are not enforced in code (the low-level MCP SDK Server does not validate arguments against inputSchema). — [src/index.ts:557-558](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L557-L558); [src/index.ts:747-751](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L747-L751); [src/index.ts:195-200](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L195-L200) (inferred)
  - *To reach the next level:* Server-side validation of URLs, enums and numeric bounds (e.g. crawl limit, max_results) is missing.
- **C L1:** Only search and crawl/map apply the array coercion; extract, research and feedback pass arguments through untouched. — [src/index.ts:567-575](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L567-L575); [src/index.ts:585-586](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L585-L586) (verified)
  - *To reach the next level:* No shared validation layer across tools.
- **D L1:** All six tools, including the arbitrary-URL extract/crawl tools and the feedback write, are exposed by default with no option to disable any. — [src/index.ts:155-157](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L155-L157) (verified)
  - *To reach the next level:* No way to select a read-only or reduced tool set.
- **B L2:** A misused tool can make Tavily fetch any public URL and spend the account's credits, but cannot touch the local host or other systems because endpoints are hardcoded. — [src/index.ts:56-63](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L56-L63); [src/index.ts:298-314](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L298-L314) (verified)
  - *To reach the next level:* No quantity bounds on crawl size or research depth.
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

The server never interprets model text as code: there is no shell, eval, subprocess or script execution anywhere in its source. All work is HTTP requests to Tavily's API.

- **Structural absence:** searched `rg -n 'child_process|\bexec\(|spawn|eval\(|new Function|vm\.'` in `src` → 0 hits (No subprocess, eval or VM usage anywhere in the server source.)

### C5 Untrusted input blast radius — 0.05 (high)

Every tool returns third-party web content as flat text mixed with the server's own labels, with no marker that it is untrusted. The server's own tool description for feedback is written as instructions to the model, and error responses from the API are relayed with suggested next actions such as agentic payment and POSTing answers to an endpoint for bonus credits. If injected web content hijacks the host model, it can use extract to send data in a URL to any host via Tavily, or the feedback tool to send free text to Tavily. The server holds no private data itself and cannot change anything, so the worst case is data leaving, not destruction.

- **S L0:** Tool outputs concatenate untrusted page content with server labels in plain text, the feedback tool description contains directives to the model, and API error envelopes with next_actions are relayed as instructions. — [src/format-results.ts:61-65](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/format-results.ts#L61-L65); [src/index.ts:433](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L433); [src/index.ts:1034-1043](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L1034-L1043) (verified)
  - *To reach the next level:* Outputs should separate content from metadata and carry no directives to the model.
- **C L0:** No source is distinguished: search, extract, crawl, map and research results all enter context the same way. — [src/index.ts:1059-1066](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L1059-L1066); [src/index.ts:1090-1095](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L1090-L1095) (verified)
  - *To reach the next level:* Untrusted content from every tool should be marked with provenance.
- **D L0:** No provenance or isolation mechanism exists to be on by default. — [src/index.ts:536-541](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L536-L541) (verified)
  - *To reach the next level:* A default mode that drops the egress leg (no extract/feedback) is missing.
- **B L1:** A hijacked host can exfiltrate data unattended through extract (attacker URL with data in the query string, fetched by Tavily) or feedback (free text posted to Tavily); the server offers no irreversible or destructive actions. — [src/index.ts:567-575](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L567-L575); [src/index.ts:469-471](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L469-L471) (verified)
  - *To reach the next level:* Removing the outbound channel (or requiring confirmation) after untrusted content is read would be needed.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.25 (high)

The server keeps no memory and reads no instruction files, but its configuration loading is not integrity-protected, which lets workspace content influence it across sessions.

- **S L0:** Configuration loading is not integrity-protected. (verified)
  - *To reach the next level:* Configuration should be read only from the operator's environment or an explicit config path.
- **C L0:** No control applies to the configuration variables it reads. (verified)
  - *To reach the next level:* All security-relevant settings (key, defaults) need to be protected.
- **D SA:** No memory store or per-user namespace exists; there is nothing to isolate across users. — searched `rg -n 'memory|AGENTS\.md|CLAUDE\.md|readFile|writeFile|fs\.'` in `src` → 1 hits (Only hit is a comment at line 841 about assembling a report in memory; no persistence or file reads.) (verified)
- **B L2:** Poisoned configuration persists across sessions but only shapes search results and billing; it cannot add tools or trigger actions. — [src/index.ts:737-744](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L737-L744) (verified)
  - *To reach the next level:* Poisoned config should be session-scoped or visibly surfaced.
- **Cap:** C6-REPOCONFIG — Workspace content can change security-relevant configuration without any trust decision.

### C7 Third-party extensions — 1.00 (high)

The server loads no plugins, launches no other MCP servers and installs nothing at runtime; it uses only its own fixed npm dependencies. (Hosts launching it with npx tavily-mcp@latest is the host's supply-chain choice, not something this server does.)

- **Structural absence:** searched `rg -n 'import\(|require\('` in `src` → 0 hits (No dynamic imports or require calls; the server loads only its static npm dependencies.); searched `rg -n 'child_process|\bexec\(|spawn|eval\(|new Function|vm\.'` in `src` → 0 hits (No subprocess, eval or VM usage anywhere in the server source.)

### C8 Secrets & sensitive-data protection — 0.30 (high)

The API key comes from the environment, is never echoed into tool results, and error messages returned to the model are built from the API's detail field rather than the raw request. There is no redaction anywhere, though, the key is sent in both the Authorization header and every JSON body, and MCP errors are printed whole to stderr. The feedback tool's description pushes the model to send its final answer and quoted snippets to Tavily, and every request carries a session ID for tracking.

- **S L1:** Secrets come from env vars; there is no masking or redaction helper, and the key is duplicated into every request body. — [src/index.ts:14-17](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L14-L17); [src/index.ts:724](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L724); searched `rg -n 'redact|mask|sanitize'` in `src` → 0 hits (No redaction code.) (verified)
  - *To reach the next level:* Type-level masking or log redaction on the main paths is missing.
- **C L1:** Only the model-bound error path is protected, by returning just the API's detail/message rather than the axios error object. — [src/index.ts:676-685](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L676-L685); [src/index.ts:108-110](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L108-L110) (verified)
  - *To reach the next level:* Logs (console.error of raw errors) are not filtered.
- **D L1:** Every request carries a tracking session ID, and the on-by-default feedback tool instructs the model to send its final answer and citations to Tavily. — [src/index.ts:94](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L94); [src/index.ts:433](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L433) (verified)
  - *To reach the next level:* Content-bearing feedback should be opt-in rather than solicited by default.
- **B L2:** A leaked key is long-lived but scoped to one Tavily account and is not reachable by the model. — [src/index.ts:14-17](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L14-L17) (verified)
  - *To reach the next level:* Key is not short-lived or rotated automatically.
- **Cap:** none

### C9 Audit & traceability — 0.00 (high)

The server keeps no record of what it did. Tool calls, arguments and results are not logged; the only output is a startup line and raw MCP errors on stderr. Any audit trail would have to come from the host or Tavily's own dashboard.

- **S L0:** No tool call is recorded; the nine console calls are startup notices, config warnings, MCP errors and the --list-tools printout. — searched `rg -n 'console\.(log|error|warn)'` in `src` → 9 hits (Startup messages, DEFAULT_PARAMETERS warnings, MCP error handler and --list-tools output; none record tool calls.) (verified)
  - *To reach the next level:* A structured per-call record (tool, arguments, status, timestamp) is missing.
- **C L0:** Nothing is recorded for any tool. — [src/index.ts:536-541](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L536-L541) (verified)
  - *To reach the next level:* Every tool path needs to be logged.
- **D L0:** No logging exists to be enabled. — [src/index.ts:107-110](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L107-L110) (verified)
  - *To reach the next level:* On-by-default audit logging is missing.
- **B L0:** With no record, failures are silent and nothing survives a crash. — [src/index.ts:107-110](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L107-L110) (verified)
  - *To reach the next level:* Records need to be written per action.
- **Cap:** none

### C10 Limits & kill switch — 0.33 (high)

The research tool has real time limits: polling stops after 5 or 15 minutes and the streaming fallback has header, idle and overall timers that tear down the connection. Crawl output is truncated to 200 characters per page. The other tools have no request timeout, crawl and map limits are model-chosen with no upper bound, and nothing rate-limits calls or cancels work in flight when the host cancels.

- **S L2:** Server-enforced bounds exist on some operations: research poll and stream timeouts, a bounded error-body read and truncated crawl previews. — [src/index.ts:780-784](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L780-L784); [src/index.ts:918-920](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L918-L920); [src/index.ts:1063-1065](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L1063-L1065) (verified)
  - *To reach the next level:* No caps on every operation and no concurrency or rate limits.
- **C L1:** Only research and crawl output are bounded; search, extract, crawl, map and feedback requests have no timeout. — [src/index.ts:87-97](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L87-L97); searched `rg -n 'timeout'` in `src` → 2 hits (Only a comment and the stream request's timeout: 0; the shared axios instance sets no timeout.) (verified)
  - *To reach the next level:* Timeouts on every Tavily call are missing.
- **D L1:** The model chooses crawl/map limit, depth and breadth with only a minimum of 1, and can pick the 15-minute pro research model. — [src/index.ts:298-314](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L298-L314); [src/index.ts:799-801](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L799-L801) (verified)
  - *To reach the next level:* Hard ceilings the model cannot raise are missing.
- **B L1:** Ceilings are large (up to 15 minutes per research call, unbounded crawl size and credit spend) and host cancellation does not abort in-flight requests. — [src/index.ts:783](https://github.com/tavily-ai/tavily-mcp/blob/1c1d54c2a619afe52544f775c8d82a56ac6d8bb5/src/index.ts#L783); searched `rg -n 'signal|cancel|AbortController'` in `src` → 5 hits (Hits are the feedback description text, a comment, and the research stream's own AbortController; no handler wires MCP cancellation.) (verified)
  - *To reach the next level:* Tight per-call ceilings and cancellation of in-flight work are missing.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Web content from search/extract/crawl/research returned as plain text (src/format-results.ts:61-65, src/index.ts:1059-1066) · [B] sensitive data/systems: Server holds only its Tavily key, never exposed to the model (src/index.ts:91-93); sensitive data comes from the host session · [C] state change / egress: extract fetches any model-chosen URL via Tavily (src/index.ts:567-575); feedback posts free text to Tavily (src/index.ts:771-777) · Same default session? Yes

## Highest-impact improvements
1. Harden configuration loading. — C6 S L0→L2, +0.150 before caps (Playbook 2)
2. Add readOnlyHint/openWorldHint annotations to every tool and mark tavily_feedback as a non-read-only write. — C2 S L0→L2, +0.150 before caps (Playbook 5)
3. Log each tool call (tool, arguments, status, timestamp) as structured JSON to stderr. — C9 S L0→L2, +0.150 before caps (Playbook 1 step 3)
4. Return results as structured content with source URL and an untrusted flag, and remove model directives from tool descriptions and relayed API envelopes. — C5 S L0→L2, +0.150 before caps (Playbook 1)
5. Set an axios timeout on the shared client and enforce server-side maxima on crawl/map limit, depth and breadth. — C10 C L1→L2, +0.075 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The README leads with the hosted remote MCP server (mcp.tavily.com), whose code is not in this repository; the score covers only the local stdio server in src/.
- No release tags exist on the repository; package.json says 0.2.23 and the server self-reports 0.2.22, neither used as the version.
- Whether the MCP SDK 1.30.0 low-level Server validates tool arguments against inputSchema was inferred from library behaviour, not read in the dependency.
- Tavily's server-side behaviour (how it fetches URLs, what feedback data it stores, account-level spend limits) was not examined.
- No reviewer-steering text was found in the repository.
