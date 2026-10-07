# Defense-in-Depth Score: Burp Suite MCP Server

**Repo:** https://github.com/PortSwigger/mcp-server · **Commit:** `642e6fa31c63db3886a353fcd7ed62037e0ceed5` · **Reviewed:** 2026-10-03
**What it is:** Burp Suite extension exposing Burp to AI clients via MCP
**Category:** Cybersecurity
**Scored configuration:** Burp extension as shipped: MCP endpoint on 127.0.0.1:9876, HTTP-request and project-data approval prompts on, config-editing tools off, credential filter on.
**Agent surface (default):** code execution opt-in · filesystem write no · network egress yes · external credentials no · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication yes

## Score: 3.8 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C2 | Approval gates | L3 | L1 | L2 | L1 | 0.45 | — | **0.45** | High |
| C3 | Tool & action scoping | L1 | L0 | L1 | L1 | 0.17 | — | **0.17** | High |
| C4 | Code-execution isolation | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L2 | 0.17 | — | **0.17** | High |
| C6 | Memory, context & configuration integrity | L2 | L3 | L2 | L2 | 0.57 | — | **0.57** | Medium |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L2 | 0.35 | — | **0.35** | High |
| C9 | Audit & traceability | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C10 | Limits & kill switch | L2 | L1 | L1 | L1 | 0.33 | — | **0.33** | High |

Controls where a risk surface exists: 2.75 / 9.0 (31%); 1 criterion scored SA (surface absent).

The server has no client authentication of its own and relies on loopback binding, Host/Origin checks and two Swing approval dialogs. Sending HTTP requests and reading proxy history, WebSocket history and Organizer items require a per-call human decision by default, and the dialog shows the exact request. Several other state-changing tools (intercept toggle, task engine state, editor contents, Repeater and Intruder tabs) are ungated, outputs carry no provenance, and there are no timeouts or rate limits. Controls are meaningful for a local single-user tool but thin on validation, logging and limits.

## Critical gaps
- The MCP HTTP endpoint has no authentication or per-client identity; any local process that passes the Host/Origin/User-Agent checks can invoke every tool. (ASI03, T9, LLM06; C1) — [src/main/kotlin/net/portswigger/mcp/KtorServerManager.kt:44](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/KtorServerManager.kt#L44); [src/main/kotlin/net/portswigger/mcp/KtorServerManager.kt:61-67](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/KtorServerManager.kt#L61-L67)

## Criterion details

### C1 Identity & least privilege — 0.30 (high)

The extension runs inside Burp and holds no credentials of its own, but its MCP endpoint has no authentication or per-client identity. Protection is loopback binding by default plus Host, Origin, Referer and User-Agent filtering aimed at DNS rebinding and browsers. Authority is that of the whole Burp instance, narrowed only by approval prompts for HTTP sends and history reads; most other tools are ungated. Failure of those prompts exposes Burp's traffic data and its ability to send requests as the user.

- **S L1:** No caller authentication; the identity is the whole Burp instance and only two tool families are authorization-gated. — [src/main/kotlin/net/portswigger/mcp/KtorServerManager.kt:44](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/KtorServerManager.kt#L44); searched `rg -n -i 'Authorization|Bearer|apiKey|api_key|token' src/main` in `src/main` → 0 hits (no authentication or token handling in server code) (verified)
  - *To reach the next level:* No per-client identity or scoped authority per tool, and gating does not cover every request.
- **C L1:** Only HTTP-send and history tools cross an authorization check; other tools use Burp's authority directly. — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:114-116](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L114-L116); [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:373-374](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L373-L374) (verified)
  - *To reach the next level:* Every tool path would need to pass the same authorization layer.
- **D L2:** Binds to loopback and approvals are on by default; the host is an operator setting and the checks only restrict Host/Origin headers. — [src/main/kotlin/net/portswigger/mcp/config/McpConfig.kt:16](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/config/McpConfig.kt#L16); [src/main/kotlin/net/portswigger/mcp/KtorServerManager.kt:77](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/KtorServerManager.kt#L77) (verified)
  - *To reach the next level:* Default is not read-only or minimal and there is no per-client credential.
- **B L1:** If authorization fails, a caller can drive Burp (send requests as the user, toggle intercept, read traffic data); no independent per-tool credentials limit this. — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:373-387](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L373-L387); [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:127](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L127) (verified)
  - *To reach the next level:* Narrow single-tenant read-mostly authority with revocable credentials.
- **Cap:** none

### C2 Approval gates — 0.45 (high)

As a tool server it enforces approval itself: sending an HTTP request opens a Swing dialog showing the target and the exact request, and proxy, WebSocket and Organizer reads each need approval, both on by default. There are separate read and write tools but no readOnlyHint or destructiveHint annotations. Intercept toggling, task-engine state, editor writes and Repeater/Intruder tab creation never pass a gate, and the approvals offer persistent allow rules and can be switched off with a checkbox without a warning. HTTP sends are irreversible and have no rate limit.

- **S L3:** Per-call server-side approval shows the exact request for HTTP sends and a data-access prompt for history reads; no risk annotations are published. — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:114-116](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L114-L116); [src/main/kotlin/net/portswigger/mcp/config/Dialogs.kt:312-324](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/config/Dialogs.kt#L312-L324); [src/main/kotlin/net/portswigger/mcp/security/HttpRequestSecurity.kt:110-116](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/security/HttpRequestSecurity.kt#L110-L116) (verified)
  - *To reach the next level:* No annotations on tools and no server-enforced read-only mode.
- **C L1:** The most powerful action (HTTP send) is gated, but several mutating tools are not. — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:379-387](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L379-L387); [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:393-403](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L393-L403); [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:165-181](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L165-L181); searched `rg -n 'ToolAnnotations|readOnlyHint|destructiveHint' src/main` in `src/main` → 0 hits (no annotations published) (verified)
  - *To reach the next level:* All tools, including state toggles and tab creation, would cross the gate.
- **D L2:** On by default; a checkbox disables it without a warning and Always Allow decisions persist in Burp extension data. — [src/main/kotlin/net/portswigger/mcp/config/McpConfig.kt:18](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/config/McpConfig.kt#L18); [src/main/kotlin/net/portswigger/mcp/security/HttpRequestSecurity.kt:44-51](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/security/HttpRequestSecurity.kt#L44-L51) (verified)
  - *To reach the next level:* Disabling should need an explicit loudly named flag, and persisted rules should be visible/expiring.
- **B L1:** A wrongly approved or bypassed HTTP send is irreversible with no rate or quantity bounds, and the ungated toggles change Burp state silently. — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:127](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L127); searched `rg -n -i 'RateLimit|Semaphore|withTimeout' src/main` in `src/main` → 0 hits (no rate limiting) (verified)
  - *To reach the next level:* Rate limits and bounded quantities on consequential actions with previews.
- **Cap:** none
- **Notes:** HTTP/1.1 approvals display the raw content while execution sends the normalized content (Tools.kt:124-126); the difference is line-ending normalization only.

### C3 Tool & action scoping — 0.17 (high)

Tools are typed Kotlin data classes decoded from JSON, so malformed shapes are rejected, but no tool argument is checked against an allowlist or bound. Hostnames and ports for HTTP sends are arbitrary (including internal addresses; approval is the only barrier), regexes are compiled directly, pagination counts have no ceiling, and configuration import takes arbitrary JSON when enabled. All tools are registered by default except config editing, which is runtime-disabled.

- **S L1:** Typed schemas only; target host, raw request text, regex and options JSON are passed through without allowlists or bounds. — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:429-435](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L429-L435); [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:319](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L319); searched `rg -n -i 'isPrivate|isLoopback|169\.254|allowlist' src/main` in `src/main` → 0 hits (no internal-address policy on tool arguments) (verified)
  - *To reach the next level:* Allowlist validation in code (host/IP policy, bounded counts, regex limits).
- **C L0:** No tool validates its arguments beyond JSON type decoding. — [src/main/kotlin/net/portswigger/mcp/tools/McpTool.kt:30-33](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/McpTool.kt#L30-L33); [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:183-185](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L183-L185) (verified)
  - *To reach the next level:* Some tools would need to validate inputs.
- **D L1:** All tools are on by default and cannot be individually disabled; only config editing is off. — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:113-113](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L113-L113); [src/main/kotlin/net/portswigger/mcp/config/McpConfig.kt:15](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/config/McpConfig.kt#L15) (verified)
  - *To reach the next level:* A read-only default tool set with write tools opt-in.
- **B L1:** A general HTTP sender can reach any host/port from the user's Burp, limited by the approval prompt. — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:421-427](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L421-L427) (verified)
  - *To reach the next level:* Scoped and quantity-bounded operations.
- **Cap:** none

### C4 Code-execution isolation — 0.10 (high)

The extension has no shell, script or process-execution path of its own. The one route to code execution is the configuration-import pair, whose own UI label says it can execute code; it is off by default and refuses unless the operator ticks the box. No isolation primitive contains it when enabled, and the process holds the full Burp and OS-user authority. Because the default configuration has no execution path, the low score reflects absence of any sandbox on that opt-in path rather than a default-on hazard.

- **S L0:** No isolation of any kind around the config-import path; it runs in Burp's process. — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:230-251](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L230-L251); searched `rg -n 'ProcessBuilder|Runtime\.getRuntime|ScriptEngine|GroovyShell' src/main` in `src/main` → 0 hits (no direct process or script execution in the extension) (verified)
  - *To reach the next level:* A container, OS sandbox or separate runtime around execution.
- **C L0:** The only model-reachable execution-capable path is unsandboxed. — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:243-245](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L243-L245); [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:245](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L245) (verified)
  - *To reach the next level:* Sandboxing of every model-reachable path.
- **D L1:** Path is disabled by default behind an explicit operator checkbox with a warning label, enforced in code. — [src/main/kotlin/net/portswigger/mcp/config/McpConfig.kt:15](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/config/McpConfig.kt#L15); [src/main/kotlin/net/portswigger/mcp/config/components/ServerConfigurationPanel.kt:51-55](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/config/components/ServerConfigurationPanel.kt#L51-L55) (verified)
  - *To reach the next level:* Sandbox policy defined outside anything the model can influence.
- **B L1:** If the opt-in path is used and abused, execution lands in Burp's process with the user's authority and environment. — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:232-233](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L232-L233) (verified)
  - *To reach the next level:* Ephemeral, network-restricted, secret-free sandbox.
- **Cap:** none
- **Notes:** Scored as G1/opt-in surface: the config-editing toggle is the single path that can introduce code execution.

### C5 Untrusted input blast radius — 0.17 (high)

Everything the model reads from Burp (proxied responses, history, scanner issues, Collaborator interactions) is attacker-influenced, and it is returned as plain text or JSON without an untrusted marker or source labelling. Tool descriptions contain usage guidance but no injected directives. The server offers no read-only or no-egress mode, so the structural limit on a hijack is the approval prompts: sending to a new host and reading history need a human, while intercept, task-engine and editor changes do not. Once a host is set to always-allow, requests to it are unattended.

- **S L1:** Tool results are plain text with no provenance flags; history items are structured JSON but carry no untrusted marker. — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:129](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L129); [src/main/kotlin/net/portswigger/mcp/tools/McpTool.kt:101-106](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/McpTool.kt#L101-L106) (verified)
  - *To reach the next level:* Provenance and an untrusted flag on returned content, and a mode that drops a Rule-of-Two leg.
- **C L0:** Untrusted traffic content is not distinguished from other tool output. — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:308](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L308); [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:254-256](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L254-L256) (verified)
  - *To reach the next level:* Tool results and descriptions would be distinguished from instructions.
- **D L0:** There is no content-based or taint-based mitigation to enable or disable; approval prompts are independent of provenance. — [src/main/kotlin/net/portswigger/mcp/security/HttpRequestSecurity.kt:110-116](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/security/HttpRequestSecurity.kt#L110-L116) (verified)
  - *To reach the next level:* A control that is on by default and keyed to untrusted content.
- **B L2:** After a hijack, both HTTP sends to unapproved hosts and history reads need human approval; only reversible Burp state toggles happen unattended (until an always-allow rule exists). — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:114-116](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L114-L116); [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:379-387](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L379-L387) (verified)
  - *To reach the next level:* Sessions that read untrusted content would have no egress or state change at all.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.57 (medium)

The server keeps no memory, retrieval store or conversation state and loads no instruction or workspace files. Its persisted state is the extension settings and an auto-approve target list in Burp extension data, which the model cannot write except through a human clicking Always Allow in a dialog. Configuration import tools, when enabled, can change Burp settings. Entries do not expire and the approval toggles share the same store.

- **S L2:** Persistent writes to the allow list happen only via human dialog choices; no memory or auto-loaded instruction files; entries have no expiry or provenance. — [src/main/kotlin/net/portswigger/mcp/security/HttpRequestSecurity.kt:44-51](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/security/HttpRequestSecurity.kt#L44-L51); [src/main/kotlin/net/portswigger/mcp/config/McpConfig.kt:53](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/config/McpConfig.kt#L53); searched `rg -n -i 'AGENTS\.md|CLAUDE\.md|\.cursorrules|dotenv|load_dotenv' src/main` in `src/main` → 0 hits (no auto-loaded instruction files) (verified)
  - *To reach the next level:* Gated writes with expiry and integrity or rollback.
- **C L3:** Only the settings store exists and it is written through validated, human-driven paths. — [src/main/kotlin/net/portswigger/mcp/config/McpConfig.kt:66-72](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/config/McpConfig.kt#L66-L72); [src/main/kotlin/net/portswigger/mcp/config/TargetValidation.kt:22-23](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/config/TargetValidation.kt#L22-L23) (verified)
  - *To reach the next level:* Provenance tagging of any retrieval or summaries.
- **D L2:** Single-user, single store; settings live in Burp's project extension data, and whether a shared project file can pre-seed the toggles was not verified. — [src/main/kotlin/net/portswigger/mcp/config/McpConfig.kt:18-19](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/config/McpConfig.kt#L18-L19) (inferred)
  - *To reach the next level:* Isolation guarantees and retention limits.
- **B L2:** Persisted allow rules influence which gated actions skip a prompt, but cannot add tools or write instructions. — [src/main/kotlin/net/portswigger/mcp/security/HttpRequestSecurity.kt:67-70](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/security/HttpRequestSecurity.kt#L67-L70) (verified)
  - *To reach the next level:* Persistence only after human review with rollback.
- **Cap:** none

### C7 Third-party extensions — 1.00 (high)

The server loads no third-party code at runtime: no plugin loader, remote tool fetching or model loading. It embeds a vendored proxy jar that Burp users install into their MCP client configuration; the jar is extracted from the extension's own resources and refreshed when its SHA-256 differs. That jar and the build dependencies are build supply-chain items, which are out of scope for this criterion.

- **Structural absence:** searched `rg -n 'URLClassLoader|ServiceLoader|Class\.forName|openStream|HttpClient|downloadFile' src/main` in `src/main` → 0 hits (no runtime loading or downloading of third-party code); searched `rg -n 'ProcessBuilder|Runtime\.getRuntime|ScriptEngine|GroovyShell' src/main` in `src/main` → 0 hits (no direct process or script execution in the extension)
- **Notes:** libs/mcp-proxy-all.jar is a committed binary that was not inspected; it is launched by the user's MCP client, not by this server.

### C8 Secrets & sensitive-data protection — 0.35 (high)

The extension stores no credentials and has no telemetry. Its one redaction control masks a short list of key names (password, certificate_password, hashed_key) in the two config-export tools, is on by default and fails closed on parse errors, but can be switched off with a checkbox. Proxy history, which can hold session material, is returned unredacted after approval, and the config-import logs write the full JSON to Burp's output. Nothing here is routinely sent to the model without a prompt in the default flow.

- **S L1:** A key-name redaction filter exists for config exports only. — [src/main/kotlin/net/portswigger/mcp/security/SecurityUtils.kt:30-33](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/security/SecurityUtils.kt#L30-L33); [src/main/kotlin/net/portswigger/mcp/security/SecurityUtils.kt:38-48](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/security/SecurityUtils.kt#L38-L48) (verified)
  - *To reach the next level:* Redaction before model-bound history and before logs on all major paths.
- **C L1:** One path (config export) is protected; history, HTTP responses and import logging are not. — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:207-212](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L207-L212); [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:232](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L232) (verified)
  - *To reach the next level:* Logs, model-bound traffic and error paths would also be covered.
- **D L2:** Redaction is on by default and there is no telemetry; the redaction can be disabled by a checkbox. — [src/main/kotlin/net/portswigger/mcp/config/McpConfig.kt:51](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/config/McpConfig.kt#L51) (verified)
  - *To reach the next level:* Redaction that cannot be disabled.
- **B L2:** No key material is held, but history returned to the model may carry live session secrets, gated by one-time or always-allow approval. — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:308](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L308); [src/main/kotlin/net/portswigger/mcp/security/DataAccessSecurity.kt:73-76](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/security/DataAccessSecurity.kt#L73-L76) (verified)
  - *To reach the next level:* No secret-bearing data reachable by the model.
- **Cap:** none

### C9 Audit & traceability — 0.30 (high)

Actions are recorded as free-text lines in Burp's own output log: HTTP sends (host and port only), approval denials, history-access grants and denials, Collaborator calls and config imports. Intercept, task-engine, editor and Repeater/Intruder tool calls leave no record, entries carry no request method, path, outcome or actor, and the server defines no structured or tamper-evident log. Actions proceed regardless of logging success.

- **S L1:** Unstructured logToOutput lines for some actions, without arguments beyond host:port. — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:122](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L122); [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:32-35](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L32-L35) (verified)
  - *To reach the next level:* Structured per-call record with arguments, status and timestamp.
- **C L1:** Main HTTP and data-access paths are logged; several mutating tools are not. — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:118](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L118); [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:379-387](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L379-L387) (verified)
  - *To reach the next level:* All built-in tools would be logged.
- **D L2:** Written by Burp's logging subsystem, not the workspace, but the extension process could still alter or the user clear it. — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:122](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L122) (verified)
  - *To reach the next level:* Writer the model cannot influence and an immutable store.
- **B L1:** Best-effort logging into Burp's output pane; action proceeds regardless of logging. — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:122-127](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L122-L127) (verified)
  - *To reach the next level:* Durable per-action records and fail-closed high-risk actions.
- **Cap:** none

### C10 Limits & kill switch — 0.33 (high)

The server's own bounds are thin. Each history item is truncated to 5,000 characters, and the pagination helper slices results, but the page size is chosen by the caller. There are no per-request timeouts, rate or concurrency limits in the extension's code, and no ceiling on HTTP requests once a target is approved. The operator can disable the server from the Burp UI, which calls a stop with a grace period.

- **S L2:** A server-enforced size cap applies to history items; page counts and everything else are caller-chosen or unbounded. — [src/main/kotlin/net/portswigger/mcp/schema/HistoryItemJson.kt:10](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/schema/HistoryItemJson.kt#L10); [src/main/kotlin/net/portswigger/mcp/tools/McpTool.kt:101](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/McpTool.kt#L101) (verified)
  - *To reach the next level:* Caps on every operation plus rate or concurrency limits.
- **C L1:** Only history item rendering is bounded; HTTP sends and listings are not. — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:127](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L127); searched `rg -n -i 'withTimeout|RateLimit|Semaphore' src/main` in `src/main` → 0 hits (no timeouts, rate or concurrency limits) (verified)
  - *To reach the next level:* Limits would cover every tool and spawned work.
- **D L1:** Page size and offset are model-supplied with no server ceiling. — [src/main/kotlin/net/portswigger/mcp/tools/Tools.kt:507](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/tools/Tools.kt#L507) (verified)
  - *To reach the next level:* Sensible bounded defaults the model cannot raise.
- **B L1:** No ceiling on request volume after approval; stopping is via the operator-controlled server toggle with a short grace period. — [src/main/kotlin/net/portswigger/mcp/KtorServerManager.kt:119](https://github.com/PortSwigger/mcp-server/blob/642e6fa31c63db3886a353fcd7ed62037e0ceed5/src/main/kotlin/net/portswigger/mcp/KtorServerManager.kt#L119) (verified)
  - *To reach the next level:* Tight per-run ceilings and cancellation of in-flight calls.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Proxied HTTP responses and history returned to the model (Tools.kt:127, Tools.kt:308) · [B] sensitive data/systems: Proxy/WebSocket/Organizer history with live session data, approval-gated (Tools.kt:300-358) · [C] state change / egress: HTTP sends approval-gated (Tools.kt:114-116); intercept/task-engine/editor changes ungated (Tools.kt:373-403) · Same default session? Yes

## Highest-impact improvements
1. Add a per-session bearer token or equivalent client authentication on the MCP endpoint. — C1 S L1→L3, +0.150 before caps (Playbook 4)
2. Put the intercept, task-engine, editor-write and Repeater/Intruder tools behind the same approval step, and publish readOnlyHint/destructiveHint annotations on every tool. — C2 C L1→L3, +0.150 before caps (Playbook 5)
3. Add an append-only audit log with method, host, path, approval outcome and timestamp for every tool call. — C9 S L1→L2, +0.075 before caps (Playbook 1 step 3)
4. Enforce server-side ceilings on count, request timeouts and request rate. — C10 S L2→L3, +0.075 before caps (Playbook 3 step 3)
5. Wrap returned traffic in a structured envelope marked untrusted with source host and URL. — C5 S L1→L3, +0.150 before caps (Playbook 1)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Behaviour of the Burp Montoya API, the Kotlin MCP SDK transport and the vendored libs/mcp-proxy-all.jar (a committed binary) was not examined; claims about them are inferred or out of scope.
- Burp stores extension settings (including the approval toggles and auto-approve list) in project extension data; whether a shared or untrusted project file can pre-seed them was not verified.
- No reviewer-directed instructions were found in the repository text.
