# Defense-in-Depth Score: AgentScope

**Repo:** https://github.com/agentscope-ai/agentscope · **Commit:** `72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6` · **Reviewed:** 2026-10-03
**What it is:** Agent framework (Alibaba) to build and run observable agents
**Category:** Agent Frameworks
**Scored configuration:** SDK Agent with default arguments (PermissionContext mode DEFAULT, no workspace, no middlewares) and the README quickstart toolkit (Bash, Grep, Glob, Read, Write, Edit) in launch_console; model-reachable paths added by the bundled agent service's defaults are also considered.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents opt-in · external communication opt-in

## Score: 2.8 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L0 | L0 | 0.07 | — | **0.07** | High |
| C2 | Approval gates | L3 | L1 | L1 | L0 | 0.35 | C2-SELFAPPROVE | **0.25** | High |
| C3 | Tool & action scoping | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | High |
| C4 | Code-execution isolation | L4 | L2 | L0 | L2 | 0.55 | G1 | **0.50** (alt) | High |
| C5 | Untrusted input blast radius | L2 | L1 | L1 | L1 | 0.33 | — | **0.33** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C7 | Third-party extensions | L1 | L0 | L1 | L2 | 0.23 | — | **0.23** | Medium |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | High |
| C9 | Audit & traceability | L2 | L2 | L0 | L1 | 0.35 | G1 | **0.35** (alt) | Medium |
| C10 | Limits & kill switch | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |


AgentScope 2.0 ships a real permission engine: by default it asks before writes and most shell commands and shows the exact call. But the shell runs directly on the host with the full environment, reads of any file are auto-approved, and MCP servers can exempt their own tools by labelling them read-only. Neither the shell auto-allow nor approval in the bundled agent service is a strict boundary. Use a sandboxed workspace and treat auto-approval as a convenience, not a boundary.

## Critical gaps
- Default Bash tool runs on the host as the OS user with the full process environment, so a hijacked or wrongly approved command reaches the user's entire account. (ASI03, T3; C1) — [src/agentscope/tool/_builtin/_bash.py:177](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_bash.py#L177); [src/agentscope/tool/_builtin/_backend.py:799-804](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_backend.py#L799-L804)
- Model-chosen shell commands run on the host as the OS user with the agent's full environment by default; sandboxed workspaces are opt-in. (ASI05, T11; C4) — [src/agentscope/tool/_builtin/_bash.py:177](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_bash.py#L177); [src/agentscope/tool/_builtin/_backend.py:799-804](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_backend.py#L799-L804)

## Criterion details

### C1 Identity & least privilege — 0.07 (high)

AgentScope has no agent identity or credential scoping. In the default configuration the Bash tool runs as the operating-system user on the host and every command inherits the agent process's full environment, including the model API key the quickstart reads from an environment variable. Whatever the user can reach (cloud CLIs, SSH keys, git credentials) the agent can reach once a command is approved or slips through the read-only auto-allow. The agent service binds its own app tools (schedules, teams) to the authenticated user id, but that does not narrow what shell commands can do.

- **S L0:** Tools run with the OS user's ambient authority; Bash subprocesses inherit the full process environment. — [src/agentscope/tool/_builtin/_bash.py:177](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_bash.py#L177); [src/agentscope/tool/_builtin/_backend.py:794-804](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_backend.py#L794-L804); searched `rg -n env=` in `src/agentscope/tool/_builtin/_backend.py` → 0 hits (LocalBackend never passes env=, so every Bash subprocess inherits the agent process's full os.environ) (verified)
  - *To reach the next level:* No scoping primitive; L1 needs a dedicated identity for agent tools.
- **C L1:** Agent-service app tools are bound to the requesting user id, but the main exec path (Bash on LocalBackend) uses ambient credentials and the full os.environ. — [src/agentscope/app/_manager/_scheduler/_tools/_schedule_create.py:135](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/app/_manager/_scheduler/_tools/_schedule_create.py#L135); searched `rg -n env=` in `src/agentscope/tool/_builtin/_backend.py` → 0 hits (LocalBackend never passes env=, so every Bash subprocess inherits the agent process's full os.environ) (verified)
  - *To reach the next level:* Subprocesses get the full environment; L2 needs every built-in tool to use a scoped identity.
- **D L0:** The default install runs with the launching user's full privileges; nothing narrower exists to default to. — [src/agentscope/tool/_builtin/_bash.py:177](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_bash.py#L177) (verified)
  - *To reach the next level:* No narrower default; L1 needs any default narrower than the host user's authority.
- **B L0:** A hijacked or wrongly approved shell call reaches the user's entire account across services (cloud CLIs, SSH agent, git tokens, all files). — [src/agentscope/tool/_builtin/_bash.py:177](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_bash.py#L177); [src/agentscope/tool/_builtin/_backend.py:794-804](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_backend.py#L794-L804) (verified)
  - *To reach the next level:* Full user account reachable; L1 needs reach limited to write access in a few systems.
- **Cap:** none

### C2 Approval gates — 0.25 (high)

AgentScope ships a real, on-by-default approval system: every tool call goes through a permission engine that asks the human unless the call is classified read-only or matches an allow rule, the console shows the exact tool name and arguments, and the approved call is the one that runs. The gaps are in what gets auto-approved: the Read tool is treated as read-only for any path, MCP tools are auto-approved whenever the server itself declares them read-only, and neither the Bash read-only auto-allow nor approval in the bundled agent service is a strict boundary. Nothing provides undo or checkpoints for approved actions.

- **S L3:** Per-call approval shows the exact tool name and full input; the confirmed call (optionally user-edited) is what executes; deny/ask/allow rules exist but match raw strings, not parsed arguments. — [src/agentscope/console/_renderer.py:385-389](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/console/_renderer.py#L385-L389); [src/agentscope/agent/_agent.py:1976-1978](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/agent/_agent.py#L1976-L1978); [src/agentscope/agent/_agent.py:2424-2429](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/agent/_agent.py#L2424-L2429); [src/agentscope/tool/_builtin/_bash.py:428](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_bash.py#L428) (verified)
  - *To reach the next level:* Rules match raw command strings by prefix/substring; L4 needs argument-level policy on parsed arguments.
- **C L1:** Every tool call crosses the engine, but auto-approval relies on self-declared or pattern-matched read-only status: MCP servers set readOnlyHint to skip the gate, Read/Glob/Grep pass for any path, and the Bash read-only auto-allow is not a strict boundary. — [src/agentscope/agent/_agent.py:2521](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/agent/_agent.py#L2521); [src/agentscope/permission/_engine.py:683-690](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/permission/_engine.py#L683-L690); [src/agentscope/tool/_adapters.py:271-274](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_adapters.py#L271-L274); [src/agentscope/tool/_adapters.py:307-311](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_adapters.py#L307-L311) (verified)
  - *To reach the next level:* Auto-approved commands are not a verified read-only allowlist and extensions self-classify; L2 needs every built-in tool path gated with only extensions bypassing.
- **D L1:** Approval is on by default (mode DEFAULT), but in the agent service approval enforcement is not tamper-resistant. — [src/agentscope/permission/_context.py:31](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/permission/_context.py#L31); [src/agentscope/permission/_engine.py:485-489](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/permission/_engine.py#L485-L489) (verified)
  - *To reach the next level:* Approval can be relaxed without operator action; L2 needs only operator configuration to be able to disable approval.
- **B L0:** Approved or bypassed actions are host shell commands and file writes with no checkpoint, rollback, or dry-run. — [src/agentscope/tool/_builtin/_bash.py:177](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_bash.py#L177); searched `rg -n -i checkpoint|rollback|undo` in `src/agentscope/tool src/agentscope/agent` → 0 hits (no checkpoint or undo for tool side effects) (verified)
  - *To reach the next level:* Irreversible actions with no undo; L1 needs at least some actions to be reversible.
- **Cap:** C2-SELFAPPROVE — In the bundled agent service approval enforcement is not tamper-resistant, so the model can satisfy approval for its own actions.

### C3 Tool & action scoping — 0.25 (high)

The default coding tools are general-purpose: Bash runs any shell string and Read accepts any path on the machine. Write, Edit and Bash carry denylist checks (sensitive dotfiles and directories, a short list of dangerous command patterns, `rm` of system directories), and Bash flags command substitution and control flow for review, but these are filters rather than allowlists. The Toolkit starts empty, which is a safe constructor default, but the README quickstart and examples register Bash, Write and Edit straight away.

- **S L1:** Argument checks are denylists: dangerous file/dir names, dangerous command substrings, rm of root-level paths; Bash otherwise passes an arbitrary shell string. — [src/agentscope/tool/_constants.py:56-68](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_constants.py#L56-L68); [src/agentscope/tool/_builtin/_write.py:133-140](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_write.py#L133-L140); [src/agentscope/tool/_builtin/_bash.py:712](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_bash.py#L712) (verified)
  - *To reach the next level:* No allowlist validation; L2 needs typed, validated arguments beyond denylists (e.g., path containment).
- **C L1:** Write, Edit and Bash apply the denylists; Read, Glob and Grep accept any path, and MCP/function tools get only JSON-schema type checks. — [src/agentscope/tool/_builtin/_read.py:126](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_read.py#L126); [src/agentscope/agent/_agent.py:2497](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/agent/_agent.py#L2497) (verified)
  - *To reach the next level:* Read/Glob/Grep and extension tools do no argument validation; L2 needs most built-in tools validating.
- **D L2:** Toolkit() is empty by default so nothing is enabled implicitly, but the README quickstart and examples register Bash/Write/Edit, so per the framework rule D is lowered one level from L3. — [src/agentscope/agent/_agent.py:201](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/agent/_agent.py#L201); [README.md:157-167](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/README.md#L157-L167) (verified)
  - *To reach the next level:* Official examples enable write and exec by default; L3 needs a read-only tool set in the documented default.
- **B L0:** A misused Bash call can run any command against any host or file the user can reach. — [src/agentscope/tool/_builtin/_bash.py:177](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_bash.py#L177) (verified)
  - *To reach the next level:* General-purpose shell over the whole machine; L1 needs some limits on reach.
- **Cap:** none

### C4 Code-execution isolation — 0.50 (high)

By default the Bash tool runs commands directly on the host through a local subprocess with the agent's full environment; there is no isolation. AgentScope does ship workspace sandboxes (Docker, Bubblewrap, Apple Container, E2B, Daytona, Kubernetes, OpenSandbox) that rebind the built-in tools to the sandbox, and the E2B option is a remote sandbox service, but all of them are opt-in and the Agent constructor has no workspace by default. The Docker workspace is a stock container (root, default capabilities, default network) with an optional read-write host mount.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Default backend is a same-user host subprocess. — [src/agentscope/tool/_builtin/_bash.py:177](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_bash.py#L177); [src/agentscope/tool/_builtin/_backend.py:794-804](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_backend.py#L794-L804) (verified)
    - *To reach the next level:* No isolation; L1 needs at least filtering or a separate working directory boundary.
  - **C L0:** No execution path is sandboxed by default. — [src/agentscope/tool/_builtin/_bash.py:177](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_bash.py#L177) (verified)
    - *To reach the next level:* Nothing sandboxed; L1 needs the main exec tool sandboxed.
  - **D L0:** Isolation is off by default; sandboxes require constructing a workspace and passing its tools. — [src/agentscope/tool/_builtin/_bash.py:177](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_bash.py#L177); [src/agentscope/workspace/_base.py:576](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/workspace/_base.py#L576) (verified)
    - *To reach the next level:* Off by default; L1 needs a sandbox on by default.
  - **B L0:** Commands run as the host user with the full environment (model API keys and any cloud/git credentials). — [src/agentscope/tool/_builtin/_backend.py:794-804](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_backend.py#L794-L804); searched `rg -n env=` in `src/agentscope/tool/_builtin/_backend.py` → 0 hits (LocalBackend never passes env=, so every Bash subprocess inherits the agent process's full os.environ) (verified)
    - *To reach the next level:* Host-equivalent reach; L1 needs at least no credentials in the execution environment.
- **opt-in E2B workspace (remote sandbox)** (alt; raw 0.55, cap G1 → 0.50) ← counted
  - **S L4:** Workspace tools execute in an E2B remote sandbox created through the E2B SDK. — [src/agentscope/workspace/_e2b/_e2b_workspace.py:236](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/workspace/_e2b/_e2b_workspace.py#L236) (verified)
  - **C L2:** The workspace rebinds all six built-in tools to the sandbox backend, but function tools and MCP clients registered directly on the Toolkit still run on the host. — [src/agentscope/workspace/_base.py:575-582](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/workspace/_base.py#L575-L582) (verified)
    - *To reach the next level:* Tools registered outside the workspace run on the host; L3 needs every model-reachable path sandboxed.
  - **D L0:** Off by default. — [src/agentscope/agent/_agent.py:201](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/agent/_agent.py#L201) (verified)
    - *To reach the next level:* Off by default; L1+ needs it on.
  - **B L2:** Only explicitly configured env vars reach the sandbox, but egress is not restricted in code. — [src/agentscope/workspace/_e2b/_e2b_workspace.py:234](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/workspace/_e2b/_e2b_workspace.py#L234); searched `rg -n -i network|internet` in `src/agentscope/workspace/_e2b` → 2 hits (one docstring about host-to-sandbox traffic and one comment about transient network errors; no egress restriction) (verified)
    - *To reach the next level:* Sandbox egress is unrestricted; L3 needs egress off or allowlisted.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.33 (high)

AgentScope does not track where content came from: tool results, MCP outputs and file contents enter the conversation with the same standing as anything else, and there is no taint-based restriction. What limits a hijacked agent is the general approval gate, which asks before most writes and shell commands in the default mode. But reads of any file are auto-approved, and MCP tools that their server labels read-only (fetch-style tools commonly are) run without a prompt, so an injected instruction can read secrets and send them out through such a tool without a human seeing it. Irreversible actions still need approval in the default mode.

- **S L2:** Writes and non-read-only shell commands need approval in the default mode, but egress through read-only-labelled tools does not; nothing is conditioned on having read untrusted content. — [src/agentscope/permission/_context.py:31](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/permission/_context.py#L31); [src/agentscope/permission/_engine.py:683-690](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/permission/_engine.py#L683-L690); [src/agentscope/tool/_adapters.py:307-311](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_adapters.py#L307-L311) (verified)
  - *To reach the next level:* Egress is not gated; L3 needs every egress and state-changing tool forced through approval once untrusted content is read.
- **C L1:** Untrusted sources are not distinguished; tool and MCP results enter context as ordinary tool results, and the gate covers sources only incidentally. — [src/agentscope/agent/_agent.py:2626](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/agent/_agent.py#L2626); searched `rg -n -i untrusted|provenance|taint` in `src/agentscope/agent src/agentscope/tool src/agentscope/permission` → 1 hits (single hit is a backend docstring about payload size caps; nothing tags or restricts untrusted content) (verified)
  - *To reach the next level:* No source is treated as untrusted; L2 needs most untrusted sources inside a limit.
- **D L1:** The gate is on by default, but in the agent service its enforcement is not tamper-resistant. — [src/agentscope/permission/_context.py:31](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/permission/_context.py#L31) (verified)
  - *To reach the next level:* The limit can be relaxed without operator action; L2 needs only the operator able to disable it.
- **B L1:** Unattended: read any file (Read auto-allowed) and send it out via a read-only-labelled MCP tool; irreversible actions still need approval in DEFAULT mode. — [src/agentscope/tool/_builtin/_read.py:126](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_read.py#L126); [src/agentscope/permission/_engine.py:683-690](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/permission/_engine.py#L683-L690); [src/agentscope/tool/_adapters.py:271-274](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_adapters.py#L271-L274); [src/agentscope/tool/_adapters.py:307-311](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_adapters.py#L307-L311) (verified)
  - *To reach the next level:* Exfiltration happens unattended; L2 needs both exfiltration and irreversible actions to require approval.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.10 (high)

The SDK itself keeps conversation state in memory and does not auto-load instruction files or .env files from the working directory. Its long-term memory middlewares are opt-in, but when enabled (Mem0 defaults to its 'both' mode) every exchange is written back automatically, the model's add_memory tool is always allowed, and retrieved memories are re-injected as an assistant message rather than marked data. Memories are namespaced by a required user id. In workspace mode, MCP server declarations persist in a `.mcp` file inside the agent-visible working directory and are read back in later sessions.

- **S L0:** Memory writes are automatic and model-initiated without validation, and retrieved memories are injected as an assistant-role message. — [src/agentscope/middleware/_longterm_memory/_mem0/_middleware.py:151](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/middleware/_longterm_memory/_mem0/_middleware.py#L151); [src/agentscope/middleware/_longterm_memory/_mem0/_tools.py:55-58](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/middleware/_longterm_memory/_mem0/_tools.py#L55-L58); [src/agentscope/middleware/_longterm_memory/_mem0/_middleware.py:663-666](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/middleware/_longterm_memory/_mem0/_middleware.py#L663-L666) (verified)
  - *To reach the next level:* No validation or provenance; L1 needs writes at least logged and memories presented as data.
- **C L0:** No memory or persisted-config path is controlled; the workspace `.mcp` file sits inside the agent-visible workdir and is read back without review. — [src/agentscope/workspace/_base.py:460-462](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/workspace/_base.py#L460-L462); [src/agentscope/workspace/_base.py:951](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/workspace/_base.py#L951) (verified)
  - *To reach the next level:* Nothing controlled; L1 needs one store gated.
- **D L1:** Mem0 memories are namespaced by a required user_id passed in every query (capped at one level above S). — [src/agentscope/middleware/_longterm_memory/_mem0/_middleware.py:252-253](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/middleware/_longterm_memory/_mem0/_middleware.py#L252-L253) (verified)
  - *To reach the next level:* Rating limited by the weak write control (C/D at most one level above S); with stronger write gating, required per-user namespaces would support L2.
- **B L1:** Poisoned memories persist across the user's sessions and are re-injected into context where they can drive tool use. — [src/agentscope/middleware/_longterm_memory/_mem0/_middleware.py:663-666](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/middleware/_longterm_memory/_mem0/_middleware.py#L663-L666) (verified)
  - *To reach the next level:* Persistence can trigger tool use; L2 needs persisted content to influence only text or gated actions.
- **Cap:** none

### C7 Third-party extensions — 0.23 (medium)

Third-party code enters through MCP servers and skills that the developer (or, in the agent service, a user browsing ClawHub or the GitHub MCP registry) chooses. Nothing is pinned or integrity-checked, and there is no re-approval when a server's tools change. Stdio MCP servers run as separate processes; the MCP library's default environment handling passes only a minimal set of variables unless an env is configured. A workspace's `.mcp` declarations live in the agent-writable working directory, so an approved file write can add a server. Separately, MCP servers can exempt their own tools from the approval gate by labelling them read-only (scored under approval gates).

- **S L1:** MCP servers are launched from a user-supplied command with no version pinning or verification. — [src/agentscope/mcp/_config.py:14-17](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/mcp/_config.py#L14-L17); [src/agentscope/mcp/_mcp_client.py:198-201](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/mcp/_mcp_client.py#L198-L201) (verified)
  - *To reach the next level:* Unpinned; L2 needs versions pinned.
- **C L0:** No extension type (MCP servers, hub skills, hub MCPs) is verified. — searched `rg -n -i sha256|signature|verify_hash|integrity` in `src/agentscope/mcp src/agentscope/app/hub` → 0 hits (verified)
  - *To reach the next level:* Nothing verified; L1 needs at least one extension type verified.
- **D L1:** Nothing third-party is enabled by default in the SDK, but workspace MCP declarations are persisted in the agent-visible workdir and reloaded without a consent step. — [src/agentscope/workspace/_base.py:460-462](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/workspace/_base.py#L460-L462); [src/agentscope/workspace/_base.py:951](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/workspace/_base.py#L951) (verified)
  - *To reach the next level:* A workspace file can add a server; L2 needs explicit installation only.
- **B L2:** Stdio MCP servers are separate processes; with env unset the MCP SDK passes only a default minimal environment (library behaviour inferred), though they still run as the same OS user. — [src/agentscope/mcp/_mcp_client.py:198-201](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/mcp/_mcp_client.py#L198-L201) (inferred)
  - *To reach the next level:* Same user with full filesystem access; L3 needs per-extension sandboxing.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.25 (high)

Model-provider API keys are held as pydantic SecretStr, which keeps them out of reprs, and the framework ships no telemetry. Beyond that there is no secret handling: shell subprocesses inherit the full environment (including those keys), the Read tool and `cat` are auto-approved for any path such as `~/.aws/credentials` or `/proc/self/environ`, and nothing redacts secrets from tool output before it reaches the model or logs.

- **S L1:** API keys are SecretStr on credential objects; no log or tool-output redaction exists. — [src/agentscope/credential/_dashscope.py:28](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/credential/_dashscope.py#L28); searched `rg -n -i redact|mask_secret|scrub` in `src/agentscope` → 18 hits (hits are Anthropic redacted_thinking handling, team-roster scrubbing, and a service API comment about shared credential entries; none redacts secrets from tool output, logs, or model input) (verified)
  - *To reach the next level:* Masking covers only credential reprs; L2 needs log filters on main paths.
- **C L1:** Only the credential-object path is protected; subprocess environments, tool results sent to the model, and logs are not. — searched `rg -n env=` in `src/agentscope/tool/_builtin/_backend.py` → 0 hits (LocalBackend never passes env=, so every Bash subprocess inherits the agent process's full os.environ); [src/agentscope/tool/_builtin/_read.py:126](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_read.py#L126) (verified)
  - *To reach the next level:* One path protected; L2 needs logs and transcripts covered.
- **D L2:** No telemetry SDK is initialised and the default log level is INFO. — [src/agentscope/_logging.py:47](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/_logging.py#L47); searched `rg -n -i sentry|posthog` in `src/agentscope` → 0 hits (verified)
  - *To reach the next level:* No redaction to keep on; L3 needs redaction always on.
- **B L0:** Long-lived provider keys and any ambient cloud credentials are reachable by every Bash subprocess and by auto-approved reads. — searched `rg -n env=` in `src/agentscope/tool/_builtin/_backend.py` → 0 hits (LocalBackend never passes env=, so every Bash subprocess inherits the agent process's full os.environ); [src/agentscope/tool/_builtin/_read.py:126](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_read.py#L126); [src/agentscope/permission/_engine.py:683-690](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/permission/_engine.py#L683-L690) (verified)
  - *To reach the next level:* Long-lived high-privilege keys exposed; L1 needs keys kept from subprocesses or narrowly scoped.
- **Cap:** none

### C9 Audit & traceability — 0.35 (medium)

Out of the box the SDK keeps no durable record: tool calls, results and denials live in the agent's in-memory state and the event stream handed to the caller, and are lost when the process exits unless the developer persists them. An opt-in OpenTelemetry tracing middleware records reply, model-call and tool-execution spans, but it does not hook the permission check, so approvals and denials are not traced, and export is best-effort. The agent service stores session state in its database.

- **default configuration** (default; raw 0.23, cap G1 → 0.23)
  - **S L1:** Tool calls and results are kept as structured blocks in in-memory agent state; nothing is written durably by default. — [src/agentscope/agent/_agent.py:2626](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/agent/_agent.py#L2626) (verified)
    - *To reach the next level:* No durable transcript; L2 needs a structured record of every tool call written somewhere.
  - **C L2:** Every tool call (built-in, MCP, function) and every denial lands in the in-memory context. — [src/agentscope/agent/_agent.py:2580-2589](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/agent/_agent.py#L2580-L2589) (verified)
    - *To reach the next level:* Approvals/approver identity are not recorded; L3 needs approvals and denials recorded with sub-agents.
  - **D L0:** Persisting any record is opt-in. — searched `rg -n FileHandler` in `src/agentscope/agent src/agentscope/tool` → 0 hits (verified)
    - *To reach the next level:* Opt-in only; L1 needs a record on by default.
  - **B L0:** The record lives in process memory and is lost on crash. — [src/agentscope/agent/_agent.py:2626](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/agent/_agent.py#L2626) (verified)
    - *To reach the next level:* Lost on crash; L1 needs best-effort persistence.
- **opt-in TracingMiddleware (OpenTelemetry)** (alt; raw 0.35, cap G1 → 0.35) ← counted
  - **S L2:** Structured OTel spans for replies, model calls and tool executions. — [src/agentscope/middleware/_tracing/_trace.py:117](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/middleware/_tracing/_trace.py#L117); [src/agentscope/middleware/_tracing/_trace.py:312](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/middleware/_tracing/_trace.py#L312) (verified)
    - *To reach the next level:* No approver/principal attribution; L3 needs actor attribution and correlation across sub-agents.
  - **C L2:** on_acting wraps every executed tool call, but permission decisions are not traced. — searched `rg -n on_check_permission` in `src/agentscope/middleware/_tracing` → 0 hits (verified)
    - *To reach the next level:* Approvals and denials are not recorded; L3 needs them.
  - **D L0:** Off by default. — [src/agentscope/agent/_agent.py:218-219](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/agent/_agent.py#L218-L219) (verified)
    - *To reach the next level:* Off by default; L1 needs it on.
  - **B L1:** Span export is best-effort through the OTel SDK. — [src/agentscope/middleware/_tracing/_trace.py:312](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/middleware/_tracing/_trace.py#L312) (inferred)
    - *To reach the next level:* Best-effort; L2 needs per-action flush with surfaced errors.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C10 Limits & kill switch — 0.45 (high)

Each reply is capped at 50 reasoning-acting iterations and each Bash command at a 2-minute default timeout (the model may raise it to at most 10 minutes). There is no wall-clock or spend cap by default; a token-budget middleware exists but is opt-in, and MCP tool calls have no timeout unless one is configured. Interrupting the agent cancels the loop, but the Bash backend only kills the shell on timeout and never kills a process group, so background or child processes can keep running.

- **S L2:** Iteration cap plus per-execution Bash timeout enforced in code; no wall-clock or default cost cap. — [src/agentscope/agent/_config.py:365-367](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/agent/_config.py#L365-L367); [src/agentscope/tool/_builtin/_bash.py:699](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_bash.py#L699); [src/agentscope/middleware/_budget.py:179](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/middleware/_budget.py#L179) (verified)
  - *To reach the next level:* No default wall-clock or token/cost cap; L3 needs all three plus rate limits.
- **C L2:** The loop cap and Bash timeout apply; MCP tool timeout defaults to None. — [src/agentscope/tool/_adapters.py:281-284](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_adapters.py#L281-L284) (verified)
  - *To reach the next level:* Spawned and delegated work doesn't share one budget; L3 needs sub-agents and processes under the same budget.
- **D L2:** Sensible defaults (50 iters, 120 s) that the operator can change; the model can raise a Bash timeout only up to 600 s. — [src/agentscope/agent/_config.py:365-367](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/agent/_config.py#L365-L367); [src/agentscope/tool/_builtin/_bash.py:699](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_bash.py#L699) (verified)
  - *To reach the next level:* No hard ceilings for delegation; L3 needs the model unable to reset limits by delegating.
- **B L1:** No spend ceiling by default, and stop/timeout kills only the immediate shell with no process-group kill, so background children survive. — [src/agentscope/tool/_builtin/_backend.py:828-829](https://github.com/agentscope-ai/agentscope/blob/72f3f6fa0b2fc38b8517f408ab616f0f2bd229e6/src/agentscope/tool/_builtin/_backend.py#L828-L829); searched `rg -n killpg|start_new_session|process_group` in `src/agentscope/tool/_builtin/_backend.py` → 0 hits (verified)
  - *To reach the next level:* Stopping can leave work running; L2 needs moderate ceilings and a stop that ends the work.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: tool and MCP results enter context unmarked (agent/_agent.py:2626) · [B] sensitive data/systems: Read auto-allowed for any path (tool/_builtin/_read.py:126; permission/_engine.py:683) and full os.environ in Bash (tool/_builtin/_backend.py:799) · [C] state change / egress: host shell (tool/_builtin/_bash.py:177) and read-only-labelled MCP tools auto-allowed (tool/_adapters.py:307) · Same default session? Yes

## Highest-impact improvements
1. Stop trusting server-declared readOnlyHint for auto-approval; ask for MCP tools unless the operator allowlists them. — C2 C L1→L2, +0.075 before caps (Playbook 5)
2. Ensure only operator configuration can relax approval for agent-service runs. — C2 D L1→L3, +0.100 before caps (Playbook 5)
3. Pass a scrubbed env to LocalBackend subprocesses (no provider keys or cloud credentials). — C8 B L0→L1, +0.050 before caps (Playbook 4)
4. Default the SDK Bash tool to a sandboxed workspace backend with an explicit unsafe-local flag. — C4 D L0→L3, +0.150 before caps (Playbook 3, step 1)
5. Persist a structured tool-call/approval log by default. — C9 D L0→L2, +0.100 before caps (Playbook 1, step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the SDK Agent quickstart; the agent service (FastAPI, multi-tenant, channels for Feishu/DingTalk/Discord) was reviewed only for scheduler, toolkit assembly and sub-agent permission inheritance. Channel trigger handling, who can answer confirmations in IM channels, RBAC/resource sharing, and storage backends were not examined.
- Some Bash auto-allow findings and MCP SDK environment scrubbing are inferred from documented tool/library behaviour, not observed.
- Workspace sandboxes other than E2B and Docker (Bubblewrap, Apple Container, Daytona, K8s, OpenSandbox) were not examined in detail; the web UI, realtime agents, A2A agent, RAG and ReMe/agentic-memory middlewares were only skimmed.
- No reviewer-steering text was found in the repository.
