# Defense-in-Depth Score: Chrome DevTools MCP

**Repo:** https://github.com/ChromeDevTools/chrome-devtools-mcp · **Commit:** `b2f522c8ba0fd2e00a679159b4aa5243de5f1b78` · **Reviewed:** 2026-10-03
**What it is:** Chrome DevTools for coding agents (debug, inspect, automate browser)
**Category:** AI Assistants
**Scored configuration:** stdio MCP server started as `npx -y chrome-devtools-mcp@latest` with no flags: launches Chrome with the persistent ~/.cache profile, default tool categories, JavaScript evaluation and file: navigation on, telemetry on.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials opt-in · persistent memory no · untrusted input yes · third party extensions opt-in · sub agents no · external communication yes

## Score: 2.7 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | Medium |
| C2 | Approval gates | L2 | L3 | L2 | L1 | 0.53 | — | **0.53** | Medium |
| C3 | Tool & action scoping | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C4 | Code-execution isolation | L2 | L3 | L1 | L1 | 0.47 | G2 | **0.25** | Medium |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | Medium |
| C6 | Memory, context & configuration integrity | L0 | L0 | L0 | L0 | 0.00 | C6-REPOCONFIG | **0.00** | Medium |
| C7 | Third-party extensions | L1 | L0 | L0 | L0 | 0.07 | — | **0.07** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L1 | L0 | 0.35 | — | **0.35** | Medium |
| C9 | Audit & traceability | L1 | L2 | L0 | L1 | 0.28 | G1 | **0.28** | High |
| C10 | Limits & kill switch | L2 | L2 | L3 | L2 | 0.55 | — | **0.55** | High |


Chrome DevTools MCP gives an agent a real Chrome browser with JavaScript execution, navigation (including local file: URLs) and input on by default, and it returns page content to the model unlabelled. Its own file writes are well confined to the temp directory or client roots, but a hijacked agent can still read local secrets through the browser, send data anywhere and act on any site signed into the persistent profile, with nothing in the server asking a human. The biggest structural problem is configuration integrity: security-relevant configuration is not integrity-protected in how it is sourced and applied.

## Critical gaps
- Worst case under prompt injection: default tools allow reading local files via file: URLs and unredacted headers, exfiltrating via scripted fetch, and irreversible actions on signed-in sites, with no server-side gate (C5-WORSTCASE). (ASI01, LLM01, T6; C5) — [src/config/mcp-options.ts:134-136](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/mcp-options.ts#L134-L136); [src/tools/script.ts:230-245](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/tools/script.ts#L230-L245); [src/config/mcp-options.ts:175-180](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/mcp-options.ts#L175-L180)

## Criterion details

### C1 Identity & least privilege — 0.25 (medium)

The server holds no API keys or tokens of its own and does not forward client tokens. By default it launches Chrome with a dedicated profile under ~/.cache/chrome-devtools-mcp rather than the user's everyday Chrome profile, which is the only real narrowing of authority. That profile is persistent and shared across every session and project, the Chrome process inherits the full environment, and the browser can open any local file the OS user can read. Configuration handling that affects the profile, executable and connection target is not integrity-protected.

- **S L1:** A dedicated persistent Chrome profile separates the agent's browser identity from the user's daily profile, but everything else runs with the OS user's ambient authority. — [src/BrowserManager.ts:229-236](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/BrowserManager.ts#L229-L236); [src/config/browser-options.ts:89-94](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/browser-options.ts#L89-L94) (verified)
  - *To reach the next level:* No per-capability scoping: read-only tools and JS/navigation tools share the same browser session and OS-user authority.
- **C L1:** Server-side file writes are confined to roots, but browser navigation to file: URLs (default on) reaches every file the user can read, so the main browser path uses ambient OS authority. — [src/config/mcp-options.ts:134-136](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/mcp-options.ts#L134-L136); [src/utils/url.ts:212](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/utils/url.ts#L212); [src/McpContext.ts:286-289](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/McpContext.ts#L286-L289) (verified)
  - *To reach the next level:* Browser-side access (file: navigation, network) is not routed through the same root/authorization checks as the server's own file IO.
- **D L1:** The narrower default (dedicated profile) can be widened through configuration that is not integrity-protected. — [src/BrowserManager.ts:196-204](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/BrowserManager.ts#L196-L204) (verified)
  - *To reach the next level:* Security-relevant identity settings should only come from user/operator scope.
- **B L1:** If misused, the server reaches every web account signed into the persistent profile (with write) plus read access to the user's files via file: URLs; Chrome page JS cannot write host files directly. — [src/BrowserManager.ts:229-236](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/BrowserManager.ts#L229-L236); [src/config/mcp-options.ts:134-136](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/mcp-options.ts#L134-L136) (inferred)
  - *To reach the next level:* Would need the browser to be confined to one site/tenant and mostly read-only access.
- **Cap:** none

### C2 Approval gates — 0.53 (medium)

As a tool server, the host owns approval; this server's contribution is risk signalling. Every tool definition must declare readOnlyHint and the registration passes it to the client, and powerful tools (evaluate_script, navigation, input, file-writing tools) are marked not read-only. There is no destructiveHint, no preview or dry-run, and no server-enforced read-only mode or confirmation step. A wrongly approved call can submit forms or act on any site signed into the persistent profile, with no undo.

- **S L2:** Read and write tools are separated with a mandatory readOnlyHint on every tool (non-read-only tools fall back to the MCP default destructiveHint=true). — [src/tools/ToolDefinition.ts:86](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/tools/ToolDefinition.ts#L86); [src/index.ts:321](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/index.ts#L321); [src/tools/script.ts:21-26](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/tools/script.ts#L21-L26); searched `rg -n 'destructiveHint|dryRun|dry_run' --glob '!src/third_party/**'` in `src` → 0 hits (No destructive hints or dry-run modes anywhere in the server source.) (verified)
  - *To reach the next level:* No preview/dry-run for destructive operations (e.g. evaluate_script, form fill, close_page).
- **C L3:** Annotations are a required field of the tool type and passed for every registered tool; generic executors for page-provided tools (WebMCP, third-party developer tools) are flagged not read-only and off by default. — [src/tools/ToolDefinition.ts:86](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/tools/ToolDefinition.ts#L86); [src/index.ts:321](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/index.ts#L321); [src/tools/webmcp.ts:28-33](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/tools/webmcp.ts#L28-L33) (verified)
  - *To reach the next level:* Unknown/page-provided tools behind execute_webmcp_tool are not individually annotated, and there is no server-side reject-by-default.
- **D L2:** Annotations are always shipped and the model cannot change them, but extra mutating tool categories (extensions, third-party, WebMCP) can be enabled through configuration that is not integrity-protected. — [src/config/category-options.ts:74](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/category-options.ts#L74) (verified)
  - *To reach the next level:* Tool-set widening should require an explicit operator flag.
- **B L1:** Approved browser actions (form submission, clicks, scripted fetches) on signed-in sites are generally irreversible; server-side file writes are confined to the temp dir/roots. — [src/tools/script.ts:230-245](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/tools/script.ts#L230-L245); [src/config/mcp-options.ts:11](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/mcp-options.ts#L11) (inferred)
  - *To reach the next level:* No checkpoints, previews, or quantity bounds on browser actions.
- **Cap:** none

### C3 Tool & action scoping — 0.45 (high)

File paths handled by the server itself are validated well: canonical realpath containment against client roots or the OS temp directory, and writes opened with O_NOFOLLOW and mode 0600. URLs, however, are only checked against a small scheme denylist, evaluate_script accepts arbitrary JavaScript, and file: navigation is allowed by default, which lets the browser read files the path checks would have refused. The default tool set includes JavaScript execution, navigation and input; categories can be disabled but most are on.

- **S L2:** Server file paths use realpath containment and O_NOFOLLOW, but URLs get only a scheme denylist and evaluate_script/initScript take raw JavaScript, so the path control is escapable through the browser. — [src/McpContext.ts:286-289](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/McpContext.ts#L286-L289); [src/McpContext.ts:696-702](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/McpContext.ts#L696-L702); [src/utils/url.ts:101-111](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/utils/url.ts#L101-L111); [src/tools/script.ts:230-245](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/tools/script.ts#L230-L245) (verified)
  - *To reach the next level:* No URL/host allowlist or internal-address blocking by default; evaluate_script is a general passthrough.
- **C L2:** All tools get strict zod schemas and a shared file-path validation layer (verifyFilesSchema), but URL validation is per-tool and browser-originated file/network access is outside it. — [src/ToolHandler.ts:146-156](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/ToolHandler.ts#L146-L156); [src/ToolHandler.ts:203](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/ToolHandler.ts#L203); [src/utils/url.ts:212](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/utils/url.ts#L212) (verified)
  - *To reach the next level:* Browser navigations and script-initiated requests should pass the same central policy (roots, URL allowlist).
- **D L2:** Tool categories are selectable, but the default set enables input, navigation, network, and JavaScript evaluation; only extensions, PWA, third-party and WebMCP are off. — [src/config/category-options.ts:74](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/category-options.ts#L74); [src/config/mcp-options.ts:128-130](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/mcp-options.ts#L128-L130); [src/config/mcp-options.ts:134-136](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/mcp-options.ts#L134-L136) (verified)
  - *To reach the next level:* Default to a read-only/inspection tool set with JS evaluation and file navigation opt-in.
- **B L1:** A misused tool can run arbitrary JS against any host, read any local file via file: URLs, and act on any site; only server-side writes are confined to temp/roots. — [src/config/mcp-options.ts:134-136](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/mcp-options.ts#L134-L136); [src/tools/script.ts:230-245](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/tools/script.ts#L230-L245); [src/config/mcp-options.ts:11](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/mcp-options.ts#L11) (verified)
  - *To reach the next level:* Default reach should be scoped (URL allowlist, no file: navigation) with bounded operations.
- **Cap:** none

### C4 Code-execution isolation — 0.25 (medium)

Model-written JavaScript (evaluate_script, initScript, javascript: URLs) runs inside Chrome's page renderer rather than in the Node server, so the isolation comes from Chrome's own renderer sandbox, which this repo leaves enabled by default. That code still runs with the page's network access and the persistent profile's cookies. The sandbox is controlled by Chrome flags, and how launch options are sourced is not integrity-protected, which caps this criterion.

- **S L2:** Execution happens in Chrome's sandboxed renderer (the repo does not add --no-sandbox by default); the sandbox itself is Chrome's, not code in this repo, so its strength is inferred. — [src/tools/script.ts:230-245](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/tools/script.ts#L230-L245); [src/BrowserManager.ts:99-105](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/BrowserManager.ts#L99-L105) (inferred)
  - *To reach the next level:* The repo does not itself define or verify a hardened sandbox policy (network denial, filesystem limits) for executed code.
- **C L3:** Every model-reachable code path (evaluate_script, initScript, javascript: navigation) executes in the browser, none in the Node process; the --no-sandbox escape hatch is documented. — [src/tools/script.ts:230-245](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/tools/script.ts#L230-L245); [src/tools/pages.ts:220-223](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/tools/pages.ts#L220-L223); [src/config/mcp-options.ts:294](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/mcp-options.ts#L294) (verified)
  - *To reach the next level:* No fail-closed check that the renderer sandbox is actually active.
- **D L1:** The sandbox is on by default, but the sourcing of sandbox-affecting launch options is not integrity-protected. (verified)
  - *To reach the next level:* Sandbox-affecting launch options should only come from operator/user scope.
- **B L1:** Code inside the renderer has full network egress plus the cookies of the persistent profile, and can navigate to file: URLs. — [src/config/mcp-options.ts:134-136](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/mcp-options.ts#L134-L136); [src/BrowserManager.ts:229-236](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/BrowserManager.ts#L229-L236); [src/config/puppeteer-options.ts:56-60](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/puppeteer-options.ts#L56-L60) (verified)
  - *To reach the next level:* Would need network egress limited/allowlisted and no credentials in the browser profile by default.
- **Cap:** G2 — Sandbox-affecting launch options are not integrity-protected.

### C5 Untrusted input blast radius — 0.00 (medium)

Page content (accessibility snapshots, console messages, network bodies, dialog text) is returned to the model as plain markdown-like text, in places right next to the server's own instructions, with no untrusted marker. Structured output exists but is experimental and off by default, and the project's security policy tells users to use trusted content or rely on the client. If a page hijacks the agent, the default tools let it read local files through file: URLs and read unredacted cookies and headers, send data anywhere with scripted fetches, and act on signed-in sites, all without the server asking anyone.

- **S L0:** Outputs mix untrusted page text with server directives to the model (e.g. a page-controlled dialog message followed by 'Call handle_dialog ... before continuing'). — [src/McpResponse.ts:1006-1008](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/McpResponse.ts#L1006-L1008); [src/McpResponse.ts:1183-1188](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/McpResponse.ts#L1183-L1188); searched `rg -n -i 'untrusted|prompt.injection' --glob '!src/third_party/**'` in `src` → 2 hits (Both hits are the chrome-untrusted: URL scheme in url.ts, not content labelling.) (verified)
  - *To reach the next level:* Separate returned content from server instructions in structured outputs by default.
- **C L0:** No source of untrusted content (pages, console, network bodies, dialogs) is distinguished from the server's own text. — [src/McpResponse.ts:1183-1188](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/McpResponse.ts#L1183-L1188); [SECURITY.md:14-17](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/SECURITY.md#L14-L17) (verified)
  - *To reach the next level:* Mark every page-derived field as untrusted with its source URL.
- **D L0:** The only partial mitigation (structured content) is experimental and off by default. — [src/config/mcp-options.ts:58-61](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/mcp-options.ts#L58-L61) (verified)
  - *To reach the next level:* Ship provenance-tagged structured output on by default.
- **B L0:** A hijacked host can, with default tools, read local secrets via file: URLs and unredacted headers, exfiltrate via evaluate_script fetch/navigation, and take irreversible actions on signed-in sites, with nothing in the server requiring a human. — [src/config/mcp-options.ts:134-136](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/mcp-options.ts#L134-L136); [src/config/mcp-options.ts:175-180](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/mcp-options.ts#L175-L180); [src/tools/script.ts:230-245](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/tools/script.ts#L230-L245) (inferred)
  - *To reach the next level:* The server would need a mode that drops a Rule-of-Two leg (e.g. no egress or no file access) on by default.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.00 (medium)

The server has no memory store. Configuration discovery and loading does not protect the integrity of security-relevant server options.

- **S L0:** Configuration integrity controls are not tamper-resistant. (verified)
  - *To reach the next level:* Integrity-protect security-relevant configuration on every path.
- **C L0:** No config path is controlled. (verified)
  - *To reach the next level:* Integrity-protect security-relevant configuration on every path.
- **D L0:** The default browser profile is persistent and shared across all sessions and projects (isolated defaults to false). — [src/config/browser-options.ts:89-94](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/browser-options.ts#L89-L94); [src/BrowserManager.ts:229-236](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/BrowserManager.ts#L229-L236) (verified)
  - *To reach the next level:* Default to an isolated, per-session profile.
- **B L0:** Poisoned configuration can persist and affect every user of a checkout; project-scoped configuration is not integrity-protected. (inferred)
  - *To reach the next level:* Poisoned configuration should be session-scoped or require review.
- **Cap:** C6-REPOCONFIG — Security-relevant configuration is not integrity-protected.

### C7 Third-party extensions — 0.07 (high)

By default the server loads no plugins, but it has several paths for running third-party code: installing unpacked Chrome extensions, executing tools a web page exposes, WebMCP tools, an ffmpeg binary, and any Chrome executable path. The extension and page-tool categories are off by default. Nothing is pinned or integrity-checked, and configuration that can set the executable or enable these categories is not integrity-protected. A substituted executable runs as the same user with the full environment.

- **S L1:** Executables and extensions are taken from user/config-chosen paths with no pinning or integrity checks. — [src/BrowserManager.ts:269-285](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/BrowserManager.ts#L269-L285); [src/tools/extensions.ts:19-30](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/tools/extensions.ts#L19-L30); searched `rg -n -i 'sha256|integrity|signature|checksum'` in `src/tools src/BrowserManager.ts src/config` → 0 hits (No verification of executables or extensions.) (verified)
  - *To reach the next level:* Pin or hash-verify launched executables and extensions.
- **C L0:** No extension type (executable, Chrome extension, page-provided tools, ffmpeg) is verified. — [src/tools/thirdPartyDeveloper.ts:66-72](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/tools/thirdPartyDeveloper.ts#L66-L72) (verified)
  - *To reach the next level:* Verify every launched or loaded component.
- **D L0:** Configuration that can set the executable or enable the extensions/third-party categories is not integrity-protected. — [src/config/category-options.ts:44-53](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/category-options.ts#L44-L53) (verified)
  - *To reach the next level:* Only user or admin scope should be able to add executables or extension categories, with the exact command shown.
- **B L0:** A substituted Chrome executable is launched by Puppeteer as the same OS user, with the server's environment. — [src/BrowserManager.ts:269-285](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/BrowserManager.ts#L269-L285); [src/BrowserManager.ts:207-222](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/BrowserManager.ts#L207-L222) (verified)
  - *To reach the next level:* Launch third-party binaries in a separate, scrubbed or sandboxed context.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.35 (medium)

Usage telemetry is on by default but content-free: string parameters are reduced to length buckets and only tool names, latency and success are sent. Header redaction for network requests exists but is off by default, so cookies and Authorization headers go to the model provider in the default flow. The optional debug log records full tool parameters, and connection credentials are not reliably kept out of it. Because file: navigation is allowed, the model can also read long-lived keys such as SSH or cloud credentials from the user's home directory.

- **S L2:** Telemetry parameters are sanitised to length buckets and network header redaction (DevTools sanitizeHeaders) is available for model-bound output. — [src/telemetry/transformation.ts:212-213](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/telemetry/transformation.ts#L212-L213); [src/formatters/NetworkFormatter.ts:169-176](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/formatters/NetworkFormatter.ts#L169-L176) (verified)
  - *To reach the next level:* No redaction in the debug log or of cookies/bodies sent to the model, and no keychain use.
- **C L2:** Telemetry and (when enabled) model-bound network headers are protected; debug logs, error text, page/cookie reads via scripts and Chrome's inherited environment are not. — [src/ToolHandler.ts:256-258](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/ToolHandler.ts#L256-L258) (verified)
  - *To reach the next level:* Cover logs and all model-bound paths (cookies, bodies) with redaction.
- **D L1:** Telemetry is on by default and content-free; header redaction is off by default. — [src/config/mcp-options.ts:122-127](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/mcp-options.ts#L122-L127); [src/config/mcp-options.ts:175-180](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/mcp-options.ts#L175-L180); [src/ToolHandler.ts:273](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/ToolHandler.ts#L273) (verified)
  - *To reach the next level:* Make telemetry opt-in and header redaction default-on.
- **B L0:** With file: navigation on by default, long-lived high-privilege keys in the user's home directory (and session cookies in the persistent profile) are reachable by the model (Chrome renders local text files, inferred). — [src/config/mcp-options.ts:134-136](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/mcp-options.ts#L134-L136); [src/utils/url.ts:212](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/utils/url.ts#L212); [src/formatters/NetworkFormatter.ts:201-203](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/formatters/NetworkFormatter.ts#L201-L203) (inferred)
  - *To reach the next level:* Block file: navigation and cookie/header exposure by default so no long-lived secrets are reachable.
- **Cap:** none

### C9 Audit & traceability — 0.28 (high)

The server keeps no record of what it did unless the operator passes --logFile or sets NODE_DEBUG. When enabled, each tool call is written as a timestamped line with its parameters, but there is no structured format, no record of results, no actor attribution, and no tamper protection. Telemetry sent to Google is aggregate and is not an audit trail.

- **S L1:** When enabled, the log is unstructured timestamped text of each tool request with JSON parameters. — [src/ToolHandler.ts:256-258](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/ToolHandler.ts#L256-L258); [src/utils/logger.ts:55-60](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/utils/logger.ts#L55-L60) (verified)
  - *To reach the next level:* Write a structured per-call record (arguments, result status, timestamps).
- **C L2:** All tools pass through ToolHandler.handle, so every built-in tool call is logged when logging is on. — [src/ToolHandler.ts:236-237](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/ToolHandler.ts#L236-L237) (verified)
  - *To reach the next level:* Approvals do not exist server-side; page-provided tool executions are only logged as the generic executor call.
- **D L0:** Logging is opt-in via --logFile or NODE_DEBUG. — [src/config/mcp-options.ts:23-27](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/config/mcp-options.ts#L23-L27); [src/utils/logger.ts:55-60](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/utils/logger.ts#L55-L60) (verified)
  - *To reach the next level:* Record tool calls by default outside the workspace.
- **B L1:** Log writes go through a buffered append stream; a stream error exits the process, but records can be lost on crash. — [src/utils/logger.ts:18-24](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/utils/logger.ts#L18-L24) (verified)
  - *To reach the next level:* Flush records per action.
- **Cap:** G1 — Tool-call logging only exists when the operator passes --logFile or sets NODE_DEBUG.

### C10 Limits & kill switch — 0.55 (high)

Every tool call is serialised through one mutex and bounded by a hard-coded 60-second timeout that configuration and the model cannot raise. Output sizes are capped for network bodies, console messages and request lists. The timeout only stops waiting: the abandoned browser work keeps running, and the server ignores MCP cancellation. Closing stdin or sending a signal shuts the server down and closes the launched browser, with a 5-second forced exit.

- **S L2:** Server-enforced 60s per-call timeout, single-call concurrency via mutex, and output caps on several operations. — [src/ToolHandler.ts:40](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/ToolHandler.ts#L40); [src/ToolHandler.ts:237](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/ToolHandler.ts#L237); [src/formatters/NetworkFormatter.ts:15](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/formatters/NetworkFormatter.ts#L15); [src/collectors/PageCollector.ts:387](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/collectors/PageCollector.ts#L387) (verified)
  - *To reach the next level:* Timeouts don't cancel in-flight work, and MCP cancellation is not honoured.
- **C L2:** The timeout and mutex wrap every tool call, including the response-building CDP calls. — [src/ToolHandler.ts:277-283](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/ToolHandler.ts#L277-L283); [src/ToolHandler.ts:207-210](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/ToolHandler.ts#L207-L210) (verified)
  - *To reach the next level:* Background work (abandoned calls, traces, screencasts) isn't counted or cancelled.
- **D L3:** The 60s ceiling is a constant; the model's own timeout parameter cannot exceed it because the outer race still applies. — [src/ToolHandler.ts:40](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/ToolHandler.ts#L40); [src/tools/ToolDefinition.ts:478-485](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/tools/ToolDefinition.ts#L478-L485) (verified)
  - *To reach the next level:* No session-level ceilings beyond per-call bounds.
- **B L2:** Per-call ceilings are moderate; on shutdown the server closes the browser with a 5s forced exit, but timed-out calls keep running until then. — [src/ToolHandler.ts:207-210](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/ToolHandler.ts#L207-L210); [src/bin/chrome-devtools-mcp-main.ts:60-66](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/b2f522c8ba0fd2e00a679159b4aa5243de5f1b78/src/bin/chrome-devtools-mcp-main.ts#L60-L66) (verified)
  - *To reach the next level:* Cancel pending calls on stop/timeout.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Web page snapshots, console, network bodies and dialog text returned to the model (src/McpResponse.ts:1183, src/McpResponse.ts:1006) · [B] sensitive data/systems: file: navigation on by default (src/config/mcp-options.ts:134) and unredacted headers/cookies (src/config/mcp-options.ts:175) · [C] state change / egress: evaluate_script arbitrary JS incl. fetch (src/tools/script.ts:230), navigation and input tools · Same default session? Yes

## Highest-impact improvements
1. Harden how configuration is sourced so security-relevant options require an explicit operator decision. — C6 S L0→L3, +0.225 before caps (Playbook 2)
2. Default --file-navigations to false (or validate file: URLs against the same roots as server file IO). — C3 B L1→L2, +0.050 before caps (Playbook 3)
3. Turn --redact-network-headers on by default. — C8 D L1→L2, +0.050 before caps (Playbook 4)
4. Write a structured per-call log by default to a user-scope directory. — C9 D L0→L2, +0.100 before caps (Playbook 1 step 3)
5. Make experimentalStructuredContent default with page-derived fields marked untrusted and their source URL, and drop directives from content-bearing output. — C5 S L0→L3, +0.225 before caps (Playbook 1)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Chrome's renderer sandbox, its handling of --no-sandbox, and how it renders local text files are Chrome behaviours inferred, not verified in this repo.
- One configuration-integrity finding depends on how MCP hosts start the server.
- Only the primary stdio MCP mode was scored; the chrome-devtools CLI/daemon mode (headless and isolated by default, unrestricted paths) was not scored. The bundled Lighthouse/DevTools third_party code was not reviewed.
- No reviewer-injection text was found in the repository.
