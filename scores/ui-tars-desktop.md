# Defense-in-Depth Score: UI-TARS Desktop / Agent TARS

**Repo:** https://github.com/bytedance/UI-TARS-desktop · **Commit:** `2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a` · **Reviewed:** 2026-10-03
**What it is:** Multimodal GUI agent stack (desktop app + Agent TARS CLI) controlling computer and browser
**Category:** AI Assistants
**Scored configuration:** Agent TARS CLI as led by the README (npx @agent-tars/cli, no flags): local environment with in-memory browser, filesystem, and shell MCP servers, workspace = current directory, web UI on 127.0.0.1.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions opt-in · sub agents no · external communication yes

## Score: 1.6 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C3 | Tool & action scoping | L2 | L1 | L0 | L0 | 0.23 | — | **0.23** | High |
| C4 | Code-execution isolation | L2 | L2 | L0 | L2 | 0.40 | G1 | **0.40** (alt) | Medium |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L0 | 0.05 | C6-REPOCONFIG | **0.05** | Medium |
| C7 | Third-party extensions | L0 | L0 | L0 | L0 | 0.00 | C7-RCELOAD | **0.00** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |


As shipped, Agent TARS gives the model an unrestricted shell, file tools, and a full browser on your own machine, with no approval step, no sandbox, and no separation between web content and your instructions. The dominant risk is goal hijack: a web page or file the agent reads can make it run commands, leak secrets from its environment, or act in logged-in websites, all unattended. Workspace configuration is also not integrity-protected, and the bundled browser's security settings are not locked down. A remote sandbox mode exists but is opt-in.

## Critical gaps
- Model-written shell commands run directly on the host as the user, with the full environment and no approval or sandbox. (ASI05, T11; C4) — [packages/agent-infra/mcp-servers/commands/src/server.ts:143](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/commands/src/server.ts#L143); [multimodal/agent-tars/core/src/environments/local/index.ts:178](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/environments/local/index.ts#L178)
- A hijacked session can both exfiltrate (arbitrary browser navigation, shell network) and take irreversible actions with no human involved. (ASI01, LLM01; C5) — [packages/agent-infra/mcp-servers/browser/src/tools/navigate.ts:18](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/browser/src/tools/navigate.ts#L18); [packages/agent-infra/mcp-servers/commands/src/server.ts:143](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/commands/src/server.ts#L143)
- The system prompt directs the model to install packages itself through the ungated shell, running remote install scripts with the user's full environment. (ASI04, T17; C7) — [multimodal/agent-tars/core/src/prompt.ts:55](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/prompt.ts#L55); [packages/agent-infra/mcp-servers/commands/src/server.ts:136](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/commands/src/server.ts#L136)
- The agent runs with the user's full ambient authority; every shell child inherits all environment secrets. (ASI03, T3; C1) — [packages/agent-infra/mcp-servers/commands/src/server.ts:136](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/commands/src/server.ts#L136); [multimodal/tarko/agent-cli/src/config/loader.ts:65-69](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent-cli/src/config/loader.ts#L65-L69)
- No approval gate exists and the system prompt tells the model to auto-confirm commands. (ASI02, ASI09; C2) — [multimodal/agent-tars/core/src/prompt.ts:81](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/prompt.ts#L81)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

Agent TARS runs every tool as the operating-system user who launched it. Shell commands inherit the full process environment (including the model API key and anything loaded from a .env file), and the browser and file tools have no separate or narrower identity. The local web server it starts is reachable only from this machine by default and demands a token when bound to the network, which controls who can drive the agent but does nothing to narrow what the agent itself can reach. A hijacked session therefore holds the user's whole account: SSH keys, cloud credentials, and every logged-in service the shell can reach.

- **S L0:** Tools run with the launching user's ambient authority; run_command passes no env option, so the child inherits the full process environment. — [packages/agent-infra/mcp-servers/commands/src/server.ts:136](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/commands/src/server.ts#L136); [packages/agent-infra/mcp-servers/commands/src/server.ts:143](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/commands/src/server.ts#L143) (verified)
  - *To reach the next level:* No dedicated or narrowed identity; L1 needs at least a dedicated identity for agent tools.
- **C L0:** There is no authorization layer in the tool path; the server token (required only off loopback) authenticates who drives the agent, not what each tool may do. — [multimodal/tarko/agent/src/agent/runner/tool-processor.ts:117](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent/src/agent/runner/tool-processor.ts#L117); [multimodal/tarko/agent-server/src/api/middleware/network-auth.ts:71](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent-server/src/api/middleware/network-auth.ts#L71); searched `rg -n -i 'authoriz|least.?privilege|scope'` in `multimodal/tarko/agent/src multimodal/tarko/mcp-agent/src multimodal/agent-tars/core/src` → 0 hits (verified)
  - *To reach the next level:* No authorization check on any tool path; L1 needs the main tool path checked in code.
- **D L0:** The default install runs with the user's full privileges and loads .env from the working directory into the process environment. — [multimodal/tarko/agent-cli/src/config/loader.ts:65-69](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent-cli/src/config/loader.ts#L65-L69); [multimodal/agent-tars/core/src/environments/local/index.ts:178](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/environments/local/index.ts#L178) (verified)
  - *To reach the next level:* No narrower default exists; L1 needs a default identity narrower than the user's.
- **B L0:** A hijacked shell reaches the user's whole account: home directory credentials, environment API keys, and any network service. — [packages/agent-infra/mcp-servers/commands/src/server.ts:143](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/commands/src/server.ts#L143); [multimodal/agent-tars/core/src/prompt.ts:55](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/prompt.ts#L55) (verified)
  - *To reach the next level:* Authority spans the user's entire account; L1 needs it limited to write access on a few systems.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

There is no human approval step anywhere in the agent loop. Shell commands, scripts, file writes, browser JavaScript, and form submissions all run the moment the model asks for them, and the system prompt even tells the model to add -y or -f flags so commands never pause for confirmation. Nothing provides checkpoints, undo, or dry runs, so a wrong or hijacked action, such as deleting files or submitting a purchase in the browser, cannot be caught or reversed.

- **S L0:** No approval mechanism exists; tool calls go straight from model output to execution. — searched `rg -n -i 'approv|confirm|permission'` in `multimodal/tarko/agent/src multimodal/tarko/mcp-agent/src multimodal/agent-tars/core/src` → 4 hits (hits: a loop-termination log line, the system prompt telling the model to auto-confirm shell commands (prompt.ts:81), and two 'mapProviderString' matches; none is an approval gate); [multimodal/tarko/agent/src/agent/runner/tool-processor.ts:117](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent/src/agent/runner/tool-processor.ts#L117) (verified)
  - *To reach the next level:* No approval primitive; L1 needs any human approval before consequential tools run.
- **C L0:** The most powerful tools (run_command, run_script, browser_evaluate) are registered and executed without any gate. — [multimodal/agent-tars/core/src/environments/local/index.ts:178](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/environments/local/index.ts#L178); [multimodal/agent-tars/core/src/environments/local/browser/browser-control-strategies/browser-hybrid-strategy.ts:52](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/environments/local/browser/browser-control-strategies/browser-hybrid-strategy.ts#L52) (verified)
  - *To reach the next level:* Nothing is gated; L1 needs at least flagged mutating tools to be gated.
- **D L0:** No gate to enable; the system prompt instructs the model to auto-confirm commands. — [multimodal/agent-tars/core/src/prompt.ts:81](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/prompt.ts#L81) (verified)
  - *To reach the next level:* No approval default; L1 needs a gate on by default.
- **B L0:** Irreversible actions (arbitrary shell, browser form submission, file overwrite) run with no checkpoint or undo. — [packages/agent-infra/mcp-servers/commands/src/server.ts:143](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/commands/src/server.ts#L143); [packages/agent-infra/mcp-servers/browser/src/tools/evaluate.ts:17](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/browser/src/tools/evaluate.ts#L17) (verified)
  - *To reach the next level:* No rollback or preview; L1 needs at least some actions to be reversible.
- **Cap:** none

### C3 Tool & action scoping — 0.23 (high)

The file tools check that paths stay inside the working directory, but that check is not a strict boundary. Every other default tool takes raw input: the shell runs any command string from any directory, the browser navigates to any URL (including internal addresses), and browser_evaluate runs arbitrary JavaScript. Shell, write, and network tools are all enabled by default, so the shell alone bypasses whatever the file tools enforce.

- **S L2:** The filesystem server resolves paths and realpaths, but its containment check is not a strict boundary. (verified)
  - *To reach the next level:* The containment check is not a strict boundary; L3 needs resolved-path containment that fails closed.
- **C L1:** Only the filesystem tools validate; run_command takes a raw string and browser_navigate a raw URL. — [packages/agent-infra/mcp-servers/commands/src/server.ts:143](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/commands/src/server.ts#L143); [packages/agent-infra/mcp-servers/browser/src/tools/navigate.ts:18](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/browser/src/tools/navigate.ts#L18); searched `rg -n -i 'timeout|killSignal'` in `packages/agent-infra/mcp-servers/commands/src` → 0 hits (no bound of any kind on the command tools) (verified)
  - *To reach the next level:* Shell and browser tools have no argument validation; L2 needs most built-in tools validated.
- **D L0:** Write, exec, and network tools are all on by default (in-memory MCP with browser, filesystem, and commands servers). — [multimodal/agent-tars/core/src/shared/config-utils.ts:27](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/shared/config-utils.ts#L27); [multimodal/agent-tars/core/src/environments/local/index.ts:178](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/environments/local/index.ts#L178) (verified)
  - *To reach the next level:* Everything enabled by default; L1 needs dangerous tools individually disableable by default config.
- **B L0:** run_command executes any command on the host with no cwd restriction. — [packages/agent-infra/mcp-servers/commands/src/server.ts:143](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/commands/src/server.ts#L143) (verified)
  - *To reach the next level:* General-purpose tools against the whole machine; L1 needs some limit on reach.
- **Cap:** none

### C4 Code-execution isolation — 0.40 (medium)

By default every shell command and script the model writes runs directly on the user's machine as the user, with the full process environment and no timeout. The bundled browser's launch settings are also not locked down. The system prompt tells the model it is in a Linux sandbox, which it is not in the default local mode. An opt-in AIO sandbox mode sends built-in tool execution to a remote endpoint instead, but it must be configured explicitly and any extra MCP servers the user adds still launch locally.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Commands run as a same-user child process via child_process.exec on the host. — [packages/agent-infra/mcp-servers/commands/src/server.ts:143](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/commands/src/server.ts#L143); [packages/agent-infra/mcp-servers/commands/src/exec-utils.ts:44](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/commands/src/exec-utils.ts#L44) (verified)
    - *To reach the next level:* No isolation primitive; L1 needs at least filtering or a separate working directory.
  - **C L0:** No execution path is isolated. — [multimodal/agent-tars/core/src/environments/local/index.ts:178](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/environments/local/index.ts#L178) (verified)
    - *To reach the next level:* Main exec tool unsandboxed; L1 needs the main exec tool sandboxed.
  - **D L0:** Local execution is the default; the sandbox mode exists only when aioSandbox is set. — [multimodal/agent-tars/core/src/agent-tars.ts:73-74](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/agent-tars.ts#L73-L74); [multimodal/agent-tars/core/src/prompt.ts:52](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/prompt.ts#L52) (verified)
    - *To reach the next level:* Isolation is off by default; L1 needs it on by default.
  - **B L0:** Commands reach the full host: home directory, credentials, and environment secrets, with unrestricted network. — [packages/agent-infra/mcp-servers/commands/src/server.ts:136](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/commands/src/server.ts#L136) (verified)
    - *To reach the next level:* Host-equivalent reach; L1 needs at least no credentials in the execution environment.
- **opt-in AIO sandbox (--aio-sandbox <url>)** (alt; raw 0.40, cap G1 → 0.40) ← counted
  - **S L2:** In AIO mode no local tools are created and built-in tools come from the remote endpoint's MCP server; the endpoint's isolation is not in this repo. — [multimodal/agent-tars/core/src/environments/aio/index.ts:61](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/environments/aio/index.ts#L61) (inferred)
    - *To reach the next level:* The sandbox's hardening isn't verifiable here; L3 needs a hardened container or OS sandbox profile shown in code.
  - **C L2:** Built-in tools move to the endpoint, but user-configured MCP servers are still merged in and launched locally. — [multimodal/agent-tars/core/src/environments/aio/index.ts:63](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/environments/aio/index.ts#L63) (verified)
    - *To reach the next level:* User-added stdio MCP servers run on the host; L3 needs every model-reachable path sandboxed.
  - **D L0:** Opt-in: only used when aioSandbox is configured. — [multimodal/agent-tars/core/src/agent-tars.ts:73-74](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/agent-tars.ts#L73-L74) (verified)
    - *To reach the next level:* Off by default; L1 needs it on by default.
  - **B L2:** Execution happens off the host, so local files and credentials are out of reach, but network egress and secrets inside the endpoint are unknown. — [multimodal/agent-tars/core/src/environments/aio/index.ts:57-64](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/environments/aio/index.ts#L57-L64) (inferred)
    - *To reach the next level:* No verified network or secret restriction inside the sandbox; L3 needs workspace-only mounts, no secrets, and restricted egress.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.00 (high)

Agent TARS reads web pages, search results, and files as a matter of course, and nothing separates that content from the user's instructions or limits what the agent can do afterwards. A hijacked session can send data anywhere by navigating the browser or running curl, and can take irreversible actions on the machine or in logged-in websites, all without a human in the loop. This is the worst-case outcome the scorecard describes.

- **S L0:** No provenance tagging, taint tracking, or capability restriction after untrusted content is read. — searched `rg -n -i 'untrusted|taint|provenance'` in `multimodal/tarko/agent/src multimodal/tarko/mcp-agent/src multimodal/agent-tars/core/src` → 0 hits (verified)
  - *To reach the next level:* Nothing limits a hijacked agent; L1 needs at least detection or spotlighting.
- **C L0:** Tool results from browsing and search enter the context with the same standing as other tool output. — [multimodal/tarko/agent/src/agent/runner/tool-processor.ts:323-329](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent/src/agent/runner/tool-processor.ts#L323-L329); [multimodal/agent-tars/core/src/shared/config-utils.ts:15](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/shared/config-utils.ts#L15) (verified)
  - *To reach the next level:* Untrusted sources aren't distinguished; L1 needs one source handled.
- **D L0:** No control to default on. — searched `rg -n -i 'untrusted|taint|provenance'` in `multimodal/tarko/agent/src multimodal/tarko/mcp-agent/src multimodal/agent-tars/core/src` → 0 hits (verified)
  - *To reach the next level:* No mitigation exists; L1 needs one on by default.
- **B L0:** Unattended egress (arbitrary URL navigation, shell network access) and irreversible actions (shell, browser form submission) coexist in every session. — [packages/agent-infra/mcp-servers/browser/src/tools/navigate.ts:18](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/browser/src/tools/navigate.ts#L18); [packages/agent-infra/mcp-servers/commands/src/server.ts:143](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/commands/src/server.ts#L143) (verified)
  - *To reach the next level:* Leak plus irreversible action with no human; L1 needs at least one of those to require approval.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.05 (medium)

Agent TARS treats the directory it is started in as the workspace and, with no trust prompt, loads .env files into the process environment and a .tarko/instructions.md into the system prompt as higher-priority user instructions; project-scoped configuration is not integrity-protected. The agent's own file tools can write these files, so a single injection can persist for future runs in that directory, and in a shared repository for other users too. The scorecard's repository-configuration cap applies.

- **S L0:** Workspace files can inject high-priority instructions with no prompt, and workspace configuration is not integrity-protected. — [multimodal/agent-tars/core/src/agent-tars.ts:262](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/agent-tars.ts#L262) (verified)
  - *To reach the next level:* Repo files control instructions silently; L1 needs at least that security-relevant config can't be set from the repo.
- **C L0:** None of the auto-loaded paths is controlled. — [multimodal/tarko/agent-cli/src/config/loader.ts:65-69](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent-cli/src/config/loader.ts#L65-L69); [multimodal/tarko/agent-cli/src/utils/workspace-config.ts:14](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent-cli/src/utils/workspace-config.ts#L14) (verified)
  - *To reach the next level:* No load path is controlled; L1 needs one controlled.
- **D L1:** Session events are stored per session id in a local SQLite database, but the agent's shell can rewrite that database and the workspace config freely. — [multimodal/tarko/agent-server/src/storage/SQLiteStorageProvider.ts:660](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent-server/src/storage/SQLiteStorageProvider.ts#L660); [multimodal/tarko/agent-cli/src/core/cli.ts:369](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent-cli/src/core/cli.ts#L369) (verified)
  - *To reach the next level:* Nothing stops the model writing outside its session's records; L2 needs namespaces the model cannot bypass.
- **B L0:** An instruction or configuration file written by the agent persists across all future sessions in that directory and reaches other users if the repository is shared. — [multimodal/tarko/agent-cli/src/utils/workspace-config.ts:30](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent-cli/src/utils/workspace-config.ts#L30) (inferred)
  - *To reach the next level:* Poisoned config persists across sessions and users; L1 needs it confined to one user's sessions.
- **Cap:** C6-REPOCONFIG — Workspace configuration is not integrity-protected.

### C7 Third-party extensions — 0.00 (high)

The system prompt tells the model to install whatever software packages it needs through the shell on its own, which means downloading and running package install scripts with the user's full environment and no consent. Extra MCP servers come from configuration, are typically launched with unpinned npx -y commands, and nothing verifies their versions or hashes; workspace configuration affecting them is not integrity-protected. Configured MCP servers do get a reduced environment, but packages installed through the shell get everything.

- **S L0:** Model-chosen package installs through the ungated shell are encouraged by the system prompt; no pinning or integrity checks exist. — [multimodal/agent-tars/core/src/prompt.ts:55](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/prompt.ts#L55); searched `rg -n -i 'sha256|integrity|signature|checksum'` in `multimodal/tarko/mcp-agent/src packages/agent-infra/mcp-client/src` → 0 hits (verified)
  - *To reach the next level:* Remote code runs unverified at the model's choice; L1 needs only user-chosen sources.
- **C L0:** No extension type is verified; even the built-in stdio mode launches servers with npx -y unpinned. — [multimodal/agent-tars/core/src/environments/local/index.ts:474](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/environments/local/index.ts#L474) (verified)
  - *To reach the next level:* No type verified; L1 needs one type verified.
- **D L0:** The model installs packages without asking, and workspace configuration is not integrity-protected. — [multimodal/agent-tars/core/src/environments/aio/index.ts:63](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/core/src/environments/aio/index.ts#L63) (verified)
  - *To reach the next level:* L1 needs at least a consent prompt.
- **B L0:** Packages installed via the shell run as the user with the full process environment; configured stdio MCP servers get PATH plus configured env but still run as the same user. — [packages/agent-infra/mcp-servers/commands/src/server.ts:136](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/commands/src/server.ts#L136); [packages/agent-infra/mcp-client/src/index.ts:266-268](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-client/src/index.ts#L266-L268) (verified)
  - *To reach the next level:* Shell-installed code gets the full environment; L1 needs at least a separate process for extensions with a scrubbed environment on every path.
- **Cap:** C7-RCELOAD — By default the model is instructed to install packages via the ungated shell (prompt.ts:55, server.ts:143), executing remote install scripts without user consent (verified condition; whether a given run installs is model-dependent).

### C8 Secrets & sensitive-data protection — 0.20 (high)

API keys come from command-line flags, config files, or a .env file in the working directory, and are loaded into the process environment that every shell command inherits, so the model can print them with a single command. The only redaction is masking the API key when the web UI shows the configuration. Session transcripts, including all tool output, are stored unencrypted in a local SQLite database, and the command server always logs command output to the console. There is no telemetry by default.

- **S L1:** Secrets come from env/flags; masking exists only for the API key in the UI config view. — [multimodal/tarko/agent-server/src/utils/config-sanitizer.ts:30](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent-server/src/utils/config-sanitizer.ts#L30); searched `rg -n -i 'redact|mask|secret'` in `multimodal/tarko/agent/src multimodal/agent-tars/core/src packages/agent-infra/mcp-servers/commands/src` → 0 hits (verified)
  - *To reach the next level:* No log or tool-output redaction; L2 needs type-level masking and log filters on main paths.
- **C L1:** One path (UI config display) is protected; subprocess env, logs, transcripts, and model-bound tool output are not. — [packages/agent-infra/mcp-servers/commands/src/server.ts:136](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/commands/src/server.ts#L136); [packages/agent-infra/mcp-servers/commands/src/server.ts:144](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/commands/src/server.ts#L144) (verified)
  - *To reach the next level:* Only one path protected; L2 needs logs and transcripts covered.
- **D L1:** Telemetry (AGIO) is opt-in, but command output is always logged, --debug logs full tool arguments, and nothing is redacted. — [multimodal/tarko/agent-server/src/core/AgentSession.ts:237](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent-server/src/core/AgentSession.ts#L237); [multimodal/tarko/agent/src/agent/runner/tool-processor.ts:115](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent/src/agent/runner/tool-processor.ts#L115) (verified)
  - *To reach the next level:* No redaction to keep on; L2 needs reasonable logging defaults with redaction available.
- **B L0:** Long-lived model and service keys in the environment are reachable by every subprocess the model starts. — [packages/agent-infra/mcp-servers/commands/src/server.ts:136](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/commands/src/server.ts#L136); [multimodal/tarko/agent-cli/src/config/loader.ts:69](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent-cli/src/config/loader.ts#L69) (verified)
  - *To reach the next level:* Long-lived keys reachable by the model; L1 needs keys at least kept out of model-reachable subprocesses.
- **Cap:** none

### C9 Audit & traceability — 0.45 (high)

Every tool call, including calls to user-added MCP servers, produces a structured event with the tool name, full arguments, result, and timing, and the CLI stores these events in a SQLite database under the user's home directory by default. The record carries no notion of who requested or approved an action (there is no approval step to record), and the agent's own shell can edit or delete the database. Storage failures are printed to the console and the agent carries on, so records can be lost silently.

- **S L2:** Structured tool_call and tool_result events with arguments and timestamps are emitted for every call. — [multimodal/tarko/agent/src/agent/runner/tool-processor.ts:269-273](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent/src/agent/runner/tool-processor.ts#L269-L273) (verified)
  - *To reach the next level:* No actor attribution or approver fields; L3 needs principal and approval attribution.
- **C L2:** All tools, built-in and MCP, pass through the same ToolProcessor that emits events; there are no approvals or denials to record. — [multimodal/tarko/agent/src/agent/runner/tool-processor.ts:117](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent/src/agent/runner/tool-processor.ts#L117); [multimodal/tarko/agent-server/src/core/AgentSession.ts:118](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent-server/src/core/AgentSession.ts#L118) (verified)
  - *To reach the next level:* No approval/denial records and no config-change records; L3 needs approvals and denials logged too.
- **D L2:** SQLite storage under ~/.agent-tars is on by default, outside the workspace but writable by the agent's shell. — [multimodal/agent-tars/cli/src/index.ts:42-43](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/agent-tars/cli/src/index.ts#L42-L43) (verified)
  - *To reach the next level:* The agent process can alter its own record; L3 needs a writer the model can't control.
- **B L1:** Event saves are asynchronous and best-effort; failures, including storage initialization failure, are only printed. — [multimodal/tarko/agent-server/src/core/AgentSession.ts:120](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent-server/src/core/AgentSession.ts#L120); [multimodal/tarko/agent-server/src/server.ts:267](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent-server/src/server.ts#L267) (verified)
  - *To reach the next level:* Failures don't stop actions; L2 needs errors surfaced and per-action flush.
- **Cap:** none

### C10 Limits & kill switch — 0.25 (high)

The only hard limit is an iteration cap of 1,000 model turns. There is no session time limit, no token or cost budget, and no timeout on shell commands. Stopping a run sets an abort flag that is checked between tool calls, but a command already running keeps going because nothing kills its process.

- **S L1:** Iteration cap only; abort is cooperative and shell commands have no timeout. — [multimodal/tarko/agent/src/agent/agent.ts:90](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent/src/agent/agent.ts#L90); searched `rg -n -i 'timeout|killSignal'` in `packages/agent-infra/mcp-servers/commands/src` → 0 hits (verified)
  - *To reach the next level:* No wall-clock, per-execution timeout, or cost cap; L2 needs at least one enforced in code.
- **C L1:** The cap and abort check cover the top-level loop only. — [multimodal/tarko/agent/src/agent/runner/loop-executor.ts:63](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent/src/agent/runner/loop-executor.ts#L63); [multimodal/tarko/agent/src/agent/runner/tool-processor.ts:249](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent/src/agent/runner/tool-processor.ts#L249) (verified)
  - *To reach the next level:* Tool executions aren't bounded; L2 needs tool timeouts.
- **D L1:** The default cap is very large (1,000 iterations) and no cost limit exists. — [multimodal/tarko/agent/src/agent/agent.ts:90](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent/src/agent/agent.ts#L90) (verified)
  - *To reach the next level:* Default ceiling is very large; L2 needs sensible defaults.
- **B L1:** Stopping aborts the controller but leaves in-flight shell children running. — [multimodal/tarko/agent/src/agent/execution-controller.ts:96](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/multimodal/tarko/agent/src/agent/execution-controller.ts#L96); [packages/agent-infra/mcp-servers/commands/src/server.ts:143](https://github.com/bytedance/UI-TARS-desktop/blob/2ff41a9e515828c5bd5b276e493d73aa0bdf4a3a/packages/agent-infra/mcp-servers/commands/src/server.ts#L143) (verified)
  - *To reach the next level:* In-flight work survives a stop; L2 needs moderate ceilings with the loop ending on stop.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: web pages and search results via browser tools (navigate.ts:18, config-utils.ts:15) · [B] sensitive data/systems: user's home directory and environment secrets via run_command (commands/src/server.ts:136,143) · [C] state change / egress: arbitrary shell and URL navigation (commands/src/server.ts:143, navigate.ts:18) · Same default session? Yes

## Highest-impact improvements
1. Add a per-call approval prompt showing the exact command/script/JS for run_command, run_script, browser_evaluate, and file writes. — C2 S L0→L3, +0.225 before caps (Playbook 5, step 1)
2. Require an explicit workspace-trust decision before loading workspace configuration. — C6 S L0→L3, +0.225 before caps (Playbook 2, step 1)
3. Make the AIO (or a local container) sandbox the default for shell execution, with an explicit flag for host mode. — C4 D L0→L3, +0.150 before caps (Playbook 3, step 1)
4. Harden validatePath containment. — C3 S L2→L3, +0.075 before caps (Playbook 3, step 1)
5. Add a timeout to exec and kill the child process group on abort. — C10 S L1→L2, +0.075 before caps (Playbook 3, step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of commit 2ff41a9 only; nothing was executed, installed, or probed.
- Scored the Agent TARS CLI (the project the README leads with). The UI-TARS desktop Electron app (apps/ui-tars) was not scored; a spot check found no approval gate there either and a 100-iteration default loop cap (src/main/store/setting.ts:27).
- The AIO sandbox image and its isolation live outside this repository and were not examined; its alt rating is inferred.
- Behaviour of third-party libraries (MCP SDK stdio environment merging and request timeouts, Puppeteer) is inferred from their documented behaviour, not read.
- Packages under multimodal/omni-tars, multimodal/gui-agent, and agent-server-next were not examined.
- Model behaviour (refusals, deception) is out of scope; only code-level controls are scored.
- No reviewer-directed prompt injection found in the repo.
