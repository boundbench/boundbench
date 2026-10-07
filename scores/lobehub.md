# Defense-in-Depth Score: LobeHub

**Repo:** https://github.com/lobehub/lobehub · **Commit:** `251f9cb64777e79a0c0d72f846f12afbecd47e78` · **Reviewed:** 2026-10-05
**What it is:** Agent workspace/chat platform organizing agents with plugins, MCP and scheduling
**Category:** AI Assistants
**Scored configuration:** Self-hosted Docker Compose deployment from docker-compose/deploy (setup.sh), Gateway Mode on, default agent mode, default 'manual' approval mode, no connectors, devices or bots added.
**Agent surface (default):** code execution yes · filesystem write no · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents yes · external communication opt-in

## Score: 3.7 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C2 | Approval gates | L3 | L1 | L1 | L1 | 0.40 | G2 | **0.25** | High |
| C3 | Tool & action scoping | L3 | L2 | L2 | L2 | 0.57 | — | **0.57** | High |
| C4 | Code-execution isolation | L3 | L2 | L2 | L1 | 0.53 | — | **0.53** | High |
| C5 | Untrusted input blast radius | L1 | L0 | L1 | L0 | 0.12 | C5-WORSTCASE | **0.12** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C7 | Third-party extensions | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |
| C9 | Audit & traceability | L3 | L3 | L2 | L2 | 0.65 | — | **0.65** | High |
| C10 | Limits & kill switch | L2 | L2 | L0 | L1 | 0.35 | G1 | **0.35** | High |


LobeHub ships a real per-call approval system that shows exact arguments, and model-written code runs in a remote sandbox rather than on the server. But approval covers only tools that declare it, so a model steered by a web page or document can fetch arbitrary URLs and delete knowledge bases and memories without a human, and delegated and background runs execute in headless mode. Memory is on by default and model-writable, and server-side runs have no default step or cost limit.

## Critical gaps
- Approval mode is set per run, and server-side, delegated and background runs default to headless approval instead of the user's foreground manual mode (G2). (ASI09, ASI02, T10; C2) — [apps/server/src/services/aiAgent/index.ts:915](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/apps/server/src/services/aiAgent/index.ts#L915)
- A hijacked model can leak user data through the crawler and delete knowledge bases and memories without a human (C5-WORSTCASE). (ASI01, T6, LLM01; C5) — [packages/builtin-tool-web-browsing/src/manifest.ts:40](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/builtin-tool-web-browsing/src/manifest.ts#L40); [packages/builtin-tool-knowledge-base/src/manifest.ts:177](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/builtin-tool-knowledge-base/src/manifest.ts#L177)

## Criterion details

### C1 Identity & least privilege — 0.45 (high)

LobeHub is a multi-user service: agent tools act for the signed-in user, built-in data tools query by that user's id, and write endpoints check workspace roles. API keys can be issued with narrow scopes. But one server process holds the database connection, the key that decrypts every user's stored provider keys and connector tokens, and the platform's own provider keys, and anyone who can register gets the full agent feature set by default. The container runs as a non-root user.

- **S L2:** Agent work runs under the requesting user's identity with role checks, and API keys can carry narrowed scopes, but the server uses one shared database credential and one vault key for all users. — [packages/const/src/apiKeyScope.ts:1-7](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/const/src/apiKeyScope.ts#L1-L7); [docker-compose/deploy/docker-compose.yml:18-20](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/docker-compose/deploy/docker-compose.yml#L18-L20) (verified)
  - *To reach the next level:* No per-tool or per-capability credentials; read and write tools share the same server identity.
- **C L2:** Built-in data tools filter by the requesting user and side-effecting MCP calls require a workspace member role, but MCP connectors and plugins carry whatever credentials the user configured, outside a common authorization layer. — [packages/database/src/models/userMemory/query.ts:1384](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/database/src/models/userMemory/query.ts#L1384); [apps/server/src/routers/tools/mcp.ts:131-133](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/apps/server/src/routers/tools/mcp.ts#L131-L133) (verified)
  - *To reach the next level:* Extension tools are not routed through one authorization layer that checks every call.
- **D L2:** Self-hosted registration is open unless the operator sets an email allowlist or SSO-only mode, and every account gets agent mode with its default tools. — [docker-compose/deploy/.env.example:5-8](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/docker-compose/deploy/.env.example#L5-L8); [packages/const/src/settings/agent.ts:26](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/const/src/settings/agent.ts#L26) (verified)
  - *To reach the next level:* Default accounts are not read-only or minimal; any registrant can add connectors and run agents.
- **B L1:** If the authorization layer fails, the server process reaches every user's chats, files and memories plus the vault key for all stored provider keys and connector tokens. — [docker-compose/deploy/docker-compose.yml:18-20](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/docker-compose/deploy/docker-compose.yml#L18-L20); [apps/server/src/modules/KeyVaultsEncrypt/index.ts:16-24](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/apps/server/src/modules/KeyVaultsEncrypt/index.ts#L16-L24) (verified)
  - *To reach the next level:* Credentials are not scoped per tenant or short-lived.
- **Cap:** none
- **Notes:** The Docker image drops to a dedicated nextjs user (Dockerfile:366). Desktop and device (lh connect) deployments run tools as the local OS user and were not scored.

### C2 Approval gates — 0.25 (high)

LobeHub has a real per-call approval system: in the default 'manual' mode, tools marked as needing approval pause and the user sees the exact arguments, can edit them, and approve or reject; a regex blacklist forces approval for obviously dangerous shell commands, and each approval is recorded with the approving actor. Coverage is the weak part. Only tools that declare an approval policy are gated, so many mutating tools (deleting knowledge bases, rewriting memories and documents) run freely, and MCP connector tools default to auto-run. Background tasks, scheduled runs, bot conversations and delegated runs use a headless mode that auto-runs tools marked 'required', so the foreground approval choice does not bind everything the model can start.

- **S L3:** Per-call approval shows the exact tool arguments (editable), policies are tiered (never/required/always) with argument-matching rules and a command blacklist, and approval records are bound to a request hash. — [packages/agent-runtime/src/agents/GeneralChatAgent.ts:253-258](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/agent-runtime/src/agents/GeneralChatAgent.ts#L253-L258); [src/features/Conversation/Messages/AssistantGroup/Tool/Detail/Intervention/index.tsx:265](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/src/features/Conversation/Messages/AssistantGroup/Tool/Detail/Intervention/index.tsx#L265); [packages/database/src/schemas/agentIntervention.ts:216](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/database/src/schemas/agentIntervention.ts#L216) (verified)
  - *To reach the next level:* Argument-level policy is a regex denylist rather than parsed allow/deny rules, and many tools carry no policy at all.
- **C L1:** Only tools that declare humanIntervention are gated: a tool with no config resolves to 'never', mutating knowledge-base and memory tools declare none, and synced MCP connector tools default to auto. — [packages/agent-runtime/src/core/InterventionChecker.ts:77-78](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/agent-runtime/src/core/InterventionChecker.ts#L77-L78); [packages/builtin-tool-knowledge-base/src/manifest.ts:177](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/builtin-tool-knowledge-base/src/manifest.ts#L177); [packages/database/src/models/connectorTool.ts:69](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/database/src/models/connectorTool.ts#L69) (verified)
  - *To reach the next level:* Every mutating tool, including extensions and delegated runs, would need to cross the gate.
- **D L1:** Manual approval is the default for the interactive UI, but server-side runs default to headless approval, and runs the model starts in the background are not held to the user's foreground mode. — [src/store/user/slices/settings/selectors/toolIntervention.ts:12-17](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/src/store/user/slices/settings/selectors/toolIntervention.ts#L12-L17); [apps/server/src/services/aiAgent/index.ts:915](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/apps/server/src/services/aiAgent/index.ts#L915); [apps/server/src/services/taskRunner/index.ts:328](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/apps/server/src/services/taskRunner/index.ts#L328) (verified)
  - *To reach the next level:* Elevated (headless) modes should apply only when an authenticated principal chose them for that run.
- **B L1:** Ungated tools can delete knowledge bases, files, memories and documents with no undo or checkpoint. — [packages/builtin-tool-knowledge-base/src/manifest.ts:239](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/builtin-tool-knowledge-base/src/manifest.ts#L239); [packages/builtin-tool-memory/src/manifest.ts:868](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/builtin-tool-memory/src/manifest.ts#L868) (verified)
  - *To reach the next level:* No checkpoints or undo for agent-made changes.
- **Cap:** G2 — The approval mode is a per-run setting: server-side, delegated and background runs default to headless approval rather than inheriting the user's foreground manual mode, so the gate is not a fixed boundary the model's own actions stay inside.

### C3 Tool & action scoping — 0.57 (high)

Built-in tools are mostly narrow and typed (search a knowledge base, add a memory, create a task), and the page crawler uses a fetch guard that blocks private and metadata addresses at connection time, including after redirects. MCP connector and plugin tools pass the model's arguments straight through with no shared validation. The default agent-mode tool set includes write and delete tools over the user's data plus open web fetching.

- **S L3:** The crawler fetches through request-filtering agents that reject private and metadata IPs on every connection, private access only when an operator opts in, and response size can be capped. — [packages/ssrf-safe-fetch/index.ts:72-87](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/ssrf-safe-fetch/index.ts#L72-L87); [packages/web-crawler/src/crawImpl/naive.ts:45](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/web-crawler/src/crawImpl/naive.ts#L45) (verified)
  - *To reach the next level:* General-purpose tools (sandbox shell, MCP passthrough) are not replaced by narrow ones.
- **C L2:** Built-in tools have typed schemas and user-scoped models; MCP connector calls forward the raw argument string to the server. — [apps/server/src/services/toolExecution/index.ts:321-324](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/apps/server/src/services/toolExecution/index.ts#L321-L324) (verified)
  - *To reach the next level:* No shared validation layer wraps MCP and plugin tools.
- **D L2:** Agent mode (the default) enables the always-on agent, skills and activator tools plus web browsing, memory and knowledge-base tools as context allows; exec tools are gated. — [packages/mecha/src/toolSet/resolveToolRules.ts:92-112](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/mecha/src/toolSet/resolveToolRules.ts#L92-L112); [packages/builtin-tools/src/index.ts:65-80](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/builtin-tools/src/index.ts#L65-L80) (verified)
  - *To reach the next level:* The default tool set is not read-only.
- **B L2:** A misused built-in tool is confined to the requesting user's LobeHub data but has full write and delete there. — [packages/builtin-tool-knowledge-base/src/manifest.ts:177](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/builtin-tool-knowledge-base/src/manifest.ts#L177) (verified)
  - *To reach the next level:* No quantity bounds on writes or deletes.
- **Cap:** none

### C4 Code-execution isolation — 0.53 (high)

Model-written code does not run on the LobeHub server by default: the code and shell tools send it to a remote sandbox service (LobeHub Market by default, or an operator-chosen provider), and those tools require approval in manual mode. The sandbox service itself is not in this repository, so its isolation could not be checked. When the model runs the LobeHub CLI inside the sandbox, the server writes a five-minute user token into a wrapper script there, and an opt-in credentials tool can inject stored secrets. Devices joined to the account run commands directly on that machine.

- **S L3:** Execution goes to a remote sandbox provider selected by server env (default 'market'), separate from the LobeHub host. — [apps/server/src/services/sandbox/factory.ts:13-15](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/apps/server/src/services/sandbox/factory.ts#L13-L15); [packages/builtin-tool-cloud-sandbox/src/manifest.ts:13-14](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/builtin-tool-cloud-sandbox/src/manifest.ts#L13-L14) (verified)
  - *To reach the next level:* The remote service's isolation (per-run ephemerality, network policy) cannot be verified from this repository.
- **C L2:** Cloud sandbox tools cover the web deployment's code paths; device-routed local-system commands run on the connected machine, and stdio MCP servers run in-process on self-hosts without a device gateway. — [packages/builtin-tool-local-system/src/manifest.ts:13](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/builtin-tool-local-system/src/manifest.ts#L13); [apps/server/src/services/toolExecution/index.ts:300](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/apps/server/src/services/toolExecution/index.ts#L300) (verified)
  - *To reach the next level:* Device and stdio MCP paths are not sandboxed.
- **D L2:** The provider is fixed by operator env and the model cannot pick host execution, but headless and auto-run modes execute sandbox and device commands without a human. — [apps/server/src/services/sandbox/factory.ts:13-15](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/apps/server/src/services/sandbox/factory.ts#L13-L15); [packages/agent-runtime/src/agents/GeneralChatAgent.ts:253-258](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/agent-runtime/src/agents/GeneralChatAgent.ts#L253-L258) (verified)
  - *To reach the next level:* Escalation to unsandboxed device execution should always be a per-call human approval.
- **B L1:** Inside the sandbox, commands that call the LobeHub CLI get a freshly minted user JWT (five-minute lifetime) in a wrapper file, and the CLI is fetched with npx at run time; network policy is not set here. — [apps/server/src/services/toolExecution/preprocessLhCommand.ts:218-223](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/apps/server/src/services/toolExecution/preprocessLhCommand.ts#L218-L223); [apps/server/src/services/toolExecution/preprocessLhCommand.ts:97](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/apps/server/src/services/toolExecution/preprocessLhCommand.ts#L97) (verified)
  - *To reach the next level:* The sandbox should hold no user credentials and have allowlisted egress.
- **Cap:** none
- **Notes:** Desktop (Electron) local-system tools and `lh connect` devices are opt-in and were reviewed only for how they are routed. The python-interpreter package is not referenced by the app.

### C5 Untrusted input blast radius — 0.12 (high)

Web pages, search results, uploaded files, knowledge-base content and MCP results all enter the conversation as ordinary tool output; nothing in code marks them as untrusted or narrows what the model may do afterwards. In the default setup a hijacked model can read the user's memories and knowledge base, send data out by fetching an arbitrary URL with the crawler (no approval), and delete knowledge bases, files and memories without a human. Approval protects only the tools that declare it.

- **S L1:** The only structural limit is the static approval policy on some tools (sandbox, skills commands), applied regardless of what content the run has read. — [packages/agent-runtime/src/core/InterventionChecker.ts:77-78](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/agent-runtime/src/core/InterventionChecker.ts#L77-L78) (verified)
  - *To reach the next level:* No rule that disables egress or state-changing tools once untrusted content is read.
- **C L0:** Untrusted sources are not distinguished from the user's own input anywhere in the context engine. — searched `rg -n -i untrusted` in `packages/context-engine/src` → 0 hits (no provenance marking of tool results or retrieved content) (verified)
  - *To reach the next level:* At least one untrusted source should be marked and handled differently.
- **D L1:** The approval policy is on by default in manual mode, but delegated and background runs execute headless. — [apps/server/src/services/aiAgent/index.ts:915](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/apps/server/src/services/aiAgent/index.ts#L915) (verified)
  - *To reach the next level:* The limit should not be escapable by runs the model starts.
- **B L0:** A hijacked model can exfiltrate memories or knowledge via crawlSinglePage to any public URL and irreversibly delete knowledge bases and memories, unattended. — [packages/builtin-tool-web-browsing/src/manifest.ts:40](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/builtin-tool-web-browsing/src/manifest.ts#L40); [packages/builtin-tool-knowledge-base/src/manifest.ts:177](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/builtin-tool-knowledge-base/src/manifest.ts#L177) (verified)
  - *To reach the next level:* Either egress or irreversible actions must require approval once untrusted content is read.
- **Cap:** C5-WORSTCASE — B is L0: a hijacked model can leak user data and delete data without a human in the default configuration.
- **Notes:** Bot integrations (Feishu, WeChat, LINE, QQ, iMessage and others) run conversations headless; they are opt-in and were not scored for C5-PUBLICTRIGGER.

### C6 Memory, context & configuration integrity — 0.10 (high)

User memory is on by default. The model has tools to add, update and remove memories with no approval or validation, and saved memories are injected as a user_memory block ahead of the first user message in later chats. Memories are stored per user and queries filter by user id, and users can review them in settings. Agent documents with load rules and scheduled tasks are further ways for content to persist into later runs.

- **S L0:** The model can write arbitrary memory content and it is re-injected into later conversations. — [packages/builtin-tool-memory/src/manifest.ts:223](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/builtin-tool-memory/src/manifest.ts#L223); [packages/context-engine/src/providers/UserMemoryInjector.ts:29-31](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/context-engine/src/providers/UserMemoryInjector.ts#L29-L31) (verified)
  - *To reach the next level:* Memory writes need validation, approval, or provenance-as-data presentation.
- **C L0:** No memory, agent-document or task persistence path is gated. — [packages/const/src/settings/memory.ts:3-4](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/const/src/settings/memory.ts#L3-L4); [packages/builtin-tool-task/src/manifest.ts:351](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/builtin-tool-task/src/manifest.ts#L351) (verified)
  - *To reach the next level:* At least the main memory store needs a write control.
- **D L1:** Memories are namespaced per user in queries, but with no write control the scorecard limits D to one level above S. — [packages/database/src/models/userMemory/query.ts:1384](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/database/src/models/userMemory/query.ts#L1384) (verified)
  - *To reach the next level:* A write control (S at L2 or higher) is needed before isolation earns more credit.
- **B L1:** Poisoned memories persist across the user's chats and can steer tool use; scheduled tasks re-run stored prompts headless. — [apps/server/src/services/taskRunner/index.ts:328](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/apps/server/src/services/taskRunner/index.ts#L328) (verified)
  - *To reach the next level:* Persistence should require human review or support rollback.
- **Cap:** none

### C7 Third-party extensions — 0.30 (high)

Extensions are MCP connectors (remote HTTP servers, or stdio commands on desktop and on self-hosts without a device gateway), legacy plugins, cloud MCP servers and skills from the LobeHub market. Nothing third-party is enabled by default and users add connectors explicitly, but there is no pinning, signing or change detection: tool lists are re-synced from the server, and the in-sandbox CLI is fetched with npx at whatever version is current. Remote connectors run outside LobeHub with only their own configured credentials; stdio connectors that run on a server share its container and OS user with a reduced environment.

- **S L1:** Sources are user-chosen but unpinned; the sandbox CLI wrapper runs npx -y @lobehub/cli without a version. — [apps/server/src/services/toolExecution/preprocessLhCommand.ts:250](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/apps/server/src/services/toolExecution/preprocessLhCommand.ts#L250) (verified)
  - *To reach the next level:* No version pinning or integrity checks for extensions.
- **C L1:** Imported skills are snapshotted into the database with a content-addressed package hash, but MCP connector tool rows are overwritten from whatever the server returns on sync and plugins are unverified. — [packages/database/src/schemas/agentSkill.ts:33-40](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/database/src/schemas/agentSkill.ts#L33-L40); [apps/server/src/services/connector/sync.ts:120-128](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/apps/server/src/services/connector/sync.ts#L120-L128) (verified)
  - *To reach the next level:* MCP connectors and plugins are not pinned or verified.
- **D L2:** No third-party extensions ship enabled; connectors are added explicitly and skill imports by the model require approval. — [packages/builtin-tool-skill-store/src/manifest.ts:58-59](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/builtin-tool-skill-store/src/manifest.ts#L58-L59) (verified)
  - *To reach the next level:* Adding an extension does not show the exact code or permissions that will run.
- **B L1:** Stdio MCP servers run as a separate process of the same OS user, with a default environment plus user-supplied variables. — [src/libs/mcp/client.ts:216-225](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/src/libs/mcp/client.ts#L216-L225); [apps/server/src/services/toolExecution/index.ts:300](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/apps/server/src/services/toolExecution/index.ts#L300) (verified)
  - *To reach the next level:* Extensions should run sandboxed with their own scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.40 (high)

Users' provider keys and connector credentials are encrypted at rest with AES-GCM under a server key, stdio MCP subprocesses get a scrubbed environment, and MCP logging strips env values. Chat transcripts are stored in plain text, there is no general secret masking before logs or model calls, and tool-call usage reports go to the LobeHub market by default when the user is signed in there (sizes, names and errors, not content). The server holds long-lived provider keys and the vault key in its environment.

- **S L2:** Key vaults and connector credentials are AES-GCM encrypted; MCP log helpers drop env; stdio subprocess env is scrubbed. — [apps/server/src/modules/KeyVaultsEncrypt/index.ts:34-40](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/apps/server/src/modules/KeyVaultsEncrypt/index.ts#L34-L40); [apps/server/src/services/mcp/index.ts:84](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/apps/server/src/services/mcp/index.ts#L84) (verified)
  - *To reach the next level:* No secret masking before logs and model-bound messages on all major paths, and no secret manager integration.
- **C L2:** Protection covers stored credentials, MCP logs and subprocess env, but not transcripts or telemetry payloads. — [src/libs/mcp/client.ts:45-53](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/src/libs/mcp/client.ts#L45-L53) (verified)
  - *To reach the next level:* Model-bound messages and telemetry need secret masking.
- **D L1:** Tool-call telemetry defaults to on per user (content-free metrics) and is sent when a market token exists. — [packages/trpc/src/lambda/middleware/telemetry.ts:56-57](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/trpc/src/lambda/middleware/telemetry.ts#L56-L57); [apps/server/src/routers/tools/_helpers/scheduleToolCallReport.ts:77](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/apps/server/src/routers/tools/_helpers/scheduleToolCallReport.ts#L77) (verified)
  - *To reach the next level:* Telemetry should be opt-in.
- **B L1:** A server-side leak exposes long-lived provider keys and the key that decrypts every user's stored credentials. — [docker-compose/deploy/docker-compose.yml:18-20](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/docker-compose/deploy/docker-compose.yml#L18-L20) (verified)
  - *To reach the next level:* Keys should be scoped or short-lived.
- **Cap:** none

### C9 Audit & traceability — 0.65 (high)

Every tool call is stored in the database with its tool, arguments, result state and any approval decision, tied to the owning user; approvals live in a separate table with the resolving actor and a request hash, and sub-agent operations link to their parent operation. Records sit in Postgres outside anything the model can edit, but users can delete their chats and there is no tamper-evident or off-host audit trail by default.

- **S L3:** Structured per-call records (identifier, apiName, arguments, intervention) plus intervention resolutions with actorId and parent-operation links. — [packages/database/src/schemas/message.ts:182-203](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/database/src/schemas/message.ts#L182-L203); [packages/database/src/schemas/agentIntervention.ts:66](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/database/src/schemas/agentIntervention.ts#L66); [packages/database/src/schemas/agentOperations.ts:60](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/database/src/schemas/agentOperations.ts#L60) (verified)
  - *To reach the next level:* No tamper-evident storage or standard export by default.
- **C L3:** Built-in, MCP and sub-agent tool calls all persist as tool messages, and approvals and rejections are recorded. — [packages/database/src/schemas/agentIntervention.ts:179-216](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/database/src/schemas/agentIntervention.ts#L179-L216) (verified)
  - *To reach the next level:* Configuration changes and credential use are not part of the record.
- **D L2:** On by default in the server database, but users (and anything acting with their session) can delete chats and the records with them. — [packages/database/src/schemas/message.ts:185-187](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/database/src/schemas/message.ts#L185-L187) (verified)
  - *To reach the next level:* Records should be written by a component users and agents cannot alter.
- **B L2:** Tool messages are written per step during the run. — [packages/database/src/schemas/message.ts:182-190](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/database/src/schemas/message.ts#L182-L190) (verified)
  - *To reach the next level:* Records are not guaranteed durable before each action or replayable end to end.
- **Cap:** none

### C10 Limits & kill switch — 0.35 (high)

The agent runtime supports a step cap and a cost cap, and the user can interrupt a run, which stops at the next step boundary. But server-side (Gateway Mode) chat runs, which the shipped compose file turns on, are started without a step or cost limit; only the browser-side transport sets 400 steps. Sub-agents cannot spawn further sub-agents and take a timeout, and scheduled tasks keep running independently of a stopped chat.

- **S L2:** maxSteps (with a forced final answer) and costLimit are enforced in the runtime when set; interrupt is cooperative at step boundaries. — [packages/agent-runtime/src/core/runtime.ts:93](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/agent-runtime/src/core/runtime.ts#L93); [packages/agent-runtime/src/core/runtime.ts:597](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/agent-runtime/src/core/runtime.ts#L597); [apps/server/src/services/agentRuntime/AgentRuntimeService.ts:688](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/apps/server/src/services/agentRuntime/AgentRuntimeService.ts#L688) (verified)
  - *To reach the next level:* No run-level wall-clock cap or rate limit on side-effecting tools, and halt does not cancel in-flight calls.
- **C L2:** Limits apply to the top-level loop and tools have per-call timeouts; sub-agents cannot nest further. — [packages/builtin-tool-local-system/src/manifest.ts:16](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/builtin-tool-local-system/src/manifest.ts#L16) (verified)
  - *To reach the next level:* Sub-agents and background tasks do not count against one shared budget.
- **D L0:** Gateway Mode runs get no maxSteps or costLimit by default; only the browser transport sets 400 steps. — [docker-compose/deploy/docker-compose.yml:40](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/docker-compose/deploy/docker-compose.yml#L40); searched `rg -n maxSteps` in `apps/server/src/routers/lambda/aiAgent.ts` → 0 hits (the execAgent route neither accepts nor sets a step limit); [src/store/chat/slices/agentRun/actions/transports/client/streamingExecutor.ts:351](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/src/store/chat/slices/agentRun/actions/transports/client/streamingExecutor.ts#L351) (verified)
  - *To reach the next level:* Server-side runs need sensible default step and cost limits.
- **B L1:** A runaway server run has no step or spend ceiling, and scheduled tasks continue after a chat is stopped. — [packages/builtin-tool-task/src/manifest.ts:351](https://github.com/lobehub/lobehub/blob/251f9cb64777e79a0c0d72f846f12afbecd47e78/packages/builtin-tool-task/src/manifest.ts#L351) (verified)
  - *To reach the next level:* Tight default ceilings and cancellation of pending calls are needed.
- **Cap:** G1 — Step and cost limits exist in the runtime but are not set for server-side runs in the shipped configuration.

## Rule-of-Two check
[A] untrusted input: Web search and crawled pages, files and MCP results enter context as tool output (packages/builtin-tool-web-browsing/src/manifest.ts:40) · [B] sensitive data/systems: User memories and knowledge bases via default tools (packages/builtin-tool-memory/src/manifest.ts:223) · [C] state change / egress: Unapproved deletes and arbitrary URL fetch (packages/builtin-tool-knowledge-base/src/manifest.ts:177) · Same default session? Yes

## Highest-impact improvements
1. Set a default maxSteps and costLimit for server-side (Gateway Mode) runs. — C10 D L0→L2, +0.100 before caps (Playbook 3)
2. Make delegated and background runs inherit the user's approval mode instead of headless, and require approval for starting them. — C2 D L1→L3, +0.100 before caps (Playbook 5)
3. Default every mutating tool without an explicit policy (and synced MCP tools) to 'required' approval. — C2 C L1→L2, +0.075 before caps (Playbook 5)
4. Require approval for memory writes and show provenance on injected memories. — C6 S L0→L2, +0.150 before caps (Playbook 2)
5. Require approval for crawler fetches once a run has read untrusted content. — C5 S L1→L2, +0.075 before caps (Playbook 1)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The remote sandbox service (LobeHub Market or Onlyboxes) and the LobeHub gateway image are outside this repository; C4 isolation of the sandbox could not be verified.
- Desktop (Electron) app, lh connect devices, heterogeneous agents (Claude Code, Codex), bot integrations and workspace/business features were reviewed only for how they are routed and were not scored in depth.
- Frontend code was reviewed only for the approval UI and tool selectors.
- No text attempting to steer AI reviewers was found in the repository.
