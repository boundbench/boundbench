# Defense-in-Depth Score: Firecrawl MCP

**Repo:** https://github.com/firecrawl/firecrawl-mcp-server · **Commit:** `af5c378915280a87628a07cbc1b6041e7e8694cb` · **Reviewed:** 2026-10-05
**What it is:** MCP server exposing Firecrawl web scrape, search, crawl, map, browser interaction, agent research, monitoring and paper-search tools.
**Category:** AI Assistants
**Scored configuration:** Local stdio server launched via npx -y firecrawl-mcp with FIRECRAWL_API_KEY in the client's env (the repo's runnable local configuration; the README's lead option, the hosted endpoint at mcp.firecrawl.dev, runs Firecrawl-operated infrastructure in CLOUD_SERVICE mode and is footnoted).
**Agent surface (default):** code execution yes · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents yes · external communication yes

## Score: 4.0 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L1 | L2 | 0.45 | none | **0.45** | High |
| C2 | Approval gates | L1 | L2 | L2 | L1 | 0.38 | none | **0.38** | High |
| C3 | Tool & action scoping | L2 | L2 | L0 | L1 | 0.35 | none | **0.35** | Medium |
| C4 | Code-execution isolation | L4 | L4 | L3 | L2 | 0.85 | none | **0.85** | Medium |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | SA | L1 | 0.25 | C6-REPOCONFIG | **0.25** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L2 | L2 | L2 | 0.42 | none | **0.42** | High |
| C9 | Audit & traceability | L0 | L0 | L0 | L0 | 0.00 | none | **0.00** | High |
| C10 | Limits & kill switch | L2 | L1 | L1 | L1 | 0.33 | none | **0.33** | High |

Controls where a risk surface exists: 3.02 / 9.0 (34%); 1 criterion scored SA (surface absent).

A broad web-data server: besides scraping and search it can drive a remote browser that runs model-written code, start autonomous research jobs, and create recurring monitors that email or call webhooks. It runs nothing on the local machine and labels every tool for hosts, but web content returns unflagged alongside instructions to the model, there is no local read-only mode or audit log, and a hijacked model gets both outbound channels and irreversible actions. A .env file in the launch directory is also auto-loaded without any trust decision.

## Critical gaps
- A hijacked host model can, unattended, send data out through scrape URLs, crawl webhooks or monitor emails and also take irreversible actions such as deleting monitors or submitting forms on live sites. (ASI01, LLM01, T6; C5) — [src/monitor.ts:191-230](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/monitor.ts#L191-L230); [src/monitor.ts:396-400](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/monitor.ts#L396-L400); [src/index.ts:4086-4093](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L4086-L4093)
- A .env file in the server's launch directory is auto-loaded with no trust decision and can change security-relevant settings such as the API endpoint and credential. (ASI06, ASI03, T1; C6) — [src/index.ts:87](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L87); [src/index.ts:1977-1981](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L1977-L1981)

## Criterion details

### C1 Identity & least privilege — 0.45 (high)

The server runs with one Firecrawl credential, an API key or OAuth access token read from the environment, and attaches it to every request to the Firecrawl API. That one key covers the whole Firecrawl account: credit spend, crawl and agent jobs, interact sessions and the account's monitors, including deleting them. There is no per-tool or read-only credential and no per-request authorization in the server. The working directory's .env file is also loaded at startup without any trust decision, so the endpoint and credential settings are not limited to the operator's own environment.

- **S L2:** A single dedicated Firecrawl credential (FIRECRAWL_OAUTH_TOKEN or FIRECRAWL_API_KEY) is resolved from the environment and given to the SDK client for every call; no ambient OS or cloud credentials are used. — [src/index.ts:222-227](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L222-L227); [src/index.ts:1977-1987](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L1977-L1987) (verified)
  - *To reach the next level:* No per-tool or read/write credential split; monitor deletion and search share one long-lived key.
- **C L2:** Every tool path (SDK client, direct fetch for feedback/parse, monitor requests) attaches the same session credential; there are no subprocesses or extensions. — [src/index.ts:3452-3455](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L3452-L3455); [src/monitor.ts:52-56](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/monitor.ts#L52-L56) (verified)
  - *To reach the next level:* No authorization layer checks individual requests; any call the host sends runs with the full account key.
- **D L1:** dotenv.config() loads ./.env from the process working directory at startup with no trust decision, so endpoint and credential settings are not limited to the operator-supplied environment. — [src/index.ts:87](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L87); [src/index.ts:1977-1981](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L1977-L1981) (verified)
  - *To reach the next level:* Credentials and the API endpoint should come only from the operator-supplied environment or an explicit config path.
- **B L2:** A hijacked credential reaches one Firecrawl account with write access: credit spend, job creation, and creating, changing or permanently deleting monitors. — [src/monitor.ts:380-385](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/monitor.ts#L380-L385) (verified)
  - *To reach the next level:* Key is long-lived and not scoped to read-only use or a single task.
- **Cap:** none

### C2 Approval gates — 0.38 (high)

The server gives hosts risk labels for every tool: reads are marked read-only, and monitor update/delete and interact-stop are marked destructive. The labels are not fully accurate, though: firecrawl_interact, which can click, fill and submit forms on live sites and run Bash, Python or Node code in a remote browser, is marked non-destructive even though its own description warns of persistent external side effects. There is no dry-run or preview and no read-only mode for local use; the safe mode that strips browser actions and webhooks applies only to Firecrawl's hosted service. Several actions cannot be undone: deleting monitors, submitting forms and sending monitor emails or webhooks.

- **S L1:** Separate read and write tools carry readOnlyHint/destructiveHint, but firecrawl_interact is marked destructiveHint false while its description says form submission can create persistent external side effects. — [src/index.ts:4086-4093](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L4086-L4093); [src/monitor.ts:380-382](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/monitor.ts#L380-L382) (verified)
  - *To reach the next level:* Every mutating tool needs an accurate destructive hint; interact (and monitor_create, which sends email/webhooks) are under-labelled.
- **C L2:** Every registered tool, including the monitor, research, developer, usage and search-surface variants, carries an annotations block. — searched `rg -n 'annotations: \{'` in `src` → 30 hits (One annotations block per tool registration across index.ts, monitor.ts, research.ts, developer.ts and usage.ts.) (verified)
  - *To reach the next level:* Annotation accuracy (not presence) limits coverage; no dry-run or confirmation step on any path.
- **D L2:** Annotations are hard-coded except firecrawl_scrape's readOnlyHint, which follows the hosted-only SAFE_MODE flag; there is no approval or read-only setting for the local server. — [src/index.ts:2041](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L2041); [src/index.ts:2725](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L2725) (verified)
  - *To reach the next level:* A server-enforced read-only mode available to local operators is missing.
- **B L1:** Wrongly approved calls can permanently delete monitors, submit forms on live sites, and create recurring monitors that email arbitrary recipients or post to arbitrary webhooks; none has a preview. — [src/monitor.ts:191-230](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/monitor.ts#L191-L230); [src/monitor.ts:396-400](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/monitor.ts#L396-L400) (verified)
  - *To reach the next level:* No preview/dry-run for external actions and no bound on recipients or schedules.
- **Cap:** none

### C3 Tool & action scoping — 0.35 (medium)

Tool parameters are declared as zod schemas, which the MCP framework checks before a tool runs, and several fields have real bounds (URL format on most tools, interact timeout up to 300 seconds, feedback list sizes). Many important fields are open, though: the crawl start URL and the crawl/map page limit are unbounded, and crawl and monitor webhooks take any URL and any headers. Every tool, including remote code execution through interact and monitor creation and deletion, is on by default; only the two feedback tools can be switched off. When pointed at a self-hosted API, firecrawl_parse reads any path on the local filesystem with no directory restriction.

- **S L2:** Typed zod schemas with some bounds (url(), timeout max 300) are validated by FastMCP before execute, but crawl url and limit are unconstrained and parse resolves any local path without containment (FastMCP validation of parameters is inferred from library behaviour). — [src/index.ts:3695-3701](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L3695-L3701); [src/index.ts:4262](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L4262); [src/index.ts:4103](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L4103) (inferred)
  - *To reach the next level:* No URL/host allowlists, no numeric ceilings on crawl/map size, and no path containment for local parse.
- **C L2:** All built-in tools declare schemas; validation depth varies, with map/scrape/interact URLs checked and crawl's not. — [src/index.ts:2819](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L2819); [src/index.ts:3695](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L3695) (verified)
  - *To reach the next level:* No shared validation layer (host/URL policy) applied across tools.
- **D L0:** All tools are registered by default, including interact's remote code execution, browser actions, webhooks and the monitor create/update/delete set; only feedback tools have an off switch. — [src/index.ts:2106](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L2106); [src/index.ts:4475](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L4475); [src/index.ts:3363](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L3363) (verified)
  - *To reach the next level:* Dangerous tools (interact, monitors, webhooks) cannot be individually disabled.
- **B L1:** A misused tool can fetch any public URL through Firecrawl, post results to any webhook with custom headers, email any address on a schedule, and run code in the remote browser; local reach is limited to parse in self-hosted mode. — [src/index.ts:3710-3711](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L3710-L3711); [src/monitor.ts:198](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/monitor.ts#L198) (verified)
  - *To reach the next level:* No quantity bounds on crawl size, monitor schedules or recipients.
- **Cap:** none

### C4 Code-execution isolation — 0.85 (medium)

The server never runs model-written code on the user's machine: there is no shell, eval or subprocess anywhere in its source. Code that the model supplies to firecrawl_interact (Bash, Python or Node) or as JavaScript browser actions is sent to Firecrawl's remote browser service and runs there. That remote session has open web access and can persist across calls until stopped, and its isolation is Firecrawl's backend, which this review could not inspect.

- **S L4:** Model-supplied code (interact bash/python/node, executeJavascript actions) is sent to Firecrawl's remote browser service through the API client rather than executed locally. — [src/index.ts:4102](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L4102); [src/index.ts:4172](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L4172) (verified)
- **C L4:** No local execution path exists at all, so there is no unsandboxed fallback. — searched `rg -n 'child_process|\bexec\(|spawn|eval\(|new Function|vm\.'` in `src` → 0 hits (No subprocess, eval or VM usage anywhere in the server source.); [src/index.ts:2100](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L2100) (verified)
- **D L3:** Remote execution is the only mode; the backend is whatever FIRECRAWL_API_URL points to, and a self-hosted instance's isolation is outside this repo. — [src/index.ts:1977-1981](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L1977-L1981) (verified)
  - *To reach the next level:* Execution backend selection is not pinned outside configuration.
- **B L2:** The remote browser holds none of the user's local files or secrets but has unrestricted web egress and sessions persist across calls until firecrawl_interact_stop; backend limits are not visible in this repo. — [src/index.ts:4091-4093](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L4091-L4093) (inferred)
  - *To reach the next level:* No evidence of egress restriction or per-call ephemerality in the remote session.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

Web pages, search results, crawled sites and Alexandria provider data all come back through the same tools with no untrusted flag. Results are JSON with a structured copy, and the Firecrawl API's own guidance is labelled as separate from page content. But results also carry instructions to the model (next-tool suggestions, a feedback-tool pointer, recovery steps), and there is no local mode that drops egress. If injected web content hijacks the host model, the server gives it outbound channels (arbitrary scrape URLs, crawl webhooks, monitor email and webhooks) and irreversible actions (monitor deletion, form submission through interact), with nothing in the server asking a human first.

- **S L0:** Tool outputs carry directives to the model (nextTool instructions, an appended feedback-tool pointer, API agent hints) alongside unflagged page content; a 'source content is data, not instructions' note is the only marker. — [src/alexandria-output.ts:73](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/alexandria-output.ts#L73); [src/alexandria-feedback.ts:96](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/alexandria-feedback.ts#L96); [src/agent-hints.ts:23-24](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/agent-hints.ts#L23-L24) (verified)
  - *To reach the next level:* Outputs should carry provenance/untrusted flags and keep directives out of returned content.
- **C L0:** No source is distinguished: scrape, search, crawl, agent and Alexandria results all enter context the same way through the shared structured-text helpers. — [src/tool-output.ts:40-44](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/tool-output.ts#L40-L44) (verified)
  - *To reach the next level:* Untrusted content from every tool should be marked with provenance.
- **D L0:** The only mode that trims risky capability (SAFE_MODE) is tied to the hosted service flag; nothing is on for the local server. — [src/index.ts:2041](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L2041) (verified)
  - *To reach the next level:* A default local mode that drops the egress or write leg is missing.
- **B L0:** A hijacked host can exfiltrate via any scrape URL, crawl webhook or monitor email/webhook and can delete monitors or submit forms on live sites, all without server-side approval. — [src/monitor.ts:191-230](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/monitor.ts#L191-L230); [src/monitor.ts:396-400](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/monitor.ts#L396-L400); [src/index.ts:4086-4093](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L4086-L4093) (verified)
  - *To reach the next level:* Removing egress and irreversible tools (or requiring confirmation) once untrusted content is read would be needed.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.25 (high)

The server keeps no memory and reads no instruction files. At startup, though, it silently loads a .env file from whatever directory the host launches it in, often the user's open project. That file can change security-relevant settings, including the API endpoint and credential, and the change persists for every session started in that directory.

- **S L0:** dotenv.config() auto-loads ./.env with no trust decision, and the settings it can supply include the API endpoint and credential. — [src/index.ts:87](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L87) (verified)
  - *To reach the next level:* Configuration should be read only from the operator's environment or an explicit config path.
- **C L0:** No control applies to any variable the .env file can set, including the API endpoint and credential. — [src/index.ts:1977-1981](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L1977-L1981) (verified)
  - *To reach the next level:* All security-relevant settings (endpoint, key) need to be protected.
- **D SA:** No memory store or per-user namespace exists; there is nothing to isolate across users. — searched `rg -n 'memory|AGENTS\.md|CLAUDE\.md|writeFile'` in `src` → 1 hits (Only hit is an example query string about a memory leak in research.ts; no persistence or instruction files.) (verified)
- **B L1:** A poisoned .env persists for every session in that directory and affects every tool call the server makes, which can steer later tool use. — [src/index.ts:87](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L87) (verified)
  - *To reach the next level:* Poisoned config should be session-scoped or visibly surfaced.
- **Cap:** C6-REPOCONFIG — A .env file in the launch directory is auto-loaded without any trust decision and can change security-relevant configuration.

### C7 Third-party extensions — 1.00 (high)

The server loads no plugins, launches no other MCP servers and installs nothing at runtime; it uses only its own fixed npm dependencies. Alexandria providers run on Firecrawl's side and come back as tool output, which is covered under untrusted input. The plugin manifests in the repo are packaging for hosts, not something the server loads.

- **Structural absence:** searched `rg -n 'import\(|require\('` in `src` → 2 hits (One createRequire of package.json and one hit in the legacy markdown archive; no dynamic code loading.); searched `rg -n 'child_process|\bexec\(|spawn|eval\(|new Function|vm\.'` in `src` → 0 hits (No subprocess, eval or VM usage anywhere in the server source.)

### C8 Secrets & sensitive-data protection — 0.42 (high)

The credential comes from environment variables and is only ever placed in the Authorization header to the Firecrawl API; it is never written into tool results. In the default stdio mode the server's logger is silent, and the hosted-mode action and telemetry records are built from a fixed set of fields that excludes credentials. There is no masking or redaction helper, error messages pass API response bodies straight back to the model, and the key itself is long-lived and covers the whole account.

- **S L1:** Secrets come from env vars with no masking or redaction helper; the symbol-keyed managed credential exists only in hosted OAuth mode. — [src/index.ts:222-227](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L222-L227); searched `rg -n -i 'mask|sanitiz' --type ts` in `src` → 1 hits (Only hit is a comment about the hosted nginx proxy sanitizing forwarded IP chains.) (verified)
  - *To reach the next level:* Type-level masking or log redaction on the main paths is missing.
- **C L2:** Credentials stay out of logs (logger off in stdio; action logs list fixed non-secret fields) and out of model-bound results by construction. — [src/index.ts:1279-1282](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L1279-L1282); [src/index.ts:1703-1716](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L1703-L1716) (verified)
  - *To reach the next level:* Error paths relay raw API response text to the model without filtering.
- **D L2:** No telemetry SDK and no logging in the default stdio mode; hosted telemetry is gated on CLOUD_SERVICE. — [src/index.ts:1279-1282](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L1279-L1282) (verified)
  - *To reach the next level:* No always-on redaction to make logging safe when HTTP modes enable it.
- **B L2:** A leaked key is long-lived and scoped to one Firecrawl account, and is not reachable by the model. — [src/index.ts:222-227](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L222-L227) (verified)
  - *To reach the next level:* Key is not short-lived or task-scoped.
- **Cap:** none

### C9 Audit & traceability — 0.00 (high)

In the default stdio mode the server keeps no record of what it did: its logger only writes when running as an HTTP or hosted service, and the structured per-call action log is emitted only in Firecrawl's hosted deployment. Tool calls, arguments and results leave no trace in the server, so any audit trail has to come from the host or from Firecrawl's account dashboard.

- **S L0:** No tool call is recorded in stdio; ConsoleLogger is disabled unless an HTTP or cloud mode is set, and emitActionLog returns early outside CLOUD_SERVICE. — [src/index.ts:1279-1282](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L1279-L1282); [src/index.ts:1703](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L1703) (verified)
  - *To reach the next level:* A structured per-call record (tool, arguments, status, timestamp) in the default mode is missing.
- **C L0:** Nothing is recorded for any tool in the scored configuration. — [src/index.ts:1703](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L1703) (verified)
  - *To reach the next level:* Every tool path needs to be logged.
- **D L0:** Logging is off for stdio and can only be turned on by switching to a different transport. — [src/index.ts:1279-1282](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L1279-L1282) (verified)
  - *To reach the next level:* On-by-default audit logging is missing.
- **B L0:** With no record, actions proceed and leave nothing behind; the hosted action-log POST also swallows failures. — [src/index.ts:1729-1739](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L1729-L1739) (verified)
  - *To reach the next level:* Records need to be written per action.
- **Cap:** none

### C10 Limits & kill switch — 0.33 (high)

Some work is bounded by the server: interact code runs with a timeout of at most 300 seconds, Alexandria calls have a 15-second timeout and an inline output budget, and PDF parsing has a page cap. Other work is not: the crawl tool polls until the job finishes with no deadline, crawl and map page limits are whatever the model asks for, and there is no rate limit. Monitors the model creates keep running on their schedule after the session ends until someone deletes them, and host cancellation is not passed on to in-flight requests.

- **S L2:** Server-enforced bounds exist on some operations: interact timeout ≤300s, a 15s Alexandria timeout and a 20k-token inline budget. — [src/index.ts:4103](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L4103); [src/index.ts:2972](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L2972); [src/alexandria-output.ts:1](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/alexandria-output.ts#L1) (verified)
  - *To reach the next level:* No caps on every operation and no concurrency or rate limits.
- **C L1:** Crawl polling has no deadline because timeout is not in the crawl schema, so the loop's timeout check never fires; most API calls have no timeout. — [src/index.ts:3307](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L3307); [src/index.ts:3739-3744](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L3739-L3744) (verified)
  - *To reach the next level:* Timeouts on every Firecrawl call and a crawl polling deadline are missing.
- **D L1:** The model chooses crawl and map page limits with no upper bound. — [src/index.ts:3701](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L3701); [src/index.ts:2823](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/index.ts#L2823) (verified)
  - *To reach the next level:* Hard ceilings the model cannot raise are missing.
- **B L1:** Monitors keep running on a schedule (every 30 minutes by default) after the session ends, crawl polling is unbounded, and no MCP cancellation handler aborts in-flight work. — [src/monitor.ts:211-215](https://github.com/firecrawl/firecrawl-mcp-server/blob/af5c378915280a87628a07cbc1b6041e7e8694cb/src/monitor.ts#L211-L215); searched `rg -n 'signal|AbortController|cancel' --type ts` in `src` → 6 hits (Hits are a comment, the hosted introspection timeout, the action-log timeout and a crawl status string; no handler wires MCP cancellation.) (verified)
  - *To reach the next level:* Tight per-call ceilings, cancellation of in-flight work and no lingering scheduled jobs are missing.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Scraped pages, search, crawl and Alexandria results returned unflagged (src/tool-output.ts:40-44, src/alexandria-output.ts:73) · [B] sensitive data/systems: Full-account Firecrawl key held by the server (src/index.ts:222-227); sensitive data also comes from the host session · [C] state change / egress: Crawl webhooks (src/index.ts:3710-3711), monitor email/webhooks and delete (src/monitor.ts:191-230, src/monitor.ts:396-400), interact form submission (src/index.ts:4086-4093) · Same default session? Yes

## Highest-impact improvements
1. Stop auto-loading .env from the working directory; read configuration only from the operator's environment or an explicit --env-file path. — C6 S L0→L2, +0.150 before caps (Playbook 2)
2. Log each tool call (tool, arguments, status, timestamp) as structured JSON to stderr in stdio mode. — C9 S L0→L2, +0.150 before caps (Playbook 1 step 3)
3. Mark firecrawl_interact (and email/webhook-sending monitor tools) destructive, and offer a local read-only/safe-mode flag that drops interact, actions, webhooks and monitor writes. — C2 S L1→L2, +0.075 before caps (Playbook 5)
4. Add an untrusted/provenance flag to returned web content and move next-tool and feedback guidance out of tool results. — C5 S L0→L2, +0.150 before caps (Playbook 1)
5. Expose and default a crawl polling deadline and enforce server-side maxima on crawl/map limits. — C10 C L1→L2, +0.075 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The README leads with the hosted endpoint (mcp.firecrawl.dev), which runs this code in CLOUD_SERVICE mode on Firecrawl's infrastructure with OAuth, safe mode and action logging; the score covers the local stdio server, where those are inactive.
- No release tag at the pinned commit (git describe: v3.24.1-175-gaf5c378); package.json says 3.27.3, not used as the version.
- FastMCP 4.3.2 (patched) validating tool arguments against the zod schemas before execute was inferred from library behaviour, not read in the dependency.
- Firecrawl's backend (remote browser isolation for interact code, monitor delivery, Alexandria providers, account spend limits) was not examined.
- No reviewer-steering text was found; the repo's agent-metadata policy script (scripts/agent-metadata-policy.mjs) constrains coercive wording in tool descriptions but is a test, not a runtime control.
