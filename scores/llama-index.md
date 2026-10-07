# Defense-in-Depth Score: LlamaIndex

**Repo:** https://github.com/run-llama/llama_index · **Commit:** `962940ddc079cc21701d28d1237c84c82a7c5164` (llama-index-core 0.14.25) · **Reviewed:** 2026-10-04
**What it is:** Python framework for building LLM agents and RAG applications over private data, with agent workflows, memory, and hundreds of tool and data integrations.
**Category:** Agent Frameworks
**Scored configuration:** llama-index-core agent workflows (FunctionAgent, ReActAgent, CodeActAgent, AgentWorkflow) with public-constructor defaults; integration packages treated as opt-in extensions.
**Agent surface (default):** code execution opt-in · filesystem write opt-in · network egress opt-in · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents opt-in · external communication opt-in

## Score: 2.2 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | Medium |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C3 | Tool & action scoping | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C4 | Code-execution isolation | L4 | L2 | L0 | L2 | 0.55 | G1 | **0.50** (alt) | Medium |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L1 | L1 | L3 | 0.28 | — | **0.28** | High |
| C7 | Third-party extensions | L0 | L0 | L1 | L0 | 0.05 | C7-RCELOAD | **0.05** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C9 | Audit & traceability | L2 | L2 | L1 | L0 | 0.35 | — | **0.35** | High |
| C10 | Limits & kill switch | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | Medium |


LlamaIndex gives you an agent loop with a default 20-step limit and little else in the way of safety controls: no approval gate, no argument validation, no code-execution isolation, and no protection against prompt injection, which its security policy leaves to the application. Every tool the developer registers runs immediately with the process's full authority. Extension loading in one core component is also not locked down. Treat it as a toolkit: the guardrails have to be built by you.

## Critical gaps
- Core ships no code-execution isolation; the official CodeActAgent example runs model code with in-process exec() and the code-interpreter tool runs it as a host subprocess with the full environment. (ASI05, T11; C4) — [docs/examples/agent/code_act_agent.ipynb:169](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/docs/examples/agent/code_act_agent.ipynb#L169); [llama-index-integrations/tools/llama-index-tools-code-interpreter/llama_index/tools/code_interpreter/base.py:34](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-integrations/tools/llama-index-tools-code-interpreter/llama_index/tools/code_interpreter/base.py#L34)
- A hijacked agent can leak data and take irreversible actions unattended: no taint tracking, no approval gate, and prompt injection is declared out of scope. (ASI01, LLM01, T6; C5) — [SECURITY.md:46](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/SECURITY.md#L46); [llama-index-core/llama_index/core/agent/workflow/base_agent.py:374](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L374)

## Criterion details

### C1 Identity & least privilege — 0.05 (medium)

LlamaIndex has no identity or authorization layer for agents: every registered tool runs with whatever credentials the developer's process and the tool's own client hold, and the default LLM reads the operator's OPENAI_API_KEY from the environment. The one binding primitive, FunctionTool partial_params, is not a strict boundary. The framework neither narrows nor widens the developer's authority.

- **S L0:** No authorization or scoping primitive; the partial_params binding is not a strict boundary. (verified)
  - *To reach the next level:* L1 needs at least a dedicated identity; the framework offers none.
- **C L0:** The agent loop calls tool.acall(**tool_input) directly with no authorization check. — [llama-index-core/llama_index/core/agent/workflow/base_agent.py:374](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L374); [llama-index-core/llama_index/core/agent/workflow/base_agent.py:658](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L658) (verified)
  - *To reach the next level:* No authorization layer exists to cover any tool path.
- **D L0:** Agents run with the process's ambient credentials; nothing narrower ships by default. — [llama-index-core/llama_index/core/agent/workflow/base_agent.py:374](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L374); [llama-index-core/llama_index/core/llms/utils.py:56](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/llms/utils.py#L56) (verified)
  - *To reach the next level:* No narrower default identity exists.
- **B L1:** The framework doesn't constrain what credentials tools hold; typical integration tool specs (Slack, Gmail, Jira, databases) write across several systems. — [llama-index-core/llama_index/core/agent/workflow/base_agent.py:374](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L374); [SECURITY.md:32](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/SECURITY.md#L32) (inferred)
  - *To reach the next level:* Nothing in the framework limits a hijacked agent to a single system or read-only access.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

There is no approval gate in the agent loop. The model's tool calls are dispatched straight to the tool function, including CodeActAgent's code-execution tool. Human-in-the-loop is possible only if the developer writes their own wait-for-event logic inside each tool. Nothing in the framework checkpoints or previews actions.

- **S L0:** No approval mechanism exists; call_tool dispatches every model tool call directly. — searched `rg -n -S 'approv|confirm|HumanResponse|InputRequired|requires_approval'` in `llama-index-core/llama_index/core/agent llama-index-core/llama_index/core/tools` → 0 hits (No approval or human-in-the-loop primitive anywhere in the agent loop or tool layer.); [llama-index-core/llama_index/core/agent/workflow/base_agent.py:658](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L658) (verified)
  - *To reach the next level:* L1 needs at least a blanket human approval primitive.
- **C L0:** No gate exists; every registered tool, including the code-execution tool, executes directly. — [llama-index-core/llama_index/core/agent/workflow/base_agent.py:374](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L374); [llama-index-core/llama_index/core/agent/workflow/codeact_agent.py:100](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/codeact_agent.py#L100) (verified)
  - *To reach the next level:* No gate to cover any path.
- **D L0:** No approval primitive to default on. — searched `rg -n -S 'approv|confirm|HumanResponse|InputRequired|requires_approval'` in `llama-index-core/llama_index/core/agent llama-index-core/llama_index/core/tools` → 0 hits (No approval or human-in-the-loop primitive anywhere in the agent loop or tool layer.) (verified)
  - *To reach the next level:* No approval primitive exists.
- **B L0:** No checkpoint, undo, preview, or dry-run primitive bounds a bad action; tools are arbitrary developer callables. — [llama-index-core/llama_index/core/agent/workflow/base_agent.py:374](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L374); searched `rg -n -S 'checkpoint|rollback|dry_run|undo'` in `llama-index-core/llama_index/core/agent llama-index-core/llama_index/core/tools` → 0 hits (verified)
  - *To reach the next level:* No checkpoint or rollback for tool side effects.
- **Cap:** none

### C3 Tool & action scoping — 0.30 (high)

Tool schemas are generated from function signatures and shown to the model, but nothing validates the model's arguments against them before the call: the kwargs go straight into the Python function. The handoff tool is the one built-in that checks its argument against an allowlist of agents. On the positive side, agents start with no tools at all and receive only what the developer lists.

- **S L1:** No schema validation at call time; only Python's own signature check and the handoff tool's agent allowlist. — [llama-index-core/llama_index/core/agent/workflow/multi_agent_workflow.py:84](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/multi_agent_workflow.py#L84) (verified)
  - *To reach the next level:* L2 needs typed validation of arguments against fn_schema before the call.
- **C L1:** Only the handoff tool validates its input; FunctionTool, the generic wrapper every tool uses, does not. — [llama-index-core/llama_index/core/agent/workflow/multi_agent_workflow.py:78](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/multi_agent_workflow.py#L78) (verified)
  - *To reach the next level:* No shared validation layer for registered tools.
- **D L2:** tools=None by default: an agent gets only the tools the developer lists, and tool_retriever picks only from developer-provided objects. — [llama-index-core/llama_index/core/agent/workflow/base_agent.py:159](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L159); [llama-index-core/llama_index/core/agent/workflow/base_agent.py:650](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L650) (verified)
  - *To reach the next level:* Capped one level above Strength; per-task read-only defaults would also need enforced argument validation.
- **B L1:** Tools are arbitrary developer callables; the official code-interpreter integration runs any Python on the host and core's binary resolver fetches any URL without blocking internal addresses. — [llama-index-integrations/tools/llama-index-tools-code-interpreter/llama_index/tools/code_interpreter/base.py:34](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-integrations/tools/llama-index-tools-code-interpreter/llama_index/tools/code_interpreter/base.py#L34); [llama-index-core/llama_index/core/utils.py:704](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/utils.py#L704) (verified)
  - *To reach the next level:* No framework-level scoping or quantity bounds on tool reach.
- **Cap:** none

### C4 Code-execution isolation — 0.50 (medium)

Core ships no isolation for model-generated code. CodeActAgent requires the developer to pass an execute function, and the official example implements it with in-process exec(), labelled not safe for production. The code-interpreter tool package runs model code with subprocess on the host, with no timeout and the full environment. An Azure Dynamic Sessions tool provides a real remote sandbox, but it is a separate opt-in package.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** No isolation primitive: CodeActAgent wraps a developer-supplied function; the official example uses in-process exec(). — [llama-index-core/llama_index/core/agent/workflow/codeact_agent.py:100](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/codeact_agent.py#L100); [docs/examples/agent/code_act_agent.ipynb:169](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/docs/examples/agent/code_act_agent.ipynb#L169); searched `rg -n -S 'sandbox|isolat'` in `llama-index-core/llama_index/core/agent` → 0 hits (verified)
    - *To reach the next level:* L1 needs at least a filtering layer; the framework provides none.
  - **C L0:** The main exec paths (CodeActAgent's execute tool, code-interpreter tool) run unsandboxed. — [llama-index-core/llama_index/core/agent/workflow/codeact_agent.py:100](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/codeact_agent.py#L100); [llama-index-integrations/tools/llama-index-tools-code-interpreter/llama_index/tools/code_interpreter/base.py:34](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-integrations/tools/llama-index-tools-code-interpreter/llama_index/tools/code_interpreter/base.py#L34) (verified)
    - *To reach the next level:* No path is sandboxed by default.
  - **D L0:** Nothing sandboxed by default; the shipped subprocess tool runs on the host. — [llama-index-integrations/tools/llama-index-tools-code-interpreter/llama_index/tools/code_interpreter/base.py:34](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-integrations/tools/llama-index-tools-code-interpreter/llama_index/tools/code_interpreter/base.py#L34) (verified)
    - *To reach the next level:* No sandbox on by default.
  - **B L0:** Code runs in-process or as a host subprocess that inherits the full environment, including provider API keys. — [llama-index-integrations/tools/llama-index-tools-code-interpreter/llama_index/tools/code_interpreter/base.py:34](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-integrations/tools/llama-index-tools-code-interpreter/llama_index/tools/code_interpreter/base.py#L34); [docs/examples/agent/code_act_agent.ipynb:169](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/docs/examples/agent/code_act_agent.ipynb#L169) (verified)
    - *To reach the next level:* Host filesystem, network, and environment are reachable.
- **opt-in Azure Dynamic Sessions code interpreter (separate package)** (alt; raw 0.55, cap G1 → 0.50) ← counted
  - **S L4:** Code is sent to Azure Dynamic Sessions, a remote sandbox service, instead of running locally. — [llama-index-integrations/tools/llama-index-tools-azure-code-interpreter/llama_index/tools/azure_code_interpreter/base.py:74](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-integrations/tools/llama-index-tools-azure-code-interpreter/llama_index/tools/azure_code_interpreter/base.py#L74); [llama-index-integrations/tools/llama-index-tools-azure-code-interpreter/llama_index/tools/azure_code_interpreter/base.py:162](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-integrations/tools/llama-index-tools-azure-code-interpreter/llama_index/tools/azure_code_interpreter/base.py#L162) (verified)
  - **C L2:** Only code sent to this tool is sandboxed; other registered tools and any CodeAct executor still run in the host process. — [llama-index-integrations/tools/llama-index-tools-azure-code-interpreter/llama_index/tools/azure_code_interpreter/base.py:149](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-integrations/tools/llama-index-tools-azure-code-interpreter/llama_index/tools/azure_code_interpreter/base.py#L149); [llama-index-core/llama_index/core/agent/workflow/base_agent.py:374](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L374) (verified)
    - *To reach the next level:* Registered tools and other exec paths still run on the host.
  - **D L0:** Separate package the developer must install and register. — [llama-index-integrations/tools/llama-index-tools-azure-code-interpreter/llama_index/tools/azure_code_interpreter/base.py:79](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-integrations/tools/llama-index-tools-azure-code-interpreter/llama_index/tools/azure_code_interpreter/base.py#L79) (verified)
    - *To reach the next level:* Off by default.
  - **B L2:** No host environment reaches the sandbox, but the session persists per tool instance and egress limits are not set in code. — [llama-index-integrations/tools/llama-index-tools-azure-code-interpreter/llama_index/tools/azure_code_interpreter/base.py:111](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-integrations/tools/llama-index-tools-azure-code-interpreter/llama_index/tools/azure_code_interpreter/base.py#L111); searched `rg -n -S 'network|egress'` in `llama-index-integrations/tools/llama-index-tools-azure-code-interpreter/llama_index/tools/azure_code_interpreter/base.py` → 0 hits (inferred)
    - *To reach the next level:* No network restriction or per-call ephemeral session.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.00 (high)

Tool results, retrieved documents, and reader output enter the model's context as ordinary tool or observation messages with no provenance or taint, and nothing changes what the agent may do after reading them. The project's SECURITY.md explicitly treats prompt injection as the application's problem. Since the framework provides no approval gate either, a hijacked agent can use any registered egress or write tool unattended.

- **S L0:** No structural limit or detection; tool outputs are appended as role=tool messages. — [llama-index-core/llama_index/core/agent/workflow/function_agent.py:159](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/function_agent.py#L159); searched `rg -n -S 'untrusted|provenance|taint'` in `llama-index-core/llama_index/core/agent llama-index-core/llama_index/core/tools` → 0 hits (No provenance or taint tracking on tool results.); [SECURITY.md:46](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/SECURITY.md#L46) (verified)
  - *To reach the next level:* L1 needs at least detection or spotlighting; L2 needs approval once untrusted content is read.
- **C L0:** No untrusted source is distinguished from any other. — [llama-index-core/llama_index/core/agent/workflow/function_agent.py:159](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/function_agent.py#L159); searched `rg -n -S 'untrusted|provenance|taint'` in `llama-index-core/llama_index/core/agent llama-index-core/llama_index/core/tools` → 0 hits (No provenance or taint tracking on tool results.) (verified)
  - *To reach the next level:* No source is handled.
- **D L0:** No mechanism to default on. — searched `rg -n -S 'untrusted|provenance|taint'` in `llama-index-core/llama_index/core/agent llama-index-core/llama_index/core/tools` → 0 hits (No provenance or taint tracking on tool results.) (verified)
  - *To reach the next level:* No mechanism exists.
- **B L0:** Docs pair RAG over private data with arbitrary tools; nothing in the framework breaks the combination, so leak plus irreversible action is unattended. — [SECURITY.md:46](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/SECURITY.md#L46); [llama-index-core/llama_index/core/agent/workflow/base_agent.py:374](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L374) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions need human approval for L2.
- **Cap:** C5-WORSTCASE — B is L0: the framework neither gates nor taints, so a hijacked agent can leak data and act irreversibly unattended.

### C6 Memory, context & configuration integrity — 0.28 (high)

By default an agent's memory is an in-process chat buffer that disappears with the run, and core loads no workspace files such as .env. When developers enable the richer Memory class with a database and memory blocks, model-extracted facts and retrieved text are written without validation and inserted into the system message by default. Sessions are keyed by an ID in database queries, but nothing prevents poisoned entries from persisting.

- **S L0:** Opt-in memory blocks persist LLM-extracted facts unvalidated and inject them into the system message by default. — [llama-index-core/llama_index/core/memory/memory.py:226](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/memory/memory.py#L226); [llama-index-core/llama_index/core/memory/memory_blocks/fact.py:119](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/memory/memory_blocks/fact.py#L119) (verified)
  - *To reach the next level:* L1 needs at least logging of writes and data-not-instruction presentation.
- **C L1:** No auto-loaded workspace config exists, but no memory store is controlled. — searched `rg -n 'load_dotenv'` in `llama-index-core/llama_index` → 0 hits (Core never auto-loads .env or instruction files.) (verified)
  - *To reach the next level:* Memory blocks and vector memory have no write control.
- **D L1:** Chat stores filter by session key in SQL and a random session ID is generated by default; capped one level above Strength. — [llama-index-core/llama_index/core/storage/chat_store/sql.py:187](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/storage/chat_store/sql.py#L187); [llama-index-core/llama_index/core/memory/memory.py:313](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/memory/memory.py#L313) (verified)
  - *To reach the next level:* Capped by Strength; L2 also needs the isolation to be paired with controlled writes.
- **B L3:** Default memory is an in-memory buffer per run; the SQL store defaults to in-memory SQLite. — [llama-index-core/llama_index/core/agent/workflow/base_agent.py:298](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L298); [llama-index-core/llama_index/core/storage/chat_store/sql.py:31](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/storage/chat_store/sql.py#L31) (verified)
  - *To reach the next level:* L4 needs persistent memory only after human review, with rollback.
- **Cap:** none

### C7 Third-party extensions — 0.05 (high)

Extensions (tool packages, MCP servers, rerank and embedding models) are chosen in code by the developer, so nothing is added silently. But nothing is verified: MCP servers launch whatever command is configured, and model loading in one core component is not locked down. Python tool packages also run in-process with all the agent's credentials. One positive: persisted object mappings load through an allowlisted unpickler.

- **S L0:** Extensions and models load without pinning or verification, and model loading in one core component is not locked down. (verified)
  - *To reach the next level:* L1 needs extension code behind an explicit opt-in.
- **C L0:** Neither MCP servers, tool packages, nor HF models are verified; only local object-mapping pickles are restricted. — [llama-index-integrations/tools/llama-index-tools-mcp/llama_index/tools/mcp/client.py:266](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-integrations/tools/llama-index-tools-mcp/llama_index/tools/mcp/client.py#L266); [llama-index-core/llama_index/core/objects/base_node_mapping.py:36](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/objects/base_node_mapping.py#L36) (verified)
  - *To reach the next level:* No extension type has version or integrity checks.
- **D L1:** Extensions must be named in developer code; no workspace file can add them. Capped one level above Strength. — [llama-index-integrations/tools/llama-index-tools-mcp/llama_index/tools/mcp/client.py:137](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-integrations/tools/llama-index-tools-mcp/llama_index/tools/mcp/client.py#L137) (verified)
  - *To reach the next level:* Capped by Strength.
- **B L0:** Extension code and tool packages run in the agent's own process with all its credentials. — [llama-index-core/llama_index/core/agent/workflow/base_agent.py:374](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L374) (verified)
  - *To reach the next level:* Extensions need at least a separate process.
- **Cap:** C7-RCELOAD — Extension code can load in-process without consent.

### C8 Secrets & sensitive-data protection — 0.30 (high)

Provider keys come from environment variables or plain string fields on the LLM classes. The base LLM class has a to_payload method that keeps API keys out of instrumentation and callback payloads, and core ships no telemetry. There is no redaction of secrets in tool results sent to the model, in logs, or in subprocess environments.

- **S L1:** Keys from env/plain str fields; to_payload masks credentials in observability payloads only. — [llama-index-core/llama_index/core/base/llms/base.py:63](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/base/llms/base.py#L63); [llama-index-integrations/llms/llama-index-llms-openai/llama_index/llms/openai/base.py:230](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-integrations/llms/llama-index-llms-openai/llama_index/llms/openai/base.py#L230) (verified)
  - *To reach the next level:* L2 needs type-level masking (SecretStr) and log filters on main paths.
- **C L1:** Only instrumentation/callback payloads are protected; tool output, logs, and subprocess env aren't. — searched `rg -n -S 'redact'` in `llama-index-core/llama_index` → 0 hits; [llama-index-integrations/tools/llama-index-tools-code-interpreter/llama_index/tools/code_interpreter/base.py:34](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-integrations/tools/llama-index-tools-code-interpreter/llama_index/tools/code_interpreter/base.py#L34) (verified)
  - *To reach the next level:* Logs and transcripts are not filtered.
- **D L2:** No telemetry in core or instrumentation; event and span handlers are Null by default. — searched `rg -n -S 'posthog|sentry_sdk|telemetry'` in `llama-index-core/llama_index llama-index-instrumentation/src` → 0 hits; [llama-index-instrumentation/src/llama_index_instrumentation/__init__.py:15](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-instrumentation/src/llama_index_instrumentation/__init__.py#L15) (verified)
  - *To reach the next level:* No redaction to be always on.
- **B L1:** Long-lived provider keys sit in the process environment, readable by in-process tools and inherited by subprocess tools. — [llama-index-core/llama_index/core/llms/utils.py:56](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/llms/utils.py#L56); [llama-index-integrations/tools/llama-index-tools-code-interpreter/llama_index/tools/code_interpreter/base.py:34](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-integrations/tools/llama-index-tools-code-interpreter/llama_index/tools/code_interpreter/base.py#L34) (verified)
  - *To reach the next level:* Keys are not scoped or short-lived.
- **Cap:** none

### C9 Audit & traceability — 0.35 (high)

Every tool call goes through one workflow step that emits structured ToolCall and ToolCallResult events with the tool name, arguments, call ID, and output, so a developer can capture a full trajectory. But nothing durable records them by default: instrumentation handlers are no-ops, and the run keeps its tool-call list only in memory, returned to the caller. There is no actor or approver attribution.

- **S L2:** Structured events for every tool call (name, kwargs, tool_id, output). — [llama-index-core/llama_index/core/agent/workflow/workflow_events.py:98](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/workflow_events.py#L98); [llama-index-core/llama_index/core/agent/workflow/base_agent.py:668](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L668) (verified)
  - *To reach the next level:* No actor/principal attribution or correlation across nested agents.
- **C L2:** All tool calls, including MCP-adapted and handoff tools, pass through call_tool; agents wrapped as tools run separate streams. — [llama-index-core/llama_index/core/agent/workflow/base_agent.py:658](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L658); [llama-index-core/llama_index/core/agent/workflow/multi_agent_workflow.py:245](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/multi_agent_workflow.py#L245) (verified)
  - *To reach the next level:* Nested agent runs and approvals aren't recorded in one trail.
- **D L1:** On by default but only in memory: tool-call results accumulate in the run context and are returned on AgentOutput.tool_calls; Null handlers mean nothing is written anywhere. — [llama-index-core/llama_index/core/agent/workflow/base_agent.py:692](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L692); [llama-index-instrumentation/src/llama_index_instrumentation/__init__.py:15](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-instrumentation/src/llama_index_instrumentation/__init__.py#L15); searched `rg -n -S 'persist|file_path|FileHandler'` in `llama-index-core/llama_index/core/agent/workflow` → 0 hits (verified)
  - *To reach the next level:* The record lives in the agent's own process; L2 needs it on by default outside the workspace.
- **B L0:** Events are lost at process exit unless captured by the caller. — [llama-index-core/llama_index/core/agent/workflow/base_agent.py:668](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L668) (verified)
  - *To reach the next level:* No durable per-action record.
- **Cap:** none

### C10 Limits & kill switch — 0.38 (medium)

Agent runs stop after 20 iterations by default, enforced in code, and in multi-agent workflows the counter is shared across handoffs. A wall-clock timeout exists in the underlying workflow engine, but agents pass timeout=None by default, and there is no token or cost cap and no per-tool timeout. Sync tools run in a thread pool, so a cancelled or timed-out run can leave a tool still running.

- **S L2:** Iteration cap enforced in code plus an optional workflow timeout enforced by llama-index-workflows. — [llama-index-core/llama_index/core/agent/workflow/base_agent.py:67](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L67); [llama-index-core/llama_index/core/agent/workflow/base_agent.py:541](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L541); [llama-index-core/llama_index/core/agent/workflow/base_agent.py:205](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L205) (inferred)
  - *To reach the next level:* No token/cost cap or rate limits on side-effecting tools.
- **C L1:** Cap covers the top-level loop (shared across AgentWorkflow handoffs); no tool timeouts. — [llama-index-core/llama_index/core/agent/workflow/multi_agent_workflow.py:534](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/multi_agent_workflow.py#L534); searched `rg -n -S 'timeout'` in `llama-index-core/llama_index/core/tools/function_tool.py` → 0 hits (verified)
  - *To reach the next level:* No per-tool timeouts and agent-as-tool runs start a fresh budget.
- **D L2:** max_iterations=20 by default; timeout defaults to None (unlimited wall clock). — [llama-index-core/llama_index/core/agent/workflow/base_agent.py:67](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L67); [llama-index-core/llama_index/core/agent/workflow/base_agent.py:169](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L169) (verified)
  - *To reach the next level:* Model-independent ceiling on delegation via agent-wrapping tools is missing.
- **B L1:** No default wall-clock or cost ceiling, and sync tools in a thread executor keep running after cancellation. — [llama-index-core/llama_index/core/tools/function_tool.py:53](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/tools/function_tool.py#L53); [llama-index-core/llama_index/core/agent/workflow/base_agent.py:169](https://github.com/run-llama/llama_index/blob/962940ddc079cc21701d28d1237c84c82a7c5164/llama-index-core/llama_index/core/agent/workflow/base_agent.py#L169) (verified)
  - *To reach the next level:* Stopping does not cancel in-flight tool calls.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: tool results and retrieved documents enter as role=tool messages (function_agent.py:159) · [B] sensitive data/systems: RAG over private data plus provider keys in the process env (llms/utils.py:56) · [C] state change / egress: any registered tool runs ungated (base_agent.py:374); host code exec via code-interpreter tool (code_interpreter/base.py:34) · Same default session? Yes

## Highest-impact improvements
1. Require an explicit, pinned opt-in for every component that loads extension or model code. — C7 S L0→L1, +0.075 before caps (Playbook 3)
2. Validate model arguments against fn_schema before every tool call. — C3 S L1→L2, +0.075 before caps (Playbook 3, step 2)
3. Add an approval hook in call_tool, consulted for every registered tool, with a requires_approval flag on ToolMetadata. — C2 S L0→L2, +0.150 before caps (Playbook 5, step 1)
4. Set a default wall-clock timeout on agents and per-tool timeouts in FunctionTool. — C10 C L1→L2, +0.075 before caps (Playbook 3, step 3)
5. Ship a default local trajectory log of ToolCall/ToolCallResult events written outside the workspace. — C9 D L0→L2, +0.100 before caps (Playbook 1, step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of commit 962940d only; nothing was executed, installed, or probed.
- Framework scored by llama-index-core defaults: absent primitives score L0 even where a developer could add them; the ~300 integration packages were sampled (MCP, code-interpreter, Azure code interpreter), not reviewed exhaustively.
- The llama-index-workflows engine (timeout, cancellation) is an external dependency not in this repository; its behaviour is inferred.
- Clone used --filter=blob:limit=1m; large blobs (data dumps, notebooks over 1 MB) were not examined.
- No reviewer-directed prompt injection found in the repo.
