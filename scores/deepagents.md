# Defense-in-Depth Score: Deep Agents

**Repo:** https://github.com/langchain-ai/deepagents · **Commit:** `44bd956bf8ca86c7cf0831132d69d38e06dd1573` · **Reviewed:** 2026-10-05
**What it is:** Batteries-included agent harness (planning, filesystem, subagents) + Deep Agents CLI
**Category:** Agent Frameworks
**Scored configuration:** The deepagents Python library via create_deep_agent() with default arguments (StateBackend, no interrupt_on, permissions, memory, skills, checkpointer or store); the Deep Agents Code CLI and partner sandbox packages are footnoted where relevant.
**Agent surface (default):** code execution opt-in · filesystem write opt-in · network egress opt-in · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents yes · external communication opt-in

## Score: 2.8 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L1 | L1 | L0 | 0.12 | — | **0.12** | High |
| C2 | Approval gates | L2 | L1 | L0 | L1 | 0.28 | G1 | **0.28** | Medium |
| C3 | Tool & action scoping | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C4 | Code-execution isolation | L4 | L2 | L0 | L2 | 0.55 | G1 | **0.50** | Medium |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C7 | Third-party extensions | L1 | L0 | L1 | L0 | 0.12 | — | **0.12** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C9 | Audit & traceability | L2 | L2 | L0 | L1 | 0.35 | G1 | **0.35** | Medium |
| C10 | Limits & kill switch | L2 | L1 | L1 | L1 | 0.33 | — | **0.33** | High |


Out of the box Deep Agents is fairly contained: files live in an in-memory virtual filesystem and the execute tool does nothing until a developer picks an execution backend. Everything protective beyond that is opt-in: human approval, path permission rules, remote sandboxes and persistence all need explicit configuration, and the core package's own shell backend runs commands directly on the host. The dominant risk is prompt injection through tool results driving the developer's tools with no approval step, plus profile plugins from any installed package running inside the agent process.

## Critical gaps
- The agent and every registered tool act with the host process's full ambient credentials; the framework has no per-request authorization or credential scoping. (ASI03; C1) — [README.md:112](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/README.md#L112); [libs/deepagents/deepagents/backends/local_shell.py:197-198](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/backends/local_shell.py#L197-L198)
- Content the agent reads can steer it into using any registered tool, including ones that send data out or change state, with no approval or capability restriction in the default configuration. (ASI01, LLM01; C5) — [README.md:112](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/README.md#L112); [libs/deepagents/deepagents/graph.py:283](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/graph.py#L283)
- Profile plugins declared by any installed package are imported and run inside the agent process with its full credentials and environment. (ASI04, LLM03; C7) — [libs/deepagents/deepagents/profiles/_builtin_profiles.py:211-232](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/profiles/_builtin_profiles.py#L211-L232)

## Criterion details

### C1 Identity & least privilege — 0.12 (high)

Deep Agents has no identity or authorization layer of its own: every tool the developer registers runs inside the application process with whatever credentials that process holds, and model provider keys come from the environment. The one place the library narrows authority is its local shell backend, which starts commands with an empty environment unless the developer passes inherit_env=True. Nothing checks per request whether the agent should be allowed to act, so a hijacked agent acts with the full authority of the host process.

- **S L0:** The framework has no credential or identity primitive; tools and the model client use the host process's ambient credentials. — searched `rg -n -i 'authoriz|least.?privilege|scoped_token|on_behalf'` in `libs/deepagents/deepagents` → 2 hits (Both hits are comments (glob root wording and the HITL edit path), not authorization code.); [libs/deepagents/deepagents/graph.py:272-274](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/graph.py#L272-L274) (verified)
  - *To reach the next level:* No per-tool or per-request credential scoping or authorization check exists in the framework.
- **C L1:** The only subprocess path the library ships (LocalShellBackend) starts with an empty environment by default, but user tools and in-process code keep full ambient credentials. — [libs/deepagents/deepagents/backends/local_shell.py:196-202](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/backends/local_shell.py#L196-L202); [libs/deepagents/deepagents/backends/local_shell.py:116](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/backends/local_shell.py#L116) (verified)
  - *To reach the next level:* No shared authorization layer covers user tools, sub-agents, or async remote sub-agents.
- **D L1:** The narrower shell environment is the default, but a single constructor flag copies the whole os.environ into every command; the process itself always runs with the operator's authority. — [libs/deepagents/deepagents/backends/local_shell.py:116](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/backends/local_shell.py#L116); [libs/deepagents/deepagents/backends/local_shell.py:197-198](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/backends/local_shell.py#L197-L198) (verified)
  - *To reach the next level:* Least privilege is not the default for the agent process; widening needs no warning.
- **B L0:** Following the framework rule, a hijacked agent reaches everything the developer's tools and the host process can reach, potentially the operator's whole account. — [README.md:112](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/README.md#L112) (verified)
  - *To reach the next level:* Nothing in the framework bounds what the process credentials can reach.
- **Cap:** none

### C2 Approval gates — 0.28 (medium)

Deep Agents can pause any tool call for human approval through LangChain's human-in-the-loop middleware, with approve, edit, reject and respond decisions, per-call predicates, and filesystem permission rules that can mark paths as needing approval. Sub-agents inherit the parent's interrupt settings. But the gate is off unless the developer passes interrupt_on or interrupt-mode permissions, and it only covers tools named in that mapping, so by default every tool call, including any shell or write tool, runs immediately.

- **S L2:** Approval is per call and offers approve, edit, reject and respond decisions with optional per-call predicates; the approval UI and execution of the approved call live in the LangChain dependency, so the exact-call guarantee is inferred. — [libs/deepagents/deepagents/middleware/_fs_interrupt.py:174-180](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/middleware/_fs_interrupt.py#L174-L180); [libs/deepagents/deepagents/graph.py:941-942](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/graph.py#L941-L942) (inferred)
  - *To reach the next level:* Exact-call rendering and approved-equals-executed are not verifiable in this repository.
- **C L1:** Only tools named in interrupt_on (or matched by interrupt-mode filesystem rules) are gated; any other tool, and runs on remote async sub-agents, proceed without approval. — [libs/deepagents/deepagents/graph.py:770](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/graph.py#L770); [libs/deepagents/deepagents/middleware/filesystem.py:402-412](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/middleware/filesystem.py#L402-L412) (verified)
  - *To reach the next level:* Unknown or unlisted tools are not gated by default and remote async sub-agents bypass the gate.
- **D L0:** interrupt_on defaults to None, so no HumanInTheLoopMiddleware is added in the default configuration. — [libs/deepagents/deepagents/graph.py:283](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/graph.py#L283); [libs/deepagents/deepagents/graph.py:941-942](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/graph.py#L941-L942) (verified)
  - *To reach the next level:* Approval is opt-in.
- **B L1:** Built-in file writes go to ephemeral in-state virtual files by default, which is low impact, but developer tools such as shell, email or API calls are irreversible and unbounded. — [libs/deepagents/deepagents/graph.py:648](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/graph.py#L648) (verified)
  - *To reach the next level:* No checkpoint or rollback for external actions and no rate limits on consequential calls.
- **Cap:** G1 — Human approval only exists when the developer passes interrupt_on or interrupt-mode permission rules.

### C3 Tool & action scoping — 0.45 (high)

The built-in file tools are reasonably narrow: paths are normalised and '..' or '~' components are rejected, the default backend is an in-memory virtual filesystem, and the on-disk backend resolves paths and checks they stay under its root by default. Optional permission rules can allow, deny or require approval per path. Developer-registered tools only get schema typing, and the shipped local shell backend passes a raw shell string through. The default tool set includes write, edit and delete tools and an execute tool that only works when an execution backend is configured.

- **S L2:** File-tool paths reject traversal components and the disk backend checks resolved-path containment, but the framework's validation for developer tools is schema typing only and the shell backend takes a raw command string. — [libs/deepagents/deepagents/backends/utils.py:884-887](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/backends/utils.py#L884-L887); [libs/deepagents/deepagents/backends/filesystem.py:203-213](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/backends/filesystem.py#L203-L213); [libs/deepagents/deepagents/backends/local_shell.py:304-314](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/backends/local_shell.py#L304-L314) (verified)
  - *To reach the next level:* No allowlist validation layer for developer tools; the shell backend is raw passthrough.
- **C L2:** Every built-in file tool validates its path argument, while developer-registered and MCP tools pass through unvalidated. — [libs/deepagents/deepagents/middleware/filesystem.py:2183-2198](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/middleware/filesystem.py#L2183-L2198) (verified)
  - *To reach the next level:* No shared validation layer wraps developer or extension tools.
- **D L2:** The default backend confines built-in writes to in-state virtual files and execution is unavailable, but the default tool set still includes write, edit and delete tools and permission rules default to allow. — [libs/deepagents/deepagents/graph.py:648](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/graph.py#L648); [libs/deepagents/deepagents/middleware/filesystem.py:3033-3036](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/middleware/filesystem.py#L3033-L3036); [libs/deepagents/deepagents/middleware/filesystem.py:402-412](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/middleware/filesystem.py#L402-L412) (verified)
  - *To reach the next level:* The default tool set is not read-only and there are no per-task tool allowlists.
- **B L1:** Following the framework rule, misuse of developer tools is bounded only by what those tools reach; built-in tools are scoped to virtual state. — [libs/deepagents/deepagents/graph.py:648](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/graph.py#L648) (verified)
  - *To reach the next level:* No quantity bounds or workspace scoping apply to developer tools.
- **Cap:** none

### C4 Code-execution isolation — 0.50 (medium)

With the default in-memory backend there is no code execution: the execute tool returns an error. Execution is enabled by choosing a backend. The core package's LocalShellBackend runs model-written commands through the host shell with no isolation (its docstring says so), though with an empty environment by default. The repository also ships remote sandbox backends (Daytona, Modal, Runloop, Vercel, LangSmith) that run both commands and file operations inside a provider sandbox, but all of these are opt-in and their network and lifetime settings come from the developer's provider configuration.

- **S L4:** Remote sandbox backends run commands in a provider-hosted sandbox via the provider API, a remote ephemeral sandbox service. — [libs/partners/daytona/langchain_daytona/sandbox.py:23](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/partners/daytona/langchain_daytona/sandbox.py#L23); [libs/partners/modal/langchain_modal/sandbox.py:91](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/partners/modal/langchain_modal/sandbox.py#L91) (verified)
- **C L2:** With a sandbox backend both execute and the file tools run through the sandbox, but CompositeBackend can route paths to host backends and the QuickJS REPL middleware runs in-process. — [libs/deepagents/deepagents/backends/sandbox.py:1536](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/backends/sandbox.py#L1536); [libs/deepagents/deepagents/backends/composite.py:228](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/backends/composite.py#L228) (verified)
  - *To reach the next level:* Not every model-reachable execution path is forced through the sandbox.
- **D L0:** Sandboxing is opt-in; the only execution backend in the core package runs on the host with no isolation. — [libs/deepagents/deepagents/backends/local_shell.py:227](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/backends/local_shell.py#L227); [libs/deepagents/deepagents/backends/local_shell.py:304-314](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/backends/local_shell.py#L304-L314); [libs/deepagents/deepagents/graph.py:648](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/graph.py#L648) (verified)
  - *To reach the next level:* Isolation is not on by default.
- **B L2:** The wrappers inject no host environment into the provider sandbox, but network egress, mounts and lifetime are whatever the developer's provider sandbox is configured with. — [libs/partners/daytona/langchain_daytona/sandbox.py:30-50](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/partners/daytona/langchain_daytona/sandbox.py#L30-L50) (inferred)
  - *To reach the next level:* No enforced egress allowlist, resource limits or per-run ephemerality in the wrapper.
- **Cap:** G1 — Sandboxed execution requires the developer to choose a sandbox backend; the default backend has no execution and the core package's execution backend is unsandboxed.

### C5 Untrusted input blast radius — 0.00 (high)

Tool results, file contents, sub-agent replies and remote async sub-agent outputs all enter the model's context as ordinary messages with no provenance or taint tracking, and nothing changes what the agent may do after reading untrusted content. The README states the project follows a 'trust the LLM' model and leaves boundaries to tools and sandboxes. In a typical deployment where the agent reads external content and holds tools that write or send data, a successful injection can drive those tools unattended.

- **S L0:** No structural limit on a hijacked agent; the project documents a 'trust the LLM' model. — [README.md:112](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/README.md#L112); searched `rg -n -i 'taint|provenance'` in `libs/deepagents/deepagents` → 0 hits (No taint or provenance tracking.) (verified)
  - *To reach the next level:* No approval or capability restriction triggered by untrusted content.
- **C L0:** Tool results and sub-agent outputs re-enter context with the same standing as other messages. — [libs/deepagents/deepagents/middleware/subagents.py:731](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/middleware/subagents.py#L731) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished.
- **D L0:** Nothing is on by default. — [libs/deepagents/deepagents/graph.py:283](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/graph.py#L283) (verified)
  - *To reach the next level:* No control exists to enable.
- **B L0:** Following the framework rule, a hijacked agent can use developer tools to leak data and take irreversible actions with no human involved. — [libs/deepagents/deepagents/graph.py:283](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/graph.py#L283); [README.md:112](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/README.md#L112) (verified)
  - *To reach the next level:* No default break in the untrusted-input, sensitive-data, egress combination.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.30 (high)

Memory is opt-in: when enabled, AGENTS.md-style files are read from the backend and appended to the system prompt on every call, and the built-in prompt tells the agent to update them with edit_file as it learns. The memory block is labelled as file data to treat as reference, but writes are not validated, gated or versioned unless the developer adds permission rules. With the default in-memory backend memory lasts one thread; with the store backend it persists across threads in whatever namespace the developer's factory returns. The library loads no workspace config or .env files on its own.

- **S L1:** The model writes memory files freely and they are re-injected into the system prompt, wrapped in a tag telling the model they are data. — [libs/deepagents/deepagents/middleware/memory.py:105-120](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/middleware/memory.py#L105-L120); [libs/deepagents/deepagents/middleware/memory.py:361](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/middleware/memory.py#L361) (verified)
  - *To reach the next level:* Memory writes are not validated or gated by default.
- **C L1:** Only filesystem permission rules, if configured, can gate writes to memory and skill paths; summaries and offloaded history are not controlled. — [libs/deepagents/deepagents/middleware/filesystem.py:402-412](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/middleware/filesystem.py#L402-L412) (verified)
  - *To reach the next level:* No control over skill files, summaries or offloaded history.
- **D L2:** The default backend is per-thread state, and the persistent store backend requires an explicit namespace factory rather than sharing a global default. — [libs/deepagents/deepagents/backends/store.py:94-103](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/backends/store.py#L94-L103); [libs/deepagents/deepagents/graph.py:648](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/graph.py#L648) (verified)
  - *To reach the next level:* The model is not prevented from writing to other namespaces routed through the backend and there are no retention limits.
- **B L1:** With a persistent backend, poisoned memory persists across the user's threads and steers tool use in later sessions. — [libs/deepagents/deepagents/backends/store.py:94](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/backends/store.py#L94) (verified)
  - *To reach the next level:* Persistent memory is not inspected or reviewed before reuse.
- **Cap:** none

### C7 Third-party extensions — 0.12 (high)

The library does not launch MCP servers or download tools itself; developers pass tools in code. But at first profile lookup it imports and runs every entry point any installed Python distribution declares under its profile plugin groups, in-process, with no allowlist, pinning or integrity check. Those plugins can add middleware and change prompts and tool behaviour. Skills are markdown instructions, not code.

- **S L1:** Profile plugins come from whatever package versions are installed, with no hash, signature or registry check. — [libs/deepagents/deepagents/profiles/_builtin_profiles.py:211-232](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/profiles/_builtin_profiles.py#L211-L232) (verified)
  - *To reach the next level:* No pinning or integrity verification for profile plugins.
- **C L0:** No extension type is verified. — searched `rg -n 'sha256|hashlib'` in `libs/deepagents/deepagents/profiles` → 0 hits (No integrity checks in the plugin loader.) (verified)
  - *To reach the next level:* No verification on any extension type.
- **D L1:** Any installed distribution declaring the entry-point groups is loaded automatically; installing the package is the only consent. — [libs/deepagents/deepagents/profiles/_builtin_profiles.py:211-232](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/profiles/_builtin_profiles.py#L211-L232) (verified)
  - *To reach the next level:* Plugins are not shown or individually enabled before they run.
- **B L0:** Plugins run in the agent's own process with all of its credentials and environment. — [libs/deepagents/deepagents/profiles/_builtin_profiles.py:220-232](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/profiles/_builtin_profiles.py#L220-L232) (verified)
  - *To reach the next level:* Plugins are not isolated from the agent process.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.30 (high)

Provider API keys are read from environment variables by the LangChain model clients, and the library adds no masking or redaction anywhere: shell output, file contents and tool results go to the model as-is. The local shell backend does start commands with an empty environment by default, which keeps keys out of subprocesses. The library sends no telemetry of its own; LangSmith tracing is opt-in. The memory prompt asks the model not to store credentials, which is guidance, not a control.

- **S L1:** Secrets come from environment variables; the only protection is the shell backend's empty default environment. — [libs/deepagents/deepagents/backends/local_shell.py:196-202](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/backends/local_shell.py#L196-L202); searched `rg -n -i 'secretstr|redact'` in `libs/deepagents/deepagents` → 2 hits (Both hits are one comment about permission-filtered grep results, not secret redaction.) (verified)
  - *To reach the next level:* No masking types or redaction filters on logs or model-bound messages.
- **C L1:** Only the subprocess environment path is protected. — [libs/deepagents/deepagents/backends/local_shell.py:196-202](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/backends/local_shell.py#L196-L202) (verified)
  - *To reach the next level:* Logs, traces and model-bound messages are not protected.
- **D L2:** No telemetry is initialised by the library and payload logging is not on by default. — searched `rg -n -i 'sentry|posthog|telemetry'` in `libs/deepagents/deepagents` → 0 hits (No telemetry SDK in the library.) (verified)
  - *To reach the next level:* Redaction is not available to enable.
- **B L1:** Long-lived provider keys and any developer-tool credentials sit in the host process environment. — [libs/deepagents/deepagents/backends/local_shell.py:116](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/backends/local_shell.py#L116) (verified)
  - *To reach the next level:* No short-lived or task-scoped credentials.
- **Cap:** none

### C9 Audit & traceability — 0.35 (medium)

The library writes no audit record of its own. Tool calls and results are kept in the conversation state returned to the caller, but sub-agent runs only return their final message, and nothing is persisted unless the developer adds a checkpointer or enables LangSmith tracing. When tracing is on, the library tags its runs and marks sub-agent runs so they appear in the trace tree; the tracer itself lives in LangChain, so that behaviour is inferred here.

- **S L2:** Optional LangSmith tracing records each model and tool run in a run tree, with deepagents adding integration and sub-agent tags. — [libs/deepagents/deepagents/graph.py:997](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/graph.py#L997); [libs/deepagents/deepagents/middleware/subagents.py:538](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/middleware/subagents.py#L538) (inferred)
  - *To reach the next level:* No actor attribution of the requesting principal or approver, and no tamper-evident storage, in this repository.
- **C L2:** Tracing covers built-in and sub-agent runs through the shared callback context; approvals and denials are not recorded by the library. — [libs/deepagents/deepagents/middleware/subagents.py:835-837](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/middleware/subagents.py#L835-L837) (inferred)
  - *To reach the next level:* Approvals, denials and remote async sub-agent internals are not recorded.
- **D L0:** Tracing and checkpointing are opt-in; by default records exist only in memory and sub-agent internals are dropped. — [libs/deepagents/deepagents/middleware/subagents.py:731-746](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/middleware/subagents.py#L731-L746); [libs/deepagents/deepagents/graph.py:287](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/graph.py#L287) (verified)
  - *To reach the next level:* Recording is not on by default.
- **B L1:** Without a checkpointer, state is lost when the process ends; tracing is best-effort background upload. — [libs/deepagents/deepagents/graph.py:287](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/graph.py#L287) (verified)
  - *To reach the next level:* Records are not durably flushed per action.
- **Cap:** G1 — Tracing and checkpointing are opt-in; the default records nothing durable.

### C10 Limits & kill switch — 0.33 (high)

Each agent graph is compiled with a recursion limit of 9,999 steps, local shell commands default to a 120-second timeout clamped to at most an hour, and glob searches have a wall-clock deadline. There is no token or cost budget. Each sub-agent invocation starts its own step count, so delegation is not charged against the parent's limit, and nothing caps how many sub-agents run in parallel. A shell command that times out has its shell killed, but processes it spawned in their own session are not.

- **S L2:** A step cap plus per-command and glob timeouts are enforced in code. — [libs/deepagents/deepagents/graph.py:995](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/graph.py#L995); [libs/deepagents/deepagents/middleware/filesystem.py:1773](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/middleware/filesystem.py#L1773); [libs/deepagents/deepagents/backends/local_shell.py:23](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/backends/local_shell.py#L23) (verified)
  - *To reach the next level:* No token or cost cap and no repeated-action breaker.
- **C L1:** The limit applies per graph; sub-agents get a fresh step count and developer tools have no timeouts. — [libs/deepagents/deepagents/middleware/subagents.py:835-837](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/middleware/subagents.py#L835-L837) (verified)
  - *To reach the next level:* Sub-agents do not share the parent's budget.
- **D L1:** The default step cap of 9,999 is very large. — [libs/deepagents/deepagents/graph.py:995](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/graph.py#L995) (verified)
  - *To reach the next level:* Defaults are not sized to bound damage.
- **B L1:** A runaway can take thousands of steps and spawn sub-agents with their own budgets before stopping. — [libs/deepagents/deepagents/graph.py:995](https://github.com/langchain-ai/deepagents/blob/44bd956bf8ca86c7cf0831132d69d38e06dd1573/libs/deepagents/deepagents/graph.py#L995) (verified)
  - *To reach the next level:* No tight per-run time or cost ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: File contents and tool results enter context unmarked (libs/deepagents/deepagents/middleware/subagents.py:731; README.md:112) · [B] sensitive data/systems: Developer tools and provider keys run with the host process's ambient credentials (libs/deepagents/deepagents/backends/local_shell.py:116) · [C] state change / egress: Write, edit, delete and developer tools run without approval by default (libs/deepagents/deepagents/graph.py:283) · Same default session? Yes

## Highest-impact improvements
1. Add a default interrupt_on for execute and any developer tool not marked read-only whenever an execution-capable backend is configured. — C2 D L0→L2, +0.100 before caps (Playbook 5)
2. Require an explicit allowlist of profile plugin entry points instead of loading every installed one. — C7 D L1→L3, +0.100 before caps (Playbook 3)
3. Lower the default recursion limit and charge sub-agent steps against the parent's budget. — C10 C L1→L3, +0.150 before caps (Playbook 3 step 3)
4. Gate egress and state-changing tools behind approval once a tool result from an untrusted source enters the thread. — C5 S L0→L2, +0.150 before caps (Playbook 1)
5. Return sub-agent tool-call histories (or a structured audit record) to the parent so a run can be reconstructed without tracing. — C9 D L0→L1, +0.050 before caps

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was installed, built, or run.
- The human-in-the-loop middleware, create_agent loop, tool-argument validation and LangSmith tracer live in the langchain/langchain-core/langgraph dependencies, which are not in this repository; their behaviour is inferred and the affected parameters are marked inferred.
- The Deep Agents Code CLI (libs/code), ACP adapter, GitHub Action (action.yml), evals and talon packages were not scored; the CLI has its own approval, MCP and memory systems and would need a separate audit.
- Partner sandbox packages (Daytona, Modal, Runloop, Vercel, QuickJS) were read only for how they route execution; provider-side sandbox defaults (network, lifetime, resources) were not examined.
- No text aimed at AI reviewers was found in README.md or the AGENTS.md files.
