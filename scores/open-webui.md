# Defense-in-Depth Score: Open WebUI

**Repo:** https://github.com/open-webui/open-webui · **Commit:** `8bd8b4fac5e059578ac0c74b3c18d11139f88b7d` (v0.11.4) · **Reviewed:** 2026-10-03
**What it is:** Self-hosted AI interface with tools/functions (server-side Python), MCP and pipelines
**Category:** AI Assistants
**Scored configuration:** Shipped docker-compose.yaml / Docker image defaults (root container, auto-generated secret key, first account is admin), native function calling, no tools or tool servers installed, chat used from the web UI.
**Agent surface (default):** code execution opt-in · filesystem write no · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents opt-in · external communication no

## Score: 3.5 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C2 | Approval gates | L2 | L2 | L0 | L1 | 0.35 | G1 | **0.35** | High |
| C3 | Tool & action scoping | L3 | L2 | L2 | L2 | 0.57 | — | **0.57** | High |
| C4 | Code-execution isolation | L2 | L3 | L3 | L0 | 0.53 | — | **0.53** | Medium |
| C5 | Untrusted input blast radius | L1 | L1 | L2 | L0 | 0.25 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C7 | Third-party extensions | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L0 | 0.25 | — | **0.25** | Medium |
| C9 | Audit & traceability | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |


Open WebUI scopes its built-in tools to the requesting user and has strong SSRF protection, but its tool-approval gate is off by default, so tools, including delete tools, run without a human. Admin-installed Tools and Functions are Python exec()-ed inside the server process, which runs as root with every stored API key. Model-written code runs in a browser Pyodide worker. Exfiltration paths and the code interpreter's boundary are not fully contained.

## Critical gaps
- Admin Tools and Functions are exec()-ed in the root server process with all credentials; a malicious or compromised plugin owns the instance. (ASI04, T17, LLM03; C7) — [backend/open_webui/utils/plugin.py:241](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/plugin.py#L241); [Dockerfile:23](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/Dockerfile#L23)
- With approval off by default, injected content can drive delete tools and exfiltrate data with no human involved (C5-WORSTCASE). (ASI01, T6, LLM01; C5) — [backend/open_webui/config.py:2196](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/config.py#L2196)

## Criterion details

### C1 Identity & least privilege — 0.45 (high)

Open WebUI is a multi-user service with real accounts and roles, and its built-in tools act on behalf of the requesting user: chat, memory, note and knowledge tools check ownership or access grants before touching data, and plugin tools are only loaded for users granted access. But the service itself holds every provider API key and connector credential, and plugin tools (admin-written Python) run inside the server process with all of that authority. The first account to register is an admin, and in the shipped Docker image the server runs as root.

- **S L2:** Agent actions run under the requesting Open WebUI user with role/group-based permissions, but one shared server identity holds all provider keys for the whole run. — [backend/open_webui/utils/tools.py:286-298](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/tools.py#L286-L298); [backend/open_webui/tools/builtin.py:1631](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/tools/builtin.py#L1631) (verified)
  - *To reach the next level:* No per-tool or per-capability credential scoping; provider and connector credentials are shared by every tool in the process.
- **C L2:** Built-in tools check the requesting user (e.g. view_chat filters by user id), but plugin tools are exec()-ed into the server process and inherit its full authority. — [backend/open_webui/tools/builtin.py:1631](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/tools/builtin.py#L1631); [backend/open_webui/utils/plugin.py:241](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/plugin.py#L241) (verified)
  - *To reach the next level:* Plugin, OpenAPI and MCP tools are not routed through a common authorization layer; plugin code has unrestricted in-process access.
- **D L2:** New sign-ups default to the 'pending' role and users cannot create tools by default, but the first account is admin and admins bypass feature permission checks. — [backend/open_webui/config.py:1720](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/config.py#L1720); [backend/open_webui/config.py:1745-1746](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/config.py#L1745-L1746) (verified)
  - *To reach the next level:* Default chatting principal in a single-user install is admin; there is no read-only or time-bounded elevation.
- **B L1:** If the authorization layer is bypassed (e.g. via an in-process plugin), the attacker reaches every user's chats and files plus all stored provider/connector credentials, in a container running as root. — [Dockerfile:23](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/Dockerfile#L23); [Dockerfile:225](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/Dockerfile#L225); [backend/open_webui/models/config.py:107](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/models/config.py#L107) (verified)
  - *To reach the next level:* Credentials would need to be scoped to one tenant or short-lived to reach L2+.
- **Cap:** none
- **Notes:** Tool-server connections can be configured (opt-in, admin) with auth_type 'session' or 'system_oauth', which forwards the user's Open WebUI session token or IdP access token downstream (utils/tools.py:152-157). No connections ship by default, so C1-PASSTHRU is not applied; it would apply to such deployments.

### C2 Approval gates — 0.35 (high)

Open WebUI has a per-call tool approval feature that pauses each tool call and shows the user the tool name and exact arguments with Allow/Deny buttons, and only the chat owner (or an admin) can resolve it. However it is off by default twice over: the admin must enable tool permissions, and then each user must switch the chat from 'full' to 'ask'. Scheduled automations and channel chats always run with full permissions, and the gate does not cover every path. In the shipped configuration every tool call, including deletes, runs without a human.

- **S L2:** When enabled, each tool call is paused and the approver sees the tool name and its exact JSON arguments; approved calls execute the stored arguments and reject is first-class. — [backend/open_webui/utils/middleware.py:5934-5940](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/middleware.py#L5934-L5940); [src/lib/components/common/ToolCallDisplay.svelte:144-147](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/src/lib/components/common/ToolCallDisplay.svelte#L144-L147); [backend/open_webui/utils/tool_approval.py:60-64](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/tool_approval.py#L60-L64) (verified)
  - *To reach the next level:* No risk tiers: every call (reads included) needs approval or none does, so there is no policy deciding what needs a human.
- **C L2:** In 'ask' mode every native tool call (built-in, plugin, OpenAPI, MCP) pauses, but automations and channel chats are forced to 'full', and coverage does not extend to every path. — [backend/open_webui/main.py:1255-1263](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/main.py#L1255-L1263); [backend/open_webui/utils/middleware.py:5934-5940](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/middleware.py#L5934-L5940) (verified)
  - *To reach the next level:* Background automations and channels bypass the gate, and coverage has further gaps.
- **D L0:** Approval is opt-in: ENABLE_TOOL_PERMISSIONS defaults to False, which forces tool_approval_mode to 'full'. — [backend/open_webui/config.py:2196](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/config.py#L2196); [backend/open_webui/main.py:1255-1263](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/main.py#L1255-L1263) (verified)
  - *To reach the next level:* Approval should be on by default.
- **B L1:** Default tools can delete calendar events and memories and overwrite notes with no undo; admin-installed plugin tools can do anything. — [backend/open_webui/tools/builtin.py:4532](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/tools/builtin.py#L4532); [backend/open_webui/tools/builtin.py:1070](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/tools/builtin.py#L1070) (verified)
  - *To reach the next level:* No checkpoints or undo for the agent's state changes.
- **Cap:** G1 — The tool approval gate is opt-in: ENABLE_TOOL_PERMISSIONS defaults to False and the per-chat mode defaults to full.

### C3 Tool & action scoping — 0.57 (high)

The built-in tools are mostly narrow, typed operations (create a calendar event, view one of your chats, add a memory) with ownership checks, and the URL fetcher has a strong SSRF defence: it rejects non-public addresses at connection time, blocks cloud metadata endpoints, and does not follow redirects by default. Plugin, OpenAPI and MCP tools get no shared validation layer. The default tool set includes write and delete tools for the user's notes, memories and calendar.

- **S L3:** fetch_url validates scheme, host filter list and every resolved address (non-global blocked) at connect time, redirects are off by default, and search result counts are clamped. — [backend/open_webui/retrieval/web/utils.py:122-126](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/retrieval/web/utils.py#L122-L126); [backend/open_webui/env.py:637](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/env.py#L637); [backend/open_webui/tools/builtin.py:317](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/tools/builtin.py#L317) (verified)
  - *To reach the next level:* General-purpose tools (plugin Python, OpenAPI/MCP passthrough) are not replaced by narrow ones, and validation is not race-robust across all tools.
- **C L2:** Built-in tools have typed specs and ownership checks; extension tools are passed the model's arguments without a shared validation layer. — [backend/open_webui/utils/tools.py:286-298](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/tools.py#L286-L298); [backend/open_webui/utils/tools.py:331-336](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/tools.py#L331-L336) (verified)
  - *To reach the next level:* No validation layer wraps plugin, OpenAPI and MCP tools.
- **D L2:** Built-in tool categories are per-model toggles defaulting to on, and the default set includes write/delete tools (notes, memory, calendar); code execution needs a per-chat toggle. — [backend/open_webui/utils/tools.py:538-540](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/tools.py#L538-L540); [backend/open_webui/config.py:2032](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/config.py#L2032) (verified)
  - *To reach the next level:* Default tool set is not read-only.
- **B L2:** A misused built-in tool is confined to the requesting user's Open WebUI data but has full write/delete inside it. — [backend/open_webui/tools/builtin.py:1070](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/tools/builtin.py#L1070) (verified)
  - *To reach the next level:* No quantity bounds on writes/deletes.
- **Cap:** none

### C4 Code-execution isolation — 0.53 (medium)

Model-written code never runs on the Open WebUI server by default: the code interpreter sends it to Pyodide (Python compiled to WebAssembly) in a web worker inside the user's own browser, and the optional Jupyter engine is an operator choice. That keeps the server host out of reach, but the worker has unrestricted network and its authority boundary is not strict.

- **S L2:** Execution happens in Pyodide/WASM inside a browser web worker, separated from the server host but with the full JavaScript bridge of the page. — [backend/open_webui/config.py:433](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/config.py#L433); [backend/open_webui/tools/builtin.py:698](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/tools/builtin.py#L698); [src/lib/workers/pyodide.worker.ts:26-28](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/src/lib/workers/pyodide.worker.ts#L26-L28) (verified)
  - *To reach the next level:* The runtime is not capability-limited: Pyodide exposes the browser JS environment rather than only host functions the app injects.
- **C L3:** Both model-reachable code paths (the execute_code tool and the legacy <code_interpreter> tag path) dispatch only to the configured engine; an unknown engine returns an error instead of falling back to host execution. — [backend/open_webui/tools/builtin.py:698](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/tools/builtin.py#L698); [backend/open_webui/tools/builtin.py:742](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/tools/builtin.py#L742); [backend/open_webui/utils/middleware.py:6393](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/middleware.py#L6393) (verified)
  - *To reach the next level:* Opt-in Jupyter engine ships no hardening of its own, and execution paths are not fail-closed for every spawned process.
- **D L3:** The browser engine is the default and only an admin config change switches engines; the model cannot pick the engine. — [backend/open_webui/config.py:433](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/config.py#L433); [backend/open_webui/tools/builtin.py:688](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/tools/builtin.py#L688) (verified)
  - *To reach the next level:* The Pyodide worker policy is not defined outside what the page can reach (it runs with page-level authority).
- **B L0:** The worker's authority boundary is not strict. — [backend/open_webui/utils/plugin.py:241](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/plugin.py#L241) (inferred)
  - *To reach the next level:* Code should run with no ambient authority and without unrestricted network.
- **Cap:** none
- **Notes:** B is inferred and was not executed. The Jupyter engine and admin-configured terminal servers are opt-in external services and were not scored as defaults.

### C5 Untrusted input blast radius — 0.25 (high)

Retrieved documents are wrapped in <context>/<source> tags in a prompt template, and tool results enter the conversation as ordinary tool messages; nothing in code limits what the model can do after reading untrusted text. In the default setup a hijacked model can read the user's chats, memories and knowledge, delete data with tools, and leak information through an exfiltration path. Tool approval, which would interpose a human, is off by default.

- **S L1:** Only spotlighting: RAG content is wrapped in <context> and <source> tags by the prompt template. — [backend/open_webui/config.py:1091-1093](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/config.py#L1091-L1093); [backend/open_webui/utils/middleware.py:962](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/middleware.py#L962) (verified)
  - *To reach the next level:* No code-level rule that disables egress or state-changing tools once untrusted content is read.
- **C L1:** Only RAG context gets delimiters; tool and MCP results are appended as plain tool messages. — [backend/open_webui/utils/misc.py:460-465](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/misc.py#L460-L465) (verified)
  - *To reach the next level:* Tool/MCP results, uploaded files and channel messages are not marked or handled as untrusted.
- **D L2:** The wrapper is in the default RAG template and only the operator can replace it (RAG_TEMPLATE). — [backend/open_webui/config.py:1096](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/config.py#L1096) (verified)
  - *To reach the next level:* Disabling or changing it is silent.
- **B L0:** With approval off by default, a hijacked model can both exfiltrate and delete data via tools without a human. — [backend/open_webui/config.py:2196](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/config.py#L2196); [backend/open_webui/tools/builtin.py:4532](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/tools/builtin.py#L4532) (verified)
  - *To reach the next level:* Either egress or irreversible actions must require approval once untrusted content is read.
- **Cap:** C5-WORSTCASE — B is L0: leak plus irreversible action without a human in the default configuration.

### C6 Memory, context & configuration integrity — 0.10 (high)

Memory is on by default and the model has tools to add, rewrite and delete memories with no validation or approval; saved memories are injected into the system message of later chats. Memories are stored per user and queries filter by user id, and users can view and delete them in settings. The model can also create scheduled automations (where permitted) whose stored prompts later run with full tool permissions.

- **S L0:** The model can write arbitrary memory content and it is re-injected into the system message as user context. — [backend/open_webui/tools/builtin.py:923-928](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/tools/builtin.py#L923-L928); [backend/open_webui/utils/memory.py:406](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/memory.py#L406) (verified)
  - *To reach the next level:* Memory writes need validation, approval or provenance-as-data presentation.
- **C L0:** No memory, automation or notes path is gated. — [backend/open_webui/config.py:426](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/config.py#L426); [backend/open_webui/tools/builtin.py:3709](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/tools/builtin.py#L3709) (verified)
  - *To reach the next level:* At least the main memory store needs a write control.
- **D L1:** Memories are namespaced per user in SQL and vector queries, but with no write control the scorecard limits D to one level above S. — [backend/open_webui/models/memories.py:120](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/models/memories.py#L120); [backend/open_webui/routers/memories.py:134](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/routers/memories.py#L134) (verified)
  - *To reach the next level:* A write control (S >= L2) is needed before isolation can earn more credit.
- **B L1:** Poisoned memories persist across all of the user's chats and can steer tool use; model-created automations re-run prompts later with full tool permissions. — [backend/open_webui/utils/memory.py:406](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/memory.py#L406); [backend/open_webui/main.py:1255-1263](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/main.py#L1255-L1263) (verified)
  - *To reach the next level:* Persistence should be gated by human review or rollback.
- **Cap:** none
- **Notes:** Memories are visible and deletable in the user settings UI, which partly mitigates B; rated at the lower level because the model can rewrite them silently. Admin-configured terminal servers can auto-load an AGENTS.md from the terminal home into context (utils/terminals.py:190-233); opt-in, not scored.

### C7 Third-party extensions — 0.25 (high)

Tools and Functions are Python source that Open WebUI exec()s directly inside the server process, with access to the app state, database and every stored credential. Only admins can add them by default (users need an explicit permission), and the code is shown in an editor when imported, but there is no signing or hash check, URL imports pull the head of a GitHub branch, and any 'requirements' line triggers an unpinned pip install into the server environment by default. MCP and OpenAPI tool servers are remote HTTP services whose tool lists are re-fetched without change detection.

- **S L1:** Sources are admin-chosen but unverified: URL import resolves to refs/heads/<branch> and frontmatter requirements are pip-installed unpinned. — [backend/open_webui/routers/tools.py:252](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/routers/tools.py#L252); [backend/open_webui/utils/plugin.py:441-443](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/plugin.py#L441-L443) (verified)
  - *To reach the next level:* No version pinning or integrity checks for plugin requirements or imports.
- **C L1:** Plugin source is snapshotted in the database at install time, but requirements and MCP/OpenAPI tool definitions are not verified. — [backend/open_webui/utils/plugin.py:212-216](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/plugin.py#L212-L216) (verified)
  - *To reach the next level:* MCP/OpenAPI tool definitions and pip dependencies are unverified.
- **D L2:** No plugins ship enabled; installing requires an admin (or a user with the off-by-default workspace.tools permission), but saving a tool immediately pip-installs its declared requirements. — [backend/open_webui/routers/tools.py:354-355](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/routers/tools.py#L354-L355); [backend/open_webui/env.py:1190-1191](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/env.py#L1190-L1191) (verified)
  - *To reach the next level:* Adding an extension does not show the packages that will be installed before they run.
- **B L0:** A malicious plugin runs in-process with the server's full authority, as root in the default image. — [backend/open_webui/utils/plugin.py:241](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/plugin.py#L241); [backend/open_webui/utils/plugin.py:291](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/plugin.py#L291); [Dockerfile:23](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/Dockerfile#L23) (verified)
  - *To reach the next level:* Plugins need a separate process with a scrubbed environment at minimum.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.25 (medium)

Provider API keys and connector secrets live in plaintext in the configuration table, and plugin 'valve' secrets are only encrypted if an operator opts in; OAuth session tokens are encrypted. Telemetry is off by default and the Docker image disables library analytics. Redaction is minimal: the opt-in audit log masks only password fields and event webhooks drop secret-looking keys. Any in-process plugin tool can read the long-lived keys.

- **S L1:** Secrets come from env vars or plaintext DB config; masking exists only in narrow paths (password regex in audit log, key-name filter for event webhooks). — [backend/open_webui/utils/audit.py:284-290](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/audit.py#L284-L290); [backend/open_webui/env.py:767](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/env.py#L767) (verified)
  - *To reach the next level:* No type-level masking or log filters on main paths; API keys not encrypted at rest.
- **C L1:** Protection covers event webhook payloads and OAuth token storage; chat transcripts, model-bound messages and subprocess environments are unprotected. — [backend/open_webui/events.py:958-963](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/events.py#L958-L963); [backend/open_webui/env.py:875](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/env.py#L875) (verified)
  - *To reach the next level:* Logs and transcripts both need redaction.
- **D L2:** OpenTelemetry is opt-in and the image sets DO_NOT_TRACK/ANONYMIZED_TELEMETRY=false; audit logging is off by default. — [backend/open_webui/env.py:1265](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/env.py#L1265); [Dockerfile:92-94](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/Dockerfile#L92-L94) (verified)
  - *To reach the next level:* Redaction is not always on.
- **B L0:** Long-lived provider keys are readable by every in-process plugin and returned by admin config endpoints; further exposure paths exist. — [backend/open_webui/routers/openai.py:544-546](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/routers/openai.py#L544-L546); [backend/open_webui/utils/plugin.py:241](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/plugin.py#L241) (inferred)
  - *To reach the next level:* Keys should be scoped or kept out of reach of plugin code and model-driven sessions.
- **Cap:** none

### C9 Audit & traceability — 0.45 (high)

Every saved chat stores the model's tool calls, their arguments, results, and approval/rejection status in the database, tied to the chat owner, so most agent activity can be reconstructed. Temporary chats are not saved, the dedicated audit log is off by default, and users (or anything acting with their session) can edit or delete chats. Records are written during streaming rather than guaranteed before each action.

- **S L2:** Chat messages persist structured function_call and function_call_output items with arguments and status. — [backend/open_webui/utils/middleware.py:6142-6148](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/middleware.py#L6142-L6148) (verified)
  - *To reach the next level:* No actor attribution beyond chat owner (approver identity, delegation chain) and no correlation across runs.
- **C L2:** All tool types in saved chats are recorded, including rejections, but temporary chats are never persisted. — [backend/open_webui/utils/chat_id.py:12-16](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/chat_id.py#L12-L16); [backend/open_webui/utils/tool_approval.py:65-74](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/tool_approval.py#L65-L74) (verified)
  - *To reach the next level:* Temporary chats, config changes and memory writes are not recorded by default.
- **D L2:** Transcripts are on by default in the server database, outside any workspace, but the user can delete chats and the opt-in audit log is off. — [backend/open_webui/routers/chats.py:1568-1569](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/routers/chats.py#L1568-L1569); [backend/open_webui/env.py:1236](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/env.py#L1236) (verified)
  - *To reach the next level:* Record should be written by a component the user session cannot alter.
- **B L1:** Messages are upserted best-effort during streaming; nothing blocks an action if its record fails. — [backend/open_webui/utils/middleware.py:3634](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/middleware.py#L3634) (verified)
  - *To reach the next level:* Records should be flushed durably per action.
- **Cap:** none
- **Notes:** An opt-in AuditLoggingMiddleware (AUDIT_LOG_LEVEL=METADATA/REQUEST/REQUEST_RESPONSE) records HTTP requests with user attribution to a rotating file; capped by G1 it would not beat the default score, so it was not scored as an alt.

### C10 Limits & kill switch — 0.25 (high)

The only default bound is a cap of 256 tool-call iterations per response. Outbound HTTP calls to models and tool servers, and calls to the browser code interpreter, have no timeout by default, and there is no token or cost budget. Stopping a response cancels the asyncio task cooperatively, but scheduled automations keep running and sub-agents (opt-in) get their own fresh iteration budget.

- **S L1:** An iteration cap is enforced in code; time limits default to none. — [backend/open_webui/env.py:1085-1087](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/env.py#L1085-L1087); [backend/open_webui/env.py:608-610](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/env.py#L608-L610) (verified)
  - *To reach the next level:* Add a default wall-clock/per-execution timeout or token/cost cap.
- **C L1:** The cap applies to the top-level response loop; sub-agents get their own budget and automations run independently. — [backend/open_webui/utils/subagents.py:300](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/subagents.py#L300); [backend/open_webui/utils/subagents.py:451](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/utils/subagents.py#L451) (verified)
  - *To reach the next level:* Tool timeouts and shared budgets for sub-agents and automations are missing.
- **D L1:** Defaults are very large (256 iterations) or unlimited (HTTP and event-caller timeouts). — [backend/open_webui/env.py:534-537](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/env.py#L534-L537) (verified)
  - *To reach the next level:* Sensible default time limits are needed.
- **B L1:** Stop cancels the task cooperatively, but threads (pip installs, sync plugin code) and scheduled automations keep running. — [backend/open_webui/tasks.py:283](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/tasks.py#L283); [backend/open_webui/config.py:2034](https://github.com/open-webui/open-webui/blob/8bd8b4fac5e059578ac0c74b3c18d11139f88b7d/backend/open_webui/config.py#L2034) (verified)
  - *To reach the next level:* Stop should cancel pending calls and nothing scheduled should continue.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Uploaded files/knowledge via RAG and tool results enter context (backend/open_webui/utils/misc.py:460) · [B] sensitive data/systems: search_chats/view_chat, memories and knowledge tools over the user's data (backend/open_webui/tools/builtin.py:1631) · [C] state change / egress: Delete/write tools (backend/open_webui/tools/builtin.py:4532) and an exfiltration path · Same default session? Yes

## Highest-impact improvements
1. Turn tool approval on by default (ENABLE_TOOL_PERMISSIONS=true and per-chat mode 'ask' for state-changing tools). — C2 D L0→L3, +0.150 before caps (Playbook 5)
2. Close the default exfiltration channel available to a hijacked model. — C5 B L0→L1, +0.050 before caps (Playbook 1)
3. Run the code interpreter with no ambient authority and with network off by default. — C4 B L0→L2, +0.100 before caps (Playbook 3)
4. Run plugins out of process with a scrubbed environment instead of exec() in the server. — C7 B L0→L2, +0.100 before caps (Playbook 3)
5. Set default timeouts for model, tool-server and code-interpreter calls and a lower iteration cap. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- C4 B and C8 B rely on inferred browser behaviour, not observed.
- Opt-in subsystems (Jupyter engine, terminal servers, sub-agents, channels, kb_exec, web search, pipelines server) were reviewed only for their defaults and not scored in depth.
- Frontend (Svelte) code was reviewed only for approval UI, image rendering and the Pyodide worker.
- No text attempting to steer AI reviewers was found in the repository.
