# Defense-in-Depth Score: Nanobrowser

**Repo:** https://github.com/nanobrowser/nanobrowser · **Commit:** `ad47282a17ecdfb894745af093e0f7332fc1f71a` · **Reviewed:** 2026-10-03
**What it is:** Chrome extension multi-agent web automation with own LLM key
**Category:** AI Assistants
**Scored configuration:** Chrome MV3 extension built from the pinned commit with default settings: empty firewall lists, replay disabled, analytics enabled, one LLM provider configured by the user.
**Agent surface (default):** code execution no · filesystem write no · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions no · sub agents yes · external communication yes

## Score: 4.4 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C3 | Tool & action scoping | L2 | L2 | L0 | L0 | 0.30 | G1 | **0.30** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L1 | L2 | L2 | L0 | 0.33 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L2 | L3 | L3 | L4 | 0.72 | — | **0.72** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C9 | Audit & traceability | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L2 | 0.45 | — | **0.45** | High |

Controls where a risk surface exists: 2.40 / 8.0 (30%); 2 criteria scored SA (surface absent).

Nanobrowser drives your real, logged-in Chrome browser on any website with no approval step: whatever the model decides to click, type or submit happens immediately. Its prompt-injection defenses are a regex filter and warning tags around page content, so a hostile page that hijacks the agent can send your data to any URL and act in your accounts with no human involved. It runs no model-generated code and loads no plugins, but keep it to a separate browser profile and set a domain allowlist in the firewall settings before using it.

## Critical gaps
- The agent acts with the user's full logged-in browser identity across all sites (<all_urls> plus the debugger permission); a hijack reaches every account the user is signed into. (ASI03, T3; C1) — [wxt.config.ts:57-58](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/wxt.config.ts#L57-L58); [wxt.config.ts:58](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/wxt.config.ts#L58)
- Prompt-injected web content can make the agent both exfiltrate data (navigate to any URL) and take irreversible actions in the user's logged-in sites with no human approval (C5-WORSTCASE). (ASI01, LLM01, T6; C5) — [src/background/agent/actions/builder.ts:181-185](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/actions/builder.ts#L181-L185); [src/background/browser/util.ts:34-37](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/browser/util.ts#L34-L37); [src/background/agent/actions/builder.ts:279-292](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/actions/builder.ts#L279-L292); [src/background/agent/messages/utils.ts:277-288](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/messages/utils.ts#L277-L288)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

Nanobrowser runs inside the user's own Chrome profile with access to every website (<all_urls>), the Chrome debugger, and all tabs, so the agent acts with every logged-in session the user has: email, banking, social media, work tools. It has no identity of its own and no per-request authorization layer; the model can switch to any open tab in any window and act there. Nothing narrows that authority by default. If the agent is hijacked, the attacker inherits the user's entire web identity.

- **S L0:** The agent drives the user's real browser with <all_urls> host permission and the debugger permission, inheriting every logged-in session. — [wxt.config.ts:57-58](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/wxt.config.ts#L57-L58); [wxt.config.ts:58](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/wxt.config.ts#L58) (verified)
  - *To reach the next level:* A dedicated, narrower identity (e.g. a separate browser profile or an origin-scoped grant) instead of the user's full ambient sessions.
- **C L0:** No authorization layer exists: every action, including switch_tab to any tab in any window, runs with the ambient session and no per-action check. — [src/background/browser/context.ts:221-230](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/browser/context.ts#L221-L230); [src/background/browser/context.ts:307-308](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/browser/context.ts#L307-L308); [src/background/agent/agents/navigator.ts:294-326](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/agents/navigator.ts#L294-L326) (verified)
  - *To reach the next level:* An authorization check on the main action path (e.g. restricting actions to tabs/origins the user granted for this task).
- **D L0:** The shipped manifest grants all-sites and debugger access; least privilege would require the user to configure a firewall allowlist, which is empty by default. — [wxt.config.ts:57-58](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/wxt.config.ts#L57-L58); [src/storage/lib/settings/firewall.ts:35-39](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/storage/lib/settings/firewall.ts#L35-L39) (verified)
  - *To reach the next level:* A default that confines the agent to a narrower scope (e.g. the starting tab's origin) until the user widens it.
- **B L0:** A hijacked agent can act in any of the user's logged-in accounts across all services. — [wxt.config.ts:57-58](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/wxt.config.ts#L57-L58); [src/background/browser/context.ts:307-308](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/browser/context.ts#L307-L308) (verified)
  - *To reach the next level:* Confinement so that a hijack reaches at most write access to multiple systems rather than the user's entire web identity.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

There is no human approval step anywhere in the action path. The navigator's chosen actions (clicking, typing into fields, sending keystrokes, navigating, opening and closing tabs) execute immediately after schema validation. The only safeguard is a line in the system prompt telling the model not to touch payment or checkout without user approval, which code does not enforce. Clicking a submit button or pressing Enter can send messages, post content, or complete purchases with no confirmation and no undo.

- **S L0:** No approval mechanism exists; the only 'approval' is a system-prompt instruction to the model. — [src/background/agent/prompts/templates/common.ts:20](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/prompts/templates/common.ts#L20); searched `rg -n -i 'approv|confirm\(|requireConfirm|human_in|ask_human|needsApproval'` in `src entrypoints public/permission` → 1 hits (the single hit is the system-prompt rule at src/background/agent/prompts/templates/common.ts:20; no approval code exists) (verified)
  - *To reach the next level:* Per-call human approval for consequential actions (form submission, typing, navigation to new origins).
- **C L0:** All actions, including the most powerful (click, input_text, send_keys), run straight from the model's output without crossing any gate. — [src/background/agent/agents/navigator.ts:294-326](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/agents/navigator.ts#L294-L326); [src/background/agent/actions/builder.ts:279-292](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/actions/builder.ts#L279-L292) (verified)
  - *To reach the next level:* Route every consequential action through one gate before execution.
- **D L0:** There is no approval to turn on; the default is fully autonomous. — [src/background/agent/agents/navigator.ts:294-326](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/agents/navigator.ts#L294-L326) (verified)
  - *To reach the next level:* Approval on by default for state-changing actions.
- **B L0:** Clicks and typed input on arbitrary sites can send email, post publicly, or complete purchases, which are irreversible. — [src/background/agent/actions/builder.ts:246-248](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/actions/builder.ts#L246-L248); [src/background/agent/actions/builder.ts:279-292](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/actions/builder.ts#L279-L292) (verified)
  - *To reach the next level:* Undo or preview for external actions, or limits on which actions can run unattended.
- **Cap:** none

### C3 Tool & action scoping — 0.30 (high)

Each action has a typed schema that is validated before it runs, and every navigation path blocks dangerous URL schemes (chrome://, javascript:, data:, file:, extension pages). An optional URL firewall can restrict the agent to allowed domains, but both its lists are empty by default, which means every website is allowed. Even when configured, the firewall is not a complete boundary. The action set itself is general-purpose: arbitrary URLs, arbitrary typed text and keystrokes on any site.

- **S L2:** Zod schemas validate every action's arguments and an optional domain allow/deny list checks navigation, but it is not a complete boundary, and the default lists allow everything. — [src/background/agent/actions/builder.ts:66-70](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/actions/builder.ts#L66-L70); [src/background/browser/context.ts:233-236](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/browser/context.ts#L233-L236); [src/background/browser/util.ts:17-32](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/browser/util.ts#L17-L32) (verified)
  - *To reach the next level:* Allowlist checks that run before any request and block internal addresses.
- **C L2:** Navigation actions (go_to_url, open_tab, search_google) are pre-checked, but the firewall does not cover every path and input_text/send_keys take arbitrary text. — [src/background/browser/context.ts:233-236](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/browser/context.ts#L233-L236); [src/background/agent/actions/builder.ts:279-292](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/actions/builder.ts#L279-L292) (verified)
  - *To reach the next level:* A shared validation layer covering every action.
- **D L0:** All actions, including typing, clicking and navigating to any host, are enabled by default; the firewall lists ship empty, which the code treats as allow-all. — [src/storage/lib/settings/firewall.ts:35-39](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/storage/lib/settings/firewall.ts#L35-L39); [src/background/browser/util.ts:34-37](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/browser/util.ts#L34-L37) (verified)
  - *To reach the next level:* Selectable action groups or a narrow default (e.g. read-only navigation) with write actions enabled explicitly.
- **B L0:** A misused action reaches any website in the user's browser with the user's sessions. — [src/background/agent/actions/builder.ts:181-185](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/actions/builder.ts#L181-L185); [wxt.config.ts:57-58](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/wxt.config.ts#L57-L58) (verified)
  - *To reach the next level:* Scoping the agent to a task-specific set of origins.
- **Cap:** G1 — The URL allow/deny firewall, the only control that bounds where actions reach, ships with empty lists, which the code treats as allow-all.

### C4 Code-execution isolation — 1.00 (high)

The agent never turns model output into code. Every page script the extension runs is a fixed function shipped with the extension, and model-chosen strings only reach typed browser operations (click by index, type text, navigate). javascript: and data: URLs are blocked on every navigation path. The web pages the agent visits do run their own scripts, but inside Chrome's normal renderer sandbox; that is the ordinary web, not model-generated code.

- **Structural absence:** searched `rg -n 'eval\(|new Function\(|Runtime\.evaluate|addScriptTag|child_process'` in `src entrypoints public` → 0 hits (no dynamic code evaluation of any kind in the extension); searched `rg -n "evaluate\('"` in `src/background/browser/page.ts` → 3 hits (all three are fixed literal strings ('1' and two window.scrollBy expressions); no model text is interpolated)
- **Notes:** javascript:/data:/file: URLs are blocked in util.ts:17-32 even with the firewall off.

### C5 Untrusted input blast radius — 0.25 (high)

Web pages are the agent's main input, and a hostile page can try to hijack it. The defenses are detection and labelling: page elements are passed through a regex filter (it strips phrases such as 'ignore previous instructions') and wrapped in 'untrusted content' tags with warnings, plus system-prompt rules. Nothing in code limits what a hijacked agent can then do: it can navigate to an attacker URL carrying data in the query string, or type and submit content in any logged-in site, with no human involved. Some page-derived text (titles of every open tab, dropdown option text) is not wrapped at all.

- **S L1:** Defenses are a regex sanitizer plus spotlighting tags and prompt warnings; no capability is restricted after untrusted content is read. — [src/background/agent/messages/utils.ts:277-288](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/messages/utils.ts#L277-L288); [src/background/services/guardrails/patterns.ts:15](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/services/guardrails/patterns.ts#L15); [src/background/agent/prompts/base.ts:38](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/prompts/base.ts#L38) (verified)
  - *To reach the next level:* Require human approval for egress or state-changing actions once untrusted content has entered the session.
- **C L2:** The page element tree, cached content and attachments are wrapped and sanitized, but titles/URLs of all open tabs, dropdown option text and action errors enter context unwrapped. — [src/background/agent/prompts/base.ts:38](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/prompts/base.ts#L38); [src/background/agent/prompts/base.ts:67-70](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/prompts/base.ts#L67-L70); [src/background/agent/actions/builder.ts:606-612](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/actions/builder.ts#L606-L612) (verified)
  - *To reach the next level:* Wrap every page-derived string, including tab titles and dropdown options.
- **D L2:** The sanitizer is on by default with no setting to turn it off; rated at most one level above Strength. — [src/background/services/guardrails/index.ts:19](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/services/guardrails/index.ts#L19) (verified)
  - *To reach the next level:* A stronger mechanism (Strength) so that the always-on default earns more credit.
- **B L0:** A hijacked agent can leak data by navigating to an attacker URL (all hosts allowed by default) and take irreversible actions in logged-in sites, both unattended. — [src/background/agent/actions/builder.ts:181-185](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/actions/builder.ts#L181-L185); [src/background/browser/util.ts:34-37](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/browser/util.ts#L34-L37); [src/background/agent/actions/builder.ts:279-292](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/actions/builder.ts#L279-L292) (verified)
  - *To reach the next level:* Human approval before egress to new origins and before state-changing actions.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.72 (high)

By default nothing the agent reads persists into future behaviour. Chat transcripts are saved for display but are never fed back to the model, and the extension loads no instruction or config files from websites or workspaces. An opt-in replay feature stores the agent's step history and can re-execute it later, but only when the user enables it in settings and explicitly types a /replay command for a specific session. Replayed actions are not re-validated beyond the URL firewall, and stored history has no expiry.

- **S L2:** Agent step history is written only when replay is enabled and is re-executed only on an explicit user command, but entries are not validated and carry no expiry. — [src/background/agent/executor.ts:221-228](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/executor.ts#L221-L228); [src/background/agent/executor.ts:382-411](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/executor.ts#L382-L411) (verified)
  - *To reach the next level:* Validation or review of stored step history before replay, with expiry.
- **C L3:** Every persistent store is covered: chat transcripts are never re-injected, step history is gated by the opt-in setting and explicit replay, and there are no auto-loaded instruction files. — [src/background/agent/executor.ts:221-228](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/executor.ts#L221-L228); [src/side-panel/SidePanel.tsx:143-147](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/side-panel/SidePanel.tsx#L143-L147); searched `rg -n chatHistoryStore` in `src/background` → 3 hits (import, the opt-in store, and the explicit replay load in executor.ts; nothing else reads stored history into the agent) (verified)
  - *To reach the next level:* Provenance tags on replayed steps end to end.
- **D L3:** Stored history is keyed per session, and the model has no action that writes extension storage. — [src/storage/lib/chat/history.ts:33-34](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/storage/lib/chat/history.ts#L33-L34) (verified)
  - *To reach the next level:* Retention limits on by default.
- **B L4:** In the default configuration step history is not stored at all, so a poisoned session cannot persist into later ones. — [src/storage/lib/settings/generalSettings.ts:34](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/storage/lib/settings/generalSettings.ts#L34); [src/background/agent/executor.ts:221-228](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/executor.ts#L221-L228) (verified)
- **Cap:** none

### C7 Third-party extensions — 1.00 (high)

Nanobrowser loads no third-party code at runtime: there are no plugins, no MCP servers, and no downloaded tools. The bundled analytics library is the 'no-external' build with remote configuration disabled, so it does not fetch code either. Ordinary dependency supply-chain risk is outside this criterion.

- **Structural absence:** searched `rg -n -i 'mcp|plugin|import\(|loadExtension|npx'` in `src entrypoints` → 7 hits (hits are Tailwind config plugin arrays/comments, a test file's dynamic import of local code, a commented-out navigator.plugins shim and a 'facebook.com/plugins' URL string; none loads third-party code)
- **Notes:** PostHog is imported from 'posthog-js/dist/module.no-external' with advanced_disable_decide: true (analytics.ts:2, 98).

### C8 Secrets & sensitive-data protection — 0.25 (high)

LLM API keys are kept in plain text in Chrome's local extension storage and are never placed in prompts. Page content sent to the model passes through a regex filter that redacts things that look like credit card numbers, SSNs, emails and 'password=...' strings, but this is pattern matching and input field values are part of the page data the model sees. Console logs record every action's arguments, including typed text, without redaction. Anonymous analytics are on by default and report which domains the agent visits.

- **S L1:** Keys come from plaintext extension storage; the only masking is regex redaction of page content before it goes to the model. — [src/storage/lib/settings/llmProviders.ts:42-49](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/storage/lib/settings/llmProviders.ts#L42-L49); [src/background/services/guardrails/patterns.ts:98-103](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/services/guardrails/patterns.ts#L98-L103) (verified)
  - *To reach the next level:* Redaction or filters on logs as well, and type-level handling of stored keys.
- **C L1:** Only the model-bound page content path is filtered; console logs include raw action arguments. — [src/background/agent/agents/navigator.ts:286](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/agents/navigator.ts#L286); [src/background/services/guardrails/patterns.ts:98-103](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/services/guardrails/patterns.ts#L98-L103) (verified)
  - *To reach the next level:* Cover logs and saved transcripts as well.
- **D L1:** Analytics are enabled by default and send visited domains to PostHog (content-free otherwise); whether release builds embed the PostHog key is inferred. — [src/storage/lib/settings/analyticsSettings.ts:23-27](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/storage/lib/settings/analyticsSettings.ts#L23-L27); [src/background/services/analytics.ts:194-209](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/services/analytics.ts#L194-L209); [src/background/services/analytics.ts:64-70](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/services/analytics.ts#L64-L70) (verified)
  - *To reach the next level:* Make telemetry opt-in.
- **B L1:** Leaked material would be long-lived LLM provider keys; the model cannot read them, but they are stored unencrypted. — [src/storage/lib/settings/llmProviders.ts:42-49](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/storage/lib/settings/llmProviders.ts#L42-L49) (verified)
  - *To reach the next level:* Scoped keys that are short-lived or rotatable.
- **Cap:** none

### C9 Audit & traceability — 0.38 (high)

The side panel saves a chat transcript for each session in extension storage, but for most actions it records the model's own one-line description of what it is doing, not the exact action and arguments. Success messages are dropped unless a replay is running. A structured record of every step, with the exact arguments, exists only when the opt-in replay setting is on. Failed writes are logged to the console and the agent keeps going.

- **S L1:** The persisted transcript holds model-authored 'intent' strings for action starts, not exact arguments; structured step history is opt-in. — [src/background/agent/actions/builder.ts:182-183](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/actions/builder.ts#L182-L183); [src/side-panel/SidePanel.tsx:234-235](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/side-panel/SidePanel.tsx#L234-L235); [src/background/agent/executor.ts:221-228](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/executor.ts#L221-L228) (verified)
  - *To reach the next level:* A structured record of every action with exact arguments and timestamps by default.
- **C L2:** Every built-in action emits a start event that the side panel saves (cache_content excepted); there are no extensions or approvals to log. — [src/side-panel/SidePanel.tsx:143-147](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/side-panel/SidePanel.tsx#L143-L147); [src/side-panel/SidePanel.tsx:229](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/side-panel/SidePanel.tsx#L229) (verified)
  - *To reach the next level:* Record approvals and denials once an approval step exists; include all actions.
- **D L2:** The transcript is on by default and stored in extension storage the model cannot reach, but the extension process itself can rewrite or delete it. — [src/side-panel/SidePanel.tsx:143-147](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/side-panel/SidePanel.tsx#L143-L147) (verified)
  - *To reach the next level:* Write the record from a component the agent process cannot alter.
- **B L1:** Writes are best-effort: errors go only to the console and the agent continues. — [src/side-panel/SidePanel.tsx:144-146](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/side-panel/SidePanel.tsx#L144-L146) (verified)
  - *To reach the next level:* Surface logging errors and flush durably per action.
- **Cap:** none

### C10 Limits & kill switch — 0.45 (high)

The agent stops after 100 steps or three consecutive failures, and each model call is capped at 4,096 output tokens. Some browser operations have short timeouts. There is no wall-clock or total cost limit. The five-actions-per-step limit is only stated in the prompt and the code doesn't enforce it, and the wait action accepts any number of seconds. Pressing stop (or closing the side panel) halts the loop between actions and aborts the pending model call, but an in-flight browser action finishes first.

- **S L2:** An iteration cap plus a per-call output token cap and some per-action timeouts are enforced; halt is cooperative. — [src/background/agent/executor.ts:145](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/executor.ts#L145); [src/background/llm/providers.ts:19](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/llm/providers.ts#L19); [src/background/browser/page.ts:1306-1308](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/browser/page.ts#L1306-L1308); [src/background/agent/types.ts:96-99](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/types.ts#L96-L99) (verified)
  - *To reach the next level:* A wall-clock limit and a total token or cost cap per task.
- **C L2:** Limits apply to the top-level loop (planner and navigator share it) and some browser operations have timeouts, but actions per step and wait duration are unbounded in code. — [src/background/agent/agents/navigator.ts:294-326](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/agents/navigator.ts#L294-L326); [src/background/agent/prompts/navigator.ts:19](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/prompts/navigator.ts#L19); [src/background/agent/actions/schemas.ts:213](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/actions/schemas.ts#L213) (verified)
  - *To reach the next level:* Enforce maxActionsPerStep and bound every action, including wait.
- **D L1:** Step and failure defaults (100, 3) are sensible, but the model can exceed the per-step action limit (stated only in the prompt) and choose any wait duration. — [src/storage/lib/settings/generalSettings.ts:25-28](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/storage/lib/settings/generalSettings.ts#L25-L28); [src/background/agent/prompts/navigator.ts:19](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/prompts/navigator.ts#L19); [src/background/agent/actions/schemas.ts:213](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/actions/schemas.ts#L213) (verified)
  - *To reach the next level:* Enforce every limit in code so the model cannot exceed it.
- **B L2:** Moderate ceilings; stop ends the loop and aborts the model call, but in-flight browser actions run to completion, and follow-up tasks start a fresh step budget. — [src/background/agent/types.ts:96-99](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/types.ts#L96-L99); [src/background/agent/executor.ts:132](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/agent/executor.ts#L132); [src/background/index.ts:262-267](https://github.com/nanobrowser/nanobrowser/blob/ad47282a17ecdfb894745af093e0f7332fc1f71a/src/background/index.ts#L262-L267) (verified)
  - *To reach the next level:* Tight per-run time and cost ceilings and cancellation of in-flight actions.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Arbitrary web page content enters every navigator step (src/background/agent/prompts/base.ts:38) · [B] sensitive data/systems: All of the user's logged-in sessions via <all_urls> and debugger access (wxt.config.ts:57-58) · [C] state change / egress: Clicks, typing and navigation to any URL (src/background/agent/actions/builder.ts:181-185, 279-292) · Same default session? Yes

## Highest-impact improvements
1. Add a per-action human approval step showing the exact action and arguments for clicks that submit, typing, key presses and navigation to new origins. — C2 S L0→L3, +0.225 before caps (Playbook 5)
2. Route every action, including switch_tab and send_keys, through that approval gate with a read-only auto-approve list (scrolling, reading). — C2 C L0→L3, +0.225 before caps (Playbook 5)
3. Once untrusted page content is in context, require approval before navigating to any origin outside the task's starting origin. — C5 S L1→L2, +0.075 before caps (Playbook 1)
4. Make PostHog analytics opt-in. — C8 D L1→L2, +0.050 before caps
5. Persist a structured record of every executed action with its exact arguments by default, not just the model's intent text. — C9 S L1→L2, +0.075 before caps

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Whether published Chrome Web Store builds embed a PostHog API key (enabling the default-on analytics) was inferred, not verified; the code only enables analytics when VITE_POSTHOG_API_KEY is set at build time.
- The DOM extraction script public/buildDomTree.js and the puppeteer-core library internals were not reviewed line by line; the review focused on the agent loop, actions, firewall, guardrails, storage and settings.
- No reviewer-steering text was found in README, AGENTS.md, CLAUDE.md or SECURITY.md.
