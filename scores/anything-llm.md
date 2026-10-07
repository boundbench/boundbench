# Defense-in-Depth Score: AnythingLLM

**Repo:** https://github.com/Mintplex-Labs/anything-llm · **Commit:** `feb04ca0a57cda6d0b3a69c62578f0af44d388fc` · **Reviewed:** 2026-10-05
**What it is:** Local-first all-in-one AI app with agents, MCP tools and RAG
**Category:** AI Assistants
**Scored configuration:** Shipped docker/docker-compose.yml with default .env (single-user mode, no password, telemetry on), default agent skills (memory, document summarizer, web scraping, web search), no MCP servers or imported skills, agent invoked with @agent from the web UI.
**Agent surface (default):** code execution opt-in · filesystem write opt-in · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents no · external communication opt-in

## Score: 2.6 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L1 | L2 | 0.45 | — | **0.45** | High |
| C2 | Approval gates | L3 | L1 | L2 | L1 | 0.45 | C2-POWERBYPASS | **0.25** | High |
| C3 | Tool & action scoping | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C4 | Code-execution isolation | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L0 | 0.05 | — | **0.05** | High |
| C7 | Third-party extensions | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C9 | Audit & traceability | L1 | L2 | L2 | L1 | 0.38 | — | **0.38** | High |
| C10 | Limits & kill switch | L1 | L1 | L2 | L2 | 0.35 | — | **0.35** | High |


AnythingLLM ships a real per-call approval step for its file, email and calendar tools, keeps opt-in filesystem access inside an allowlisted folder, and blocks Community Hub code downloads unless the operator enables them. The default agent can still store anything in the workspace's shared vector memory and fetch arbitrary URLs without asking, so injected content can persist across users and send document contents out. MCP servers and imported skills run with the server's full credentials, and MCP tools are never gated.

## Critical gaps
- Imported agent skills are loaded into the server process and MCP servers inherit the server's full environment, so a malicious extension gets every stored provider key. (ASI04, T17, LLM03; C7) — [server/utils/agents/imported.js:23](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/imported.js#L23); [server/utils/MCP/hypervisor/index.js:311](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/MCP/hypervisor/index.js#L311)
- The single most powerful default actions (memory store and arbitrary-URL fetch) and all MCP tools are outside the approval gate by design (C2-POWERBYPASS). (ASI02, ASI09, T10; C2) — [server/utils/MCP/index.js:94](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/MCP/index.js#L94); [server/utils/agents/aibitat/plugins/memory.js:75](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/plugins/memory.js#L75)

## Criterion details

### C1 Identity & least privilege — 0.45 (high)

The agent runs inside the AnythingLLM server and uses the server's own credentials: every LLM, embedding and search provider key from the instance's .env file, plus any connector it has been given. Built-in tools are scoped to the workspace the chat belongs to, for example document search and memory only touch that workspace's vector namespace. MCP servers are launched with the server's full environment, and imported skills run inside the server process. The shipped Docker setup starts in single-user mode, and until the operator sets a password every request is treated as the instance owner.

- **S L2:** Agent tools act within the invoking workspace (vector searches use the workspace namespace), but one server-wide identity holds every provider key for the whole run. — [server/utils/agents/aibitat/plugins/memory.js:95](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/plugins/memory.js#L95); [docker/docker-compose.yml:19](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/docker/docker-compose.yml#L19) (verified)
  - *To reach the next level:* No per-tool or per-capability credential scoping; read and write tools share the same server credentials.
- **C L2:** Built-in tools use the workspace scope, but MCP servers inherit the server's full environment and imported skills are loaded into the server process. — [server/utils/MCP/hypervisor/index.js:311](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/MCP/hypervisor/index.js#L311); [server/utils/helpers/shell.js:16](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/helpers/shell.js#L16); [server/utils/agents/imported.js:23](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/imported.js#L23) (verified)
  - *To reach the next level:* MCP servers and imported skills do not pass through the same authorization layer as built-in tools.
- **D L1:** The default install is single-user mode, where requests pass without authentication until AUTH_TOKEN and JWT_SECRET are set, so every caller acts as the owner; the default skill set itself is narrow. — [server/utils/middleware/validatedRequest.js:16-20](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/middleware/validatedRequest.js#L16-L20); [docker/docker-compose.yml:25](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/docker/docker-compose.yml#L25); [server/utils/agents/defaults.js:10-15](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/defaults.js#L10-L15) (verified)
  - *To reach the next level:* Multi-user mode with per-user roles is opt-in; the default principal is the instance owner.
- **B L2:** If authorization fails, an attacker reaches every workspace's documents and chats and the long-lived provider keys in the server's .env file. — [docker/docker-compose.yml:19](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/docker/docker-compose.yml#L19); [server/models/systemSettings.js:890](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/models/systemSettings.js#L890) (verified)
  - *To reach the next level:* Credentials are long-lived and instance-wide rather than tenant-scoped or short-lived.
- **Cap:** none
- **Notes:** Opt-in connectors (Gmail, Outlook, Google Calendar, SQL databases) widen the agent's authority to those external accounts; they were not part of the scored default.

### C2 Approval gates — 0.25 (high)

AnythingLLM has a real per-call approval step: built-in tools that write files, create documents, send or draft email, change calendar events or create scheduled jobs pause and show the user the tool's arguments with approve and reject buttons, and API sessions without a human deny such calls. But the gate is opt-in per tool, so the default memory-store tool, the web scraper, the SQL agent, agent flows and every MCP tool run without asking. An operator environment variable can auto-approve any or all skills, users can tick 'always allow' per skill, and scheduled jobs auto-approve everything. There is no undo for agent actions.

- **S L3:** Gated calls pause before execution and the approval card shows the tool's structured arguments (for example path and content); the approved arguments are the ones executed and rejection is a first-class result. — [server/utils/agents/aibitat/plugins/filesystem/write-text-file.js:61-66](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/plugins/filesystem/write-text-file.js#L61-L66); [frontend/src/components/WorkspaceChat/ChatContainer/ChatHistory/ToolApprovalRequest/index.jsx:82](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/frontend/src/components/WorkspaceChat/ChatContainer/ChatHistory/ToolApprovalRequest/index.jsx#L82) (verified)
  - *To reach the next level:* No argument-level allow/deny policy; risk tiering is decided by whether each tool author calls the gate.
- **C L1:** Only tools that call requestToolApproval are gated; the default memory store, web scraper, SQL queries, agent flows and all MCP tools never cross the gate. — [server/utils/agents/aibitat/plugins/filesystem/write-text-file.js:61-66](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/plugins/filesystem/write-text-file.js#L61-L66); [server/utils/MCP/index.js:94](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/MCP/index.js#L94); searched `rg -n "requestToolApproval"` in `server/utils/MCP server/utils/agentFlows server/utils/agents/aibitat/plugins/sql-agent server/utils/agents/aibitat/plugins/web-scraping.js server/utils/agents/aibitat/plugins/memory.js` → 0 hits (No approval call in the MCP tool wrapper, agent flows, SQL agent, web scraper or memory tool.) (verified)
  - *To reach the next level:* Every state-changing or outbound tool, including MCP and flow tools, should go through the gate.
- **D L2:** The gate is on by default for the tools that use it, but AGENT_AUTO_APPROVED_SKILLS (including '<all>') disables it from the environment, per-skill 'always allow' choices accumulate per user, and scheduled jobs auto-approve every call. — [server/utils/agents/aibitat/plugins/websocket.js:224](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/plugins/websocket.js#L224); [server/utils/helpers/agents.js:20](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/helpers/agents.js#L20); [server/utils/agents/aibitat/plugins/websocket.js:239](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/plugins/websocket.js#L239); [server/jobs/run-scheduled-job.js:76](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/jobs/run-scheduled-job.js#L76); [frontend/src/components/WorkspaceChat/ChatContainer/ChatHistory/ToolApprovalRequest/index.jsx:196](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/frontend/src/components/WorkspaceChat/ChatContainer/ChatHistory/ToolApprovalRequest/index.jsx#L196) (verified)
  - *To reach the next level:* Disabling approval is a plain environment setting with no session or time bound, and scheduled runs bypass it entirely.
- **B L1:** No checkpoints or undo exist for agent actions; opt-in tools can send email and MCP tools can make arbitrary changes. — [server/utils/MCP/index.js:94](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/MCP/index.js#L94); [server/utils/agents/aibitat/index.js:1099](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/index.js#L1099) (verified)
  - *To reach the next level:* No rollback for state changes and no previews or quantity bounds for external actions.
- **Cap:** C2-POWERBYPASS — In the default skill set the only state-changing tool (memory store) and the only outbound tool (web scraper) are not gated, and MCP tools never are.

### C3 Tool & action scoping — 0.45 (high)

Tools are typed with JSON schemas, and the opt-in filesystem tools have solid path containment that resolves symlinks against an allowlisted root. The default web scraper accepts any URL; its address check only rejects some private IP literals, allows loopback, ignores hostnames, and is described in its own source as a convenience rather than a security control. The opt-in SQL tool sends raw model-written SQL, relying on its description to keep it read-only. Extension tools get no shared validation layer.

- **S L2:** Validation is mixed: filesystem paths are realpath-contained, but URLs pass a literal-IP denylist that allows 127.0.0.1 and SQL is raw passthrough. — [server/utils/agents/aibitat/plugins/filesystem/lib.js:433](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/plugins/filesystem/lib.js#L433); [collector/utils/url/index.js:16](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/collector/utils/url/index.js#L16); [collector/utils/url/index.js:51](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/collector/utils/url/index.js#L51); [collector/utils/url/index.js:21](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/collector/utils/url/index.js#L21); [server/utils/agents/aibitat/plugins/sql-agent/query.js:81](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/plugins/sql-agent/query.js#L81) (verified)
  - *To reach the next level:* No URL allowlist or resolved-address check; SQL is not parameterized or read-only-enforced.
- **C L2:** Most built-in tools have typed schemas and some checks; MCP and imported-skill arguments pass straight to the extension. — [server/utils/MCP/index.js:94](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/MCP/index.js#L94); [server/utils/agents/aibitat/plugins/sql-agent/query.js:16](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/plugins/sql-agent/query.js#L16) (verified)
  - *To reach the next level:* No shared validation layer wraps extension tools.
- **D L2:** The default skill set is memory (search and store), document summarizer, web scraping and web search; file, SQL, email and calendar tools are opt-in toggles. — [server/utils/agents/defaults.js:10-15](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/defaults.js#L10-L15) (verified)
  - *To reach the next level:* Default set includes a persistent write (memory store) and arbitrary-URL fetches.
- **B L1:** A misused scraper can fetch any host the server can reach, including loopback and the Docker host gateway the compose file maps. — [docker/docker-compose.yml:31](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/docker/docker-compose.yml#L31); [collector/utils/url/index.js:51](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/collector/utils/url/index.js#L51) (verified)
  - *To reach the next level:* No host allowlist or internal-address block bounds the fetch tool.
- **Cap:** none

### C4 Code-execution isolation — 0.10 (high)

AnythingLLM has no shell or code-interpreter tool, and the default skill set never executes model-written code. The one model-driven execution path is the opt-in SQL agent, which sends model-written SQL straight to an operator-configured database with no isolation or read-only enforcement. Imported skills and MCP servers are third-party code and are rated under extensions. The Docker image runs as a non-root user but the shipped compose file adds the SYS_ADMIN capability.

- **S L0:** Model-written SQL runs directly on the configured database connection with no isolation primitive. — [server/utils/agents/aibitat/plugins/sql-agent/query.js:81](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/plugins/sql-agent/query.js#L81) (verified)
  - *To reach the next level:* Queries should run under a read-only role or an enforced read-only session.
- **C L0:** With no isolation layer, the SQL path is not covered. — [server/utils/agents/aibitat/plugins/sql-agent/query.js:81](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/plugins/sql-agent/query.js#L81); [server/utils/agents/aibitat/plugins/sql-agent/query.js:16](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/plugins/sql-agent/query.js#L16) (verified)
  - *To reach the next level:* No execution path passes through an isolation boundary.
- **D L1:** The SQL agent is off by default and enabled by an admin skill toggle, but once on nothing restricts execution. — [server/utils/agents/defaults.js:10-15](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/defaults.js#L10-L15) (verified)
  - *To reach the next level:* No isolation exists to be on by default.
- **B L1:** Model-written SQL reaches everything the configured database account can do, including writes. — [server/utils/agents/aibitat/plugins/sql-agent/query.js:81](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/plugins/sql-agent/query.js#L81) (verified)
  - *To reach the next level:* Database access is not limited to a read-only or narrowly scoped account by the product.
- **Cap:** none
- **Notes:** The container runs as UID 1000 (docker/Dockerfile:57) with cap_add SYS_ADMIN (docker/docker-compose.yml:17). The separate open-computer/ subproject (work in progress) was not scored.

### C5 Untrusted input blast radius — 0.05 (high)

Nothing in the code treats untrusted content differently from the user's own instructions: web pages, uploaded documents and tool or MCP results are fed back to the model as ordinary function results. The default skill set lets the agent read workspace documents and fetch any URL without approval, so a hijacked agent can send private document content out through a URL. Irreversible actions such as sending email are opt-in and gated by approval.

- **S L0:** No structural limit applies once untrusted content is read; tool output enters as plain function messages. — [server/utils/agents/aibitat/index.js:1076](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/index.js#L1076); searched `rg -n -i "untrusted|prompt injection"` in `server/utils/agents server/utils/chats` → 0 hits (No marking or handling of untrusted content anywhere in the agent or chat code.) (verified)
  - *To reach the next level:* No rule disables egress or state-changing tools after untrusted content enters the session.
- **C L0:** Untrusted sources are not distinguished from user input. — [server/utils/agents/aibitat/index.js:1076](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/index.js#L1076); searched `rg -n -i "untrusted|prompt injection"` in `server/utils/agents server/utils/chats` → 0 hits (No marking or handling of untrusted content anywhere in the agent or chat code.) (verified)
  - *To reach the next level:* At least web and document content should be marked and handled as untrusted.
- **D L0:** There is no control to enable. — searched `rg -n -i "untrusted|prompt injection"` in `server/utils/agents server/utils/chats` → 0 hits (No marking or handling of untrusted content anywhere in the agent or chat code.) (verified)
  - *To reach the next level:* A taint-aware control would need to exist and be on by default.
- **B L1:** With default skills, a hijacked agent can read workspace documents and leak them through the arbitrary-URL scraper unattended; irreversible actions are opt-in and approval-gated. — [server/utils/agents/defaults.js:10-15](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/defaults.js#L10-L15); [server/utils/agents/aibitat/plugins/memory.js:95](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/plugins/memory.js#L95); [server/utils/agents/aibitat/plugins/filesystem/write-text-file.js:61-66](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/plugins/filesystem/write-text-file.js#L61-L66) (verified)
  - *To reach the next level:* Outbound fetches after untrusted input should require approval or be blocked.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.05 (high)

The default memory tool lets the model store any text into the workspace's vector database with no approval or validation. That text is stored like a document chunk, without a workspace document record, and is then retrieved into later agent sessions and into ordinary chats for every user of the workspace. The separate per-user memories feature is off by default. Isolation is per workspace, not per user.

- **S L0:** The model can write arbitrary text into the workspace vector store and it is later retrieved as context. — [server/utils/agents/aibitat/plugins/memory.js:75](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/plugins/memory.js#L75); [server/utils/agents/aibitat/plugins/memory.js:129-134](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/plugins/memory.js#L129-L134) (verified)
  - *To reach the next level:* Memory writes need approval, validation or provenance-as-data presentation.
- **C L0:** No persistent store written by the agent is controlled. — [server/utils/agents/aibitat/plugins/memory.js:129-134](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/plugins/memory.js#L129-L134); [server/utils/chats/stream.js:197](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/chats/stream.js#L197) (verified)
  - *To reach the next level:* At least the main memory store needs a write control.
- **D L1:** Stored memory is namespaced by workspace and enforced in queries, but shared by all users of the workspace. — [server/utils/agents/aibitat/plugins/memory.js:95](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/plugins/memory.js#L95); [server/utils/chats/stream.js:197](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/chats/stream.js#L197) (verified)
  - *To reach the next level:* No per-user namespace for agent-stored memory.
- **B L0:** Poisoned memory persists across sessions and across the workspace's users, reaches agent sessions that can call tools, and is not listed as a document in the workspace UI. — [server/utils/chats/stream.js:197](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/chats/stream.js#L197); searched `rg -n "Document.create|workspace_documents"` in `server/utils/vectorDbProviders` → 0 hits (Vector providers write vectors only; agent-stored memory gets no workspace document record.) (verified)
  - *To reach the next level:* Persistence should require human review or be easily listed and purged.
- **Cap:** none

### C7 Third-party extensions — 0.25 (high)

Nothing third-party is enabled by default, and only admins can add extensions. Community Hub downloads are disabled unless the operator sets an environment variable, are then limited to items the hub marks verified, are shown for code review, and arrive inactive. But imported skills are loaded with require() into the server process with all its credentials, there is no hash or signature check on what is downloaded, and MCP servers are user-chosen commands started with the server's full environment.

- **S L1:** MCP servers run whatever command the admin configures, unpinned; hub skills are restricted to verified items but the downloaded archive is not hash- or signature-checked. — [server/utils/MCP/hypervisor/index.js:409](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/MCP/hypervisor/index.js#L409); [server/utils/middleware/communityHubDownloadsEnabled.js:34](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/middleware/communityHubDownloadsEnabled.js#L34) (verified)
  - *To reach the next level:* Extensions should be pinned, and downloads checked against a hash or signature.
- **C L1:** Only Community Hub skills pass a verification filter; MCP servers and skills placed directly in storage are not verified. — [server/utils/middleware/communityHubDownloadsEnabled.js:34](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/middleware/communityHubDownloadsEnabled.js#L34); [server/utils/MCP/hypervisor/index.js:409](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/MCP/hypervisor/index.js#L409) (verified)
  - *To reach the next level:* MCP servers and manually installed skills are not covered by any verification.
- **D L2:** Hub downloads are off by default and admin-only, the review step shows the skill code, and imports arrive inactive; MCP servers are added only by an admin-controlled config file. — [server/utils/middleware/communityHubDownloadsEnabled.js:23](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/middleware/communityHubDownloadsEnabled.js#L23); [server/utils/agents/imported.js:345](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/imported.js#L345); [frontend/src/pages/GeneralSettings/CommunityHub/ImportItem/Steps/PullAndReview/HubItem/AgentSkill.jsx:50](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/frontend/src/pages/GeneralSettings/CommunityHub/ImportItem/Steps/PullAndReview/HubItem/AgentSkill.jsx#L50); [server/endpoints/mcpServers.js:14](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/endpoints/mcpServers.js#L14) (verified)
  - *To reach the next level:* No change detection or re-approval when an extension's code or tools change; D limited to one level above S.
- **B L0:** Imported skills run in-process with every credential, and MCP servers get the server's full environment. — [server/utils/agents/imported.js:23](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/imported.js#L23); [server/utils/MCP/hypervisor/index.js:311](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/MCP/hypervisor/index.js#L311); [server/utils/helpers/shell.js:16](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/helpers/shell.js#L16) (verified)
  - *To reach the next level:* Extensions should run in a separate process with a scrubbed environment.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.25 (high)

Provider and connector keys live in the server's plaintext .env file, which the compose file mounts into the container. The settings API returns only whether a key is set, not its value. Agent tool calls are logged to the console with their full arguments, nothing masks secrets in logs or model-bound content, and anonymous telemetry, which includes tool names, is on by default. MCP servers inherit every key in the environment when enabled.

- **S L1:** Secrets come from a plaintext .env file; the settings API masks keys as booleans. — [docker/docker-compose.yml:19](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/docker/docker-compose.yml#L19); [server/models/systemSettings.js:890](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/models/systemSettings.js#L890) (verified)
  - *To reach the next level:* No masking of secrets in logs or model-bound messages; no encryption at rest for provider keys.
- **C L1:** Masking covers the settings API only; logs print full tool arguments and subprocess environments are not scrubbed. — [server/utils/agents/aibitat/index.js:1096](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/index.js#L1096); [server/utils/MCP/hypervisor/index.js:311](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/MCP/hypervisor/index.js#L311) (verified)
  - *To reach the next level:* Logs, transcripts and subprocess environments should be covered.
- **D L1:** Telemetry is on unless DISABLE_TELEMETRY is set and reports events such as tool names, but not prompt content. — [server/utils/telemetry/index.js:9](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/telemetry/index.js#L9); [server/utils/telemetry/index.js:24](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/telemetry/index.js#L24); [server/utils/agents/aibitat/index.js:1100](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/index.js#L1100) (verified)
  - *To reach the next level:* Telemetry should be opt-in.
- **B L1:** Leaked keys are long-lived provider and connector keys of moderate scope; enabled MCP servers receive all of them. — [docker/docker-compose.yml:19](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/docker/docker-compose.yml#L19); [server/utils/MCP/hypervisor/index.js:311](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/MCP/hypervisor/index.js#L311) (verified)
  - *To reach the next level:* Keys should be scoped and short-lived, and kept out of subprocess environments.
- **Cap:** none

### C9 Audit & traceability — 0.38 (high)

Every tool call the agent makes, including MCP and imported-skill calls, is printed to the server console with its arguments, and the chat record stores the prompt, final reply and requesting user. Neither is a structured audit trail: tool calls are not saved with the chat, approvals and rejections are not durably recorded, and agent activity is not written to the instance's event log. Scheduled jobs are the exception and keep a per-run trace of tool calls.

- **S L1:** Tool calls are unstructured console lines; the stored chat record has user attribution but no tool calls. — [server/utils/agents/aibitat/index.js:1096](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/index.js#L1096); [server/utils/agents/index.js:47](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/index.js#L47); [server/utils/agents/aibitat/plugins/chat-history.js:119](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/plugins/chat-history.js#L119) (verified)
  - *To reach the next level:* No structured per-call record with arguments, result status and timestamps for interactive sessions.
- **C L2:** The console line is emitted on the shared execution path for all tools, including MCP and imported skills; approvals and denials are not recorded. — [server/utils/agents/aibitat/index.js:1096](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/index.js#L1096); [server/utils/agents/aibitat/index.js:1099](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/index.js#L1099); searched `rg -n "EventLogs"` in `server/utils/agents` → 0 hits (Agent code writes nothing to the instance's event log.) (verified)
  - *To reach the next level:* Approvals and denials, and memory writes, are not recorded.
- **D L2:** Logging is on by default to stdout, outside anything the agent's tools write, but nothing protects or retains it. — [server/utils/agents/index.js:47](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/index.js#L47) (verified)
  - *To reach the next level:* Records are not written by a component separated from the agent process.
- **B L1:** Console logging is best-effort and is lost with container logs; actions proceed regardless. — [server/utils/agents/index.js:47](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/index.js#L47) (verified)
  - *To reach the next level:* Records should be durable per action.
- **Cap:** none

### C10 Limits & kill switch — 0.35 (high)

Each agent response may chain at most 10 tool calls by default (configurable), after which the model must answer. Stopping the session or closing the socket aborts in-flight LLM requests and stops further turns, but running tool calls are not cancelled. There is no token or spend budget and no wall-clock limit on interactive sessions; scheduled jobs have a five-minute default timeout.

- **S L1:** An iteration cap on chained tool calls per response is enforced in code; abort cancels LLM requests; no token, cost or interactive time limit. — [server/utils/agents/aibitat/index.js:92](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/index.js#L92); [server/utils/agents/aibitat/index.js:1059](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/index.js#L1059); [server/utils/agents/aibitat/index.js:434](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/index.js#L434); searched `rg -n -i "maxCost|max_cost|costLimit|tokenBudget|max_tokens_budget"` in `server/utils/agents` → 0 hits (No token or spend budget in the agent runtime.) (verified)
  - *To reach the next level:* Add a token/cost cap or per-tool and per-run timeouts.
- **C L1:** The cap applies to the top-level agent loop; tool calls have no timeouts. — [server/utils/agents/aibitat/index.js:1059](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/index.js#L1059) (verified)
  - *To reach the next level:* Tool timeouts should be enforced on every tool.
- **D L2:** The default cap is 10 calls per response, set by the operator through AGENT_MAX_TOOL_CALLS; the model cannot raise it. — [server/utils/agents/aibitat/index.js:92](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/index.js#L92) (verified)
  - *To reach the next level:* No hard ceiling that configuration cannot exceed, and D limited to one level above S.
- **B L2:** Ceilings are moderate per response; stopping ends the loop and LLM requests but in-flight tools run to completion, and scheduled jobs keep running on their schedule. — [server/utils/agents/aibitat/index.js:434](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/utils/agents/aibitat/index.js#L434); [server/jobs/helpers/scheduled-job-helper.js:9](https://github.com/Mintplex-Labs/anything-llm/blob/feb04ca0a57cda6d0b3a69c62578f0af44d388fc/server/jobs/helpers/scheduled-job-helper.js#L9) (verified)
  - *To reach the next level:* No per-run spend ceiling and no cancellation of in-flight tools.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: web pages and uploaded documents via the default web-scraping and summarizer skills (server/utils/agents/defaults.js:10-15) · [B] sensitive data/systems: workspace documents and stored memory searched by rag-memory (server/utils/agents/aibitat/plugins/memory.js:95) · [C] state change / egress: arbitrary-URL fetches (collector/utils/url/index.js:51) and memory writes (server/utils/agents/aibitat/plugins/memory.js:129) · Same default session? Yes

## Highest-impact improvements
1. Call requestToolApproval in the shared MCP tool wrapper, the memory store action, the SQL query tool and agent-flow tools. — C2 C L1→L3, +0.150 before caps (Playbook 5)
2. Start MCP servers with only PATH plus the server's own configured env instead of the full process environment. — C7 B L0→L2, +0.100 before caps (Playbook 3)
3. Require approval for memory stores and register agent-stored memories as listed, deletable workspace documents with provenance. — C6 S L0→L2, +0.150 before caps (Playbook 2)
4. Resolve hostnames and block loopback, private and link-local addresses for agent-initiated fetches. — C3 S L2→L3, +0.075 before caps (Playbook 3)
5. Persist each tool call, its arguments, result status and the approval decision with the chat record. — C9 S L1→L2, +0.075 before caps (Playbook 1)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The pinned commit is 15 commits after the v1.17.0 tag, so no version is recorded.
- The open-computer/ subproject (a separate work-in-progress agent OS), the desktop app build, the browser extension, the embed widget and the Telegram bot were not scored in depth.
- Opt-in connectors (Gmail, Outlook, Google Calendar, SQL) were read only to confirm their approval calls; their OAuth scopes were not reviewed.
