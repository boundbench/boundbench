# Defense-in-Depth Score: LibreChat

**Repo:** https://github.com/LibreChat-AI/LibreChat · **Commit:** `f10b1d91f1eee3a2c82d5247bf620351486b7c1b` · **Reviewed:** 2026-10-05
**What it is:** Self-hosted multi-model chat platform with agents, MCP, code interpreter and actions
**Category:** AI Assistants
**Scored configuration:** Shipped docker-compose.yml with .env.example values and no librechat.yaml overrides: default agent capabilities, tool approval and memory unset, no MCP servers, first registered account is admin, chats used from the web UI.
**Agent surface (default):** code execution opt-in · filesystem write no · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents yes · external communication opt-in

## Score: 4.0 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C2 | Approval gates | L3 | L2 | L0 | L1 | 0.42 | G1 | **0.42** | Medium |
| C3 | Tool & action scoping | L3 | L2 | L2 | L1 | 0.53 | — | **0.53** | High |
| C4 | Code-execution isolation | L2 | L3 | L3 | L2 | 0.62 | — | **0.62** | Medium |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L2 | L2 | 0.35 | — | **0.35** | High |
| C7 | Third-party extensions | L1 | L0 | L2 | L2 | 0.28 | — | **0.28** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | Medium |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |


LibreChat is a well-structured multi-user service: requests are tied to accounts and roles, outbound actions get SSRF protection, model-written code runs in a separate remote service rather than on the server, and stdio MCP servers can only be added by admins. The dominant risk is that its human-approval gate for tool calls is off by default, so content an agent reads (files, search results, tool outputs) can drive any tool it holds, including outbound HTTP actions and MCP write tools, with nobody approving. Token spending limits are also off by default, and the server process holds every user's stored credentials.

## Critical gaps
- With tool approval off by default and no provenance controls, injected content can make an agent send data out and take external write actions through actions or MCP tools with no human involved (C5-WORSTCASE). (ASI01, T6, LLM01; C5) — [packages/api/src/agents/hitl/policy.ts:97-101](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/agents/hitl/policy.ts#L97-L101); [packages/data-provider/src/config.ts:830-847](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-provider/src/config.ts#L830-L847)

## Criterion details

### C1 Identity & least privilege — 0.45 (high)

LibreChat is a multi-user service: every request is tied to a logged-in account, role permissions are checked on each feature, and model provider keys default to 'user provided', so each user supplies and pays for their own. Per-user OAuth tokens are kept for actions and MCP servers, and stdio MCP servers get a stripped environment. But the server process itself still holds every stored credential (encrypted user keys, action secrets, the database connection), and an action's stored API key is used for whoever runs the agent. The first account to register becomes administrator.

- **S L2:** Requests run under the requesting LibreChat user with role-based permission checks, and provider keys default to per-user keys, but the server process holds every stored credential for the whole run. — [packages/api/src/middleware/access.ts:137-160](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/middleware/access.ts#L137-L160); [.env.example:597](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/.env.example#L597); [api/server/services/ActionService.js:214](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/api/server/services/ActionService.js#L214) (verified)
  - *To reach the next level:* No per-tool or per-capability credential narrowing; on-behalf-of token exchange exists only as an opt-in for MCP servers.
- **C L2:** Built-in tools, actions and MCP connections resolve credentials per user, and stdio MCP servers get only a default environment, but an action's stored API key is applied to every user who runs the agent rather than being re-authorized per requester. — [api/server/services/ActionService.js:455](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/api/server/services/ActionService.js#L455); [packages/api/src/mcp/connection.ts:1502](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/mcp/connection.ts#L1502) (verified)
  - *To reach the next level:* No common authorization layer checks each tool call against the requesting principal; shared-agent actions reuse the author's stored credential.
- **D L2:** Ordinary users cannot create MCP servers or share agents by default, but the first registered account is an administrator with every permission. — [api/server/services/AuthService.js:426](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/api/server/services/AuthService.js#L426); [packages/data-provider/src/roles.ts:256-262](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-provider/src/roles.ts#L256-L262); [packages/data-provider/src/roles.ts:236-241](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-provider/src/roles.ts#L236-L241) (verified)
  - *To reach the next level:* Default chatting principal in a single-user install is admin, and widening user permissions is an unlogged admin change.
- **B L1:** If authorization fails inside the server process, everything it holds is reachable: all users' encrypted keys and the decryption key, connected OAuth tokens and action secrets, and an unauthenticated MongoDB on the internal network. — [docker-compose.yml:63](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/docker-compose.yml#L63); [packages/data-schemas/src/crypto/index.ts:115-128](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-schemas/src/crypto/index.ts#L115-L128) (verified)
  - *To reach the next level:* Credentials are long-lived and span multiple external systems per user; nothing confines a compromise to one tenant.
- **Cap:** none

### C2 Approval gates — 0.42 (medium)

LibreChat ships a real human-approval gate for agent tool calls: when enabled, a paused call shows the tool name and its exact arguments, and the user can approve, reject, or edit the arguments; administrators can write allow, deny and ask rules by tool-name pattern. But the gate is off unless an administrator adds a toolApproval block to librechat.yaml, and the shipped example has it commented out. In the default deployment every tool an agent has, including actions that call external APIs and MCP tools, runs with no human in the loop.

- **S L3:** When enabled, each paused call presents the exact tool name and arguments with approve, reject and edit decisions, and policy rules decide by tool-name pattern which calls need a human. — [packages/api/src/agents/hitl/policy.ts:298-316](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/agents/hitl/policy.ts#L298-L316); [packages/api/src/agents/hitl/policy.ts:14](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/agents/hitl/policy.ts#L14); [client/src/components/Chat/Messages/Content/ToolApproval.tsx:51-66](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/client/src/components/Chat/Messages/Content/ToolApproval.tsx#L51-L66) (verified)
  - *To reach the next level:* No built-in argument-level policy; per-argument decisions require writing a custom hook module.
- **C L2:** The policy is applied by tool name to every tool in the run, including MCP and action tools, but the interception itself happens inside the external agents SDK package, so coverage of every path could not be confirmed from this repository. — [packages/api/src/agents/hitl/policy.ts:65-79](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/agents/hitl/policy.ts#L65-L79) (inferred)
  - *To reach the next level:* Coverage of sub-agent and background tool paths is enforced in an external package and is not verifiable here.
- **D L0:** Approval only runs when endpoints.agents.toolApproval.enabled is true; the example config ships it commented out. — [packages/api/src/agents/hitl/policy.ts:97-101](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/agents/hitl/policy.ts#L97-L101); [librechat.example.yaml:773-778](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/librechat.example.yaml#L773-L778) (verified)
  - *To reach the next level:* Approval is opt-in.
- **B L1:** Default agents can call user-defined HTTP actions against any public host and any connected MCP tool, with no undo for external effects. — [packages/data-provider/src/config.ts:830-847](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-provider/src/config.ts#L830-L847); [api/server/services/ToolService.js:667](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/api/server/services/ToolService.js#L667) (verified)
  - *To reach the next level:* No previews, dry-runs or rate limits on consequential actions.
- **Cap:** G1 — The tool-approval gate is opt-in through librechat.yaml and off in the shipped configuration.

### C3 Tool & action scoping — 0.53 (high)

Outbound HTTP from actions and user-added MCP servers goes through a domain check, and when no admin allowlist is set, actions use connection-time SSRF protection that blocks private and metadata addresses. Users cannot add command-launching (stdio) MCP servers; only admins can, in librechat.yaml. Tools have typed schemas. But actions are deliberately general: an agent author can point one at any public API with any operations, and the default capability list turns on actions, code execution, web search, sub-agents and the rest of the tool families at once.

- **S L3:** Action and MCP URLs are checked against admin allowlists or, without one, against private/loopback/metadata ranges, with SSRF-safe agents that re-check resolved addresses at connect time. — [packages/api/src/auth/domain.ts:312-324](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/auth/domain.ts#L312-L324); [api/server/services/ActionService.js:199](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/api/server/services/ActionService.js#L199); [api/server/services/ToolService.js:667](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/api/server/services/ToolService.js#L667) (verified)
  - *To reach the next level:* General-purpose HTTP actions are not replaced by narrow tools; arguments are schema-typed, not allowlisted.
- **C L2:** Actions, user-added MCP servers, provider endpoints and media fetches share the SSRF and domain helpers, but arguments passed to MCP tools are forwarded unvalidated to the server. — [packages/data-provider/src/mcp.ts:534-557](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-provider/src/mcp.ts#L534-L557); [api/server/services/ToolService.js:667](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/api/server/services/ToolService.js#L667) (verified)
  - *To reach the next level:* No shared validation layer wraps extension (MCP) tool arguments.
- **D L2:** Admins can trim agent capabilities and authors choose tools per agent, but the default capability list enables actions, code execution, web search, sub-agents and tools together. — [packages/data-provider/src/config.ts:830-847](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-provider/src/config.ts#L830-L847); [packages/data-provider/src/config.ts:830-847](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-provider/src/config.ts#L830-L847) (verified)
  - *To reach the next level:* Default capability set is not read-only; write and egress tool families are on unless removed.
- **B L1:** A misused action can call any public host and any operation in the author's OpenAPI spec with the stored credentials. — [api/server/services/ActionService.js:455](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/api/server/services/ActionService.js#L455); [packages/api/src/auth/domain.ts:312-324](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/auth/domain.ts#L312-L324) (verified)
  - *To reach the next level:* Actions have no quantity or operation bounds beyond the spec.
- **Cap:** none

### C4 Code-execution isolation — 0.62 (medium)

LibreChat does not run model-written code on its own server. The code-execution and bash tools send code over HTTP to a separate Code Interpreter API service (configured by base URL and an API key), and a separate opt-in mode routes commands to workers the user enrolls on their own machine. The only local process launches in the server are operator plugin hooks and document conversion, both with a stripped environment. The isolation of the remote service lives outside this repository and could not be examined, so its strength and blast radius are inferred.

- **S L2:** Model code runs in a remote code-execution service reached over HTTP rather than on the LibreChat host; that service's isolation is not in this repository. — [packages/api/src/agents/execution.ts:260-278](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/agents/execution.ts#L260-L278); [packages/api/src/files/provision/service.ts:207](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/files/provision/service.ts#L207) (inferred)
  - *To reach the next level:* The sandbox implementation is outside the scored repository, so kernel-level isolation cannot be verified.
- **C L3:** Every model-reachable execution tool (execute_code, bash_tool, file authoring) is built against the remote code API or an enrolled worker; no host execution path exists for model code. — [api/server/services/ToolService.js:2305-2331](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/api/server/services/ToolService.js#L2305-L2331); searched `rg -n child_process -g '!*.spec.*' -g '!*.test.*' -g '!**/__tests__/**'` in `packages/api/src api/server api/app` → 5 hits (Hits are build-time git (app/build.ts), the operator plugin-hook runner and its reaper (agents/hooks, stripped env), and LibreOffice document conversion (stripped env); none executes model-written code on the host.) (verified)
  - *To reach the next level:* No fail-closed guarantee is verifiable for processes the remote service spawns.
- **D L3:** The execution endpoint and key come from operator environment variables; there is no local fallback and the model cannot select host execution. — [packages/api/src/agents/execution.ts:260-278](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/agents/execution.ts#L260-L278); [packages/api/src/agents/hooks/executor.ts:29](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/agents/hooks/executor.ts#L29) (verified)
  - *To reach the next level:* Sandbox policy lives in an external service whose configuration is not visible here.
- **B L2:** The remote sandbox receives the user's uploaded files and session files but not server credentials; its network egress and persistence settings are not visible here. — [api/server/services/ToolService.js:2305-2331](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/api/server/services/ToolService.js#L2305-L2331) (inferred)
  - *To reach the next level:* Egress, resource limits and session lifetime of the remote sandbox are not verifiable from this repository.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

Agents read content their user did not write: uploaded files, web search results, action and MCP tool outputs, and instructions of agents shared by other users. Nothing in the code tracks where content came from or restricts what a turn can do after reading it, and with the approval gate off by default an injected instruction can drive every tool the agent holds. Those tools can include outbound HTTP actions to any public host and connected MCP tools with write access, so data leakage and external side effects can both happen unattended.

- **S L0:** No provenance tagging, quarantine or taint-based gating of tool results or retrieved content was found; approval (when enabled) is by tool name, not by what the turn has read. — searched `rg -n -i 'prompt.injection|untrusted' -g '!*.spec.*' -g '!*.test.*'` in `packages/api/src/agents packages/api/src/mcp packages/api/src/tools` → 6 hits (Hits concern client-input sanitising, OAuth metadata hints and activity-label generation; none tags or restricts untrusted content entering the model context.) (verified)
  - *To reach the next level:* No control that disables or gates egress and write tools once untrusted content enters a turn.
- **C L0:** Tool and MCP results enter the conversation with the same standing as other context. — searched `rg -n -i 'prompt.injection|untrusted' -g '!*.spec.*' -g '!*.test.*'` in `packages/api/src/agents packages/api/src/mcp packages/api/src/tools` → 6 hits (Hits concern client-input sanitising, OAuth metadata hints and activity-label generation; none tags or restricts untrusted content entering the model context.) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished.
- **D L0:** No such control exists, and the related approval gate is off by default. — [packages/api/src/agents/hitl/policy.ts:97-101](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/agents/hitl/policy.ts#L97-L101) (verified)
  - *To reach the next level:* No default-on containment for untrusted content.
- **B L0:** A hijacked default agent can send data out through actions or MCP tools and take external write actions with no human involved. — [packages/data-provider/src/config.ts:830-847](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-provider/src/config.ts#L830-L847); [api/server/services/ToolService.js:667](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/api/server/services/ToolService.js#L667); [packages/api/src/agents/hitl/policy.ts:97-101](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/agents/hitl/policy.ts#L97-L101) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions are not gated in the default configuration.
- **Cap:** C5-WORSTCASE — In the default configuration a hijacked agent can leak data and take irreversible external actions unattended.

### C6 Memory, context & configuration integrity — 0.35 (high)

Long-term memory is off unless an administrator adds a memory block to librechat.yaml. When on, the model can save and delete memory entries with a tool, entries are stored per user and per agent, and they are fed back into later conversations as context; users can view, edit and opt out of memories. Entries are size-limited but not validated or marked by source. Agents, prompts and skills are user-authored and only shared with others when a role grants sharing, which ordinary users lack by default. There is no workspace or repository auto-loading in this server product.

- **S L1:** When memory is enabled, the model writes entries with no validation beyond a character limit and they are re-injected as context. — [packages/api/src/agents/memory.ts:257](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/agents/memory.ts#L257); [packages/data-schemas/src/app/memory.ts:12-35](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-schemas/src/app/memory.ts#L12-L35) (verified)
  - *To reach the next level:* Memory entries carry no provenance and writes are not gated or expired.
- **C L1:** Memory is per-user and inspectable, but summaries and shared agent instructions are not controlled as persistence paths. — [packages/data-schemas/src/schema/memory.ts:48](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-schemas/src/schema/memory.ts#L48) (verified)
  - *To reach the next level:* No control covers summaries or shared agent/skill content.
- **D L2:** Memory is off by default and, when on, queries are scoped by user and agent. — [packages/data-schemas/src/app/memory.ts:12-35](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-schemas/src/app/memory.ts#L12-L35); [packages/data-schemas/src/schema/memory.ts:48](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-schemas/src/schema/memory.ts#L48) (verified)
  - *To reach the next level:* No per-tenant storage separation or default retention limit.
- **B L2:** Poisoned memory, when enabled, persists across one user's conversations and can steer later tool use, but users can see and delete entries and other users are not affected. — [packages/data-provider/src/roles.ts:58-64](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-provider/src/roles.ts#L58-L64); [packages/data-schemas/src/schema/memory.ts:48](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-schemas/src/schema/memory.ts#L48) (verified)
  - *To reach the next level:* No rollback or review step before persisted content influences tools.
- **Cap:** none

### C7 Third-party extensions — 0.28 (high)

Third-party code enters LibreChat through MCP servers and operator plugins. Only administrators can add command-launching MCP servers (in librechat.yaml), and ordinary users cannot add remote MCP servers by default. Nothing pins or verifies the packages those commands launch, and changed tool lists from a server are refreshed without re-approval. Stdio MCP servers run as separate processes with a minimal default environment, under the same container user as LibreChat.

- **S L1:** MCP servers and plugins are operator-chosen, but launch commands and remote endpoints are taken as configured with no version pinning or integrity check. — [packages/data-provider/src/mcp.ts:343-344](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-provider/src/mcp.ts#L343-L344); [packages/api/src/plugins/deployment.ts:58-62](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/plugins/deployment.ts#L58-L62) (verified)
  - *To reach the next level:* No pinning or hash/signature verification of MCP packages or plugins.
- **C L0:** No extension type is verified; tool-list changes trigger a refresh rather than re-approval. — [packages/api/src/mcp/toolsChanged.ts:33-40](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/mcp/toolsChanged.ts#L33-L40) (verified)
  - *To reach the next level:* No verification for any extension type.
- **D L2:** No MCP server or plugin is enabled by default, stdio servers can only come from admin config, and ordinary users cannot create MCP servers by default. — [packages/data-provider/src/mcp.ts:534-557](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-provider/src/mcp.ts#L534-L557); [packages/data-provider/src/roles.ts:256-262](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-provider/src/roles.ts#L256-L262) (verified)
  - *To reach the next level:* Adding an extension does not show users what will run, and admins can grant MCP creation to users.
- **B L2:** Stdio MCP servers run as separate processes with only the MCP SDK's default environment plus their own configured variables. — [packages/api/src/mcp/connection.ts:1502](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/mcp/connection.ts#L1502); [docker-compose.yml:14](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/docker-compose.yml#L14) (verified)
  - *To reach the next level:* No per-extension sandbox or scoped credentials; servers share the container user and network.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.45 (high)

Stored user API keys, action secrets and OAuth tokens are encrypted at rest with a server key, which the server generates when none is configured. Logs pass through a redaction filter for API-key, bearer-token and secret patterns on both console and file output, and tracing to Langfuse or OpenTelemetry is opt-in. The example environment turns debug file logging on. Message content, files and tool results are sent to the model provider and stored in the database without redaction unless an administrator configures PII filters.

- **S L2:** User credentials are AES-encrypted at rest and log output is redacted by key and pattern, but model-bound content is not redacted by default. — [packages/data-schemas/src/crypto/index.ts:115-128](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-schemas/src/crypto/index.ts#L115-L128); [packages/data-schemas/src/config/parsers.ts:41-52](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-schemas/src/config/parsers.ts#L41-L52) (verified)
  - *To reach the next level:* Redaction does not cover model-bound messages by default, and no secret manager is used.
- **C L2:** Console and file logs are redacted and stdio subprocesses get stripped environments, but saved transcripts and model-bound messages are unfiltered by default. — [packages/data-schemas/src/config/winston.ts:36-40](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-schemas/src/config/winston.ts#L36-L40); [packages/api/src/mcp/connection.ts:1502](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/mcp/connection.ts#L1502) (verified)
  - *To reach the next level:* Model-bound messages and transcripts are not covered.
- **D L2:** Telemetry export is opt-in through environment keys; the example environment enables debug file logging, which is redacted. — [.env.example:195](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/.env.example#L195); [packages/api/src/langfuse/config.ts:81-82](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/langfuse/config.ts#L81-L82) (verified)
  - *To reach the next level:* Debug logging is on in the shipped example environment and stored transcripts are not minimised.
- **B L1:** A leak from the server exposes long-lived provider keys and OAuth tokens of every user, moderately scoped by each user's own provider. — [api/server/services/ActionService.js:455](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/api/server/services/ActionService.js#L455); [.env.example:597](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/.env.example#L597) (verified)
  - *To reach the next level:* Keys are long-lived; no short-lived or per-task credentials.
- **Cap:** none

### C9 Audit & traceability — 0.45 (medium)

Every message and its tool calls, with arguments and outputs, are stored in MongoDB per user and conversation, so a session can be reconstructed. LibreChat also has an append-only, hash-chained audit log, but today it only records role grants and permission changes, not agent runs, tool calls or approvals. Records live in the same database the server process writes to.

- **S L2:** Tool calls are recorded as structured message content and tool-call documents tied to user and conversation. — [packages/data-schemas/src/schema/toolCall.ts:21-40](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-schemas/src/schema/toolCall.ts#L21-L40); [packages/data-schemas/src/schema/message.ts:139](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-schemas/src/schema/message.ts#L139); [packages/data-schemas/src/models/auditLog.ts:6](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-schemas/src/models/auditLog.ts#L6) (verified)
  - *To reach the next level:* Approver identity and correlation across sub-agents are not recorded in a tamper-evident store; the hash-chained audit log covers only admin grants.
- **C L2:** Built-in, action and MCP tool calls are persisted with the message; the audit log's action list covers grant and permission events only. — [packages/data-schemas/src/types/admin.ts:68-73](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-schemas/src/types/admin.ts#L68-L73) (verified)
  - *To reach the next level:* Approvals, denials, configuration changes and memory writes are not in the audit record.
- **D L2:** Message persistence is always on and stored in MongoDB outside any tool's reach, but the server process can alter it. — [packages/data-schemas/src/schema/message.ts:139](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-schemas/src/schema/message.ts#L139) (verified)
  - *To reach the next level:* Records are written by the same process that runs the agent.
- **B L1:** Conversation records are written as part of message saving; there is no fail-closed path tying actions to a written record. — [packages/data-schemas/src/schema/toolCall.ts:21-40](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-schemas/src/schema/toolCall.ts#L21-L40) (inferred)
  - *To reach the next level:* No per-action durable write before execution.
- **Cap:** none

### C10 Limits & kill switch — 0.40 (high)

Each agent turn is capped at 50 graph steps by default, sub-agents derive their turn budget from the same limit, model calls have response timeouts, and per-user message concurrency and rate limits are on in the example environment. Users can stop a generation, which aborts the running job. But token spending limits (balance) are off by default, there is no default maximum, so any agent author can set a much higher step limit on their own agent, and scheduled agent runs fire without the user present.

- **S L2:** A step cap plus model response timeouts and argument-size guards are enforced in code; token budgets exist only through the opt-in balance system. — [packages/api/src/agents/config.ts:8](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/agents/config.ts#L8); [packages/data-provider/src/config.ts:1335-1336](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-provider/src/config.ts#L1335-L1336); [packages/api/src/stream/GenerationJobManager.ts:963](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/stream/GenerationJobManager.ts#L963) (verified)
  - *To reach the next level:* No default token or cost cap.
- **C L2:** The step limit applies to the top-level run and is reused to size sub-agent turns, but each sub-agent gets its own budget rather than sharing the parent's. — [packages/api/src/agents/config.ts:77-83](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/agents/config.ts#L77-L83) (verified)
  - *To reach the next level:* Sub-agents and scheduled runs do not draw from one shared budget.
- **D L1:** The default limit of 50 can be raised without bound by any agent author through the per-agent recursion_limit unless an admin sets maxRecursionLimit. — [packages/api/src/agents/config.ts:41-58](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/agents/config.ts#L41-L58); [packages/api/src/agents/validation.ts:592](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/api/src/agents/validation.ts#L592); [packages/data-provider/src/config.ts:2822-2823](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-provider/src/config.ts#L2822-L2823) (verified)
  - *To reach the next level:* No default ceiling on per-agent limits; balance is off by default.
- **B L1:** Abort stops the running job, but spend is unbounded by default and scheduled runs continue on their own. — [api/server/routes/agents/index.js:570](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/api/server/routes/agents/index.js#L570); [packages/data-provider/src/roles.ts:280-283](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/packages/data-provider/src/roles.ts#L280-L283); [.env.example:864-868](https://github.com/LibreChat-AI/LibreChat/blob/f10b1d91f1eee3a2c82d5247bf620351486b7c1b/.env.example#L864-L868) (verified)
  - *To reach the next level:* No default spend ceiling, and schedules keep firing after a stop.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Uploaded files, web search results, action and MCP tool outputs enter the agent context (packages/data-provider/src/config.ts:830) · [B] sensitive data/systems: User files, memories and per-user connected credentials (api/server/services/ActionService.js:455) · [C] state change / egress: User-defined HTTP actions to any public host and MCP tools (api/server/services/ToolService.js:667) · Same default session? Yes

## Highest-impact improvements
1. Turn tool approval on by default (ask for actions, MCP and code tools) so administrators opt out rather than in. — C2 D L0→L3, +0.150 before caps (Playbook 5)
2. Once a turn has read files, search results or tool outputs, force egress and write tools through approval. — C5 S L0→L2, +0.150 before caps (Playbook 1)
3. Ship a default maxRecursionLimit and enable balance with a conservative per-user token budget. — C10 D L1→L2, +0.050 before caps (Playbook 3 step 3)
4. Pin MCP server packages and plugins by version and digest, and require admin re-approval when a server's tool list changes. — C7 S L1→L3, +0.150 before caps (Playbook 3)
5. Extend the hash-chained audit log to agent runs, tool calls and approval decisions. — C9 C L2→L3, +0.075 before caps (Playbook 1 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The agent loop, tool-approval interception and the code-execution sandbox live in external packages and services (@librechat/agents, the Code Interpreter API) that are not in this repository; C2 coverage and C4 strength and blast radius are inferred.
- Opt-in subsystems (attached personal code workers, schedules and triggers, remote agents API, plugins, OBO token exchange, PII filters) were reviewed only for their defaults.
- The separate admin panel and RAG API container images were not reviewed.
- No text attempting to steer AI reviewers was found in the repository.
