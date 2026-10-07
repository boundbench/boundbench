# Defense-in-Depth Score: mcp-agent

**Repo:** https://github.com/lastmile-ai/mcp-agent · **Commit:** `f62d849350816588b1c6294e7914bbe4d8b84072` (0.2.6) · **Reviewed:** 2026-10-04
**What it is:** Python framework for building agents and workflows on the Model Context Protocol, with a CLI scaffolder and Temporal-backed durable execution.
**Category:** Agent Frameworks
**Scored configuration:** Library defaults plus the `mcp-agent init` scaffolded mcp_agent.config.yaml (asyncio engine, filesystem server rooted at '.', fetch server, console+file logging).
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions yes · sub agents yes · external communication opt-in

## Score: 1.7 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L3 | L1 | L0 | L2 | 0.40 | G1 | **0.40** (alt) | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C3 | Tool & action scoping | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | C6-REPOCONFIG | **0.10** | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | High |
| C9 | Audit & traceability | L2 | L2 | L0 | L1 | 0.35 | G1 | **0.35** (alt) | Medium |
| C10 | Limits & kill switch | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |


mcp-agent is a thin orchestration layer: as shipped it adds no approval gate, no sandbox, and no argument validation between the model and its MCP tools. The scaffolded default gives one unattended session web fetch, a filesystem server over the project directory, and the plaintext secrets file in that same directory, so a prompt injection can read API keys and overwrite files. Sub-agent specs and config are also auto-loaded from the working directory, where they can trigger Python module imports. Opt-in OAuth scoping and OpenTelemetry tracing are the strongest controls on offer.

## Critical gaps
- Configured stdio MCP servers and local tools run unsandboxed as the user, with the default filesystem server rooted at the project that holds the plaintext secrets file. (ASI05, T11; C4) — [src/mcp_agent/mcp/mcp_server_registry.py:162-166](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/mcp/mcp_server_registry.py#L162-L166); [src/mcp_agent/data/templates/mcp_agent.config.yaml:29-31](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/mcp_agent.config.yaml#L29-L31)
- Rule of Two is broken by the default template: fetch, a filesystem server over the project (including the secrets file), and file write combine in one unattended session. (ASI01, LLM01, T6; C5) — [src/mcp_agent/data/templates/mcp_agent.config.yaml:29-31](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/mcp_agent.config.yaml#L29-L31); [src/mcp_agent/data/templates/mcp_agent.config.yaml:34-36](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/mcp_agent.config.yaml#L34-L36)
- Sub-agent auto-discovery is on by default and imports Python modules named in .claude/agents files from the working directory; config and .env are also discovered from the working directory with no trust prompt. (ASI06, T1, ASI04; C6) — [src/mcp_agent/config.py:638-645](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/config.py#L638-L645); [src/mcp_agent/workflows/factory.py:661-669](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/workflows/factory.py#L661-L669); [src/mcp_agent/config.py:1242-1257](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/config.py#L1242-L1257)
- The plaintext secrets file lives inside the directory the default filesystem MCP server exposes, so the model can read long-lived API keys. (ASI03, LLM02; C8) — [src/mcp_agent/config.py:1480-1490](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/config.py#L1480-L1490); [src/mcp_agent/data/templates/mcp_agent.config.yaml:29-31](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/mcp_agent.config.yaml#L29-L31)

## Criterion details

### C1 Identity & least privilege — 0.40 (high)

By default an mcp-agent app runs with the developer's own authority: LLM API keys come from env vars or a plaintext secrets file, and stdio MCP servers are launched as the same OS user. The one narrowing step is that stdio servers get a minimal default environment plus their configured variables rather than the full parent environment. For HTTP MCP servers the framework offers an opt-in OAuth client whose tokens are cached per user identity and per requested scope set, which is a real per-capability scoping primitive but is off unless configured per server.

- **default configuration** (default; raw 0.12 → 0.12)
  - **S L0:** Default authority is the operator's ambient OS user and long-lived API keys; no dedicated or scoped identity is created. — [src/mcp_agent/mcp/mcp_server_registry.py:162-166](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/mcp/mcp_server_registry.py#L162-L166); [src/mcp_agent/data/templates/secrets.yaml:7-8](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/secrets.yaml#L7-L8) (verified)
    - *To reach the next level:* No dedicated agent identity or per-tool scoped credential in the default flow.
  - **C L1:** stdio subprocesses receive a reduced default environment plus configured env, but no tool call passes an authorization check. — [src/mcp_agent/mcp/mcp_server_registry.py:162-166](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/mcp/mcp_server_registry.py#L162-L166); [src/mcp_agent/agents/agent.py:1105-1124](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/agents/agent.py#L1105-L1124) (verified)
    - *To reach the next level:* No authorization layer that every tool path traverses.
  - **D L0:** The scaffolded default config gives the agent a filesystem server rooted at the project directory under the user's own account. — [src/mcp_agent/data/templates/mcp_agent.config.yaml:29-31](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/mcp_agent.config.yaml#L29-L31) (verified)
    - *To reach the next level:* No near-minimal default identity; least privilege requires manual configuration.
  - **B L1:** A hijacked agent can write the project tree and reach any host via fetch, with the operator's LLM keys in-process. — [src/mcp_agent/data/templates/mcp_agent.config.yaml:29-31](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/mcp_agent.config.yaml#L29-L31); [src/mcp_agent/data/templates/mcp_agent.config.yaml:34-36](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/mcp_agent.config.yaml#L34-L36) (verified)
    - *To reach the next level:* Authority is not limited to one system or to mostly-read access.
- **opt-in per-server OAuth with per-user token store** (alt; raw 0.40, cap G1 → 0.40) ← counted
  - **S L3:** When enabled for an HTTP server, OAuth tokens are requested with operator-listed scopes and cached under a key of user identity, resource, and scope fingerprint. — [src/mcp_agent/config.py:91-95](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/config.py#L91-L95); [src/mcp_agent/mcp/mcp_server_registry.py:236-245](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/mcp/mcp_server_registry.py#L236-L245); [src/mcp_agent/oauth/store/base.py:10-18](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/oauth/store/base.py#L10-L18) (verified)
    - *To reach the next level:* Tokens are not exchanged/downscoped per request or revoked after use.
  - **C L1:** Only HTTP servers individually configured with OAuth use it; stdio servers and local functions keep ambient authority. — [src/mcp_agent/config.py:91-95](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/config.py#L91-L95); [src/mcp_agent/mcp/mcp_server_registry.py:162-166](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/mcp/mcp_server_registry.py#L162-L166) (verified)
    - *To reach the next level:* stdio servers, local functions, and sub-agents do not go through the scoped identity.
  - **D L0:** OAuth for a downstream server defaults to disabled. — [src/mcp_agent/config.py:91-95](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/config.py#L91-L95) (verified)
    - *To reach the next level:* Scoped identity is not on by default.
  - **B L2:** With OAuth, a stolen token is bounded by the scopes the operator requested for that one server. — [src/mcp_agent/oauth/store/base.py:10-18](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/oauth/store/base.py#L10-L18) (verified)
    - *To reach the next level:* Tokens are not short-lived task-scoped credentials limited to one tenant.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C2 Approval gates — 0.00 (high)

The framework has no approval gate for tool calls. The hook that runs before each tool call simply returns the request unchanged unless a developer subclasses it, and the human-input tool is something the model chooses to call, not a checkpoint on actions. The only built-in human approval covers MCP sampling requests, not tool execution. In the scaffolded default config the agent can overwrite project files and fetch arbitrary URLs with no human in the loop.

- **S L0:** pre_tool_call is a no-op that returns the request; no tool call requires human approval. — [src/mcp_agent/workflows/llm/augmented_llm.py:559-563](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/workflows/llm/augmented_llm.py#L559-L563); searched `rg -n -S 'requires_approval|tool_approval|approve_tool|auto_approve'` in `src/mcp_agent` → 0 hits (No approval primitive for tool calls anywhere in the package.) (verified)
  - *To reach the next level:* No per-call human approval showing the exact call.
- **C L0:** Every dispatch path (local functions, MCP tools) executes directly. — [src/mcp_agent/agents/agent.py:1105-1124](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/agents/agent.py#L1105-L1124) (verified)
  - *To reach the next level:* The most powerful tools are not gated.
- **D L0:** No approval exists by default; adding one requires custom code. — [src/mcp_agent/workflows/llm/augmented_llm.py:559-563](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/workflows/llm/augmented_llm.py#L559-L563) (verified)
  - *To reach the next level:* Approval is not on by default.
- **B L0:** Default filesystem server can overwrite files in the project with no checkpoint, alongside unrestricted fetch. — [src/mcp_agent/data/templates/mcp_agent.config.yaml:29-31](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/mcp_agent.config.yaml#L29-L31); [src/mcp_agent/data/templates/mcp_agent.config.yaml:34-36](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/mcp_agent.config.yaml#L34-L36) (verified)
  - *To reach the next level:* No reversibility or checkpointing for file or external actions.
- **Cap:** none

### C3 Tool & action scoping — 0.25 (high)

Tool arguments are passed through to MCP servers without framework-side validation; only local Python function tools get typed pydantic parsing. Enforcement of the per-server allowed_tools setting does not cover every path. The default template enables filesystem write and unrestricted web fetch.

- **S L1:** No path or URL validation exists, and allowed_tools enforcement does not cover every path. — searched `rg -n -S 'realpath|is_relative_to|allowed_hosts|169\.254'` in `src/mcp_agent` → 3 hits (All 3 hits are in a bundled example's yarn.lock; the framework has no path/URL argument validation.) (verified)
  - *To reach the next level:* No allowlist validation of arguments (resolved paths, host allowlists, bounds) in code.
- **C L1:** Only local function tools get typed schema parsing; MCP tool arguments pass through. — [src/mcp_agent/agents/agent.py:1105-1124](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/agents/agent.py#L1105-L1124); [src/mcp_agent/mcp/mcp_aggregator.py:895-898](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/mcp/mcp_aggregator.py#L895-L898) (verified)
  - *To reach the next level:* Most tools, including MCP tools, are not validated.
- **D L1:** Default template enables filesystem (read/write) and fetch; tools can be filtered per server via allowed_tools. — [src/mcp_agent/data/templates/mcp_agent.config.yaml:29-31](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/mcp_agent.config.yaml#L29-L31); [src/mcp_agent/data/templates/mcp_agent.config.yaml:34-36](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/mcp_agent.config.yaml#L34-L36) (verified)
  - *To reach the next level:* Default tool set includes write and network tools.
- **B L1:** Misuse reaches the whole project directory with write access and any internet host; an alternative shipped template roots filesystem at /. — [src/mcp_agent/data/templates/mcp_agent.config.yaml:29-31](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/mcp_agent.config.yaml#L29-L31); [src/mcp_agent/data/templates/config_claude.yaml:14-17](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/config_claude.yaml#L14-L17) (verified)
  - *To reach the next level:* Not scoped to a workspace without broad egress.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (high)

mcp-agent has no isolation mechanism. Configured stdio MCP servers, including the default filesystem server launched with npx, run as ordinary subprocesses of the same OS user, and local function tools run in the agent's own process. Any code those paths execute can reach the user's home directory, network, and the project's secrets file.

- **S L0:** stdio servers are spawned directly on the host as the same user; no sandbox primitive exists. — [src/mcp_agent/mcp/mcp_server_registry.py:162-166](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/mcp/mcp_server_registry.py#L162-L166); searched `rg -n -S 'sandbox|seccomp|gvisor|firecracker|nsjail|bwrap'` in `src/mcp_agent` → 9 hits (All hits are Temporal workflow-sandbox references or comments in a bundled ChatGPT web example; none isolates executed code.) (verified)
  - *To reach the next level:* No OS-level separation of executed code.
- **C L0:** No execution path is sandboxed. — [src/mcp_agent/mcp/mcp_server_registry.py:162-166](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/mcp/mcp_server_registry.py#L162-L166); [src/mcp_agent/agents/agent.py:1105-1124](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/agents/agent.py#L1105-L1124) (verified)
  - *To reach the next level:* No path goes through an isolation boundary.
- **D L0:** There is nothing to enable; host execution is the only mode. — searched `rg -n -S 'sandbox|seccomp|gvisor|firecracker|nsjail|bwrap'` in `src/mcp_agent` → 9 hits (All hits are Temporal workflow-sandbox references or comments in a bundled ChatGPT web example; none isolates executed code.) (verified)
  - *To reach the next level:* Isolation is not on by default.
- **B L0:** Spawned servers run as the user with HOME available and the default filesystem server rooted at the project containing the secrets file. — [src/mcp_agent/mcp/mcp_server_registry.py:162-166](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/mcp/mcp_server_registry.py#L162-L166); [src/mcp_agent/data/templates/mcp_agent.config.yaml:29-31](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/mcp_agent.config.yaml#L29-L31); [src/mcp_agent/data/templates/secrets.yaml:7-8](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/secrets.yaml#L7-L8) (verified)
  - *To reach the next level:* Executed code can reach the host filesystem and credentials.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

Tool and MCP results are appended to the conversation as ordinary tool messages with no untrusted marking, and nothing in the framework changes what the agent may do after reading external content. In the scaffolded default, one session combines web fetch (untrusted input and an exfiltration channel), the project directory including the secrets file, and file write. A successful prompt injection can therefore both leak keys and overwrite files without any human involvement.

- **S L0:** No structural limit on a hijacked agent; tool results enter context unmarked. — [src/mcp_agent/workflows/llm/augmented_llm_openai.py:633-635](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/workflows/llm/augmented_llm_openai.py#L633-L635); searched `rg -n -S 'untrusted|taint|provenance|prompt_injection|quarantin'` in `src/mcp_agent` → 0 hits (No untrusted-content marking or taint tracking.) (verified)
  - *To reach the next level:* No approval or capability restriction triggered by untrusted content.
- **C L0:** Untrusted sources are not distinguished from principal input. — [src/mcp_agent/workflows/llm/augmented_llm_openai.py:633-635](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/workflows/llm/augmented_llm_openai.py#L633-L635) (verified)
  - *To reach the next level:* No untrusted source is handled.
- **D L0:** No mechanism exists to be on by default. — searched `rg -n -S 'untrusted|taint|provenance|prompt_injection|quarantin'` in `src/mcp_agent` → 0 hits (No untrusted-content marking or taint tracking.) (verified)
  - *To reach the next level:* No default control.
- **B L0:** Default session has fetch (input and egress), the secrets file in the filesystem root, and write access, all unattended. — [src/mcp_agent/data/templates/mcp_agent.config.yaml:29-31](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/mcp_agent.config.yaml#L29-L31); [src/mcp_agent/data/templates/mcp_agent.config.yaml:34-36](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/mcp_agent.config.yaml#L34-L36); [src/mcp_agent/config.py:1480-1490](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/config.py#L1480-L1490) (verified)
  - *To reach the next level:* Exfiltration and irreversible writes are not gated by a human.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.10 (high)

Conversation memory is in-process and session-scoped, but configuration is not. At startup the app searches the current directory and every parent for its config and secrets files, reads a .env file from the working directory (which can set values such as the OpenAI base URL), and auto-loads sub-agent definitions from .claude/agents and .mcp-agent/agents in the working directory. Those definition files can name Python callables, which the loader imports by module name, so files in the workspace can trigger code import without any trust prompt. Because the default filesystem server is rooted at the same directory, the agent itself can write these files for the next run.

- **S L0:** Workspace files (config discovered from cwd upward, .env, .claude/agents specs with importable function refs) load silently and can add MCP servers or trigger imports. — [src/mcp_agent/config.py:1242-1257](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/config.py#L1242-L1257); [src/mcp_agent/config.py:638-645](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/config.py#L638-L645); [src/mcp_agent/app.py:371-386](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/app.py#L371-L386); [src/mcp_agent/workflows/factory.py:661-669](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/workflows/factory.py#L661-L669); [src/mcp_agent/workflows/factory.py:723-728](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/workflows/factory.py#L723-L728) (verified)
  - *To reach the next level:* Security-relevant workspace config is not behind an explicit trust decision.
- **C L0:** None of the auto-loaded config paths is controlled. — [src/mcp_agent/config.py:1140-1146](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/config.py#L1140-L1146); [src/mcp_agent/app.py:371-386](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/app.py#L371-L386) (verified)
  - *To reach the next level:* No memory or config path is controlled.
- **D L1:** Chat memory is a per-LLM in-process list, so it is not shared across users, but nothing isolates the auto-loaded workspace files. — [src/mcp_agent/workflows/llm/augmented_llm.py:107-112](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/workflows/llm/augmented_llm.py#L107-L112) (verified)
  - *To reach the next level:* Isolation of persistent context is not enforced beyond process scope.
- **B L1:** A poisoned config or agent-spec file persists across the user's runs and can add tool servers or import code. — [src/mcp_agent/app.py:371-386](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/app.py#L371-L386); [src/mcp_agent/workflows/factory.py:661-669](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/workflows/factory.py#L661-L669) (verified)
  - *To reach the next level:* Poisoned context can trigger tool use and code import on later runs.
- **Cap:** C6-REPOCONFIG — Files in the working directory (which the default filesystem server can write) are auto-loaded without a trust decision and can add MCP servers, redirect the model endpoint via .env, or cause module imports via sub-agent specs.

### C7 Third-party extensions — 0.05 (high)

MCP servers are whatever the config names, and the scaffolded template launches them with npx -y and uvx with no version pins, so the latest package is fetched and executed at each launch. There is no integrity check or re-approval when a server changes. stdio servers do get a reduced default environment, but they run as the same user, and function references in sub-agent spec files are imported into the agent's own process.

- **S L0:** Default template runs npx -y and uvx with unpinned packages; no integrity verification exists. — [src/mcp_agent/data/templates/mcp_agent.config.yaml:29-31](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/mcp_agent.config.yaml#L29-L31); [src/mcp_agent/data/templates/mcp_agent.config.yaml:34-36](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/mcp_agent.config.yaml#L34-L36); searched `rg -n -S 'sha256|integrity|checksum|signature'` in `src/mcp_agent/mcp src/mcp_agent/config.py` → 0 hits (No extension pinning or integrity verification in the MCP client/config layer.) (verified)
  - *To reach the next level:* Extensions are not pinned.
- **C L0:** No extension type is verified. — searched `rg -n -S 'sha256|integrity|checksum|signature'` in `src/mcp_agent/mcp src/mcp_agent/config.py` → 0 hits (No extension pinning or integrity verification in the MCP client/config layer.) (verified)
  - *To reach the next level:* No extension type is verified.
- **D L0:** Workspace files (.env, discovered config, .claude/agents) can add servers or imported callables silently. — [src/mcp_agent/config.py:1242-1257](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/config.py#L1242-L1257); [src/mcp_agent/app.py:371-386](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/app.py#L371-L386) (verified)
  - *To reach the next level:* Workspace files can add extensions without consent.
- **B L1:** stdio servers are separate same-user processes with a reduced default environment; function refs from spec files run in-process. — [src/mcp_agent/mcp/mcp_server_registry.py:162-166](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/mcp/mcp_server_registry.py#L162-L166); [src/mcp_agent/workflows/factory.py:661-669](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/workflows/factory.py#L661-L669) (verified)
  - *To reach the next level:* Extensions are not uniformly separate processes with only their own configuration.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.25 (high)

API keys come from env vars or a plaintext, gitignored secrets file that sits in the project directory. The log serializer masks values under sensitive-looking keys, but leaves the first ten characters visible, and an environment variable turns masking off entirely. Usage telemetry is flagged on but currently sends nothing. The biggest problem is placement: the default filesystem server is rooted at the same directory as the secrets file, so the model can read the long-lived keys.

- **S L1:** Secrets from env/plaintext YAML; partial masking (first 10 chars shown) in the log serializer only. — [src/mcp_agent/logging/json_serializer.py:40-50](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/logging/json_serializer.py#L40-L50); [src/mcp_agent/data/templates/secrets.yaml:7-8](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/secrets.yaml#L7-L8) (verified)
  - *To reach the next level:* No type-level masking or full redaction on main paths.
- **C L1:** Masking applies to the structured log serializer; tool results, model-bound messages, and OTel attributes are not redacted. — [src/mcp_agent/logging/json_serializer.py:40-50](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/logging/json_serializer.py#L40-L50); [src/mcp_agent/workflows/llm/augmented_llm.py:587-591](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/workflows/llm/augmented_llm.py#L587-L591) (verified)
  - *To reach the next level:* Redaction does not cover transcripts and model-bound messages.
- **D L2:** Telemetry is a no-op despite enabled=True; logging defaults to info; redaction can be disabled with LOG_SECRETS. — [src/mcp_agent/telemetry/usage_tracking.py:7-13](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/telemetry/usage_tracking.py#L7-L13); [src/mcp_agent/config.py:742](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/config.py#L742); [src/mcp_agent/logging/json_serializer.py:40](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/logging/json_serializer.py#L40) (verified)
  - *To reach the next level:* Redaction is not always on.
- **B L0:** Long-lived LLM keys in the secrets file are readable by the model through the default filesystem server rooted at the project directory. — [src/mcp_agent/config.py:1480-1490](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/config.py#L1480-L1490); [src/mcp_agent/data/templates/mcp_agent.config.yaml:29-31](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/mcp_agent.config.yaml#L29-L31) (verified)
  - *To reach the next level:* High-privilege keys are reachable by the model.
- **Cap:** none

### C9 Audit & traceability — 0.35 (medium)

By default the aggregator logs each MCP tool call at info level with tool, server, and agent names but not the arguments, and the scaffolded config writes these logs to a file inside the project directory that the agent's own filesystem tool can edit. Logs are batched and flushed every two seconds. Opt-in OpenTelemetry tracing records tool arguments and results under agent-named spans and can export over OTLP, which is a much better record but is off by default.

- **default configuration** (default; raw 0.25 → 0.25)
  - **S L1:** Info-level event per MCP tool call without arguments. — [src/mcp_agent/mcp/mcp_aggregator.py:871-879](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/mcp/mcp_aggregator.py#L871-L879) (verified)
    - *To reach the next level:* No structured record of every tool call with arguments in the default config.
  - **C L1:** Only the MCP aggregator path logs; local function and human-input tool calls are not logged by default. — [src/mcp_agent/mcp/mcp_aggregator.py:871-879](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/mcp/mcp_aggregator.py#L871-L879); [src/mcp_agent/agents/agent.py:1105-1124](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/agents/agent.py#L1105-L1124) (verified)
    - *To reach the next level:* Not all built-in tool paths are recorded.
  - **D L1:** Default template writes logs to logs/mcp-agent.log inside the filesystem server's root. — [src/mcp_agent/data/templates/mcp_agent.config.yaml:20-23](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/mcp_agent.config.yaml#L20-L23); [src/mcp_agent/data/templates/mcp_agent.config.yaml:29-31](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/data/templates/mcp_agent.config.yaml#L29-L31) (verified)
    - *To reach the next level:* Logs are stored where the agent's tools can edit them.
  - **B L1:** Logs are batched and flushed on an interval. — [src/mcp_agent/config.py:1114-1118](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/config.py#L1114-L1118) (verified)
    - *To reach the next level:* Records are not flushed per action.
- **opt-in OpenTelemetry tracing** (alt; raw 0.35, cap G1 → 0.35) ← counted
  - **S L2:** With tracing on, tool-call spans record name, arguments, and agent name. — [src/mcp_agent/workflows/llm/augmented_llm.py:587-591](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/workflows/llm/augmented_llm.py#L587-L591); [src/mcp_agent/core/context.py:93](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/core/context.py#L93) (verified)
    - *To reach the next level:* No requesting-principal/approver attribution or tamper-evident storage.
  - **C L2:** Spans cover LLM-level and agent-level tool calls including local functions. — [src/mcp_agent/workflows/llm/augmented_llm.py:587-591](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/workflows/llm/augmented_llm.py#L587-L591); [src/mcp_agent/agents/agent.py:1083-1093](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/agents/agent.py#L1083-L1093) (verified)
    - *To reach the next level:* Approvals/denials and extension internals are not recorded.
  - **D L0:** OTel is disabled by default. — [src/mcp_agent/config.py:825](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/config.py#L825); [src/mcp_agent/core/context.py:93](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/core/context.py#L93) (verified)
    - *To reach the next level:* Tracing is opt-in.
  - **B L1:** Span export is handled by the OTel SDK's exporters (batching behaviour inferred from library defaults). — [src/mcp_agent/config.py:825](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/config.py#L825) (inferred)
    - *To reach the next level:* Records are not durable per action.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C10 Limits & kill switch — 0.30 (high)

Each LLM generate call is limited to 10 tool-use iterations and 2048 output tokens per completion. There is no wall-clock limit, no cost cap, and tool calls have no timeout by default, so a hanging MCP server blocks indefinitely. Orchestrator, router, and swarm patterns create sub-agents that each get their own fresh iteration budget. A budgeted deep-orchestrator workflow exists but applies only to that workflow.

- **S L1:** Iteration cap of 10 per generate call; per-completion output-token cap only. — [src/mcp_agent/workflows/llm/augmented_llm.py:155](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/workflows/llm/augmented_llm.py#L155); [src/mcp_agent/workflows/llm/augmented_llm.py:141](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/workflows/llm/augmented_llm.py#L141); [src/mcp_agent/workflows/llm/augmented_llm_openai.py:261](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/workflows/llm/augmented_llm_openai.py#L261) (verified)
  - *To reach the next level:* No wall-clock, per-tool timeout, or total token/cost cap by default.
- **C L1:** Limit applies to the top-level loop of each LLM; executor and MCP reads have no timeout. — [src/mcp_agent/executor/executor.py:46-47](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/executor/executor.py#L46-L47); [src/mcp_agent/config.py:254](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/config.py#L254) (verified)
  - *To reach the next level:* Tool calls have no default timeout and sub-agents get fresh budgets.
- **D L2:** Default of 10 iterations is sensible and set by the developer, not the model. — [src/mcp_agent/workflows/llm/augmented_llm.py:155](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/workflows/llm/augmented_llm.py#L155) (verified)
  - *To reach the next level:* Sub-agents reset the budget by delegation.
- **B L1:** A hung tool or server call has no ceiling. — [src/mcp_agent/executor/executor.py:46-47](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/executor/executor.py#L46-L47); [src/mcp_agent/config.py:254](https://github.com/lastmile-ai/mcp-agent/blob/f62d849350816588b1c6294e7914bbe4d8b84072/src/mcp_agent/config.py#L254) (verified)
  - *To reach the next level:* No tight per-run time ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: fetch MCP server in default template (src/mcp_agent/data/templates/mcp_agent.config.yaml:34-36) · [B] sensitive data/systems: secrets file loaded from the project dir that the filesystem server exposes (src/mcp_agent/config.py:1480-1490, mcp_agent.config.yaml:29-31) · [C] state change / egress: filesystem write and fetch egress, no gate (src/mcp_agent/workflows/llm/augmented_llm.py:559-563) · Same default session? Yes

## Highest-impact improvements
1. Turn sub-agent auto-discovery off by default (or require an explicit trust flag) and never import function references from working-directory spec files. — C6 S L0→L2, +0.150 before caps (Playbook 2)
2. Harden enforcement of the allowed_tools setting so it holds on every call path. — C3 S L1→L2, +0.075 before caps (Playbook 3)
3. Ship a default pre_tool_call approval that prompts for any tool not annotated readOnlyHint, showing the exact arguments. — C2 S L0→L3, +0.225 before caps (Playbook 5)
4. Scaffold the filesystem server onto a dedicated workspace subdirectory that excludes the secrets and config files. — C8 B L0→L1, +0.050 before caps (Playbook 4)
5. Set a default per-tool-call timeout and a run-level wall-clock limit shared with sub-agents. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Behaviour of the upstream MCP SDK's get_default_environment and pydantic-settings .env/nested-env merging is inferred from library conventions, not read in this repo.
- The CLI cloud deployment commands, Temporal worker internals, and bundled examples were reviewed only where they set defaults.
- No text aimed at AI reviewers was found in the repository.
