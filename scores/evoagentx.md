# Defense-in-Depth Score: EvoAgentX

**Repo:** https://github.com/anative-lab/evoagentx · **Commit:** `d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54` (0.1.4) · **Reviewed:** 2026-10-04
**What it is:** Python framework for building, running and self-evolving multi-agent workflows with tools, memory and prompt/workflow optimizers.
**Category:** Agent Frameworks
**Scored configuration:** Library defaults: CustomizeAgent/WorkFlow with default constructor arguments and bundled toolkits as shipped (no HITLManager, PythonInterpreterToolkit without allowed_imports).
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents yes · external communication yes

## Score: 1.8 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L1 | L1 | L0 | L0 | 0.15 | G1 | **0.15** (alt) | High |
| C3 | Tool & action scoping | L0 | L1 | L1 | L0 | 0.12 | — | **0.12** | High |
| C4 | Code-execution isolation | L2 | L1 | L0 | L2 | 0.33 | G1 | **0.33** (alt) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | C6-REPOCONFIG | **0.10** | High |
| C7 | Third-party extensions | L1 | L0 | L2 | L1 | 0.23 | — | **0.23** | Medium |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L0 | L0 | 0.15 | — | **0.15** | High |
| C9 | Audit & traceability | L1 | L2 | L1 | L1 | 0.33 | — | **0.33** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | — | **0.40** | High |


EvoAgentX gives agents powerful bundled tools (shell, in-process Python, arbitrary HTTP, file I/O, email) but the framework itself adds almost no safety layer: tool calls run without approval or argument validation, and model-written Python executes inside the agent process alongside its API keys. A prompt injection from any web page, search result or email can leak secrets and take irreversible actions unattended. The opt-in Docker interpreter and human-in-the-loop manager are thin and off by default.

## Critical gaps
- Tools run with the developer's ambient OS and API-key authority; a hijack reaches the whole machine and connected accounts. (ASI03, T3; C1) — [evoagentx/models/litellm_model.py:45](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/models/litellm_model.py#L45); [evoagentx/tools/cmd_toolkit.py:200-212](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/cmd_toolkit.py#L200-L212)
- Default Python execution is in-process exec() with full builtins, sharing the process that holds API keys. (ASI05, T11; C4) — [evoagentx/tools/interpreter_python.py:324-335](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/interpreter_python.py#L324-L335); [evoagentx/models/litellm_model.py:45](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/models/litellm_model.py#L45)
- Untrusted tool output enters context as user messages while unguarded egress and state-changing tools are available: leak plus irreversible action unattended. (ASI01, LLM01, T6; C5) — [evoagentx/memory/context_manager.py:104-117](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/memory/context_manager.py#L104-L117); [evoagentx/tools/request.py:10-19](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/request.py#L10-L19); [evoagentx/tools/gmail_tools.py:1409](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/gmail_tools.py#L1409)
- load_dotenv() at import reads .env from the working directory, so a workspace file can set credentials and model endpoints. (ASI06, ASI04, T1; C6) — [evoagentx/tools/search_google.py:8](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/search_google.py#L8); [evoagentx/rag/embeddings/openai_embedding.py:44](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/rag/embeddings/openai_embedding.py#L44)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

EvoAgentX has no identity or authorization layer of its own. Each bundled toolkit builds its own client from environment variables (Gmail, Telegram, databases, search APIs), and the model wrapper copies provider API keys into the process environment, which the shell tool and the in-process Python executor then inherit in full. Every tool runs with whatever the operating-system user and those keys can do. A hijacked agent therefore has the developer's whole account reach.

- **S L0:** Tools use ambient OS-user authority and long-lived API keys read from the environment; no scoped or per-request identity exists. — [evoagentx/models/litellm_model.py:45](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/models/litellm_model.py#L45); [evoagentx/tools/telegram_tools.py:61-63](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/telegram_tools.py#L61-L63) (verified)
  - *To reach the next level:* Introduce a dedicated, role-scoped credential per agent or tool instead of ambient env keys.
- **C L0:** Tools construct their own privileged clients and the shell subprocess is started without an env= argument, inheriting the full environment. — [evoagentx/tools/cmd_toolkit.py:200-212](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/cmd_toolkit.py#L200-L212); searched `rg -n 'env='` in `evoagentx/tools` → 0 hits (no subprocess in the tools package passes an explicit environment) (verified)
  - *To reach the next level:* Route every tool through one authorization layer and pass subprocesses a scrubbed environment.
- **D L0:** The default is the developer's full ambient authority; narrowing requires the developer to write their own wrappers. — [evoagentx/models/litellm_model.py:45](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/models/litellm_model.py#L45) (verified)
  - *To reach the next level:* Ship a narrower default identity (e.g. read-only credentials) that write tools must explicitly elevate.
- **B L0:** With shell and Python execution as the OS user plus provider, mail and messaging keys, a hijack reaches the user's whole machine and connected accounts. — [evoagentx/tools/cmd_toolkit.py:200-212](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/cmd_toolkit.py#L200-L212); [evoagentx/tools/interpreter_python.py:324-335](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/interpreter_python.py#L324-L335); [evoagentx/tools/gmail_tools.py:1409](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/gmail_tools.py#L1409) (verified)
  - *To reach the next level:* Bound credentials to one system or tenant and scrub them from execution paths.
- **Cap:** none

### C2 Approval gates — 0.15 (high)

The framework's tool executor runs every tool call the model makes with no approval step. A human-in-the-loop manager exists, but it is off unless a developer creates and activates it (otherwise it auto-approves), it approves whole workflow actions rather than individual tool calls, and its tool-call review mode is unimplemented. The bundled shell toolkit does prompt per command and shows the exact command, but Python execution, file writes, HTTP requests and email sending have no gate.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** The default executor looks up the tool by name and calls it directly; there is no approval of any kind on the framework's tool path. — [evoagentx/actions/customize_action.py:263-280](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/actions/customize_action.py#L263-L280); searched `rg -n -S 'approv|confirm|permission'` in `evoagentx/actions/customize_action.py` → 1 hits (single hit is a comment about asking the model to confirm no tool is needed; the executor has no approval check) (verified)
    - *To reach the next level:* Add a per-call approval that shows the exact tool name and arguments before execution.
  - **C L0:** The most powerful bundled paths (python_execute, http_request, write_file, Gmail send) reach execution without crossing any gate; only CMDToolkit asks per command inside its own code. — [evoagentx/actions/customize_action.py:263-280](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/actions/customize_action.py#L263-L280); [evoagentx/tools/cmd_toolkit.py:116-148](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/cmd_toolkit.py#L116-L148) (verified)
    - *To reach the next level:* Make the gate part of the shared executor so every registered tool, including MCP and generated tools, traverses it.
  - **D L0:** No gate is on by default; the HITL manager defaults to inactive and auto-approves. — [evoagentx/hitl/approval_manager.py:17-61](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/hitl/approval_manager.py#L17-L61) (verified)
    - *To reach the next level:* Enable approval for consequential tools by default.
  - **B L0:** Bypassed actions include arbitrary shell/Python, arbitrary HTTP methods and sending email, all irreversible with no undo. — [evoagentx/tools/request.py:10-19](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/request.py#L10-L19); [evoagentx/tools/gmail_tools.py:1409](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/gmail_tools.py#L1409); [evoagentx/tools/interpreter_python.py:324-335](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/interpreter_python.py#L324-L335) (verified)
    - *To reach the next level:* Add checkpoints, previews or rate limits on consequential actions.
- **opt-in HITLManager workflow interceptor** (alt; raw 0.15, cap G1 → 0.15) ← counted
  - **S L1:** When activated, the HITL interceptor asks for approve/reject of a whole agent action from its inputs before it runs; the later tool calls inside that action are not shown, and tool-call review raises NotImplementedError. — [evoagentx/hitl/approval_manager.py:208-211](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/hitl/approval_manager.py#L208-L211); [evoagentx/hitl/interceptor_agent.py:55-63](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/hitl/interceptor_agent.py#L55-L63) (verified)
    - *To reach the next level:* Approve each tool call with its exact arguments.
  - **C L1:** Only workflow nodes the developer wraps with an interceptor agent are covered; standalone agents and tool calls inside approved actions are not. — [evoagentx/workflow/workflow.py:163-165](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/workflow/workflow.py#L163-L165) (verified)
    - *To reach the next level:* Apply the gate to every tool path, including standalone CustomizeAgent runs.
  - **D L0:** Opt-in: requires constructing HITLManager, calling activate(), and inserting interceptor agents. — [evoagentx/hitl/approval_manager.py:17-61](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/hitl/approval_manager.py#L17-L61) (verified)
    - *To reach the next level:* Turn it on by default.
  - **B L0:** Same irreversible actions remain reachable once an action is approved. — [evoagentx/tools/request.py:10-19](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/request.py#L10-L19); [evoagentx/tools/gmail_tools.py:1409](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/gmail_tools.py#L1409) (verified)
    - *To reach the next level:* Add undo/preview/rate limits for consequential actions.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C3 Tool & action scoping — 0.12 (high)

Tool arguments are not validated at run time: the base Tool class only checks schema declarations when a tool class is defined, and the executor passes model arguments straight into the tool. Bundled tools are general-purpose: a raw shell string, arbitrary Python, any URL with any HTTP method, and file reads/writes on any absolute path. The storage helper's base-directory containment is not a complete boundary. A dynamic toolkit even lets the model create new code-backed tools at run time.

- **S L0:** No framework validation layer; bundled tools accept raw shell, raw URLs and raw paths. — [evoagentx/actions/customize_action.py:263-280](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/actions/customize_action.py#L263-L280); [evoagentx/tools/request.py:10-19](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/request.py#L10-L19); [evoagentx/tools/file_tool.py:264-276](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/file_tool.py#L264-L276); searched `rg -n -S 'allowed_hosts|allowed_domains|169\.254|is_relative_to|realpath'` in `evoagentx` → 0 hits (verified)
  - *To reach the next level:* Validate arguments in code against allowlists (resolved-path containment, host allowlists, bounds).
- **C L1:** A few bundled tools validate weakly (storage handler base-path handling; CMD danger regex that only labels the prompt); most do not. — [evoagentx/tools/cmd_toolkit.py:37-47](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/cmd_toolkit.py#L37-L47) (verified)
  - *To reach the next level:* Validate in a central layer every tool inherits.
- **D L1:** CustomizeAgent starts with no tools, but toolkits bundle read with write/exec (FileToolkit read+write+append) and the Alita toolkit lets the model add tools. — [evoagentx/tools/file_tool.py:410-420](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/file_tool.py#L410-L420); [evoagentx/tools/alita_agent.py:189-195](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/alita_agent.py#L189-L195) (verified)
  - *To reach the next level:* Provide read-only toolkits by default and stop the model from creating tools on its own.
- **B L0:** A misused tool can run any command, reach any host or write any path on the machine. — [evoagentx/tools/cmd_toolkit.py:200-212](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/cmd_toolkit.py#L200-L212); [evoagentx/tools/request.py:10-19](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/request.py#L10-L19) (verified)
  - *To reach the next level:* Scope tools to a workspace and bound quantities.
- **Cap:** none

### C4 Code-execution isolation — 0.33 (high)

Model-written Python runs by default in the agent's own process through exec() with full builtins; the import allowlist is skipped unless the developer supplies one, and the code's own docstring warns it is unsafe for untrusted code. Shell commands run on the host as the user. Other host-side execution paths exist, including workflow operator exec(), and not every model-influenced execution path is confined. An opt-in Docker interpreter exists, but it starts a stock container with no resource limits, no network restriction and no execution timeout.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Default Python executor is in-process exec() with full builtins; shell is a same-user subprocess. — [evoagentx/tools/interpreter_python.py:324-335](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/interpreter_python.py#L324-L335); [evoagentx/tools/cmd_toolkit.py:200-212](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/cmd_toolkit.py#L200-L212); [evoagentx/tools/interpreter_python.py:293-295](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/interpreter_python.py#L293-L295) (verified)
    - *To reach the next level:* Run model code in an OS-level or kernel-separated sandbox by default.
  - **C L0:** No execution path is sandboxed by default, including workflow operator exec(). — [evoagentx/workflow/operators.py:389](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/workflow/operators.py#L389) (verified)
    - *To reach the next level:* Route every model-reachable execution path through a sandbox.
  - **D L0:** Isolation is off by default; the import allowlist defaults to empty, which disables the check entirely. — [evoagentx/tools/interpreter_python.py:50](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/interpreter_python.py#L50); [evoagentx/tools/interpreter_python.py:324](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/interpreter_python.py#L324) (verified)
    - *To reach the next level:* Make sandboxed execution the default.
  - **B L0:** Executed code shares the agent process, which holds provider API keys in os.environ and full network and filesystem access. — [evoagentx/tools/interpreter_python.py:324-335](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/interpreter_python.py#L324-L335); [evoagentx/models/litellm_model.py:45](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/models/litellm_model.py#L45) (verified)
    - *To reach the next level:* Keep secrets and the host filesystem out of the execution environment.
- **opt-in DockerInterpreterToolkit** (alt; raw 0.33, cap G1 → 0.33) ← counted
  - **S L2:** Stock docker containers.run with only image, command and working_dir; no user, capability, seccomp or read-only settings. — [evoagentx/tools/interpreter_docker.py:159-164](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/interpreter_docker.py#L159-L164) (verified)
    - *To reach the next level:* Harden the container (non-root, drop caps, no-new-privileges, seccomp, read-only rootfs).
  - **C L1:** Covers only docker_execute tools; CMDToolkit, operators exec and other host-side paths still run on host. — [evoagentx/tools/interpreter_docker.py:234](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/interpreter_docker.py#L234); [evoagentx/tools/cmd_toolkit.py:200-212](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/cmd_toolkit.py#L200-L212) (verified)
    - *To reach the next level:* Sandbox every model-reachable execution path.
  - **D L0:** Opt-in; the developer must choose the Docker toolkit and supply an image; confirmation defaults off. — [evoagentx/tools/interpreter_docker.py:29](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/interpreter_docker.py#L29) (verified)
    - *To reach the next level:* Default to the sandboxed interpreter.
  - **B L2:** Host files are copied in rather than mounted and no env is passed, but the container has default network egress and no CPU/memory/PID limits. — [evoagentx/tools/interpreter_docker.py:159-164](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/interpreter_docker.py#L159-L164) (verified)
    - *To reach the next level:* Disable or allowlist network and add resource limits.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.00 (high)

Nothing in the framework limits what a hijacked agent can do after reading untrusted content. Search, browser, crawler, RSS, arXiv and Gmail tools pull external text into context, and in the default prompt mode tool results are appended as user-role messages with the same standing as the developer's instructions. The same agent can hold egress and state-changing tools (arbitrary HTTP, email sending, shell, Python) with no approval in the way. A successful prompt injection can leak keys or data and take irreversible actions unattended.

- **S L0:** No taint tracking, quarantine or approval tied to untrusted content. — searched `rg -n -S 'untrusted|prompt.injection|provenance|taint'` in `evoagentx/tools` → 2 hits (both hits are docstring warnings in interpreter_python.py, not controls) (verified)
  - *To reach the next level:* Disable or gate egress and state-changing tools once untrusted content enters the session.
- **C L0:** Tool results enter context as user messages in default mode; no source is distinguished. — [evoagentx/memory/context_manager.py:104-117](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/memory/context_manager.py#L104-L117) (verified)
  - *To reach the next level:* Mark tool results and MCP descriptions as untrusted data.
- **D L0:** No control exists to be on by default. — [evoagentx/memory/context_manager.py:104-117](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/memory/context_manager.py#L104-L117) (verified)
  - *To reach the next level:* Ship an on-by-default untrusted-content policy.
- **B L0:** A hijacked agent can exfiltrate via http_request and send email or run commands, with no human involved. — [evoagentx/tools/request.py:10-19](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/request.py#L10-L19); [evoagentx/tools/gmail_tools.py:1409](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/gmail_tools.py#L1409); [evoagentx/tools/cmd_toolkit.py:200-212](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/cmd_toolkit.py#L200-L212) (verified)
  - *To reach the next level:* Ensure sessions that read untrusted content cannot both reach secrets and egress unattended.
- **Cap:** C5-WORSTCASE — Default worst case is data leak plus irreversible action with no human involved.

### C6 Memory, context & configuration integrity — 0.10 (high)

Importing the tools package calls load_dotenv() in several modules, which reads a .env file from the current working directory and can silently set API keys and endpoints such as OPENAI_API_BASE. If the agent runs inside an untrusted checkout, that file can redirect model traffic or swap credentials. The opt-in long-term memory agent stores conversation messages, including model outputs, and pastes retrieved memories back into the prompt as context with no validation or provenance.

- **S L0:** A workspace .env is auto-loaded at import with no trust decision, and memory entries are re-injected as plain prompt context. — [evoagentx/tools/search_google.py:8](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/search_google.py#L8); [evoagentx/tools/api_pipeline.py:15](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/api_pipeline.py#L15); [evoagentx/rag/embeddings/openai_embedding.py:44](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/rag/embeddings/openai_embedding.py#L44) (verified)
  - *To reach the next level:* Load configuration only from user scope and present memories as provenance-tagged data.
- **C L0:** Neither the dotenv path nor the memory store is controlled. — [evoagentx/agents/long_term_memory_agent.py:432](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/agents/long_term_memory_agent.py#L432) (verified)
  - *To reach the next level:* Control every memory store and auto-loaded file.
- **D L1:** LongTermMemory uses a per-instance random corpus_id that queries filter on, but it is not per user, and the control around it is weak. — [evoagentx/memory/long_term_memory.py:33](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/memory/long_term_memory.py#L33) (verified)
  - *To reach the next level:* Enforce per-user namespaces the model cannot change.
- **B L1:** Saved memory persists across the user's sessions and feeds a model that can call tools; a poisoned .env affects every run from that directory. — [evoagentx/memory/memory_manager.py:58](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/memory/memory_manager.py#L58) (verified)
  - *To reach the next level:* Make poisoned context session-scoped or easily inspected and purged.
- **Cap:** C6-REPOCONFIG — load_dotenv() at import reads .env from the working directory, letting workspace files set credentials and model endpoints without a trust decision.

### C7 Third-party extensions — 0.23 (medium)

Third-party extensions come in through MCP: the developer supplies a server config and the framework launches or connects to it via FastMCP with no version pinning, hash check or change detection. Tool descriptions from servers are passed to the model verbatim. Stdio servers run as the same OS user. Separately, importing the RAG embeddings module tries to pip-install the ollama package, unpinned and without a prompt, if it is missing.

- **S L1:** MCP servers are developer-chosen but unpinned and unverified. — [evoagentx/tools/mcp.py:221](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/mcp.py#L221); searched `rg -n 'sha256|hash_check|verify_signature|pinned'` in `evoagentx` → 2 hits (both are content hashes for memory/RAG dedup, not extension verification) (verified)
  - *To reach the next level:* Pin versions and verify integrity of extensions.
- **C L0:** No extension type is verified; runtime pip install of ollama also unverified. — [evoagentx/rag/embeddings/ollama_embedding.py:15](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/rag/embeddings/ollama_embedding.py#L15) (verified)
  - *To reach the next level:* Verify every extension type.
- **D L2:** MCP servers load only from an explicit config or config_path argument, but nothing shows what will run. — [evoagentx/tools/mcp.py:298-337](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/mcp.py#L298-L337) (verified)
  - *To reach the next level:* Show the exact command/package before enabling a server.
- **B L1:** Stdio MCP servers are separate processes running as the same user; env scrubbing depends on FastMCP/MCP SDK defaults (inferred), not on this code. — [evoagentx/tools/mcp.py:219-224](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/mcp.py#L219-L224) (inferred)
  - *To reach the next level:* Sandbox each extension with its own scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.15 (high)

Provider and tool API keys come from environment variables or plain config fields, and the LiteLLM wrapper copies them into os.environ, where every shell command and in-process code execution can read them. There is no redaction anywhere: tool arguments, tool results and raw model responses are logged at INFO to stdout by default. The one protective detail is that saved agent files exclude the LLM config. No telemetry was found.

- **S L1:** Secrets come from env vars; agent serialization excludes llm_config, but no masking or redaction exists. — [evoagentx/agents/agent.py:63](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/agents/agent.py#L63); searched `rg -n -S 'redact|SecretStr'` in `evoagentx` → 0 hits (verified)
  - *To reach the next level:* Use masked secret types and log/model-bound redaction.
- **C L1:** Only the saved-agent path omits keys; logs, subprocess env and model-bound messages are unprotected. — [evoagentx/agents/agent.py:63](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/agents/agent.py#L63) (verified)
  - *To reach the next level:* Protect logs, transcripts and subprocess environments too.
- **D L0:** Verbose payload logging (tool args, results, raw LLM output) is on by default at INFO. — [evoagentx/actions/customize_action.py:275](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/actions/customize_action.py#L275); [evoagentx/actions/customize_action.py:375](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/actions/customize_action.py#L375); [evoagentx/core/logging.py:13](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/core/logging.py#L13) (verified)
  - *To reach the next level:* Default to content-free logs with redaction.
- **B L0:** Long-lived provider keys sit in os.environ, reachable by every subprocess and by model-written Python. — [evoagentx/models/litellm_model.py:45](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/models/litellm_model.py#L45); [evoagentx/tools/cmd_toolkit.py:200-212](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/cmd_toolkit.py#L200-L212) (verified)
  - *To reach the next level:* Keep keys out of execution environments and use scoped, short-lived tokens.
- **Cap:** none

### C9 Audit & traceability — 0.33 (high)

Every tool call that goes through the main agent loop, including MCP tools, is logged with its name, arguments and result through loguru to stdout. Logs are unstructured text, carry no actor or approver attribution, and are only written to a file if the developer calls save_logger. Workflow execution keeps an in-memory trajectory of messages, but nothing is durable or tamper-evident by default.

- **S L1:** Unstructured INFO log lines record tool name, arguments and results. — [evoagentx/actions/customize_action.py:275](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/actions/customize_action.py#L275); [evoagentx/actions/customize_action.py:412](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/actions/customize_action.py#L412) (verified)
  - *To reach the next level:* Write a structured per-call record (args, status, timestamps) to a session transcript.
- **C L2:** All tools invoked via CustomizeAction (built-in, MCP, generated) pass the logging line; approvals/denials and ActionAgent function calls are not recorded as audit events. — [evoagentx/actions/customize_action.py:263-280](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/actions/customize_action.py#L263-L280) (verified)
  - *To reach the next level:* Record approvals, denials and every execution path.
- **D L1:** Logging to stdout is on by default; file persistence is opt-in and lives where the agent's own file tools can reach it. — [evoagentx/core/logging.py:13-29](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/core/logging.py#L13-L29) (verified)
  - *To reach the next level:* Persist records by default outside the agent's write reach.
- **B L1:** Best-effort console logging; nothing persists on crash unless a file sink was added. — [evoagentx/core/logging.py:13](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/core/logging.py#L13) (verified)
  - *To reach the next level:* Flush durable records per action.
- **Cap:** none

### C10 Limits & kill switch — 0.40 (high)

Each agent action stops after 20 model calls by default and each workflow task after 5 agent executions, and the shell and MCP tools have 30-second timeouts. There is no cost or wall-clock budget (cost is tracked but never enforced), the model chooses the shell timeout itself, the Python and Docker executors have no timeout, and the workflow scheduler loop has no overall cap. Timeouts stop waiting but leave threads or MCP calls running.

- **S L2:** Iteration caps plus per-execution timeouts on shell and MCP tools are enforced in code. — [evoagentx/actions/customize_action.py:43](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/actions/customize_action.py#L43); [evoagentx/actions/customize_action.py:351](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/actions/customize_action.py#L351); [evoagentx/tools/mcp.py:244](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/mcp.py#L244) (verified)
  - *To reach the next level:* Add wall-clock and token/cost caps and rate limits on side-effecting tools.
- **C L2:** Covers the per-action loop and some tool timeouts; the workflow loop and sub-agents have no shared budget. — [evoagentx/workflow/workflow.py:173](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/workflow/workflow.py#L173); [evoagentx/workflow/workflow.py:42](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/workflow/workflow.py#L42) (verified)
  - *To reach the next level:* Make sub-agents and workflow tasks count against one budget.
- **D L1:** The model passes the shell timeout as a tool argument and can raise it; Python exec has no timeout at all. — [evoagentx/tools/cmd_toolkit.py:274-290](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/tools/cmd_toolkit.py#L274-L290); [evoagentx/models/model_utils.py:100](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/models/model_utils.py#L100) (verified)
  - *To reach the next level:* Fix limits outside model control.
- **B L1:** No spend ceiling; timed-out calls keep running in threads (asyncio.to_thread) or in the MCP event loop. — [evoagentx/actions/customize_action.py:280](https://github.com/anative-lab/evoagentx/blob/d77fd6b9a3e76c8dd83bebe3374c53a3f5d16f54/evoagentx/actions/customize_action.py#L280) (verified)
  - *To reach the next level:* Cancel in-flight work on timeout/stop and add a spend ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Search/browser/RSS/Gmail tool results appended as user messages (evoagentx/memory/context_manager.py:115) · [B] sensitive data/systems: Provider keys copied into os.environ (evoagentx/models/litellm_model.py:45); full host filesystem via file/shell tools · [C] state change / egress: http_request any method/URL (evoagentx/tools/request.py:19), Gmail send_draft (evoagentx/tools/gmail_tools.py:1409), shell (evoagentx/tools/cmd_toolkit.py:205) · Same default session? Yes

## Highest-impact improvements
1. Add a per-call approval hook in CustomizeAction._call_single_tool, on by default for exec, write, network and messaging tools, showing exact arguments. — C2 S L0→L3, +0.225 before caps (Playbook 5)
2. Make DockerInterpreter the default code executor and harden it (non-root, cap-drop, network none, resource limits, exec timeout). — C4 D L0→L3, +0.150 before caps (Playbook 3)
3. Remove import-time load_dotenv() calls; load config only from explicit paths or user scope. — C6 S L0→L2, +0.150 before caps (Playbook 2)
4. Pass a scrubbed env to CMDToolkit subprocesses and stop writing provider keys into os.environ. — C8 B L0→L2, +0.100 before caps (Playbook 4)
5. Add resolved-path containment to file tools and a host allowlist with internal-address blocking to http_request. — C3 S L0→L3, +0.225 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Clone used --filter=blob:limit=1m; blobs over 1MB (mostly assets/corpora) were not examined.
- FastMCP / MCP SDK subprocess environment behaviour was inferred, not read.
- Benchmark harnesses, optimizers' internals, and the research_tools subpackage were sampled, not read exhaustively.
- No text aimed at AI reviewers was found. README.md:309 describes PythonInterpreterToolkit as 'Safely execute ... with sandboxed imports', which the default (empty allowed_imports disables the check) does not support.
