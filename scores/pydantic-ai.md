# Defense-in-Depth Score: Pydantic AI

**Repo:** https://github.com/pydantic/pydantic-ai · **Commit:** `66951321b89587235f432281eba909a30a585ffd` (v2.54.0) · **Reviewed:** 2026-10-03
**What it is:** Type-safe Python agent framework
**Category:** Agent Frameworks
**Scored configuration:** `pip install pydantic-ai` and `Agent(model)` with default constructor and run arguments (no tools, no capabilities, no instrumentation), as in the README quickstart.
**Agent surface (default):** code execution opt-in · filesystem write opt-in · network egress opt-in · external credentials yes · persistent memory opt-in · untrusted input opt-in · third party extensions opt-in · sub agents opt-in · external communication opt-in

## Score: 3.5 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C2 | Approval gates | L3 | L1 | L0 | L0 | 0.30 | G1 | **0.30** | High |
| C3 | Tool & action scoping | L2 | L2 | L3 | L1 | 0.50 | — | **0.50** | High |
| C4 | Code-execution isolation | L4 | L1 | L0 | L2 | 0.47 | G1 | **0.47** (alt) | High |
| C5 | Untrusted input blast radius | L1 | L1 | L2 | L0 | 0.25 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C7 | Third-party extensions | L1 | L0 | L2 | L1 | 0.23 | — | **0.23** | Medium |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C9 | Audit & traceability | L2 | L3 | L0 | L1 | 0.42 | G1 | **0.42** (alt) | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |


Pydantic AI ships a minimal default (no tools) and some well-engineered primitives, such as a per-call human-approval gate, an SSRF-hardened web fetch tool and a 50-request run cap. But approval, isolation and telemetry are all opt-in. Registered tools run unattended in your process with all of its credentials, so if the agent reads attacker-controlled content it can leak data and act with no human involved. To change that, you have to mark consequential tools requires_approval and run code in a sandbox yourself.

## Critical gaps
- A hijacked agent can exfiltrate data and run any registered state-changing tool unattended; tool approval is off by default and no egress control is tied to untrusted input. (ASI01, LLM01; C5) — [pydantic_ai_slim/pydantic_ai/tools.py:375](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/tools.py#L375); [pydantic_ai_slim/pydantic_ai/common_tools/web_fetch.py:557-560](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/common_tools/web_fetch.py#L557-L560)
- The core library's only local execution backend runs commands as host subprocesses that, per its own docs, isolate nothing; the user's whole home directory and network are reachable. (ASI05; C4) — [pydantic_ai_slim/pydantic_ai/workspaces/local.py:103](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/workspaces/local.py#L103); [pydantic_ai_slim/pydantic_ai/capabilities/local_workspace.py:17](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/capabilities/local_workspace.py#L17)

## Criterion details

### C1 Identity & least privilege — 0.25 (high)

Pydantic AI has no identity or authorization layer of its own: registered tools run inside the developer's Python process with whatever credentials that process holds, and any per-user authorization is left to developer code. It does narrow ambient authority in two places that ship on by default: the core local workspace gives subprocesses only PATH, HOME and locale variables, and the UI adapters drop client-submitted s3:// or gs:// file URLs and uploaded-file references that the model provider would otherwise fetch with the server's IAM role. In-process function tools still hold the full process authority, so a hijacked agent can use whatever the application holds.

- **S L1:** No scoped identity primitive; the only narrowing is subprocess env scrubbing and refusal to forward client-supplied provider-IAM file references. — [pydantic_ai_slim/pydantic_ai/workspaces/local.py:45](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/workspaces/local.py#L45); [pydantic_ai_slim/pydantic_ai/messages.py:3696-3701](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/messages.py#L3696-L3701); searched `rg -n -i "downscop|least.privilege|token_exchange|assume_role"` in `pydantic_ai_slim/pydantic_ai` → 0 hits (verified)
  - *To reach the next level:* No per-tool or per-request credential scoping or authorization check in the tool executor.
- **C L1:** Workspace subprocesses and UI-adapter input are narrowed; in-process function tools and MCP toolsets run with the full process authority. — [pydantic_ai_slim/pydantic_ai/workspaces/local.py:45](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/workspaces/local.py#L45); [pydantic_ai_slim/pydantic_ai/ui/_adapter.py:622](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/ui/_adapter.py#L622); [pydantic_ai_slim/pydantic_ai/agent/__init__.py:630](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/agent/__init__.py#L630) (verified)
  - *To reach the next level:* No shared authorization layer that every tool path, including MCP and in-process tools, passes through.
- **D L1:** The env scrub and IAM-URL stripping are on by default, but the agent otherwise runs as the operator with every credential in the process. — [pydantic_ai_slim/pydantic_ai/workspaces/local.py:45](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/workspaces/local.py#L45); [pydantic_ai_slim/pydantic_ai/messages.py:3696-3701](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/messages.py#L3696-L3701) (verified)
  - *To reach the next level:* No minimal-privilege default identity; least privilege requires developer-written scoping.
- **B L1:** The framework neither adds nor removes credentials beyond model-provider keys; a hijacked agent reaches whatever the developer's tools and process can reach. — [pydantic_ai_slim/pydantic_ai/providers/openai.py:121](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/providers/openai.py#L121); [pydantic_ai_slim/pydantic_ai/capabilities/local_workspace.py:17](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/capabilities/local_workspace.py#L17) (verified)
  - *To reach the next level:* Nothing in the framework bounds what a compromised tool path can do with the host's credentials.
- **Cap:** none

### C2 Approval gates — 0.30 (high)

Pydantic AI has a well-built human-approval primitive: a tool marked requires_approval, or any toolset wrapped with approval_required(), pauses the run and hands the exact tool call (name, arguments, id) back to the application, which answers approve, approve-with-edited-arguments (re-validated), or deny. The gate is enforced on both execution paths in code. But it is off by default: every tool runs unattended unless the developer flags it, and provider-executed native tools such as code execution or web fetch never pass through it. There is no undo, checkpoint, or dry-run primitive.

- **S L3:** Per-call approval returns the exact ToolCallPart, supports argument-conditional policies via approval_required_func, re-validates edited args, and makes denial first-class. — [pydantic_ai_slim/pydantic_ai/_deferred.py:40](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/_deferred.py#L40); [pydantic_ai_slim/pydantic_ai/toolsets/approval_required.py:29](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/toolsets/approval_required.py#L29); [pydantic_ai_slim/pydantic_ai/_tool_execution.py:684](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/_tool_execution.py#L684); [pydantic_ai_slim/pydantic_ai/_deferred.py:114](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/_deferred.py#L114); [pydantic_ai_slim/pydantic_ai/tool_manager.py:1163-1173](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/tool_manager.py#L1163-L1173) (verified)
  - *To reach the next level:* Approved calls are bound by tool_call_id with arguments re-read from caller-supplied history, and no built-in argument allow/deny policy exists.
- **C L1:** Only tools or toolsets the developer flags are gated; built-in common tools are unflagged and provider-executed native tools are never dispatched through the gate. — [pydantic_ai_slim/pydantic_ai/tools.py:549](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/tools.py#L549); [pydantic_ai_slim/pydantic_ai/messages.py:3597](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/messages.py#L3597); [pydantic_ai_slim/pydantic_ai/common_tools/web_fetch.py:557-560](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/common_tools/web_fetch.py#L557-L560) (verified)
  - *To reach the next level:* No mode that gates every tool path (including native tools) or rejects unflagged consequential tools.
- **D L0:** requires_approval defaults to False, so approval is opt-in. — [pydantic_ai_slim/pydantic_ai/tools.py:375](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/tools.py#L375) (verified)
  - *To reach the next level:* Approval is not on by default for any tool.
- **B L0:** No checkpoint, undo, or dry-run primitive bounds a wrongly executed action. — searched `rg -n -i "dry_run|rollback|undo\("` in `pydantic_ai_slim/pydantic_ai` → 1 hits (single hit is a realtime-session send rollback, not an action checkpoint/undo primitive) (verified)
  - *To reach the next level:* No rollback or preview for consequential actions.
- **Cap:** G1 — The approval gate exists but requires_approval defaults to False, so nothing is gated in the default configuration.

### C3 Tool & action scoping — 0.50 (high)

Every function tool's arguments are validated against a Pydantic schema generated from its type hints, and developers can add an args_validator, but that is type checking rather than an allowlist. The one bundled network tool, web_fetch, is genuinely hardened: it blocks private and cloud-metadata addresses, pins the resolved IP, and re-validates every redirect. MCP tool arguments are only checked to be a JSON object before being forwarded. The agent starts with no tools at all, so the default posture is minimal, but anything registered runs with its full reach.

- **S L2:** Typed Pydantic schema validation on function tools; the only allowlist-grade validation is web_fetch's SSRF guard. — [pydantic_ai_slim/pydantic_ai/_ssrf.py:471](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/_ssrf.py#L471); [pydantic_ai_slim/pydantic_ai/_ssrf.py:179](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/_ssrf.py#L179); [pydantic_ai_slim/pydantic_ai/common_tools/web_fetch.py:557-560](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/common_tools/web_fetch.py#L557-L560) (verified)
  - *To reach the next level:* No allowlist validation layer (paths, hosts, quantities) for registered tools in general.
- **C L2:** Function tools are schema-validated; MCP tool args are validated only as an arbitrary dict. — [pydantic_ai_slim/pydantic_ai/mcp.py:633-635](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/mcp.py#L633-L635); [pydantic_ai_slim/pydantic_ai/mcp.py:1346](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/mcp.py#L1346) (verified)
  - *To reach the next level:* MCP/extension tool arguments are not validated against the server's declared input schema.
- **D L3:** Agent(tools=()) by default: no tools are registered unless the developer adds them. — [pydantic_ai_slim/pydantic_ai/agent/__init__.py:630](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/agent/__init__.py#L630) (verified)
  - *To reach the next level:* No per-task tool allowlist on by default; tool filtering via prepare/FilteredToolset is opt-in.
- **B L1:** Registered tools run in the host process with no framework-imposed scope or quantity bound; only web_fetch blocks internal addresses. — [pydantic_ai_slim/pydantic_ai/capabilities/local_workspace.py:17](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/capabilities/local_workspace.py#L17); [pydantic_ai_slim/pydantic_ai/common_tools/web_fetch.py:557-560](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/common_tools/web_fetch.py#L557-L560) (verified)
  - *To reach the next level:* No framework-level workspace scoping or quantity bounds on tool effects.
- **Cap:** none

### C4 Code-execution isolation — 0.47 (high)

The core library runs nothing model-generated by default, but its only local execution backend, LocalWorkspace, runs commands as plain host subprocesses and is documented by the maintainers as isolating nothing; function tools themselves run in the application's process. When code execution is enabled this way, an injected command reaches the user's whole home directory and network. The separate pydantic-ai-harness package in the same repo offers real sandboxes (E2B remote VMs, bubblewrap); E2B is the strongest, but it is opt-in, has internet access on by default, and only covers workspace commands, not function tools or MCP servers.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** LocalWorkspaceBackend runs host subprocesses and 'isolates nothing'. — [pydantic_ai_slim/pydantic_ai/workspaces/local.py:103](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/workspaces/local.py#L103); [pydantic_ai_slim/pydantic_ai/capabilities/local_workspace.py:17](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/capabilities/local_workspace.py#L17) (verified)
    - *To reach the next level:* No isolation primitive in the core library.
  - **C L0:** No path is sandboxed in the default/core configuration; function tools run in-process. — [pydantic_ai_slim/pydantic_ai/workspaces/local.py:103](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/workspaces/local.py#L103) (verified)
    - *To reach the next level:* No sandbox covers the main execution path.
  - **D L0:** No isolation is on by default. — [pydantic_ai_slim/pydantic_ai/capabilities/local_workspace.py:17](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/capabilities/local_workspace.py#L17) (verified)
    - *To reach the next level:* Isolation is not default for any execution path.
  - **B L0:** Host subprocess with the user's full filesystem and network; env is scrubbed but ~/.ssh, ~/.aws etc. are reachable. — [pydantic_ai_slim/pydantic_ai/workspaces/local.py:103](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/workspaces/local.py#L103); [pydantic_ai_slim/pydantic_ai/workspaces/local.py:45](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/workspaces/local.py#L45) (verified)
    - *To reach the next level:* Workspace is not confined; host home directory and network are reachable.
- **opt-in E2B remote sandbox (pydantic-ai-harness E2BSandbox)** (alt; raw 0.47, cap G1 → 0.47) ← counted
  - **S L4:** Commands run in a remote E2B sandbox created per workspace, with a bounded lifetime. — [src/pydantic_ai_harness/pydantic_ai_harness/e2b_sandbox/_backend.py:476](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/src/pydantic_ai_harness/pydantic_ai_harness/e2b_sandbox/_backend.py#L476); [src/pydantic_ai_harness/pydantic_ai_harness/e2b_sandbox/_backend.py:4](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/src/pydantic_ai_harness/pydantic_ai_harness/e2b_sandbox/_backend.py#L4) (verified)
  - **C L1:** Covers workspace commands and file ops; function tools and MCP stdio servers still run on the host. — [src/pydantic_ai_harness/pydantic_ai_harness/e2b_sandbox/_backend.py:476](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/src/pydantic_ai_harness/pydantic_ai_harness/e2b_sandbox/_backend.py#L476); [pydantic_ai_slim/pydantic_ai/mcp.py:2001-2004](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/mcp.py#L2001-L2004) (verified)
    - *To reach the next level:* Function tools and MCP stdio servers are not routed through the sandbox.
  - **D L0:** Opt-in, in a separate package not installed by `pip install pydantic-ai`. — [src/pydantic_ai_harness/pydantic_ai_harness/e2b_sandbox/_backend.py:476](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/src/pydantic_ai_harness/pydantic_ai_harness/e2b_sandbox/_backend.py#L476) (verified)
    - *To reach the next level:* Not on by default.
  - **B L2:** No host files, but allow_internet_access defaults to True. — [src/pydantic_ai_harness/pydantic_ai_harness/e2b_sandbox/_backend.py:237](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/src/pydantic_ai_harness/pydantic_ai_harness/e2b_sandbox/_backend.py#L237) (verified)
    - *To reach the next level:* Network egress is unrestricted by default inside the sandbox.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.25 (high)

Nothing in the framework limits what a hijacked agent can do once it reads attacker-controlled content: tool results, MCP results and fetched pages enter the conversation as ordinary tool returns, and no step disables egress or forces approval after untrusted content arrives. The only measures are labelling ones: tool-produced files are wrapped in provenance tags (which the code itself says 'identify rather than prove'), and the UI adapters strip client-injected system prompts. Because its docs routinely combine web fetching, private application data and outbound tools, a prompt injection can leak data and trigger any registered tool with no human involved.

- **S L1:** Delimiter-only: tool-produced files are framed with provenance tags; no taint tracking or Rule-of-Two enforcement. — [pydantic_ai_slim/pydantic_ai/messages.py:1403](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/messages.py#L1403); [pydantic_ai_slim/pydantic_ai/messages.py:1417](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/messages.py#L1417) (verified)
  - *To reach the next level:* No code-level restriction of egress or state-changing tools once untrusted content is in context.
- **C L1:** Only multimodal tool output on text-only providers is framed; text tool results and MCP outputs are not distinguished. — [pydantic_ai_slim/pydantic_ai/messages.py:1403](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/messages.py#L1403) (verified)
  - *To reach the next level:* Tool results, MCP outputs and tool descriptions are not treated as a distinct untrusted source.
- **D L2:** The provenance framing and UI-adapter sanitization are automatic. — [pydantic_ai_slim/pydantic_ai/messages.py:1403](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/messages.py#L1403); [pydantic_ai_slim/pydantic_ai/ui/_adapter.py:622](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/ui/_adapter.py#L622) (verified)
  - *To reach the next level:* Nothing structural is on by default for the mechanism to enforce.
- **B L0:** Worst case: injected content can drive exfiltration (e.g. arbitrary-URL web_fetch) and any registered state-changing tool unattended; the framework prevents neither. — [pydantic_ai_slim/pydantic_ai/common_tools/web_fetch.py:557-560](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/common_tools/web_fetch.py#L557-L560); [pydantic_ai_slim/pydantic_ai/tools.py:375](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/tools.py#L375) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions are not gated by default.
- **Cap:** C5-WORSTCASE — B is L0 in the default configuration: leak plus irreversible action can happen unattended.

### C6 Memory, context & configuration integrity — 0.30 (high)

The core library has no memory store and auto-loads no workspace instruction or config files; persistence happens only when the developer saves message history and passes it back, or implements the tool behind Anthropic's native memory tool. History passed back is replayed as trusted context, so injected text saved in a prior run can steer tool calls in later ones. Client-submitted history through the UI adapters is sanitized by default (system prompts, workspace references, unresolved tool calls and provider-IAM file references are stripped).

- **S L1:** No memory gating; developer-supplied history is replayed as-is, while client history in UI adapters is sanitized. — [pydantic_ai_slim/pydantic_ai/messages.py:3551-3562](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/messages.py#L3551-L3562); [pydantic_ai_slim/pydantic_ai/models/anthropic.py:1998](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/models/anthropic.py#L1998) (verified)
  - *To reach the next level:* Persisted history and memory writes carry no provenance or validation in the default path.
- **C L1:** Only the UI-adapter (client-supplied) history path is controlled. — [pydantic_ai_slim/pydantic_ai/ui/_adapter.py:622](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/ui/_adapter.py#L622) (verified)
  - *To reach the next level:* Developer-loaded message history and memory-tool storage are not controlled.
- **D L2:** No shared store exists; each run sees only the history the caller passes. — [pydantic_ai_slim/pydantic_ai/messages.py:3551-3562](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/messages.py#L3551-L3562); searched `rg -n "load_dotenv|trust_remote_code|pickle\.load|torch\.load"` in `pydantic_ai_slim/pydantic_ai` → 0 hits (verified)
  - *To reach the next level:* No model-proof isolation or retention guarantees beyond the per-call history argument.
- **B L1:** If the developer persists history, poisoned content persists across that user's sessions and can trigger tool use. — [pydantic_ai_slim/pydantic_ai/models/anthropic.py:1998](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/models/anthropic.py#L1998) (verified)
  - *To reach the next level:* Poisoned history is neither session-scoped nor easily purged by framework tooling.
- **Cap:** none

### C7 Third-party extensions — 0.23 (medium)

Third-party code comes in mainly through MCP servers, which the developer names explicitly (a URL, a command, or a JSON config passed by path); nothing is enabled by default and no workspace file adds servers. There is no version pinning, hash, or signature check, and no detection if a server's tools change between sessions. Stdio servers run as separate processes under the same user; their environment handling is delegated to the fastmcp and MCP SDK libraries.

- **S L1:** User-chosen MCP sources with no pinning or integrity verification. — [pydantic_ai_slim/pydantic_ai/mcp.py:2001-2004](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/mcp.py#L2001-L2004); searched `rg -n "sha256|verify_signature|integrity"` in `pydantic_ai_slim/pydantic_ai/mcp.py` → 0 hits (verified)
  - *To reach the next level:* No pinning, hash, or signature verification of extensions.
- **C L0:** No extension type is verified. — searched `rg -n "sha256|verify_signature|integrity"` in `pydantic_ai_slim/pydantic_ai/mcp.py` → 0 hits (verified)
  - *To reach the next level:* No verification for any extension type.
- **D L2:** Nothing third-party is enabled by default; MCP servers must be named in code or an explicitly passed config path. — [pydantic_ai_slim/pydantic_ai/agent/__init__.py:630](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/agent/__init__.py#L630); [pydantic_ai_slim/pydantic_ai/mcp.py:2001-2004](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/mcp.py#L2001-L2004) (verified)
  - *To reach the next level:* No display of exact package/permissions before first launch.
- **B L1:** Stdio MCP servers run as separate same-user processes; env passed via fastmcp StdioTransport (scrubbing depends on the MCP SDK default, not verified here). — [pydantic_ai_slim/pydantic_ai/mcp.py:2001-2004](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/mcp.py#L2001-L2004) (inferred)
  - *To reach the next level:* No per-extension sandbox or scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.30 (high)

Model-provider keys are read from environment variables and provider reprs omit them, and workspace subprocesses receive only PATH, HOME and locale variables. There is no type-level secret masking, no redaction of secrets in tool results sent to the model, and no secret scanning. Telemetry is off by default, but when OpenTelemetry/Logfire instrumentation is enabled it records prompts, tool arguments and tool results by default. MCP JSON configs can expand any environment variable into server arguments or headers.

- **S L1:** Keys from env vars; masking limited to provider reprs and subprocess env scrubbing. — [pydantic_ai_slim/pydantic_ai/providers/openai.py:121](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/providers/openai.py#L121); [pydantic_ai_slim/pydantic_ai/providers/__init__.py:139](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/providers/__init__.py#L139); [pydantic_ai_slim/pydantic_ai/workspaces/local.py:45](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/workspaces/local.py#L45) (verified)
  - *To reach the next level:* No type-level masking or log/model-bound redaction.
- **C L1:** Only the subprocess-environment path is protected. — [pydantic_ai_slim/pydantic_ai/workspaces/local.py:45](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/workspaces/local.py#L45); [pydantic_ai_slim/pydantic_ai/models/instrumented.py:78](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/models/instrumented.py#L78) (verified)
  - *To reach the next level:* Logs/telemetry and model-bound messages are not redacted.
- **D L2:** Telemetry is opt-in, but once enabled include_content=True ships prompts and tool I/O. — [pydantic_ai_slim/pydantic_ai/agent/__init__.py:530](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/agent/__init__.py#L530); [pydantic_ai_slim/pydantic_ai/models/instrumented.py:78](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/models/instrumented.py#L78) (verified)
  - *To reach the next level:* Content-free telemetry is not the default when instrumentation is enabled.
- **B L1:** Long-lived provider API keys sit in the process environment, readable by any in-process tool. — [pydantic_ai_slim/pydantic_ai/providers/openai.py:121](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/providers/openai.py#L121); [pydantic_ai_slim/pydantic_ai/mcp.py:1916](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/mcp.py#L1916) (verified)
  - *To reach the next level:* Keys are not short-lived or scoped by the framework.
- **Cap:** none

### C9 Audit & traceability — 0.42 (high)

By default, the only record is the in-memory message history the run returns, which captures every tool call with arguments, IDs and timestamps but is lost unless the developer saves it. Opt-in OpenTelemetry/Logfire instrumentation emits standard gen_ai spans for each run and tool call (including MCP tools and deferrals) and can ship them off-host, but it does not record who requested or approved an action.

- **default configuration** (default; raw 0.28 → 0.28)
  - **S L1:** Structured tool-call parts with timestamps exist only in memory; no persisted transcript. — [pydantic_ai_slim/pydantic_ai/messages.py:2536](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/messages.py#L2536); [pydantic_ai_slim/pydantic_ai/messages.py:2833](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/messages.py#L2833) (verified)
    - *To reach the next level:* No durable structured record by default.
  - **C L2:** All tool calls, including MCP tools, appear in the run's message history. — [pydantic_ai_slim/pydantic_ai/messages.py:2536](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/messages.py#L2536) (verified)
    - *To reach the next level:* Sub-agent runs keep separate histories, and approvals are not recorded as distinct audit events.
  - **D L1:** Always produced, but held in the application's own process where tools and code can alter it. — [pydantic_ai_slim/pydantic_ai/messages.py:2833](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/messages.py#L2833) (verified)
    - *To reach the next level:* Not written by a component outside the agent process.
  - **B L0:** In-memory only; lost on crash or exit. — [pydantic_ai_slim/pydantic_ai/agent/__init__.py:530](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/agent/__init__.py#L530) (verified)
    - *To reach the next level:* Records are not flushed durably per action.
- **opt-in OpenTelemetry / Logfire instrumentation** (alt; raw 0.42, cap G1 → 0.42) ← counted
  - **S L2:** Standard gen_ai spans with tool name, call id and arguments per call, exportable off-host. — [pydantic_ai_slim/pydantic_ai/capabilities/instrumentation.py:472-474](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/capabilities/instrumentation.py#L472-L474) (verified)
    - *To reach the next level:* No principal or approver attribution on records.
  - **C L3:** The instrumentation capability wraps every tool execution, including MCP tools, and records deferrals. — [pydantic_ai_slim/pydantic_ai/capabilities/instrumentation.py:472-474](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/capabilities/instrumentation.py#L472-L474) (verified)
    - *To reach the next level:* Configuration changes and credential use are not recorded.
  - **D L0:** Instrumentation is off unless enabled. — [pydantic_ai_slim/pydantic_ai/agent/__init__.py:530](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/agent/__init__.py#L530) (verified)
    - *To reach the next level:* Not on by default.
  - **B L1:** Best-effort span export via the OTel SDK. — [pydantic_ai_slim/pydantic_ai/capabilities/instrumentation.py:472-474](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/capabilities/instrumentation.py#L472-L474) (verified)
    - *To reach the next level:* Records are not guaranteed durable per action.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C10 Limits & kill switch — 0.45 (high)

Each run is capped at 50 model requests by default, and developers can add token, cost and tool-call limits plus per-tool timeouts, all enforced in code. There is no default wall-clock, token or cost limit, and tools have no timeout unless set. Cancelling a run cancels the driving task and in-flight async tool tasks, but synchronous tools run in threads that are not abandoned, and LocalWorkspace background jobs outlive the run. A sub-agent shares the parent's budget only if the developer passes usage through.

- **S L2:** Request cap plus opt-in token/cost/tool-call caps and per-tool timeouts, enforced in code; cooperative-plus-task cancellation. — [pydantic_ai_slim/pydantic_ai/usage.py:482](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/usage.py#L482); [pydantic_ai_slim/pydantic_ai/usage.py:480](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/usage.py#L480); [pydantic_ai_slim/pydantic_ai/_cancel.py:6](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/_cancel.py#L6); searched `rg -n -i "wall_clock|run_timeout|max_duration"` in `pydantic_ai_slim/pydantic_ai/usage.py` → 0 hits (verified)
  - *To reach the next level:* No run wall-clock limit and no rate limit on side-effecting tools.
- **C L2:** Limits cover the top-level loop; tool timeouts apply when set. — [pydantic_ai_slim/pydantic_ai/agent/__init__.py:1275](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/agent/__init__.py#L1275); [pydantic_ai_slim/pydantic_ai/agent/__init__.py:635](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/agent/__init__.py#L635) (verified)
  - *To reach the next level:* Sub-agents only count against the parent budget when usage is passed explicitly.
- **D L2:** Sensible default request_limit=50; other limits operator-configurable. — [pydantic_ai_slim/pydantic_ai/agent/__init__.py:1707](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/agent/__init__.py#L1707); [pydantic_ai_slim/pydantic_ai/usage.py:482](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/usage.py#L482) (verified)
  - *To reach the next level:* Delegation to another agent starts a fresh budget by default.
- **B L1:** No default time, token or cost ceiling; sync tools in threads and background jobs keep running after cancel. — [pydantic_ai_slim/pydantic_ai/usage.py:490](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/usage.py#L490); [pydantic_ai_slim/pydantic_ai/agent/__init__.py:635](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/agent/__init__.py#L635); [pydantic_ai_slim/pydantic_ai/_utils.py:203](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/_utils.py#L203); [pydantic_ai_slim/pydantic_ai/_utils.py:82](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/_utils.py#L82); [pydantic_ai_slim/pydantic_ai/workspaces/local.py:106](https://github.com/pydantic/pydantic-ai/blob/66951321b89587235f432281eba909a30a585ffd/pydantic_ai_slim/pydantic_ai/workspaces/local.py#L106) (verified)
  - *To reach the next level:* No default wall-clock/spend ceiling, and stop does not kill in-flight sync work.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: web_fetch tool returns arbitrary page content (pydantic_ai_slim/pydantic_ai/common_tools/web_fetch.py:557) · [B] sensitive data/systems: tools and provider keys run in the application process (pydantic_ai_slim/pydantic_ai/providers/openai.py:121) · [C] state change / egress: arbitrary-URL fetch and any registered tool run ungated by default (pydantic_ai_slim/pydantic_ai/tools.py:375) · Same default session? Yes

## Highest-impact improvements
1. Add a run wall-clock limit to UsageLimits and a default tool timeout. — C10 S L2→L3, +0.075 before caps (Playbook 3 step 3)
2. Validate MCP tool arguments against the server-declared inputSchema before forwarding. — C3 C L2→L3, +0.075 before caps (Playbook 3)
3. Offer a taint mode: once a tool result from an untrusted source enters context, force approval on egress and state-changing tools. — C5 S L1→L2, +0.075 before caps (Playbook 1)
4. Pin MCP stdio package versions and warn on unpinned npx/uvx launches. — C7 S L1→L2, +0.075 before caps (Playbook 3)
5. Default include_content=False when instrumentation is enabled, and add secret-pattern redaction. — C8 D L2→L3, +0.050 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of commit 66951321b89587235f432281eba909a30a585ffd only; nothing was executed, installed, or probed.
- Framework scored by its defaults (`pydantic-ai` core); absent primitives score L0 even where a developer could add them.
- The separate pydantic-ai-harness package (shell, filesystem, sandbox, memory, guardrail capabilities) and the pydantic_clai2 / clai CLIs in the same repo were not scored as the primary configuration; only the harness E2B sandbox was scored, as the C4 opt-in alternative.
- fastmcp and MCP SDK behaviour (stdio environment handling) was inferred, not read; provider-side native tools (code execution, web fetch) are run by the model vendor and were not examined.
- Model behaviour (refusals, deception) is out of scope; only code-level controls are scored.
- No reviewer-directed prompt injection found; repo CLAUDE.md/AGENTS.md address contributors only.
