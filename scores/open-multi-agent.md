# Defense-in-Depth Score: Open Multi-Agent (OMA)

**Repo:** https://github.com/open-multi-agent/open-multi-agent · **Commit:** `75a67581c98df3bce3c90035dcaf445a989155d2` (@open-multi-agent/core 1.21.1) · **Reviewed:** 2026-10-04
**What it is:** Self-hosted TypeScript multi-agent orchestration runtime with task DAGs, tool calling, MCP, and durable approvals.
**Category:** Agent Frameworks
**Scored configuration:** @open-multi-agent/core library with public constructor defaults: new OpenMultiAgent() and AgentConfig without tools, onToolCall, journal, budgets, or a custom shell executor.
**Agent surface (default):** code execution opt-in · filesystem write opt-in · network egress opt-in · external credentials opt-in · persistent memory opt-in · untrusted input opt-in · third party extensions opt-in · sub agents yes · external communication opt-in

## Score: 3.4 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C2 | Approval gates | L3 | L2 | L0 | L4 | 0.57 | G1 | **0.50** | High |
| C3 | Tool & action scoping | L2 | L2 | L3 | L1 | 0.50 | — | **0.50** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C7 | Third-party extensions | L1 | L0 | L2 | L1 | 0.23 | — | **0.23** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C9 | Audit & traceability | L3 | L3 | L0 | L1 | 0.50 | G1 | **0.50** (alt) | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |


OMA has unusually honest, well-built pieces: built-in tools are denied by default, the file tools are confined to a workspace directory, and its optional approval gate binds each decision to a hash of the exact call. But that gate and the consequential-action confirmation are both off by default, and the bash tool runs directly on the host with no sandbox. Once a developer grants bash, MCP or custom tools, injected content can make an agent leak data and take irreversible actions without any human involved.

## Critical gaps
- Granted bash runs `bash -c` on the host as the same user, with HOME and network reachable; the authors state it is not a sandbox. (ASI05, T11; C4) — [packages/core/src/tool/shell/local.ts:42](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/shell/local.ts#L42); [packages/core/src/tool/shell/local.ts:30](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/shell/local.ts#L30)
- Tool and MCP outputs enter context with no taint or approval tie, and approval is off by default, so a hijacked tool-using agent can exfiltrate and act irreversibly unattended (C5-WORSTCASE). (ASI01, LLM01, T6; C5) — [packages/core/src/agent/runner.ts:1371-1373](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/agent/runner.ts#L1371-L1373); [packages/core/src/orchestrator/orchestrator.ts:588](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/orchestrator/orchestrator.ts#L588)

## Criterion details

### C1 Identity & least privilege — 0.30 (high)

The framework holds only the LLM provider keys you give it and has no identity or authorization layer of its own. Agents can carry a per-agent credentials bag, but the code itself calls it a convenience, not a boundary: tool code runs in the same process and can read process.env. The bash tool gets a scrubbed environment. The external-agent backends (ACP and process) get the parent's whole environment. A fresh agent has no tools, so the default authority is small, but nothing narrows what a granted tool can reach.

- **S L1:** Per-agent credentials bag exists but is an in-process scoping convenience; no authorization check maps requests to a least-privilege policy. — [packages/core/src/types.ts:654](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/types.ts#L654); [packages/core/src/types.ts:1085](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/types.ts#L1085) (verified)
  - *To reach the next level:* No deterministic authorization layer or credential that is actually narrowed per agent or per tool.
- **C L1:** bash gets an allowlisted, credential-stripped environment, but ACP and process backends spread the full process.env into the child. — [packages/core/src/tool/shell/local.ts:45](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/shell/local.ts#L45); [packages/core/src/tool/shell/local.ts:123-124](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/shell/local.ts#L123-L124); [packages/core/src/agent/acp-backend.ts:421](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/agent/acp-backend.ts#L421); [packages/core/src/agent/process-backend-io.ts:36](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/agent/process-backend-io.ts#L36) (verified)
  - *To reach the next level:* ACP/process backends and custom tools still see ambient credentials; no common authorization layer.
- **D L2:** Default agent resolves to zero built-in tools; granting write/exec or the 'full' preset is a plain operator config change. — [packages/core/src/tool/grants.ts:66](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/grants.ts#L66); [packages/core/src/agent/runner.ts:1848](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/agent/runner.ts#L1848) (verified)
  - *To reach the next level:* Elevation (toolPreset 'full', defaultToolPreset) is a silent config change rather than a time-bounded explicit elevation.
- **B L1:** If a granted bash or custom tool is misused, it runs as the OS user: HOME is passed through, so on-disk credentials (cloud, gh, ssh) are reachable, though provider keys in env are stripped from bash. — [packages/core/src/tool/shell/local.ts:45](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/shell/local.ts#L45); [packages/core/src/tool/shell/local.ts:12-14](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/shell/local.ts#L12-L14) (verified)
  - *To reach the next level:* On-disk user credentials remain reachable from a granted bash tool; nothing scopes the agent to one system.
- **Cap:** none

### C2 Approval gates — 0.50 (high)

There is a well-built per-call approval hook: it runs after input validation and before execution, sees the exact parsed arguments, can allow, deny or suspend, and durable approvals are bound to a hash of exactly what the reviewer saw. The hook is not installed by default, and the built-in 'consequential confirmation' guard is also off by default. Delegated agents inherit the hook. External ACP and process backends do not pass through it. A fresh agent has no tools at all, so with the shipped defaults no consequential action can run until the developer grants one.

- **S L3:** Per-call gate receives the exact validated input and a consequential risk flag; durable approvals are hash-bound to the reviewed invocation and rejection is first-class. — [packages/core/src/tool/executor.ts:253-255](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/executor.ts#L253-L255); [packages/core/src/tool/executor.ts:229](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/executor.ts#L229); [packages/core/src/tool/executor.ts:239](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/executor.ts#L239); [packages/core/src/tool/built-in/bash.ts:31](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/built-in/bash.ts#L31) (verified)
  - *To reach the next level:* No argument-level allow/deny/escalate policy engine in core; the gate's policy is entirely application code.
- **C L2:** The gate sits in the ToolExecutor and covers built-in, custom, MCP and delegated-agent tools (gate inherited); ACP/process backends replace the loop and skip it, and MCP tools are never flagged consequential. — [packages/core/src/tool/executor.ts:214](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/executor.ts#L214); [packages/core/src/orchestrator/agent-config.ts:45](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/orchestrator/agent-config.ts#L45); [packages/core/src/agent/runner.ts:1848](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/agent/runner.ts#L1848); searched `rg -n consequential` in `packages/core/src/tool/mcp.ts` → 0 hits (MCP tools never carry the consequential flag) (verified)
  - *To reach the next level:* External-agent backends bypass the gate and MCP/custom tools default to non-consequential.
- **D L0:** No onToolCall gate is set by default and requireConsequentialConfirmation defaults to false. — [packages/core/src/orchestrator/orchestrator.ts:587](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/orchestrator/orchestrator.ts#L587); [packages/core/src/orchestrator/orchestrator.ts:588](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/orchestrator/orchestrator.ts#L588) (verified)
  - *To reach the next level:* Approval is opt-in; consequential tools run unattended once granted.
- **B L4:** In the default configuration an agent receives zero built-in tools (enforced in grant resolution and again at call time), so a wrongly approved or ungated call has nothing consequential to reach; writes appear only after operator configuration. — [packages/core/src/tool/grants.ts:66](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/grants.ts#L66); [packages/core/src/agent/runner.ts:1848](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/agent/runner.ts#L1848) (verified)
- **Cap:** G1 — The per-call gate and consequential confirmation exist but are both off in the default OpenMultiAgent configuration.

### C3 Tool & action scoping — 0.50 (high)

Every tool's input is validated against a Zod schema, and the built-in file tools enforce absolute, symlink-resolved containment inside a default workspace directory. bash takes a raw command string, and MCP tools accept any input. Built-in tools are denied by default and the model cannot grant itself new ones, but custom tools are granted just by registering them. Once bash is granted it reaches the whole machine.

- **S L2:** Schema validation on every call plus realpath containment for file tools, but the most powerful tool (bash) is raw passthrough. — [packages/core/src/tool/built-in/path-safety.ts:89](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/built-in/path-safety.ts#L89); [packages/core/src/tool/built-in/bash.ts:41](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/built-in/bash.ts#L41); [packages/core/src/tool/mcp.ts:432](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/mcp.ts#L432) (verified)
  - *To reach the next level:* bash and MCP tools have no allowlist-style argument validation.
- **C L2:** All built-in file tools validate paths; bash does not, and MCP tools use z.any() input schemas. — [packages/core/src/tool/built-in/path-safety.ts:89](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/built-in/path-safety.ts#L89); [packages/core/src/tool/mcp.ts:432](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/mcp.ts#L432); [packages/core/src/tool/built-in/bash.ts:41](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/built-in/bash.ts#L41) (verified)
  - *To reach the next level:* No shared validation layer for extension tools; bash unvalidated.
- **D L3:** Default grant resolution gives zero built-in tools; read-only, readwrite and full presets must be chosen explicitly, and the model cannot load tools. — [packages/core/src/tool/grants.ts:66](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/grants.ts#L66); [packages/core/src/tool/grants.ts:85](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/grants.ts#L85) (verified)
  - *To reach the next level:* Custom/runtime tools are granted on registration, with no per-task allowlist by default.
- **B L1:** File tools are confined to <cwd>/.agent-workspace by default, but a granted bash tool has whole-machine reach. — [packages/core/src/orchestrator/orchestrator.ts:569](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/orchestrator/orchestrator.ts#L569); [packages/core/src/tool/built-in/path-safety.ts:37](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/built-in/path-safety.ts#L37); [packages/core/src/tool/shell/local.ts:42](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/shell/local.ts#L42) (verified)
  - *To reach the next level:* No quantity bounds; bash is unconfined.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

The only code-execution path is the bash tool. By default it runs `bash -c` on the host as the same user, and the code itself says this is not a sandbox. A pluggable ShellExecutor interface lets developers send commands elsewhere, but core ships no container or VM executor. The environment is scrubbed of credential-named variables and the process tree is killed on timeout. Still, the home directory, the network and the whole filesystem are reachable.

- **S L0:** LocalShellExecutor spawns bash -c on the host; the authors state it is not a sandbox or security boundary. — [packages/core/src/tool/shell/local.ts:42](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/shell/local.ts#L42); [packages/core/src/tool/shell/local.ts:30](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/shell/local.ts#L30); [packages/core/src/tool/built-in/bash.ts:23](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/built-in/bash.ts#L23) (verified)
  - *To reach the next level:* No isolation primitive (container, OS sandbox, microVM) shipped.
- **C L0:** The main exec path is unsandboxed; no path is isolated. — [packages/core/src/tool/built-in/bash.ts:23](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/built-in/bash.ts#L23); searched `rg -n -i 'docker|firecracker|gvisor|seccomp|landlock|sandbox-exec|bwrap'` in `packages/core/src` → 1 hits (single hit is a comment in the bash risk classifier saying it is not a container/VM/seccomp) (verified)
  - *To reach the next level:* No execution path goes through an isolation boundary.
- **D L0:** No sandbox exists to be on by default; the default executor is the host shell. — [packages/core/src/tool/built-in/bash.ts:23](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/built-in/bash.ts#L23) (verified)
  - *To reach the next level:* No sandbox on by default.
- **B L0:** Commands run as the host user with HOME passed through and unrestricted network; only env-var credentials are stripped. — [packages/core/src/tool/shell/local.ts:45](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/shell/local.ts#L45); [packages/core/src/tool/shell/local.ts:12-14](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/shell/local.ts#L12-L14) (verified)
  - *To reach the next level:* Home directory and network remain reachable; no workspace-only mount or egress limit.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

Tool results, MCP outputs and MCP tool descriptions go into the conversation exactly as received, with nothing marking them as untrusted, and no rule ties having read untrusted content to stricter handling. The LLM egress policy controls only which model endpoints the adapters call; bash, MCP and custom tools are outside it. The framework's own docs show agents with read/write and shell tools, MCP servers holding GitHub tokens, and file contents flowing to the model, and nothing breaks that combination. A hijacked agent could therefore leak data and take irreversible actions without anyone approving them.

- **S L0:** No structural limit: tool output is appended as a normal tool_result message with no taint tracking or approval tie. — [packages/core/src/agent/runner.ts:1371-1373](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/agent/runner.ts#L1371-L1373); searched `rg -n -i 'untrusted|prompt.injection|taint'` in `packages/core/src` → 7 hits (comments only (dashboard escaping, routing profiler prompt, file-edit docstring); none limits tool-result influence) (verified)
  - *To reach the next level:* Nothing gates egress or state change after untrusted content is read.
- **C L0:** Untrusted sources are not distinguished; MCP tool descriptions are passed to the model verbatim. — [packages/core/src/agent/runner.ts:1371-1373](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/agent/runner.ts#L1371-L1373); [packages/core/src/tool/mcp.ts:431](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/mcp.ts#L431) (verified)
  - *To reach the next level:* No untrusted source is distinguished from principal input.
- **D L0:** No control exists to be on by default. — [packages/core/src/agent/runner.ts:1371-1373](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/agent/runner.ts#L1371-L1373) (verified)
  - *To reach the next level:* No default untrusted-input control.
- **B L0:** Per the framework rule, documented normal use combines untrusted input, private data and egress/state change (bash, MCP with tokens), and the default approval is off, so leak plus irreversible action can happen unattended. — [packages/core/src/orchestrator/orchestrator.ts:588](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/orchestrator/orchestrator.ts#L588); [packages/core/src/tool/shell/local.ts:42](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/shell/local.ts#L42) (verified)
  - *To reach the next level:* No mechanism forces human approval for egress or irreversible actions once untrusted content is read.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.50 (high)

Nothing is loaded automatically from the workspace: no .env, no instruction files, no project settings. The CLI reads only config files you name explicitly. Team shared memory is opt-in and kept in memory by default. The orchestrator writes each agent's output there under that agent's name, and later agents see it as a 'Shared Team Memory' summary in their context. Writes are not validated, so text injected into one agent's output reaches teammates' tool-using turns for as long as that team object lives.

- **S L2:** No auto-loaded workspace config; memory writes are framework-made and attributed by agent namespace, but not validated and injected as plain context. — [packages/core/src/orchestrator/task-execution.ts:1595](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/orchestrator/task-execution.ts#L1595); [packages/core/src/memory/shared.ts:367](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/memory/shared.ts#L367); searched `rg -n -i 'dotenv|loadEnvFile|AGENTS\.md|CLAUDE\.md|\.cursorrules'` in `packages/core/src` → 1 hits (single hit is a comment referencing docs/external-agents.md; no auto-loading) (verified)
  - *To reach the next level:* Memory writes are not gated or validated and entries are not framed as data.
- **C L2:** Shared memory is namespaced and attributed; checkpoints and summaries carry no provenance controls. — [packages/core/src/orchestrator/task-execution.ts:1595](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/orchestrator/task-execution.ts#L1595); [packages/core/src/memory/shared.ts:367](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/memory/shared.ts#L367) (verified)
  - *To reach the next level:* Checkpoint stores and summaries are not provenance-tagged or validated.
- **D L2:** Shared memory is opt-in and a separate in-process store per Team instance. — [packages/core/src/team/team.ts:121-122](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/team/team.ts#L121-L122) (verified)
  - *To reach the next level:* No per-tenant storage separation or retention limit by default.
- **B L2:** Poisoned entries persist for the team object's lifetime (and across processes with a FileStore) and can steer later tool-using agents. — [packages/core/src/orchestrator/task-execution.ts:1595](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/orchestrator/task-execution.ts#L1595); [packages/core/src/memory/shared.ts:367](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/memory/shared.ts#L367) (verified)
  - *To reach the next level:* Poisoned memory can influence ungated tool use; no review or rollback.
- **Cap:** none

### C7 Third-party extensions — 0.23 (high)

The framework never installs or enables extensions itself. MCP servers and external agent CLIs run only from a command the developer writes. There is no pinning, hash check or re-approval when a server's tools change, and tool descriptions are taken as given. MCP children get the developer-supplied environment (or the MCP SDK's minimal default). ACP and process backend children inherit the parent's full environment.

- **S L1:** Extensions are developer-chosen commands with no version pinning or integrity check in the loader. — [packages/core/src/tool/mcp.ts:406](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/mcp.ts#L406); [packages/core/src/tool/mcp.ts:403-405](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/mcp.ts#L403-L405) (verified)
  - *To reach the next level:* No pinning or integrity verification of MCP servers or external agent commands.
- **C L0:** No extension type is verified. — searched `rg -n 'sha256|integrity|signature'` in `packages/core/src/tool/mcp.ts packages/core/src/agent/acp-backend.ts packages/core/src/agent/process-backend-io.ts` → 0 hits (verified)
  - *To reach the next level:* No extension type is verified.
- **D L2:** Nothing third-party is enabled by default; adding a server requires explicit developer code naming the command. — [packages/core/src/tool/mcp.ts:406](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/mcp.ts#L406) (verified)
  - *To reach the next level:* Capped one level above S; no display of what will run or permissions requested at add time.
- **B L1:** ACP/process backend children are separate processes with the full parent environment; MCP children get config.env or the SDK default. — [packages/core/src/agent/acp-backend.ts:421](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/agent/acp-backend.ts#L421); [packages/core/src/agent/process-backend-io.ts:36](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/agent/process-backend-io.ts#L36) (verified)
  - *To reach the next level:* External-agent children are not given a scrubbed environment.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.45 (high)

Secrets come from environment variables or config. A regex redactor scrubs credential patterns from bash output before it reaches the model, from trace and dashboard payloads, and from the credentials bag. bash children get an allowlisted environment. No telemetry is sent anywhere by default. Redaction in the journal and in persisted stores is opt-in, checkpoints and shared memory store agent output verbatim, file_read output is not redacted, and the ACP/process backends get the full environment.

- **S L2:** Regex/name-based redaction applied to bash output, traces and dashboards; credentials key auto-redacted. — [packages/core/src/tool/built-in/bash.ts:75](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/built-in/bash.ts#L75); [packages/core/src/utils/redaction.ts:3-4](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/utils/redaction.ts#L3-L4) (verified)
  - *To reach the next level:* No secret manager or encryption at rest; redaction is pattern-based and not applied to all model-bound tool results.
- **C L2:** Covers bash model-bound output, traces, dashboard and bash env; not file_read results, checkpoints, shared memory or ACP/process env. — [packages/core/src/tool/built-in/bash.ts:75](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/built-in/bash.ts#L75); [packages/core/src/journal/jsonl-journal.ts:75](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/journal/jsonl-journal.ts#L75); [packages/core/src/agent/acp-backend.ts:421](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/agent/acp-backend.ts#L421) (verified)
  - *To reach the next level:* Model-bound file/custom/MCP tool results and persisted state are not redacted by default.
- **D L2:** No telemetry SDK; trace tool I/O capture is opt-in; journal redaction must be enabled explicitly. — [packages/core/src/journal/jsonl-journal.ts:75](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/journal/jsonl-journal.ts#L75); searched `rg -n -i 'posthog|sentry|analytics\.track|telemetry\.send|mixpanel'` in `packages/core/src` → 0 hits (verified)
  - *To reach the next level:* Redaction of persisted journal/checkpoint state is not always on.
- **B L1:** Long-lived provider API keys live in the process environment, reachable by custom tools and ACP/process children. — [packages/core/src/agent/acp-backend.ts:421](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/agent/acp-backend.ts#L421); [packages/core/src/types.ts:654](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/types.ts#L654) (verified)
  - *To reach the next level:* No short-lived or scoped keys.
- **Cap:** none

### C9 Audit & traceability — 0.50 (high)

By default every run returns an in-memory list of tool calls (tool name, input, output, duration) to the calling code, but nothing is written down and it disappears if the process crashes. The opt-in run journal is a much better record. It is append-only and structured, with run, task and agent attribution, approval request and decision events, memory-write events and lineage hashes, and it can be checked offline. It is best-effort and not hash-chained against tampering.

- **default configuration** (default; raw 0.28 → 0.28)
  - **S L1:** Default ToolCallRecord holds tool name, input, output and duration only; no timestamps, status or actor. — [packages/core/src/types.ts:1354-1358](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/types.ts#L1354-L1358) (verified)
    - *To reach the next level:* Default record lacks timestamps, result status and actor attribution.
  - **C L2:** Every tool call through the runner loop is recorded; ACP/process backends replace the loop. — [packages/core/src/types.ts:1354-1358](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/types.ts#L1354-L1358); [packages/core/src/agent/runner.ts:1970-1974](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/agent/runner.ts#L1970-L1974) (verified)
    - *To reach the next level:* Approvals/denials and external backends are not in the default record.
  - **D L1:** The record exists by default but only as an in-process value returned to the application. — [packages/core/src/types.ts:1354-1358](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/types.ts#L1354-L1358); [packages/core/src/orchestrator/orchestrator.ts:578](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/orchestrator/orchestrator.ts#L578) (verified)
    - *To reach the next level:* Default record is not stored outside the agent process.
  - **B L0:** In-memory only by default; lost on crash. — [packages/core/src/orchestrator/orchestrator.ts:578](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/orchestrator/orchestrator.ts#L578) (verified)
    - *To reach the next level:* Records are not flushed durably per action by default.
- **opt-in run journal (JsonlRunJournal)** (alt; raw 0.50, cap G1 → 0.50) ← counted
  - **S L3:** Structured events with timestamps, runId, taskId, agentName, approval request/decision with reviewer, and content-hash lineage. — [packages/core/src/journal/events.ts:47-55](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/journal/events.ts#L47-L55); [packages/core/src/journal/events.ts:229-237](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/journal/events.ts#L229-L237) (verified)
    - *To reach the next level:* Not hash-chained or signed; no tamper-evident storage.
  - **C L3:** Tool calls/results, approvals, memory writes, context rewrites and checkpoints are journaled. — [packages/core/src/journal/events.ts:172-178](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/journal/events.ts#L172-L178); [packages/core/src/journal/events.ts:222-226](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/journal/events.ts#L222-L226) (verified)
    - *To reach the next level:* Credential use and configuration changes are not journaled.
  - **D L0:** Off unless the application supplies a journal instance. — [packages/core/src/orchestrator/orchestrator.ts:578](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/orchestrator/orchestrator.ts#L578) (verified)
    - *To reach the next level:* Journal is opt-in.
  - **B L1:** Writes are best-effort; a failed append does not stop the run. — [packages/core/src/journal/journal.ts:11](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/journal/journal.ts#L11) (verified)
    - *To reach the next level:* Writes do not fail closed and are not durable per action.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C10 Limits & kill switch — 0.40 (high)

An agent run stops after 10 turns by default. Delegation is capped at depth 3 and concurrency at 5. bash commands time out after 30 seconds by default and the whole process tree is killed on timeout or abort. The model can set any bash timeout it likes, though. Whole-run and per-call wall-clock limits are unset by default, and token or cost ceilings exist only if configured. Stopping via AbortSignal interrupts in-flight shell commands.

- **S L2:** Turn cap plus per-execution bash timeout enforced in code; token/cost/wall-clock caps exist but are opt-in. — [packages/core/src/agent/runner.ts:592](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/agent/runner.ts#L592); [packages/core/src/tool/built-in/bash.ts:56](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/built-in/bash.ts#L56); [packages/core/src/orchestrator/orchestrator.ts:571-572](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/orchestrator/orchestrator.ts#L571-L572) (verified)
  - *To reach the next level:* Wall-clock and token/cost caps are not on by default; no rate limits on side-effecting tools.
- **C L2:** Limits act on the agent loop and bash; delegation depth and concurrency are capped, delegated token usage is added to the parent. — [packages/core/src/orchestrator/run-context.ts:41-42](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/orchestrator/run-context.ts#L41-L42); [packages/core/src/tool/built-in/delegate.ts:70-71](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/built-in/delegate.ts#L70-L71) (verified)
  - *To reach the next level:* Delegated runs get a fresh turn budget; no shared per-run turn or time budget by default.
- **D L1:** maxTurns defaults to 10, but the model chooses the bash timeout with no upper bound. — [packages/core/src/tool/built-in/bash.ts:56](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/built-in/bash.ts#L56); [packages/core/src/tool/built-in/bash.ts:42-48](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/built-in/bash.ts#L42-L48) (verified)
  - *To reach the next level:* The model can raise its own per-command timeout.
- **B L1:** No spend ceiling or run deadline by default; a model-chosen long bash timeout can run for hours; abort does kill the process tree. — [packages/core/src/agent/runner.ts:690-691](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/agent/runner.ts#L690-L691); [packages/core/src/tool/shell/local.ts:72](https://github.com/open-multi-agent/open-multi-agent/blob/75a67581c98df3bce3c90035dcaf445a989155d2/packages/core/src/tool/shell/local.ts#L72) (verified)
  - *To reach the next level:* No default per-run time or cost ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: tool and MCP results appended as tool_result messages (packages/core/src/agent/runner.ts:1371) · [B] sensitive data/systems: per-agent credentials and workspace files reachable from tools (packages/core/src/types.ts:1085) · [C] state change / egress: granted bash on host with network (packages/core/src/tool/shell/local.ts:42) · Same default session? Yes

## Highest-impact improvements
1. Default requireConsequentialConfirmation to true so consequential tools are denied without an explicit gate decision. — C2 D L0→L3, +0.150 before caps (Playbook 5)
2. Clamp the model-supplied bash timeout to a configurable maximum and set a default run timeoutMs. — C10 D L1→L2, +0.050 before caps (Playbook 3 step 3)
3. Pass an allowlisted environment to ACP/process backends instead of spreading process.env. — C7 B L1→L2, +0.050 before caps (Playbook 3)
4. Once a tool result has entered a run, force consequential calls through the gate (fail closed when none is configured). — C5 S L0→L2, +0.150 before caps (Playbook 1)
5. Ship a hardened container ShellExecutor and use it as the default for granted bash. — C4 S L0→L3, +0.225 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of commit 75a6758 only; nothing was executed, installed, or probed.
- Framework scored by its defaults: absent primitives score L0 even where a developer could add them (e.g. a custom ShellExecutor or onToolCall policy).
- MCP child environment when config.env is omitted is inferred from @modelcontextprotocol/sdk behaviour (getDefaultEnvironment allowlist); SDK source not reviewed.
- packages/create-oma-app templates, release-bot, otel and bench were not scored; only packages/core.
- No reviewer-directed prompt injection found in the repo.
