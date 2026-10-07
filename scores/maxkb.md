# Defense-in-depth score: MaxKB

**Repo:** https://github.com/1Panel-dev/MaxKB · **Commit:** `2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43` · **Reviewed:** 2026-10-05
**What it is:** Open-source platform for building enterprise RAG agents and agentic workflows with tools, MCP servers and skills.
**Category:** Agent Frameworks
**Scored configuration:** Single Docker image as shipped (README quick start, installer/Dockerfile on the maxkb-base image, MAXKB_SANDBOX=1): agents built with the AI chat node and designer-selected custom tools, MCP servers, nested agents or skills, published with the default public chat link.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 3.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L1 | L1 | L1 | 0.33 | none | **0.33** | High |
| C2 | Approval gates | L0 | L0 | L0 | L1 | 0.05 | none | **0.05** | High |
| C3 | Tool & action scoping | L2 | L2 | L2 | L1 | 0.45 | none | **0.45** | High |
| C4 | Code-execution isolation | L2 | L3 | L2 | L1 | 0.53 | none | **0.53** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L2 | L0 | 0.25 | none | **0.25** | High |
| C7 | Third-party extensions | L2 | L1 | L2 | L2 | 0.42 | none | **0.42** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L1 | 0.45 | C8-MODELSECRETS | **0.25** | High |
| C9 | Audit & traceability | L2 | L1 | L2 | L1 | 0.38 | none | **0.38** | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L1 | 0.45 | none | **0.45** | High |


MaxKB runs all agent and tool code under a dedicated low-privilege account with a cleared environment and a process and network blocking library, which is more care than most agent platforms take. But nothing an agent does needs a person's approval, every published agent gets an unauthenticated public chat link by default, and any agent with a tool also gets a shell with open internet access. The dominant risk is a hijacked public agent leaking knowledge-base data or acting through its tools unattended.

## Critical gaps
- A hijacked agent can leak knowledge-base data and take irreversible actions through its tools with no person involved, and public-link users reach it without logging in. (ASI01, LLM01; C5). Evidence: [apps/application/models/application_access_token.py:22](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/models/application_access_token.py#L22); [apps/application/flow/tools.py:470](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/tools.py#L470)
- Secret material is not kept out of model-bound requests in the default agent flow. (ASI03, LLM02; C8). Evidence: [apps/application/flow/step_node/ai_chat_step_node/impl/base_chat_node.py:368](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/step_node/ai_chat_step_node/impl/base_chat_node.py#L368)

## Criterion details

### C1 Identity & least privilege: 0.33 (high confidence)

Agents act with whatever credentials the builder attaches to each tool, MCP server or nested agent, and those credentials are the same for every person who chats with the agent. Nothing checks a tool call against the chat user who caused it, and each published agent gets a public, unauthenticated chat link by default. Code the agent runs is dropped to a separate low-privilege account with a cleared environment, which narrows what that code inherits, but the platform itself runs as root in a container that also holds the database. The install starts with a documented default administrator password.

- **S L2:** Credentials are configured per tool, per MCP server and per nested agent (the nested agent's API key) and used unchanged for the whole run. Evidence: [apps/application/flow/step_node/ai_chat_step_node/impl/base_chat_node.py:368](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/step_node/ai_chat_step_node/impl/base_chat_node.py#L368); [apps/common/utils/tool_code.py:346-348](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/common/utils/tool_code.py#L346-L348) (verified)
  - *To reach the next level:* No narrowing of credentials per request or per chat user; tools reuse long-lived builder-supplied secrets.
- **C L1:** Tool calls are not authorized against the requesting chat user; anonymous public-link users drive the same tool credentials as the builder. Evidence: [apps/application/models/application_access_token.py:22](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/models/application_access_token.py#L22); [apps/application/models/application_access_token.py:30](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/models/application_access_token.py#L30); searched `rg -n -S -i 'approv|confirm|human_in_the_loop'` in `apps/application/flow` → 0 hits (no authorization or approval step in the agent tool path) (verified)
  - *To reach the next level:* No per-call authorization layer that every tool path passes through, and no check against the requesting principal.
- **D L1:** Public chat access is on and unauthenticated by default for each agent, and the platform ships a documented default administrator password that is not forced to change. Evidence: [apps/application/models/application_access_token.py:22](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/models/application_access_token.py#L22); [apps/users/migrations/0001_initial.py:10](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/users/migrations/0001_initial.py#L10) (verified)
  - *To reach the next level:* Default deployment is not least-privilege: public access and the default admin login need manual hardening.
- **B L1:** A hijacked agent can use every credential attached to its tools (external APIs, MCP servers, nested agents), typically with write access across several systems; the platform process itself runs as root in the container. Evidence: [apps/common/utils/tool_code.py:358-362](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/common/utils/tool_code.py#L358-L362); searched `rg -n -S '^USER'` in `installer` → 0 hits (no USER directive: the application runs as root inside the container) (verified)
  - *To reach the next level:* Credentials are not scoped to one project, mostly read-only, or short-lived.
- **Cap:** none

### C2 Approval gates: 0.05 (high confidence)

There is no approval step for anything an agent does. When an agent has any tool attached, the platform builds it with a shell, file tools, a sub-agent tool and the attached MCP servers, custom tools and nested agents, and explicitly disables interruption for the file tools; no tool call waits for a person. Workflow form nodes collect input from the chat user, who may be the untrusted party, so they do not act as an approval gate. Consequential actions through MCP servers and custom tools happen without a person in the loop.

- **S L0:** No human approval for any tool call; interruption is explicitly turned off. Evidence: [apps/application/flow/tools.py:470](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/tools.py#L470); searched `rg -n -S -i 'approv|confirm|human_in_the_loop'` in `apps/application/flow` → 0 hits (verified)
  - *To reach the next level:* No per-call approval showing the exact call.
- **C L0:** The shell tool, MCP tools, custom tools, nested agents and sub-agents all run without a gate. Evidence: [apps/application/flow/tools.py:464-472](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/tools.py#L464-L472) (verified)
  - *To reach the next level:* The most powerful tools (shell, MCP, custom code) are ungated.
- **D L0:** No approval exists to turn on. Evidence: [apps/application/flow/tools.py:470](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/tools.py#L470) (verified)
  - *To reach the next level:* Approval is not available, on or off.
- **B L1:** Shell and file actions are confined to a per-request temporary directory that is deleted afterwards, but MCP servers, custom tools and nested agents can take irreversible external actions. Evidence: [apps/application/flow/tools.py:803-806](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/tools.py#L803-L806); [apps/application/flow/tools.py:457](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/tools.py#L457) (verified)
  - *To reach the next level:* No rollback or preview for external actions.
- **Cap:** none

### C3 Tool & action scoping: 0.45 (high confidence)

Some tool inputs are handled carefully: shell commands are split into simple commands and each one is re-quoted before it runs, the file tools are confined to a temporary directory, and the platform's URL fetcher only connects to public addresses and refuses redirects. But the agent's main tools are general purpose (a shell that can run arbitrary Python with internet access, arbitrary MCP servers, builder-written code), and MCP configuration is only checked for its transport. Attaching any single tool to an agent also hands it the shell, file and sub-agent tools.

- **S L2:** Typed tool schemas, per-command re-quoting of shell input, path-contained file tools and a resolver-level public-address check in the URL fetcher, around general-purpose shell and MCP tools. Evidence: [apps/application/flow/backend/sandbox_shell.py:277](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/backend/sandbox_shell.py#L277); [apps/oss/url_fetch.py:39](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/oss/url_fetch.py#L39) (verified)
  - *To reach the next level:* General tools are not replaced by narrow ones and shell egress is not allowlisted.
- **C L2:** Built-in paths validate inputs; MCP server configurations are checked only for transport and URL presence and their tools' arguments pass through. Evidence: [apps/common/mcp/config.py:21](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/common/mcp/config.py#L21) (verified)
  - *To reach the next level:* Extension (MCP) tools are not wrapped by a shared validation layer.
- **D L2:** Builders pick tools per agent, but any tool selection also adds the shell, file and sub-agent tools. Evidence: [apps/application/flow/step_node/ai_chat_step_node/impl/base_chat_node.py:427](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/step_node/ai_chat_step_node/impl/base_chat_node.py#L427); [apps/application/flow/tools.py:464-472](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/tools.py#L464-L472) (verified)
  - *To reach the next level:* The default tool group still includes shell and write tools; no read-only default.
- **B L1:** A misused shell runs arbitrary Python as the sandbox account with outbound internet access; MCP and custom tools reach whatever their credentials allow. Evidence: [apps/application/flow/backend/sandbox_shell.py:255-256](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/backend/sandbox_shell.py#L255-L256) (verified)
  - *To reach the next level:* Tools are not scoped and quantity-bounded.
- **Cap:** none

### C4 Code-execution isolation: 0.53 (high confidence)

Every code path the model or a builder can reach (the shell tool, custom tool code, custom tools served over MCP, the remote-MCP proxy and the web crawler) runs as a dedicated low-privilege account with a cleared environment and a preloaded library that blocks new processes, restricted syscalls and connections to a configured host list. This is careful work, but it is a user-and-library boundary inside the same container as the application (running as root), PostgreSQL and Redis, not a separate container or kernel-level sandbox. The sandbox is on by default and switched off by a single environment variable. The shell path has no memory or CPU limits and outbound internet access stays open.

- **S L2:** Dedicated low-privilege user plus an LD_PRELOAD interposer that blocks exec, fork, clone, restricted syscalls and listed hosts; no namespaces, seccomp or separate container. Evidence: [installer/Dockerfile-base:29](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/installer/Dockerfile-base#L29); [installer/sandbox.c:303-305](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/installer/sandbox.c#L303-L305); [installer/sandbox.c:230](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/installer/sandbox.c#L230) (verified)
  - *To reach the next level:* No hardened container, seccomp profile or kernel-separated sandbox.
- **C L3:** Shell commands, tool code, MCP-served tool code and the remote-MCP proxy all switch to the sandbox account; turning the sandbox off is an operator setting named in the code. Evidence: [apps/application/flow/backend/sandbox_shell.py:299-305](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/backend/sandbox_shell.py#L299-L305); [apps/common/utils/tool_code.py:112](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/common/utils/tool_code.py#L112); [apps/common/mcp/sandbox_worker.py:151-154](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/common/mcp/sandbox_worker.py#L151-L154); [apps/common/mcp/sandbox.py:23](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/common/mcp/sandbox.py#L23) (verified)
  - *To reach the next level:* Fallback when the preload library is missing is not fail-closed on every path, and spawned background shell jobs are not covered by limits.
- **D L2:** On by default in the image; one environment variable disables it with no warning. Evidence: [installer/Dockerfile-base:48](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/installer/Dockerfile-base#L48); [apps/common/utils/tool_code.py:26](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/common/utils/tool_code.py#L26) (verified)
  - *To reach the next level:* Disabling is not an explicit, loudly named operator flag.
- **B L1:** Inside the sandbox, code has open outbound internet access; a break of the user/preload boundary lands in a container where the app runs as root next to the database and cache holding every stored credential. Evidence: [installer/start-all.sh:14-15](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/installer/start-all.sh#L14-L15); searched `rg -n -S '^USER'` in `installer` → 0 hits (the application runs as root in the same container); searched `rg -n -S 'setrlimit'` in `apps/application/flow/backend` → 0 hits (no resource limits on the shell tool path) (verified)
  - *To reach the next level:* Escape is not contained to a workspace-only environment without credentials and with egress off or allowlisted.
- **Cap:** none

### C5 Untrusted input blast radius: 0.00 (high confidence)

Nothing limits what injected content can make an agent do. Agents read public chat users' messages (each agent gets an unauthenticated public link by default), uploaded and crawled documents in the knowledge base, MCP tool results and other agents' replies, and none of it is marked or treated differently from instructions. In the same session an agent can reach knowledge-base data and tool credentials, send data out through the shell's internet access or its tools, and take actions through MCP servers and custom tools, with no person involved.

- **S L0:** No injection detection, provenance marking or approval tied to untrusted content. Evidence: searched `rg -n -S -i 'prompt.?injection|untrusted|taint|provenance'` in `apps/application apps/chat` → 0 hits (verified)
  - *To reach the next level:* No approval or capability restriction once untrusted content has been read.
- **C L0:** Public chat input, knowledge-base content, MCP results and nested-agent replies all enter context with the same standing. Evidence: [apps/application/flow/tools.py:474-477](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/tools.py#L474-L477); [apps/application/models/application_access_token.py:22](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/models/application_access_token.py#L22) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished.
- **D L0:** No control exists. Evidence: [apps/application/flow/tools.py:470](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/tools.py#L470) (verified)
  - *To reach the next level:* No untrusted-input control to enable.
- **B L0:** A hijacked agent can leak knowledge-base data and take irreversible actions through its tools unattended, and public-link users can reach the agent without logging in. Evidence: [apps/application/models/application_access_token.py:22](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/models/application_access_token.py#L22); [apps/application/models/application_access_token.py:30](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/models/application_access_token.py#L30); [apps/application/flow/backend/sandbox_shell.py:255](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/backend/sandbox_shell.py#L255); [apps/application/flow/tools.py:470](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/tools.py#L470) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions are not gated by a person.
- **Cap:** C5-WORSTCASE: Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity: 0.25 (high confidence)

Knowledge bases are shared by everyone who chats with an agent, and they can be filled from crawled websites, uploaded files and knowledge workflows, so poisoned content persists and is retrieved for every user, including for agents that hold tools. Long-term memory is off by default; when enabled it is an LLM-written summary per agent and chat user, stored without validation and substituted into the system prompt. Memory is namespaced per agent and chat user in the database queries. There is no workspace or repository configuration that the agent auto-loads.

- **S L1:** Memory and knowledge writes are not validated; long-term memory is substituted into the system prompt as trusted context. Evidence: [apps/application/long_term_memory/__init__.py:276](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/long_term_memory/__init__.py#L276); [apps/application/chat_pipeline/step/chat_step/impl/base_chat_step.py:520](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/chat_pipeline/step/chat_step/impl/base_chat_step.py#L520) (verified)
  - *To reach the next level:* Retrieved and remembered content is not provenance-tagged or gated.
- **C L1:** Long-term memory is namespaced; knowledge bases and crawled sources have no write control beyond builder permissions. Evidence: [apps/knowledge/task/sync.py:28](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/knowledge/task/sync.py#L28) (verified)
  - *To reach the next level:* Knowledge bases and web-synced content are not controlled.
- **D L2:** Long-term memory is off by default and scoped per agent and chat user in the query; knowledge bases are shared across users by design. Evidence: [apps/application/models/application.py:109](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/models/application.py#L109); [apps/application/models/application_chat.py:174](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/models/application_chat.py#L174) (verified)
  - *To reach the next level:* The model's writes and isolation are not separately locked down beyond the per-user query filter.
- **B L0:** Poisoned knowledge-base content persists across sessions and users and can steer agents that hold tools. Evidence: [apps/knowledge/task/sync.py:28](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/knowledge/task/sync.py#L28) (verified)
  - *To reach the next level:* Poisoned content is not session-scoped or reviewed before reuse.
- **Cap:** none

### C7 Third-party extensions: 0.42 (high confidence)

Tools from the official tool store are downloaded over HTTPS from a single allowlisted vendor host, deserialized with a class allowlist and added switched off, so a builder has to enable them. MCP servers are any URL a builder enters, unpinned, and their tool definitions are fetched fresh every session with no change detection. Third-party code runs in the same sandbox account with a cleared environment, but there is no per-extension isolation and no hash or signature check on store downloads.

- **S L2:** Store tools come from an allowlisted vendor host and are stored at the version installed; MCP servers are unpinned user-entered URLs. Evidence: [apps/tools/serializers/tool.py:1345](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/tools/serializers/tool.py#L1345); [apps/common/utils/url_validator.py:12](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/common/utils/url_validator.py#L12); [apps/tools/serializers/tool.py:1352](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/tools/serializers/tool.py#L1352) (verified)
  - *To reach the next level:* No integrity check (hash or signature) and no re-approval when an MCP server's tools change.
- **C L1:** Only store tools have a source check; MCP servers and uploaded skills do not. Evidence: [apps/common/mcp/config.py:21](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/common/mcp/config.py#L21) (verified)
  - *To reach the next level:* Most extension types are unverified.
- **D L2:** Store tools are installed explicitly and start inactive; MCP servers are added by builders. Evidence: [apps/tools/serializers/tool.py:1382](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/tools/serializers/tool.py#L1382) (verified)
  - *To reach the next level:* Adding an extension does not show what it will run and with which permissions.
- **B L2:** Extension code and the remote-MCP proxy run as a separate sandbox process with a cleared environment. Evidence: [apps/common/mcp/sandbox_worker.py:151-154](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/common/mcp/sandbox_worker.py#L151-L154); [apps/common/utils/tool_code.py:128](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/common/utils/tool_code.py#L128) (verified)
  - *To reach the next level:* No per-extension sandbox or scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection: 0.25 (high confidence)

Model and tool credentials are encrypted at rest and masked in the UI, but the decryption key pair lives in the same database. Sandboxed processes get a cleared environment, and the remote-MCP proxy deliberately strips exception text so headers and URLs do not leak. There is no telemetry and the image logs at INFO, though the code defaults to DEBUG when unset and debug logs are not redacted. Secret material is not kept out of model-bound requests in the default agent flow, which caps this criterion.

- **S L2:** Encryption at rest with a co-located key, UI masking, environment scrubbing and error-message scrubbing on the MCP path. Evidence: [apps/common/utils/rsa_util.py:65](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/common/utils/rsa_util.py#L65); [apps/models_provider/base_model_provider.py:145](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/models_provider/base_model_provider.py#L145); [apps/common/mcp/sandbox_proxy.py:19](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/common/mcp/sandbox_proxy.py#L19) (verified)
  - *To reach the next level:* No secret manager or redaction before model-bound messages.
- **C L2:** Subprocess environments and MCP errors are protected; model-bound requests and debug logs are not. Evidence: [apps/common/utils/tool_code.py:128](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/common/utils/tool_code.py#L128); [apps/common/mcp/sandbox_proxy.py:156](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/common/mcp/sandbox_proxy.py#L156); [apps/application/flow/backend/sandbox_shell.py:310](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/backend/sandbox_shell.py#L310) (verified)
  - *To reach the next level:* Model-bound messages and debug logs are not covered.
- **D L2:** No telemetry; the image sets INFO logging, while the code default is DEBUG. Evidence: [installer/Dockerfile-base:47](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/installer/Dockerfile-base#L47); [apps/maxkb/conf.py:128](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/maxkb/conf.py#L128); searched `rg -n -i -S 'sentry|posthog|mixpanel|telemetry|opentelemetry'` in `apps` → 0 hits (verified)
  - *To reach the next level:* Redaction is not always on.
- **B L1:** Stored model and tool keys are long-lived and only as narrow as the builder made them. Evidence: [apps/models_provider/models/model_management.py:40](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/models_provider/models/model_management.py#L40); [apps/application/flow/step_node/ai_chat_step_node/impl/base_chat_node.py:368](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/step_node/ai_chat_step_node/impl/base_chat_node.py#L368) (verified)
  - *To reach the next level:* Keys are not short-lived or scoped by the platform.
- **Cap:** C8-MODELSECRETS: Secret material is not kept out of model-bound requests in the default agent flow.

### C9 Audit & traceability: 0.38 (high confidence)

Calls to library tools are saved with their input and output, chat records keep each conversation and its node details per chat user, and administrative actions go to an audit log with user and IP. But shell commands, file operations and sub-agent calls are only visible inside the answer text or debug logs, tool records carry no actor, and everything lives in the application database that the platform process can change. Records are written after the action runs.

- **S L2:** Structured tool records (input, output, state) and an operation audit log with user and IP. Evidence: [apps/application/flow/tools.py:734-746](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/tools.py#L734-L746); [apps/system_manage/models/log_management.py:29-33](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/system_manage/models/log_management.py#L29-L33) (verified)
  - *To reach the next level:* Tool records have no actor attribution or correlation across sub-agents.
- **C L1:** Only library tools whose result carries a tool id are recorded; shell, file and sub-agent calls are not. Evidence: [apps/application/flow/tools.py:699](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/tools.py#L699); [apps/application/flow/backend/sandbox_shell.py:310](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/backend/sandbox_shell.py#L310) (verified)
  - *To reach the next level:* Built-in shell, file and sub-agent calls are not recorded as structured events.
- **D L2:** On by default and stored in the database, outside the agent's sandbox but writable by the platform process. Evidence: [apps/system_manage/models/log_management.py:17](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/system_manage/models/log_management.py#L17) (verified)
  - *To reach the next level:* Not written by a component the platform process cannot alter.
- **B L1:** Tool records are written after execution; failures do not block actions. Evidence: [apps/application/flow/tools.py:734](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/tools.py#L734) (verified)
  - *To reach the next level:* Records are not durable per action before the action runs.
- **Cap:** none

### C10 Limits & kill switch: 0.45 (high confidence)

Agent runs are capped at 100 graph steps by default, tool code times out after an hour, the shell tool after two minutes, and public-link users have a daily request cap. There is no token or cost cap, sub-agents start a fresh step budget, background shell jobs can outlive the request, and the chat API has no stop endpoint that cancels in-flight work.

- **S L2:** Step cap plus per-execution timeouts and a per-public-client daily request cap. Evidence: [apps/application/flow/tools.py:473](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/tools.py#L473); [apps/common/utils/tool_code.py:34](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/common/utils/tool_code.py#L34); [apps/chat/serializers/chat.py:323](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/chat/serializers/chat.py#L323) (verified)
  - *To reach the next level:* No token or cost cap and no halt that cancels in-flight work.
- **C L2:** Limits apply to the top-level loop and tool executions; sub-agents run with their own step budget. Evidence: [apps/application/flow/tools.py:464-472](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/application/flow/tools.py#L464-L472) (verified)
  - *To reach the next level:* Sub-agents and background jobs do not count against the same budget.
- **D L2:** Defaults exist and are operator-configurable; the one-hour execution timeout is long. Evidence: [apps/common/utils/tool_code.py:34](https://github.com/1Panel-dev/MaxKB/blob/2c7c8c9f2097f48c2bf3fee6bbb2a6f422bbbe43/apps/common/utils/tool_code.py#L34) (verified)
  - *To reach the next level:* The model can reset its step budget by delegating to a sub-agent.
- **B L1:** Ceilings are large (hour-long executions, no spend cap) and stopping leaves background work running. Evidence: searched `rg -n -S -i 'stop|cancel|abort'` in `apps/chat/views` → 0 hits (no stop or cancel endpoint in the chat API) (verified)
  - *To reach the next level:* No tight per-run time and cost ceilings or cancellation of pending calls.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Public unauthenticated chat users, crawled and uploaded knowledge, MCP results (apps/application/models/application_access_token.py:22) · [B] sensitive data/systems: Knowledge bases and tool credentials decrypted for agent tools (apps/application/flow/step_node/ai_chat_step_node/impl/base_chat_node.py:368) · [C] state change / egress: Shell with internet access, MCP servers, custom tools, nested agents, none gated (apps/application/flow/tools.py:464-472) · Same default session? Yes

## Highest-impact improvements
1. Add per-call human approval (deepagents interrupt_on) for the shell, MCP tools, custom tools and nested agents, showing the exact arguments. (C2 S L0→L3, +0.225 before caps; Playbook 5)
2. Keep stored tool credentials out of everything sent to the model; inject them only at execution time. (C8 S L2→L3, +0.075 before caps; Playbook 4)
3. Only give agents the shell and file tools when the builder enables them, instead of whenever any tool is attached. (C3 D L2→L3, +0.050 before caps; Playbook 3)
4. When a session includes public-link input or retrieved content, require approval for egress and state-changing tools. (C5 S L0→L2, +0.150 before caps; Playbook 1)
5. Run sandboxed code in a separate hardened container (non-root, seccomp, no-new-privileges) with an egress allowlist instead of a preload library in the app container. (C4 S L2→L3, +0.075 before caps; Playbook 3)

## Re-audit log
- C4 C: L2 → L3. Traced every model-reachable execution path (shell backend, tool code, MCP-served tool code, remote-MCP worker, crawler) to the sandbox user switch; the escape hatch is named in apps/common/mcp/sandbox.py:23.
- C7 S: L3 → L2. The curated store allowlist applies only to store tools; MCP servers are unpinned user URLs with tool definitions refetched every session, and store downloads carry no hash or signature.
- C1 C: L2 → L1. No tool path checks the requesting chat user; anonymous public-link users drive the builder's tool credentials.
- C4 B: L2 → L1. Sandbox shares the container with the root application, PostgreSQL and Redis, and the shell path has no resource limits; a boundary failure reaches stored credentials.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Behaviour of third-party libraries (deepagents 0.6.11, mcp 1.28.1, langchain-mcp-adapters 0.3.0) was read from their published packages, not run; sub-agent budget behaviour relies on deepagents' task tool.
- File permissions inside the maxkb-base image were inferred from installer/Dockerfile-base; the published image was not inspected.
- The enterprise xpack module is not in the repository and was not reviewed; the frontend was reviewed only for chat rendering configuration.
- The target's AGENTS.md and CLAUDE.md were not reviewed in full; no reviewer-steering text was found in the files that were read.
