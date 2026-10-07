# Defense-in-Depth Score: Microsoft Agent Framework

**Repo:** https://github.com/microsoft/agent-framework · **Commit:** `e9857bd75a21843803431a2e7dfa41ee9b017ac7` · **Reviewed:** 2026-10-04
**What it is:** Microsoft's multi-language (Python/.NET) framework for building AI agents and multi-agent workflows.
**Category:** Agent Frameworks
**Scored configuration:** Python package defaults: Agent with developer-registered FunctionTools/MCP tools using public constructor defaults, plus bundled agent_framework_tools shell tools with their defaults.
**Agent surface (default):** code execution opt-in · filesystem write opt-in · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents opt-in · external communication opt-in

## Score: 3.4 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C2 | Approval gates | L3 | L1 | L0 | L0 | 0.30 | G1 | **0.30** | High |
| C3 | Tool & action scoping | L2 | L2 | L3 | L1 | 0.50 | — | **0.50** | High |
| C4 | Code-execution isolation | L3 | L1 | L3 | L3 | 0.60 | G1 | **0.50** (alt) | High |
| C5 | Untrusted input blast radius | L3 | L2 | L2 | L2 | 0.57 | G1 | **0.50** (alt) | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C7 | Third-party extensions | L1 | L0 | L2 | L2 | 0.28 | — | **0.28** | Medium |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |
| C9 | Audit & traceability | L2 | L2 | L0 | L1 | 0.35 | G1 | **0.35** | High |
| C10 | Limits & kill switch | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |


Microsoft Agent Framework ships well-engineered safety primitives (exact-call approvals bound to the session, a hardened Docker shell, a Hyperlight sandbox, and an information-flow-control layer against prompt injection), but almost all are opt-in. With constructor defaults, registered and MCP tools run without approval, tool results are not treated as untrusted, and tools use the developer's ambient credentials. The bundled local shell tool is the exception: it requires approval by default, but runs on the host with the full process environment.

## Critical gaps
- In the default configuration nothing distinguishes untrusted tool/MCP results from instructions, and registered tools run without approval, so an injected agent holding the developer's credentials can both exfiltrate and act unattended. (ASI01, LLM01; C5) — [python/packages/core/agent_framework/_tools.py:667](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_tools.py#L667); [python/packages/core/agent_framework/security.py:3050](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/security.py#L3050)

## Criterion details

### C1 Identity & least privilege — 0.05 (high)

The framework runs every tool with whatever credentials the developer hands the chat client or the tool, and offers no primitive to scope, downscope, or check authority per request in the default path. The bundled LocalShellTool inherits the full parent process environment by default, so cloud and API credentials in the environment are reachable from shell commands. An opt-in experimental information-flow layer can tag content with tenant/user principals, but that is not an authorization layer and is off by default. The main independent safeguard is that the shell tool requires human approval by default.

- **S L0:** Tools and clients use ambient, developer-supplied credentials (API keys, AzureCliCredential); the core has no token exchange or per-tool credential scoping. — searched `rg -n -S 'on_behalf_of|token_exchange|downscop'` in `python/packages/core/agent_framework` → 0 hits (No credential downscoping or on-behalf-of primitive in core.) (verified)
  - *To reach the next level:* No per-tool or per-capability credential scoping primitive.
- **C L0:** There is no authorization layer in the tool executor; LocalShellTool passes the full os.environ to subprocesses by default. — [python/packages/tools/agent_framework_tools/shell/_tool.py:187](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_tool.py#L187) (verified)
  - *To reach the next level:* No shared authorization check applied to every tool path.
- **D L0:** Default install runs tools with the operator's full authority; least privilege requires developer hardening (clean_env, scoped keys). — [python/packages/tools/agent_framework_tools/shell/_tool.py:156](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_tool.py#L156) (verified)
  - *To reach the next level:* No narrower default identity for tools.
- **B L1:** A hijacked tool path reaches whatever the developer's credentials and the inherited environment allow; the default per-call approval on the shell tool is the surviving independent layer. — [python/packages/tools/agent_framework_tools/shell/_tool.py:187](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_tool.py#L187); [python/packages/tools/agent_framework_tools/shell/_tool.py:160](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_tool.py#L160) (verified)
  - *To reach the next level:* Nothing in the framework restricts credentials to one project or read-only scope.
- **Cap:** none

### C2 Approval gates — 0.30 (high)

The framework has a solid human-approval mechanism: a tool marked always_require pauses the run and returns an approval request carrying the exact function call and arguments, and by default an approval response only counts if it matches a request the framework recorded in the session, so replayed or fabricated approvals in message history cannot authorize execution. But the default for any registered function tool, MCP tool, and agent-as-tool is never_require, so approval is opt-in per tool. The bundled shell tools do default to approval (and refuse to turn it off without an explicit acknowledge_unsafe flag), while bundled memory-write tools never require it. There are no checkpoints or rollback for tool side effects.

- **S L3:** Per-call approval requests embed the exact function call (name and arguments); approvals are bound to framework-recorded requests by default and rejection is a first-class outcome; per-tool approval_mode forms risk tiers. — [python/packages/core/agent_framework/_types.py:1421](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_types.py#L1421); [python/packages/core/agent_framework/_tools.py:2425](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_tools.py#L2425); [python/packages/core/agent_framework/_tools.py:1861](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_tools.py#L1861) (verified)
  - *To reach the next level:* No allow/deny/escalate policy on parsed argument values beyond exact-argument standing approvals.
- **C L1:** The gate covers any tool flagged always_require, including MCP and agent-as-tool, but bundled mutating tools such as write_memory are registered never_require. — [python/packages/core/agent_framework/_tools.py:2425](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_tools.py#L2425); [python/packages/core/agent_framework/_harness/_memory.py:1268](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_harness/_memory.py#L1268); [python/packages/core/agent_framework/_agents.py:651](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_agents.py#L651) (verified)
  - *To reach the next level:* Not every built-in mutating tool is gated; unflagged tools pass silently.
- **D L0:** FunctionTool, MCPTool, and Agent.as_tool all default to never_require; only the shell tools and skills default to approval. — [python/packages/core/agent_framework/_tools.py:667](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_tools.py#L667); [python/packages/core/agent_framework/_mcp.py:894](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_mcp.py#L894); [python/packages/tools/agent_framework_tools/shell/_tool.py:160](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_tool.py#L160) (verified)
  - *To reach the next level:* Approval is not on by default for registered tools.
- **B L0:** The framework prevents nothing irreversible: tools registered without approval run arbitrary side effects, and there is no checkpoint/undo for tool actions. — [python/packages/core/agent_framework/_tools.py:667](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_tools.py#L667); searched `rg -n 'checkpoint|rollback|undo'` in `python/packages/core/agent_framework/_tools.py` → 1 hits (Single hit is an unrelated comment ('undo the' in a docstring); no tool-action rollback.) (verified)
  - *To reach the next level:* No rollback, preview, or quantity bounds on consequential actions.
- **Cap:** G1 — Approval is opt-in per tool: FunctionTool defaults approval_mode to never_require.

### C3 Tool & action scoping — 0.50 (high)

Every function tool built from a Python signature or Pydantic model gets typed, recursive argument validation before execution, and MCP calls forward only the parameter names the server declared. The bundled file store resolves paths against its root, rejects symlinks and opens with O_NOFOLLOW. Beyond types, the framework has no central allowlist for paths, hosts, or quantities, and the shipped shell tools accept any command string (their regex policy is explicitly documented as not a security feature). A plain Agent starts with no tools, so least agency is the default.

- **S L2:** Arguments are validated against Pydantic-typed schemas; no central allowlist validation of paths, URLs, or quantities (the file store's resolved-path containment is tool-specific). — [python/packages/core/agent_framework/_tools.py:47](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_tools.py#L47); [python/packages/core/agent_framework/_tools.py:861](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_tools.py#L861); [python/packages/tools/agent_framework_tools/shell/_policy.py:9](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_policy.py#L9) (verified)
  - *To reach the next level:* No framework-level allowlist validation (path containment, host allowlists, numeric bounds) shared by tools.
- **C L2:** Pydantic validation applies to all FunctionTools; MCP tools only get declared-key filtering, not full schema validation. — [python/packages/core/agent_framework/_mcp.py:2940](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_mcp.py#L2940); [python/packages/core/agent_framework/_harness/_file_access.py:1271](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_harness/_file_access.py#L1271) (verified)
  - *To reach the next level:* Extension (MCP) tools are not wrapped by the full validation layer.
- **D L3:** Agent ships with no tools; every tool, including write and exec tools, must be registered explicitly by the developer. — [python/packages/core/agent_framework/_mcp.py:910](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_mcp.py#L910); [python/packages/core/agent_framework/_mcp.py:1507](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_mcp.py#L1507) (verified)
  - *To reach the next level:* Model-driven tool loading exists as an opt-in (MCP progressive disclosure, off by default) and no per-task allowlist is enforced by default.
- **B L1:** Framework does not bound what a registered tool reaches; the bundled LocalShellTool runs any command on the host with only a 30s timeout. — [python/packages/tools/agent_framework_tools/shell/_tool.py:158](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_tool.py#L158); [python/packages/tools/agent_framework_tools/shell/_policy.py:9](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_policy.py#L9) (verified)
  - *To reach the next level:* No workspace scoping or quantity bounds on tool effects by default.
- **Cap:** none

### C4 Code-execution isolation — 0.50 (high)

Code execution is an add-on: the core runs Python tool functions in-process, and the tools package ships a LocalShellTool that runs commands directly on the host with the parent's environment, protected only by its default per-command approval. A well-hardened DockerShellTool (no network, non-root nobody user, read-only root, all capabilities dropped, no-new-privileges, memory/pid limits, and rejection of extra Docker flags that would undo them) and a Hyperlight WebAssembly sandbox are available, but only if the developer chooses them. Other paths such as skill script runners and stdio MCP servers run on the host.

- **default configuration** (default; raw 0.05 → 0.05)
  - **S L0:** LocalShellTool executes commands as a same-user host subprocess; the regex policy is not a boundary. — [python/packages/tools/agent_framework_tools/shell/_tool.py:187](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_tool.py#L187); [python/packages/tools/agent_framework_tools/shell/_policy.py:9](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_policy.py#L9) (verified)
    - *To reach the next level:* No isolation primitive on the default execution tool.
  - **C L0:** No execution path is sandboxed by default. — [python/packages/tools/agent_framework_tools/shell/_tool.py:187](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_tool.py#L187) (verified)
    - *To reach the next level:* The main exec tool is not sandboxed.
  - **D L0:** Sandboxing requires the developer to pick DockerShellTool or Hyperlight. — [python/packages/tools/agent_framework_tools/shell/_tool.py:156](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_tool.py#L156) (verified)
    - *To reach the next level:* Isolation is off by default.
  - **B L1:** Host-equivalent reach with inherited credentials; per-call approval on LocalShellTool is the surviving independent layer. — [python/packages/tools/agent_framework_tools/shell/_tool.py:187](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_tool.py#L187); [python/packages/tools/agent_framework_tools/shell/_tool.py:160](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_tool.py#L160) (verified)
    - *To reach the next level:* No workspace-only mount, scrubbed environment, or network restriction by default.
- **opt-in DockerShellTool (hardened container)** (alt; raw 0.60, cap G1 → 0.50) ← counted
  - **S L3:** Container launched with --network none, non-root user, read-only root, --cap-drop ALL, no-new-privileges, memory and pids limits. — [python/packages/tools/agent_framework_tools/shell/_docker.py:76](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_docker.py#L76); [python/packages/tools/agent_framework_tools/shell/_docker.py:289](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_docker.py#L289); [python/packages/tools/agent_framework_tools/shell/_docker.py:299](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_docker.py#L299) (verified)
    - *To reach the next level:* Not kernel-separated (shared host kernel); microVM/gVisor would be L4.
  - **C L1:** Only DockerShellTool commands run inside; in-process tools, skill script runners and stdio MCP servers run on the host. — [python/packages/core/agent_framework/_mcp.py:3713](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_mcp.py#L3713); [python/packages/tools/agent_framework_tools/shell/_docker.py:76](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_docker.py#L76) (verified)
    - *To reach the next level:* Other execution paths (skill scripts, MCP stdio, LocalShellTool) are not sandboxed.
  - **D L3:** Once chosen, isolation flags are on by default and extra_run_args that would override them are rejected at construction. — [python/packages/tools/agent_framework_tools/shell/_docker.py:458](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_docker.py#L458); [python/packages/tools/agent_framework_tools/shell/_docker.py:76](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_docker.py#L76) (verified)
    - *To reach the next level:* Sandbox policy is set in developer code the agent's process controls; not defined outside it.
  - **B L3:** No network by default, read-only workspace mount, only explicitly passed env vars, resource limits. — [python/packages/tools/agent_framework_tools/shell/_docker.py:76](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_docker.py#L76); [python/packages/tools/agent_framework_tools/shell/_docker.py:299](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_docker.py#L299) (verified)
    - *To reach the next level:* Container is reused per session in persistent mode rather than destroyed per call.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.50 (high)

By default, tool and MCP results enter the conversation with no taint tracking, and since registered tools default to no approval, a prompt-injected agent can combine untrusted input, the developer's data and credentials, and outbound tools without a human. The framework ships an unusually strong opt-in defense, FIDES: content labels propagate through the run, untrusted results can be hidden behind variable references and processed by a quarantined LLM, and a policy middleware blocks (or escalates to approval) any tool not marked as accepting untrusted input once the context is tainted. It is experimental and off unless the developer installs SecureAgentConfig.

- **default configuration** (default; raw 0.00, cap C5-WORSTCASE → 0.00)
  - **S L0:** No structural limit on a hijacked agent in the default Agent; tool results are plain tool messages. — [python/packages/core/agent_framework/security.py:3050](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/security.py#L3050) (verified)
    - *To reach the next level:* No Rule-of-Two enforcement by default.
  - **C L0:** Untrusted sources are not distinguished by default. — [python/packages/core/agent_framework/security.py:3050](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/security.py#L3050) (verified)
    - *To reach the next level:* Tool results are not labeled as untrusted by default.
  - **D L0:** The defense is opt-in and experimental. — [python/packages/core/agent_framework/security.py:3050](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/security.py#L3050) (verified)
    - *To reach the next level:* Off by default.
  - **B L0:** Assuming hijack: unattended exfiltration and irreversible actions through any registered tool, as tools default to never_require. — [python/packages/core/agent_framework/_tools.py:667](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_tools.py#L667); [python/packages/core/agent_framework/_mcp.py:894](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_mcp.py#L894) (verified)
    - *To reach the next level:* Neither egress nor state change requires a human by default.
- **opt-in FIDES information-flow control (SecureAgentConfig)** (alt; raw 0.57, cap G1 → 0.50) ← counted
  - **S L3:** Taint-tracked labels with a policy middleware that blocks or escalates any non-accepting tool once the context is UNTRUSTED, plus quarantined-LLM processing of hidden content. — [python/packages/core/agent_framework/security.py:2810](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/security.py#L2810); [python/packages/core/agent_framework/security.py:2387](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/security.py#L2387) (verified)
    - *To reach the next level:* Plan is not fixed before untrusted data is read; untrusted values are not capability-tracked end to end.
  - **C L2:** Tool results default to UNTRUSTED integrity and MCP results can be labeled, but tool descriptions and peer-agent messages are not covered by label tracking. — [python/packages/core/agent_framework/security.py:1322](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/security.py#L1322) (verified)
    - *To reach the next level:* Tool descriptions and sub-agent messages are not labeled.
  - **D L2:** Once enabled, blocking is on by default; the developer can widen allow_untrusted_tools or switch to approval silently. — [python/packages/core/agent_framework/security.py:2387](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/security.py#L2387) (verified)
    - *To reach the next level:* No warning when exemptions are configured.
  - **B L2:** Tools not declared as accepting untrusted input are blocked or need approval in tainted context; declared-accepting tools remain an egress path. — [python/packages/core/agent_framework/security.py:2810](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/security.py#L2810) (verified)
    - *To reach the next level:* Accepting tools can still carry data out without a human.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C6 Memory, context & configuration integrity — 0.10 (high)

The framework does not auto-load workspace instruction files or a .env from the current directory; .env files are read only when the developer passes a path. Its memory subsystem, however, lets the model write durable memories without approval or validation, and the experimental MemoryContextProvider automatically extracts facts from transcripts (including tool results) and reinjects them into later sessions. Memory stores are namespaced per session or owner in storage paths, which limits cross-user spread.

- **S L0:** write_memory is a never_require tool with no validation, and stored memories are reinjected as context in later turns. — [python/packages/core/agent_framework/_harness/_memory.py:1268](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_harness/_memory.py#L1268); [python/packages/core/agent_framework/_harness/_memory.py:33](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_harness/_memory.py#L33) (verified)
  - *To reach the next level:* No gating, validation, or provenance on memory writes.
- **C L0:** No memory store applies write controls. — [python/packages/core/agent_framework/_harness/_memory.py:1268](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_harness/_memory.py#L1268) (verified)
  - *To reach the next level:* No memory path is controlled.
- **D L1:** File memory is namespaced per session id (or explicit scope) and the cross-session store keys by owner from session state. — [python/packages/core/agent_framework/_harness/_file_memory.py:322](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_harness/_file_memory.py#L322); [python/packages/core/agent_framework/_harness/_memory.py:672](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_harness/_memory.py#L672) (verified)
  - *To reach the next level:* Isolation depends on developer-supplied scope/owner values; the model-writable store is not otherwise protected.
- **B L1:** Poisoned memories in the cross-session provider persist across the owner's sessions and can steer tool use. — [python/packages/core/agent_framework/_harness/_memory.py:1268](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_harness/_memory.py#L1268); searched `rg -n 'AGENTS\.md|CLAUDE\.md|find_dotenv'` in `python/packages/core/agent_framework` → 0 hits (No auto-loaded workspace instruction files or implicit .env discovery in core.) (verified)
  - *To reach the next level:* Memory is not ephemeral or human-reviewed before persisting.
- **Cap:** none

### C7 Third-party extensions — 0.28 (medium)

Third-party code enters mainly through MCP servers and skills that the developer configures in code; nothing is enabled by default and the workspace cannot add servers. There is no version pinning or integrity check, and when a server announces a changed tool list the client reloads it silently, so a server update can change tool definitions without re-approval. Stdio servers run as separate processes; when no env is given the MCP SDK's default restricted environment applies.

- **S L1:** MCP servers are developer-chosen commands/URLs with no pinning or hash verification. — searched `rg -n -i 'sha256|checksum|signature_verif|pin_version'` in `python/packages/core/agent_framework/_mcp.py` → 0 hits (MCP client performs no integrity check of server code or tool definitions.) (verified)
  - *To reach the next level:* No version pinning or integrity check.
- **C L0:** No extension type is verified. — searched `rg -n -i 'sha256|checksum|signature_verif|pin_version'` in `python/packages/core/agent_framework/_mcp.py` → 0 hits (MCP client performs no integrity check of server code or tool definitions.) (verified)
  - *To reach the next level:* No verification on any extension type.
- **D L2:** No third-party extension is enabled by default; developers add MCP servers explicitly in code, but tool-list changes reload silently. — [python/packages/core/agent_framework/_mcp.py:2354](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_mcp.py#L2354) (verified)
  - *To reach the next level:* No re-approval when a server's tool definitions change.
- **B L2:** Stdio servers run as separate processes; env is passed through only if set, otherwise the mcp SDK default (a small inherited set) applies. — [python/packages/core/agent_framework/_mcp.py:3713](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_mcp.py#L3713) (inferred)
  - *To reach the next level:* Not sandboxed per extension with scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.40 (high)

API keys are wrapped in a SecretString with a masked repr, tool exceptions are redacted before being returned to the model by default, and OpenTelemetry sensitive data (prompts, arguments, results) is off by default. Gaps: a version/feature header is sent on by default, tool exceptions are logged in full, and the bundled shell tool hands the whole process environment, including secrets, to model-chosen commands.

- **S L2:** Type-level masking (SecretString) and redacted error messages for model-bound tool failures; sensitive telemetry gated behind a flag. — [python/packages/core/agent_framework/_settings.py:52](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_settings.py#L52); [python/packages/core/agent_framework/_tools.py:861](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_tools.py#L861); [python/packages/core/agent_framework/observability.py:1108](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/observability.py#L1108) (verified)
  - *To reach the next level:* No secret manager integration or redaction of secrets in tool results sent to the model.
- **C L2:** Telemetry and model-bound error paths are protected; subprocess environments and error logs are not. — [python/packages/core/agent_framework/_tools.py:1166](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_tools.py#L1166); [python/packages/tools/agent_framework_tools/shell/_tool.py:187](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_tool.py#L187) (verified)
  - *To reach the next level:* Subprocess env and logged exceptions are unprotected.
- **D L1:** User-agent/feature-usage telemetry is on by default (content-free); sensitive OTel off by default. — [python/packages/core/agent_framework/_telemetry.py:20](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_telemetry.py#L20); [python/packages/core/agent_framework/observability.py:1108](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/observability.py#L1108) (verified)
  - *To reach the next level:* Telemetry is on by default rather than opt-in.
- **B L1:** Long-lived developer API keys and env credentials reachable from the shell tool's subprocess environment. — [python/packages/tools/agent_framework_tools/shell/_tool.py:187](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_tool.py#L187) (verified)
  - *To reach the next level:* No short-lived or scoped credentials.
- **Cap:** none

### C9 Audit & traceability — 0.35 (high)

Each tool invocation runs inside an OpenTelemetry execute_tool span with tool name, call id, timing and error status, and arguments/results are added when sensitive-data capture is enabled. Instrumentation code is on by default, but nothing is recorded anywhere unless the developer configures an exporter, and there is no tamper-evident storage or approver attribution.

- **S L2:** Structured spans per tool call with status and timing; arguments only when sensitive data is enabled. — [python/packages/core/agent_framework/_tools.py:1152](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_tools.py#L1152); [python/packages/core/agent_framework/observability.py:1212](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/observability.py#L1212) (verified)
  - *To reach the next level:* No actor attribution separating requesting principal and approver.
- **C L2:** FunctionTool invocations (including MCP-wrapped tools) are traced; approvals and denials are not recorded as audit events. — [python/packages/core/agent_framework/_tools.py:1152](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_tools.py#L1152) (verified)
  - *To reach the next level:* Approvals and denials are not traced.
- **D L0:** Spans are created by default but no exporter is configured, so nothing is persisted unless the developer sets one up. — [python/packages/core/agent_framework/observability.py:1107](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/observability.py#L1107) (verified)
  - *To reach the next level:* Recording is effectively opt-in.
- **B L1:** OTel batch export is best-effort with no fail-closed behaviour. — searched `rg -n 'hash_chain|hmac|append_only'` in `python/packages/core/agent_framework/observability.py` → 0 hits (No tamper-evident audit storage.) (verified)
  - *To reach the next level:* Records are not durable per action.
- **Cap:** G1 — Without a developer-configured exporter no record is retained.

### C10 Limits & kill switch — 0.38 (high)

The tool loop is capped at 40 model round-trips by default, and the bundled shell tools have a 30-second per-command timeout that kills the whole process group. A total function-call cap and a wall-clock budget exist but default to unlimited, and there is no token or cost cap. Sub-agents and background agents run with their own fresh budgets as asyncio tasks.

- **S L1:** Iteration cap enforced in code; per-execution timeouts exist only on the bundled shell tools, and the wall-clock budget is opt-in. — [python/packages/core/agent_framework/_tools.py:104](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_tools.py#L104); [python/packages/core/agent_framework/_tools.py:5121](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_tools.py#L5121); [python/packages/tools/agent_framework_tools/shell/_killtree.py:126](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_killtree.py#L126); [python/packages/core/agent_framework/_tools.py:1855](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_tools.py#L1855) (verified)
  - *To reach the next level:* No wall-clock, generic per-tool timeout, or token/cost cap enforced for registered tools by default.
- **C L2:** Top-level loop plus shell-tool timeouts; sub-agents and background tasks get their own budgets. — [python/packages/core/agent_framework/_harness/_background_agents.py:492](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_harness/_background_agents.py#L492); [python/packages/tools/agent_framework_tools/shell/_tool.py:158](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/tools/agent_framework_tools/shell/_tool.py#L158) (verified)
  - *To reach the next level:* Sub-agents and background tasks don't count against the parent budget.
- **D L2:** Sensible iteration default (40) the model cannot change; operator-configurable. — [python/packages/core/agent_framework/_tools.py:104](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_tools.py#L104); [python/packages/core/agent_framework/_tools.py:1854](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_tools.py#L1854) (verified)
  - *To reach the next level:* No hard ceilings; function-call and duration limits default to None.
- **B L1:** No wall-clock or spend ceiling by default; background agent tasks can keep running. — [python/packages/core/agent_framework/_tools.py:1855](https://github.com/microsoft/agent-framework/blob/e9857bd75a21843803431a2e7dfa41ee9b017ac7/python/packages/core/agent_framework/_tools.py#L1855); searched `rg -n 'max_cost|cost_limit|token_budget'` in `python/packages/core/agent_framework/_tools.py python/packages/core/agent_framework/_agents.py` → 0 hits (No token or cost cap in the function-invocation loop or Agent.) (verified)
  - *To reach the next level:* No tight default time/cost ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Tool and MCP results enter context unlabeled by default (python/packages/core/agent_framework/_mcp.py:894) · [B] sensitive data/systems: Developer-supplied credentials and inherited env (python/packages/tools/agent_framework_tools/shell/_tool.py:187) · [C] state change / egress: Any registered tool runs without approval by default (python/packages/core/agent_framework/_tools.py:667) · Same default session? Yes

## Highest-impact improvements
1. Default FunctionTool and MCPTool approval_mode to always_require for tools not explicitly marked read-only. — C2 D L0→L3, +0.150 before caps (Playbook 5)
2. Promote FIDES label tracking and policy enforcement out of experimental and enable it by default for tool and MCP results. — C5 D L0→L3, +0.150 before caps (Playbook 1)
3. Make LocalShellTool default to clean_env=True so subprocesses don't inherit secrets. — C1 C L0→L1, +0.075 before caps (Playbook 4)
4. Set a default max_duration_seconds and max_function_calls, and count sub-agent work against the parent budget. — C10 B L1→L2, +0.050 before caps (Playbook 3 step 3)
5. Require re-approval when an MCP server's tool list changes instead of silently reloading. — C7 D L2→L3, +0.050 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the Python packages (core, tools, hyperlight); the .NET implementation, hosting packages (A2A, MCP hosting, Telegram, Foundry), DevUI and declarative agents were not reviewed in depth.
- C7 B is inferred from the mcp Python SDK's documented default environment when env is None.
- No text aimed at AI reviewers was found in the files examined (contributor AGENTS.md files were treated as data).
