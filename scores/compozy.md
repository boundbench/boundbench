# Defense-in-Depth Score: CompozyOS

**Repo:** https://github.com/compozy/compozy · **Commit:** `2bc4e324dee3b81df45cf5374a426458627e4dca` · **Reviewed:** 2026-10-04
**What it is:** Local-first Go daemon that runs and supervises ACP agent CLIs (Claude Code, Codex, OpenClaw, Hermes) with loops, automations, memory, approvals and a web UI.
**Category:** Agent Frameworks
**Scored configuration:** Fresh install with built-in defaults and the onboarding bootstrap (permissions.mode = approve-all, provider env_policy filtered, home_policy operator), local daemon on localhost.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 2.7 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L0 | L0 | 0.07 | C1-SELFESC | **0.07** | High |
| C2 | Approval gates | L3 | L1 | L0 | L1 | 0.35 | G2 | **0.25** | Medium |
| C3 | Tool & action scoping | L2 | L2 | L0 | L0 | 0.30 | — | **0.30** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L1 | L1 | L1 | L0 | 0.20 | C5-WORSTCASE | **0.20** | High |
| C6 | Memory, context & configuration integrity | L0 | L1 | L1 | L1 | 0.17 | C6-REPOCONFIG | **0.17** | High |
| C7 | Third-party extensions | L2 | L1 | L0 | L2 | 0.33 | — | **0.33** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L0 | 0.40 | — | **0.40** | High |
| C9 | Audit & traceability | L3 | L2 | L2 | L2 | 0.57 | — | **0.57** | Medium |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |


CompozyOS ships with every agent permission request and native tool call auto-approved (approve-all is both the code default and what onboarding writes), and its own security policy says the local backend is not an isolation boundary. Agents run as your OS user with your existing CLI logins, so a prompt-injected session can read credentials and push, delete or send without a human. A cloned repo's .compozy/config.toml or mcp.json can also change provider commands, add MCP servers and set the permission mode with no trust prompt. The approval gate, redaction, extension digest checks and audit events are real, but you have to switch to approve-reads and run untrusted work in a separate sandbox to benefit.

## Critical gaps
- The model can author a new agent with an arbitrary command and approve-all permissions via compozy__agent_create, auto-approved in the default mode. (ASI03, T3; C1) — [internal/tools/builtin/workspace.go:115-130](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/tools/builtin/workspace.go#L115-L130); [internal/daemon/native_create_tools.go:145-151](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/daemon/native_create_tools.go#L145-L151); [internal/config/defaults.go:44-46](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/defaults.go#L44-L46)
- Agents run with the operator's full account authority (operator home, ambient credentials) by default. (ASI03, T3; C1) — [internal/config/provider_effective.go:44-49](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/provider_effective.go#L44-L49); [internal/config/defaults.go:44-46](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/defaults.go#L44-L46)
- A workspace's .compozy/config.toml is merged without a trust decision and can set permissions.mode, so repo content can switch approvals off. (ASI09, ASI02; C2) — [internal/config/config_load.go:179-184](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/config_load.go#L179-L184); [internal/config/merge.go:115-118](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/merge.go#L115-L118); [internal/config/workspace_overlay.go:45-58](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/workspace_overlay.go#L45-L58)
- No code-execution isolation: agents and tools run directly on the host as the operator. (ASI05, T11; C4) — [SECURITY.md:16-21](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/SECURITY.md#L16-L21); searched `rg -n -i 'seccomp|landlock|bwrap|firejail|gvisor|firecracker|seatbelt|sandbox-exec'` in `internal` → 0 hits (no OS sandbox primitive anywhere in the daemon)
- Worst case under prompt injection: data exfiltration plus irreversible actions with no human in the loop, because approve-all is the default. (ASI01, LLM01, T6; C5) — [internal/config/defaults.go:44-46](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/defaults.go#L44-L46); [internal/acp/permission.go:173-175](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/acp/permission.go#L173-L175)
- Workspace .compozy/config.toml and mcp.json can redefine provider commands, add MCP servers and change permissions without any trust prompt. (ASI06, T1; C6) — [internal/config/config_load.go:179-184](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/config_load.go#L179-L184); [internal/config/workspace_overlay.go:45-58](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/workspace_overlay.go#L45-L58); [internal/config/merge.go:115-121](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/merge.go#L115-L121)

## Criterion details

### C1 Identity & least privilege — 0.07 (high)

CompozyOS launches agent CLIs as the operator's own OS account, using the operator's existing CLI logins by default, and the agent's shell can reach everything that account can (SSH keys, cloud credentials, gh auth). The only narrowing is a name-based filter that strips secret-looking environment variables from the provider process. There is no per-request authorization layer, and a native compozy__agent_create tool lets the model author a new agent definition with its own command and approve-all permissions, which in the default configuration runs without a human.

- **S L0:** Provider processes run with the operator's home and login state by default; the only narrowing is a denylist of credential-shaped env var names. — [internal/config/provider_effective.go:44-49](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/provider_effective.go#L44-L49); [internal/session/manager_start_env.go:103-106](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/session/manager_start_env.go#L103-L106); [internal/procutil/env.go:62-81](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/procutil/env.go#L62-L81) (verified)
  - *To reach the next level:* Give agents a dedicated, scoped identity instead of the operator's ambient account and CLI logins.
- **C L1:** The env filter applies to provider, MCP and extension launches, but no authorization check sits between the agent and the operator's files or credentials. — [internal/session/manager_start_env.go:103-106](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/session/manager_start_env.go#L103-L106); [internal/mcp/executor_client.go:284-286](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/mcp/executor_client.go#L284-L286); [internal/extension/manager_env_resolution.go:62-70](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/extension/manager_env_resolution.go#L62-L70) (verified)
  - *To reach the next level:* Route every tool path through one authorization layer in code, not just env scrubbing at launch.
- **D L0:** The default config ships approve-all with operator home policy, so the agent acts with the full user account out of the box. — [internal/config/defaults.go:44-46](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/defaults.go#L44-L46); [internal/config/provider_effective.go:44-49](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/provider_effective.go#L44-L49) (verified)
  - *To reach the next level:* Ship a near-minimal default (approve-reads or deny-all, isolated home) that needs explicit operator elevation.
- **B L0:** A hijacked session holds the operator's entire account across services (shell access to ~/.ssh, ~/.aws, provider OAuth state). — [internal/config/provider_effective.go:44-49](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/provider_effective.go#L44-L49); [SECURITY.md:16-21](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/SECURITY.md#L16-L21) (verified)
  - *To reach the next level:* Limit what the agent's identity can reach, for example an isolated provider home and no access to operator credential files.
- **Cap:** C1-SELFESC — compozy__agent_create accepts command, permissions (including approve-all) and ACP options from the model, letting the agent mint a more privileged agent for itself; auto-approved under the default approve-all mode.

### C2 Approval gates — 0.25 (medium)

CompozyOS has a real approval mechanism: in approve-reads or deny-all mode, provider permission requests and native tool calls that are not read-only wait for a human, the approver sees the raw tool input, and an unanswered request is rejected on timeout. But the shipped default and the onboarding bootstrap both set approve-all, which auto-approves every provider permission request and every native tool call. A workspace's .compozy/config.toml can also set the permission mode, and the gate only sees what the provider CLI chooses to forward.

- **S L3:** When enabled, per-call approval shows the exact raw tool input, with risk tiers (read kinds auto-approved) and reject-on-timeout. — [internal/acp/permission.go:176-180](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/acp/permission.go#L176-L180); [internal/acp/permission_event.go:93-97](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/acp/permission_event.go#L93-L97); [internal/acp/handlers.go:264-268](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/acp/handlers.go#L264-L268); [internal/tools/policy.go:335-351](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/tools/policy.go#L335-L351) (verified)
  - *To reach the next level:* Add argument-level allow/deny rules and bind the approved call to exactly the executed call.
- **C L1:** The gate only sees permission requests the provider CLI chooses to send; provider-side allow rules reach shell input without a Compozy approval, and the gate does not cover every tool path. — [internal/acp/permission.go:173-175](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/acp/permission.go#L173-L175) (inferred)
  - *To reach the next level:* Make every consequential path, including provider-internal tool execution, traverse the Compozy gate.
- **D L0:** Default permissions.mode is approve-all and bootstrap writes approve-all, so approval is effectively opt-in; a workspace config file can also set the mode. — [internal/config/defaults.go:44-46](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/defaults.go#L44-L46); [internal/config/bootstrap.go:86-89](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/bootstrap.go#L86-L89); [internal/config/merge.go:115-118](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/merge.go#L115-L118); [internal/config/config_load.go:179-184](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/config_load.go#L179-L184) (verified)
  - *To reach the next level:* Default to approve-reads and accept the permission mode only from user/global scope.
- **B L1:** Auto-approved actions include shell commands, git pushes and external API calls with no checkpoint or rollback. — [internal/tools/builtin/terminal.go:31-45](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/tools/builtin/terminal.go#L31-L45); [internal/acp/permission.go:173-175](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/acp/permission.go#L173-L175) (verified)
  - *To reach the next level:* Add filesystem checkpoints/rollback and previews for external actions.
- **Cap:** G2 — A repo-controlled <workspace>/.compozy/config.toml can set permissions.mode, and the agent can write that file with its own shell.

### C3 Tool & action scoping — 0.30 (high)

Native compozy__* tools have typed JSON schemas, and the ACP file operations resolve symlinks and reject paths outside the session's workspace roots. Agent-facing config edits are checked against a trust-root denylist. But the default tool surface includes general-purpose shell execution (the provider's shell and compozy__terminal_exec) that goes around all of that, and every toolset is enabled by default.

- **S L2:** Typed schemas plus resolved-path containment for ACP fs operations, but a general shell tool is shipped and escapes those checks. — [internal/acp/permission.go:188-214](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/acp/permission.go#L188-L214); [internal/acp/permission.go:168-171](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/acp/permission.go#L168-L171); [internal/config/tool_surface_security.go:86-110](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/tool_surface_security.go#L86-L110); [internal/tools/builtin/terminal.go:31-45](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/tools/builtin/terminal.go#L31-L45) (verified)
  - *To reach the next level:* Replace general shell access with narrow tools, or enforce command allowlists on parsed arguments.
- **C L2:** Most built-in native tools validate against schemas; the provider's own tools are outside Compozy's validation. — [internal/tools/policy.go:335-351](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/tools/policy.go#L335-L351); [internal/acp/permission.go:168-171](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/acp/permission.go#L168-L171) (verified)
  - *To reach the next level:* Wrap every tool, including provider-executed ones, in a shared validation layer.
- **D L0:** Tools are enabled by default, including terminal exec, config mutation and agent creation, with no read-only default toolset. — [internal/config/tools.go:95-98](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/tools.go#L95-L98); [internal/tools/builtin/terminal.go:31-45](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/tools/builtin/terminal.go#L31-L45) (verified)
  - *To reach the next level:* Ship a read-only default toolset and require explicit enabling for write/exec tools.
- **B L0:** A misused shell tool reaches the whole machine as the operator. — [SECURITY.md:16-21](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/SECURITY.md#L16-L21); [internal/tools/builtin/terminal.go:31-45](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/tools/builtin/terminal.go#L31-L45) (verified)
  - *To reach the next level:* Scope tools to the workspace with bounded, reversible operations.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

There is no execution isolation. CompozyOS's own security policy states that the local backend launches the agent and its tool host directly on the daemon host and is not an isolation boundary, and the source contains no container, seccomp, Landlock, Seatbelt or VM sandbox primitive. Shell commands, terminals, worktree setup commands and MCP servers all run as the operator's user with their home directory reachable.

- **S L0:** No isolation primitive; agent CLIs and tool processes are same-user host processes. — searched `rg -n -i 'seccomp|landlock|bwrap|firejail|gvisor|firecracker|seatbelt|sandbox-exec'` in `internal` → 0 hits (no OS sandbox primitive anywhere in the daemon); [SECURITY.md:16-21](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/SECURITY.md#L16-L21) (verified)
  - *To reach the next level:* Run agent processes in at least a hardened OS sandbox (Landlock/Seatbelt or hardened container).
- **C L0:** No execution path is sandboxed. — searched `rg -n -i 'seccomp|landlock|bwrap|firejail|gvisor|firecracker|seatbelt|sandbox-exec'` in `internal` → 0 hits (no OS sandbox primitive anywhere in the daemon); [internal/tools/builtin/terminal.go:31-45](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/tools/builtin/terminal.go#L31-L45) (verified)
  - *To reach the next level:* Sandbox the main execution paths, starting with provider shell and terminal_exec.
- **D L0:** Nothing to enable; isolation does not exist in the default or any configuration. — searched `rg -n -i 'seccomp|landlock|bwrap|firejail|gvisor|firecracker|seatbelt|sandbox-exec'` in `internal` → 0 hits (no OS sandbox primitive anywhere in the daemon) (verified)
  - *To reach the next level:* Ship a sandbox on by default.
- **B L0:** Executed code is host-equivalent: the operator's home directory, credentials and network are all reachable. — [SECURITY.md:16-21](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/SECURITY.md#L16-L21); [internal/config/provider_effective.go:44-49](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/provider_effective.go#L44-L49) (verified)
  - *To reach the next level:* Limit execution to a workspace-only mount with no secrets and restricted egress.
- **Cap:** none

### C5 Untrusted input blast radius — 0.20 (high)

Some native tool and terminal results carry an 'untrusted model data' flag, and tool descriptions tell the model to treat output as untrusted, but nothing in the runtime acts on that flag to restrict tools after untrusted content is read. Agents browse, read repositories and receive webhook payloads, while holding shell, network and the operator's credentials with approvals auto-granted by default. A hijacked session can both exfiltrate data and take irreversible actions with no human involved.

- **S L1:** Results are tagged untrusted_model_data, but no code disables egress or state-changing tools once untrusted content is read. — [internal/tools/result.go:39-40](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/tools/result.go#L39-L40) (verified)
  - *To reach the next level:* Enforce Rule of Two in code: after untrusted content enters a session, force egress and writes through human approval.
- **C L1:** The flag covers native terminal/tool results only; web pages, files, MCP results and webhook payloads read by the provider are not distinguished. — [internal/tools/result.go:39-40](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/tools/result.go#L39-L40) (verified)
  - *To reach the next level:* Tag and handle every untrusted source, including provider tool results and webhook payloads.
- **D L1:** The tagging is always on, but the approval layer it would rely on is off by default and can be switched off by a workspace config file. — [internal/tools/result.go:39-40](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/tools/result.go#L39-L40); [internal/config/merge.go:115-118](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/merge.go#L115-L118) (verified)
  - *To reach the next level:* Make the limit independent of repo-controlled configuration.
- **B L0:** In the default approve-all mode a hijacked agent can read secrets and push, delete or send with no human involved. — [internal/config/defaults.go:44-46](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/defaults.go#L44-L46); [internal/acp/permission.go:173-175](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/acp/permission.go#L173-L175); [internal/tools/builtin/terminal.go:31-45](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/tools/builtin/terminal.go#L31-L45) (verified)
  - *To reach the next level:* Require human approval for egress and irreversible actions in sessions that read untrusted content.
- **Cap:** C5-WORSTCASE — Leak plus irreversible action, unattended, in the default configuration.

### C6 Memory, context & configuration integrity — 0.17 (high)

Compozy's own memory goes through a write controller with validation, and the config_set tool refuses trust-root keys. But any workspace's .compozy/config.toml and .compozy/mcp.json are loaded with no trust decision. They can change the provider launch command and base URL, add MCP servers, change the permission mode and loosen tool policy, and a workspace .env is read for config variable lookup. The agent's shell can write those same files, so one injection persists into every later session in that workspace.

- **S L0:** Repo-controlled workspace config can add MCP servers, change provider commands and set permissions with no prompt. — [internal/config/config_load.go:179-184](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/config_load.go#L179-L184); [internal/config/workspace_overlay.go:45-58](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/workspace_overlay.go#L45-L58); [internal/config/merge.go:115-121](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/merge.go#L115-L121) (verified)
  - *To reach the next level:* Require an explicit workspace-trust decision before loading security-relevant workspace config.
- **C L1:** Memory writes pass through a validating write controller; workspace config, mcp.json, agents and .env are uncontrolled. — [internal/tools/builtin/memory.go:52-65](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/tools/builtin/memory.go#L52-L65); [internal/config/config_load.go:179-184](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/config_load.go#L179-L184); [internal/config/dotenv.go:74-75](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/dotenv.go#L74-L75) (verified)
  - *To reach the next level:* Control all auto-loaded workspace files, not only the memory store.
- **D L1:** Memory is scoped per workspace/global, but scoping is by config, not enforced isolation against the agent's own shell writes. — [internal/tools/builtin/memory.go:52-65](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/tools/builtin/memory.go#L52-L65) (verified)
  - *To reach the next level:* Enforce namespace isolation so the model cannot write other scopes or change isolation.
- **B L1:** Poisoned workspace config persists across the user's sessions in that workspace and changes which commands and tools run. — [internal/config/config_load.go:179-184](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/config_load.go#L179-L184); [internal/config/provider_resolve.go:247](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/provider_resolve.go#L247) (verified)
  - *To reach the next level:* Make poisoned context session-scoped or easy to inspect and purge before it can trigger tools.
- **Cap:** C6-REPOCONFIG — <workspace>/.compozy/config.toml and mcp.json load without a trust decision and can set providers.*.command, mcp servers and permissions.mode.

### C7 Third-party extensions — 0.33 (high)

Marketplace extensions are checked against catalog SHA-256 digests, and unverified side-loads are blocked by policy, but an expected-digest pin may be omitted. External tool sources are disabled by default for Compozy's own tool registry, and extension and Compozy-launched MCP processes get a scrubbed environment. However, MCP servers declared in a workspace's .compozy/mcp.json are passed to provider sessions without consent, unpinned.

- **S L2:** Curated extensions are digest-verified; MCP server commands from config are run as written with no pinning. — [internal/extension/marketplace_trust.go:16-19](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/extension/marketplace_trust.go#L16-L19); [internal/extension/source_changed.go:22-27](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/extension/source_changed.go#L22-L27) (verified)
  - *To reach the next level:* Pin and hash-verify every extension type, including MCP server packages.
- **C L1:** Verification covers marketplace extensions only; MCP servers from config/mcp.json are not verified. — [internal/extension/marketplace_trust.go:16-19](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/extension/marketplace_trust.go#L16-L19); [internal/config/provider_resolve.go:247](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/provider_resolve.go#L247) (verified)
  - *To reach the next level:* Extend verification to MCP servers and other launched tools.
- **D L0:** A workspace mcp.json silently adds MCP servers to sessions; external registry sources stay disabled by default. — [internal/config/config_load.go:179-184](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/config_load.go#L179-L184); [internal/config/provider_resolve.go:247](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/provider_resolve.go#L247); [internal/tools/policy.go:290-301](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/tools/policy.go#L290-L301) (verified)
  - *To reach the next level:* Allow only user/admin scope to add extensions and show the exact command before enabling.
- **B L2:** Extensions and Compozy-launched MCP servers run as separate processes with a scrubbed environment, but as the same OS user. — [internal/extension/manager_env_resolution.go:62-70](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/extension/manager_env_resolution.go#L62-L70); [internal/mcp/executor_client.go:284-286](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/mcp/executor_client.go#L284-L286) (verified)
  - *To reach the next level:* Sandbox each extension with its own scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.40 (high)

Vault secrets are AES-GCM encrypted at rest, but the key sits in a 0600 file beside them that the agent's same-user shell can read. Exact redaction of registered secrets is always on, and heuristic redaction runs by default before events, transcripts and replays are stored. No third-party telemetry was found. However, the agent reaches the operator's long-lived credentials directly through its shell, and nothing redacts what it sends to the model provider.

- **S L2:** Encrypted vault with a locally readable key, plus exact and heuristic redaction on event and log paths. — [internal/vault/ciphertext.go:16-17](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/vault/ciphertext.go#L16-L17); [internal/vault/crypto.go:38-44](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/vault/crypto.go#L38-L44); [internal/daemon/boot_config.go:22](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/daemon/boot_config.go#L22) (verified)
  - *To reach the next level:* Use the OS keychain for the vault key and redact model-bound messages on all major paths.
- **C L2:** Redaction covers the event ledger, transcripts, extension stderr and subprocess env names, but not model-bound messages. — [internal/daemon/boot_config.go:22](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/daemon/boot_config.go#L22); [internal/session/manager_start_env.go:103-106](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/session/manager_start_env.go#L103-L106); [internal/config/defaults.go:97-99](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/defaults.go#L97-L99) (verified)
  - *To reach the next level:* Extend redaction to model-bound messages and error paths.
- **D L2:** No telemetry; redaction on by default, but the heuristic layer is operator-disableable. — [internal/config/defaults.go:56](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/defaults.go#L56); [internal/daemon/boot_config.go:22](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/daemon/boot_config.go#L22) (verified)
  - *To reach the next level:* Make redaction always on and minimise stored transcripts by default.
- **B L0:** Long-lived, high-privilege operator credentials (provider OAuth, SSH, cloud) are reachable by the model's shell. — [internal/config/provider_effective.go:44-49](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/provider_effective.go#L44-L49); [SECURITY.md:16-21](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/SECURITY.md#L16-L21) (verified)
  - *To reach the next level:* Keep long-lived credentials out of reach of agent processes.
- **Cap:** none

### C9 Audit & traceability — 0.57 (medium)

Native tool calls emit structured events with tool ID, risk class, workspace, session, turn, agent name, actor kind, policy decision and correlation ID. The arguments are stored as a digest of the redacted input. Provider permission requests are recorded with the raw input and who resolved them (user, timeout or provider policy). Records live in SQLite under the Compozy home, outside the workspace but writable by the agent's same-user shell, and are not hash-chained.

- **S L3:** Structured events with actor kind, agent, approver and correlation IDs. — [internal/tools/dispatch_events.go:59-81](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/tools/dispatch_events.go#L59-L81); [internal/acp/handlers.go:264-268](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/acp/handlers.go#L264-L268) (verified)
  - *To reach the next level:* Add tamper-evident storage or off-host export.
- **C L2:** Native tools and permission decisions are recorded; provider-internal tool calls that never request permission depend on the provider's ACP updates. — [internal/tools/dispatch_events.go:59-81](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/tools/dispatch_events.go#L59-L81); [internal/tools/dispatch_events.go:81](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/tools/dispatch_events.go#L81) (inferred)
  - *To reach the next level:* Record every provider tool call and configuration change in the same audit stream.
- **D L2:** On by default under the Compozy home, outside the workspace, but the agent's process user can alter it. — [internal/config/defaults.go:97-99](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/defaults.go#L97-L99) (verified)
  - *To reach the next level:* Write audit records from a component the model cannot control.
- **B L2:** Events are appended per action to SQLite stores. — [internal/tools/dispatch_events.go:59-81](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/tools/dispatch_events.go#L59-L81) (inferred)
  - *To reach the next level:* Make records durable per action with a full replayable trajectory, failing closed for high-risk actions.
- **Cap:** none

### C10 Limits & kill switch — 0.40 (high)

Loops ship with an iteration cap of 50 and optional token and wall-clock budgets, goal sessions cap at 20 turns, and stopping a provider kills its whole process group. But interactive sessions have no default timeout, token or cost cap, loop budgets default to unlimited, and the model can raise session.limits.timeout and max_concurrent_agents through config_set. Scheduled automations keep firing independently of any one session.

- **S L2:** Iteration caps plus token/wall-clock budgets in loops and a process-group kill on stop. — [internal/config/loops.go:111-116](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/loops.go#L111-L116); [internal/config/goals.go:10](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/goals.go#L10); [internal/procutil/process_group_unix.go:96](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/procutil/process_group_unix.go#L96) (verified)
  - *To reach the next level:* Add rate limits on side-effecting tools and enforce step, time and cost caps on every session.
- **C L2:** Limits apply to loops and goal sessions; interactive sessions and delegated child sessions are not shown to share a budget. — [internal/config/loops.go:111-116](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/loops.go#L111-L116); [internal/config/defaults.go:14-16](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/defaults.go#L14-L16) (verified)
  - *To reach the next level:* Count sub-sessions, automations and spawned processes against the same budget.
- **D L1:** Token and wall-clock budgets default to 0 (unlimited), session timeout is unset, and the model can raise its own limits via config_set. — [internal/config/loops.go:111-116](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/loops.go#L111-L116); [internal/config/defaults.go:14-16](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/defaults.go#L14-L16); [internal/config/tool_surface.go:76-77](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/tool_surface.go#L76-L77) (verified)
  - *To reach the next level:* Ship sensible default caps that the model cannot raise.
- **B L1:** A runaway interactive session has no spend or time ceiling; up to 20 concurrent agents and scheduled automations keep running. — [internal/config/defaults.go:14-16](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/defaults.go#L14-L16); [internal/config/defaults.go:40-42](https://github.com/compozy/compozy/blob/2bc4e324dee3b81df45cf5374a426458627e4dca/internal/config/defaults.go#L40-L42) (verified)
  - *To reach the next level:* Set tight per-run time and cost ceilings and stop scheduled work on halt.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Provider web/file reads, webhook payloads and terminal output (internal/tools/result.go:40) · [B] sensitive data/systems: Operator home and CLI logins via home_policy operator (internal/config/provider_effective.go:49) · [C] state change / egress: Shell and terminal_exec auto-approved under approve-all (internal/acp/permission.go:174-175) · Same default session? Yes

## Highest-impact improvements
1. Default permissions.mode to approve-reads (code default and bootstrap) and accept it only from global scope. — C2 D L0→L3, +0.150 before caps (Playbook 5)
2. Require an explicit workspace-trust decision before loading <workspace>/.compozy/config.toml, mcp.json, agents and .env. — C6 S L0→L3, +0.225 before caps (Playbook 2)
3. Reject command, permissions and unrestricted ACP modes in model-invoked compozy__agent_create, or force a human approval regardless of mode. — C1 D L0→L2, +0.100 before caps (Playbook 4)
4. Ship a default OS sandbox (Landlock/Seatbelt or container) for provider processes with workspace-only writes. — C4 S L0→L3, +0.225 before caps (Playbook 3)
5. Default a session wall-clock and token budget and remove session.limits.timeout from the agent-mutable config set. — C10 D L1→L3, +0.100 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Behaviour of the provider CLIs themselves (Claude Code, Codex, etc.), including their own permission rules and sandboxes, was not examined; only Compozy's own controls were scored.
- The web UI, desktop app, gateway remote-access paths and TypeScript SDK/extensions were only sampled; the Go daemon (internal/) was the focus.
- C2 coverage, C9 coverage and C9 failure mode are partly inferred, as labelled.
- No text aimed at AI reviewers was found in the repository.
