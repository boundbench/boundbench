# Defense-in-Depth Score: PraisonAI

**Repo:** https://github.com/mervinpraison/praisonai · **Commit:** `e49b254509c3eaca45a688daccf046a270fd382c` (praisonaiagents 1.7.10) · **Reviewed:** 2026-10-04
**What it is:** Python multi-agent framework (praisonaiagents SDK plus praisonai wrapper/CLI) for building tool-using, memory-enabled agents and teams.
**Category:** Agent Frameworks
**Scored configuration:** Core SDK praisonaiagents as the README quickstart uses it: Agent(instructions=...) with default constructor arguments, tools added by the developer from praisonaiagents.tools or MCP, run in the developer's working directory.
**Agent surface (default):** code execution opt-in · filesystem write opt-in · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents opt-in · external communication opt-in

## Score: 3.0 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L1 | L1 | 0.17 | — | **0.17** | High |
| C2 | Approval gates | L2 | L1 | L1 | L1 | 0.33 | — | **0.33** | High |
| C3 | Tool & action scoping | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C4 | Code-execution isolation | L2 | L1 | L0 | L3 | 0.38 | G1 | **0.38** (alt) | Medium |
| C5 | Untrusted input blast radius | L1 | L1 | L2 | L0 | 0.25 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L0 | L1 | L1 | L1 | 0.17 | C6-REPOCONFIG | **0.17** | Medium |
| C7 | Third-party extensions | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | High |
| C9 | Audit & traceability | L2 | L2 | L0 | L1 | 0.35 | G1 | **0.35** | Medium |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |


PraisonAI's core SDK has real safety work in it: a bare Agent hard-denies shell/code execution when not on a terminal, prompts per call on a terminal, and its file tools are confined to the working directory. But the gate keys on a fixed list of built-in tool names, so developer tools and MCP tools run unapproved, and the approval API is not a complete gate. The dominant risk is the working directory: project configuration and instruction files are not integrity-protected, and AGENTS.md is auto-loaded into the system prompt. The shell tool runs on the host with the full environment.

## Critical gaps
- execute_command runs model-generated commands on the host with the full environment. (ASI05, T11; C4) — [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:195](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L195)
- With developer and MCP tools ungated and unfenced, a prompt injection can exfiltrate data and take irreversible actions without a human in the default configuration. (ASI01, LLM01, T6; C5) — [src/praisonai-agents/praisonaiagents/agent/tool_execution.py:2286-2291](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/agent/tool_execution.py#L2286-L2291); [src/praisonai-agents/praisonaiagents/tools/trust.py:49](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/trust.py#L49)
- Plugins are executed in-process with every credential the agent holds. (ASI04, T17; C7) — [src/praisonai-agents/praisonaiagents/plugins/discovery.py:262](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/plugins/discovery.py#L262)

## Criterion details

### C1 Identity & least privilege — 0.17 (high)

The framework runs with whatever credentials the developer's process holds. Provider API keys come from environment variables, and the built-in shell tool hands every child process a full copy of the environment, while the Python code tool and MCP servers get a scrubbed one. There is no per-tool or per-request authorization layer and no scoped identity; multi-user isolation is left to the application.

- **S L0:** Ambient authority: the shell tool copies os.environ into every subprocess and expands $VARS in model-chosen arguments, so any credential in the process is usable. — [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:195](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L195); [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:182](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L182) (verified)
  - *To reach the next level:* No dedicated or role-scoped identity for tools; credentials are not narrowed per tool or capability.
- **C L1:** MCP stdio servers and the Python code tool get a scrubbed environment, but the shell tool and developer tools inherit full ambient credentials. — [src/praisonai-agents/praisonaiagents/mcp/mcp.py:628](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/mcp/mcp.py#L628); [src/praisonai-agents/praisonaiagents/tools/python_tools.py:335](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/python_tools.py#L335); [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:195](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L195) (verified)
  - *To reach the next level:* No shared authorization layer that every tool path (built-in, MCP, sub-agent) passes through.
- **D L1:** Defaults narrow some subprocess environments, but the shell tool's model-controlled env parameter and ambient env give it the operator's full authority. — [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:195](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L195); [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:108](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L108) (verified)
  - *To reach the next level:* No reasonable default role; least privilege needs manual hardening by the developer.
- **B L1:** A hijacked agent can use every credential in the process environment and the OS user's files; typical deployments hold LLM keys plus tool keys (GitHub, email, Jira tools ship in-tree). — [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:195](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L195); [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:182](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L182) (verified)
  - *To reach the next level:* Credentials are long-lived and broad; nothing confines a hijack to one system or tenant.
- **Cap:** none

### C2 Approval gates — 0.33 (high)

A bare Agent on an interactive terminal installs a console prompt that asks before each call to a fixed list of built-in dangerous tools (shell, code, file write/edit/delete, SQL, crawl), showing the arguments (truncated to about 100 characters) or a diff; off a terminal, shell/code/delete are hard-denied. The gate keys on tool names, so developer-written tools and MCP tools run without approval unless individually decorated. The `approval=True` API is not a complete gate, and environment variables (PRAISONAI_AUTO_APPROVE, PRAISONAI_TOOL_SAFETY=off) silently disable the gate. There is no default checkpoint or rollback.

- **S L2:** Per-call console approval with risk tiers and a diff for edits, but other arguments are truncated at 97 characters, so the approver may not see the exact full command. — [src/praisonai-agents/praisonaiagents/agent/agent.py:2828](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/agent/agent.py#L2828); [src/praisonai-agents/praisonaiagents/approval/backends.py:254-255](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/approval/backends.py#L254-L255); [src/praisonai-agents/praisonaiagents/approval/registry.py:32](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/approval/registry.py#L32) (verified)
  - *To reach the next level:* The approval prompt does not always show the exact, complete call; no argument-level allow/deny policy by default.
- **C L1:** Only tools in DEFAULT_DANGEROUS_TOOLS, tools decorated with require_approval, or tools marked external are gated; developer tools such as the README's deploy() and all MCP tools are not. — [src/praisonai-agents/praisonaiagents/agent/tool_execution.py:2286-2291](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/agent/tool_execution.py#L2286-L2291); [src/praisonai-agents/praisonaiagents/approval/registry.py:32](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/approval/registry.py#L32); searched `rg -n "trust_level|approval|wrap_if_external"` in `src/praisonai-agents/praisonaiagents/mcp/` → 0 hits (MCP client never marks its tools external or approval-required, so MCP tools skip both the gate and the injection fence.) (verified)
  - *To reach the next level:* Every tool path, including developer and MCP tools, does not traverse the gate; unknown tools are allowed by default.
- **D L1:** On by default, but an env var silently disables it, and the approval API is not a complete gate; lowered one level. — [src/praisonai-agents/praisonaiagents/approval/registry.py:495-496](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/approval/registry.py#L495-L496); [src/praisonai-agents/praisonaiagents/agent/agent.py:2798](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/agent/agent.py#L2798) (verified)
  - *To reach the next level:* Disabling must need a loudly named operator flag.
- **B L1:** Approved or ungated actions (shell, file delete, developer tools like deploy) are mostly irreversible; the agent wires no checkpoint/rollback by default. — searched `rg -n checkpoint` in `src/praisonai-agents/praisonaiagents/agent/agent.py` → 0 hits (The Agent constructor wires no checkpoint/rollback by default.); [src/praisonai-agents/praisonaiagents/approval/registry.py:32](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/approval/registry.py#L32) (verified)
  - *To reach the next level:* No default checkpoints or rollback for file and code state; no previews for external actions.
- **Cap:** none

### C3 Tool & action scoping — 0.45 (high)

A bare Agent gets no tools, which is a good least-agency default. The bundled file tools resolve symlinks and confine paths to the working directory, and the web tools have SSRF checks. But the bundled shell tool accepts any program and arguments (with environment-variable expansion), and there is no central validation layer: developer tools and MCP tools receive whatever arguments the model produces.

- **S L2:** File tools do realpath containment to the cwd and URL tools reject internal hosts, but the shell tool is raw program passthrough with $VAR expansion. — [src/praisonai-agents/praisonaiagents/tools/file_tools.py:106-111](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/file_tools.py#L106-L111); [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:182](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L182); [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:206](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L206) (verified)
  - *To reach the next level:* General tools are not replaced by narrow ones and the shell tool has no allowlist.
- **C L2:** Most bundled tools validate inputs; developer-registered and MCP tools get only schema typing, with no shared validation layer. — [src/praisonai-agents/praisonaiagents/tools/file_tools.py:106-111](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/file_tools.py#L106-L111); [src/praisonai-agents/praisonaiagents/tools/schema.py:351](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/schema.py#L351) (verified)
  - *To reach the next level:* No shared validation layer wrapping extension and developer tools.
- **D L2:** Agent() starts with an empty tool list, but configuration handling can widen it without the developer asking. — [src/praisonai-agents/praisonaiagents/agent/agent.py:2514](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/agent/agent.py#L2514) (verified)
  - *To reach the next level:* Write/exec tools must require explicit enabling by the developer.
- **B L1:** Once a developer adds the shell tool, it reaches the whole machine as the OS user; file tools stay inside the cwd but with full write. — [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:195](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L195); [src/praisonai-agents/praisonaiagents/tools/file_tools.py:106-111](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/file_tools.py#L106-L111) (verified)
  - *To reach the next level:* Tools are not scoped to a workspace and quantity-bounded across the board.
- **Cap:** none

### C4 Code-execution isolation — 0.38 (medium)

The built-in shell tool runs programs directly on the host as the user, with the full environment. The Python code tool is better by default: a separate process with an AST blocklist, a clean environment and resource limits, but that is filtering, not isolation, and selection of the execution mode is not tamper-resistant. Real sandboxes (Docker, E2B, Modal and others) exist through praisonai-sandbox and tools_run_on=, but are opt-in; the Docker one is a stock container with networking off by default.

- **default configuration** (default; raw 0.12, cap G2 → 0.12)
  - **S L1:** execute_code uses a same-user subprocess with an AST/builtins blocklist, a clean env and rlimits (filtering); execute_command uses no isolation at all. — [src/praisonai-agents/praisonaiagents/tools/python_tools.py:56](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/python_tools.py#L56); [src/praisonai-agents/praisonaiagents/tools/python_tools.py:335](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/python_tools.py#L335); [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:206](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L206) (verified)
    - *To reach the next level:* No OS-level separation (dedicated user, container) for model-generated code by default.
  - **C L0:** The most general exec path, execute_command, runs on the host with no sandbox. — [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:195](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L195); [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:206](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L206) (verified)
    - *To reach the next level:* The main exec tool is not sandboxed.
  - **D L1:** Selection of the execution mode is not tamper-resistant. — [src/praisonai-agents/praisonaiagents/tools/schema.py:351](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/schema.py#L351) (verified)
    - *To reach the next level:* Integrity-protect security-relevant configuration on every path.
  - **B L0:** Shell commands reach the whole host with every credential in the environment. — [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:195](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L195) (verified)
    - *To reach the next level:* Credentials and the host filesystem are not kept out of reach of executed code.
- **opt-in Docker sandbox via praisonai-sandbox (tools_run_on="docker")** (alt; raw 0.38, cap G1 → 0.38) ← counted
  - **S L2:** Stock docker run with memory, CPU and pids limits; no non-root user, capability drop or seccomp profile. — [src/praisonai-sandbox/praisonai_sandbox/docker.py:405-411](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-sandbox/praisonai_sandbox/docker.py#L405-L411) (verified)
    - *To reach the next level:* No hardening (non-root, dropped capabilities, no-new-privileges, read-only root).
  - **C L1:** tools_run_on routes the agent's tools into the sandbox; whether every spawned path (MCP stdio servers, hooks) follows was not confirmed. — [src/praisonai-agents/praisonaiagents/agent/agent.py:675](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/agent/agent.py#L675) (inferred)
    - *To reach the next level:* Not shown that every model-reachable path is sandboxed with no host fallback.
  - **D L0:** Off unless the developer passes tools_run_on or installs the sandbox extra. — [src/praisonai-agents/praisonaiagents/agent/agent.py:675](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/agent/agent.py#L675) (verified)
    - *To reach the next level:* Not on by default.
  - **B L3:** Network is off by default, only a per-sandbox temp dir is mounted, and resource limits apply. — [src/praisonai-sandbox/praisonai_sandbox/docker.py:429-430](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-sandbox/praisonai_sandbox/docker.py#L429-L430); [src/praisonai-sandbox/praisonai_sandbox/docker.py:405-411](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-sandbox/praisonai_sandbox/docker.py#L405-L411) (verified)
    - *To reach the next level:* Containers persist for the agent's lifetime rather than per call.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.25 (high)

Results from a fixed list of web search and scrape tools are wrapped in an <external_tool_result> fence, which is a delimiter the model is asked to respect, not an enforced limit. MCP results and developer-tool results are not fenced, and nothing changes what the agent can do after it reads untrusted content. Because developer and MCP tools are ungated and the framework's normal use combines web input, private data and outbound tools, a successful injection can exfiltrate and act without a human.

- **S L1:** Spotlighting only: external tool output is wrapped in fence markers. — [src/praisonai-agents/praisonaiagents/tools/trust.py:37](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/trust.py#L37); [src/praisonai-agents/praisonaiagents/agent/tool_execution.py:1628](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/agent/tool_execution.py#L1628) (verified)
  - *To reach the next level:* No capability restriction or approval tied to having read untrusted content.
- **C L1:** Only a hardcoded set of web tool names is fenced; MCP results are not, despite the comment. — [src/praisonai-agents/praisonaiagents/tools/trust.py:49](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/trust.py#L49); searched `rg -n "trust_level|approval|wrap_if_external"` in `src/praisonai-agents/praisonaiagents/mcp/` → 0 hits (MCP client never marks its tools external or approval-required, so MCP tools skip both the gate and the injection fence.) (verified)
  - *To reach the next level:* Tool/MCP results, tool descriptions and other agents' messages are not covered.
- **D L2:** The fence is on by default for listed tools; it cannot be configured away by content, but the developer can drop it. — [src/praisonai-agents/praisonaiagents/agent/tool_execution.py:1628](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/agent/tool_execution.py#L1628) (verified)
  - *To reach the next level:* Disabling is not explicit and warned.
- **B L0:** With ungated developer/MCP tools and unfenced MCP output, a hijack can leak data through any fetch or MCP tool and take irreversible actions (e.g. the README's deploy tool) with no human. — [src/praisonai-agents/praisonaiagents/agent/tool_execution.py:2286-2291](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/agent/tool_execution.py#L2286-L2291); searched `rg -n "trust_level|approval|wrap_if_external"` in `src/praisonai-agents/praisonaiagents/mcp/` → 0 hits (MCP client never marks its tools external or approval-required, so MCP tools skip both the gate and the injection fence.) (verified)
  - *To reach the next level:* No default where exfiltration and irreversible actions both require approval.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.17 (medium)

By default the Agent auto-loads instruction files (AGENTS.md, CLAUDE.md, .cursorrules and others) from the working directory into the system prompt without asking the user to trust the folder. Neither @imports nor project configuration files are integrity-protected. Long-term memory is off by default and its user_id namespace is optional.

- **S L0:** Instruction files load silently as high-priority context, and project configuration is not integrity-protected. — [src/praisonai-agents/praisonaiagents/agent/agent.py:2700](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/agent/agent.py#L2700); [src/praisonai-agents/praisonaiagents/agent/chat_mixin.py:419-422](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/agent/chat_mixin.py#L419-L422) (verified)
  - *To reach the next level:* No workspace-trust decision before project config or instruction files take effect.
- **C L1:** Project single-file plugins have an env-var trust gate; instruction files have none, and config/@import handling is not integrity-protected. — [src/praisonai-agents/praisonaiagents/plugins/manager.py:783](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/plugins/manager.py#L783); [src/praisonai-agents/praisonaiagents/memory/rules_manager.py:140](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/memory/rules_manager.py#L140) (verified)
  - *To reach the next level:* Instruction files, config defaults and retrieval stores are not controlled.
- **D L1:** Memory namespace user_id defaults to None, so isolation depends on the developer passing one. — [src/praisonai-agents/praisonaiagents/config/feature_configs.py:263](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/config/feature_configs.py#L263) (inferred)
  - *To reach the next level:* Per-user namespaces are not enforced by default.
- **B L1:** A poisoned AGENTS.md or config persists in the repo, applies to every session in that directory, and can steer tool use. — [src/praisonai-agents/praisonaiagents/agent/chat_mixin.py:419-422](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/agent/chat_mixin.py#L419-L422) (verified)
  - *To reach the next level:* Poisoned context is neither session-scoped nor easily purged.
- **Cap:** C6-REPOCONFIG — Project configuration is not integrity-protected.

### C7 Third-party extensions — 0.25 (high)

MCP servers are launched from whatever command the developer writes (the README uses `npx -y`, which is unpinned) with no pinning or integrity check, though stdio servers do get a scrubbed environment. Project-local plugin files need an environment variable before they load, but the remaining plugin gates do not cover every path. Plugins run in-process with the agent's full access.

- **S L1:** Developer-chosen sources, unpinned (README launches MCP with npx -y); no hash or signature checks. — [README.md:194](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/README.md#L194); searched `rg -n "sha256|hashlib|integrity"` in `src/praisonai-agents/praisonaiagents/mcp/ src/praisonai-agents/praisonaiagents/plugins/` → 3 hits (Hits are the OAuth PKCE code-challenge digest and a loop-detection plugin docstring; nothing verifies an MCP server or plugin.) (verified)
  - *To reach the next level:* Versions are not pinned or verified.
- **C L1:** Only project single-file plugins have a load gate; MCP servers and entry-point plugins have none. — [src/praisonai-agents/praisonaiagents/plugins/manager.py:783](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/plugins/manager.py#L783); [src/praisonai-agents/praisonaiagents/plugins/discovery.py:58](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/plugins/discovery.py#L58); searched `rg -n "sha256|hashlib|integrity"` in `src/praisonai-agents/praisonaiagents/mcp/ src/praisonai-agents/praisonaiagents/plugins/` → 3 hits (Hits are the OAuth PKCE code-challenge digest and a loop-detection plugin docstring; nothing verifies an MCP server or plugin.) (verified)
  - *To reach the next level:* Most extension types are not verified.
- **D L2:** MCP servers and plugins are added explicitly in code, but plugin enablement through configuration is not consent-gated on every path. — [src/praisonai-agents/praisonaiagents/plugins/discovery.py:58](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/plugins/discovery.py#L58) (verified)
  - *To reach the next level:* Adding an extension doesn't show the exact package, command and permissions.
- **B L0:** Plugins are imported and executed in-process with every credential the agent holds; MCP stdio servers get a scrubbed env. — [src/praisonai-agents/praisonaiagents/plugins/discovery.py:262](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/plugins/discovery.py#L262); [src/praisonai-agents/praisonaiagents/mcp/mcp.py:628](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/mcp/mcp.py#L628) (verified)
  - *To reach the next level:* Extensions are not separated from the agent process and its credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.25 (high)

Keys come from environment variables. Telemetry is opt-in and content-free, and artifact storage and status output redact secrets. But the shell tool expands $VARIABLES in model-chosen arguments and passes the full environment to child processes, so the model can read any key; instruction-file import handling is not confined either.

- **S L1:** Secrets from env vars; redaction on artifacts and status output only. — [src/praisonai-agents/praisonaiagents/config/feature_configs.py:1242](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/config/feature_configs.py#L1242); [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:182](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L182) (verified)
  - *To reach the next level:* No redaction before model-bound messages or a keychain/secret-manager default.
- **C L1:** Artifact store redaction only; subprocess environments, model-bound messages and debug logs are not scrubbed. — [src/praisonai-agents/praisonaiagents/config/feature_configs.py:1242](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/config/feature_configs.py#L1242); [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:195](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L195); [src/praisonai-agents/praisonaiagents/agent/tool_execution.py:986](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/agent/tool_execution.py#L986) (verified)
  - *To reach the next level:* Logs, transcripts and subprocess environments are not all protected.
- **D L2:** Telemetry is opt-in and sends usage metrics only; default logging is reasonable. — [src/praisonai-agents/praisonaiagents/telemetry/telemetry.py:91](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/telemetry/telemetry.py#L91) (verified)
  - *To reach the next level:* Redaction is not always on across paths.
- **B L0:** Long-lived provider and tool keys are reachable by every shell subprocess and by the model via $VAR expansion. — [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:195](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L195); [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:182](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L182) (verified)
  - *To reach the next level:* Keys reachable by the model and subprocesses are long-lived and unscoped.
- **Cap:** none

### C9 Audit & traceability — 0.35 (medium)

Tool calls are logged only at debug level by default. A structured trace emitter records the start and end of each tool call with arguments and timing, but its default sink discards everything, so nothing is kept unless the developer wires one up. Approvals are not part of that trace.

- **S L2:** When enabled, trace events record each tool call's arguments, result and duration in a structured schema. — [src/praisonai-agents/praisonaiagents/agent/tool_execution.py:1118](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/agent/tool_execution.py#L1118); [src/praisonai-agents/praisonaiagents/agent/tool_execution.py:986](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/agent/tool_execution.py#L986) (verified)
  - *To reach the next level:* No actor attribution (approver, requesting principal) or correlation across sub-agents shown.
- **C L2:** The trace hook sits in the shared execute_tool path, so built-in, developer and MCP tool calls are covered. — [src/praisonai-agents/praisonaiagents/agent/tool_execution.py:1118](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/agent/tool_execution.py#L1118) (verified)
  - *To reach the next level:* Approvals and denials are not recorded.
- **D L0:** The default emitters are NoOp and disabled. — [src/praisonai-agents/praisonaiagents/trace/protocol.py:400](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/trace/protocol.py#L400); [src/praisonai-agents/praisonaiagents/trace/context_events.py:57](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/trace/context_events.py#L57) (verified)
  - *To reach the next level:* Not on by default.
- **B L1:** Best-effort: with the default NoOp sink, nothing survives a crash. — [src/praisonai-agents/praisonaiagents/trace/context_events.py:57](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/trace/context_events.py#L57) (inferred)
  - *To reach the next level:* Records are not flushed durably per action.
- **Cap:** G1 — The structured trace is off by default (NoOp sink, enabled=False).

### C10 Limits & kill switch — 0.40 (high)

The agent loop caps iterations at 20 and tool calls per turn at 10 by default, the shell and code tools have per-call timeouts, and sub-agents spawned through the subagent tool are limited to depth 3. There is no default wall-clock limit or spend cap. The shell tool's timeout is a model-chosen argument with no ceiling.

- **S L2:** Iteration cap plus per-execution timeouts enforced in code; wall-clock and USD budget exist but default to None. — [src/praisonai-agents/praisonaiagents/config/feature_configs.py:875](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/config/feature_configs.py#L875); [src/praisonai-agents/praisonaiagents/config/feature_configs.py:890](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/config/feature_configs.py#L890); [src/praisonai-agents/praisonaiagents/config/feature_configs.py:937](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/config/feature_configs.py#L937); [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:107](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L107) (verified)
  - *To reach the next level:* No wall-clock and cost cap by default, and no rate limits on side-effecting tools.
- **C L2:** Top-level loop and tool timeouts; sub-agent depth is capped. — [src/praisonai-agents/praisonaiagents/config/feature_configs.py:899](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/config/feature_configs.py#L899); [src/praisonai-agents/praisonaiagents/tools/subagent_tool.py:72](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/subagent_tool.py#L72) (verified)
  - *To reach the next level:* Sub-agents and spawned processes do not share the parent's budget.
- **D L1:** The model can raise the shell timeout because it is an exposed tool argument with no maximum. — [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:107](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L107); [src/praisonai-agents/praisonaiagents/tools/schema.py:351](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/schema.py#L351) (verified)
  - *To reach the next level:* The model must not be able to raise its own limits.
- **B L1:** No wall-clock or spend ceiling by default, and a model-chosen timeout lets a single command run indefinitely. — [src/praisonai-agents/praisonaiagents/config/feature_configs.py:890](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/config/feature_configs.py#L890); [src/praisonai-agents/praisonaiagents/config/feature_configs.py:937](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/config/feature_configs.py#L937); [src/praisonai-agents/praisonaiagents/tools/shell_tools.py:107](https://github.com/mervinpraison/praisonai/blob/e49b254509c3eaca45a688daccf046a270fd382c/src/praisonai-agents/praisonaiagents/tools/shell_tools.py#L107) (verified)
  - *To reach the next level:* No tight per-run time and cost ceilings by default.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: web search/scrape tools and MCP results (tools/trust.py:37) · [B] sensitive data/systems: full process environment (tools/shell_tools.py:195) · [C] state change / egress: ungated developer and MCP tools (agent/tool_execution.py:2286-2291) · Same default session? Yes

## Highest-impact improvements
1. Harden the approval API so the documented configuration is a human gate. — C2 D L1→L2, +0.050 before caps (Playbook 5)
2. Harden project-scope configuration handling. — C6 S L0→L2, +0.150 before caps (Playbook 2)
3. Remove execution-control parameters (such as env and timeout) from the model-facing tool schemas of execute_code/execute_command. — C4 D L1→L2, +0.050 before caps (Playbook 3)
4. Gate unknown tools (developer and MCP) by default, not just a fixed list of built-in names. — C2 C L1→L2, +0.075 before caps (Playbook 5)
5. Pass a scrubbed environment to execute_command and stop expanding $VARS in model-supplied arguments. — C8 B L0→L1, +0.050 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scope is the core SDK src/praisonai-agents/praisonaiagents; the praisonai wrapper/CLI, bots/gateway, desktop, mobile, Rust and TypeScript packages were not scored.
- The repo was cloned with a blob size filter; files over 1 MB were not examined.
- No text aimed at AI reviewers was found in the files read.
