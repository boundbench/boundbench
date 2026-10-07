# Defense-in-Depth Score: Swarms

**Repo:** https://github.com/kyegomez/swarms · **Commit:** `a87cece8ba8c770faadbb8f5dde198d24f1dab5d` (16.0.1) · **Reviewed:** 2026-10-04
**What it is:** Python framework for building single agents and multi-agent systems (sequential, concurrent, hierarchical, graph workflows) on LiteLLM.
**Category:** Agent Frameworks
**Scored configuration:** Python library defaults: Agent() constructor arguments (max_loops=1, telemetry on), with the first-class autonomous mode (max_loops='auto', selected_tools='all') scored for tool-path criteria.
**Agent surface (default):** code execution opt-in · filesystem write opt-in · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents yes · external communication opt-in

## Score: 1.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C3 | Tool & action scoping | L0 | L0 | L1 | L0 | 0.05 | — | **0.05** | High |
| C4 | Code-execution isolation | L1 | L1 | L1 | L0 | 0.20 | G2 | **0.20** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L0 | L1 | 0.05 | C6-REPOCONFIG | **0.05** | High |
| C7 | Third-party extensions | L1 | L0 | L2 | L1 | 0.23 | — | **0.23** | High |
| C8 | Secrets & sensitive-data protection | L0 | L1 | L0 | L0 | 0.07 | — | **0.07** | High |
| C9 | Audit & traceability | L1 | L1 | L0 | L0 | 0.15 | G1 | **0.15** | High |
| C10 | Limits & kill switch | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | High |


Swarms ships almost no safety controls on its tool path: no approval gate, no sandbox, no argument validation, and no restriction after reading untrusted content. Its autonomous mode gives the model a host shell with the full environment behind only a denylist that is not a strict boundary, plus file tools that accept any absolute path. Telemetry is on by default and sends tasks, outputs and agent configuration to a vendor endpoint, and secret handling and transport security are not locked down either.

## Critical gaps
- Autonomous-mode run_bash executes model-written commands on the host with shell=True and the full environment, guarded only by a substring denylist. (ASI05, T11; C4) — [swarms/structs/autonomous_loop_utils.py:1114-1121](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L1114-L1121); [swarms/structs/autonomous_loop_utils.py:991-995](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L991-L995)
- No authorization layer or scoped identity: a hijacked agent acts with the operator's full ambient authority. (ASI03, T3; C1) — [swarms/structs/autonomous_loop_utils.py:1113-1120](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L1113-L1120); [swarms/tools/mcp_manager.py:1458](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/tools/mcp_manager.py#L1458)
- A hijacked autonomous agent can both exfiltrate secrets and delete files unattended; nothing ties untrusted content to any restriction. (ASI01, LLM01, T6; C5) — [swarms/agents/tool_manager.py:681-682](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/agents/tool_manager.py#L681-L682); [swarms/structs/autonomous_loop_utils.py:969](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L969)
- Importing swarms auto-loads a .env from the working directory or any parent, letting a workspace redirect model endpoints and keys. (ASI06, ASI04; C6) — [swarms/env.py:8](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/env.py#L8); [swarms/utils/loguru_logger.py:7](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/utils/loguru_logger.py#L7)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

Swarms agents run with whatever authority the hosting Python process has. LLM provider keys come from environment variables (including any .env found above the working directory), and the autonomous mode's shell tool and every stdio MCP server inherit the full process environment. There is no per-tool identity, no scoped credential, and no authorization check anywhere on the tool path, so a hijacked agent acts with the operator's full local and cloud authority.

- **S L0:** No dedicated identity: the shell tool runs as the OS user with the inherited environment and the MCP stdio launcher copies all of os.environ. — [swarms/structs/autonomous_loop_utils.py:1113-1120](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L1113-L1120); [swarms/tools/mcp_manager.py:1458](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/tools/mcp_manager.py#L1458) (verified)
  - *To reach the next level:* Use a dedicated, role-scoped identity instead of the operator's ambient credentials.
- **C L0:** No authorization layer exists; registered tools are called directly with model-supplied arguments. — [swarms/tools/base_tool.py:2767](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/tools/base_tool.py#L2767); searched `rg -n -i 'approv|confirm'` in `swarms/agents swarms/tools swarms/structs/agent.py swarms/structs/autonomous_loop_utils.py` → 5 hits (All hits are docstrings or prompt text ('confirm completion', 'Confirmation message'); no approval gate exists on any tool path.) (verified)
  - *To reach the next level:* Route at least the main tool path through an authorization check in code.
- **D L0:** Default install inherits all of the operator's environment and any .env credentials; narrowing requires the developer to build it. — [swarms/env.py:8](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/env.py#L8); [swarms/tools/mcp_manager.py:1458](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/tools/mcp_manager.py#L1458) (verified)
  - *To reach the next level:* Ship a minimal default role and scrub the environment passed to tools.
- **B L0:** With bash on the host as the user and the full environment, a hijack reaches the user's entire account (SSH keys, cloud CLIs, tokens). — [swarms/structs/autonomous_loop_utils.py:1114-1121](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L1114-L1121); [swarms/structs/autonomous_loop_utils.py:1113-1120](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L1113-L1120) (verified)
  - *To reach the next level:* Bound what the agent's credentials can reach to one system or project.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

There is no human approval step anywhere in the framework. The tool executor calls registered Python functions directly, the autonomous mode runs shell commands, writes and deletes files, and spawns sub-agents without asking anyone, and MCP tool calls go straight to the server. Interactive mode only asks the human for the next task, not for permission to act.

- **S L0:** No approval mechanism exists; tool calls go straight from model output to func(**arguments). — [swarms/tools/base_tool.py:2767](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/tools/base_tool.py#L2767); searched `rg -n -i 'approv|confirm'` in `swarms/agents swarms/tools swarms/structs/agent.py swarms/structs/autonomous_loop_utils.py` → 5 hits (All hits are docstrings or prompt text ('confirm completion', 'Confirmation message'); no approval gate exists on any tool path.) (verified)
  - *To reach the next level:* Add per-call human approval showing the exact call for consequential tools.
- **C L0:** The most powerful path (run_bash with shell=True) is ungated, as are delete_file and all MCP tools. — [swarms/agents/autonomous_loop.py:408](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/agents/autonomous_loop.py#L408); [swarms/structs/autonomous_loop_utils.py:1114-1121](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L1114-L1121) (verified)
  - *To reach the next level:* Make every consequential tool path cross a gate.
- **D L0:** Nothing to turn on; autonomous mode enables all built-in tools (selected_tools='all') with no gate. — [swarms/structs/agent.py:406](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/agent.py#L406) (verified)
  - *To reach the next level:* Ship approval on by default for write/exec tools.
- **B L0:** Ungated actions include irreversible os.remove on absolute paths and arbitrary shell commands, with no checkpoint or undo. — [swarms/structs/autonomous_loop_utils.py:969](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L969); [swarms/structs/autonomous_loop_utils.py:715-719](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L715-L719) (verified)
  - *To reach the next level:* Add checkpoints/rollback for file and code state.
- **Cap:** none

### C3 Tool & action scoping — 0.05 (high)

Built-in tools are general-purpose and unbounded. The autonomous mode's file tools accept absolute paths and simply use them, so create, update, read and delete reach anywhere the process can, and run_bash accepts an arbitrary shell string. The framework passes model arguments to user tools without validating them against the schema. Developers can narrow the autonomous tool set with selected_tools, but the default is every tool.

- **S L0:** Raw passthrough: arbitrary shell strings and absolute file paths are accepted without containment. — [swarms/structs/autonomous_loop_utils.py:715-719](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L715-L719); [swarms/structs/autonomous_loop_utils.py:1114-1121](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L1114-L1121) (verified)
  - *To reach the next level:* Validate paths with resolved-path containment and replace the shell with narrow tools.
- **C L0:** No tool validates inputs; the executor calls func(**arguments) directly. — [swarms/tools/base_tool.py:2767](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/tools/base_tool.py#L2767) (verified)
  - *To reach the next level:* Add a shared validation layer that every tool passes through.
- **D L1:** Autonomous mode enables all tools including run_bash and delete_file by default; selected_tools can disable them individually. — [swarms/structs/agent.py:406](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/agent.py#L406); [swarms/agents/autonomous_loop.py:408](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/agents/autonomous_loop.py#L408) (verified)
  - *To reach the next level:* Default to a read-only tool set with write/exec explicitly enabled.
- **B L0:** A misused tool reaches the whole machine: any command, any path the user can write. — [swarms/structs/autonomous_loop_utils.py:715-719](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L715-L719); [swarms/structs/autonomous_loop_utils.py:1114-1121](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L1114-L1121) (verified)
  - *To reach the next level:* Scope tools to the workspace with bounded operations.
- **Cap:** none

### C4 Code-execution isolation — 0.20 (high)

Model-written shell commands in autonomous mode run directly on the host with shell=True, in the process's working directory, with the full inherited environment. The only control is a substring denylist that is not a strict boundary. No container or OS sandbox exists anywhere in the framework, and stdio MCP servers also launch on the host.

- **S L1:** Filtering only: a substring/regex denylist checked before subprocess.run(shell=True) on the host. — [swarms/structs/autonomous_loop_utils.py:991-995](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L991-L995); [swarms/structs/autonomous_loop_utils.py:1114-1121](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L1114-L1121); searched `rg -n -i 'docker|sandbox|seccomp|firejail|nsjail|e2b'` in `swarms` → 2 hits (Hits are a logger name (wandb.docker.auth) and prompt text; no isolation backend exists.) (verified)
  - *To reach the next level:* Run commands in an OS-level boundary (container, dedicated user, or sandbox profile).
- **C L1:** The denylist covers run_bash only; stdio MCP servers and user-registered tools run on the host unfiltered. — [swarms/tools/mcp_manager.py:1458](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/tools/mcp_manager.py#L1458); [swarms/tools/base_tool.py:2767](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/tools/base_tool.py#L2767) (verified)
  - *To reach the next level:* Send every model-reachable execution path through the same boundary.
- **D L1:** The denylist is on whenever run_bash is offered, but it can be defeated at runtime. — [swarms/structs/autonomous_loop_utils.py:991-995](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L991-L995) (verified)
  - *To reach the next level:* Make the control something the model cannot route around.
- **B L0:** Host-equivalent: commands run as the user on the host with all environment credentials and unrestricted network. — [swarms/structs/autonomous_loop_utils.py:1114-1121](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L1114-L1121); [swarms/structs/autonomous_loop_utils.py:1113-1120](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L1113-L1120) (verified)
  - *To reach the next level:* Remove credentials from the execution environment and restrict filesystem and network.
- **Cap:** G2 — The model chooses the command text and the denylist (the only control) is defeatable at runtime.

### C5 Untrusted input blast radius — 0.00 (high)

Tool results, MCP results and other agents' messages are appended to the conversation and sent back to the model with no marking or restriction. Nothing in the framework distinguishes untrusted content or limits what the agent can do after reading it. In autonomous mode a single session combines reading files and tool output, holding the user's credentials, and running shell commands with network access, so an injected instruction can both exfiltrate secrets and take destructive action with no human involved.

- **S L0:** Nothing structurally limits a hijacked agent; there is no taint or approval tied to untrusted content. — [swarms/agents/tool_manager.py:681-682](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/agents/tool_manager.py#L681-L682); searched `rg -n -i 'untrusted|prompt.injection|provenance|taint'` in `swarms/agents swarms/tools swarms/structs` → 4 hits (Hits are prompt prose ('uncertainties', 'consensus'); nothing distinguishes untrusted content.) (verified)
  - *To reach the next level:* Gate egress and state-changing tools once untrusted content enters the session.
- **C L0:** Tool and MCP results enter the conversation as ordinary messages with the same standing as the user's task. — [swarms/agents/tool_manager.py:681-682](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/agents/tool_manager.py#L681-L682) (verified)
  - *To reach the next level:* Distinguish untrusted sources, including tool results and peer-agent messages.
- **D L0:** No control exists to be on by default. — searched `rg -n -i 'untrusted|prompt.injection|provenance|taint'` in `swarms/agents swarms/tools swarms/structs` → 4 hits (Hits are prompt prose ('uncertainties', 'consensus'); nothing distinguishes untrusted content.) (verified)
  - *To reach the next level:* Ship an untrusted-content restriction on by default.
- **B L0:** Hijacked autonomous agent can exfiltrate (shell network access) and delete/modify files unattended. — [swarms/structs/autonomous_loop_utils.py:1114-1121](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L1114-L1121); [swarms/structs/autonomous_loop_utils.py:969](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L969) (verified)
  - *To reach the next level:* Require approval for exfiltration and irreversible actions.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.05 (high)

Importing swarms searches upward from the current working directory for a .env file and loads it into the process environment, so a cloned or untrusted project directory can set provider API base URLs and keys (read by LiteLLM), turn telemetry on or off, or move the workspace. Optional persistent memory (off by default) appends every message, including tool output, to MEMORY.md and re-injects it as a System message in later sessions, keyed only by agent name with no validation or per-user namespace.

- **S L0:** Repo-controlled .env is loaded silently at import; when memory is on, all messages are written and re-injected as System context. — [swarms/env.py:8](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/env.py#L8); [swarms/structs/conversation.py:277-282](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/conversation.py#L277-L282) (verified)
  - *To reach the next level:* Validate or gate memory writes and stop loading security-relevant settings from the working directory.
- **C L0:** Neither the .env path nor the MEMORY.md path is controlled. — [swarms/utils/loguru_logger.py:7](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/utils/loguru_logger.py#L7); [swarms/structs/conversation.py:277-282](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/conversation.py#L277-L282) (verified)
  - *To reach the next level:* Control at least the main memory store.
- **D L0:** MEMORY.md is keyed by agent_name under a shared workspace (default name swarm-worker-01), with no per-user namespace. — [swarms/structs/agent.py:769](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/agent.py#L769) (verified)
  - *To reach the next level:* Namespace memory per user/session by default.
- **B L1:** Poisoned memory persists across the user's sessions as System context and can drive tool use; memory itself is opt-in. — [swarms/structs/agent.py:408](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/agent.py#L408); [swarms/structs/conversation.py:277-282](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/conversation.py#L277-L282) (verified)
  - *To reach the next level:* Limit poisoned memory to text output or gated actions.
- **Cap:** C6-REPOCONFIG — A .env in the working directory or any parent is auto-loaded at import (swarms/env.py:8), letting the workspace redirect the model API base URL and keys; LiteLLM reads provider base URLs from the environment (inferred library behaviour).

### C7 Third-party extensions — 0.23 (high)

Third-party code enters mainly through MCP servers the developer configures. Nothing is enabled by default, but there is no version pinning, integrity check or re-approval when a server's tools change, and stdio servers are launched as the same user with a full copy of the process environment, including every API key. Marketplace prompts can be pulled by ID at construction time without verification.

- **S L1:** User-chosen MCP commands/URLs run as given, unpinned and unverified. — [swarms/tools/mcp_manager.py:1464-1466](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/tools/mcp_manager.py#L1464-L1466) (verified)
  - *To reach the next level:* Pin extension versions.
- **C L0:** No extension type (MCP stdio, MCP HTTP, marketplace prompts) is verified. — [swarms/tools/mcp_manager.py:1458](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/tools/mcp_manager.py#L1458) (verified)
  - *To reach the next level:* Verify at least one extension type.
- **D L2:** No MCP server or marketplace prompt is enabled by default; developers add them explicitly in code, but the framework never shows what will run. — [swarms/structs/agent.py:541](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/agent.py#L541) (verified)
  - *To reach the next level:* Show the exact command and permissions when an extension is added.
- **B L1:** Stdio servers run as a separate process, same user, with the full environment. — [swarms/tools/mcp_manager.py:1458](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/tools/mcp_manager.py#L1458) (verified)
  - *To reach the next level:* Scrub the environment passed to extension processes.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.07 (high)

Telemetry is on by default and ships each Agent's constructor configuration plus each run's task and output to a vendor endpoint on railway.app. Secret handling on the telemetry, error-logging and model-provider transport paths is not locked down either, and shell and MCP subprocesses inherit every key. Only the MCP manager's own summary and its OAuth token cache are protected.

- **S L0:** No secret masking exists on logging or telemetry paths, and secret handling and transport security are not locked down. (verified)
  - *To reach the next level:* Mask secrets on logging/telemetry paths and harden transport defaults.
- **C L1:** Only MCPManager.to_dict redacts secrets and the OAuth token cache is written 0600; Agent.to_dict, telemetry, error logs and subprocess env are unprotected. — [swarms/tools/mcp_manager.py:765](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/tools/mcp_manager.py#L765); [swarms/structs/agent.py:1467](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/agent.py#L1467); searched `rg -n -i 'redact|SecretStr|mask_secret'` in `swarms/structs swarms/telemetry swarms/utils swarms/agents` → 0 hits (No redaction helper in the agent, telemetry, logging or utils paths.) (verified)
  - *To reach the next level:* Protect logs and transcripts as well.
- **D L0:** Telemetry to a third party is on unless SWARMS_TELEMETRY_ON is set off, and it carries prompts, outputs and config. — [swarms/telemetry/otel.py:65-66](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/telemetry/otel.py#L65-L66); [swarms/telemetry/otel.py:29-31](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/telemetry/otel.py#L29-L31); [swarms/structs/agent.py:2578](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/agent.py#L2578); [swarms/structs/agent.py:684](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/agent.py#L684); [swarms/telemetry/otel.py:537](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/telemetry/otel.py#L537) (verified)
  - *To reach the next level:* Make telemetry opt-in and content-free.
- **B L0:** Long-lived provider keys are reachable by every shell and MCP subprocess. — [swarms/structs/autonomous_loop_utils.py:1113-1120](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L1113-L1120); [swarms/tools/mcp_manager.py:1458](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/tools/mcp_manager.py#L1458) (verified)
  - *To reach the next level:* Use scoped keys and keep them out of subprocess environments.
- **Cap:** none
- **Notes:** SECURITY.md lists 'No Telemetry' as a feature, which the code at this commit contradicts (otel.py:65-66).

### C9 Audit & traceability — 0.15 (high)

Tool outputs are appended to the in-memory conversation with timestamps, and loguru writes general logs to WORKSPACE_DIR/logs, but individual tool calls are only logged in verbose mode and the conversation is saved only if autosave is turned on. There is no actor attribution, no correlation across sub-agents, and records live in the workspace that the agent's own file tools can modify.

- **S L1:** Unstructured log lines for some actions; tool outputs kept only in an in-memory conversation. — [swarms/agents/tool_manager.py:681-682](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/agents/tool_manager.py#L681-L682) (verified)
  - *To reach the next level:* Persist a structured record of every tool call with arguments and status.
- **C L1:** Only the callable-tool path records output into the conversation; approvals do not exist and sub-agent calls are not correlated. — [swarms/agents/tool_manager.py:681-682](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/agents/tool_manager.py#L681-L682); [swarms/agents/tool_manager.py:810](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/agents/tool_manager.py#L810) (verified)
  - *To reach the next level:* Record all built-in tool calls including MCP and sub-agents.
- **D L0:** A durable record requires opting into autosave; logs go under the workspace the agent can write. — [swarms/structs/conversation.py:78](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/conversation.py#L78); [swarms/utils/loguru_logger.py:36](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/utils/loguru_logger.py#L36) (verified)
  - *To reach the next level:* Persist the tool-call record by default outside the workspace.
- **B L0:** Logging is best-effort with errors swallowed and the conversation lives in memory, so records are lost on crash. — [swarms/utils/loguru_logger.py:72-76](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/utils/loguru_logger.py#L72-L76) (verified)
  - *To reach the next level:* Flush records per action and surface failures.
- **Cap:** G1 — A durable conversation/tool-call record requires opting into autosave (conversation.py:78 default False).

### C10 Limits & kill switch — 0.38 (high)

The default Agent runs a single loop. Autonomous mode is bounded by planning attempts (5), total iterations (100) and loops per subtask (20), and shell commands time out after 60 seconds. There is no wall-clock or token/cost budget, user tools have no timeout, and sub-agents created by the model get their own fresh loop budgets with no cap on how many are spawned. Stopping relies on KeyboardInterrupt; background sub-agent tasks are not guaranteed to stop.

- **S L2:** Iteration caps plus a 60s per-command timeout for run_bash, enforced in code. — [swarms/structs/autonomous_loop_utils.py:42-44](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L42-L44); [swarms/structs/autonomous_loop_utils.py:1087](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L1087) (verified)
  - *To reach the next level:* Add wall-clock and token/cost caps and rate limits on side-effecting tools.
- **C L1:** Caps apply to the top-level loop; sub-agents start fresh budgets and user tools have no timeout. — [swarms/structs/autonomous_loop_utils.py:1408-1410](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L1408-L1410) (verified)
  - *To reach the next level:* Apply tool timeouts to all tools and count sub-agents against the parent budget.
- **D L2:** Sensible defaults (max_loops=1) that the operator can raise. — [swarms/structs/agent.py:315](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/agent.py#L315); [swarms/structs/autonomous_loop_utils.py:1408-1410](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L1408-L1410) (verified)
  - *To reach the next level:* Prevent the model from escaping limits by delegating.
- **B L1:** Autonomous runs can reach 100+ LLM iterations plus unbounded sub-agents with no spend ceiling. — [swarms/structs/autonomous_loop_utils.py:42-44](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L42-L44); [swarms/structs/autonomous_loop_utils.py:1408-1410](https://github.com/kyegomez/swarms/blob/a87cece8ba8c770faadbb8f5dde198d24f1dab5d/swarms/structs/autonomous_loop_utils.py#L1408-L1410) (verified)
  - *To reach the next level:* Add tight per-run time and cost ceilings.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: tool/MCP results appended as conversation messages (swarms/agents/tool_manager.py:681) · [B] sensitive data/systems: full process environment and API keys inherited by shell (swarms/structs/autonomous_loop_utils.py:1114) · [C] state change / egress: run_bash shell=True and delete_file os.remove (swarms/structs/autonomous_loop_utils.py:1116, 969) · Same default session? Yes

## Highest-impact improvements
1. Make telemetry opt-in and content-free. — C8 D L0→L2, +0.100 before caps (Playbook 4)
2. Harden transport-security defaults and redact secrets on logging and telemetry paths. — C8 S L0→L2, +0.150 before caps (Playbook 4)
3. Add a per-call approval callback, on by default, for run_bash, file writes/deletes and MCP calls. — C2 S L0→L3, +0.225 before caps (Playbook 5)
4. Stop auto-loading .env from the working directory at import; load only an explicit path. — C6 S L0→L1, +0.075 before caps (Playbook 2)
5. Confine file tools to the agent workspace with realpath containment. — C3 S L0→L3, +0.225 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- LiteLLM behaviour (reading provider base URLs from environment, and one transport setting) is inferred from the library, not verified in this repo.
- The CLI (swarms/cli), examples/, and most multi-agent structures were not reviewed in depth; scoring focused on Agent, the autonomous loop, tool execution, MCP, telemetry and memory.
- No text aimed at AI reviewers was found; SECURITY.md claims 'No Telemetry', which the code contradicts.
