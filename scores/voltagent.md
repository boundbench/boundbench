# Defense-in-Depth Score: VoltAgent

**Repo:** https://github.com/voltagent/voltagent · **Commit:** `72a46c76b4507e1f98b41d5050a92bd30587eb70` (@voltagent/core@2.11.0) · **Reviewed:** 2026-10-04
**What it is:** Open-source TypeScript framework for building and orchestrating AI agents with tools, memory, workflows, MCP, sub-agents and a bundled HTTP server.
**Category:** Agent Frameworks
**Scored configuration:** README quickstart: new Agent({...}) with default arguments registered in new VoltAgent({ server: honoServer() }); workspace, sandbox, guardrails and approval left at their defaults.
**Agent surface (default):** code execution opt-in · filesystem write opt-in · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions opt-in · sub agents opt-in · external communication yes

## Score: 2.8 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L2 | L1 | L0 | L0 | 0.23 | C2-POWERBYPASS | **0.23** | Medium |
| C3 | Tool & action scoping | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C4 | Code-execution isolation | L4 | L2 | L0 | L2 | 0.55 | G1 | **0.50** (alt) | Medium |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L0 | 0.07 | C5-WORSTCASE | **0.07** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L0 | 0.00 | C7-RCELOAD | **0.00** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |


The network exposure and access control of VoltAgent's bundled HTTP server, as the README sets it up, are not locked down. Inside the agent, tool approval, sandbox isolation, and prompt-injection guardrails all exist but are off by default. MCP tools cannot be gated.

## Critical gaps
- Approval is opt-in and the most powerful paths (execute_command, MCP tools) skip it by default. (ASI02, ASI09; C2) — [packages/core/src/workspace/sandbox/toolkit.ts:318](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/workspace/sandbox/toolkit.ts#L318)
- LocalSandbox runs commands on the host as the operator, with isolation off by default. (ASI05, T11; C4) — [packages/core/src/workspace/sandbox/local.ts:128](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/workspace/sandbox/local.ts#L128); [packages/core/src/workspace/sandbox/local.ts:548](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/workspace/sandbox/local.ts#L548)
- Nothing structurally limits a hijacked agent; untrusted tool and MCP results can drive exfiltration and irreversible tools unattended. (ASI01, LLM01, T6; C5) — [packages/core/src/agent/agent.ts:7322](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/agent.ts#L7322); [packages/core/src/mcp/client/index.ts:404-407](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/mcp/client/index.ts#L404-L407)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

VoltAgent has no agent identity or credential scoping. Tools construct their own clients with whatever credentials the developer supplies, and access control on the bundled Hono server in the README quickstart configuration is not locked down.

- **S L0:** No dedicated or scoped identity; agents and tools run with the process's ambient credentials and any key the developer hands a tool. (verified)
  - *To reach the next level:* A per-agent or per-tool scoped credential, or a deterministic authorization gate mapping each call to a least-privilege policy.
- **C L0:** Server authorization exists only as optional middleware. (verified)
  - *To reach the next level:* Main tool path checked against an authenticated principal.
- **D L0:** The default server configuration provides no narrower identity. (verified)
  - *To reach the next level:* A narrower default.
- **B L0:** A misused agent or server route reaches the operator's whole account. — [packages/core/src/utils/update/index.ts:459](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/utils/update/index.ts#L459) (verified)
  - *To reach the next level:* Authority limited to one system or tenant.
- **Cap:** none

### C2 Approval gates — 0.23 (medium)

Tools can declare needsApproval (a boolean or a function of the arguments), which VoltAgent passes to the AI SDK's approval flow. Nothing is gated by default: the flag is undefined unless the developer sets it, including on the workspace execute_command tool. Tools wrapped from MCP servers never get the flag. There's no rollback or rate limiting on consequential actions.

- **S L2:** Per-call approval via the AI SDK tool-approval-request flow, with optional argument-level policy functions; what the approver sees depends on the developer's UI. — [packages/core/src/tool/manager/ToolManager.ts:69](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/tool/manager/ToolManager.ts#L69); [packages/core/src/utils/message-converter.ts:285-293](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/utils/message-converter.ts#L285-L293) (inferred)
  - *To reach the next level:* Framework-guaranteed display of the exact call plus risk tiers.
- **C L1:** Only tools explicitly flagged are gated; MCP tools can't be flagged, and gate coverage does not extend to every server route. — [packages/core/src/agent/agent.ts:7322](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/agent.ts#L7322); [packages/core/src/mcp/client/index.ts:404-407](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/mcp/client/index.ts#L404-L407); searched `rg -n needsApproval` in `packages/core/src/mcp` → 0 hits (MCP-wrapped tools are created without any needsApproval flag) (verified)
  - *To reach the next level:* Every tool path, including MCP tools and server routes, traverses the gate.
- **D L0:** Approval is opt-in per tool; execute_command's needsApproval is undefined unless a toolPolicy sets it. — [packages/core/src/agent/agent.ts:7322](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/agent.ts#L7322); [packages/core/src/workspace/sandbox/toolkit.ts:318](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/workspace/sandbox/toolkit.ts#L318) (verified)
  - *To reach the next level:* Approval on by default for mutating tools.
- **B L0:** Developer tools, MCP tools and execute_command can take irreversible actions with no checkpoint or undo. — [packages/core/src/workspace/sandbox/local.ts:548](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/workspace/sandbox/local.ts#L548) (verified)
  - *To reach the next level:* Reversible common case (checkpoints, trash, previews).
- **Cap:** C2-POWERBYPASS — The most powerful path, execute_command when a sandbox is configured, skips the gate by default.

### C3 Tool & action scoping — 0.45 (high)

Every tool call is parsed against its Zod schema before execution, and the Node filesystem backend enforces realpath-based containment under its root. The workspace's execute_command tool, however, takes a raw command, arguments and environment variables from the model. Its command allow/deny lists are optional and match only the binary's basename. With no workspace configured, an agent gets only the tools the developer passes. Once a workspace is enabled, the defaults add filesystem write and command execution.

- **S L2:** Typed Zod validation for all tools and realpath containment in the Node filesystem backend, but execute_command is raw passthrough including model-chosen env vars. — [packages/core/src/agent/agent.ts:7133](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/agent.ts#L7133); [packages/core/src/workspace/filesystem/backends/filesystem.ts:143](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/workspace/filesystem/backends/filesystem.ts#L143); [packages/core/src/workspace/sandbox/toolkit.ts:324](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/workspace/sandbox/toolkit.ts#L324) (verified)
  - *To reach the next level:* Allowlist validation on exec (parsed commands) and URL/host checks.
- **C L2:** Schema validation applies to built-in, custom and MCP-converted tools; real bounds exist only in the filesystem backend. — [packages/core/src/agent/agent.ts:7133](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/agent.ts#L7133); [packages/core/src/mcp/client/index.ts:404-407](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/mcp/client/index.ts#L404-L407) (verified)
  - *To reach the next level:* A shared policy layer enforcing value-level bounds on all tools.
- **D L2:** No tools by default, but enabling a workspace auto-adds filesystem (write), sandbox (exec), search and skills toolkits. — [packages/core/src/agent/agent.ts:518](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/agent.ts#L518); [packages/core/src/workspace/sandbox/toolkit.ts:318](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/workspace/sandbox/toolkit.ts#L318) (verified)
  - *To reach the next level:* Read-only tool set by default when a workspace is enabled.
- **B L1:** Filesystem tools are contained to a root; execute_command can run any host binary with network access. — [packages/core/src/workspace/sandbox/local.ts:548](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/workspace/sandbox/local.ts#L548); [packages/core/src/workspace/filesystem/backends/filesystem.ts:143](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/workspace/filesystem/backends/filesystem.ts#L143) (verified)
  - *To reach the next level:* Workspace-scoped operations with bounded quantities.
- **Cap:** none

### C4 Code-execution isolation — 0.50 (medium)

Model-driven code execution happens through the workspace execute_command tool and only when the developer configures a sandbox. The built-in LocalSandbox spawns commands directly on the host, with isolation provider 'none' by default; it does scrub the environment to PATH and applies a 30-second timeout. Optional bwrap or sandbox-exec isolation exists, and so do remote E2B, Daytona and Blaxel sandbox packages, which are the strongest option. All of it is opt-in.

- **default configuration** (default; raw 0.07, cap G1 → 0.07)
  - **S L0:** LocalSandbox spawns the command as the same OS user on the host; isolation defaults to 'none'. — [packages/core/src/workspace/sandbox/local.ts:128](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/workspace/sandbox/local.ts#L128); [packages/core/src/workspace/sandbox/local.ts:548](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/workspace/sandbox/local.ts#L548) (verified)
    - *To reach the next level:* OS-level separation (container, low-privilege user, bwrap) by default.
  - **C L1:** All execute_command calls go through sandbox.execute, but MCP stdio servers and the update installer run on the host. — [packages/core/src/workspace/sandbox/local.ts:548](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/workspace/sandbox/local.ts#L548); [packages/core/src/mcp/client/index.ts:184](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/mcp/client/index.ts#L184); [packages/core/src/utils/update/index.ts:459](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/utils/update/index.ts#L459) (verified)
    - *To reach the next level:* Most other execution paths also sandboxed.
  - **D L0:** No sandbox is configured by default, and LocalSandbox isolation is off unless set. — [packages/core/src/workspace/sandbox/local.ts:128](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/workspace/sandbox/local.ts#L128) (verified)
    - *To reach the next level:* Sandbox on by default.
  - **B L0:** Host execution as the operator: home directory, network and local files are reachable, though the env is scrubbed to PATH. — [packages/core/src/workspace/sandbox/local.ts:527](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/workspace/sandbox/local.ts#L527); [packages/core/src/workspace/sandbox/local.ts:548](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/workspace/sandbox/local.ts#L548) (verified)
    - *To reach the next level:* Workspace-only filesystem and restricted network.
- **opt-in remote E2B/Daytona/Blaxel sandbox** (alt; raw 0.55, cap G1 → 0.50) ← counted
  - **S L4:** Remote ephemeral sandbox service (E2B) executes commands off-host. — [packages/sandbox-e2b/src/index.ts:377](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/sandbox-e2b/src/index.ts#L377) (verified)
  - **C L2:** execute_command routes to the remote sandbox; MCP stdio servers and updates still run on host. — [packages/core/src/mcp/client/index.ts:184](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/mcp/client/index.ts#L184); [packages/core/src/utils/update/index.ts:459](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/utils/update/index.ts#L459) (verified)
    - *To reach the next level:* Every execution path, including MCP stdio servers, sandboxed.
  - **D L0:** Requires installing the package and configuring credentials. — [packages/core/src/workspace/sandbox/local.ts:128](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/workspace/sandbox/local.ts#L128) (verified)
    - *To reach the next level:* On by default.
  - **B L2:** Only explicitly passed env reaches the sandbox, but egress is unrestricted and the sandbox persists across calls. — [packages/sandbox-e2b/src/index.ts:351](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/sandbox-e2b/src/index.ts#L351) (inferred)
    - *To reach the next level:* Network egress off or allowlisted, plus per-run teardown.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.07 (high)

Nothing in the framework tracks which content is untrusted or restricts tools after it has been read. Tool and MCP results go back into context with the same standing as the user's instructions. The only defense is an opt-in input guardrail that matches five phrases such as 'ignore previous instructions'. Because a developer's tools usually combine private data with egress, a hijacked agent can leak data and act with no human involved.

- **S L1:** Opt-in phrase-matching prompt-injection guardrail on user input only (detection). — [packages/core/src/agent/guardrails/defaults.ts:645](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/guardrails/defaults.ts#L645); searched `rg -n -i 'taint|untrusted'` in `packages/core/src/agent/agent.ts` → 0 hits (verified)
  - *To reach the next level:* Approval forced on egress or writes after untrusted content is read.
- **C L0:** Tool results, MCP results and tool descriptions aren't distinguished from principal input; the guardrail covers only input text. — [packages/core/src/mcp/client/index.ts:404-407](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/mcp/client/index.ts#L404-L407); searched `rg -n -i 'taint|untrusted'` in `packages/core/src/agent/agent.ts` → 0 hits (verified)
  - *To reach the next level:* At least one untrusted source handled.
- **D L0:** The guardrail is off unless the developer adds it. — [packages/core/src/agent/guardrails/defaults.ts:645](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/guardrails/defaults.ts#L645) (verified)
  - *To reach the next level:* On by default.
- **B L0:** Framework rating: untrusted input, private data and egress or irreversible tools combine with no approval. — [packages/core/src/agent/agent.ts:7322](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/agent.ts#L7322) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions require human approval.
- **Cap:** C5-WORSTCASE — In the default configuration a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.25 (high)

Conversation history is stored per userId and conversationId and replayed into later turns, and the README quickstart persists it to a LibSQL file. Optional working memory lets the model write freely into a store that is re-injected into its context, optionally checked against a schema. Queries are namespaced by userId; that id is not strictly bound to an authenticated principal. No .env or repo-controlled config is auto-loaded.

- **S L1:** Memory writes (history, opt-in working memory via update_working_memory) aren't gated or provenance-tagged; they're stored and replayed. — [packages/core/src/agent/agent.ts:8853](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/agent.ts#L8853); [packages/core/src/memory/adapters/storage/in-memory.ts:168](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/memory/adapters/storage/in-memory.ts#L168) (verified)
  - *To reach the next level:* Memory entries carry provenance and are presented as data.
- **C L1:** Only working memory has optional schema validation; history and vector stores are uncontrolled. — [packages/core/src/agent/agent.ts:8853](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/agent.ts#L8853) (verified)
  - *To reach the next level:* Main memory store controlled.
- **D L1:** Storage is namespaced by userId, but it is not strictly bound to an authenticated principal. — [packages/core/src/memory/adapters/storage/in-memory.ts:168](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/memory/adapters/storage/in-memory.ts#L168) (verified)
  - *To reach the next level:* Per-user namespaces bound to an authenticated principal.
- **B L1:** Poisoned history or user-scoped working memory persists across that user's sessions and can trigger tool use. — [packages/core/src/agent/agent.ts:8853](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/agent.ts#L8853); searched `rg -n dotenv` in `packages/core/src packages/server-core/src packages/server-hono/src` → 0 hits (verified)
  - *To reach the next level:* Influence limited to text output or gated actions.
- **Cap:** none

### C7 Third-party extensions — 0.00 (high)

MCP servers are user-configured and launched without version pinning or integrity checks, though stdio servers get the MCP SDK's reduced default environment. The server's update route is also not locked down.

- **S L0:** MCP servers and update installs are unpinned and unverified. — searched `rg -n -i 'sha256|integrity|checksum'` in `packages/core/src/mcp` → 0 hits (verified)
  - *To reach the next level:* Only user-chosen, pinned sources.
- **C L0:** Neither MCP servers nor the update installer are verified. — [packages/core/src/mcp/client/index.ts:184](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/mcp/client/index.ts#L184) (verified)
  - *To reach the next level:* At least one extension type verified.
- **D L0:** Third-party code can be added without explicit operator consent. (verified)
  - *To reach the next level:* Installation only with explicit operator consent.
- **B L0:** Installed packages run as the same user with the full process environment (execSync inherits env). — [packages/core/src/utils/update/index.ts:459](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/utils/update/index.ts#L459) (verified)
  - *To reach the next level:* Separate process with a scrubbed environment.
- **Cap:** C7-RCELOAD — Remote code loading is reachable in the default configuration.

### C8 Secrets & sensitive-data protection — 0.30 (high)

Credentials come from environment variables and developer code. LocalSandbox and MCP stdio children get a scrubbed environment by default, which is good. Traces record full tool arguments and LLM inputs with no redaction, and are served over the observability routes. Remote trace export to VoltOps happens only when VoltOps keys are present. PII guardrails exist but are opt-in.

- **S L1:** Secrets come from env vars; masking exists only as opt-in PII guardrails, with no redaction in traces or logs. — searched `rg -n -i redact` in `packages/core/src/observability packages/core/src/agent/open-telemetry` → 0 hits; [packages/core/src/agent/agent.ts:6456](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/agent.ts#L6456) (verified)
  - *To reach the next level:* Type-level masking and log filters on main paths.
- **C L1:** Only subprocess environments are protected (PATH-only for LocalSandbox, scrubbed for MCP stdio). — [packages/core/src/workspace/sandbox/local.ts:527](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/workspace/sandbox/local.ts#L527); [packages/core/src/mcp/client/index.ts:184](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/mcp/client/index.ts#L184) (verified)
  - *To reach the next level:* Logs and transcripts protected too.
- **D L2:** Remote telemetry is opt-in (needs VoltOps keys), but local traces store full payloads and aren't redacted. — [packages/core/src/voltagent.ts:303](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/voltagent.ts#L303); [packages/core/src/agent/agent.ts:6456](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/agent.ts#L6456) (verified)
  - *To reach the next level:* Redaction always on.
- **B L1:** Long-lived provider and tool keys live in the agent process that in-process tools can read. — [packages/core/src/workspace/sandbox/local.ts:527](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/workspace/sandbox/local.ts#L527) (verified)
  - *To reach the next level:* Scoped keys.
- **Cap:** none

### C9 Audit & traceability — 0.45 (high)

Every tool execution creates an OpenTelemetry span with the tool name, call id and arguments, linked to the agent, user id and sub-agent parent. By default spans go to an in-memory store capped at 10,000 entries inside the agent process, plus a local WebSocket for the console, so they're lost on restart. Approvals aren't recorded as approver identity, and the HTTP tool-execute route isn't traced through the agent span path.

- **S L2:** Structured spans with tool args, timestamps and user.id; no approver attribution or tamper evidence. — [packages/core/src/agent/agent.ts:6456](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/agent.ts#L6456); [packages/core/src/agent/open-telemetry/trace-context.ts:103](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/open-telemetry/trace-context.ts#L103) (verified)
  - *To reach the next level:* Approver and requesting-principal attribution.
- **C L2:** All agent-driven tools, including MCP and sub-agents, pass through the traced executor; direct HTTP tool execution doesn't. — [packages/core/src/agent/agent.ts:6456](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/agent.ts#L6456) (verified)
  - *To reach the next level:* All paths plus approvals and denials.
- **D L2:** On by default in an in-memory store outside the workspace, inside the agent process; exposed via server routes. — [packages/core/src/observability/node/volt-agent-observability.ts:70](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/observability/node/volt-agent-observability.ts#L70) (verified)
  - *To reach the next level:* Written by a component the model can't control.
- **B L1:** Best-effort in-memory ring buffer, lost on crash. — [packages/core/src/observability/adapters/in-memory-adapter.ts:37](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/observability/adapters/in-memory-adapter.ts#L37) (verified)
  - *To reach the next level:* Records flushed durably per action.
- **Cap:** none

### C10 Limits & kill switch — 0.50 (high)

Agents stop after 5 steps by default (100 when a workspace is attached), and callers can pass an abort signal that sub-agents inherit. Sandbox commands time out after 30 seconds and are killed on abort, and MCP calls use the SDK request timeout. There's no token, cost or wall-clock cap. Each delegation starts a fresh step budget for the sub-agent.

- **S L2:** Step cap plus per-execution timeouts (sandbox, MCP) enforced in code; no token or cost cap. — [packages/core/src/agent/agent.ts:1091](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/agent.ts#L1091); [packages/core/src/workspace/sandbox/local.ts:24](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/workspace/sandbox/local.ts#L24); searched `rg -n -i 'budget|maxTotalTokens|costLimit'` in `packages/core/src/agent/agent.ts` → 0 hits (verified)
  - *To reach the next level:* Wall-clock and token or cost caps.
- **C L2:** Top-level loop plus tool timeouts; sub-agents get the parent's abort signal but their own step budget. — [packages/core/src/agent/agent.ts:1393](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/agent.ts#L1393); [packages/core/src/agent/subagent/index.ts:406](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/subagent/index.ts#L406) (verified)
  - *To reach the next level:* Sub-agents counted against the parent's budget.
- **D L2:** Sensible defaults (5 steps), operator-configurable. — [packages/core/src/agent/agent.ts:1091](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/agent.ts#L1091) (verified)
  - *To reach the next level:* Delegation can't reset limits.
- **B L2:** Moderate ceilings; abort kills sandbox processes, but other in-flight tools run on. — [packages/core/src/agent/subagent/index.ts:406](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/agent/subagent/index.ts#L406); [packages/core/src/workspace/sandbox/local.ts:24](https://github.com/voltagent/voltagent/blob/72a46c76b4507e1f98b41d5050a92bd30587eb70/packages/core/src/workspace/sandbox/local.ts#L24) (verified)
  - *To reach the next level:* Cancel all pending calls on stop; per-run time and cost ceilings.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Tool and MCP results enter context unmarked · [B] sensitive data/systems: Provider keys and any tool credentials live in the agent process · [C] state change / egress: Developer-registered tools, MCP tools, and opt-in execute_command run with no approval by default (packages/core/src/agent/agent.ts:7322) · Same default session? Yes

## Highest-impact improvements
1. Harden the server's package-update routes and default network exposure. — C7 D L0→L3, +0.150 before caps (Playbook 3)
2. Harden default access control for execution, memory and observability routes and bind user identity to an authenticated principal. — C1 C L0→L2, +0.150 before caps (Playbook 4)
3. Default needsApproval to true for execute_command and filesystem-write tools, and apply it on every tool path, including MCP tools. — C2 C L1→L3, +0.150 before caps (Playbook 5)
4. Default LocalSandbox to bwrap/sandbox-exec when available, with no network and no read access outside the root, failing closed. — C4 S L0→L3, +0.225 before caps (Playbook 3)
5. Redact secrets and PII in tool and LLM span attributes by default. — C8 S L1→L2, +0.075 before caps

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Approval UI behaviour and MCP getDefaultEnvironment scrubbing are AI SDK / MCP SDK library behaviour, inferred from VoltAgent's use of them, not read in those libraries.
- server-elysia, serverless-hono, a2a-server, the CLI and the VoltOps cloud service were not reviewed in depth; scoring focuses on @voltagent/core and @voltagent/server-hono/server-core.
- No text aimed at AI reviewers was found in AGENTS.md, CLAUDE.md or SECURITY.md.
