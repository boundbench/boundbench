# Defense-in-Depth Score: Eino

**Repo:** https://github.com/cloudwego/eino · **Commit:** `58d184303f4e73413c454a9626f79c17af0fd9fb` · **Reviewed:** 2026-10-03
**What it is:** LLM/AI application and agent framework in Go (ByteDance)
**Category:** Agent Frameworks
**Scored configuration:** adk.NewChatModelAgent (and deep.New) run via adk.NewRunner with default arguments and developer-registered tools, including the built-in filesystem/execute middleware when a Backend and Shell are supplied.
**Agent surface (default):** code execution opt-in · filesystem write opt-in · network egress opt-in · external credentials opt-in · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents opt-in · external communication opt-in

## Score: 1.4 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | Medium |
| C2 | Approval gates | L2 | L1 | L0 | L0 | 0.23 | G1 | **0.23** (alt) | High |
| C3 | Tool & action scoping | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | Medium |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | Medium |
| C7 | Third-party extensions | L1 | L0 | L0 | L0 | 0.07 | — | **0.07** | High |
| C8 | Secrets & sensitive-data protection | L1 | L0 | L2 | L1 | 0.23 | — | **0.23** | Medium |
| C9 | Audit & traceability | L2 | L2 | L0 | L0 | 0.30 | G1 | **0.30** (alt) | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |


Eino is an orchestration library: it runs whatever tools you register, immediately and in parallel, with no approval gate, no sandbox, and no authorization layer by default. Its built-in shell 'execute' tool passes model-written commands straight to your Shell implementation while telling the model it runs in an isolated sandbox. The dominant risk is prompt injection through tool results turning a DeepAgent with shell, file, and web tools into an unattended attacker; all containment must be built by the integrator, using the opt-in interrupt/resume primitive for approvals.

## Critical gaps
- Assuming a successful prompt injection, the documented DeepAgent configuration can both exfiltrate data and take irreversible actions with no human involved (C5 worst case). (ASI01, T6, LLM01; C5) — [README.md:81](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/README.md#L81); [compose/tool_node.go:1240](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/compose/tool_node.go#L1240)
- The built-in execute tool forwards raw model commands to the developer's Shell with no isolation, while its default description tells the model commands run in an isolated sandbox. (ASI05, T11, LLM05; C4) — [adk/middlewares/filesystem/filesystem.go:1028-1031](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/filesystem/filesystem.go#L1028-L1031); [adk/middlewares/filesystem/prompt.go:206](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/filesystem/prompt.go#L206)
- Skill packages are auto-discovered without pinning or consent, and the framework instructs the model to run their scripts through the unsandboxed execute tool with full agent authority. (ASI04, T17, LLM03; C7) — [adk/middlewares/skill/prompt.go:38](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/skill/prompt.go#L38); [adk/middlewares/skill/filesystem_backend.go:47-48](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/skill/filesystem_backend.go#L47-L48)

## Criterion details

### C1 Identity & least privilege — 0.05 (medium)

Eino's core library holds no credentials of its own and has no identity or authorization layer: every tool a developer registers runs as an ordinary Go function inside the host process, with whatever API keys, cloud credentials, and file access that process has. Nothing in the framework scopes a tool to a narrower identity or checks, per call, whether the requesting user may perform the action. The only mention of authorization is a line in the DeepAgent system prompt, which is not an enforcement mechanism. How much a hijacked agent can do is therefore decided entirely by the tools the developer wires in.

- **S L0:** Tools execute in-process with the host process's ambient authority; the framework offers no per-tool identity, scoped credential, or authorization primitive. — [compose/tool_node.go:1106](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/compose/tool_node.go#L1106); searched `rg -n -i 'authoriz|principal|credential|least.privilege' --glob !*_test.go --type go` in `adk compose flow components callbacks` → 2 hits (Both hits are the DeepAgent system prompt (adk/prebuilt/deep/prompt.go:62,99) asking the model to require 'authorization context' for dual-use security work - a prompt, not a control.) (verified)
  - *To reach the next level:* No per-tool or per-capability credential scoping or deterministic authorization check in the tool path.
- **C L0:** Tools construct and use their own clients inside their Go function bodies; no authorization layer sits between ToolsNode dispatch and tool execution. — [components/tool/utils/invokable_func.go:191](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/components/tool/utils/invokable_func.go#L191); [compose/tool_node.go:1106](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/compose/tool_node.go#L1106) (verified)
  - *To reach the next level:* No shared authorization layer that every tool call (including sub-agent tools) passes through.
- **D L0:** The default ChatModelAgent and DeepAgent constructors run tools with full process authority; least privilege is left entirely to the developer. — searched `rg -n -i 'authoriz|principal|credential|least.privilege' --glob !*_test.go --type go` in `adk compose flow components callbacks` → 2 hits (Both hits are the DeepAgent system prompt (adk/prebuilt/deep/prompt.go:62,99) asking the model to require 'authorization context' for dual-use security work - a prompt, not a control.); [README.md:36](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/README.md#L36) (verified)
  - *To reach the next level:* No narrower default identity; least privilege requires the developer to build it.
- **B L1:** The framework neither narrows nor widens credentials; README examples combine an env-var model key with shell/python tools, so a hijack reaches whatever the host process can reach. — [README.md:81](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/README.md#L81); [README.md:36](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/README.md#L36) (inferred)
  - *To reach the next level:* Nothing in the framework confines tool authority to one system; L2 needs agent tools limited to one system or read-only reach.
- **Cap:** none

### C2 Approval gates — 0.23 (high)

Eino ships no approval gate. When the model emits a tool call, ToolsNode runs it immediately (in parallel with any other calls), including the built-in shell 'execute', 'write_file', and 'edit_file' tools. The framework does provide an interrupt/resume primitive that a tool can call to pause and wait for a human, and the paused call resumes with the same arguments, but each developer must write that check into each tool themselves; nothing is gated by default. A wrongly executed action has no framework-level undo.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** No approval mechanism exists in the default tool path; tool calls are dispatched straight to execution. — searched `rg -n -i approv --glob !*_test.go` in `adk compose flow components callbacks` → 0 hits (No approval gate, confirmation middleware, or approval flag exists in the framework's agent, graph, or tool packages.); [compose/tool_node.go:1106](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/compose/tool_node.go#L1106) (verified)
    - *To reach the next level:* No per-call human approval in the default tool path.
  - **C L0:** The most powerful built-in path, the execute tool, sends the model's command directly to the Shell with no gate. — [adk/middlewares/filesystem/filesystem.go:1028-1031](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/filesystem/filesystem.go#L1028-L1031); searched `rg -n -i approv --glob !*_test.go` in `adk compose flow components callbacks` → 0 hits (No approval gate, confirmation middleware, or approval flag exists in the framework's agent, graph, or tool packages.) (verified)
    - *To reach the next level:* The execute and write tools reach execution without crossing any gate.
  - **D L0:** No approval is on by default; any human-in-the-loop check must be coded into each tool via the opt-in interrupt primitive. — [components/tool/interrupt.go:44](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/components/tool/interrupt.go#L44); searched `rg -n -i approv --glob !*_test.go` in `adk compose flow components callbacks` → 0 hits (No approval gate, confirmation middleware, or approval flag exists in the framework's agent, graph, or tool packages.) (verified)
    - *To reach the next level:* Approval is not on by default for any tool.
  - **B L0:** Default-registered tools include shell execution and file writes with no checkpoint/undo of side effects; the README's DeepAgent example adds shell and python tools. — [adk/middlewares/filesystem/filesystem.go:1028-1031](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/filesystem/filesystem.go#L1028-L1031); [README.md:81](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/README.md#L81) (verified)
    - *To reach the next level:* No rollback, preview, or dry-run for consequential actions.
- **opt-in tool.Interrupt / interrupt-resume human-in-the-loop** (alt; raw 0.23, cap G1 → 0.23) ← counted
  - **S L2:** A tool can pause via tool.Interrupt with developer-chosen info and is resumed with its original persisted input, giving per-call approval, but the framework does not guarantee the approver sees the exact arguments and has no risk tiers. — [components/tool/interrupt.go:27-44](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/components/tool/interrupt.go#L27-L44); [compose/graph_call_options.go:53](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/compose/graph_call_options.go#L53) (verified)
    - *To reach the next level:* No framework-rendered exact call and no risk tiers deciding what needs a human.
  - **C L1:** Only tools whose authors call Interrupt are gated; built-in filesystem and execute tools never call it. — [adk/middlewares/filesystem/filesystem.go:1028-1031](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/filesystem/filesystem.go#L1028-L1031); searched `rg -n -i approv --glob !*_test.go` in `adk compose flow components callbacks` → 0 hits (No approval gate, confirmation middleware, or approval flag exists in the framework's agent, graph, or tool packages.) (verified)
    - *To reach the next level:* Built-in execute/write tools and sub-agent tools are not gated.
  - **D L0:** Entirely opt-in per tool. — [components/tool/interrupt.go:44](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/components/tool/interrupt.go#L44) (verified)
    - *To reach the next level:* Not on by default.
  - **B L0:** Same environment as the default: no undo or bounds on consequential actions. — [adk/middlewares/filesystem/filesystem.go:1028-1031](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/filesystem/filesystem.go#L1028-L1031) (verified)
    - *To reach the next level:* No rollback or bounded quantities for consequential actions.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C3 Tool & action scoping — 0.20 (high)

The framework's validation layer is thin. Tools built with its InferTool helper get their JSON arguments decoded into a Go struct, which rejects wrong types, and calls to tools the agent wasn't given are rejected by default; but schema constraints such as required fields, enums, or numeric bounds are not enforced, and tools implemented directly receive the raw argument string. The built-in 'execute' tool passes an arbitrary shell command straight through, and the filesystem middleware enables write, edit, and execute tools by default whenever a backend and shell are supplied. Jinja prompt templates have file-inclusion keywords disabled, a small but real hardening.

- **S L1:** Arguments are only type-decoded into Go structs (no schema/bounds enforcement), unknown tool names error by default, and the built-in execute tool is raw command passthrough. — [components/tool/utils/invokable_func.go:191](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/components/tool/utils/invokable_func.go#L191); [compose/tool_node.go:927-928](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/compose/tool_node.go#L927-L928); [adk/middlewares/filesystem/filesystem.go:1028-1031](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/filesystem/filesystem.go#L1028-L1031) (verified)
  - *To reach the next level:* No allowlist validation (path containment, URL/host allowlists, bounds) in any shared layer or built-in tool.
- **C L1:** Typed decoding applies only to InferTool-based tools; directly implemented tools receive raw argument strings, and there is no central policy layer. — [components/tool/utils/invokable_func.go:191](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/components/tool/utils/invokable_func.go#L191); [compose/tool_node.go:224](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/compose/tool_node.go#L224) (verified)
  - *To reach the next level:* Most built-in tools do not validate argument contents; the ToolArgumentsHandler hook is opt-in.
- **D L1:** With a Backend and Shell, the filesystem middleware registers write_file, edit_file, and execute by default; each can be individually disabled. — [adk/middlewares/filesystem/filesystem.go:505-527](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/filesystem/filesystem.go#L505-L527); [adk/middlewares/filesystem/filesystem.go:68-71](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/filesystem/filesystem.go#L68-L71) (verified)
  - *To reach the next level:* Tool groups are not selectable as a read-only default; write and exec are on unless individually disabled.
- **B L0:** A misused execute tool can run any command the developer's Shell permits; the framework adds no scope or quantity bounds. — [adk/middlewares/filesystem/filesystem.go:1028-1031](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/filesystem/filesystem.go#L1028-L1031); [README.md:81](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/README.md#L81) (verified)
  - *To reach the next level:* No workspace scoping or quantity bounds on built-in tools.
- **Cap:** none

### C4 Code-execution isolation — 0.00 (medium)

Eino exposes a first-class 'execute' tool that sends whatever command the model writes to a developer-supplied Shell implementation, with no isolation of its own: the core repo contains no sandbox, container, or process-separation code. Worse, the tool description the framework shows the model claims 'Commands run in an isolated sandbox environment', which is true only if the developer's Shell happens to provide one. Isolation is therefore entirely the integrator's responsibility, and with a local shell a hijacked agent runs commands as the host user.

- **S L0:** No isolation primitive: the execute tool forwards the raw command to the injected Shell interface. — [adk/middlewares/filesystem/filesystem.go:1028-1031](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/filesystem/filesystem.go#L1028-L1031); [adk/filesystem/backend.go:298-300](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/filesystem/backend.go#L298-L300); searched `rg -n -i 'sandbox|seccomp|chroot|landlock' --glob !*_test.go --type go` in `.` → 3 hits (All three hits are prompt text (deep/prompt.go:38, filesystem/prompt.go:186,206); no isolation code exists.) (verified)
  - *To reach the next level:* No isolation boundary (container, OS sandbox, or capability runtime) provided or required by the framework.
- **C L0:** No execution path is sandboxed by the framework. — [adk/middlewares/filesystem/filesystem.go:1028-1031](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/filesystem/filesystem.go#L1028-L1031); searched `rg -n '"os/exec"|exec\.Command|syscall\.Exec|plugin\.Open' --glob !*_test.go` in `.` → 0 hits (The core repo never spawns processes itself; execution is delegated to the developer-supplied Shell.) (verified)
  - *To reach the next level:* No execution path goes through a framework-enforced sandbox.
- **D L0:** No sandbox exists to be on by default; the default tool description asserts sandboxing that the framework does not enforce. — [adk/middlewares/filesystem/prompt.go:206](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/filesystem/prompt.go#L206); searched `rg -n -i 'sandbox|seccomp|chroot|landlock' --glob !*_test.go --type go` in `.` → 3 hits (All three hits are prompt text (deep/prompt.go:38, filesystem/prompt.go:186,206); no isolation code exists.) (verified)
  - *To reach the next level:* No sandbox on by default.
- **B L0:** With a host Shell, model commands run as the host process user with its environment and network; the framework imposes no mount, network, or resource limits. — [adk/middlewares/filesystem/filesystem.go:1028-1031](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/filesystem/filesystem.go#L1028-L1031); [README.md:81](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/README.md#L81) (inferred)
  - *To reach the next level:* No workspace-only mount, egress restriction, or secret-free environment enforced.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

Tool results, including web or file content, enter the conversation as ordinary tool messages with no provenance tag, and nothing in the framework changes what the agent may do after it has read untrusted content. Content loaded by the AGENTS.md middleware is injected as a user message, giving file content the same standing as the principal's own request. Because the documented DeepAgent setup combines web search, shell, and Python tools in one session with no gate, a successful prompt injection can both exfiltrate data and take irreversible actions unattended.

- **S L0:** No structural limit on a hijacked agent; tool output is appended to context and the loop continues with all tools available. — [compose/tool_node.go:1240](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/compose/tool_node.go#L1240); searched `rg -n -i 'untrusted|provenance|taint' --glob !*_test.go --type go` in `adk compose flow components` → 2 hits (Hits are DeepAgent prompt prose ('truthfulness', deep/prompt.go:73) and a doc comment asking Document loaders to keep a source URI (components/document/interface.go:42); nothing tags tool results or acts on provenance.) (verified)
  - *To reach the next level:* No approval or capability restriction triggered by reading untrusted content.
- **C L0:** Untrusted sources are not distinguished; AGENTS.md content is even injected with user-message standing. — [adk/middlewares/agentsmd/agentsmd.go:173](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/agentsmd/agentsmd.go#L173); [compose/tool_node.go:1240](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/compose/tool_node.go#L1240) (verified)
  - *To reach the next level:* No untrusted source is handled differently from principal input.
- **D L0:** No control exists to be on by default. — searched `rg -n -i 'untrusted|provenance|taint' --glob !*_test.go --type go` in `adk compose flow components` → 2 hits (Hits are DeepAgent prompt prose ('truthfulness', deep/prompt.go:73) and a doc comment asking Document loaders to keep a source URI (components/document/interface.go:42); nothing tags tool results or acts on provenance.) (verified)
  - *To reach the next level:* No default-on mitigation.
- **B L0:** The README's DeepAgent example combines web search (untrusted input) with shell and python (egress and irreversible actions) in one ungated session. — [README.md:81](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/README.md#L81); [adk/middlewares/filesystem/filesystem.go:1028-1031](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/filesystem/filesystem.go#L1028-L1031) (verified)
  - *To reach the next level:* No leg of the Rule of Two is removed or gated by default.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.10 (medium)

Core Eino has no long-term memory module, but it ships two auto-load paths: the AGENTS.md middleware silently injects configured instruction files (and anything they @import) into every run as a user message, and the skill middleware rescans a skills directory and offers every SKILL.md it finds. Neither validates content, and import resolution in the AGENTS.md loader is not confined. If these point into the same backend that the agent's write_file tool can modify, an injection can write itself into instructions that load in every later session; nothing in the framework prevents that loop.

- **S L0:** Instruction files are loaded with no validation and the agent's own write_file tool can modify them on a shared backend, so model-written text is re-injected as user-standing context. — [adk/middlewares/agentsmd/agentsmd.go:100](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/agentsmd/agentsmd.go#L100) (verified)
  - *To reach the next level:* No write gating or provenance on instruction files; nothing stops the agent writing files that are auto-loaded later.
- **C L0:** Neither AGENTS.md loading, skill scanning, nor checkpoint restore is controlled. — [adk/middlewares/skill/filesystem_backend.go:47-48](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/skill/filesystem_backend.go#L47-L48) (verified)
  - *To reach the next level:* No auto-loaded file or store is controlled.
- **D L1:** Runner state is per run and checkpoints are keyed by developer-chosen IDs, but AGENTS.md and skills come from one backend path shared by every user of the agent. — [adk/runner.go:72](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/runner.go#L72); [internal/core/interrupt.go:31-34](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/internal/core/interrupt.go#L31-L34) (verified)
  - *To reach the next level:* No per-user/session namespace enforced on instruction or skill stores.
- **B L1:** Poisoned instruction files persist for the backend's lifetime and can drive tool use in later sessions; the only in-repo backend is process-lifetime in-memory. — [adk/middlewares/agentsmd/agentsmd.go:173](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/agentsmd/agentsmd.go#L173) (inferred)
  - *To reach the next level:* Poisoned instructions can trigger tool use; L2 needs persistence limited to influencing text or gated actions.
- **Cap:** none

### C7 Third-party extensions — 0.07 (high)

Eino's skill middleware is its extension mechanism: it rescans a directory for skill folders, offers each one to the model, and its system prompt tells the model that skills may contain Python scripts or other executables and to run them with absolute paths. Skills are not pinned, hashed, or individually approved; dropping a folder into the skills directory enables it on the next run. Any script a skill ships runs through the same unsandboxed execute tool with the agent's full authority. The core repo has no MCP client or plugin loader.

- **S L1:** Skills come from a developer-chosen directory, unpinned and unverified, rescanned at use. — [adk/middlewares/skill/filesystem_backend.go:66](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/skill/filesystem_backend.go#L66); searched `rg -n -i 'sha256|checksum|signature|verify' --glob !*_test.go --type go` in `adk/middlewares/skill` → 0 hits (Skills are loaded with no integrity or pinning check.) (verified)
  - *To reach the next level:* No version pinning or integrity check of skill packages.
- **C L0:** No extension type is verified. — searched `rg -n -i 'sha256|checksum|signature|verify' --glob !*_test.go --type go` in `adk/middlewares/skill` → 0 hits (Skills are loaded with no integrity or pinning check.) (verified)
  - *To reach the next level:* No extension type verified.
- **D L0:** Any SKILL.md folder placed in the skills directory, including one inside an agent-writable backend, is enabled automatically without a consent step. — [adk/middlewares/skill/filesystem_backend.go:47-48](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/skill/filesystem_backend.go#L47-L48) (verified)
  - *To reach the next level:* No explicit install or consent step for new skills.
- **B L0:** The framework instructs the model to execute skill scripts, which run via the unsandboxed execute tool with the agent's full environment. — [adk/middlewares/skill/prompt.go:38](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/skill/prompt.go#L38); [adk/middlewares/filesystem/filesystem.go:1028-1031](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/filesystem/filesystem.go#L1028-L1031) (verified)
  - *To reach the next level:* No per-extension process separation or environment scrubbing.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.23 (medium)

The core framework stores no credentials, sends no telemetry, and only logs error values in a few retry/failover paths, so it adds little exposure of its own. But it also has no secret-handling primitives: there is no redaction before callbacks, logs, checkpoints, or model-bound messages, and tool output (including anything a shell tool prints) flows to the model provider unfiltered. Import resolution in the AGENTS.md loader is also not confined, which affects what reaches model context.

- **S L1:** No secrets in committed config or defaults, but no masking or redaction anywhere; credentials are expected from env vars per the README. — searched `rg -n -i 'redact|sanitiz|mask' --glob !*_test.go --type go` in `.` → 2 hits (Hits are a bitmask comment (turn_loop.go:1045) and an example handler name in a comment (chatmodel.go:375); no redaction code.); [README.md:36](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/README.md#L36) (verified)
  - *To reach the next level:* No type-level masking or log/transcript redaction.
- **C L0:** No path (logs, callbacks, checkpoints, model-bound tool results) is protected. — searched `rg -n -i 'redact|sanitiz|mask' --glob !*_test.go --type go` in `.` → 2 hits (Hits are a bitmask comment (turn_loop.go:1045) and an example handler name in a comment (chatmodel.go:375); no redaction code.); [adk/failover_chatmodel.go:301](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/failover_chatmodel.go#L301) (verified)
  - *To reach the next level:* No path redacted.
- **D L2:** No telemetry ships; framework logging is limited to error strings and callback handlers are opt-in. — [internal/callbacks/manager.go:30](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/internal/callbacks/manager.go#L30); searched `rg -n -i 'redact|sanitiz|mask' --glob !*_test.go --type go` in `.` → 2 hits (Hits are a bitmask comment (turn_loop.go:1045) and an example handler name in a comment (chatmodel.go:375); no redaction code.) (verified)
  - *To reach the next level:* Redaction does not exist to be always on.
- **B L1:** Long-lived provider keys from the environment sit in the same process as every tool, and shell tools inherit that environment. — [README.md:36](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/README.md#L36); [adk/middlewares/filesystem/filesystem.go:1028-1031](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/middlewares/filesystem/filesystem.go#L1028-L1031) (inferred)
  - *To reach the next level:* Keys are not scoped or short-lived.
- **Cap:** none

### C9 Audit & traceability — 0.30 (high)

By default Eino records nothing: the Runner emits a stream of agent events (model messages, tool results, agent name and run path) to the calling application, which may log or discard them, and sub-agent internal events are not forwarded unless enabled. A callback system can observe the start, end, and errors of every component including tools and is propagated to sub-agents, but it is opt-in, records no approvals, and handler failures are silent. Any durable audit trail must be built by the integrator.

- **default configuration** (default; raw 0.15 → 0.15)
  - **S L1:** The default event stream carries tool calls and results with agent name/run path but is transient and timestamp-free. — [adk/interface.go:419-435](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/interface.go#L419-L435) (verified)
    - *To reach the next level:* No structured, persisted record of every tool call with timestamps.
  - **C L1:** Events cover the top-level agent; nested agent-tool events are not emitted unless EmitInternalEvents is set. — [adk/chatmodel.go:143-155](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/chatmodel.go#L143-L155) (verified)
    - *To reach the next level:* Sub-agent tool calls are not in the default stream.
  - **D L0:** Nothing is persisted by default; the application must consume and store events. — [internal/callbacks/manager.go:30](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/internal/callbacks/manager.go#L30) (verified)
    - *To reach the next level:* No default-on record.
  - **B L0:** Events are lost if the caller does not persist them; actions proceed regardless. — [adk/interface.go:419-435](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/interface.go#L419-L435) (verified)
    - *To reach the next level:* No per-action durable record.
- **opt-in callbacks (AppendGlobalHandlers)** (alt; raw 0.30, cap G1 → 0.30) ← counted
  - **S L2:** Callback handlers receive structured OnStart/OnEnd/OnError for every component including tools, with RunInfo naming the component. — [internal/callbacks/interface.go:39](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/internal/callbacks/interface.go#L39) (verified)
    - *To reach the next level:* No built-in actor/approver attribution or correlation IDs across sub-agents.
  - **C L2:** Global handlers apply to all tools and propagate through sub-agents via context, but approvals and denials are not recorded events. — [callbacks/interface.go:103](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/callbacks/interface.go#L103) (verified)
    - *To reach the next level:* Approvals and denials are not recorded.
  - **D L0:** Opt-in only. — [internal/callbacks/manager.go:30](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/internal/callbacks/manager.go#L30) (verified)
    - *To reach the next level:* Not on by default.
  - **B L0:** Handler methods return no error, so logging failures are silent and execution proceeds. — [internal/callbacks/interface.go:39](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/internal/callbacks/interface.go#L39) (verified)
    - *To reach the next level:* Logging failures are not surfaced.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C10 Limits & kill switch — 0.25 (high)

ChatModelAgent stops after 20 model calls by default, which bounds a single agent loop, and a cancel API can stop a run at safe points or immediately. There is no wall-clock limit, no per-tool timeout, and no token or cost budget; tool calls in one turn run in parallel without a concurrency cap; and each sub-agent or agent-tool starts with its own fresh 20-iteration budget. The LoopAgent workflow defaults to unlimited iterations. Immediate cancel stops the graph waiting, but Go tool goroutines that ignore context cancellation keep running.

- **S L1:** Iteration cap (default 20) enforced in code; no wall-clock, per-tool timeout, or token/cost cap. — [adk/react.go:345](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/react.go#L345); searched `rg -n -i 'WithTimeout|WithDeadline' --glob !*_test.go --type go` in `adk compose` → 2 hits (Both hits are receiveWithDeadline in compose/graph_manager.go (484, 501), which only bounds how long an interrupt waits for running tasks; no per-tool-call timeout or run wall-clock limit exists.) (verified)
  - *To reach the next level:* No wall-clock, per-execution timeout, or token/cost cap.
- **C L1:** The cap applies per agent loop; sub-agents get fresh budgets and tool calls have no timeout. — [adk/prebuilt/deep/task_tool.go:96](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/prebuilt/deep/task_tool.go#L96); [compose/tool_node.go:1106](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/compose/tool_node.go#L1106) (verified)
  - *To reach the next level:* No tool timeouts; sub-agents do not count against the parent's budget.
- **D L1:** ChatModelAgent defaults to 20, but LoopAgent defaults to unlimited and delegation resets the budget. — [adk/workflow.go:353](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/workflow.go#L353); [adk/chatmodel.go:305-308](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/chatmodel.go#L305-L308) (verified)
  - *To reach the next level:* LoopAgent has no default ceiling and delegation resets limits.
- **B L1:** No time or spend ceiling; immediate cancel abandons running tool tasks, which re-run on resume, rather than killing them. — [compose/graph_call_options.go:53](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/compose/graph_call_options.go#L53); [adk/cancel.go:47-55](https://github.com/cloudwego/eino/blob/58d184303f4e73413c454a9626f79c17af0fd9fb/adk/cancel.go#L47-L55) (verified)
  - *To reach the next level:* Stopping does not guarantee in-flight tool work stops; no time or cost ceilings.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Tool results (e.g. web search, file reads) appended as tool messages, compose/tool_node.go:1240 · [B] sensitive data/systems: Host-process credentials and files reachable by in-process tools and the Shell, README.md:36, adk/middlewares/filesystem/filesystem.go:1029 · [C] state change / egress: execute/write_file/edit_file tools run ungated, adk/middlewares/filesystem/filesystem.go:505-527 · Same default session? Yes

## Highest-impact improvements
1. Ship a default-on approval ToolCallMiddleware in the filesystem middleware and DeepAgent that interrupts execute/write_file/edit_file and shows the exact arguments, reusing tool.Interrupt. — C2 S L0→L3, +0.225 before caps (Playbook 5)
2. Remove the 'isolated sandbox' claim from ExecuteToolDesc and require Shell implementations to declare an isolation level, refusing an unsandboxed Shell unless the developer sets an explicit unsafe flag. — C4 D L0→L2, +0.100 before caps (Playbook 3)
3. Add default per-tool-call timeouts and a run wall-clock limit, and charge sub-agent iterations against the parent's budget. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)
4. Harden AGENTS.md import resolution and keep instruction/skill directories outside the agent-writable backend by default. — C6 S L0→L1, +0.075 before caps (Playbook 2)
5. Give LoopAgent a finite default MaxIterations instead of unlimited. — C10 D L1→L2, +0.050 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Only the core cloudwego/eino repository was reviewed. Concrete ChatModel clients, Shell/Backend implementations (e.g. local shell or sandbox backends), MCP tools, and callback handlers live in cloudwego/eino-ext and were not examined; any isolation or redaction they provide is not credited here.
- Blast-radius ratings for C1, C4, C6, and C8 are inferred from the framework's lack of containment and the README's documented tool combinations, not from a specific deployed configuration.
- No reviewer-steering or prompt-injection text aimed at auditors was found in the repository.
