# Defense-in-Depth Score: TaskWeaver

**Repo:** https://github.com/microsoft/taskweaver · **Commit:** `d44ddef23f90059fb17999d3095db4240e98f955` · **Reviewed:** 2026-10-04
**What it is:** Microsoft's code-first agent framework that plans data-analytics tasks and executes LLM-generated Python in a stateful Jupyter kernel.
**Category:** Agent Frameworks
**Scored configuration:** TaskWeaverApp / `python -m taskweaver -p ./project/` with the shipped project config: planner + code_interpreter roles, container kernel mode, code verification off, two sample plugins enabled.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory opt-in · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 3.0 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L2 | L2 | L3 | 0.47 | — | **0.47** | High |
| C2 | Approval gates | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C3 | Tool & action scoping | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C4 | Code-execution isolation | L2 | L3 | L2 | L2 | 0.57 | — | **0.57** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L2 | L2 | L2 | L3 | 0.55 | — | **0.55** | High |
| C7 | Third-party extensions | L1 | L0 | L1 | L1 | 0.17 | — | **0.17** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C9 | Audit & traceability | L1 | L2 | L2 | L2 | 0.42 | — | **0.42** | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |


TaskWeaver runs every piece of model-written Python immediately, with no human approval, so its safety rests almost entirely on the per-session Docker container it uses by default. That container keeps host credentials out and fails closed when Docker is missing, but it is unhardened, has full network egress and no resource limits. A prompt injection in data or tool output can therefore exfiltrate the user's files or call external APIs unattended.

## Critical gaps
- A hijacked TaskWeaver session can read the user's data files and send them anywhere, or call arbitrary external APIs, with no human in the loop: model code executes immediately in a container with full network egress. (ASI01, LLM01, T6; C5) — [taskweaver/code_interpreter/code_interpreter/code_interpreter.py:267-272](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/code_interpreter/code_interpreter/code_interpreter.py#L267-L272); [taskweaver/ces/environment.py:257-279](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L257-L279); [taskweaver/planner/planner.py:169-176](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/planner/planner.py#L169-L176)

## Criterion details

### C1 Identity & least privilege — 0.47 (high)

TaskWeaver has no per-request authorization layer or scoped identity of its own. In the default container mode the code kernel gets a scrubbed environment with only TaskWeaver variables, so the LLM API key held by the host process is not visible to generated code; but every plugin's configuration is loaded into the same kernel the model's code runs in. Switching to local mode, a single config value, hands the kernel a full copy of the host environment, which the code itself marks as a TODO to filter.

- **S L1:** The container kernel is a separate low-privilege identity (container user, no host env) but broadly scoped: full network and every plugin's configuration in one shared interpreter. — [taskweaver/ces/environment.py:240-249](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L240-L249); [taskweaver/ces/environment.py:616-625](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L616-L625) (verified)
  - *To reach the next level:* No per-tool or per-plugin credential scoping; all plugin configs share one kernel with model-written code.
- **C L2:** All built-in code paths run in the scrubbed container kernel by default; plugins share that kernel and can see each other's configuration. — [taskweaver/ces/environment.py:240-249](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L240-L249); [taskweaver/code_interpreter/code_executor.py:132-144](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/code_interpreter/code_executor.py#L132-L144) (verified)
  - *To reach the next level:* Plugins do not get separately scoped authority; there is no authorization layer every path traverses.
- **D L2:** Default container mode passes no host credentials; widening to the full host environment is one operator config/env value (execution_service.kernel_mode=local). — [taskweaver/module/execution_service.py:18-21](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/module/execution_service.py#L18-L21); [taskweaver/ces/environment.py:212-214](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L212-L214) (verified)
  - *To reach the next level:* Widening is a plain config value, not an explicit elevation; the local-mode warning does not mention that the full host environment, including API keys, is passed.
- **B L3:** In the default configuration the kernel holds no external credentials beyond those an operator puts in plugin configs; the LLM key stays in the host process. — [taskweaver/ces/environment.py:240-249](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L240-L249); [project/plugins/sql_pull_data.yaml:29-35](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/project/plugins/sql_pull_data.yaml#L29-L35) (verified)
  - *To reach the next level:* Network egress from the kernel is unrestricted and any plugin credential is reachable by generated code; credentials are long-lived.
- **Cap:** none

### C2 Approval gates — 0.05 (high)

There is no human approval step anywhere in TaskWeaver. Code generated by the model is optionally checked by a static filter (off by default in container mode) and then executed immediately; plugin calls happen inside that code. A search of the Python source finds no confirmation or approval logic, and the README's own wish-list asks for 'user confirmation before running the plugin'.

- **S L0:** No approval mechanism exists; generated code goes straight from verification to execution. — [taskweaver/code_interpreter/code_interpreter/code_interpreter.py:267-272](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/code_interpreter/code_interpreter/code_interpreter.py#L267-L272); searched `rg -n -i 'confirm|approv' -g '*.py'` in `taskweaver` → 0 hits (No approval or confirmation logic in any Python source.) (verified)
  - *To reach the next level:* No per-call human approval showing the exact code before it runs.
- **C L0:** The most powerful path, arbitrary Python execution, is ungated. — [taskweaver/code_interpreter/code_interpreter/code_interpreter.py:267-272](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/code_interpreter/code_interpreter/code_interpreter.py#L267-L272); [taskweaver/code_interpreter/code_interpreter_cli_only/code_interpreter_cli_only.py:71-78](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/code_interpreter/code_interpreter_cli_only/code_interpreter_cli_only.py#L71-L78) (verified)
  - *To reach the next level:* No gate on code execution, CLI commands, or plugin calls.
- **D L0:** Nothing to turn on; no approval setting exists. — searched `rg -n -i 'confirm|approv' -g '*.py'` in `taskweaver` → 0 hits (No approval or confirmation logic in any Python source.) (verified)
  - *To reach the next level:* No default-on approval.
- **B L1:** Unapproved code in the default container can delete or overwrite files in the session working directory and make arbitrary outbound network calls; nothing is checkpointed. — [taskweaver/ces/environment.py:257-279](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L257-L279) (verified)
  - *To reach the next level:* No checkpoint/rollback for the working directory and no preview of external actions.
- **Cap:** none

### C3 Tool & action scoping — 0.05 (high)

The core 'tool' is arbitrary Python executed in a Jupyter kernel, so a hijacked agent can do anything the kernel can. An AST-based module allowlist and function blocklist exists, but it is off by default in the default container mode and is credited under code-execution isolation, not here. Plugins declare typed parameters in YAML, but the model calls them from free-form code, so nothing enforces those schemas.

- **S L0:** Raw passthrough: the model writes arbitrary Python that is executed as-is; verification is off by default in container mode. — [taskweaver/code_interpreter/code_interpreter/code_interpreter.py:31](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/code_interpreter/code_interpreter/code_interpreter.py#L31); [taskweaver/code_interpreter/code_interpreter/code_interpreter.py:267-272](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/code_interpreter/code_interpreter/code_interpreter.py#L267-L272) (verified)
  - *To reach the next level:* No allowlist validation of what generated code may do (paths, hosts, quantities).
- **C L0:** No tool validates its inputs in the default configuration. — [taskweaver/code_interpreter/code_interpreter/code_interpreter.py:267-272](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/code_interpreter/code_interpreter/code_interpreter.py#L267-L272) (verified)
  - *To reach the next level:* No validation layer applied to code or plugin calls by default.
- **D L0:** Code execution with filesystem write and network is the default capability set. — [taskweaver/session/session.py:28](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/session/session.py#L28) (verified)
  - *To reach the next level:* No read-only default tool set.
- **B L1:** Arbitrary code limited only by the container: session working directory read-write and unrestricted network. — [taskweaver/ces/environment.py:257-279](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L257-L279) (verified)
  - *To reach the next level:* No narrowing to a workspace without network; no quantity bounds.
- **Cap:** none
- **Notes:** The AST verifier (code_verification.py) is forced on in local mode; its credit is taken once, in C4.

### C4 Code-execution isolation — 0.57 (high)

By default every piece of model-written code, including CLI-only shell commands and plugin code, runs in a per-session Docker container that is started from a stock python:3.10-slim based image. The container gets a scrubbed environment and only the session's working and kernel directories mounted, and TaskWeaver refuses to start rather than falling back to the host if Docker is unavailable. But the container is not hardened: no dropped capabilities, no resource limits, default network with full egress, kernel ports published on the host, and its entrypoint runs as root before dropping to a user with the host UID. Local mode, a single config value, runs code on the host behind an escapable AST filter.

- **S L2:** Stock Docker container with default capabilities and no seccomp/no-new-privileges/read-only settings; kernel runs as a non-root user after a root entrypoint. — [taskweaver/ces/environment.py:257-279](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L257-L279); [docker/ces_container/entrypoint.sh:7-13](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/docker/ces_container/entrypoint.sh#L7-L13); searched `rg -n 'cap_drop|mem_limit|network_mode|read_only|security_opt|pids_limit'` in `taskweaver` → 0 hits (No container hardening or limit options anywhere.) (verified)
  - *To reach the next level:* Not hardened: no cap_drop, no-new-privileges, read-only root, or seccomp profile.
- **C L3:** All model-reachable execution (code interpreter, CLI-only '!' commands, plugin-only mode, plugins) goes through the same kernel; local mode is the documented escape hatch; no host fallback when Docker fails. — [taskweaver/ces/environment.py:141-149](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L141-L149); [taskweaver/code_interpreter/code_interpreter_cli_only/code_interpreter_cli_only.py:71-78](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/code_interpreter/code_interpreter_cli_only/code_interpreter_cli_only.py#L71-L78); [taskweaver/ces/environment.py:616-625](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L616-L625) (verified)
  - *To reach the next level:* Fail-closed coverage of processes spawned inside is fine, but the opt-out is a plain config value rather than an explicitly named dangerous flag.
- **D L2:** On by default; disabled by setting execution_service.kernel_mode=local in the project config or the EXECUTION_SERVICE_KERNEL_MODE env var, which only prints an advisory message. — [taskweaver/module/execution_service.py:18-21](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/module/execution_service.py#L18-L21); [taskweaver/module/execution_service.py:23-30](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/module/execution_service.py#L23-L30); [taskweaver/config/config_mgt.py:92](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/config/config_mgt.py#L92) (verified)
  - *To reach the next level:* Disabling should need an explicit, loudly named operator flag rather than an ordinary config/env value.
- **B L2:** Session working directory mounted read-write plus unrestricted network egress; no secrets in env; no CPU/memory/PID limits; container removed on session stop; kernel ports published on host interfaces. — [taskweaver/ces/environment.py:257-279](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L257-L279); [taskweaver/ces/environment.py:405-431](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L405-L431) (verified)
  - *To reach the next level:* No network restriction or resource limits; kernel ports are bound on all host interfaces.
- **Cap:** none
- **Notes:** Default image is the unpinned taskweavercontainers/taskweaver-executor:latest, pulled automatically (see C7).

### C5 Untrusted input blast radius — 0.00 (high)

Nothing in TaskWeaver separates untrusted content from instructions. Execution output, plugin results (the default-enabled Klarna search plugin fetches web data), and the content of any file the code reads are appended to the planner's conversation as ordinary user-role messages. Because generated code runs without approval in a container with full network egress, an injected instruction can both leak the user's data and take irreversible external actions unattended.

- **S L0:** No structural limit; the only defences are prompt instructions to the model. — [taskweaver/code_interpreter/code_interpreter/code_generator_prompt.yaml:25](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/code_interpreter/code_interpreter/code_generator_prompt.yaml#L25); searched `rg -n -i 'untrusted|injection' -g '*.py'` in `taskweaver` → 0 hits (No provenance or taint handling in code.) (verified)
  - *To reach the next level:* No rule-of-two enforcement or approval once untrusted content enters the session.
- **C L0:** Worker and tool output enters context with the same role as the user's own messages. — [taskweaver/planner/planner.py:169-176](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/planner/planner.py#L169-L176) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished from principal input.
- **D L0:** No control exists to enable. — [taskweaver/planner/planner.py:169-176](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/planner/planner.py#L169-L176) (verified)
  - *To reach the next level:* No default-on control.
- **B L0:** Exfiltration (arbitrary outbound HTTP from code) and irreversible actions (external API calls, file deletion in the workspace) both happen unattended. — [taskweaver/code_interpreter/code_interpreter/code_interpreter.py:267-272](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/code_interpreter/code_interpreter/code_interpreter.py#L267-L272); [taskweaver/ces/environment.py:257-279](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L257-L279); [project/plugins/klarna_search.py:21](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/project/plugins/klarna_search.py#L21) (verified)
  - *To reach the next level:* Exfiltration or irreversible actions should require approval.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.55 (high)

In the default configuration conversation memory lives only in the running session, and the long-term 'experience' feature is off; when enabled, experiences are only created when the user runs /save, but the saved chat (including any injected content) is LLM-summarised and re-injected into every later session's prompt for that project, with no provenance or expiry. Config, plugins and examples load from the project directory, which the README has the operator pass explicitly; when -p is omitted TaskWeaver walks up from the current directory to find a taskweaver_config.json. In the default container mode generated code cannot write the project directory.

- **S L2:** Persistent experience writes require the user's /save, but the content is not validated, carries no provenance, never expires, and is injected as trusted prompt context. — [taskweaver/chat/console/chat.py:478](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/chat/console/chat.py#L478); [taskweaver/role/role.py:50-53](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/role/role.py#L50-L53) (verified)
  - *To reach the next level:* No validation/expiry of saved experience and no provenance marking when it is re-injected.
- **C L2:** The experience store is human-gated; static examples and the project config/plugins load silently from the project directory (discovered upward from cwd when -p is omitted). — [taskweaver/utils/app_utils.py:37-45](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/utils/app_utils.py#L37-L45) (verified)
  - *To reach the next level:* Auto-loaded project config and examples have no trust decision when discovered implicitly.
- **D L2:** Session memory is per-session in process; experience, when on, is one directory shared by every session of the project. — [taskweaver/session/session.py:23-26](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/session/session.py#L23-L26); [taskweaver/app/session_store.py:29-30](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/app/session_store.py#L29-L30) (verified)
  - *To reach the next level:* No per-user namespace for experience; the model can't change isolation but nothing enforces per-user separation either.
- **B L3:** With the shipped default (experience off) poisoned context is session-scoped and the container cannot write project files. — [taskweaver/role/role.py:50-53](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/role/role.py#L50-L53); [taskweaver/ces/environment.py:257-279](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L257-L279) (verified)
  - *To reach the next level:* When experience is enabled a poisoned save persists across all users of the project and drives code generation; no rollback.
- **Cap:** none

### C7 Third-party extensions — 0.17 (high)

Plugins are Python files placed in the project's plugins folder; any YAML there with enabled: true is picked up by glob and its code executed in the same kernel as model code, with no pinning, hash or consent prompt. Two shipped plugins are enabled by default. The sandbox image itself is pulled as an unpinned ':latest' tag from Docker Hub and silently re-pulled whenever the registry copy changes. Generated code can also pip-install packages inside the container because network is open.

- **S L1:** User-chosen sources but nothing pinned: plugins are local files with no integrity check, and the executor image is ':latest', re-pulled when the registry changes. — [taskweaver/ces/environment.py:114](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L114); [taskweaver/ces/environment.py:160-171](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L160-L171) (verified)
  - *To reach the next level:* Pin the executor image by digest and record plugin hashes.
- **C L0:** No extension type (plugins, ext roles, executor image) is verified. — searched `rg -n -i 'sha256|checksum|signature|digest' -g '*.py'` in `taskweaver` → 2 hits (Both hits are md5 helpers for caching/mocks, not verification.) (verified)
  - *To reach the next level:* Integrity verification for at least plugins and the image.
- **D L1:** Plugins are enabled by dropping files into the project folder and auto-globbed with no display of what will run; two plugins and image auto-updates are on by default. — [taskweaver/memory/plugin.py:320](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/memory/plugin.py#L320); [project/plugins/klarna_search.yaml:2](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/project/plugins/klarna_search.yaml#L2) (verified)
  - *To reach the next level:* Adding a plugin should show the exact code/permissions and require explicit consent.
- **B L1:** Plugins run in-process in the shared kernel with model code and every other plugin's configuration (inside the container by default). — [taskweaver/ces/environment.py:616-625](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L616-L625); [taskweaver/code_interpreter/code_executor.py:132-144](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/code_interpreter/code_executor.py#L132-L144) (verified)
  - *To reach the next level:* Per-plugin process isolation with only its own configuration.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.25 (high)

The LLM API key is read from the project's taskweaver_config.json in plaintext or from environment variables, and nothing in the codebase masks or redacts secrets. The default container mode keeps host environment variables out of the code kernel, which is the one protected path. Every LLM prompt is dumped to a JSON file in the session directory by default and the kernel writes a DEBUG-level log; remote telemetry and tracing are opt-in.

- **S L1:** Secrets come from a plaintext config file or env vars; no masking or redaction code anywhere. — [project/taskweaver_config.json:3](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/project/taskweaver_config.json#L3); searched `rg -n -i 'redact|mask_secret|SecretStr' -g '*.py'` in `taskweaver` → 0 hits (verified)
  - *To reach the next level:* No type-level masking or log redaction.
- **C L1:** Only the subprocess-environment path is protected (container mode scrubs env); logs, prompt dumps and model-bound messages are not. — [taskweaver/ces/environment.py:240-249](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L240-L249) (verified)
  - *To reach the next level:* Redaction of logs and prompt dumps.
- **D L1:** Remote telemetry is opt-in, but full prompt dumps per LLM call and DEBUG kernel logging are on by default and unredacted. — [taskweaver/planner/planner.py:362-363](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/planner/planner.py#L362-L363); [taskweaver/ces/kernel/kernel_logging.py:4-8](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/kernel/kernel_logging.py#L4-L8); [taskweaver/logging/__init__.py:22](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/logging/__init__.py#L22) (verified)
  - *To reach the next level:* Content logging should be off or redacted by default.
- **B L1:** A leaked LLM provider key is long-lived and spends on the operator's account; plugin keys sit in the kernel model code shares. — [taskweaver/llm/base.py:42-43](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/llm/base.py#L42-L43); [taskweaver/ces/environment.py:616-625](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L616-L625) (verified)
  - *To reach the next level:* Scoped, short-lived credentials.
- **Cap:** none

### C9 Audit & traceability — 0.42 (high)

TaskWeaver writes a plain-text log in the project's logs folder that records every piece of code before it runs and every message between planner and workers, plus a JSON dump of every LLM prompt in the session directory. In the default container mode these files sit outside what generated code can reach. There is no structured per-action record with result status, no actor or approver attribution, and OpenTelemetry tracing is opt-in.

- **S L1:** Unstructured INFO text log of executed code and inter-role messages; JSON prompt dumps. — [taskweaver/code_interpreter/code_interpreter/code_interpreter.py:267](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/code_interpreter/code_interpreter/code_interpreter.py#L267); [taskweaver/session/session.py:209-212](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/session/session.py#L209-L212) (verified)
  - *To reach the next level:* Structured record of each execution with arguments, result status and timestamps.
- **C L2:** All code execution goes through the code-interpreter roles that log it, so plugin calls are captured inside the logged code. — [taskweaver/code_interpreter/code_interpreter/code_interpreter.py:267](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/code_interpreter/code_interpreter/code_interpreter.py#L267); [taskweaver/code_interpreter/code_interpreter/code_generator.py:411-412](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/code_interpreter/code_interpreter/code_generator.py#L411-L412) (verified)
  - *To reach the next level:* No record of approvals/denials (none exist) and ext roles log inconsistently.
- **D L2:** On by default and written by the host process to project/logs, outside the container mounts. — [taskweaver/logging/__init__.py:30](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/logging/__init__.py#L30); [taskweaver/ces/environment.py:257-279](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L257-L279) (verified)
  - *To reach the next level:* Logs are ordinary files any local-mode code or the host user can edit; nothing tamper-evident.
- **B L2:** Python FileHandler writes each record as it is emitted, before execution; failures go to stderr and execution continues. — [taskweaver/logging/__init__.py:109-118](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/logging/__init__.py#L109-L118) (verified)
  - *To reach the next level:* Durable replayable trajectory or fail-closed logging for risky actions.
- **Cap:** none
- **Notes:** Opt-in OpenTelemetry tracing (module/tracing.py:16) records code as span attributes.

### C10 Limits & kill switch — 0.45 (high)

Each user request is capped at 10 internal planner/worker exchanges and three retries per code attempt, and the host stops waiting for a code execution after 180 seconds of silence. There is no token, cost or wall-clock budget, and the timeout does not interrupt the kernel, as a TODO in the code admits, so long-running code keeps going until the session is stopped and its container removed. The container has no CPU or memory limits.

- **S L2:** Iteration cap (max_internal_chat_round_num=10) plus a 180 s per-message execution wait, enforced in code. — [taskweaver/session/session.py:217-220](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/session/session.py#L217-L220); [taskweaver/ces/environment.py:544-549](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L544-L549) (verified)
  - *To reach the next level:* No token/cost or wall-clock cap; timeout does not interrupt execution.
- **C L2:** Applies to the top-level round loop and to every kernel execution. — [taskweaver/session/session.py:22](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/session/session.py#L22); [taskweaver/ces/environment.py:549](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L549) (verified)
  - *To reach the next level:* Spawned processes inside the kernel are not bounded.
- **D L2:** Sensible defaults, operator-configurable; the model cannot raise them. — [taskweaver/session/session.py:22](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/session/session.py#L22) (verified)
  - *To reach the next level:* No hard ceiling configuration cannot exceed, and nothing bounds rounds per session.
- **B L1:** Timed-out code keeps running in the kernel and background processes persist until the session ends; no spend ceiling. — [taskweaver/ces/environment.py:544](https://github.com/microsoft/taskweaver/blob/d44ddef23f90059fb17999d3095db4240e98f955/taskweaver/ces/environment.py#L544) (verified)
  - *To reach the next level:* Interrupt in-flight execution on timeout/stop and add a spend ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Execution and plugin output re-enter the planner as user-role messages (taskweaver/planner/planner.py:169-176) · [B] sensitive data/systems: User data files in the session working directory, plugin configs in the kernel (taskweaver/ces/environment.py:616-625) · [C] state change / egress: Unapproved code execution with default Docker network egress (taskweaver/ces/environment.py:257-279) · Same default session? Yes

## Highest-impact improvements
1. Add a per-execution human approval step showing the exact code before it runs (on by default for interactive use). — C2 S L0→L3, +0.225 before caps (Playbook 5)
2. Start the executor container with network disabled (or proxy-allowlisted) and CPU/memory/PID limits. — C4 B L2→L3, +0.050 before caps (Playbook 3 step 1)
3. Harden the container: cap_drop ALL, no-new-privileges, read-only root, non-root entrypoint, kernel ports bound to 127.0.0.1. — C4 S L2→L3, +0.075 before caps (Playbook 3)
4. Interrupt the kernel on timeout and add token/cost and wall-clock budgets per session. — C10 S L2→L3, +0.075 before caps (Playbook 3 step 3)
5. Pin the executor image by digest instead of auto-pulling ':latest'. — C7 S L1→L2, +0.075 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Optional roles (web_search, web_explorer, document_retriever, recepta, image_reader) and the playground UIs were skimmed, not fully audited; document_retriever pickle-loads an operator-configured index.
- Behaviour of docker-py port publishing (binds 0.0.0.0 when host port is None) and Jupyter HMAC protection is inferred from library behaviour.
- No text aimed at AI reviewers was found in the repository.
