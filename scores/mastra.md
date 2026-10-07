# Defense-in-Depth Score: Mastra

**Repo:** https://github.com/mastra-ai/mastra · **Commit:** `20c9a8cb991473e8644598ab8b1e5ee61cc6865b` · **Reviewed:** 2026-10-03
**What it is:** TypeScript framework for AI applications and agents
**Category:** Agent Frameworks
**Scored configuration:** @mastra/core Agent and Mastra instance with default constructor arguments; first-class features (Workspace with LocalFilesystem/LocalSandbox, Memory, MCPClient, bundled server) scored at their own defaults.
**Agent surface (default):** code execution opt-in · filesystem write opt-in · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents opt-in · external communication yes

## Score: 3.0 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L0 | L1 | 0.12 | — | **0.12** | High |
| C2 | Approval gates | L3 | L2 | L0 | L0 | 0.38 | G1 | **0.38** | High |
| C3 | Tool & action scoping | L2 | L2 | L1 | L0 | 0.35 | — | **0.35** | High |
| C4 | Code-execution isolation | L3 | L3 | L0 | L2 | 0.55 | G1 | **0.50** (alt) | High |
| C5 | Untrusted input blast radius | L1 | L1 | L0 | L0 | 0.15 | G1 | **0.15** (alt) | High |
| C6 | Memory, context & configuration integrity | L1 | L0 | L2 | L1 | 0.23 | — | **0.23** | High |
| C7 | Third-party extensions | L1 | L0 | L2 | L2 | 0.28 | — | **0.28** | High |
| C8 | Secrets & sensitive-data protection | L1 | L2 | L1 | L1 | 0.33 | — | **0.33** | High |
| C9 | Audit & traceability | L2 | L2 | L0 | L1 | 0.35 | G1 | **0.35** (alt) | Medium |
| C10 | Limits & kill switch | L2 | L1 | L0 | L1 | 0.28 | G1 | **0.28** (alt) | High |


Mastra has a well-built human-approval gate and opt-in sandboxing, but almost none of it is on by default. Tools run without approval, the default LocalSandbox executes shell commands directly on the host, and nothing limits what an agent hijacked through a tool result can do. Deployers should enable requireToolApproval, use bubblewrap/Seatbelt or a remote sandbox, configure server auth, and add timeouts before exposing an agent to untrusted content.

## Critical gaps
- The default LocalSandbox executes model-chosen shell commands directly on the host as the developer's OS user, with isolation 'none'. (ASI05; C4) — [packages/core/src/workspace/sandbox/local-sandbox.ts:119](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-sandbox.ts#L119); [packages/core/src/workspace/sandbox/local-process-manager.ts:303](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-process-manager.ts#L303)
- A hijacked agent can both leak data and take irreversible actions through any registered tool with no human involved, because approval is off by default and nothing tracks untrusted content. (ASI01, LLM01; C5) — [packages/core/src/tools/tool.ts:384](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/tools/tool.ts#L384); [packages/core/src/workspace/tools/tools.ts:153](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/tools/tools.ts#L153)

## Criterion details

### C1 Identity & least privilege — 0.12 (high)

Mastra has no agent identity of its own: tools run inside the developer's Node process and use whatever credentials that process holds, with no per-tool or per-request credential scoping. Two good narrowing measures exist: the local command sandbox and MCP stdio servers get a scrubbed environment rather than the full process environment. The bundled HTTP server relies on the developer configuring authentication, and its default access control is not locked down. A hijacked agent can do whatever the developer's integration credentials allow.

- **S L0:** Tools execute in-process with the host's ambient credentials; the framework issues no scoped or per-tool identity. — [packages/core/src/workspace/sandbox/local-process-manager.ts:276](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-process-manager.ts#L276); searched `rg -n -S -e 'downscop|token_exchange|tokenExchange|on_behalf'` in `packages/core/src/tools packages/core/src/agent` → 0 hits (no credential-narrowing primitive in the tool or agent layers) (verified)
  - *To reach the next level:* No dedicated, role-scoped identity for the agent or its tools.
- **C L1:** Subprocess paths narrow ambient authority (LocalSandbox passes only PATH plus configured env; MCP stdio gets the SDK's curated env), but in-process tools see the full process environment and no authorization layer sits in the tool executor. — [packages/core/src/workspace/sandbox/local-sandbox.ts:676](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-sandbox.ts#L676); [packages/mcp/src/client/client.ts:552](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/mcp/src/client/client.ts#L552) (verified)
  - *To reach the next level:* No authorization layer that every tool path (including in-process tools and sub-agents) passes through.
- **D L0:** Server authentication is not configured by default, and the default access-control behaviour is not locked down. (verified)
  - *To reach the next level:* No authenticated, least-privilege default; per-user scoping requires the developer to configure server auth and mapUserToResourceId.
- **B L1:** A hijacked agent holds every credential the developer wired into the process and its tools; the framework adds no boundary beyond the subprocess env scrub. — [packages/core/src/workspace/sandbox/local-sandbox.ts:255](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-sandbox.ts#L255); [packages/mcp/src/client/client.ts:552](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/mcp/src/client/client.ts#L552) (verified)
  - *To reach the next level:* Credentials are long-lived and as broad as the developer configures; nothing narrows them to one system or tenant.
- **Cap:** none

### C2 Approval gates — 0.38 (high)

Mastra has a well-built human approval mechanism: when enabled, a tool call suspends, the exact tool name and arguments are streamed to the client, and only a decision delivered through the workflow resume boundary (not model-written data) can approve or decline it. But it is off by default: every tool's requireApproval defaults to false, the agent-wide requireToolApproval is unset, and the workspace's file-write, delete and shell tools ship with approval disabled. Nothing in the framework provides undo for actions a tool takes.

- **S L3:** Per-call approval suspends the run and shows the exact tool name and arguments; approval comes only from the workflow resume data, and policy functions that throw fail to 'requires approval'. — [packages/core/src/loop/workflows/agentic-execution/tool-call-step.ts:703](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/loop/workflows/agentic-execution/tool-call-step.ts#L703); [packages/core/src/loop/workflows/agentic-execution/tool-call-step.ts:520](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/loop/workflows/agentic-execution/tool-call-step.ts#L520); [packages/core/src/loop/workflows/agentic-execution/tool-approval-verdict.ts:52-57](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/loop/workflows/agentic-execution/tool-approval-verdict.ts#L52-L57) (verified)
  - *To reach the next level:* No argument-level allow/deny/escalate policy layer shipped by the framework; policies are developer-written functions.
- **C L2:** When the global flag is set every registered tool (including MCP tools) passes the verdict, but a per-tool needsApprovalFn overrides the global seed, and unknown tools are not rejected. — [packages/core/src/loop/workflows/agentic-execution/tool-approval-verdict.ts:62-69](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/loop/workflows/agentic-execution/tool-approval-verdict.ts#L62-L69); [packages/core/src/loop/workflows/agentic-execution/tool-approval-verdict.ts:64](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/loop/workflows/agentic-execution/tool-approval-verdict.ts#L64) (verified)
  - *To reach the next level:* A per-tool or MCP server-level approval function can exempt calls that a global requireToolApproval: true meant to gate; no default-deny for unknown tools.
- **D L0:** Approval is opt-in: tools default to requireApproval false and workspace tools default to no approval. — [packages/core/src/tools/tool.ts:384](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/tools/tool.ts#L384); [packages/core/src/workspace/tools/tools.ts:153](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/tools/tools.ts#L153) (verified)
  - *To reach the next level:* Approval is not on by default for consequential tools.
- **B L0:** Registered tools (and the workspace delete/write/shell tools) can take irreversible actions with no framework-level undo, preview or rate limit. — [packages/core/src/workspace/tools/tools.ts:152](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/tools/tools.ts#L152); searched `rg -n -e 'checkpoint'` in `packages/core/src/tools/tool.ts packages/core/src/agent/agent.ts` → 0 hits (no checkpoint/rollback in the tool or agent layers) (verified)
  - *To reach the next level:* No checkpoint/rollback or dry-run for the common case.
- **Cap:** G1 — The approval gate exists but tools default to requireApproval false and requireToolApproval is unset.

### C3 Tool & action scoping — 0.35 (high)

Every Mastra tool built with createTool gets its input validated against its schema before it runs, and MCP tool schemas are converted so they are validated too; tools that arrive with a plain JSON schema (e.g. Vercel AI SDK tools) skip validation. The workspace filesystem tools are well scoped: paths are confined to the workspace by default with symlink-aware realpath checks. But the workspace also ships a free-form shell tool whose command string goes to the host shell, and all workspace tools are enabled by default once a workspace is attached.

- **S L2:** Typed schema validation for every standard-schema tool, plus realpath containment in LocalFilesystem; the execute_command tool accepts an arbitrary shell string. — [packages/core/src/workspace/filesystem/local-filesystem.ts:203](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/filesystem/local-filesystem.ts#L203); [packages/core/src/workspace/tools/execute-command.ts:18-19](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/tools/execute-command.ts#L18-L19); [packages/core/src/workspace/sandbox/local-process-manager.ts:303](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-process-manager.ts#L303) (verified)
  - *To reach the next level:* General tools are not replaced by narrow ones; the shell tool is raw passthrough and no URL/host allowlists apply to tools.
- **C L2:** createTool and MCP tools go through the shared validator, but tools with non-standard (plain JSON) schemas bypass it. — [packages/core/src/tools/validation.ts:497](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/tools/validation.ts#L497) (verified)
  - *To reach the next level:* Validation is not applied to every tool type through one central policy layer.
- **D L1:** A bare Agent has no tools, but attaching a Workspace enables all its tools, including write, delete and shell, by default; each can be disabled individually. — [packages/core/src/workspace/tools/tools.ts:152](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/tools/tools.ts#L152) (verified)
  - *To reach the next level:* Write/exec workspace tools are not off by default; no read-only default tool group.
- **B L0:** With the default LocalSandbox, the shell tool runs any command on the host as the developer's OS user, including in an absolute cwd outside the workspace. — [packages/core/src/workspace/sandbox/local-sandbox.ts:237](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-sandbox.ts#L237); [packages/core/src/workspace/sandbox/local-process-manager.ts:231-232](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-process-manager.ts#L231-L232) (verified)
  - *To reach the next level:* Misused tools are not limited to the workspace or bounded in quantity.
- **Cap:** none

### C4 Code-execution isolation — 0.50 (high)

Agents only execute code if the developer attaches a Workspace with a sandbox, but the default sandbox, LocalSandbox, runs commands directly on the host through the system shell with isolation set to 'none'. Its environment is scrubbed to PATH plus configured variables, but the process still reaches the whole filesystem, the network, and anything else the OS user can. Opt-in isolation is solid: bubblewrap or Seatbelt confine writes to the workspace, deny network by default, and refuse to start if the backend is missing; remote sandboxes (E2B, Daytona, Modal and others) are also available. Because those are opt-in, the criterion is capped.

- **default configuration** (default; raw 0.00, cap G1 → 0.00)
  - **S L0:** LocalSandbox defaults to isolation 'none' and spawns commands with the host shell as the current user. — [packages/core/src/workspace/sandbox/local-sandbox.ts:119](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-sandbox.ts#L119); [packages/core/src/workspace/sandbox/local-sandbox.ts:237](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-sandbox.ts#L237) (verified)
    - *To reach the next level:* No OS-level separation by default.
  - **C L0:** With no isolation, no execution path is sandboxed. — [packages/core/src/workspace/sandbox/local-process-manager.ts:303](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-process-manager.ts#L303) (verified)
    - *To reach the next level:* The main execution tool does not go through a sandbox by default.
  - **D L0:** Isolation is opt-in. — [packages/core/src/workspace/sandbox/local-sandbox.ts:237](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-sandbox.ts#L237) (verified)
    - *To reach the next level:* Sandboxing is not on by default.
  - **B L0:** Commands run with the OS user's full filesystem (home directory, ~/.ssh, ~/.aws) and network; same-user processes can also read the parent process's environment. — [packages/core/src/workspace/sandbox/local-sandbox.ts:119](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-sandbox.ts#L119); [packages/core/src/workspace/sandbox/local-process-manager.ts:231-232](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-process-manager.ts#L231-L232) (verified)
    - *To reach the next level:* Reachable state is host-equivalent for the OS user.
- **opt-in bubblewrap/Seatbelt isolation (LocalSandbox isolation: 'bwrap' | 'seatbelt')** (alt; raw 0.55, cap G1 → 0.50) ← counted
  - **S L3:** bwrap creates PID/IPC/UTS namespaces, unshares the network unless allowed, mounts system paths read-only and binds only the workspace read-write. — [packages/core/src/workspace/sandbox/native-sandbox/bubblewrap.ts:62-65](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/native-sandbox/bubblewrap.ts#L62-L65); [packages/core/src/workspace/sandbox/native-sandbox/bubblewrap.ts:104](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/native-sandbox/bubblewrap.ts#L104) (verified)
    - *To reach the next level:* Not kernel-separated (microVM/gVisor); no seccomp filter or resource limits in the profile.
  - **C L3:** Foreground and background commands are wrapped by the isolation backend, and construction fails if the backend is unavailable rather than falling back to the host. — [packages/core/src/workspace/sandbox/local-sandbox.ts:241-244](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-sandbox.ts#L241-L244); [packages/core/src/workspace/sandbox/local-sandbox.ts:1047-1049](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-sandbox.ts#L1047-L1049) (verified)
    - *To reach the next level:* MCP stdio servers and other extension processes are not sandboxed.
  - **D L0:** The backend must be chosen explicitly; the default is 'none'. — [packages/core/src/workspace/sandbox/local-sandbox.ts:237](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-sandbox.ts#L237) (verified)
    - *To reach the next level:* Isolation is not on by default.
  - **B L2:** Workspace mounted read-write, network off by default, scrubbed env; but no CPU/memory/PID limits and the workspace persists. — [packages/core/src/workspace/sandbox/native-sandbox/bubblewrap.ts:104](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/native-sandbox/bubblewrap.ts#L104); [packages/core/src/workspace/sandbox/local-process-manager.ts:276](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-process-manager.ts#L276) (verified)
    - *To reach the next level:* No CPU/memory/PID limits in the sandbox profile.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.15 (high)

Nothing in the framework structurally limits what a hijacked agent can do. Tool results, MCP results and fetched content enter the conversation the same way as any other message, and no taint tracking or Rule-of-Two enforcement disables egress or writes once untrusted content has been read. Mastra ships an optional LLM-based PromptInjectionDetector, but it only checks the incoming user messages, not tool results, and it is a classifier rather than a boundary. Because approval is off by default, an injected instruction can both leak data and take irreversible actions with no human in the loop.

- **default configuration** (default; raw 0.00, cap C5-WORSTCASE → 0.00)
  - **S L0:** No structural limit on a hijacked agent in the default configuration. — searched `rg -n -S -e 'taint|quarantin'` in `packages/core/src/loop packages/core/src/agent/agent.ts` → 0 hits (no taint tracking or quarantined-LLM design in the loop) (verified)
    - *To reach the next level:* No detection or approval tied to untrusted content by default.
  - **C L0:** Tool and MCP results are not distinguished from principal instructions. — searched `rg -n -S -e 'taint|quarantin'` in `packages/core/src/loop packages/core/src/agent/agent.ts` → 0 hits (verified)
    - *To reach the next level:* Untrusted sources are not distinguished.
  - **D L0:** No control is on by default. — [packages/core/src/tools/tool.ts:384](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/tools/tool.ts#L384) (verified)
    - *To reach the next level:* No default-on untrusted-input control.
  - **B L0:** With approval off, any registered egress or write tool can be driven by injected content to exfiltrate data and take irreversible actions unattended. — [packages/core/src/tools/tool.ts:384](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/tools/tool.ts#L384); [packages/core/src/workspace/tools/tools.ts:152-153](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/tools/tools.ts#L152-L153) (verified)
    - *To reach the next level:* Exfiltration and irreversible actions are not forced through human approval.
- **opt-in PromptInjectionDetector input processor** (alt; raw 0.15, cap G1 → 0.15) ← counted
  - **S L1:** An LLM classifier that can block/warn/rewrite detected injections. — [packages/core/src/processors/processors/prompt-injection-detector.ts:133](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/processors/processors/prompt-injection-detector.ts#L133) (verified)
    - *To reach the next level:* Detection only; no capability is removed once untrusted content is read.
  - **C L1:** Runs as an input processor on incoming messages only; tool results are not scanned. — [packages/core/src/processors/processors/prompt-injection-detector.ts:180](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/processors/processors/prompt-injection-detector.ts#L180) (verified)
    - *To reach the next level:* Tool results, MCP outputs and sub-agent messages are not covered.
  - **D L0:** Must be added explicitly as a processor. — [packages/core/src/processors/processors/prompt-injection-detector.ts:133](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/processors/processors/prompt-injection-detector.ts#L133) (verified)
    - *To reach the next level:* Not on by default.
  - **B L0:** Same worst case as the default: leak plus irreversible action unattended. — [packages/core/src/tools/tool.ts:384](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/tools/tool.ts#L384) (verified)
    - *To reach the next level:* Exfiltration and irreversible actions are not forced through human approval.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C6 Memory, context & configuration integrity — 0.23 (high)

Memory is opt-in per agent; the default Memory configuration persists the last 10 messages of each thread and replays them in later turns, with working memory and semantic recall off. Threads are tied to a resource (user) and the agent refuses to use a thread owned by a different resource. Nothing validates what is written: when working memory is enabled the model rewrites a free-text profile that is re-injected on every turn across all of that user's threads. Memory isolation is not a strict boundary.

- **S L1:** Persisted messages keep their roles, but writes are not validated; working memory (when enabled) is free text written by the model via a tool and re-injected. — [packages/core/src/memory/memory.ts:89-95](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/memory/memory.ts#L89-L95); [packages/memory/src/tools/working-memory.ts:224](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/memory/src/tools/working-memory.ts#L224) (verified)
  - *To reach the next level:* No provenance-as-data presentation or gating of memory writes.
- **C L0:** No memory path is validated or gated. — searched `rg -n -S -e 'taint|quarantin'` in `packages/core/src/loop packages/core/src/agent/agent.ts` → 0 hits (no validation on what flows into persisted history) (verified)
  - *To reach the next level:* Neither history, working memory nor semantic recall is controlled.
- **D L2:** Threads and working memory are namespaced per thread/resource and ownership is enforced, but namespace isolation is not a strict boundary. — [packages/core/src/agent/memory-thread-ownership.ts:22](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/agent/memory-thread-ownership.ts#L22) (verified)
  - *To reach the next level:* Namespace isolation is not fully enforced.
- **B L1:** Poisoned history persists in the thread across sessions and can trigger tool use in later turns; resource-scoped working memory extends that across the user's threads. — [packages/core/src/memory/memory.ts:90](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/memory/memory.ts#L90); [packages/memory/src/tools/working-memory.ts:224](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/memory/src/tools/working-memory.ts#L224) (verified)
  - *To reach the next level:* Poisoned memory is not limited to text output or gated actions.
- **Cap:** none

### C7 Third-party extensions — 0.28 (high)

Mastra loads third-party code mainly as MCP servers that the developer lists in code: stdio servers are launched from whatever command is given (often an unpinned npx package) with no version pinning, hash check, or re-approval when tool definitions change. Stdio servers receive only the MCP SDK's curated environment plus explicitly configured variables, not the full process environment. Nothing is enabled automatically. Access control on the optional editor's stored MCP configs is not locked down.

- **S L1:** Servers are developer-chosen but unpinned; no integrity or signature check. — searched `rg -n -e 'sha256|integrity|checksum'` in `packages/mcp/src/client/client.ts` → 0 hits (no integrity verification of launched servers) (verified)
  - *To reach the next level:* No version pinning or integrity verification.
- **C L0:** No extension type is verified. — searched `rg -n -e 'sha256|integrity|checksum'` in `packages/mcp/src/client/client.ts` → 0 hits (verified)
  - *To reach the next level:* No verification for any extension type.
- **D L2:** Nothing third-party is enabled by default; the developer adds servers explicitly in code, but the optional editor can add stdio servers from stored config. — [packages/editor/src/namespaces/mcp.ts:82-87](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/editor/src/namespaces/mcp.ts#L82-L87) (verified)
  - *To reach the next level:* Stored/editor-added extensions do not require a user- or admin-scoped consent step.
- **B L2:** Stdio servers run as separate processes with a scrubbed, curated environment. — [packages/mcp/src/client/client.ts:552](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/mcp/src/client/client.ts#L552) (verified)
  - *To reach the next level:* Extensions are not sandboxed per extension or limited in network/file access.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.33 (high)

Model and integration API keys come from environment variables and stay in the developer's process; Mastra keeps them out of subprocesses by scrubbing the LocalSandbox and MCP stdio environments. When observability is enabled, a SensitiveDataFilter that redacts fields like password, token and apiKey is applied by default to traces. Plain logs are not redacted. Anonymous usage telemetry (token counts and feature use, no prompt content) is sent to PostHog by default unless MASTRA_TELEMETRY_DISABLED is set.

- **S L1:** Keys are plain environment variables; masking exists on one path (field-name redaction on traces, which are opt-in) plus env scrubbing for subprocesses. — [observability/mastra/src/default.ts:144](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/observability/mastra/src/default.ts#L144); [packages/core/src/workspace/sandbox/local-process-manager.ts:276](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-process-manager.ts#L276) (verified)
  - *To reach the next level:* No type-level masking or log filters on the main logging path.
- **C L2:** Traces and subprocess environments are covered; the logger has no redaction and model-bound messages are not scrubbed. — searched `rg -n -e 'redact|mask'` in `packages/core/src/logger` → 0 hits (no redaction in the logger) (verified)
  - *To reach the next level:* Logs and model-bound messages are not protected.
- **D L1:** Content-free usage telemetry is on by default. — [packages/core/src/telemetry/posthog.ts:15-21](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/telemetry/posthog.ts#L15-L21) (verified)
  - *To reach the next level:* Telemetry is not opt-in.
- **B L1:** Long-lived provider and integration keys live in the process environment, reachable through host shell execution in the default sandbox. — [packages/core/src/workspace/sandbox/local-sandbox.ts:237](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-sandbox.ts#L237) (verified)
  - *To reach the next level:* Keys are not scoped or short-lived.
- **Cap:** none

### C9 Audit & traceability — 0.35 (medium)

Out of the box, Mastra installs a no-op observability layer and logs tool execution only at debug level, which the default logger does not print, so there is no record of what an agent did unless the developer configures it. With observability enabled, every tool and MCP tool call becomes a structured span with arguments, run, thread and resource IDs, nested under the agent run, and can be exported to OpenTelemetry and other backends. Approvals and declines are not recorded as their own events, and exports are best-effort.

- **default configuration** (default; raw 0.07, cap G1 → 0.07)
  - **S L0:** Default records tool calls only at debug level. — [packages/core/src/tools/tool-builder/builder.ts:939](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/tools/tool-builder/builder.ts#L939); [packages/core/src/mastra/index.ts:1716-1719](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/mastra/index.ts#L1716-L1719) (verified)
    - *To reach the next level:* No structured record of tool calls by default.
  - **C L1:** The debug log covers the createTool path. — [packages/core/src/tools/tool-builder/builder.ts:939](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/tools/tool-builder/builder.ts#L939) (verified)
    - *To reach the next level:* Approvals and sub-agent activity are not recorded.
  - **D L0:** Observability is a no-op unless configured. — [packages/core/src/mastra/index.ts:1716-1719](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/mastra/index.ts#L1716-L1719) (verified)
    - *To reach the next level:* No record on by default.
  - **B L0:** Without observability nothing durable is written. — [packages/core/src/mastra/index.ts:1719](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/mastra/index.ts#L1719) (verified)
    - *To reach the next level:* Records are not flushed per action.
- **opt-in @mastra/observability tracing** (alt; raw 0.35, cap G1 → 0.35) ← counted
  - **S L2:** Structured TOOL_CALL / MCP_TOOL_CALL spans with run, thread and resource IDs. — [packages/core/src/tools/tool-builder/builder.ts:881](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/tools/tool-builder/builder.ts#L881); [packages/core/src/tools/tool-builder/builder.ts:930-932](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/tools/tool-builder/builder.ts#L930-L932) (verified)
    - *To reach the next level:* No approver attribution in the record.
  - **C L2:** All tools, including MCP tools, are traced; approvals and denials are not. — searched `rg -n -e 'SpanType\.TOOL_APPROVAL|approvedBy|approver'` in `packages/core/src/observability packages/core/src/loop/workflows/agentic-execution/tool-call-step.ts` → 0 hits (approvals are not traced as events) (verified)
    - *To reach the next level:* Approvals and denials are not recorded.
  - **D L0:** Must be configured explicitly. — [packages/core/src/mastra/index.ts:1716-1719](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/mastra/index.ts#L1716-L1719) (verified)
    - *To reach the next level:* Not on by default.
  - **B L1:** Spans are exported asynchronously by exporters; best-effort. — [packages/core/src/mastra/index.ts:1716-1719](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/mastra/index.ts#L1716-L1719) (inferred)
    - *To reach the next level:* Records are not durable per action.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C10 Limits & kill switch — 0.28 (high)

Every agent run stops after 5 model steps unless the developer sets maxSteps or a stop condition. There is no default wall-clock or token/cost limit: run and step timeouts exist but are opt-in, and the local sandbox applies a command timeout only when one is passed (its documented 30-second default is not implemented in the process manager). When delegating, the model can raise a sub-agent's step budget if that sub-agent has no configured default, and each sub-agent starts its own budget. Abort signals are passed to tools and background shell processes can be killed by process group.

- **default configuration** (default; raw 0.25 → 0.25)
  - **S L1:** Iteration cap only by default. — [packages/core/src/llm/model/model.loop.ts:168](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/llm/model/model.loop.ts#L168); [packages/core/src/loop/timeout.ts:62](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/loop/timeout.ts#L62) (verified)
    - *To reach the next level:* No default wall-clock or token/cost limit.
  - **C L1:** The step cap applies to the top-level loop; sandbox commands have no timeout unless one is passed. — [packages/core/src/workspace/sandbox/local-process-manager.ts:52](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-process-manager.ts#L52) (verified)
    - *To reach the next level:* No default tool timeouts; sub-agents start fresh budgets.
  - **D L1:** The model can request a larger maxSteps for a sub-agent; it is capped only if the sub-agent has its own default. — [packages/core/src/agent/agent.ts:320-325](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/agent/agent.ts#L320-L325); [packages/core/src/agent/agent.ts:5590](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/agent/agent.ts#L5590) (verified)
    - *To reach the next level:* The model can raise its sub-agents' limits.
  - **B L1:** No wall-clock ceiling: a single shell or tool call can run indefinitely, and background processes can outlive the run. — [packages/core/src/workspace/sandbox/local-process-manager.ts:52](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-process-manager.ts#L52) (verified)
    - *To reach the next level:* No tight per-run time or cost ceilings.
- **opt-in modelSettings.timeout (stepMs/totalMs)** (alt; raw 0.28, cap G1 → 0.28) ← counted
  - **S L2:** Run and step wall-clock timeouts abort via AbortController when configured. — [packages/core/src/loop/timeout.ts:62](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/loop/timeout.ts#L62) (verified)
    - *To reach the next level:* No token/cost cap or side-effect rate limits.
  - **C L1:** Applies to the model loop; tool processes are governed separately. — [packages/core/src/loop/timeout.ts:62](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/loop/timeout.ts#L62) (verified)
    - *To reach the next level:* Sub-agents and spawned processes do not share the budget.
  - **D L0:** Opt-in. — [packages/core/src/loop/timeout.ts:62](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/loop/timeout.ts#L62) (verified)
    - *To reach the next level:* Not on by default.
  - **B L1:** Background processes can outlive a stop. — [packages/core/src/workspace/sandbox/local-process-manager.ts:52](https://github.com/mastra-ai/mastra/blob/20c9a8cb991473e8644598ab8b1e5ee61cc6865b/packages/core/src/workspace/sandbox/local-process-manager.ts#L52) (verified)
    - *To reach the next level:* Stopping does not guarantee in-flight work ends.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

## Rule-of-Two check
[A] untrusted input: Tool and MCP results enter context with no taint tracking (packages/core/src/processors/processors/prompt-injection-detector.ts:180 scans user input only) · [B] sensitive data/systems: Developer credentials in the process environment; per-user memory (packages/core/src/memory/memory.ts:90) · [C] state change / egress: Any registered tool without approval (packages/core/src/tools/tool.ts:384); workspace write/shell tools (packages/core/src/workspace/tools/tools.ts:152-153) · Same default session? Yes

## Highest-impact improvements
1. Default requireApproval to true for workspace write, delete and execute_command tools, and expose a one-line agent-wide requireToolApproval default. — C2 D L0→L3, +0.150 before caps (Playbook 5)
2. Make LocalSandbox auto-select bwrap/Seatbelt when available and refuse host execution without an explicit isolation: 'none' opt-out. — C4 D L0→L3, +0.150 before caps (Playbook 3 step 1)
3. Ship observability with a local storage exporter on by default so every tool call and approval is recorded. — C9 D L0→L2, +0.100 before caps
4. Implement the documented 30s LocalSandbox command timeout and add a default run wall-clock limit. — C10 S L1→L2, +0.075 before caps
5. Harden memory namespace isolation. — C6 D L2→L3, +0.050 before caps

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the framework (@mastra/core, @mastra/mcp, @mastra/memory, server adapters, observability) at default constructor arguments; mastracode, the agent-controller harness (with its yolo mode), deployers, and the many workspace/integration packages were only sampled.
- Remote sandbox providers under workspaces/ (E2B, Daytona, Modal, etc.) were not reviewed in depth; the C4 alternative scores the core bubblewrap backend.
- The optional editor's stored MCP client routes are an opt-in path and were not scored as default.
- No reviewer-steering text was found in AGENTS.md, CLAUDE.md or README.md.
