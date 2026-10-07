# Defense-in-Depth Score: Webcmd

**Repo:** https://github.com/agentrhq/webcmd · **Commit:** `9d8ea440fd3d86ddadf53e66b6f27db6134b0312` (v0.8.4) · **Reviewed:** 2026-10-04
**What it is:** Self-learning browser automation CLI for AI coding agents: drives a real logged-in browser, with site memory, plugins and external-CLI passthrough.
**Category:** AI Assistants
**Scored configuration:** Local mode (bundled Cloak browser), installed via npm with the shipped webcmd-browser skill, invoked by a host coding agent through Bash; no flags.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents no · external communication yes

## Score: 2.5 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | High |
| C2 | Approval gates | L1 | L0 | L0 | L0 | 0.07 | C2-POWERBYPASS | **0.07** | High |
| C3 | Tool & action scoping | L1 | L1 | L0 | L0 | 0.15 | — | **0.15** | High |
| C4 | Code-execution isolation | L3 | L1 | L1 | L0 | 0.35 | G2 | **0.25** | High |
| C5 | Untrusted input blast radius | L2 | L1 | L2 | L0 | 0.33 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L0 | 0.00 | C7-RCELOAD | **0.00** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L0 | 0.40 | — | **0.40** | High |
| C9 | Audit & traceability | L2 | L1 | L0 | L1 | 0.28 | G1 | **0.28** (alt) | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |


Webcmd gives an AI agent a real browser logged into the user's accounts and lets it run any program against any site, while its skill pre-approves every webcmd command in the host agent. Browser programs run in a QuickJS sandbox, but the bridge to Playwright is not a complete boundary. External-CLI passthrough and plugins run arbitrary unpinned code on the host. A prompt injection in any page can leak data and act on the user's accounts with no human in the loop.

## Critical gaps
- A hijacked agent acts with every account logged into the browser profile and, via external-CLI passthrough, the user's gh/docker credentials. (ASI03; C1) — [skills/webcmd-browser/SKILL.md:67](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/skills/webcmd-browser/SKILL.md#L67); [src/external.ts:219](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external.ts#L219)
- The shipped skill pre-approves every 'webcmd' command, so browser runs, plugin installs, and passthrough to registered binaries skip the host agent's approval prompt. (ASI02, ASI09; C2) — [skills/webcmd-browser/SKILL.md:4](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/skills/webcmd-browser/SKILL.md#L4); [src/cli.ts:2188](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/cli.ts#L2188)
- Registered external binaries run directly on the host, outside the QuickJS sandbox, whose host bridge is also not a complete boundary. (ASI05; C4) — [src/external.ts:256](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external.ts#L256)
- Untrusted page content, the user's logged-in sessions, and unrestricted navigation share one session, so a prompt injection can leak data and act on the user's accounts unattended. (ASI01; C5) — [skills/webcmd-browser/SKILL.md:4](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/skills/webcmd-browser/SKILL.md#L4); [src/browser/run/playwright-transport.ts:55-67](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/playwright-transport.ts#L55-L67)
- Invoking an external CLI that isn't installed auto-runs an unpinned global npm install with no consent, and plugins from any git repo load in-process. (ASI04; C7) — [src/external.ts:171](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external.ts#L171); [src/external.ts:193](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external.ts#L193); [src/discovery.ts:78](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/discovery.ts#L78)

## Criterion details

### C1 Identity & least privilege — 0.25 (high)

Webcmd drives a real browser with whichever logged-in profile the agent names, so it acts with the user's full sessions on every site logged into that profile. Profiles are separate cookie jars, which lets a user keep a narrow 'work' or 'social' identity, but nothing inside a profile limits which site or action the agent can reach. Webcmd also passes external tools such as gh and docker the user's full environment and ambient credentials. A hijacked agent therefore carries the user's accounts across services.

- **S L1:** Authority is a named browser profile (a dedicated cookie jar) that is broad within itself: any site, any action, with the user's logged-in sessions. — [skills/webcmd-browser/SKILL.md:67](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/skills/webcmd-browser/SKILL.md#L67); [src/browser/run/playwright-transport.ts:55-67](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/playwright-transport.ts#L55-L67) (verified)
  - *To reach the next level:* No per-site or per-action scoping of a profile, and no read-only identity for read commands.
- **C L1:** Browser commands use the selected profile, but external-CLI passthrough and plugins run with the user's full environment and ambient credentials. — [src/external.ts:219](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external.ts#L219); [src/discovery.ts:78](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/discovery.ts#L78) (verified)
  - *To reach the next level:* Subprocesses and plugins don't use a scoped identity; external CLIs inherit the full environment.
- **D L2:** A fresh profile has no logins; authority widens only when the user logs in during a human handoff, which is an operator action with no further warning. — [skills/webcmd-browser/SKILL.md:67](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/skills/webcmd-browser/SKILL.md#L67); [skills/webcmd-browser/SKILL.md:67-68](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/skills/webcmd-browser/SKILL.md#L67-L68) (verified)
  - *To reach the next level:* No read-only default; write capability on every logged-in site comes with the login.
- **B L0:** A hijacked session reaches every account logged into the profile (social, email, shopping, AI tools) and, through passthrough, gh/docker with the user's own credentials. — [skills/webcmd-browser/SKILL.md:67](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/skills/webcmd-browser/SKILL.md#L67); [src/external-clis.yaml:24](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external-clis.yaml#L24); [src/external.ts:219](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external.ts#L219) (verified)
  - *To reach the next level:* Blast radius isn't confined to one system or to read-only operations.
- **Cap:** none

### C2 Approval gates — 0.07 (high)

Webcmd gives a host agent no approval signal for its most powerful tool: 'browser run' executes any program against any logged-in site and mixes reads and writes with no risk annotation, dry-run, or read-only mode. Site adapter commands must declare read or write, which helps. Worse, the shipped skill pre-approves every 'webcmd' command in Claude Code, so browser runs, plugin installs, and passthrough to docker, gh, or any binary the model registers skip the host's per-call prompt. The only payment safeguard is a sentence in the skill.

- **S L1:** Adapter commands must declare access 'read' or 'write' at registration, but the main 'browser run' tool mixes reads and writes with no annotation, preview, or read-only mode. — [src/registry.ts:247-250](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/registry.ts#L247-L250); [skills/webcmd-browser/SKILL.md:18](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/skills/webcmd-browser/SKILL.md#L18) (verified)
  - *To reach the next level:* No risk signal on browser run, no separate read/write tools for it, no dry-run.
- **C L0:** The shipped skill sets allowed-tools Bash(webcmd:*), so the host auto-approves every webcmd invocation, including browser run, plugin install, and external passthrough. — [skills/webcmd-browser/SKILL.md:4](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/skills/webcmd-browser/SKILL.md#L4); [src/cli.ts:2188](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/cli.ts#L2188); [src/cli.ts:2227](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/cli.ts#L2227) (verified)
  - *To reach the next level:* The most powerful paths would need to stay behind the host's per-call approval.
- **D L0:** No approval exists by default, and installing the skill removes the host's own prompts for webcmd commands. — [skills/webcmd-browser/SKILL.md:4](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/skills/webcmd-browser/SKILL.md#L4) (verified)
  - *To reach the next level:* A default approval or confirmation step would be needed for write actions.
- **B L0:** Wrongly approved actions include irreversible posts, messages, purchases, and arbitrary passthrough commands; payment confirmation is only prompt text. — [skills/webcmd-browser/SKILL.md:18](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/skills/webcmd-browser/SKILL.md#L18); [src/browser/run/runner.ts:288](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/runner.ts#L288) (verified)
  - *To reach the next level:* No undo, preview, or rate limit on consequential browser actions.
- **Cap:** C2-POWERBYPASS — The shipped skill's allowed-tools Bash(webcmd:*) (verified) pre-approves the browser-run and external-passthrough paths in the host (Claude Code skill-permission behaviour inferred).

### C3 Tool & action scoping — 0.15 (high)

The main tool is a general-purpose program runner: any URL, any site, any action in the logged-in browser, limited only by a short denylist of Playwright methods. The separate 'web fetch' command does block private and internal addresses, and adapter commands take typed arguments, but browser run, page.request, and external passthrough have no URL or argument allowlists. Every tool is on by default.

- **S L1:** Browser run is bounded only by a denylist of Playwright protocol methods; URLs, including localhost and metadata addresses, aren't validated. — [src/browser/run/playwright-transport.ts:55-67](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/playwright-transport.ts#L55-L67); searched `rg -n 'isSafeFetchHostname|isSafeAddress|safe-proxy'` in `src/browser` → 0 hits (The SSRF guard used by web fetch is never applied to the browser path.) (verified)
  - *To reach the next level:* No URL/host allowlist or private-address block on the browser path; general tools aren't replaced by narrow ones.
- **C L1:** Only web fetch (SSRF-safe proxy) and typed adapter arguments validate inputs; browser run and external passthrough don't. — [src/fetch/client.ts:152](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/fetch/client.ts#L152); [src/external.ts:219](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external.ts#L219) (verified)
  - *To reach the next level:* Most built-in tools would need validation, including browser run and passthrough.
- **D L0:** Browser run, passthrough to external CLIs, and plugin install are all available by default. — [skills/webcmd-browser/SKILL.md:4](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/skills/webcmd-browser/SKILL.md#L4); [src/external-clis.yaml:24](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external-clis.yaml#L24) (verified)
  - *To reach the next level:* No read-only default tool set.
- **B L0:** A misused tool reaches any site under the user's sessions and arbitrary binaries through passthrough. — [src/external.ts:256](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external.ts#L256); [skills/webcmd-browser/SKILL.md:67](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/skills/webcmd-browser/SKILL.md#L67) (verified)
  - *To reach the next level:* Tool reach isn't scoped to a project or bounded in quantity.
- **Cap:** none

### C4 Code-execution isolation — 0.25 (high)

Browser-run programs execute in a QuickJS engine compiled to WebAssembly, with no Node APIs and only injected host functions, which is a real boundary. But the host-side Playwright bridge uses a denylist that is not a strict boundary. Separately, external-CLI passthrough runs any binary the model registers directly on the host, and plugins load in-process. An escape lands in a Node process holding the user's files, network, and browser sessions.

- **S L3:** QuickJS (quickjs-emscripten) guest with no Node APIs; only writeArtifact and the Playwright transport are injected, plus memory and CPU interrupts. — [src/browser/run/runner.ts:355-358](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/runner.ts#L355-L358); [src/browser/run/runner.ts:550-551](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/runner.ts#L550-L551); [src/browser/run/runner.ts:351](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/runner.ts#L351); [src/browser/run/quickjs-host.ts:52-55](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/quickjs-host.ts#L52-L55) (verified)
  - *To reach the next level:* Not a kernel-separated sandbox; the injected Playwright capability is broad.
- **C L1:** Browser run is sandboxed, but external passthrough (spawnSync on the host) and plugins (in-process import) reach the host, and the sandbox's host bridge is not a complete boundary. — [src/external.ts:219](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external.ts#L219); [src/discovery.ts:78](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/discovery.ts#L78) (verified)
  - *To reach the next level:* Passthrough and plugin execution would need to run inside a boundary, and the host transport would need an allowlist.
- **D L1:** The QuickJS sandbox is always on for browser run, but the model can get unsandboxed execution without a human by registering a binary and invoking it through passthrough. — [src/cli.ts:2188](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/cli.ts#L2188); [src/external.ts:256](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external.ts#L256); [src/external.ts:219](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external.ts#L219) (verified)
  - *To reach the next level:* Escalation to host execution would need per-call human approval.
- **B L0:** An escape lands in the webcmd Node process: the user's home directory, unrestricted network, and the browser's logged-in sessions. — [src/browser/run/artifacts.ts:18-32](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/artifacts.ts#L18-L32); [src/external.ts:219](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external.ts#L219) (verified)
  - *To reach the next level:* Nothing narrows what the host process holds.
- **Cap:** G2 — The model can obtain unsandboxed execution through external register plus passthrough; the QuickJS sandbox's host bridge is also not a complete boundary.

### C5 Untrusted input blast radius — 0.25 (high)

Webcmd's job is to read untrusted web pages into an agent that holds the user's logged-in sessions and can navigate anywhere, so all three Rule-of-Two legs are present in one session. Browser-run output is structured JSON with page URL and title kept apart from the result, but nothing marks content as untrusted or restricts what the agent can do after reading it. The skill's 'page content is untrusted' line is a prompt, not a control. A hijacked agent can exfiltrate by navigating to an attacker URL and can post, message, or buy with no human involved.

- **S L2:** Browser-run results are structured, with page URL and title separate from the program's result and logs. — [src/browser/run/types.ts:121-125](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/types.ts#L121-L125) (verified)
  - *To reach the next level:* No untrusted flag or provenance the host can act on, and no read-only or no-egress mode.
- **C L1:** Only browser-run results carry this structure; there's no provenance tagging on snapshot text, fetch output, or memory context. — [src/browser/run/types.ts:121-125](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/types.ts#L121-L125); searched `rg -n -i 'untrusted'` in `src/browser/run` → 0 hits (No untrusted flag anywhere in browser-run output.) (verified)
  - *To reach the next level:* Most untrusted sources would need the same structured separation.
- **D L2:** Structured output is always on and has no switch. — [src/browser/run/types.ts:121-125](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/types.ts#L121-L125) (verified)
  - *To reach the next level:* Nothing stops the agent acting on the content it reads.
- **B L0:** A hijack can leak data (navigate or page.request to any URL with the user's sessions) and take irreversible actions on logged-in sites, unattended under the skill's pre-approval. — [skills/webcmd-browser/SKILL.md:4](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/skills/webcmd-browser/SKILL.md#L4); [src/browser/run/playwright-transport.ts:55-67](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/playwright-transport.ts#L55-L67); [skills/webcmd-browser/SKILL.md:67](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/skills/webcmd-browser/SKILL.md#L67) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions would need human approval.
- **Cap:** C5-WORSTCASE — B is L0: leaking data and taking irreversible actions is possible unattended in the default configuration.

### C6 Memory, context & configuration integrity — 0.45 (high)

Webcmd keeps 'site memory' in a local git repository under the user's home directory and feeds it back to later agents as navigation guidance. Writes go through checks: every fact needs a verification date, size limits apply, candidate observations are rejected if they look like secrets, and git history lets the user roll back. The first visit to a site may pull a seed from webcmd's cloud service, which is written into memory without the same fact validation. Nothing is loaded from the working directory, but memory has no expiry and can steer future actions.

- **S L2:** Checkpoint validation (dated facts, line bounds) and secret rejection on candidates; candidates carry provenance; no workspace config loaded. — [src/site-memory/checkpoint.ts:273-299](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/site-memory/checkpoint.ts#L273-L299); [src/site-memory/checkpoint.ts:260](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/site-memory/checkpoint.ts#L260); [src/site-memory/candidates.ts:46](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/site-memory/candidates.ts#L46); searched `rg -n 'dotenv|loadEnvFile'` in `src` → 0 hits (No workspace .env or project config loading.) (verified)
  - *To reach the next level:* Memory has no expiry, and seed content bypasses fact validation.
- **C L2:** Checkpoint writes are validated, but the remote seed path writes SITE.md directly. — [src/site-memory/context.ts:110](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/site-memory/context.ts#L110); [src/site-memory/checkpoint.ts:273-299](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/site-memory/checkpoint.ts#L273-L299) (verified)
  - *To reach the next level:* Every write path would need the same validation, including seeds.
- **D L2:** Memory lives per OS user under ~/.webcmd/sites; it's a single-user local store. — [src/site-memory/local-store.ts:352](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/site-memory/local-store.ts#L352) (verified)
  - *To reach the next level:* The model can write any product namespace; no retention limit.
- **B L1:** Poisoned memory persists across the user's sessions and guides future browser actions, though git history allows inspection and rollback. — [src/site-memory/checkpoint.ts:273-299](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/site-memory/checkpoint.ts#L273-L299); [src/site-memory/context.ts:110](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/site-memory/context.ts#L110) (verified)
  - *To reach the next level:* Memory would need to be read-only for actions, or applied only after human review.
- **Cap:** none

### C7 Third-party extensions — 0.00 (high)

Plugins install by cloning the latest commit of any git repository the agent names. Dependencies are installed with scripts disabled, but the plugin then loads in-process with full access. External CLIs are worse: invoking one that isn't installed auto-installs it with an unpinned global 'npm install -g' (install scripts run) and no prompt, and the model can register new CLIs with its own install command. All of this sits behind the skill's blanket pre-approval.

- **S L0:** External CLIs auto-install unpinned npm packages globally on first use, and model-registered install commands run the same way; plugins clone HEAD of any repo. — [src/external.ts:171](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external.ts#L171); [src/external.ts:193](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external.ts#L193); [src/external-clis.yaml:38](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external-clis.yaml#L38); [src/external.ts:259](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external.ts#L259); [src/plugin.ts:241](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/plugin.ts#L241) (verified)
  - *To reach the next level:* Sources would need pinning and integrity checks before install.
- **C L0:** Neither plugins nor external CLIs are verified; the lock file records a commit only after cloning. — [src/plugin.ts:241](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/plugin.ts#L241); searched `rg -n 'sha256|integrity|signature'` in `src/plugin.ts` → 1 hits (Single hit is a comment about override reconciliation, not install verification.) (verified)
  - *To reach the next level:* At least one extension type would need verification.
- **D L0:** External CLIs install automatically on first invocation with no consent step. — [src/external.ts:171](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external.ts#L171); [src/external.ts:193](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external.ts#L193) (verified)
  - *To reach the next level:* Installs would need explicit consent showing the package and command.
- **B L0:** Plugins run in-process; external CLIs run as the user with the full environment. — [src/discovery.ts:78](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/discovery.ts#L78); [src/external.ts:219](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external.ts#L219) (verified)
  - *To reach the next level:* Extensions would need a separate process with a scrubbed environment.
- **Cap:** C7-RCELOAD — Invoking a registered external CLI that isn't installed runs an unpinned 'npm install -g' (with install scripts) without consent (verified in external.ts).

### C8 Secrets & sensitive-data protection — 0.40 (high)

Webcmd has no telemetry and redacts secret-looking fields (cookie, token, authorization, and similar) from browser-run results, logs, and traces before they reach the agent. The hosted API key, an opt-in, uses the macOS keychain or a 0600 file. But the high-value secrets are the browser's session cookies, and their protection from browser-run programs is not a complete boundary. By default the first visit to each site also sends the domain to webcmd's cloud seed service. External CLIs inherit the full environment.

- **S L2:** Key-name and pattern redaction on browser-run results and logs; keychain on macOS, 0600 file elsewhere. — [src/observation/redaction.ts:14](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/observation/redaction.ts#L14); [src/browser/run/runner.ts:701](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/runner.ts#L701); [src/hosted/credentials.ts:168](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/hosted/credentials.ts#L168); [src/hosted/credentials.ts:295](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/hosted/credentials.ts#L295) (verified)
  - *To reach the next level:* No encrypted store on Linux/Windows, and redaction is pattern-based, not by design.
- **C L2:** Model-bound browser-run output, logs, and traces are redacted; subprocess environments aren't scrubbed. — [src/browser/run/runner.ts:701](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/runner.ts#L701); [src/browser/run/runner.ts:379](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/runner.ts#L379); [src/external.ts:219](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external.ts#L219) (verified)
  - *To reach the next level:* Subprocess environments and session material would need covering.
- **D L2:** No telemetry; redaction always applied; the seed lookup discloses visited domains by default, and you can opt out. — [PRIVACY.md:7](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/PRIVACY.md#L7); [src/site-memory/seed-client.ts:8](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/site-memory/seed-client.ts#L8); [src/site-memory/seed-client.ts:19](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/site-memory/seed-client.ts#L19) (verified)
  - *To reach the next level:* The domain-disclosing seed lookup would need to be opt-in.
- **B L0:** Long-lived session cookies for every logged-in site are at risk from model-written programs. — [skills/webcmd-browser/SKILL.md:67](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/skills/webcmd-browser/SKILL.md#L67) (verified)
  - *To reach the next level:* Session material would need to stay out of the model's reach.
- **Cap:** none

### C9 Audit & traceability — 0.28 (high)

By default webcmd keeps no durable record of what the agent did in the browser. The daemon holds an in-memory buffer of the last 200 messages, only for failed commands, and any local caller can clear it. Detailed traces of actions, network, and console exist but are off by default and are written only when a command finishes. Site-memory changes are committed to git, which is the only durable record.

- **default configuration** (default; raw 0.20 → 0.20)
  - **S L1:** Only failed commands are logged, unstructured, to an in-memory ring buffer. — [src/daemon/server.ts:303-305](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/daemon/server.ts#L303-L305); [src/daemon/server.ts:515](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/daemon/server.ts#L515); searched `rg -n 'appendFile'` in `src` → 0 hits (No persistent log writer anywhere.) (verified)
    - *To reach the next level:* No structured record of every tool call by default.
  - **C L1:** Successful actions aren't recorded at all. — [src/daemon/server.ts:515](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/daemon/server.ts#L515) (verified)
    - *To reach the next level:* All browser and passthrough commands would need recording.
  - **D L1:** The buffer is on, but it's in memory and DELETE /logs clears it. — [src/daemon/server.ts:366](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/daemon/server.ts#L366) (verified)
    - *To reach the next level:* The record would need to live outside the reach of the agent's own calls.
  - **B L0:** Records are lost when the daemon exits; failures to log are silent. — [src/daemon/server.ts:303-305](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/daemon/server.ts#L303-L305) (verified)
    - *To reach the next level:* Records would need flushing per action.
- **opt-in --trace on (observation artifacts)** (alt; raw 0.28, cap G1 → 0.28) ← counted
  - **S L2:** Trace artifacts record action, network, and console events as JSONL with redaction. — [src/observation/artifact.ts:71-73](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/observation/artifact.ts#L71-L73); [src/observation/events.ts:1](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/observation/events.ts#L1) (verified)
    - *To reach the next level:* No actor attribution or tamper evidence.
  - **C L1:** Traces cover browser commands only, not passthrough or plugin installs. — [src/observation/artifact.ts:71](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/observation/artifact.ts#L71) (verified)
    - *To reach the next level:* All tool paths would need tracing.
  - **D L0:** Trace defaults to 'off'. — [src/browser/command-catalog.ts:193](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/command-catalog.ts#L193); [src/command-surface.ts:476](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/command-surface.ts#L476) (verified)
    - *To reach the next level:* Tracing would need to be on by default.
  - **B L1:** Trace files are written when the command finishes, so a crash loses them. — [src/observation/artifact.ts:71-73](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/observation/artifact.ts#L71-L73) (verified)
    - *To reach the next level:* Records would need flushing per action.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C10 Limits & kill switch — 0.40 (high)

Each browser run has a default 30-second wall-clock and CPU limit, a 128 MB memory cap, and an output cap. Ctrl-C cancels the in-flight run in the daemon. But the agent can raise the timeout to any value with --timeout, since there is no ceiling. External passthrough commands have no limit, and the daemon and browser keep running in the background after a command ends.

- **S L2:** Server-enforced timeout, CPU interrupt, memory and output caps on browser run; SIGINT cancels the daemon run. — [src/browser/run/types.ts:5](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/types.ts#L5); [src/browser/run/types.ts:8](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/types.ts#L8); [src/browser/run/runner.ts:351](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/runner.ts#L351); [src/signal-cancel.ts:31](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/signal-cancel.ts#L31) (verified)
  - *To reach the next level:* No caps on every operation (passthrough has none) and no rate limits.
- **C L2:** Limits cover browser run and connection timeouts but not external passthrough or plugin installs. — [src/browser/run/runner.ts:351](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/runner.ts#L351); [src/external.ts:219](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/external.ts#L219) (verified)
  - *To reach the next level:* Spawned processes would need to count against the same limits.
- **D L1:** Defaults are sensible, but the timeout accepts any positive integer the agent passes. — [src/browser/run/runner.ts:134](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/runner.ts#L134) (verified)
  - *To reach the next level:* A hard ceiling the model can't exceed.
- **B L1:** A runaway can be given an arbitrarily long timeout, and the daemon and browser keep running after commands end. — [src/browser/run/runner.ts:134](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/browser/run/runner.ts#L134); [src/daemon.ts:31](https://github.com/agentrhq/webcmd/blob/9d8ea440fd3d86ddadf53e66b6f27db6134b0312/src/daemon.ts#L31) (verified)
  - *To reach the next level:* Tight ceilings and no background process left running after stop.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Arbitrary web pages read by browser run/snapshot (src/browser/run/types.ts:121) · [B] sensitive data/systems: Logged-in browser profiles and cookies (skills/webcmd-browser/SKILL.md:67; src/browser/run/playwright-transport.ts:55) · [C] state change / egress: Unrestricted navigation, page.request, form submission, external passthrough (src/external.ts:219) · Same default session? Yes

## Highest-impact improvements
1. Replace the Playwright protocol denylist with an allowlist of the methods browser programs need. — C4 C L1→L2, +0.075 before caps (Playbook 3 step 1)
2. Drop allowed-tools Bash(webcmd:*) from the shipped skill, or narrow it to read-only subcommands, so the host still prompts for browser runs, installs, and passthrough. — C2 C L0→L2, +0.150 before caps (Playbook 5)
3. Remove auto-install from external-CLI passthrough and require explicit, pinned installs with consent. — C7 D L0→L2, +0.100 before caps (Playbook 3)
4. Write a durable, per-action JSONL log of every browser command (URL, program hash, result status) by default. — C9 S L1→L2, +0.075 before caps (Playbook 1 step 3)
5. Add a hard ceiling to --timeout for browser run. — C10 D L1→L2, +0.050 before caps

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Some findings rely on playwright-core 1.61.1 server behaviour read in upstream source and were not run.
- The effect of the skill's allowed-tools frontmatter is Claude Code host behaviour, inferred and not tested.
- Hosted (Webcmd Cloud) mode, the SLAB macOS runtime, Electron-app adapters and the cloakbrowser package internals were not examined in depth.
- No text aimed at AI reviewers was found (rg for auditor/AI reviewer/ignore previous returned nothing).
