# Defense-in-Depth Score: Jan

**Repo:** https://github.com/janhq/jan · **Commit:** `98c3b84a5a218fafbe8428ce5c930566e5943475` · **Reviewed:** 2026-10-05
**What it is:** Offline ChatGPT alternative desktop app with MCP tool calling
**Category:** AI Assistants
**Scored configuration:** Stable Jan desktop release, chat surface, fresh install with default settings (web search on, built-in sandboxed shell, no MCP servers active, analytics off).
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials opt-in · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents no · external communication opt-in

## Score: 4.7 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L3 | L2 | 0.55 | — | **0.55** | High |
| C2 | Approval gates | L3 | L2 | L2 | L2 | 0.57 | — | **0.57** | High |
| C3 | Tool & action scoping | L2 | L2 | L1 | L2 | 0.45 | — | **0.45** | High |
| C4 | Code-execution isolation | L3 | L3 | L4 | L2 | 0.75 | — | **0.75** | High |
| C5 | Untrusted input blast radius | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | High |
| C6 | Memory, context & configuration integrity | L2 | L3 | L3 | L1 | 0.57 | — | **0.57** | High |
| C7 | Third-party extensions | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |


Jan's chat shell runs in an OS sandbox with no network, its own disposable folder, no secrets and no view of your home directory, and it is not offered at all when no sandbox is available. Every MCP tool call asks you first. The dominant risk is the web access that is on by default: web search and page fetch run without approval, so text injected into a page or document can make the model send your conversation or memory notes to an outside URL. The chat loop also has no step or spend limit, and MCP servers you add run unpinned with your full environment.

## Critical gaps
- None: no criterion is capped and no blast radius is at the worst level.

## Criterion details

### C1 Identity & least privilege — 0.55 (high)

Jan runs as the desktop user and has no per-request authorization layer, but it deliberately narrows what its built-in tools can reach. The chat shell starts from an emptied environment with only a short allowlist of variables, and the user's home directory and the Jan data folder (settings, keys, models) are masked from it. Web search uses a keyless hosted provider by default, so no credentials are attached to built-in tools. MCP servers the user adds are the exception: they are started with the app's full inherited environment plus whatever keys the user configured for them.

- **S L2:** Built-in tools carry no credentials: the shell's environment is cleared to an allowlist and the home directory and data folder are hidden from it; authority is otherwise the user's own. — [src-tauri/plugins/tauri-plugin-agent-tools/src/tools/proc.rs:426](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/tools/proc.rs#L426); [src-tauri/plugins/tauri-plugin-agent-tools/src/tools/jail.rs:396](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/tools/jail.rs#L396); [src-tauri/plugins/tauri-plugin-agent-tools/src/commands.rs:975](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/commands.rs#L975) (verified)
  - *To reach the next level:* No per-tool or per-capability credentials and no deterministic authorization check before a tool acts.
- **C L2:** The narrowing covers the built-in shell and file tools, but MCP server processes are spawned with the inherited environment and their own configured keys. — [src-tauri/src/core/mcp/helpers.rs:522](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/src/core/mcp/helpers.rs#L522); [src-tauri/src/core/mcp/helpers.rs:565](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/src/core/mcp/helpers.rs#L565); searched `rg -n 'env_clear'` in `src-tauri/src/core/mcp` → 0 hits (MCP server processes are spawned without clearing the inherited environment) (verified)
  - *To reach the next level:* MCP servers do not pass through the same environment scrubbing or any authorization layer.
- **D L3:** Out of the box the chat surface can write only to a disposable per-conversation sandbox; tools with real write authority come only from MCP servers the user adds. — [web-app/src/lib/agentTools.ts:63](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/lib/agentTools.ts#L63); [src-tauri/src/core/mcp/constants.rs:27](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/src/core/mcp/constants.rs#L27); [src-tauri/plugins/tauri-plugin-agent-tools/src/tools/mod.rs:377](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/tools/mod.rs#L377) (verified)
  - *To reach the next level:* No automatic expiry of elevated authority (MCP servers stay configured until removed).
- **B L2:** If the narrowing fails, the agent holds the user's model-provider keys and any MCP credentials the user added, on one machine and one user account. — [src-tauri/src/core/server/provider_secrets.rs:1-11](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/src/core/server/provider_secrets.rs#L1-L11); [src-tauri/src/core/mcp/helpers.rs:565](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/src/core/mcp/helpers.rs#L565) (verified)
  - *To reach the next level:* Credentials are long-lived and not scoped per task; MCP servers can reach the user's whole environment.
- **Cap:** none

### C2 Approval gates — 0.57 (high)

Every MCP tool call asks the user first, unless they already trusted that tool or server. The prompt sits next to the full tool input and offers deny, allow once, allow for this chat, or always allow. Jan's own tools never prompt: web search and page fetches run unattended, and the shell runs without approval because it is confined to a disposable sandbox with no network. A single settings switch, off by default and clearly described, auto-approves every MCP call. 'Always allow' grants are stored in app settings with no screen to review or revoke them.

- **S L3:** Per-call approval for MCP tools with the exact input shown next to deny/allow-once/allow-thread/allow-always; built-in tools form a separate auto-allowed tier. — [web-app/src/hooks/useToolApprovalRequests.ts:46-50](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/hooks/useToolApprovalRequests.ts#L46-L50); [web-app/src/containers/message/ToolCallCard.tsx:138-139](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/containers/message/ToolCallCard.tsx#L138-L139); [web-app/src/components/ai-elements/tool.tsx:352-373](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/components/ai-elements/tool.tsx#L352-L373) (verified)
  - *To reach the next level:* No argument-level allow/deny rules; a thread or global grant covers later calls with different arguments.
- **C L2:** All MCP tools are gated and unknown tool names are refused, but the built-in web fetch (outbound requests) and the sandboxed shell are exempt. — [web-app/src/routes/threads/$threadId.tsx:107-111](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/routes/threads/$threadId.tsx#L107-L111); [web-app/src/routes/threads/$threadId.tsx:504](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/routes/threads/$threadId.tsx#L504); [web-app/src/routes/threads/$threadId.tsx:577](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/routes/threads/$threadId.tsx#L577) (verified)
  - *To reach the next level:* Built-in egress (web_fetch) does not cross the approval gate.
- **D L2:** Approval is on by default; one settings toggle auto-approves all MCP calls, and always-allow grants accumulate with no review screen. — [web-app/src/hooks/useToolApproval.ts:32](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/hooks/useToolApproval.ts#L32); [web-app/src/locales/en/mcp-servers.json:51](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/locales/en/mcp-servers.json#L51); searched `rg -n 'approvedServers|approvedToolsGlobal'` in `web-app/src/routes web-app/src/containers` → 0 hits (no settings view reads the persisted grants) (verified)
  - *To reach the next level:* No confirmation or session bound on the allow-all switch, and no UI to list or revoke persistent grants.
- **B L2:** Default-reachable actions are confined to a disposable sandbox or are fetches; anything irreversible comes from user-added MCP servers, with no checkpoint or undo. — [web-app/src/lib/agentTools.ts:63](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/lib/agentTools.ts#L63); [src-tauri/src/core/mcp/constants.rs:27](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/src/core/mcp/constants.rs#L27) (verified)
  - *To reach the next level:* No rollback or dry-run for MCP actions and no rate limit on approvals.
- **Cap:** none

### C3 Tool & action scoping — 0.45 (high)

Jan's file tools resolve and re-check paths against the workspace before writing, but the chat surface does not use them. Its default tools are general-purpose: a shell that takes any command string, and a web fetch that takes any URL. Both are bounded by other layers (the OS sandbox and a hosted fetch provider) rather than by argument validation. MCP tool arguments are passed through as given. Web access is on by default and can be switched off; the shell cannot be disabled on its own.

- **S L2:** File tools use canonicalized containment checks, but the default chat tools are a raw shell string and an arbitrary URL. — [src-tauri/plugins/tauri-plugin-agent-tools/src/tools/handlers.rs:733](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/tools/handlers.rs#L733); [web-app/src/lib/webSearchTool.ts:75-76](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/lib/webSearchTool.ts#L75-L76); [src-tauri/plugins/tauri-plugin-agent-tools/src/tools/gate.rs:214](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/tools/gate.rs#L214) (verified)
  - *To reach the next level:* No allowlist validation for the general-purpose shell and fetch tools.
- **C L2:** Built-in tools apply their own checks; MCP tool arguments get no shared validation layer. — [src-tauri/src/core/mcp/helpers.rs:565](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/src/core/mcp/helpers.rs#L565); [web-app/src/routes/threads/$threadId.tsx:107-111](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/routes/threads/$threadId.tsx#L107-L111) (verified)
  - *To reach the next level:* No shared validation layer that extension tools inherit.
- **D L1:** Exec (sandboxed shell) and network (web search and fetch) tools are on by default; web access has a toggle, the shell has none. — [web-app/src/hooks/useWebSearchConfig.ts:70](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/hooks/useWebSearchConfig.ts#L70); [web-app/src/lib/agentTools.ts:63](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/lib/agentTools.ts#L63); [web-app/src/lib/agentTools.ts:126](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/lib/agentTools.ts#L126) (verified)
  - *To reach the next level:* No read-only default tool set; the shell cannot be individually disabled.
- **B L2:** A misused shell is confined to the conversation's sandbox; a misused fetch can reach any public URL. — [web-app/src/lib/webSearchTool.ts:75-76](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/lib/webSearchTool.ts#L75-L76); [src-tauri/plugins/tauri-plugin-agent-tools/src/commands.rs:973](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/commands.rs#L973) (verified)
  - *To reach the next level:* No quantity bounds or host allowlist on outbound fetches.
- **Cap:** none

### C4 Code-execution isolation — 0.75 (high)

The chat shell runs inside an OS sandbox: bubblewrap on Linux, Seatbelt on macOS and AppContainer on Windows. Writes are limited to a per-conversation workspace, the home directory and Jan data folder are hidden, network is off, and the environment is emptied. If no sandbox backend is available the shell is not offered instead of running unconfined, and the desktop has no switch that runs it on the host. Two things limit it: the rest of the filesystem outside the home directory stays readable, and there are no CPU or memory limits. MCP servers the user adds run directly on the host.

- **S L3:** OS sandbox profiles (bubblewrap with all namespaces unshared, Seatbelt deny-network policy, AppContainer) with writes limited to the workspace and network denied by default. — [src-tauri/plugins/tauri-plugin-agent-tools/src/tools/jail.rs:445](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/tools/jail.rs#L445); [src-tauri/plugins/tauri-plugin-agent-tools/src/tools/jail.rs:578](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/tools/jail.rs#L578); [src-tauri/plugins/tauri-plugin-agent-tools/src/tools/jail.rs:451](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/tools/jail.rs#L451) (verified)
  - *To reach the next level:* Not kernel-separated (no microVM or gVisor).
- **C L3:** The shell, the only model-reachable exec path in chat, always goes through the sandbox and fails closed; user-added MCP stdio servers run on the host. — [src-tauri/plugins/tauri-plugin-agent-tools/src/tools/handlers.rs:861](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/tools/handlers.rs#L861); [src-tauri/plugins/tauri-plugin-agent-tools/src/tools/jail.rs:283](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/tools/jail.rs#L283); [src-tauri/src/core/mcp/helpers.rs:522](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/src/core/mcp/helpers.rs#L522) (verified)
  - *To reach the next level:* Processes started by extensions (MCP servers) are not sandboxed.
- **D L4:** Sandboxing is always on for the desktop; the only override can switch it to 'none', which removes the shell rather than running it unconfined, and the policy is defined in code. — [src-tauri/plugins/tauri-plugin-agent-tools/src/tools/mod.rs:377](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/tools/mod.rs#L377); [src-tauri/plugins/tauri-plugin-agent-tools/src/tools/jail.rs:202](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/tools/jail.rs#L202); [src-tauri/plugins/tauri-plugin-agent-tools/src/tools/handlers.rs:861](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/tools/handlers.rs#L861); [web-app/src/lib/agentTools.ts:126](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/lib/agentTools.ts#L126) (verified)
- **B L2:** Inside: an ephemeral workspace, no network, no secrets in the environment, home hidden; but the host filesystem outside home is readable and only process-count and file-size limits apply. — [src-tauri/plugins/tauri-plugin-agent-tools/src/commands.rs:973](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/commands.rs#L973); [src-tauri/plugins/tauri-plugin-agent-tools/src/tools/proc.rs:426](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/tools/proc.rs#L426); [src-tauri/plugins/tauri-plugin-agent-tools/src/tools/proc.rs:245-247](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/tools/proc.rs#L245-L247) (verified)
  - *To reach the next level:* No CPU or memory limits, and reads are not limited to the workspace.
- **Cap:** none

### C5 Untrusted input blast radius — 0.38 (high)

Jan does not separate untrusted content from instructions: web pages, search results, attached documents and MCP results enter the conversation as ordinary tool output. What limits a hijacked chat is that MCP actions always need a click and the shell has no network. But web search and page fetch run without approval and can request any URL, so injected text can send conversation content, attached documents or memory notes out through a fetched URL. Nothing irreversible can happen without a human in the default setup.

- **S L2:** MCP actions always require approval and the shell has no network, but outbound fetches are not gated. — [web-app/src/hooks/useToolApprovalRequests.ts:46-50](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/hooks/useToolApprovalRequests.ts#L46-L50); [src-tauri/plugins/tauri-plugin-agent-tools/src/commands.rs:973](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/commands.rs#L973); [web-app/src/routes/threads/$threadId.tsx:107-111](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/routes/threads/$threadId.tsx#L107-L111) (verified)
  - *To reach the next level:* Egress is not disabled or gated once untrusted content enters the conversation.
- **C L1:** Tool results, web pages and documents are not distinguished from other context; only the MCP path is gated, regardless of source. — searched `rg -n -i 'untrusted|prompt.?injection'` in `web-app/src/lib/custom-chat-transport.ts web-app/src/lib/webSearchTool.ts` → 0 hits (tool and web results are not marked or treated as untrusted) (verified)
  - *To reach the next level:* No source of untrusted content is tracked or treated differently.
- **D L2:** The approval layer is on by default; the operator can switch it off from settings. — [web-app/src/hooks/useToolApproval.ts:32](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/hooks/useToolApproval.ts#L32); [web-app/src/locales/en/mcp-servers.json:51](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/locales/en/mcp-servers.json#L51) (verified)
  - *To reach the next level:* Disabling has no extra confirmation.
- **B L1:** A hijacked chat can exfiltrate conversation content through web_fetch with no human involved, but cannot take irreversible actions without approval. — [web-app/src/hooks/useWebSearchConfig.ts:70](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/hooks/useWebSearchConfig.ts#L70); [web-app/src/lib/custom-chat-transport.ts:1116-1121](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/lib/custom-chat-transport.ts#L1116-L1121); [web-app/src/lib/webSearchTool.ts:75-76](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/lib/webSearchTool.ts#L75-L76) (verified)
  - *To reach the next level:* Egress is not behind approval.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.57 (high)

Chat has one persistent store that reaches future conversations: memory notes, injected as background context into every chat. In the stable chat surface only the user writes them, by clicking Remember on a message or editing them in settings; the model has no memory-writing tool there. There are no workspace instruction files or project configs to auto-load. The concern is what happens once a note is poisoned: it has no expiry or provenance and is presented as the user's own notes in every later conversation, where it can steer tool use.

- **S L2:** Memory writes in chat need a user action, but notes have no expiry or provenance and load as trusted background context. — [web-app/src/lib/agentWorkspace.ts:93-105](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/lib/agentWorkspace.ts#L93-L105); [web-app/src/lib/custom-chat-transport.ts:1759-1760](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/lib/custom-chat-transport.ts#L1759-L1760) (verified)
  - *To reach the next level:* No expiry or source restriction on memory notes.
- **C L3:** The memory store, attachments and assistant instructions are all written only through user actions; there are no auto-loaded workspace files. — [web-app/src/lib/agentWorkspace.ts:93-105](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/lib/agentWorkspace.ts#L93-L105); [web-app/src/lib/agentTools.ts:63](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/lib/agentTools.ts#L63) (verified)
  - *To reach the next level:* Retrieved content and memory are not provenance-tagged end to end.
- **D L3:** Single-user local store; the chat model has no tool to write memory or settings, and the shell cannot see the data folder. — [web-app/src/lib/agentTools.ts:63](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/lib/agentTools.ts#L63); [src-tauri/plugins/tauri-plugin-agent-tools/src/commands.rs:975](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/commands.rs#L975) (verified)
  - *To reach the next level:* No retention limit on memory by default.
- **B L1:** A poisoned note persists across all of the user's conversations and can steer tool use. — [web-app/src/lib/custom-chat-transport.ts:1759-1760](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/lib/custom-chat-transport.ts#L1759-L1760) (verified)
  - *To reach the next level:* Notes are not limited to text output or gated actions.
- **Cap:** none

### C7 Third-party extensions — 0.30 (high)

No third-party code runs by default: every MCP server in the shipped catalog is inactive, and adding one is an explicit user step. Once added, servers are launched as written, often through 'npx -y' with an unpinned or '@latest' package. Jan does not pin or verify them and does not prompt again when they change. They run as separate processes with the user's full environment. Model downloads are checked against a SHA-256 when one is known, and app updates are signed.

- **S L1:** MCP servers run from user-chosen sources, unpinned; the shipped catalog itself uses '@latest'. — [src-tauri/src/core/mcp/constants.rs:22](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/src/core/mcp/constants.rs#L22); [src-tauri/src/core/mcp/helpers.rs:522](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/src/core/mcp/helpers.rs#L522) (verified)
  - *To reach the next level:* No version pinning or integrity check for MCP servers.
- **C L1:** Model files are hash-checked when a hash is known; MCP servers are not verified. — [src-tauri/src/core/downloads/helpers.rs:175-182](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/src/core/downloads/helpers.rs#L175-L182); [src-tauri/src/core/mcp/constants.rs:22](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/src/core/mcp/constants.rs#L22) (verified)
  - *To reach the next level:* MCP servers, the main extension type, have no verification.
- **D L2:** Nothing third-party is enabled by default; servers are added explicitly by the user in settings. — [src-tauri/src/core/mcp/constants.rs:27](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/src/core/mcp/constants.rs#L27) (verified)
  - *To reach the next level:* No explicit review of the exact package and permissions at the moment a server is enabled.
- **B L1:** Each MCP server runs as a separate process with the same user and the full inherited environment. — [src-tauri/src/core/mcp/helpers.rs:522](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/src/core/mcp/helpers.rs#L522); searched `rg -n 'env_clear'` in `src-tauri/src/core/mcp` → 0 hits (MCP server processes are spawned without clearing the inherited environment) (verified)
  - *To reach the next level:* No environment scrubbing or per-server sandbox.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.50 (high)

Provider and web-search API keys are kept in the OS keyring, with an encrypted owner-only file as a fallback, and are excluded from the settings file. Product analytics are opt-in, with autocapture and session recording off. The shell sees none of these keys because its environment is emptied and the data folder is hidden. Gaps: MCP server keys sit in plain text in the MCP config, MCP processes inherit the full environment, and nothing masks secrets in logs, transcripts or model-bound messages.

- **S L2:** OS keyring storage for provider and search keys, plain-text MCP env values, and no secret-masking layer. — [src-tauri/src/core/server/provider_secrets.rs:1-11](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/src/core/server/provider_secrets.rs#L1-L11); [web-app/src/hooks/useWebSearchConfig.ts:96](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/hooks/useWebSearchConfig.ts#L96); searched `rg -n -i 'redact'` in `web-app/src/lib src-tauri/src/core/mcp src-tauri/src/core/threads` → 0 hits (no secret-masking helper on the chat, MCP or transcript paths) (verified)
  - *To reach the next level:* No secret masking before logs and model-bound messages.
- **C L2:** At-rest storage and the shell environment are protected; MCP subprocess environments, logs and transcripts are not. — [src-tauri/plugins/tauri-plugin-agent-tools/src/tools/proc.rs:426](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/tools/proc.rs#L426); searched `rg -n 'env_clear'` in `src-tauri/src/core/mcp` → 0 hits (MCP server processes are spawned without clearing the inherited environment) (verified)
  - *To reach the next level:* Subprocess environments for MCP servers and model-bound content are not covered.
- **D L2:** Analytics are opt-in and content-light; there is no secret masking to leave on. — [web-app/src/hooks/useAnalytic.ts:65](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/hooks/useAnalytic.ts#L65); [web-app/src/providers/AnalyticProvider.tsx:21](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/providers/AnalyticProvider.tsx#L21) (verified)
  - *To reach the next level:* Secret masking is not always on.
- **B L2:** Leaked keys are provider-scoped but long-lived; they are not reachable from the shell. — [src-tauri/src/core/server/provider_secrets.rs:1-11](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/src/core/server/provider_secrets.rs#L1-L11); [src-tauri/plugins/tauri-plugin-agent-tools/src/commands.rs:975](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/commands.rs#L975) (verified)
  - *To reach the next level:* No short-lived or rotated credentials.
- **Cap:** none

### C9 Audit & traceability — 0.45 (high)

Every conversation is saved as a per-thread JSONL file in the Jan data folder, and each assistant message records its tool calls with their arguments and results. Denied calls are recorded as errors. Approvals and who gave them are not, and there is no audit trail separate from the editable chat history. Records are written by the app once a message completes, not as a durable per-action log.

- **S L2:** Structured per-thread transcript with tool name, arguments and result for each call. — [src-tauri/src/core/threads/mod.rs:5](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/src/core/threads/mod.rs#L5); [web-app/src/lib/messages.ts:96-114](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/lib/messages.ts#L96-L114) (verified)
  - *To reach the next level:* No actor attribution or record of approvals.
- **C L2:** Built-in and MCP tool calls are recorded in the transcript, and denials appear as errors; approvals and configuration changes are not recorded. — [web-app/src/routes/threads/$threadId.tsx:522](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/routes/threads/$threadId.tsx#L522); [web-app/src/lib/messages.ts:96-114](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/lib/messages.ts#L96-L114) (verified)
  - *To reach the next level:* Approvals, grants and settings changes are not recorded.
- **D L2:** On by default and stored outside the shell's reach, but in the same editable chat history the app rewrites. — [src-tauri/src/core/threads/mod.rs:5](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/src/core/threads/mod.rs#L5); [src-tauri/plugins/tauri-plugin-agent-tools/src/commands.rs:975](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/commands.rs#L975) (verified)
  - *To reach the next level:* No record written by a component separate from the chat UI.
- **B L1:** Best-effort: records are written when messages complete; no fail-closed or durable per-action logging. — [web-app/src/lib/messages.ts:96-114](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/lib/messages.ts#L96-L114) (verified)
  - *To reach the next level:* Records are not flushed durably per action.
- **Cap:** none

### C10 Limits & kill switch — 0.20 (high)

The chat tool loop has no step or token limit: it keeps calling the model as long as the model keeps asking for tools, until the user presses Stop. Individual calls are bounded. MCP calls time out after 30 seconds. Shell commands are moved to the background after 30 seconds (the model can choose a longer timeout), and Stop kills the conversation's shell process groups. The step and token budgets in the code apply only to the preview Cowork surface.

- **S L1:** Per-call timeouts and a process-group kill on Stop, but no iteration, token or wall-clock cap on the chat loop. — [web-app/src/routes/threads/$threadId.tsx:193-201](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/routes/threads/$threadId.tsx#L193-L201); searched `rg -n 'maxToolSteps|MAX_AGENT_STEPS|stepCountIs'` in `web-app/src/routes/threads web-app/src/lib/custom-chat-transport.ts` → 0 hits (the chat tool loop has no step or token cap; the caps in coworkBudget.ts apply only to the nightly Cowork surface); [src-tauri/src/core/mcp/constants.rs:2](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/src/core/mcp/constants.rs#L2) (verified)
  - *To reach the next level:* No iteration cap on the chat tool loop.
- **C L1:** Timeouts apply to MCP calls and shell commands; nothing bounds the loop itself. — [src-tauri/src/core/mcp/constants.rs:2](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/src/core/mcp/constants.rs#L2); [src-tauri/plugins/tauri-plugin-agent-tools/src/tools/handlers.rs:883](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/tools/handlers.rs#L883) (verified)
  - *To reach the next level:* No loop-level limit for tool timeouts to combine with.
- **D L1:** Timeout defaults exist, but the model can raise the shell timeout per call and there is no default loop limit. — [src-tauri/plugins/tauri-plugin-agent-tools/src/tools/handlers.rs:883](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/src-tauri/plugins/tauri-plugin-agent-tools/src/tools/handlers.rs#L883); searched `rg -n 'maxToolSteps|MAX_AGENT_STEPS|stepCountIs'` in `web-app/src/routes/threads web-app/src/lib/custom-chat-transport.ts` → 0 hits (the chat tool loop has no step or token cap; the caps in coworkBudget.ts apply only to the nightly Cowork surface) (verified)
  - *To reach the next level:* No sensible default step or spend limit.
- **B L0:** A runaway chat can loop and spend on a paid provider until the user notices and presses Stop. — [web-app/src/routes/threads/$threadId.tsx:193-201](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/routes/threads/$threadId.tsx#L193-L201); [web-app/src/containers/ChatInput.tsx:870](https://github.com/janhq/jan/blob/98c3b84a5a218fafbe8428ce5c930566e5943475/web-app/src/containers/ChatInput.tsx#L870) (verified)
  - *To reach the next level:* No ceiling on steps or spend.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Web pages, search results and attached documents via web_fetch/web_search and RAG (web-app/src/lib/webSearchTool.ts:75) · [B] sensitive data/systems: Conversation history, attached documents and memory notes injected into the system prompt (web-app/src/lib/custom-chat-transport.ts:1759) · [C] state change / egress: web_fetch to any URL without approval (web-app/src/routes/threads/$threadId.tsx:107) · Same default session? Yes

## Highest-impact improvements
1. Add a default step and token cap to the chat tool loop, matching the budget the Cowork runner already enforces. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)
2. Require approval for web_fetch (or for any fetch after untrusted content entered the conversation) instead of auto-allowing it. — C5 B L1→L2, +0.050 before caps (Playbook 1)
3. Launch MCP servers with a cleared environment plus only their configured variables, as the shell already does. — C7 B L1→L2, +0.050 before caps (Playbook 3)
4. Add a settings view to list and revoke 'always allow' grants, and confirm before enabling auto-approve for all MCP tools. — C2 D L2→L3, +0.050 before caps (Playbook 5)
5. Pin MCP catalog entries to exact versions instead of '@latest'. — C7 S L1→L2, +0.075 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the stable desktop chat surface. The Cowork agent surface (full file, memory, skill and sub-agent toolset, shell network on by default, no per-call approval) ships only in nightly and dev builds (web-app/src/lib/version.ts:17) and was not scored; neither was the bundled `jan` CLI agent, which has its own permission gate and config files.
- Windows AppContainer and macOS Seatbelt profiles were reviewed from source only; their effective confinement on each OS version was not verified.
- UI rendering libraries, the local OpenAI-compatible API server (off by default) and the MLX/llama.cpp engine code were not audited in depth.
