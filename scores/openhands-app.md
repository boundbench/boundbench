# Defense-in-Depth Score: OpenHands (Agent Canvas)

**Repo:** https://github.com/openhands/openhands · **Commit:** `a6bba78ffd5a8b31620770f52383b1a2c0477fcd` (1.24.0) · **Reviewed:** 2026-10-04
**What it is:** OpenHands Agent Canvas: self-hosted control center and local-stack launcher for the OpenHands coding agent and ACP agents.
**Category:** Coding
**Scored configuration:** `npm install -g @openhands/agent-canvas && agent-canvas` with no flags (README Option 1): agent server on the host via uvx, default settings, local backend.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents yes · external communication yes

## Score: 1.8 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | C1-SELFESC | **0.00** | High |
| C2 | Approval gates | L1 | L1 | L0 | L1 | 0.20 | C2-SELFAPPROVE | **0.20** | High |
| C3 | Tool & action scoping | L0 | L1 | L0 | L0 | 0.07 | — | **0.07** | High |
| C4 | Code-execution isolation | L2 | L1 | L0 | L1 | 0.28 | G1 | **0.28** (alt) | Low |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | C6-REPOCONFIG | **0.10** | Medium |
| C7 | Third-party extensions | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | Medium |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C9 | Audit & traceability | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | Low |
| C10 | Limits & kill switch | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |


As shipped, Agent Canvas runs the OpenHands agent directly on your machine as you, with shell, file and browser tools, and no approval step (confirmation mode is off). The launcher passes the agent server your whole shell environment, and its handling of backend credentials is not locked down. A cloned repository's .openhands/hooks.json is attached to new conversations without a trust prompt. The opt-in Docker runtime and confirmation mode help, but the dominant risk is unattended, host-level execution after a prompt injection.

## Critical gaps
- By default all agent code runs on the host as the user with the launcher's full environment; there is no sandbox. (ASI05; C4) — [src/api/agent-server-adapter.ts:1155-1157](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1155-L1157); [scripts/dev-with-automation.mjs:670](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/scripts/dev-with-automation.mjs#L670); [README.md:66](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/README.md#L66)
- With confirmation off and host execution, a prompt injection can exfiltrate data and take irreversible actions unattended. (ASI01; C5) — [src/api/agent-server-adapter.ts:741-742](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L741-L742); [src/api/agent-server-adapter.ts:146](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L146)
- Workspace .openhands/hooks.json is loaded and attached to new conversations with no trust decision. (ASI06; C6) — [src/api/hooks-service.ts:16-18](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/hooks-service.ts#L16-L18); [src/api/agent-server-adapter.ts:1430-1431](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1430-L1431)
- MCP, ACP and plugin processes run under the unsandboxed agent server with its full environment. (ASI04; C7) — [scripts/dev-with-automation.mjs:670](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/scripts/dev-with-automation.mjs#L670); [src/api/agent-server-adapter.ts:1155-1157](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1155-L1157)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

The default launcher starts the agent server directly on the user's machine as the logged-in user, handing it the launcher's entire environment, so any cloud, GitHub or other credentials in the shell are within the agent's reach. On top of that, the launcher's handling of backend credentials is not locked down, so the agent can widen its own permissions. There is no per-tool identity or authorization layer in this repository.

- **S L0:** The agent server runs as the OS user with the launcher's full environment. — [scripts/dev-with-automation.mjs:670](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/scripts/dev-with-automation.mjs#L670) (verified)
  - *To reach the next level:* Give the agent a dedicated, narrowly scoped identity.
- **C L0:** No authorization layer sits between the agent's tools and the user's ambient credentials; every subprocess inherits the full environment. — [scripts/dev-with-automation.mjs:670](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/scripts/dev-with-automation.mjs#L670); [src/api/agent-server-adapter.ts:1155-1157](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1155-L1157) (verified)
  - *To reach the next level:* Route tool actions through an authorization check and scrub subprocess environments.
- **D L0:** The README-led default runs the agent server on the host with full filesystem access and no narrower role. — [README.md:66](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/README.md#L66); [src/api/agent-server-adapter.ts:1155-1157](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1155-L1157) (verified)
  - *To reach the next level:* Ship a default that runs with near-minimal authority and requires explicit elevation.
- **B L0:** A hijacked agent holds the user's full account authority on the host (plus additional backend reach). — [src/api/agent-server-adapter.ts:1279-1283](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1279-L1283) (verified)
  - *To reach the next level:* Limit the agent to one project's narrowly scoped, short-lived credentials.
- **Cap:** C1-SELFESC — An agent-reachable path lets the agent change its own approval policy and permissions.

### C2 Approval gates — 0.20 (high)

Confirmation mode is off by default, so the agent's shell, file editor and browser run every action with no human approval. When a user turns it on, the default pairing is a 'confirm risky' policy where an LLM security analyzer decides which actions need a human, so most actions are approved by a model. The approval path is also not a complete boundary. Model-requested child conversations are launched by the browser without any human prompt, and the planning conversation is hard-coded to never confirm.

- **S L1:** When enabled, the default analyzer setting maps to ConfirmRisky with an LLM security analyzer, so a model decides which actions reach a human. — [src/api/agent-server-adapter.ts:745-746](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L745-L746); [src/services/settings.ts:13-14](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/services/settings.ts#L13-L14) (verified)
  - *To reach the next level:* Require per-call human approval showing the exact command with deterministic risk tiers.
- **C L1:** Client-side child-conversation launches execute on receipt of the action event with no gate, and planning conversations are hard-coded NeverConfirm. — [src/contexts/conversation-websocket-context.tsx:846-853](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/contexts/conversation-websocket-context.tsx#L846-L853); [src/services/child-conversation-launch.ts:543-547](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/services/child-conversation-launch.ts#L543-L547); [src/api/agent-server-adapter.ts:1550](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1550) (verified)
  - *To reach the next level:* Route every tool path, including client tools and child conversations, through the same gate.
- **D L0:** confirmation_mode defaults to false, which maps to a NeverConfirm policy. — [src/services/settings.ts:13-14](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/services/settings.ts#L13-L14); [src/api/agent-server-adapter.ts:741-742](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L741-L742) (verified)
  - *To reach the next level:* Turn approval on by default for consequential actions.
- **B L1:** Unapproved actions run on the host with network access; only code changes inside the default git worktree are easily reversible. — [src/api/agent-server-adapter.ts:1155-1157](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1155-L1157); [src/api/agent-server-adapter.ts:1373-1376](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1373-L1376) (verified)
  - *To reach the next level:* Add checkpoints/rollback for filesystem state and previews for external actions.
- **Cap:** C2-SELFAPPROVE — The approval path is not a complete boundary, and the agent can get around it.

### C3 Tool & action scoping — 0.07 (high)

Every conversation gets a general-purpose terminal, a file editor and, unless an environment variable disables it, a browser, plus a client tool that launches more agent conversations. The terminal accepts arbitrary shell commands and nothing in this repository validates or bounds their arguments. The only argument checks found are schema checks on the child-conversation launch tool. Write, exec and network tools are all on by default.

- **S L0:** The default tool set includes a raw shell terminal with no argument allowlist in this repository. — [src/api/agent-server-adapter.ts:146](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L146) (verified)
  - *To reach the next level:* Replace general tools with narrow ones or validate arguments against allowlists in code.
- **C L1:** Only the child-conversation client tool validates its parameters (target enum, non-empty task). — [src/services/child-conversation-launch.ts:116-130](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/services/child-conversation-launch.ts#L116-L130) (verified)
  - *To reach the next level:* Apply a shared validation layer to all built-in and extension tools.
- **D L0:** Terminal, file editor, task tracker and browser are all enabled by default. — [src/api/agent-server-adapter.ts:146](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L146); [src/api/agent-server-adapter.ts:157-159](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L157-L159) (verified)
  - *To reach the next level:* Make dangerous tools individually disableable and off unless needed.
- **B L0:** A misused terminal runs any command against the whole host in the default local runtime. — [src/api/agent-server-adapter.ts:1155-1157](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1155-L1157); [README.md:66](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/README.md#L66) (verified)
  - *To reach the next level:* Scope tools to the project workspace with bounds.
- **Cap:** none

### C4 Code-execution isolation — 0.28 (low)

In the default npm launch the agent server, and therefore every shell command and script the agent runs, executes directly on the host as the user; the README itself warns the agent has full filesystem access. A per-conversation Docker runtime exists but must be enabled with OH_CONVERSATION_RUNTIME=docker, and its container hardening lives in the external agent-server package, so it cannot be verified here. Even then, the workspace is mounted read-write and the automation service stays on the host.

- **default configuration** (default; raw 0.00, cap G1 → 0.00)
  - **S L0:** Conversations use LocalWorkspace on the host by default; there is no isolation primitive. — [src/api/agent-server-adapter.ts:1155-1157](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1155-L1157); [README.md:66](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/README.md#L66) (verified)
    - *To reach the next level:* Run model-influenced code in an OS-level or stronger sandbox.
  - **C L0:** No execution path is sandboxed by default. — [src/api/agent-server-adapter.ts:1155-1157](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1155-L1157) (verified)
    - *To reach the next level:* Sandbox the main exec tool and the paths it spawns.
  - **D L0:** Sandboxing is opt-in via the OH_CONVERSATION_RUNTIME env var, which is only forwarded when set. — [scripts/dev-safe.mjs:791-799](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/scripts/dev-safe.mjs#L791-L799); [README.md:66](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/README.md#L66) (verified)
    - *To reach the next level:* Enable a sandbox by default.
  - **B L0:** Code runs as the user on the host with the launcher's full environment, which is host-equivalent. — [scripts/dev-with-automation.mjs:670](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/scripts/dev-with-automation.mjs#L670); [src/api/agent-server-adapter.ts:1155-1157](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1155-L1157) (verified)
    - *To reach the next level:* Confine execution to a workspace-only mount with no secrets and restricted egress.
- **opt-in per-conversation Docker runtime (OH_CONVERSATION_RUNTIME=docker)** (alt; raw 0.28, cap G1 → 0.28) ← counted
  - **S L2:** Conversations run in a DockerExecutionWorkspace container; its hardening is defined in the external agent-server package and not visible here. — [src/api/agent-server-adapter.ts:1150-1154](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1150-L1154); [scripts/dev-safe.mjs:791-799](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/scripts/dev-safe.mjs#L791-L799) (inferred)
    - *To reach the next level:* Show a hardened container profile (non-root, dropped capabilities, seccomp, no-new-privileges) in code.
  - **C L1:** The conversation's tools run in the container, but Canvas, the outer agent server and the automation service stay on the host. — [README.md:122](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/README.md#L122) (inferred)
    - *To reach the next level:* Sandbox every model-reachable path, including automations and extension processes.
  - **D L0:** Off by default; enabled only by an env var. — [scripts/dev-safe.mjs:791-799](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/scripts/dev-safe.mjs#L791-L799) (verified)
    - *To reach the next level:* Enable by default.
  - **B L1:** Per the README (no code here corroborates it), the conversation's workspace and persisted state are mounted into the container; nothing in this repository restricts network egress. — [README.md:120](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/README.md#L120) (inferred)
    - *To reach the next level:* Restrict egress and keep secrets out of the container environment.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.00 (high)

Nothing in this repository limits what a hijacked agent can do after it reads hostile content from a web page, file, issue or tool result. There is no untrusted-content tagging, and the only classifier (the LLM security analyzer) applies only when confirmation mode is on, which it is not by default. In the default configuration a successful prompt injection can make the agent read the user's files and credentials and send them anywhere through the shell or browser, or run destructive commands, with no human in the loop.

- **S L0:** No structural limit on a hijacked agent exists; the optional LLM analyzer is only consulted under confirmation mode. — [src/api/agent-server-adapter.ts:741-742](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L741-L742); searched `rg -n -i 'untrusted|prompt.injection|taint'` in `src/api` → 1 hits (single hit is a comment about system-prompt injection of skills, not an untrusted-content control) (verified)
  - *To reach the next level:* Disable egress and state-changing tools, or force approval, once untrusted content enters the session.
- **C L0:** Tool results, web pages and files are not distinguished from user instructions in this repository. — searched `rg -n -i 'untrusted|prompt.injection|taint'` in `src/api` → 1 hits (single hit is a comment about system-prompt injection of skills, not an untrusted-content control) (verified)
  - *To reach the next level:* Mark every untrusted source and apply the limit to all of them.
- **D L0:** With confirmation_mode false there is no control on by default. — [src/services/settings.ts:13-14](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/services/settings.ts#L13-L14); [src/api/agent-server-adapter.ts:741-742](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L741-L742) (verified)
  - *To reach the next level:* Turn on an untrusted-content control by default.
- **B L0:** A hijacked agent can read host files and credentials and exfiltrate them through the terminal or browser, and take irreversible actions, unattended. — [src/api/agent-server-adapter.ts:146](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L146); [src/api/agent-server-adapter.ts:157-159](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L157-L159); [src/api/agent-server-adapter.ts:1155-1157](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1155-L1157) (verified)
  - *To reach the next level:* Require human approval for both exfiltration paths and irreversible actions.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.10 (medium)

When a conversation starts, Canvas asks the agent server for the workspace's .openhands/hooks.json and attaches whatever hook configuration it finds, with no trust prompt. Hooks run commands, so a cloned repository can make the agent execute code just by being opened. Project skills and instruction files are also auto-loaded from the repository by default. Persistent memory is opt-in and stored under the workspace's .openhands/memory folder, where the agent and the repository itself can write.

- **S L0:** Repository-controlled .openhands/hooks.json is loaded and attached to the conversation without any user decision. — [src/api/hooks-service.ts:16-18](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/hooks-service.ts#L16-L18); [src/api/hooks-service.ts:33-35](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/hooks-service.ts#L33-L35); [src/api/agent-server-adapter.ts:1430-1431](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1430-L1431) (verified)
  - *To reach the next level:* Require an explicit workspace-trust decision before loading repo hooks, and show what will run.
- **C L0:** Neither hooks nor project skills/instruction files pass through any control. — [src/api/agent-server-adapter.ts:1430-1431](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1430-L1431); [src/api/agent-server-adapter.ts:947-949](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L947-L949) (verified)
  - *To reach the next level:* Control every auto-loaded file and memory store.
- **D L1:** Single-user local install; memory lives per workspace under .openhands/memory, which nothing here isolates beyond the directory layout. — [src/mocks/settings-handlers.ts:372-373](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/mocks/settings-handlers.ts#L372-L373) (inferred)
  - *To reach the next level:* Namespace and enforce memory isolation in code.
- **B L1:** Poisoned repo hooks and skills re-fire in every conversation started on that workspace and can trigger tool use. — [src/api/agent-server-adapter.ts:1430-1431](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1430-L1431); [src/api/agent-server-adapter.ts:947-949](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L947-L949) (inferred)
  - *To reach the next level:* Scope persistence to reviewed content with easy purge/rollback.
- **Cap:** C6-REPOCONFIG — Workspace .openhands/hooks.json is loaded and attached as hook_config with no user trust decision (hooks-service.ts, agent-server-adapter.ts); hook execution itself happens in the external agent server, which runs configured hook commands.

### C7 Third-party extensions — 0.25 (medium)

Third-party code comes in through user-configured MCP servers, ACP agent commands (Claude Code, Codex, etc.) and plugins from git sources. None is enabled by default, and the plugin launch dialog makes the user tick a trust box that names the sources. But versions aren't pinned or hash-checked: the MCP form suggests 'npx -y <package-name>' and plugin refs are optional. Those processes run on the host as the user with the agent server's full environment. Public skills come from an exact-pinned npm package bundled at build time.

- **S L1:** Sources are user-chosen but unpinned; the UI suggests npx -y launch commands and plugin refs are optional. — [src/routes/agent-settings.tsx:64](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/routes/agent-settings.tsx#L64); [src/components/features/launch/plugin-launch-modal.tsx:198-218](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/components/features/launch/plugin-launch-modal.tsx#L198-L218) (verified)
  - *To reach the next level:* Pin extension versions and verify integrity.
- **C L1:** Only bundled public skills are effectively pinned (exact npm version); MCP, ACP and plugin sources aren't verified. — [package.json:26](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/package.json#L26) (verified)
  - *To reach the next level:* Verify all extension types.
- **D L2:** MCP config is empty by default and plugin launches need an explicit trust checkbox that lists sources. — [src/services/settings.ts:28](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/services/settings.ts#L28); [src/components/features/launch/plugin-launch-modal.tsx:198-218](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/components/features/launch/plugin-launch-modal.tsx#L198-L218) (verified)
  - *To reach the next level:* Show the exact package, command and permissions for every extension and keep workspace from adding them.
- **B L0:** MCP/ACP/plugin processes are spawned by the unsandboxed agent server, which itself holds the launcher's full environment. — [scripts/dev-with-automation.mjs:670](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/scripts/dev-with-automation.mjs#L670); [src/api/agent-server-adapter.ts:1155-1157](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1155-L1157) (inferred)
  - *To reach the next level:* Run each extension in its own sandbox with a scrubbed environment.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.20 (high)

Saved secrets are encrypted by the agent server with a key that the launcher stores in a 0600 file next to them, and they reach conversations as lookups rather than inline values. But the launcher passes its entire shell environment to the agent server, and its handling of backend credentials is not locked down. A content-free install ping goes to PostHog regardless of consent, and the agent server gets a PostHog key by default unless DO_NOT_TRACK is set.

- **S L1:** Secrets are encrypted at rest with a co-located 0600 key file, but backend credential handling is not locked down. — [scripts/dev-safe.mjs:830](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/scripts/dev-safe.mjs#L830); [scripts/dev-safe.mjs:168](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/scripts/dev-safe.mjs#L168) (verified)
  - *To reach the next level:* Keep high-privilege keys out of model-reachable environments and redact before logs and model-bound messages.
- **C L1:** Settings secrets use LookupSecret indirection, but subprocess environments are not scrubbed by the launcher. — [src/api/agent-server-adapter.ts:1279-1283](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1279-L1283); [scripts/dev-with-automation.mjs:670](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/scripts/dev-with-automation.mjs#L670) (verified)
  - *To reach the next level:* Protect subprocess environments, logs and telemetry paths too.
- **D L1:** An install event is sent regardless of consent, and agent-server telemetry gets a PostHog key unless DO_NOT_TRACK is set. — [src/services/telemetry.ts:9](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/services/telemetry.ts#L9); [src/services/telemetry.ts:760](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/services/telemetry.ts#L760); [scripts/dev-safe.mjs:742-744](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/scripts/dev-safe.mjs#L742-L744) (verified)
  - *To reach the next level:* Make all telemetry opt-in.
- **B L0:** The launcher's whole environment is reachable by the model and every subprocess. — [scripts/dev-with-automation.mjs:670](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/scripts/dev-with-automation.mjs#L670) (verified)
  - *To reach the next level:* Use scoped, short-lived credentials not exposed to the agent.
- **Cap:** none

### C9 Audit & traceability — 0.40 (low)

The launcher points the agent server at a conversations directory under ~/.openhands, where (in the external agent server) events are persisted, and asks for structured JSON logs. Nothing in this repository adds actor attribution, approver records or tamper evidence. Because the default runtime is unsandboxed, the agent's own terminal can edit or delete those records.

- **S L2:** Conversation events are persisted by the agent server to OH_CONVERSATIONS_PATH and logs are JSON; content is defined in the external package. — [scripts/dev-safe.mjs:820](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/scripts/dev-safe.mjs#L820); [scripts/dev-with-automation.mjs:971](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/scripts/dev-with-automation.mjs#L971) (inferred)
  - *To reach the next level:* Attribute actions to agent vs human approver with correlation IDs across child conversations.
- **C L2:** Agent-server tool events are covered; client-tool child launches are handled in the browser and only reported back as a message. — [src/contexts/conversation-websocket-context.tsx:846-853](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/contexts/conversation-websocket-context.tsx#L846-L853) (inferred)
  - *To reach the next level:* Record every tool call including client tools, approvals and denials.
- **D L1:** Records are on by default under ~/.openhands, which the unsandboxed agent can write. — [scripts/dev-safe.mjs:643-649](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/scripts/dev-safe.mjs#L643-L649); [src/api/agent-server-adapter.ts:1155-1157](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1155-L1157) (verified)
  - *To reach the next level:* Write the record from a component the model cannot alter.
- **B L1:** Nothing in this repository makes actions depend on the record being written. — [scripts/dev-safe.mjs:820](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/scripts/dev-safe.mjs#L820) (inferred)
  - *To reach the next level:* Flush records durably per action.
- **Cap:** none

### C10 Limits & kill switch — 0.25 (high)

Each conversation gets a 500-iteration cap and stuck detection, and the stop button interrupts the conversation on the agent server. There is no default cost budget and no wall-clock limit set by Canvas. The model can start any number of child conversations, each with a fresh 500-iteration budget, so the cap does not bound the whole run. Stopping one conversation does not stop its children, and it is not shown to kill background terminal processes.

- **S L1:** Iteration cap (500) only; max_budget_per_task defaults to null. — [src/api/agent-server-adapter.ts:151](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L151); [src/api/agent-server-adapter.ts:1365-1368](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1365-L1368); [src/services/settings.ts:29](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/services/settings.ts#L29) (verified)
  - *To reach the next level:* Add a wall-clock and cost cap enforced in code.
- **C L1:** The cap applies per conversation; model-launched child conversations get their own budget. — [src/services/child-conversation-launch.ts:543-547](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/services/child-conversation-launch.ts#L543-L547); [src/api/agent-server-adapter.ts:151](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L151) (verified)
  - *To reach the next level:* Count child conversations and spawned processes against the parent's budget.
- **D L1:** The default is a large 500 iterations and the model can multiply it by delegating. — [src/api/agent-server-adapter.ts:151](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L151); [src/api/agent-server-adapter.ts:1361-1364](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/api/agent-server-adapter.ts#L1361-L1364) (verified)
  - *To reach the next level:* Stop the model from resetting its limits by delegating.
- **B L1:** Stop interrupts one conversation; children and background processes are not shown to be stopped, and spend is unbounded. — [src/hooks/mutation/conversation-mutation-utils.ts:55-60](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/hooks/mutation/conversation-mutation-utils.ts#L55-L60); [src/services/settings.ts:29](https://github.com/openhands/openhands/blob/a6bba78ffd5a8b31620770f52383b1a2c0477fcd/src/services/settings.ts#L29) (verified)
  - *To reach the next level:* Cancel pending calls and children on stop, and add tight time and cost ceilings.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: browser tool and workspace files (src/api/agent-server-adapter.ts:146-159) · [B] sensitive data/systems: full launcher env in agent env (scripts/dev-with-automation.mjs:670) · [C] state change / egress: host terminal with NeverConfirm (src/api/agent-server-adapter.ts:741-742) · Same default session? Yes

## Highest-impact improvements
1. Lock down backend credential handling for automations. — C2 C L1→L2, +0.075 before caps (Playbook 4)
2. Turn confirmation mode on by default with AlwaysConfirm for terminal/browser actions. — C2 D L0→L3, +0.150 before caps (Playbook 5)
3. Require an explicit workspace-trust prompt showing hook commands before attaching .openhands/hooks.json. — C6 S L0→L3, +0.225 before caps (Playbook 2)
4. Make the Docker conversation runtime the default when Docker is available, with a loud flag for host mode. — C4 D L0→L3, +0.150 before caps (Playbook 3 step 1)
5. Pass an allowlisted environment to the agent server instead of the full process.env. — C8 C L1→L2, +0.075 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- At this commit the openhands/openhands repository contains the Agent Canvas frontend and local-stack launcher; the agent loop, tools, confirmation enforcement, secret masking, hook execution and Docker sandbox live in OpenHands/software-agent-sdk (pinned openhands-agent-server==1.50.1), which was not examined. Behaviour attributed to it is marked inferred.
- Docker image (Option 2), Helm chart and Electron desktop modes were only skimmed and are not the scored configuration.
- No text aimed at AI reviewers was found in the repository.
