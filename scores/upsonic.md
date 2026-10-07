# Defense-in-Depth Score: Upsonic

**Repo:** https://github.com/upsonic/upsonic · **Commit:** `101f0313b0ddb96cd4078354879b2ff57005db29` (0.77.3) · **Reviewed:** 2026-10-04
**What it is:** Python framework for building autonomous and traditional AI agents, with filesystem/shell toolkits, MCP, skills, memory, and multi-agent teams.
**Category:** Agent Frameworks
**Scored configuration:** AutonomousAgent constructed with default arguments as in the README quick start (shell and filesystem toolkits on, workspace = cwd, InMemoryStorage, no policies, no approval flags).
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents opt-in · external communication opt-in

## Score: 2.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L2 | L1 | L0 | L0 | 0.23 | C2-POWERBYPASS | **0.23** | High |
| C3 | Tool & action scoping | L1 | L2 | L1 | L0 | 0.28 | — | **0.28** | High |
| C4 | Code-execution isolation | L4 | L2 | L0 | L1 | 0.50 | G1 | **0.50** (alt) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | C6-REPOCONFIG | **0.10** | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L0 | 0.00 | C7-RCELOAD | **0.00** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | High |
| C9 | Audit & traceability | L2 | L2 | L1 | L0 | 0.35 | — | **0.35** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |


Upsonic's headline AutonomousAgent runs whatever shell commands the model writes directly on your machine, with your full environment (API keys, cloud credentials) and no approval step; the 'dangerous commands are blocked' claim is a five-entry substring denylist. The framework has a real per-call approval primitive and an E2B remote sandbox, but both are opt-in. It also auto-loads a .env and AGENTS.md from the working directory, so a cloned repo can steer endpoints and instructions. Treat the default as unsandboxed host execution and add confirmation flags, E2B, and a scrubbed environment before pointing it at anything untrusted.

## Critical gaps
- Default AutonomousAgent runs model-chosen shell commands on the host with the full environment and no approval. (ASI05, T11; C4) — [src/upsonic/agent/autonomous_agent/shell_toolkit.py:140-153](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L140-L153)
- Every tool and subprocess inherits the user's full ambient credentials. (ASI03, T3; C1) — [src/upsonic/agent/autonomous_agent/shell_toolkit.py:140](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L140)
- A hijacked default agent can exfiltrate secrets and delete files unattended. (ASI01, LLM01; C5) — [src/upsonic/agent/autonomous_agent/filesystem_toolkit.py:815-817](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/filesystem_toolkit.py#L815-L817)
- The default shell tool's approval bypass covers the most powerful action. (ASI09, ASI02; C2) — [src/upsonic/tools/config.py:20-21](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/tools/config.py#L20-L21)
- A .env in the working directory is auto-loaded at import and AGENTS.md is silently loaded into the system prompt. (ASI06, T1; C6) — [src/upsonic/utils/logging_config.py:79-84](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/utils/logging_config.py#L79-L84)
- The model can install and run remote packages unattended; extensions run with the full environment. (ASI04, T17; C7) — [src/upsonic/agent/autonomous_agent/shell_toolkit.py:129-131](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L129-L131); [src/upsonic/tools/mcp.py:370](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/tools/mcp.py#L370)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

Upsonic agents run with whatever authority the Python process has. The built-in shell tool and every MCP stdio server receive a full copy of the process environment, so model provider keys, cloud credentials, and anything else exported are available to model-chosen commands. There is no per-tool identity, no scoped credential, and no authorization check between a tool call and its execution.

- **S L0:** Tools act with the launching user's ambient authority; shell subprocesses inherit the entire os.environ. — [src/upsonic/agent/autonomous_agent/shell_toolkit.py:140-153](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L140-L153); [src/upsonic/tools/mcp.py:370](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/tools/mcp.py#L370); searched `rg -n 'os\.environ\.copy\(\)|\*\*os\.environ'` in `src/upsonic` → 4 hits (all four hits pass the full parent environment to the shell tool (sync/async) and MCP stdio servers) (verified)
  - *To reach the next level:* No dedicated or scoped identity; tools do not narrow the ambient credentials.
- **C L0:** No authorization layer exists on any tool path; built-in shell, filesystem and MCP tools all execute directly. — searched `rg -n 'os\.environ\.copy\(\)|\*\*os\.environ'` in `src/upsonic` → 4 hits (all four hits pass the full parent environment to the shell tool (sync/async) and MCP stdio servers); [src/upsonic/agent/autonomous_agent/shell_toolkit.py:134-145](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L134-L145) (verified)
  - *To reach the next level:* No tool path passes an authorization check before acting.
- **D L0:** AutonomousAgent ships shell and filesystem tools enabled with the operator's full authority. — [src/upsonic/agent/autonomous_agent/autonomous_agent.py:106-107](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/autonomous_agent.py#L106-L107) (verified)
  - *To reach the next level:* Least privilege would need a narrower default role; none exists.
- **B L0:** A hijacked agent can use every credential of the OS user (SSH keys, cloud CLIs, exported API keys) through the shell tool. — [src/upsonic/agent/autonomous_agent/shell_toolkit.py:145-153](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L145-L153); searched `rg -n 'os\.environ\.copy\(\)|\*\*os\.environ'` in `src/upsonic` → 4 hits (all four hits pass the full parent environment to the shell tool (sync/async) and MCP stdio servers) (verified)
  - *To reach the next level:* No credential or scope boundary limits what a hijacked shell call can reach.
- **Cap:** none

### C2 Approval gates — 0.23 (high)

The framework has a real human-in-the-loop primitive: a tool marked requires_confirmation pauses the run and hands the exact tool name and arguments to the calling code for approval. It is off for every tool by default, and none of AutonomousAgent's built-in shell, file-write, or delete tools set it, so the default agent runs shell commands and deletes files with no human involved. Optional safety policies are LLM or keyword classifiers, not approval gates.

- **S L2:** When enabled, a ConfirmationPause hands the caller the exact tool_name and tool_args per call, but the only 'tier' is a per-tool boolean. — [src/upsonic/tools/execution.py:82-83](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/tools/execution.py#L82-L83); [src/upsonic/agent/pipeline/manager.py:813-819](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/pipeline/manager.py#L813-L819) (verified)
  - *To reach the next level:* No risk tiers or argument-level policy deciding which calls need a human.
- **C L1:** Only tools explicitly flagged requires_confirmation are gated; the shell and filesystem toolkits set no flag. — [src/upsonic/tools/config.py:20-21](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/tools/config.py#L20-L21); [src/upsonic/agent/autonomous_agent/shell_toolkit.py:106](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L106) (verified)
  - *To reach the next level:* Every mutating tool, including shell, delete and MCP tools, would need to traverse the gate by default.
- **D L0:** requires_confirmation defaults to False, so approval is opt-in. — [src/upsonic/tools/config.py:20-21](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/tools/config.py#L20-L21) (verified)
  - *To reach the next level:* Approval is not on by default for any tool.
- **B L0:** Unapproved actions include arbitrary shell commands and recursive directory deletion with no checkpoint or undo. — [src/upsonic/agent/autonomous_agent/filesystem_toolkit.py:815-817](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/filesystem_toolkit.py#L815-L817); [src/upsonic/agent/autonomous_agent/shell_toolkit.py:145-147](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L145-L147) (verified)
  - *To reach the next level:* No checkpoints, rollback, or dry-run for filesystem or shell actions.
- **Cap:** C2-POWERBYPASS — The default AutonomousAgent shell tool (run_command) is not flagged for confirmation, so the most powerful action skips the gate.

### C3 Tool & action scoping — 0.28 (high)

The filesystem tools resolve each path and refuse anything outside the workspace, which is a sound check. That check is moot because the shell tool, on by default, accepts any shell string with only a five-entry substring denylist (for example 'rm -rf /'), and runs it with shell=True. The model can also choose the command timeout and extra environment variables. Registered user tools get schema typing but no shared argument policy.

- **S L1:** Filesystem paths get resolve()+relative_to containment, but the default shell tool only applies a substring denylist to a raw shell string. — [src/upsonic/agent/autonomous_agent/shell_toolkit.py:74-80](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L74-L80); [src/upsonic/agent/autonomous_agent/shell_toolkit.py:89-93](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L89-L93); [src/upsonic/agent/autonomous_agent/filesystem_toolkit.py:78-85](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/filesystem_toolkit.py#L78-L85) (verified)
  - *To reach the next level:* The general shell tool would need replacing by narrow tools or an allowlist matched on parsed commands.
- **C L2:** Both built-in toolkits apply some validation; MCP and user-registered tools get only type/schema handling. — [src/upsonic/agent/autonomous_agent/filesystem_toolkit.py:64-87](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/filesystem_toolkit.py#L64-L87); [src/upsonic/agent/autonomous_agent/shell_toolkit.py:82-102](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L82-L102) (verified)
  - *To reach the next level:* No shared validation layer wraps extension and user tools.
- **D L1:** Shell and filesystem write/delete tools are on by default but can be disabled by flag. — [src/upsonic/agent/autonomous_agent/autonomous_agent.py:106-107](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/autonomous_agent.py#L106-L107) (verified)
  - *To reach the next level:* The default tool set would need to be read-only, with write and exec requiring explicit enabling.
- **B L0:** A misused shell call can run any command on the host as the user; cwd is the workspace but nothing confines it. — [src/upsonic/agent/autonomous_agent/shell_toolkit.py:145-153](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L145-L153) (verified)
  - *To reach the next level:* Tools would need to be scoped to the workspace with bounded effects.
- **Cap:** none

### C4 Code-execution isolation — 0.50 (high)

By default, model-written shell commands and Python run directly on the host as the user, via subprocess with shell=True and the full environment, in a working directory that defaults to wherever the script was launched. Skill scripts also run on the host. Upsonic offers an opt-in E2B tool kit that runs code in a remote sandbox, but its upload and download tools read and write arbitrary host paths chosen by the model, and using it does not remove the host shell tool unless the developer disables it.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Default execution is a same-user host subprocess; the 'workspace sandboxing' is only the working directory. — [src/upsonic/agent/autonomous_agent/shell_toolkit.py:145-153](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L145-L153); [src/upsonic/agent/autonomous_agent/shell_toolkit.py:299-301](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L299-L301) (verified)
    - *To reach the next level:* No isolation primitive (container, OS sandbox, microVM) on the default exec path.
  - **C L0:** Neither the main shell tool nor skill script execution is sandboxed. — [src/upsonic/skills/utils.py:157-163](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/skills/utils.py#L157-L163); [src/upsonic/agent/autonomous_agent/shell_toolkit.py:219-225](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L219-L225) (verified)
    - *To reach the next level:* The main exec tool would need to run inside an isolation boundary.
  - **D L0:** No sandbox is on by default; host execution is the shipped behaviour. — [src/upsonic/agent/autonomous_agent/autonomous_agent.py:106-107](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/autonomous_agent.py#L106-L107) (verified)
    - *To reach the next level:* A sandbox would need to be on by default.
  - **B L0:** Executed code has the host filesystem, network, and every credential in the inherited environment. — [src/upsonic/agent/autonomous_agent/shell_toolkit.py:140-153](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L140-L153); [src/upsonic/tools/mcp.py:370](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/tools/mcp.py#L370) (verified)
    - *To reach the next level:* Execution would need workspace-only mounts, no secrets in the environment, and restricted egress.
- **opt-in E2B remote sandbox tools** (alt; raw 0.50, cap G1 → 0.50) ← counted
  - **S L4:** E2BTools runs code and commands in a remote E2B sandbox created per toolkit. — [src/upsonic/tools/custom_tools/e2b.py:88](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/tools/custom_tools/e2b.py#L88); [src/upsonic/tools/custom_tools/e2b.py:121-128](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/tools/custom_tools/e2b.py#L121-L128) (verified)
  - **C L2:** Code runs remotely, but e2b_download_file writes model-chosen host paths and skill scripts still run on the host. — [src/upsonic/tools/custom_tools/e2b.py:330-337](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/tools/custom_tools/e2b.py#L330-L337); [src/upsonic/skills/utils.py:157](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/skills/utils.py#L157) (verified)
    - *To reach the next level:* Host file transfer tools and skill scripts would need to be confined too.
  - **D L0:** Opt-in: requires constructing E2BTools and disabling the default shell tool. — [src/upsonic/tools/custom_tools/e2b.py:73](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/tools/custom_tools/e2b.py#L73) (verified)
    - *To reach the next level:* Not on by default.
  - **B L1:** Upload/download take arbitrary host paths, so host files can be exfiltrated or overwritten via the sandbox, which has network access. — [src/upsonic/tools/custom_tools/e2b.py:309-312](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/tools/custom_tools/e2b.py#L309-L312) (verified)
    - *To reach the next level:* Host transfer would need path containment and sandbox egress restricted.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.00 (high)

Tool results, file contents, command output, and MCP tool descriptions enter the model context with no provenance or untrusted marker, and nothing in the agent loop changes what the model may do after reading them. In the default AutonomousAgent, a hijacked model can read secrets from the environment through the shell, send them anywhere over the network, and delete files, all without a human. The safety engine's injection and content policies are optional classifiers.

- **S L0:** No structural limit on a hijacked agent; untrusted content is not tracked. — searched `rg -n -i 'untrusted|taint'` in `src/upsonic` → 12 hits (no hit is a taint or provenance control: weaviate 'certainty', a Telegram field, an MCP warning string, a reliability-layer prompt label and skill docs); [src/upsonic/agent/agent.py:2331-2334](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/agent.py#L2331-L2334) (verified)
  - *To reach the next level:* Approval or capability removal would need to trigger once untrusted content is read.
- **C L0:** No untrusted source is distinguished from the principal's instructions. — searched `rg -n -i 'untrusted|taint'` in `src/upsonic` → 12 hits (none are controls) (verified)
  - *To reach the next level:* Untrusted sources would need to be distinguished at all.
- **D L0:** No control is on by default; injection policies are opt-in constructor arguments. — [src/upsonic/agent/agent.py:269](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/agent.py#L269) (verified)
  - *To reach the next level:* A limit would need to be on by default.
- **B L0:** A hijacked default agent can exfiltrate environment secrets via shell network tools and delete or overwrite files unattended. — [src/upsonic/agent/autonomous_agent/shell_toolkit.py:140-153](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L140-L153); [src/upsonic/tools/mcp.py:370](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/tools/mcp.py#L370); [src/upsonic/agent/autonomous_agent/filesystem_toolkit.py:815-817](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/filesystem_toolkit.py#L815-L817) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions would need human approval.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.
- **Notes:** Messaging interfaces (Slack, Telegram, WhatsApp, mail) are an opt-in deployment whose access control is not locked down; C5-PUBLICTRIGGER was not applied to the scored default.

### C6 Memory, context & configuration integrity — 0.10 (high)

Importing Upsonic loads a .env file from the current working directory, and the AutonomousAgent workspace defaults to that same directory; that file can set provider base URLs, keys, or the telemetry DSN for any variable not already set. An AGENTS.md in the workspace is read silently into the system prompt, and the agent's own write_file tool can create or edit that file, so an injection can persist into every later session. Session memory itself defaults to an in-memory store keyed by a random session id.

- **S L0:** Workspace AGENTS.md is injected into the system prompt without a trust prompt, the model can write it, and a cwd .env is auto-loaded at import. — [src/upsonic/utils/logging_config.py:79-84](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/utils/logging_config.py#L79-L84); [src/upsonic/agent/agent.py:611-615](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/agent.py#L611-L615); [src/upsonic/agent/context_managers/system_prompt_manager.py:232-233](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/context_managers/system_prompt_manager.py#L232-L233) (verified)
  - *To reach the next level:* Instruction and env files from the workspace would need an explicit trust decision.
- **C L0:** Neither the .env, AGENTS.md, nor memory paths are validated. — [src/upsonic/agent/context_managers/llm_manager.py:12](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/context_managers/llm_manager.py#L12); [src/upsonic/agent/context_managers/system_prompt_manager.py:232-233](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/context_managers/system_prompt_manager.py#L232-L233) (verified)
  - *To reach the next level:* At least one store would need controlled writes.
- **D L1:** Default memory is per-session InMemoryStorage with random session/user ids, but workspace files are shared across sessions; capped one level above S. — [src/upsonic/agent/autonomous_agent/autonomous_agent.py:232](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/autonomous_agent.py#L232); [src/upsonic/storage/memory/memory.py:151](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/storage/memory/memory.py#L151) (verified)
  - *To reach the next level:* The mechanism (S) would need strengthening before isolation earns more credit.
- **B L1:** A poisoned AGENTS.md persists across the user's sessions in that workspace and can drive shell and file tools. — [src/upsonic/agent/autonomous_agent/filesystem_toolkit.py:225-230](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/filesystem_toolkit.py#L225-L230) (verified)
  - *To reach the next level:* Poisoned context would need to be session-scoped or human-reviewed.
- **Cap:** C6-REPOCONFIG — A .env in the working directory (the default workspace) is auto-loaded at import and can set unset endpoint, key, and telemetry variables without any trust decision.

### C7 Third-party extensions — 0.00 (high)

The default AutonomousAgent loads no plugins, but its shell tool's own documentation suggests 'pip install -r requirements.txt', and nothing stops the model installing and running any package. When developers add extensions, MCP stdio servers launch through npx/uvx (and docker) with the full environment, GitHub skills are pulled from the tip of a branch with no hash, and prebuilt agents clone the latest Upsonic master at runtime; skill scripts run on the host.

- **S L0:** Model-chosen package installs run unattended through the default shell tool; optional skill/MCP sources are unpinned. — [src/upsonic/agent/autonomous_agent/shell_toolkit.py:129-131](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L129-L131); [src/upsonic/skills/loader/github.py:74](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/skills/loader/github.py#L74); [src/upsonic/tools/mcp.py:110-113](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/tools/mcp.py#L110-L113) (verified)
  - *To reach the next level:* Sources would need to be user-chosen and pinned, with model-driven installs blocked or approved.
- **C L0:** No extension type (skills, MCP, prebuilt templates, packages) is integrity-checked. — searched `rg -n 'hash|sha256|verify'` in `src/upsonic/skills/loader src/upsonic/tools/mcp.py` → 3 hits (all three are the cache-directory key in remote_base.py, not integrity verification) (verified)
  - *To reach the next level:* At least one extension type would need pinning or integrity checks.
- **D L0:** The model can install packages with no consent in the default configuration; prebuilt agents fetch remote templates automatically. — [src/upsonic/prebuilt/applied_scientist/agent.py:1007](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/prebuilt/applied_scientist/agent.py#L1007); [src/upsonic/prebuilt/prebuilt_agent_base.py:167-176](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/prebuilt/prebuilt_agent_base.py#L167-L176) (verified)
  - *To reach the next level:* Third-party code would need explicit install with consent.
- **B L0:** Installed packages, MCP stdio servers and skill scripts run as the same user with the full environment. — [src/upsonic/tools/mcp.py:370](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/tools/mcp.py#L370); [src/upsonic/skills/utils.py:157-163](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/skills/utils.py#L157-L163) (verified)
  - *To reach the next level:* Extensions would need at least a scrubbed environment.
- **Cap:** C7-RCELOAD — In the default AutonomousAgent the model can pip/npm-install and run remote packages via the unapproved shell tool, which documents pip install as an example.

### C8 Secrets & sensitive-data protection — 0.25 (high)

API keys come from environment variables (often from an auto-loaded .env), and the shell tool and MCP servers get the whole environment, so a single 'env' command puts every key into the model's context and the provider's logs. Only some vector-database configs use masked secret types; there is no redaction of logs, console tool-call printing, or model-bound messages. Error telemetry to Sentry is opt-in.

- **S L1:** Secrets are read from env vars; SecretStr masking exists only on a few vector DB config fields. — [src/upsonic/vectordb/config.py:84](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/vectordb/config.py#L84); searched `rg -n -i 'redact|SecretStr'` in `src/upsonic` → 33 hits (hits are opt-in safety-policy replacement text, provider thinking-block handling, and four vector DB SecretStr fields; none redact logs or model-bound messages) (verified)
  - *To reach the next level:* Type-level masking and log filters on main paths would be needed.
- **C L1:** Only vector-DB config objects mask their keys (SecretStr reprs); logs, printed tool calls, model-bound messages and subprocess env are unprotected. — [src/upsonic/vectordb/config.py:274](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/vectordb/config.py#L274); [src/upsonic/agent/autonomous_agent/shell_toolkit.py:140-153](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L140-L153); [src/upsonic/tools/mcp.py:370](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/tools/mcp.py#L370) (verified)
  - *To reach the next level:* Logs and transcripts would also need redaction.
- **D L2:** Telemetry is opt-in via UPSONIC_TELEMETRY with no default DSN; there is no redaction to turn off. — [src/upsonic/utils/logging_config.py:195-199](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/utils/logging_config.py#L195-L199) (verified)
  - *To reach the next level:* Redaction would need to be always on.
- **B L0:** Long-lived provider and cloud keys in the environment are reachable by every shell subprocess the model runs. — [src/upsonic/agent/autonomous_agent/shell_toolkit.py:140-153](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L140-L153); [src/upsonic/tools/mcp.py:370](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/tools/mcp.py#L370) (verified)
  - *To reach the next level:* Keys reachable by tools would need to be scoped and short-lived.
- **Cap:** none

### C9 Audit & traceability — 0.35 (high)

Each run keeps a structured list of tool calls (name, arguments, result) on the task object and emits tool-call events, and calls are printed to the console by default. By default this lives only in process memory (InMemoryStorage), so it disappears when the process exits or crashes, and there is no actor attribution or tamper-evident storage. OpenTelemetry instrumentation exists but is opt-in.

- **S L2:** Tool calls are recorded as structured dicts with tool_name, params and tool_result on the task. — [src/upsonic/utils/tool_usage.py:52-54](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/utils/tool_usage.py#L52-L54); [src/upsonic/tasks/tasks.py:779-789](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/tasks/tasks.py#L779-L789) (verified)
  - *To reach the next level:* No actor attribution (requesting principal, approver) or correlation across agents.
- **C L2:** All tool calls extracted from model messages are recorded, including MCP tools; approvals and sub-agents are not shown to be recorded. — [src/upsonic/utils/tool_usage.py:10-16](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/utils/tool_usage.py#L10-L16) (verified)
  - *To reach the next level:* Approvals/denials and sub-agent calls would need to be in the record.
- **D L1:** Recording is on by default but held in the agent's own process memory with no durable store. — [src/upsonic/agent/autonomous_agent/autonomous_agent.py:232](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/autonomous_agent.py#L232); [src/upsonic/agent/autonomous_agent/autonomous_agent.py:161](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/autonomous_agent.py#L161) (verified)
  - *To reach the next level:* The record would need to be written outside the agent's process.
- **B L0:** The default in-memory record is lost on crash or exit and failures are silent. — [src/upsonic/agent/autonomous_agent/autonomous_agent.py:232](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/autonomous_agent.py#L232) (verified)
  - *To reach the next level:* Records would need to be flushed per action.
- **Cap:** none

### C10 Limits & kill switch — 0.40 (high)

Agents stop executing tools after 100 tool calls by default, and shell commands get a 120-second timeout. There is no token or cost cap: a UsageLimits class is defined but never used, and the model can pass its own, longer timeout to each shell command. Cancellation is checked between steps, and the shell timeout kills only the direct child process.

- **S L2:** Iteration cap (tool_call_limit=100) plus a per-command shell timeout are enforced in code. — [src/upsonic/agent/agent.py:2328-2333](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/agent.py#L2328-L2333); [src/upsonic/agent/autonomous_agent/shell_toolkit.py:138](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L138); searched `rg -n -S 'UsageLimits'` in `src` → 3 hits (defined in usage.py and mentioned in a comment; never instantiated or enforced) (verified)
  - *To reach the next level:* No wall-clock or token/cost cap, and no rate limits on side-effecting tools.
- **C L2:** The loop cap and tool timeouts apply; sub-agent budget sharing was not shown. — [src/upsonic/tools/config.py:80-82](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/tools/config.py#L80-L82); [src/upsonic/run/cancel.py:51-55](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/run/cancel.py#L51-L55) (verified)
  - *To reach the next level:* Sub-agents and spawned processes would need to count against the same budget.
- **D L1:** Defaults exist, but the model chooses the per-call shell timeout and can raise it without bound. — [src/upsonic/agent/autonomous_agent/shell_toolkit.py:107-111](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L107-L111) (verified)
  - *To reach the next level:* The model would need to be unable to raise its own limits.
- **B L1:** No spend ceiling; stop is cooperative and a timed-out shell=True command can leave grandchild processes running. — [src/upsonic/agent/autonomous_agent/shell_toolkit.py:145-152](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/agent/autonomous_agent/shell_toolkit.py#L145-L152); [src/upsonic/run/cancel.py:51-55](https://github.com/upsonic/upsonic/blob/101f0313b0ddb96cd4078354879b2ff57005db29/src/upsonic/run/cancel.py#L51-L55) (verified)
  - *To reach the next level:* Tight time/cost ceilings and cancellation of in-flight calls would be needed.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: File contents, command output and MCP results enter context as ordinary tool returns (src/upsonic/agent/agent.py:2331) · [B] sensitive data/systems: Full os.environ including provider/cloud keys given to the shell tool (src/upsonic/agent/autonomous_agent/shell_toolkit.py:140) · [C] state change / egress: Unapproved shell=True commands and recursive delete (src/upsonic/agent/autonomous_agent/shell_toolkit.py:145, filesystem_toolkit.py:817) · Same default session? Yes

## Highest-impact improvements
1. Set requires_confirmation on run_command, run_python, write/edit/move/delete in the AutonomousAgent toolkits by default. — C2 D L0→L3, +0.150 before caps (Playbook 5)
2. Pass a minimal allowlisted environment to shell subprocesses and MCP stdio servers instead of os.environ. — C8 B L0→L2, +0.100 before caps (Playbook 4)
3. Stop auto-loading .env from the working directory at import; require an explicit path from the developer. — C6 S L0→L2, +0.150 before caps (Playbook 2)
4. Wire UsageLimits into the agent loop with default token/cost and wall-clock caps, and clamp model-supplied shell timeouts. — C10 S L2→L3, +0.075 before caps (Playbook 3 step 3)
5. Make a sandbox backend (E2B or a hardened container) the default executor for AutonomousAgent shell tools, failing closed. — C4 D L0→L3, +0.150 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the AutonomousAgent defaults; the base Agent has no built-in shell/file tools, but shares the same approval, memory, .env, and limit mechanisms.
- Team/deepagent delegation, Ralph mode, graph/graphv2, interfaces, and storage backends other than in-memory were only skimmed; sub-agent budget sharing was not verified.
- No text aimed at AI reviewers was found; the repo's CLAUDE.md is contributor guidance for coding assistants, not a reviewer-steering attempt.
