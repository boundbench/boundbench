# Defense-in-Depth Score: MS-Agent

**Repo:** https://github.com/modelscope/ms-agent · **Commit:** `a56afcc73f0049ba822af9569fd413a2e90677eb` (1.6.0) · **Reviewed:** 2026-10-04
**What it is:** ModelScope's Python agent framework with CLI, TUI and WebUI for tool-using, multi-agent and long-running tasks.
**Category:** Agent Frameworks
**Scored configuration:** README-led WebUI (`ms-agent ui`) and TUI defaults: permission_mode restricted, shipped ms_agent/agent/agent.yaml tools (file_system + local python_env code_executor); the SDK LLMAgent() and `ms-agent run` default to auto mode and are footnoted.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents opt-in · external communication opt-in

## Score: 3.0 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C2 | Approval gates | L2 | L2 | L1 | L1 | 0.40 | G2 | **0.25** | High |
| C3 | Tool & action scoping | L1 | L2 | L1 | L1 | 0.33 | — | **0.33** | High |
| C4 | Code-execution isolation | L2 | L1 | L0 | L2 | 0.33 | G1 | **0.33** (alt) | High |
| C5 | Untrusted input blast radius | L2 | L1 | L1 | L2 | 0.38 | G2 | **0.25** | High |
| C6 | Memory, context & configuration integrity | L0 | L1 | L1 | L1 | 0.17 | C6-REPOCONFIG | **0.17** | High |
| C7 | Third-party extensions | L1 | L1 | L0 | L2 | 0.25 | — | **0.25** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |


MS-Agent ships shell, Python and notebook execution on the host, and its Python tool runs model code inside the agent process itself, with no sandbox by default. The WebUI and TUI ask before most actions, but any project directory can switch that off through its own .ms_agent/config.yaml, permission_memory.json or hooks.json, with no workspace-trust prompt. The SDK and `ms-agent run` default to auto-approve. Treat opening an untrusted repository as running its code.

## Critical gaps
- Model-generated Python executes in-process via exec() by default, with the agent's credentials and the user's full filesystem in reach. (ASI05; C4) — [ms_agent/tools/code/local_code_executor.py:782-786](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/code/local_code_executor.py#L782-L786); [ms_agent/agent/agent.yaml:97-101](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/agent/agent.yaml#L97-L101)
- Workspace files the agent opens (.ms_agent/config.yaml, permission_memory.json, hooks.json) can switch off or bypass the approval gate with no trust decision. (ASI09, ASI02; C2) — [webui/backend/app/backends/ms_agent/config.py:729-739](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/webui/backend/app/backends/ms_agent/config.py#L729-L739); [ms_agent/config/resolver.py:288-291](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/config/resolver.py#L288-L291); [ms_agent/hooks/permission_resolve.py:75-88](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/hooks/permission_resolve.py#L75-L88)
- The same repo-controlled config defeats the only limit on a prompt-injected agent, and the SDK/`ms-agent run` default (auto) leaves exfiltration and irreversible actions unattended. (ASI01, LLM01; C5) — [webui/backend/app/backends/ms_agent/config.py:729-739](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/webui/backend/app/backends/ms_agent/config.py#L729-L739); [ms_agent/permission/config.py:187](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/permission/config.py#L187)
- Repo-controlled .ms_agent/hooks.json, config.yaml and ./.env are loaded with no workspace-trust prompt, enabling shell hooks, MCP servers and auto-approval. (ASI06, ASI04; C6) — [ms_agent/hooks/factory.py:153](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/hooks/factory.py#L153); [ms_agent/config/resolver.py:288-291](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/config/resolver.py#L288-L291); [ms_agent/config/env.py:26](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/config/env.py#L26)

## Criterion details

### C1 Identity & least privilege — 0.25 (high)

MS-Agent runs as the operating-system user and adds no identity or credential scoping of its own. It does narrow what spawned processes inherit: shell commands, MCP stdio servers and hooks get a scrubbed environment without API keys. But the default python_executor tool runs model code inside the agent process itself, where every environment variable, the provider API keys and the user's home-directory credentials are reachable. A repo config file can also add variables back into the shell environment.

- **S L1:** Shell, MCP stdio and hook subprocesses get an allowlisted environment, but the agent otherwise acts with the OS user's full ambient authority and holds long-lived provider keys in process. — [ms_agent/tools/code/local_code_executor.py:324](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/code/local_code_executor.py#L324); [ms_agent/tools/code/local_code_executor.py:416](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/code/local_code_executor.py#L416); [ms_agent/tools/mcp_client.py:95-98](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/mcp_client.py#L95-L98) (verified)
  - *To reach the next level:* No per-tool or per-capability credential scoping; nothing narrows the OS user's authority beyond environment scrubbing for some subprocesses.
- **C L1:** The default python_executor runs exec() in the agent's own process, so it bypasses the environment scrubbing that protects the shell path. — [ms_agent/tools/code/local_code_executor.py:782-786](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/code/local_code_executor.py#L782-L786); [ms_agent/tools/code/local_code_executor.py:782](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/code/local_code_executor.py#L782) (verified)
  - *To reach the next level:* Every execution path (including python_executor and notebook kernels) would need the same scrubbed identity.
- **D L1:** The scrubbed shell environment is the default, but tools.code_executor.shell_env from any config layer, including the repo's .ms_agent/config.yaml, is merged on top. — [ms_agent/tools/code/local_code_executor.py:451-456](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/code/local_code_executor.py#L451-L456); [ms_agent/config/resolver.py:288-291](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/config/resolver.py#L288-L291) (verified)
  - *To reach the next level:* Widening the subprocess environment should require operator scope, not a workspace-controlled config layer.
- **B L1:** A hijacked agent holds the OS user's authority across every system the user can reach (cloud CLIs, git remotes, SSH) plus the provider keys in process; only the per-call approval in restricted mode stands in the way. — [ms_agent/tools/code/local_code_executor.py:782-786](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/code/local_code_executor.py#L782-L786); [ms_agent/cli/tui.py:49](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/cli/tui.py#L49) (verified)
  - *To reach the next level:* Authority would need to be confined to one project/system with mostly read access.
- **Cap:** none

### C2 Approval gates — 0.25 (high)

The WebUI and TUI default to a 'restricted' mode that asks a human before any tool outside a small whitelist (read_file, grep, glob, todo, memory, skills), with options to allow once, for the session, always, edit the arguments, or deny. The SDK constructor and `ms-agent run` default to 'auto', which approves everything except a few network commands. The approval prompts in the terminal truncate long arguments, so a long script can be partly hidden. A cloned repository can turn approval off: its .ms_agent/config.yaml can set permission.mode to auto, its .ms_agent/permission_memory.json can pre-seed always-allow rules, and its .ms_agent/hooks.json can register a PreToolUse hook whose 'allow' skips the approval step entirely. No checkpoints are taken by default.

- **S L2:** Per-call approval with allow/session/always/edit/deny outcomes, but the TUI and CLI prompts truncate the arguments (8 lines/400 chars; 500 chars), so the approver may not see the exact call. — [ms_agent/tui/permission.py:112-115](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tui/permission.py#L112-L115); [ms_agent/permission/handler.py:88-90](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/permission/handler.py#L88-L90); [ms_agent/permission/enforcer.py:207-217](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/permission/enforcer.py#L207-L217) (verified)
  - *To reach the next level:* Show the full exact call (complete command, code, diff) for every approval.
- **C L2:** All tools including MCP pass the single gate in ToolManager.single_call_tool, but the WebUI auto-approves the memory write tool, and matching of network ask rules does not cover every path. — [ms_agent/tools/tool_manager.py:644](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/tool_manager.py#L644); [ms_agent/tools/tool_manager.py:705](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/tool_manager.py#L705); [webui/backend/app/backends/ms_agent/config.py:720](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/webui/backend/app/backends/ms_agent/config.py#L720) (verified)
  - *To reach the next level:* Auto-approved tools would need to be a verified read-only allowlist and ask rules would need robust matching.
- **D L1:** Restricted is the WebUI/TUI default, but workspace files the agent operates on (.ms_agent/config.yaml permission.mode, .ms_agent/permission_memory.json, .ms_agent/hooks.json PreToolUse allow) switch it to auto-approve with no trust prompt; the SDK default is auto. — [webui/backend/app/schemas/project.py:43](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/webui/backend/app/schemas/project.py#L43); [ms_agent/cli/tui.py:49](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/cli/tui.py#L49); [webui/backend/app/backends/ms_agent/config.py:729-739](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/webui/backend/app/backends/ms_agent/config.py#L729-L739); [ms_agent/config/resolver.py:288-291](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/config/resolver.py#L288-L291); [ms_agent/permission/memory.py:42-43](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/permission/memory.py#L42-L43); [ms_agent/hooks/permission_resolve.py:75-88](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/hooks/permission_resolve.py#L75-L88); [ms_agent/permission/config.py:187](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/permission/config.py#L187) (verified)
  - *To reach the next level:* Approval settings and persisted allow-rules would need to come only from user scope, never from the repo.
- **B L1:** Approved or bypassed shell/python calls can delete files, push code or send data anywhere with no undo; automatic snapshots are disabled by default. — [ms_agent/agent/llm_agent.py:216](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/agent/llm_agent.py#L216); [ms_agent/tools/code/local_code_executor.py:782-786](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/code/local_code_executor.py#L782-L786) (verified)
  - *To reach the next level:* Default checkpoints or rollback for filesystem state.
- **Cap:** G2 — A repo-controlled .ms_agent/config.yaml, permission_memory.json or hooks.json PreToolUse hook disables or bypasses the approval gate at runtime with no trust decision.

### C3 Tool & action scoping — 0.33 (high)

The file tools check resolved paths against the workspace and a list of sensitive paths, and shell commands go through a parser that extracts paths from known commands and blocks writes outside the workspace. That shell check lets unknown commands through, and inline interpreter code cannot be inspected. The python_executor and notebook tools take arbitrary code with no argument validation at all. The default tool set enables write, shell, Python and notebook execution together.

- **S L1:** File tools use resolved-path containment, but the shell validator is heuristic (unregistered commands allowed) and python_executor/notebook_executor are raw code passthrough. — [ms_agent/permission/path_validator.py:164-166](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/permission/path_validator.py#L164-L166); [ms_agent/permission/shell_validator.py:423](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/permission/shell_validator.py#L423); [ms_agent/permission/safety.py:86](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/permission/safety.py#L86) (verified)
  - *To reach the next level:* Replace general exec tools with narrow ones or validate their effects with allowlists in code.
- **C L2:** SafetyGuard validates shell, write/edit/read/grep/glob; python_executor, notebook_executor and every MCP tool fall through to allow. — [ms_agent/permission/safety.py:61-86](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/permission/safety.py#L61-L86) (verified)
  - *To reach the next level:* A shared validation layer covering exec and extension tools.
- **D L1:** The shipped agent.yaml enables write_file, edit_file, shell_executor, python_executor and notebook_executor by default; each can be excluded via config. — [ms_agent/agent/agent.yaml:97-101](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/agent/agent.yaml#L97-L101); [ms_agent/agent/agent.yaml:82-101](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/agent/agent.yaml#L82-L101) (verified)
  - *To reach the next level:* Read-only default tool set with write/exec requiring explicit enabling.
- **B L1:** python_executor and shell reach the whole machine as the OS user; the only independent layer that holds is the per-call approval in restricted mode. — [ms_agent/tools/code/local_code_executor.py:782-786](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/code/local_code_executor.py#L782-L786) (verified)
  - *To reach the next level:* Tools scoped to the workspace with quantity bounds.
- **Cap:** none

### C4 Code-execution isolation — 0.33 (high)

The shipped default runs model code with no isolation at all. Shell commands run as the user on the host (with a cleaned environment), and python_executor calls exec() inside the agent's own process, with full access to its memory, environment variables and API keys. A Docker sandbox backend (ms-enclave) exists, but it is opt-in. It is also a stock container with networking on and the workspace mounted read-write, and hooks and MCP servers still run on the host. If the approval step is bypassed, model code effectively has host-level access.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Default implementation python_env: same-user subprocess for shell and in-process exec() for Python. — [ms_agent/agent/agent.yaml:97-101](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/agent/agent.yaml#L97-L101); [ms_agent/tools/code/local_code_executor.py:782-786](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/code/local_code_executor.py#L782-L786) (verified)
    - *To reach the next level:* Any OS-level separation (container, low-privilege user, sandbox profile) in the default path.
  - **C L0:** No execution path is sandboxed by default; hooks and MCP stdio servers also launch on the host. — [ms_agent/tools/code/local_code_executor.py:782-786](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/code/local_code_executor.py#L782-L786); [ms_agent/tools/code/local_code_executor.py:940-946](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/code/local_code_executor.py#L940-L946) (verified)
    - *To reach the next level:* Sandbox the main exec tools.
  - **D L0:** Sandboxing is off in the shipped agent.yaml (implementation: python_env). — [ms_agent/agent/agent.yaml:97-101](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/agent/agent.yaml#L97-L101) (verified)
    - *To reach the next level:* Sandbox on by default.
  - **B L0:** In-process exec gives model code the agent process's credentials, network and the user's whole filesystem. — [ms_agent/tools/code/local_code_executor.py:782-786](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/code/local_code_executor.py#L782-L786); [ms_agent/tools/code/local_code_executor.py:791](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/code/local_code_executor.py#L791) (verified)
    - *To reach the next level:* Workspace-only reach with no secrets and restricted egress.
- **opt-in ms-enclave Docker sandbox (tools.code_executor.implementation: sandbox)** (alt; raw 0.33, cap G1 → 0.33) ← counted
  - **S L2:** Stock Docker container via ms-enclave with privileged false and memory/CPU limits; no evidence of dropped capabilities, non-root or seccomp in this repo. — [ms_agent/tools/code/code_executor.py:153-168](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/code/code_executor.py#L153-L168) (verified)
    - *To reach the next level:* Hardened container (non-root, dropped caps, no-new-privileges, read-only root) or kernel isolation.
  - **C L1:** Only the code_executor tools move into the container; hooks and MCP stdio servers still launch on the host. — [ms_agent/tools/tool_manager.py:207-208](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/tool_manager.py#L207-L208); [ms_agent/tools/mcp_client.py:445-448](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/mcp_client.py#L445-L448) (verified)
    - *To reach the next level:* Every model-reachable execution path in the sandbox.
  - **D L0:** Opt-in; the shipped agent.yaml selects python_env. — [ms_agent/agent/agent.yaml:97-101](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/agent/agent.yaml#L97-L101) (verified)
    - *To reach the next level:* On by default.
  - **B L2:** Workspace mounted read-write at /data and network enabled by default inside the container. — [ms_agent/tools/code/code_executor.py:136](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/code/code_executor.py#L136); [ms_agent/tools/code/code_executor.py:165-166](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/code/code_executor.py#L165-L166) (verified)
    - *To reach the next level:* Egress off or allowlisted by default.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.25 (high)

Nothing tracks where content came from: files, shell output, MCP results and the repo's AGENTS.md all enter the model's context on equal footing, and AGENTS.md goes into the system prompt. What limits a hijacked agent in the default WebUI/TUI mode is the general per-call approval. Shell, Python and file writes need a click, but reads and memory writes do not, so injected text can persist itself into project memory without anyone approving it. The same repo-controlled config files that switch approval off also remove this limit, and in the SDK's default auto mode a hijack can both exfiltrate data and take irreversible actions unattended.

- **S L2:** In restricted mode exec, write and network-capable tools ask a human regardless of what was read, but memory writes are auto-approved and nothing is tied to provenance. — [webui/backend/app/schemas/project.py:43](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/webui/backend/app/schemas/project.py#L43); [webui/backend/app/backends/ms_agent/config.py:715-721](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/webui/backend/app/backends/ms_agent/config.py#L715-L721) (verified)
  - *To reach the next level:* Disable or force approval on every egress and state-changing tool once untrusted content is read, enforced in code.
- **C L1:** Untrusted sources are not distinguished at all; the repo's AGENTS.md is injected into the system prompt and tool results enter context as normal messages. — [ms_agent/prompting/workspace_files.py:9](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/prompting/workspace_files.py#L9); searched `rg -n -i 'taint|untrusted' --type py` in `ms_agent` → 1 hits (The single hit is a docstring about trust_remote_code, not provenance tracking.) (verified)
  - *To reach the next level:* Distinguish tool, MCP and file content as untrusted across all sources.
- **D L1:** The limiting gate is on in WebUI/TUI but a workspace .ms_agent/config.yaml can set permission.mode auto. — [webui/backend/app/backends/ms_agent/config.py:729-739](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/webui/backend/app/backends/ms_agent/config.py#L729-L739); [ms_agent/config/resolver.py:288-291](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/config/resolver.py#L288-L291) (verified)
  - *To reach the next level:* Nothing the agent reads (including workspace config) can configure the limit away.
- **B L2:** In restricted mode exfiltration and irreversible actions need approval; unattended, a hijack can read workspace files and write persistent project memory. — [webui/backend/app/backends/ms_agent/config.py:720](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/webui/backend/app/backends/ms_agent/config.py#L720); [ms_agent/tools/tool_manager.py:705](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/tool_manager.py#L705) (verified)
  - *To reach the next level:* Make the only unattended effects low-sensitivity and non-persistent.
- **Cap:** G2 — A repo-controlled .ms_agent/config.yaml switches permission mode to auto, removing the approval that is the only limit on a hijacked agent.

### C6 Memory, context & configuration integrity — 0.17 (high)

A project directory controls a lot of the agent's behavior with no workspace-trust prompt. Its .ms_agent/config.yaml can change any setting, including tools, MCP servers and permission mode. Its .ms_agent/hooks.json registers shell-command hooks that run automatically, its permission_memory.json holds always-allow rules, its AGENTS.md is injected into the system prompt, and the CLI loads ./.env from the current directory. Project memory (MEMORY.md under .ms_agent/memory, on by default in the WebUI) is written by an auto-approved memory tool. A regex scanner rejects a few obvious injection phrases, but the memory is then fed back into later sessions.

- **S L0:** Repo-controlled files add hooks (shell commands), MCP servers and auto-approve rules with no prompt. — [ms_agent/config/resolver.py:288-291](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/config/resolver.py#L288-L291); [ms_agent/hooks/factory.py:153](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/hooks/factory.py#L153); [ms_agent/hooks/factory.py:41](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/hooks/factory.py#L41); [ms_agent/permission/memory.py:42-43](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/permission/memory.py#L42-L43); searched `rg -n -i 'trust.*(workspace|folder|project)|workspace.trust|trusted_projects' --type py` in `ms_agent webui/backend/app` → 0 hits (No workspace-trust mechanism exists.) (verified)
  - *To reach the next level:* Require an explicit workspace-trust decision before loading security-relevant project config.
- **C L1:** Only the memory store has a write-time control (regex denylist); config files, hooks, AGENTS.md and .env are uncontrolled. — [ms_agent/memory/unified/security.py:52](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/memory/unified/security.py#L52) (verified)
  - *To reach the next level:* Control every auto-loaded file and setting.
- **D L1:** Memory is namespaced per project id, but lives inside the project directory where the repo itself can pre-seed it. — [webui/backend/app/backends/ms_agent/config.py:645](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/webui/backend/app/backends/ms_agent/config.py#L645); [webui/backend/app/schemas/project.py:23](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/webui/backend/app/schemas/project.py#L23) (verified)
  - *To reach the next level:* Storage the workspace cannot write, isolated per user.
- **B L1:** Poisoned memory or repo config persists across the user's sessions in that project and can trigger tool use (hooks run commands). — [ms_agent/hooks/factory.py:153](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/hooks/factory.py#L153); [ms_agent/config/env.py:26](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/config/env.py#L26) (verified)
  - *To reach the next level:* Session-scoped, or persistent only after human review.
- **Cap:** C6-REPOCONFIG — Workspace .ms_agent/config.yaml, hooks.json, permission_memory.json and ./.env enable hooks/MCP servers, loosen approval or change endpoints without a trust decision.

### C7 Third-party extensions — 0.25 (high)

Extensions come from MCP servers, plugins, hooks and skills. Plugins installed from GitHub are pinned to a resolved commit, but MCP servers launch whatever command the config names with no pinning or integrity check. Project-scope plugins.json and mcp.json inside the workspace are picked up without consent. Loading Python tool plugins named in a config does require trust_remote_code. MCP stdio servers and hook commands run as separate processes with a scrubbed environment.

- **S L1:** Plugin installs verify a resolved git SHA, but MCP servers are launched from user/project config with no pinning or integrity check. — [ms_agent/plugins/installer.py:582](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/plugins/installer.py#L582); searched `rg -n -i 'sha256|integrity|checksum'` in `ms_agent/tools/mcp_client.py ms_agent/mcp ms_agent/config/mcp_manager.py` → 0 hits (No integrity checks on MCP servers.) (verified)
  - *To reach the next level:* Pin and verify every extension type.
- **C L1:** Only GitHub plugin installs are pinned; MCP servers, hooks and workspace plugins are not verified. — [ms_agent/plugins/installer.py:527-528](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/plugins/installer.py#L527-L528) (verified)
  - *To reach the next level:* Verification across most extension types.
- **D L0:** Workspace .ms_agent/plugins.json and .ms_agent/mcp.json are merged into the running config and enabled plugins load without a prompt. — [ms_agent/config/resolver.py:335-336](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/config/resolver.py#L335-L336); [ms_agent/plugins/config_manager.py:37-38](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/plugins/config_manager.py#L37-L38); [ms_agent/plugins/runtime.py:105-110](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/plugins/runtime.py#L105-L110) (verified)
  - *To reach the next level:* Nothing third-party enabled from the workspace; adding one shows the exact command.
- **B L2:** MCP stdio servers and command hooks run as separate processes with an allowlisted environment; Python tool plugins load in-process but only with trust_remote_code. — [ms_agent/tools/mcp_client.py:95-98](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/mcp_client.py#L95-L98); [ms_agent/hooks/executors/command.py:78-80](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/hooks/executors/command.py#L78-L80); [ms_agent/tools/tool_manager.py:269](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/tool_manager.py#L269) (verified)
  - *To reach the next level:* Per-extension sandboxing with scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.30 (high)

There is no telemetry (the bundled mem0 telemetry is explicitly switched off), and API keys are masked when the WebUI sends them back to the browser. Spawned shells, MCP servers and hooks get environments without secrets. Provider keys are kept in plaintext settings, and session transcripts are written with no redaction. The in-process python_executor can read every key the agent holds and print it into the conversation.

- **S L1:** Secrets come from env vars and plaintext settings.json; masking exists only in the WebUI API response path and env scrubbing for subprocesses. — [webui/backend/app/backends/ms_agent/mapping.py:42](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/webui/backend/app/backends/ms_agent/mapping.py#L42); [ms_agent/tools/code/local_code_executor.py:416](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/code/local_code_executor.py#L416) (verified)
  - *To reach the next level:* Type-level masking and log filters on main paths.
- **C L1:** Subprocess environments are protected; logs, transcripts and model-bound tool output are not redacted. — searched `rg -n 'redact|mask'` in `ms_agent/session/session_log.py` → 0 hits (No redaction in the session transcript writer.) (verified)
  - *To reach the next level:* Redact secrets in logs and transcripts.
- **D L2:** No telemetry SDK; mem0 telemetry forced off; logging defaults reasonable. — [webui/backend/app/backends/ms_agent/config.py:34](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/webui/backend/app/backends/ms_agent/config.py#L34) (verified)
  - *To reach the next level:* Always-on redaction.
- **B L1:** Long-lived provider and search API keys sit in the agent process, reachable by python_executor. — [ms_agent/tools/code/local_code_executor.py:782-786](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/code/local_code_executor.py#L782-L786) (verified)
  - *To reach the next level:* Scoped, short-lived keys.
- **Cap:** none

### C9 Audit & traceability — 0.45 (high)

Each session is written as an append-only JSONL log under ~/.ms_agent/projects, outside the workspace, flushed line by line. It records every message, including tool calls and results, and the WebUI also records each approval or denial with its arguments. Records carry no actor identity beyond role. The agent's own process (and approved shell commands) can still edit the log, and a failed permission write is swallowed silently.

- **S L2:** Structured JSONL record of every message with seq and timestamps, plus permission markers. — [ms_agent/session/session_log.py:543-545](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/session/session_log.py#L543-L545); [ms_agent/session/session_log.py:157-164](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/session/session_log.py#L157-L164) (verified)
  - *To reach the next level:* Actor attribution (requesting principal, approver) and cross-agent correlation IDs.
- **C L2:** All tool calls on the main agent and WebUI approvals are recorded; sub-agent and hook activity are not shown to reach the same log. — [webui/backend/app/backends/ms_agent/runtime.py:289-303](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/webui/backend/app/backends/ms_agent/runtime.py#L289-L303) (verified)
  - *To reach the next level:* Include sub-agents, hooks and denials on every surface.
- **D L2:** On by default in WebUI/TUI and stored outside the workspace, but in a directory the agent process can write. — [ms_agent/project/paths.py:12](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/project/paths.py#L12); [ms_agent/tui/app.py:160](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tui/app.py#L160) (verified)
  - *To reach the next level:* Written by a component the model cannot control.
- **B L1:** Flushed per line, but persistence failures are swallowed (logger.debug) and actions proceed. — [webui/backend/app/backends/ms_agent/runtime.py:304-305](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/webui/backend/app/backends/ms_agent/runtime.py#L304-L305) (verified)
  - *To reach the next level:* Surface logging errors.
- **Cap:** none

### C10 Limits & kill switch — 0.40 (high)

The agent loop stops after a round cap, but the interfaces raise it to 1000 rounds (9999 in the shipped agent.yaml). Each tool call is bounded by a 120-second default timeout, and the model itself can raise that to 600 seconds. There is no token, cost or wall-clock budget. A python_executor timeout stops waiting but leaves the thread running. Killing a background shell task only marks it killed, because the kill code does not handle asyncio subprocesses.

- **S L2:** Iteration cap plus enforced per-tool timeouts. — [ms_agent/agent/llm_agent.py:2901](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/agent/llm_agent.py#L2901); [ms_agent/tools/tool_manager.py:43-45](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/tool_manager.py#L43-L45); searched `rg -n 'max_cost|cost_limit|token_budget|budget_usd|wall_clock'` in `ms_agent/agent ms_agent/tools/tool_manager.py` → 0 hits (No cost or wall-clock budget.) (verified)
  - *To reach the next level:* Token/cost and wall-clock caps.
- **C L2:** Top-level loop and every tool call through ToolManager are bounded; delegated agents get their own fresh round budget. — [ms_agent/capabilities/wrappers/agent_delegate.py:252](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/capabilities/wrappers/agent_delegate.py#L252) (verified)
  - *To reach the next level:* Sub-agents and background tasks counting against the parent budget.
- **D L1:** Defaults are very large (1000 rounds in TUI/WebUI, 9999 in agent.yaml) and the model can raise per-call timeouts to the 600s ceiling. — [ms_agent/tui/app.py:167](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tui/app.py#L167); [ms_agent/agent/agent.yaml:75](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/agent/agent.yaml#L75) (verified)
  - *To reach the next level:* Sensible defaults the model cannot raise.
- **B L1:** Stopping leaves work running: TaskManager.kill only terminates mp.Process/futures, not asyncio subprocesses, and timed-out exec threads keep running. — [ms_agent/utils/task_manager.py:89-96](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/utils/task_manager.py#L89-L96); [ms_agent/tools/code/local_code_executor.py:791](https://github.com/modelscope/ms-agent/blob/a56afcc73f0049ba822af9569fd413a2e90677eb/ms_agent/tools/code/local_code_executor.py#L791) (verified)
  - *To reach the next level:* Stop cancels pending calls and kills spawned processes.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: workspace files, AGENTS.md and tool/MCP results (ms_agent/prompting/workspace_files.py:9) · [B] sensitive data/systems: provider API keys and user files reachable from in-process exec (ms_agent/tools/code/local_code_executor.py:786) · [C] state change / egress: shell/python execution and file writes (ms_agent/agent/agent.yaml:97-101) · Same default session? Yes

## Highest-impact improvements
1. Ignore permission, hooks, MCP and plugin settings from workspace .ms_agent/ files (and ./.env) unless the user makes an explicit workspace-trust decision. — C2 D L1→L3, +0.100 before caps (Playbook 2)
2. Run python_executor in a subprocess with the scrubbed environment instead of in-process exec(), and make the sandbox backend the default. — C4 S L0→L2, +0.150 before caps (Playbook 3 step 1)
3. Show the full untruncated call in TUI/CLI approval prompts. — C2 S L2→L3, +0.075 before caps (Playbook 5)
4. Kill asyncio subprocesses (process group) in TaskManager.kill and on stop. — C10 B L1→L2, +0.050 before caps (Playbook 3 step 3)
5. Require approval for memory writes and store project memory outside the workspace. — C6 C L1→L2, +0.075 before caps (Playbook 2)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Shape set to agent_application because the README leads with the WebUI; the SDK/`ms-agent run` default (permission mode auto, config.py:187) is weaker and would lower C2 and C5 further.
- WebUI frontend markdown rendering (possible image-based exfiltration) was not examined; projects/ applications, cron, ACP/A2A and agent_hub sync were not scored.
- Sub-agent permission-handler inheritance and the launch path of project mcp.json servers were not fully traced.
- No text aimed at AI reviewers was found in AGENTS.md, README or docs.
