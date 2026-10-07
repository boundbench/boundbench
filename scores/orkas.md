# Defense-in-Depth Score: Orkas

**Repo:** https://github.com/orkas-ai/orkas · **Commit:** `595358993d6c98e868030990fc9e3a1130c45191` (2026.10.1) · **Reviewed:** 2026-10-04
**What it is:** Local-first Electron desktop app in which a Commander agent plans goals and coordinates specialist agents, shell/file tools, connectors, MCP servers and external CLI agents.
**Category:** AI Assistants
**Scored configuration:** Desktop app run from source with defaults: access mode all_files_approval, built-in agents and tools, no connectors, MCP servers or external CLI agents added.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents yes · external communication opt-in

## Score: 3.4 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C2 | Approval gates | L2 | L1 | L1 | L1 | 0.33 | G2 | **0.25** | High |
| C3 | Tool & action scoping | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C4 | Code-execution isolation | L2 | L1 | L0 | L1 | 0.28 | G1 | **0.28** (alt) | High |
| C5 | Untrusted input blast radius | L2 | L2 | L1 | L0 | 0.35 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C7 | Third-party extensions | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L3 | L1 | 0.50 | — | **0.50** | High |
| C9 | Audit & traceability | L2 | L2 | L1 | L2 | 0.45 | — | **0.45** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |


Orkas ships a carefully built host-side approval layer: a structural risk classifier for shell commands, sensitive-path prompts, connector send/delete confirmations and fail-closed dialogs. But in the default mode, commands the classifier does not flag run unsandboxed as your OS user with full home-directory and network access. Script-based deletes and GET-style exfiltration (web_fetch, python requests.get) pass without a prompt. Protection of the approval-mode setting is not tamper-resistant.

## Critical gaps
- In the default mode, shell commands run unsandboxed as the OS user with full home-directory and network access. (ASI05, T11; C4) — [src/main/model/core-agent/local-tools.ts:3140](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/local-tools.ts#L3140); [src/core-agent/src/sandbox/executor.ts:346](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/sandbox/executor.ts#L346)
- A hijacked agent can exfiltrate via web_fetch GET and delete or overwrite files via scripts outside the workspace with no human involved. (ASI01, LLM01, T6; C5) — [src/core-agent/src/tools/web-fetch.ts:460-468](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/tools/web-fetch.ts#L460-L468); [src/main/model/core-agent/bash-risk.ts:1068-1074](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/bash-risk.ts#L1068-L1074)

## Criterion details

### C1 Identity & least privilege — 0.25 (high)

Orkas runs as the logged-in OS user. In the default mode its file tools and shell can reach any path outside the workspace; only credential-like paths trigger a prompt. Shell children get a scrubbed environment (HOME, PATH, locale and Orkas paths), so model API keys are not handed to commands. External CLI agents (Claude Code, Codex, OpenClaw and others) inherit the app's full environment. No per-tool or per-request identity exists, and the narrower workspace-only mode is opt-in.

- **S L1:** Ambient OS-user authority narrowed only by subprocess environment scrubbing and sensitive-path prompts. — [src/core-agent/src/sandbox/executor.ts:548-558](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/sandbox/executor.ts#L548-L558); [src/main/model/core-agent/client.ts:188](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/client.ts#L188) (verified)
  - *To reach the next level:* No per-tool or per-capability credential scoping; read and write tools share the user's full authority.
- **C L1:** The bash tool gets the scrubbed environment, but ACP/OpenClaw CLI agents are spawned with the full process.env. — [src/main/features/local_agents/backends/_acp.ts:52](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/features/local_agents/backends/_acp.ts#L52); [src/core-agent/src/sandbox/executor.ts:548-558](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/sandbox/executor.ts#L548-L558) (verified)
  - *To reach the next level:* Sub-agent CLIs and other spawned helpers do not go through the same scrubbed environment.
- **D L1:** Default mode is all_files_approval (outside-workspace access); workspace_approval is opt-in. — [src/main/features/permissions.ts:41](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/features/permissions.ts#L41) (verified)
  - *To reach the next level:* Default is not workspace-only.
- **B L1:** A hijack acts with the user's whole home directory plus any connected accounts (Gmail, Drive, GitHub and similar), with sends and deletes still prompted. — [src/main/model/core-agent/local-tools.ts:1529](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/local-tools.ts#L1529); [src/main/model/core-agent/connector-meta-tools.ts:403](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/connector-meta-tools.ts#L403) (verified)
  - *To reach the next level:* Authority is not limited to one project or to read-mostly access.
- **Cap:** none

### C2 Approval gates — 0.25 (high)

Risky shell commands, sensitive paths, out-of-workspace deletes and connector send/delete actions open a blocking dialog. The dialog fails closed when no renderer is present, and task-wide grants live only in memory. The gate is a risk classifier, not an allowlist: ordinary commands, inline Python/Node scripts that delete files, GET-based network calls and connector edits run without asking. The dialog shows only the first 800 characters of a command. Protection of the approval-mode setting is not tamper-resistant.

- **S L2:** Per-call blocking dialog with host-derived key facts, but the command preview is cut at 800 characters. — [src/main/model/core-agent/bash-permissions.ts:46](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/bash-permissions.ts#L46); [src/main/model/core-agent/bash-permissions.ts:219-221](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/bash-permissions.ts#L219-L221); [src/main/model/core-agent/bash-permissions.ts:265-268](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/bash-permissions.ts#L265-L268) (verified)
  - *To reach the next level:* The approver does not always see the full command, and there is no argument-level allow/deny policy.
- **C L1:** Only commands the denylist-style classifier flags are gated; inline interpreter code is scanned only for external mutations, and connector 'W' writes are never gated. — [src/main/model/core-agent/local-tools.ts:4357](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/local-tools.ts#L4357); [src/main/model/core-agent/bash-risk.ts:1068-1074](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/bash-risk.ts#L1068-L1074); [src/main/features/connectors/action_policy.ts:15](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/features/connectors/action_policy.ts#L15) (verified)
  - *To reach the next level:* Auto-run commands are not a verified read-only allowlist; script deletes, GET egress and connector edits bypass the gate.
- **D L1:** On by default, but protection of the approval-mode setting is not tamper-resistant. (verified)
  - *To reach the next level:* The approval-mode setting needs robust protection.
- **B L1:** Bypassed actions include unrecoverable deletes and overwrites outside the workspace and connector edits; only Gmail mass-delete and a few account-level actions are hard-blocked. — [src/main/features/connectors/action_policy.ts:23](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/features/connectors/action_policy.ts#L23); [src/main/model/core-agent/local-tools.ts:1529](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/local-tools.ts#L1529) (verified)
  - *To reach the next level:* No checkpoints or rollback for files changed outside the workspace, and no rate limit on consequential actions.
- **Cap:** G2 — Protection of the approval-mode setting is not tamper-resistant, which undermines the gate at runtime.

### C3 Tool & action scoping — 0.38 (high)

File tools resolve paths and check them against workspace roots. write_file refuses to overwrite foreign files, connectors carry exact per-action risk tables and batch limits, and a few connector actions are blocked outright. The main tools stay general-purpose, though: bash takes any command string and web_fetch takes any URL, with no private-address block outside programmatic calls. In the default mode both reach the whole machine. The Commander can load more tool groups, including shell execution, on its own.

- **S L1:** Shell is raw passthrough filtered by a denylist classifier; web_fetch follows redirects to any URL. — [src/core-agent/src/sandbox/command-policy.ts:8](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/sandbox/command-policy.ts#L8); [src/core-agent/src/tools/web-fetch.ts:460-468](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/tools/web-fetch.ts#L460-L468); searched `rg -n 'isPrivate|169\.254|localhost'` in `src/core-agent/src/tools/web-fetch.ts` → 0 hits (The direct web_fetch path has no private/link-local address block.) (verified)
  - *To reach the next level:* No allowlist validation for shell or URLs (no internal-address block on the direct fetch path).
- **C L2:** File, delete and connector tools validate paths or actions; custom MCP tools are treated as unknown-risk but not argument-validated. — [src/main/model/core-agent/local-tools.ts:1529](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/local-tools.ts#L1529); [src/main/features/connectors/action_policy.ts:100](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/features/connectors/action_policy.ts#L100); [src/main/model/core-agent/programmatic-tool-policy.ts:122](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/programmatic-tool-policy.ts#L122) (verified)
  - *To reach the next level:* Extension tools are not wrapped by a shared argument-validation layer.
- **D L2:** Tools come in groups, but the default set includes write and exec, and groups are 'loadable' by the Commander. — [src/main/model/core-agent/tool-catalog.ts:50](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/tool-catalog.ts#L50); [src/main/model/core-agent/tool-catalog.ts:97](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/tool-catalog.ts#L97) (verified)
  - *To reach the next level:* The default tool set includes write and exec; read-only would require explicit enabling.
- **B L1:** A misused bash or web_fetch call can touch any user file or host; sensitive-path prompts are the only limit. — [src/main/model/core-agent/local-tools.ts:3140](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/local-tools.ts#L3140); [src/main/features/local_access_policy.ts:57](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/features/local_access_policy.ts#L57) (verified)
  - *To reach the next level:* Tools are not scoped to the workspace in the default mode.
- **Cap:** none

### C4 Code-execution isolation — 0.28 (high)

Shell commands run as ordinary child processes of the app, under the user's account. The environment is scrubbed, but nothing isolates the filesystem or network. A macOS Seatbelt profile limiting writes to the workspace exists, but it applies only in the opt-in workspace-only mode on macOS. It still allows all reads and all network, and its enforcement is not a complete boundary. Linux and Windows have no isolation at all.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Default execution is a same-user host subprocess. — [src/main/model/core-agent/local-tools.ts:3140](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/local-tools.ts#L3140); [src/core-agent/src/sandbox/executor.ts:346](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/sandbox/executor.ts#L346); searched `rg -n -i 'docker|firecracker|gvisor|landlock|seccomp|bwrap|bubblewrap'` in `src/core-agent/src/sandbox` → 0 hits (No container, microVM, Landlock or seccomp backend exists in the sandbox package.) (verified)
    - *To reach the next level:* No OS-level isolation primitive is applied in the default mode.
  - **C L0:** No execution path is sandboxed in the default mode (bash, process sessions, skills, MCP stdio and CLI agents all run on the host). — [src/main/model/core-agent/local-tools.ts:3140](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/local-tools.ts#L3140) (verified)
    - *To reach the next level:* Not even the main bash tool is isolated by default.
  - **D L0:** The only isolation (macOS write profile) is off in the default all_files_approval mode. — [src/main/model/core-agent/local-tools.ts:3140](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/local-tools.ts#L3140); [src/main/features/permissions.ts:41](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/features/permissions.ts#L41) (verified)
    - *To reach the next level:* Isolation is not on by default.
  - **B L0:** A command reaches the full home directory (including ~/.ssh and the Orkas key store) and unrestricted network. — [src/core-agent/src/sandbox/executor.ts:550](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/sandbox/executor.ts#L550); [src/core-agent/src/sandbox/executor.ts:681](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/sandbox/executor.ts#L681) (verified)
    - *To reach the next level:* Workspace-only filesystem access and denied or allowlisted egress.
- **opt-in workspace_approval mode on macOS (Seatbelt write profile)** (alt; raw 0.28, cap G1 → 0.28) ← counted
  - **S L2:** Seatbelt profile denies file writes outside the workspace roots but starts from (allow default), so reads and network are unrestricted. — [src/core-agent/src/sandbox/executor.ts:335](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/sandbox/executor.ts#L335) (verified)
    - *To reach the next level:* Network is not denied by default and reads are not confined, so this falls short of the hardened-profile anchor.
  - **C L1:** Applies to bash and process sessions on macOS only; Linux/Windows and CLI agents run unsandboxed. — [src/core-agent/src/sandbox/executor.ts:346](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/sandbox/executor.ts#L346) (verified)
    - *To reach the next level:* Other platforms and spawned agents are not covered.
  - **D L0:** Opt-in mode. — [src/core-agent/src/sandbox/executor.ts:346](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/sandbox/executor.ts#L346); [src/main/features/permissions.ts:41](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/features/permissions.ts#L41) (verified)
    - *To reach the next level:* Not on by default.
  - **B L1:** Inside the profile a command can still read the whole home directory and use the network. — [src/core-agent/src/sandbox/executor.ts:335](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/sandbox/executor.ts#L335); [src/core-agent/src/sandbox/executor.ts:681](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/sandbox/executor.ts#L681) (verified)
    - *To reach the next level:* Secrets are still readable and network egress is open.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.25 (high)

No mechanism tracks or acts on untrusted content. Some dangerous operations always ask for approval, whatever the input source: curl/ssh-style egress, external mutations, sensitive paths, and connector sends and deletes. Others do not: web_fetch can GET any URL with data in the query string, inline Python or Node scripts are checked only for external-API writes, and script-based deletes go unflagged. Injected content can therefore both leak data and destroy files without a human.

- **S L2:** Some egress and mutating operations always require approval, but not consistently (GET egress and script deletes are not covered). — [src/main/model/core-agent/bash-risk.ts:284](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/bash-risk.ts#L284); [src/main/model/core-agent/bash-risk.ts:1068-1074](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/bash-risk.ts#L1068-L1074); [src/main/model/core-agent/connector-meta-tools.ts:403](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/connector-meta-tools.ts#L403) (verified)
  - *To reach the next level:* No Rule-of-Two enforcement once untrusted content is in context.
- **C L2:** The partial approvals apply equally to every source because they are source-agnostic. — [src/main/model/core-agent/local-tools.ts:4357](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/local-tools.ts#L4357); searched `rg -n 'isPrivate|169\.254|localhost'` in `src/core-agent/src/tools/web-fetch.ts` → 0 hits (The direct web_fetch path has no private/link-local address block.) (verified)
  - *To reach the next level:* Tool results and other agents' messages have the same standing as user instructions.
- **D L1:** On by default, but protection of the approval-mode setting is not tamper-resistant. (verified)
  - *To reach the next level:* Nothing the agent reads should be able to configure the protection away.
- **B L0:** Default config allows unattended exfiltration (web_fetch GET to any URL) and unattended irreversible deletes or overwrites via scripts outside the workspace. — [src/core-agent/src/tools/web-fetch.ts:460-468](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/tools/web-fetch.ts#L460-L468); [src/main/model/core-agent/bash-risk.ts:1068-1074](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/bash-risk.ts#L1068-L1074); [src/main/model/core-agent/local-tools.ts:1529](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/local-tools.ts#L1529) (verified)
  - *To reach the next level:* Exfiltration or irreversible actions would need a human approval.
- **Cap:** C5-WORSTCASE — A hijacked agent can leak data and take irreversible file actions without a human in the default configuration.

### C6 Memory, context & configuration integrity — 0.30 (high)

The memory tool tells the model to write durable memory on its own initiative. Entries are screened only by a short regex list for injection phrases, then frozen into the system prompt of later sessions. Agents also learn reusable skills through a skill-management tool and reflection. AGENTS.md files from the working directory up to the repository root are loaded automatically. Workspace files cannot add tools, MCP servers or approval rules. Memory is stored per user and per agent, with shared and user tiers that apply across agents.

- **S L1:** Model writes are filtered by a regex denylist and re-injected as system-prompt context; AGENTS.md loads silently. — [src/core-agent/src/tools/memory-tool.ts:46](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/tools/memory-tool.ts#L46); [src/main/features/memory.ts:84](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/features/memory.ts#L84); [src/main/features/memory.ts:6](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/features/memory.ts#L6) (verified)
  - *To reach the next level:* Memory entries carry no provenance and writes need no review or validation beyond regexes.
- **C L1:** Only the memory store has a write filter; learned skills and auto-loaded AGENTS.md files are uncontrolled. — [src/core-agent/src/agent/repository-instructions.ts:51](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/agent/repository-instructions.ts#L51); [src/main/features/memory.ts:84](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/features/memory.ts#L84) (verified)
  - *To reach the next level:* Instruction files and learned skills have no write or load control.
- **D L2:** Memory lives under per-user data paths with agent and project tiers bound by the host. — [src/main/features/memory.ts:6](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/features/memory.ts#L6) (verified)
  - *To reach the next level:* Shared and user tiers are writable by every agent, so the model can write across namespaces.
- **B L1:** Poisoned memory persists across the user's sessions and agents and can steer tool use. — [src/main/features/memory.ts:6](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/features/memory.ts#L6); [src/core-agent/src/tools/memory-tool.ts:46](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/tools/memory-tool.ts#L46) (verified)
  - *To reach the next level:* Writes are not session-scoped or gated by human review.
- **Cap:** none

### C7 Third-party extensions — 0.30 (high)

Third-party additions mostly need explicit consent. A custom MCP install shows the exact command or URL. Skill imports must come from a path the user supplied, and marketplace installs are user-confirmed and version-recorded. Sources are not integrity-checked, though: marketplace integrity handling is not a complete check, and custom MCP commands run as written. Extensions run as the same OS user. MCP servers get only their configured environment, while external CLI agents inherit the full app environment.

- **S L1:** User-chosen sources with no signature check; marketplace items are versioned but integrity handling is not a complete check. — [src/main/features/connectors/install_confirm.ts:9](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/features/connectors/install_confirm.ts#L9) (verified)
  - *To reach the next level:* Integrity verification against a pinned hash or signature.
- **C L1:** Only marketplace items carry version pins; custom MCP servers, imported skills and CLI agents are unverified. (verified)
  - *To reach the next level:* Most extension types are not verified.
- **D L2:** Custom MCP installs show the exact command and skill imports require a user-supplied path. — [src/main/features/connectors/install_confirm.ts:9](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/features/connectors/install_confirm.ts#L9); [src/main/features/group_chat/bus.ts:8520](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/features/group_chat/bus.ts#L8520) (verified)
  - *To reach the next level:* Marketplace updates are not re-approved, and the mechanism's weak verification limits this level.
- **B L1:** Extensions run as separate processes of the same user; MCP gets its configured env, CLI agents get the full env. — [src/main/features/connectors/mcp-client.ts:152](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/features/connectors/mcp-client.ts#L152); [src/main/features/local_agents/backends/_acp.ts:52](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/features/local_agents/backends/_acp.ts#L52) (verified)
  - *To reach the next level:* No per-extension sandbox or scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.50 (high)

Logs go through structured field redaction and positional secret masking, and permission logs deliberately omit command and path text. Shell children never receive provider keys in their environment. In the open-source build, stored keys and OAuth tokens are only obfuscated: the AES key comes from a constant salt in the source plus the user id, so the agent's own shell can decrypt them. Nothing redacts secrets in tool results sent to the model, and transcripts are plain JSONL. No third-party telemetry was found.

- **S L2:** Field-level and positional log redaction; credentials stored with source-derivable obfuscation. — [src/main/util/crypto-vault.ts:4](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/util/crypto-vault.ts#L4); [src/main/util/crypto-vault.ts:42](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/util/crypto-vault.ts#L42); [src/main/util/log-sanitize.ts:22](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/util/log-sanitize.ts#L22); [src/main/logger.ts:97](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/logger.ts#L97) (verified)
  - *To reach the next level:* No OS keychain or real encryption at rest, and no redaction before model-bound messages.
- **C L2:** Logs, permission records, connector previews and subprocess env are protected; model-bound tool results and transcripts are not. — [src/main/features/connectors/action_confirm.ts:24](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/features/connectors/action_confirm.ts#L24); searched `rg -n 'redact|sanitize'` in `src/core-agent/src/tools/builtin.ts` → 0 hits (Shell, file and fetch tool results reach the model without secret redaction.) (verified)
  - *To reach the next level:* Model-bound messages and saved transcripts are not redacted.
- **D L3:** No third-party telemetry SDK; logger redaction is always on. — [src/main/logger.ts:97](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/logger.ts#L97); [src/main/model/core-agent/bash-permissions.ts:239](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/bash-permissions.ts#L239) (verified)
  - *To reach the next level:* Transcripts are not encrypted or minimised by default.
- **B L1:** Long-lived provider API keys and connector tokens sit in a decryptable store the agent's shell can read. — [src/main/util/crypto-vault.ts:4](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/util/crypto-vault.ts#L4) (verified)
  - *To reach the next level:* Keys are not scoped or short-lived.
- **Cap:** none

### C9 Audit & traceability — 0.45 (high)

Each session is persisted as JSONL with tool calls and results, and persistence failures raise errors instead of passing silently. Permission requests and denials are logged with masked ids and risk categories, deliberately without command text. Records sit in the Orkas data directory. In the default mode the agent's file and shell tools can write there, and nothing chains, signs or exports the records.

- **S L2:** Structured per-session transcript of tool_use/tool_result pairs. — [src/main/model/core-agent/session-store.ts:24](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/session-store.ts#L24); [src/main/model/core-agent/bash-permissions.ts:240](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/bash-permissions.ts#L240) (verified)
  - *To reach the next level:* No actor/approver attribution on tool records and no correlation across sub-agents.
- **C L2:** Built-in tool calls and approval requests are recorded; external CLI agents' native actions are only mirrored as events. — [src/main/model/core-agent/bash-permissions.ts:240](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/bash-permissions.ts#L240); [src/main/model/core-agent/bash-permissions.ts:239](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/bash-permissions.ts#L239) (verified)
  - *To reach the next level:* Approval decisions are not tied to the exact recorded call, and extensions are covered only partially.
- **D L1:** On by default but stored under the data root, which tools can write in the default mode. — [src/main/paths.ts:76](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/paths.ts#L76); searched `rg -n -i 'hmac|signature|opentelemetry'` in `src/main/model/core-agent/session-store.ts src/core-agent/src/agent/persistent-session.ts` → 0 hits (Session transcripts are plain JSONL with no integrity protection or standard export.) (verified)
  - *To reach the next level:* Records should be written outside anything the model's tools can edit.
- **B L2:** Persistence errors are surfaced rather than swallowed. — [src/core-agent/src/agent/persistent-session.ts:179](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/agent/persistent-session.ts#L179) (verified)
  - *To reach the next level:* No replayable durable trail guaranteed per action, and actions are not blocked on log failure.
- **Cap:** none

### C10 Limits & kill switch — 0.40 (high)

Each turn has a tool-round cap (100, or 120 for the Commander), a 24-hour execution deadline and a 30-minute idle watchdog, plus a 60-minute default timeout per shell command. Stopping kills the whole process group, and agent concurrency is capped at 4 per conversation. The model can pass a longer shell timeout or start background processes that outlive the conversation. There is no token or cost ceiling.

- **S L2:** Iteration cap plus wall-clock and per-command timeouts enforced in code; process-group kill on stop. — [src/core-agent/src/config/schema.ts:37](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/config/schema.ts#L37); [src/main/util/agent-execution-budget.ts:5](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/util/agent-execution-budget.ts#L5); [src/core-agent/src/sandbox/executor.ts:187](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/sandbox/executor.ts#L187); searched `rg -n -i 'max_?cost|cost_?limit|budget_?usd|spend_?limit'` in `src/core-agent/src` → 0 hits (No token or spend ceiling in the agent core.) (verified)
  - *To reach the next level:* No token/cost cap and no rate limit on side-effecting tools.
- **C L2:** Top-level loop plus tool timeouts; each dispatched agent gets its own round budget. — [src/main/features/group_chat/actor-budgets.ts:15](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/features/group_chat/actor-budgets.ts#L15); [src/main/features/group_chat/bus.ts:3708](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/features/group_chat/bus.ts#L3708); [src/core-agent/src/sandbox/executor.ts:91](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/sandbox/executor.ts#L91) (verified)
  - *To reach the next level:* Sub-agents and background processes do not share one budget.
- **D L1:** Defaults are very large (24h, 60-minute commands), and the model can raise the bash timeout or detach processes. — [src/core-agent/src/tools/builtin.ts:258](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/tools/builtin.ts#L258); [src/core-agent/src/tools/builtin.ts:251](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/tools/builtin.ts#L251); [src/main/util/agent-execution-budget.ts:5](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/util/agent-execution-budget.ts#L5) (verified)
  - *To reach the next level:* The model must not be able to raise its own limits.
- **B L1:** Ceilings are hours long, and background processes keep running after the conversation ends. — [src/core-agent/src/tools/builtin.ts:251](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/core-agent/src/tools/builtin.ts#L251); [src/main/model/core-agent/bash-permissions.ts:321](https://github.com/orkas-ai/orkas/blob/595358993d6c98e868030990fc9e3a1130c45191/src/main/model/core-agent/bash-permissions.ts#L321) (verified)
  - *To reach the next level:* Background work should not survive a stop.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: web_fetch/web_search results, files, connector and MCP results enter context (src/core-agent/src/tools/web-fetch.ts:460) · [B] sensitive data/systems: the user's home directory, Orkas provider keys and connector tokens on disk (src/main/util/crypto-vault.ts:4) · [C] state change / egress: unsandboxed bash and arbitrary-URL web_fetch (src/core-agent/src/sandbox/executor.ts:346, src/core-agent/src/tools/web-fetch.ts:468) · Same default session? Yes

## Highest-impact improvements
1. Protect the Orkas data root (sessions, memory, secrets, configuration) and the audit trail from the agent's own write and bash paths. — C2 D L1→L2, +0.050 before caps (Playbook 5)
2. Show the full command (scrollable) in the bash approval dialog instead of an 800-character preview. — C2 S L2→L3, +0.075 before caps (Playbook 5)
3. Apply the macOS Seatbelt write profile in every mode (workspace plus explicitly approved roots), deny network by default, and add Landlock/bubblewrap on Linux. — C4 S L0→L2, +0.150 before caps (Playbook 3)
4. Gate web_fetch to non-public addresses and require approval for script/inline-interpreter network and delete operations the classifier cannot see into. — C5 B L0→L1, +0.050 before caps (Playbook 1)
5. Add a per-run token/cost ceiling, cap model-supplied bash timeouts, and stop background processes when a conversation is stopped. — C10 S L2→L3, +0.075 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the open-source source build; the hosted packaged build adds a private secret backend and account layer that are not in this repository.
- The repository is very large (2,395 files); the renderer UI, video/image studio tools, office tools, and most connector catalogs were sampled, not read in full.
- Protection of the approval-mode setting was traced through code but not exercised.
- No text aimed at AI reviewers was found in the repository.
