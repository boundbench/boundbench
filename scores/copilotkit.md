# Defense-in-Depth Score: CopilotKit

**Repo:** https://github.com/CopilotKit/CopilotKit · **Commit:** `7f40d355a449f28adc527680fe81c2803df7b387` · **Reviewed:** 2026-10-05
**What it is:** Frontend stack for in-app agents and generative UI; AG-UI protocol
**Category:** Agent Frameworks
**Scored configuration:** CopilotRuntime with default options (in-memory runner, no hooks) serving a BuiltInAgent with classic default arguments, used from the React CopilotKit provider and CopilotChat; MCP servers, MCP Apps, Open Generative UI, Intelligence and channels scored at their own defaults when enabled.
**Agent surface (default):** code execution opt-in · filesystem write no · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents no · external communication opt-in

## Score: 2.5 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C2 | Approval gates | L1 | L1 | L0 | L0 | 0.15 | G1 | **0.15** | High |
| C3 | Tool & action scoping | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |
| C4 | Code-execution isolation | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | Medium |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L2 | L2 | L0 | L2 | 0.40 | — | **0.40** | High |
| C7 | Third-party extensions | L1 | L0 | L2 | L2 | 0.28 | — | **0.28** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C9 | Audit & traceability | L2 | L2 | L1 | L0 | 0.35 | — | **0.35** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |


CopilotKit connects an agent to your app's UI and tools, and by default it adds almost no safety controls around them. Every tool the developer registers, in the browser or on the server, runs as soon as the model calls it; human-in-the-loop is a building block the developer has to wire into each action. The runtime endpoint has no built-in authentication and keeps all users' conversation history in one in-memory store. Deployers should put authentication and per-user checks in front of the runtime, build approvals into consequential tools, and set step and output limits before exposing an agent to untrusted content.

## Critical gaps
- Nothing bounds a hijacked agent: untrusted tool and MCP results share context with private app data and with tools that act, and no tool is gated by default. (ASI01, LLM01; C5) — [packages/runtime/src/agent/index.ts:2105](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L2105); [packages/core/src/core/run-handler.ts:1176](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/core/src/core/run-handler.ts#L1176)

## Criterion details

### C1 Identity & least privilege — 0.05 (high)

CopilotKit gives the agent no identity of its own. The built-in agent reads model-provider keys from the server's environment, and server-side tools run inside the developer's process with whatever credentials that process holds. The runtime endpoint has no built-in authentication; developers add it through request hooks or middleware. Per-user authorization of thread data only exists when the hosted Intelligence backend is configured. A hijacked agent can do whatever the developer's tools and keys allow.

- **S L0:** Provider keys come from process environment variables and server tools run with the host's ambient credentials; nothing issues scoped or per-tool identities. — [packages/runtime/src/agent/index.ts:314](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L314); searched `rg -n -S -e 'downscop|tokenExchange|token_exchange|onBehalfOf'` in `packages/runtime/src/agent packages/runtime/src/v2` → 0 hits (no credential-narrowing primitive) (verified)
  - *To reach the next level:* No dedicated, role-scoped identity for the agent or its tools.
- **C L0:** No authorization layer sits in front of tool execution; access control on runtime routes is whatever the developer puts in beforeRequestMiddleware or hooks. — [packages/runtime/src/v2/runtime/core/runtime.ts:179](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/core/runtime.ts#L179); [packages/runtime/src/agent/index.ts:925](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L925) (verified)
  - *To reach the next level:* No authorization check that every tool path passes through.
- **D L0:** The default runtime ships without authentication and with the in-memory runner, which has no notion of a requesting user; per-user checks appear only on the Intelligence path. — [packages/runtime/src/v2/runtime/core/runtime.ts:537](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/core/runtime.ts#L537); [packages/runtime/src/v2/runtime/handlers/handle-stop.ts:56](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/handlers/handle-stop.ts#L56) (verified)
  - *To reach the next level:* No authenticated, least-privilege default; per-user scoping requires configuring the Intelligence backend or custom middleware.
- **B L1:** A hijacked agent holds every credential the developer wired into server tools plus the model-provider keys; the framework adds no boundary around them. — [packages/runtime/src/agent/index.ts:314](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L314); [packages/runtime/src/agent/index.ts:925](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L925) (verified)
  - *To reach the next level:* Credentials are long-lived and as broad as the developer configures; nothing narrows them to one system or tenant.
- **Cap:** none

### C2 Approval gates — 0.15 (high)

CopilotKit's human-in-the-loop support is a building block, not a gate. A developer can register a human-in-the-loop tool whose UI shows the call's arguments and returns the user's answer, or mark a server tool as an interrupt that pauses the run. Neither wraps the execution of other tools: ordinary frontend tools, server tools and MCP tools run as soon as the model calls them, and the built-in agent never sets the AI SDK's approval flag. Nothing is gated unless the developer designs it that way.

- **S L1:** The human-in-the-loop primitive returns the user's response as a tool result; whether a later action proceeds is then up to the model unless the developer performs the action inside the approval UI. — [packages/core/src/types.ts:108](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/core/src/types.ts#L108); [packages/runtime/src/agent/index.ts:457](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L457); [packages/react-core/src/v2/hooks/use-human-in-the-loop.tsx:44](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/react-core/src/v2/hooks/use-human-in-the-loop.tsx#L44) (verified)
  - *To reach the next level:* No per-call approval bound to execution of the exact approved call, with risk tiers deciding what needs a human.
- **C L1:** Only tools the developer builds as human-in-the-loop or interrupt tools involve a human; frontend handlers, server tools and MCP tools execute directly. — [packages/core/src/core/run-handler.ts:1176](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/core/src/core/run-handler.ts#L1176); [packages/runtime/src/agent/index.ts:2105](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L2105) (verified)
  - *To reach the next level:* Server tools and MCP tools have no path through any gate.
- **D L0:** No tool requires approval by default. — [packages/runtime/src/agent/index.ts:2105](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L2105); [packages/runtime/src/agent/index.ts:933](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L933) (verified)
  - *To reach the next level:* Approval is opt-in per tool.
- **B L0:** The framework offers no undo, checkpoint, preview or rate limit for tool actions; consequences are whatever the developer's tools do. — [packages/runtime/src/agent/index.ts:925](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L925); [packages/core/src/core/run-handler.ts:1176](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/core/src/core/run-handler.ts#L1176) (verified)
  - *To reach the next level:* No rollback or bounded, reversible action model for the common case.
- **Cap:** G1 — Human-in-the-loop and interrupt tools exist but apply only to tools the developer explicitly builds that way.

### C3 Tool & action scoping — 0.40 (high)

Tool arguments are checked against their declared schemas: client-declared tools are validated with a JSON Schema validator on the server, and Zod-defined server tools go through the AI SDK's schema check. Non-Zod server tool schemas are passed without a validator, and the browser only checks that arguments parse to an object. There are no path, URL or quantity allowlists in the framework, and the agent always receives a tool that can replace the entire shared application state. What a misused tool can reach is whatever the developer's handlers reach.

- **S L2:** Typed schema validation of tool arguments (JSON Schema for client tools, Zod via the AI SDK for server tools); no allowlists or bounds. — [packages/runtime/src/agent/index.ts:874](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L874); [packages/core/src/core/run-handler.ts:1142](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/core/src/core/run-handler.ts#L1142) (verified)
  - *To reach the next level:* No allowlist validation in code (resolved paths, host allowlists, numeric bounds) for tool arguments.
- **C L2:** Client tools and Zod server tools are validated; non-Zod server tool schemas are passed without a validator. — [packages/runtime/src/agent/index.ts:874](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L874); [packages/runtime/src/agent/index.ts:932](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L932) (verified)
  - *To reach the next level:* Not every tool is validated through one shared layer; non-Zod server tools and MCP tools are not covered by the framework's own validator.
- **D L1:** Every registered tool is offered to the model, and the built-in state snapshot and delta tools are always added. — [packages/runtime/src/agent/index.ts:1541](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L1541); [packages/runtime/src/agent/index.ts:1366](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L1366) (verified)
  - *To reach the next level:* No least-agency default or per-task tool allowlist; dangerous tools are on as soon as they are registered.
- **B L1:** Tools reach whatever the developer's handlers reach in the browser session or server process, plus full replacement of shared agent state. — [packages/runtime/src/agent/index.ts:1541](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L1541); [packages/core/src/core/run-handler.ts:1176](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/core/src/core/run-handler.ts#L1176) (verified)
  - *To reach the next level:* Nothing scopes or quantity-bounds tool effects.
- **Cap:** none

### C4 Code-execution isolation — 0.38 (medium)

The runtime has no server-side code execution: it launches no subprocesses and connects to MCP servers only over HTTP or SSE. Model-written code runs only in the browser through two opt-in features. Open Generative UI renders model-generated HTML and scripts inside a sandboxed iframe from a third-party sandbox library. MCP Apps renders HTML supplied by an MCP server in an iframe that is not held to the same origin isolation, and the server's resource metadata can widen that frame's content security policy. Once enabled, code from these paths can make network requests from the user's browser.

- **S L2:** Open Generative UI runs model-generated markup in a websandbox iframe with no extra sandbox attributes; the isolation itself comes from the third-party library. — [packages/react-core/src/v2/components/OpenGenerativeUIRenderer.tsx:233](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/react-core/src/v2/components/OpenGenerativeUIRenderer.tsx#L233); [packages/react-core/src/v2/components/OpenGenerativeUIRenderer.tsx:243](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/react-core/src/v2/components/OpenGenerativeUIRenderer.tsx#L243) (inferred)
  - *To reach the next level:* No verified capability-only runtime or hardened isolation primitive owned by the framework.
- **C L1:** The Open Generative UI path is sandboxed by origin; the MCP Apps widget path does not get the same opaque-origin separation from the host page. — [packages/react-core/src/v2/components/OpenGenerativeUIRenderer.tsx:233](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/react-core/src/v2/components/OpenGenerativeUIRenderer.tsx#L233); searched `rg -n -e 'sandbox'` in `packages/mcp-apps-renderer/src/sandbox.ts` → 10 hits (all hits are in the MCP Apps proxy document that hosts the widget frame (comments, attribute handling and message relay); see C4 summary) (verified)
  - *To reach the next level:* Not every browser execution path runs in an origin-isolated sandbox.
- **D L2:** Both features are opt-in runtime options and their sandboxing cannot be turned off by a flag, but an MCP Apps resource's own metadata extends the frame's script and frame sources. — [packages/runtime/src/v2/runtime/core/runtime.ts:109](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/core/runtime.ts#L109); [packages/runtime/src/v2/runtime/core/runtime.ts:107](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/core/runtime.ts#L107); [packages/mcp-apps-renderer/src/session.ts:637](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/mcp-apps-renderer/src/session.ts#L637) (verified)
  - *To reach the next level:* Sandbox policy is partly defined by data the MCP server supplies.
- **B L1:** Code that runs in these frames has unrestricted network access from the user's browser, and MCP Apps widgets are not confined away from the host application's origin. — [packages/mcp-apps-renderer/src/session.ts:637](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/mcp-apps-renderer/src/session.ts#L637); searched `rg -n -e 'sandbox'` in `packages/mcp-apps-renderer/src/sandbox.ts` → 10 hits (all hits are in the MCP Apps proxy document that hosts the widget frame (comments, attribute handling and message relay); see C4 summary) (verified)
  - *To reach the next level:* No egress restriction and no guaranteed separation from the host page's session.
- **Cap:** none
- **Notes:** Server-side execution surface verified absent: rg 'StdioClientTransport|child_process' over packages/runtime/src and packages/core/src returns 0 hits; both browser execution features are off unless configured.

### C5 Untrusted input blast radius — 0.00 (high)

Nothing in CopilotKit limits what a hijacked agent can do. Tool results, MCP results and application context enter the model's messages as ordinary content with no provenance or taint tracking, and every registered tool stays callable after untrusted content has been read. The one relevant default is that system-role messages sent from the browser are dropped. Because the framework's normal use combines private application context, untrusted content and tools that act, an injected instruction can both read data and trigger actions with no human involved.

- **S L0:** No structural limit: no taint tracking, no Rule-of-Two enforcement, no detection. — searched `rg -n -i -e 'untrusted|prompt.?injection'` in `packages/runtime/src/agent` → 0 hits (no provenance marking, taint tracking or injection handling in the agent); [packages/runtime/src/agent/index.ts:654](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L654) (verified)
  - *To reach the next level:* No approval or capability restriction tied to untrusted content entering the session.
- **C L0:** Untrusted sources are not distinguished; tool and MCP results enter context with the same standing as other messages. — searched `rg -n -i -e 'untrusted|prompt.?injection'` in `packages/runtime/src/agent` → 0 hits (no provenance marking, taint tracking or injection handling in the agent); [packages/runtime/src/agent/index.ts:1619](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L1619) (verified)
  - *To reach the next level:* No source of untrusted content is handled differently.
- **D L0:** There is no control to enable. — searched `rg -n -i -e 'untrusted|prompt.?injection'` in `packages/runtime/src/agent` → 0 hits (no provenance marking, taint tracking or injection handling in the agent) (verified)
  - *To reach the next level:* No untrusted-input control exists, on or off.
- **B L0:** In the documented use (app context shared with the agent, tools that act on the app or backends, untrusted tool and MCP results) a hijacked agent can read private data and take state-changing actions unattended. — [packages/runtime/src/agent/index.ts:1541](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L1541); [packages/core/src/core/run-handler.ts:1176](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/core/src/core/run-handler.ts#L1176); [packages/runtime/src/agent/index.ts:925](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L925) (verified)
  - *To reach the next level:* Reaching a higher level needs approval for egress and irreversible tools once untrusted content is read.
- **Cap:** C5-WORSTCASE — In the default configuration a hijacked agent can leak data and take irreversible actions with no human involved (applied automatically).

### C6 Memory, context & configuration integrity — 0.40 (high)

By default there is no long-term memory: conversation history lives in an in-memory store inside the runtime process, is evicted under memory limits and disappears on restart. The model has no tool to write persistent memory unless the hosted Intelligence features (user and project memories, learned skills) are configured. The default store is a single process-wide store keyed only by thread ID, with no per-user ownership. As a web framework it auto-loads no workspace instruction or config files.

- **S L2:** No model-writable persistent memory in the default configuration, and no workspace files are auto-loaded; learned skill tool names are reserved so other tools can't impersonate them. — [packages/runtime/src/v2/runtime/runner/in-memory.ts:127](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/runner/in-memory.ts#L127); [packages/runtime/src/agent/learned-skills.ts:33](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/learned-skills.ts#L33) (verified)
  - *To reach the next level:* No gating, validation or expiry on what enters persisted thread or memory content.
- **C L2:** Default thread history is bounded and non-durable; opt-in Intelligence memories and learned skills add persistent stores with policy hooks but no content validation. — [packages/runtime/src/v2/runtime/runner/in-memory.ts:127](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/runner/in-memory.ts#L127); [packages/runtime/src/v2/runtime/handlers/shared/memory-policy.ts:64](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/handlers/shared/memory-policy.ts#L64) (verified)
  - *To reach the next level:* Not all stores (thread history, memories, learned skills) are validated or provenance-tagged.
- **D L0:** The default runner keeps every user's threads in one process-wide store keyed only by thread ID, and the Intelligence memory policy defaults to read-write on both user and project scope when none is configured. — [packages/runtime/src/v2/runtime/runner/in-memory.ts:346](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/runner/in-memory.ts#L346); searched `rg -n -e 'userId|ownerId'` in `packages/runtime/src/v2/runtime/runner/in-memory.ts` → 0 hits (the default in-memory runner keys thread history by thread ID only); [packages/runtime/src/v2/runtime/handlers/shared/memory-policy.ts:64](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/handlers/shared/memory-policy.ts#L64) (verified)
  - *To reach the next level:* No per-user namespace enforced by default.
- **B L2:** In the default configuration poisoned content persists in a thread's in-memory history until eviction or restart, and that history is shared by whoever continues the thread, so it can steer later turns and their tool calls. — [packages/runtime/src/v2/runtime/runner/in-memory.ts:127](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/runner/in-memory.ts#L127); [packages/runtime/src/v2/runtime/runner/in-memory.ts:444](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/runner/in-memory.ts#L444); [packages/runtime/src/v2/runtime/runner/in-memory.ts:346](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/runner/in-memory.ts#L346) (verified)
  - *To reach the next level:* Thread history is not scoped to one user's session or easily inspected and purged per user.
- **Cap:** none

### C7 Third-party extensions — 0.28 (high)

Third-party extensions are MCP servers the developer lists in code, reached over HTTP or SSE, so their code runs elsewhere rather than inside the runtime. Nothing is enabled by default and the workspace can't add servers, but there is no pinning, integrity check or re-approval when a server's tools change. Tool descriptions and results from these servers go straight to the model. With MCP Apps enabled, a server can also ship HTML that runs in the user's browser, without the same isolation as model-generated UI.

- **S L1:** MCP servers are developer-chosen URLs with no version pinning, integrity check or change detection. — [packages/runtime/src/agent/index.ts:1092](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L1092); [packages/runtime/src/agent/index.ts:1619](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L1619) (verified)
  - *To reach the next level:* No pinning or integrity verification of extension tools and UI resources.
- **C L0:** No extension type is verified. — [packages/runtime/src/agent/index.ts:1619](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L1619); [packages/mcp-apps-renderer/src/session.ts:637](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/mcp-apps-renderer/src/session.ts#L637) (verified)
  - *To reach the next level:* Neither MCP tools nor MCP Apps resources are verified.
- **D L2:** No MCP server is configured by default and servers can only be added in the developer's code, but adding one shows nothing about what it exposes. — [packages/runtime/src/agent/index.ts:1092](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L1092) (verified)
  - *To reach the next level:* No review of the exact tools and permissions an MCP server brings when it is added.
- **B L2:** A malicious remote server runs out of process and receives only what the model sends it and its configured headers; its tool output enters the model context, and its MCP Apps HTML, when enabled, runs in the user's browser. — [packages/runtime/src/agent/index.ts:1619](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L1619); searched `rg -n -e 'sandbox'` in `packages/mcp-apps-renderer/src/sandbox.ts` → 10 hits (all hits are in the MCP Apps proxy document that hosts the widget frame (comments, attribute handling and message relay); see C4 summary) (verified)
  - *To reach the next level:* Extension UI and output are not confined per extension with declared network and data access.
- **Cap:** none
- **Notes:** No local MCP stdio launch exists (rg 'StdioClientTransport|child_process' over packages/runtime/src and packages/core/src: 0 hits).

### C8 Secrets & sensitive-data protection — 0.25 (high)

Model-provider keys are read from environment variables and stay on the server; the model has no tool that reads them. The runtime's error reporter strips authorization, cookie and API-key headers before handing request details to the developer's error handler, but there is no redaction of tool input, tool output or model-bound messages. Anonymous usage telemetry (runtime and model metadata, not prompts) is on by default and can be turned off with an environment variable.

- **S L1:** Secrets come from environment variables; one path (error reports) masks sensitive request headers. — [packages/runtime/src/agent/index.ts:314](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L314); [packages/runtime/src/v2/runtime/core/runtime-error-reporter.ts:43](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/core/runtime-error-reporter.ts#L43) (verified)
  - *To reach the next level:* No type-level masking or redaction filter on logs and tool I/O.
- **C L1:** Only the error-report path is protected; model-bound messages, tool results and in-memory thread history are not filtered. — [packages/runtime/src/v2/runtime/core/runtime-error-reporter.ts:43](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/core/runtime-error-reporter.ts#L43); searched `rg -n -i -e 'redact'` in `packages/runtime/src/agent` → 0 hits (no redaction of tool I/O or model-bound messages) (verified)
  - *To reach the next level:* Logs, transcripts and model-bound messages are not covered.
- **D L1:** Telemetry is on by default and content-free (runtime and model metadata); opt-out via COPILOTKIT_TELEMETRY_DISABLED or DO_NOT_TRACK. — [packages/runtime/src/v1-deprecated/lib/telemetry-disclosure.ts:43](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v1-deprecated/lib/telemetry-disclosure.ts#L43); [packages/runtime/src/v1-deprecated/lib/telemetry-disclosure.ts:21](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v1-deprecated/lib/telemetry-disclosure.ts#L21); [packages/runtime/src/v2/runtime/telemetry/events.ts:13](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/telemetry/events.ts#L13) (verified)
  - *To reach the next level:* Telemetry is not opt-in.
- **B L1:** Long-lived model-provider keys and whatever integration keys the developer supplies live in the server process. — [packages/runtime/src/agent/index.ts:314](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L314) (verified)
  - *To reach the next level:* Keys are long-lived; nothing scopes or rotates them.
- **Cap:** none

### C9 Audit & traceability — 0.35 (high)

The only built-in record of what the agent did is the in-memory thread history kept by the default runner: it holds the run's events, including tool calls and results, but it has no actor attribution, is evicted under memory limits and is lost when the process restarts. Request hooks, an error handler and a development-only debug event feed let developers build their own logging, but none of it is an audit trail by default.

- **S L2:** Run events, including tool call arguments and results, are stored per thread in structured form. — [packages/runtime/src/v2/runtime/runner/in-memory.ts:444](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/runner/in-memory.ts#L444) (verified)
  - *To reach the next level:* No actor attribution (requesting user, approver) or correlation across agents.
- **C L2:** Server-side tool calls (including MCP tools) appear in the run's event stream; frontend tool executions are recorded only as results sent back on the next run. — [packages/runtime/src/v2/runtime/runner/in-memory.ts:444](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/runner/in-memory.ts#L444); [packages/core/src/core/run-handler.ts:1176](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/core/src/core/run-handler.ts#L1176) (verified)
  - *To reach the next level:* Approvals and denials, and browser-side executions, are not recorded as such.
- **D L1:** On by default but held in the runtime's own memory, evicted under limits and cleared by a runtime operation. — [packages/runtime/src/v2/runtime/runner/in-memory.ts:127](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/runner/in-memory.ts#L127); [packages/runtime/src/v2/runtime/runner/in-memory.ts:554](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/runner/in-memory.ts#L554) (verified)
  - *To reach the next level:* Records are not stored outside the agent process.
- **B L0:** Records are lost on restart or crash and evicted silently after one warning. — [packages/runtime/src/v2/runtime/runner/in-memory.ts:127](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/runner/in-memory.ts#L127) (verified)
  - *To reach the next level:* Records are not durable per action.
- **Cap:** none

### C10 Limits & kill switch — 0.25 (high)

The built-in agent passes no step limit to the AI SDK unless the developer sets maxSteps, so the SDK default of one model step per request applies (more when learned skills are loaded); the browser then re-runs the agent after frontend tool calls up to a hard depth of 100. There are no token, cost or wall-clock limits and no timeouts on tool or model calls. A stop endpoint aborts the active run's model stream, and server tools receive the abort signal.

- **S L1:** Iteration caps only (AI SDK default step count on the server, 100 follow-up runs in the browser); no time or cost limit. — [packages/runtime/src/agent/index.ts:1377](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L1377); [packages/core/src/core/run-handler.ts:142](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/core/src/core/run-handler.ts#L142); searched `rg -n -i -e 'maxTokens|costLimit|budget'` in `packages/runtime/src/agent/index.ts` → 0 hits (no token or cost budget in the agent loop); searched `rg -n -i -e 'timeout'` in `packages/runtime/src/agent/index.ts` → 1 hits (the single hit is a comment about MCP server failures; no tool or model-call timeout) (verified)
  - *To reach the next level:* No per-execution timeout or token/cost cap enforced in code.
- **C L1:** Limits apply to the top-level loop only; tool calls and MCP calls have no timeouts. — [packages/core/src/core/run-handler.ts:142](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/core/src/core/run-handler.ts#L142); searched `rg -n -i -e 'timeout'` in `packages/runtime/src/agent/index.ts` → 1 hits (the single hit is a comment about MCP server failures; no tool or model-call timeout) (verified)
  - *To reach the next level:* No tool timeouts.
- **D L1:** The follow-up cap of 100 runs is very large, and the server step limit is whatever the developer sets. — [packages/core/src/core/run-handler.ts:142](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/core/src/core/run-handler.ts#L142); [packages/runtime/src/agent/index.ts:1377](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L1377) (verified)
  - *To reach the next level:* Defaults are not sensible ceilings on spend or time.
- **B L1:** A runaway can make up to 100 chained runs with no spend ceiling; stopping aborts the model stream but in-flight tool handlers decide for themselves whether to honour the signal. — [packages/runtime/src/v2/runtime/handlers/handle-stop.ts:99](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/v2/runtime/handlers/handle-stop.ts#L99); [packages/runtime/src/agent/index.ts:1700](https://github.com/CopilotKit/CopilotKit/blob/7f40d355a449f28adc527680fe81c2803df7b387/packages/runtime/src/agent/index.ts#L1700) (verified)
  - *To reach the next level:* No tight per-run time or cost ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: tool, MCP and remote-agent results enter model context unmarked (packages/runtime/src/agent/index.ts:1619) · [B] sensitive data/systems: application context and state shared with the agent, plus developer server tools (packages/runtime/src/agent/index.ts:925) · [C] state change / egress: frontend tool handlers and server tools run without approval (packages/core/src/core/run-handler.ts:1176); state snapshot tool always present (index.ts:1541) · Same default session? Yes

## Highest-impact improvements
1. Ship an approval option on BuiltInAgent and frontend tools that gates execution of the exact call (for example by setting the AI SDK's needsApproval) and make it the default for tools that change state. — C2 C L1→L3, +0.150 before caps (Playbook 5)
2. Require an identify-user hook on CopilotRuntime and scope the default runner's thread storage and endpoints to the requesting user. — C6 D L0→L2, +0.100 before caps (Playbook 4)
3. Set a default maxSteps, a lower browser follow-up depth, and timeouts on tool and MCP calls. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)
4. Make telemetry opt-in. — C8 D L1→L2, +0.050 before caps
5. Persist run events with user attribution to a durable store outside the runtime process by default. — C9 D L1→L2, +0.050 before caps (Playbook 1 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The monorepo is large; review focused on the TypeScript runtime (packages/runtime), core (packages/core), React bindings (packages/react-core), the MCP Apps renderer and the Open Generative UI renderer. The Python, Go, .NET and Ruby runtimes, the channels packages (Slack, Teams, Discord, Telegram, WhatsApp) and the deprecated v1 GraphQL runtime were not examined in depth.
- Behaviour of third-party libraries (AI SDK default step count and schema validation, @jetbrains/websandbox iframe attributes, Streamdown rendering defaults) is inferred from their documented or published code, not from the pinned repository.
- Remote agents connected through AG-UI (LangGraph, Mastra and others) carry their own controls, which are outside this score.
