# Defense-in-Depth Score: Zed (Agent panel)

**Repo:** https://github.com/zed-industries/zed · **Commit:** `a84689073d296dfd39987bc7dd478e43ef76d83a` · **Reviewed:** 2026-10-03
**What it is:** High-performance code editor with built-in agentic coding and ACP host
**Category:** Coding
**Scored configuration:** Zed desktop editor, built-in Zed Agent in the Agent Panel on a local project, default settings (Write profile, tool_permissions default confirm, sandboxing feature flag on), fresh install.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions opt-in · sub agents yes · external communication opt-in

## Score: 4.0 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L0 | L2 | 0.17 | — | **0.17** | High |
| C2 | Approval gates | L3 | L3 | L2 | L2 | 0.65 | — | **0.65** | High |
| C3 | Tool & action scoping | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C4 | Code-execution isolation | L3 | L2 | L3 | L0 | 0.53 | — | **0.53** | High |
| C5 | Untrusted input blast radius | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C6 | Memory, context & configuration integrity | L2 | L2 | L3 | L1 | 0.50 | — | **0.50** | High |
| C7 | Third-party extensions | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C8 | Secrets & sensitive-data protection | L2 | L1 | L1 | L0 | 0.28 | — | **0.28** | High |
| C9 | Audit & traceability | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |


Zed's built-in agent asks before every command, edit, fetch and MCP call by default, and runs terminal commands in a genuine OS sandbox with no network and writes confined to the project. The dominant risk is leakage rather than destruction: the chat view is not locked down against unattended data egress, and sandboxed commands still see your full environment and home directory. There are no step, time or spend limits on a turn.

## Critical gaps
- Sandboxed terminal commands inherit the user's full shell environment and can read the whole filesystem (including ~/.ssh and cloud credential files); only writes and network are restricted. (ASI05, T11, LLM05; C4) — [crates/acp_thread/src/terminal.rs:930-944](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/acp_thread/src/terminal.rs#L930-L944); [crates/sandbox/src/macos_seatbelt.rs:231-234](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/sandbox/src/macos_seatbelt.rs#L231-L234)
- Every agent terminal command (sandboxed or not) and every MCP server inherits the user's full environment, so long-lived keys exported there are reachable by the model's commands. (ASI03, LLM02, T9; C8) — [crates/acp_thread/src/terminal.rs:930-944](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/acp_thread/src/terminal.rs#L930-L944); [crates/context_server/src/transport/stdio_transport.rs:37](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/context_server/src/transport/stdio_transport.rs#L37)

## Criterion details

### C1 Identity & least privilege — 0.17 (high)

Zed's agent runs with your own account and does nothing to narrow it. Every terminal command the agent runs inherits your full shell environment (including any API keys or cloud tokens exported there), and MCP servers inherit it too. Zed's own model-provider keys stay in the OS keychain and are not handed to tools. What limits the damage is not a narrower identity but other layers: the default sandbox blocks network and out-of-project writes, and every terminal command needs your approval.

- **S L0:** The agent and its subprocesses act with the user's full ambient authority; terminal commands get the project's full login-shell environment and no credential is narrowed or substituted. — [crates/acp_thread/src/terminal.rs:930-944](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/acp_thread/src/terminal.rs#L930-L944); [crates/context_server/src/transport/stdio_transport.rs:37](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/context_server/src/transport/stdio_transport.rs#L37); searched `rg -n 'env_clear|env_remove'` in `crates/acp_thread/src crates/agent/src crates/context_server/src` → 0 hits (No environment scrubbing anywhere on the agent terminal or MCP launch paths.) (verified)
  - *To reach the next level:* Scrub credential-like variables from subprocess environments or broker per-tool scoped credentials.
- **C L1:** Built-in file tools are confined to project paths, but the terminal and every MCP server receive the full environment. — [crates/acp_thread/src/terminal.rs:930-944](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/acp_thread/src/terminal.rs#L930-L944); [crates/context_server/src/transport/stdio_transport.rs:37](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/context_server/src/transport/stdio_transport.rs#L37) (verified)
  - *To reach the next level:* Pass only scoped credentials to extensions, MCP servers and terminal commands through one authorization layer.
- **D L0:** The default install runs as the user with all ambient credentials available to agent subprocesses; narrowing them requires manual changes. — [crates/acp_thread/src/terminal.rs:930-944](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/acp_thread/src/terminal.rs#L930-L944); [crates/context_server/src/transport/stdio_transport.rs:37](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/context_server/src/transport/stdio_transport.rs#L37) (verified)
  - *To reach the next level:* Ship a narrowed default (scrubbed env, no ambient credentials) that the operator must explicitly widen.
- **B L2:** If the authorization layer fails, the default-on network-off sandbox and per-command approval still stand between a hijacked agent and use of the user's credentials. — [crates/sandbox/src/linux_bubblewrap.rs:320-321](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/sandbox/src/linux_bubblewrap.rs#L320-L321); [assets/settings/default.json:1241-1246](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/assets/settings/default.json#L1241-L1246) (verified)
  - *To reach the next level:* Keep credentials out of agent reach entirely so a hijack is limited to one project, mostly read.
- **Cap:** none

### C2 Approval gates — 0.65 (high)

Approval is on by default: every built-in tool that changes files, runs commands, fetches URLs, searches the web or invokes a skill, and every MCP tool, asks you first. For terminal commands you see the exact command, and allow/deny rules are regular expressions checked against each parsed sub-command of chained shell commands, with command substitution refused unless everything is auto-allowed. File edits are approved by path before the new content is shown, but every agent edit can be reviewed and rejected afterwards and Zed takes git checkpoints you can restore. Auto-approval for everything is a single ordinary setting with no loud name, and nothing undoes MCP or network side effects.

- **S L3:** Per-call approval shows the exact terminal command or file path, with argument-level regex allow/deny/confirm rules and reject as a first-class outcome. — [crates/agent/src/tools/terminal_tool.rs:430-433](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/tools/terminal_tool.rs#L430-L433); [crates/agent/src/tools/tool_permissions.rs:644](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/tools/tool_permissions.rs#L644); [crates/agent/src/thread.rs:6584-6592](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/thread.rs#L6584-L6592) (verified)
  - *To reach the next level:* File-edit approvals cover only the path, so the approved call is not the exact diff that gets written.
- **C L3:** All default-enabled mutating tools and MCP tools go through the settings-driven gate, sub-agents use the same Thread code (depth capped at 1), and terminal chains are parsed so allowlists apply per sub-command. — [crates/agent/src/tool_permissions.rs:261](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/tool_permissions.rs#L261); [crates/agent/src/tool_permissions.rs:336-345](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/tool_permissions.rs#L336-L345); [crates/agent/src/thread.rs:5850-5854](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/thread.rs#L5850-L5854); [crates/agent/src/tools.rs:233-235](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/tools.rs#L233-L235) (verified)
  - *To reach the next level:* Not every flag-gated tool path is guaranteed to cross the gate.
- **D L2:** Confirm is the shipped default and agent settings cannot be set from project files, but setting tool_permissions.default to allow (or clicking 'Always for <tool>') disables prompting through an ordinary user setting without a loud name. — [assets/settings/default.json:1241-1246](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/assets/settings/default.json#L1241-L1246); [crates/settings_content/src/project.rs:44-96](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/settings_content/src/project.rs#L44-L96) (verified)
  - *To reach the next level:* Require an explicitly named, warned operator flag for global auto-approve, and time-bound elevated modes.
- **B L2:** Agent edits are reversible via per-hunk reject and git checkpoints, and terminal writes are sandboxed to the project with .git protected, but MCP actions and approved network calls have no preview or undo. — [crates/acp_thread/src/acp_thread.rs:6487](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/acp_thread/src/acp_thread.rs#L6487); [crates/action_log/src/action_log.rs:924](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/action_log/src/action_log.rs#L924) (verified)
  - *To reach the next level:* Add previews or dry-runs for external actions and rate limits on consequential actions.
- **Cap:** none

### C3 Tool & action scoping — 0.50 (high)

Zed's file tools are well scoped: paths are resolved against the project with symlink-escape checks, and files matching the private-files list (.env, keys, certificates) are refused by read and search. The fetch tool asks for each host, re-checks every redirect and refuses loopback, private and cloud-metadata addresses. But the default 'Write' profile also gives the agent a general shell, delete, and all MCP tools, and the shell accepts any command apart from command substitution; its reach is bounded only by the sandbox.

- **S L2:** File and fetch tools validate in code (resolved-path containment, private-file denylist, per-hop host checks with internal-address blocking), but the default terminal tool accepts arbitrary shell text. — [crates/agent/src/tools/fetch_tool.rs:177-190](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/tools/fetch_tool.rs#L177-L190); [crates/agent/src/tools/fetch_tool.rs:354-358](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/tools/fetch_tool.rs#L354-L358); [crates/agent/src/tools/read_file_tool.rs:311](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/tools/read_file_tool.rs#L311); [crates/agent/src/tools/tool_permissions.rs:391](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/tools/tool_permissions.rs#L391) (verified)
  - *To reach the next level:* Replace or bound the general shell with narrow tools so no default tool is a raw passthrough.
- **C L2:** Most built-in tools validate their arguments; MCP tools have no shared validation layer beyond the approval gate. — [crates/agent/src/tools/grep_tool.rs:145-151](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/tools/grep_tool.rs#L145-L151); [crates/agent/src/tools/context_server_registry.rs:349-350](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/tools/context_server_registry.rs#L349-L350) (verified)
  - *To reach the next level:* Wrap extension tools in a shared validation layer.
- **D L2:** Profiles exist (Write, Ask, Minimal), but the default profile is Write with terminal, delete, fetch and all MCP servers enabled. — [assets/settings/default.json:1291-1319](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/assets/settings/default.json#L1291-L1319); [assets/settings/default.json:1295](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/assets/settings/default.json#L1295) (verified)
  - *To reach the next level:* Default to a read-only profile and require explicit enabling of write and exec tools.
- **B L2:** A misused tool is scoped to the open project for writes (file tools and the sandboxed terminal), with full write inside it. — [crates/sandbox/src/linux_bubblewrap.rs:296-306](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/sandbox/src/linux_bubblewrap.rs#L296-L306) (verified)
  - *To reach the next level:* Bound quantities and make tool operations reversible by construction.
- **Cap:** none

### C4 Code-execution isolation — 0.53 (high)

Agent terminal commands run inside a real OS sandbox by default on macOS (Seatbelt), Linux (bubblewrap with user, PID, IPC and network namespaces plus a seccomp filter that blocks Unix sockets) and Windows via WSL. Writes are limited to the open project and a temporary directory, Git metadata is read-only, and network is off. Leaving the sandbox needs a stated reason and your approval, and if bubblewrap is missing Zed asks rather than silently running on the host. But the sandbox can read your whole home directory and receives your full environment, so credentials are visible to anything the agent runs; MCP servers, language servers and remote (SSH) projects run without it, and sandbox boundary handling for project configuration does not cover every path.

- **S L3:** OS sandbox profiles: bubblewrap with unshared user/pid/ipc/net namespaces and a seccomp socket filter, Seatbelt deny-default; writes limited to worktrees, network denied by default. — [crates/sandbox/src/linux_bubblewrap.rs:308-321](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/sandbox/src/linux_bubblewrap.rs#L308-L321); [crates/sandbox/src/linux_bubblewrap.rs:867-874](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/sandbox/src/linux_bubblewrap.rs#L867-L874); [crates/sandbox/src/macos_seatbelt.rs:231](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/sandbox/src/macos_seatbelt.rs#L231) (verified)
  - *To reach the next level:* Use kernel-separated isolation (microVM/gVisor) or a remote ephemeral sandbox.
- **C L2:** The terminal tool (the only model exec path) and fetch go through the sandbox, with a sandbox-creation failure on Linux/WSL turned into a user prompt; MCP stdio servers, language servers and remote projects run on the host. — [crates/agent/src/tools/terminal_tool.rs:434-435](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/tools/terminal_tool.rs#L434-L435); [crates/agent/src/sandboxing.rs:220-227](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/sandboxing.rs#L220-L227); [crates/agent/src/tools/terminal_tool.rs:798-801](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/tools/terminal_tool.rs#L798-L801); [crates/context_server/src/transport/stdio_transport.rs:37](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/context_server/src/transport/stdio_transport.rs#L37); [crates/agent/src/sandboxing.rs:53](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/sandboxing.rs#L53) (verified)
  - *To reach the next level:* Sandbox MCP stdio servers; remote projects get no sandbox. Apply the sandbox boundary on every path.
- **D L3:** Sandboxing is enabled for all users by a flag that overrides settings, allow_unsandboxed defaults to false and is user-scope only, and every escalation (network, extra paths, unsandboxed) needs a reason and per-call human approval. — [crates/feature_flags/src/flags.rs:107-115](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/feature_flags/src/flags.rs#L107-L115); [crates/agent_settings/src/agent_settings.rs:469](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent_settings/src/agent_settings.rs#L469); [crates/agent/src/tools/terminal_tool.rs:677-691](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/tools/terminal_tool.rs#L677-L691) (verified)
  - *To reach the next level:* Keep the policy fully outside model/workspace reach: 'always' grants and the fallback 'run unsandboxed always' persist without time bounds.
- **B L0:** Inside the sandbox the full parent environment (any exported keys/tokens) is present and the entire filesystem, including ~/.ssh and ~/.aws, is readable. — [crates/acp_thread/src/terminal.rs:930-944](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/acp_thread/src/terminal.rs#L930-L944); [crates/sandbox/src/macos_seatbelt.rs:231-234](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/sandbox/src/macos_seatbelt.rs#L231-L234) (verified)
  - *To reach the next level:* Scrub secrets from the sandbox environment and deny reads of credential directories by default.
- **Cap:** none

### C5 Untrusted input blast radius — 0.45 (high)

Zed does not try to detect prompt injection, but its defaults blunt a hijack: every egress and state-changing tool (terminal, edits, fetch, web search, MCP) needs your approval whatever the agent has read, the terminal sandbox has no network, and read/search refuse .env and key files. The gap is the chat view itself, which is not locked down against unattended data egress. Tool results, files and MCP output all enter the conversation with no provenance marking.

- **S L2:** Egress and state-changing tools require human approval by default regardless of provenance, but the chat view is not locked down against unattended egress. — [assets/settings/default.json:1241-1246](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/assets/settings/default.json#L1241-L1246) (verified)
  - *To reach the next level:* Force every egress path through approval once untrusted content is read.
- **C L2:** The approval and sandbox limits apply to actions whatever the source, but untrusted sources are not distinguished and tool/MCP results enter context with the same standing as the user's text. — [crates/agent/src/thread.rs:746-758](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/thread.rs#L746-L758) (verified)
  - *To reach the next level:* Mark tool, file and MCP results as untrusted data and apply the limit to every channel.
- **D L2:** The limits are on by default and content cannot change agent settings, but the operator can silently switch to auto-approve. — [assets/settings/default.json:1241-1246](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/assets/settings/default.json#L1241-L1246) (verified)
  - *To reach the next level:* Warn loudly when the approval default is changed to allow.
- **B L1:** With default settings a hijacked agent has an unattended exfiltration channel, while irreversible actions still need approval. (verified)
  - *To reach the next level:* Close the unattended exfiltration channel so both leaks and irreversible actions require approval.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.50 (high)

Zed has no long-term memory store, and settings that could add MCP servers or language servers from a repository's .zed/settings.json only load after you explicitly trust that folder; agent permissions and sandbox settings can only be set in your user settings. Project skills also wait for trust, and the agent's edits to .zed/, .agents/skills/ or your config directory always prompt. However, instruction files in the project root (.rules, AGENTS.md, CLAUDE.md and similar) are loaded into the system prompt silently even in untrusted folders, and the agent can rewrite them (after an approved edit or command) to steer future sessions.

- **S L2:** Security-relevant project config needs an explicit worktree-trust decision and agent security settings are user-scope only, but instruction files load silently as system-prompt context. — [crates/agent/src/agent.rs:1319-1336](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/agent.rs#L1319-L1336); [crates/agent/src/agent.rs:1127](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/agent.rs#L1127); [assets/settings/default.json:2711](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/assets/settings/default.json#L2711) (verified)
  - *To reach the next level:* Gate instruction files behind the same workspace-trust decision or show them for review before loading.
- **C L2:** Project settings, MCP/LSP config and project skills are gated; root instruction files are not. — [crates/prompt_store/src/prompts.rs:22-32](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/prompt_store/src/prompts.rs#L22-L32) (verified)
  - *To reach the next level:* Cover instruction files and persisted compaction summaries with the same control.
- **D L3:** State lives in user scope (settings, thread DB in the data dir); edits to settings locations always prompt and sandboxed commands cannot write outside the project. — [crates/agent/src/tools/tool_permissions.rs:755-757](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/tools/tool_permissions.rs#L755-L757); [crates/agent/src/db.rs:444-446](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/db.rs#L444-L446) (verified)
  - *To reach the next level:* Add retention limits on stored threads by default.
- **B L1:** A poisoned AGENTS.md or .rules persists across the user's sessions in that repo and can steer tool use, still subject to approval and sandbox. — [crates/agent/src/agent.rs:1319-1336](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/agent.rs#L1319-L1336) (verified)
  - *To reach the next level:* Require human review before changed instruction files are re-loaded.
- **Cap:** none

### C7 Third-party extensions — 0.30 (high)

No third-party MCP server is enabled by default; you add one in settings (or a trusted project's settings), and its tools still need approval. Nothing pins or verifies MCP server code, and changed tool definitions are not re-approved. Zed extensions come from Zed's registry and auto-update by default, and the default capability grant lets them run any process, download any file and install any npm package. MCP servers launch as ordinary processes with your full environment.

- **S L1:** MCP servers and extensions run from user-chosen sources without pinning or integrity checks, and extensions auto-update. — [crates/extension_host/src/extension_settings.rs:30-34](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/extension_host/src/extension_settings.rs#L30-L34); [crates/context_server/src/transport/stdio_transport.rs:37](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/context_server/src/transport/stdio_transport.rs#L37) (verified)
  - *To reach the next level:* Pin versions and verify integrity (hash/signature) for MCP servers and extension updates.
- **C L1:** No extension type is verified beyond coming from the configured source; only Zed extensions use a curated registry. — [assets/settings/default.json:2345-2348](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/assets/settings/default.json#L2345-L2348) (verified)
  - *To reach the next level:* Apply verification to MCP servers, extension-provided servers and external agents alike.
- **D L2:** Nothing third-party is enabled by default and adding MCP servers is explicit, but a trusted workspace's .zed/settings.json can add servers and the install does not display what will run. — [crates/settings_content/src/project.rs:68-70](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/settings_content/src/project.rs#L68-L70) (verified)
  - *To reach the next level:* Restrict extension/MCP addition to user scope and show the exact command and permissions on add.
- **B L1:** MCP servers run as separate processes of the same user with the full inherited environment; extensions are granted process:exec for any command by default. — [crates/context_server/src/transport/stdio_transport.rs:37](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/context_server/src/transport/stdio_transport.rs#L37); [assets/settings/default.json:2346](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/assets/settings/default.json#L2346) (verified)
  - *To reach the next level:* Scrub the environment and sandbox each extension with scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.28 (high)

Zed keeps its own model-provider keys in the OS keychain, and the agent's read and search tools refuse files on the private-files list (.env, keys, certificates) by default. There is no redaction of tool output sent to the model, of logs, or of saved conversation history, and terminal commands and MCP servers inherit your full environment, so any key exported in your shell is within reach of a command the agent runs. Usage metrics and crash reports are on by default; agent telemetry events carry metadata (model, token counts), not prompts.

- **S L2:** Provider credentials are stored in the OS keychain and secret-named files are excluded from model-bound file reads, but there is no redaction of tool output, logs or transcripts. — [crates/zed_credentials_provider/src/zed_credentials_provider.rs:68-77](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/zed_credentials_provider/src/zed_credentials_provider.rs#L68-L77); [crates/agent/src/tools/read_file_tool.rs:311-313](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/tools/read_file_tool.rs#L311-L313); searched `rg -n -i 'redact|mask_secret|scrub'` in `crates/agent/src crates/acp_thread/src` → 22 hits (18 hits are RedactedThinking model-content handling and 4 are in an eval fixture file; none redact secrets.) (verified)
  - *To reach the next level:* Redact secrets before logs and before model-bound tool output on all major paths.
- **C L1:** Only the file-read/search path is protected; subprocess environments, terminal output to the model, logs and stored threads are not. — [crates/agent/src/tools/grep_tool.rs:145-151](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/tools/grep_tool.rs#L145-L151); searched `rg -n 'env_clear|env_remove'` in `crates/acp_thread/src crates/agent/src crates/context_server/src` → 0 hits (No environment scrubbing anywhere on the agent terminal or MCP launch paths.) (verified)
  - *To reach the next level:* Extend protection to subprocess env, tool output, logs and transcripts.
- **D L1:** Metrics and crash diagnostics are on by default; agent telemetry is content-free metadata. — [assets/settings/default.json:1699-1703](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/assets/settings/default.json#L1699-L1703); [crates/agent/src/thread.rs:2897-2905](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/thread.rs#L2897-L2905) (verified)
  - *To reach the next level:* Make telemetry and crash reporting opt-in.
- **B L0:** Long-lived user keys in the environment are passed to every agent subprocess and are readable inside the sandbox. — [crates/acp_thread/src/terminal.rs:930-944](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/acp_thread/src/terminal.rs#L930-L944) (verified)
  - *To reach the next level:* Keep long-lived keys out of subprocess environments.
- **Cap:** none

### C9 Audit & traceability — 0.38 (high)

Each agent thread, including its tool calls, tool results and sub-agent threads (linked to their parent), is saved to a local database in Zed's data directory, outside the project the agent can write to. The record has no per-call timestamps and does not record your approvals or denials, saves are whole-thread snapshots whose failures are only logged, and there is no tamper evidence or export.

- **S L1:** Thread snapshots store tool-use and tool-result content but only a thread-level updated_at, with no per-call timestamps or actor/approval fields. — [crates/agent/src/thread.rs:744-758](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/thread.rs#L744-L758); [crates/agent/src/db.rs:54-57](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/db.rs#L54-L57) (verified)
  - *To reach the next level:* Record each tool call with timestamps and result status as a structured event.
- **C L2:** Built-in and MCP tool calls are in the thread record and sub-agent threads are saved with a parent link, but approvals and denials are not recorded. — [crates/agent/src/db.rs:29-31](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/db.rs#L29-L31) (verified)
  - *To reach the next level:* Record approval requests and decisions alongside tool calls.
- **D L2:** On by default and written by the Zed process into data_dir/threads, outside the sandbox's writable roots, but the Zed process can rewrite or delete it. — [crates/agent/src/db.rs:444-446](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/db.rs#L444-L446) (verified)
  - *To reach the next level:* Write the record through a component the agent process cannot alter.
- **B L1:** Saves are coalesced snapshots and failures are only logged while the agent continues. — [crates/agent/src/agent.rs:1856-1860](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/agent.rs#L1856-L1860) (verified)
  - *To reach the next level:* Surface write errors and flush a durable record per action.
- **Cap:** none

### C10 Limits & kill switch — 0.25 (high)

There is no limit on how many steps, how long, or how many tokens an agent turn may use. MCP calls time out after 60 seconds by default, model retries are capped, and terminal commands only time out if the model asks for a timeout. Stopping works well: cancel stops the turn, cancels running sub-agents and kills running terminal commands.

- **S L1:** No step, wall-clock or token cap on the agent loop; only per-request MCP timeouts and a retry cap, with a halt that kills in-flight terminal processes. — searched `rg -n -i 'max_turns|max_steps|max_iterations|max_tool_calls|turn_limit'` in `crates/agent/src` → 0 hits (No turn or step limit exists in the agent crate.); [assets/settings/default.json:2966](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/assets/settings/default.json#L2966); [crates/agent/src/thread.rs:2323-2326](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/thread.rs#L2323-L2326); [crates/agent/src/tools/terminal_tool.rs:1006-1010](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/tools/terminal_tool.rs#L1006-L1010) (verified)
  - *To reach the next level:* Add enforced step, wall-clock and token/cost caps.
- **C L1:** The only bounds are per-call (MCP timeout, model-chosen terminal timeout); sub-agent depth is capped at 1 but sub-agents share no budget. — [crates/agent/src/thread.rs:77](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/thread.rs#L77) (verified)
  - *To reach the next level:* Count sub-agents and spawned processes against a shared budget.
- **D L1:** Terminal timeouts are chosen by the model and the loop is unlimited by default. — [crates/agent/src/tools/terminal_tool.rs:59](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/crates/agent/src/tools/terminal_tool.rs#L59) (verified)
  - *To reach the next level:* Ship sensible default ceilings the model cannot raise.
- **B L1:** A runaway turn can loop and spend indefinitely on auto-approved read tools, though acting needs approval and stop kills in-flight terminals. — [assets/settings/default.json:1241-1246](https://github.com/zed-industries/zed/blob/a84689073d296dfd39987bc7dd478e43ef76d83a/assets/settings/default.json#L1241-L1246) (verified)
  - *To reach the next level:* Add tight per-run time and cost ceilings.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Repo files, rules files, fetch/web_search results and MCP output enter context (agent.rs:1326; fetch_tool.rs:177) · [B] sensitive data/systems: Non-private project files via read_file and the full user environment in terminal commands (acp_thread terminal.rs:930-944) · [C] state change / egress: An unattended egress channel; approved terminal/edit/fetch/MCP calls (default.json:1246) · Same default session? Yes

## Highest-impact improvements
1. Close the unattended egress channel in the agent chat view. — C5 B L1→L2, +0.050 before caps (Playbook 1)
2. Scrub credential-like variables from the environment passed to agent terminal commands and MCP servers. — C8 B L0→L1, +0.050 before caps (Playbook 4)
3. Deny sandbox reads of credential directories (~/.ssh, ~/.aws, ~/.config/gcloud) and strip secrets from the sandbox env. — C4 B L0→L2, +0.100 before caps (Playbook 3)
4. Add default step, wall-clock and token caps per agent turn, shared with sub-agents. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)
5. Record per-call timestamps and approval decisions in the persisted thread record. — C9 S L1→L2, +0.075 before caps (Playbook 1 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scope was limited to the built-in Zed Agent (agent, acp_thread, sandbox, settings, context_server, extension_host, agent_ui) via sparse checkout; external ACP agents (Claude Code, Gemini CLI, etc.), Terminal Threads, edit prediction, collaboration and the zed.dev cloud backend were not reviewed.
- Server-delivered feature flags can change tool availability; only the in-repo defaults were scored.
- Zed-hosted model plans may enforce provider-side spend limits; that backend is not in this repository and was not credited.
- Windows/WSL sandbox behaviour was read but not traced in full; the NTFS/DrvFs caveat is documented by Zed itself.
- No reviewer-injection text was found in AGENTS.md/.rules (CLAUDE.md and GEMINI.md are symlinks to .rules).
