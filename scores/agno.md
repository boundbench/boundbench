# Defense-in-Depth Score: Agno

**Repo:** https://github.com/agno-agi/agno · **Commit:** `3ca74c272fa5017fae4fab234e988378c8f14c63` (v3.1.1) · **Reviewed:** 2026-10-03
**What it is:** Framework/runtime to build, run and manage agent platforms
**Category:** Agent Frameworks
**Scored configuration:** Python library defaults: Agent(model=..., tools=[...]) with default constructor arguments and bundled toolkits as constructed with their defaults; AgentOS not enabled.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions opt-in · sub agents yes · external communication yes

## Score: 2.0 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L0 | L0 | 0.15 | G1 | **0.15** (alt) | High |
| C2 | Approval gates | L3 | L1 | L0 | L0 | 0.30 | C2-POWERBYPASS | **0.25** | High |
| C3 | Tool & action scoping | L2 | L1 | L1 | L0 | 0.28 | — | **0.28** | High |
| C4 | Code-execution isolation | L4 | L1 | L0 | L2 | 0.47 | G1 | **0.47** (alt) | Medium |
| C5 | Untrusted input blast radius | L1 | L0 | L0 | L0 | 0.07 | G1 | **0.07** (alt) | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L0 | 0.05 | — | **0.05** | High |
| C7 | Third-party extensions | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C8 | Secrets & sensitive-data protection | L1 | L0 | L1 | L0 | 0.12 | — | **0.12** | High |
| C9 | Audit & traceability | L2 | L2 | L0 | L1 | 0.35 | G1 | **0.35** (alt) | High |
| C10 | Limits & kill switch | L1 | L1 | L0 | L0 | 0.15 | G1 | **0.15** | High |


Agno gives developers powerful toolkits (shell, in-process Python, files, web, MCP) but few guardrails switched on. Out of the box no tool needs approval, code runs on the host with every API key in the environment, and there is no step, time or cost limit. Its human-in-the-loop confirmation and remote sandbox toolkits are well built but opt-in, so safety depends on the developer turning them on.

## Critical gaps
- A hijacked Agno agent holds every credential in the host process environment: tools read keys from env and ShellTools passes the full environment to subprocesses. (ASI03, T3, LLM06; C1) — [libs/agno/agno/tools/shell.py:55-59](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/shell.py#L55-L59); [libs/agno/agno/tools/github.py:23](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/github.py#L23)
- Shell and Python-exec tools run without any approval by default; the confirmation gate is opt-in per tool. (ASI02, ASI09, T10, LLM06; C2) — [libs/agno/agno/tools/function.py:1263](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/function.py#L1263); [libs/agno/agno/tools/shell.py:55-59](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/shell.py#L55-L59)
- ShellTools and PythonTools run model-generated commands and code on the host (PythonTools in-process via exec) with the full environment and no sandbox. (ASI05, T11, LLM05; C4) — [libs/agno/agno/tools/shell.py:55-59](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/shell.py#L55-L59); [libs/agno/agno/tools/python.py:202](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/python.py#L202)
- Untrusted tool output enters context unmarked and a hijacked agent can both exfiltrate (arbitrary URL fetch) and act irreversibly (shell) with no human involved. (ASI01, T6, LLM01; C5) — [libs/agno/agno/models/base.py:3145](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/models/base.py#L3145); [libs/agno/agno/tools/webtools.py:38](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/webtools.py#L38); [libs/agno/agno/tools/shell.py:55-59](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/shell.py#L55-L59)

## Criterion details

### C1 Identity & least privilege — 0.15 (high)

Agno has no notion of a scoped agent identity. Every toolkit builds its own client from whatever API keys sit in the process environment (for example a GitHub token), and the shell tool starts commands that inherit the full environment, so a hijacked agent holds everything the hosting process holds. AgentOS can check a caller's JWT scopes before a run starts, but that is off by default and only guards the HTTP entry point, not what tools do with their credentials.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** Tools read long-lived keys straight from the process environment and construct their own clients; there is no per-tool or per-request credential narrowing. — [libs/agno/agno/tools/github.py:23](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/github.py#L23); [libs/agno/agno/models/openai/chat.py:108](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/models/openai/chat.py#L108) (verified)
    - *To reach the next level:* Give tools dedicated, role-scoped credentials instead of reading ambient env keys.
  - **C L0:** Each toolkit constructs its own privileged client and ShellTools passes the whole environment to every subprocess (no env= argument). — [libs/agno/agno/tools/shell.py:55-59](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/shell.py#L55-L59); [libs/agno/agno/tools/github.py:23](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/github.py#L23) (verified)
    - *To reach the next level:* Route tool credential use through one authorization layer and scrub subprocess environments.
  - **D L0:** A default Agent runs with whatever authority the host process has; nothing narrows it unless the developer hand-hardens credentials. — [libs/agno/agno/tools/github.py:23](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/github.py#L23); [libs/agno/agno/os/app.py:299](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/os/app.py#L299) (verified)
    - *To reach the next level:* Ship a minimal default credential posture (read-only or explicit grants per toolkit).
  - **B L0:** Rated as if all credentials are reachable: any key in the environment (cloud, GitHub, email, DB) is usable by a hijacked agent through ShellTools or the toolkit that owns it. — [libs/agno/agno/tools/shell.py:55-59](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/shell.py#L55-L59); [libs/agno/agno/tools/github.py:23](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/github.py#L23) (verified)
    - *To reach the next level:* Bound what a hijacked agent can reach to one scoped, short-lived credential.
- **opt-in AgentOS authorization (JWT scopes per route)** (alt; raw 0.15, cap G1 → 0.15) ← counted
  - **S L1:** With authorization=True, AgentOS checks the caller's JWT scopes per route (e.g. agents:run) before a run, but tools still use the process's ambient keys. — [libs/agno/agno/os/scopes.py:439](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/os/scopes.py#L439); [libs/agno/agno/os/app.py:299](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/os/app.py#L299) (verified)
    - *To reach the next level:* Narrow the credentials tools actually use, not just who may start a run.
  - **C L1:** Only the HTTP layer is checked; the tool executor reached from the agent loop uses ambient credentials. — [libs/agno/agno/os/scopes.py:439](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/os/scopes.py#L439); [libs/agno/agno/tools/shell.py:55-59](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/shell.py#L55-L59) (verified)
    - *To reach the next level:* Apply authorization at the tool executor for every tool, MCP server and member agent.
  - **D L0:** authorization defaults to False on AgentOS. — [libs/agno/agno/os/app.py:299](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/os/app.py#L299) (verified)
    - *To reach the next level:* Enable authorization by default.
  - **B L0:** Even with route scopes, a hijacked run still holds every env credential. — [libs/agno/agno/tools/shell.py:55-59](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/shell.py#L55-L59) (verified)
    - *To reach the next level:* Bound tool credentials per run.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C2 Approval gates — 0.25 (high)

Agno has a solid human-in-the-loop primitive: a tool marked as requiring confirmation pauses the run and hands the caller the exact tool name and arguments to approve or reject. But nothing is marked by default — not even the shell, Python-exec or file-write tools — so out of the box the agent runs every tool call without asking anyone. Developers must opt in per tool, and ShellTools' own docstring tells them to.

- **S L3:** When a function has requires_confirmation set, the loop pauses and records a ToolExecution with the exact tool_args for the caller to confirm or reject. — [libs/agno/agno/models/base.py:2445-2452](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/models/base.py#L2445-L2452); [libs/agno/agno/run/requirement.py:109-114](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/run/requirement.py#L109-L114) (verified)
  - *To reach the next level:* Add argument-level allow/deny rules and guarantee the executed arguments equal the approved ones.
- **C L1:** Only tools explicitly flagged are gated; ShellTools, PythonTools and FileTools register their mutating functions unflagged. — [libs/agno/agno/models/base.py:2445](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/models/base.py#L2445); [libs/agno/agno/tools/toolkit.py:174](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/toolkit.py#L174); [libs/agno/agno/tools/shell.py:55-59](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/shell.py#L55-L59) (verified)
  - *To reach the next level:* Gate every mutating tool path, including MCP and member agents, and reject unknown tools by default.
- **D L0:** requires_confirmation defaults to None and requires_confirmation_tools to an empty list, so approval is opt-in. — [libs/agno/agno/tools/function.py:1263](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/function.py#L1263); [libs/agno/agno/tools/toolkit.py:174](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/toolkit.py#L174) (verified)
  - *To reach the next level:* Turn approval on by default for exec, write and egress tools.
- **B L0:** Ungated tools include arbitrary shell and in-process Python execution with no undo or checkpoints. — [libs/agno/agno/tools/shell.py:55-59](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/shell.py#L55-L59); [libs/agno/agno/tools/python.py:202](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/python.py#L202) (verified)
  - *To reach the next level:* Add checkpoints/rollback and previews for consequential actions.
- **Cap:** C2-POWERBYPASS — The most powerful path (ShellTools.run_shell_command / PythonTools.run_python_code) executes without crossing the gate by default.

### C3 Tool & action scoping — 0.28 (high)

All Python tools get typed argument validation through pydantic, and the file toolkits confine paths to a base directory after resolving symlinks. But the most powerful bundled tools take raw input: ShellTools runs any argv, PythonTools runs any code, and the web tools fetch any URL with redirects followed and no block on internal addresses. Toolkits also enable their write functions by default (FileTools.save_file is on).

- **S L2:** Typed schemas via pydantic validate_call on every callable, and resolved-path containment for file tools, but shell, Python and URL tools are raw passthrough. — [libs/agno/agno/tools/function.py:1751](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/function.py#L1751); [libs/agno/agno/utils/path_safety.py:36](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/utils/path_safety.py#L36); [libs/agno/agno/tools/webtools.py:38](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/webtools.py#L38) (verified)
  - *To reach the next level:* Replace general tools with narrow ones and add host allowlists that block internal addresses.
- **C L1:** Type validation covers all Python tools, but meaningful validation (containment) covers only file-path helpers, not shell/exec/URL tools; MCP tools are passed through to the server. — [libs/agno/agno/tools/function.py:1751](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/function.py#L1751); searched `rg -n -S '169\.254|ssrf|is_link_local|is_loopback|ip_address\('` in `libs/agno/agno/tools` → 0 hits (no internal-address or SSRF checks in any bundled toolkit) (verified)
  - *To reach the next level:* Add a shared policy layer that validates arguments for every tool including MCP.
- **D L1:** An Agent starts with no tools, but each toolkit enables its write/exec functions by default (FileTools save_file=True, ShellTools run_shell_command=True); they can be excluded individually. — [libs/agno/agno/agent/agent.py:190](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/agent/agent.py#L190); [libs/agno/agno/tools/file/file.py:98](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/file/file.py#L98); [libs/agno/agno/tools/shell.py:12-13](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/shell.py#L12-L13) (verified)
  - *To reach the next level:* Make toolkits read-only by default with write/exec requiring explicit enabling.
- **B L0:** A misused ShellTools or PythonTools call reaches the whole machine; URL tools reach any host. — [libs/agno/agno/tools/shell.py:55-59](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/shell.py#L55-L59); [libs/agno/agno/tools/webtools.py:38](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/webtools.py#L38) (verified)
  - *To reach the next level:* Scope tools to a workspace with quantity bounds.
- **Cap:** none

### C4 Code-execution isolation — 0.47 (medium)

The built-in code tools run directly on the host: ShellTools spawns host processes, PythonTools calls exec() inside the agent's own process, skill scripts and MCP stdio servers start as local processes, and the docstrings say plainly that none of this is a sandbox. Remote sandboxes (E2B, Daytona) exist as separate opt-in toolkits, but they only cover their own tools; everything else still runs on the host with the agent's credentials.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** No isolation: ShellTools uses same-user subprocess and PythonTools uses in-process exec. — [libs/agno/agno/tools/shell.py:55-59](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/shell.py#L55-L59); [libs/agno/agno/tools/python.py:202](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/python.py#L202) (verified)
    - *To reach the next level:* Run model-generated code in a hardened container or microVM by default.
  - **C L0:** No execution path is sandboxed in the default toolkits. — searched `rg -n -S -e 'sandbox|seccomp|landlock|bwrap|docker'` in `libs/agno/agno/agent libs/agno/agno/models/base.py libs/agno/agno/tools/function.py` → 0 hits (no isolation layer anywhere in the agent loop or tool executor); [libs/agno/agno/skills/utils.py:163-170](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/skills/utils.py#L163-L170) (verified)
    - *To reach the next level:* Route every exec path through an isolation layer.
  - **D L0:** No sandbox in the default configuration. — searched `rg -n -S -e 'sandbox|seccomp|landlock|bwrap|docker'` in `libs/agno/agno/agent libs/agno/agno/models/base.py libs/agno/agno/tools/function.py` → 0 hits (no isolation layer anywhere in the agent loop or tool executor) (verified)
    - *To reach the next level:* Make sandboxed execution the default for exec tools.
  - **B L0:** Code runs as the host user with the full environment, including every API key, plus network. — [libs/agno/agno/tools/shell.py:55-59](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/shell.py#L55-L59); [libs/agno/agno/tools/python.py:202](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/python.py#L202) (verified)
    - *To reach the next level:* Isolate execution from host secrets and network.
- **opt-in E2B / Daytona remote sandbox toolkits** (alt; raw 0.47, cap G1 → 0.47) ← counted
  - **S L4:** E2BTools creates a remote ephemeral sandbox via the E2B service. — [libs/agno/agno/tools/e2b.py:49](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/e2b.py#L49) (verified)
  - **C L1:** Only that toolkit's tools are sandboxed; ShellTools, PythonTools, skill scripts and MCP stdio servers still run on the host. — [libs/agno/agno/tools/e2b.py:49](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/e2b.py#L49); [libs/agno/agno/skills/utils.py:163-170](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/skills/utils.py#L163-L170) (verified)
    - *To reach the next level:* Route every exec path (skills, MCP stdio, shell) through the sandbox.
  - **D L0:** Opt-in: requires adding E2BTools and an E2B_API_KEY. — [libs/agno/agno/tools/e2b.py:40](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/e2b.py#L40) (verified)
    - *To reach the next level:* Make remote sandbox execution the default.
  - **B L2:** No host secrets are passed into the sandbox and it times out after 300 s, but the model can extend the timeout and the sandbox has outbound network by default (E2B behaviour, inferred). — [libs/agno/agno/tools/e2b.py:49](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/e2b.py#L49); [libs/agno/agno/tools/e2b.py:615](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/e2b.py#L615) (inferred)
    - *To reach the next level:* Restrict sandbox egress to an allowlist and prevent the model extending its lifetime.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.07 (high)

Tool results — web pages, files, MCP outputs, other agents' replies — go into the conversation as ordinary tool messages, with nothing marking them as untrusted and nothing changing what the agent may do after reading them. The bundled prompt-injection guardrail is opt-in, only matches a short list of phrases, and only checks the user's own input, not tool output. A hijacked agent with the usual toolkits can both leak secrets (any URL fetch) and act irreversibly (shell) without a human.

- **default configuration** (default; raw 0.00, cap C5-WORSTCASE → 0.00)
  - **S L0:** No structural limit on a hijacked agent; tool output is appended to the messages unchanged. — [libs/agno/agno/models/base.py:3145](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/models/base.py#L3145) (verified)
    - *To reach the next level:* Force egress and state-changing tools through approval once untrusted content enters a session.
  - **C L0:** Untrusted sources are not distinguished: tool results enter context as regular tool messages. — [libs/agno/agno/models/base.py:3145](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/models/base.py#L3145) (verified)
    - *To reach the next level:* Tag tool results, MCP outputs and member-agent messages as untrusted and act on the tag.
  - **D L0:** No control is on by default; the guardrail must be added as a pre_hook. — [libs/agno/agno/agent/agent.py:207](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/agent/agent.py#L207) (verified)
    - *To reach the next level:* Ship an on-by-default taint/approval policy.
  - **B L0:** With bundled toolkits a hijacked agent can exfiltrate via URL fetch (expand_url follows any redirect) and act irreversibly via shell, unattended. — [libs/agno/agno/tools/webtools.py:38](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/webtools.py#L38); [libs/agno/agno/tools/shell.py:55-59](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/shell.py#L55-L59) (verified)
    - *To reach the next level:* Require human approval for egress and irreversible actions after untrusted input.
- **opt-in PromptInjectionGuardrail** (alt; raw 0.07, cap G1 → 0.07) ← counted
  - **S L1:** Substring match on a fixed phrase list; detection only. — [libs/agno/agno/guardrails/prompt_injection.py:40](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/guardrails/prompt_injection.py#L40) (verified)
    - *To reach the next level:* Replace detection with structural limits (Rule of Two).
  - **C L0:** Checks only run_input (the user's message), never tool results or MCP outputs. — [libs/agno/agno/guardrails/prompt_injection.py:40](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/guardrails/prompt_injection.py#L40) (verified)
    - *To reach the next level:* Apply to every untrusted source, especially tool results.
  - **D L0:** Opt-in pre_hook. — [libs/agno/agno/guardrails/prompt_injection.py:40](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/guardrails/prompt_injection.py#L40) (verified)
    - *To reach the next level:* Enable by default.
  - **B L0:** Same worst case: a hijack via tool output is never inspected. — [libs/agno/agno/models/base.py:3145](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/models/base.py#L3145) (verified)
    - *To reach the next level:* Remove the exfil+action combination.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C6 Memory, context & configuration integrity — 0.05 (high)

Long-term user memory is off by default, but once enabled the model can write memories through a tool (via an LLM memory manager) and they are injected into every later system prompt with no provenance check, approval or expiry. Memories are keyed by user_id, but when an app doesn't pass one every caller shares the same 'default' bucket. Agno does not auto-load instruction files or a .env from the working directory.

- **S L0:** When agentic memory is on, the model's update_user_memory tool writes freely and memories are re-injected into the system prompt as trusted context. — [libs/agno/agno/agent/_messages.py:328-333](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/agent/_messages.py#L328-L333); [libs/agno/agno/agent/_default_tools.py:36-52](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/agent/_default_tools.py#L36-L52) (verified)
  - *To reach the next level:* Gate memory writes (approval, validation or source restriction) and add expiry.
- **C L0:** No memory or knowledge write path is validated. — [libs/agno/agno/agent/_messages.py:328-333](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/agent/_messages.py#L328-L333) (verified)
  - *To reach the next level:* Control every store (memories, summaries, learnings, knowledge).
- **D L1:** Memory queries filter by user_id, but a missing user_id falls back to a shared 'default' namespace. — [libs/agno/agno/memory/manager.py:178](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/memory/manager.py#L178); [libs/agno/agno/agent/agent.py:132](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/agent/agent.py#L132) (verified)
  - *To reach the next level:* Require a user/session namespace and refuse writes without one.
- **B L0:** Poisoned memories persist across sessions and, in the shared default namespace, across users, and steer later tool use. — [libs/agno/agno/memory/manager.py:178](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/memory/manager.py#L178); [libs/agno/agno/agent/_messages.py:328-333](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/agent/_messages.py#L328-L333) (verified)
  - *To reach the next level:* Make memory session-scoped or human-reviewed with rollback.
- **Cap:** none

### C7 Third-party extensions — 0.10 (high)

Agno loads whatever MCP servers, skills and packages the developer points it at, with no pinning, hash check or re-approval. The MCP command check allows npx/uvx (so whatever version is latest at launch), but MCP stdio servers get only a minimal environment. Skill scripts run on the host with the full environment, and the bundled PythonTools lets the model pip-install any package it names.

- **S L0:** PythonTools exposes a model-chosen package installer (pip install <package_name>) and skill scripts run unverified; MCP launches via npx/uvx are unpinned. — [libs/agno/agno/tools/python.py:231](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/python.py#L231); [libs/agno/agno/utils/mcp.py:402](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/utils/mcp.py#L402); [libs/agno/agno/skills/utils.py:163-170](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/skills/utils.py#L163-L170) (verified)
  - *To reach the next level:* Pin and hash-verify extensions and remove model-reachable installers.
- **C L0:** No extension type (MCP, skills, packages) is verified. — [libs/agno/agno/utils/mcp.py:402](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/utils/mcp.py#L402); [libs/agno/agno/skills/utils.py:163-170](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/skills/utils.py#L163-L170) (verified)
  - *To reach the next level:* Verify all extension types.
- **D L1:** Nothing third-party is enabled by default, but PythonTools registers pip_install_package with no enable flag, so adding the toolkit lets the model install packages on first use. — [libs/agno/agno/tools/python.py:231](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/python.py#L231) (verified)
  - *To reach the next level:* Show the exact package/command for each install and require explicit consent.
- **B L1:** MCP stdio servers get a scrubbed environment (mcp get_default_environment), but skill scripts run as a separate process with the full environment. — [libs/agno/agno/tools/mcp/mcp.py:411-418](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/mcp/mcp.py#L411-L418); [libs/agno/agno/skills/utils.py:163-170](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/skills/utils.py#L163-L170) (verified)
  - *To reach the next level:* Run every extension with a scrubbed environment and its own scoped credentials.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.12 (high)

API keys come from environment variables and are not masked; nothing scrubs them from subprocess environments, so a shell or skill command can read every key. Anonymous telemetry is on by default and sends run metadata (model, flags such as has_tools) to Agno's API, but no prompts or tool output. Debug logging, when turned on, prints full messages without redaction.

- **S L1:** Secrets are read from env vars; no general redaction layer for logs, tool results or model-bound messages was found. — [libs/agno/agno/models/openai/chat.py:108](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/models/openai/chat.py#L108); [libs/agno/agno/tools/github.py:23](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/github.py#L23) (verified)
  - *To reach the next level:* Add type-level masking and log redaction on main paths.
- **C L0:** No path (logs, model-bound messages, subprocess environment) is systematically protected. — [libs/agno/agno/tools/shell.py:55-59](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/shell.py#L55-L59); [libs/agno/agno/models/message.py:381](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/models/message.py#L381) (verified)
  - *To reach the next level:* Protect at least logs and subprocess environments.
- **D L1:** Telemetry is on by default (telemetry=True) but content-free; verbose logs are one flag away and unredacted. — [libs/agno/agno/agent/agent.py:386](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/agent/agent.py#L386); [libs/agno/agno/agent/_telemetry.py:13-23](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/agent/_telemetry.py#L13-L23); [libs/agno/agno/api/settings.py:57](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/api/settings.py#L57) (verified)
  - *To reach the next level:* Make telemetry opt-in.
- **B L0:** Long-lived, high-privilege keys in the process env are reachable by every ShellTools subprocess. — [libs/agno/agno/tools/shell.py:55-59](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/shell.py#L55-L59); [libs/agno/agno/tools/github.py:23](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/github.py#L23) (verified)
  - *To reach the next level:* Use scoped short-lived credentials not exposed to subprocesses.
- **Cap:** none

### C9 Audit & traceability — 0.35 (high)

With default settings Agno keeps no durable record of what the agent did: tool calls go only to debug-level logs, and a few toolkits print an INFO line. If you configure a database, each run is stored with its tool calls, arguments, results, timestamps, user_id and approval flags, but by default only when the run ends, so a crash mid-run loses it. OpenTelemetry tracing is available as an opt-in.

- **default configuration** (default; raw 0.20 → 0.20)
  - **S L1:** Default: unstructured INFO lines in some toolkits (e.g. 'Running shell command'); message/tool logging is debug-only. — [libs/agno/agno/tools/shell.py:54](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/shell.py#L54); [libs/agno/agno/models/message.py:381](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/models/message.py#L381) (verified)
    - *To reach the next level:* Write a structured record of every tool call by default.
  - **C L1:** Only a few toolkits log at INFO; most tool calls are unrecorded without debug mode. — [libs/agno/agno/models/message.py:381](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/models/message.py#L381) (verified)
    - *To reach the next level:* Record every tool call including MCP and member agents.
  - **D L1:** INFO logging is on by default but goes to the process's own stdout/handlers. — [libs/agno/agno/models/message.py:381](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/models/message.py#L381) (verified)
    - *To reach the next level:* Write records outside the agent process's control.
  - **B L0:** Without a db nothing is persisted; logging failures are silent. — [libs/agno/agno/agent/agent.py:140](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/agent/agent.py#L140) (verified)
    - *To reach the next level:* Flush a durable record per action.
- **opt-in database run storage** (alt; raw 0.35, cap G1 → 0.35) ← counted
  - **S L2:** With db set, RunOutput stores ToolExecution entries with tool_name, tool_args, result, created_at and confirmation fields. — [libs/agno/agno/models/response.py:30-43](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/models/response.py#L30-L43); [libs/agno/agno/agent/agent.py:140](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/agent/agent.py#L140) (verified)
    - *To reach the next level:* Add approver identity and tamper-evident storage.
  - **C L2:** Covers tool calls recorded in the run output; child runs are linked by child_run_id. — [libs/agno/agno/models/response.py:30-43](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/models/response.py#L30-L43) (verified)
    - *To reach the next level:* Record approvals/denials with approver, config changes and memory writes.
  - **D L0:** db defaults to None. — [libs/agno/agno/agent/agent.py:140](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/agent/agent.py#L140) (verified)
    - *To reach the next level:* Persist runs by default.
  - **B L1:** Default checkpoint mode writes only at terminal states, so a crash mid-run loses the record. — [libs/agno/agno/agent/agent.py:154](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/agent/agent.py#L154) (verified)
    - *To reach the next level:* Persist per action.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C10 Limits & kill switch — 0.15 (high)

An Agno agent has no step, time or cost limit by default: the model loop runs until the model stops calling tools. tool_call_limit is opt-in, and when it trips it only returns an error message to the model while the loop keeps going. Runs can be cancelled, but cancellation is a flag checked between steps, so in-flight tool calls (like a shell command with no timeout) keep running.

- **S L1:** Only an opt-in tool-call cap and a cooperative cancel flag; no wall-clock or token/cost cap on the agent loop. — [libs/agno/agno/models/base.py:2434](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/models/base.py#L2434); [libs/agno/agno/run/cancel.py:99](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/run/cancel.py#L99) (verified)
  - *To reach the next level:* Add wall-clock and token/cost caps enforced in code.
- **C L1:** The cap applies to the top-level model loop only; ShellTools has no per-call timeout. — [libs/agno/agno/models/base.py:2434](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/models/base.py#L2434); [libs/agno/agno/tools/shell.py:55-59](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/tools/shell.py#L55-L59) (verified)
  - *To reach the next level:* Apply timeouts to every tool and count sub-agents against the parent budget.
- **D L0:** tool_call_limit defaults to None and the loop is 'while True'. — [libs/agno/agno/agent/agent.py:193](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/agent/agent.py#L193); [libs/agno/agno/models/base.py:700](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/models/base.py#L700) (verified)
  - *To reach the next level:* Ship sensible default limits.
- **B L0:** No ceiling: a runaway agent can loop and spend indefinitely. — [libs/agno/agno/models/base.py:700](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/models/base.py#L700); [libs/agno/agno/agent/agent.py:193](https://github.com/agno-agi/agno/blob/3ca74c272fa5017fae4fab234e988378c8f14c63/libs/agno/agno/agent/agent.py#L193) (verified)
  - *To reach the next level:* Add tight default ceilings and a halt that cancels in-flight calls.
- **Cap:** G1 — tool_call_limit, the only limit, is opt-in (defaults to None).

## Rule-of-Two check
[A] untrusted input: Tool results (web, files, MCP, member agents) appended unmarked as tool messages, models/base.py:3145 · [B] sensitive data/systems: Process env API keys read by toolkits and inherited by ShellTools subprocesses, tools/github.py:23, tools/shell.py:55 · [C] state change / egress: ShellTools host exec (tools/shell.py:55) and arbitrary URL fetch with redirects (tools/webtools.py:38) · Same default session? Yes

## Highest-impact improvements
1. Default requires_confirmation=True for ShellTools, PythonTools, CodingTools run_shell and FileTools write functions. — C2 D L0→L2, +0.100 before caps (5)
2. Set a default tool_call_limit / max model iterations and stop the loop (not just return an error) when it trips. — C10 D L0→L2, +0.100 before caps (3)
3. Pass a scrubbed env= to ShellTools and skill-script subprocesses instead of inheriting os.environ. — C8 C L0→L1, +0.075 before caps (4)
4. Make telemetry opt-in. — C8 D L1→L2, +0.050 before caps
5. Require an explicit user_id for memory writes instead of the shared 'default' namespace. — C6 D L1→L2, +0.050 before caps (2)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was installed, built or run.
- Scored the core library libs/agno/agno; AgentOS deployment features (authorization, user isolation, interfaces such as Slack/Telegram), agno_infra and agnoctl were reviewed only where cited and not scored as the primary mode.
- Of ~150 bundled toolkits only the high-impact ones (shell, python, coding, file, web, MCP, skills, E2B/Daytona, GitHub) were read in detail.
- E2B sandbox network behaviour and the mcp SDK's get_default_environment contents are library behaviour inferred, not read in this repo.
- No reviewer-steering text was found in README.md, AGENTS.md or CLAUDE.md.
