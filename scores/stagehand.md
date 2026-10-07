# Defense-in-depth score: Stagehand

**Repo:** https://github.com/browserbase/stagehand · **Commit:** `0336409092b85beeba38ea3cd4953f45eb96ed6b` (@browserbasehq/stagehand 4.1.0) · **Reviewed:** 2026-10-05
**What it is:** Browserbase's TypeScript/Python/Go SDK for browser agents: Playwright-style page control plus model-driven act, observe and extract calls, run by a runtime extension inside Chrome.
**Category:** AI Assistants
**Scored configuration:** TypeScript SDK as in the README quickstart: localBrowser.launch() with default arguments (fresh temporary Chrome profile) and Stagehand.create({ browser, model }), with the developer's code calling act, observe and extract.
**Agent surface (default):** code execution no · filesystem write no · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions no · sub agents no · external communication yes

## Score: 4.5 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L1 | L1 | 0.25 | none | **0.25** | High |
| C2 | Approval gates | L2 | L1 | L0 | L0 | 0.23 | G1 | **0.23** (alt) | High |
| C3 | Tool & action scoping | L3 | L3 | L2 | L2 | 0.65 | none | **0.65** | High |
| C4 | Code-execution isolation | L2 | L3 | L2 | L1 | 0.53 | none | **0.53** | High |
| C5 | Untrusted input blast radius | L1 | L1 | L2 | L0 | 0.25 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L1 | L1 | 0.25 | none | **0.25** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L1 | 0.45 | none | **0.45** | High |
| C9 | Audit & traceability | L2 | L2 | L0 | L1 | 0.35 | G1 | **0.35** (alt) | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L2 | 0.50 | none | **0.50** | High |

Controls where a risk surface exists: 3.45 / 9.0 (38%); 1 criterion scored SA (surface absent).

Stagehand keeps the model on a short leash: each act() call lets it pick one or two clicks or keystrokes on elements that actually exist on the current page, it never writes code or chooses URLs directly, and secrets passed as variables stay out of the prompt. But there is no approval step, no limit on what a hostile page can steer those actions toward, and nothing scopes the logged-in browser sessions the README encourages you to keep. A malicious page can make act() type sensitive values into the wrong field and submit it without a human seeing it.

## Critical gaps
- A page that hijacks act() can steer it to type secrets or page data into an attacker-chosen field and submit, or click irreversible actions, with no human in the loop. (ASI01, LLM01; C5). Evidence: [packages/extension/prompt.ts:186-189](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/prompt.ts#L186-L189); [packages/extension/services/actService.ts:473-485](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L473-L485); [packages/extension/services/actService.ts:179-181](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L179-L181)

## Criterion details

### C1 Identity & least privilege: 0.25 (high confidence)

By default Stagehand launches Chrome with a brand-new temporary profile that is deleted on close, so out of the box the model acts with no logged-in accounts. Nothing narrows authority beyond that: every act() call can operate on any site the session is signed into, secrets passed as variables can be placed into any field the model chooses, and Chrome inherits the full process environment. The README's lead example keeps a persistent profile so sign-ins survive between runs, which gives the model those accounts every time.

- **S L1:** The identity is a dedicated temporary browser profile; anything signed into during the session is usable by every act() call with no per-request scoping. Evidence: [packages/sdk-ts/src/browser/localBrowser.ts:178-181](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/browser/localBrowser.ts#L178-L181); [packages/extension/services/actService.ts:473-485](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L473-L485) (verified)
  - *To reach the next level:* No per-site or per-action authority: variables and sessions are not bound to the domains they belong to.
- **C L1:** All model actions run through one browser identity with no authorization check, and the Chrome subprocess is started with the full parent environment. Evidence: [packages/sdk-ts/src/browser/localBrowser.ts:133-136](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/browser/localBrowser.ts#L133-L136); [packages/extension/services/actService.ts:172-177](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L172-L177) (verified)
  - *To reach the next level:* No authorization layer between the model's chosen action and the browser session, and the browser environment is not scrubbed.
- **D L1:** The default profile is temporary and removed on close, but the README's lead example switches to a persistent signed-in profile. Evidence: [packages/sdk-ts/src/browser/localBrowser.ts:178-181](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/browser/localBrowser.ts#L178-L181); [README.md:58](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/README.md#L58) (verified)
  - *To reach the next level:* Official examples should keep the narrow default, or widening should warn.
- **B L1:** Under the framework rule the documented persistent-profile flow gives a hijacked act() write access to every site the developer signed into. Evidence: [README.md:58](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/README.md#L58); [packages/extension/services/actService.ts:172-177](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L172-L177) (verified)
  - *To reach the next level:* Authority is not limited to one site or to read-only operations.
- **Cap:** none

### C2 Approval gates: 0.23 (high confidence)

There is no approval gate: act() sends the page and instruction to the model and immediately performs whatever click, fill or key press it returns. Stagehand does offer a preview pattern, which its README promotes: observe() returns the exact element, method and arguments it would use, and act() can then run exactly that action, so a developer can build a review step. That pattern is opt-in and has no built-in approve or reject flow. Actions on real websites, such as submissions and purchases, cannot be undone.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** act() with a text instruction executes the model's chosen action directly with no approval hook. Evidence: [packages/extension/services/actService.ts:172-177](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L172-L177); searched `rg -n -S 'approv|confirm|requireApproval|onBeforeAction'` in `packages/sdk-ts/src packages/extension/services packages/extension/controllers packages/extension/handlers` → 0 hits (no approval or confirmation mechanism in the SDK or runtime action path) (verified)
    - *To reach the next level:* No per-call approval showing the exact action before it runs.
  - **C L0:** Every model-chosen action, including the two-step follow-up and self-heal retries, executes ungated. Evidence: [packages/extension/services/actService.ts:179-181](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L179-L181) (verified)
    - *To reach the next level:* No gate on any action path.
  - **D L0:** No approval exists in any configuration of the act path. Evidence: [packages/sdk-ts/src/stagehand.ts:234-246](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/stagehand.ts#L234-L246) (verified)
    - *To reach the next level:* Approval would need to be on by default.
  - **B L0:** Clicks and form submissions on logged-in sites are irreversible and there is no checkpoint or undo. Evidence: [packages/extension/services/actService.ts:172-177](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L172-L177) (verified)
    - *To reach the next level:* No rollback, preview requirement or quantity bounds on consequential actions.
- **opt-in observe-then-act preview** (alt; raw 0.23, cap G1 → 0.23) ← counted
  - **S L2:** observe() returns the exact selector, method and arguments, and act(action) executes that exact action deterministically, so a developer can review it first. Evidence: [README.md:67](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/README.md#L67); [packages/extension/services/actService.ts:93-102](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L93-L102) (verified)
    - *To reach the next level:* No built-in approval UI, risk tiers or reject outcome; the developer must write the review step.
  - **C L1:** The preview only covers actions the developer routes through observe(); act() with a text instruction bypasses it. Evidence: [packages/sdk-ts/src/stagehand.ts:234-246](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/stagehand.ts#L234-L246) (verified)
    - *To reach the next level:* Text-instruction act() calls are not routed through the preview.
  - **D L0:** The pattern is opt-in; the default act() path has no preview. Evidence: [packages/sdk-ts/src/stagehand.ts:234-246](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/stagehand.ts#L234-L246) (verified)
    - *To reach the next level:* Preview would need to be the default.
  - **B L0:** Approved actions on websites remain irreversible. Evidence: [packages/extension/services/actService.ts:172-177](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L172-L177) (verified)
    - *To reach the next level:* No undo or bounds on consequential actions.
- **Cap:** G1: Opt-in mechanism: off in the scored default configuration.

### C3 Tool & action scoping: 0.65 (high confidence)

This is Stagehand's strongest area. The model never gets a general-purpose tool: in act() it can only choose an element that exists in the page snapshot Stagehand captured, and a method from a fixed list (click, fill, type, press, scroll, select, hover, drag); anything else is rejected. It cannot run JavaScript or navigate to a URL of its choosing, and extract() and observe() give it no actions at all. The gaps are that clicking links can still take the browser to any host, including local network addresses, unless the developer configures the optional domain allow and block lists, and that typed text is unrestricted.

- **S L3:** Model output is mapped to an element ID that must exist in the captured snapshot and to a method in a fixed handler map; unknown methods throw. Evidence: [packages/extension/services/actService.ts:438-439](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L438-L439); [packages/extension/handlers/handlerUtils/actHandlerUtils.ts:83-94](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/handlers/handlerUtils/actHandlerUtils.ts#L83-L94); [packages/extension/types/private/handlers.ts:2-14](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/types/private/handlers.ts#L2-L14) (verified)
  - *To reach the next level:* Navigation reached by clicking is not limited to allowed hosts or kept off internal addresses by default.
- **C L3:** The first inference, the two-step follow-up and self-heal all go through the same element mapping and method dispatcher. Evidence: [packages/extension/handlers/handlerUtils/actHandlerUtils.ts:83-94](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/handlers/handlerUtils/actHandlerUtils.ts#L83-L94); [packages/extension/services/actService.ts:179-181](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L179-L181); [packages/extension/services/actService.ts:233-257](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L233-L257) (verified)
  - *To reach the next level:* Server-cached actions replayed in Browserbase mode are not re-checked against the current snapshot.
- **D L2:** The model can act only when the developer calls act(), but act() always exposes the full method set and the domain policy is off by default. Evidence: [packages/extension/understudy/context.ts:124](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/understudy/context.ts#L124); [packages/extension/understudy/domainPolicy.ts:102-106](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/understudy/domainPolicy.ts#L102-L106) (verified)
  - *To reach the next level:* No per-call method allowlist, and host restrictions are opt-in.
- **B L2:** A misused act() is limited to one or two interactions on the current page per call, but those can submit forms and follow links anywhere. Evidence: [packages/extension/services/actService.ts:179-181](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L179-L181); [packages/extension/understudy/domainPolicy.ts:206-226](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/understudy/domainPolicy.ts#L206-L226) (verified)
  - *To reach the next level:* Destinations are not bounded by default.
- **Cap:** none

### C4 Code-execution isolation: 0.53 (high confidence)

The model never writes code that Stagehand runs: there is no shell, no Python and no tool for running JavaScript in the page. The only code involved is the JavaScript of the websites it visits, which runs in Chrome's renderer sandbox. Stagehand leaves that sandbox on by default but turns it off without warning when it sees a CI environment variable or runs as root on Linux. Inside the sandbox, page code has full network access and the cookies of the site it belongs to.

- **S L2:** Untrusted page JavaScript runs in Chrome's renderer sandbox, an OS-level sandbox that keeps network access and the origin's cookies. Evidence: [packages/sdk-ts/src/browser/localBrowser.ts:362-370](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/browser/localBrowser.ts#L362-L370); [packages/sdk-ts/src/browser/localBrowser.ts:127-132](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/browser/localBrowser.ts#L127-L132) (verified)
  - *To reach the next level:* No kernel-separated or network-restricted isolation of the browser.
- **C L3:** The only model-reachable execution is page JavaScript, all inside the renderer; the escape hatch is the documented chromiumSandbox option. Evidence: [packages/sdk-ts/src/browser/localBrowser.ts:362-370](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/browser/localBrowser.ts#L362-L370); [packages/extension/handlers/handlerUtils/actHandlerUtils.ts:83-94](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/handlers/handlerUtils/actHandlerUtils.ts#L83-L94) (verified)
  - *To reach the next level:* Sandbox removal under CI or root is a silent fallback rather than fail-closed.
- **D L2:** The sandbox is on by default but is silently dropped when CI is set or the process runs as root on Linux. Evidence: [packages/sdk-ts/src/browser/localBrowser.ts:362-370](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/browser/localBrowser.ts#L362-L370) (verified)
  - *To reach the next level:* Disabling should require an explicit operator flag only.
- **B L1:** Code inside the renderer has full network egress and the page's logged-in session. Evidence: [packages/sdk-ts/src/browser/localBrowser.ts:133-136](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/browser/localBrowser.ts#L133-L136); [README.md:58](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/README.md#L58) (verified)
  - *To reach the next level:* Egress and session data are not restricted inside the sandbox.
- **Cap:** none

### C5 Untrusted input blast radius: 0.25 (high confidence)

Every act, observe and extract call puts the page's accessibility tree straight into the prompt next to the developer's instruction, with no marking as untrusted and no detection. What helps is structural: the developer's code decides which operation runs on which page, extract() and observe() cannot change anything, and act() is limited to a couple of interactions with existing elements. Within an act() call, though, a hostile page can steer which element is clicked and what is typed, including the developer's secret variables, and submit it with no human involved.

- **S L1:** The operation type is fixed by developer code, but inside act() the model that reads raw page content chooses elements and arguments with no taint tracking. Evidence: [packages/extension/prompt.ts:186-189](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/prompt.ts#L186-L189); searched `rg -n -i 'untrusted|prompt injection|<untrusted'` in `packages/extension/prompt.ts packages/extension/inference.ts` → 0 hits (page content is not marked or treated as untrusted) (verified)
  - *To reach the next level:* No approval or capability restriction triggered by having read untrusted page content.
- **C L1:** All page-derived content (tree, screenshots, iframes) enters the prompt the same way as the instruction. Evidence: [packages/extension/prompt.ts:186-189](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/prompt.ts#L186-L189) (verified)
  - *To reach the next level:* No source is distinguished as untrusted.
- **D L2:** The structural limits are built into the act pipeline and cannot be changed by page content, but they are weak. Evidence: [packages/extension/services/actService.ts:179-181](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L179-L181) (verified)
  - *To reach the next level:* Nothing stronger to keep on by default.
- **B L0:** A hijacked act() can type secret variables or page data into an attacker-chosen field and submit it, and can click irreversible buttons on logged-in sites, all unattended. Evidence: [packages/extension/services/actService.ts:473-485](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L473-L485); [packages/extension/services/actService.ts:179-181](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L179-L181); [packages/extension/services/actService.ts:172-177](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L172-L177) (verified)
  - *To reach the next level:* Either leaks or irreversible actions would need a human in the loop.
- **Cap:** C5-WORSTCASE: A page that hijacks act() can both leak data through form submission and take irreversible actions with no human involved.

### C6 Memory, context & configuration integrity: 0.25 (high confidence)

Stagehand has no long-term memory, loads no instruction files and does not read a .env file, and the default browser profile is thrown away after each run. Two things persist in documented use. The README's lead example keeps a browser profile on disk, so cookies and site storage a page writes carry into later runs. And with Browserbase, server-side caching is on by default: a successful act() is stored and later replayed on a matching page without asking the model, with no check on what was stored.

- **S L1:** No model-writable memory, but cached act results are stored after any successful run and replayed as actions without validation. Evidence: [packages/extension/services/cacheService.ts:61-71](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/cacheService.ts#L61-L71); [packages/extension/services/actService.ts:233-257](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L233-L257) (verified)
  - *To reach the next level:* Cached actions are not validated, provenance-tagged or approved before replay.
- **C L1:** Neither the cache nor a persistent browser profile has controls on what is written. Evidence: [packages/extension/services/cacheService.ts:61-71](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/cacheService.ts#L61-L71); [README.md:58](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/README.md#L58) (verified)
  - *To reach the next level:* No store is controlled.
- **D L1:** Locally each launch gets a fresh profile and the cache is inactive, but the README defaults to a persistent profile and shows caching enabled with Browserbase. Evidence: [packages/sdk-ts/src/browser/localBrowser.ts:178-181](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/browser/localBrowser.ts#L178-L181); [README.md:58](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/README.md#L58); [README.md:331](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/README.md#L331) (verified)
  - *To reach the next level:* Official examples should keep per-session isolation.
- **B L1:** A poisoned cached action persists across the project's sessions and replays as a browser action without a model call. Evidence: [packages/extension/services/actService.ts:233-257](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L233-L257); [packages/extension/services/cacheService.ts:61-71](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/cacheService.ts#L61-L71) (verified)
  - *To reach the next level:* Persistence should only influence text or gated actions.
- **Cap:** none

### C7 Third-party extensions: 1.00 (high confidence)

Stagehand loads no third-party code at runtime. The only extension it installs into Chrome is its own runtime, bundled in the package; there are no plugins, MCP servers, model downloads or package installs that the model can reach.

- **Structural absence:** searched `rg -n -S 'mcpServers|StdioClientTransport|loadPlugin|plugin|npx|pip install|trust_remote_code|--load-extension'` in `packages/sdk-ts/src packages/extension/services packages/extension/controllers packages/extension/llm packages/extension/runtime.ts` → 1 hits (single hit is an error message suggesting --load-extension for Stagehand's own runtime extension); [packages/sdk-ts/src/extensionAssets.ts:14-20](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/extensionAssets.ts#L14-L20)

### C8 Secrets & sensitive-data protection: 0.45 (high confidence)

Secrets meant for web forms are handled well: the developer passes them as variables, the model only ever sees placeholder names, the real values are substituted at the moment of typing, and results report the placeholders. Model and Browserbase API keys are passed as plain configuration into the browser runtime. Tracing is off unless configured. There is no redaction filter for logs or traces, and with Browserbase the variables also go to the server-side cache.

- **S L2:** Variables are opaque placeholders substituted at execution time and returned as placeholders; there is no keychain or redaction layer. Evidence: [packages/extension/services/actService.ts:473-485](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L473-L485); [packages/extension/services/actService.ts:488-506](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L488-L506); searched `rg -n -S 'redact|mask|scrub'` in `packages/extension/logger.ts packages/extension/services packages/extension/handlers packages/sdk-ts/src/stagehand.ts` → 0 hits (no redaction in the logging path) (verified)
  - *To reach the next level:* No secret store and no redaction before logs and traces.
- **C L2:** Model-bound prompts and action results are protected for variables; logs, traces and the Browserbase cache request are not. Evidence: [packages/extension/services/actService.ts:488-506](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L488-L506); [packages/extension/services/cacheService.ts:83-93](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/cacheService.ts#L83-L93) (verified)
  - *To reach the next level:* Logs, traces and cache requests are not covered.
- **D L2:** Tracing is inert without an explicit telemetry config and the default log level is info. Evidence: [packages/extension/tracing.ts:122-123](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/tracing.ts#L122-L123); [packages/sdk-ts/src/clientSchemas.ts:222](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/clientSchemas.ts#L222) (verified)
  - *To reach the next level:* Redaction is not always on.
- **B L1:** Long-lived site credentials and provider API keys are in play, and variables are not bound to the sites they belong to. Evidence: [packages/protocol/schemas.ts:843-848](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/protocol/schemas.ts#L843-L848); [packages/protocol/schemas.ts:1622](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/protocol/schemas.ts#L1622); [packages/extension/services/actService.ts:473-485](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L473-L485) (verified)
  - *To reach the next level:* Keys are not scoped or short-lived.
- **Cap:** none

### C9 Audit & traceability: 0.35 (high confidence)

By default Stagehand prints info-level logs to standard error, which record that an inference finished and how many tokens it used but not which element was clicked or what was typed. The details of each action come back to the developer's code in the act() result, not in a log. When a developer configures OpenTelemetry, every RPC call and action gets a timestamped span with trace correlation shipped to their collector, but that is opt-in, best-effort and has no actor attribution.

- **default configuration** (default; raw 0.30 → 0.30)
  - **S L1:** Default logs are unstructured info lines that omit the executed action. Evidence: [packages/extension/services/actService.ts:289-296](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L289-L296); [packages/sdk-ts/src/stagehand.ts:385](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/stagehand.ts#L385) (verified)
    - *To reach the next level:* No structured record of each action with arguments and timestamps by default.
  - **C L1:** Only inference completion is logged at the default level for the act path. Evidence: [packages/extension/services/actService.ts:289-296](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L289-L296) (verified)
    - *To reach the next level:* Executed actions are not logged.
  - **D L2:** Logs are on by default and written by the SDK process, out of the model's reach. Evidence: [packages/sdk-ts/src/clientSchemas.ts:222](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/clientSchemas.ts#L222); [packages/sdk-ts/src/stagehand.ts:385](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/stagehand.ts#L385) (verified)
    - *To reach the next level:* Not written by an independent component.
  - **B L1:** Log notifications are fire-and-forget; nothing is persisted. Evidence: [packages/sdk-ts/src/stagehand.ts:385](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/stagehand.ts#L385) (verified)
    - *To reach the next level:* Records are not durable per action.
- **opt-in OpenTelemetry tracing** (alt; raw 0.35, cap G1 → 0.35) ← counted
  - **S L2:** Every RPC request gets a span with method and timestamps, with W3C trace propagation from the SDK. Evidence: [packages/extension/rpcRouter.ts:93-101](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/rpcRouter.ts#L93-L101); [packages/extension/tracing.ts:122-123](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/tracing.ts#L122-L123) (verified)
    - *To reach the next level:* No actor or principal attribution.
  - **C L2:** All RPC methods, including act, observe, extract and page commands, are traced. Evidence: [packages/extension/rpcRouter.ts:93-101](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/rpcRouter.ts#L93-L101) (verified)
    - *To reach the next level:* No approvals to record; no config-change records.
  - **D L0:** Tracing is inert without an explicit telemetry config. Evidence: [packages/extension/tracing.ts:122-123](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/tracing.ts#L122-L123) (verified)
    - *To reach the next level:* Tracing would need to be on by default.
  - **B L1:** Spans are batch-exported best-effort. Evidence: [packages/extension/tracing.ts:159-176](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/tracing.ts#L159-L176) (verified)
    - *To reach the next level:* Export failures do not stop actions.
- **Cap:** G1: Opt-in mechanism: off in the scored default configuration.

### C10 Limits & kill switch: 0.50 (high confidence)

Each act() call is bounded in code to at most two actions and a few model calls, and there is no autonomous agent loop: the developer's code decides how many calls to make. Page commands have default timeouts, but act, observe and extract have no time limit unless the caller sets one, and model calls have no timeout or spending cap. Closing a locally launched browser kills Chrome's whole process group, which stops anything in flight.

- **S L2:** Per-call step bound plus default timeouts on page commands and an optional per-call timeout; halt kills the browser process group. Evidence: [packages/extension/services/actService.ts:179-181](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L179-L181); [packages/sdk-ts/src/rpcClient.ts:78-82](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/rpcClient.ts#L78-L82); [packages/sdk-ts/src/browser/localBrowser.ts:566-575](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/browser/localBrowser.ts#L566-L575) (verified)
  - *To reach the next level:* No token or cost cap and no default time limit on act, observe or extract.
- **C L2:** The step bound covers every act() call and timeouts cover page commands; there are no sub-agents or background tasks. Evidence: [packages/extension/services/actService.ts:179-181](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L179-L181); [packages/sdk-ts/src/rpcClient.ts:78-82](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/rpcClient.ts#L78-L82) (verified)
  - *To reach the next level:* Model calls are not covered by any timeout.
- **D L2:** The step bound is fixed in code; time limits are left to the caller. Evidence: [packages/sdk-ts/src/rpcClient.ts:78-82](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/rpcClient.ts#L78-L82); [packages/extension/services/actService.ts:55](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/extension/services/actService.ts#L55) (verified)
  - *To reach the next level:* No sensible default time limit for model-driven calls.
- **B L2:** Each call can do little, and closing the browser kills everything, but a caller loop has no spend ceiling. Evidence: [packages/sdk-ts/src/browser/localBrowser.ts:566-575](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/browser/localBrowser.ts#L566-L575); [packages/sdk-ts/src/rpcClient.ts:78-82](https://github.com/browserbase/stagehand/blob/0336409092b85beeba38ea3cd4953f45eb96ed6b/packages/sdk-ts/src/rpcClient.ts#L78-L82) (verified)
  - *To reach the next level:* No per-run time or cost ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Raw page accessibility tree placed in the act/observe/extract prompt (packages/extension/prompt.ts:188) · [B] sensitive data/systems: Logged-in browser session and developer-supplied variables substituted into actions (packages/extension/services/actService.ts:482) · [C] state change / egress: Clicks, typing and form submission on the live page (packages/extension/handlers/handlerUtils/actHandlerUtils.ts:114) · Same default session? Yes

## Highest-impact improvements
1. Bind each variable to the domains it may be typed on and refuse substitution elsewhere. (C1 S L1→L2, +0.075 before caps; Playbook 4)
2. Add an onBeforeAction hook to act() that shows the exact element, method and arguments and supports reject, and enable it for fill and submit-like actions by default. (C2 S L0→L3, +0.225 before caps; Playbook 5)
3. Log every executed action (method, selector, placeholder arguments, timestamp) at info level. (C9 S L1→L2, +0.075 before caps)
4. Apply a default timeout to act, observe, extract and model calls, and add an optional token budget per instance. (C10 D L2→L3, +0.050 before caps)
5. Keep Chrome's sandbox on under CI and require the explicit chromiumSandbox option to turn it off. (C4 D L2→L3, +0.050 before caps)

## Re-audit log
- C3 D: L3 → L2. act() always exposes the full method set and host limits are opt-in, so it is not a read-only default with explicit write enabling.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the TypeScript SDK (packages/sdk-ts), its in-browser runtime (packages/extension) and the shared protocol (packages/protocol). The browse CLI (packages/cli), the agent-harness integrations and facade MCP server (packages/integrations), the Python and Go SDKs, docs, examples and evals were not scored.
- Browserbase cloud behaviour (session isolation, server-side cache keying, Model Gateway) runs outside this repository and was not examined; ratings that depend on it describe only the client code.
