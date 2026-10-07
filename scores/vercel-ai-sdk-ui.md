# Defense-in-Depth Score: Vercel AI SDK UI

**Repo:** https://github.com/vercel/ai (`packages/react`) · **Commit:** `08d7f0a75e1466d28c3b71c2be4c0d89d552acf9` (@ai-sdk/react@4.0.130-28-g08d7f0a7) · **Reviewed:** 2026-10-04
**What it is:** Framework-agnostic chat and agent UI layer of the Vercel AI SDK: useChat/useCompletion/useObject hooks, the UI message stream protocol, client-side tools and tool-approval flows.
**Category:** Agent Frameworks
**Scored configuration:** Documented default useChat setup: @ai-sdk/react (packages/react) over the shared chat core in packages/ai/src/ui and packages/ai/src/ui-message-stream, with the server route streamText({ messages: await convertToModelMessages(messages), tools }) returned via toUIMessageStream/createUIMessageStreamResponse, default arguments throughout (no toolApproval, no experimental_toolApprovalSecret, no telemetry integration, no abortSignal forwarding); Vue/Svelte bindings share the same core.
**Agent surface (default):** code execution no · filesystem write no · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents no · external communication yes

## Score: 3.6 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L3 | L2 | L0 | L0 | 0.38 | G1 | **0.38** | High |
| C3 | Tool & action scoping | L2 | L3 | L2 | L1 | 0.53 | — | **0.53** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | Medium |
| C7 | Third-party extensions | L1 | L1 | L2 | L2 | 0.35 | — | **0.35** | Medium |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C9 | Audit & traceability | L2 | L2 | L0 | L1 | 0.35 | G1 | **0.35** | Medium |
| C10 | Limits & kill switch | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |

Controls where a risk surface exists: 2.60 / 9.0 (29%); 1 criterion scored SA (surface absent).

AI SDK UI ships a careful tool-approval design, with exact-input approval prompts, schema re-validation and optional HMAC-signed approvals, but approval is off by default: in the documented chat route every tool the model calls runs immediately with the server's full credentials. The server also trusts the message history the browser posts, so without the experimental approval secret a modified client can replay fabricated tool calls and approvals to run any registered tool. Prompt injection through tool results is the dominant risk; deployers should enable toolApproval and experimental_toolApprovalSecret, authenticate the route, forward req.signal, and cap automatic resubmissions.

## Critical gaps
- Server tools run with the server's ambient credentials for any caller, and a client-fabricated tool call plus approval in the posted history executes any registered tool unless the opt-in approval secret is set. (ASI03, T3, ASI02; C1) — [packages/ai/src/generate-text/collect-tool-approvals.ts:54](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/collect-tool-approvals.ts#L54); [packages/ai/src/generate-text/validate-tool-approvals.ts:68](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/validate-tool-approvals.ts#L68); [packages/ai/src/generate-text/validate-tool-approvals.ts:158-168](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/validate-tool-approvals.ts#L158-L168); [content/docs/04-ai-sdk-ui/03-chatbot-tool-usage.mdx:568](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/content/docs/04-ai-sdk-ui/03-chatbot-tool-usage.mdx#L568)
- Approval is off by default, so content injected through a tool result can drive any registered tool, including exfiltration and irreversible actions, with no human involved. (ASI01, LLM01, T6; C5) — [packages/ai/src/generate-text/resolve-tool-approval.ts:109-110](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/resolve-tool-approval.ts#L109-L110); [packages/ai/src/generate-text/stream-text.ts:547](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/stream-text.ts#L547)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

AI SDK UI has no notion of who is asking. The documented chat route reads whatever message history the browser posts and runs the developer's server-side tools with the server process's own credentials (provider API keys from environment variables, plus whatever the tools hold), with no per-request authorization hook in the UI layer. Because the client supplies the whole history, a caller can also fabricate an assistant tool call with a matching approval and have the server execute any registered tool with inputs of its choosing, unless the developer opts into approval signing. Every tool therefore runs with the server's full authority on behalf of any caller who can reach the route.

- **S L0:** Server tools run with the deploying process's ambient credentials; the UI layer issues no scoped or per-request identity. — [packages/provider-utils/src/load-api-key.ts:30](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/provider-utils/src/load-api-key.ts#L30); searched `rg -n -i 'authoriz|userId|principal'` in `packages/ai/src/ui packages/ai/src/ui-message-stream packages/react/src` → 8 hits (All 8 hits are test fixtures (Authorization headers in use-object tests, a userId tool input in a stream test); no authorization primitive exists.) (verified)
  - *To reach the next level:* No scoped identity or per-request authority; tools inherit the server process's credentials.
- **C L0:** No authorization check sits between the posted message history and tool execution; client-fabricated tool calls reach execution without the model. — [packages/ai/src/generate-text/collect-tool-approvals.ts:54](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/collect-tool-approvals.ts#L54); [packages/ai/src/generate-text/validate-tool-approvals.ts:68](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/validate-tool-approvals.ts#L68); [packages/ai/src/generate-text/validate-tool-approvals.ts:158-168](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/validate-tool-approvals.ts#L158-L168); [content/docs/04-ai-sdk-ui/02-chatbot.mdx:86](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/content/docs/04-ai-sdk-ui/02-chatbot.mdx#L86); [content/docs/04-ai-sdk-ui/02-chatbot.mdx:91](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/content/docs/04-ai-sdk-ui/02-chatbot.mdx#L91) (verified)
  - *To reach the next level:* No authorization layer that every tool path (including history-replayed approvals) passes through.
- **D L0:** The documented default route has no authentication and executes tools with server authority; least privilege is entirely the developer's job. — [content/docs/04-ai-sdk-ui/02-chatbot.mdx:86](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/content/docs/04-ai-sdk-ui/02-chatbot.mdx#L86); [content/docs/04-ai-sdk-ui/02-chatbot.mdx:91](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/content/docs/04-ai-sdk-ui/02-chatbot.mdx#L91); [content/docs/04-ai-sdk-ui/03-chatbot-message-persistence.mdx:12](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/content/docs/04-ai-sdk-ui/03-chatbot-message-persistence.mdx#L12) (verified)
  - *To reach the next level:* A default that narrows authority (e.g. signed approvals and per-request principal binding on by default).
- **B L0:** Nothing in the framework bounds what the server-side tools can do; the docs present payments, deletions and external API calls as normal tool uses. — [content/docs/04-ai-sdk-ui/03-chatbot-tool-usage.mdx:402](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/content/docs/04-ai-sdk-ui/03-chatbot-tool-usage.mdx#L402) (verified)
  - *To reach the next level:* Framework-enforced bounds on what a hijacked or confused-deputy tool call can reach.
- **Cap:** none

### C2 Approval gates — 0.38 (high)

A real per-call approval flow exists: the server can mark tools as needing user approval (statically, per tool, or by a function of the input), the browser receives the exact tool input in an approval-requested state, and denials are first-class. The server re-validates approved inputs against the tool schema and can verify an HMAC signature that binds the approval to the exact call. But approval is off by default: unless the developer configures it, every tool the model calls executes immediately. Signature checking is a separate experimental opt-in; without it, approvals are just fields in the client-posted history and a modified client can fabricate them. Client-side tools run in the browser through developer callbacks with no SDK gate.

- **S L3:** Per-call approval with the exact input exposed to the approver, policy tiers (approved/denied/user-approval, per tool or by function), and first-class denial. — [packages/ai/src/generate-text/resolve-tool-approval.ts:66](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/resolve-tool-approval.ts#L66); [packages/ai/src/ui/ui-messages.ts:321](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/ui/ui-messages.ts#L321); [packages/ai/src/ui/chat.ts:539](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/ui/chat.ts#L539); [packages/ai/src/generate-text/validate-tool-approvals.ts:126](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/validate-tool-approvals.ts#L126) (verified)
  - *To reach the next level:* The approved call is bound to the executed call only when the opt-in HMAC secret is set; there is no default binding.
- **C L2:** Server tools passed to streamText go through the same approval resolution, but without the opt-in secret a client-fabricated approval reaches execution, and client-side tools run in the browser with no SDK gate. — [packages/ai/src/generate-text/collect-tool-approvals.ts:54](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/collect-tool-approvals.ts#L54); [packages/ai/src/generate-text/validate-tool-approvals.ts:68](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/validate-tool-approvals.ts#L68); [packages/ai/src/generate-text/validate-tool-approvals.ts:158-168](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/validate-tool-approvals.ts#L158-L168); [packages/ai/src/ui/chat.ts:220](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/ui/chat.ts#L220) (verified)
  - *To reach the next level:* Every path must traverse the gate: approvals bound to server-issued requests by default and client-executed tools covered.
- **D L0:** Approval is opt-in: with no toolApproval/needsApproval configuration a tool call resolves to not-applicable and runs. — [packages/ai/src/generate-text/resolve-tool-approval.ts:109-110](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/resolve-tool-approval.ts#L109-L110); [packages/ai/src/generate-text/stream-text.ts:547](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/stream-text.ts#L547) (verified)
  - *To reach the next level:* Approval on by default for tools with side effects.
- **B L0:** No undo, rate limit or quantity bound on approved actions is provided by the framework; documented use cases include payments and deletions. — [content/docs/04-ai-sdk-ui/03-chatbot-tool-usage.mdx:402](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/content/docs/04-ai-sdk-ui/03-chatbot-tool-usage.mdx#L402) (verified)
  - *To reach the next level:* Checkpoints, previews or bounded quantities for consequential actions.
- **Cap:** G1 — The approval gate exists but is opt-in in the documented default route; tools without toolApproval/needsApproval run immediately.

### C3 Tool & action scoping — 0.53 (high)

Every tool call the model makes is parsed against the tool's declared input schema before execution, unknown tool names are rejected, and approvals replayed from client history are re-validated against the schema. That is typed validation, not allowlisting: the framework has no path, URL or quantity checks of its own, and what a tool can do is whatever the developer's execute function does. The developer chooses the tool set and can narrow it per step with activeTools.

- **S L2:** Typed schema validation (zod/JSON schema) of every model tool call; no allowlist, URL or path checks in the framework. — [packages/ai/src/generate-text/parse-tool-call.ts:256](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/parse-tool-call.ts#L256); [packages/ai/src/generate-text/parse-tool-call.ts:243](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/parse-tool-call.ts#L243) (verified)
  - *To reach the next level:* Allowlist validation in code (resolved paths, host allowlists, numeric bounds) as a framework primitive.
- **C L3:** Schema validation is central: all model tool calls go through parseToolCall and history-replayed approvals are re-validated, so every registered tool inherits it. — [packages/ai/src/generate-text/parse-tool-call.ts:256](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/parse-tool-call.ts#L256); [packages/ai/src/generate-text/validate-tool-approvals.ts:126](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/validate-tool-approvals.ts#L126) (verified)
  - *To reach the next level:* A central policy layer beyond schemas (argument allow/deny rules) that new tools inherit automatically.
- **D L2:** The SDK ships no tools; all developer-registered tools are active unless narrowed with activeTools. — [packages/ai/src/generate-text/stream-text.ts:526](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/stream-text.ts#L526) (verified)
  - *To reach the next level:* A read-only default tool set with write/exec requiring explicit enabling.
- **B L1:** A misused tool reaches whatever the developer's execute function reaches; the framework adds no scoping or quantity bounds. — [packages/ai/src/generate-text/resolve-tool-approval.ts:109-110](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/resolve-tool-approval.ts#L109-L110) (verified)
  - *To reach the next level:* Framework-level scoping or quantity bounds on tool effects.
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

Nothing in the scoped code interprets model output as code: the hooks, the UI message stream and streamText contain no shell, eval or interpreter. streamText accepts an experimental sandbox handle but only passes it through to developer tools; sandbox packages are separate and outside this scope. Developers who add code-running tools get no isolation from AI SDK UI.

- **Structural absence:** searched `rg -n -S 'eval\(|new Function\(|child_process|execSync|spawn\(|execFile'` in `packages/react/src packages/vue/src packages/svelte/src packages/ai/src/ui packages/ai/src/ui-message-stream packages/ai/src/generate-text` → 2 hits (Both hits are in generate-text/generated-file-download.node.test.ts (a test); no code-execution path in the UI layer or streamText. experimental_sandbox on streamText is only passed through to developer tools.)
- **Notes:** Third-party HTML rendered by the experimental MCP App renderer is scored under C7.

### C5 Untrusted input blast radius — 0.00 (high)

Tool results, MCP tool output and any fetched content enter the model's context as ordinary tool messages with nothing marking them as untrusted, and nothing in the framework changes what the model may do after reading them. With approval off by default, an injected instruction can drive any registered tool, including ones that send data out. The one structural help is that client-posted system-role messages are rejected by default. The MCP App bridge can forward app-originated chat messages to the host if the developer wires that handler.

- **S L0:** No provenance tracking, taint or Rule-of-Two enforcement; tool output is fed back as normal tool messages. — searched `rg -n -i 'untrusted|taint|provenance'` in `packages/ai/src/ui packages/ai/src/ui-message-stream packages/ai/src/generate-text` → 0 hits (No untrusted-content handling in the UI layer or streamText.) (verified)
  - *To reach the next level:* Detection or provenance-driven gating once untrusted content has been read.
- **C L0:** Untrusted sources are not distinguished from the user's own messages. — [packages/ai/src/ui/convert-to-model-messages.ts:160](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/ui/convert-to-model-messages.ts#L160) (verified)
  - *To reach the next level:* Distinguishing at least one untrusted source (tool results) in code.
- **D L0:** No control exists to be on by default. — [packages/ai/src/generate-text/resolve-tool-approval.ts:109-110](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/resolve-tool-approval.ts#L109-L110); [packages/ai/src/generate-text/stream-text.ts:547](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/stream-text.ts#L547) (verified)
  - *To reach the next level:* An on-by-default limit on tools after untrusted content is read.
- **B L0:** A hijacked session can both leak data through developer tools and take irreversible actions with no human involved, since approval is off by default. — [packages/ai/src/generate-text/resolve-tool-approval.ts:109-110](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/resolve-tool-approval.ts#L109-L110); [packages/ai/src/generate-text/stream-text.ts:547](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/stream-text.ts#L547); [content/docs/04-ai-sdk-ui/03-chatbot-tool-usage.mdx:402](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/content/docs/04-ai-sdk-ui/03-chatbot-tool-usage.mdx#L402) (verified)
  - *To reach the next level:* Egress and irreversible actions require human approval by default.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.25 (medium)

The UI layer keeps no memory store of its own, but its core loop re-injects the whole client-held conversation, including past tool results, into the model every turn, and the documented persistence pattern saves that history server-side for later sessions. The server trusts that history: assistant turns, tool results and approvals in it are taken as genuine unless the developer opts into approval signing or validateUIMessages. One real guard: system-role messages posted by the client are rejected by default. The persistence guide keys chats by a client-supplied id and explicitly leaves authorization out.

- **S L1:** System-role messages in posted history are rejected by default, but all other history (assistant turns, tool results) is accepted as genuine with no integrity check. — [packages/ai/src/prompt/standardize-prompt.ts:35](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/prompt/standardize-prompt.ts#L35); [packages/ai/src/prompt/standardize-prompt.ts:89](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/prompt/standardize-prompt.ts#L89); [packages/ai/src/generate-text/tool-approval-signature.ts:139](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/tool-approval-signature.ts#L139) (verified)
  - *To reach the next level:* Provenance on persisted history and integrity protection beyond the opt-in approval signature.
- **C L1:** Only the system-role path is controlled; replayed tool results and assistant turns are not. — [packages/ai/src/ui/convert-to-model-messages.ts:90](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/ui/convert-to-model-messages.ts#L90); [packages/ai/src/ui/convert-to-model-messages.ts:160](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/ui/convert-to-model-messages.ts#L160) (verified)
  - *To reach the next level:* Control over the main history path (tool results, assistant turns) as well.
- **D L1:** Chat state is per Chat instance in the browser, but any server persistence is the developer's; the documented store is keyed by client-supplied chat id with authorization explicitly out of scope. — [content/docs/04-ai-sdk-ui/03-chatbot-message-persistence.mdx:12](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/content/docs/04-ai-sdk-ui/03-chatbot-message-persistence.mdx#L12) (inferred)
  - *To reach the next level:* Per-user namespaces enforced by the framework when history is loaded.
- **B L1:** Injected text that lands in persisted history is replayed into later sessions of the same chat and can trigger tool use. — [content/docs/04-ai-sdk-ui/02-chatbot.mdx:86](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/content/docs/04-ai-sdk-ui/02-chatbot.mdx#L86); [content/docs/04-ai-sdk-ui/02-chatbot.mdx:91](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/content/docs/04-ai-sdk-ui/02-chatbot.mdx#L91) (inferred)
  - *To reach the next level:* Session-scoped history, or persisted history that is easily inspected and purged.
- **Cap:** none

### C7 Third-party extensions — 0.35 (medium)

The default chat setup loads no third-party code. @ai-sdk/react does export an experimental MCP App renderer that runs HTML supplied by an MCP server inside a double iframe; the inner frame has no same-origin access, tool calls from the app are denied unless the host allow-lists them, and device permissions are deny-by-default. The HTML is whatever the server serves at render time, with no pinning or integrity check, and a restrictive content security policy is only built when the server itself supplies CSP metadata. Enforcement of the inner sandbox depends on the developer-hosted proxy page.

- **S L1:** MCP App HTML is fetched from the developer-chosen MCP server at render time with no version pin or integrity check. — [packages/react/src/mcp-apps/app-renderer.tsx:55](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/react/src/mcp-apps/app-renderer.tsx#L55); [packages/react/src/mcp-apps/app-frame.tsx:124](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/react/src/mcp-apps/app-frame.tsx#L124) (verified)
  - *To reach the next level:* Pinned versions of the app resources.
- **C L1:** No extension type is verified; the renderer is the only extension surface in scope and it is unverified. — [packages/react/src/mcp-apps/index.ts:1](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/react/src/mcp-apps/index.ts#L1) (verified)
  - *To reach the next level:* Verification of the one extension type in scope.
- **D L2:** Nothing third-party runs unless the developer renders the experimental component, but the content that will run is not shown or fixed. — [packages/react/src/mcp-apps/index.ts:1](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/react/src/mcp-apps/index.ts#L1) (verified)
  - *To reach the next level:* Showing or pinning exactly what will run when an extension is added.
- **B L2:** App HTML runs in an opaque-origin inner iframe with a deny-by-default tool allowlist, but network egress is unrestricted when the server provides no CSP metadata, and the inner sandbox is applied by the developer's proxy page. — [packages/react/src/mcp-apps/sandbox.ts:55](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/react/src/mcp-apps/sandbox.ts#L55); [packages/react/src/mcp-apps/sandbox.ts:72](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/react/src/mcp-apps/sandbox.ts#L72); [packages/react/src/mcp-apps/bridge.ts:365](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/react/src/mcp-apps/bridge.ts#L365) (inferred)
  - *To reach the next level:* Network limited to declared origins regardless of server metadata, enforced by the SDK rather than the host's proxy.
- **Cap:** none
- **Notes:** The @ai-sdk/mcp client (connecting MCP servers as tools) is outside this scope.

### C8 Secrets & sensitive-data protection — 0.45 (high)

Provider API keys stay on the server and never go to the browser in the documented setup. Errors from local tools and the stream are masked to a generic message before reaching the client by default, telemetry sends nothing unless an integration is registered, and per-request context values are excluded from telemetry unless explicitly included. There is no redaction of what goes to the model provider, and the provider keys are long-lived environment variables.

- **S L2:** Error masking on the client-bound stream and default exclusion of runtime/tool context from telemetry. — [packages/ai/src/ui-message-stream/to-ui-message-stream.ts:29](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/ui-message-stream/to-ui-message-stream.ts#L29); [packages/ai/src/ui-message-stream/to-ui-message-chunk.ts:323](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/ui-message-stream/to-ui-message-chunk.ts#L323); [packages/ai/src/telemetry/filter-included-context.ts:29](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/telemetry/filter-included-context.ts#L29) (verified)
  - *To reach the next level:* Redaction before model-bound messages, and a secret store rather than environment variables.
- **C L2:** Client-bound errors and telemetry context are covered; model-bound messages are not. — [packages/ai/src/ui-message-stream/to-ui-message-stream.ts:29](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/ui-message-stream/to-ui-message-stream.ts#L29); [packages/ai/src/telemetry/filter-included-context.ts:29](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/telemetry/filter-included-context.ts#L29) (verified)
  - *To reach the next level:* Coverage of model-bound messages and tool results.
- **D L2:** Telemetry has no integrations by default; error masking is on but replaced by any custom onError. — [packages/ai/src/telemetry/telemetry-registry.ts:14](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/telemetry/telemetry-registry.ts#L14); [packages/ai/src/telemetry/create-telemetry-dispatcher.ts:133](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/telemetry/create-telemetry-dispatcher.ts#L133) (verified)
  - *To reach the next level:* Masking that cannot be silently replaced and content-free telemetry when enabled.
- **B L1:** Provider keys are long-lived environment variables held by the server process that runs every tool. — [packages/provider-utils/src/load-api-key.ts:30](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/provider-utils/src/load-api-key.ts#L30) (verified)
  - *To reach the next level:* Scoped or short-lived credentials.
- **Cap:** none

### C9 Audit & traceability — 0.35 (medium)

The framework emits structured lifecycle events, including the start and end of every tool execution, to telemetry integrations and callbacks, and the browser holds a structured record of each call in the message parts. None of it is kept by default: no integration is registered, so nothing is recorded server-side unless the developer adds one. Approvals and denials are not separate audit events, and client-side tool runs are only seen by the server as results in the next request.

- **S L2:** Structured per-tool events (onToolExecutionStart/End) with arguments and results via telemetry integrations. — [packages/ai/src/telemetry/telemetry.ts:108](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/telemetry/telemetry.ts#L108) (verified)
  - *To reach the next level:* Actor attribution (requesting principal, approver) in the record.
- **C L2:** All server-executed tools emit events; approvals/denials and client-executed tools are not separately recorded. — [packages/ai/src/telemetry/telemetry.ts:108](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/telemetry/telemetry.ts#L108) (verified)
  - *To reach the next level:* Recording approvals, denials and client-side tool runs.
- **D L0:** Recording is opt-in: no telemetry integration is registered by default. — [packages/ai/src/telemetry/telemetry-registry.ts:14](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/telemetry/telemetry-registry.ts#L14) (verified)
  - *To reach the next level:* On-by-default recording outside the agent's control.
- **B L1:** Telemetry callbacks are best-effort; tool execution does not depend on the record being written. — [packages/ai/src/telemetry/create-telemetry-dispatcher.ts:133](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/telemetry/create-telemetry-dispatcher.ts#L133) (inferred)
  - *To reach the next level:* Errors surfaced and records flushed per action.
- **Cap:** G1 — Telemetry recording requires registering an integration; nothing is recorded by default.

### C10 Limits & kill switch — 0.30 (high)

Each server request runs one model step by default, which keeps a single request small. But the multi-step loop in AI SDK UI runs from the browser: with sendAutomaticallyWhen set, as the docs recommend for tools and approvals, the client resubmits after every completed tool round with no cap. Timeouts and token limits are opt-in. The stop() helper aborts the browser request, but the server keeps running the model call and any in-flight tool unless the developer forwards the request's abort signal, which the default route does not.

- **S L1:** A per-request step cap (one step) exists; timeouts and token caps are optional and the client-driven resubmission loop has no counter. — [packages/ai/src/generate-text/stream-text.ts:423](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/stream-text.ts#L423); [packages/ai/src/ui/chat.ts:1055](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/ui/chat.ts#L1055) (verified)
  - *To reach the next level:* A wall-clock or token/cost cap enforced by default alongside the step cap.
- **C L1:** The step cap applies per server request only; automatic client resubmissions start a fresh budget every time. — [packages/ai/src/ui/chat.ts:1055](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/ui/chat.ts#L1055) (verified)
  - *To reach the next level:* Limits that span the whole client-driven conversation loop.
- **D L2:** Sensible per-request default (one step); operators can raise it, and auto-resubmission is opt-in. — [packages/ai/src/generate-text/stream-text.ts:423](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/generate-text/stream-text.ts#L423) (verified)
  - *To reach the next level:* A default cap on automatic resubmissions that the loop cannot reset.
- **B L1:** Stopping aborts only the browser fetch; the server continues the model call and tool execution unless req.signal is forwarded. — [packages/ai/src/ui/chat.ts:702](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/packages/ai/src/ui/chat.ts#L702); [content/docs/06-advanced/02-stopping-streams.mdx:47](https://github.com/vercel/ai/blob/08d7f0a75e1466d28c3b71c2be4c0d89d552acf9/content/docs/06-advanced/02-stopping-streams.mdx#L47) (verified)
  - *To reach the next level:* Stop propagating to the server and cancelling in-flight calls by default.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: tool results and MCP output re-entering context (packages/ai/src/ui/convert-to-model-messages.ts:160) · [B] sensitive data/systems: whatever the developer's tools and server credentials reach (packages/provider-utils/src/load-api-key.ts:30) · [C] state change / egress: developer tools run without approval by default (packages/ai/src/generate-text/resolve-tool-approval.ts:109) · Same default session? Yes

## Highest-impact improvements
1. Require approval by default for tools with an execute function (or at least for tools not marked read-only). — C2 D L0→L3, +0.150 before caps (Playbook 5)
2. Make signed approvals the default (derive a per-deployment secret) and refuse replayed approvals the server did not issue. — C1 C L0→L2, +0.150 before caps (Playbook 4)
3. Add a default cap on automatic resubmissions in AbstractChat (e.g. maxAutomaticSends) and forward the request abort signal in the default route helper. — C10 B L1→L2, +0.050 before caps (Playbook 3, step 3)
4. Mark tool-result content as untrusted and let toolApproval policies key off whether untrusted content has entered the session. — C5 S L0→L2, +0.150 before caps (Playbook 1)
5. Ship a default local audit integration that records tool executions, approvals and denials. — C9 D L0→L2, +0.100 before caps

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was installed, built or run.
- Scope is the UI layer (packages/react, packages/ai/src/ui, packages/ai/src/ui-message-stream) plus the server pieces of the documented default route (streamText, convertToModelMessages, toUIMessageStream); @ai-sdk/mcp, sandbox-*, harness-*, rsc and provider packages were not examined. Vue and Svelte bindings were not read individually; they wrap the shared AbstractChat core.
- Version: the pinned commit is untagged; version is git describe output 28 commits after the @ai-sdk/react@4.0.130 release tag.
- C6 D/B, C7 B and C9 B are inferred from source and docs; developer-hosted pieces (MCP App sandbox proxy, chat persistence) are outside the SDK.
- No reviewer-directed instructions were found in the files read.
