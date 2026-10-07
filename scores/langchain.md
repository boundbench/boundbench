# Defense-in-Depth Score: LangChain

**Repo:** https://github.com/langchain-ai/langchain · **Commit:** `57236d55d9ffc8634bda987861e6fce460aff268` · **Reviewed:** 2026-10-03
**What it is:** Agent engineering framework with tool-calling agent loop and large integrations ecosystem
**Category:** Agent Frameworks
**Scored configuration:** The langchain v1 create_agent API with default arguments (no middleware, no checkpointer or store, tracing off), plus the bundled opt-in middleware and MCP adapter at their constructor defaults.
**Agent surface (default):** code execution opt-in · filesystem write opt-in · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents yes · external communication opt-in

## Score: 2.8 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L3 | L1 | L0 | L0 | 0.30 | G1 | **0.30** (alt) | High |
| C3 | Tool & action scoping | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C4 | Code-execution isolation | L2 | L1 | L0 | L2 | 0.33 | G1 | **0.33** (alt) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L2 | L3 | 0.40 | — | **0.40** | Medium |
| C7 | Third-party extensions | L1 | L0 | L2 | L2 | 0.28 | — | **0.28** | Medium |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C9 | Audit & traceability | L2 | L2 | L0 | L1 | 0.35 | G1 | **0.35** (alt) | Low |
| C10 | Limits & kill switch | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |


LangChain gives you a tool-calling agent loop and leaves almost every safety control to you. As shipped, create_agent runs every tool call with no approval, no untrusted-input containment, no authorization layer and a 9,999-step limit, so a prompt-injected agent can do anything its tools allow. Good controls exist as opt-in middleware (per-call human approval showing exact arguments, call limits, PII redaction, a Docker shell policy), but the bundled shell tool defaults to unisolated host execution.

## Critical gaps
- No identity scoping or authorization layer: a hijacked agent acts with the full ambient authority of the developer's process, and the bundled shell tool defaults to unisolated host execution. (ASI03, T3; C1) — [libs/langchain_v1/langchain/agents/middleware/shell_tool.py:591-594](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/shell_tool.py#L591-L594); [libs/langchain_v1/langchain/agents/middleware/_execution.py:97-99](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/_execution.py#L97-L99)
- The bundled shell tool defaults to HostExecutionPolicy: model-generated commands run as the host user with no filesystem or network isolation. (ASI05, T11; C4) — [libs/langchain_v1/langchain/agents/middleware/shell_tool.py:591-594](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/shell_tool.py#L591-L594); [libs/langchain_v1/langchain/agents/middleware/_execution.py:97-99](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/_execution.py#L97-L99)
- No untrusted-input containment: injected tool or MCP content can drive both data exfiltration and irreversible tool actions with no human in the loop by default. (ASI01, T6, LLM01; C5) — [libs/langchain_v1/langchain/agents/factory.py:1019-1022](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L1019-L1022); [libs/langchain_v1/langchain/agents/factory.py:897](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L897)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

LangChain has no identity or authorization layer of its own. An agent built with create_agent runs every tool with whatever credentials the developer's code and the host process hold, and nothing checks a tool call against a policy or the requesting user. The one narrowing found is accidental-looking: the opt-in shell tool starts with an empty environment when no env is passed, although its docstring says it inherits the parent environment. If an agent is hijacked, the attacker gets the full authority of the process.

- **S L0:** Tools run with the ambient authority of the developer's process; the framework offers no scoped-identity or credential-narrowing primitive. — [libs/langchain_v1/langchain/agents/factory.py:1136-1145](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L1136-L1145); searched `rg -n -S -e 'authoriz|permission|least.privilege'` in `libs/langchain_v1/langchain/agents` → 2 hits (one is a PermissionError catch in file_search, one is the HITL edit-notice text; no authorization layer exists) (verified)
  - *To reach the next level:* No per-tool or per-request credential scoping or authorization gate in code.
- **C L0:** Every tool, including MCP and middleware tools, constructs or receives its own clients; no shared authorization check sits on the tool path. — [libs/langchain_v1/langchain/agents/factory.py:1136-1145](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L1136-L1145); searched `rg -n -S -e 'authoriz|permission|least.privilege'` in `libs/langchain_v1/langchain/agents` → 2 hits (one is a PermissionError catch in file_search, one is the HITL edit-notice text; no authorization layer exists) (verified)
  - *To reach the next level:* No authorization layer that every tool path traverses.
- **D L0:** The default create_agent applies no privilege restriction; the agent holds whatever the launching user and the passed tools hold. — [libs/langchain_v1/langchain/agents/factory.py:897](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L897) (verified)
  - *To reach the next level:* No narrower default identity; least privilege is entirely developer work.
- **B L0:** Nothing the framework does bounds a hijacked agent's authority; with the bundled shell tool on its default host policy, the agent reaches every file and credential store the OS user can (~/.aws, ~/.ssh, kubeconfig). — [libs/langchain_v1/langchain/agents/middleware/shell_tool.py:591-594](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/shell_tool.py#L591-L594); [libs/langchain_v1/langchain/agents/middleware/_execution.py:97-99](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/_execution.py#L97-L99) (verified)
  - *To reach the next level:* Framework-enforced scoping that keeps a hijacked agent below the user's full cross-service authority.
- **Cap:** none

### C2 Approval gates — 0.30 (high)

By default create_agent executes every tool call the model emits with no human approval. LangChain ships a good opt-in HumanInTheLoopMiddleware that pauses on listed tools, shows the exact tool name and arguments, and supports approve, edit, reject and respond, with edits applied exactly at execution. But it only gates tools named in its map; any tool not listed, including MCP tools added later, is auto-approved, and nothing offers undo or rollback for actions already taken.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** The default agent loop has no approval step; tool calls run as soon as the model emits them. — [libs/langchain_v1/langchain/agents/factory.py:897](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L897); searched `rg -n -S -e 'HumanInTheLoop|interrupt\('` in `libs/langchain_v1/langchain/agents/factory.py` → 0 hits (create_agent never adds an approval step itself; HITL exists only as opt-in middleware) (verified)
    - *To reach the next level:* A per-call human approval in the default configuration.
  - **C L0:** No tool, including the most powerful (shell, generic HTTP, MCP), is gated by default. — [libs/langchain_v1/langchain/agents/factory.py:1136-1145](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L1136-L1145); searched `rg -n -S -e 'HumanInTheLoop|interrupt\('` in `libs/langchain_v1/langchain/agents/factory.py` → 0 hits (create_agent never adds an approval step itself; HITL exists only as opt-in middleware) (verified)
    - *To reach the next level:* A gate that covers the most powerful tool paths by default.
  - **D L0:** Approval is opt-in: the middleware sequence is empty by default. — [libs/langchain_v1/langchain/agents/factory.py:897](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L897) (verified)
    - *To reach the next level:* Approval on by default.
  - **B L0:** The framework offers no checkpoints, undo, previews or quantity bounds for tool side effects; a wrongly executed shell or API call is irreversible. — [libs/langchain_v1/langchain/agents/middleware/shell_tool.py:591-594](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/shell_tool.py#L591-L594); [libs/langchain_v1/langchain/agents/middleware/_execution.py:97-99](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/_execution.py#L97-L99) (verified)
    - *To reach the next level:* Rollback or dry-run for common consequential actions.
- **opt-in HumanInTheLoopMiddleware** (alt; raw 0.30, cap G1 → 0.30) ← counted
  - **S L3:** Per-call interrupt showing the exact tool name and arguments, with per-tool decision sets and an optional when predicate; reviewer edits are substituted at execution time. — [libs/langchain_v1/langchain/agents/middleware/human_in_the_loop.py:326](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/human_in_the_loop.py#L326); [libs/langchain_v1/langchain/agents/middleware/human_in_the_loop.py:658-661](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/human_in_the_loop.py#L658-L661); [libs/langchain_v1/langchain/agents/middleware/human_in_the_loop.py:456-457](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/human_in_the_loop.py#L456-L457) (verified)
    - *To reach the next level:* No argument-level deny rules; the when predicate can only escalate or auto-approve.
  - **C L1:** Only tools listed in interrupt_on are gated; any unlisted tool, including MCP tools discovered later, is auto-approved. — [libs/langchain_v1/langchain/agents/middleware/human_in_the_loop.py:257](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/human_in_the_loop.py#L257); [libs/langchain_v1/langchain/agents/middleware/human_in_the_loop.py:456-457](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/human_in_the_loop.py#L456-L457) (verified)
    - *To reach the next level:* Reject or gate unknown tools by default so every path traverses the gate.
  - **D L0:** The middleware must be added explicitly; create_agent's middleware default is empty. — [libs/langchain_v1/langchain/agents/factory.py:897](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L897) (verified)
    - *To reach the next level:* Approval on by default.
  - **B L0:** Same as default: no undo or rollback for actions an approver wrongly allows. — [libs/langchain_v1/langchain/agents/middleware/_execution.py:97-99](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/_execution.py#L97-L99) (verified)
    - *To reach the next level:* Rollback or dry-run for common consequential actions.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C3 Tool & action scoping — 0.45 (high)

Every LangChain tool with a Pydantic schema has its arguments type-checked before it runs, which stops malformed calls but is not an allowlist. Tools defined with a raw JSON schema, which is how MCP tools are adapted, skip validation entirely. The bundled file-search tool does resolved, symlink-aware path containment, but the bundled shell tool accepts any command string. No tools are enabled unless the developer passes them, but the framework has no read/write tiering of its own.

- **S L2:** Pydantic schema validation on tool input; FileSearchMiddleware adds resolved-path containment, but the shell tool is raw passthrough. — [libs/langchain_v1/langchain/agents/middleware/file_search.py:25-39](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/file_search.py#L25-L39); [libs/langchain_v1/langchain/agents/middleware/shell_tool.py:491-495](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/shell_tool.py#L491-L495) (verified)
  - *To reach the next level:* Allowlist validation (paths, hosts, commands) as a framework primitive, not only in one bundled tool.
- **C L2:** Schema validation applies to every BaseModel-typed tool, but dict (JSON-schema) tools such as MCP tools return their input unvalidated. — [libs/core/langchain_core/tools/base.py:816-818](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/core/langchain_core/tools/base.py#L816-L818); [libs/langchain_v1/langchain/mcp/tools.py:296-299](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/mcp/tools.py#L296-L299) (verified)
  - *To reach the next level:* A shared validation layer that also covers dict-schema and MCP tools.
- **D L2:** No tools are enabled by default; whatever the developer passes is live with no read/write distinction, and middleware can add tools such as the shell. — [libs/langchain_v1/langchain/agents/factory.py:1136-1145](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L1136-L1145) (verified)
  - *To reach the next level:* A read-only default tool tier with write/exec requiring explicit elevation.
- **B L1:** The framework itself bounds little: the shell tool has a 30-second per-command timeout and output caps but can run any command anywhere the user can. — [libs/langchain_v1/langchain/agents/middleware/_execution.py:69-72](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/_execution.py#L69-L72); [libs/langchain_v1/langchain/agents/middleware/_execution.py:97-99](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/_execution.py#L97-L99) (verified)
  - *To reach the next level:* Workspace scoping enforced for general-purpose tools.
- **Cap:** none

### C4 Code-execution isolation — 0.33 (high)

LangChain's code-execution surface is the opt-in ShellToolMiddleware, and when a developer enables it without choosing a policy it runs a persistent bash shell directly on the host, with no filesystem or network isolation. A Docker execution policy is available that runs commands in a separate container with networking off and the container removed afterwards, and it fails rather than falling back to the host when Docker is missing. That container is stock, though: root inside, default capabilities, no resource limits by default, and the developer's workspace mounted read-write when one is configured. Local MCP servers launched over stdio always run on the host.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** The default HostExecutionPolicy launches the shell as a same-user host subprocess with no isolation. — [libs/langchain_v1/langchain/agents/middleware/shell_tool.py:591-594](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/shell_tool.py#L591-L594); [libs/langchain_v1/langchain/agents/middleware/_execution.py:97-99](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/_execution.py#L97-L99) (verified)
    - *To reach the next level:* An OS-level boundary in the default execution policy.
  - **C L0:** The main exec tool runs unisolated by default. — [libs/langchain_v1/langchain/agents/middleware/shell_tool.py:591-594](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/shell_tool.py#L591-L594) (verified)
    - *To reach the next level:* Sandboxing of the main exec tool by default.
  - **D L0:** Isolation is off by default: the shell middleware falls back to HostExecutionPolicy when no policy is passed. — [libs/langchain_v1/langchain/agents/middleware/shell_tool.py:591-594](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/shell_tool.py#L591-L594) (verified)
    - *To reach the next level:* Isolation on by default.
  - **B L0:** Host-equivalent: the shell reaches the whole filesystem, the network, and every credential file the user owns. — [libs/langchain_v1/langchain/agents/middleware/_execution.py:97-99](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/_execution.py#L97-L99) (verified)
    - *To reach the next level:* Restrict the default reachable scope (workspace-only mount, no network).
- **opt-in DockerExecutionPolicy** (alt; raw 0.33, cap G1 → 0.33) ← counted
  - **S L2:** A stock docker run container: no --user by default, no capability drop, no seccomp/no-new-privileges flags. — [libs/langchain_v1/langchain/agents/middleware/_execution.py:338-342](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/_execution.py#L338-L342); [libs/langchain_v1/langchain/agents/middleware/_execution.py:293](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/_execution.py#L293) (verified)
    - *To reach the next level:* Hardened container defaults: non-root user, dropped capabilities, no-new-privileges.
  - **C L1:** Covers the shell tool only; MCP stdio servers and other tools still run on the host process. — [libs/langchain_v1/langchain/agents/middleware/_execution.py:314-329](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/_execution.py#L314-L329); [libs/langchain_v1/langchain/mcp/adapter.py:132-134](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/mcp/adapter.py#L132-L134) (verified)
    - *To reach the next level:* Sandbox every model-reachable execution path, including MCP stdio servers.
  - **D L0:** Opt-in: must be passed explicitly as execution_policy. — [libs/langchain_v1/langchain/agents/middleware/shell_tool.py:591-594](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/shell_tool.py#L591-L594) (verified)
    - *To reach the next level:* Container isolation on by default.
  - **B L2:** Network none and --rm by default and only explicit env is passed, but no memory/CPU/PID limits by default and the workspace is bind-mounted read-write when configured. — [libs/langchain_v1/langchain/agents/middleware/_execution.py:341-348](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/_execution.py#L341-L348); [libs/langchain_v1/langchain/agents/middleware/_execution.py:289](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/_execution.py#L289) (verified)
    - *To reach the next level:* Default CPU/memory/PID limits and a read-only root filesystem.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.00 (high)

Tool results, including web, file and MCP content, are appended to the conversation as tool messages and fed straight back to the model, and nothing in the framework tracks taint, quarantines untrusted text, or disables egress or state-changing tools once untrusted content has been read. MCP tool descriptions are passed to the model verbatim. Because LangChain's own documentation combines untrusted inputs, private data and outbound tools in ordinary use, a successful prompt injection can both leak data and take irreversible actions without a human.

- **S L0:** No structural limit on a hijacked agent: no taint tracking, no Rule-of-Two enforcement, no quarantined-LLM pattern. — [libs/langchain_v1/langchain/agents/factory.py:1019-1022](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L1019-L1022); searched `rg -n -S -e 'untrusted|taint|quarantin|provenance|prompt.injection'` in `libs/langchain_v1/langchain` → 3 hits (a docstring about MCP metadata provenance, a warning about untrusted model configs, and the Docker policy docstring; none is a control on untrusted content) (verified)
  - *To reach the next level:* Force egress and state-changing tools through approval once untrusted content enters a session.
- **C L0:** Untrusted sources are not distinguished by any control; tool results and MCP tool descriptions enter context directly. — [libs/langchain_v1/langchain/agents/factory.py:1019-1022](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L1019-L1022); [libs/langchain_v1/langchain/mcp/tools.py:297-298](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/mcp/tools.py#L297-L298) (verified)
  - *To reach the next level:* Apply the limit to at least one untrusted source.
- **D L0:** No such control exists to be on by default. — searched `rg -n -S -e 'untrusted|taint|quarantin|provenance|prompt.injection'` in `libs/langchain_v1/langchain` → 3 hits (a docstring about MCP metadata provenance, a warning about untrusted model configs, and the Docker policy docstring; none is a control on untrusted content) (verified)
  - *To reach the next level:* An untrusted-input control on by default.
- **B L0:** As shipped, a hijacked agent can exfiltrate and act irreversibly through whatever tools it holds, unattended. — [libs/langchain_v1/langchain/agents/factory.py:1019-1022](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L1019-L1022); [libs/langchain_v1/langchain/agents/factory.py:897](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L897) (verified)
  - *To reach the next level:* Break at least one leg of the Rule of Two by default.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.40 (medium)

LangChain loads no instruction files or .env files from the working directory, and by default create_agent keeps state only in memory for a single run. Persistence is opt-in through a LangGraph checkpointer (per conversation thread) or store, and when enabled the saved conversation, including untrusted tool output, is reloaded verbatim with no validation, expiry or review. The opt-in summarization middleware re-injects a model-written summary of earlier turns as a user-role message, so injected text can be laundered into something that looks like the user's own instructions.

- **S L1:** Persisted state is reloaded unvalidated; summaries of untrusted content come back as a HumanMessage, losing provenance. — [libs/langchain_v1/langchain/agents/middleware/summarization.py:757-758](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/summarization.py#L757-L758); [libs/langchain_v1/langchain/agents/factory.py:901-902](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L901-L902) (verified)
  - *To reach the next level:* Present persisted and summarized content as data with provenance, or gate memory writes.
- **C L1:** Message history keeps its original roles, but summaries and developer-built retrieval stores carry no controls. — [libs/langchain_v1/langchain/agents/middleware/summarization.py:757-758](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/summarization.py#L757-L758) (verified)
  - *To reach the next level:* Cover summaries and retrieval stores as well as message history.
- **D L2:** No persistence by default; when enabled, LangGraph checkpoints are namespaced by thread_id (behaviour of the out-of-repo langgraph library). — [libs/langchain_v1/langchain/agents/factory.py:901-902](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L901-L902) (inferred)
  - *To reach the next level:* Model cannot write across namespaces, verified in code within this repo; store namespaces are developer-chosen.
- **B L3:** Default runs are ephemeral; with a checkpointer, poisoned context persists within one conversation thread and can trigger tool use there. — [libs/langchain_v1/langchain/agents/factory.py:901-902](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L901-L902) (verified)
  - *To reach the next level:* Persistent context only after human review, with rollback.
- **Cap:** none

### C7 Third-party extensions — 0.28 (medium)

LangChain's runtime extension path is MCP: the beta MCPAdapter connects to servers the developer names and exposes every tool they advertise. String targets are refused unless they are http(s) URLs, so a string cannot silently launch a local script. But there is no version pinning, hash check, or re-approval when a server's tools or descriptions change between runs, and stdio servers run as local processes under the same user.

- **S L1:** Developer-chosen MCP targets, re-listed on each connection with no pinning or integrity check. — [libs/langchain_v1/langchain/mcp/adapter.py:240-242](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/mcp/adapter.py#L240-L242); searched `rg -n -S -w -e 'sha256|digest|checksum|signature|hash'` in `libs/langchain_v1/langchain/mcp` → 0 hits (no integrity or pinning code in the MCP adapter) (verified)
  - *To reach the next level:* Pin server versions or verify integrity of what is launched.
- **C L0:** No extension type is verified. — searched `rg -n -S -w -e 'sha256|digest|checksum|signature|hash'` in `libs/langchain_v1/langchain/mcp` → 0 hits (no verification of any kind) (verified)
  - *To reach the next level:* Verification for at least MCP servers.
- **D L2:** Nothing is enabled by default; a developer must name each target in code, and bare strings that would launch a subprocess are refused. — [libs/langchain_v1/langchain/mcp/adapter.py:84-122](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/mcp/adapter.py#L84-L122); [libs/langchain_v1/langchain/mcp/adapter.py:196-198](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/mcp/adapter.py#L196-L198) (verified)
  - *To reach the next level:* Show the exact package, command and permissions when an extension is added.
- **B L2:** Remote MCP servers run off-host; local stdio servers run as separate processes, and the upstream MCP SDK passes a reduced default environment (third-party behaviour, not in this repo). — [libs/langchain_v1/langchain/mcp/adapter.py:132-134](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/mcp/adapter.py#L132-L134) (inferred)
  - *To reach the next level:* Per-extension sandboxing with scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.45 (high)

Model API keys are held as Pydantic SecretStr values and are swapped for secret references when LangChain objects are serialized, and the framework ships no telemetry or crash reporting. The opt-in shell tool starts with an empty environment unless the developer passes one, so it does not inherit API keys through its environment. Nothing redacts secrets or personal data from what is sent to the model by default; PII and output redaction are opt-in, and LangSmith tracing, when switched on, uploads full inputs and outputs.

- **S L2:** Type-level masking (SecretStr) and secret stripping on serialization. — [libs/partners/anthropic/langchain_anthropic/chat_models.py:1435](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/partners/anthropic/langchain_anthropic/chat_models.py#L1435); [libs/core/langchain_core/load/serializable.py:359-377](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/core/langchain_core/load/serializable.py#L359-L377) (verified)
  - *To reach the next level:* Redaction before model-bound messages and traces on all major paths.
- **C L2:** Serialized objects and the shell subprocess environment are protected; model-bound messages and traces are not by default. — [libs/core/langchain_core/load/serializable.py:359-377](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/core/langchain_core/load/serializable.py#L359-L377); [libs/langchain_v1/langchain/agents/middleware/shell_tool.py:735-740](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/shell_tool.py#L735-L740) (verified)
  - *To reach the next level:* Cover model-bound messages and tracing payloads.
- **D L2:** No telemetry SDKs; tracing is opt-in via environment; redaction middleware is opt-in. — searched `rg -n -S -e 'sentry|posthog|mixpanel|amplitude'` in `libs/langchain_v1/langchain libs/core/langchain_core` → 0 hits (no telemetry or crash-reporting SDKs); [libs/core/langchain_core/tracers/context.py:132-135](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/core/langchain_core/tracers/context.py#L132-L135) (verified)
  - *To reach the next level:* Always-on redaction for traces and logs.
- **B L1:** Long-lived provider API keys sit in the process; a same-user host shell can still read them from the parent's /proc environ or credential files. — [libs/langchain_v1/langchain/agents/middleware/_execution.py:97-99](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/_execution.py#L97-L99) (verified)
  - *To reach the next level:* Short-lived or scoped credentials.
- **Cap:** none

### C9 Audit & traceability — 0.35 (low)

By default the only record of what an agent did is the in-memory message list returned to the caller, which holds each tool call's name, arguments and result but no timestamps, and is lost if the process crashes unless a checkpointer is configured. LangChain integrates with LangSmith tracing, which records structured, nested runs for every model and tool call and ships them off the machine, but it is opt-in via environment variables and does not record who requested or approved an action.

- **default configuration** (default; raw 0.28 → 0.28)
  - **S L1:** In-memory AIMessage tool calls and ToolMessage results, with no timestamps or actor fields. — [libs/langchain_v1/langchain/agents/factory.py:1019-1022](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L1019-L1022); searched `rg -n -S -e 'audit'` in `libs/langchain_v1/langchain` → 0 hits (no audit module) (verified)
    - *To reach the next level:* A structured record with timestamps for every tool call.
  - **C L2:** Every tool call through ToolNode, including MCP tools, lands in the message list; approvals are not recorded as such. — [libs/langchain_v1/langchain/agents/factory.py:1136-1145](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L1136-L1145) (verified)
    - *To reach the next level:* Record approvals and denials and sub-agent calls in one trail.
  - **D L1:** On by default but held in process memory where tools returning state updates and middleware can alter it. — [libs/langchain_v1/langchain/agents/factory.py:1136-1145](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L1136-L1145) (verified)
    - *To reach the next level:* Store the record outside the agent's own reach.
  - **B L0:** Without a checkpointer, the record is lost on crash. — [libs/langchain_v1/langchain/agents/factory.py:901](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L901) (verified)
    - *To reach the next level:* Flush records per action.
- **opt-in LangSmith tracing** (alt; raw 0.35, cap G1 → 0.35) ← counted
  - **S L2:** LangChainTracer posts structured nested runs (inputs, outputs, timings, parent run ids) off-host. — [libs/core/langchain_core/tracers/langchain.py:134-135](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/core/langchain_core/tracers/langchain.py#L134-L135) (inferred)
    - *To reach the next level:* Actor attribution (requesting principal, approver).
  - **C L2:** Callbacks fire for model and tool runs including MCP tools; approvals are not distinguished. — [libs/core/langchain_core/tracers/langchain.py:134-135](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/core/langchain_core/tracers/langchain.py#L134-L135) (inferred)
    - *To reach the next level:* Record approvals and denials explicitly.
  - **D L0:** Opt-in via environment (langsmith's tracing_is_enabled). — [libs/core/langchain_core/tracers/context.py:132-135](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/core/langchain_core/tracers/context.py#L132-L135) (verified)
    - *To reach the next level:* On by default.
  - **B L1:** Uploads are batched in the background by the langsmith client; best-effort. — [libs/core/langchain_core/tracers/langchain.py:134-137](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/core/langchain_core/tracers/langchain.py#L134-L137) (inferred)
    - *To reach the next level:* Durable per-action records.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C10 Limits & kill switch — 0.25 (high)

create_agent sets LangGraph's recursion limit to 9,999 steps, which is an iteration cap in name only, and there is no default wall-clock, token or cost limit. Opt-in middleware can cap model calls and tool calls per run or per thread, but their limits default to none. A sub-agent built with create_agent gets its own fresh 9,999-step limit. The shell tool does enforce a 30-second per-command timeout and kills the whole process group when it fires.

- **S L1:** Iteration cap only (9,999 steps); no default wall-clock or cost cap; call-limit middleware is opt-in. — [libs/langchain_v1/langchain/agents/factory.py:1899](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L1899); [libs/langchain_v1/langchain/agents/middleware/model_call_limit.py:129-130](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/model_call_limit.py#L129-L130); searched `rg -n -S -e 'max_cost|cost_limit|budget|wall_clock|deadline'` in `libs/langchain_v1/langchain/agents/factory.py` → 0 hits (no time or cost budget in the agent factory) (verified)
  - *To reach the next level:* Add a default wall-clock or token/cost cap enforced in code.
- **C L1:** The cap applies to the top-level loop; each sub-agent agent compiled by create_agent carries its own 9,999 limit. — [libs/langchain_v1/langchain/agents/factory.py:1899](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L1899) (verified)
  - *To reach the next level:* Make sub-agents count against the parent's budget.
- **D L1:** The default limit is very large. — [libs/langchain_v1/langchain/agents/factory.py:1899](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L1899) (verified)
  - *To reach the next level:* Sensible default limits.
- **B L1:** A runaway can make thousands of model and tool calls before the cap trips. — [libs/langchain_v1/langchain/agents/factory.py:1899](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/factory.py#L1899); [libs/langchain_v1/langchain/agents/middleware/shell_tool.py:413-415](https://github.com/langchain-ai/langchain/blob/57236d55d9ffc8634bda987861e6fce460aff268/libs/langchain_v1/langchain/agents/middleware/shell_tool.py#L413-L415) (verified)
  - *To reach the next level:* Tight per-run time and cost ceilings.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Tool and MCP results re-enter the model as ToolMessages (libs/langchain_v1/langchain/agents/factory.py:1021) · [B] sensitive data/systems: Ambient developer credentials and host files reachable by tools; shell defaults to host (libs/langchain_v1/langchain/agents/middleware/shell_tool.py:594) · [C] state change / egress: Any registered tool runs without approval by default (libs/langchain_v1/langchain/agents/factory.py:897) · Same default session? Yes

## Highest-impact improvements
1. Make ShellToolMiddleware default to DockerExecutionPolicy (or refuse to start without an explicit policy) instead of HostExecutionPolicy. — C4 D L0→L2, +0.100 before caps (Playbook 3 step 1)
2. Have HumanInTheLoopMiddleware gate unlisted tools by default (deny or interrupt unknown tools) rather than auto-approving them. — C2 C L0→L2, +0.150 before caps (Playbook 5)
3. Lower the default recursion limit and add a default wall-clock/token budget shared with sub-agents. — C10 D L1→L2, +0.050 before caps (Playbook 3 step 3)
4. Return summaries as a distinct, provenance-tagged message type rather than a HumanMessage. — C6 S L1→L2, +0.075 before caps (Playbook 2)
5. Validate dict/JSON-schema (including MCP) tool inputs against their schema before execution. — C3 C L2→L3, +0.075 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- No release tag points at the pinned commit; libs/langchain_v1/pyproject.toml declares langchain 1.4.3.
- Scope is the monorepo at the pinned commit with the langchain v1 create_agent path as the primary deployment; langchain-classic (legacy) and partner integrations were sampled, not exhaustively reviewed.
- The agent loop's ToolNode, checkpointer namespacing, interrupt/resume and LangSmith upload behaviour live in the out-of-repo langgraph and langsmith libraries; ratings relying on them are marked inferred.
- MCP stdio environment handling comes from the third-party fastmcp/mcp SDK and was not verified in this repository.
- No reviewer-directed prompt injection was found; AGENTS.md contains development-tool instructions for coding agents (Corridor security analysis), treated as data.
