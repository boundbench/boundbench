# Defense-in-Depth Score: Google Analytics MCP

**Repo:** https://github.com/googleanalytics/google-analytics-mcp · **Commit:** `75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a` · **Reviewed:** 2026-10-05
**What it is:** Local stdio MCP server exposing read-only Google Analytics Admin and Data API tools (account summaries, property details, core, realtime, funnel and conversions reports).
**Category:** Data & Analytics
**Scored configuration:** Local stdio server launched via pipx run analytics-mcp with Application Default Credentials supplied through GOOGLE_APPLICATION_CREDENTIALS, as in the README's Gemini and Claude Code setup.
**Agent surface (default):** code execution no · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 6.4 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L3 | L3 | L3 | L3 | 0.75 | — | **0.75** | High |
| C2 | Approval gates | L1 | L2 | L2 | L4 | 0.53 | — | **0.53** | High |
| C3 | Tool & action scoping | L2 | L2 | L3 | L3 | 0.60 | — | **0.60** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L2 | L2 | L3 | L3 | 0.60 | — | **0.60** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L2 | L2 | L3 | 0.47 | — | **0.47** | High |
| C9 | Audit & traceability | L0 | L1 | L1 | L0 | 0.12 | — | **0.12** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L2 | 0.30 | — | **0.30** | High |

Controls where a risk surface exists: 3.38 / 7.0 (48%); 3 criteria scored SA (surface absent).

A small, read-only server: every tool is a Google Analytics read call, and the credential is requested with the analytics read-only scope only, so a misused session can read analytics data but cannot change anything. It runs no code, loads no workspace files and keeps no memory. Its gaps are around visibility: tools carry no read-only labels for the host, report values that outsiders can write into a property come back unmarked, and the server keeps no record of the calls it makes and sets no limits of its own.

## Critical gaps
- None: no criterion is capped and no blast radius is at the worst level.

## Criterion details

### C1 Identity & least privilege — 0.75 (high)

The server has no credential of its own: it uses the Google Application Default Credentials found in the operator's environment, but it asks for them with only the Google Analytics read-only scope, and all four API clients share that one narrowed credential. Every tool only reads (account lists, property details, reports), so even a misused session can read analytics data but not change any account, property or setting. The read-only scope is a constant in code that no tool or input can change. Access still spans every Google Analytics account and property the signed-in user can see; there is no allowlist of properties.

- **S L3:** All Google API clients are built from google.auth.default() called with only the analytics.readonly OAuth scope, and every tool is a read call, so the credential is narrowed to the minimum the job needs. — [analytics_mcp/tools/client.py:49-52](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/client.py#L49-L52); [analytics_mcp/tools/client.py:78-86](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/client.py#L78-L86) (verified)
  - *To reach the next level:* No per-request or task-scoped credential (token downscoping, short-lived per-task issuance); one ADC-derived identity serves the whole run.
- **C L3:** Every tool reaches Google through one of four client factories that all take credentials from the same _get_credentials() helper; there are no extensions, sub-agents or tool-spawned processes. — [analytics_mcp/tools/client.py:89-120](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/client.py#L89-L120); [analytics_mcp/coordinator.py:74-86](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/coordinator.py#L74-L86) (verified)
  - *To reach the next level:* No property allowlist or per-request authorization check evaluated before the call; any property the ADC identity can read is in reach.
- **D L3:** The default is read-only: the scope is a hard-coded module constant with no environment variable or argument to widen it, and the package has no write path. — [analytics_mcp/tools/client.py:49-52](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/client.py#L49-L52); [analytics_mcp/tools/client.py:81-85](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/client.py#L81-L85) (verified)
  - *To reach the next level:* Which identity is used is whatever the ADC chain resolves in the operator's environment; there is no dedicated, time-bounded identity for the server.
- **B L3:** If the server is misused it can read Google Analytics configuration and report data, but cannot write anywhere; the reach is one service, read-only. — [analytics_mcp/tools/admin/info.py:31-38](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/admin/info.py#L31-L38); [analytics_mcp/tools/client.py:49-52](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/client.py#L49-L52) (verified)
  - *To reach the next level:* Reach covers every account and property the user can access rather than one property, and the underlying ADC refresh credential is long-lived.
- **Cap:** none
- **Notes:** Whether google-auth narrows an authorized_user ADC refresh to the requested scope is library behaviour (inferred); the rating rests on the server requesting only the read-only scope and calling only read RPCs. The README's sample gcloud login commands also add the cloud-platform scope to the ADC grant; the server itself never requests it.

### C2 Approval gates — 0.53 (high)

The MCP host, not this server, decides what to approve, so this rates what the server gives the host. All nine tools only read data, and nothing in the code can create, change or delete anything in Google Analytics, so there is nothing consequential for an approval to miss. However, no tool carries read-only or destructive labels, so a host cannot tell from the tool list that they are safe to auto-approve.

- **S L1:** No tool declares readOnlyHint/destructiveHint annotations, but no tool mixes reads and writes: every tool is a get/list/run-report call. — searched `rg -n 'readOnlyHint|destructiveHint|ToolAnnotations|annotations='` in `analytics_mcp` → 0 hits (No risk annotations anywhere.); [analytics_mcp/coordinator.py:92](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/coordinator.py#L92) (verified)
  - *To reach the next level:* Every tool should carry accurate readOnlyHint=true/destructiveHint=false annotations so hosts can act on them.
- **C L2:** Every tool path goes through the same dispatcher to read-only API methods with the read-only-scoped credential; unknown tool names return an error. — [analytics_mcp/coordinator.py:160-188](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/coordinator.py#L160-L188); [analytics_mcp/tools/client.py:49-52](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/client.py#L49-L52) (verified)
  - *To reach the next level:* No risk signalling is attached on any path for the host to gate on.
- **D L2:** The read-only property is fixed in code (scope constant, read-only methods only); no setting enables writes. — [analytics_mcp/tools/client.py:49-52](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/client.py#L49-L52) (verified)
  - *To reach the next level:* No annotation or server-enforced read-only mode is shipped that the default could lock in.
- **B L4:** The default configuration permits no consequential action: every tool calls a list/get/run-report read RPC and the credential is requested with the read-only scope. — [analytics_mcp/tools/reporting/core.py:171-172](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/reporting/core.py#L171-L172); [analytics_mcp/tools/admin/info.py:73-75](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/admin/info.py#L73-L75); [analytics_mcp/tools/client.py:49-52](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/client.py#L49-L52) (verified)
- **Cap:** none

### C3 Tool & action scoping — 0.60 (high)

Each tool is narrow and purpose-built (list accounts, get a property, run a report) rather than a general HTTP or query tool. The property ID is checked in code to be a number or 'properties/<number>' before it is used, and the rest of the arguments are converted into typed Google API request objects, but dimension names, filters and row limits are passed through for Google's API to validate. There is no server-side cap on rows per report or allowlist of properties. The default tool set is read-only and cannot be extended at runtime.

- **S L2:** Typed arguments are built into protobuf request objects and the property ID is format-validated in construct_property_rn; other fields (dimensions, filters, limit, offset) are passed through unchecked. — [analytics_mcp/tools/utils.py:22-44](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/utils.py#L22-L44); [analytics_mcp/tools/reporting/core.py:164-169](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/reporting/core.py#L164-L169) (verified)
  - *To reach the next level:* No numeric bound on limit/offset and no property allowlist enforced in code; validation of names and filters is left to the remote API.
- **C L2:** Every tool that takes a property ID validates it through the same helper; free-form filter and funnel arguments are not validated. — [analytics_mcp/tools/reporting/funnel.py:136-164](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/reporting/funnel.py#L136-L164); [analytics_mcp/tools/reporting/metadata.py:489-492](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/reporting/metadata.py#L489-L492) (verified)
  - *To reach the next level:* Filter expressions, segments and funnel steps are not validated by any shared layer before reaching the API.
- **D L3:** The whole default tool set is read-only and fixed at import time; there is no write or exec tool to enable and no way for the model to load new tools. — [analytics_mcp/coordinator.py:74-84](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/coordinator.py#L74-L84) (verified)
  - *To reach the next level:* No per-task tool allowlist or option to expose only a subset of tools.
- **B L3:** A misused tool can only run read-only reports and lookups against Google Analytics, bounded by the Data API's own row maximum per request. — [analytics_mcp/tools/reporting/core.py:128-131](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/reporting/core.py#L128-L131); [analytics_mcp/tools/admin/info.py:34-36](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/admin/info.py#L34-L36) (verified)
  - *To reach the next level:* No server-enforced row cap or restriction to specific properties; misuse can enumerate all of a user's properties and pull large reports.
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

The server never interprets model-written text as code: there is no shell, eval, template or script execution. The only process-related code wraps the Google auth library's own gcloud lookup so that it does not inherit the server's stdio; it takes no input from the model.

- **Structural absence:** searched `rg -n 'subprocess|eval\(|exec\(|os\.system|Popen|compile\('` in `analytics_mcp` → 5 hits (All 5 hits are in client.py's prevent_stdio_inheritance(), which only wraps subprocess.Popen so that google-auth's own gcloud lookup gets stdin=DEVNULL; no model-supplied text reaches any of them. No eval/exec/shell anywhere.); [analytics_mcp/tools/client.py:59-75](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/client.py#L59-L75)

### C5 Untrusted input blast radius — 0.60 (high)

Report results can include text that outsiders control, such as page titles, page paths, campaign names and event names sent to a property by visitors or anyone holding its public measurement ID. The server returns them as structured JSON (headers, rows, metadata) but does not mark any value as untrusted. Its tool descriptions contain usage guidance only. Because the server can only read analytics data and send requests to Google's APIs, a hijacked model can read analytics data through it but cannot use it to send data elsewhere or change anything; leaking or acting would need other tools in the host.

- **S L2:** Outputs are the API response protobufs converted to structured JSON, keeping dimension/metric headers, row values and metadata in separate fields; descriptions carry no instructions beyond tool-usage hints. — [analytics_mcp/coordinator.py:169-172](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/coordinator.py#L169-L172); [analytics_mcp/tools/utils.py:47-51](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/utils.py#L47-L51) (verified)
  - *To reach the next level:* No provenance or untrusted flag on dimension values that third parties can write into a property.
- **C L2:** Every tool returns the same structured JSON shape through one dispatcher. — [analytics_mcp/coordinator.py:160-172](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/coordinator.py#L160-L172); searched `rg -n 'untrusted|provenance|source_url'` in `analytics_mcp` → 0 hits (verified)
  - *To reach the next level:* No source is distinguished as untrusted; free-text dimension values enter model context unmarked.
- **D L3:** The output structure is fixed in code; nothing the model reads can change it. — [analytics_mcp/coordinator.py:169-172](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/coordinator.py#L169-L172) (verified)
  - *To reach the next level:* There is no provenance mechanism whose default could be locked.
- **B L3:** A hijacked session can only read analytics data through this server; its only network peer is Google's Analytics APIs and it has no state-changing action. — [analytics_mcp/tools/client.py:89-120](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/client.py#L89-L120); [analytics_mcp/tools/client.py:49-52](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/client.py#L49-L52) (verified)
  - *To reach the next level:* Reachable analytics data (revenue, ad spend, custom dimensions) is business-sensitive across all the user's properties, so it is not low-sensitivity.
- **Cap:** none

### C6 Memory, context & configuration integrity — 1.00 (high)

The server keeps no memory, retrieval store or conversation history, writes no files and loads no .env or instruction files from the working directory. Credentials and project settings come only from the standard Google auth chain in the operator's environment.

- **Structural absence:** searched `rg -n 'dotenv|open\(|\.write|json\.load|Path\(|environ|getenv|memory|cache|AGENTS'` in `analytics_mcp` → 2 hits (Both hits are 'original_popen(' substrings in client.py lines 69 and 72; no dotenv, file reads/writes, environment parsing, memory, cache or instruction-file loading exists in the package.); [analytics_mcp/tools/client.py:78-86](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/client.py#L78-L86)

### C7 Third-party extensions — 1.00 (high)

The server loads no plugins, launches no other MCP servers and downloads no code or models at runtime; its nine tools are fixed in the package.

- **Structural absence:** searched `rg -n 'importlib|entry_points|__import__|plugin|pickle|trust_remote_code|mcpServers'` in `analytics_mcp` → 1 hits (The single hit is 'from importlib import metadata' used to read the package's own version for the user agent; nothing loads plugins, MCP servers, models or downloaded code.); [analytics_mcp/coordinator.py:74-84](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/coordinator.py#L74-L84)

### C8 Secrets & sensitive-data protection — 0.47 (high)

The server never handles a key itself: credentials are loaded by the Google auth library, attached to API calls inside the client libraries, and never placed in tool output. There is no telemetry, and successful calls are not logged. There is also no masking or redaction anywhere; tool errors are passed back to the model and stderr as the raw exception text. The credential in use is narrowed to read-only analytics access, but the ADC file it comes from is a long-lived grant (the README's sample login also adds the cloud-platform scope).

- **S L1:** Credentials come from the ambient ADC chain and are never touched by server code, but nothing masks or redacts anything; exception text is returned verbatim. — [analytics_mcp/coordinator.py:174-183](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/coordinator.py#L174-L183); [analytics_mcp/tools/client.py:78-86](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/client.py#L78-L86) (verified)
  - *To reach the next level:* No redaction filter on error strings returned to the model or printed to stderr.
- **C L2:** Model-bound results contain only API response data, there is no telemetry or transcript, and no subprocess receives credentials from the server; this is by omission rather than by redaction. — searched `rg -n '(?i)sentry|telemetry|posthog|opentelemetry'` in `analytics_mcp` → 0 hits (No telemetry SDK or exporter.); [analytics_mcp/coordinator.py:174-178](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/coordinator.py#L174-L178) (verified)
  - *To reach the next level:* Error paths (model-bound and stderr) are not filtered.
- **D L2:** No telemetry exists and logging is limited to startup/shutdown lines and errors on stderr. — [analytics_mcp/server.py:31](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/server.py#L31) (verified)
  - *To reach the next level:* No redaction exists that could be always on.
- **B L3:** The in-process credential is requested with only the analytics.readonly scope and yields short-lived access tokens; the server accepts no API key of its own. — [analytics_mcp/tools/client.py:49-52](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/client.py#L49-L52); [analytics_mcp/tools/client.py:78-86](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/client.py#L78-L86) (verified)
  - *To reach the next level:* The ADC refresh grant behind it is long-lived and may carry broader scopes (the README sample adds cloud-platform).
- **Cap:** none

### C9 Audit & traceability — 0.12 (high)

The server keeps no record of what it did. Successful tool calls are not logged at all; only failed calls print a single unstructured line with the tool name and error to stderr, which the MCP host may or may not keep. There are no timestamps, arguments, caller identity or durable storage, so an incident could not be reconstructed from this server.

- **S L0:** Successful tool calls are not recorded; only errors are printed as one unstructured stderr line. — searched `rg -n 'print\(|logging|logger'` in `analytics_mcp` → 5 hits (Four hits are startup/shutdown messages in server.py; the fifth is the error print in coordinator.py. No logging module is used.); [analytics_mcp/coordinator.py:174-178](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/coordinator.py#L174-L178) (verified)
  - *To reach the next level:* Write a structured record (tool, arguments, outcome, timestamp) for every call.
- **C L1:** The single error print sits in the shared dispatcher, so failures of every tool reach it. — [analytics_mcp/coordinator.py:160-178](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/coordinator.py#L160-L178) (verified)
  - *To reach the next level:* Successful calls on every path are not recorded.
- **D L1:** The error print is always on and goes to stderr, whose retention is up to the host. — [analytics_mcp/coordinator.py:175-178](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/coordinator.py#L175-L178) (verified)
  - *To reach the next level:* Records are not written anywhere durable outside the server process by default.
- **B L0:** With no record of successful calls, actions proceed with nothing written. — [analytics_mcp/coordinator.py:165-172](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/coordinator.py#L165-L172) (verified)
  - *To reach the next level:* Records should be flushed per action and errors surfaced durably.
- **Cap:** none

### C10 Limits & kill switch — 0.30 (high)

The server sets no timeouts, size caps, rate limits or concurrency limits of its own. Report tools accept an optional row limit chosen by the model, and Google's Analytics APIs enforce their own per-request row maximum and per-property quotas, which bound how much any session can pull. Calls run in worker threads that cannot be cancelled once started, and account listings page through every result.

- **S L1:** Only caller-chosen limit/offset parameters exist on the report tools; the server enforces no timeout, cap or rate limit. — [analytics_mcp/tools/reporting/core.py:164-167](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/reporting/core.py#L164-L167); searched `rg -n 'timeout|Semaphore|rate_limit|max_rows|wait_for'` in `analytics_mcp` → 0 hits (verified)
  - *To reach the next level:* Server-enforced caps on output size or per-call timeouts.
- **C L1:** The optional limit applies to run_report, run_realtime_report and run_conversions_report only; account summaries, Ads links and annotations page through all results. — [analytics_mcp/tools/admin/info.py:34-36](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/admin/info.py#L34-L36); [analytics_mcp/tools/reporting/realtime.py:156-157](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/reporting/realtime.py#L156-L157) (verified)
  - *To reach the next level:* Bounds on listing tools and the funnel report.
- **D L1:** With no limit argument the API default row count applies, and the model can raise the limit up to the API maximum. — [analytics_mcp/tools/reporting/core.py:90](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/reporting/core.py#L90) (verified)
  - *To reach the next level:* A server default cap that the model cannot raise.
- **B L2:** Google's per-request row maximum and per-property quotas cap a runaway session outside the server, but in-flight calls run to completion in threads. — [analytics_mcp/tools/reporting/core.py:171-174](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/reporting/core.py#L171-L174); [analytics_mcp/tools/reporting/core.py:139](https://github.com/googleanalytics/google-analytics-mcp/blob/75c0e1be8322e5eaa5c2e3ef7e79cf4760060f9a/analytics_mcp/tools/reporting/core.py#L139) (verified)
  - *To reach the next level:* No cancellation of in-flight calls and no server-side rate limit.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Report dimension values (page titles, paths, campaign and event names) returned unmarked (analytics_mcp/coordinator.py:169-172) · [B] sensitive data/systems: Analytics data across every property the ADC identity can read (analytics_mcp/tools/client.py:78-86) · [C] state change / egress: None in this server: only read RPCs to Google Analytics APIs with the read-only scope (analytics_mcp/tools/client.py:49-52) · Same default session? No

## Highest-impact improvements
1. Declare readOnlyHint=true and destructiveHint=false (and openWorldHint) on all nine tools so MCP hosts can auto-approve them safely. — C2 S L1→L2, +0.075 before caps (Playbook 5)
2. Log every tool call as a structured stderr record with tool name, arguments, outcome and timestamp. — C9 S L0→L2, +0.150 before caps (Playbook 1 step 3)
3. Enforce a server-side default and maximum row limit and a per-call timeout on every API call, and cap paging in the listing tools. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)
4. Pass error text returned to the model and printed to stderr through a redaction helper. — C8 S L1→L2, +0.075 before caps (Playbook 4)
5. Offer an operator allowlist of property IDs and enforce it in construct_property_rn. — C3 S L2→L3, +0.075 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- No release tag points at the pinned commit (it is two commits after the 0.7.0 tag; pyproject.toml says 0.7.0), so version is null.
- Whether google-auth narrows an authorized_user ADC refresh to the requested analytics.readonly scope, and the GAPIC clients' default retry/timeout settings, were inferred from library behaviour, not read in the dependencies.
- The ADK FunctionTool wrapper and adk_to_mcp_tool_type conversion (google-adk) were not reviewed; argument validation inside them was not credited.
- Google's server-side enforcement (property access, quotas, row maximums) was not examined.
- skills-lock.json lists ADK documentation skills for contributors' AI tooling; it is not loaded by the server at runtime.
- No reviewer-steering text was found in the repository.
