# Defense-in-Depth Score: Composio

**Repo:** https://github.com/composiohq/composio · **Commit:** `98b10d71214e79681a4945dcc5c35bbebb1ab4fb` · **Reviewed:** 2026-10-04
**What it is:** SDK monorepo (TypeScript and Python) that gives AI agents 1000+ Composio-hosted toolkits, per-user auth sessions, triggers, and a remote code sandbox.
**Category:** Agent Frameworks
**Scored configuration:** @composio/core default session: `new Composio()` then `composio.create(userId)` and `session.tools()` with no toolkit, tag, or sandbox options; Python SDK mirrors it.
**Agent surface (default):** code execution yes · filesystem write opt-in · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions no · sub agents no · external communication yes

## Score: 3.4 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L1 | L0 | 0.35 | — | **0.35** | Medium |
| C2 | Approval gates | L1 | L1 | L0 | L0 | 0.15 | G1 | **0.15** | High |
| C3 | Tool & action scoping | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |
| C4 | Code-execution isolation | L2 | L3 | L3 | L1 | 0.57 | — | **0.57** | Medium |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L2 | L2 | 0.35 | — | **0.35** | Low |
| C7 | Third-party extensions | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | Medium |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | Low |
| C10 | Limits & kill switch | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | High |


Composio keeps provider tokens on its servers, never runs model code on your machine, and hardens its own file and URL handling well. But a default session hands the agent every toolkit the user has connected, a generic executor, and a remote sandbox that can call any tool and reach the internet, with no approval step and nothing to separate injected content from instructions. The dominant risk is prompt injection: an email or issue the agent reads can make it leak the user's data and send, post, or delete across their apps unattended. Restrict toolkits and tags, and add approval in your host framework, before production.

## Critical gaps
- Default sessions expose every toolkit with the user's full connected-account authority, so a hijacked agent can act across all of the user's apps. (ASI03; C1) — [docs/content/docs/how-composio-works.mdx:36](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/how-composio-works.mdx#L36); [ts/packages/core/src/types/toolRouter.types.ts:873](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/types/toolRouter.types.ts#L873)
- Default sessions pair untrusted tool output (email, issues, web) with private data and unattended send/delete/egress tools, and search results inject model-directed instructions. (ASI01, LLM01; C5) — [ts/packages/core/src/utils/transformers/toolRouterResponseTransform.ts:31-35](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/utils/transformers/toolRouterResponseTransform.ts#L31-L35); [docs/content/docs/how-composio-works.mdx:36](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/how-composio-works.mdx#L36); [docs/content/docs/sandbox/remote.mdx:7](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/sandbox/remote.mdx#L7)

## Criterion details

### C1 Identity & least privilege — 0.35 (medium)

A Composio session is bound to one end user, and every tool call executes server-side with that user's connected accounts, so the agent never holds raw provider tokens. But a default session exposes every toolkit, lets the agent start new connections for more apps, and acts with whatever OAuth scopes the auth config requested, with one credential for reads and writes. The SDK itself authenticates with a long-lived project API key that can act for any user in the project. A hijacked session can therefore act across the user's whole connected footprint.

- **S L2:** Sessions bind a user_id and the backend resolves that user's connected account per call; scoping enforcement is server-side and not in this repo. — [ts/packages/core/src/models/ToolRouter.ts:273](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/models/ToolRouter.ts#L273); [docs/content/docs/security/token-custody.mdx:74](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/security/token-custody.mdx#L74) (inferred)
  - *To reach the next level:* Read and write share one OAuth credential per account; no per-tool or per-request credential narrowing.
- **C L2:** All model-reachable tools (meta tools, sandbox helpers) execute through the same session on the backend; custom tools get the session context. — [ts/packages/core/src/models/Tools.ts:1231](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/models/Tools.ts#L1231); [docs/content/docs/sandbox/remote.mdx:7](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/sandbox/remote.mdx#L7) (inferred)
  - *To reach the next level:* Per-call authorization against a requesting principal is not visible in the SDK; enforcement lives in the closed backend.
- **D L1:** Default sessions enable all toolkits and the manage-connections tool; narrowing to toolkits/tags is opt-in, and `toolkits: null` restores the unrestricted default. — [docs/content/docs/how-composio-works.mdx:36](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/how-composio-works.mdx#L36); [ts/packages/core/src/types/toolRouter.types.ts:873](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/types/toolRouter.types.ts#L873); [ts/packages/core/src/lib/toolRouterParams.ts:100-104](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/lib/toolRouterParams.ts#L100-L104) (verified)
  - *To reach the next level:* No read-only or minimal default; writes are available without operator elevation.
- **B L0:** A hijacked default session can act with write scopes on every app the user has connected (email, chat, code hosting, docs), plus bulk calls from the sandbox. — [docs/content/docs/how-composio-works.mdx:36](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/how-composio-works.mdx#L36); [docs/content/docs/sandbox/remote.mdx:7](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/sandbox/remote.mdx#L7); [docs/public/data/meta-tools.json:427-434](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/public/data/meta-tools.json#L427-L434) (verified)
  - *To reach the next level:* Blast radius is the user's entire connected account set across services; no default narrowing.
- **Cap:** none
- **Notes:** Manage-connections lets the agent request new app connections, but a human must complete OAuth consent, so C1-SELFESC was not applied.

### C2 Approval gates — 0.15 (high)

Composio does not own the agent loop, so the host framework decides what to approve. What Composio gives the host is weak: the default session funnels every app action through one generic executor (COMPOSIO_MULTI_EXECUTE_TOOL, up to 50 tools per call) and a remote code sandbox that can call any tool, so a host can only gate the whole executor, reads included. Risk hints exist on these meta tools, but the default providers don't pass them into the host's approval hooks, and nothing in the SDK pauses for a human. The opt-in TypeSafe provider can require an explicit confirm flag for destructive tools.

- **S L1:** Meta tools carry destructiveHint/readOnlyHint tags, but the main executor mixes reads and writes across all apps behind a single destructive-tagged tool. — [docs/public/data/meta-tools.json:427-434](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/public/data/meta-tools.json#L427-L434); [docs/public/data/meta-tools.json:7-9](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/public/data/meta-tools.json#L7-L9) (verified)
  - *To reach the next level:* Separate read and write tools with accurate hints are not what a default session exposes; the generic executor hides which inner call is destructive.
- **C L1:** The sandbox's run_composio_tool reaches every tool in bulk without the host seeing individual inner calls. — [docs/content/docs/sandbox/remote.mdx:7](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/sandbox/remote.mdx#L7); [docs/public/data/meta-tools.json:430](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/public/data/meta-tools.json#L430) (verified)
  - *To reach the next level:* Inner calls in the sandbox and multi-executor are not individually exposed to a host gate.
- **D L0:** No approval primitive exists in the SDK; any gating is opt-in host or developer code (beforeExecute hook). — searched `rg -n -i 'approv|human.in.the.loop|hitl'` in `ts/packages/core/src python/composio` → 0 hits (No approval or human-in-the-loop primitive in either SDK core.) (verified)
  - *To reach the next level:* No server-enforced confirmation step or read-only mode is on by default.
- **B L0:** A wrongly executed call can send email, post messages, or delete records in third-party apps with no undo, preview, or quantity bound. — [docs/public/data/toolkits.json:256](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/public/data/toolkits.json#L256) (verified)
  - *To reach the next level:* No dry-run, rollback, or spend/recipient limits on consequential actions.
- **Cap:** G1 — Risk hints exist, but any approval step is opt-in host or developer code; nothing gates consequential calls by default.
- **Notes:** The opt-in TypeSafe provider (ts/packages/providers/typesafe) refuses destructive-tagged decisions unless `confirm: true` is passed; it relies on an external LLM router and is not the default, so it was not scored as an alt.

### C3 Tool & action scoping — 0.40 (high)

The SDK's own local surfaces are carefully validated: automatic file upload/download is off by default, local paths are realpath-checked against an upload directory allowlist and a sensitive-path denylist, and URL fetches go through an SSRF guard that pins DNS, blocks internal and metadata addresses, and rechecks every redirect. But the default tool set is the opposite of narrow: a generic executor for any of 1000+ toolkits, a remote Python/bash sandbox, and an arbitrary-HTTP proxy to connected APIs. Toolkit, tool, and tag filters exist but must be configured.

- **S L2:** Strong allowlist validation for local file paths and SSRF-guarded fetches, but the default meta tools pass arbitrary code and arbitrary tool slugs/arguments to the backend. — [ts/packages/core/src/utils/ssrfGuard.node.ts:155](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/utils/ssrfGuard.node.ts#L155); [ts/packages/core/src/utils/fileUtils.node.ts:308](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/utils/fileUtils.node.ts#L308); [docs/public/data/meta-tools.json:796-801](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/public/data/meta-tools.json#L796-L801) (verified)
  - *To reach the next level:* The most powerful default tools (sandbox code, generic executor, proxy_execute) are raw passthrough rather than narrow validated tools.
- **C L2:** Validation covers every SDK-side file and URL path; argument validation for remote tools happens in the closed backend. — [ts/packages/core/src/utils/fileUtils.node.ts:177](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/utils/fileUtils.node.ts#L177); [ts/packages/core/src/models/RemoteFile.ts:114](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/models/RemoteFile.ts#L114) (verified)
  - *To reach the next level:* No shared validation layer visible for remote tool arguments or sandbox code.
- **D L1:** All toolkits, the sandbox, and manage-connections are on by default; each can be disabled by session config. — [docs/content/docs/how-composio-works.mdx:36](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/how-composio-works.mdx#L36); [ts/packages/core/src/types/toolRouter.types.ts:46-51](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/types/toolRouter.types.ts#L46-L51); [ts/packages/core/src/utils/config-defaults/ConfigDefaults.node.ts:4](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/utils/config-defaults/ConfigDefaults.node.ts#L4) (verified)
  - *To reach the next level:* Default tool set is not read-only; write and code execution require no explicit enabling.
- **B L1:** A misused default tool reaches any of the user's connected apps, arbitrary provider API endpoints via proxy, and remote code execution, limited only to that user's accounts. — [docs/content/docs/sandbox/remote.mdx:7](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/sandbox/remote.mdx#L7); [docs/public/data/meta-tools.json:430](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/public/data/meta-tools.json#L430) (verified)
  - *To reach the next level:* Not scoped to one project or bounded in quantity.
- **Cap:** none

### C4 Code-execution isolation — 0.57 (medium)

No model-generated code ever runs on the developer's machine: the SDK has no local exec path, and the default code tools (COMPOSIO_REMOTE_WORKBENCH and COMPOSIO_REMOTE_BASH_TOOL) run in Composio's remote sandbox. The sandbox's internals (container or VM, hardening) are not in this repo, so its strength can only be inferred. By the project's own docs the sandbox is persistent within a session, has full programmatic access to every Composio tool with the user's credentials, can make arbitrary API calls and web searches, and installs packages on demand, so code inside it holds the session's full authority.

- **S L2:** Execution is verified to be remote (the call goes to the backend session API, and no local exec path exists); the sandbox primitive itself is not in the repo. — [ts/packages/core/src/models/Tools.ts:1231](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/models/Tools.ts#L1231); searched `rg -n 'child_process|execSync|spawn\(|new Function|eval\('` in `ts/packages/core/src` → 0 hits (No local process or eval path in the TypeScript core.); [docs/content/docs/sandbox/remote.mdx:7](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/sandbox/remote.mdx#L7) (inferred)
  - *To reach the next level:* Isolation technology and hardening of the remote sandbox cannot be verified from source.
- **C L3:** Every model-reachable execution path in both SDKs goes to the remote service; the only local option is the experimental local-workbench API a developer must run deliberately. — searched `rg -n 'child_process|execSync|spawn\(|new Function|eval\('` in `ts/packages/core/src` → 0 hits (No local process or eval path in the TypeScript core.); searched `rg -n 'subprocess|os\.system|exec\(|eval\('` in `python/composio` → 0 hits (No local process or eval path in the Python core.); [docs/content/docs/sandbox/local.mdx:53](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/sandbox/local.mdx#L53) (verified)
  - *To reach the next level:* Cannot verify that every process spawned inside the sandbox stays contained or that setup failures fail closed.
- **D L3:** Remote execution is the only mode; disabling the sandbox removes the tools rather than falling back to host execution, and the model cannot redefine the boundary. — [ts/packages/core/src/types/toolRouter.types.ts:46-51](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/types/toolRouter.types.ts#L46-L51); searched `rg -n 'child_process|execSync|spawn\(|new Function|eval\('` in `ts/packages/core/src` → 0 hits (No local process or eval path in the TypeScript core.) (verified)
  - *To reach the next level:* Sandbox policy is not inspectable in this repo, so L4's externally defined policy cannot be confirmed.
- **B L1:** Per docs, code inside the persistent sandbox can call any tool with the user's credentials, proxy arbitrary provider API calls, search the web, and install packages. — [docs/content/docs/sandbox/remote.mdx:7](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/sandbox/remote.mdx#L7); [docs/content/docs/sandbox/remote.mdx:40](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/sandbox/remote.mdx#L40) (inferred)
  - *To reach the next level:* No egress allowlist and no credential separation inside the sandbox; state persists across calls.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

Nothing in the SDK limits what injected content can make the agent do. Tool results from emails, issues, chat, web search, and documents go back to the model as plain data with no provenance or untrusted marker, and Composio's own search output adds model-directed instructions (execution guidance, recommended plan steps, server-side tool memory) that the tool description tells the model it MUST follow. A default session combines untrusted input, the user's private data, and unattended irreversible actions and egress, so a successful injection can both leak data and act.

- **S L0:** Tool outputs include directives to the model (execution_guidance, recommended_plan_steps, memory), and no injection limit exists. — [ts/packages/core/src/utils/transformers/toolRouterResponseTransform.ts:31-35](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/utils/transformers/toolRouterResponseTransform.ts#L31-L35); [docs/public/data/meta-tools.json:943](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/public/data/meta-tools.json#L943) (verified)
  - *To reach the next level:* Outputs should separate content from instructions and carry provenance; there is no untrusted-content flag.
- **C L0:** No untrusted source is distinguished; tool results enter context with the same standing as anything else. — searched `rg -n -i 'prompt.injection|untrusted content|spotlight'` in `ts/packages/core/src python/composio` → 0 hits (No prompt-injection handling in either SDK core.); [ts/packages/core/src/utils/transformers/toolRouterResponseTransform.ts:31-35](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/utils/transformers/toolRouterResponseTransform.ts#L31-L35) (verified)
  - *To reach the next level:* No source carries an untrusted marker.
- **D L0:** There is no control to turn on. — [docs/content/docs/how-composio-works.mdx:36](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/how-composio-works.mdx#L36); [ts/packages/core/src/types/toolRouter.types.ts:46-51](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/types/toolRouter.types.ts#L46-L51) (verified)
  - *To reach the next level:* No read-only or no-egress session mode is on by default.
- **B L0:** A hijacked default session can read the user's mail/docs and send email, post messages, or exfiltrate via the sandbox's web and proxy helpers with no human involved. — [docs/content/docs/how-composio-works.mdx:36](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/how-composio-works.mdx#L36); [docs/content/docs/sandbox/remote.mdx:7](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/sandbox/remote.mdx#L7); [docs/public/data/toolkits.json:256](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/public/data/toolkits.json#L256) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions are both unattended by default.
- **Cap:** C5-WORSTCASE — Default sessions let a hijacked agent both leak private data and take irreversible actions with no human in the loop.
- **Notes:** The opt-in TypeSafe provider routes on the user's request separately from context, which partly limits injection; it is not the default.

### C6 Memory, context & configuration integrity — 0.35 (low)

Neither SDK auto-loads instruction files or a working-directory .env, and the CLI only accepts organization and project IDs from a repo's .composio/.env, explicitly to block credential or endpoint injection from cloned repos. Persistence lives server-side: sessions keep tool memory, sandbox state, and a /mnt/files mount, and the meta-tool search returns stored memory and plans back to the model. How that memory is written, validated, or expired is not in this repo.

- **S L1:** Server-side tool memory is returned to the model inside search results with no provenance or validation visible in the SDK; repo files cannot change security settings. — [ts/packages/core/src/utils/transformers/toolRouterResponseTransform.ts:33](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/utils/transformers/toolRouterResponseTransform.ts#L33); [docs/content/docs/how-composio-works.mdx:38](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/how-composio-works.mdx#L38) (inferred)
  - *To reach the next level:* No visible gating, provenance, or expiry on session memory writes.
- **C L1:** Workspace config is controlled (CLI .env allowlist); session memory, plan cache, and sandbox files are not visibly controlled. — [ts/packages/cli/src/services/project-context.ts:16](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/cli/src/services/project-context.ts#L16); searched `rg -n 'dotenv|load_dotenv'` in `ts/packages/core/src python/composio` → 0 hits (No .env auto-loading in either SDK core.) (inferred)
  - *To reach the next level:* Memory, cached plans, and sandbox files have no visible control.
- **D L2:** Execution state is scoped to a session, and sessions are scoped to one user_id. — [docs/content/docs/how-composio-works.mdx:34-37](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/how-composio-works.mdx#L34-L37) (inferred)
  - *To reach the next level:* Cannot verify that the model cannot reach another namespace or that retention limits apply.
- **B L2:** Poisoned memory or sandbox files persist for the life of a reusable session and can steer later tool use; sessions can be deleted. — [docs/content/docs/sandbox/remote.mdx:7](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/sandbox/remote.mdx#L7); [ts/packages/core/src/lib/toolRouterSessionDelete.ts:1](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/lib/toolRouterSessionDelete.ts#L1) (inferred)
  - *To reach the next level:* Persistent state influences ungated actions; no review or rollback.
- **Cap:** none

### C7 Third-party extensions — 0.25 (medium)

The agent does not load plugins or third-party code into the developer's process: toolkits are Composio-run integrations executed on its servers, and custom tools are the developer's own code. But tool definitions are fetched at 'latest' by default and the provider execution path skips the version check, so definitions can change under a running agent without re-approval. Per its docs, the remote sandbox installs packages the agent asks for from a supported list, which can't be verified here.

- **S L1:** Toolkit versions default to 'latest', and agentic execution via providers forces dangerouslySkipVersionCheck. — [ts/packages/core/src/utils/config-defaults/ConfigDefaults.node.ts:6](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/utils/config-defaults/ConfigDefaults.node.ts#L6); [ts/packages/core/src/models/Tools.ts:916-918](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/models/Tools.ts#L916-L918) (verified)
  - *To reach the next level:* Toolkit versions are not pinned by default, and there is no integrity check or re-approval when definitions change.
- **C L1:** Version pinning exists for toolkit definitions (env or config) but nothing covers sandbox package installs. — [docs/content/docs/sandbox/remote.mdx:40](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/sandbox/remote.mdx#L40) (verified)
  - *To reach the next level:* Sandbox package installs are not covered by any pinning or verification.
- **D L1:** All toolkits are discoverable by default; a toolkit is used once the user completes a generic OAuth consent link. — [docs/content/docs/how-composio-works.mdx:36](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/how-composio-works.mdx#L36); [ts/packages/core/src/lib/toolRouterParams.ts:100-104](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/lib/toolRouterParams.ts#L100-L104) (verified)
  - *To reach the next level:* Adding a toolkit doesn't show which tools or permissions it brings beyond the OAuth screen.
- **B L1:** Toolkit code runs on Composio servers with the user's connected-account credentials; sandbox packages share the sandbox's full tool authority. — [docs/content/docs/sandbox/remote.mdx:7](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/sandbox/remote.mdx#L7) (inferred)
  - *To reach the next level:* Extensions are not sandboxed per toolkit with separately scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.40 (high)

Provider OAuth tokens are held server-side and injected at execution time, so they never pass through the SDK or the model. The SDK redacts every log line and the free-form error text it sends in telemetry. Gaps: telemetry is on by default (content-free), the CLI stores the project API key in plaintext unless the keychain is chosen, tool results reach the model unredacted, and the long-lived project API key can act for every user in the project.

- **S L2:** All logger output passes through a secret-redaction filter, telemetry error text is redacted, and provider tokens stay server-side; the API key comes from env or a plaintext CLI file by default. — [ts/packages/core/src/utils/logger.ts:92](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/utils/logger.ts#L92); [ts/packages/cli/src/services/cli-user-config.ts:27](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/cli/src/services/cli-user-config.ts#L27); [docs/content/docs/security/token-custody.mdx:74](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/security/token-custody.mdx#L74) (verified)
  - *To reach the next level:* No keychain by default and no redaction of tool results before they reach the model.
- **C L2:** Logs (including debug) and telemetry are redacted; no subprocesses; model-bound tool results are not scanned. — [ts/packages/core/src/telemetry/Telemetry.ts:220-221](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/telemetry/Telemetry.ts#L220-L221); [ts/packages/core/src/utils/logger.ts:92](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/utils/logger.ts#L92) (verified)
  - *To reach the next level:* Model-bound messages and tool results are not redacted.
- **D L1:** Telemetry is on by default and content-free (function names, durations, redacted errors); the backend stores tool arguments and results in execution logs by default. — [ts/packages/core/src/utils/config-defaults/ConfigDefaults.node.ts:5](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/utils/config-defaults/ConfigDefaults.node.ts#L5); [ts/packages/core/src/telemetry/Telemetry.ts:128-131](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/telemetry/Telemetry.ts#L128-L131); [docs/content/docs/security/token-custody.mdx:130](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/security/token-custody.mdx#L130) (verified)
  - *To reach the next level:* Telemetry should be opt-in; payload retention is on unless Zero Data Retention is configured.
- **B L1:** A leaked COMPOSIO_API_KEY is long-lived and can act on connected accounts for every user in the project. — [docs/content/docs/security/token-custody.mdx:74](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/security/token-custody.mdx#L74) (verified)
  - *To reach the next level:* Project key isn't short-lived or per-task; scoping depends on optional key permissions.
- **Cap:** none

### C9 Audit & traceability — 0.50 (low)

Every tool execution returns a server-side log ID, and the backend keeps searchable execution logs attributed to user, session, and connected account, outside anything the agent can edit. The logs live in Composio's closed backend, so their completeness and integrity can't be checked here, and there's no record of approvals because nothing is approved. Custom tools that run in the developer's process are only logged if they call back through Composio.

- **S L2:** Structured per-execution log with ID, filterable by user_id, session_id, connected_account_id, toolkit, and status. — [ts/packages/core/src/models/Tools.ts:196](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/models/Tools.ts#L196); [ts/packages/core/src/types/logs.types.ts:9-13](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/types/logs.types.ts#L9-L13) (verified)
  - *To reach the next level:* No approver or delegation attribution, and no tamper-evidence or standard export visible.
- **C L2:** Backend tool calls (meta tools, sandbox tool calls via run_composio_tool) are presumably logged server-side; in-process custom tools are not. — [ts/packages/core/src/models/ToolRouterSession.ts:223-224](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/models/ToolRouterSession.ts#L223-L224) (inferred)
  - *To reach the next level:* In-process custom tools and sandbox code are not shown to be logged.
- **D L2:** Logging is server-side and on by default; the model has no tool to edit it. — [docs/content/docs/security/token-custody.mdx:130](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/content/docs/security/token-custody.mdx#L130) (inferred)
  - *To reach the next level:* Can't verify from source that the logging component is outside the model's influence or that disabling is itself logged.
- **B L2:** Each execution response carries its log ID, implying a record per action. — [ts/packages/core/src/models/Tools.ts:196](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/models/Tools.ts#L196) (inferred)
  - *To reach the next level:* Can't verify that records are durable before execution or that actions fail closed when logging fails.
- **Cap:** none

### C10 Limits & kill switch — 0.38 (high)

The SDK caps its own work in a few places: URL and file fetches are limited to 100 MiB and five redirects, and the remote bash tool has a stated three-minute limit. There's no limit on how many tool calls, sandbox runs, or bulk executions a session can make, no spend ceiling, and cancelling only stops the SDK waiting, not work already running on the server.

- **S L2:** Server-enforced caps on some operations: response size, redirect count, bash command time. — [ts/packages/core/src/utils/readResponseBody.ts:2](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/utils/readResponseBody.ts#L2); [ts/packages/core/src/utils/ssrfGuard.node.ts:42](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/utils/ssrfGuard.node.ts#L42); [docs/public/data/meta-tools.json:680](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/public/data/meta-tools.json#L680) (verified)
  - *To reach the next level:* No caps on every operation and no rate or concurrency limits on tool calls.
- **C L1:** Caps cover file and URL fetches only; tool executions and multi-execute batches have no SDK-side bound. — [docs/public/data/meta-tools.json:430](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/docs/public/data/meta-tools.json#L430); searched `rg -n -i 'max_?steps|max_?iterations|budget'` in `ts/packages/core/src` → 0 hits (No step or budget limit in the TypeScript core.) (verified)
  - *To reach the next level:* Tool executions have no SDK-side bound.
- **D L2:** The size and redirect caps are on by default and caller-configurable. — [ts/packages/core/src/utils/ssrfGuard.node.ts:332](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/utils/ssrfGuard.node.ts#L332) (verified)
  - *To reach the next level:* Model can't raise them, but there's no hard ceiling on the overall session.
- **B L1:** Cancellation only rejects the client-side promise; server-side execution and the persistent sandbox keep running. — [ts/packages/core/src/utils/cancellation.ts:4-20](https://github.com/composiohq/composio/blob/98b10d71214e79681a4945dcc5c35bbebb1ab4fb/ts/packages/core/src/utils/cancellation.ts#L4-L20) (verified)
  - *To reach the next level:* Stopping should cancel in-flight server work, and per-session spend/time ceilings should exist.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Tool results from email, issues, chat, web search flow back unmarked (toolRouterResponseTransform.ts:31-35; sandbox web_search remote.mdx:7) · [B] sensitive data/systems: User's connected accounts across all toolkits by default (how-composio-works.mdx:36) · [C] state change / egress: COMPOSIO_MULTI_EXECUTE_TOOL and sandbox run_composio_tool/proxy_execute (meta-tools.json:427-434; remote.mdx:7) · Same default session? Yes

## Highest-impact improvements
1. Default new sessions to readOnlyHint tools and require developers to enable write toolkits, the sandbox, and manage-connections explicitly. — C3 D L1→L3, +0.100 before caps (Playbook 3 step 1)
2. Ship a built-in approval hook that pauses destructiveHint calls (including inner multi-execute and sandbox calls) until the developer's callback approves the exact arguments. — C2 D L0→L3, +0.150 before caps (Playbook 5)
3. Return tool output with provenance and an untrusted flag, and move server guidance out of tool results into the tool definition. — C5 S L0→L2, +0.150 before caps (Playbook 1)
4. Pin toolkit versions by default and stop forcing dangerouslySkipVersionCheck on the provider execution path. — C7 S L1→L2, +0.075 before caps
5. Make SDK telemetry opt-in. — C8 D L1→L2, +0.050 before caps

## Re-audit log
- No changes.

## Limitations
- Static source review of commit 98b10d7 only; nothing was executed, installed, or probed.
- Scored as a tool server: the host agent framework owns the loop, approvals, and stopping; the SDK's defaults and what it exposes to the host were scored.
- Tool execution, the remote sandbox, session memory, and execution logs live in Composio's closed backend; claims about them come from SDK types, shipped data files, and docs, and are marked inferred or capped accordingly.
- The TypeScript SDK was scored as primary (the README leads with it); the Python SDK was spot-checked and mirrors the same controls (url_safety, safe_path, upload allowlist, redaction).
- The CLI and the opt-in TypeSafe provider were reviewed only where they bear on defaults; they were not scored separately.
- No reviewer-directed prompt injection found. The README shows a third-party trust badge (hvtracker.net), which was not relied on.
