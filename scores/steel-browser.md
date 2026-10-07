# Defense-in-Depth Score: Steel Browser

**Repo:** https://github.com/steel-dev/steel-browser · **Commit:** `e5902fc50f767702894902916ecf5489de7de457` · **Reviewed:** 2026-10-04
**What it is:** Open-source, self-hostable browser API (REST, WebSocket and Chrome DevTools) for AI agents and automation.
**Category:** AI Assistants
**Scored configuration:** Shipped docker-compose.yml, no flags, default env.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents no · external communication yes

## Score: 2.4 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C3 | Tool & action scoping | L2 | L1 | L0 | L0 | 0.23 | — | **0.23** | High |
| C4 | Code-execution isolation | L2 | L3 | L2 | L2 | 0.57 | — | **0.57** | High |
| C5 | Untrusted input blast radius | L2 | L1 | L2 | L0 | 0.33 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | Medium |
| C7 | Third-party extensions | L1 | L1 | L0 | L1 | 0.20 | — | **0.20** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C9 | Audit & traceability | L2 | L1 | L1 | L1 | 0.33 | — | **0.33** | High |
| C10 | Limits & kill switch | L2 | L1 | L1 | L1 | 0.33 | — | **0.33** | High |


Steel Browser gives whoever drives it full control of a browser that holds logged-in sessions, and the network exposure and access control of its REST API and Chrome DevTools port in the shipped configuration are not locked down. Inside the root-run container, Chrome's own sandbox and site isolation are switched off, and the API will load Chrome extensions from arbitrary paths. Only run it on a private network behind your own authenticating proxy.

## Critical gaps
- Untrusted web content driving or reaching the browser can exfiltrate data and take irreversible logged-in actions with no human involved. (ASI01, T6, LLM01; C5) — [api/src/modules/actions/actions.controller.ts:111-113](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/modules/actions/actions.controller.ts#L111-L113)

## Criterion details

### C1 Identity & least privilege — 0.05 (high)

Steel Browser has no per-caller identity or separation between callers: whoever drives it gets full control of the single shared browser, including every logged-in website session it holds. Access control on its HTTP, WebSocket and Chrome DevTools endpoints in the shipped configuration is not locked down.

- **S L0:** No dedicated identity scopes callers; a caller acts with the server's full authority over the browser and its logged-in sessions. (verified)
  - *To reach the next level:* Requires at least a dedicated API key or token scoping who may drive the browser.
- **C L0:** No authorization layer covers the API paths. — [api/src/plugins/browser-socket/browser-socket.ts:61-64](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/plugins/browser-socket/browser-socket.ts#L61-L64) (verified)
  - *To reach the next level:* An authorization check would have to cover the main API path.
- **D L0:** The shipped configuration has no narrower default identity. (verified)
  - *To reach the next level:* Needs a narrower default.
- **B L1:** A hijacked caller can act as every account logged into the browser across many sites and reach the container's network, but holds no cloud or cluster credentials. — [api/src/modules/actions/actions.controller.ts:111-113](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/modules/actions/actions.controller.ts#L111-L113) (verified)
  - *To reach the next level:* Would require authority limited to one site or tenant.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

As a tool server, Steel Browser does not give the host anything to gate on: endpoints carry no read-only or destructive hints, there is no dry-run, and the raw Chrome DevTools connection mixes reading and every possible browser action in one channel. Any irreversible action on a website (submitting forms, purchases, posts) is one CDP call away with no server-side confirmation.

- **S L0:** No risk signalling; the CDP passthrough mixes reads and writes in one tool. — searched `rg -n -i 'readOnlyHint|destructiveHint|x-read-only|dryRun|dry_run'` in `api/src` → 0 hits (No risk annotations, dry-run or read-only mode on any endpoint.); [api/src/plugins/browser-socket/browser-socket.ts:61-64](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/plugins/browser-socket/browser-socket.ts#L61-L64) (verified)
  - *To reach the next level:* Would need risk annotations on endpoints.
- **C L0:** The most powerful path, raw CDP over WebSocket, is entirely ungated. — [api/src/plugins/browser-socket/browser-socket.ts:61-64](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/plugins/browser-socket/browser-socket.ts#L61-L64); [api/src/services/cdp/cdp.service.ts:927-928](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/cdp/cdp.service.ts#L927-L928) (verified)
  - *To reach the next level:* The CDP path would have to be covered by some server-side gate or mode.
- **D L0:** No gating or read-only mode exists to enable. — searched `rg -n -i 'readOnlyHint|destructiveHint|x-read-only|dryRun|dry_run'` in `api/src` → 0 hits (No risk annotations, dry-run or read-only mode on any endpoint.) (verified)
  - *To reach the next level:* Would need a gate on by default.
- **B L0:** Browser actions on third-party sites are irreversible and unbounded. — [api/src/plugins/browser-socket/browser-socket.ts:61-64](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/plugins/browser-socket/browser-socket.ts#L61-L64) (verified)
  - *To reach the next level:* Would need reversible or previewed actions.
- **Cap:** none

### C3 Tool & action scoping — 0.23 (high)

Request bodies are typed with Zod schemas and the file API checks that resolved paths stay inside its base directory, and named extensions are matched against a bundled directory. But URLs are passed straight to the browser and to a server-side fetch that follows redirects, with no block on localhost or cloud metadata addresses, and the session API accepts arbitrary extension paths and Chrome preferences. The full CDP channel is on by default, so the effective tool is 'do anything a browser can do'.

- **S L2:** Typed schemas plus path containment via resolve/startsWith (no realpath) and a named-extension allowlist. — [api/src/services/file.service.ts:104-110](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/file.service.ts#L104-L110); [api/src/utils/url.ts:51-55](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/utils/url.ts#L51-L55); [api/src/utils/extensions.ts:22](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/utils/extensions.ts#L22) (verified)
  - *To reach the next level:* Needs URL host allowlisting that blocks internal addresses and rechecks redirects.
- **C L1:** Only the file API and named extensions validate; URLs, orgExtensions paths, userPreferences and CDP do not. — searched `rg -n -i '169\.254|isPrivate|ssrf'` in `api/src` → 0 hits (No internal-address or host allowlist check anywhere.); [api/src/services/cdp/cdp.service.ts:822-823](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/cdp/cdp.service.ts#L822-L823); [api/src/modules/sessions/sessions.schema.ts:85-88](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/modules/sessions/sessions.schema.ts#L85-L88) (verified)
  - *To reach the next level:* Most endpoints would need validation.
- **D L0:** Every capability including raw CDP is enabled by default. — [api/src/services/cdp/cdp.service.ts:927-928](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/cdp/cdp.service.ts#L927-L928) (verified)
  - *To reach the next level:* Dangerous capabilities would need to be individually disableable.
- **B L0:** A misused endpoint reaches any host, including internal services, through a browser holding logged-in sessions. — [api/src/modules/actions/actions.controller.ts:111-113](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/modules/actions/actions.controller.ts#L111-L113); searched `rg -n -i '169\.254|isPrivate|ssrf'` in `api/src` → 0 hits (No internal-address or host allowlist check anywhere.) (verified)
  - *To reach the next level:* Would need scoping to specific hosts or sites.
- **Cap:** none

### C4 Code-execution isolation — 0.57 (high)

Chromium executes untrusted page JavaScript and caller-supplied scripts via CDP inside the shipped Docker container. That container runs as root with no hardening, and because it runs as root the code automatically disables Chrome's own sandbox and also turns off site isolation. Chrome's environment is scrubbed to a few variables, but the container has unrestricted network access and a read-write bind mount of the host's ./.cache directory.

- **S L2:** The boundary is a stock Docker container running as root with default capabilities; Chrome's sandbox is disabled. — searched `rg -n '^USER'` in `Dockerfile api/Dockerfile` → 0 hits (No USER directive: the container (and Chromium) run as root.); searched `rg -n 'cap_drop|security_opt|read_only|mem_limit|user:'` in `docker-compose.yml` → 0 hits (Compose service has no user, capability, read-only or resource hardening.); [api/src/services/cdp/cdp.service.ts:865-867](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/cdp/cdp.service.ts#L865-L867) (verified)
  - *To reach the next level:* Needs a hardened container: non-root, dropped capabilities, no-new-privileges, seccomp.
- **C L3:** All browser, Selenium and server-side fetch execution runs inside the container image; running outside Docker is the documented dev escape hatch. — [docker-compose.yml:3](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/docker-compose.yml#L3) (verified)
  - *To reach the next level:* Processes spawned by the browser would need confinement that fails closed rather than silently dropping the Chrome sandbox.
- **D L2:** The Chrome sandbox is silently disabled whenever the process is root, which is the shipped default. — [api/src/services/cdp/cdp.service.ts:865-867](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/cdp/cdp.service.ts#L865-L867); [api/src/services/cdp/cdp.service.ts:899-900](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/cdp/cdp.service.ts#L899-L900) (verified)
  - *To reach the next level:* Disabling the sandbox would need an explicit operator flag rather than an automatic fallback.
- **B L2:** Inside: no secrets in Chrome's env, but unrestricted network egress and a read-write host bind mount. — [api/src/services/cdp/cdp.service.ts:957-961](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/cdp/cdp.service.ts#L957-L961); [docker-compose.yml:10](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/docker-compose.yml#L10); searched `rg -n 'cap_drop|security_opt|read_only|mem_limit|user:'` in `docker-compose.yml` → 0 hits (Compose service has no user, capability, read-only or resource hardening.) (verified)
  - *To reach the next level:* Would need network egress off or allowlisted and resource limits.
- **Cap:** none

### C5 Untrusted input blast radius — 0.25 (high)

Steel Browser's whole job is to load untrusted web pages for an agent. Scrape output separates content from metadata and records the source URL, but nothing marks content as untrusted or offers a read-only or no-egress mode. If an agent driving it is hijacked by page content, it can browse anywhere, use logged-in cookies, and submit forms with no server-side check.

- **S L2:** Structured scrape responses separate content from metadata with a urlSource field. — [api/src/modules/actions/actions.schema.ts:23-24](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/modules/actions/actions.schema.ts#L23-L24); [api/src/modules/actions/actions.controller.ts:168](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/modules/actions/actions.controller.ts#L168) (verified)
  - *To reach the next level:* Needs an explicit untrusted flag on content and provenance on every output.
- **C L1:** Only the scrape endpoint carries provenance; screenshots, PDFs and CDP results do not. — [api/src/modules/actions/actions.controller.ts:168](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/modules/actions/actions.controller.ts#L168); [api/src/plugins/browser-socket/browser-socket.ts:61-64](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/plugins/browser-socket/browser-socket.ts#L61-L64) (verified)
  - *To reach the next level:* Most output paths would need provenance.
- **D L2:** Structured output is always on but nothing can be enabled to drop a Rule-of-Two leg. — [api/src/modules/actions/actions.schema.ts:23-24](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/modules/actions/actions.schema.ts#L23-L24) (verified)
  - *To reach the next level:* Would need a warned, explicit opt-out of a protective mode.
- **B L0:** A hijacked session can exfiltrate via any URL and take irreversible logged-in actions unattended. — [api/src/modules/actions/actions.controller.ts:111-113](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/modules/actions/actions.controller.ts#L111-L113) (verified)
  - *To reach the next level:* Would need egress or irreversible actions to require approval.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.00 (medium)

Steel has no AI memory store or instruction files, but the browser profile is its persistence: by default every session reuses the same Chrome user-data directory and nothing clears cookies or site storage between sessions or callers. Anything a malicious page plants (cookies, local storage, service workers) can carry into later sessions for any caller of the shared server. Uploaded files are cleaned on shutdown.

- **S L0:** Pages can write arbitrary persistent browser state into a reused profile with no validation. — [api/src/services/cdp/cdp.service.ts:161](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/cdp/cdp.service.ts#L161); searched `rg -n -i 'clearBrowserCookies|clearDataForOrigin'` in `api/src` → 0 hits (Nothing clears cookies or site data between sessions.) (verified)
  - *To reach the next level:* Would need at least logging or per-session fresh profiles.
- **C L0:** No persistence path (profile, persist=true user-data-dir) is controlled. — [api/src/services/cdp/cdp.service.ts:161](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/cdp/cdp.service.ts#L161); [api/src/services/session.service.ts:180-182](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/session.service.ts#L180-L182) (verified)
  - *To reach the next level:* At least the browser profile would need control.
- **D L0:** One profile is shared across all sessions and all callers by default. — [api/src/services/cdp/cdp.service.ts:161](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/cdp/cdp.service.ts#L161) (verified)
  - *To reach the next level:* Would need per-session isolation by default.
- **B L0:** Poisoned browser state persists across sessions and callers and can drive later browsing actions (inferred from Chrome persisting state to the reused profile). — [api/src/services/cdp/cdp.service.ts:161](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/cdp/cdp.service.ts#L161); searched `rg -n -i 'clearBrowserCookies|clearDataForOrigin'` in `api/src` → 0 hits (Nothing clears cookies or site data between sessions.) (inferred)
  - *To reach the next level:* Would need state to be session-scoped or easily purged.
- **Cap:** none

### C7 Third-party extensions — 0.20 (high)

Chrome extensions are the extension surface. Named extensions are restricted to a directory bundled in the image, which by default holds only Steel's own recorder. However, the session API also accepts arbitrary filesystem paths through extra.orgExtensions.paths and loads them with no integrity check. Such an extension runs in the root-owned, unsandboxed browser with access to all sites.

- **S L1:** Extensions come from caller-chosen local paths with no pinning or verification. — [api/src/services/cdp/cdp.service.ts:822-823](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/cdp/cdp.service.ts#L822-L823); searched `rg -n -i 'sha256|integrity|signature|verify'` in `api/src/utils/extensions.ts api/src/services/cdp/cdp.service.ts` → 0 hits (Extension paths are loaded with no hash or signature check.) (verified)
  - *To reach the next level:* Would need pinned or verified extension sources.
- **C L1:** Only named extensions are constrained to the bundled directory; path-based ones are not. — [api/src/utils/extensions.ts:22](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/utils/extensions.ts#L22); [api/src/services/cdp/cdp.service.ts:822-823](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/cdp/cdp.service.ts#L822-L823) (verified)
  - *To reach the next level:* All extension types would need verification.
- **D L0:** Any API caller can add an extension per session with no consent step. — [api/src/modules/sessions/sessions.schema.ts:85-88](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/modules/sessions/sessions.schema.ts#L85-L88); [api/src/services/cdp/cdp.service.ts:822-823](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/cdp/cdp.service.ts#L822-L823) (verified)
  - *To reach the next level:* Would need an explicit operator consent step for adding extensions.
- **B L1:** Extensions run in the same root Chrome as all browsing data, in a separate renderer process. — [api/src/services/cdp/cdp.service.ts:858-861](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/cdp/cdp.service.ts#L858-L861); [api/src/services/cdp/cdp.service.ts:865-867](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/cdp/cdp.service.ts#L865-L867) (verified)
  - *To reach the next level:* Would need a scrubbed, scoped environment per extension.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.45 (high)

Network header and body logging is off by default, the interaction recorder omits typed values and flags password fields, and no telemetry exporter is configured. But API responses expose sensitive session material. Browser cookies are long-lived account sessions.

- **S L2:** Log minimisation on main paths: request bodies and headers off unless explicitly enabled, input values omitted with a sensitive flag. — [api/src/services/cdp/instrumentation/network-events.ts:29](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/cdp/instrumentation/network-events.ts#L29); [api/src/services/cdp/instrumentation/browser-interaction-script.ts:293-294](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/cdp/instrumentation/browser-interaction-script.ts#L293-L294) (verified)
  - *To reach the next level:* Needs redaction before outputs and secret storage encrypted at rest.
- **C L2:** CDP network and interaction logs are minimised, but API responses are not protected. (verified)
  - *To reach the next level:* API responses and model-bound outputs would need protection too.
- **D L2:** No telemetry exporter; dangerous logging is opt-in. — searched `rg -n -i 'otlp|exporter|NodeSDK'` in `api/src` → 0 hits (OpenTelemetry API is used only as an optional no-op tracer; no exporter is configured.); [api/src/services/cdp/instrumentation/network-events.ts:29](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/cdp/instrumentation/network-events.ts#L29) (verified)
  - *To reach the next level:* Redaction would need to be always on.
- **B L1:** Leaked material is long-lived site session cookies, moderately scoped per site. (verified)
  - *To reach the next level:* Would need short-lived or scoped credentials.
- **Cap:** none

### C9 Audit & traceability — 0.33 (high)

Every API request is logged with client IP, URL, method and status, and the browser instrumentation records navigations, network and console events, which compose persists to a DuckDB file. Individual CDP commands sent by a client are not recorded as such, there is no caller identity beyond IP, and the stored logs are not protected from the clients they record. Writes are buffered and flushed every two seconds.

- **S L2:** Structured request log and structured browser event records with timestamps. — [api/src/plugins/request-logger.ts:29-31](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/plugins/request-logger.ts#L29-L31); [api/src/plugins/browser.ts:41-43](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/plugins/browser.ts#L41-L43) (verified)
  - *To reach the next level:* Needs actor attribution beyond client IP.
- **C L1:** REST requests and browser-side events are recorded; raw CDP commands are not. — [api/src/plugins/request-logger.ts:29-31](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/plugins/request-logger.ts#L29-L31); [api/src/plugins/browser-socket/browser-socket.ts:61-64](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/plugins/browser-socket/browser-socket.ts#L61-L64) (verified)
  - *To reach the next level:* Every action path including CDP commands would need recording.
- **D L1:** On by default (DuckDB in compose), but API clients can delete it. — [docker-compose.yml:14](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/docker-compose.yml#L14); [api/src/modules/logs/logs.routes.ts:149-158](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/modules/logs/logs.routes.ts#L149-L158) (verified)
  - *To reach the next level:* Logs would need to be out of reach of the clients they record.
- **B L1:** Buffered writes flushed every 2s or 200 events; failures are logged and actions proceed. — [api/src/plugins/browser.ts:41-43](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/plugins/browser.ts#L41-L43) (verified)
  - *To reach the next level:* Records would need flushing per action.
- **Cap:** none

### C10 Limits & kill switch — 0.33 (high)

Navigation in the action endpoints times out after 30 seconds, request bodies are capped at 100 MB and file storage per session at 100 MB, and releasing a session kills the Chrome process. But sessions have no timeout by default, the KILL_TIMEOUT setting is never read, delays are unbounded, and there are no rate limits despite the architecture docs claiming them.

- **S L2:** Server-enforced caps on some operations: goto timeout, body size, per-session file storage; release kills Chrome. — [api/src/modules/actions/actions.controller.ts:79](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/modules/actions/actions.controller.ts#L79); [api/src/index.ts:16](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/index.ts#L16); [api/src/services/cdp/cdp.service.ts:521](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/cdp/cdp.service.ts#L521) (verified)
  - *To reach the next level:* Would need caps on every operation plus rate or concurrency limits.
- **C L1:** Timeouts apply to the REST action endpoints only, not CDP sessions. — [api/src/modules/actions/actions.controller.ts:79](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/modules/actions/actions.controller.ts#L79); [api/src/services/session.service.ts:34](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/session.service.ts#L34) (verified)
  - *To reach the next level:* Would need limits that also cover CDP-driven sessions.
- **D L1:** Session timeout defaults to 0 (unlimited) and KILL_TIMEOUT is unused. — [api/src/services/session.service.ts:34](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/session.service.ts#L34); searched `rg -n 'KILL_TIMEOUT'` in `api/src` → 1 hits (Defined in env.ts with default 0 and never read anywhere.) (verified)
  - *To reach the next level:* Would need a sensible default session time limit.
- **B L1:** A session can run indefinitely until a client releases it; nothing outside bounds it. — [api/src/services/session.service.ts:34](https://github.com/steel-dev/steel-browser/blob/e5902fc50f767702894902916ecf5489de7de457/api/src/services/session.service.ts#L34); searched `rg -n -i 'rate.?limit'` in `api/src api/package.json` → 0 hits (No rate limiter despite docs/ARCHITECTURE.md claiming request rate limiting.) (verified)
  - *To reach the next level:* Would need moderate ceilings on session lifetime.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Arbitrary web pages loaded by the browser (api/src/utils/scrape/safeGoTo.ts:94) · [B] sensitive data/systems: Logged-in cookies and local storage held in the browser profile · [C] state change / egress: Unrestricted navigation, form submission via CDP and server-side fetch (api/src/modules/actions/actions.controller.ts:111) · Same default session? Yes

## Highest-impact improvements
1. Harden default network exposure and access control for the REST, WebSocket and CDP endpoints. — C1 D L0→L2, +0.100 before caps (Playbook 4)
2. Tighten cross-origin access to the API and CDP endpoints. — C5 B L0→L1, +0.050 before caps (Playbook 1)
3. Run the container as a non-root user so Chrome's sandbox stays on, and stop disabling site isolation. — C4 S L2→L3, +0.075 before caps (Playbook 3)
4. Remove or operator-gate extra.orgExtensions.paths so callers cannot load arbitrary extensions. — C7 D L0→L3, +0.150 before caps (Playbook 3)
5. Use a fresh user-data directory per session unless persistence is explicitly requested. — C6 D L0→L2, +0.100 before caps (Playbook 2)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The ui/ frontend and the vendored Selenium/ChromeDriver binaries were not examined in depth.
- One cross-origin finding is inferred, not tested.
- Cross-session persistence of cookies via the reused profile is inferred from Chrome's standard on-disk profile behaviour.
- No text aimed at AI reviewers was found in the repository.
