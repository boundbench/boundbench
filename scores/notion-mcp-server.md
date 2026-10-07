# Defense-in-depth score: Notion MCP Server

**Repo:** https://github.com/makenotion/notion-mcp-server · **Commit:** `730ae781ba28beeaf0865025a3f2ed4c25ea2387` (v2.5.2) · **Reviewed:** 2026-10-05
**What it is:** Notion's self-hosted MCP server that exposes the Notion API as tools generated from a bundled OpenAPI spec.
**Category:** AI Assistants
**Scored configuration:** `npx -y @notionhq/notion-mcp-server` with NOTION_TOKEN set and no flags: stdio transport, all 24 tools from the bundled spec, one Notion integration token for every call.
**Agent surface (default):** code execution no · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication yes

## Score: 5.4 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L2 | L2 | L2 | 0.42 | none | **0.42** | High |
| C2 | Approval gates | L2 | L3 | L3 | L2 | 0.62 | none | **0.62** | High |
| C3 | Tool & action scoping | L2 | L2 | L0 | L2 | 0.40 | none | **0.40** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L1 | 0.12 | none | **0.12** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L3 | L2 | 0.55 | none | **0.55** | High |
| C9 | Audit & traceability | L0 | L0 | L0 | L0 | 0.00 | none | **0.00** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L1 | 0.25 | none | **0.25** | High |

Controls where a risk surface exists: 2.38 / 7.0 (34%); 3 criteria scored SA (surface absent).

Run as shipped, this server gives the model read and write access to every Notion page the operator's integration can see, through 24 narrow tools that only ever talk to api.notion.com. It runs no code, keeps no memory and loads no plugins, and every write tool is labelled destructive so the MCP host can ask before acting. The dominant risk is prompt injection: page and comment content comes back unmarked, nothing in the server separates reading it from writing pages or comments, and the server keeps no record of what it did. Scope the integration in Notion (for example read-only capabilities) to cut its authority.

## Critical gaps
- Unmarked page and comment content, every page shared with the integration, and page-writing and comment tools are all available in one default session with nothing in the server separating them. (ASI01, T6, LLM01; C5). Evidence: [src/openapi-mcp-server/mcp/proxy.ts:214-220](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/mcp/proxy.ts#L214-L220); [scripts/notion-openapi.json:1515](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/scripts/notion-openapi.json#L1515)
- The server records nothing about the tool calls it executes, so writes and deletes leave no server-side trail. (T8, ASI10; C9). Evidence: [src/openapi-mcp-server/mcp/proxy.ts:209-221](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/mcp/proxy.ts#L209-L221)

## Criterion details

### C1 Identity & least privilege: 0.42 (high confidence)

The server authenticates every call with the single Notion integration token the operator puts in its environment. How much that token can reach is decided entirely in Notion (the integration's capabilities and which pages are shared with it); the server itself does not narrow it, and read and write tools use the same credential. No tool can change the integration's own permissions or sharing. An opt-in HTTP mode can instead accept a per-connection Notion token from each client, which is off in the scored configuration.

- **S L1:** Every request carries the one bearer token built from NOTION_TOKEN (or headers from OPENAPI_MCP_HEADERS); the server adds no scoping of its own. Evidence: [src/openapi-mcp-server/mcp/proxy.ts:268-274](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/mcp/proxy.ts#L268-L274); [src/openapi-mcp-server/mcp/proxy.ts:145-151](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/mcp/proxy.ts#L145-L151) (verified)
  - *To reach the next level:* Read tools and write tools share one credential; the server never uses a narrower token for reads.
- **C L2:** All tools go through the one HttpClient built with the integration headers, and the only destination is the spec's api.notion.com server; there are no subprocesses or plugins. Evidence: [src/openapi-mcp-server/client/http-client.ts:42-52](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/client/http-client.ts#L42-L52); [scripts/notion-openapi.json:14](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/scripts/notion-openapi.json#L14) (verified)
  - *To reach the next level:* No authorization check in code per tool or per requesting principal before the token is attached.
- **D L2:** The credential comes only from operator environment, and the shipped spec has no endpoint that manages integration capabilities, sharing or users, so the model cannot widen its own authority. Evidence: [src/openapi-mcp-server/mcp/proxy.ts:248-278](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/mcp/proxy.ts#L248-L278); searched `rg -n -S -e 'oauth|/v1/(integrations|tokens|workspaces)'` in `scripts/notion-openapi.json` → 0 hits (the bundled spec has no endpoint that manages integrations, tokens or workspace settings) (verified)
  - *To reach the next level:* The server does not default to a read-only identity; least privilege depends on how the operator configures the integration in Notion.
- **B L2:** A hijacked session gets read and write access to every page and data source shared with the integration in one Notion workspace. Evidence: [scripts/notion-openapi.json:964](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/scripts/notion-openapi.json#L964); [scripts/notion-openapi.json:1819](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/scripts/notion-openapi.json#L1819) (verified)
  - *To reach the next level:* Writes are not limited to non-destructive operations; delete, overwrite and schema-update tools ship with the same token.
- **Cap:** none
- **Notes:** Token passthrough (C1-PASSTHRU pattern) exists only behind --enable-token-passthrough on the HTTP transport and is off by default (scripts/server-options.ts:28), so the cap does not apply to the scored configuration.

### C2 Approval gates: 0.62 (high confidence)

As a tool server it relies on the MCP host to ask the user before acting. What it provides is a label on every tool, derived from the HTTP method: GET tools are marked read-only and everything else is marked destructive, so no write tool is presented as safe, although two read-only search and query tools are also marked destructive. There is no preview or dry-run, and no server-side read-only mode. If a host auto-approves or the user approves the wrong call, page content can be overwritten, blocks and pages moved to trash, and comments posted; most of that can be recovered from Notion's trash and page history.

- **S L2:** Reads and writes are separate tools, and every tool carries readOnlyHint (GET) or destructiveHint (all other methods). Evidence: [src/openapi-mcp-server/mcp/proxy.ts:173-187](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/mcp/proxy.ts#L173-L187) (verified)
  - *To reach the next level:* No preview or dry-run for destructive operations such as whole-page replacement or block deletion.
- **C L3:** Annotations are computed for every generated tool by one code path, only GET operations (all read-only in the spec) are marked read-only, a failed lookup falls through to destructive, and unknown tool names are rejected. Evidence: [src/openapi-mcp-server/mcp/proxy.ts:184-186](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/mcp/proxy.ts#L184-L186); [src/openapi-mcp-server/mcp/proxy.ts:200-203](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/mcp/proxy.ts#L200-L203) (verified)
  - *To reach the next level:* Two read-only POST tools (search, data-source query) are flagged destructive, which adds approval noise; there is no per-argument risk signal.
- **D L3:** Annotations are hard-coded from the bundled spec, which is loaded from the install directory, so neither the model nor tool content can change them. Evidence: [scripts/start-server.ts:28](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/scripts/start-server.ts#L28) (verified)
  - *To reach the next level:* No server-enforced read-only mode or confirmation step that the host must complete.
- **B L2:** Deletes set pages and blocks to trash (restorable), but whole-page Markdown replacement, data-source schema updates and posted comments are only as recoverable as Notion's own page history. Evidence: [scripts/notion-openapi.json:1118-1121](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/scripts/notion-openapi.json#L1118-L1121); [scripts/notion-openapi.json:2457](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/scripts/notion-openapi.json#L2457) (verified)
  - *To reach the next level:* No checkpoints, previews or rate limits on writes inside the server.
- **Cap:** none

### C3 Tool & action scoping: 0.40 (high confidence)

Each tool maps to one Notion API endpoint with a typed input schema generated from the bundled spec, and every request goes to the fixed api.notion.com server, so there is no general URL fetch, shell or query language. The server does not validate arguments against those schemas itself; it decodes JSON-looking strings and forwards them, relying on Notion's API to reject bad input. With no flags all 24 tools load, including page overwrite, block deletion and schema updates, and there is no option to select a smaller or read-only set. A generic local-file upload helper exists in the HTTP client, but no operation in the shipped spec uses it.

- **S L2:** Narrow per-endpoint tools with typed schemas and a fixed destination host; arguments are not validated in code before forwarding. Evidence: [src/openapi-mcp-server/mcp/proxy.ts:205-211](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/mcp/proxy.ts#L205-L211); searched `rg -n -S -e 'ajv|zod|safeParse|validateArgs'` in `src/openapi-mcp-server/mcp src/openapi-mcp-server/client` → 0 hits (no argument validation library or call in the tool execution path) (verified)
  - *To reach the next level:* No server-side schema validation or numeric bounds on arguments; bounds are left to Notion's API.
- **C L2:** Every tool is generated the same way, from the same spec, and reaches only the spec's base URL; no tool accepts a free-form URL, path or command. Evidence: [src/openapi-mcp-server/client/http-client.ts:199-218](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/client/http-client.ts#L199-L218) (verified)
  - *To reach the next level:* No shared validation layer that enforces the generated schemas for every tool.
- **D L0:** All 24 operations in the bundled spec, including delete, overwrite and schema-update tools, are registered on every start with no server option to disable any. Evidence: [src/openapi-mcp-server/openapi/parser.ts:174-191](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/openapi/parser.ts#L174-L191); searched `rg -n -S -e 'read.?only|disabled?.?tools|--tools|toolset'` in `scripts/server-options.ts scripts/start-server.ts` → 0 hits (the CLI parser offers transport, port, host and auth options only) (verified)
  - *To reach the next level:* No tool-group selection or read-only default; write tools cannot be switched off in the server.
- **B L2:** A misused tool is confined to the Notion workspace pages shared with the integration but has full write inside them, with no quantity limits. Evidence: [scripts/notion-openapi.json:2405](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/scripts/notion-openapi.json#L2405) (verified)
  - *To reach the next level:* No bounds on how many pages, blocks or comments a run can change.
- **Cap:** none

### C4 Code-execution isolation: 1.00 (high confidence)

The server never runs model-influenced code: there is no shell, subprocess, eval or template execution anywhere in its source, and tool calls only become HTTPS requests to Notion's API. The one eval in the source is inside a commented-out block. The optional Docker image runs the same Node process and adds no execution path.

- **Structural absence:** searched `rg -n -S -g '!**/__tests__/**' -e 'child_process|exec\(|spawn|eval\(|new Function|vm\.|execSync'` in `src scripts/start-server.ts scripts/server-options.ts` → 2 hits (one hit is RegExp.exec in token parsing (token.ts:105), the other a commented-out eval (parser.ts:353); no execution path exists)

### C5 Untrusted input blast radius: 0.12 (high confidence)

Page content, blocks and comments, which other workspace members, guests or imported content may have written, are returned to the model as the raw Notion API JSON, with nothing marking them as third-party text. In the default configuration the same session can read that content, read every page shared with the integration, and write pages or post comments, and nothing in the server separates those. A successful prompt injection can therefore copy private page content into a page or comment the attacker can read, without a human involved unless the host asks. Deletions go to Notion's trash rather than being permanent.

- **S L1:** Tool results are the upstream JSON serialized into one text item, with no provenance or untrusted flag added by the server; tool descriptions carry no directives to the model. Evidence: [src/openapi-mcp-server/mcp/proxy.ts:214-220](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/mcp/proxy.ts#L214-L220) (verified)
  - *To reach the next level:* No server-added separation of user-authored content from metadata, and no untrusted marker the host can act on.
- **C L0:** No content source (pages, blocks, comments, search results, Markdown export) is distinguished from any other. Evidence: searched `rg -n -S -e 'untrusted|provenance|injection|sanitiz'` in `src scripts/start-server.ts scripts/server-options.ts` → 0 hits (no untrusted-content handling anywhere in the server) (verified)
  - *To reach the next level:* Mark every user-authored content source as untrusted in outputs.
- **D L0:** No untrusted-content control exists, and there is no mode that drops write or comment tools. Evidence: [src/openapi-mcp-server/openapi/parser.ts:174-191](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/openapi/parser.ts#L174-L191) (verified)
  - *To reach the next level:* Offer a default or mode that removes a Rule-of-Two leg, such as a read-only tool set.
- **B L1:** A hijacked session can read every shared page and write it into pages or comments an attacker can see, but deletions set pages and blocks to trash where they can be restored. Evidence: [scripts/notion-openapi.json:1515](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/scripts/notion-openapi.json#L1515); [scripts/notion-openapi.json:2220](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/scripts/notion-openapi.json#L2220); [scripts/notion-openapi.json:1118-1121](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/scripts/notion-openapi.json#L1118-L1121) (verified)
  - *To reach the next level:* Make writes and comments unavailable or gated in sessions that have read user-authored content.
- **Cap:** none

### C6 Memory, context & configuration integrity: 1.00 (high confidence)

The server keeps no memory, retrieval store or conversation history, and it reads nothing from the user's working directory: the API spec is loaded by an absolute path inside its own install directory and configuration comes only from environment variables set in the MCP host's config. Nothing the model reads can persist into later sessions through the server.

- **Structural absence:** searched `rg -n -S -g '!**/__tests__/**' -e 'dotenv|memory|AGENTS\.md|CLAUDE\.md|readFileSync|sqlite|localStorage|cache'` in `src scripts/start-server.ts scripts/server-options.ts` → 2 hits (init-server.ts:20 reads the bundled spec whose absolute path is built from the module directory (start-server.ts:28); parser.ts:66 is an in-memory schema cache comment)

### C7 Third-party extensions: 1.00 (high confidence)

The server loads no plugins, MCP servers, remote tools or model files at runtime; its tools come only from the OpenAPI spec bundled with the package. How the host installs the server itself (for example an unpinned npx command) is the project's own distribution and is out of scope here.

- **Structural absence:** searched `rg -n -S -g '!**/__tests__/**' -e 'plugin|import\(|require\(|npx|mcpServers|registry|extension'` in `src scripts/start-server.ts scripts/server-options.ts` → 0 hits (no dynamic import, plugin loader or extension registry in the server source)

### C8 Secrets & sensitive-data protection: 0.55 (high confidence)

The Notion token is read from the environment and only ever attached to outgoing API requests; it is never returned to the model. The server's logging is deliberately thin: failed API calls log only the HTTP status, tool errors log only the error message, and the optional per-connection token is logged only as a redacted prefix. There is no telemetry, crash reporting or stored transcript. Page content that itself contains secrets is passed to the model unfiltered, and the integration token is long-lived, though limited to the pages shared with it.

- **S L2:** Token comes from env vars; a redaction helper masks per-connection tokens in logs and API error logging is restricted to status fields. Evidence: [src/openapi-mcp-server/mcp/token.ts:120-124](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/mcp/token.ts#L120-L124); [src/openapi-mcp-server/client/http-client.ts:234-239](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/client/http-client.ts#L234-L239) (verified)
  - *To reach the next level:* No keychain or secret-manager storage, and no redaction of secrets inside content sent to the model.
- **C L2:** Logs and error messages carry no credentials, there are no subprocesses or transcripts, and the token never enters tool output. Evidence: [src/openapi-mcp-server/mcp/proxy.ts:223](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/mcp/proxy.ts#L223) (verified)
  - *To reach the next level:* No redaction pass on model-bound content (secrets stored in Notion pages reach the model as-is).
- **D L3:** No telemetry or analytics exists at all, and the minimal logging is not configurable into a verbose payload log. Evidence: searched `rg -n -S -e 'sentry|posthog|telemetry|analytics|mixpanel'` in `src scripts/start-server.ts scripts/server-options.ts` → 0 hits (no telemetry SDK or analytics call) (verified)
  - *To reach the next level:* Nothing to encrypt or minimise beyond this, but content-level redaction is not offered as an always-on default.
- **B L2:** A leaked integration token is long-lived but limited to the pages and capabilities granted to that integration, and the model cannot read it. Evidence: [src/openapi-mcp-server/mcp/proxy.ts:267-275](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/mcp/proxy.ts#L267-L275) (verified)
  - *To reach the next level:* Tokens are not short-lived or rotated by the server.
- **Cap:** none

### C9 Audit & traceability: 0.00 (high confidence)

The server keeps no record of what it does. A successful tool call writes nothing at all, and a failed one prints only an error message or HTTP status to stderr, without the tool's arguments, the page it touched or a timestamp. After an incident the only trace is whatever the MCP host logged and Notion's own page history.

- **S L0:** The tool-call handler returns results without logging the call; only failures print a message to stderr. Evidence: [src/openapi-mcp-server/mcp/proxy.ts:196-223](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/mcp/proxy.ts#L196-L223); searched `rg -n -S -e 'audit|logger|winston|pino|opentelemetry|otel|trace|jsonl'` in `src scripts/start-server.ts scripts/server-options.ts` → 0 hits (no logging or tracing framework) (verified)
  - *To reach the next level:* Write a structured record of every tool call with arguments, result status and timestamp.
- **C L0:** No tool path is recorded. Evidence: [src/openapi-mcp-server/mcp/proxy.ts:209-221](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/mcp/proxy.ts#L209-L221) (verified)
  - *To reach the next level:* Record every tool call, including writes and deletes.
- **D L0:** There is no audit record to enable. Evidence: searched `rg -n -S -e 'audit|logger|winston|pino|opentelemetry|otel|trace|jsonl'` in `src scripts/start-server.ts scripts/server-options.ts` → 0 hits (no logging or tracing framework) (verified)
  - *To reach the next level:* Ship a per-call record on by default, written outside anything the model controls.
- **B L0:** Actions proceed with nothing recorded, so nothing is lost or surfaced on failure. Evidence: [src/openapi-mcp-server/mcp/proxy.ts:214-220](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/mcp/proxy.ts#L214-L220) (verified)
  - *To reach the next level:* Flush a durable record per action and surface failures to write it.
- **Cap:** none

### C10 Limits & kill switch: 0.25 (high confidence)

The server sets no bounds on its own work: Notion API requests have no timeout, response size is not capped, and there are no rate limits on writes or comments. The only limits are caller-chosen page sizes defined by Notion's API (up to 100 items per page) and Notion's own server-side rate limits, plus a small cap on how many times a string argument is JSON-decoded. A runaway host loop can keep editing pages until Notion throttles it.

- **S L1:** Only caller-chosen limits exist (Notion page_size parameters); the server caps JSON-decode passes on arguments but sets no timeouts or output caps. Evidence: [scripts/notion-openapi.json:364-370](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/scripts/notion-openapi.json#L364-L370); [src/openapi-mcp-server/mcp/proxy.ts:80](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/mcp/proxy.ts#L80); searched `rg -n -S -e 'timeout|rate.?limit|throttle|maxContentLength|maxBodyLength|AbortController|signal'` in `src scripts/start-server.ts scripts/server-options.ts` → 1 hits (the only hit is a test name (http-client.test.ts:220); no timeouts, rate limits or cancellation in the server) (verified)
  - *To reach the next level:* Add server-enforced request timeouts and response-size caps on API calls.
- **C L1:** Page-size limits apply only to list and query endpoints; write tools have no bound. Evidence: [src/openapi-mcp-server/client/http-client.ts:42-53](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/client/http-client.ts#L42-L53) (verified)
  - *To reach the next level:* Apply timeouts and per-session ceilings to every tool, including writes.
- **D L1:** The default page size is Notion's maximum of 100 and the model chooses it per call; nothing else has a default limit. Evidence: [scripts/notion-openapi.json:364-370](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/scripts/notion-openapi.json#L364-L370) (verified)
  - *To reach the next level:* Set sensible server-side defaults (timeout, write rate) the model cannot raise.
- **B L1:** Before Notion's own rate limits intervene, a runaway loop can make unlimited edits and comments, and a hung request is never cancelled by the server. Evidence: [src/openapi-mcp-server/client/http-client.ts:218](https://github.com/makenotion/notion-mcp-server/blob/730ae781ba28beeaf0865025a3f2ed4c25ea2387/src/openapi-mcp-server/client/http-client.ts#L218) (verified)
  - *To reach the next level:* Add per-session ceilings on side-effecting calls and cancellation of in-flight requests.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: page, block and comment content returned verbatim (src/openapi-mcp-server/mcp/proxy.ts:214-220) · [B] sensitive data/systems: every page shared with the operator's integration token (src/openapi-mcp-server/mcp/proxy.ts:268-274) · [C] state change / egress: create-a-comment and update-page-markdown tools (scripts/notion-openapi.json:1515, 2405) · Same default session? Yes

## Highest-impact improvements
1. Add a read-only flag (and ideally make it the default) that registers only GET and read-only query tools. (C3 D L0→L3, +0.150 before caps; Playbook 3)
2. Write a structured per-call record (tool, arguments, target IDs, status, timestamp) to a log outside the model's reach. (C9 S L0→L2, +0.150 before caps; Playbook 1 step 3)
3. Wrap returned page and comment content with provenance (page ID, author) and an untrusted flag the host can act on. (C5 S L1→L3, +0.150 before caps; Playbook 1)
4. Set an HTTP timeout and response-size cap on every Notion API call, plus a per-session write rate limit. (C10 S L1→L2, +0.075 before caps; Playbook 3 step 3)
5. Offer a dry-run or preview for whole-page replacement and block deletion. (C2 S L2→L3, +0.075 before caps; Playbook 5)

## Re-audit log
- C5 B: L0 → L1. Delete and trash operations in the shipped spec set in_trash and are restorable (scripts/notion-openapi.json:1118-1121); exfiltration through page and comment writes remains unattended, so only one leg of the L0 anchor holds.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the default stdio transport. The opt-in Streamable HTTP transport (bearer auth on by default, loopback bind) and opt-in per-connection token passthrough were reviewed but not scored.
- How far the integration token reaches (capabilities, shared pages) is configured in Notion, not in this code; ratings assume a standard internal integration with content read and write.
- Behaviour of third-party libraries (MCP SDK, openapi-client-axios) was inferred from their source at the locked versions, not exercised.
- The README states this self-hosted server is no longer actively maintained and recommends Notion's hosted MCP server instead; maintenance status is not scored.
- No text aimed at AI reviewers was found in the repository.
