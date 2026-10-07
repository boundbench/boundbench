# Defense-in-Depth Score: PI-Desktop

**Repo:** https://github.com/vastsa/pi-desktop · **Commit:** `831b66dfa8e819d2079c31b1cd98206818d9ac15` (0.16.1) · **Reviewed:** 2026-10-04
**What it is:** Local-first Electron desktop workspace for AI coding agents, built on the pi agent harness with a Rust host core, plugins, MCP servers and subagents.
**Category:** Coding
**Scored configuration:** Desktop app as installed, Agent mode, global permission mode default 'ask', no plugins installed, a project folder opened.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents yes · external communication opt-in

## Score: 2.7 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L1 | L1 | 0.17 | — | **0.17** | High |
| C2 | Approval gates | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C3 | Tool & action scoping | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C6 | Memory, context & configuration integrity | L0 | L1 | L1 | L1 | 0.17 | C6-REPOCONFIG | **0.17** | High |
| C7 | Third-party extensions | L0 | L1 | L0 | L0 | 0.07 | C7-RCELOAD | **0.07** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L3 | L1 | 0.50 | — | **0.50** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C10 | Limits & kill switch | L1 | L2 | L1 | L1 | 0.33 | — | **0.33** | High |


PI-Desktop has a real, centralized approval gate: in the default 'ask' mode every Write, Edit, Bash and MCP call stops for a per-call approval card. Behind that gate there is no sandbox. Approved commands run as you, with your full environment and network. Repository-supplied configuration is not integrity-protected, and rendered model output is not fully confined.

## Critical gaps
- Shell commands and MCP servers run unsandboxed as the desktop user with the full inherited environment (credentials, home directory, network). (ASI05, T11; C4) — [packages/host-runtime/src/host-process.ts:101-107](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/packages/host-runtime/src/host-process.ts#L101-L107); [apps/desktop/electron/main/plugin-mcp.ts:160-170](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/apps/desktop/electron/main/plugin-mcp.ts#L160-L170)
- MCP stdio servers are spawned with the full process environment and no verification. (ASI04, T17; C7) — [apps/desktop/electron/main/plugin-mcp.ts:160-170](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/apps/desktop/electron/main/plugin-mcp.ts#L160-L170); [apps/desktop/electron/main/user-mcp.ts:432](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/apps/desktop/electron/main/user-mcp.ts#L432)

## Criterion details

### C1 Identity & least privilege — 0.17 (high)

PI-Desktop runs every action with the desktop user's full authority. The Bash runner and the agent sidecar inherit the whole process environment, so any cloud, GitHub or SSH credential the user has is reachable from a shell command. One real narrowing exists: the sidecar can only call an allowlist of host methods and can resolve only the provider key bound to its own session, and plugin processes get a scrubbed environment. User-configured and project MCP servers, however, also receive the full environment.

- **S L0:** Commands run as the OS user with the full inherited environment; no scoped or per-tool credential exists. — [packages/host-runtime/src/host-process.ts:101-107](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/packages/host-runtime/src/host-process.ts#L101-L107); [apps/desktop/electron/main/agent-sidecar.ts:78-81](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/apps/desktop/electron/main/agent-sidecar.ts#L78-L81) (verified)
  - *To reach the next level:* Scrub the Bash/MCP subprocess environment or issue scoped credentials per capability.
- **C L1:** The sidecar's host-proxy allowlist and session-bound provider auth narrow the agent loop, and plugins get a scrubbed env, but Bash and MCP children inherit everything. — [packages/host-runtime/src/agent-sidecar.ts:44-51](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/packages/host-runtime/src/agent-sidecar.ts#L44-L51); [apps/desktop/electron/main/child-process-env.ts:58](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/apps/desktop/electron/main/child-process-env.ts#L58); [apps/desktop/electron/main/plugin-mcp.ts:160-170](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/apps/desktop/electron/main/plugin-mcp.ts#L160-L170) (verified)
  - *To reach the next level:* Route every subprocess (Bash, MCP stdio) through the same environment-scrubbing layer plugins use.
- **D L1:** Default install runs as the user with ambient credentials; the approval gate (ask) is the only default narrowing and users can switch to auto. — [crates/host-core/src/rpc/mod.rs:3662-3676](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/rpc/mod.rs#L3662-L3676); [packages/host-runtime/src/host-process.ts:101-107](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/packages/host-runtime/src/host-process.ts#L101-L107) (verified)
  - *To reach the next level:* Ship a minimal default identity for tool subprocesses that needs explicit operator elevation.
- **B L1:** A hijacked shell reaches everything the user's ambient credentials reach (multiple systems), behind the per-call Bash approval in the default mode. — [packages/host-runtime/src/host-process.ts:101-107](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/packages/host-runtime/src/host-process.ts#L101-L107); [crates/host-core/src/permissions.rs:120-131](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/permissions.rs#L120-L131) (verified)
  - *To reach the next level:* Limit what a hijacked tool process can reach to one project or read-only scope.
- **Cap:** none

### C2 Approval gates — 0.50 (high)

Every built-in, MCP, plugin and subagent tool call is routed through one host-core permission gate, and the default mode is 'ask': Write, Edit and Bash need a per-call approval card showing the arguments, MCP tools always need approval, and denials are first-class. The gate does not cover every path and its approval card does not always show the exact call, plugins can self-declare a tool as low-risk to get it auto-approved, and 'Allow for session' approves a tool by name for all later arguments. File edits have review snapshots for rollback; shell side effects do not.

- **S L2:** Per-call approval with risk tiers and the raw argument object, but the preview does not always show the exact call and session grants are by tool name, not arguments. — [crates/host-core/src/permissions.rs:120-131](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/permissions.rs#L120-L131); [crates/host-core/src/rpc/mod.rs:3905-3910](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/rpc/mod.rs#L3905-L3910) (verified)
  - *To reach the next level:* Show the complete call and add argument-level allow/deny rules so an approval covers exactly what executes.
- **C L2:** Built-in, MCP, plugin and subagent calls all hit tools.execute, but the gate does not cover every path and plugin manifests can self-declare low risk for auto-approval. — [crates/host-core/src/rpc/mod.rs:4033-4041](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/rpc/mod.rs#L4033-L4041); [crates/host-core/src/permissions.rs:124-125](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/permissions.rs#L124-L125) (verified)
  - *To reach the next level:* Gate every tool path and stop trusting plugin-declared low risk.
- **D L2:** Ask is the default, but a plain settings/composer choice of 'auto' or a user-scope subagent/scheduled task declaring auto silently removes all prompts; project-scope subagents cannot. — [crates/host-core/src/rpc/mod.rs:3662-3676](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/rpc/mod.rs#L3662-L3676); [crates/host-core/src/permissions.rs:250-257](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/permissions.rs#L250-L257); [packages/shared/src/subagent-definition.ts:202-205](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/packages/shared/src/subagent-definition.ts#L202-L205) (verified)
  - *To reach the next level:* Make auto a loudly named, time-bounded elevation rather than a persistent default setting.
- **B L2:** Write/Edit changes get review snapshots that can be rolled back; Bash and MCP side effects are irreversible. — [crates/host-core/src/review.rs:215-238](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/review.rs#L215-L238) (verified)
  - *To reach the next level:* Add checkpoints for shell side effects and previews for external actions.
- **Cap:** none

### C3 Tool & action scoping — 0.45 (high)

File tools are well scoped: paths are resolved through symlinks and must stay inside the project or the session scratch folder, and any outside path needs an explicit approval. But the default tool set always includes a raw Bash tool that accepts any command string, plus write tools, and MCP tool arguments are passed through unvalidated. A misused shell command can reach the whole machine.

- **S L2:** Realpath-based workspace containment for Read/Glob/Grep/Write/Edit, but Bash takes an arbitrary command string. — [crates/host-core/src/workspace.rs:188-210](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/workspace.rs#L188-L210); [crates/host-core/src/tools/mod.rs:3007](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/tools/mod.rs#L3007) (verified)
  - *To reach the next level:* Replace or constrain the general shell with narrow tools or parsed-command allowlists.
- **C L2:** All built-in file tools validate paths; Bash and MCP/plugin tool arguments are not validated by a shared layer. — [crates/host-core/src/rpc/mod.rs:1299-1321](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/rpc/mod.rs#L1299-L1321); [crates/host-core/src/rpc/mod.rs:4033-4041](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/rpc/mod.rs#L4033-L4041) (verified)
  - *To reach the next level:* Add a shared validation layer that extension tools inherit.
- **D L2:** Agent mode (default) exposes write and exec tools; Plan/Goal modes are a selectable read-mostly group that still includes Bash. — [crates/host-core/src/permissions.rs:144-149](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/permissions.rs#L144-L149); [crates/host-core/src/permissions.rs:120-131](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/permissions.rs#L120-L131) (verified)
  - *To reach the next level:* Default to a read-only tool set and require explicit enabling of write/exec.
- **B L1:** Bash runs with the user's authority anywhere on the machine (approval-gated in ask mode). — [crates/host-core/src/tools/mod.rs:3007](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/tools/mod.rs#L3007); [packages/host-runtime/src/host-process.ts:101-107](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/packages/host-runtime/src/host-process.ts#L101-L107) (verified)
  - *To reach the next level:* Scope command execution to the workspace with bounded quantities.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

There is no execution sandbox. Approved shell commands run through host-core's runner as the desktop user, in the project directory, with the full inherited environment and unrestricted network. The only protection is the approval prompt; once a command runs, it can do anything the user can.

- **S L0:** Same-user subprocess; no container, OS sandbox profile or VM. — searched `rg -n -i 'sandbox-exec|seatbelt|landlock|bwrap|bubblewrap|seccomp|firejail|gvisor|firecracker'` in `crates packages apps/desktop/electron` → 0 hits (No OS sandbox primitive anywhere in host-core, runtime or Electron main.); [packages/host-runtime/src/host-process.ts:101-107](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/packages/host-runtime/src/host-process.ts#L101-L107) (verified)
  - *To reach the next level:* Run shell commands in an OS sandbox (Seatbelt/Landlock+seccomp) or container with workspace-only writes and no network by default.
- **C L0:** No execution path is sandboxed. — searched `rg -n -i 'sandbox-exec|seatbelt|landlock|bwrap|bubblewrap|seccomp|firejail|gvisor|firecracker'` in `crates packages apps/desktop/electron` → 0 hits (No OS sandbox primitive anywhere in host-core, runtime or Electron main.) (verified)
  - *To reach the next level:* Sandbox the Bash tool and every spawned helper (MCP stdio servers, hooks).
- **D L0:** No sandbox exists to be on by default. — searched `rg -n -i 'sandbox-exec|seatbelt|landlock|bwrap|bubblewrap|seccomp|firejail|gvisor|firecracker'` in `crates packages apps/desktop/electron` → 0 hits (No OS sandbox primitive anywhere in host-core, runtime or Electron main.) (verified)
  - *To reach the next level:* Ship the sandbox on by default.
- **B L0:** Host-equivalent: home directory, credentials in environment, full network. — [packages/host-runtime/src/host-process.ts:101-107](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/packages/host-runtime/src/host-process.ts#L101-L107); [crates/host-core/src/tools/mod.rs:3007](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/tools/mod.rs#L3007) (verified)
  - *To reach the next level:* Confine execution to the workspace with no secrets and allowlisted egress.
- **Cap:** none

### C5 Untrusted input blast radius — 0.05 (high)

Nothing in the code tracks whether untrusted content (repository files, MCP results, web search results) has entered a session, and nothing changes once it has. Project instruction files and tool results go straight into context. Reading files in the workspace needs no approval, and an unattended exfiltration path exists. Destructive actions still need approval in the default mode.

- **S L0:** No taint tracking, quarantine or provenance-aware gating; project memory is framed only by a prompt sentence. — [packages/agent-runtime/src/project-memory-prompt.ts:12](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/packages/agent-runtime/src/project-memory-prompt.ts#L12); [packages/agent-runtime/src/project-instructions.ts:5-10](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/packages/agent-runtime/src/project-instructions.ts#L5-L10) (verified)
  - *To reach the next level:* Disable or force approval for egress-capable paths once untrusted content is in context.
- **C L0:** Tool results, MCP outputs and repo instruction files enter context with no distinction. — [packages/agent-runtime/src/project-instructions.ts:5-10](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/packages/agent-runtime/src/project-instructions.ts#L5-L10); [crates/host-core/src/rpc/mod.rs:4033-4041](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/rpc/mod.rs#L4033-L4041) (verified)
  - *To reach the next level:* Mark every untrusted source and apply the limit to tool results and MCP outputs.
- **D L0:** No control exists to be on by default. — [packages/agent-runtime/src/project-instructions.ts:5-10](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/packages/agent-runtime/src/project-instructions.ts#L5-L10) (verified)
  - *To reach the next level:* Ship the untrusted-content limit on by default.
- **B L1:** Workspace reads are auto-approved and an unattended exfiltration channel exists; irreversible actions need approval in ask mode. — [crates/host-core/src/permissions.rs:122](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/permissions.rs#L122) (verified)
  - *To reach the next level:* Close unattended egress paths so exfiltration also needs a human.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.17 (high)

Opening a project silently trusts files the repository ships: AGENTS.md and CLAUDE.md are auto-loaded into the prompt, and other project-supplied capability files are not integrity-protected. Project memory is written only from the UI, and project-level subagents cannot raise their own permission mode, but neither offsets the repo-config path.

- **S L0:** Instruction files load silently, and project capability files are not integrity-protected. (verified)
  - *To reach the next level:* Require an explicit workspace-trust decision before loading project capability files.
- **C L1:** Project memory is UI-only and project subagents cannot declare permission, but project capability files, skills and instruction files are uncontrolled. — [packages/shared/src/subagent-definition.ts:202-205](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/packages/shared/src/subagent-definition.ts#L202-L205) (verified)
  - *To reach the next level:* Cover every auto-loaded project file and setting with the same trust check.
- **D L1:** State is per project path on a single-user desktop, but project-scoped activation is not gated by a trust decision. (verified)
  - *To reach the next level:* Default project-scoped capabilities to disabled until the user enables them.
- **B L1:** A poisoned project config persists across all the user's sessions on that project and triggers code execution. (verified)
  - *To reach the next level:* Make project config inert until reviewed, with an easy purge.
- **Cap:** C6-REPOCONFIG — Project-supplied configuration can take effect without an explicit trust decision.

### C7 Third-party extensions — 0.07 (high)

Plugins are handled carefully: they run in a separate process per plugin with a scrubbed environment, a permission broker, and marketplace packages are checked against a SHA-256 digest. MCP servers are not. A user-typed stdio server can name any command (typically an unpinned package runner) and is started with the user's full environment; project-supplied servers are not gated by a trust decision.

- **S L0:** MCP commands are unverified and user MCP commands are unpinned; project-supplied servers are not gated by a trust decision. — [apps/desktop/electron/main/user-mcp.ts:432](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/apps/desktop/electron/main/user-mcp.ts#L432) (verified)
  - *To reach the next level:* Pin MCP server packages and verify integrity.
- **C L1:** Only marketplace plugins are hash-checked; MCP servers are unverified. — [crates/host-core/src/plugins/install.rs:30-31](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/plugins/install.rs#L30-L31) (verified)
  - *To reach the next level:* Extend verification to MCP servers and imported skills.
- **D L0:** Extension enablement is not consent-gated on every path. (verified)
  - *To reach the next level:* Allow extensions only from user/admin scope and show the exact command before first launch.
- **B L0:** MCP stdio servers run as the same user with the full process environment. — [apps/desktop/electron/main/plugin-mcp.ts:160-170](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/apps/desktop/electron/main/plugin-mcp.ts#L160-L170) (verified)
  - *To reach the next level:* Spawn MCP servers with a scrubbed environment and sandbox, as plugins get.
- **Cap:** C7-RCELOAD — Extension loading can run code without consent.

### C8 Secrets & sensitive-data protection — 0.50 (high)

There is no telemetry and crash dumps stay local. Logs and the audit table pass through redaction for tokens, keys and bearer headers. Provider keys are stored AES-GCM encrypted, but the key sits in an owner-only file right next to them, so in practice this is plaintext protected by file permissions. Shell and MCP subprocesses inherit the full environment, and tool output containing secrets is sent to the model unredacted.

- **S L2:** Redaction on logs and audit; secrets encrypted with a co-located 0600 machine key (permissions-equivalent). — [crates/host-core/src/secrets.rs:15-48](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/secrets.rs#L15-L48); [crates/host-core/src/secrets.rs:64-78](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/secrets.rs#L64-L78); [apps/desktop/electron/main/logger.ts:122-128](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/apps/desktop/electron/main/logger.ts#L122-L128) (verified)
  - *To reach the next level:* Use the OS keychain (safeStorage is already used for remote-host tokens) and redact model-bound tool output.
- **C L2:** Logs, audit records and error text are redacted; model-bound messages and subprocess environments are not. — [crates/host-core/src/audit.rs:29-36](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/audit.rs#L29-L36); [apps/desktop/electron/main/logger.ts:122-128](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/apps/desktop/electron/main/logger.ts#L122-L128); [packages/host-runtime/src/host-process.ts:101-107](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/packages/host-runtime/src/host-process.ts#L101-L107) (verified)
  - *To reach the next level:* Redact secrets in model-bound tool results and scrub subprocess environments.
- **D L3:** No telemetry SDK; crash reporting never uploads; redaction is built into the logger and audit writer. — [apps/desktop/electron/main/bootstrap/startup.ts:131](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/apps/desktop/electron/main/bootstrap/startup.ts#L131); [crates/host-core/src/audit.rs:31](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/audit.rs#L31) (verified)
  - *To reach the next level:* Encrypt or minimise stored transcripts by default.
- **B L1:** Long-lived provider API keys and the user's ambient credentials are reachable by any approved shell command. — [crates/host-core/src/secrets.rs:15-48](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/secrets.rs#L15-L48); [packages/host-runtime/src/host-process.ts:101-107](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/packages/host-runtime/src/host-process.ts#L101-L107) (verified)
  - *To reach the next level:* Use scoped, short-lived provider tokens and keep them out of subprocess reach.
- **Cap:** none

### C9 Audit & traceability — 0.45 (high)

Host-core writes a structured, redacted audit row for every tool it executes or denies, including whether a prompt was shown and how long approval took, into a SQLite database in the app data directory. MCP, plugin and subagent calls pass through the same path. Some host-local tool paths are not audited, the audit row omits the arguments (only the session transcript has them), there is no approver identity, and write failures are silently ignored.

- **S L2:** Structured per-call records (tool, status, timings, prompted) plus the session transcript with arguments. — [crates/host-core/src/rpc/mod.rs:4129-4143](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/rpc/mod.rs#L4129-L4143); [crates/host-core/src/audit.rs:29-36](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/audit.rs#L29-L36) (verified)
  - *To reach the next level:* Add actor attribution (approver, delegating session) and correlation IDs to audit records.
- **C L2:** Every host-core tool path, including MCP, plugins and subagents, plus denials; host-local tools are not recorded. — [crates/host-core/src/rpc/mod.rs:4129-4143](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/rpc/mod.rs#L4129-L4143) (verified)
  - *To reach the next level:* Audit host-local tools too.
- **D L2:** On by default and outside the workspace, but in a database the user-level Bash process can modify. — [crates/host-core/src/audit.rs:29-36](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/audit.rs#L29-L36) (verified)
  - *To reach the next level:* Write audit records through a component the agent's processes cannot alter.
- **B L1:** Records are committed per action but append errors are discarded (`let _ =`) and the action proceeds. — [crates/host-core/src/rpc/mod.rs:4129-4143](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/rpc/mod.rs#L4129-L4143) (verified)
  - *To reach the next level:* Surface audit write failures and refuse high-risk actions when the record cannot be written.
- **Cap:** none

### C10 Limits & kill switch — 0.33 (high)

The main agent loop has no step, token or cost limit. Bash commands default to a 60-second timeout but the model can request up to six hours per call, and subagents are capped at 10 running at once and six hours each. Stopping a turn kills the shell's whole process group. Scheduled tasks can keep starting new runs in the background.

- **S L1:** Per-command timeouts and a process-group kill exist, but no iteration, wall-clock or cost cap on the agent loop. — searched `rg -n -i 'maxTurns|max_turns|maxSteps|max_steps|maxIterations'` in `packages/agent-runtime/src` → 2 hits (Both hits are test assertions that subagent builtins do NOT carry maxTurns; no loop iteration cap exists.); [crates/host-core/src/tools/mod.rs:40-41](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/tools/mod.rs#L40-L41); [crates/host-core/src/tools/mod.rs:150](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/tools/mod.rs#L150) (verified)
  - *To reach the next level:* Add a session iteration cap and a token/cost budget enforced in code.
- **C L2:** Tool timeouts, a global tool admission budget, and subagent concurrency/duration caps; the top-level loop itself is unbounded. — [crates/host-core/src/tool_budget.rs:7-13](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/tool_budget.rs#L7-L13); [packages/shared/src/subagent-definition.ts:177-211](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/packages/shared/src/subagent-definition.ts#L177-L211) (verified)
  - *To reach the next level:* Make subagents and scheduled runs count against one shared session budget.
- **D L1:** The model can raise any Bash timeout to six hours; subagent default duration is the six-hour maximum. — [crates/host-core/src/tools/mod.rs:40-41](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/tools/mod.rs#L40-L41); [packages/shared/src/subagent-definition.ts:177-211](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/packages/shared/src/subagent-definition.ts#L177-L211) (verified)
  - *To reach the next level:* Use tight defaults the model cannot raise.
- **B L1:** No spend ceiling; a runaway loop can continue indefinitely and scheduled tasks keep firing. — searched `rg -n -i 'maxTurns|max_turns|maxSteps|max_steps|maxIterations'` in `packages/agent-runtime/src` → 2 hits (Both hits are test assertions that subagent builtins do NOT carry maxTurns; no loop iteration cap exists.); [crates/host-core/src/tools/mod.rs:40-41](https://github.com/vastsa/pi-desktop/blob/831b66dfa8e819d2079c31b1cd98206818d9ac15/crates/host-core/src/tools/mod.rs#L40-L41) (verified)
  - *To reach the next level:* Add tight per-run time and cost ceilings.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Repository files and auto-loaded AGENTS.md/CLAUDE.md (packages/agent-runtime/src/project-instructions.ts:5), MCP tool results · [B] sensitive data/systems: Workspace files read without approval (crates/host-core/src/permissions.rs:122) and ambient env credentials (packages/host-runtime/src/host-process.ts:104) · [C] state change / egress: An unattended egress path; approval-gated Bash/Write (crates/host-core/src/permissions.rs:123) · Same default session? Yes

## Highest-impact improvements
1. Require a workspace-trust decision before enabling project-supplied capability files. — C6 S L0→L3, +0.225 before caps (Playbook 2)
2. Show the exact MCP server command before first launch. — C7 D L0→L3, +0.150 before caps (Playbook 3)
3. Run Bash and MCP stdio servers inside an OS sandbox (Seatbelt/Landlock) with workspace-only writes and a scrubbed environment. — C4 S L0→L3, +0.225 before caps (Playbook 3)
4. Close the unattended exfiltration channel. — C5 B L1→L2, +0.050 before caps (Playbook 1)
5. Add a session step cap and token/cost budget enforced in the runtime. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Upstream @earendil-works/pi-* packages (agent loop internals) and the patches/ directory were not reviewed beyond how PI-Desktop calls them.
- Plugin permission broker (plugin-runtime.ts, ~5k lines), remote-host (RACP/SSH) mode and live-voice were sampled, not audited in full.
- The native-Pi-session import path uses upstream ProjectTrustStore for project resources; that trust check was not traced and is not the default session path.
- No text aimed at AI reviewers was found in AGENTS.md, CLAUDE.md or README.
