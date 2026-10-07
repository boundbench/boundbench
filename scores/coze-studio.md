# Defense-in-depth score: Coze Studio

**Repo:** https://github.com/coze-dev/coze-studio · **Commit:** `fefb05ff27be1da939612fbf9faf5db62583b8ae` · **Reviewed:** 2026-10-05
**What it is:** Open-source low-code platform for building, debugging and publishing AI agents, apps and workflows.
**Category:** Agent Frameworks
**Scored configuration:** Docker Compose stack as shipped via `make web` (docker/docker-compose.yml with docker/.env.example): single agents with author-bound plugins, workflows, databases and variables, workflow code nodes on the default sandbox runner, open registration.
**Agent surface (default):** code execution yes · filesystem write no · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 3.0 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L1 | L0 | 0.20 | none | **0.20** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | none | **0.00** | High |
| C3 | Tool & action scoping | L2 | L2 | L3 | L1 | 0.50 | none | **0.50** | High |
| C4 | Code-execution isolation | L3 | L3 | L2 | L0 | 0.55 | none | **0.55** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L2 | L1 | 0.30 | none | **0.30** | High |
| C7 | Third-party extensions | L1 | L1 | L2 | L3 | 0.40 | none | **0.40** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L0 | L1 | 0.20 | none | **0.20** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L2 | 0.50 | none | **0.50** | High |
| C10 | Limits & kill switch | L2 | L1 | L1 | L1 | 0.33 | none | **0.33** | Medium |


Coze Studio agents call plugins, workflows and a raw-SQL database tool with no human approval, and nothing limits what injected content can make them do, including sending data to any URL. Workflow code runs in a Deno and Pyodide sandbox by default, but that sandbox lives inside the server container that holds every deployment secret. Credential storage and logging defaults are weak, and registration is open. Read the README's public-network warning and close registration before exposing it.

## Critical gaps
- Untrusted content can drive unattended exfiltration and database writes or deletes in one session. (ASI01, LLM01; C5). Evidence: [backend/domain/agent/singleagent/internal/agentflow/node_tool_database.go:66](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/agent/singleagent/internal/agentflow/node_tool_database.go#L66); [backend/domain/plugin/service/tool/invocation_http.go:141](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/plugin/service/tool/invocation_http.go#L141)
- The code sandbox runs inside the server container and inherits its environment, which holds all deployment secrets. (ASI05, T11; C4). Evidence: [backend/infra/coderunner/impl/script/sandbox.py:126-131](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/infra/coderunner/impl/script/sandbox.py#L126-L131); [docker/docker-compose.yml:392](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/docker/docker-compose.yml#L392)
- The server identity holds deployment-wide credentials for all tenants' data. (ASI03, T3; C1). Evidence: [docker/docker-compose.yml:401](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/docker/docker-compose.yml#L401); [docker/.env.example:18](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/docker/.env.example#L18)

## Criterion details

### C1 Identity & least privilege: 0.20 (high confidence)

The agent acts with the platform server's own authority and with whatever credentials the plugin author attached. Service-token plugins use one static credential for every end user of a published agent; only OAuth plugins act on behalf of the individual user. Resource authorization is a creator-only check, and the README itself lists horizontal privilege escalation in some APIs as a known risk. The server process holds credentials for the shared database, object store and search index of every tenant, so a failure of the authorization layer exposes the whole deployment.

- **S L1:** Plugin tools inject either a per-plugin service token shared by all users of the agent or a per-user OAuth token; platform actions use the server's single database identity. Evidence: [backend/domain/plugin/service/tool/invocation_http.go:164-170](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/plugin/service/tool/invocation_http.go#L164-L170); [docker/.env.example:18](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/docker/.env.example#L18) (verified)
  - *To reach the next level:* No per-capability credential split or per-request authority for service-token plugins and platform data access.
- **C L1:** Resource access is checked against the resource creator, but workflow HTTP nodes and plugins reach any address from the server's network position, outside any authorization layer (SSRF is listed as a known risk in the README). Evidence: [backend/domain/permission/authz_checker.go:102](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/permission/authz_checker.go#L102); [README.md:71](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/README.md#L71); searched `rg -n -i 'ssrf|isprivate|isloopback|169\.254'` in `backend/domain backend/infra backend/pkg` → 0 hits (No internal-address filtering for plugin or workflow HTTP requests.) (verified)
  - *To reach the next level:* Tool-originated outbound requests and data access are not routed through one authorization layer.
- **D L1:** Registration is open by default, so any visitor can create an account and build agents and workflows that run with the server's authority. Evidence: [docker/.env.example:261](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/docker/.env.example#L261) (verified)
  - *To reach the next level:* No minimal default role; open registration and full builder rights are the default.
- **B L0:** The server container receives the whole .env (database, object store, model and plugin encryption secrets) and serves all tenants, so a hijacked identity reaches every tenant's data. Evidence: [docker/docker-compose.yml:392](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/docker/docker-compose.yml#L392); [docker/docker-compose.yml:401](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/docker/docker-compose.yml#L401); [docker/.env.example:18](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/docker/.env.example#L18) (verified)
  - *To reach the next level:* Credentials are deployment-wide and long-lived rather than scoped to one tenant or task.
- **Cap:** none

### C2 Approval gates: 0.00 (high confidence)

There is no human approval step for any tool call. The ReAct agent invokes plugins, workflows (which can contain code and HTTP nodes) and a raw-SQL database tool directly, and the run handler persists and streams the calls but never pauses for a decision. Workflow question and input nodes exist, but they are placed by the author for data entry and do not gate actions. Write and delete operations on external APIs and on user databases happen unattended.

- **S L0:** Tools are handed straight to the ReAct agent with no approval hook. Evidence: [backend/domain/agent/singleagent/internal/agentflow/agent_flow_builder.go:173-176](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/agent/singleagent/internal/agentflow/agent_flow_builder.go#L173-L176); searched `rg -n -i 'approv|confirm'` in `backend/domain/agent backend/domain/conversation/agentrun backend/domain/workflow/internal/nodes/plugin backend/domain/plugin/service/tool` → 0 hits (No approval or confirmation step anywhere in the agent loop, the run handler, or the plugin execution path.) (verified)
  - *To reach the next level:* No per-call human approval of tool calls.
- **C L0:** No tool path, including code-running workflows and the SQL tool, crosses an approval gate. Evidence: [backend/domain/agent/singleagent/internal/agentflow/agent_flow_builder.go:173-176](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/agent/singleagent/internal/agentflow/agent_flow_builder.go#L173-L176); searched `rg -n -i 'approv|confirm'` in `backend/domain/agent backend/domain/conversation/agentrun backend/domain/workflow/internal/nodes/plugin backend/domain/plugin/service/tool` → 0 hits (No approval or confirmation step anywhere in the agent loop, the run handler, or the plugin execution path.) (verified)
  - *To reach the next level:* No gate exists for any tool, including the most powerful ones.
- **D L0:** There is no approval setting to turn on. Evidence: searched `rg -n -i 'approv|confirm'` in `backend/domain/agent backend/domain/conversation/agentrun backend/domain/workflow/internal/nodes/plugin backend/domain/plugin/service/tool` → 0 hits (No approval or confirmation step anywhere in the agent loop, the run handler, or the plugin execution path.) (verified)
  - *To reach the next level:* Approval is not available, on by default or otherwise.
- **B L0:** The model-written SQL tool advertises INSERT, UPDATE and DELETE, and plugins call external write APIs, with no undo. Evidence: [backend/domain/agent/singleagent/internal/agentflow/node_tool_database.go:66](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/agent/singleagent/internal/agentflow/node_tool_database.go#L66); [backend/domain/plugin/service/tool/invocation_http.go:141](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/plugin/service/tool/invocation_http.go#L141) (verified)
  - *To reach the next level:* No checkpoints, previews or rollback for consequential actions.
- **Cap:** none

### C3 Tool & action scoping: 0.50 (high confidence)

Agents only receive the plugins, workflows and databases their author explicitly attaches, which keeps the default tool set empty. Plugin arguments follow typed OpenAPI schemas with required-field checks, and model-written SQL is parsed, rewritten onto the bound table, checked against a denylist and a table-name pattern, and filtered per user in the default read-write mode. Outbound HTTP from plugins and workflow nodes accepts any URL, with no internal-address filtering.

- **S L2:** SQL goes through a parser rewrite plus denylist and pattern checks; plugin parameters are typed; HTTP destinations are unrestricted. Evidence: [backend/domain/memory/database/service/database_impl.go:2210](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/memory/database/service/database_impl.go#L2210); [backend/domain/memory/database/service/database_impl.go:2217](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/memory/database/service/database_impl.go#L2217); [backend/domain/plugin/service/tool/invocation_args.go:307-308](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/plugin/service/tool/invocation_args.go#L307-L308); searched `rg -n -i 'ssrf|isprivate|isloopback|169\.254'` in `backend/domain backend/infra backend/pkg` → 0 hits (No internal-address filtering for plugin or workflow HTTP requests.) (verified)
  - *To reach the next level:* No URL/host allowlist or internal-address blocking, and SQL is filtered by denylist rather than a structural allowlist.
- **C L2:** Plugins and the database tool validate arguments (including read-only table mode); the workflow HTTP node and plugin server URLs are not validated. Evidence: [backend/domain/memory/database/service/database_impl.go:1059](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/memory/database/service/database_impl.go#L1059); [backend/domain/workflow/internal/nodes/httprequester/http_requester.go:368](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/workflow/internal/nodes/httprequester/http_requester.go#L368) (verified)
  - *To reach the next level:* Not every built-in tool validates its inputs; HTTP destinations are unchecked.
- **D L3:** An agent starts with no tools; database, plugin and workflow tools are added only when the author binds them, and the model cannot add tools itself. Evidence: [backend/domain/agent/singleagent/internal/agentflow/agent_flow_builder.go:123](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/agent/singleagent/internal/agentflow/agent_flow_builder.go#L123); [backend/domain/agent/singleagent/internal/agentflow/agent_flow_builder.go:173-176](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/agent/singleagent/internal/agentflow/agent_flow_builder.go#L173-L176) (verified)
  - *To reach the next level:* No per-task narrowing below the author's bound set; once bound, write tools are always offered.
- **B L1:** A misused plugin or HTTP node can reach any host from the server's network, and the SQL tool can write and delete within the bound table. Evidence: [backend/domain/plugin/service/tool/invocation_http.go:51](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/plugin/service/tool/invocation_http.go#L51); [backend/domain/plugin/service/tool/invocation_http.go:141](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/plugin/service/tool/invocation_http.go#L141) (verified)
  - *To reach the next level:* Reach is not limited to approved destinations or bounded in quantity.
- **Cap:** none

### C4 Code-execution isolation: 0.55 (high confidence)

Workflow code nodes run in a sandbox by default: Python executes under Pyodide inside Deno with no network, environment, subprocess or FFI permission, file access limited to a module cache, a 60-second timeout and a 100 MB memory cap, and errors do not fall back to host execution. An operator can switch to a local runner that executes Python directly in the server container, through an environment variable or the admin settings, without a warning. The Deno process runs inside the main server container and inherits its environment, which holds the deployment's secrets, so an escape from the runtime sandbox lands next to every credential.

- **S L3:** The default runner executes code in Pyodide under Deno's permission model with net, env, run and ffi denied and read/write limited to the module directory. Evidence: [backend/infra/coderunner/impl/script/sandbox.py:10](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/infra/coderunner/impl/script/sandbox.py#L10); [backend/infra/coderunner/impl/script/sandbox.py:47-53](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/infra/coderunner/impl/script/sandbox.py#L47-L53) (verified)
  - *To reach the next level:* Not kernel-separated (no microVM, gVisor or remote ephemeral sandbox).
- **C L3:** Every model-reachable code path (workflow code nodes) goes through the configured runner; the only other subprocess runs the bundled document parsers. Evidence: [backend/infra/coderunner/impl/impl.go:53-54](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/infra/coderunner/impl/impl.go#L53-L54); searched `rg -n 'exec\.Command'` in `backend` → 3 hits (Direct runner (opt-in), sandbox runner (default), and the document parser running bundled scripts.) (verified)
  - *To reach the next level:* The sandbox does not cover the documented local-runner escape hatch, and per-process limits outside the runner are absent.
- **D L2:** An empty CODE_RUNNER_TYPE selects the sandbox, but any other value or the admin configuration switches to direct host execution silently. Evidence: [backend/bizpkg/config/base/base.go:69](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/bizpkg/config/base/base.go#L69); [backend/infra/coderunner/impl/impl.go:53-54](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/infra/coderunner/impl/impl.go#L53-L54) (verified)
  - *To reach the next level:* Disabling the sandbox is not an explicit, warned operator flag.
- **B L0:** The Deno process is started without a scrubbed environment inside the server container, whose environment carries all deployment secrets, and network access is the server's. Evidence: [backend/infra/coderunner/impl/script/sandbox.py:126-131](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/infra/coderunner/impl/script/sandbox.py#L126-L131); [docker/docker-compose.yml:392](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/docker/docker-compose.yml#L392) (verified)
  - *To reach the next level:* Credentials are present in the sandbox process environment and the sandbox is not ephemeral or separately networked.
- **Cap:** none

### C5 Untrusted input blast radius: 0.00 (high confidence)

Nothing limits what untrusted content can make the agent do. Published agents take messages from end users through the API and chat SDK, and plugin responses, HTTP results and knowledge documents return to the model as plain tool output with no provenance or taint handling. In the same session the agent can read private databases and knowledge, send data to any URL through plugins, and write or delete records, all without a human. On a multi-user deployment a hijacked agent can also touch data and credentials that other users configured.

- **S L0:** No structural limit on a hijacked agent: no injection controls, no approval after untrusted reads. Evidence: searched `rg -n -i 'prompt.?injection|guardrail|untrusted|jailbreak'` in `backend/domain backend/application backend/crossdomain` → 0 hits (No injection detection, provenance tagging or taint tracking in the backend.) (verified)
  - *To reach the next level:* No capability is disabled or gated once untrusted content enters a session.
- **C L0:** Plugin results are returned verbatim as tool output and are not distinguished from trusted input. Evidence: [backend/domain/agent/singleagent/internal/agentflow/node_tool_plugin.go:147](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/agent/singleagent/internal/agentflow/node_tool_plugin.go#L147); searched `rg -n -i 'prompt.?injection|guardrail|untrusted|jailbreak'` in `backend/domain backend/application backend/crossdomain` → 0 hits (No injection detection, provenance tagging or taint tracking in the backend.) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished from the principal's input.
- **D L0:** No such control exists to enable. Evidence: searched `rg -n -i 'prompt.?injection|guardrail|untrusted|jailbreak'` in `backend/domain backend/application backend/crossdomain` → 0 hits (No injection detection, provenance tagging or taint tracking in the backend.) (verified)
  - *To reach the next level:* No default-on protection.
- **B L0:** Unattended exfiltration through plugins or HTTP nodes and unattended database writes and deletes are both available in one session, on a multi-user platform. Evidence: [backend/domain/agent/singleagent/internal/agentflow/node_tool_database.go:66](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/agent/singleagent/internal/agentflow/node_tool_database.go#L66); [backend/domain/plugin/service/tool/invocation_http.go:141](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/plugin/service/tool/invocation_http.go#L141) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions are not gated by a human.
- **Cap:** C5-WORSTCASE: Untrusted content can drive unattended exfiltration and irreversible database writes in the default configuration.

### C6 Memory, context & configuration integrity: 0.30 (high confidence)

Agents can persist state through user variables and user databases that the model writes with tools, and these are fed back into later conversations and can drive further tool calls. Isolation is per user by default: variables are keyed to the end user and connector, and databases default to a mode that adds a per-user filter to queries. Nothing validates, reviews or expires what the model writes, and there is no provenance or rollback. There are no workspace instruction files or repo configs to auto-load, since this is a hosted platform.

- **S L1:** The variables tool writes model-chosen values straight to persistent per-user variables with no validation or review. Evidence: [backend/domain/agent/singleagent/internal/agentflow/node_tool_variables.go:121](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/agent/singleagent/internal/agentflow/node_tool_variables.go#L121) (verified)
  - *To reach the next level:* Memory writes are not gated, validated or expired.
- **C L1:** Variables and database rows are both written by the model without control; knowledge bases are author-managed. Evidence: [backend/domain/agent/singleagent/internal/agentflow/node_tool_variables.go:121](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/agent/singleagent/internal/agentflow/node_tool_variables.go#L121); [backend/domain/agent/singleagent/internal/agentflow/node_tool_database.go:66](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/agent/singleagent/internal/agentflow/node_tool_database.go#L66) (verified)
  - *To reach the next level:* No memory store has write controls.
- **D L2:** Per-user isolation is the default: databases default to limited (per-user) read-write mode, and variables are keyed to the end user. Evidence: [frontend/packages/studio/stores/bot-detail/src/store/bot-skill/defaults.ts:72](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/frontend/packages/studio/stores/bot-detail/src/store/bot-skill/defaults.ts#L72); [backend/domain/agent/singleagent/internal/agentflow/node_tool_variables.go:47](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/agent/singleagent/internal/agentflow/node_tool_variables.go#L47) (verified)
  - *To reach the next level:* Isolation relies on a query filter in shared tables rather than separate storage, and the author can switch a table to shared mode.
- **B L1:** Poisoned variables or rows persist across the user's sessions and are re-injected into context where they can steer tool calls. Evidence: [backend/domain/agent/singleagent/internal/agentflow/node_tool_variables.go:121](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/agent/singleagent/internal/agentflow/node_tool_variables.go#L121) (verified)
  - *To reach the next level:* Poisoned state is not limited to text output or gated actions.
- **Cap:** none

### C7 Third-party extensions: 0.40 (high confidence)

Extensions are plugins: OpenAPI descriptions of remote HTTP services, either from the operator-curated official catalog or created by users with their own server URL. No third-party code runs inside the platform; MCP invocation is not implemented and the Coze SaaS plugin source is off by default. User-created plugins point at whatever the remote server currently does, with no pinning or change detection, but a malicious plugin only receives the arguments sent to it and its own credentials.

- **S L1:** User-created plugins call a user-chosen server URL whose behaviour can change at any time; nothing is pinned or verified. Evidence: [backend/domain/plugin/service/tool/invocation_http.go:141](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/plugin/service/tool/invocation_http.go#L141); [backend/domain/plugin/service/tool/invocation_mcp.go:32](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/plugin/service/tool/invocation_mcp.go#L32) (verified)
  - *To reach the next level:* No pinning or integrity check of plugin endpoints and definitions.
- **C L1:** Only the official catalog is curated; user plugins are unverified. Evidence: [backend/domain/plugin/service/tool/invocation_http.go:141](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/plugin/service/tool/invocation_http.go#L141) (verified)
  - *To reach the next level:* Verification does not cover user-created plugins.
- **D L2:** No third-party source is enabled by default (the SaaS plugin source is off); users add plugins explicitly. Evidence: [backend/bizpkg/config/base/base.go:81](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/bizpkg/config/base/base.go#L81) (verified)
  - *To reach the next level:* Any registered user can add plugins, and adding one doesn't show a permissions summary.
- **B L3:** Plugins run remotely and receive only the call arguments plus their own injected credential, with no access to the platform process or other plugins' secrets. Evidence: [backend/domain/plugin/service/tool/invocation_http.go:158-170](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/plugin/service/tool/invocation_http.go#L158-L170) (verified)
  - *To reach the next level:* No limit on what data the model can send to a plugin's declared endpoint.
- **Cap:** none

### C8 Secrets & sensitive-data protection: 0.20 (high confidence)

Secrets come from environment variables and the database. Plugin credentials are encrypted at rest, but model provider keys are stored without encryption (a source comment leaves this to the deployer), and credential encryption is not hardened in the default deployment. There is no log redaction, and the shipped environment and Helm values set debug-level logging. Frontend monitoring is a no-op stub, so no telemetry leaves the deployment.

- **S L1:** Plugin credentials and OAuth tokens are encrypted before storage, model connection keys are stored as-is, and no log redaction exists. Evidence: [backend/crossdomain/plugin/model/plugin_manifest.go:84](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/crossdomain/plugin/model/plugin_manifest.go#L84); [backend/bizpkg/config/modelmgr/model_save.go:122-123](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/bizpkg/config/modelmgr/model_save.go#L122-L123) (verified)
  - *To reach the next level:* No masking or redaction on log paths and no protected storage for model keys.
- **C L1:** Only stored plugin credentials get any protection; logs, error messages and model-bound messages have none. Evidence: searched `rg -n -i 'redact|desensitiz|sanitize'` in `backend/domain backend/infra backend/pkg backend/api/middleware` → 6 hits (One hit is a string sanitizer for the OceanBase search store; the other five are the unrelated 'RequiredAction' run type. No log or message redaction.) (verified)
  - *To reach the next level:* Logs and transcripts are not protected.
- **D L0:** Debug-level logging is the shipped default and is not redacted. Evidence: [docker/.env.example:3](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/docker/.env.example#L3); [helm/charts/opencoze/values.yaml:101](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/helm/charts/opencoze/values.yaml#L101) (verified)
  - *To reach the next level:* Logging defaults are verbose and payload logging is not redacted.
- **B L1:** Leaked material is long-lived provider keys and plugin credentials, scoped by whatever the provider issued. Evidence: [docker/.env.example:213](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/docker/.env.example#L213) (verified)
  - *To reach the next level:* Keys are not short-lived or rotated by the platform.
- **Cap:** none

### C9 Audit & traceability: 0.50 (high confidence)

Every agent tool call and tool response is stored as a message with run, conversation, agent and user identifiers, and workflow runs store each node's inputs, outputs, status, duration and errors in the database. This gives a structured, per-action record outside anything the model can edit. There is no approval record (there are no approvals), no tamper-evident storage or standard export, and records are written alongside execution rather than before it.

- **S L2:** Structured records of tool calls and workflow node executions with timestamps, inputs and outputs. Evidence: [docker/volumes/mysql/schema.sql:52](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/docker/volumes/mysql/schema.sql#L52); [docker/volumes/mysql/schema.sql:60](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/docker/volumes/mysql/schema.sql#L60) (verified)
  - *To reach the next level:* No approver attribution, delegation chain or tamper evidence.
- **C L2:** Agent tool calls (plugins, workflows, database, variables) and workflow node executions are recorded. Evidence: [backend/domain/conversation/agentrun/internal/message_event.go:145-148](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/conversation/agentrun/internal/message_event.go#L145-L148) (verified)
  - *To reach the next level:* Configuration changes and credential use are not recorded, and there are no approvals or denials to record.
- **D L2:** Recording is always on and stored in the platform database, which the model cannot reach, but end users can delete their own conversations. Evidence: [backend/domain/conversation/agentrun/internal/message_event.go:145-148](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/conversation/agentrun/internal/message_event.go#L145-L148) (verified)
  - *To reach the next level:* Records can be altered or deleted through normal product features.
- **B L2:** Records are written per action and write failures surface as run errors. Evidence: [backend/domain/conversation/agentrun/internal/message_event.go:148](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/conversation/agentrun/internal/message_event.go#L148) (verified)
  - *To reach the next level:* Actions are not blocked until their record is durably written.
- **Cap:** none

### C10 Limits & kill switch: 0.33 (medium confidence)

The agent loop inherits the agent framework's default step cap because no limit is set, and sandboxed code has a 60-second timeout. Workflows, which agents can call as tools, have no run timeout and no node-count limit by default, and plugin HTTP calls have no timeout. Cancelling an agent run only updates its database status; workflow cancellation is cooperative, checked every 200 ms. There are no token or cost budgets.

- **S L2:** The ReAct agent is built without MaxStep, so the library default step limit applies; code execution has a 60-second timeout. Evidence: [backend/domain/agent/singleagent/internal/agentflow/agent_flow_builder.go:173-176](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/agent/singleagent/internal/agentflow/agent_flow_builder.go#L173-L176); [backend/bizpkg/config/base/base.go:71](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/bizpkg/config/base/base.go#L71) (inferred)
  - *To reach the next level:* No token or cost cap and no run-level time bound.
- **C L1:** Limits apply to the agent loop and code nodes; workflow runs and plugin calls are unbounded. Evidence: [backend/domain/workflow/internal/execute/consts.go:25-28](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/workflow/internal/execute/consts.go#L25-L28); [backend/domain/plugin/service/tool/invocation_http.go:51](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/plugin/service/tool/invocation_http.go#L51) (verified)
  - *To reach the next level:* Tool calls and workflow runs invoked by the agent don't share a time limit.
- **D L1:** Workflow run timeout and node-count limit default to unlimited. Evidence: [backend/domain/workflow/internal/execute/consts.go:25-28](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/workflow/internal/execute/consts.go#L25-L28) (verified)
  - *To reach the next level:* No sensible default limits for workflows.
- **B L1:** Cancelling an agent run only marks the record cancelled, so in-flight work continues. Evidence: [backend/domain/conversation/agentrun/internal/dal/dao.go:197-202](https://github.com/coze-dev/coze-studio/blob/fefb05ff27be1da939612fbf9faf5db62583b8ae/backend/domain/conversation/agentrun/internal/dal/dao.go#L197-L202) (verified)
  - *To reach the next level:* Stopping a run does not cancel in-flight tool calls.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: End-user chat on published agents and plugin/HTTP results returned raw to the model (backend/domain/agent/singleagent/internal/agentflow/node_tool_plugin.go:147) · [B] sensitive data/systems: User databases, knowledge bases and plugin credentials bound to the agent (backend/domain/agent/singleagent/internal/agentflow/agent_flow_builder.go:102-133) · [C] state change / egress: Plugins to arbitrary server URLs and model-written INSERT/UPDATE/DELETE (backend/domain/plugin/service/tool/invocation_http.go:141; node_tool_database.go:66) · Same default session? Yes

## Highest-impact improvements
1. Add an approval hook in the agent's tool node for write-capable plugins, workflows and SQL writes, showing the exact call. (C2 S L0→L3, +0.225 before caps; Playbook 5, step 1)
2. Block internal and link-local addresses (rechecked after redirects) for plugin and workflow HTTP requests. (C3 S L2→L3, +0.075 before caps; Playbook 3, step 1)
3. Start the code sandbox with a scrubbed environment, or run it in a separate container with no secrets. (C4 B L0→L2, +0.100 before caps; Playbook 3, step 2)
4. Ship info-level logging by default and add a redaction filter to the logger. (C8 D L0→L2, +0.100 before caps; Playbook 4, step 2)
5. Set default workflow run timeouts and node limits, a plugin HTTP timeout, and make run cancellation cancel the running context. (C10 C L1→L2, +0.075 before caps; Playbook 3, step 3)

## Re-audit log
- C4 C: L4 → L3. No host fallback exists, but the opt-in local runner is selected silently by any non-sandbox value, so full fail-closed coverage was not credited.
- C9 S: L3 → L2. Records carry run, agent and user identifiers, but there is no approver attribution or delegation chain; burden of proof keeps it at L2.
- C3 D: L2 → L3. Rechecked the builder: an agent with nothing bound gets no tools, and every tool is added by explicit author binding.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The agent step cap is inferred from the default of the cloudwego/eino v0.4.8 ReAct agent (node count plus 10 graph steps), read from that library's source, because coze-studio sets no MaxStep.
- The internals of the jsr:@langchain/pyodide-sandbox package and the Deno version shipped in the image were not examined; the C4 strength rating relies on Deno's documented permission model.
- The chat UI's markdown renderer is an external package and was not examined for remote-image rendering.
- The Helm chart, the OceanBase compose variants and the commercial SaaS plugin source were not scored; the Helm values also default to debug logging.
- The repository's agent-instruction file was not reviewed; no reviewer-steering text was observed in the files read.
