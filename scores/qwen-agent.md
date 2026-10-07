# Defense-in-Depth Score: Qwen-Agent

**Repo:** https://github.com/QwenLM/Qwen-Agent · **Commit:** `31a4d36d123688581a9e9744427272b33ce940e0` (0.0.34) · **Reviewed:** 2026-10-04
**What it is:** Alibaba Qwen team's Python framework for building LLM agents with function calling, a Docker code interpreter, RAG, MCP and a Gradio UI.
**Category:** Agent Frameworks
**Scored configuration:** Library defaults: FnCallAgent/Assistant constructed with default arguments, tools as shown in the README quickstart (code_interpreter) where a criterion needs a tool path.
**Agent surface (default):** code execution opt-in · filesystem write opt-in · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents opt-in · external communication opt-in

## Score: 2.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C3 | Tool & action scoping | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | High |
| C4 | Code-execution isolation | L2 | L1 | L2 | L2 | 0.42 | — | **0.42** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L1 | L0 | L0 | L1 | 0.12 | — | **0.12** | High |
| C7 | Third-party extensions | L1 | L0 | L2 | L2 | 0.28 | — | **0.28** | Medium |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C9 | Audit & traceability | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C10 | Limits & kill switch | L2 | L1 | L2 | L2 | 0.42 | — | **0.42** | High |


Qwen-Agent is a capable framework with almost no safety primitives of its own. Every model-chosen tool call runs immediately with no approval step, argument checks are limited to JSON types, and document and web tools fetch any URL or read any local file. The Docker code interpreter is a real but unhardened boundary (root, full network, read-write mount), and an exported math agent runs code directly on the host. Treat any deployment that reads untrusted content as able to leak data and change files unattended.

## Critical gaps
- Untrusted web/document content, any local file readable by the process, and unrestricted egress combine in one unattended session with no gate. (ASI01, LLM01; C5) — [qwen_agent/utils/utils.py:192-200](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/utils/utils.py#L192-L200); [qwen_agent/utils/utils.py:192-194](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/utils/utils.py#L192-L194); [qwen_agent/agent.py:188-192](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/agent.py#L188-L192)

## Criterion details

### C1 Identity & least privilege — 0.05 (high)

Qwen-Agent has no notion of an agent identity or per-request authorization. It runs with the full authority of the Python process: API keys come from environment variables or config, tools build their own HTTP clients, and file-handling tools read any local path the OS user can read. Nothing narrows that ambient authority or checks a request against a policy, so a hijacked agent acts with everything the process holds.

- **S L0:** Credentials are ambient process-wide env vars/config (e.g. DASHSCOPE_API_KEY) and tools act with the OS user's full filesystem and network authority. — [qwen_agent/llm/qwen_dashscope.py:170](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/llm/qwen_dashscope.py#L170); [qwen_agent/utils/utils.py:192-194](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/utils/utils.py#L192-L194) (verified)
  - *To reach the next level:* No dedicated or role-scoped identity for the agent or its tools.
- **C L0:** Each tool calls its own clients directly from the executor with no authorization layer between the model's call and the action. — [qwen_agent/agent.py:188-192](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/agent.py#L188-L192) (verified)
  - *To reach the next level:* No authorization check on any tool path.
- **D L0:** The default install runs every enabled tool as the launching OS user; nothing restricts it without the developer writing code. — [qwen_agent/agent.py:188-192](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/agent.py#L188-L192); [qwen_agent/utils/utils.py:192-194](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/utils/utils.py#L192-L194) (verified)
  - *To reach the next level:* No narrower default role; least privilege requires the developer to build it.
- **B L1:** A hijacked agent holds the OS user's file read access (doc parser copies any local path), LLM/search API keys, and whatever MCP servers the developer attached; the code-interpreter container does not receive host env vars. — [qwen_agent/utils/utils.py:192-194](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/utils/utils.py#L192-L194); [qwen_agent/tools/code_interpreter.py:257-262](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/code_interpreter.py#L257-L262) (verified)
  - *To reach the next level:* Reach is not limited to one system or project.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

There is no human-approval step anywhere in the framework. The agent loop hands every model-generated tool call straight to the tool, including code execution, file writes and deletes through the storage tool, and any MCP tool. Developers would have to wrap tools themselves, and nothing in the framework helps them do so. A wrongly chosen call simply runs.

- **S L0:** No approval mechanism: the executor calls tool.call immediately on every model-emitted function call. — [qwen_agent/agent.py:188-192](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/agent.py#L188-L192); searched `rg -n -i -S 'approv|confirm|human_in_the_loop|require_approval'` in `qwen_agent` → 1 hits (Only hit is the word 'confirm' in a schema-validation error message in tools/base.py; no approval mechanism exists.) (verified)
  - *To reach the next level:* No per-call human approval exists.
- **C L0:** The most powerful tools (code_interpreter, MCP tools, storage) are all reached through the same ungated executor. — [qwen_agent/agent.py:188-192](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/agent.py#L188-L192); [qwen_agent/agents/fncall_agent.py:97-102](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/agents/fncall_agent.py#L97-L102) (verified)
  - *To reach the next level:* Code execution and MCP paths are not gated.
- **D L0:** There is no approval to turn on; the default is fully autonomous. — searched `rg -n -i -S 'approv|confirm|human_in_the_loop|require_approval'` in `qwen_agent` → 1 hits (Only hit is the word 'confirm' in a schema-validation error message in tools/base.py; no approval mechanism exists.) (verified)
  - *To reach the next level:* Approval is not available, let alone on by default.
- **B L0:** Framework tools can delete or overwrite files (storage delete) and MCP tools can take arbitrary external actions, with no checkpoint or undo. — [qwen_agent/tools/mcp_manager.py:366-368](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/mcp_manager.py#L366-L368) (verified)
  - *To reach the next level:* No checkpoints, rollback, or dry-run for consequential actions.
- **Cap:** none

### C3 Tool & action scoping — 0.25 (high)

Tool arguments are checked only against their JSON schema types, and not every tool does even that. The document and web tools accept any URL or local path, follow redirects and fetch internal addresses; the storage tool's path handling is not a strict boundary. Code-interpreter and MCP calls skip schema validation entirely. No tools are on by default, but the README's first example turns on code execution.

- **S L1:** Validation is limited to JSON-schema type checks; URLs and paths pass through unchecked (no host allowlist, no realpath containment). — [qwen_agent/tools/base.py:157-159](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/base.py#L157-L159); [qwen_agent/utils/utils.py:192-200](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/utils/utils.py#L192-L200); searched `rg -n -i -S 'realpath|commonpath|is_relative_to'` in `qwen_agent` → 0 hits (No resolved-path containment check in any tool.) (verified)
  - *To reach the next level:* No allowlist validation: resolved-path containment or URL host/internal-address checks.
- **C L1:** Some tools call _verify_json_format_args, but code_interpreter parses raw code and MCP tools use bare json.loads with no validation. — [qwen_agent/tools/code_interpreter.py:117-121](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/code_interpreter.py#L117-L121); [qwen_agent/tools/mcp_manager.py:273-274](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/mcp_manager.py#L273-L274) (verified)
  - *To reach the next level:* Most built-in tools do not validate values, and extension tools are not wrapped by a shared validation layer.
- **D L2:** FnCallAgent's function_list defaults to None (no tools), but official examples routinely enable code_interpreter, so D is lowered one level from the read-only default. — [qwen_agent/agents/fncall_agent.py:31](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/agents/fncall_agent.py#L31); [README.md:149](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/README.md#L149) (verified)
  - *To reach the next level:* Official examples enable exec tools by default; no per-task tool allowlist.
- **B L0:** Enabled tools are general-purpose: arbitrary Python with network egress, arbitrary URL fetch, arbitrary local file read, and file write/delete. — [qwen_agent/utils/utils.py:192-194](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/utils/utils.py#L192-L194); [qwen_agent/tools/code_interpreter.py:257-262](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/code_interpreter.py#L257-L262) (verified)
  - *To reach the next level:* Tools are not scoped to a workspace or bounded in quantity.
- **Cap:** none

### C4 Code-execution isolation — 0.42 (high)

The built-in code interpreter runs code in a local Docker container and refuses to start if Docker is missing, which is a real boundary. But the container is a stock image running as root with default capabilities, full network egress, and the work directory mounted read-write, and its kernel ports are published on all host interfaces. The framework also exports a math agent whose Python executor runs code directly on the host with exec, and MCP stdio servers run on the host. The mounted host directory can be redirected by an environment variable that a .env file in the working directory can supply.

- **S L2:** Code runs in a stock python:3.12-slim container started with plain docker run: root inside, default capabilities, no seccomp/read-only hardening. — [qwen_agent/tools/resource/code_interpreter_image.dockerfile:1](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/resource/code_interpreter_image.dockerfile#L1); [qwen_agent/tools/code_interpreter.py:257-262](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/code_interpreter.py#L257-L262) (verified)
  - *To reach the next level:* Container is not hardened (non-root, dropped capabilities, no-new-privileges, network off).
- **C L1:** code_interpreter is containerised, but the exported TIRMathAgent executes model code in-process on the host via PythonExecutor, and MCP stdio servers launch on the host. — [qwen_agent/agents/tir_agent.py:65](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/agents/tir_agent.py#L65); [qwen_agent/tools/python_executor.py:49](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/python_executor.py#L49); [qwen_agent/tools/python_executor.py:98](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/python_executor.py#L98); [qwen_agent/tools/mcp_manager.py:366-368](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/mcp_manager.py#L366-L368) (verified)
  - *To reach the next level:* Other model-reachable execution paths (PythonExecutor/TIRMathAgent, MCP stdio servers) run unsandboxed.
- **D L2:** The container is always used and missing Docker fails closed, but the mounted host directory is set by the M6_CODE_INTERPRETER_WORK_DIR env var or cfg without warning. — [qwen_agent/tools/code_interpreter.py:423-424](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/code_interpreter.py#L423-L424); [qwen_agent/tools/code_interpreter.py:96](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/code_interpreter.py#L96); [qwen_agent/tools/mcp_manager.py:47](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/mcp_manager.py#L47) (verified)
  - *To reach the next level:* Sandbox mount policy can be changed silently by an env var (which a CWD .env loaded by MCPManager can supply).
- **B L2:** Inside the container: workspace mounted read-write and unrestricted network; no host secrets are passed (no -e flags), but kernel ports are published on all host interfaces (HMAC-keyed). — [qwen_agent/tools/code_interpreter.py:257-262](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/code_interpreter.py#L257-L262); [qwen_agent/tools/code_interpreter.py:263-264](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/code_interpreter.py#L263-L264) (verified)
  - *To reach the next level:* No network restriction, no CPU/memory/PID limits, container runs as root.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

Nothing limits what a hijacked agent can do. Web pages, documents, and MCP tool results and descriptions enter the conversation as ordinary function messages with no provenance marking, and no capability is disabled or gated once untrusted content has been read. With the tools the README demonstrates (document/web reading plus a networked code interpreter), injected text can make the agent send data out and modify files with no human involved.

- **S L0:** No injection defence of any kind; tool results are appended as FUNCTION messages and drive further tool calls. — [qwen_agent/agents/fncall_agent.py:97-102](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/agents/fncall_agent.py#L97-L102); [qwen_agent/agent.py:188-192](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/agent.py#L188-L192) (verified)
  - *To reach the next level:* No gating of egress or state-changing tools after untrusted content is read.
- **C L0:** Web/doc content, MCP results and MCP tool descriptions are all passed to the model undistinguished. — [qwen_agent/tools/mcp_manager.py:198-203](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/mcp_manager.py#L198-L203); [qwen_agent/agents/fncall_agent.py:97-102](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/agents/fncall_agent.py#L97-L102) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished from principal input.
- **D L0:** No control exists to be on by default. — [qwen_agent/agent.py:188-192](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/agent.py#L188-L192) (verified)
  - *To reach the next level:* No untrusted-input control is shipped.
- **B L0:** Untrusted input (web_extractor/doc_parser fetch), private data (any local file readable by doc parser) and egress (arbitrary URL fetch, networked code container) combine in one unattended session. — [qwen_agent/utils/utils.py:192-200](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/utils/utils.py#L192-L200); [qwen_agent/utils/utils.py:192-194](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/utils/utils.py#L192-L194); [qwen_agent/tools/code_interpreter.py:257-262](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/code_interpreter.py#L257-L262) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions are not gated by a human.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.12 (high)

Qwen-Agent has no long-term agent memory, but it persists parsed documents in a shared workspace cache keyed only by URL, with no per-user separation or expiry, so content fetched once (including attacker-controlled pages) is re-served to later sessions and users. The storage tool, if enabled, lets the model write arbitrary files that later reads return. The MCP manager also calls load_dotenv(), silently loading a .env file from the current working directory into the process environment.

- **S L1:** Cache writes are logged at info level but not validated, and a CWD .env is loaded silently by load_dotenv(). — [qwen_agent/tools/simple_doc_parser.py:425](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/simple_doc_parser.py#L425); [qwen_agent/tools/mcp_manager.py:47](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/mcp_manager.py#L47) (verified)
  - *To reach the next level:* No provenance on cached entries; .env loading is not gated by a trust decision.
- **C L0:** Neither the parse cache, the storage tool, nor the .env load path is controlled. — [qwen_agent/tools/simple_doc_parser.py:425](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/simple_doc_parser.py#L425); [qwen_agent/tools/mcp_manager.py:47](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/mcp_manager.py#L47) (verified)
  - *To reach the next level:* No persistence path is controlled.
- **D L0:** All caches live under one global DEFAULT_WORKSPACE with no per-user or per-session namespace, including behind the multi-user Gradio WebUI. — [qwen_agent/settings.py:27](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/settings.py#L27); searched `rg -n -i -S 'user_id|namespace|tenant'` in `qwen_agent` → 7 hits (Hits are token-truncation variables (last_user_idx) in llm/base.py and CSS comments; no per-user namespace for workspace or caches.) (verified)
  - *To reach the next level:* No per-user/session namespaces.
- **B L1:** Cached parsed content persists across sessions and users and is fed back as tool output that can steer tool use; it reproduces what the source served, and can be purged by deleting the workspace directory. — [qwen_agent/tools/simple_doc_parser.py:425](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/simple_doc_parser.py#L425) (verified)
  - *To reach the next level:* Persisted content is not limited to text output or gated actions.
- **Cap:** none
- **Notes:** C6-REPOCONFIG not applied: load_dotenv() reads the application's own working directory, not a workspace the agent is directed to operate on; the agent can only reach it through an opt-in tool whose path handling is not a strict boundary. load_dotenv's default override=False (python-dotenv behaviour) means it only fills unset variables.

### C7 Third-party extensions — 0.28 (medium)

Third-party code enters mainly through MCP servers that the developer lists in code. Nothing is enabled by default and a workspace file cannot add servers, but the README example launches unpinned packages with 'npx -y', there is no version pinning or integrity check, and server-provided tool descriptions go straight into the model's tool list. Stdio servers run as separate host processes; the MCP SDK passes them a minimal default environment unless the config supplies one.

- **S L1:** MCP servers are developer-chosen commands with no pinning or verification; official examples use npx -y with latest packages. — [qwen_agent/tools/mcp_manager.py:366-368](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/mcp_manager.py#L366-L368); [README.md:199-202](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/README.md#L199-L202) (verified)
  - *To reach the next level:* No version pinning.
- **C L0:** No extension type (stdio MCP, SSE/HTTP MCP) is verified. — searched `rg -n -i -S 'hashlib.sha256\(.*digest|integrity|signature'` in `qwen_agent` → 3 hits (Hits are prompt text about function signatures and the Jupyter HMAC scheme; no extension integrity check.) (verified)
  - *To reach the next level:* No verification for any extension type.
- **D L2:** No MCP server is enabled unless the developer passes an mcpServers dict in code; the framework never shows or confirms what will run. — [qwen_agent/agent.py:218-219](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/agent.py#L218-L219) (verified)
  - *To reach the next level:* Adding a server does not show the exact package/command/permissions to a user for consent.
- **B L2:** Stdio servers are separate host processes; with env unset the MCP Python SDK's stdio_client supplies only a minimal default environment (inferred from SDK behaviour), but there is no sandbox or per-extension scoping. — [qwen_agent/tools/mcp_manager.py:366-368](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/mcp_manager.py#L366-L368) (inferred)
  - *To reach the next level:* No per-extension sandbox or scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.25 (high)

API keys are read from environment variables or config and kept in plain process memory. There is no redaction anywhere: debug logging prints full model inputs, and tool exceptions with full tracebacks are returned to the model. On the positive side there is no telemetry, the default log level is INFO, and the code container is started without host environment variables.

- **S L1:** Secrets come from env vars/cfg; no masking or redaction helper exists. — [qwen_agent/llm/qwen_dashscope.py:170](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/llm/qwen_dashscope.py#L170); searched `rg -n -i -S 'redact|SecretStr|mask_secret'` in `qwen_agent` → 0 hits (No redaction or secret-masking helper anywhere in the package.) (verified)
  - *To reach the next level:* No type-level masking or log filters.
- **C L1:** Only the subprocess path is protected: docker run passes no -e and MCP stdio uses the SDK default env; logs and model-bound messages (tracebacks) are not. — [qwen_agent/tools/code_interpreter.py:257-262](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/code_interpreter.py#L257-L262); [qwen_agent/agent.py:195-203](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/agent.py#L195-L203) (verified)
  - *To reach the next level:* Logs and transcripts are not protected.
- **D L1:** No telemetry and INFO-level logs by default, but QWEN_AGENT_DEBUG=1 enables full unredacted LLM-input logging. — [qwen_agent/log.py:21-26](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/log.py#L21-L26); [qwen_agent/llm/oai.py:189](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/llm/oai.py#L189) (verified)
  - *To reach the next level:* No redaction exists to be on by default.
- **B L1:** Long-lived provider and search API keys live in the process environment, reachable by any in-process code path (e.g. the unsandboxed PythonExecutor). — [qwen_agent/llm/qwen_dashscope.py:170](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/llm/qwen_dashscope.py#L170); [qwen_agent/tools/python_executor.py:49](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/python_executor.py#L49) (verified)
  - *To reach the next level:* Keys are not scoped or short-lived.
- **Cap:** none

### C9 Audit & traceability — 0.25 (high)

The framework does not record tool calls. Calls and results appear in the message list returned to the caller, but nothing is persisted, and the only log lines are scattered INFO messages (downloads, container start) and warnings when a tool raises. There is no actor attribution, correlation across sub-agents, or durable trail.

- **S L1:** Only unstructured stderr log lines for some actions (file downloads, MCP init, tool errors); successful tool calls are not logged. — [qwen_agent/agent.py:202](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/agent.py#L202); [qwen_agent/log.py:21-26](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/log.py#L21-L26) (verified)
  - *To reach the next level:* No structured record of every tool call.
- **C L1:** Logging covers the error branch of the main executor and a few tools; successful calls, MCP calls and sub-agent runs are not recorded. — [qwen_agent/agent.py:202](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/agent.py#L202); [qwen_agent/agent.py:188-192](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/agent.py#L188-L192) (verified)
  - *To reach the next level:* Not all built-in tool calls are recorded.
- **D L1:** Logging goes to a stderr StreamHandler on by default; nothing is stored. — [qwen_agent/log.py:21-26](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/log.py#L21-L26) (verified)
  - *To reach the next level:* Records are not stored outside the agent's reach.
- **B L1:** Best-effort stderr output; nothing durable, so a trajectory cannot be replayed after a crash. — [qwen_agent/log.py:21-26](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/log.py#L21-L26) (verified)
  - *To reach the next level:* Records are not flushed durably per action.
- **Cap:** none

### C10 Limits & kill switch — 0.42 (high)

Each agent run is capped at 20 model calls by default, and code-interpreter executions time out after 30 seconds. There is no token, cost, or wall-clock budget, MCP calls and URL downloads wait indefinitely, and a router or group chat starts each sub-agent with a fresh 20-call budget. On exit the framework stops Docker containers and terminates MCP processes.

- **S L2:** Iteration cap (MAX_LLM_CALL_PER_RUN=20) plus a 30s SIGALRM timeout on code-interpreter execution, both in code. — [qwen_agent/settings.py:24](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/settings.py#L24); [qwen_agent/agents/fncall_agent.py:75-78](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/agents/fncall_agent.py#L75-L78); [qwen_agent/tools/code_interpreter.py:114](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/code_interpreter.py#L114); searched `rg -n -i -S 'max_cost|budget|wall_clock|deadline'` in `qwen_agent` → 0 hits (No cost, token-spend, or wall-clock budget.) (verified)
  - *To reach the next level:* No wall-clock or token/cost cap; no rate limits on side-effecting tools.
- **C L1:** The cap applies per agent loop only; MCP calls block on future.result() with no timeout and sub-agents run with their own fresh budget. — [qwen_agent/tools/mcp_manager.py:278-280](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/mcp_manager.py#L278-L280); [qwen_agent/agents/router.py:85](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/agents/router.py#L85) (verified)
  - *To reach the next level:* Tool timeouts are missing on MCP and HTTP paths.
- **D L2:** Sensible default of 20 calls, operator-configurable via env var; the model cannot raise it. — [qwen_agent/settings.py:24](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/settings.py#L24); [qwen_agent/agents/router.py:85](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/agents/router.py#L85) (verified)
  - *To reach the next level:* Delegation to sub-agents resets the budget.
- **B L2:** Moderate ceilings; on exit containers are stopped and MCP processes terminated, but in-flight tool calls are not cancelled. — [qwen_agent/tools/code_interpreter.py:74-77](https://github.com/QwenLM/Qwen-Agent/blob/31a4d36d123688581a9e9744427272b33ce940e0/qwen_agent/tools/code_interpreter.py#L74-L77) (verified)
  - *To reach the next level:* No cancellation of in-flight calls and no provider-side spend ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: web_extractor/simple_doc_parser fetch arbitrary URLs (qwen_agent/utils/utils.py:200); MCP results (qwen_agent/tools/mcp_manager.py:453) · [B] sensitive data/systems: any local path copied by save_url_to_local_work_dir (qwen_agent/utils/utils.py:194); API keys in env (qwen_agent/llm/qwen_dashscope.py:170) · [C] state change / egress: networked code container with rw mount (qwen_agent/tools/code_interpreter.py:257-262); storage put/delete · Same default session? Yes

## Highest-impact improvements
1. Add an approval hook in Agent._call_tool that shows the exact tool name and arguments and is on by default for code, write and MCP tools. — C2 S L0→L3, +0.225 before caps (Playbook 5)
2. Run the code container with --network none, --user nobody, --cap-drop ALL, --security-opt no-new-privileges, resource limits, and publish kernel ports on 127.0.0.1 only. — C4 B L2→L3, +0.050 before caps (Playbook 3)
3. Resolve and contain paths (realpath + commonpath) in file tools, and block non-HTTP(S) and internal addresses in URL fetches. — C3 S L1→L3, +0.150 before caps (Playbook 3)
4. Log every tool call (name, arguments, status, timestamp, agent name) as a structured record, including MCP calls. — C9 S L1→L2, +0.075 before caps (Playbook 1 step 3)
5. Route PythonExecutor/TIRMathAgent through the Docker sandbox instead of in-process exec. — C4 C L1→L2, +0.075 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- MCP stdio environment scrubbing is inferred from the mcp Python SDK's stdio_client default behaviour, not verified in this repository.
- qwen_server/, browser_qwen/, benchmark/ and examples/ were not audited in depth; the score covers the qwen_agent package.
- No text aimed at AI reviewers was found in the repository.
