# Defense-in-depth score: mcp-use

**Repo:** https://github.com/mcp-use/mcp-use · **Commit:** `90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8` · **Reviewed:** 2026-10-05
**What it is:** TypeScript and Python framework for building MCP servers, MCP clients and model-driven agents that call MCP tools.
**Category:** Agent Frameworks
**Scored configuration:** TypeScript library defaults: MCPAgent from @mcp-use/agent with default constructor arguments, driving an MCPClient built from a developer-supplied mcpServers map (stdio and HTTP servers); code mode, Langfuse observability and the remote agent not enabled.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions yes · sub agents no · external communication yes

## Score: 2.5 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L1 | L1 | 0.25 | none | **0.25** | Medium |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | none | **0.00** | High |
| C3 | Tool & action scoping | L1 | L0 | L1 | L1 | 0.17 | none | **0.17** | High |
| C4 | Code-execution isolation | L4 | L1 | L0 | L2 | 0.47 | G1 | **0.47** (alt) | Medium |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L1 | L3 | 0.35 | none | **0.35** | High |
| C7 | Third-party extensions | L1 | L0 | L1 | L2 | 0.23 | none | **0.23** | Medium |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L0 | L1 | 0.20 | none | **0.20** | Medium |
| C9 | Audit & traceability | L2 | L1 | L0 | L1 | 0.28 | G1 | **0.28** (alt) | Medium |
| C10 | Limits & kill switch | L2 | L2 | L3 | L2 | 0.55 | none | **0.55** | Medium |


mcp-use makes it easy to connect a model to MCP servers, but its agent adds almost no safeguards of its own. Out of the box no tool call needs approval, locally launched servers run unsandboxed as the user, and content read through one server can steer calls to every other, so a prompt injection can leak data and take actions with no human involved. Usage telemetry is on by default. The protections that do exist are a default 10-step cap, reduced environments for launched servers, and an opt-in E2B cloud sandbox for model-written code.

## Critical gaps
- A prompt injection in content read through any configured server can use every other tool to leak data and take irreversible actions with no approval step. (ASI01, T6, LLM01; C5). Evidence: [libraries/typescript/packages/agent/src/llm/toolLoop.ts:176-189](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/llm/toolLoop.ts#L176-L189); [libraries/typescript/packages/agent/src/llm/toolLoop.ts:195-202](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/llm/toolLoop.ts#L195-L202)
- Locally launched MCP servers run directly on the host as the current user, with no sandbox in the default configuration. (ASI05, T11, LLM05; C4). Evidence: [libraries/typescript/packages/client/src/transport/stdio.ts:109-116](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/transport/stdio.ts#L109-L116)

## Criterion details

### C1 Identity & least privilege: 0.25 (medium confidence)

The agent acts with whatever credentials the developer puts in each server's configuration (bearer tokens, headers, or an OAuth login) plus the operator's own account for locally launched servers. Each server only receives its own credentials, and locally launched servers start with a reduced environment rather than every variable of the parent process, which is a real narrowing. There is no authorization layer that checks individual tool calls against a policy, no split between read and write credentials, and no short-lived or per-task tokens, so a hijacked agent holds every configured server's full authority.

- **S L1:** Credentials are per server (token, headers or OAuth provider in the server config), but they are long-lived, shared by read and write tools, and stdio servers run with the OS user's authority. Evidence: [libraries/typescript/packages/client/src/core/config.ts:193-201](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/core/config.ts#L193-L201); [libraries/typescript/packages/agent/src/agents/mcp_agent.ts:243-247](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/agents/mcp_agent.ts#L243-L247) (verified)
  - *To reach the next level:* No role-scoped identity or least-privilege policy applied by the framework; read and write share one credential per server.
- **C L1:** Every stdio server gets only its configured env layered over the MCP SDK's small default environment (inferred from the SDK's documented behaviour, corroborated by the code comment), but no authorization check sits on any tool path. Evidence: [libraries/typescript/packages/client/src/transport/stdio.ts:112-114](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/transport/stdio.ts#L112-L114); [libraries/typescript/packages/agent/src/adapters/native_adapter.ts:74-84](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/adapters/native_adapter.ts#L74-L84) (inferred)
  - *To reach the next level:* No authorization layer in code that every tool call passes through.
- **D L1:** The default grants whatever the developer configured for each server; nothing in the framework narrows or time-bounds it. Evidence: [libraries/typescript/packages/client/src/core/config.ts:124-132](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/core/config.ts#L124-L132) (verified)
  - *To reach the next level:* No read-only or minimal default; widening is just adding credentials to the server map.
- **B L1:** A hijacked agent can use every configured server's credentials in one session, and stdio servers act as the OS user (the documented examples include a filesystem server and a browser server). Evidence: [libraries/typescript/packages/agent/examples/integrations/browser_use.ts:17-19](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/examples/integrations/browser_use.ts#L17-L19); [libraries/typescript/packages/agent/src/agents/mcp_agent.ts:264-275](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/agents/mcp_agent.ts#L264-L275) (verified)
  - *To reach the next level:* Authority spans several systems with write access; nothing scopes it to one project or to read-only.
- **Cap:** none

### C2 Approval gates: 0.00 (high confidence)

There is no human approval step anywhere in the agent. Every tool call the model requests is sent straight to the MCP server, whether it reads data, writes files, drives a browser or calls a write API. The tool risk hints that MCP servers publish (read-only, destructive) are not consulted. A developer who wants a confirmation step has to write it around the agent themselves.

- **S L0:** The tool loop dispatches each model-requested call immediately with no approval hook. Evidence: [libraries/typescript/packages/agent/src/llm/toolLoop.ts:176-189](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/llm/toolLoop.ts#L176-L189); searched `rg -n -S -e 'approv|confirm|destructiveHint|readOnlyHint'` in `libraries/typescript/packages/agent/src` → 0 hits (no approval, confirmation or risk-hint handling in the agent package) (verified)
  - *To reach the next level:* No per-call approval of any kind.
- **C L0:** Every tool, resource and prompt exposed to the model executes without a gate. Evidence: [libraries/typescript/packages/agent/src/adapters/native_adapter.ts:74-89](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/adapters/native_adapter.ts#L74-L89) (verified)
  - *To reach the next level:* The most powerful tools of any configured server are ungated.
- **D L0:** No approval exists to be on by default. Evidence: [libraries/typescript/packages/agent/src/agents/agent_options.ts:50-87](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/agents/agent_options.ts#L50-L87) (verified)
  - *To reach the next level:* Approval would have to be built by the developer.
- **B L0:** Calls reach whatever configured servers can do, including file writes and browser actions in the documented examples, with no checkpoint or undo. Evidence: [libraries/typescript/packages/agent/examples/integrations/browser_use.ts:17-19](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/examples/integrations/browser_use.ts#L17-L19) (verified)
  - *To reach the next level:* No reversibility, previews or rate limits on consequential calls.
- **Cap:** none

### C3 Tool & action scoping: 0.17 (high confidence)

The agent passes the model's tool arguments to each MCP server exactly as generated; it does not check paths, URLs or quantities itself and leaves all validation to the server. Its only scoping control is a list of tool names to hide (disallowedTools), which is empty by default, and calls to names it never exposed are rejected. By default every tool of every configured server is exposed, and resources and prompts are also turned into callable tools. Servers built with the mcp-use server framework do validate inputs against their typed schemas, but that protects those servers, not the agent's path to third-party ones.

- **S L1:** The agent's only primitive is a tool-name denylist; arguments pass through unvalidated. Evidence: [libraries/typescript/packages/agent/src/agents/mcp_agent.ts:147-151](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/agents/mcp_agent.ts#L147-L151); [libraries/typescript/packages/agent/src/adapters/native_adapter.ts:76-84](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/adapters/native_adapter.ts#L76-L84) (verified)
  - *To reach the next level:* No typed argument validation, path containment or URL allowlist on the agent's tool path.
- **C L0:** No tool on the agent path has its arguments checked by the framework. Evidence: searched `rg -n -S -e 'validate|safeParse|allowlist|realpath'` in `libraries/typescript/packages/agent/src/llm/toolLoop.ts libraries/typescript/packages/agent/src/adapters/native_adapter.ts` → 0 hits (dispatch path from model output to connector has no validation step) (verified)
  - *To reach the next level:* Validation would need to apply to at least some tool paths.
- **D L1:** All tools of every configured server are on by default, plus resources and prompts as tools; individual tools can be hidden by name. Evidence: [libraries/typescript/packages/agent/src/agents/mcp_agent.ts:147-150](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/agents/mcp_agent.ts#L147-L150) (verified)
  - *To reach the next level:* No tool groups or read-only default set.
- **B L1:** A misused tool reaches whatever its server reaches; the framework adds no workspace or quantity bounds. Evidence: [libraries/typescript/packages/agent/examples/integrations/browser_use.ts:17-19](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/examples/integrations/browser_use.ts#L17-L19) (verified)
  - *To reach the next level:* No framework-level scoping to a project or bounded quantities.
- **Cap:** none

### C4 Code-execution isolation: 0.47 (medium confidence)

In the default setup the only code the framework runs is the local MCP servers it launches, and those start directly on the host as the same user, with access to the user's files and network. They get a reduced environment rather than the parent's full one, but there is no container or OS sandbox. An optional code mode lets the model write JavaScript that calls tools; it runs by default in Node's built-in vm module, which is not a security boundary, or, when the developer chooses it, in a remote E2B cloud sandbox. That E2B option is a strong boundary for model-written code, but it is off by default, the sandbox can still call every configured tool back on the host, and it does not cover the locally launched servers.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** stdio MCP servers are spawned on the host as the current user; the default command is npx. Evidence: [libraries/typescript/packages/client/src/transport/stdio.ts:63-66](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/transport/stdio.ts#L63-L66); [libraries/typescript/packages/client/src/transport/stdio.ts:109-116](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/transport/stdio.ts#L109-L116) (verified)
    - *To reach the next level:* No isolation primitive for launched servers.
  - **C L0:** No execution path is sandboxed in the default configuration. Evidence: searched `rg -n -S -e 'sandbox|docker|seccomp|landlock|bwrap'` in `libraries/typescript/packages/client/src/transport` → 0 hits (transport layer that launches servers has no sandbox code) (verified)
    - *To reach the next level:* Not even the main execution path is sandboxed.
  - **D L0:** Host execution is the only mode for launched servers; code mode is off and its default executor is node:vm. Evidence: [libraries/typescript/packages/client/src/core/node.ts:268](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/core/node.ts#L268); [libraries/typescript/packages/client/src/core/node.ts:351-358](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/core/node.ts#L351-L358) (verified)
    - *To reach the next level:* Nothing to turn on for stdio servers; sandboxed code execution is opt-in.
  - **B L0:** Launched servers run as the OS user with the home directory and network reachable, so a compromised server is host-equivalent for that user. Evidence: [libraries/typescript/packages/client/src/transport/stdio.ts:109-116](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/transport/stdio.ts#L109-L116) (verified)
    - *To reach the next level:* No mount, network or resource restrictions on launched processes.
- **opt-in code mode with the E2B executor** (alt; raw 0.47, cap G1 → 0.47) ← counted
  - **S L4:** Model-written code runs in a remote E2B sandbox created per executor. Evidence: [libraries/typescript/packages/client/src/code-mode/executor-e2b.ts:54-57](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/code-mode/executor-e2b.ts#L54-L57) (verified)
  - **C L1:** Only code-mode scripts go to the sandbox; stdio servers still run on the host and tool calls from the sandbox execute on the host. Evidence: [libraries/typescript/packages/client/src/code-mode/executor-e2b.ts:240-255](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/code-mode/executor-e2b.ts#L240-L255) (verified)
    - *To reach the next level:* Locally launched servers and the tool bridge are outside the sandbox.
  - **D L0:** Code mode is off by default and its default executor is the in-process vm. Evidence: [libraries/typescript/packages/client/src/core/node.ts:351-358](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/core/node.ts#L351-L358) (verified)
    - *To reach the next level:* E2B must be selected explicitly with an API key.
  - **B L2:** The sandbox holds no host files or secrets and lives up to five minutes, but it can invoke every configured tool on the host and has the E2B template's default network access (inferred from E2B's defaults). Evidence: [libraries/typescript/packages/client/src/code-mode/executor-e2b.ts:22](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/code-mode/executor-e2b.ts#L22) (inferred)
    - *To reach the next level:* No egress restriction and no per-call teardown; tool calls proxied to the host are unrestricted.
- **Cap:** G1: Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius: 0.00 (high confidence)

Tool results, resources and tool descriptions from MCP servers go into the model's context with the same standing as anything else, without marking where they came from. Nothing changes after the agent reads a web page or a file: every write and outbound tool stays available with no approval. The documented examples combine a browser or filesystem server with other tools in one session, so a prompt injection in fetched content can read data and send it out or change things, unattended. Containing that is left entirely to the developer.

- **S L0:** Tool results are appended as ordinary tool messages; there is no detection, taint tracking or capability restriction. Evidence: [libraries/typescript/packages/agent/src/llm/toolLoop.ts:195-202](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/llm/toolLoop.ts#L195-L202); searched `rg -n -S -e 'untrusted|provenance|taint|quarantin'` in `libraries/typescript/packages/agent/src` → 0 hits (no handling of untrusted content in the agent) (verified)
  - *To reach the next level:* No mechanism at all, not even marking of untrusted content.
- **C L0:** No source is distinguished; server-supplied tool descriptions are passed to the model verbatim. Evidence: [libraries/typescript/packages/agent/src/adapters/native_adapter.ts:104-109](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/adapters/native_adapter.ts#L104-L109) (verified)
  - *To reach the next level:* Untrusted sources are not separated from principal input.
- **D L0:** No mechanism exists to be on by default. Evidence: [libraries/typescript/packages/agent/src/agents/agent_options.ts:50-87](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/agents/agent_options.ts#L50-L87) (verified)
  - *To reach the next level:* Nothing to enable.
- **B L0:** A session with a browser or fetch server, a filesystem server and other write tools (as in the examples) can leak data and act irreversibly with no human involved. Evidence: [libraries/typescript/packages/agent/examples/integrations/browser_use.ts:17-19](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/examples/integrations/browser_use.ts#L17-L19); [libraries/typescript/packages/agent/src/llm/toolLoop.ts:176-189](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/llm/toolLoop.ts#L176-L189) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions are both unattended.
- **Cap:** C5-WORSTCASE: A hijacked default session can leak data and take irreversible actions without a human.

### C6 Memory, context & configuration integrity: 0.35 (high confidence)

The agent has no long-term memory, vector store or auto-loaded instruction or settings files. Its only memory is the conversation history kept in the agent object between runs, which is on by default: it stores each prompt and the model's final answer (not raw tool results) and is re-sent as earlier turns. That history lives only in process memory and can be cleared with one call, so a poisoned answer lasts at most as long as the agent object. If a developer shares one agent object between users, they also share its history, because nothing separates users.

- **S L1:** Model answers are appended to the history unvalidated and replayed as prior assistant turns; no instruction files are auto-loaded. Evidence: [libraries/typescript/packages/agent/src/agents/mcp_agent.ts:492-501](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/agents/mcp_agent.ts#L492-L501); [libraries/typescript/packages/agent/src/agents/mcp_agent.ts:372-374](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/agents/mcp_agent.ts#L372-L374) (verified)
  - *To reach the next level:* History entries are not presented as data with provenance.
- **C L1:** The single store (conversation history) is the only path, and it has no write control; there are no auto-loaded workspace files to control. Evidence: searched `rg -n -S -e 'dotenv|AGENTS\.md|CLAUDE\.md|vector|writeFile'` in `libraries/typescript/packages/agent/src` → 1 hits (single hit is a doc comment in observability/langfuse.ts suggesting users load env themselves; no auto-loaded files or persistent stores) (verified)
  - *To reach the next level:* The history store itself has no validation or gating.
- **D L1:** History is per agent object; separating users depends on the developer creating one object per user. Evidence: [libraries/typescript/packages/agent/src/agents/mcp_agent.ts:150](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/agents/mcp_agent.ts#L150) (verified)
  - *To reach the next level:* No per-user namespace enforced by the framework.
- **B L3:** Poisoned history lives only in process memory for the life of the agent object and can be purged with clearConversationHistory(). Evidence: [libraries/typescript/packages/agent/src/agents/mcp_agent.ts:396-399](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/agents/mcp_agent.ts#L396-L399) (verified)
  - *To reach the next level:* History is not reviewed before reuse and has no rollback.
- **Cap:** none

### C7 Third-party extensions: 0.23 (medium confidence)

MCP servers are the extensions here, and the developer lists them explicitly in code; nothing in the working directory can add one. The framework launches whatever command it is given, with npx as the default command, and does not pin versions or check hashes or signatures. The official examples use npx -y with @latest packages, so a fresh, unreviewed version is downloaded and run on each start. Launched servers do get a reduced environment instead of all the parent's variables, but they run as the same user with full file and network access.

- **S L1:** Sources are developer-chosen but nothing pins or verifies them; examples use npx -y with @latest. Evidence: [libraries/typescript/packages/client/src/transport/stdio.ts:63-66](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/transport/stdio.ts#L63-L66); [libraries/typescript/packages/agent/examples/integrations/browser_use.ts:17-19](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/examples/integrations/browser_use.ts#L17-L19) (verified)
  - *To reach the next level:* No version pinning enforced or encouraged by the framework.
- **C L0:** No extension type (stdio or HTTP server) is verified. Evidence: searched `rg -n -S -e 'sha256|integrity|checksum|signature|pinned'` in `libraries/typescript/packages/client/src/transport libraries/typescript/packages/client/src/core` → 0 hits (no integrity checks where servers are configured and launched) (verified)
  - *To reach the next level:* No verification for any extension type.
- **D L1:** Servers are added explicitly in code and never from workspace files, but official examples routinely auto-install unpinned packages with npx -y, so D is lowered one level. Evidence: [libraries/typescript/packages/agent/examples/integrations/browser_use.ts:17-19](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/examples/integrations/browser_use.ts#L17-L19); [libraries/typescript/packages/agent/src/agents/mcp_agent.ts:205-212](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/agents/mcp_agent.ts#L205-L212) (verified)
  - *To reach the next level:* Examples and defaults should not auto-install unpinned packages.
- **B L2:** stdio servers are separate processes whose environment is the configured env over the MCP SDK's small default set (inferred from SDK behaviour, corroborated by the code comment), but they run as the same user. Evidence: [libraries/typescript/packages/client/src/transport/stdio.ts:112-114](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/transport/stdio.ts#L112-L114) (inferred)
  - *To reach the next level:* No per-server sandbox or scoped file and network access.
- **Cap:** none

### C8 Secrets & sensitive-data protection: 0.20 (medium confidence)

API keys and server tokens come from environment variables or the server configuration, and OAuth tokens are saved as plain files readable only by the user under ~/.mcp-use/oauth. Locally launched servers get a reduced environment, which keeps unrelated secrets in the parent process away from them. Nothing redacts secrets from logs or from tool results sent to the model, and debug logging prints full tool arguments and results. Anonymous usage telemetry to a third-party analytics service is on by default in both the TypeScript and Python libraries and is not content-free; it can be turned off with an environment variable.

- **S L1:** Secrets come from env or config; stored OAuth tokens are plaintext files with 0600 permissions; there is no masking type or redaction helper. Evidence: [libraries/typescript/packages/client/src/auth/storage-file.ts:26](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/auth/storage-file.ts#L26); [libraries/typescript/packages/client/src/auth/storage-file.ts:67](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/auth/storage-file.ts#L67); searched `rg -n -S -e 'redact|scrub|SecretString'` in `libraries/typescript/packages/agent/src libraries/typescript/packages/client/src/transport libraries/typescript/packages/client/src/core` → 0 hits (no redaction in agent, transport or client core) (verified)
  - *To reach the next level:* No type-level masking or log filters on the main paths.
- **C L1:** Only the subprocess-environment path is narrowed (configured env over the SDK default set, inferred from SDK behaviour); logs and model-bound tool results are not. Evidence: [libraries/typescript/packages/client/src/transport/stdio.ts:112-114](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/transport/stdio.ts#L112-L114); [libraries/typescript/packages/client/src/transport/base.ts:713-717](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/transport/base.ts#L713-L717) (inferred)
  - *To reach the next level:* Logs and transcripts are not protected.
- **D L0:** Telemetry to PostHog is on by default with an opt-out environment variable and is not content-free; the Python library's own documentation says it collects query and response content. Evidence: [libraries/typescript/packages/client/src/telemetry/telemetry.ts:154](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/telemetry/telemetry.ts#L154); [libraries/python/mcp_use/telemetry/telemetry.py:139](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/python/mcp_use/telemetry/telemetry.py#L139); [docs/python/development/telemetry.mdx:25](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/docs/python/development/telemetry.mdx#L25) (verified)
  - *To reach the next level:* Telemetry should be opt-in or strictly content-free.
- **B L1:** The process holds long-lived model API keys and per-server tokens; their scope is whatever the developer issued. Evidence: [libraries/typescript/packages/client/src/core/config.ts:193-201](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/core/config.ts#L193-L201) (verified)
  - *To reach the next level:* No short-lived or narrowly scoped credentials issued by the framework.
- **Cap:** none

### C9 Audit & traceability: 0.28 (medium confidence)

The default agent keeps no record of what it did: tool calls and their arguments are logged only at debug level, which is off by default. The streaming API hands each tool call to the developer's code as an event, so an application can build its own log, but the framework does not. The LangChain-based agent can send traces to Langfuse when the optional package and API keys are present; those traces are structured but only cover that agent variant and depend on an external service.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Tool calls are recorded only by debug-level log lines. Evidence: [libraries/typescript/packages/client/src/transport/base.ts:713-717](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/transport/base.ts#L713-L717) (verified)
    - *To reach the next level:* No structured record of tool calls at a default log level.
  - **C L0:** Nothing is recorded by default for any tool path. Evidence: [libraries/typescript/packages/agent/src/llm/toolLoop.ts:176-203](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/llm/toolLoop.ts#L176-L203) (verified)
    - *To reach the next level:* Even the main tool path is not recorded by default.
  - **D L0:** Recording requires debug logging or the developer's own handling of stream events. Evidence: [libraries/typescript/packages/client/src/utils/logging.ts:48-54](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/utils/logging.ts#L48-L54) (verified)
    - *To reach the next level:* No record is on by default.
  - **B L0:** With no record written, nothing is flushed or durable. Evidence: [libraries/typescript/packages/client/src/transport/base.ts:713-717](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/transport/base.ts#L713-L717) (verified)
    - *To reach the next level:* No per-action record exists to flush.
- **opt-in Langfuse tracing (LangChain agent)** (alt; raw 0.28, cap G1 → 0.28) ← counted
  - **S L2:** The LangChain agent attaches a Langfuse callback handler that records tool calls with inputs and outputs (inferred from the Langfuse LangChain integration's behaviour). Evidence: [libraries/typescript/packages/agent/src/agents/mcp_agent_langchain.ts:391-393](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/agents/mcp_agent_langchain.ts#L391-L393) (inferred)
    - *To reach the next level:* No actor attribution or correlation beyond what Langfuse adds.
  - **C L1:** Only the LangChain agent variant is traced; the default native MCPAgent has no observability hook. Evidence: searched `rg -n -S -e 'observ|Langfuse|callbacks'` in `libraries/typescript/packages/agent/src/agents/mcp_agent.ts` → 1 hits (single hit is the AgentStep observation field; native MCPAgent has no observability wiring) (verified)
    - *To reach the next level:* The native agent's tool calls are not covered.
  - **D L0:** Tracing needs the optional package and Langfuse API keys; without them it is disabled. Evidence: [libraries/typescript/packages/agent/src/observability/langfuse.ts:343-347](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/observability/langfuse.ts#L343-L347) (verified)
    - *To reach the next level:* Off unless the developer configures it.
  - **B L1:** Trace export is asynchronous and best-effort through the Langfuse client (inferred from library behaviour). Evidence: [libraries/typescript/packages/agent/src/observability/manager.ts:62](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/observability/manager.ts#L62) (inferred)
    - *To reach the next level:* Records are not durable per action.
- **Cap:** G1: Opt-in mechanism: off in the scored default configuration.

### C10 Limits & kill switch: 0.55 (medium confidence)

Each run is capped at 10 model turns by default (5 for the LangChain variant), and the model cannot raise that cap. Individual tool calls rely on the MCP SDK's default request timeout, but model calls have no timeout and there is no token or cost budget. A caller can pass an abort signal, which is checked between turns and between tool calls; a tool call already in progress is not cancelled and runs to completion.

- **S L2:** A step cap of 10 is enforced in code, and tool calls inherit the MCP SDK's default per-request timeout (inferred from SDK behaviour; the client's own docs state a 60-second default); no token or cost cap. Evidence: [libraries/typescript/packages/agent/src/llm/toolLoop.ts:151-161](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/llm/toolLoop.ts#L151-L161); [libraries/typescript/packages/client/src/react/types.ts:636](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/react/types.ts#L636) (inferred)
  - *To reach the next level:* No token or cost budget and no timeout on model calls.
- **C L2:** The cap covers the single agent loop and every tool call goes through the SDK request path with its timeout; there are no sub-agents to escape the budget. Evidence: [libraries/typescript/packages/client/src/transport/base.ts:714-716](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/client/src/transport/base.ts#L714-L716) (inferred)
  - *To reach the next level:* No shared budget across agents or caps on concurrent agent objects.
- **D L3:** maxSteps defaults to 10 and is set by the developer per agent or per run; the model has no way to change it. Evidence: [libraries/typescript/packages/agent/src/agents/mcp_agent.ts:143](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/agents/mcp_agent.ts#L143); [libraries/typescript/packages/agent/src/agents/mcp_agent.ts:423-430](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/agents/mcp_agent.ts#L423-L430) (verified)
  - *To reach the next level:* No hard ceiling that configuration cannot exceed.
- **B L2:** Ceilings are moderate; abort is checked between steps and the signal is not passed to tool calls, so in-flight calls finish. Evidence: [libraries/typescript/packages/agent/src/llm/toolLoop.ts:176-181](https://github.com/mcp-use/mcp-use/blob/90b6b8b197e095a2a800f3d6ce73e5ca4d1601e8/libraries/typescript/packages/agent/src/llm/toolLoop.ts#L176-L181) (verified)
  - *To reach the next level:* Stopping does not cancel pending tool calls, and there is no cost ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Tool results, resources and tool descriptions from configured servers enter context unmarked (libraries/typescript/packages/agent/src/llm/toolLoop.ts:195-202); examples include a browser server (libraries/typescript/packages/agent/examples/integrations/browser_use.ts:19). · [B] sensitive data/systems: Per-server credentials and the OS user's files via stdio servers (libraries/typescript/packages/client/src/core/config.ts:201, libraries/typescript/packages/client/src/transport/stdio.ts:109-116). · [C] state change / egress: Every write or network tool of every configured server is callable without approval (libraries/typescript/packages/agent/src/llm/toolLoop.ts:181). · Same default session? Yes

## Highest-impact improvements
1. Add an approval hook to the tool loop that is on by default for tools not annotated read-only, showing the exact tool name and arguments. (C2 S L0→L3, +0.225 before caps; Playbook 5)
2. Make the approval default-on and require an explicit, loudly named option to auto-approve. (C2 D L0→L3, +0.150 before caps; Playbook 5)
3. Once a tool result from an untrusted source enters the session, route write and egress tools through the approval hook. (C5 S L0→L3, +0.225 before caps; Playbook 1)
4. Make usage telemetry opt-in, or restrict it to an explicit allowlist of content-free fields in both libraries. (C8 D L0→L2, +0.100 before caps; Playbook 4)
5. Write a structured record of every tool call (name, arguments, result status, timestamp) at the default log level. (C9 S L0→L2, +0.150 before caps; Playbook 1 step 3)

## Re-audit log
- C3 S: L2 → L1. Typed schema validation exists only in the mcp-use server framework, which protects servers built with it; the agent's own path to configured servers passes arguments through, leaving only the tool-name denylist.
- C8 D: L1 → L0. L1 requires content-free telemetry; telemetry is on by default and the Python library's documentation states it collects query and response content, so the lower anchor applies.
- C6 B: L4 → L3. Structural absence was considered and rejected: conversation history is on by default and replays model output across runs of one agent object, so the surface exists; it is in-memory and purgeable, which meets L3, not L4.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the TypeScript agent stack (@mcp-use/agent MCPAgent over @mcp-use/client). The MCP server framework, Inspector, CLI, tunnel and create-mcp-use-app were read only where they affect the agent path; their own protections (localhost default bind, DNS-rebinding checks, per-request OAuth for servers) are not credited here because they protect servers built with the framework, not the agent.
- The Python library (mcp_use) mirrors the agent design and was checked only for differences: its telemetry, by its own documentation, includes query and response text, and it offers an opt-in E2B sandbox for stdio servers. It was not scored separately.
- Behaviour of third-party libraries (the MCP SDK's default subprocess environment and request timeout, E2B sandbox network defaults, the Langfuse LangChain handler) is inferred from their documented behaviour, not read at a pinned version; parameters that depend on it are marked inferred.
- The managed remote agent (agentId, Manufact cloud) is a hosted service outside the repository and was not assessed.
- No text in the repository attempting to steer reviewers was found in the files read.
