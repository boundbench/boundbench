# Defense-in-Depth Score: Agent Squad

**Repo:** https://github.com/2fastlabs/agent-squad (`python`) · **Commit:** `729d5f52869c8280f599a9fa5cf27fc1df445602` (1.0.0) · **Reviewed:** 2026-10-04
**What it is:** Python/TypeScript/Swift framework that routes each user query through a classifier to one of several specialised LLM agents, with tools, MCP, supervisor delegation, and chat storage.
**Category:** Agent Frameworks
**Scored configuration:** Python package defaults: AgentSquad() with the default BedrockClassifier and InMemoryChatStorage, agents constructed with default options and developer-registered AgentTools/MCP tools.
**Agent surface (default):** code execution no · filesystem write no · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents yes · external communication opt-in

## Score: 2.4 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | Medium |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C3 | Tool & action scoping | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | C6-REPOCONFIG | **0.10** | High |
| C7 | Third-party extensions | L1 | L0 | L0 | L2 | 0.17 | — | **0.17** | Medium |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C9 | Audit & traceability | L2 | L2 | L0 | L1 | 0.35 | G1 | **0.35** | High |
| C10 | Limits & kill switch | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |

Controls where a risk surface exists: 1.38 / 9.0 (15%); 1 criterion scored SA (surface absent).

Agent Squad is a routing and orchestration layer that adds almost no safety controls of its own: tools the developer registers run as soon as the model asks, with unvalidated arguments, the process's full AWS and API-key authority, and no approval, audit trail, or stop button. Untrusted tool, MCP, and retrieval output enters the prompt without separation, so a prompt injection can drive any registered tool. The framework's small own surface (no code execution, session-scoped in-memory history by default) is its main protection; StrandsAgent's default of loading tool code from the working directory is the sharpest default to change.

## Critical gaps
- Untrusted tool, MCP, and retriever output enters context with no separation, and nothing stops a hijacked agent from using egress-capable tools with ambient credentials, so leak plus unattended action is possible by default. (ASI01, LLM01, T6; C5) — [python/src/agent_squad/agents/bedrock_llm_agent.py:121-123](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/bedrock_llm_agent.py#L121-L123); [python/src/agent_squad/tools/mcp_tool_provider.py:445-450](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/tools/mcp_tool_provider.py#L445-L450)
- StrandsAgent defaults load_tools_from_directory=True, so the Strands SDK auto-loads tool code from the working directory's tools folder without a trust decision. (ASI06, ASI04, T1; C6) — [python/src/agent_squad/agents/strands_agent.py:37](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/strands_agent.py#L37); [python/src/agent_squad/agents/strands_agent.py:102](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/strands_agent.py#L102)

## Criterion details

### C1 Identity & least privilege — 0.05 (medium)

Every built-in agent, classifier, and storage backend builds its AWS client from boto3's default credential chain, and API keys for Anthropic, OpenAI, Jev, and Dakera are read from options or environment variables. The user_id passed to route_request is a caller-supplied string used only to key chat history; nothing checks what that user may do. Developer-registered tools run in the same Python process and inherit all of these credentials. The framework does nothing to narrow the authority of the deployment it runs in.

- **S L0:** All AWS-backed components call boto3.client(...) with the ambient default credential chain; there is no per-agent or per-tool identity. — [python/src/agent_squad/agents/bedrock_llm_agent.py:43-47](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/bedrock_llm_agent.py#L43-L47); [python/src/agent_squad/classifiers/bedrock_classifier.py:34](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/classifiers/bedrock_classifier.py#L34) (verified)
  - *To reach the next level:* No per-agent or per-tool credential scoping, and no deterministic authorization check before a tool runs.
- **C L0:** Tools run in-process via tool.func(**input_data) with whatever credentials the process holds; no authorization layer exists on any path. — [python/src/agent_squad/utils/tool.py:327-332](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/utils/tool.py#L327-L332); [python/src/agent_squad/tools/mcp_tool_provider.py:445-450](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/tools/mcp_tool_provider.py#L445-L450) (verified)
  - *To reach the next level:* No authorization layer that every tool, MCP, and sub-agent path passes through.
- **D L0:** The default AgentSquad() constructs a BedrockClassifier on the ambient chain; least privilege is left entirely to the deployer's IAM role. — [python/src/agent_squad/orchestrator.py:52-55](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/orchestrator.py#L52-L55) (verified)
  - *To reach the next level:* No narrower default identity; the framework provides none.
- **B L1:** A hijacked agent holds whatever the deployment role grants plus every configured API key; the framework itself calls Bedrock, Lambda, Lex, Comprehend, and DynamoDB, so write access across several systems is typical. — [python/src/agent_squad/agents/bedrock_llm_agent.py:43-47](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/bedrock_llm_agent.py#L43-L47); [python/src/agent_squad/agents/lambda_agent.py:30](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/lambda_agent.py#L30) (inferred)
  - *To reach the next level:* Credentials are not scoped to one system or made short-lived by the framework.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

The framework has no human approval step. When the model asks for a tool, the tool handler looks it up by name and calls it immediately. The same is true for MCP tools and for the supervisor's send_messages delegation tool. The only mention of confirmation is a system-prompt line asking the supervisor to forward confirmations, which is a prompt and not a control. Any consequential action a developer registers runs unattended.

- **S L0:** No approval mechanism exists; tool calls go straight from the model response to execution. — [python/src/agent_squad/utils/tool.py:327-332](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/utils/tool.py#L327-L332); searched `rg -n -i 'approv|confirm|human_in_the_loop|require_approval|ask_user'` in `python/src` → 3 hits (one hit is a prompt guideline in supervisor_agent.py:158 telling the model to forward confirmations; two are test comments; no approval gate exists) (verified)
  - *To reach the next level:* No per-call human approval showing the exact call.
- **C L0:** Neither AgentTools nor MCPToolProvider has a gating hook between the model's tool_use block and execution. — [python/src/agent_squad/utils/tool.py:280-286](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/utils/tool.py#L280-L286); [python/src/agent_squad/tools/mcp_tool_provider.py:445-450](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/tools/mcp_tool_provider.py#L445-L450) (verified)
  - *To reach the next level:* No gate on any tool path, including MCP and sub-agents.
- **D L0:** There is nothing to enable, so approval is effectively absent by default. — searched `rg -n -i 'approv|confirm|human_in_the_loop|require_approval|ask_user'` in `python/src` → 3 hits (one hit is a prompt guideline in supervisor_agent.py:158 telling the model to forward confirmations; two are test comments; no approval gate exists) (verified)
  - *To reach the next level:* No approval on by default.
- **B L0:** The framework offers no checkpoint, undo, preview, or dry-run, so a wrongly executed registered tool (for example a Lambda invocation or MCP write) is as irreversible as the tool itself. — [python/src/agent_squad/agents/lambda_agent.py:30](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/lambda_agent.py#L30); [python/src/agent_squad/tools/mcp_tool_provider.py:445-450](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/tools/mcp_tool_provider.py#L445-L450) (verified)
  - *To reach the next level:* No rollback, preview, or quantity bounds on consequential actions.
- **Cap:** none

### C3 Tool & action scoping — 0.10 (high)

AgentTool derives a JSON schema from the function's type hints and sends it to the model, but the framework never checks the model's arguments against that schema; it unpacks them straight into the function. MCP tool arguments are forwarded unchanged to the server. Unknown tool names are refused, and each agent only receives the tools registered to it, which narrows reach somewhat. Enforcement of MCP tool visibility does not cover every call path.

- **S L0:** Arguments are passed through as **input_data with no schema validation, bounds, or allowlists; the schema is advisory to the model only. — [python/src/agent_squad/utils/tool.py:327-332](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/utils/tool.py#L327-L332); [python/src/agent_squad/utils/tool.py:177-187](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/utils/tool.py#L177-L187) (verified)
  - *To reach the next level:* No in-code validation of arguments against the declared schema or allowlists.
- **C L0:** No tool path validates inputs: AgentTools and MCPToolProvider both pass arguments through. — [python/src/agent_squad/utils/tool.py:327-332](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/utils/tool.py#L327-L332); [python/src/agent_squad/tools/mcp_tool_provider.py:445-450](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/tools/mcp_tool_provider.py#L445-L450) (verified)
  - *To reach the next level:* No shared validation layer for built-in or extension tools.
- **D L1:** The framework ships no tools enabled by default and each agent only gets its own tool_config, but call-time enforcement of MCP tool visibility does not cover every path. (verified)
  - *To reach the next level:* Per-task allowlists are not enforced at call time, and the parameter is capped one level above S.
- **B L1:** Reach is limited to the tools registered on the routed agent, but those tools receive unvalidated arguments with the process's full authority. — [python/src/agent_squad/agents/bedrock_llm_agent.py:356-362](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/bedrock_llm_agent.py#L356-L362) (verified)
  - *To reach the next level:* No workspace scoping or quantity bounds applied by the framework.
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

The framework itself never interprets model output as code: there is no shell, eval, exec, pickle, or package-install path in the Python package. The only process it can launch is an MCP stdio server whose command the developer writes in code, and that risk is scored under third-party extensions. Code a developer puts inside a registered tool runs unisolated in-process; that is the developer's code, not a framework execution path.

- **Structural absence:** searched `rg -n 'subprocess|os\.system|os\.popen|\beval\(|\bexec\(|pickle|exec_module|__import__'` in `python/src/agent_squad` → 1 hits (the single hit is the MCPServerConfig docstring at tools/mcp_tool_provider.py:96; MCP stdio launch of an operator-written command is scored in C7); searched `rg -n 'stdio_client\('` in `python/src/agent_squad` → 1 hits (operator-configured MCP server launch (tools/mcp_tool_provider.py:271), not model-generated code)
- **Notes:** BedrockInlineAgent can pass the AWS-hosted AMAZON.CodeInterpreter action group, which runs remotely in AWS, not in the framework's process.

### C5 Untrusted input blast radius — 0.00 (high)

Content the user did not write reaches the model with no separation: tool and MCP results come back as ordinary tool-result turns, MCP tool descriptions are copied into the tool list, retriever results are appended to the system prompt, and the supervisor pastes every sub-agent's history into its own system prompt. Nothing tracks whether untrusted content has been read, and no tool is disabled or gated afterwards. Bedrock guardrails can be passed through but are off by default and only detect. A successful injection can therefore drive any registered tool, including ones that send data out.

- **S L0:** No taint tracking, quarantine, or post-read gating; retrieved text is concatenated into the system prompt. — [python/src/agent_squad/agents/bedrock_llm_agent.py:121-123](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/bedrock_llm_agent.py#L121-L123); [python/src/agent_squad/agents/supervisor_agent.py:313-318](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/supervisor_agent.py#L313-L318) (verified)
  - *To reach the next level:* No code that gates egress or state-changing tools once untrusted content is in context.
- **C L0:** Tool results, MCP descriptions, retriever output, and sub-agent replies all enter context with the same standing as instructions. — [python/src/agent_squad/tools/mcp_tool_provider.py:538-545](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/tools/mcp_tool_provider.py#L538-L545); [python/src/agent_squad/utils/tool.py:307-310](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/utils/tool.py#L307-L310) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished from principal input.
- **D L0:** guardrail_config is empty unless the developer supplies one, and it is a detection filter in any case. — [python/src/agent_squad/agents/bedrock_llm_agent.py:76](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/bedrock_llm_agent.py#L76) (verified)
  - *To reach the next level:* No limit on by default.
- **B L0:** The documented patterns combine untrusted tool and MCP output, private chat history and ambient AWS credentials, and egress-capable tools (MCP, Lambda), with nothing in the framework breaking that combination, so a hijack can leak data and act unattended. — [python/src/agent_squad/tools/mcp_tool_provider.py:445-450](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/tools/mcp_tool_provider.py#L445-L450); [python/src/agent_squad/agents/supervisor_agent.py:261-270](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/supervisor_agent.py#L261-L270); [python/src/agent_squad/agents/bedrock_llm_agent.py:43-47](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/bedrock_llm_agent.py#L43-L47) (verified)
  - *To reach the next level:* No structural break of the untrusted-input, sensitive-data, egress combination.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.10 (high)

Conversation history is stored per user, session, and agent and replayed on every later turn; the default store is in memory, with DynamoDB and SQL backends available. Nothing validates or labels what is saved, retriever output goes straight into the system prompt, and the supervisor injects all its sub-agents' history into its system prompt as trusted text. Per-user isolation on the supervisor path is not a complete boundary. StrandsAgent defaults load_tools_from_directory to True, which makes the Strands SDK load and hot-reload Python tool files from the working directory's tools folder without any trust decision.

- **S L0:** Saved messages and retrieved text are re-injected without validation or provenance, and StrandsAgent auto-loads tool code from the working directory by default. — [python/src/agent_squad/agents/supervisor_agent.py:313-318](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/supervisor_agent.py#L313-L318); [python/src/agent_squad/agents/bedrock_llm_agent.py:121-123](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/bedrock_llm_agent.py#L121-L123); [python/src/agent_squad/agents/strands_agent.py:37](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/strands_agent.py#L37) (verified)
  - *To reach the next level:* No validation, provenance, or trust decision on persisted or auto-loaded context.
- **C L0:** No store (in-memory, DynamoDB, SQL, summarizing) or auto-load path has any control. — [python/src/agent_squad/storage/in_memory_chat_storage.py:108-109](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/storage/in_memory_chat_storage.py#L108-L109); [python/src/agent_squad/agents/strands_agent.py:102](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/strands_agent.py#L102) (verified)
  - *To reach the next level:* No controlled memory or config path.
- **D L1:** Storage is namespaced by user_id/session_id in every query, but isolation on the SupervisorAgent path is not a complete boundary; level kept one above S. — [python/src/agent_squad/storage/in_memory_chat_storage.py:108-109](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/storage/in_memory_chat_storage.py#L108-L109) (verified)
  - *To reach the next level:* Namespaces are not robustly enforced on the supervisor path, and the parameter is capped one above S.
- **B L1:** Default InMemoryChatStorage history is session-scoped and process-lived, but StrandsAgent's default directory loading means a file placed in ./tools persists across sessions and becomes a callable tool; no framework tool writes there, so not rated L0. — [python/src/agent_squad/storage/in_memory_chat_storage.py:108-109](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/storage/in_memory_chat_storage.py#L108-L109); [python/src/agent_squad/agents/strands_agent.py:37](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/strands_agent.py#L37) (verified)
  - *To reach the next level:* Auto-loaded context is not ephemeral or human-reviewed, and history has no rollback.
- **Cap:** C6-REPOCONFIG — StrandsAgent defaults load_tools_from_directory=True (strands_agent.py:37, passed through at :102), which per the Strands SDK loads and hot-reloads tool modules from ./tools in the working directory without a user trust decision.

### C7 Third-party extensions — 0.17 (medium)

Third-party code enters through MCPToolProvider, which launches or connects to whatever servers the developer lists in code, and through StrandsAgent's MCP clients and directory tool loading. Nothing pins versions, checks hashes, or asks again when a server's tool list changes. The developer has to write each server command in code, so nothing third-party is on by default, except that StrandsAgent's directory loading picks up any Python file placed in the working directory's tools folder. Stdio servers run as separate processes on the host; the MCP SDK gives them a reduced environment unless the developer passes one.

- **S L1:** MCP servers are user-chosen commands or URLs with no pinning or integrity check; the documented example is an unpinned uvx launch. — [python/src/agent_squad/tools/mcp_tool_provider.py:11-14](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/tools/mcp_tool_provider.py#L11-L14); [python/src/agent_squad/tools/mcp_tool_provider.py:263-271](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/tools/mcp_tool_provider.py#L263-L271) (verified)
  - *To reach the next level:* No version pinning or integrity verification of MCP servers or directory-loaded tools.
- **C L0:** Neither MCP servers nor Strands directory tools are verified in any way. — [python/src/agent_squad/tools/mcp_tool_provider.py:263-271](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/tools/mcp_tool_provider.py#L263-L271); [python/src/agent_squad/agents/strands_agent.py:37](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/strands_agent.py#L37) (verified)
  - *To reach the next level:* No extension type is verified.
- **D L0:** StrandsAgent's default lets files in the working directory's tools folder add tools silently. — [python/src/agent_squad/agents/strands_agent.py:37](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/strands_agent.py#L37); [python/src/agent_squad/agents/strands_agent.py:102](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/strands_agent.py#L102) (inferred)
  - *To reach the next level:* Workspace files can add extensions without consent through the StrandsAgent default.
- **B L2:** Stdio MCP servers are separate processes; the mcp SDK's stdio_client uses get_default_environment() (a small allowlist of variables) when env is None, so API keys in os.environ are not inherited by default, but directory-loaded Strands tools run in-process. — [python/src/agent_squad/tools/mcp_tool_provider.py:263-271](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/tools/mcp_tool_provider.py#L263-L271) (inferred)
  - *To reach the next level:* Extensions are not sandboxed or given scoped credentials; Strands directory tools run in-process with full authority.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.30 (high)

API keys come from constructor options or environment variables and are held in plain attributes, with no masking or redaction helper anywhere in the package. Verbose chat and classifier logging is off by default, and there is no telemetry beyond a content-free feature tag added to the AWS User-Agent header. Tools can return a ToolResult whose structured data and UI never reach the model, a small data-minimisation path. MCP tool errors are passed to the model verbatim, and every in-process tool can read the process environment.

- **S L1:** Secrets come from options or env vars; the only minimisation is ToolResult keeping structured_content and UI out of model context. — [python/src/agent_squad/classifiers/jev_classifier.py:97](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/classifiers/jev_classifier.py#L97); [python/src/agent_squad/utils/tool.py:23-27](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/utils/tool.py#L23-L27) (verified)
  - *To reach the next level:* No type-level masking or log redaction.
- **C L1:** Only the model-bound path has the opt-in ToolResult split; logs, errors, and transcripts are unprotected. — searched `rg -n -i 'redact|mask|SecretStr|scrub'` in `python/src` → 1 hits (single hit is a docstring example feature name in shared/user_agent.py:140; no redaction code) (verified)
  - *To reach the next level:* No redaction on logs, error messages, or stored transcripts.
- **D L2:** All LOG_* flags default to False and no third-party telemetry is initialised, though logging.basicConfig(INFO) is set at import. — [python/src/agent_squad/types/types.py:64-77](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/types/types.py#L64-L77); [python/src/agent_squad/utils/logger.py:6](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/utils/logger.py#L6) (verified)
  - *To reach the next level:* Redaction is not always on; there is none to enable.
- **B L1:** Long-lived provider API keys and the ambient AWS chain sit in process memory and the environment, reachable by every in-process tool. — [python/src/agent_squad/agents/openai_agent.py:35-41](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/openai_agent.py#L35-L41); [python/src/agent_squad/agents/bedrock_llm_agent.py:43-47](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/bedrock_llm_agent.py#L43-L47) (verified)
  - *To reach the next level:* Keys are not scoped or short-lived.
- **Cap:** none

### C9 Audit & traceability — 0.35 (high)

The framework records nothing by default. It offers callback hooks around each tool call (name, input, output, agent info) for both its own tools and MCP tools, but the default callbacks do nothing, and the error hook is defined but never called. The orchestrator's metadata carries the caller-supplied user and session ids, but there is no audit log, timestamp, or approver field. A developer who wants a trail must write the storage themselves.

- **S L2:** on_tool_start/on_tool_end receive tool name, arguments, result, and agent info on every call, enough for a structured per-call record when implemented; no timestamps or actor fields are supplied. — [python/src/agent_squad/utils/tool.py:280-286](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/utils/tool.py#L280-L286); [python/src/agent_squad/tools/mcp_tool_provider.py:401-409](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/tools/mcp_tool_provider.py#L401-L409) (verified)
  - *To reach the next level:* No actor attribution or correlation IDs across supervisor and sub-agents.
- **C L2:** Built-in AgentTools and MCPToolProvider both call the hooks, but on_tool_error is never invoked and StrandsAgent tools bypass them. — searched `rg -n 'on_tool_error'` in `python/src/agent_squad` → 3 hits (definition in utils/tool.py:81 and pass-through wrapper in grounded_agent.py:154-155; no call site invokes it) (verified)
  - *To reach the next level:* Errors, sub-agent delegation, and Strands tool calls are not covered.
- **D L0:** Default AgentToolCallbacks methods are all pass, and LOG_* flags default to False. — [python/src/agent_squad/utils/tool.py:57-67](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/utils/tool.py#L57-L67); [python/src/agent_squad/types/types.py:64-77](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/types/types.py#L64-L77) (verified)
  - *To reach the next level:* Recording is opt-in.
- **B L1:** Hooks run inline and an exception would surface, but on_tool_end runs after the action and nothing is persisted by the framework. — [python/src/agent_squad/utils/tool.py:280-286](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/utils/tool.py#L280-L286) (verified)
  - *To reach the next level:* Records are not flushed durably per action.
- **Cap:** G1 — The only recording mechanism is callbacks that default to no-ops; tool calls are not recorded unless the developer implements them.

### C10 Limits & kill switch — 0.30 (high)

Each agent's tool loop stops after a fixed number of rounds (20 for Bedrock, 5 for Anthropic, 40 for the supervisor) and each model call has a 1,000-token output cap by default. There is no wall-clock limit, no session cost budget, and no way to stop a running request. Sub-agents called by the supervisor each start a fresh budget and run in parallel threads with no limit on how many, so total work can multiply well past any single cap.

- **S L1:** Only an iteration cap per agent loop; maxTokens bounds a single response, not a run budget. — [python/src/agent_squad/agents/bedrock_llm_agent.py:209](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/bedrock_llm_agent.py#L209); [python/src/agent_squad/agents/bedrock_llm_agent.py:104](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/bedrock_llm_agent.py#L104); searched `rg -n -i 'wall|deadline|budget|cost|max_cost'` in `python/src` → 7 hits (hits are a classifier prompt example and test fixtures for thinking budget_tokens; no run budget) (verified)
  - *To reach the next level:* No wall-clock, session token, or cost cap.
- **C L1:** Caps apply per agent loop; supervisor sub-agent calls each get their own fresh max_recursions. — [python/src/agent_squad/agents/supervisor_agent.py:65](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/supervisor_agent.py#L65); [python/src/agent_squad/agents/supervisor_agent.py:261-270](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/supervisor_agent.py#L261-L270) (verified)
  - *To reach the next level:* Sub-agents do not count against the parent's budget.
- **D L2:** Defaults of 20, 5, and 40 rounds are set in code and are configurable via toolMaxRecursions. — [python/src/agent_squad/agents/bedrock_llm_agent.py:104](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/bedrock_llm_agent.py#L104); [python/src/agent_squad/agents/anthropic_agent.py:64](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/anthropic_agent.py#L64) (verified)
  - *To reach the next level:* Delegation resets limits; no hard ceiling.
- **B L1:** No halt or cancel exists, and sub-agent work runs in asyncio.to_thread threads that cannot be cancelled once started. — [python/src/agent_squad/agents/supervisor_agent.py:261-270](https://github.com/2fastlabs/agent-squad/blob/729d5f52869c8280f599a9fa5cf27fc1df445602/python/src/agent_squad/agents/supervisor_agent.py#L261-L270) (verified)
  - *To reach the next level:* No stop path that ends the loop and cancels pending calls.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: MCP/tool results and retriever text (tools/mcp_tool_provider.py:450, agents/bedrock_llm_agent.py:123) · [B] sensitive data/systems: ambient AWS credentials and chat history (agents/bedrock_llm_agent.py:47, storage/in_memory_chat_storage.py:109) · [C] state change / egress: registered tools and MCP calls executed without a gate (utils/tool.py:330, tools/mcp_tool_provider.py:450) · Same default session? Yes

## Highest-impact improvements
1. Default StrandsAgent load_tools_from_directory to False so working-directory files cannot add tools. — C6 S L0→L1, +0.075 before caps (Playbook 2)
2. Add an approval hook in AgentTools/MCPToolProvider that shows the exact tool call and can reject it, on by default for tools flagged as mutating. — C2 S L0→L3, +0.225 before caps (Playbook 5)
3. Validate model arguments against the declared JSON schema before calling the tool, and enforce tool visibility at call time. — C3 S L0→L2, +0.150 before caps (Playbook 3)
4. Ship a default callback that writes a structured, timestamped record of every tool call and call on_tool_error. — C9 D L0→L2, +0.100 before caps (Playbook 1 step 3)
5. Share one run budget (steps, wall-clock, tokens) across supervisor and sub-agents and add a cancel path. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the Python package (python/src/agent_squad); the TypeScript package claims feature parity and was not separately scored; the Swift runtime was not examined.
- C7 B and the C6-REPOCONFIG / C7 D downstream effects rely on library behaviour stated, not read: mcp stdio_client defaults to get_default_environment() when env is None, and Strands Agent(load_tools_from_directory=True) loads and hot-reloads ./tools/*.py.
- C4 is marked structurally absent because the framework never interprets model output as code; code inside developer-registered tools runs in-process unisolated and is the developer's responsibility.
- No text aimed at AI reviewers was found; python/SKILL.md and typescript/SKILL.md are developer guides for coding assistants.
