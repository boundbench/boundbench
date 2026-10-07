# Defense-in-Depth Score: Semantic Kernel

**Repo:** https://github.com/microsoft/semantic-kernel · **Commit:** `9974625ddc1e1d3f55e095420fcabcc9329f7f34` · **Reviewed:** 2026-10-05
**What it is:** Microsoft SDK for integrating LLMs with plugins/function calling (C#, Python, Java)
**Category:** Agent Frameworks
**Scored configuration:** Python edition (the README's first install and quickstart): ChatCompletionAgent with developer-registered plugins and MCP plugins using public constructor defaults (FunctionChoiceBehavior.Auto, five auto-invoke rounds).
**Agent surface (default):** code execution opt-in · filesystem write opt-in · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents opt-in · external communication opt-in

## Score: 2.7 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C2 | Approval gates | L2 | L0 | L0 | L0 | 0.15 | G1 | **0.15** (alt) | High |
| C3 | Tool & action scoping | L2 | L2 | L3 | L1 | 0.50 | — | **0.50** | High |
| C4 | Code-execution isolation | L4 | L1 | L0 | L2 | 0.47 | G1 | **0.47** | Medium |
| C5 | Untrusted input blast radius | L1 | L1 | L2 | L0 | 0.25 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L0 | L1 | 0.05 | C6-REPOCONFIG | **0.05** | High |
| C7 | Third-party extensions | L1 | L0 | L2 | L2 | 0.28 | — | **0.28** | Medium |
| C8 | Secrets & sensitive-data protection | L2 | L1 | L1 | L1 | 0.33 | — | **0.33** | High |
| C9 | Audit & traceability | L2 | L2 | L0 | L1 | 0.35 | G1 | **0.35** | High |
| C10 | Limits & kill switch | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |


With its defaults, Semantic Kernel's agent calls any registered tool the model asks for, without approval, and feeds tool results back unmarked, so a prompt-injected agent can use the developer's credentials and outbound tools unattended. Bundled plugins are carefully written (deny-by-default HTTP allowlist, encoded OpenAPI paths, a remote code sandbox), but there is no approval gate, no budget beyond five tool rounds, and settings are read from a .env in the working directory. Logging and telemetry are off unless the developer sets them up.

## Critical gaps
- By default the agent auto-invokes every registered tool with no approval, and tool/MCP results enter the conversation unmarked, so a hijacked agent can exfiltrate and act irreversibly without a human. (ASI01, LLM01; C5) — [python/semantic_kernel/agents/chat_completion/chat_completion_agent.py:121](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/agents/chat_completion/chat_completion_agent.py#L121); [python/semantic_kernel/kernel.py:454-461](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/kernel.py#L454-L461)
- Settings classes read a .env file from the working directory by default, so that directory can set the model endpoint or enable sensitive telemetry without a trust decision. (ASI06; C6) — [python/semantic_kernel/kernel_pydantic.py:65-72](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/kernel_pydantic.py#L65-L72)

## Criterion details

### C1 Identity & least privilege — 0.05 (high)

Semantic Kernel runs every tool with whatever credentials the developer gives the AI service connector or the tool itself, usually long-lived API keys read from the environment. The framework has no way to scope a credential per tool, exchange tokens per request, or check authority before a tool runs. MCP servers launched over stdio receive only the MCP library's default minimal environment unless the developer passes one, which limits what those processes inherit.

- **S L0:** Connectors and tools use developer-supplied ambient credentials (API keys, Azure token credentials); the core has no downscoping or on-behalf-of primitive. — searched `rg -n -S 'on_behalf_of|token_exchange|downscop'` in `python/semantic_kernel` → 0 hits (No credential downscoping or token-exchange primitive in the Python package.) (verified)
  - *To reach the next level:* No per-tool or per-capability credential scoping primitive.
- **C L0:** invoke_function_call checks only that a function exists (and, if filters are set, is on the allowlist) before running it; there is no authorization layer. — [python/semantic_kernel/kernel.py:342-357](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/kernel.py#L342-L357); [python/semantic_kernel/kernel.py:444-448](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/kernel.py#L444-L448) (verified)
  - *To reach the next level:* No shared authorization check applied to every tool path.
- **D L0:** The default agent auto-invokes every registered function with the developer's full authority. — [python/semantic_kernel/agents/chat_completion/chat_completion_agent.py:121](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/agents/chat_completion/chat_completion_agent.py#L121) (verified)
  - *To reach the next level:* No narrower default identity for tools.
- **B L1:** A hijacked tool path reaches whatever the developer's keys and tool clients allow; the framework adds no independent layer that narrows them. — [python/semantic_kernel/connectors/ai/open_ai/settings/azure_open_ai_settings.py:104-106](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/connectors/ai/open_ai/settings/azure_open_ai_settings.py#L104-L106) (verified)
  - *To reach the next level:* Nothing in the framework restricts credentials to one project or to read-only use.
- **Cap:** none

### C2 Approval gates — 0.15 (high)

The default ChatCompletionAgent advertises every registered function and runs whatever the model calls, up to five rounds, with no human approval step. The framework offers function-invocation filters, a hook where developers can write their own approval logic, but ships no approval gate for kernel functions. The one built-in approval path covers MCP tools hosted by the Azure AI Foundry Agent Service: when that service asks, a developer callback sees the exact tool name and arguments and the call is denied if no callback is set. That path is narrow and depends on the developer enabling approval on the service side.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** No approval mechanism for kernel functions; auto-invoked calls go straight through the filter stack to the function. — [python/semantic_kernel/agents/chat_completion/chat_completion_agent.py:121](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/agents/chat_completion/chat_completion_agent.py#L121); [python/semantic_kernel/kernel.py:444-448](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/kernel.py#L444-L448) (verified)
    - *To reach the next level:* No per-call human approval primitive for registered tools.
  - **C L0:** All registered native, OpenAPI and MCP functions run without a gate. — searched `rg -n -S 'requires_approval|require_approval|needs_approval'` in `python/semantic_kernel` → 1 hits (The only hit is a docstring on the hosted-MCP approval request; no kernel-function approval flag exists.) (verified)
    - *To reach the next level:* No tool path crosses an approval gate by default.
  - **D L0:** Approval is not part of the default configuration. — [python/semantic_kernel/agents/chat_completion/chat_completion_agent.py:121](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/agents/chat_completion/chat_completion_agent.py#L121) (verified)
    - *To reach the next level:* Approval is opt-in (developer-written filter).
  - **B L0:** Registered tools may send, delete or write with no undo, and the framework provides no checkpoint or rollback. — [python/semantic_kernel/connectors/ai/chat_completion_client_base.py:138-154](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/connectors/ai/chat_completion_client_base.py#L138-L154) (verified)
    - *To reach the next level:* No rollback, preview or dry-run for consequential actions.
- **Azure AI Agent hosted-MCP approval callback** (alt; raw 0.15, cap G1 → 0.15) ← counted
  - **S L2:** The developer callback receives the exact server label, function name and raw arguments, and denial is the outcome when no callback is configured or it fails. — [python/semantic_kernel/agents/azure_ai/agent_thread_actions.py:1174-1199](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/agents/azure_ai/agent_thread_actions.py#L1174-L1199); [python/semantic_kernel/agents/azure_ai/mcp_tool_approval.py:17-33](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/agents/azure_ai/mcp_tool_approval.py#L17-L33) (verified)
    - *To reach the next level:* No risk tiers or argument-level policy; the developer callback is the whole decision.
  - **C L0:** Only hosted MCP calls on AzureAIAgent are covered; kernel functions, local MCP plugins and other agents are not. — [python/semantic_kernel/agents/azure_ai/agent_thread_actions.py:265-297](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/agents/azure_ai/agent_thread_actions.py#L265-L297) (verified)
    - *To reach the next level:* Most powerful tool paths are exempt.
  - **D L0:** The service only pauses for approval when the developer registers the MCP tool with require_approval set. — [python/semantic_kernel/agents/azure_ai/mcp_tool_approval.py:20-22](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/agents/azure_ai/mcp_tool_approval.py#L20-L22) (verified)
    - *To reach the next level:* Opt-in.
  - **B L0:** Same as the default: approved or ungated calls can be irreversible. — [python/semantic_kernel/agents/azure_ai/agent_thread_actions.py:1205](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/agents/azure_ai/agent_thread_actions.py#L1205) (verified)
    - *To reach the next level:* No rollback or preview.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C3 Tool & action scoping — 0.50 (high)

Every model-requested call is checked against the function's declared parameters: unknown or missing arguments are rejected and values are coerced into the declared Python or Pydantic types before the function runs. Bundled plugins are mostly careful: the HTTP plugin denies every host unless an allowlist is configured and disables redirects when one is, and OpenAPI path parameters are encoded and dot-segments rejected. The default agent has no tools until the developer adds some. Gaps: there is no central argument-policy layer, MCP servers can change their tool list at runtime and the new tools are loaded automatically, and the code-interpreter plugin's download function writes to any local path the model names unless download directories are configured.

- **S L2:** Typed schemas plus parameter-name checks on every call; individual plugins add allowlists (HTTP hosts and ports, upload directories), but the download path is unrestricted by default. — [python/semantic_kernel/kernel.py:375-397](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/kernel.py#L375-L397); [python/semantic_kernel/functions/kernel_function_from_method.py:124-150](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/functions/kernel_function_from_method.py#L124-L150); [python/semantic_kernel/core_plugins/sessions_python_tool/sessions_python_plugin.py:234-237](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/core_plugins/sessions_python_tool/sessions_python_plugin.py#L234-L237) (verified)
  - *To reach the next level:* No framework-level allowlist validation; the download path is permissive by default.
- **C L2:** Built-in plugins validate (HttpPlugin, OpenAPI path building, sessions upload); developer and MCP tools get only schema-level checks. — [python/semantic_kernel/core_plugins/http_plugin.py:98-113](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/core_plugins/http_plugin.py#L98-L113); [python/semantic_kernel/connectors/openapi_plugin/models/rest_api_operation.py:319-333](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/connectors/openapi_plugin/models/rest_api_operation.py#L319-L333) (verified)
  - *To reach the next level:* No shared validation layer for extension tools.
- **D L3:** No tools are registered by default and HttpPlugin denies all hosts unless allow_all_domains or allowed_domains is set; a per-call function allowlist exists via FunctionChoiceBehavior filters. — [python/semantic_kernel/core_plugins/http_plugin.py:46-68](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/core_plugins/http_plugin.py#L46-L68); [python/semantic_kernel/connectors/mcp.py:525-526](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/connectors/mcp.py#L525-L526) (verified)
  - *To reach the next level:* MCP servers can add new tools during a session, which are loaded without review.
- **B L1:** Developer tools, OpenAPI operations and MCP tools are as broad as their definitions; nothing bounds quantities. — [python/semantic_kernel/connectors/ai/chat_completion_client_base.py:138-154](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/connectors/ai/chat_completion_client_base.py#L138-L154) (verified)
  - *To reach the next level:* No quantity bounds or workspace scoping in the framework.
- **Cap:** none

### C4 Code-execution isolation — 0.47 (medium)

The core library does not run model-written code locally. The bundled code tool, SessionsPythonTool, sends code to Azure Container Apps dynamic sessions, a remote sandbox service, but it is opt-in. Configured stdio MCP servers and the developer's own plugins run on the host as ordinary processes or in-process code. Files the model downloads from the remote session can be written to any local path unless the developer restricts download directories.

- **S L4:** Code runs remotely in an Azure Container Apps session pool reached over HTTPS; nothing executes locally. — [python/semantic_kernel/core_plugins/sessions_python_tool/sessions_python_plugin.py:37](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/core_plugins/sessions_python_tool/sessions_python_plugin.py#L37); [python/semantic_kernel/core_plugins/sessions_python_tool/sessions_python_plugin.py:272](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/core_plugins/sessions_python_tool/sessions_python_plugin.py#L272) (verified)
- **C L1:** Only code sent to SessionsPythonTool is isolated; MCP stdio servers and in-process plugins run on the host. — [python/semantic_kernel/connectors/mcp.py:715-728](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/connectors/mcp.py#L715-L728) (verified)
  - *To reach the next level:* Other execution paths run on the host; L2 needs most paths sandboxed.
- **D L0:** Opt-in tool; not part of the default configuration. — [python/semantic_kernel/agents/chat_completion/chat_completion_agent.py:121](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/agents/chat_completion/chat_completion_agent.py#L121) (verified)
  - *To reach the next level:* Off by default; L1 needs it on by default.
- **B L2:** No host secrets are passed into the session and a session id persists per plugin instance; session network egress is a service-side setting, and sandbox output can be written to any host path by default. — [python/semantic_kernel/core_plugins/sessions_python_tool/sessions_python_settings.py:33](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/core_plugins/sessions_python_tool/sessions_python_settings.py#L33); [python/semantic_kernel/core_plugins/sessions_python_tool/sessions_python_plugin.py:234-237](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/core_plugins/sessions_python_tool/sessions_python_plugin.py#L234-L237) (inferred)
  - *To reach the next level:* Egress control is not set by the framework and downloads are not confined to a directory by default.
- **Cap:** G1 — The remote sandbox applies only when the developer adds SessionsPythonTool; nothing is isolated in the default configuration.

### C5 Untrusted input blast radius — 0.25 (high)

Tool and MCP results are appended to the conversation as ordinary tool messages with nothing marking them as untrusted, and the default agent then lets the model call any registered tool again without a human. The framework's prompt templates HTML-encode variable values and function output by default, which stops injected text from forging chat roles inside templates, but that is a formatting defense and does not cover tool results in the function-calling loop. A hijacked agent can therefore combine untrusted input, the developer's data and credentials, and outbound tools unattended.

- **S L1:** Template rendering escapes inserted values unless allow_dangerously_set_content is set; nothing else limits a hijacked agent. — [python/semantic_kernel/prompt_template/prompt_template_base.py:75-84](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/prompt_template/prompt_template_base.py#L75-L84) (verified)
  - *To reach the next level:* No approval or capability restriction once untrusted content has been read.
- **C L1:** Escaping applies to template variables and function output inlined in templates; tool results in the auto-invoke loop enter context unchanged. — [python/semantic_kernel/kernel.py:454-461](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/kernel.py#L454-L461) (verified)
  - *To reach the next level:* Tool and MCP results are not distinguished from instructions.
- **D L2:** Escaping is on by default; the developer can disable it per template or per variable with a named flag. — [python/semantic_kernel/prompt_template/prompt_template_base.py:36-42](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/prompt_template/prompt_template_base.py#L36-L42) (verified)
  - *To reach the next level:* C and D may not exceed S by more than one level; disabling is a quiet config flag.
- **B L0:** With default auto-invocation and no approval, an injected agent can both exfiltrate through any registered outbound tool and take irreversible actions. — [python/semantic_kernel/agents/chat_completion/chat_completion_agent.py:121](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/agents/chat_completion/chat_completion_agent.py#L121); [python/semantic_kernel/kernel.py:454-461](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/kernel.py#L454-L461) (verified)
  - *To reach the next level:* Neither exfiltration nor irreversible actions require approval.
- **Cap:** C5-WORSTCASE — Default configuration lets a hijacked agent leak data and act irreversibly without a human.

### C6 Memory, context & configuration integrity — 0.05 (high)

Every settings class, including the AI service connectors and the telemetry settings, reads a .env file from the process's working directory by default, filling any value not already set in the environment. A .env in the directory the app is started from can therefore set the model endpoint or turn on sensitive telemetry without any trust decision. The opt-in text memory plugin lets the model save anything into a collection it names, and recalled memories return as plain context with no provenance, review, or per-user namespace.

- **S L0:** Settings load .env from the working directory without a prompt, and the memory plugin accepts any model-written text. — [python/semantic_kernel/kernel_pydantic.py:65-72](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/kernel_pydantic.py#L65-L72); [python/semantic_kernel/core_plugins/text_memory_plugin.py:84-104](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/core_plugins/text_memory_plugin.py#L84-L104) (verified)
  - *To reach the next level:* No gate on memory writes and no trust decision before loading working-directory configuration.
- **C L0:** Neither the .env path nor the memory stores are controlled. — [python/semantic_kernel/kernel_pydantic.py:65-72](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/kernel_pydantic.py#L65-L72) (verified)
  - *To reach the next level:* No memory or configuration path is controlled.
- **D L0:** Memory collections are chosen by the model or developer; there is no per-user namespace by default. — [python/semantic_kernel/core_plugins/text_memory_plugin.py:92](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/core_plugins/text_memory_plugin.py#L92) (verified)
  - *To reach the next level:* No per-user or per-session namespace enforced by default.
- **B L1:** Saved memories persist across sessions and can steer later tool use; working-directory configuration persists for every run started there. — [python/semantic_kernel/core_plugins/text_memory_plugin.py:88-104](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/core_plugins/text_memory_plugin.py#L88-L104) (verified)
  - *To reach the next level:* Persistent poisoned context can trigger tool use.
- **Cap:** C6-REPOCONFIG — A .env in the working directory is loaded by default and can set security-relevant values such as the model endpoint without a trust decision.

### C7 Third-party extensions — 0.28 (medium)

Third-party code enters through MCP servers, OpenAPI plugins, native plugin directories and Hugging Face models, all configured by the developer in code; nothing is enabled by default and no workspace file adds extensions. There is no pinning or integrity check, and when an MCP server announces a changed tool list the plugin reloads it silently. Stdio MCP servers run as separate processes with the MCP library's minimal default environment unless the developer passes one.

- **S L1:** Extensions are whatever command, URL or path the developer supplies, fetched or launched unpinned each run. — [python/semantic_kernel/connectors/mcp.py:715-728](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/connectors/mcp.py#L715-L728); searched `rg -n -S 'sha256|verify_signature|integrity'` in `python/semantic_kernel/connectors/mcp.py python/semantic_kernel/connectors/openapi_plugin python/semantic_kernel/functions/kernel_plugin.py` → 0 hits (verified)
  - *To reach the next level:* No version pinning or integrity checks.
- **C L0:** No extension type is verified. — [python/semantic_kernel/connectors/mcp.py:525-526](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/connectors/mcp.py#L525-L526) (verified)
  - *To reach the next level:* No extension type is verified.
- **D L2:** Extensions are added only explicitly in developer code; MCP sampling requests are denied unless a consent callback or auto-approve is set. — [python/semantic_kernel/connectors/mcp.py:395-402](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/connectors/mcp.py#L395-L402) (verified)
  - *To reach the next level:* Tool list changes are accepted without showing what changed.
- **B L2:** With env left unset, the MCP SDK's stdio client passes only a small default set of variables (inferred from that library's behaviour); native plugins run in-process. — [python/semantic_kernel/connectors/mcp.py:713-722](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/connectors/mcp.py#L713-L722) (inferred)
  - *To reach the next level:* No per-extension sandbox or scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.33 (high)

API keys are held as Pydantic SecretStr values, and OpenTelemetry capture of prompts, arguments and results is off by default. Tool-call arguments are logged at INFO level, and the full text of tool exceptions is returned to the model, with no masking on either path. A version header is added to outbound requests unless AZURE_TELEMETRY_DISABLED is set. Keys are typically long-lived.

- **S L2:** Type-level masking for keys; no secret-scrubbing helpers on log or model-bound paths. — [python/semantic_kernel/connectors/ai/open_ai/settings/azure_open_ai_settings.py:104-106](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/connectors/ai/open_ai/settings/azure_open_ai_settings.py#L104-L106) (verified)
  - *To reach the next level:* No secret scrubbing before logs or model-bound messages.
- **C L1:** Only telemetry content is protected (sensitive capture off); logs include tool arguments and tool exceptions go to the model verbatim. — [python/semantic_kernel/kernel.py:426](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/kernel.py#L426); [python/semantic_kernel/kernel.py:476-479](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/kernel.py#L476-L479) (verified)
  - *To reach the next level:* Logs and model-bound error messages are unprotected.
- **D L1:** Content-free version telemetry header on by default; sensitive diagnostics off by default but can be enabled from a working-directory .env. — [python/semantic_kernel/utils/telemetry/user_agent.py:10-25](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/utils/telemetry/user_agent.py#L10-L25); [python/semantic_kernel/utils/telemetry/model_diagnostics/model_diagnostics_settings.py:30-31](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/utils/telemetry/model_diagnostics/model_diagnostics_settings.py#L30-L31) (verified)
  - *To reach the next level:* Telemetry header is on by default and sensitive diagnostics are toggled by a loadable setting.
- **B L1:** Long-lived provider and tool API keys, moderately scoped. — [python/semantic_kernel/connectors/ai/open_ai/settings/azure_open_ai_settings.py:104-106](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/connectors/ai/open_ai/settings/azure_open_ai_settings.py#L104-L106) (verified)
  - *To reach the next level:* No short-lived or scoped credentials.
- **Cap:** none

### C9 Audit & traceability — 0.35 (high)

Every kernel function invocation, including MCP and OpenAPI tools, runs inside an OpenTelemetry execute_tool span with the tool name, call id and timing; arguments and results are attached only when sensitive diagnostics are enabled. Nothing is recorded anywhere unless the developer configures an exporter, and there is no actor attribution or tamper-evident storage.

- **S L2:** Structured spans per tool call with name, call id and status. — [python/semantic_kernel/utils/telemetry/model_diagnostics/function_tracer.py:22-63](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/utils/telemetry/model_diagnostics/function_tracer.py#L22-L63); [python/semantic_kernel/functions/kernel_function.py:264-268](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/functions/kernel_function.py#L264-L268) (verified)
  - *To reach the next level:* No actor attribution or approver fields.
- **C L2:** All kernel functions, including extension tools, are spanned; there are no approvals to record. — [python/semantic_kernel/functions/kernel_function.py:264](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/functions/kernel_function.py#L264) (verified)
  - *To reach the next level:* Approvals, denials and sub-agent correlation are not recorded.
- **D L0:** Spans go nowhere unless the developer installs an OpenTelemetry exporter. — [python/semantic_kernel/utils/telemetry/model_diagnostics/model_diagnostics_settings.py:30](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/utils/telemetry/model_diagnostics/model_diagnostics_settings.py#L30) (verified)
  - *To reach the next level:* Opt-in only.
- **B L1:** Export is best-effort through whatever OpenTelemetry processor the developer sets up. — [python/semantic_kernel/functions/kernel_function.py:264](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/functions/kernel_function.py#L264) (verified)
  - *To reach the next level:* Records are not durable per action.
- **Cap:** G1 — Tool-call records exist only when the developer configures an OpenTelemetry exporter.

### C10 Limits & kill switch — 0.30 (high)

The function-calling loop stops after five rounds by default, but each round can run any number of tool calls in parallel, and there is no token, cost or wall-clock budget. Tool calls have no framework timeout and MCP requests have none unless the developer sets one. Group chats default to 99 iterations and orchestration group chats have no round limit; cancelling an orchestration stops new messages but lets in-flight work finish.

- **S L1:** Iteration cap only; cancellation is cooperative. — [python/semantic_kernel/connectors/ai/function_choice_behavior.py:18](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/connectors/ai/function_choice_behavior.py#L18); [python/semantic_kernel/agents/orchestration/orchestration_base.py:69-80](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/agents/orchestration/orchestration_base.py#L69-L80) (verified)
  - *To reach the next level:* No token, cost or wall-clock cap and no tool timeouts.
- **C L1:** The cap applies per model request; parallel calls, sub-agents and group chats each have their own counters. — [python/semantic_kernel/connectors/ai/chat_completion_client_base.py:138-154](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/connectors/ai/chat_completion_client_base.py#L138-L154); [python/semantic_kernel/agents/orchestration/group_chat.py:151](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/agents/orchestration/group_chat.py#L151) (verified)
  - *To reach the next level:* Tool timeouts and shared budgets are absent.
- **D L2:** Five auto-invoke rounds is a sensible default; the developer can change it. — [python/semantic_kernel/connectors/ai/function_choice_behavior.py:58](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/connectors/ai/function_choice_behavior.py#L58) (verified)
  - *To reach the next level:* Sub-agents and group chats reset limits; 99-iteration and unlimited defaults elsewhere.
- **B L1:** No time or spend ceiling, unlimited orchestration rounds by default, and in-flight work continues after cancel. — [python/semantic_kernel/agents/strategies/termination/termination_strategy.py:22](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/agents/strategies/termination/termination_strategy.py#L22); [python/semantic_kernel/connectors/mcp.py:338](https://github.com/microsoft/semantic-kernel/blob/9974625ddc1e1d3f55e095420fcabcc9329f7f34/python/semantic_kernel/connectors/mcp.py#L338) (verified)
  - *To reach the next level:* No tight time or cost ceilings.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Tool and MCP results appended unmarked (python/semantic_kernel/kernel.py:461) · [B] sensitive data/systems: Developer API keys and tool credentials (python/semantic_kernel/connectors/ai/open_ai/settings/azure_open_ai_settings.py:106) · [C] state change / egress: Any registered tool auto-invoked (python/semantic_kernel/agents/chat_completion/chat_completion_agent.py:121) · Same default session? Yes

## Highest-impact improvements
1. Add a per-function approval flag enforced in invoke_function_call that pauses and returns the exact call for human approval. — C2 S L0→L3, +0.225 before caps (Playbook 5)
2. Stop reading .env from the working directory unless env_file_path is passed explicitly. — C6 S L0→L1, +0.075 before caps (Playbook 2)
3. Add default timeouts for tool invocations and MCP requests, and a token or cost budget for the auto-invoke loop. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)
4. Scrub secrets from exception text before returning tool errors to the model and stop logging tool arguments at INFO. — C8 C L1→L2, +0.075 before caps (Playbook 4)
5. Default SessionsPythonTool downloads to a configured directory instead of any path. — C3 S L2→L3, +0.075 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Multi-language repo: the Python edition (python/) was scored because the README leads with it. The .NET edition in dotnet/ was not scored and may differ; Java has moved to the separate semantic-kernel-java repository and was not reviewed.
- The README states Semantic Kernel is succeeded by Microsoft Agent Framework (scored separately).
- Commit is not at a release tag; the Python package version at this commit is 1.44.1.
- Azure Container Apps session isolation and egress, and the MCP SDK's default stdio environment, are inferred from those services' and libraries' behaviour, not read here.
- No text aimed at AI reviewers was found in the repository.
